
## 1. AKS cluster with outbound type load balancer

Let's create an AKS cluster with default LoadBalancer outboundType and let's see how it works.

```sh
az group create -n rg-aks-lb -l westeurope

az aks create -g rg-aks-lb -n aks-lb `
              --outbound-type loadBalancer # default

az aks get-credentials -g rg-aks-lb -n aks-lb
```

Check the created resources in the node resource group. 
Note the public Load Balancer (LB) and the public IP address (PIP). 
These resources will be used for egress traffic.

![](images/65_aks_egress_lb_natgw_udr__lb-resources.png)

Verify that outbound egress traffic uses Load Balancer public IP address.

```sh
kubectl run nginx --image=nginx
kubectl exec nginx -it -- curl http://ifconfig.me
# 20.126.14.246
```

![](images/65_aks_egress_lb_natgw_udr__lb-pip.png)

Note that is the public IP of the Load Balancer. This is the default behavior of AKS clusters.

When user creates a kubernetes public service object or an ingress controller, a new public IP address will be created and attached to the Load Balancer for ingress traffic.

kubectl expose deployment nginx --name nginx --port=80 --type LoadBalancer 
kubectl get svc
# NAME         TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)        AGE
# kubernetes   ClusterIP      10.0.0.1      <none>          443/TCP        10h
# nginx        LoadBalancer   10.0.106.59   20.31.208.171   80:31371/TCP   9s
20.31.208.171 is the new created public IP address attached to the Load Balancer.



To conclude, with Load Balancer mode, AKS uses one (default) single IP address for egress and one or more IP addresses for ingress traffic.

