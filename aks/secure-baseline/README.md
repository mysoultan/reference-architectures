# Azure Kubernetes Service (AKS) Secure Baseline Reference Implementation

This reference implementation demonstrates the _recommended infrastructure architecture_ for hosting applications on an [AKS cluster](https://azure.microsoft.com/services/kubernetes-service).

This is meant to guide an interdisciplinary team or multiple teams like networking, security and development through the process of getting this secure baseline infrastructure deployed.

## Guidance

This project has a companion set of articles that describe challenges, design patterns, and best practices for a secure AKS cluster. You can find this article on the Azure Architecture Center at [Baseline architecture for a secure AKS cluster](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks/secure-baseline/).

## Architecture

This architecture is infrastructure focused, more so than workload. It mainly concentrates on the AKS cluster itself, including identity, post-deployment configuration, secret management, and network considerations.

The implementation presented here is the _minimum recommended baseline_ for any AKS cluster. This implementation integrates with Azure services that will deliver observability, provide a network topology that will support multi-regional growth, and keep the in-cluster traffic secure as well. This is not a "dev" or a "production" cluster. Instead consider starting your AKS journey with this baseline infrastructure and then apply your specific scenario and requirements on top of it, pre-production and production.

Much like the cluster itself is managed via declarative Infrastructure as Code (IaC), we recommend customers adopt a GitOps process for inner-cluster configuration management. An implementation of this is demonstrated in this reference, using the Open Sourced Software (OSS) [Flux](https://fluxcd.io).

Throughout the reference implementation, you will see reference to Contoso Bicycle.  They are a fictional small and fast-growing startup that provides online web services to its clientele on the west coast of North America. They have no on-premises data centers and all their containerized line of business applications are now about to be orchestrated by secure, enterprise-ready AKS clusters.

This implementation uses the [ASP.NET Core Docker sample web app](https://github.com/dotnet/dotnet-docker/tree/master/samples/aspnetapp) as an example workload. This workload purposefully uninteresting, as it is here exclusively to help you experience the baseline infrastructure.

### Core components that compose this baseline

#### Azure platform

* AKS v1.17
  * System and User nodepool separation
  * AKS-managed Azure AD integration
  * Managed Identities
  * Azure CNI
  * Azure Monitor for Containers
* Azure Virtual Networks (hub-spoke)
* Azure Application Gateway (WAF)
* AKS-managed Internal Load Balancers
* Azure Firewall

#### In-cluster OSS components

* [Flux GitOps Operator](https://fluxcd.io)
* [Traefik Ingress Controller](https://docs.microsoft.com/azure/dev-spaces/how-to/ingress-https-traefik)
* [Azure AD Pod Identity](https://github.com/Azure/aad-pod-identity)
* [Azure KeyVault Secret Store CSI Provider](https://github.com/Azure/secrets-store-csi-driver-provider-azure)
* [Kured](https://docs.microsoft.com/azure/aks/node-updates-kured)

![TODO, Apply Description](https://docs.microsoft.com/azure/architecture/reference-architectures/containers/aks/secure-baseline/images/baseline-network-topology.png)

## Getting started

A deployment of AKS-hosted workloads typically has a separation of prerequisites, the infrastructure, and finally the workload. This reference implementation is similar. Also, be aware while there may seem like a lot of steps, our primary purpose is to illustrate the topology and decisions of a baseline cluster. Lifecycle management of your cluster will depend on your situation (team roles, organizational standards, etc).

Please start this journey in the **Preparing for the cluster** section.

### Preparing for the cluster

There are considerations that must be addressed before you start deploying your cluster. Do I have enough permissions in my subscription and AD tenant to do a deployment of this size? How much of this will be handled by my team directly vs having another team be responsible?

* [ ] Begin by ensuring you [install and meet the prerequisites](./01-prerequisites.md).
* [ ] Plan your [Azure Active Directory integration](./02-aad.md).

Now you're ready to think about the networking, see **Build target network**.

### Build target network

Microsoft's recommended baseline for a secure, baseline cluster is one in which you deploy it into a carefully planned network, sized appropriately for your needs. Organizations typically favor a traditional hub-spoke model. This may be handled by your team, or a networking team in your organization.

* [ ] Build out the [hub-spoke network](./03-networking.md) for the cluster.

Now you can move on to **Deploying the cluster**.

### Deploying the cluster

This is the heart of the guidance in this reference implementation; paired with strong recommendations on network topology found in the prior section. Here you will lay out the the Azure resources for your cluster. This includes not only the AKS service, but adjacent services such as WAF, Azure Monitor, Azure Container Registry, and Azure Key Vault. Also critical is that the cluster is placed under GitOps.

* [ ] Procure [client-facing TLS Certificate](./04-client-tls.md).
* [ ] Deploy the [AKS cluster and supporting services](./04-aks-cluster.md).
* [ ] Place the cluster under [GitOps management](./05-gitops.md).

We perform the prior steps manually here, for you to understand the involved components, you should orchestrate the prior two steps using your CI/CD pipeline, as you would any infrastructure as code (IaC). We have included [a starter GitHub workflow](./TODO) that demonstrates this.

### Deploy your workload

While the focus of this implementation is the infrastructure, without a workload deployed to the cluster it will be hard to see how these decisions come together to work as an reliable application platform for business. The deployment of this workload would typically follow a CI/CD pattern and may involve even more advanced deployment strategies (blue/green, etc). The following steps represent a manual deployment, suitable for illustration purposes of this infrastructure.

* [ ] Just like the cluster, there are some [workload prerequisites](./06-workload-prerequisites.md) to address.
* [ ] Generate [internal TLS certificate and deploy ingress solution](./07-secret-managment-and-ingress-controller.md).
* [ ] Deploy the [workload](./08-workload.md).
* [ ] [Validation](./09-validation.md) <== TODO, should this be part of the deploy workload step?

## Clean up resources

Most of the Azure resources deployed in the prior steps will incur ongoing charges unless removed.

* [ ] When you are done exploring this reference implementation, [cleanup all resources](./10-cleanup.md).

## Deployment alternatives

We have provided some sample deployment scripts that you could adapt for your own purposes while doing a POC/spike on this. Those scripts are found in the [inner-loop-scripts directory](./inner-loop-scripts). They include some additional considerations, and include some additional narrative as well. Consider checking them out. They consolidate the manual, walk-through steps performed above into combined execution steps.

## Kubernetes ecosystem acknowledgement

Kubernetes is a very flexible platform, giving infrastructure and application operators many choices to achieve their business and technology objectives. At points along your journey you will need to consider when to take dependencies on platform features, OSS solutions, support channels, and processes. The takeaway from this reference implementation is the process followed, the reasoning behind the choices, and ultimately we encourage this to be place for you to start conversations within your own team/organization on where you go from here. Start here, adapt to your specific requirements, and ultimately deliver a solution that delights your users.

## Related documentation

* [Azure Kubernetes Service Documentation](https://docs.microsoft.com/azure/aks/)
* [Microsoft Azure Well-Architected Framework](https://docs.microsoft.com/azure/architecture/framework/)
* [Microservices architecture on AKS](https://docs.microsoft.com/azure/architecture/reference-architectures/microservices/aks)
