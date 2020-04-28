# Microsoft Azure & Snyk Workshop

Welcome! This workshop will provide you with sample patterns and reference 
architectures for securing Microsoft Azure workloads with Snyk. We will provide
you with step-by-step examples and sample code that will walk you through 
deployment of supporting infrastructure, sample applications, and configuration 
of the various Snyk integrations to Microsoft Azure.

## Prerequisites

In order to complete the exercises in this workshop, you will need both a 
[Microsoft Azure](https://azure.microsoft.com/) & [Snyk](https://snyk.io/) account.
- [Create](https://azure.microsoft.com/en-us/free) a free Microsoft Azure account.
- [Create](https://snyk.io/login) a free Snyk account.

## Getting Started

### Configure the local environment

Most of the work we will do will involve using the [Azure Command-Line Interface 
(CLI)](https://docs.microsoft.com/en-us/cli/azure/?view=azure-cli-latest). Detailed
documentation on installing the Azure CLI for [Windows](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-windows?view=azure-cli-latest), 
[macOS](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-macos?view=azure-cli-latest), and
[Linux](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-yum?view=azure-cli-latest) is available in 
[Azure documentation](https://docs.microsoft.com/en-us/azure/). These examples will be based on macOS.

#### Install Homebrew

If you don't already have it, [install Homebrew](https://docs.brew.sh/Installation.html) then install 
the Azure CLI with the following command:

```bash
brew update && brew install azure-cli
```

#### Authenticate with the Azure CLI

Once installed, you will need to sign in to your Azure account from the CLI. 
Run the following command:

```bash
az login
```

The CLI will attempt to open your default browser and load the Azure login page. Provide your Azure account
credentials in the browser and upon successfull authentication you will see the 
following response in your browser window:

![azure_cli_login](images/azure_cli_login.png)

If you encounter a problem, please review the [Install Azure CLI on macOS](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-macos?view=azure-cli-latest)
documentation pages for additional guidance.

## Provision Azure services

In order to understand the various Snyk integration points to Azure, we are going to
deploy and configure some supporting resources. The objective for these exercises is to demonstrate
how Snyk secures your workloads. We will provide basic patterns intended for use
in learning environments. For a deeper dive and learning more about Azure, we suggest
referencing Microsoft's self-paced [training modules](https://docs.microsoft.com/en-us/learn/browse/?products=azure).

### Deploy Azure Kubernetes Service (AKS)

The following examples are based on an [Azure Quickstart](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough) 
for deploying AKS using the CLI. We will deploy a cluster as well as a sample
multi-container application. The application will include both a web front end
as well as a Redis instance.

#### Create a resource group

We begin by creating an [Azure resource group](https://docs.microsoft.com/en-us/learn/modules/control-and-organize-with-azure-resource-manager/2-principles-of-resource-groups)
to logical organize the resources we will deploy and manage. Here, we will also define
the location where our resources will run in Azure. In this case, we will deploy to the
`eastus` location. From your terminal, run the following command:

```bash
az group create --name mySnykAKSResourceGroup --location eastus
```

When successfully completed, you will see output similar to the following:

```json
{
  "id": "/subscriptions/<guid>/resourceGroups/mySnykAKSResourceGroup",
  "location": "eastus",
  "managedBy": null,
  "name": "mySnykAKSResourceGroup",
  "properties": {
    "provisioningState": "Succeeded"
  },
  "tags": null,
  "type": "Microsoft.Resources/resourceGroups"
}
```

You can also validate the creation of the resource group in the Azure portal as illustrate below:

![azure_resource_group_01](images/azure_resource_groups_01.png)

#### Create the AKS cluster

Next, we are going to create a cluster named `mySnykAKSCluster` in our recently created 
`mySnykAKSResourceGroup`. Our cluster will have one node and will have monitoring enabled.

```bash
az aks create --resource-group mySnykAKSResourceGroup --name mySnykAKSCluster --node-count 1 --enable-addons monitoring --generate-ssh-keys
```

This may take several minutes to complete. You will see the following outputs in the terminal:

```text
Finished service principal creation[##########################]  100.0000%
- Running...
AAD role propagation done[[##########################]  100.0000%
```

Once the deployment completes the CLI will return a lengthy JSON response containing
details about your cluster. You can also view this within the Azure portal:

![azure_resource_group_02](images/azure_resource_groups_02.png)

![azure_resource_group_03](images/azure_resource_groups_03.png)

![azure_resource_group_04](images/azure_resource_groups_04.png)

#### Connect to the cluster

To manage our AKS cluster, we will use [kubectl](https://kubernetes.io/docs/user-guide/kubectl/).
Since we are using the Azure CLI, we will need to install `kubectl` with the following
command:

```bash
az aks install-cli
```

Next, we will need to configure `kubectl` to connect to AKS by downloading
our credentials and configuring the CLI to use these. 

```bash
az aks get-credentials --resource-group mySnykAKSResourceGroup --name mySnykAKSCluster
```

If successful, you should see output similar to this:

```text
Merged "mySnykAKSCluster" as current context in $HOME/.kube/config
```

Now, we are ready to verify our connection to our cluster.

```bash
kubectl get nodes
```

When the node is ready, you should see an example output similar to the following:

```text
NAME                                STATUS   ROLES   AGE   VERSION
aks-nodepool1-27048785-vmss000000   Ready    agent   5m    v1.15.10
```

#### Deploy a sample application

We will deploy our sample application using a [Kubernetes manifest](https://docs.microsoft.com/en-us/azure/aks/concepts-clusters-workloads#deployments-and-yaml-manifests)
file. A sample file named `azure-vote.yaml` is provided for your convenience. The manifest 
includes two Kubernetes deployments: one for a sample Azure Vote Python application and 
the other for a [Redis](https://redislabs.com/) instance. Two Kubernetes [services](https://docs.microsoft.com/en-us/azure/aks/concepts-network#services) are also created:
one is an internal service for the Redis instance and the other is an external service to allow access to the application from the 
internet.

If you have cloned the repository and are working from the local directory, run the following command 
to deploy the application from the `YAML` manifest:

```bash
kubectl apply -f templates/azure-vote.yaml
```

If successful, you should see output similar to the following:

```text
deployment "azure-vote-back" created
service "azure-vote-back" created
deployment "azure-vote-front" created
service "azure-vote-front" created
```

#### Test the application

We will invoke the [kubectl get service](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get) from our CLI with the
`--watch` argument to monitor the application deployment and obtain the `EXTERNAL-IP` of 
the `LoadBalancer`. 

Run the following command:

```bash
kubectl get service azure-vote-front --watch
```

While the application is deploying you may see output similar to the following:

```text
NAME               TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
azure-vote-front   LoadBalancer   10.0.37.27   <pending>     80:30572/TCP   6s
```

Note that `EXTERNAL-IP` displays a status of `pending`. Wait until this displays a valid
public IP address then copy and paste this value into your web browser.

Your browser should resolve and display the following:

![azure_voting_app](images/azure_voting_app.png)

## Setup the Snyk account

Visit [https://snyk.io](https://snyk.io) and log in.

![snyk_login_01](images/snyk_login_01.png)

If you do not have an account, you can [sign up](https://app.snyk.io/signup) for a 
free account. Snyk offers a [free plan](https://snyk.io/plans/) which includes:

- Unlimited tests on open-source projects
- 200 tests for open source vulnerabilities on private projects and 100 tests for container vulnerabilities
- Fixes for open source and container vulnerabilities
- CLI scans
- Cloud source code integration to GitHub, Azure Repos, and others
- CI/CD pipeline integration
- Continuous monitoring
- Public container registry integration to ACR and others
- Helm plugin to test images from Helm charts

### Configure the Kubernetes integration

From the Snyk web console, navigate to `Integrations`. Search and select
`Kubernetes`. Click `Connect` and copy the `Integration ID` to your clipboard.
The `Integration ID` will be a UUID with a format similar to `abcd1234-abcd-1234-abcd-1234abcd1234`.

![snyk_integrations_01](images/snyk_integrations_01.png)

Let's create an environment variable for our `Integration ID`:

```bash
IntegrationId=<value>
```

#### Install the Snyk controller

From the terminal, ensure that you have helm installed by running the following command:

```bash
brew update && brew install helm
```

Then, add the Snyk Charts repository to Helm with the following command:

```bash
helm repo add snyk-charts https://snyk.github.io/kubernetes-monitor/
```

If successful, you will see output similar to the following:

```text
"snyk-charts" has been added to your repositories
```

Once added, we will need to create a unique namespace for the Snyk controller. Run the following
command:

```bash
kubectl create namespace snyk-monitor
```

If successful, you will see output similar to the following:

```text
namespace/snyk-monitor created
```

The Snyk monitor runs by using your Snyk `Integration ID`, and using a `dockercfg` file. If you are not using any 
private registries, create a Kubernetes secret called `snyk-monitor` containing the Snyk `Integration ID` from the 
previous step and run the following command:


```bash
kubectl create secret generic snyk-monitor -n snyk-monitor \
        --from-literal=dockercfg.json={} \
        --from-literal=integrationId=$IntegrationId 
```

If successful, you will see output similar to the following:

```text
secret/snyk-monitor created
```

Now, install the Snyk Helm chart to your AKS cluster:

```bash
helm upgrade --install snyk-monitor snyk-charts/snyk-monitor \
             --namespace snyk-monitor \
             --set clusterName="mySnykAKSCluster" 
```

If successful, you will see output similar to the following:

```text
Release "snyk-monitor" does not exist. Installing it now.
NAME: snyk-monitor
LAST DEPLOYED: Tue Apr 28 16:34:04 2020
NAMESPACE: snyk-monitor
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

We can also validate our pod is running with the following command:

```bash
kubectl get pods --namespace snyk-monitor
```

You will want to see the `STATUS` display `Running` as in the following example output:

```text
NAME                            READY   STATUS    RESTARTS   AGE
snyk-monitor-544ff7ccd9-qkwj8   1/1     Running   0          4m47s
```

#### Adding Kubernetes workloads