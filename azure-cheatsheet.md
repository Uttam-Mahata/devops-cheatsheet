# Azure CLI Cheatsheet

> A comprehensive reference for the Azure CLI (az) — from resource groups and compute to AKS, App Service, and storage.

---

## Table of Contents

- [Setup & Configuration](#setup--configuration)
- [Resource Groups & Subscriptions](#resource-groups--subscriptions)
- [IAM & RBAC](#iam--rbac)
- [Virtual Machines](#virtual-machines)
- [Azure Storage](#azure-storage)
- [Virtual Networks](#virtual-networks)
- [Azure SQL & Databases](#azure-sql--databases)
- [AKS](#aks)
- [Azure Container Registry (ACR)](#azure-container-registry-acr)
- [App Service](#app-service)
- [Azure Functions](#azure-functions)
- [Azure Key Vault](#azure-key-vault)
- [Monitor & Log Analytics](#monitor--log-analytics)
- [Advanced Scenarios](#advanced-scenarios)

---

## Setup & Configuration

```bash
# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Check version
az --version

# Login (opens browser)
az login

# Login with service principal
az login --service-principal \
  --username APP_ID \
  --password PASSWORD \
  --tenant TENANT_ID

# Login with device code (headless)
az login --use-device-code

# List subscriptions
az account list --output table

# Set active subscription
az account set --subscription "My Subscription"

# Show current account
az account show

# List available regions
az account list-locations --output table

# Enable CLI tab completion
az completion --shell bash >> ~/.bashrc
```

---

## Resource Groups & Subscriptions

```bash
# List resource groups
az group list --output table

# Create a resource group
az group create \
  --name my-rg \
  --location eastus

# Show resource group details
az group show --name my-rg

# List all resources in a group
az resource list --resource-group my-rg --output table

# Delete resource group (and all its resources)
az group delete --name my-rg --yes --no-wait

# Move resources between groups
az resource move \
  --destination-group new-rg \
  --ids /subscriptions/SUB_ID/resourceGroups/old-rg/providers/Microsoft.Compute/virtualMachines/my-vm

# Tag a resource group
az group update \
  --name my-rg \
  --tags env=production team=platform

# List resources by tag
az resource list --tag env=production --output table
```

---

## IAM & RBAC

```bash
# List role assignments in a resource group
az role assignment list \
  --resource-group my-rg \
  --output table

# List all built-in roles
az role definition list --output table

# Assign a role to a user
az role assignment create \
  --assignee user@example.com \
  --role "Contributor" \
  --resource-group my-rg

# Assign role to a service principal
az role assignment create \
  --assignee <SP_OBJECT_ID> \
  --role "Reader" \
  --scope /subscriptions/<SUB_ID>

# Remove a role assignment
az role assignment delete \
  --assignee user@example.com \
  --role "Contributor" \
  --resource-group my-rg

# Create a service principal
az ad sp create-for-rbac \
  --name my-sp \
  --role Contributor \
  --scopes /subscriptions/<SUB_ID>/resourceGroups/my-rg

# Create SP with JSON output (for CI/CD)
az ad sp create-for-rbac \
  --name my-sp \
  --role Contributor \
  --scopes /subscriptions/<SUB_ID> \
  --sdk-auth

# List service principals
az ad sp list --display-name my-sp

# Get service principal credentials
az ad sp credential list --id <SP_APP_ID>

# Reset SP credentials
az ad sp credential reset --id <SP_APP_ID>

# Create a managed identity
az identity create \
  --name my-identity \
  --resource-group my-rg

# List managed identities
az identity list --resource-group my-rg
```

---

## Virtual Machines

```bash
# List VMs
az vm list --output table
az vm list --resource-group my-rg --output table

# Create a VM
az vm create \
  --resource-group my-rg \
  --name web-vm \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name my-vnet \
  --subnet my-subnet \
  --public-ip-sku Standard

# Create with existing SSH key
az vm create \
  --resource-group my-rg \
  --name web-vm \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub

# Show VM details (includes IP)
az vm show \
  --resource-group my-rg \
  --name web-vm \
  --show-details --output table

# Get public IP of VM
az vm list-ip-addresses \
  --resource-group my-rg \
  --name web-vm \
  --output table

# SSH into VM
ssh azureuser@$(az vm list-ip-addresses \
  --resource-group my-rg --name web-vm \
  --query '[0].virtualMachine.network.publicIpAddresses[0].ipAddress' \
  --output tsv)

# Start / stop / restart / deallocate
az vm start  --resource-group my-rg --name web-vm
az vm stop   --resource-group my-rg --name web-vm
az vm restart --resource-group my-rg --name web-vm
az vm deallocate --resource-group my-rg --name web-vm   # stops billing for compute

# Resize VM
az vm resize \
  --resource-group my-rg \
  --name web-vm \
  --size Standard_D4s_v3

# List available sizes in a region
az vm list-sizes --location eastus --output table

# Open a port in NSG
az vm open-port \
  --resource-group my-rg \
  --name web-vm \
  --port 80

# Run a script on VM
az vm run-command invoke \
  --resource-group my-rg \
  --name web-vm \
  --command-id RunShellScript \
  --scripts "apt-get update && apt-get install -y nginx"

# Create VM snapshot
az snapshot create \
  --resource-group my-rg \
  --name vm-snapshot \
  --source /subscriptions/<SUB_ID>/resourceGroups/my-rg/providers/Microsoft.Compute/disks/web-vm_OsDisk

# Delete VM (leaves disks/NICs by default)
az vm delete --resource-group my-rg --name web-vm --yes
```

### VM Scale Sets

```bash
# Create scale set
az vmss create \
  --resource-group my-rg \
  --name my-vmss \
  --image Ubuntu2204 \
  --vm-sku Standard_B2s \
  --instance-count 3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --upgrade-policy-mode automatic

# Scale out/in
az vmss scale \
  --resource-group my-rg \
  --name my-vmss \
  --new-capacity 5

# Enable autoscale
az monitor autoscale create \
  --resource-group my-rg \
  --resource my-vmss \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-vmss \
  --min-count 2 --max-count 10 --count 3
```

---

## Azure Storage

```bash
# Create storage account
az storage account create \
  --name mystorageaccount \
  --resource-group my-rg \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2

# List storage accounts
az storage account list --output table

# Get connection string
az storage account show-connection-string \
  --name mystorageaccount \
  --resource-group my-rg \
  --output tsv

# Set connection string for CLI use
export AZURE_STORAGE_CONNECTION_STRING=$(az storage account show-connection-string \
  --name mystorageaccount --resource-group my-rg --output tsv)

# Create a blob container
az storage container create --name my-container

# List containers
az storage container list --output table

# Upload a file
az storage blob upload \
  --container-name my-container \
  --file ./app.zip \
  --name app.zip

# Upload a directory
az storage blob upload-batch \
  --source ./dist \
  --destination my-container \
  --destination-path app/

# Download a file
az storage blob download \
  --container-name my-container \
  --name app.zip \
  --file ./downloaded.zip

# List blobs
az storage blob list \
  --container-name my-container \
  --output table

# Generate SAS URL
az storage blob generate-sas \
  --container-name my-container \
  --name app.zip \
  --permissions r \
  --expiry 2024-12-31 \
  --output tsv

# Delete blob
az storage blob delete \
  --container-name my-container \
  --name app.zip

# Enable static website
az storage blob service-properties update \
  --account-name mystorageaccount \
  --static-website \
  --index-document index.html \
  --404-document error.html
```

---

## Virtual Networks

```bash
# Create VNet
az network vnet create \
  --resource-group my-rg \
  --name my-vnet \
  --address-prefix 10.0.0.0/16

# Add subnet
az network vnet subnet create \
  --resource-group my-rg \
  --vnet-name my-vnet \
  --name my-subnet \
  --address-prefixes 10.0.1.0/24

# List VNets
az network vnet list --resource-group my-rg --output table

# Create NSG
az network nsg create \
  --resource-group my-rg \
  --name my-nsg

# Add NSG rule (allow HTTP)
az network nsg rule create \
  --resource-group my-rg \
  --nsg-name my-nsg \
  --name allow-http \
  --priority 100 \
  --protocol Tcp \
  --destination-port-range 80 \
  --access Allow \
  --direction Inbound

# Associate NSG with subnet
az network vnet subnet update \
  --resource-group my-rg \
  --vnet-name my-vnet \
  --name my-subnet \
  --network-security-group my-nsg

# Create public IP
az network public-ip create \
  --resource-group my-rg \
  --name my-pip \
  --sku Standard \
  --allocation-method Static

# Create Load Balancer
az network lb create \
  --resource-group my-rg \
  --name my-lb \
  --sku Standard \
  --frontend-ip-name frontend \
  --backend-pool-name backend \
  --public-ip-address my-pip
```

---

## Azure SQL & Databases

### Azure SQL

```bash
# Create SQL server
az sql server create \
  --resource-group my-rg \
  --name my-sql-server \
  --location eastus \
  --admin-user sqladmin \
  --admin-password "P@ssw0rd123!"

# Create database
az sql db create \
  --resource-group my-rg \
  --server my-sql-server \
  --name mydb \
  --service-objective S2

# Allow Azure services to access SQL server
az sql server firewall-rule create \
  --resource-group my-rg \
  --server my-sql-server \
  --name AllowAzure \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Allow your IP
az sql server firewall-rule create \
  --resource-group my-rg \
  --server my-sql-server \
  --name AllowMyIP \
  --start-ip-address MY.IP.ADD.RESS \
  --end-ip-address MY.IP.ADD.RESS

# List databases
az sql db list --resource-group my-rg --server my-sql-server --output table

# Get connection string
az sql db show-connection-string \
  --server my-sql-server \
  --name mydb \
  --client ado.net
```

### Cosmos DB

```bash
# Create Cosmos DB account
az cosmosdb create \
  --resource-group my-rg \
  --name my-cosmos \
  --kind GlobalDocumentDB \
  --locations regionName=eastus failoverPriority=0

# Create database and container
az cosmosdb sql database create \
  --account-name my-cosmos \
  --resource-group my-rg \
  --name mydb

az cosmosdb sql container create \
  --account-name my-cosmos \
  --resource-group my-rg \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path /id
```

---

## AKS

```bash
# Create AKS cluster
az aks create \
  --resource-group my-rg \
  --name my-aks \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --enable-managed-identity

# Get kubectl credentials
az aks get-credentials \
  --resource-group my-rg \
  --name my-aks

# List clusters
az aks list --output table

# Show cluster details
az aks show --resource-group my-rg --name my-aks --output table

# Scale node pool
az aks scale \
  --resource-group my-rg \
  --name my-aks \
  --node-count 5 \
  --nodepool-name nodepool1

# Enable autoscaler
az aks update \
  --resource-group my-rg \
  --name my-aks \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 10

# Add a node pool
az aks nodepool add \
  --resource-group my-rg \
  --cluster-name my-aks \
  --name gpupool \
  --node-vm-size Standard_NC6 \
  --node-count 1

# Upgrade cluster
az aks upgrade \
  --resource-group my-rg \
  --name my-aks \
  --kubernetes-version 1.29.0

# Get available upgrades
az aks get-upgrades \
  --resource-group my-rg \
  --name my-aks --output table

# Stop cluster (save costs)
az aks stop --resource-group my-rg --name my-aks

# Start cluster
az aks start --resource-group my-rg --name my-aks

# Delete cluster
az aks delete --resource-group my-rg --name my-aks --yes
```

---

## Azure Container Registry (ACR)

```bash
# Create registry
az acr create \
  --resource-group my-rg \
  --name myregistry \
  --sku Standard \
  --admin-enabled true

# Login to ACR
az acr login --name myregistry

# Get ACR login server
az acr show --name myregistry --query loginServer --output tsv

# Tag and push image
docker tag myapp:latest myregistry.azurecr.io/myapp:latest
docker push myregistry.azurecr.io/myapp:latest

# List images
az acr repository list --name myregistry --output table

# List tags for an image
az acr repository show-tags \
  --name myregistry \
  --repository myapp --output table

# Build image in ACR (no local Docker needed)
az acr build \
  --registry myregistry \
  --image myapp:latest .

# Delete image
az acr repository delete \
  --name myregistry \
  --image myapp:old-tag --yes

# Attach ACR to AKS
az aks update \
  --resource-group my-rg \
  --name my-aks \
  --attach-acr myregistry
```

---

## App Service

```bash
# Create App Service plan
az appservice plan create \
  --resource-group my-rg \
  --name my-plan \
  --sku B2 \
  --is-linux

# Create web app (Node.js)
az webapp create \
  --resource-group my-rg \
  --plan my-plan \
  --name my-unique-app \
  --runtime "NODE:20-lts"

# Create web app from container
az webapp create \
  --resource-group my-rg \
  --plan my-plan \
  --name my-unique-app \
  --deployment-container-image-name myregistry.azurecr.io/myapp:latest

# List web apps
az webapp list --output table

# Deploy from ZIP
az webapp deployment source config-zip \
  --resource-group my-rg \
  --name my-unique-app \
  --src app.zip

# Deploy from local git
az webapp deployment source config-local-git \
  --resource-group my-rg \
  --name my-unique-app

# Set environment variables
az webapp config appsettings set \
  --resource-group my-rg \
  --name my-unique-app \
  --settings NODE_ENV=production DB_URL="postgres://..."

# Stream live logs
az webapp log tail \
  --resource-group my-rg \
  --name my-unique-app

# Restart web app
az webapp restart \
  --resource-group my-rg \
  --name my-unique-app

# Create deployment slot
az webapp deployment slot create \
  --resource-group my-rg \
  --name my-unique-app \
  --slot staging

# Swap slots (staging → production)
az webapp deployment slot swap \
  --resource-group my-rg \
  --name my-unique-app \
  --slot staging \
  --target-slot production

# Scale up App Service plan
az appservice plan update \
  --resource-group my-rg \
  --name my-plan \
  --sku P2V3
```

---

## Azure Functions

```bash
# Create function app (consumption plan)
az functionapp create \
  --resource-group my-rg \
  --consumption-plan-location eastus \
  --runtime node \
  --runtime-version 20 \
  --functions-version 4 \
  --name my-function-app \
  --storage-account mystorageaccount \
  --os-type Linux

# Deploy function code
func azure functionapp publish my-function-app

# List function apps
az functionapp list --output table

# List functions in app
az functionapp function list \
  --resource-group my-rg \
  --name my-function-app --output table

# Set app settings
az functionapp config appsettings set \
  --resource-group my-rg \
  --name my-function-app \
  --settings COSMOS_DB_URL="..." NODE_ENV=production

# Stream logs
az webapp log tail \
  --resource-group my-rg \
  --name my-function-app

# Restart
az functionapp restart \
  --resource-group my-rg \
  --name my-function-app

# Delete
az functionapp delete \
  --resource-group my-rg \
  --name my-function-app
```

---

## Azure Key Vault

```bash
# Create Key Vault
az keyvault create \
  --name my-keyvault \
  --resource-group my-rg \
  --location eastus \
  --enable-rbac-authorization false

# Set a secret
az keyvault secret set \
  --vault-name my-keyvault \
  --name db-password \
  --value "s3cr3tP@ss"

# Get a secret
az keyvault secret show \
  --vault-name my-keyvault \
  --name db-password \
  --query value --output tsv

# List secrets
az keyvault secret list \
  --vault-name my-keyvault --output table

# Grant access to a service principal
az keyvault set-policy \
  --name my-keyvault \
  --spn <SP_APP_ID> \
  --secret-permissions get list

# Grant access to a managed identity
az keyvault set-policy \
  --name my-keyvault \
  --object-id <IDENTITY_PRINCIPAL_ID> \
  --secret-permissions get list

# Create/import a certificate
az keyvault certificate create \
  --vault-name my-keyvault \
  --name my-cert \
  --policy "$(az keyvault certificate get-default-policy)"

# Delete secret
az keyvault secret delete \
  --vault-name my-keyvault \
  --name db-password

# Soft-delete recovery
az keyvault secret recover \
  --vault-name my-keyvault \
  --name db-password
```

---

## Monitor & Log Analytics

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group my-rg \
  --workspace-name my-workspace

# List workspaces
az monitor log-analytics workspace list --output table

# Query logs (KQL)
az monitor log-analytics query \
  --workspace /subscriptions/<SUB>/resourceGroups/my-rg/providers/Microsoft.OperationalInsights/workspaces/my-workspace \
  --analytics-query "AzureActivity | where OperationName has 'delete' | limit 20"

# Create metric alert
az monitor metrics alert create \
  --resource-group my-rg \
  --name high-cpu-alert \
  --resource /subscriptions/<SUB>/resourceGroups/my-rg/providers/Microsoft.Compute/virtualMachines/web-vm \
  --metric "Percentage CPU" \
  --operator GreaterThan \
  --threshold 80 \
  --evaluation-frequency 5m \
  --window-size 15m \
  --severity 2 \
  --action /subscriptions/<SUB>/resourceGroups/my-rg/providers/microsoft.insights/actionGroups/my-ag

# Create action group
az monitor action-group create \
  --resource-group my-rg \
  --name my-ag \
  --short-name myag \
  --email-receiver name=admin address=admin@example.com

# Create activity log alert (audit trail)
az monitor activity-log alert create \
  --resource-group my-rg \
  --name delete-rg-alert \
  --condition category=Administrative and operationName=Microsoft.Resources/resourceGroups/delete \
  --action-group my-ag
```

---

## Advanced Scenarios

### Scenario 1 — Blue/Green deployment with App Service slots

```bash
# Deploy to staging slot
az webapp deployment source config-zip \
  --resource-group my-rg \
  --name my-app \
  --slot staging \
  --src app.zip

# Warm up & test staging, then swap
az webapp deployment slot swap \
  --resource-group my-rg \
  --name my-app \
  --slot staging \
  --target-slot production
```

### Scenario 2 — Pull ACR image in AKS without a secret

```bash
# Attach ACR — sets up managed identity pull permission automatically
az aks update \
  --resource-group my-rg \
  --name my-aks \
  --attach-acr myregistry
```

### Scenario 3 — Enable diagnostic logs on a VM and send to Log Analytics

```bash
az monitor diagnostic-settings create \
  --name vm-diag \
  --resource /subscriptions/<SUB>/resourceGroups/my-rg/providers/Microsoft.Compute/virtualMachines/web-vm \
  --workspace my-workspace \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

### Scenario 4 — Lock a resource group to prevent accidental deletion

```bash
az lock create \
  --name no-delete \
  --resource-group my-rg \
  --lock-type CanNotDelete

# List locks
az lock list --resource-group my-rg

# Remove lock
az lock delete --name no-delete --resource-group my-rg
```

### Scenario 5 — Export ARM template from existing resource group

```bash
az group export --name my-rg > arm-template.json
```

---

## Quick Reference Card

| Task                              | Command                                                         |
|-----------------------------------|-----------------------------------------------------------------|
| Login                             | `az login`                                                      |
| Set subscription                  | `az account set --subscription "Name"`                          |
| Create resource group             | `az group create --name rg --location eastus`                   |
| Create VM                         | `az vm create --resource-group rg --name vm --image Ubuntu2204` |
| Get AKS credentials               | `az aks get-credentials --resource-group rg --name aks`         |
| ACR login                         | `az acr login --name myregistry`                                |
| Stream App Service logs           | `az webapp log tail --resource-group rg --name app`             |
| Get Key Vault secret              | `az keyvault secret show --vault-name kv --name secret`         |
| Swap deployment slots             | `az webapp deployment slot swap --slot staging --target-slot production` |
| Lock resource group               | `az lock create --name lock --resource-group rg --lock-type CanNotDelete` |

---

> Use managed identities instead of service principal key files wherever possible.
