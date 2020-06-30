# Workflow Prerequisites

## Generate the Ingress Controller TLS Certificate

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
Next Step: [Secret Managment and Ingress Controller](./08-secret-managment-and-ingress-controller.md)
