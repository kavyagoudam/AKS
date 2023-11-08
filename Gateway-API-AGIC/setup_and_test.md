## 1. Setup the environment
create the envirornment variables

```bash
$AKS_NAME="aks-app-gw-for-cont"
$RG_NAME="rg-aks"
$LOCATION="eastus"
$IDENTITY_ALB="identity-azure-alb"
$AGFC_NAME="agwc-alb" # Name of the Application Gateway for Containers
$AGFC_SUBNET_PREFIX="10.225.0.0/24"
$AGFC_SUBNET_NAME="subnet-alb" # subnet name can be any non-reserved subnet name (i.e. GatewaySubnet, AzureFirewallSubnet, AzureBastionSubnet would all be invalid)
$AGFC_FRONTEND_NAME="frontend-app"
$AGFC_ASSOCIATION="association-app"
```

Create a resource group
```bash
az group create --name $RG_NAME --location $LOCATION -o table
```
Create AKS cluster with CNI plugin and OIDC & Workload Identity enabled

```bash
az aks create `
    --resource-group $RG_NAME `
    --name $AKS_NAME `
    --location $LOCATION `
    --network-plugin azure `
    --enable-oidc-issuer `
    --enable-workload-identity `
    --output table

```
## Install the ALB controller with its managed identity

create managed identity

```bash

az identity create --resource-group $RG_NAME --name $IDENTITY_ALB

$IDENTITY_ALB_PRINCIPAL_ID=$(az identity show -g $RG_NAME -n $IDENTITY_ALB --query principalId -otsv)
echo $IDENTITY_ALB_PRINCIPAL_ID

```

apply reader role to AKS managed AKS cluster.

```bash

$MC_RG=$(az aks show --name $AKS_NAME --resource-group $RG_NAME --query "nodeResourceGroup" -o tsv)
echo $MC_RG

$MC_RG_ID=$(az group show --name $MC_RG --query id -otsv)
echo $MC_RG_ID

az role assignment create --assignee-object-id $IDENTITY_ALB_PRINCIPAL_ID `
        --assignee-principal-type ServicePrincipal `
        --scope $MC_RG_ID --role "acdd72a7-3385-48ef-bd42-f606fba81ae7"


```

Set up federation with AKS OIDC issuer

```bash

$AKS_OIDC_ISSUER="$(az aks show -n $AKS_NAME -g $RG_NAME --query "oidcIssuerProfile.issuerUrl" -o tsv)"
echo $AKS_OIDC_ISSUER

az identity federated-credential create --name "identity-azure-alb" `
    --identity-name $IDENTITY_ALB `
    --resource-group $RG_NAME `
    --issuer $AKS_OIDC_ISSUER `
    --subject "system:serviceaccount:azure-alb-system:alb-controller-sa"

```

Verify the applied configuration

```bash
az aks get-credentials --resource-group $RG_NAME --name $AKS_NAME --overwrite-existing

```

Install ALB controller via helm charts.

```bash

$IDENTITY_ALB_CLIENT_ID=$(az identity show -g $RG_NAME -n $IDENTITY_ALB --query clientId -o tsv)
echo $IDENTITY_ALB_CLIENT_ID

helm install alb-controller oci://mcr.microsoft.com/application-lb/charts/alb-controller `
     --version 0.4.023971 `
     --set albController.podIdentity.clientID=$IDENTITY_ALB_CLIENT_ID

```
verify ALB Installation

```bash

kubectl get pods -n azure-alb-system
kubectl get gatewayclass azure-alb-external -o yaml

```

## Create Application gateway for container resource

create resource ALB

```bash

az network alb create -g $RG_NAME -n $AGFC_NAME

```
Create a frontend resource in the App Gateway for Containers

```bash

az network alb frontend create -g $RG_NAME -n $AGFC_FRONTEND_NAME --alb-name $AGFC_NAME

```
## Create a new Subnet for the AppGw for Containers association

