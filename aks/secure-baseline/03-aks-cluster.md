### Create the Secure AKS cluster

1. Query your tenant id

   ```bash
   export TENANT_ID=$(az account show --query tenantId --output tsv)

   # Login into the tenant where you are a User Administrator. Re-use the TENANT_ID
   # env var if your are User Administrator from the Azure subscription tenant
   az login --tenant <tenant-id-with-user-admin-permissions> --allow-no-subscriptions

   export K8S_RBAC_AAD_PROFILE_TENANTID=$(az account show --query tenantId --output tsv)
   ```

1. Create a [new AAD user and group](./inner-loop-scripts/azcli/aad/aad.azcli) for Kubernetes RBAC purposes
   > :bulb: You can execute `.azcli` files from Visual Studio Code.
1. Provision [a regional hub and spoke virtual network](./inner-loop-scripts/azcli/network-deploy.azcli)
1. Create [the baseline AKS cluster](./inner-loop-scripts/azcli/cluster-deploy.azcli)

-> Nativate: [GitOps](./04-gitops.md)
