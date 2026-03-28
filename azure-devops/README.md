# Azure DevOps Cheatsheet

Azure DevOps is Microsoft's end-to-end DevOps platform offering hosted Git repositories, CI/CD pipelines, agile project boards, package feeds, and test management — all in a single integrated service. It can be used fully in the cloud (Azure DevOps Services) or on-premises (Azure DevOps Server). The `az devops` CLI extension lets you automate every aspect of the platform from your terminal.

---

## Installation & Setup

### Install the Azure CLI and DevOps Extension

The `az devops` commands are a CLI extension that ships separately from the core Azure CLI.

```bash
# Install the Azure CLI (Ubuntu/Debian)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Add the Azure DevOps extension
az extension add --name azure-devops

# Authenticate – opens a browser for interactive login
az login

# Or authenticate with a Personal Access Token (PAT)
export AZURE_DEVOPS_EXT_PAT=<your-pat>
echo $AZURE_DEVOPS_EXT_PAT | az devops login --organization https://dev.azure.com/MyOrg

# Set a default organisation and project so you don't repeat them every command
az devops configure --defaults organization=https://dev.azure.com/MyOrg project=MyProject
```

**Key options:**
| Flag | Description |
|------|-------------|
| `--organization` | Full URL of your Azure DevOps org |
| `--project` | Project name or ID |
| `--output` | Output format: `json`, `table`, `tsv`, `yaml` |
| `--query` | JMESPath query to filter output |

---

## Projects & Organizations

### List and Manage Projects

```bash
# List all projects in your organisation
az devops project list --output table

# Show details of a single project
az devops project show --project MyProject

# Create a new project (Git + Agile)
az devops project create \
  --name "NewProject" \
  --description "My new project" \
  --source-control git \
  --process Agile \
  --visibility private

# Delete a project (prompts for confirmation)
az devops project delete --id <project-id> --yes
```

### Teams

```bash
# List all teams in a project
az devops team list --output table

# Create a team
az devops team create --name "Platform Team" --description "Platform engineering team"

# Add a member to a team
az devops team member add \
  --team "Platform Team" \
  --members user@example.com
```

---

## Repos

### Repository Management

```bash
# List repositories
az repos list --output table

# Create a new repo
az repos create --name my-service

# Show repo details
az repos show --repository my-service

# Clone URL helper
az repos show --repository my-service --query remoteUrl -o tsv
```

### Branches

```bash
# List all branches in a repo
az repos ref list --repository my-service --filter heads --output table

# Create a branch from main
az repos ref create \
  --name refs/heads/feature/my-feature \
  --object-id $(az repos ref list --repository my-service \
      --filter heads/main --query "[0].objectId" -o tsv) \
  --repository my-service

# Delete a remote branch
az repos ref delete \
  --name refs/heads/feature/old-branch \
  --object-id <commit-sha> \
  --repository my-service
```

### Pull Requests

```bash
# Create a pull request
az repos pr create \
  --title "Add payment service" \
  --description "Implements the payment microservice" \
  --source-branch feature/payment \
  --target-branch main \
  --repository my-service \
  --auto-complete false \
  --squash false

# List open PRs
az repos pr list --status active --output table

# Show details of a specific PR
az repos pr show --id 42

# Approve a PR
az repos pr set-vote --id 42 --vote approve

# Complete (merge) a PR
az repos pr update --id 42 --status completed --squash true

# Add a reviewer
az repos pr reviewer add --id 42 --reviewers user@example.com
```

### Branch Policies

```bash
# Require a minimum number of reviewers on main
az repos policy approver-count create \
  --branch main \
  --is-blocking true \
  --minimum-approver-count 2 \
  --repository-id $(az repos show --repository my-service --query id -o tsv)

# Require successful build before merge
az repos policy build create \
  --branch main \
  --is-blocking true \
  --build-definition-id <pipeline-id> \
  --repository-id <repo-id> \
  --queue-on-source-update true

# Require comment resolution before merge
az repos policy comment-required create \
  --branch main \
  --is-blocking true \
  --repository-id <repo-id>
```

