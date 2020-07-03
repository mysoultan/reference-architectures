# Create the Secure AKS cluster

Previously you have provisioned [the hub and spoke vnets](./04-networking). Now it is time to create
the AKS cluster.

---

## Deploy - Option 1 Azure Portal

use the deploy to azure button to create the cluster from the Azure Portal

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmspnp%2Freference-architectures%2Ffcp%2Faks-baseline%2Faks%2Fsecure-baseline%2Fcluster-stamp.json)

---
Next Step: [GitOps](./06-gitops.md)

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
   az deployment group create --resource-group rg-bu0001a0008 --template-file cluster-stamp.json --parameters targetVnetResourceId=$TARGET_VNET_RESOURCE_ID k8sRbacAadProfileAdminGroupObjectID=$K8S_RBAC_AAD_PROFILE_ADMIN_GROUP_OBJECTID k8sRbacAadProfileTenantId=$K8S_RBAC_AAD_PROFILE_TENANTID appGatewayListernerCertificate=$APP_GATEWAY_LISTERNER_CERTIFICATE
   ```

---
Next Step: [GitOps](./06-gitops.md)

## Deploy - Option 3 GitHub Actions (fork is required)

1. Create the Azure Credentials for the GitHub CD workflow

   ```bash
   # Create an Azure Service Principal
   az ad sp create-for-rbac --name "github-workflow-aks-cluster" --sdk-auth --skip-assignment > sp.json
   export APP_ID=$(grep -oP '(?<="clientId": ").*?[^\\](?=",)' sp.json)

   # Wait for it to get propagated
   until az ad sp show --id ${APP_ID} &> /dev/null ; do echo "Waiting for Azure AD propagation" && sleep 5; done

   # assign the following Azure Role-Based Access Control (RBAC) built-in role for creating resource groups and place deployments at subscription level
   az role assignment create --assignee $APP_ID --role 'Contributor'

   # assign the following Azure Role-Based Access Control (RBAC) built-in role  since granting RBAC access to other resources during the cluster creation will be required at subscription level (e.g. AKS-managed Internal Load Balancer, ACR, Managed Identities, etc.)
   az role assignment create --assignee $APP_ID --role 'User Access Administrator'
   ```

   > :bulb: you are going to need the content from `sp.json` file to create the `AZURE_CREDENTIALS` secret from your GitHub repository.

1. Copy the file GitHub workflow into the proper directory

   ```bash
   mkdir -p .github/workflows
   cp .github/workflows/aks-deploy.yaml
   ```

   > :bulb: you might want to convert this GitHub workflow into a template since your organization might need to handle multiple AKS clusters.
   > For more information, please take a look at [Sharing Workflow Templates within your organization](https://docs.github.com/en/actions/configuring-and-managing-workflows/sharing-workflow-templates-within-your-organization)

1. Open the workflow file `.github/workflows/aks-deploy.yaml` and follow the intructions

1. Push the changes to your forked repo

   ```bash
   git add -u && git commit -m "setup GitHub CD workflow"
   git push origin HEAD:remotes/origin/master
   ```

1. Open a PR against `master`. Once the GitHub wokflow finished successfully, please merge into master. This will trigger the AKS cluster creation.

---
Next Step: [Workflow Prerequisites](./07-workload-prerequisites.md)

