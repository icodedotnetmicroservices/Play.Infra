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
