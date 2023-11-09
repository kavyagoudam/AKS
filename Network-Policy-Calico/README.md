## Restrict Egress/Outbound Layer 3 traffic using Calico Network Policy

1. with Azure Firewall, All pods can access the external service, To avoid this, Network Policy can be implemented to ingress and egress traffic.
2. Cillium and calico plugins can be used to implement the network Policy
3. Istio and linkerd can be other options as well



1. Create demo environment
```bash

$RG_NAME = "rg-aks-cluster-calico"
$AKS_NAME = "aks-cluster-calico"

# create an azure rsource group
az group create -n $RG_NAME --location westeurope

# create AKS cluster with Calico enabled
az aks create -g $RG_NAME -n $AKS_NAME --network-plugin azure --network-policy calico

```


2. Get AKS credentials

```
az aks get-credentials -g $RG_NAME -n $AKS_NAME --overwrite-existing

```

3. Deploy sample online service, just to get public IP
```bash

$EXTERNAL_IP=(az container create -g $RG_NAME -n aci-app --image nginx:latest --ports 80  --ip-address public --query ipAddress.ip --output tsv)
$EXTERNAL_IP
# 4.175.23.231
```

4. Deploy nginx pod and access to $EXTERNAL_IP from the pod
```bash

kubectl run nginx --image=nginx
kubectl exec -it nginx -- curl http://$EXTERNAL_IP
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# <style>

```

## Restrict all ingress and egress traffic

Deny all ingress and egress traffic using the following network policy.

```bash

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

```

Deploy network policy to deny all traffic.

```bash

kubectl apply -f deny_all.yaml

```
Verify egress traffic is denied
```bash
kubectl exec -it nginx -- curl http://$EXTERNAL_IP --max-time 5
# curl: (28) Connection timed out after 5000 milliseconds
# command terminated with exit code 28
```

## Allow egress traffic to external service IP address

Allow Nginx pod egress traffic to external service IP address by using a network policy like the following.

```bash

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-ip
spec:
  podSelector:
    matchLabels:
      run: nginx
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 4.175.23.231/32

```
Make sure to update pod labels and destination IP address or CIDR range.

Get labels of Nginx pod.

```bash
kubectl get pods --show-labels

```

Deploy the allow egress policy.

```bash
kubectl apply -f allow_egress_ip.yaml
```

Access the external service from nginx pod

```bash
kubectl exec -it nginx -- curl http://$EXTERNAL_IP
# <title>Welcome to nginx!</title>
```

##  Verify that egress traffic to external IP is blocked to other pods

```bash
kubectl run nginx1 --image=nginx
kubectl exec -it nginx1 -- curl http://$EXTERNAL_IP --max-time 5
# curl: (28) Connection timed out after 5001 milliseconds
# command terminated with exit code 28

```
Delete the allow-egress-ip network policy

```bash
kubectl delete -f allow_egress_ip.yaml

```
## Logging denied traffic

Calico enables logging denied and allowed traffic to syslog. you need to use the Calico Network Policy instead of Kubernetes Network Policy. you need to install Calico API Server to deploy Calico Network Policy using kubectl instead of calicoctl. Src: https://docs.tigera.io/calico/latest/operations/install-apiserver

```bash

kubectl apply -f calico_api_server.yaml
kubectl get tigerastatus apiserver
# NAME        AVAILABLE   PROGRESSING   DEGRADED   SINCE
# apiserver   True        False         False      6s

```
Deploy Calico Network Policy that denies egress traffic and logs it to syslog.
Once the Calico API server has been installed, you can use kubectl to interact with the Calico APIs.

deploy calico network policy to enable logging.

```bash
kubectl apply -f logging_traffic.yaml

```


Enable AKS monitoring and syslog features as logged traffic will be sent to syslog.

```bash
az aks enable-addons -a monitoring --enable-syslog -g $RG_NAME -n $AKS_NAME
```

Test logging with nginx pod.

```bash
kubectl exec -it nginx1 -- curl http://$EXTERNAL_IP --max-time 5
```
Check generated logs in Log Analytics by running this KQL query:

```bash
Syslog 
| project TimeGenerated, SyslogMessage
| where SyslogMessage has "<EXTERNAL_IP>"
// | where SyslogMessage has "calico-packet"

```



