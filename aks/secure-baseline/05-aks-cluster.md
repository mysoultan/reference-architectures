# Create the Secure AKS cluster

## Deploy - Option 1 Azure Portal

use the deploy to azure button to create the cluster from the Azure Portal

[![Deploy to Azure](https://docs.microsoft.com/en-us/azure/templates/media/deploy-to-azure.svg)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmspnp%2Freference-architectures%2Ffcp%2Faks-baseline%2Faks%2Fsecure-baseline%2Fcluster-stamp.json)

---
-> Navigate: [GitOps](./06-gitops.md)

## Deploy - Option 2 CLI

1. Create the AKS cluster resource group
   > The app team working on behalf of business unit 0001 (BU001) is looking to create an AKS cluster
   > of the app they are creating (Application ID: 0008). They have worked with the organization's
   > networking team and have been provisioned a spoke network in which to lay their cluster and
   > network-aware external resources into (such as Application Gateway). They took that information
   > and added it to their cluster-stamp.json and parameters file.

   > They create this resource group to be the parent group for the application

   ```bash
   # [This takes less than one minute.]
   az group create --name rg-bu0001a0008 --location eastus2
   ```

1. Get the AKS cluster spoke vnet id

   > the app team will bring its own vNet. This is the spoke vnet that had been already
   > provisioned by the network team

   ```bash
   TARGET_VNET_RESOURCE_ID=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0008 --query properties.outputs.clusterVnetResourceId.value -o tsv)
   ```
1. Create the AKS cluster
   > And then deploy the cluster into it.

   ```bash
   # [This takes about 15 minutes.]

   # before executing this command please edit the
   az deployment group create --resource-group rg-bu0001a0008 --template-file ./cluster-stamp.json --parameters targetVnetResourceId=$TARGET_VNET_RESOURCE_ID k8sRbacAadProfileAdminGroupObjectID=$K8S_RBAC_AAD_ADMIN_GROUP_OBJECTID k8sRbacAadProfileTenantId=$K8S_RBAC_AAD_PROFILE_TENANTID appGatewayListernerCertificate=$APP_GATEWAY_LISTERNER_CERTIFICATE
   ```
1. Obtain the Azure KeyVault details and give the current user permissions to
   create certificates.

   > Finally the app team wants to import a wildcard certificate `*.aks-ingress.contoso.com`  to AzureKeyVault
   > A while later this certificate is going to be the one served by a Traefik Ingress Controller wich is
   > deployed downstream

   ```bash
   KEYVAULT_NAME=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.keyVaultName.value -o tsv)
   az keyvault set-policy --certificate-permissions create -n $KEYVAULT_NAME --upn $(az account show --query user.name -o tsv)
   ```
1. Generate the wildcard certificate for the Ingress Controller using Azure KeyVault

   > Contoso Bicycle needs to buy CA certificate which is a standard TLS cert at the Ingress Controller level.

   > :warning: Do not use the certificates created by these scripts for actual deployments. The self-signed certificates are provided for ease of illustration purposes only. For your cluster, use your organization's requirements for procurement and lifetime management of TLS certificates, even for development purposes.

   Cluster Ingress Controller Wildcard Certificate: `*.aks-ingress.contoso.com`

   ```bash
   cat <<EOF | az keyvault certificate create --vault-name $KEYVAULT_NAME -n traefik-ingress-internal-aks-ingress-contoso-com-tls -p @-
   {
     "issuerParameters": {
       "certificateTransparency": null,
       "name": "Self"
     },
     "keyProperties": {
       "curve": null,
       "exportable": true,
       "keySize": 2048,
       "keyType": "RSA",
       "reuseKey": true
     },
     "lifetimeActions": [
       {
         "action": {
           "actionType": "AutoRenew"
         },
         "trigger": {
           "daysBeforeExpiry": 90
         }
       }
     ],
     "secretProperties": {
       "contentType": "application/x-pkcs12"
     },
     "x509CertificateProperties": {
       "keyUsage": [
         "cRLSign",
         "dataEncipherment",
         "digitalSignature",
         "keyEncipherment",
         "keyAgreement",
         "keyCertSign"
       ],
       "subject": "O=Contoso Aks Ingress, CN=*.aks-ingress.contoso.com",
       "validityInMonths": 12
     }
   }
   EOF
   ```

1. Query the BU 0001's Azure Application Gateway Name

    ```bash
    export APP_GATEWAY_NAME=$(az deployment group show -g rg-bu0001a0008 -n cluster-stamp-bu0001a0008 --query properties.outputs.agwName.value -o tsv)
    ```

1. Configure the trusted root cert at the Azure Application Gateway level

   ```bash
   az network application-gateway root-cert create -g rg-bu0001a0008 --gateway-name $APP_GATEWAY_NAME --name root-cert-wildcard-aks-ingress-contoso --keyvault-secret $(az keyvault certificate show --vault-name $KEYVAULT_NAME -n traefik-ingress-internal-aks-ingress-contoso-com-tls --query id -o tsv)
   ```
---
-> Navigate: [GitOps](./06-gitops.md)