---

## Boards

### Work Items

```bash
# Create a User Story
az boards work-item create \
  --title "As a user I want to reset my password" \
  --type "User Story" \
  --assigned-to user@example.com \
  --iteration "MyProject\Sprint 1"

# Show a work item
az boards work-item show --id 1001

# Update a work item (change state)
az boards work-item update --id 1001 --state Active

# Add a comment to a work item
az boards work-item update --id 1001 \
  --discussion "Investigation complete, moving to development."

# Delete a work item
az boards work-item delete --id 1001 --yes

# Link two work items (parent-child)
az boards work-item relation add \
  --id 1001 \
  --relation-type "System.LinkTypes.Hierarchy-Forward" \
  --target-id 1002
```

### Queries

```bash
# Run a saved WIQL query
az boards query --id <query-guid>

# Execute an inline WIQL query
az boards query \
  --wiql "SELECT [System.Id],[System.Title],[System.State] \
          FROM WorkItems \
          WHERE [System.TeamProject] = 'MyProject' \
          AND [System.State] = 'Active' \
          ORDER BY [System.ChangedDate] DESC"
```

### Sprints (Iterations)

```bash
# List iterations (sprints)
az boards iteration project list --output table

# Create a new sprint
az boards iteration project create \
  --name "Sprint 5" \
  --start-date "2024-09-01" \
  --finish-date "2024-09-14"

# Assign an iteration to a team
az boards iteration team add \
  --team "Platform Team" \
  --id <iteration-id>
```

---

## Pipelines

### YAML Pipeline — Full Example

Azure Pipelines are defined in a `azure-pipelines.yml` file at the root of your repository.

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - release/*
  paths:
    exclude:
      - docs/**

pr:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest   # Microsoft-hosted agent

variables:
  buildConfiguration: Release
  imageTag: $(Build.BuildId)

stages:
  - stage: Build
    displayName: "Build & Test"
    jobs:
      - job: BuildJob
        steps:
          - task: UseDotNet@2
            inputs:
              version: "8.x"

          - script: dotnet restore
            displayName: "Restore NuGet packages"

          - script: dotnet build --configuration $(buildConfiguration)
            displayName: "Build"

          - script: dotnet test --no-build --logger trx
            displayName: "Unit Tests"

          - task: PublishTestResults@2
            inputs:
              testResultsFormat: VSTest
              testResultsFiles: "**/*.trx"

          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: "$(Build.ArtifactStagingDirectory)"
              artifactName: "drop"

  - stage: Deploy
    displayName: "Deploy to Staging"
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployWeb
        environment: staging
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: drop
                - script: echo "Deploying to staging..."
```

### Triggers

```yaml
# Schedule trigger – nightly at 02:00 UTC on weekdays
schedules:
  - cron: "0 2 * * 1-5"
    displayName: Nightly Build
    branches:
      include:
        - main
    always: true           # Run even if no code changes

# Pipeline completion trigger
resources:
  pipelines:
    - pipeline: upstream
      source: "MyOrg/MyProject/upstream-pipeline"
      trigger:
        branches:
          - main
```

### Stages, Jobs, and Steps

```yaml
stages:
  - stage: Validate
    jobs:
      - job: Lint
        steps:
          - script: npm run lint

      - job: SecurityScan
        dependsOn: []          # Run in parallel with Lint
        steps:
          - script: snyk test

  - stage: Package
    dependsOn: Validate
    jobs:
      - job: Docker
        steps:
          - task: Docker@2
            inputs:
              command: buildAndPush
              repository: myapp
              dockerfile: Dockerfile
              containerRegistry: my-acr-service-connection
              tags: |
                $(Build.BuildId)
                latest
```

### Templates

Templates allow reuse of steps, jobs, or stages across pipelines.

```yaml
# templates/build-steps.yml
parameters:
  - name: configuration
    type: string
    default: Release

