# Play.Infra

Play Economy Infrastructure components.

## Add the Github package source

```powershell
$owner="icodedotnetmicroservices"
$gh_pat="[PAT HERE]"

dotnet nuget add source --username USERNAME --password $gh_pat --store-password-in-clear-text --name github "https://nuget.pkg.github.com/$owner/index.json"
```

## Creating the Azure resource group

```powershell
$appname = "playeconomy"
az group create --name $appname --location eastus
```

## Creating the Cosmos DB account

```powershell
$cosmosdbname = "playeconomymicroservices"
az cosmosdb create --name $cosmosdbname --resource-group $appname --kind MongoDB --enable-free-tier
```

## Creating the Service Bus namespace

```powershell
$servicebusname = "playeconomymicroservices"
az servicebus namespace create --name $servicebusname --resource-group $appname --sku Standard
```

## Creating the Container Registry

```powershell
$containerregisteryname = "acrplayeconomy"
az acr create --name $containerregisteryname --resource-group $appname --sku Basic
```

## Creating the AKS Cluster

```powershell
$aksclustername = "aksclusterplayeconomy"
az feature register --name EnablePodIdentityPreview --namespace Microsoft.ContainerService
az extension add --name aks-preview

az aks create -n $aksclustername -g $appname --node-vm-size DS2_v2 --location centralus --node-count 2 --attach-acr $containerregisteryname --enable-pod-identity --network-plugin azure

az aks get-credentials --name $aksclustername --resource-group $appname
```

## Creating The Azure Key Vault

```powershell
$azurekeyvaultname = "azkeyvaultplayeconomy"
az keyvault create -n $azurekeyvaultname -g $appname
```

## Installing Emissary-ingress

```powershell
$namespace= "emissary"
helm repo add datawire https://app.getambassador.io
helm repo update

kubectl apply -f https://app.getambassador.io/yaml/emissary/2.2.2/emissary-crds.yaml
kubectl wait --timeout=90s --for=condition=available deployment emissary-apiext -n emissary-system

 helm install emissary-ingress datawire/emissary-ingress  --set service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=$appname -n $namespace --create-namespace

 kubectl rollout status  deployment/emissary-ingress  -n $namespace -w

```

## Configuring Emissary-ingress routing

```powershell
kubectl apply -f .\emissary-ingress\listener.yaml -n $namespace
kubectl apply -f .\emissary-ingress\mappings.yaml -n $namespace
```

## Installing cert-manager

```powershell
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager --version v1.8.0  --set installCRDs=true --namespace $namespace
```

## Creating the Cluster Issuer

```powershell
kubectl apply -f .\cert-manager\cluster-issuer.yaml -n $namespace
kubectl apply -f .\cert-manager\acme-challenge.yaml -n $namespace
```

## Creating the tls certificate

```powershell
kubectl apply -f .\emissary-ingress\tls-certificate.yaml -n $namespace
```

## Enabling TLS and HTTPS

```powershell
kubectl apply -f .\emissary-ingress\host.yaml -n $namespace
```

## Packaging and publishing the microservice Helm chart

```powershell
helm package .\helm\microservice

$helmUser=[guid]::Empty.Guid
$helmPassword= az acr login --name acr$appname --expose-token --output tsv --query accessToken

$env:HEML_EXPERIMENTAL_OCI=1
helm registry login "acr$appname.azurecr.io" --username $helmUser --password $helmPassword

helm push microservice-0.1.0.tgz oci://acr$appname.azurecr.io/helm
```
## Create Github service principal
```powershell
$appId = az ad sp create-for-rbac -n "GitHub" --skip-assignment --query appId --output tsv

az role assignment create --assignee $appId --role "AcrPush" --resource-group $appname
az role assignment create --assignee $appId --role "Azure Kubernetes Service Cluster User Role" --resource-group $appname
az role assignment create --assignee $appId --role "Azure Kubernetes Service Contributor Role" --resource-group $appname
```