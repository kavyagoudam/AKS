## External-DNS

## Introduction:

External DNS is a kubernetes controller that simplifies the management of DNS records for services and ingress resources deployed in a Kubernetes cluster, making it easier to associate domain names with services exposed externally to the cluster. It automates the process of updating DNS records in external DNS providers to reflect changes in the Kubernetes cluster, such as the creation or deletion of Ingress resources.

External DNS watches for new Ingresses and Services with specific annotations, then creates corresponding DNS records in Azure DNS or other DNS provider.

External DNS available as a opensource project in GitHub:  https://github.com/kubernetes-sigs/external-dns

## Authentication:

External DNS pods authenticate to Azure DNS using one of the three methods.
1. Service principle
2. kubelet Managed Identity
3. User Assigned MAnaged Identity

## Flow Diagram:- 
refer the image FlowDiagram.drawio.png

1. Whenever a service or ingress resources are created, An public ip will be created. by default. 
2. For each public ip we need to create a A record, in DNS Zone
3. Using a External DNS, We can automate the process of updated A record in DNS Zone
4. Service principle will be created with DNS Zone Contributor access to DNS Zone
5. Cluster Role is created with appropriate permission to service and ingress resource.


##  AKS Setup Using Azure CLI

    Create a AKS cluster with public access enabled
``` bash
$AKS_RG="rg-ext-dns"
$AKS_NAME="ext-dns-demo"
az group create -n $AKS_RG -l eastus
az aks create -g $AKS_RG -n $AKS_NAME --kubernetes-version "1.26.6" --node-count 3 --network-plugin azure
az aks get-credentials -n $AKS_NAME -g $AKS_RG --overwrite-existing
```

Install the Ingress Controller using helm charts

``` bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx `
     --create-namespace `
     --namespace ingress-nginx `
     --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

Create Azure DNS Zone
``` bash
$DNS_ZONE_NAME="<replace with your domain name>"
$DNS_ZONE_RG="azure-dns-for-ext-dns"
az group create -n $DNS_ZONE_RG -l east-us
az network dns zone create -g $DNS_ZONE_RG -n $DNS_ZONE_NAME
```

Authentication between External Pods and Azure DNS

External DNS pods authenticate to Azure DNS using one of the three methods.
1. Service principle
   ``` bash
    $EXTERNALDNS_SPN_NAME="spn-external-dns-aks"

    # Create the service principal
    az ad sp create-for-rbac --name $EXTERNALDNS_SPN_NAME
    {   
        "appId": "XXXXXXXXXXXXXXXXXXXXXXXXXXX",
        "displayName": "spn-external-dns-aks",
        "password": "********************",
        "tenant": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
    }
    $EXTERNALDNS_SPN_APP_ID= "<appId of spn |refer output of above command>"
    $EXTERNALDNS_SPN_PASSWORD="<password of spn | refer output of above command>"
```
3. kubelet Managed Identity
4. User Assigned MAnaged Identity

Assign the RBAC for the service principal :

Grant access to Azure DNS zone for the service principal.

# fetch DNS id and RG used to grant access to the service principal
``` bash
$DNS_ZONE_ID=$(az network dns zone show -n $DNS_ZONE_NAME -g $DNS_ZONE_RG --query "id" -o tsv)
$DNS_ZONE_RG_ID=$(az group show -g $DNS_ZONE_RG --query "id" -o tsv)
```
# assign reader to the resource group
``` bash
az role assignment create --role "Reader" --assignee $EXTERNALDNS_SPN_APP_ID --scope $DNS_ZONE_RG_ID
```
# assign contributor to DNS Zone itself
``` bash
az role assignment create --role "DNS Zone Contributor" --assignee $EXTERNALDNS_SPN_APP_ID --scope $DNS_ZONE_ID
```
verify the role Assignment
``` bash
az role assignment list --all --assignee $EXTERNALDNS_SPN_APP_ID -o table
```

Create a Kubernetes secret for the service principal
``` bash
@"
{
  "tenantId": "$(az account show --query tenantId -o tsv)",
  "subscriptionId": "$(az account show --query id -o tsv)",
  "resourceGroup": "$DNS_ZONE_RG",
  "aadClientId": "$EXTERNALDNS_SPN_APP_ID",
  "aadClientSecret": "$EXTERNALDNS_SPN_PASSWORD"
}
"@ > azure.json
```

Deploy the credentials as a Kubernetes secret.
``` bash
kubectl create namespace external-dns
namespace/external-dns created

kubectl create secret generic azure-config-file -n external-dns --from-file azure.json
secret/azure-config-file created
```
6. Deploy External DNS

External DNS can be deployed using menifest file or helm charts. in this example, we will be using menifest file from the GitHub repositiry.

https://github.com/kubernetes-sigs/external-dns


Before deploying the yaml, change the namespace name in ClusterRoleBinding in external-dns.yaml file
``` bash
kubectl create ns external-dns

kubectl apply -f external-dns.yaml -n external-dns
```
verify the deployment
``` bash
kubectl get pods,sa -n external-dns
```
7. Using the external DNS with kubernetes service
``` bash
    kubectl apply -f app-lb.yaml 
    kubectl get pods,svc
```
Check the logs in  external DNS pod
``` bash
kubectl logs <external-dns-pod name> -n external-dns
```
In portal go to DNS zone to check Updated A record in corresponding zones

Create a sample app exposed through ingress
``` bash
kubectl apply -f app-ingress.yaml

kubectl get pods,svc,ingress
```
Reference:
    https://github.com/HoussemDellai/docker-kubernetes-course/tree/main

Future work:
    1. Check for other authentication method.
    2. automate the process using powershell / golang
