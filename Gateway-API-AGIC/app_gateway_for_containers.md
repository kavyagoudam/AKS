# Azure Application Gateway for Containers are new

```bash
_______________________         
|            |pods| <-|--------|App Gateway| <---|User|
| |--> svc            |            |
| |--> Ingress        |            |
| |                   |            |
| |---> |AGIC| <------|--------|Azure ARM|
|_____________________|
```