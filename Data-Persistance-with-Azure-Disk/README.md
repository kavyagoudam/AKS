## Data persistance with Azure Disk

Data should be saved or persisted outside the pods, to Achieve this in Azure, we have multiple options
1. Azure files
2. Azure blob
3. Azure Disk

In this article we will be focusing on Azure Disk


Azure Disk in LRS mode

```bash
_______________________
|           |pods2|---|-----|
|     |pods2|         |     |
|                     |     |       _______________          ________> |replica1|
|                     |     |-----> | Azure Disk  |          | 
|_____________________|             |_____________|------------------> |replica2|
                                /mnt/AzureDisk/  /Database   |
                                                             |_______> |replica3|

_______________________
|                      |
|                      |
|                      |
|                      | 
|______________________|

```

Azure disk by default in RWO mode, it means only one mode can be mounted at a time. All pods inside a node can be mounted to disk, but only one pod can write at a time

By Default Azure Disk will be mounted in LRS mode


## 0. Setup demo environment

```powershell
# Variables
$AKS_RG="rg-aks-az"
$AKS_NAME="aks-cluster"

# Create and connect to AKS cluster
az group create --name $AKS_RG --location westeurope

az aks create -n $AKS_NAME -g $AKS_RG --node-count 3 --zones 1 2 3 

az aks get-credentials -n $AKS_NAME -g $AKS_RG --overwrite-existing

kubectl get nodes
```

## 1. Deploy a sample deployment with PVC (Azure Disk with LRS)

```powershell
kubectl apply -f lrs-disk-deploy.yaml
# persistentvolumeclaim/azure-managed-disk-lrs created
# deployment.apps/nginx-lrs created
```

Verify resources deployed successfully

```powershell
kubectl get pods,pv,pvc
# NAME                             READY   STATUS    RESTARTS   AGE
# pod/nginx-lrs-5fc6787dff-2f8zj   1/1     Running   0          70s
# NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS
# persistentvolume/pvc-7734f114-cdef-4cf9-ae60-9cd840f641a6   5Gi        RWO            Delete           Bound    default/azure-managed-disk-lrs   managed-csi 

# NAME                                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# persistentvolumeclaim/azure-managed-disk-lrs   Bound    pvc-7734f114-cdef-4cf9-ae60-9cd840f641a6   5Gi        RWO            managed-csi    70s
```

Check the worker node for the pod

```powershell
kubectl get pods -o wide
# NAME                         READY   STATUS    RESTARTS   AGE   IP            NODE
# nginx-lrs-5fc6787dff-2f8zj   1/1     Running   0          27s   10.244.2.12   aks-nodepool1-49785470-vmss000001

kubectl get nodes
# NAME                                STATUS   ROLES   AGE     VERSION
# aks-nodepool1-49785470-vmss000000   Ready    agent   3h32m   v1.23.12
# aks-nodepool1-49785470-vmss000001   Ready    agent   3h32m   v1.23.12
# aks-nodepool1-49785470-vmss000002   Ready    agent   3h32m   v1.23.12
```

Worker Nodes are deployed into the 3 Azure Availability Zones

```powershell
Set-Alias -Name grep -Value select-string # if using powershell

kubectl describe nodes | grep topology.kubernetes.io/zone
# topology.kubernetes.io/zone=westeurope-1
# topology.kubernetes.io/zone=westeurope-2
# topology.kubernetes.io/zone=westeurope-3
```

Check the Availability Zone for our pod

```powershell
# Get Pod's node name
$NODE_NAME=$(kubectl get pods -l app=nginx-lrs -o jsonpath='{.items[0].spec.nodeName}')
echo $NODE_NAME
# aks-nodepool1-49785470-vmss000001

kubectl get nodes $NODE_NAME -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}'
# westeurope-2

kubectl describe nodes $NODE_NAME | grep topology.kubernetes.io/zone
# topology.kubernetes.io/zone=westeurope-2
```

## 2. Simulate node failure (delete node)

```powershell
kubectl delete node $NODE_NAME
# node "aks-nodepool1-49785470-vmss000001" deleted
```

Check the pod, we have a problem!

