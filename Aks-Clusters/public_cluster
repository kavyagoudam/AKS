Public AKS Cluster access

Control plane is exposed on Public FQDN and public ip address, which will be accessable via internet. Authorised ip ranges can be enabled to whitelist the ip ranges.


To create the public cluster follow below steps

1. Create resource group.

az group create -n public-cluster -l eastus

2. create public AKS cluster

az aks create -n public-aks-cluster -g public-cluster

3. fetch public endpoint.

az aks show -n public-aks-cluster -g public-cluster --query fqdn
# The behavior of this command has been altered by the following extension: aks-preview
# "public-aks-public-cluster-5c09ef-2wqkaxek.hcp.eastus.azmk8s.io"

4. nslookup the given fqdn

nslookup public-aks-public-cluster-5c09ef-2wqkaxek.hcp.eastus.azmk8s.io
# Server:  reliance.reliance
# Address:  2405:201:d01d:3010::c0a8:1d01

# DNS request timed out.
#    timeout was 2 seconds.
# Non-authoritative answer:
# Name:    public-aks-public-cluster-5c09ef-2wqkaxek.hcp.eastus.azmk8s.io
# Address:  52.149.241.137

5. check for private endpoint. since AKS is public cluster we not be able to see the private endpoint.

az aks show -n public-aks-cluster -g public-cluster --query privateFqdn

In public cluster, worker nodes connection go though the control plane via public ip.

To demonstrate that, 

1. get the svc of aks cluster.
kubectl get svc
# NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
# kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   13m

By default, kubernetes service is created in cluster.  

1. When you describe the kubernetes service you will see the public ip address in Endpoints. this is the public ip address, worker nodes will use to connect to the controle plane.

kubectl describe svc kubernetes
# Name:              kubernetes
# Namespace:         default
# Labels:            component=apiserver
#                    provider=kubernetes
# Annotations:       <none>
# Selector:          <none>
# Type:              ClusterIP
# IP Family Policy:  SingleStack
# IP Families:       IPv4
# IP:                10.0.0.1
# IPs:               10.0.0.1
# Port:              https  443/TCP
# TargetPort:        443/TCP
# Endpoints:         52.149.241.137:443
# Session Affinity:  None
# Events:            <none>


