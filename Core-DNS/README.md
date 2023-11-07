Custom Domain names using kubernetes CoreDNS

Introduction:-
    The main objective to setup custom Domain in Kubernetes clusters. means to replace default.svc.kubernetes.local to custome domain like mydomain.com
    Core DNS is a opensource tool available on GitHub. which is written in go. Many plugins are available for multiple clouds.

Service discovery with CoreDNS.
    core DNS integrates with kubernetes via kubernetes plugin or etcd with etcd plugin.
    core-dns is already installed in k8s by default.

kubectl get pods,svc,configmaps -n kube-system -l=k8s-app=kube-dns 

NAME                           READY   STATUS    RESTARTS   AGE
pod/coredns-76b9877f49-5xw45   1/1     Running   0          13m
pod/coredns-76b9877f49-zf6d9   1/1     Running   0          13m

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
service/kube-dns   ClusterIP   10.0.0.10    <none>        53/UDP,53/TCP   136m

NAME                       DATA   AGE
configmap/coredns          1      136m
configmap/coredns-custom   1      136m

ConfigMaps

Core -DNS uses configmap to get configuration details. there are two configmaps coredns uses.
1. coredns -- basic configmaps
2. coredns-Custom -- available to user to be able to customize this DNS configuration


How to configure Cusom domain name using core-dns and test it

1. create a basic deployment and service using nginx image

kubectl create deployment nginx --image=nginx --replicas=3

deployment.apps/nginx created

kubectl expose deployment nginx --name nginx --port=80

service/nginx exposed

2. Now, create and deploy the custom domain name resolvable inside the kubernetes

kubectl apply -f custom_coredns.yaml 

Warning: resource configmaps/coredns-custom is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
configmap/coredns-custom configured

3. Once the configmap is deployed. delete the coredns pods using below command.

kubectl delete pod -n  kube-system -l k8s-app=kube-dns

4. once the new pods are created in coredns. curl to the nginx pod with custom domain name

kubectl exec -it <nginx pod name>  -- curl http://nginx.default.aks.com

