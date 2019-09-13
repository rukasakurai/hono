+++
title = "Microsoft Azure"
weight = 475
aliases = [
    "/deployment/azure"
]
+++

This guide describes how Eclipse Hono™ can be deployed on Microsoft Azure. It includes:

- Leveraging [Azure Resource Manager (ARM)](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-overview) templates for an automated infrastructure deployment.
- Deploy Eclipse Hono™ into [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/intro-kubernetes).
- Leverage [Azure Service Bus](https://docs.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview) for the Eclipse Hono™ AMQP network.
- Push Eclipse Hono™ docker images to an [Azure Container Registry (ACR)](https://azure.microsoft.com/en-us/services/container-registry/).

<!--more-->

{{% warning title="Use for demos only" %}}
This deployment model is not meant for productive use but rather for evaluation as well as demonstration purposes or as a baseline to evolve a production grade [Application architecture](https://docs.microsoft.com/en-us/azure/architecture/guide/) out of it which includes Eclipse Hono™.
{{% /warning %}}

## Prerequisites

- An [Azure subscription](https://azure.microsoft.com/en-us/get-started/).
- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) installed to setup the infrastructure.
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [helm](https://helm.sh/docs/using_helm/#installing-helm) installed to deploy Eclipse Hono™ into [Azure Kubernetes Service (AKS)](https://docs.microsoft.com/en-us/azure/aks/intro-kubernetes).
- [Docker](https://www.docker.com) installed
- Build requirements are met s described here

## HowTo install Eclipse Hono™

### Download Eclipse Hono™

```bash
git clone https://github.com/eclipse/hono.git
```

### Build and push the docker images

In this example we expect that you have a [Azure Container Registry (ACR)](https://azure.microsoft.com/en-us/services/container-registry/) available. If not you can follow this [guidance](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli).

```bash
# Resource group where the ACR is deployed.
acr_resourcegroupname={YOUR_ACR_RG}
# Name of your ACR.
acr_registry_name={YOUR_ACR_NAME}
# Full name of the ACR.
acr_login_server=$acr_registry_name.azurecr.io
# Authenticate your docker daemon with the ACR.
az acr login --name $ACR_NAME
# Build images.
cd hono
mvn install -Pbuild-docker-image -Ddocker.registry=$acr_login_server
# Push images to ACR.

```

### Deploy to Microsoft Azure

First we are going to setup the basic Kubernetes infrastructure.

As described [here](https://docs.microsoft.com/en-gb/azure/aks/kubernetes-service-principal) we will create an explicit service principal. Later we add roles to this principal, e.g. to access the [Azure Container Registry (ACR)](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-intro).

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
cd deploy/src/main/deploy/azure/
az group deployment create --name HonoBasicInfrastructure --resource-group $resourcegroup_name --template-file arm/honoInfrastructureDeployment.json --parameters uniqueSolutionPrefix=$unique_solution_prefix servicePrincipalObjectId=$object_id_principal servicePrincipalClientId=$app_id_principal servicePrincipalClientSecret=$password_principal
```

Now we can retrieve settings from the deployment for the following steps:

```bash
aks_cluster_name=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.aksClusterName.value -o tsv`
http_ip_address=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.httpPublicIPAddress.value -o tsv`
amqp_ip_address=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.amqpPublicIPAddress.value -o tsv`
mqtt_ip_address=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.mqttPublicIPAddress.value -o tsv`
registry_ip_address=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.registryPublicIPAddress.value -o tsv`
network_ip_address=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.networkPublicIPAddress.value -o tsv`
service_bus_namespace=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.serviceBusNamespaceName.value -o tsv`
service_bus_key_name=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.serviceBusKeyName.value -o tsv`
service_bus_key=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.serviceBusKey.value -o tsv`
```

Now you can set your cluster in `kubectl`.

```bash
az aks get-credentials --resource-group $resourcegroup_name --name $aks_cluster_name
```

Next deploy helm on to the AKS cluster as well create retain storage for the device registry. It will take a moment until tiller is booted up.

```bash
kubectl apply -f helm-rbac.yaml
helm init --service-account tiller
kubectl apply -f managed-premium-retain.yaml
```

Next we prepare the k8s environment:

```bash
k8s_namespace=honons
kubectl create namespace $k8s_namespace
```

Finally install Eclipse Hono™. Leveraging the _managed-premium-retain_ storage in combination with _deviceRegistry.resetFiles=false_ parameter is optional but ensures that Device registry storage will retain future update deployments.

```bash
# Go back to hono/deploy directory
cd ../../../../
# Install Eclipse Hono™ with helm
helm install target/deploy/helm/eclipse-hono/ \
    --dep-up \
    --name hono \
    --namespace $k8s_namespace \
    --set adapters.mqtt.svc.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$resourcegroup_name \
    --set adapters.http.svc.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$resourcegroup_name \
    --set adapters.amqp.svc.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$resourcegroup_name \
    --set deviceRegistry.svc.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$resourcegroup_name \
    --set dispatchRouter.svc.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-resource-group"=$resourcegroup_name \
    --set deviceRegistry.storageClass=managed-premium-retain \
    --set deviceRegistry.resetFiles=false \
    --set azure.serviceBus.enabled=true \
    --set azure.serviceBus.saslUsername=$service_bus_key_name \
    --set azure.serviceBus.saslPassword=$service_bus_key \
    --set azure.serviceBus.host=$service_bus_namespace.servicebus.windows.net \
    --set adapters.mqtt.svc.loadBalancerIP=$mqtt_ip_address \
    --set adapters.http.svc.loadBalancerIP=$http_ip_address \
    --set adapters.amqp.svc.loadBalancerIP=$amqp_ip_address \
    --set deviceRegistry.svc.loadBalancerIP=$registry_ip_address \
    --set dispatchRouter.svc.loadBalancerIP=$network_ip_address
```

Have fun with the Eclipse Hono™ on Microsoft Azure!

## Next steps

You can follow the steps as described in the [getting started](https://www.eclipse.org/hono/getting-started/) tutorial with the following differences:

Compared to a plain k8s deployment Azure provides us DNS names with static IPs for the Eclipse Hono™ endpoints. To retrieve them:

```bash
HTTP_ADAPTER_IP=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.httpPublicIPFQDN.value -o tsv`
AMQP_ADAPTER_IP=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.amqpPublicIPFQDN.value -o tsv`
MQTT_ADAPTER_IP=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.mqttPublicIPFQDN.value -o tsv`
REGISTRY_IP=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.registryPublicIPFQDN.value -o tsv`
AMQP_NETWORK_IP=`az group deployment show --name HonoBasicInfrastructure --resource-group $resourcegroup_name --query properties.outputs.networkPublicIPFQDN.value -o tsv`
```

As Azure Service Bus does not support auto creation of queues you have to create a queue per tenant (ID), e.g. after you have created your tenant run:

```bash
az servicebus queue create --resource-group $resourcegroup_name \
    --namespace-name $service_bus_namespace \
    --name $MY_TENANT
```

## Additional topics

### Monitoring

You can monitor your cluster using Azure Monitor Insights.

Navigate to [https://portal.azure.com](https://portal.azure.com) -> your resource group -> your kubernetes cluster

On an overview tab you fill find an information about your cluster (status, location, version, etc.). Also, here you will find a "Monitor Containers" link. Navigate to "Monitor Containers" and explore metrics and statuses of your Cluster, Nodes, Controllers and Containers.

### Cleaning up

Use the following command to delete all created resources once they are no longer needed:

```sh
az group delete --name $resourcegroup_name --yes --no-wait
```
