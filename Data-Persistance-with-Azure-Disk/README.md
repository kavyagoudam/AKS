## Data persistance with Azure Disk

Data should be saved or persisted outside the pods, to Achieve this in Azure, we have multiple options
1. Azure files
2. Azure blob
3. Azure Disk

In this article we will be focusing on Azure Disk


Azure Disk in LRS mode

![](images/LRS_Disk.drawio)

Azure disk by default in RWO mode, it means only one mode can be mounted at a time. All pods inside a node can be mounted to disk, but only one pod can write at a time