steps:
  - script: dotnet build --configuration ${{ parameters.configuration }}
    displayName: "Build (${{ parameters.configuration }})"
  - script: dotnet test --no-build
    displayName: "Test"
```

```yaml
# azure-pipelines.yml  – consuming the template
stages:
  - stage: Build
    jobs:
      - job: BuildApp
        steps:
          - template: templates/build-steps.yml
            parameters:
              configuration: Release
```

### Variables

```yaml
variables:
  # Inline variable
  appName: myapp

  # Variable group (Library)
  - group: production-secrets

  # Conditionally set variable
  - ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}:
    - name: environment
      value: production
  - ${{ else }}:
    - name: environment
      value: staging
```

```bash
# Set a pipeline variable at runtime (output variable)
echo "##vso[task.setvariable variable=myVar;isOutput=true]hello"

# Set a secret variable at runtime
echo "##vso[task.setvariable variable=mySecret;isSecret=true]s3cr3t"
```

### Artifacts & Caching

```yaml
# Publish a pipeline artifact
- task: PublishPipelineArtifact@1
  inputs:
    targetPath: $(Build.ArtifactStagingDirectory)
    artifact: myArtifact
    publishLocation: pipeline

# Download in a later stage/job
- task: DownloadPipelineArtifact@2
  inputs:
    artifact: myArtifact
    path: $(Pipeline.Workspace)/myArtifact

# Cache node_modules between runs
- task: Cache@2
  inputs:
    key: 'npm | "$(Agent.OS)" | package-lock.json'
    restoreKeys: |
      npm | "$(Agent.OS)"
    path: $(npm_config_cache)
  displayName: Cache npm packages
```

### Pipeline CLI Commands

```bash
# List pipelines
az pipelines list --output table

# Create a pipeline from a YAML file in a repo
az pipelines create \
  --name "my-service-ci" \
  --repository my-service \
  --branch main \
  --yml-path azure-pipelines.yml

# Trigger a manual run
az pipelines run --name "my-service-ci" --branch main

# Show recent runs
az pipelines runs list --pipeline-name "my-service-ci" --output table

# Download artifact from a run
az pipelines runs artifact download \
  --artifact-name drop \
  --run-id <run-id> \
  --path ./artifacts
```

---

## Artifacts

Azure Artifacts hosts NuGet, npm, Maven, Python, and Universal packages in private or public feeds.

```bash
# List feeds
az artifacts universal list --feed MyFeed

# Create a feed
az artifacts feed create --name MyFeed

# Publish a Universal Package
az artifacts universal publish \
  --feed MyFeed \
  --name mypackage \
  --version 1.0.0 \
  --description "My package" \
  --path ./dist

# Download a Universal Package
az artifacts universal download \
  --feed MyFeed \
  --name mypackage \
  --version 1.0.0 \
  --path ./download
```

### Using npm with Azure Artifacts

```bash
# Authenticate npm to the feed
az artifacts generate-npmrc --registry https://pkgs.dev.azure.com/MyOrg/_packaging/MyFeed/npm/registry/

# Publish a package
npm publish --registry https://pkgs.dev.azure.com/MyOrg/_packaging/MyFeed/npm/registry/
```

---

## Environments & Approvals

Environments represent deployment targets (Kubernetes namespaces, VMs, or generic).

```bash
# List environments
az pipelines environment list --output table

# Create an environment
az pipelines environment create --name staging --description "Staging cluster"

# Add an approval check via the portal or REST API
# (CLI does not yet fully support approval configuration — use the UI or REST)
```

```yaml
# Reference an environment with a manual approval gate in YAML
jobs:
  - deployment: DeployProd
    environment: production    # must have approval check configured in Azure DevOps UI
    strategy:
      runOnce:
        deploy:
          steps:
            - script: ./deploy.sh prod
```

---

## Variable Groups & Key Vault

Variable groups let you share variables across multiple pipelines, and can be linked to an Azure Key Vault to pull secrets automatically.

```bash
# Create a variable group
az pipelines variable-group create \
  --name "production-config" \
  --variables API_URL=https://api.example.com LOG_LEVEL=info

