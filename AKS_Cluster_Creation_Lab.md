# AKS Cluster Creation Lab
## Prerequisite Steps
### Azure CLI
Azure CLI `2.64.0` or later is required. Check if Azure CLI is installed and up-to-date with `az version`


- Installation instructions: https://learn.microsoft.com/en-us/cli/azure/install-azure-cli
    - **Note:** If the `az` command does not work after installation, open a new terminal window.
- Update instructions: https://learn.microsoft.com/en-us/cli/azure/update-azure-cli

### Azure Managed Grafana CLI extension
Azure Managed Grafana requires the `amg` CLI extension version `2.3.1` or later. Check if the extension  is installed and up-to-date by running:
    
```shell
    az extension list --query "[?name=='amg']" --output table
```

- The extension can be installed by running `az extension add --name amg`
- Existing installations can be updated by running `az extension update --name amg`

Further details about the extension are available on:
- https://learn.microsoft.com/en-us/cli/azure/azure-cli-extensions-overview
- https://github.com/Azure/azure-cli-extensions/blob/main/src/amg/README.md

### kubectl and kubelogin
[`kubectl`](https://kubernetes.io/docs/reference/kubectl/) is the CLI tool used to interact with Kubernetes clusters and [`kubelogin`](https://azure.github.io/kubelogin/index.html) enables Entra ID authentication. These can be installed automatically by running `az aks install-cli` or manually following the instructions on:
- https://kubernetes.io/docs/tasks/tools/#kubectl
- https://azure.github.io/kubelogin/install.html

**Note:** If the `kubectl` command does not work after installation, open a new terminal window.

## Deployment Steps
### Set base variables
The following variables will be used to define the region and configuration of the required resources. Adjust as necessary.

#### PowerShell
```powershell
$LOCATION="eastus2"
$RG_NAME="rg-$($LOCATION)-aksdemo"
$ACR_SUFFIX="$(Get-Random -Maximum 9999)"
$ACR_NAME="acr$($LOCATION)$($ACR_SUFFIX)"
$LOG_ANALYTICS_WORKSPACE_NAME="log-$($LOCATION)-aksdemo"
$AZURE_MONITOR_WORKSPACE_NAME="amw-$($LOCATION)-aksdemo"
$MANAGED_GRAFANA_NAME="amg-$($LOCATION)-aksdemo"
$NSG_NAME="nsg-$($LOCATION)-aksdemo"
$VNET_NAME="vnet-$($LOCATION)-aksdemo"
$AKS_CONTROL_PLANE_MI_NAME="mi-$($LOCATION)-aksdemo-controlplane"
$AKS_NAME="aks-$($LOCATION)-aksdemo"
$AKS_VM_SIZE="Standard_D4ds_v5"
```

#### Bash
```shell
LOCATION="eastus2"
RG_NAME="rg-${LOCATION}-aksdemo"
ACR_SUFFIX="${RANDOM}"
ACR_NAME="acr${LOCATION}${ACR_SUFFIX}"
LOG_ANALYTICS_WORKSPACE_NAME="log-${LOCATION}-aksdemo"
AZURE_MONITOR_WORKSPACE_NAME="amw-${LOCATION}-aksdemo"
MANAGED_GRAFANA_NAME="amg-${LOCATION}-aksdemo"
NSG_NAME="nsg-${LOCATION}-aksdemo"
VNET_NAME="vnet-${LOCATION}-aksdemo"
AKS_CONTROL_PLANE_MI_NAME="mi-${LOCATION}-aksdemo-controlplane"
AKS_NAME="aks-${LOCATION}-aksdemo"
AKS_VM_SIZE="Standard_D4ds_v5"
```

### Create Resource Group
Create the Resource Group which will hold all of the demo environment resources

`az group create --name $RG_NAME --location $LOCATION`

### Create Azure Container Registry
Create an Azure Container Registry for hosting custom container images

`az acr create --resource-group $RG_NAME --name $ACR_NAME --sku Basic --location $LOCATION`

### Create Log Analytics Workspace
Create a Log Analytics Workspace to enable Container Insights

`az monitor log-analytics workspace create --workspace-name $LOG_ANALYTICS_WORKSPACE_NAME --resource-group $RG_NAME --sku PerGB2018`

### Create Azure Monitor Workspace
Create an Azure Monitor Workspace to enable Managed Prometheus

`az monitor account create --name $AZURE_MONITOR_WORKSPACE_NAME --resource-group $RG_NAME --location $LOCATION`

### Create Azure Managed Grafana
Create an Azure Managed Grafana instance to visualize data stored in Managed Prometheus

`az grafana create --name $MANAGED_GRAFANA_NAME --resource-group $RG_NAME --location $LOCATION`

### Create Azure Kubernetes Service cluster identity
Create a managed identity for the Azure Kubernetes Service cluster

`az identity create --name $AKS_CONTROL_PLANE_MI_NAME --resource-group $RG_NAME --location $LOCATION`

### Create Networking Resources
Create the NSG, VNet and Subnet which will host the Kubernetes compute resources
```shell
az network nsg create --name $NSG_NAME --resource-group $RG_NAME --location $LOCATION
az network vnet create --name $VNET_NAME --resource-group $RG_NAME  --location $LOCATION --address-prefixes 10.0.0.0/16
az network vnet subnet create --vnet-name $VNET_NAME --resource-group $RG_NAME --name "AKS-Subnet" --address-prefixes 10.0.0.0/24 --network-security-group $NSG_NAME
```

### Assign role binding for Azure Kubernetes Service cluster identity to Subnet
In order for AKS to join the Kubernetes compute resources to the VNet, the cluster's managed identity requires the `Network Contributor` on the subnet

**Note:** If the role assignment command fails to find the managed identity, retry after a minute or two to give the resource time to propagate.

#### PowerShell
```powershell
$SUBNET_ID=$(az network vnet subnet show --vnet-name $VNET_NAME --resource-group $RG_NAME --name "AKS-Subnet" --query id --output tsv)
$AKS_CONTROL_PLANE_MI_CLIENT_ID=$(az identity show --name $AKS_CONTROL_PLANE_MI_NAME --resource-group $RG_NAME --query principalId --output tsv)
az role assignment create --assignee $AKS_CONTROL_PLANE_MI_CLIENT_ID --scope $SUBNET_ID --role "Network Contributor"
```

#### Bash
```shell
SUBNET_ID=$(az network vnet subnet show --vnet-name $VNET_NAME --resource-group $RG_NAME --name "AKS-Subnet" --query id --output tsv)
AKS_CONTROL_PLANE_MI_CLIENT_ID=$(az identity show --name $AKS_CONTROL_PLANE_MI_NAME --resource-group $RG_NAME --query principalId --output tsv)
az role assignment create --assignee $AKS_CONTROL_PLANE_MI_CLIENT_ID --scope $SUBNET_ID --role "Network Contributor"
```

### Create Azure Kubernetes Service cluster
The following commands will create the AKS cluster and link it to each of the resources created above.

**Note:** You may receive a warning about the `docker_bridge_cidr` but this can be safely ignored.

#### PowerShell
```powershell
$ACR_ID=$(az acr show --resource-group $RG_NAME --name $ACR_NAME --query id --output tsv)
$LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace show --workspace-name $LOG_ANALYTICS_WORKSPACE_NAME --resource-group $RG_NAME --query id --output tsv)
$AZURE_MONITOR_WORKSPACE_ID=$(az monitor account show --name $AZURE_MONITOR_WORKSPACE_NAME --resource-group $RG_NAME --query id --output tsv)
$MANAGED_GRAFANA_ID=$(az grafana show --name $MANAGED_GRAFANA_NAME --resource-group $RG_NAME --query id --output tsv)
$AKS_CONTROL_PLANE_MI_RESOURCE_ID=$(az identity show --name $AKS_CONTROL_PLANE_MI_NAME --resource-group $RG_NAME --query id --output tsv)

az aks create --name $AKS_NAME `
    --resource-group $RG_NAME `
    --location $LOCATION `
    --assign-identity $AKS_CONTROL_PLANE_MI_RESOURCE_ID `
    --node-count 2 `
    --node-vm-size $AKS_VM_SIZE `
    --network-plugin azure `
    --network-plugin-mode overlay `
    --vnet-subnet-id $SUBNET_ID `
    --service-cidr 100.64.0.0/16 `
    --dns-service-ip 100.64.0.10 `
    --pod-cidr 100.65.0.0/16 `
    --enable-addons monitoring `
    --workspace-resource-id $LOG_ANALYTICS_WORKSPACE_ID `
    --enable-azure-monitor-metrics `
    --azure-monitor-workspace-resource-id $AZURE_MONITOR_WORKSPACE_ID `
    --grafana-resource-id $MANAGED_GRAFANA_ID `
    --attach-acr $ACR_ID `
    --no-ssh-key `
    --zones 1 2 3
```

#### Bash
```shell
ACR_ID=$(az acr show --resource-group $RG_NAME --name $ACR_NAME --query id --output tsv)
LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace show --workspace-name $LOG_ANALYTICS_WORKSPACE_NAME --resource-group $RG_NAME --query id --output tsv)
AZURE_MONITOR_WORKSPACE_ID=$(az monitor account show --name $AZURE_MONITOR_WORKSPACE_NAME --resource-group $RG_NAME --query id --output tsv)
MANAGED_GRAFANA_ID=$(az grafana show --name $MANAGED_GRAFANA_NAME --resource-group $RG_NAME --query id --output tsv)
AKS_CONTROL_PLANE_MI_RESOURCE_ID=$(az identity show --name $AKS_CONTROL_PLANE_MI_NAME --resource-group $RG_NAME --query id --output tsv)

