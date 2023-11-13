## Securing Cluster secrets using secret store CSI Volume
    Kubernetes already have secrets bit it is encrypted at rest, while in transit, it can be decoded using base64.
So We prefer to store secrets, externally secret store provider like Hashicord vault, Azure key vault.

To achieve this we can use tools like "secret store CSI driver". This projects aims for securely access to sensitive data from the pods. So it will mount a volume to pod and that volume will expose sensitive data.
In the the backend it will require a specific provider like Azure keyvault.

## Secrete Store CSI Driver architecture

```bash
                         ______________    ______________            _______________________        /\
        schedule        |             |   |              |          | node-driver-register |       /  \
|pod|  -----------------|--->|API|----|-->| kubelet -----|--------->| secrets-store        |----->/    \ ------->|Azure keyvault|
                        |             |   |              |          | livenes-probe        |      \    /
                        |             |   |              |          |                      |       \__/
                        |_____________|   |______________|          |______________________|     secrets store
                         k8s master        node                     Secrets store CSI driver     csi provider
```

## secret store CSI in AKS - authentication options

1. service principle (SPN)
    works for any kubernetes cluster, but SPN credentials stored in k8s
2. System managed identity attached in VMSS (nodepool)
    Uses one single identity that will be used by all pods and all nodes
3. User managed identity attached in vmss (node pool)
    Identity available to all pods and all nodes
4. User managed identity attached to vmss with pod identity
    pod identity interceps all to IMDS / identity
5. workload identity with service account
    work for any kubernetes cluster, no credentials, native to kubernetes

Demo:-
1. Create an AKS cluster with Secret Store CSI driiver and Workload Identity
```bash
$AKS_RG="rg-aks-demo"
$AKS_NAME="aks-cluster"

az group create -n $AKS_RG -l westeurope

az aks create -g $AKS_RG -n $AKS_NAME --enable-addons azure-keyvault-secrets-provider --enable-oidc-issuer --enable-workload-identity

az aks get-credentials --name $AKS_NAME -g $AKS_RG --overwrite-existing


```

2. Verify connection to the cluster
```bash

kubectl get nodes
$AKS_OIDC_ISSUER=$(az aks show -n $AKS_NAME -g $AKS_RG --query "oidcIssuerProfile.issuerUrl" -otsv)
echo $AKS_OIDC_ISSUER
kubectl get pods -n kube-system -l 'app in (secrets-store-csi-driver, secrets-store-provider-azure)'
az aks show -n $AKS_NAME -g $AKS_RG --query addonProfiles.azureKeyvaultSecretsProvider.identity.clientId -o tsv

```

3. Create Keyvault resource with RBAC mode and assign RBAC role for admin

```bash
$AKV_NAME="akv4aks4app079"

az keyvault create -n $AKV_NAME -g $AKS_RG --enable-rbac-authorization

$AKV_ID=$(az keyvault show -n $AKV_NAME -g $AKS_RG --query id -o tsv)
echo $AKV_ID

$CURRENT_USER_ID=$(az ad signed-in-user show --query id -o tsv)
echo $CURRENT_USER_ID

az role assignment create --assignee $CURRENT_USER_ID `
        --role "Key Vault Administrator" `
        --scope $AKV_ID
```
4. Create keyvault secret

```bash
$AKV_SECRET_NAME="MySecretPassword"
az keyvault secret set --vault-name $AKV_NAME --name $AKV_SECRET_NAME --value "P@ssw0rd123!"

```
5. Create user managed identity resource

```bash
$IDENTITY_NAME="user-identity-aks-4-akv"
az identity create -g $AKS_RG -n $IDENTITY_NAME

$IDENTITY_ID=$(az identity show -g $AKS_RG -n $IDENTITY_NAME --query "id" -o tsv)
echo $IDENTITY_ID

$IDENTITY_CLIENT_ID=$(az identity show -g $AKS_RG -n $IDENTITY_NAME --query "clientId" -o tsv)
echo $IDENTITY_CLIENT_ID
```


