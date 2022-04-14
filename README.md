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
1. Manually create a Private cluster on Azure and onboard to Azure Arc. **Ensure you follow the steps in the order and also read the inline comments**. You can run these from your developer system with Azure CLI installed.
    ```bash
    # Login to your subscription and ensure you are in the correct subscription
    az login
    az account set -s "<subscription_name>"
    
    # Create a Service Principal for your subscription to be used in this exercise. You can scope this to your resource groups as well. This must also include your ARC (connected cluster and ACR) resource group that you will create later. So, you can skip this step for now and do it later once you create the RG for your ARC cluster resource. You can also leave this at the subscription level for your test and create it here.

    az ad sp create-for-rbac --name "<your_SP_name>" --role contributor --scopes /subscriptions/<subscription_id> --sdk-auth
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
    ```
After a few mins you should be able to see the new resource created on Azure portal.

![Onboarded kubernetes cluster on Azure portal](/media/arcCluster.png)

In the same session run the below commands to check the pods running in the cluster for Arc.

```bash
    kubectl get pods -n azure-arc
    
    ARM_ID_CLUSTER=$(az connectedk8s show -n $ARC_CLUSTER_NAME -g $ARC_RG_NAME --query id -o tsv)
```

![Arc Pods list from bastion host](/media/arcpods.png)

In the same session run the below commands to check the pods running in the cluster for Azure Arc.

### Enable 'cluster connect' and assign rights to the service principal for Kubernetes RBAC
Run the below command in the same session from the within your bastion host.

```bash
az connectedk8s enable-features --features cluster-connect -n $ARC_CLUSTER_NAME -g $ARC_RG_NAME

$AAD_ENTITY_OBJECT_ID= $(az ad sp show --id <appid_of_your_service_principal> --query objectId -o tsv)

kubectl create clusterrolebinding admin-user-binding --clusterrole cluster-admin --user=$AAD_ENTITY_OBJECT_ID
```

Now your cluster is ready to accept external query in a secure fashion using the Azure-Arc, Cluster connect and RBAC implementation of the service principal. You can test by logging in using the service principal credentials in your terminal and using **"connectedk8s proxy"** command.

**These commands should be run from the development system (outside the cluster network)**.

```bash
$AzCred = Get-Credential -UserName <appid_of_your_service_principal>

az login --service-principal -u $AzCred.UserName -p $AzCred.GetNetworkCredential().Password --tenant <YOUR_TENANT_ID>

az connectedk8s proxy -n $ARC_CLUSTER_NAME -g $ARC_RG_NAME

```

The **connectedk8s proxy** implementation creates the necessary *kubeconfig* file on the system outside the network as well. You will find the new kubeconfig file in the 'C:\users\User_Name\ .kube\' folder. Note the server endpoint pointing to the loopback IP of 127.0.0.1 proxy endpoint.

![Arc kubeconfig file ](/media/arckubeconfig.png)

Run the below command to test the output.

```bash
kubectl get pods -n kube-system

```
### Create and configure your 'GitHub action'
You can now use your github repo and github action to deploy an application into the cluster without having to run a self-hosted agent inside the network perimeter. To test this functionality, we have a sample code and a github action here in [this repo](https://github.com/Bapic/gh-arc-app-deploy). This repo contains a sample application called 'azure-vote-app', and its source code. The application contains a front end python app and a redis cache as temporary data store. The **.github/workflows** folder contains the github action that you will use to deploy this application to the cluster. If you are using a cloud cloud managed Kubernetes cluster, you should be able to use the Kubernetes service type as "LoadBalancer" and access the application over the internet. Let's get started.

- Fork the repo to your github account and clone it to your developer system using ```git clone https://github.com/Bapic/gh-arc-app-deploy``` command.
- Edit the **.github/workflow/github-action-arc-app-deploy.yaml** file and make necessary variable changes with your values.
![Github action file variable update](/media/yamlvariableupdate.png)

- Edit the **/manifest/azure-vote-frontend-deployment.yaml** file and update <Your_ACR_NAME> in line no. 17
![Azure-vote-front manifest file ](/media/manifestfile.png)

- Push the changes to your Github repo from your local system. Ensure that changes have taken effect in your repo.
- Open your repo using web browser and go to **Actions** tab. You should see the "Deploy to Connected Cluster" workflow.
![Github Action workflow update ](/media/actionworkflow.png)

### Run Github action and test the solution
Last step before you run the workflow is to set up a GitHub Secret. Using *Settings* tab in your Github portal, create a new Secret with name - **AZURE_CREDENTIALS**. The values should include the details of your Service Principal that you had notes earlier. it will look something like this.

```bash
{
  "clientId": "<YOUR_SP_APP_ID>",
  "clientSecret": "<YOUR_SP_PASSWORD>",
  "subscriptionId": "<YOUR_SUBSCRIPTION_ID>",
  "tenantId": "<YOUR_TENANT_ID>",
  "activeDirectoryEndpointUrl": "https://login.microsoftonline.com",
  "resourceManagerEndpointUrl": "https://management.azure.com/",
  "activeDirectoryGraphResourceId": "https://graph.windows.net/",
  "sqlManagementEndpointUrl": "https://management.core.windows.net:8443/",
  "galleryEndpointUrl": "https://gallery.azure.com/",
  "managementEndpointUrl": "https://management.core.windows.net/"
}
```
![Create github secret ](/media/githubsecret.png)

Now your github action is all set to run against the connected Kubernetes cluster.
- Using the web browser in your github repo - click on the **Run workflow** to trigger the workflow.
![Arc kubeconfig file ](/media/runworkflow.png)

Once it starts, you should be able to see all the stages of Build of the source code, push to your ACR of the container image and deployment to your connected cluster. In a few mins your will have your cluster ready with the application running. It will deploy the application in the 'default' namespace.
![Build and deploy using github Action ](/media/builddeploy.png)


- Check the resources created in the cluster. Run this from within your bastion host session

```bash
kubectl get po
```
![Deployed pods ](/media/deployedpods.png)
If you are using a cloud load balancer, you should be able to see the services of the application are accessible over the internet using the Public IP of the load balancer. Collect the service IP (internal or public) using kubectl command from the basion host.

```bash
kubectl get svc
```
Here is an example of **EXTERNAL IP** exposed over the internet
![service IP ](/media/svcip.png)

- Access your application by using the IP address in your browser.
![Access application ](/media/accessapp.png)
### Cleanup the environment
If you want to clean up the entire environment, just delete the resource groups created so far.

### Summary
You have now successfully deployed an application using Azure Arc enabled Kubernetes cluster and GitHub action. For production environments additional security and configurations etc. are recommended. There is one more alternative that you can use in such closed environments - GitOps. All these and much more we will explore in upcoming post. Hope you found this post useful. Stay tuned for more.