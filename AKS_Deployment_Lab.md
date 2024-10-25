# AKS Deployment Lab
## Prerequisite Steps
### AKS Cluster Creation Lab
All steps in the [AKS Cluster Creation Lab](./AKS_Cluster_Creation_Lab.md) must be completed prior to this lab.

### Define variables for Azure CLI
If the shell was closed since the prior lab, repopulate required variables:
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
AKS_NAME="aks-${LOCATION}-aksdemo"
```

### Ensure cluster is running
Clusters in MCAPS subscriptions may be stopped due to cost savings. Ensure the cluster is running either via the Azure Portal, or with Azure CLI:

`az aks show --name $AKS_NAME --resource-group $RG_NAME --query "[name, powerState.code]" -o tsv`

You can start the Kubernetes cluster with the following command:

`az aks start --name $AKS_NAME --resource-group $RG_NAME`

### Retrieve Kubernetes config file
In order for the kubectl CLI tool to interact with the Kubernetes cluster, you must download a configuration file with cluster details and an authentication token. Run the following command to retrieve the configuration file:

`az aks get-credentials --name $AKS_NAME --resource-group $RG_NAME --admin --overwrite-existing`


Run the following commands to confirm access to the Kubernetes cluster

`kubectl get pods --namespace kube-system`

### Download deployment YAML files
YAML files will be used to deploy resources to the cluster for this lab. The files can be downloaded either via Git with `git clone https://github.com/microsoft/RCG_AKS_Enablement` or manually from:
- [prod-aks-store-quickstart-1.3.0.yaml](./prod-aks-store-quickstart-1.3.0.yaml)
- [prod-aks-store-quickstart-1.5.2.yaml](./prod-aks-store-quickstart-1.5.2.yaml)
- [qa-aks-store-quickstart-1.5.2.yaml](./qa-aks-store-quickstart-1.5.2.yaml)

## Deployment Steps
### Deploy version 1.3.0 to production
The first deployment will create a namespace named `prod` and will use version 1.3.0 of the container images:

`kubectl apply -f prod-aks-store-quickstart-1.3.0.yaml`

Check for the new namespace creation and change context to the namespace:
```shell
kubectl get namespaces
kubectl config set-context --current --namespace='prod'
```

Now that kubectl is configured to reference the `prod` namespace, you can check the status of the deployed resources:
```shell
kubectl get pods
kubectl get services
```

> Tip: You can add the `--watch` option to the above commands to continue watching for changes. This is helpful if you are waiting for a load balancer IP to be created or for pods to start running.

Copy the `EXTERNAL-IP` from the `store-front` load balancer service and access it in your browser to view the running deployment. Browse to the `/health` path to see version info.

### Deploy version 1.5.2 side by side to QA
To test the development of version 1.5.2, you can create an additional deployment to a new namespace with both versions running side by side:

`kubectl apply -f qa-aks-store-quickstart-1.5.2.yaml`

Check for the new namespace creation and change context to the namespace:
```shell
kubectl get namespaces
kubectl config set-context --current --namespace='qa'
```

Now that kubectl is configured to reference the `qa` namespace, you can check the status of the deployed resources:
```shell
kubectl get pods
kubectl get services
```

Copy the `EXTERNAL-IP` from the `store-front` load balancer service and access it in your browser to view the running deployment. Browse to the `/health` path to see version info.

### Deploy version 1.5.2 to production
Once it has been confirmed that version 1.5.2 is functional, overwrite the production deployment with the new version:

```shell
kubectl config set-context --current --namespace='prod'
kubectl apply -f prod-aks-store-quickstart-1.5.2.yaml
```

Kubernetes will perform a staged rollout of the new pods, to allow for a more seamless transition. Rollouts can be checked on a per deployment basis. For example:

`kubectl rollout status deployment/store-front`

Once a rollout is complete, you can check the history on a per deployment basis. For example:

`kubectl rollout history deployment/store-front`

This will show a simplified view. If you would like to see full details, run the following:

`kubectl rollout history deployment/store-front -o yaml`

Copy the `EXTERNAL-IP` from the `store-front` load balancer service with the `kubectl get services` command, and browse to the `/health` path to see updated version info.

### Rollback version 1.3.0 to production
If an issue is found with the deployment, you can rollback to a prior version. If you have the original YAML file, you can simply re-apply it

`kubectl apply -f prod-aks-store-quickstart-1.3.0.yaml`

You can additionally use the deployment rollback feature:

```shell
kubectl rollout undo deployment.apps/order-service
kubectl rollout undo deployment.apps/product-service
kubectl rollout undo deployment.apps/store-front
```

Copy the `EXTERNAL-IP` from the `store-front` load balancer service with the `kubectl get services` command, and browse to the `/health` path to see reverted version info.