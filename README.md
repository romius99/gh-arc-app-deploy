# Deploy application to Azure-Arc enabled Kuberntes cluster using 'Cluster Connect' and 'GitHub Action'

Azure-Arc enabled Kubernetes helps you to organize, inventory, manage, monitor and secure Kuberntes clusters hosted outside of Azure from a single control pane. The cluster could reside anywhere on-premises, any cloud service provider (AWS, GCP, Azure, Alibaba etc.) or in any edge location in your enterprise. Read more about Azure-Arc enabled Kubernetes [here](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/overview).

## Scenario

In most enterprise environments, you will have your cluster behind a firewall, protected in a specific virtual network perimeter and only specific incoming and outgoing traffic will be allowed depending on workload requirements. In such scenarios, your Kubernetes API server endpoint, DNS FQDN resolution will have restricted access. Your nodes and API server communication remain on the private network only. Only required application and service traffic are allowed. One of the challenges of such environment is that it does not support cloud hosted CI/CD agents such as Azure DevOps Microsoft-hosted agents or GitHub hosted runners. You will need an agent VM which can access the cluster from within the network perimeter such as [self-hosted Azure DevOps agents](https://docs.microsoft.com/en-us/azure/devops/pipelines/agents/agents?tabs=browser&view=azure-devops) or [self-hosted runners](https://docs.github.com/en/enterprise-server@3.2/actions/hosting-your-own-runners/about-self-hosted-runners). For those who require more control and security, this is the way to go. However, in certain cases with Dev/Test environment or wherein you would like to focus on productivity and less maintenance overhead, you would like to use the cloud-hosted agents for build and deploy.  

## Solution

In this exercise we will learn how to use **cluster connect** and **GitHub action** to build and deploy a sample application to a Arc-enabled Kubernetes cluster. Here is a diagram to illustrate the solution and steps to achieve the same.

*Disclaimer: The steps mentioned below are for non-production, test or experimental learning scenarios only. It will help you to understand and take an informed decision on its usage. Once confident, you can create a more robust production ready solution.*

[IMAGE GOES HERE]
------
## Implementation steps
Below are the high level steps you can follow to implement and test the solution. before you get started, understand what you need as prerequisite.
### Prerequisites
1. An Azure Arc-enabled Kubernetes cluster and understanding how Azure Arc works.
1. A jumpbox or a bastion host system with Azure CLI, cli extensions e.g., (*connectedk8s*)the *kubeconfig* file to be able to access the cluster.
    > Remember: the cluster API server or the nodes can not accessed outside the network boundary.
1. A developer system with Visual Studio code, Azure CLI, necessary extensions (*connectedk8s*). This system will be used for developing our application, github actions and testing the functionalities.
1. One Github account
1. One Azure container registry

If you **do not** have a Azure Arc enabled Kubernetes cluster you can use **any one** of the below option to create one
1. Follow this [jumpstart scenario](https://azurearcjumpstart.io/azure_arc_jumpstart/azure_arc_k8s/) on your favorite cloud or Kubernetes flavor in an automated fasion
1. [This](https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/quickstart-connect-cluster?tabs=azure-cli) Microsoft document to create a kubernetes cluster and onboard it manually
1. Manually create a Private cluster on Azure and onboard to Azure Arc. You can run these from your developer system with Azure CLI installed.
    ```bash
    # Login to your subscription and ensure you are in the correct subscription
    az login
    az account set -s "<subscription_name>"
    
    # create a Service Principal for your subscription to be used in this exercise
    az ad sp create-for-rbac --name "<your_SP_name>" --role contributor --scopes /subscriptions/<subscription_id>/resourceGroups/<your_rg_name> --sdk-auth
    # Note the JSON Output after you run the command with the AppID, Secret, Tenant ID and Subscription ID
    
    # Enable necessary resource providers and extensions
    az provider register --namespace Microsoft.Kubernetes --wait
    az provider register --namespace Microsoft.KubernetesConfiguration --wait

    az extension add --upgrade --yes -n connectedk8s
    az extension add --upgrade --yes --name k8s-extension

    # Create a Private AKS cluster and test the functionality. Replace the location and resource group, cluster name with as per your choice.
    location='your_location'
    rgname='your_rg_name'
    privclustername='your_private_cluster_name'
    
    az group create -l $location -n $rgname

    az aks create -n $privclustername -g $rgname --load-balancer-sku standard --enable-private-cluster --enable-managed-identity --private-dns-zone system --disable-public-fqdn
    
    # It will take approx. 15 min to create the cluster. Test the cluster resources once complete. 'command invoke' command helps authenticate using Azure APIs and run the command against your Private cluster.
    az aks command invoke --resource-group $rgname --name $privclustername --command "kubectl get pods -n kube-system"

    # Once complete, create a jumpbox with Windows or Linux operating system in the same subnet where your AKS nodes exist. Ensure you have Azure CLI and necessary extensions installed as mentioned above.

    # Run all of the below commands from inside the bastion host or jumpbox once you login to the subscription using Azure CLI.
    az aks get-credentials --resource-group $rgname --name $privclustername
    # List all deployments in all namespaces
    kubectl get deployments --all-namespaces=true
    
    # Create necessary variables, resource groups, Azure container registry before onboarding your cluster to Azure Arc. Feel free to change the values below to more appropriate ones as you may think fit.
    K8S_ARC_PREFIX="k8sArcPriv"
    ARC_RG_NAME="${K8S_ARC_PREFIX}-RG"
    ARC_CLUSTER_NAME="${K8S_ARC_PREFIX}-cluster"
    LOCATION="eastus"
    ACR_NAME='your_acr_name'
    
    az group create -n $ARC_RG_NAME -l $LOCATION
    az acr create --resource-group $ARC_RG_NAME --name $ACR_NAME --sku Basic
    # Once complete verify the resources created and note the ACR name for future use. It will be in the form of <your_acr_name>.azurecr.io
    
    # Onboard your cluster to Azure Arc and verify you are able to see all pods running and you are able to see the resources on Azure Portal.
    az connectedk8s connect -g $ARC_RG_NAME -n $ARC_CLUSTER_NAME -l $LOCATION
    kubectl get pods -n azure-arc
    ARM_ID_CLUSTER=$(az connectedk8s show -n $ARC_CLUSTER_NAME -g $ARC_RG_NAME --query id -o tsv)
 
    ```
### Enable 'cluster connect' and assign rights to the service principal for Kubernetes RBAC
Enable Cluster Connect on the Arc-enabled cluster. Run the below command from the within your bastion host system from within the same session.
    
    ```bash
    az connectedk8s enable-features --features cluster-connect -n $ARC_CLUSTER_NAME -g $ARC_RG_NAME
    $AAD_ENTITY_OBJECT_ID= $(az ad sp show --id <appid_of_your_service_principal> --query objectId -o tsv)
    kubectl create clusterrolebinding admin-user-binding --clusterrole cluster-admin --user=$AAD_ENTITY_OBJECT_ID
    ```
Now your cluster is ready to accept external query in a secure fashion using the Azure-Arc, Cluster connect and RBAC implementation of the service principal
### Create and configure your 'GitHub action'


### Run Github action and test the solution

### Cleanup the environment


### Summary








## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.microsoft.com.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
