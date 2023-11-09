## AKS cluster with outbound type user defined routing (UDR)

UDR is useful when we want to filter and control AKS egress traffic through an NVA like Azure Firewall. This option is widely used by enterprises adopting Hub & Spoke architecture and using Azure Landing Zones.

How it works ?

Simply you enable the UDR mode in AKS to not create or use a Load Balancer for egress traffic. Then you create a Route Table and attach it to the cluster Subnet with a routing rule from 0.0.0.0/0 to the Firewall private IP address.
```
# 4.1. Set configuration via environment variables

$RG="rg-aks-udr"
$LOC="westeurope"
$AKSNAME="aks-udr"
$VNET_NAME="aks-vnet"
$AKSSUBNET_NAME="aks-subnet"
# DO NOT CHANGE FWSUBNET_NAME - This is currently a requirement for Azure Firewall.
$FWSUBNET_NAME="AzureFirewallSubnet"
$FWNAME="hub-firewall"
$FWPUBLICIP_NAME="firewall-publicip"
$FWIPCONFIG_NAME="firewall-config"
$FWROUTE_TABLE_NAME="firewall-routetable"
$FWROUTE_NAME="firewall-route"
$FWROUTE_NAME_INTERNET="firewall-route-internet"

# 4.2. Create a virtual network with multiple subnets

# Create Resource Group

az group create --name $RG --location $LOC

# Create a virtual network with two subnets to host the AKS cluster and the Azure Firewall. Each will have their own subnet. Let's start with the AKS network.

# Dedicated virtual network with AKS subnet

az network vnet create `
    --resource-group $RG `
    --name $VNET_NAME `
    --location $LOC `
    --address-prefixes 10.42.0.0/16 `
    --subnet-name $AKSSUBNET_NAME `
    --subnet-prefix 10.42.1.0/24

# Dedicated subnet for Azure Firewall (Firewall name cannot be changed)

az network vnet subnet create `
    --resource-group $RG `
    --vnet-name $VNET_NAME `
    --name $FWSUBNET_NAME `
    --address-prefix 10.42.2.0/24

# 3. Create and set up an Azure Firewall with a UDR

# Azure Firewall inbound and outbound rules must be configured. The main purpose of the firewall is to enable organizations to configure granular ingress and egress traffic rules into and out of the AKS Cluster.

az network public-ip create -g $RG -n $FWPUBLICIP_NAME -l $LOC --sku "Standard"

# Install Azure Firewall preview CLI extension

az extension add --name azure-firewall --upgrade

# Deploy Azure Firewall

az network firewall create -g $RG -n $FWNAME -l $LOC --enable-dns-proxy true

# Configure Firewall IP Config

az network firewall ip-config create -g $RG -f $FWNAME -n $FWIPCONFIG_NAME --public-ip-address $FWPUBLICIP_NAME --vnet-name $VNET_NAME

# Capture Firewall IP Address for Later Use

$FWPUBLIC_IP=$(az network public-ip show -g $RG -n $FWPUBLICIP_NAME --query "ipAddress" -o tsv)
$FWPRIVATE_IP=$(az network firewall show -g $RG -n $FWNAME --query "ipConfigurations[0].privateIPAddress" -o tsv)
echo $FWPRIVATE_IP

# Create UDR and add a route for Azure Firewall

az network route-table create -g $RG -l $LOC --name $FWROUTE_TABLE_NAME

az network route-table route create -g $RG --name $FWROUTE_NAME --route-table-name $FWROUTE_TABLE_NAME --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address $FWPRIVATE_IP

az network route-table route create -g $RG --name $FWROUTE_NAME_INTERNET --route-table-name $FWROUTE_TABLE_NAME --address-prefix $FWPUBLIC_IP/32 --next-hop-type Internet

# 4. Adding firewall rules

# Add FW Network Rules

az network firewall network-rule create -g $RG -f $FWNAME --collection-name 'aksfwnr' -n 'apiudp' --protocols 'UDP' --source-addresses '*' --destination-addresses "AzureCloud.$LOC" --destination-ports 1194 --action allow --priority 100

az network firewall network-rule create -g $RG -f $FWNAME --collection-name 'aksfwnr' -n 'apitcp' --protocols 'TCP' --source-addresses '*' --destination-addresses "AzureCloud.$LOC" --destination-ports 9000

az network firewall network-rule create -g $RG -f $FWNAME --collection-name 'aksfwnr' -n 'time' --protocols 'UDP' --source-addresses '*' --destination-fqdns 'ntp.ubuntu.com' --destination-ports 123

# Add FW Application Rules

az network firewall application-rule create -g $RG -f $FWNAME --collection-name 'aksfwar' -n 'fqdn' --source-addresses '*' --protocols 'http=80' 'https=443' --fqdn-tags "AzureKubernetesService" --action allow --priority 100

# 5. Associate the route table to AKS

# Associate route table with next hop to Firewall to the AKS subnet

az network vnet subnet update -g $RG --vnet-name $VNET_NAME --name $AKSSUBNET_NAME --route-table $FWROUTE_TABLE_NAME

# 6. Deploy AKS with outbound type of UDR to the existing network

$SUBNETID=$(az network vnet subnet show -g $RG --vnet-name $VNET_NAME --name $AKSSUBNET_NAME --query id -o tsv)
echo $SUBNETID

az aks create -g $RG -n $AKSNAME -l $LOC `
  --node-count 3 `
  --network-plugin azure `
  --outbound-type userDefinedRouting `
  --vnet-subnet-id $SUBNETID `
  --api-server-authorized-ip-ranges $FWPUBLIC_IP

