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
