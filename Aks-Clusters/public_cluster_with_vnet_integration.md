Public AKS cluster with vnet integration
1. control plane is exposed on public FQDN with public ip address
2. control plane endpoint is exposed to the internet
3. public FQDN could not be disabled except on private cluster
4. DevOps CD pipelines could Access through public or private ip
5. worker node access control plane through inernal load balancer( private ip)



To create the public cluster follow below steps

1. Create resource group.
az group create -n public-cluster-vnet-int -l eastus
2. vnet integration is still under preview. So we need register "EnableAPIServerVnetIntegrationPreview"
az extension add --name aks-preview
az feature register --namespace "Microsoft.ContainerService" --name "EnableAPIServerVnetIntegrationPreview"
3. To verify, registration.
az feature show --namespace "Microsoft.ContainerService" --name "EnableAPIServerVnetIntegrationPreview"

4. create public AKS cluster with vnet integration

az aks create -n public-aks-cluster -g public-cluster-vnet-int --enable-apiserver-vnet-integration

5. fetch public endpoint, You will get the public endpoint but it will be resolved to public ip of the cluster.

az aks show -n public-aks-cluster -g public-cluster-vnet-int --query fqdn
# The behavior of this command has been altered by the following extension: aks-preview
# "public-aks-public-cluster-v-5c09ef-i82tbb2n.hcp.eastus.azmk8s.io"

6. nslookup the given fqdn
nslookup public-aks-public-cluster-v-5c09ef-i82tbb2n.hcp.eastus.azmk8s.io
# Server:  reliance.reliance
# Address:  2405:201:d01d:3010::c0a8:1d01

# Non-authoritative answer:
# Name:    public-aks-public-cluster-v-5c09ef-i82tbb2n.hcp.eastus.azmk8s.io
# Address:  20.237.11.41

7. when we try to query private endpoint, we will not get private FQDN.
az aks show -n public-aks-cluster -g public-cluster-vnet-int --query privateFqdn
Private FQDN doesn't exist but private IP address will be available. .

how worker nodes access to control plane?
  AKS vnet, integration creates internal load balancer.
  ILB private ip address is used to access the control plane
  AKS vnet integration creates  new subnet in AKS Vnet
  It injects the ILB and configure delegation
1.To check that
kubectl get svc 
kubectl describe svc kubernetes
# Name:              kubernetes
# Namespace:         default
# Labels:            component=apiserver
#                   provider=kubernetes
# Annotations:       <none>
# Selector:          <none>
# Type:              ClusterIP
# IP Family Policy:  SingleStack
# IP Families:       IPv4
# IP:                10.0.0.1
# IPs:               10.0.0.1
# Port:              https  443/TCP
# TargetPort:        443/TCP
# Endpoints:         10.226.0.4:443
# Session Affinity:  None
# Events:            <none>
You can see Endpoint is private ip address(10.226.0.4), which will be used to access the control plane.

In the console go to - managed cluster resource group An internal load balancer called kube-apiserver is created with the same ip.