az aks create --name $AKS_NAME \
    --resource-group $RG_NAME \
    --location $LOCATION \
    --assign-identity $AKS_CONTROL_PLANE_MI_RESOURCE_ID \
    --node-count 2 \
    --node-vm-size $AKS_VM_SIZE \
    --network-plugin azure \
    --network-plugin-mode overlay \
    --vnet-subnet-id $SUBNET_ID \
    --service-cidr 100.64.0.0/16 \
    --dns-service-ip 100.64.0.10 \
    --pod-cidr 100.65.0.0/16 \
    --enable-addons monitoring \
    --workspace-resource-id $LOG_ANALYTICS_WORKSPACE_ID \
    --enable-azure-monitor-metrics \
    --azure-monitor-workspace-resource-id $AZURE_MONITOR_WORKSPACE_ID \
    --grafana-resource-id $MANAGED_GRAFANA_ID \
    --attach-acr $ACR_ID \
    --no-ssh-key \
    --zones 1 2 3
```


### Retrieve Kubernetes config file
In order for the kubectl CLI tool to interact with the Kubernetes cluster, you must download a configuration file with cluster details and an authentication token. Run the following command to retrieve the configuration file:

`az aks get-credentials --name $AKS_NAME --resource-group $RG_NAME --admin`

**Note:** This command is using the generic cluster administrative account which is not bound to Entra ID. This is against best practices and Entra ID authentication will be covered in a later module

### Test Kubernetes cluster access
Run the following commands to check access to the new Kubernetes cluster

```shell
kubectl get nodes
kubectl get pods --namespace kube-system
```

### Stop / Start Azure Kubernetes Service cluster
You can stop the Kubernetes compute resources to minimize costs with the following command:

`az aks stop --name $AKS_NAME --resource-group $RG_NAME`

You can start the Kubernetes again with the following command:

`az aks start --name $AKS_NAME --resource-group $RG_NAME`

**Note:** If you closed your terminal window, you will need to repopulate your variables in order to make the above commands work:

#### PowerShell
```powershell
$LOCATION="eastus2"
$RG_NAME="rg-$($LOCATION)-aksdemo"
$AKS_NAME="aks-$($LOCATION)-aksdemo"
```

#### Bash
```shell
LOCATION="eastus2"
RG_NAME="rg-${LOCATION}-aksdemo"
AKS_NAME="aks-${LOCATION}-aksd
```
