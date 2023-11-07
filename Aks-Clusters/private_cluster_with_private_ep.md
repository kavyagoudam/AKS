## private AKS CLuster Access

1. Control plane is exposed to on public FQDN but with private ip address
2. Control plane endpoint is not exposed to the internet
3. public FQDN could be disabled or enabled
4. worker node connects to the control plane via private endpoint
5. Private FQDN is resolvable only thorugh the private DNS zone


To create the Private cluster follow below steps

1. Create resource group.
az group create -n private-cluster -l eastus

2. create private AKS cluster
az aks create -n private-aks-cluster -g private-cluster --enable-private-cluster

3. fetch public endpoint, You will get the public endpoint but it will be resolved to private ip of the cluster.

az aks show -n private-aks-cluster -g private-cluster --query fqdn
# The behavior of this command has been altered by the following extension: aks-preview
# "private-ak-private-cluster-5c09ef-1kuj8sjt.hcp.eastus.azmk8s.io"

4. nslookup the given fqdn
nslookup private-aks-private-cluster-5c09ef-2wqkaxek.hcp.eastus.azmk8s.io
# Server:  reliance.reliance
# Address:  2405:201:d01d:3010::c0a8:1d01

# DNS request timed out.
# timeout was 2 seconds.
# DNS request timed out.
#    timeout was 2 seconds.
# DNS request timed out.
#     timeout was 2 seconds.
# Non-authoritative answer:
# Name:    private-ak-private-cluster-5c09ef-1kuj8sjt.hcp.eastus.azmk8s.io
# Address:  10.224.0.4
5. when we try to querry private endpoint, we will get private FQDN with private link(observe in the output)
az aks show -n private-aks-cluster -g private-cluster --query privateFqdn
# The behavior of this command has been altered by the following extension: aks-preview
# "private-ak-private-cluster-5c09ef-ldftc27u.504074f4-bc92-4c82-9a87-12d4306594de.privatelink.eastus.azmk8s.io"

6. when we try to resolve private fqdn, it will be not be resolvable. since it has private ip.

nslookup private-ak-private-cluster-5c09ef-ldftc27u.504074f4-bc92-4c82-9a87-12d4306594de.privatelink.eastus.azmk8s.io

# Server:  reliance.reliance
# Address:  2405:201:d01d:3010::c0a8:1d01

# DNS request timed out.
#    timeout was 2 seconds.
# *** reliance.reliance can't find private-ak-private-cluster-5c09ef-ldftc27u.504074f4-bc92-4c82-9a87-12d4306594de.privatelink.eastus.azmk8s.io: Non-existent domain
7. we can disable the public FQDN using the below command.

az aks update -n private-aks-cluster -g private-cluster --disable-public-fqdn
if you query again, you will not get fqdn.
az aks show -n private-aks-cluster -g private-cluster --query fqdn