```powershell
kubectl get pods -o wide
# NAME                         READY   STATUS    RESTARTS   AGE    IP       NODE
# nginx-lrs-5fc6787dff-wk99x   0/1     Pending   0          107s   <none>   <none>
```

Check Pod events

```powershell
kubectl describe pod | grep Warning
# Warning  FailedScheduling  20s (x6 over 5m47s)  default-scheduler  0/3 nodes are available: 1 node(s) were unschedulable, 2 node(s) had volume node affinity conflict.
```

The problem here is that the PV (Azure Disk) is zonal (uses 1 single availability zone (AZ)). 
It could be detached from the VM and mounted into another VM. But that VM must be within the same AZ.
If it is not within then same AZ, it will fail. And the pod will not be able to mount the disk.

## 3. Resolving the issue (adding nodes in the same AZ)

What if we did have another VM inside the same Availability Zone ?

Adding nodes to the availability zones

```powershell
az aks scale -n $AKS_NAME -g $AKS_RG --node-count 6
```

Check the created Nodes (note the missing node we deleted earlier!)

```powershell
kubectl get nodes
# NAME                                STATUS   ROLES   AGE    VERSION
# aks-nodepool1-49785470-vmss000000   Ready    agent   5h3m   v1.23.12
# aks-nodepool1-49785470-vmss000002   Ready    agent   5h3m   v1.23.12
# aks-nodepool1-49785470-vmss000003   Ready    agent   100s   v1.23.12
# aks-nodepool1-49785470-vmss000004   Ready    agent   100s   v1.23.12
# aks-nodepool1-49785470-vmss000005   Ready    agent   102s   v1.23.12
```

Verify we have nodes in all 3 availability zones (including AZ-2 where we have the Disk/PV)

```powershell
kubectl describe nodes | grep topology.kubernetes.io/zone
# topology.kubernetes.io/zone=westeurope-1
# topology.kubernetes.io/zone=westeurope-3
# topology.kubernetes.io/zone=westeurope-1
# topology.kubernetes.io/zone=westeurope-2
# topology.kubernetes.io/zone=westeurope-3
```

Check pod is now rescheduled into a node within AZ-2

```powershell
kubectl get pods -o wide
# NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE
# nginx-lrs-5fc6787dff-wk99x   1/1     Running   0          29m   10.244.5.2   aks-nodepool1-49785470-vmss000004
```

Verify the node is in availability zone 2

```powershell
$NODE_NAME=$(kubectl get pods -l app=nginx-lrs -o jsonpath='{.items[0].spec.nodeName}')
$NODE_NAME
# aks-nodepool1-49785470-vmss000004

kubectl get nodes $NODE_NAME -o jsonpath='{.metadata.labels.topology\.kubernetes\.io/zone}'
# westeurope-2
```

## Conclusion

In AKS, we are encourages to use High availability nodes, it means nodes to spread accros the Availability zone's. If pods scheduled in other nodes, Azure disk with LRS mode will not wotk. 

To overcome this problem Azure offer another mode called ZRS

## ZRS mode

With ZRS mode, each replica of azure disk will be stored in each zones.


```bash
_______________________
|           |pods2|---|---->|
|     |pods2|         |     |
|                     |     |
|                     |     | 
|_____________________|     |  unmount 
                        AZ1 | 
--------------------------- |                                                   |------------> |replica 1|
                            |                                                   |                       AZ1
_______________________     |                _________________________          |          -------------------
|                      |    |                |                        |         |
|         |pod |-------|--->|--------------> |       Azure Disk       | --------|------------> |replica 2|
|      |pod2|          |    |  mount         |________________________|         |                       AZ2
|                      |    |                                                   |         ---------------------
|______________________|    |                                                   |
                        AZ2 |                                                   |-------------> |replica 3|
--------------------------- |                                                                           AZ3
_______________________     |                                                             ---------------------
|                      |    |
|         |pod |-------|--->|
|      |pod2|          |
|                      | 
|______________________|
                        AZ3
----------------------------


```
## Deploy a sample deployment with PVC (Azure Disk with zrs)