```bash
$CLUSTER_SUBNET_ID=$(az vmss list --resource-group $MC_RG --query '[0].virtualMachineProfile.networkProfile.networkInterfaceConfigurations[0].ipConfigurations[0].subnet.id' -o tsv)
echo $CLUSTER_SUBNET_ID

$VNET_NAME=$(az network vnet show --ids $CLUSTER_SUBNET_ID --query name -o tsv)
echo $VNET_NAME

$VNET_RG=$(az network vnet show --ids $CLUSTER_SUBNET_ID --query resourceGroup -o tsv)
echo $VNET_RG

$VNET_ID=$(az network vnet show --ids $CLUSTER_SUBNET_ID --query id -o tsv)
echo $VNET_ID

az network vnet subnet create `
  --resource-group $VNET_RG `
  --vnet-name $VNET_NAME `
  --name $AGFC_SUBNET_NAME `
  --address-prefixes $AGFC_SUBNET_PREFIX `
  --delegations "Microsoft.ServiceNetworking/trafficControllers"

```
Delegate AppGw for Containers Configuration Manager role to AKS Managed Cluster RG

```bash

$ALB_SUBNET_ID=$(az network vnet subnet show --name $AGFC_SUBNET_NAME --resource-group $VNET_RG --vnet-name $VNET_NAME --query '[id]' --output tsv)
echo $ALB_SUBNET_ID

$IDENTITY_ALB_PRINCIPAL_ID=$(az identity show -g $RG_NAME -n $IDENTITY_ALB --query principalId -otsv)
echo $IDENTITY_ALB_PRINCIPAL_ID

$RG_ID=$(az group show --name $RG_NAME --query id -otsv)
echo $RG_ID

az role assignment create --assignee-object-id $IDENTITY_ALB_PRINCIPAL_ID `
        --assignee-principal-type ServicePrincipal `
        --scope $RG_ID `
        --role "fbc52c3f-28ad-4303-a892-8a056630b8f1" 

# Delegate Network Contributor permission for join to association subnet

az role assignment create --assignee-object-id $IDENTITY_ALB_PRINCIPAL_ID `
        --assignee-principal-type ServicePrincipal `
        --scope $ALB_SUBNET_ID `
        --role "4d97b98b-1d4f-4787-a291-c67834d212e7"

```

## Create the AppGw for Containers association and connect it to the referenced subnet

```bash

az network alb association create -g $RG_NAME -n $AGFC_ASSOCIATION `
           --alb-name $AGFC_NAME `
           --subnet $ALB_SUBNET_ID

```

## Create Kubernetes Gateway resource

```bash

$AGFC_ID=$(az network alb show --resource-group $RG_NAME --name $AGFC_NAME --query id -o tsv)
echo $AGFC_ID

@"
apiVersion: v1
kind: Namespace
metadata:
  name: ns-gateway
---
apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: gateway-app
  namespace: ns-gateway
  annotations:
    alb.networking.azure.io/alb-id: $AGFC_ID
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: http-listener
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All # Same
  addresses:
  - type: alb.networking.azure.io/alb-frontend
    value: $AGFC_FRONTEND_NAME
"@ > gateway_byo.yaml

kubectl apply -f gateway_byo.yaml


```

Once the gateway resource has been created, ensure the status is valid, the listener is Programmed, and an address is assigned to the gateway

```bash

kubectl get gateway gateway-app -n ns-gateway -o yaml

```

Get the address assigned to the gateway / frontend

```bash
$FQDN=$(kubectl get gateway gateway-app -n ns-gateway -o jsonpath='{.status.addresses[0].value}')
echo $FQDN

```

## Deploy an application to test it

Deploy a new namespace, deployment and service
```bash

kubectl apply -f ns_deploy_svc.yaml

```

Create a new httproute resource

```bash

kubectl apply -f httproute.yaml
kubectl get httproute -A

```

