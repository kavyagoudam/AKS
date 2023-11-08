# AKS Egress Traffic with Load Balancer, NAT Gateway, and User Defined Route

## Introduction

Welcome to this lab where we will explore the different outbound types in Azure Kubernetes Service (AKS).
Outbound traffic refers to the network traffic that originates from a pod or node in a cluster and is destined for external destinations.
Outbound traffic will leave the cluster through one of the supported load balancing solutions for egress. 
These solutions are the outbound types in AKS.

There are four outbound types.  
1. Load Balancer (default)  
2. Managed NAT Gateway  
3. User defined NAT Gateway  
4. User Defined Routes (UDR)  

The egress traffic destination could be:  
1. Microsoft Container Registry (MCR) used to pull system container images  
2. Azure Container Registry (ACR) used to pull user container images  
3. Ubuntu or Windows update server used to download node images updates  
4. External resources used by the apps like Azure Key vault, Storage Account, Cosmos DB, etc  
5. External third party REST API  

![](images/65_aks_egress_lb_natgw_udr__architecture.png)

We will discuss the differences between these outbound types and their use cases.
By the end of this lab, participants will have a better understanding of the outbound types available in AKS and will be able to implement the appropriate outbound type to suit their specific requirements.

For each type, you will learn how to configure it, their features and the IP address they use to leave the cluster.


