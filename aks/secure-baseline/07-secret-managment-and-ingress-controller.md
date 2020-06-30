# Traefik Ingress Controller with Azure KeyVault CSI integration

The application is designed to be exposed outside of their AKS cluster. Therefore, an Ingress Controller must be deployed, and Traefik is selected. Since the ingress controller will be exposing a TLS certificate, they use the Azure KeyVault CSI Provider to mount their TLS certificate managed and stored in Azure KeyVault so Traefik can use it.

```bash
# Get the AKS Ingress Controller Managed Identity details
export TRAEFIK_USER_ASSIGNED_IDENTITY_RESOURCE_ID=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.aksIngressControllerUserManageIdentityResourceId.value -o tsv)
export TRAEFIK_USER_ASSIGNED_IDENTITY_CLIENT_ID=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.aksIngressControllerUserManageIdentityClientId.value -o tsv)

# Get Azure KeyVault name
export KEYVAULT_NAME=$(az deployment group show --resource-group rg-bu0001a0008 -n cluster-stamp --query properties.outputs.keyVaultName.value -o tsv)

# Ensure Flux has created the following namespace and then press Ctrl-C
kubectl get ns a0008 -w

# Create the Traefik Azure Identity and the Azure Identity Binding to let
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

# Create a SecretProviderClasses resource with with your Azure KeyVault parameters
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
# During the Traefik's pod creation time, aad-pod-identity will need to retrieve token for Azure KeyVault. This process can take time to complete and it's possible for the pod volume mount to fail during this time but the volume mount will eventually succeed. For more information, please refer to https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/pod-identity-mode.md

kubectl wait --namespace a0008 --for=condition=ready pod --selector=app.kubernetes.io/name=traefik-ingress-ilb --timeout=90s
```
---
-> Navigate: [Workload](./08-workload.md)
