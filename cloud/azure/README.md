# Eclipse Hono deployment on Microsoft Azure

This guide describes the most simplistic installation of Twin Reflector hono on Microsoft Azure. It is not meant for productive use but rather for evaluation as well as demonstration purposes or as a baseline to evolve a production grade [Application architecture](https://docs.microsoft.com/en-us/azure/architecture/guide/) out of it which includes the Twin Reflector hono.

## Prerequisites

- An [Azure subscription](https://azure.microsoft.com/en-us/get-started/).
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) installed to setup the infrastructure.
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [helm](https://helm.sh/docs/using_helm/#installing-helm) installed to deploy the Twin Reflector hono into [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/intro-kubernetes).
- Twin Reflector hono is build and image pushed to an [Azure Container Registry (ACR)](https://docs.microsoft.com/en-in/azure/container-registry/container-registry-intro).
- [Docker](https://www.docker.com) installed

TODO hint to build requirements

## HowTo install Eclipse Hono

### Download Eclipse Hono

```bash
git clone https://github.com/eclipse/hono.git
```

### Package and push the docker image

In this example we expect that you have a [Azure Container Registry (ACR)](https://azure.microsoft.com/en-us/services/container-registry/) available However, pushing your image to Docker Hub works as well.

```bash
acr_resourcegroupname=YOUR_ACR_RG
acr_registry_name=YOUR_ACR_NAME
acr_login_server=$acr_registry_name.azurecr.io
cd hono
mvn install -Pbuild-docker-image -Ddocker.registry=$acr_login_server
```

### Deploy to Microsoft Azure

First we are going to setup the basic Kubernetes infrastructure.

As described [here](https://docs.microsoft.com/en-gb/azure/aks/kubernetes-service-principal) we will create an explicit service principal. Late we add roles to this principal, e.g. to access a [Azure Container Registry (ACR)](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-intro).

```bash
service_principal=`az ad sp create-for-rbac --name http://honoServicePrincipal --skip-assignment --output tsv`
app_id_principal=`echo $service_principal|cut -f1 -d ' '`
password_principal=`echo $service_principal|cut -f4 -d ' '`
object_id_principal=`az ad sp show --id $app_id_principal --query objectId --output tsv`
acr_id_access_registry=`az acr show --resource-group $acr_resourcegroupname --name $acr_registry_name --query "id" --output tsv`
```

Note: it might take a few seconds until the principal is available for the next steps.

```bash
az role assignment create --assignee $app_id_principal --scope $acr_id_access_registry --role Reader

resourcegroup_name=hono
az group create --name $resourcegroup_name --location "westeurope"
```

With the next command we will use the provided [Azure Resource Manager (ARM)](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-overview) templates to setup the AKS cluster. This might take a while.

```bash
unique_solution_prefix=myprefix
cd cloud/azure/
az group deployment create --name HonoBasicInfrastructure --resource-group $resourcegroup_name --template-file arm/honoInfrastructureDeployment.json --parameters uniqueSolutionPrefix=$unique_solution_prefix servicePrincipalObjectId=$object_id_principal servicePrincipalClientId=$app_id_principal servicePrincipalClientSecret=$password_principal
```

The output of the command will provide you with the name of your AKS cluster as well as the created Vnet which should be `YOUR_PREFIXhonoaks` and `YOUR_PREFIXhonovnet`.

Note: AKS cluster name, IP address name for the load balancer as well as virtual network name can be provided as parameter to the template as well.

Retrieve configuration settings from the deployment:

```bash
HTTP_ADAPTER_IP=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.httpPublicIPFQDN.value -o tsv`
AMQP_ADAPTER_IP=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.amqpPublicIPFQDN.value -o tsv`
MQTT_ADAPTER_IP=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.mqttPublicIPFQDN.value -o tsv`
REGISTRY_IP=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.registryPublicIPFQDN.value -o tsv`
AMQP_NETWORK_IP=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.networkPublicIPFQDN.value -o tsv`
```

```bash
aks_cluster_name=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.aksClusterName.value -o tsv`
http_ip_address=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.httpPublicIPAddress.value -o tsv`
amqp_ip_address=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.amqpPublicIPAddress.value -o tsv`
mqtt_ip_address=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.mqttPublicIPAddress.value -o tsv`
registry_ip_address=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.registryPublicIPAddress.value -o tsv`
network_ip_address=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.networkPublicIPAddress.value -o tsv`
```

Now you can set your cluster in `kubectl`.

```bash
az aks get-credentials --resource-group $resourcegroup_name --name $aks_cluster_name
```

Next deploy helm on your cluster. It will take a moment until tiller is booted up. So maybe time again to get up and stretch a bit.

```bash
kubectl apply -f helm-rbac.yaml
helm init --service-account tiller
```

Next we prepare the k8s environment and our chart for deployment.

```bash
k8s_namespace=honons
kubectl create namespace $k8s_namespace
```

Now install hono:

```bash
cd ../../deploy/helm
helm install target/deploy/helm/eclipse-hono/ \
    --dep-up \
    --name hono \
    --namespace $k8s_namespace \
    --set adapters.mqtt.svc.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$resourcegroup_name \
    --set adapters.http.svc.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$resourcegroup_name \
    --set adapters.amqp.svc.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$resourcegroup_name \
    --set deviceRegistry.svc.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$resourcegroup_name \
    --set networkServer.svc.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$resourcegroup_name \
    --set adapters.mqtt.svc.loadBalancerIP=$mqtt_ip_address \
    --set adapters.http.svc.loadBalancerIP=$http_ip_address \
    --set adapters.amqp.svc.loadBalancerIP=$amqp_ip_address \
    --set deviceRegistry.svc.loadBalancerIP=$device_ip_address \
    --set authServer.svc.loadBalancerIP=$auth_ip_address
```

Have fun with the Eclipse hono on Microsoft Azure!
