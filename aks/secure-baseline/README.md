# AKS Contoso Bicycle (Secure Baseline)

## Introduction

This reference implementation shows the recommended architecture for a typical
organization moving containerized business applications to an AKS cluster
with security in mind.

This is meant to guide an interdisciplinary team or multiple teams like networking,
security and development through a fictional process of getting this secure baseline
infrastructure up and running.

### Guidance

This project has a companion set of articles that describe challenges, design patterns, and best practices for a Secure AKS cluster. You can find this article on the Azure Architecture Center:

[Baseline architecture for a secure AKS cluster](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks/secure-baseline/)

## Architecture

This architecture is mainly concentrated in the AKS cluster identity, secret
management and keep end-to-end traffic securely.

An AKS cluster can be smoothly integrated with other Azure services that will
deliver observability, provide a network topology thinking about
a multi-regional growing and keep the in-cluster traffic secure as well.

Additionally, GitOps is paramount for the cluster management lifecycle so
this another key topic that will be also handled as part of this Reference Implementation.

Contoso Bicycle is a fictional small and fast-growing startup that provides
online web services to its clientele in the west coast, North America.
They have no on-premises datacenters and all their containerized line of
business applications are now about to be orchestrated by a Secure AKS cluster.

Contoso Bicycle is planning to use the [ASPNET Core Docker sample web app](https://github.com/dotnet/dotnet-docker/tree/master/samples/aspnetapp)
as a first experiment that will help them to evaluate and test their new infrastructure.
This is the only part that is not going to reflect a real-world application.

### Core components that compose this baseline:

Azure platform:
1. AKS v1.17.X
   * System and User nodepool
   * AKS-managed Azure AD integration
   * Managed Identities
   * Azure CNI
   * Azure Monitor for Containers
1. Azure Virtual Networks
1. Azure Application Gateway
1. AKS-managed Internal Load Balancers
1. Azure Firewall

In-cluster components:

1. Flux v1.19.0
1. Traefik Ingress Controller v2.2.1
1. AAD Pod Identity v1.6.1
1. Azure KeyVault CSI Provider v0.0.6
1. Kured v1.4.0

![](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks/secure-baseline/images/baseline-network-topology.png)

## Install the Secure AKS cluster baseline Reference Implementation

### Prequisites

1. An Azure subscription. If you don't have an Azure subscription, you can create a [free account](https://azure.microsoft.com/free).
   > Important: the user initiating the deployment process must have the following roles:
   > 1. to deploy the Secure AKS cluster: `Microsoft.Authorization/roleAssignments/write` permission. For more information, please refer to [the Container Insights doc](https://docs.microsoft.com/en-us/azure/azure-monitor/insights/container-insights-troubleshoot#authorization-error-during-onboarding-or-update-operation)
   > 1. to integrate AKS-managed Azure AD: [User Administrator](https://docs.microsoft.com/en-us/azure/active-directory/users-groups-roles/directory-assign-admin-roles#user-administrator-permissions). If you are not part of the `User Administrator` group from the Tenant associated to your Azure subscription, please consider [creating a new one](https://docs.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-access-create-new-tenant#create-a-new-tenant-for-your-organization).
1. [Azure CLI installed](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) or try from shell.azure.com by clicking below

   <a href="https://shell.azure.com" title="Launch Azure Cloud Shell"><img name="launch-cloud-shell" src="https://docs.microsoft.com/azure/includes/media/cloud-shell-try-it/launchcloudshell.png" /></a>

1. [Register the AAD-V2 feature for AKS-managed Azure AD](https://docs.microsoft.com/en-us/azure/aks/managed-aad#before-you-begin)
1. Clone or download this repo locally.
   ```bash
   git clone https://github.com/mspnp/reference-architectures.git && \
   cd reference-architectures/aks/secure-baseline
   ```

   > :bulb: Tip: The deployment steps shown here use Bash shell commands. On Windows, you can use the [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/about#what-is-wsl-2) to run Bash.

1. [OpenSSL](https://github.com/openssl/openssl#download)

### Acquire the Contoso Bicycle CA certificates

1. Generate a CA self-signed cert

   > Contoso Bicycle needs to buy CA certificates, they preference is to
   > own two different TLS certificates. The first one is going to be serve
   > in front of the Azure Application Gateway and another one at the
   > Ingress Controller level. This is something that can be prefectly achieved
   > when configuring Azure Application Gateway End-to-end TLS encryption.

   > :warning: WARNING
   > Do not use the certificates created by these scripts for production. The certificates are provided for demonstration purposes only. For your production cluster, use your security best practices for digital certificates creation and lifetime management.
   > Self-signed certificates are not trusted by default and they can be difficult to maintain. Also, they may use outdated hash and cipher suites that may not be strong. For better security, purchase a certificate signed by a well-known certificate authority.

   Cluster Ingress Controller Certificate: `*.aks-ingress.contoso.com`

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
         -out traefik-ingress-internal-aks-ingress-contoso-com-tls.crt \
         -keyout traefik-ingress-internal-aks-ingress-contoso-com-tls.key \
         -subj "/CN=*.aks-ingress.contoso.com/O=Contoso Aks Ingress" && \
   rootCertWilcardIngressController=$(cat traefik-ingress-internal-aks-ingress-contoso-com-tls.crt | base64 -w 0)
   ```

   Azure Application Gateway Certificate: `bicycle.contoso.com`

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
          -out appgw.crt \
          -keyout appgw.key \
          -subj "/CN=bicycle.contoso.com/O=Contoso Bicycle" && \
   openssl pkcs12 -export -out appgw.pfx -in appgw.crt -inkey appgw.key -passout pass: && \
   appGatewayListernerCertificate=$(cat appgw.pfx | base64 -w 0)
   ```

### Create the Secure AKS cluster

1. Query your tenant ids
   ```bash
   export TENANT_ID=$(az account show --query tenantId --output tsv)

   # Login into the tenant you are User Admistrator. Re-use the TENANT_ID
   # env var if your are User Administrator from the Azure subscription tenant
   az login --tenant <tenant-id-with-user-admin-permissions> --allow-no-subscriptions

   export K8S_RBAC_AAD_PROFILE_TENANTID=$(az account show --query tenantId --output tsv)
   ```
   > :bulb: Tip: you could execute [scripts](./deploy) to get the following
   > infrastructure assets provisioned. However,for a better experience,
   > we do recommend to execute the following steps manuallly.

1. Create a [new AAD user and group](./aad/aad.azcli) for Kubernetes RBAC purposes
   > :bulb: Tip: execute this step from VSCode for a better experience. Discard
   > this if you are using Azure Cloud Shell
1. Provision [a regional hub and spoke virtual networks](./networking/network-deploy.azcli)
   > :bulb: Tip: execute this step from VSCode for a better experience. Discard
   > this if you are using Azure Cloud Shell
1. create [the BU 0001's app team secure AKS cluster (ID: A0008)](./cluster-deploy.azcli)
   > :bulb: Tip: execute this step from VSCode for a better experience

### Flux as the GitOps solution

   > The Flux's operator user from the Gitops team wants to deploy Flux
   > for the Secure AKS Cluster (Application ID: 0008 under the BU001).
   > But before executing this action, this user  checks for open PRs againts the
   > cluster's IaaC git repository looking for authored Kubernets manifest files coming from
   > the Azure AD team Admin team or any other. After merging all the PRs, the FLux's operator
   > can procceed with the deployment. The Kubernetes namespace
   > `cluster-baseline-settings` the desired logical division of the cluster
   > to home Flux and any other baseline setting among others:
   >   - Cluster Role Bindings for the AKS-managed Azure AD integration
   >   - AAD Pod Identity
   >   - CSI driver and Azure KeyVault CSI Provider
   >   - the App team (Application ID: 0008) namespace named a0008

1. Install kubectl 1.18 or later
   ```bash
   sudo az aks install-cli

   # ensure you got a version 1.18 or greater
   kubectl version --client
   ```
1. Get AKS Secure cluster  name
   ```bash
   export AKS_CLUSTER_NAME=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.aksClusterName.value -o tsv)
   ```
1. Get AKS Kubeconfig Credntials
   ```bash
   az aks get-credentials -n $AKS_CLUSTER_NAME -g rg-bu0001a0008 --admin
   ```
1. Deploy Flux
   ```bash
   kubectl create namespace cluster-baseline-settings
   kubectl apply -f https://raw.githubusercontent.com/mspnp/reference-architectures/master/aks/secure-baseline/cluster-baseline-settings/flux.yaml
   kubectl wait --namespace cluster-baseline-settings \
     --for=condition=ready pod \
     --selector=app.kubernetes.io/name=flux \
     --timeout=90s
   ```

### Traefik Ingress Controller with Azure KeyVault CSI integration

> The app team knows that sooner rather than later they will need
> to expose their backend services outside their AKS cluster. Therefore,
> they are tasked to deploy an Ingress Controller and their preference is Traefik.
> They want to manage their secrets in a very secure manner, so they opted to use
> the Azure KeyVault CSI Provider to mount their TLS certificate that
> happens to be are stored in Azure KeyVault.

```bash
# Get the AKS Ingress Controller User Assigned Identity details
export TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.aksIngressControllerUserManageIdentityResourceId.value -o tsv)
export TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.aksIngressControllerUserManageIdentityClientId.value -o tsv)

# Get Azure KeyVault name
export KEYVAULT_NAME=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.keyVaultName.value -o tsv)

# Ensure Flux has created the following namespace and then press Ctrl-C
kubectl get ns a0008 -w

# Create the traefik Azure Identity and the Azure Identity Binding to let
# Azure Active Directory Pod Identity to get tokens on behalf of the Traefik's User Assigned
# Identity and later on assign them to the Traefik's pod
cat <<EOF | kubectl apply -f -
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentity
metadata:
  name: aksic-to-keyvault-identity
  namespace: a0008
spec:
  type: 0
  resourceID: $TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID
  clientID: $TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID
---
apiVersion: "aadpodidentity.k8s.io/v1"
kind: AzureIdentityBinding
metadata:
  name: aksic-to-keyvault-identity-binding
  namespace: a0008
spec:
  azureIdentity: aksic-to-keyvault-identity
  selector: traefik-ingress-controller
EOF

# Create a `SecretProviderClasses` resource with with your Azure KeyVault parameters
# for the Secrets Store CSI driver.
cat <<EOF | kubectl apply -f -
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: aks-ingress-contoso-com-tls-secret-csi-akv
  namespace: a0008
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"
    keyvaultName: "${KEYVAULT_NAME}"
    objects:  |
      array:
        - |
          objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
          objectAlias: tls.crt
          objectType: cert
        - |
          objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
          objectAlias: tls.key
          objectType: secret
    tenantId: "${TENANT_ID}"
EOF

# Install Traefik ingress controller with Azure CSI Provider to obtain
# the TLS certificates as the in-cluster secret management solution.

kubectl apply -f https://raw.githubusercontent.com/mspnp/reference-architectures/master/aks/workload/traefik.yaml

# Wait for Traefik to be ready
# During the Traefik's pod creation time, aad-pod-identity will need to create the AzureAssignedIdentity for the pod based on the AzureIdentity
# and AzureIdentityBinding, retrieve token for Azure KeyVault. This process can take time to complete and it's possible
# for the pod volume mount to fail during this time but the volume mount will eventually succeed.
# For more information, please refer to https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/pod-identity-mode.md

kubectl wait --namespace a0008 \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=traefik-ingress-ilb \
  --timeout=90s
```

### The ASPNET Core Docker sample web app

> The app team is about to conclude this journey, but they need an app to test
> their new infrastructure blocks. For this task they picked out the
> [ASPNET Core Docker sample web app](https://github.com/dotnet/dotnet-docker/tree/master/samples/aspnetapp).
> Addittionally, they will include as part of the desired configuration for it
> some of the following concepts:
> - Ingress resource object
> - Network Policy to allow Ingress Controller establish connection with the app

```bash
kubectl apply -f https://raw.githubusercontent.com/mspnp/reference-architectures/master/aks/secure-baseline/workload/aspnetapp.yaml

# The ASPNET Core Docker sample web app is all setup. Wait until is ready to process requests running:
kubectl wait --namespace a0008 \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/name=aspnetapp \
  --timeout=90s

# In this momment your Ingress Controller (Traefik) is reading your Ingress
# resource object configuration, updating its status and creating a router to
# fulfill the new exposed workloads route.
# Please take a look at this and notice that the Address is set with the Internal Load Balancer Ip from
# the configured subnet

kubectl get ingress aspnetapp-ingress -n a0008

# Validate the router to the workload is configured, SSL offloading and allowing only known Ips
# Please notice only the Azure Application Gateway is whitelisted as known client for
# the workload's router. Therefore, please expect a Http 403 response
# as a way to probe the router has been properly configured

kubectl -n a0008 run -i --rm --tty curl --image=curlimages/curl -- sh
curl --insecure -k -I --resolve bu0001a0008-00.aks-ingress.contoso.com:443:10.240.4.4 https://bu0001a0008-00.aks-ingress.contoso.com
exit 0
```

### Test the web app

> The app team conducts a final acceptance test to be sure that traffic is
> flowing end-to-end as expected, so they place a request against the Azure
> Application Gateway

```bash
# query the BU 0001's Azure Application Gateway Public Ip

export APPGW_PUBLIC_IP=$(az deployment group show --resource-group rg-enterprise-networking-spokes -n spoke-BU0001A0008 --query properties.outputs.appGwPublicIpAddress.value -o tsv)
```

1. Map the Azure Application Gateway public ip address to the application domain names. To do that, please open your hosts file (C:\windows\system32\drivers\etc\hosts or /etc/hosts) and add the following record in local host file:
   \${APPGW_PUBLIC_IP} bicycle.contoso.com

1. In your browser navigate the site anyway (A warning will be present)
   https://bicycle.contoso.com/

## Clean up

> Once the test phase is carried out, the actual Contoso Bicycle line of
> business application ID 0008 could be deployed to its new AKS cluster for
> the Business Unit 001

```bash
az group delete -n rg-bu0001a0008 --yes && \
az group delete -n rg-enterprise-networking-spokes --yes && \
az group delete -n rg-enterprise-networking-hubs --yes && \
az keyvault purge --name $KEYVAULT_NAME --location eastus2 --yes
```
