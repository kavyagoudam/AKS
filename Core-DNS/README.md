markdown
Copy code
# Custom Domain Names using Kubernetes CoreDNS

## Introduction

The main objective is to set up a custom domain in Kubernetes clusters, replacing the default.svc.kubernetes.local with a custom domain like mydomain.com. CoreDNS is an open-source tool available on GitHub, written in Go, with many plugins available for various cloud platforms.

## Service Discovery with CoreDNS

CoreDNS integrates with Kubernetes via the Kubernetes plugin or with etcd using the etcd plugin. CoreDNS is already installed in Kubernetes by default. You can verify its status using the following command:
```bash
kubectl get pods,svc,configmaps -n kube-system -l=k8s-app=kube-dns
The output will display CoreDNS pods and services:

NAME                           READY   STATUS    RESTARTS   AGE
pod/coredns-76b9877f49-5xw45   1/1     Running   0          13m
pod/coredns-76b9877f49-zf6d9   1/1     Running   0          13m

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
service/kube-dns   ClusterIP   10.0.0.10    <none>        53/UDP,53/TCP   136m

NAME                       DATA   AGE
configmap/coredns          1      136m
configmap/coredns-custom   1      136m
```
## ConfigMaps
CoreDNS uses ConfigMaps to get configuration details. There are two ConfigMaps that CoreDNS uses:

coredns: Basic configuration.
coredns-custom: Custom DNS configuration available for users to customize.
## How to Configure Custom Domain Name using CoreDNS and Test it

Create a basic deployment and service using the Nginx image:

```bash
kubectl create deployment nginx --image=nginx --replicas=3
kubectl expose deployment nginx --name nginx --port=80
```

Now, create and deploy the custom domain name for resolution inside Kubernetes:

``` bash
kubectl apply -f custom_coredns.yaml
```
Note: You might encounter a warning about missing annotations, but it can be ignored in this context.

Once the coredns-custom ConfigMap is deployed, delete the CoreDNS pods using the following command:

```bash
kubectl delete pod -n kube-system -l k8s-app=kube-dns
```
After the new CoreDNS pods are created, you can test the custom domain by using curl to access the Nginx service with the custom domain name:

``` bash

kubectl exec -it <nginx-pod-name> -- curl http://nginx.default.aks.com
```
Replace <nginx-pod-name> with the actual name of one of your Nginx pods.