6. Assign RBAC role to user managed identity for Keyvault's secret

```bash

$AKV_ID=$(az keyvault show -n $AKV_NAME -g $AKS_RG --query id -o tsv)
echo $AKV_ID

az role assignment create --assignee $IDENTITY_CLIENT_ID `
        --role "Key Vault Secrets User" `
        --scope $AKV_ID

sleep 60

```

7. Create service account for user managed identity

```bash
$NAMESPACE_APP="app-07" # can be changed to namespace of your workload

kubectl create namespace $NAMESPACE_APP


$SERVICE_ACCOUNT_NAME="workload-identity-sa"

@"
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: $IDENTITY_CLIENT_ID
  labels:
    azure.workload.identity/use: "true"
  name: $SERVICE_ACCOUNT_NAME
"@ > service-account.yaml

kubectl apply -f service-account.yaml --namespace $NAMESPACE_APP

```

8. Configure identity federation

```bash
$FEDERATED_IDENTITY_NAME="aks-federated-identity-app"

az identity federated-credential create -n $FEDERATED_IDENTITY_NAME `
            -g $AKS_RG `
            --identity-name $IDENTITY_NAME `
            --issuer $AKS_OIDC_ISSUER `
            --subject system:serviceaccount:${NAMESPACE_APP}:${SERVICE_ACCOUNT_NAME}

```

9. Configure secret provider class to get secret from Keyvault and to use user managed identity

```bash
$TENANT_ID=$(az account list --query "[?isDefault].tenantId" -o tsv)
echo $TENANT_ID

$SECRET_PROVIDER_CLASS="akv-spc-app"

@"
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: $SECRET_PROVIDER_CLASS # needs to be unique per namespace
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: "${IDENTITY_CLIENT_ID}"  # Setting this to use workload identity
    keyvaultName: ${AKV_NAME}         # Set to the name of your key vault
    cloudName: "AzurePublicCloud"
    objects:  |
      array:
        - |
          objectName: $AKV_SECRET_NAME
          objectType: secret  # object types: secret, key, or cert
          objectVersion: ""   # [OPTIONAL] object versions, default to latest if empty
    tenantId: "${TENANT_ID}"  # The tenant ID of the key vault
"@ > secretProviderClass.yaml

kubectl apply -f secretProviderClass.yaml -n $NAMESPACE_APP

kubectl get secretProviderClass -n $NAMESPACE_APP


```

10. Test access to Secret in keyvault with sample deployment

```bash
@"
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-deploy
  name: nginx-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-deploy
  template:
    metadata:
      labels:
        app: nginx-deploy
    spec:
      serviceAccountName: $SERVICE_ACCOUNT_NAME
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: secrets-store-inline
          mountPath: "/mnt/secrets-store"
          readOnly: true
      volumes:
        - name: secrets-store-inline
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: $SECRET_PROVIDER_CLASS
"@ > nginx-pod.yaml

kubectl apply -f nginx-pod.yaml -n $NAMESPACE_APP

kubectl get pods -n $NAMESPACE_APP

$POD_NAME=$(kubectl get pod -l app=nginx-deploy -o jsonpath="{.items[0].metadata.name}" -n $NAMESPACE_APP)
echo $POD_NAME

```

And finally, here we can see the password

```bash
kubectl exec -it $POD_NAME -n $NAMESPACE_APP -- ls /mnt/secrets-store


kubectl exec -it $POD_NAME -n $NAMESPACE_APP -- cat /mnt/secrets-store/$AKV_SECRET_NAME

```
## Feature
1. The plugins can retrieve secrets, key and certificates from keyvault
2. Autorotation feature with poll intervell
3. Used also to retrieve TLS certificates for Ingress controller or app pods
4. Supports mounting multiple secrets store objects as a single volume
5. along with mounting secretas volume we can also set as env variables

## Best Practices

1. Use i managed identity per application
2. Deploy the driver and provider into kubesystem namespace or seperate dedicated namespace


reference:-

https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver
