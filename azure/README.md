# Azure CLI Cheatsheet

The Azure CLI (`az`) is Microsoft's cross-platform command-line tool for managing Azure resources. It provides a consistent interface across all Azure services and is available on Linux, macOS, and Windows. Commands follow the pattern `az <group> <subgroup> <command> [options]`. The Azure CLI is the preferred automation tool for Azure deployments, CI/CD pipelines, and operational tasks, complementing the Azure Portal and Azure PowerShell.

---

## Table of Contents

1. [Installation & Login](#installation--login)
2. [Resource Groups & Subscriptions](#resource-groups--subscriptions)
3. [IAM & RBAC](#iam--rbac)
4. [Virtual Machines](#virtual-machines)
5. [Virtual Networks](#virtual-networks)
6. [Storage](#storage)
7. [AKS](#aks)
8. [ACR](#acr)
9. [App Service](#app-service)
10. [Azure Functions](#azure-functions)
11. [Key Vault](#key-vault)
12. [Azure SQL & Cosmos DB](#azure-sql--cosmos-db)
13. [Monitor & Alerts](#monitor--alerts)
14. [Common Patterns & Tips](#common-patterns--tips)

---

## Installation & Login

### Install Azure CLI

Installs the Azure CLI on Ubuntu/Debian via the Microsoft package repository.

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az version
```

For macOS:
```bash
brew update && brew install azure-cli
```

### `az login`

Authenticates interactively via browser-based OAuth2 device flow. Stores the token in `~/.azure/`.

**Key flags:**
- `--use-device-code` — print a device code and URL for headless/remote environments
- `--tenant` — authenticate to a specific Azure AD tenant
- `--service-principal` — authenticate as a service principal (for CI/CD)

```bash
az login
az login --use-device-code
az login --tenant "mytenant.onmicrosoft.com"
```

### `az login` with Service Principal

Authenticate non-interactively using a service principal — the standard approach for automation.

**Key flags:**
- `--service-principal` — flag to indicate SP login
- `-u` — application (client) ID
- `-p` — password/secret or certificate path
- `--tenant` — Azure AD tenant ID

```bash
az login \
  --service-principal \
  -u $ARM_CLIENT_ID \
  -p $ARM_CLIENT_SECRET \
  --tenant $ARM_TENANT_ID
```

### `az account show`

Displays the currently active subscription including its ID, tenant, and user.

```bash
az account show
az account show --output table
```

### `az account list`

Lists all Azure subscriptions the current identity has access to.

**Key flags:**
- `--all` — include disabled subscriptions
- `--refresh` — re-fetch from ARM rather than using cached data

```bash
az account list --output table
az account list --all --query "[].{Name:name, ID:id, State:state}" --output table
```

### `az account set`

Switches the active subscription context.

**Key flags:**
- `--subscription` — subscription ID or name

```bash
az account set --subscription "My Production Subscription"
az account set --subscription "00000000-0000-0000-0000-000000000000"
```

---

## Resource Groups & Subscriptions

### `az group create`

Creates a new resource group — the logical container for Azure resources in a region.

**Key flags:**
- `--name` — resource group name (required)
- `--location` — Azure region (e.g. `eastus`, `westeurope`) (required)
- `--tags` — key=value metadata tags

```bash
az group create \
  --name myapp-prod-rg \
  --location eastus \
  --tags Environment=production Team=platform Project=myapp
```

### `az group list`

Lists all resource groups in the active subscription.

```bash
az group list --output table
az group list --query "[?location=='eastus'].name" --output tsv
```

### `az group delete`

Deletes a resource group and all resources within it. Irreversible.

**Key flags:**
- `--name` — resource group name
- `--yes` — skip confirmation prompt
- `--no-wait` — return immediately without waiting for deletion

```bash
az group delete --name myapp-staging-rg --yes --no-wait
```

### `az group show`

Displays details of a specific resource group.

```bash
az group show --name myapp-prod-rg
```

### `az resource list`

Lists all resources in a subscription or resource group.

**Key flags:**
- `--resource-group` — scope to a specific group
- `--resource-type` — filter by resource type (e.g. `Microsoft.Compute/virtualMachines`)
- `--tag` — filter by tag (e.g. `Environment=production`)

```bash
az resource list --resource-group myapp-prod-rg --output table
az resource list --tag Environment=production --output table
```

---

## IAM & RBAC

### `az role assignment create`

Grants an Azure RBAC role to a user, group, or service principal at a specified scope.

**Key flags:**
- `--assignee` — object ID, user principal name, or service principal name
- `--role` — built-in or custom role name or ID
- `--scope` — ARM resource path (subscription, resource group, or resource)

```bash
# Assign Contributor to a service principal on a resource group
az role assignment create \
  --assignee "00000000-0000-0000-0000-000000000000" \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/myapp-prod-rg"

# Assign Reader at the subscription level
az role assignment create \
  --assignee "alice@example.com" \
  --role "Reader" \
  --scope "/subscriptions/$SUBSCRIPTION_ID"
```

### `az role assignment list`

Lists all RBAC assignments at a given scope.

**Key flags:**
- `--scope` — ARM scope path
- `--assignee` — filter by a specific user/SP
- `--all` — include inherited role assignments from parent scopes

```bash
az role assignment list \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/myapp-prod-rg" \
  --output table

az role assignment list --assignee "alice@example.com" --all --output table
```

### `az role assignment delete`

Removes a role assignment.

```bash
az role assignment delete \
  --assignee "alice@example.com" \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/myapp-prod-rg"
```

### `az ad sp create-for-rbac`

Creates a service principal and optionally assigns an RBAC role — the one-stop command for CI/CD credential setup.

**Key flags:**
- `--name` — display name for the SP
- `--role` — RBAC role to assign
- `--scopes` — scope(s) to assign the role on
- `--years` — credential validity in years (default 1)
- `--sdk-auth` — output in Azure SDK credential format (useful for GitHub Actions)

```bash
az ad sp create-for-rbac \
  --name "github-actions-deploy" \
  --role "Contributor" \
  --scopes "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/myapp-prod-rg" \
  --sdk-auth
```

### `az role definition list`

Lists available built-in and custom roles.

```bash
az role definition list --output table --query "[?roleType=='BuiltInRole'].{Name:roleName, Description:description}" | head -40
```

---

## Virtual Machines

### `az vm list`

Lists all VMs in the active subscription or a specific resource group.

**Key flags:**
- `--resource-group` — scope to a resource group
- `--show-details` — include power state and public IPs
- `--output` — `table`, `json`, `tsv`

```bash
az vm list --resource-group myapp-prod-rg --show-details --output table
```

### `az vm create`

Creates a new virtual machine with all required supporting resources.

**Key flags:**
- `--resource-group` — target resource group (required)
- `--name` — VM name (required)
- `--image` — OS image (e.g. `Ubuntu2204`, `Win2022Datacenter`, `RHEL`)
- `--size` — VM SKU (e.g. `Standard_B2s`, `Standard_D4s_v5`)
- `--admin-username` — admin account name
- `--ssh-key-values` — path to SSH public key file (Linux)
- `--authentication-type` — `ssh` or `password`
- `--vnet-name` / `--subnet` — place VM in existing VNet
- `--public-ip-sku` — `Standard` or `Basic`; use `""` for no public IP
- `--nsg` — attach an existing NSG

```bash
az vm create \
  --resource-group myapp-prod-rg \
  --name web-server-01 \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --ssh-key-values ~/.ssh/id_rsa.pub \
  --vnet-name myapp-vnet \
  --subnet web-subnet \
  --public-ip-sku Standard
```

### `az vm start` / `az vm stop` / `az vm deallocate`

Starts, stops (OS shutdown, billed), or deallocates (stops billing) a VM.

```bash
az vm start --resource-group myapp-prod-rg --name web-server-01
az vm stop --resource-group myapp-prod-rg --name web-server-01
az vm deallocate --resource-group myapp-prod-rg --name web-server-01
```

### `az vm show`

Returns detailed metadata about a VM, including hardware profile, OS profile, and network interfaces.

```bash
az vm show \
  --resource-group myapp-prod-rg \
  --name web-server-01 \
  --show-details
```

### `az vm open-port`

Adds an inbound NSG rule to allow traffic on a specific port. Quick for dev/test — use NSG rules for production.

```bash
az vm open-port --resource-group myapp-prod-rg --name web-server-01 --port 443
```

### `az vm run-command invoke`

Runs a shell script or command on a VM without needing SSH access — uses the VM agent.

**Key flags:**
- `--command-id` — `RunShellScript` (Linux) or `RunPowerShellScript` (Windows)
- `--scripts` — script content to execute

```bash
az vm run-command invoke \
  --resource-group myapp-prod-rg \
  --name web-server-01 \
  --command-id RunShellScript \
  --scripts "sudo apt-get update && sudo apt-get install -y nginx"
```

---

## Virtual Networks

### `az network vnet create`

Creates a virtual network with an optional initial subnet.

**Key flags:**
- `--resource-group` — target resource group
- `--name` — VNet name
- `--address-prefixes` — IPv4 CIDR block(s)
- `--subnet-name` / `--subnet-prefixes` — create an initial subnet
- `--location` — Azure region

```bash
az network vnet create \
  --resource-group myapp-prod-rg \
  --name myapp-vnet \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name web-subnet \
  --subnet-prefixes 10.0.1.0/24
```

### `az network vnet subnet create`

Adds a subnet to an existing VNet.

```bash
az network vnet subnet create \
  --resource-group myapp-prod-rg \
  --vnet-name myapp-vnet \
  --name db-subnet \
  --address-prefixes 10.0.2.0/24
```

### `az network nsg create`

Creates a Network Security Group (NSG) — a stateful firewall.

```bash
az network nsg create \
  --resource-group myapp-prod-rg \
  --name web-nsg
```

### `az network nsg rule create`

Adds an inbound or outbound security rule to an NSG.

**Key flags:**
- `--nsg-name` — target NSG
- `--priority` — rule priority (100–4096, lower = higher priority)
- `--direction` — `Inbound` or `Outbound`
- `--access` — `Allow` or `Deny`
- `--protocol` — `Tcp`, `Udp`, `*`
- `--destination-port-ranges` — port(s) or range (e.g. `443` or `8000-9000`)
- `--source-address-prefixes` — source CIDR(s) or service tags

```bash
az network nsg rule create \
  --resource-group myapp-prod-rg \
  --nsg-name web-nsg \
  --name allow-https \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443 \
  --source-address-prefixes Internet
```

### `az network public-ip create`

Allocates a public IP address resource.

```bash
az network public-ip create \
  --resource-group myapp-prod-rg \
  --name myapp-pip \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3
```

---

## Storage

### `az storage account create`

Creates a new Azure Storage account.

**Key flags:**
- `--name` — globally unique storage account name (3–24 chars, lowercase alphanumeric)
- `--resource-group` — target resource group
- `--location` — Azure region
- `--sku` — `Standard_LRS`, `Standard_GRS`, `Standard_ZRS`, `Premium_LRS`
- `--kind` — `StorageV2` (general-purpose v2, recommended), `BlobStorage`
- `--access-tier` — `Hot` or `Cool`
- `--https-only` — enforce HTTPS (strongly recommended)
- `--min-tls-version` — minimum TLS version (`TLS1_2` recommended)

```bash
az storage account create \
  --name myappprodstorage \
  --resource-group myapp-prod-rg \
  --location eastus \
  --sku Standard_ZRS \
  --kind StorageV2 \
  --https-only true \
  --min-tls-version TLS1_2
```

### `az storage container create`

Creates a blob container within a storage account.

**Key flags:**
- `--name` — container name
- `--account-name` — storage account
- `--public-access` — `off` (default), `blob`, or `container`

```bash
az storage container create \
  --name uploads \
  --account-name myappprodstorage \
  --public-access off
```

### `az storage blob upload`

Uploads a file to a blob container.

**Key flags:**
- `--file` — local file path
- `--container-name` — target container
- `--name` — blob name (path) in the container
- `--account-name` — storage account
- `--overwrite` — overwrite if exists

```bash
az storage blob upload \
  --account-name myappprodstorage \
  --container-name uploads \
  --name reports/2024/january.csv \
  --file ./january.csv \
  --overwrite
```

### `az storage blob list`

Lists blobs in a container.

**Key flags:**
- `--prefix` — filter by blob name prefix

```bash
az storage blob list \
  --account-name myappprodstorage \
  --container-name uploads \
  --prefix reports/2024/ \
  --output table
```

### `az storage blob download`

Downloads a blob to a local file.

```bash
az storage blob download \
  --account-name myappprodstorage \
  --container-name uploads \
  --name reports/2024/january.csv \
  --file ./january-downloaded.csv
```

---

## AKS

### `az aks create`

Creates a new Azure Kubernetes Service cluster.

**Key flags:**
- `--resource-group` — resource group (required)
- `--name` — cluster name (required)
- `--node-count` — initial node count
- `--node-vm-size` — VM SKU for nodes (e.g. `Standard_D4s_v5`)
- `--enable-addons` — comma-separated list: `monitoring`, `ingress-appgw`, etc.
- `--enable-managed-identity` — use managed identity instead of SP (recommended)
- `--network-plugin` — `azure` (CNI) or `kubenet`
- `--zones` — availability zones for nodes
- `--kubernetes-version` — e.g. `1.29`

```bash
az aks create \
  --resource-group myapp-prod-rg \
  --name prod-cluster \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --enable-managed-identity \
  --enable-addons monitoring \
  --network-plugin azure \
  --zones 1 2 3 \
  --kubernetes-version 1.29
```

### `az aks get-credentials`

Merges the cluster's kubeconfig into `~/.kube/config`, enabling `kubectl` access.

**Key flags:**
- `--admin` — get admin credentials (bypasses Azure AD RBAC — for break-glass only)
- `--overwrite-existing` — replace an existing context with the same name

```bash
az aks get-credentials \
  --resource-group myapp-prod-rg \
  --name prod-cluster \
  --overwrite-existing
kubectl get nodes
```

### `az aks list`

Lists all AKS clusters in the subscription or a resource group.

```bash
az aks list --resource-group myapp-prod-rg \
  --query "[].{Name:name, K8sVersion:kubernetesVersion, NodeCount:agentPoolProfiles[0].count, State:provisioningState}" \
  --output table
```

### `az aks nodepool add`

Adds a new node pool to an existing AKS cluster.

**Key flags:**
- `--cluster-name` — target AKS cluster
- `--name` — node pool name
- `--node-count` / `--min-count` / `--max-count` — count and autoscaler bounds
- `--enable-cluster-autoscaler` — enable the cluster autoscaler for this pool
- `--node-vm-size` — VM SKU
- `--node-taints` — Kubernetes taints (e.g. `sku=gpu:NoSchedule`)

```bash
az aks nodepool add \
  --resource-group myapp-prod-rg \
  --cluster-name prod-cluster \
  --name gpupool \
  --node-count 1 \
  --min-count 0 \
  --max-count 4 \
  --enable-cluster-autoscaler \
  --node-vm-size Standard_NC6s_v3 \
  --node-taints sku=gpu:NoSchedule
```

### `az aks upgrade`

Upgrades the Kubernetes version of the control plane and/or node pools.

```bash
az aks upgrade \
  --resource-group myapp-prod-rg \
  --name prod-cluster \
  --kubernetes-version 1.30
```

---

## ACR

### `az acr create`

Creates a new Azure Container Registry.

**Key flags:**
- `--resource-group` — resource group
- `--name` — globally unique registry name
- `--sku` — `Basic`, `Standard`, `Premium` (Premium supports geo-replication, private link)
- `--admin-enabled` — enable the admin user account (not recommended for production)

```bash
az acr create \
  --resource-group myapp-prod-rg \
  --name myappregistry \
  --sku Standard
```

### `az acr login`

Authenticates Docker to push/pull from the registry. Uses your `az login` credentials.

```bash
az acr login --name myappregistry
```

### `az acr build`

Builds a container image in the cloud using ACR Tasks — no local Docker required.

**Key flags:**
- `--registry` — target ACR name
- `--image` — image name and tag (e.g. `myapp:v1.2`)
- `--file` — Dockerfile path (default `Dockerfile` in source directory)

```bash
az acr build \
  --registry myappregistry \
  --image myapp:v1.2 \
  --file ./Dockerfile \
  .
```

### `az acr repository list`

Lists all repositories in the registry.

```bash
az acr repository list --name myappregistry --output table
```

### `az acr repository show-tags`

Lists all tags for a specific repository.

```bash
az acr repository show-tags --name myappregistry --repository myapp --output table
```

---

## App Service

### `az appservice plan create`

Creates an App Service plan that defines the region, compute capacity, and pricing tier.

**Key flags:**
- `--resource-group` — resource group
- `--name` — plan name
- `--sku` — pricing tier: `FREE`, `SHARED`, `B1`, `B2`, `S1`, `P1v3`, etc.
- `--is-linux` — use Linux workers (required for container-based apps)
- `--number-of-workers` — number of instances

```bash
az appservice plan create \
  --resource-group myapp-prod-rg \
  --name myapp-plan \
  --sku P1v3 \
  --is-linux \
  --number-of-workers 2
```

### `az webapp create`

Creates a new Web App on an App Service plan.

**Key flags:**
- `--plan` — App Service plan name
- `--runtime` — runtime stack (e.g. `NODE:20-lts`, `PYTHON:3.12`, `DOTNETCORE:8.0`)
- `--deployment-container-image-name` — Docker image for container-based apps

```bash
az webapp create \
  --resource-group myapp-prod-rg \
  --plan myapp-plan \
  --name myapp-webapp \
  --runtime "NODE:20-lts"
```

### `az webapp config appsettings set`

Sets application settings (environment variables) on a Web App.

```bash
az webapp config appsettings set \
  --resource-group myapp-prod-rg \
  --name myapp-webapp \
  --settings \
    NODE_ENV=production \
    DATABASE_URL="@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/db-url/)" \
    LOG_LEVEL=info
```

### `az webapp deployment source config-zip`

Deploys a ZIP package to a Web App — simple and effective for most app types.

**Key flags:**
- `--src` — local path to the ZIP file

```bash
zip -r app.zip . --exclude "node_modules/*" ".git/*"
az webapp deployment source config-zip \
  --resource-group myapp-prod-rg \
  --name myapp-webapp \
  --src ./app.zip
```

### `az webapp log tail`

Streams live application and HTTP logs from a Web App.

```bash
az webapp log tail \
  --resource-group myapp-prod-rg \
  --name myapp-webapp
```

---

## Azure Functions

### `az functionapp create`

Creates a new Azure Functions app with the required storage account and hosting plan.

**Key flags:**
- `--resource-group` — resource group
- `--consumption-plan-location` — region for Consumption plan (scales to zero)
- `--runtime` — runtime: `python`, `node`, `dotnet`, `java`, `powershell`
- `--runtime-version` — e.g. `3.11` for Python
- `--storage-account` — storage account for function state and code
- `--functions-version` — Azure Functions host version (e.g. `4`)
- `--os-type` — `Linux` or `Windows`

```bash
az functionapp create \
  --resource-group myapp-prod-rg \
  --consumption-plan-location eastus \
  --runtime python \
  --runtime-version 3.11 \
  --functions-version 4 \
  --name myapp-functions \
  --storage-account myappprodstorage \
  --os-type Linux
```

### `az functionapp config appsettings set`

Sets environment variables and connection strings for the Function App.

```bash
az functionapp config appsettings set \
  --resource-group myapp-prod-rg \
  --name myapp-functions \
  --settings \
    ENVIRONMENT=production \
    SERVICE_BUS_CONNECTION="Endpoint=sb://..."
```

### `az functionapp list`

Lists all Function Apps in the subscription or a resource group.

```bash
az functionapp list --resource-group myapp-prod-rg \
  --query "[].{Name:name, State:state, Location:location}" \
  --output table
```

### `az functionapp show`

Displays details of a Function App including its host name and state.

```bash
az functionapp show \
  --resource-group myapp-prod-rg \
  --name myapp-functions
```

---

## Key Vault

### `az keyvault create`

Creates a new Azure Key Vault.

**Key flags:**
- `--resource-group` — resource group
- `--name` — globally unique vault name
- `--location` — Azure region
- `--sku` — `standard` or `premium` (premium adds HSM-backed keys)
- `--enable-soft-delete` — protect against accidental deletion (on by default)
- `--retention-days` — soft-delete retention (7–90 days)

```bash
az keyvault create \
  --resource-group myapp-prod-rg \
  --name myapp-keyvault \
  --location eastus \
  --sku standard \
  --retention-days 90
```

### `az keyvault secret set`

Creates or updates a secret in Key Vault.

**Key flags:**
- `--vault-name` — target vault
- `--name` — secret name
- `--value` — secret value (avoid passing sensitive values via CLI args; prefer `--file`)
- `--file` — read secret value from a file

```bash
az keyvault secret set \
  --vault-name myapp-keyvault \
  --name db-password \
  --value "SuperSecretPassword123!"

# More secure: read from a file
az keyvault secret set \
  --vault-name myapp-keyvault \
  --name db-password \
  --file ./db-password.txt
```

### `az keyvault secret show`

Retrieves the value of a secret from Key Vault.

**Key flags:**
- `--vault-name` — target vault
- `--name` — secret name
- `--version` — specific version ID (default is current)

```bash
az keyvault secret show \
  --vault-name myapp-keyvault \
  --name db-password \
  --query value --output tsv
```

### `az keyvault secret list`

Lists all secrets in a vault (names only, not values).

```bash
az keyvault secret list \
  --vault-name myapp-keyvault \
  --output table
```

### `az keyvault set-policy`

Grants an identity permissions to access Key Vault secrets, keys, or certificates.

**Key flags:**
- `--object-id` — Azure AD object ID of the user/SP/managed identity
- `--secret-permissions` — e.g. `get list set delete`
- `--key-permissions` — e.g. `get list sign verify`

```bash
# Grant a managed identity access to read secrets
az keyvault set-policy \
  --name myapp-keyvault \
  --object-id "$(az webapp identity show --name myapp-webapp --resource-group myapp-prod-rg --query principalId -o tsv)" \
  --secret-permissions get list
```

---

## Azure SQL & Cosmos DB

### `az sql server create`

Creates an Azure SQL logical server (the container for SQL databases).

**Key flags:**
- `--resource-group` — resource group
- `--name` — globally unique server name
- `--location` — Azure region
- `--admin-user` — server admin username
- `--admin-password` — server admin password

```bash
az sql server create \
  --resource-group myapp-prod-rg \
  --name myapp-sql-server \
  --location eastus \
  --admin-user sqladmin \
  --admin-password "SecurePassword123!"
```

### `az sql db create`

Creates a database on a SQL logical server.

**Key flags:**
- `--server` — logical server name
- `--name` — database name
- `--service-objective` — compute tier (e.g. `S1`, `GP_Gen5_4`, `HS_Gen5_8`)
- `--zone-redundant` — enable zone redundancy for HA

```bash
az sql db create \
  --resource-group myapp-prod-rg \
  --server myapp-sql-server \
  --name myapp-db \
  --service-objective GP_Gen5_4 \
  --zone-redundant
```

### `az sql server firewall-rule create`

Opens a firewall rule to allow connections to the SQL server from a CIDR range.

```bash
# Allow Azure services
az sql server firewall-rule create \
  --resource-group myapp-prod-rg \
  --server myapp-sql-server \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Allow a specific IP
az sql server firewall-rule create \
  --resource-group myapp-prod-rg \
  --server myapp-sql-server \
  --name AllowMyOffice \
  --start-ip-address 203.0.113.10 \
  --end-ip-address 203.0.113.10
```

### `az cosmosdb create`

Creates a new Azure Cosmos DB account.

**Key flags:**
- `--name` — globally unique account name
- `--resource-group` — resource group
- `--locations` — region(s) for the account (format: `regionName=<name> failoverPriority=<n>`)
- `--kind` — `GlobalDocumentDB` (Core/SQL API), `MongoDB`, `Cassandra`, `Gremlin`, `Table`
- `--default-consistency-level` — `Session`, `Eventual`, `Strong`, `BoundedStaleness`

```bash
az cosmosdb create \
  --resource-group myapp-prod-rg \
  --name myapp-cosmos \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
  --kind GlobalDocumentDB \
  --default-consistency-level Session
```

---

## Monitor & Alerts

### `az monitor metrics list`

Retrieves metric time-series data for an Azure resource.

**Key flags:**
- `--resource` — resource ID
- `--metric` — metric name (e.g. `Percentage CPU`)
- `--interval` — time grain (e.g. `PT5M` = 5 minutes, `PT1H` = 1 hour)
- `--start-time` / `--end-time` — ISO 8601 time range

```bash
az monitor metrics list \
  --resource "/subscriptions/$SUB_ID/resourceGroups/myapp-prod-rg/providers/Microsoft.Compute/virtualMachines/web-server-01" \
  --metric "Percentage CPU" \
  --interval PT5M \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --output table
```

### `az monitor alert create` (classic)

Creates a metric alert rule on a resource.

**Key flags:**
- `--name` — alert rule name
- `--resource-group` — resource group
- `--target` — resource ID of the monitored resource
- `--condition` — metric condition expression
- `--action` — action group ID to notify

```bash
az monitor metrics alert create \
  --name high-cpu-alert \
  --resource-group myapp-prod-rg \
  --scopes "/subscriptions/$SUB_ID/resourceGroups/myapp-prod-rg/providers/Microsoft.Compute/virtualMachines/web-server-01" \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action "/subscriptions/$SUB_ID/resourceGroups/myapp-prod-rg/providers/microsoft.insights/actionGroups/ops-team"
```

### `az monitor action-group create`

Creates an action group that defines who/how to notify when an alert fires.

**Key flags:**
- `--action` — notification action (e.g. `email`, `sms`, `webhook`, `azurefunction`)

```bash
az monitor action-group create \
  --resource-group myapp-prod-rg \
  --name ops-team \
  --action email ops-email alice@example.com
```

### `az monitor log-analytics workspace create`

Creates a Log Analytics workspace for centralised log collection.

```bash
az monitor log-analytics workspace create \
  --resource-group myapp-prod-rg \
  --workspace-name myapp-logs \
  --location eastus \
  --sku PerGB2018 \
  --retention-time 90
```

### `az monitor log-analytics query`

Runs a KQL (Kusto Query Language) query against a Log Analytics workspace.

**Key flags:**
- `--workspace` — workspace resource ID or name
- `--analytics-query` — KQL query string
- `--timespan` — query time range (ISO 8601 duration, e.g. `PT1H` = last 1 hour)

```bash
az monitor log-analytics query \
  --workspace myapp-logs \
  --analytics-query "AppRequests | where ResultCode >= 500 | summarize count() by bin(TimeGenerated, 5m)" \
  --timespan PT1H \
  --output table
```

---

## Common Patterns & Tips

1. **Set a default resource group and location to avoid repeating them in every command.**
   ```bash
   az configure --defaults group=myapp-prod-rg location=eastus
   ```
   These defaults are persisted in `~/.azure/config` and apply to all commands that accept `--resource-group` and `--location`.

2. **Use `--query` with JMESPath to extract exactly the fields you need.**
   ```bash
   # Get the primary connection string for a storage account
   az storage account show-connection-string \
     --name myappprodstorage \
     --query connectionString --output tsv
   ```
   The `--output tsv` format is perfect for assigning results to shell variables.

3. **Prefer Managed Identities over service principals with secrets for Azure-hosted workloads.**
   Managed identities are automatically rotated by Azure and eliminate the need to manage client secrets. Enable one with:
   ```bash
   az webapp identity assign --name myapp-webapp --resource-group myapp-prod-rg
   ```
   Then grant it Key Vault or storage access via RBAC rather than storing credentials in app settings.

4. **Use `--no-wait` for long-running operations in scripts, then poll with `az <resource> wait`.**
   ```bash
   az aks create --no-wait ...
   az aks wait --name prod-cluster --resource-group myapp-prod-rg --created
   ```
   This pattern frees your terminal while the resource provisions.

5. **Store frequently used resource IDs in shell variables to avoid repeated lookups.**
   ```bash
   VAULT_ID=$(az keyvault show --name myapp-keyvault --query id -o tsv)
   WEBAPP_IDENTITY=$(az webapp identity show --name myapp-webapp --resource-group myapp-prod-rg --query principalId -o tsv)
   ```

6. **Use `az deployment group what-if` before applying ARM/Bicep templates.**
   ```bash
   az deployment group what-if \
     --resource-group myapp-prod-rg \
     --template-file main.bicep \
     --parameters @params.json
   ```
   This shows exactly which resources will be created, modified, or deleted — similar to `terraform plan`.

7. **Use `az acr build` instead of local Docker builds in CI pipelines.**
   ACR Tasks run the build in Azure, so you don't need Docker installed on your CI runner. The built image is immediately available in ACR without an extra push step.

8. **Reference Key Vault secrets directly in App Service settings using the `@Microsoft.KeyVault()` syntax.**
   ```bash
   az webapp config appsettings set \
     --name myapp-webapp \
     --resource-group myapp-prod-rg \
     --settings DB_PASSWORD="@Microsoft.KeyVault(SecretUri=https://myapp-keyvault.vault.azure.net/secrets/db-password/)"
   ```
   This keeps secrets out of your deployment pipeline and auto-refreshes when the secret rotates.

9. **Use `--output table` for readability when exploring resources interactively.**
   ```bash
   az vm list --show-details --output table
   az aks list --output table
   ```
   Switch to `--output json` when piping to `jq` for complex transformations.

10. **Tag all resources consistently at creation time and enforce it with Azure Policy.**
    A minimal tagging standard enables cost tracking, automated cleanup, and access scoping:
    ```bash
    az group create --name myapp-prod-rg --location eastus \
      --tags Environment=production Team=platform Project=myapp Owner=alice@example.com
    ```
    Tags propagate to cost reports in Azure Cost Management, making per-team chargeback straightforward.
