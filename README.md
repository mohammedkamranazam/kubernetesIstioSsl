# Kubernetes, Istio, SSL - Setup

## Login into Azure

```bash
az login --use-device-login
```

## Set Azure Subscription Context

```bash
az account set --subscription xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

## Create ServicePrincipal

```bash
az ad sp create-for-rbac --role="Contributor"
```

## Create ResourceGroup

```bash
az group create --location southIndia --name MyResourceGroup
```

## Create AKS Using ServicePrincipal

```bash
az aks create --resource-group MyResourceGroup --name myK8SName --node-count 2 --kubernetes-version 1.15.7 --node-vm-size "Standard_B2ms" --generate-ssh-keys --service-principal xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx --client-secret xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

## Create ContainerRegistry

```bash
az acr create --resource-group MyResourceGroup --name myACRName --sku Standard
```

## Get ContainerRegistry Id

```bash
az acr show --resource-group MyResourceGroup --name myACRName --query "id" --output tsv

Output:
"id": "/subscriptions/<subscriptionId>/resourceGroups/<resourceGroup>/providers/Microsoft.ContainerRegistry/registries/<containerRegistryName>
```

## Assign ServicePrincipal Access to ContainerRegistry

### --assignee is ServicePrincipal Id

```bash
az role assignment create --assignee xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx --scope /subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/MyResourceGroup/providers/Microsoft.ContainerRegistry/registries/myACRName --role Owner
```

## Select The Kubernetes Cluster/Fetch Credentials to Administor

```bash
az aks get-credentials --resource-group MyResourceGroup --name myK8SName
```

## Get Kubernetes Current Context

```bash
kubectl config get-contexts
```

## Create Namespace on Kubernetes

```bash
kubectl create namespace myNamespace
```

## Set Kubernetes Context to Namespace

```bash
kubectl config set-context --current --namespace=myNamespace
```

## Verify Kubernetes Current Context

```bash
kubectl config get-contexts
```

## Install Istio

```bash
version istio-1.3.0
```

## Change cmd path to Istio Install Directory

```bash
kubectl apply -f istio-demo-auth.yaml
```

## Istio Side Car Injection

### Check Injection on Namespace

Check if injection is enabled on the namespace or not.

```bash
kubectl get namespace -L istio-injection
```

### Enable Injection on Namespace

```bash
kubectl label namespace myNamespace istio-injection=enabled
```

## A) Run All Yaml Files in Step Wise Sequence

```bash
kubectl apply -f istio-cluster-rbac-config.yaml
kubectl apply -f istio-auth-policy.yaml
```

## Get Kubernetes Secrets

```bash
kubectl get secrets -n istio-system
```

## Delete Certificate Secret if Already Exists

```bash
kubectl delete secret istio-ingressgateway-certs -n istio-system
```

## Extract Certificates from PFX File

```bash
openssl pkcs12 -in cert.pfx -out certificate.txt -nodes
```

After extracting remove all bag properties from certificate.txt. Put domain certificate on top, then intermediate certificate and then root certificate at the bottom.

Make sure all certificates are one after the another without any lines or bag properties in between.

Format example as shown below: if not followed then ssl will not work on firefox

```bash
-----BEGIN CERTIFICATE-----
domain certificate
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
intermediate certificate
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
root certificate
-----END CERTIFICATE-----
```

## Apply The Certificates to Istio

```bash
kubectl create -n istio-system secret tls istio-ingressgateway-certs --key private.key --cert public-chain.crt
```

## Check Certifcate Successfully Applied or Not

### Get Gateway Pod Name

```bash
kubectl -n istio-system get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}'
```

## Check Certifcate Exists or Not in The Pod Directory

```bash
kubectl exec -it -n istio-system istio-ingressgateway-xxxxxxxxxx-xxxxx -- ls -al /etc/istio/ingressgateway-certs
```

## Display The Certificate Contents in Console

Verify the format as mentioned before.

```bash
kubectl exec -i -n istio-system  istio-ingressgateway-xxxxxxxxxx-xxxxx  -- cat /etc/istio/ingressgateway-certs/tls.crt
```

## Check Microservice Pod TLS Through Istio Gateway

```bash
istioctl authn tls-check istio-ingressgateway-xxxxxxxxxx-xxxxx.istio-system myMicroservice-service.myNamespace.svc.cluster.local
```

## Create Azure Traffic Manager Profile

```bash
az network traffic-manager profile create --name myTrafficManagerProfileName --resource-group MyResourceGroup --routing-method Performance --path "/" --protocol HTTPS --unique-dns-name myDns --ttl 30 --port 443
```

## Add Azure Traffic Manager Endpoint IP to The Istio Ingress Gateway LoadBalancer

```bash
az network traffic-manager endpoint create --name EndpointName-K8S --profile-name myProfileName --resource-group MyResourceGroup --type externalEndpoints --endpoint-location southIndia --target 12.123.1.123
```

## B) Run All Yaml Files in Step Wise Sequence

kubectl apply -f istio-gateway.yaml
kubectl apply -f istio-gateway-svc.yaml
kubectl apply -f istio-service-role.yaml

## Install Docker for Desktop

### Switch to Linux Container

### Login to Container Registry

```bash
az acr login --name myACRName
```

## Build Docker Image

```bash
docker build -t myACRName.azurecr.io/myMicroserviceImageName:latest -f Dockerfile .
```

## Push Docker Image to Container Registry

```bash
docker push myACRName.azurecr.io/myMicroserviceImageName:latest
```

## Run Micro Service Yaml File

```bash
kubectl apply -f my-microservice-deployment-file.yaml
```

## Check Logs

### Get Ingress Gateway Pod Id

```bash
kubectl get pods -n istio-system
```

## Get Logs For Ingress Gateway or Micro Service Pod

```bash
kubectl logs istio-ingressgateway-xxxxxxxxx-xxxxx --all-containers=true -n istio-system --tail=15
```

```bash
kubectl logs myMicroservicePod-xxxxxxxxx-xxxxx --all-containers=true -n myNamespace --tail=15
```

## Istio Default Auth Policy

Apply this to reset the auth policies.

```bash
--kubectl apply -f istio-default-auth-policy.yaml
```
