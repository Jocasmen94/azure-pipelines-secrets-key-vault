# azure-pipelines-secrets-key-vault

Demoing the use of Azure Key Vault Secrets within Azure DevOps Pipelines.

## 1) Create Azure Key Vault with Secrets and persmissions

The following script will create a Key Vault and Secret:

```bash
# create the variables
KEYVAULT_RG="rg-keyvault-devops"
KEYVAULT_NAME="keyvault019"
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# create new resource group
az group create -n rg-keyvault-devops -l westeurope

# create key vault with RBAC option (not Access Policy)
az keyvault create --name $KEYVAULT_NAME \
   --resource-group $KEYVAULT_RG \
   --enable-rbac-authorization
```

```bash
# assign RBAC role to the current user to manage secrets
USER_ID=$(az ad signed-in-user show --query objectId -o tsv)

KEYVAULT_ID=$(az keyvault show --name $KEYVAULT_NAME \
   --resource-group $KEYVAULT_RG \
   --query id \
   --output tsv)

az role assignment create --role "Key Vault Secrets Officer" \
   --scope $KEYVAULT_ID \
   --assignee-object-id $USER_ID
``` 

```bash
# create a secret
az keyvault secret set --name "DatabasePassword" \
  --value "mySecretPassword" \
  --vault-name $KEYVAULT_NAME
```

## 2) Create Service Principal to access Key Vault from Azure DevOps Pipelines

```bash
# create a service principal
SPN=$(az ad sp create-for-rbac -n "spn-keyvault-devops")

echo $SPN | jq .

SPN_APPID=$(echo $SPN | jq .appId)

SPN_ID=$(az ad sp list --display-name "spn-keyvault-devops" --query [0].objectId --out tsv)
<!-- SPN_ID=$(az ad sp show --id $SPN_APPID --query objectId --out tsv) -->

# assign RBAC role to the service principal
az role assignment create --role "Key Vault Secrets User" \
   --scope $KEYVAULT_ID \
   --assignee-object-id $SPN_ID
```

## 3) Create a pipeline to access Key Vault Secrets

3.1) Create Service Connection using the SPN

3.2) Create YAML pipeline