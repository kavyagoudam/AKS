
# Azure Application Gateway for Containers
  1. It is a new Load balancing solution for exposing service within a AKS cluster, it is a evolution of existing AGIC.
  2. Existing AGIC only supports Ingress doesn't supports Gateway API


# Ingress API vs Gateway API

## Ingress API
```bash

| User | --> | LB | --> | Ingress | --> | service | --> | pods |
               |             |
        ( APP Gateway )  (Ingress Class)

```
## AppGateway API
```bash

| User | --> | LB | ----------> | Gateway | --> | http routes | --> | service | --> | pods |
               |                     |                          |
    (App GW for Containers)   (Gateway class)                   |-> | service | --> | pods |

```
# AGIC vs Application Gateway for containers
## AGIC

```bash
_______________________         
|            |pods| <-|--------|App Gateway| <---|User|
| |--> svc            |            |
| |--> Ingress        |            |
| |                   |            |
| |---> |AGIC| <------|--------|Azure ARM|
|_____________________|
```
## Application Gateway for containers
```bash
_______________________
| |-> svc      |pods|  |-------|App GW for containers| <--|user|
| |->Httproute         |         |
| |-> Gateway          |         | 
| |                    |         |
| |                    |         |
| |-> |ALB conroller|<-|---------|
|______________________|
```

Let's focus on Application Gateway for containers.
  Ingress resources are used to expose the services in kubernetes, it is supported by AGIC. But Ingress resources have some limitation over RBAC. To over come the limitation, Gateway Api is introduced.

What is Gateway API?
1. An open source project managed by the SIG-network community
   https://github.com/kubernetes-sigs/gateway-api
2. An API(collectionof resources) that model service networking in kubernetes
3. these resources are GatewayClass, Gateway, HTTPRoute, TCPRoute, Service etc
4. Aim to evolve kubernetes service networking through expressive, extensible and role-orianted interfaces that are implemented by many vendors and have broad industry support
5. it is new Application load balancing(layer 7) and dynamic traffic management for AKS, it is a new offering under app gateway product family
6. 
refer architecture view diagram for the API gateway.


What is ALB controller?
1. ALB is a kubernetes deployment installed via Helm chart
2. Creates the App Gatewayfor container if you use BYO mode or managed mode
3. When applicationLoadBalancer resource is deleted App gateway for containers will be deleted.
4. Supports Workload identity and managed identity
5. Watched for CRDs like httpRoute, Ingress, Gateway and apploication load balancer and propogates configuration to the app gateway for containers.

When we install the ALB controller via helm chart, a new namespace(azure-alb-system) and 2 pods will created.
1. alb-controller pod propogates configuration to application gateway
2. alb-controller-bootstrap pod is responsible for management of CRD's

## application gateway for containers associations
Defines a connection point into a virtual network 
1:1 mapping of an association resources to a delegated to a subnet
IT should have it's own subnet with /24 cidr range

## Benefits:

1. traffice splitting / weighted round robin
2. Mutual authentication(mTls) to the back end target
3. kubernetes supports for ingress and gateway API
4. better RBAC model for seperation of concerns
5. near real time updates to add or move pods, routes and probes