# Add a secret variable to a group
az pipelines variable-group variable create \
  --group-id <id> \
  --name DB_PASSWORD \
  --value "s3cr3t" \
  --secret true

# Link a variable group to an Azure Key Vault
az pipelines variable-group create \
  --name "kv-secrets" \
  --authorize true \
  --key-vault MyKeyVault \
  --service-endpoint <azure-rm-service-connection-name> \
  --secrets DB_PASSWORD API_KEY
```

```yaml
# Consume a variable group in a pipeline
variables:
  - group: production-config
  - group: kv-secrets

steps:
  - script: echo "Connecting to $(API_URL)"
```

---

## Self-Hosted Agents

Self-hosted agents run on your own infrastructure, giving you control over hardware, software, and network access.

```bash
# Download the agent
mkdir myagent && cd myagent
curl -O https://vstsagentpackage.azureedge.net/agent/3.236.1/vsts-agent-linux-x64-3.236.1.tar.gz
tar zxvf vsts-agent-linux-x64-3.236.1.tar.gz

# Configure the agent (interactive)
./config.sh \
  --url https://dev.azure.com/MyOrg \
  --auth pat \
  --token <your-pat> \
  --pool "My Linux Pool" \
  --agent MyAgent01 \
  --unattended

# Install and start as a systemd service
sudo ./svc.sh install
sudo ./svc.sh start

# Run as a Docker container
docker run \
  -e AZP_URL=https://dev.azure.com/MyOrg \
  -e AZP_TOKEN=<your-pat> \
  -e AZP_AGENT_NAME=docker-agent-01 \
  -e AZP_POOL="Docker Pool" \
  mcr.microsoft.com/azure-pipelines/vsts-agent:ubuntu-22.04
```

```yaml
# Use a self-hosted agent pool in a pipeline
pool:
  name: "My Linux Pool"
  demands:
    - docker          # only pick agents that have docker installed
    - Agent.OS -equals Linux
```

---

## Common Patterns & Tips

1. **Use `--output table` + `--query` together** to get readable summaries:
   ```bash
   az pipelines runs list --pipeline-name "ci" \
     --query "[].{ID:id,Branch:sourceBranch,Status:status,Result:result}" \
     --output table
   ```

2. **Store PATs in environment variables, never in shell history.** Use `read -s` to prompt without echo:
   ```bash
   read -s AZURE_DEVOPS_EXT_PAT
   export AZURE_DEVOPS_EXT_PAT
   az devops login
   ```

3. **Use pipeline templates for DRY pipelines.** Extract common `steps`, `jobs`, or `stages` into `templates/` and reference them from every pipeline that needs them, keeping the main YAML file under 50 lines.

4. **Gate production deployments with approval + timer checks.** In the Azure DevOps UI add both an *Approvals* check and a *Business Hours* check to your `production` environment so deploys only run during work hours after a human approves.

5. **Cache aggressively.** Use the `Cache@2` task for dependency managers (npm, pip, Maven) keyed on the lock file hash. Cold builds on hosted agents can be 5× slower than warm cached builds.

6. **Use `condition:` on stages to skip unnecessary work:**
   ```yaml
   condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
   ```

7. **Set `failFast: true` on a matrix job** to cancel sibling jobs immediately when one fails, saving agent minutes on flaky test matrices.

8. **Scope service connections to a single pipeline** — in the service connection settings tick *Grant access permission to all pipelines: OFF* and explicitly approve each pipeline. This reduces blast radius if a pipeline is compromised.

9. **Use Universal Packages for build outputs** rather than pipeline artifacts when you need cross-pipeline or long-term retention; artifact retention policies on pipeline runs are often set to 30 days.

10. **Rotate PATs regularly and prefer Managed Identities or Workload Identity Federation** for Azure service connections instead of service principal client secrets, which expire and require manual rotation.