```

Check the created resources. Note there are no Load Balancer created. That is because with UDR mode, egress traffic will be managed by Route Table and Firewall. So there is no need for a Load Balancer for egress. However, if you create a public Kubernetes service or ingress controller, AKS will create the Load Balancer to handle the ingress traffic. In this case, the egress traffic will still be handled by Firewall.

Azure Firewall will deny all traffic by default. However, AKS, during the creation, needs to connect to external resources like MCR, Ubuntu updates server, etc. This is in order to pull system container images and get nodes updates. For that reason, we opened traffic into these resources and more using the Service Tag or also named FQDN Tag called AzureKubernetesService.

Verify the egress traffic is using the Firewall public IP.

```bash

kubectl run nginx --image=nginx
# pod/nginx created

kubectl exec nginx -it -- /bin/bash
# error: unable to upgrade connection: container not found ("nginx")

kubectl get pods
# NAME    READY   STATUS         RESTARTS   AGE
# nginx   0/1     ErrImagePull   0          32s

```
The above error is because Azure Firewall blocks access to non allowed endpoints. Let's create an application rule to allow access to Docker Hub to pull the nginx container image.
```bash

az network firewall application-rule create -g $RG -f $FWNAME --collection-name 'dockerhub-registry' -n 'dockerhub-registry' --action allow --priority 200 --source-addresses '*' --protocols 'https=443' --target-fqdns hub.docker.com registry-1.docker.io production.cloudflare.docker.com auth.docker.io cdn.auth0.com login.docker.com
# Creating rule collection 'dockerhub-registry'.
# {
#   "actions": [],
#   "description": null,
#   "direction": "Inbound",
#   "fqdnTags": [],
#   "name": "dockerhub-registry",
#   "priority": 0,
#   "protocols": [
#     {
#       "port": 443,
#       "protocolType": "Https"
#     }
#   ],
#   "sourceAddresses": [
#     "*"
#   ],
#   "sourceIpGroups": [],
#   "targetFqdns": [
#     "hub.docker.com",
#     "registry-1.docker.io",
#     "production.cloudflare.docker.com",
#     "auth.docker.io",
#     "cdn.auth0.com",
#     "login.docker.com"
#   ]
# }

```

Let's retry now. The image should be pulled and pod works just fine.
```bash
kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# nginx   1/1     Running   0          21m
```
Let's see what IP address is used by the pod to access external services. We use ifconfig.me to view the IP address on the remote server.

```bash
kubectl exec nginx -it -- curl http://ifconfig.me
# Action: Deny. Reason: No rule matched. Proceeding with default action.

```

Again, access is blocked by the Firewall. Create an application rule to allow access to ifconfig.me.
```bash

az network firewall application-rule create -g $RG -f $FWNAME --collection-name 'ifconfig' -n 'ifconfig' --action allow --priority 300 --source-addresses '*' --protocols 'http=80' --target-fqdns ifconfig.me

```

Let's retry again now. We should see the Pod outbound traffic uses the Firewall public IP address.

```bash

kubectl exec nginx -it -- curl http://ifconfig.me
# 20.229.246.163

```