```bash

# for any confilict , delete lrs_disk_deploy.yaml
kubectl delete -f lrs_disk_deploy.yaml
kubectl apply -f zrs_disk_deploy.yaml -f .\zrs_disk_pvc_sc.yaml

```

with ZRS, it is possible to unmount volume from one node az1 to mount volume from node2 of az2, But still it should be mounted to one node at a time. To overcome this issue, we have another option called Shared Disk



## Shared Disk

With shared disk, we can mount azure disk to more node in different AZ's 

But in shared disk, will be in CRD mode, even though it is mounted with options RWX ( it is used to enabled multiple mount in node)


```bash
_______________________
|           |pods2|---|---->|
|     |pods2|         |     |
|                     |     |
|                     |     | 
|_____________________|     |  unmount 
                        AZ1 | 
--------------------------- |                                                   |------------> |replica 1|
                            |                                                   |                       AZ1
_______________________     |                _________________________          |          -------------------
|                      |    |                |                        |         |
|         |pod |-------|--->|--------------> |       Azure Disk       | --------|------------> |replica 2|
|      |pod2|          |    |  mount         |________________________|         |                       AZ2
|                      |    |                                                   |         ---------------------
|______________________|    |                                                   |
                        AZ2 |                                                   |-------------> |replica 3|
--------------------------- |                                                                           AZ3
_______________________     |                                                             ---------------------
|                      |    |
|         |pod |-------|--->|
|      |pod2|          |
|                      | 
|______________________|
                        AZ3
----------------------------

```


## Setup demo environment

```powershell
# Variables
$AKS_RG="rg-aks-shared-disk"
$AKS_NAME="aks-cluster"

# Create and connect to AKS cluster
az group create --name $AKS_RG --location westeurope

az aks create --name $AKS_NAME --resource-group $AKS_RG --node-count 6 --zones 1 2 3 --kubernetes-version "1.25.2" --network-plugin azure

az aks get-credentials -n $AKS_NAME -g $AKS_RG --overwrite-existing

kubectl get nodes
# NAME                                STATUS   ROLES   AGE   VERSION
# aks-nodepool1-39303592-vmss000000   Ready    agent   67m   v1.25.2
# aks-nodepool1-39303592-vmss000001   Ready    agent   66m   v1.25.2
# aks-nodepool1-39303592-vmss000002   Ready    agent   67m   v1.25.2
# aks-nodepool1-39303592-vmss000003   Ready    agent   54m   v1.25.2
# aks-nodepool1-39303592-vmss000004   Ready    agent   54m   v1.25.2
# aks-nodepool1-39303592-vmss000005   Ready    agent   54m   v1.25.2
```

## Deploy a sample deployment with StorageClass and PVC (ZRS Shared Azure Disk)

```powershell
kubectl apply -f zrs_shared_disk_pvc_sc.yaml,zrs_shared_disk_deploy.yaml
# storageclass.storage.k8s.io/zrs-shared-managed-csi created
# persistentvolumeclaim/zrs-shared-pvc-azuredisk created
# deployment.apps/deployment-azuredisk created
```

Verify resources deployed successfully

```powershell
kubectl get pods,pv,pvc,sc
# NAME                        READY   STATUS    RESTARTS   AGE
# pod/nginx-79f645c46-6vzq4   1/1     Running   0          5m19s
# pod/nginx-79f645c46-jpjtl   1/1     Running   0          5m19s
# pod/nginx-79f645c46-l5slj   1/1     Running   0          5m19s

# NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
#   STORAGECLASS             REASON   AGE
# persistentvolume/pvc-b1d0fc7c-6c48-4020-b830-5c9a33c116c1   256Gi      RWX            Delete           Bound    default/zrs-shared-pvc-azuredisk   zrs-shared-managed-csi            5m17s

# NAME                                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS             AGE
# persistentvolumeclaim/zrs-shared-pvc-azuredisk   Bound    pvc-b1d0fc7c-6c48-4020-b830-5c9a33c116c1   256Gi      RWX            zrs-shared-managed-csi   5m19s
```

Verify the pods are deployed in multiple nodes

