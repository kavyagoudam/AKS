## Using Azure Blob Storage in AKS

## Introduction

The Azure Blob storage Container Storage Interface (CSI) driver is a CSI specification-compliant driver used by Azure Kubernetes Service (AKS) to manage the lifecycle of Azure Blob storage. The CSI is a standard for exposing arbitrary block and file storage systems to containerized workloads on Kubernetes.

By adopting and using CSI, AKS now can write, deploy, and iterate plug-ins to expose new or improve existing storage systems in Kubernetes. Using CSI drivers in AKS avoids having to touch the core Kubernetes code and wait for its release cycles.

1. Azure Blob support 2 type of storage a. NFSv3 b. BlobFiles
2. Blob-csi-driver is available as a open source project in github repository
3. we can install the CSI driver as add-on plugins in --enable-blob-driver

# Demo
1. setup envirornment

```bash
$AKS_RG="rg-aks-storage-blob"
$AKS_NAME="aks-cluster"

az group create --name $AKS_RG --location westeurope

az aks create --name $AKS_NAME --resource-group $AKS_RG --node-count 3 --zones 1 2 3 --network-plugin azure  --enable-blob-driver

az aks get-credentials -n $AKS_NAME -g $AKS_RG --overwrite-existing

kubectl get nodes
```

2. Verify the blob driver (DaemonSet) was installed

```bash
Set-Alias -Name grep -Value select-string # if using powershell
kubectl get pods -n kube-system | grep csi
```
When the Azure Blob storage CSI driver is enabled on AKS, there are two built-in storage classes: azureblob-fuse-premium and azureblob-nfs-premium.
```bash
kubectl get storageclass
kubectl get sc azureblob-fuse-premium -o yaml
kubectl get sc azureblob-nfs-premium -o yaml
```

3.  Deploy a sample Statefulset with mount volume

```bash
kubectl apply -f azure_blob_nfs_ss.yaml
```

Check created resources
```bash
kubectl get sts,pods,pvc,pv,secret
#Check the created secret
kubectl get secret -o yaml
```

4. Check the following configuration from console

Check the created storage account in Azure portal
View Storage Account configuration
Check the network configuration and note how it added access to AKS VNET
Check the storage account access keys
View the created container blob, note the same name as the PVC
View the content (the file we created inside the container) of the blob
View the file content

Get the created storage accounts using Azure CLI
```bash
$NODE_RG=$(az aks show -g $AKS_RG -n $AKS_NAME --query nodeResourceGroup -o tsv)

```
View the mounted Blob volume within the pod

```bash
kubectl exec -it statefulset-blob-nfs-0 -- df -h
```
View the content of a Blob file

```bash
kubectl exec -it statefulset-blob-nfs-0 -- cat /mnt/azureblob/data
```
What if I scale the Statefulset

```bash
kubectl scale statefulset statefulset-blob-nfs --replicas=3
```

Scaling statefull will not create new storage account, but it will create seperate containers(pvc-*)

# Attach Azure Blob Fuse to AKS using Managed Identity
## Introduction

In this lab, we will attach an Azure Blob Fuse to an AKS pod. This could be done either dynamically or statically. Dynamically means that AKS will create and manage automatically the storage account and blobs. Static creation means we can bring our own storage account and blobs. But with this option we need to configure the authentication and authorization.

The available options are:

Azure Storage Account key
Azure Storage Account key SAS Token
Azure Service Principal (SPN)
Azure Managed Identity

We will explore the latter option using Managed Identity. 

1. Create Storage account
```bash
$STORAGE_ACCOUNT_NAME="storage4aks013"
$CONTAINER_NAME="container01"
$IDENTITY_NAME="identity-storage-account"
az storage account create -n $STORAGE_ACCOUNT_NAME -g $AKS_RG -l westeurope --sku Premium_ZRS --kind BlockBlobStorage

az storage container create --account-name $STORAGE_ACCOUNT_NAME -n $CONTAINER_NAME

```

Upload a file into the SA container

```bash
$STORAGE_ACCOUNT_KEY=$(az storage account keys list --account-name $STORAGE_ACCOUNT_NAME --query '[0].value' -o tsv)

az storage blob upload `
           --account-name $STORAGE_ACCOUNT_NAME `
           -c $CONTAINER_NAME `
           --name blobfile.html `
           --file blobfile.html `
           --auth-mode key `
           --account-key $STORAGE_ACCOUNT_KEY
```
Assign admin role to my self
```bash
$CURRENT_USER_ID=$(az ad signed-in-user show --query id -o tsv)
$STORAGE_ACCOUNT_ID=$(az storage account show -n $STORAGE_ACCOUNT_NAME --query id)

az role assignment create --assignee $CURRENT_USER_ID `
        --role "Storage Account Contributor" `
        --scope $STORAGE_ACCOUNT_ID
```
Create Managed Identity
```bash
az identity create -g $AKS_RG -n $IDENTITY_NAME
```

Assign RBAC role

```bash
$IDENTITY_CLIENT_ID=$(az identity show -g $AKS_RG -n $IDENTITY_NAME --query "clientId" -o tsv)
$STORAGE_ACCOUNT_ID=$(az storage account show -n $STORAGE_ACCOUNT_NAME --query id)

az role assignment create --assignee $IDENTITY_CLIENT_ID `
        --role "Storage Blob Data Owner" `
        --scope $STORAGE_ACCOUNT_ID
```
Attach Managed Identity to AKS VMSS (System nodepool only recommended)
```bash
$IDENTITY_ID=$(az identity show -g $AKS_RG -n $IDENTITY_NAME --query "id" -o tsv)

$NODE_RG=$(az aks show -g $AKS_RG -n $AKS_NAME --query nodeResourceGroup -o tsv)

$VMSS_NAME=$(az vmss list -g $NODE_RG --query [0].name -o tsv)

az vmss identity assign -g $NODE_RG -n $VMSS_NAME --identities $IDENTITY_ID
```

Configure Persistent Volume (PV) with managed identity

```bash
@"
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-blob
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: azureblob-fuse-premium
  mountOptions:
    - -o allow_other
    - --file-cache-timeout-in-seconds=120
  csi:
    driver: blob.csi.azure.com
    readOnly: false
    volumeHandle: $STORAGE_ACCOUNT_NAME-$CONTAINER_NAME
    volumeAttributes:
      resourceGroup: $AKS_RG
      storageAccount: $STORAGE_ACCOUNT_NAME
      containerName: $CONTAINER_NAME
      # refer to https://github.com/Azure/azure-storage-fuse#environment-variables
      AzureStorageAuthType: msi  # key, sas, msi, spn
      AzureStorageIdentityResourceID: $IDENTITY_ID
"@ > pv_blobfuse.yaml
```

Deploy the application

```bash
kubectl apply -f pv_blobfuse.yaml -f pvc_blobfuse.yaml -f nginx_pod_blob.yaml
kubectl get pods,svc,pvc,pv
```
Verify the Blob storage mounted successfully

```bash
$POD_NAME=$(kubectl get pods -l app=nginx-app -o jsonpath='{.items[0].metadata.name}')

kubectl exec -it $POD_NAME -- df -h
kubectl exec -it $POD_NAME -- ls /usr/share/nginx/html
```