```powershell
kubectl get pods -o wide
# NAME                    READY   STATUS    RESTARTS   AGE     IP             NODE                             
# nginx-79f645c46-6vzq4   1/1     Running   0          6m24s   10.224.0.123   aks-nodepool1-39303592-vmss000000
# nginx-79f645c46-jpjtl   1/1     Running   0          6m24s   10.224.0.61    aks-nodepool1-39303592-vmss000004
# nginx-79f645c46-l5slj   1/1     Running   0          6m24s   10.224.0.126   aks-nodepool1-39303592-vmss000005
```

Verify the pods are deployed in multiple availability zones

```powershell
Set-Alias -Name grep -Value select-string # if using powershell
kubectl describe nodes | grep topology.kubernetes.io/zone
# topology.kubernetes.io/zone=westeurope-1
# topology.kubernetes.io/zone=westeurope-2
# topology.kubernetes.io/zone=westeurope-3
# topology.kubernetes.io/zone=westeurope-1
# topology.kubernetes.io/zone=westeurope-2
# topology.kubernetes.io/zone=westeurope-3

$NODE_RG=$(az aks show -g $AKS_RG -n $AKS_NAME --query nodeResourceGroup -o tsv)
echo $NODE_RG
# MC_rg-aks-shared-disk_aks-cluster_westeurope

az disk list -g $NODE_RG -o table
# Name                                      ResourceGroup                                 Location    Zones    Sku          SizeGb    ProvisioningState
# ----------------------------------------  --------------------------------------------  ----------  -------  -----------  --------  -------------------
# pvc-8ed6c59d-4de5-44ba-9f25-00e58992daea  MC_rg-aks-shared-disk_aks-cluster_westeurope  westeurope           Premium_ZRS  256       Succeeded
```

Check the Disk config on the Azure portal

![](images/45_shared_disk__shared-disk.png)

Verify the Disk is accessible by 3 nodes

```powershell
az disk list -g $NODE_RG --query [0].managedByExtended
# [
#   "/subscriptions/82f6d75e-85f4-434a-ab74-5dddd9fa8910/resourceGroups/mc_rg-aks-shared-disk_aks-cluster_westeurope/providers/Microsoft.Compute/virtualMachineScaleSets/aks-nodepool1-39303592-vmss/virtualMachines/aks-nodepool1-39303592-vmss_5",
#   "/subscriptions/82f6d75e-85f4-434a-ab74-5dddd9fa8910/resourceGroups/mc_rg-aks-shared-disk_aks-cluster_westeurope/providers/Microsoft.Compute/virtualMachineScaleSets/aks-nodepool1-39303592-vmss/virtualMachines/aks-nodepool1-39303592-vmss_4",
#   "/subscriptions/82f6d75e-85f4-434a-ab74-5dddd9fa8910/resourceGroups/mc_rg-aks-shared-disk_aks-cluster_westeurope/providers/Microsoft.Compute/virtualMachineScaleSets/aks-nodepool1-39303592-vmss/virtualMachines/aks-nodepool1-39303592-vmss_0"
# ]
```

Verify access to Shared Disk on Pod #1

```powershell
$POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
echo $POD_NAME
# nginx-79f645c46-b4vd6

kubectl exec -it $POD_NAME -- dd if=/dev/zero of=/dev/sdx bs=1024k count=100
# 100+0 records in
# 100+0 records out
# 104857600 bytes (105 MB, 100 MiB) copied, 0.047578 s, 2.2 GB/s
```

Verify access to Shared Disk on Pod #2

```powershell
$POD_NAME=$(kubectl get pods -l app=nginx -o jsonpath='{.items[1].metadata.name}')
echo $POD_NAME
# nginx-79f645c46-jp9r8

kubectl exec -it $POD_NAME -- dd if=/dev/zero of=/dev/sdx bs=1024k count=100
# 100+0 records in
# 100+0 records out
# 104857600 bytes (105 MB, 100 MiB) copied, 0.0502999 s, 2.1 GB/s
```

## Resources
https://github.com/kubernetes-sigs/azuredisk-csi-driver/blob/master/deploy/example/failover/README.md