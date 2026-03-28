# Azure DevOps Cheatsheet

> A comprehensive reference for Azure DevOps — Pipelines, Repos, Boards, Artifacts, and the az devops CLI.

---

## Table of Contents

- [Setup & Configuration](#setup--configuration)
- [Projects & Organizations](#projects--organizations)
- [Azure Repos (Git)](#azure-repos-git)
- [Azure Boards](#azure-boards)
- [Azure Pipelines — YAML Reference](#azure-pipelines--yaml-reference)
- [Pipeline Templates & Reuse](#pipeline-templates--reuse)
- [Multi-Stage Pipelines](#multi-stage-pipelines)
- [Service Connections](#service-connections)
- [Variable Groups & Secrets](#variable-groups--secrets)
- [Azure Artifacts](#azure-artifacts)
- [Environments & Approvals](#environments--approvals)
- [Self-Hosted Agents](#self-hosted-agents)
- [Advanced Scenarios](#advanced-scenarios)

---

## Setup & Configuration

```bash
# Install the Azure DevOps CLI extension
az extension add --name azure-devops

# Login
az login
az devops configure --defaults organization=https://dev.azure.com/my-org

# Set default project
az devops configure --defaults project=my-project

# Show current defaults
az devops configure --list

# List all projects in an org
az devops project list --output table
```

---

## Projects & Organizations

```bash
# Create a project
az devops project create \
  --name my-project \
  --organization https://dev.azure.com/my-org \
  --visibility private \
  --process Agile

# Show project details
az devops project show --project my-project

# Delete project
az devops project delete --id <PROJECT_ID> --yes
```

---

## Azure Repos (Git)

```bash
# List repositories
az repos list --output table

# Create a repository
az repos create --name my-repo

# Show repo details (includes clone URLs)
az repos show --repository my-repo

# Clone repo (get URL first)
git clone https://my-org@dev.azure.com/my-org/my-project/_git/my-repo

# Import an external repo
az repos import create \
  --git-source-url https://github.com/user/repo.git \
  --repository my-repo

# List branches
az repos ref list \
  --repository my-repo \
  --filter heads \
  --output table

# Create a branch policy (require PR reviews)
az repos policy approver-count create \
  --allow-downvotes false \
  --branch main \
  --branch-match-type exact \
  --creator-vote-counts false \
  --enabled true \
  --minimum-approver-count 2 \
  --repository-id <REPO_ID>

# Require linked work items on PR
az repos policy work-item-linking create \
  --blocking true \
  --branch main \
  --enabled true \
  --repository-id <REPO_ID>

# Require successful build on PR
az repos policy build create \
  --blocking true \
  --branch main \
  --enabled true \
  --build-definition-id <PIPELINE_ID> \
  --repository-id <REPO_ID> \
  --queue-on-source-update-only true \
  --manual-queue-only false \
  --display-name "CI Build"

# List pull requests
az repos pr list --repository my-repo --output table

# Create a pull request
az repos pr create \
  --repository my-repo \
  --source-branch feature/login \
  --target-branch main \
  --title "feat: add login" \
  --description "Closes #123" \
  --reviewers user@example.com

# Approve a pull request
az repos pr set-vote --id <PR_ID> --vote approve

# Complete (merge) a PR
az repos pr update \
  --id <PR_ID> \
  --status completed \
  --merge-strategy squash \
  --delete-source-branch true
```

---

## Azure Boards

```bash
# List work items
az boards work-item query \
  --wiql "SELECT [System.Id],[System.Title],[System.State] FROM WorkItems WHERE [System.TeamProject]='my-project' ORDER BY [System.Id]" \
  --output table

# Show a work item
az boards work-item show --id 42

# Create a work item
az boards work-item create \
  --title "Fix login bug" \
  --type Bug \
  --description "Users can't log in with SSO" \
  --assigned-to user@example.com \
  --area my-project\Team1

# Update a work item
az boards work-item update \
  --id 42 \
  --state Active \
  --assigned-to developer@example.com

# Add a comment
az boards work-item comment add \
  --id 42 \
  --text "Root cause found — JWT expiry check is off by 1"

# List work item relations
az boards work-item relation show --id 42

# Create a relation (link two work items)
az boards work-item relation add \
  --id 42 \
  --relation-type "Child" \
  --target-id 99

# List iterations (sprints)
az boards iteration project list --output table

# Create a sprint
az boards iteration project create \
  --name "Sprint 5" \
  --start-date 2024-04-01 \
  --finish-date 2024-04-14
```

---

## Azure Pipelines — YAML Reference

### Minimal Pipeline

```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: ubuntu-latest

steps:
  - script: echo "Hello, Azure DevOps!"
    displayName: "Say Hello"
```

### Full CI Pipeline (Node.js)

```yaml
trigger:
  branches:
    include:
      - main
      - feature/*
  paths:
    exclude:
      - '**/*.md'

pr:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

variables:
  NODE_VERSION: '20.x'
  IMAGE_NAME: 'myregistry.azurecr.io/myapp'

stages:
  # ── Stage 1: Build & Test ──────────────────────────────────
  - stage: Build
    displayName: Build & Test
    jobs:
      - job: BuildJob
        displayName: Build and Test
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: $(NODE_VERSION)
            displayName: Install Node.js

          - script: npm ci
            displayName: Install dependencies

          - script: npm run lint
            displayName: Lint

          - script: npm test -- --coverage
            displayName: Run tests

          - task: PublishTestResults@2
            condition: always()
            inputs:
              testResultsFormat: JUnit
              testResultsFiles: '**/test-results.xml'

          - task: PublishCodeCoverageResults@1
            inputs:
              codeCoverageTool: Cobertura
              summaryFileLocation: '**/coverage/cobertura-coverage.xml'

          - script: npm run build
            displayName: Build

          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: dist
              artifactName: build-output

  # ── Stage 2: Docker Build & Push ──────────────────────────
  - stage: Docker
    displayName: Build & Push Image
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - job: DockerJob
        steps:
          - task: Docker@2
            displayName: Build and push
            inputs:
              containerRegistry: myacr-connection
              repository: myapp
              command: buildAndPush
              Dockerfile: Dockerfile
              tags: |
                $(Build.BuildId)
                latest
```

### Full CD Pipeline (Deploy to AKS)

```yaml
  # ── Stage 3: Deploy to Staging ────────────────────────────
  - stage: DeployStaging
    displayName: Deploy to Staging
    dependsOn: Docker
    jobs:
      - deployment: DeployToStaging
        displayName: Deploy to Staging AKS
        environment: staging
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@1
                  displayName: Deploy
                  inputs:
                    action: deploy
                    connectionType: azureResourceManager
                    azureSubscriptionConnection: my-azure-connection
                    azureResourceGroup: my-rg
                    kubernetesCluster: my-aks
                    namespace: staging
                    manifests: |
                      k8s/deployment.yaml
                      k8s/service.yaml
                    containers: |
                      myregistry.azurecr.io/myapp:$(Build.BuildId)

  # ── Stage 4: Deploy to Production (with approval) ─────────
  - stage: DeployProduction
    displayName: Deploy to Production
    dependsOn: DeployStaging
    jobs:
      - deployment: DeployToProduction
        environment: production           # gate approval configured in Environments
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@1
                  inputs:
                    action: deploy
                    connectionType: azureResourceManager
                    azureSubscriptionConnection: my-azure-connection
                    azureResourceGroup: my-rg
                    kubernetesCluster: my-aks
                    namespace: production
                    manifests: k8s/deployment.yaml
                    containers: |
                      myregistry.azurecr.io/myapp:$(Build.BuildId)
```

### Useful Pipeline Snippets

```yaml
# ── Checkout specific ref ─────────────────────────────────
steps:
  - checkout: self
    fetchDepth: 0              # full history (needed for versioning)
    persistCredentials: true

# ── Conditional step ─────────────────────────────────────
  - script: npm run deploy
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

# ── Retry a step ─────────────────────────────────────────
  - script: ./run-flaky-test.sh
    retryCountOnTaskFailure: 3

# ── Timeout on a step ────────────────────────────────────
  - script: npm run e2e
    timeoutInMinutes: 30

# ── Output a variable from a step ─────────────────────────
  - bash: echo "##vso[task.setvariable variable=MY_VAR;isOutput=true]hello"
    name: setVar

  - bash: echo "$(setVar.MY_VAR)"

# ── Download artifact from another pipeline ───────────────
  - task: DownloadPipelineArtifact@2
    inputs:
      buildType: specific
      project: my-project
      definition: 42
      artifactName: build-output
      targetPath: $(Pipeline.Workspace)/artifacts

# ── Cache dependencies ────────────────────────────────────
  - task: Cache@2
    inputs:
      key: 'npm | "$(Agent.OS)" | package-lock.json'
      restoreKeys: |
        npm | "$(Agent.OS)"
      path: $(npm_config_cache)
    displayName: Cache npm packages
```

---

## Pipeline Templates & Reuse

### Calling a template

```yaml
# azure-pipelines.yml
stages:
  - template: templates/build.yml
    parameters:
      nodeVersion: '20.x'
      runTests: true
```

### `templates/build.yml`

```yaml
parameters:
  - name: nodeVersion
    type: string
    default: '20.x'
  - name: runTests
    type: boolean
    default: true

stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - task: NodeTool@0
            inputs:
              versionSpec: ${{ parameters.nodeVersion }}

          - script: npm ci

          - ${{ if parameters.runTests }}:
            - script: npm test
              displayName: Run tests

          - script: npm run build
```

### Shared step template

```yaml
# templates/steps/docker-push.yml
parameters:
  - name: imageTag
    type: string
  - name: registry
    type: string

steps:
  - task: Docker@2
    inputs:
      containerRegistry: ${{ parameters.registry }}
      repository: myapp
      command: buildAndPush
      tags: ${{ parameters.imageTag }}
```

---

## Multi-Stage Pipelines

```yaml
stages:
  - stage: CI
    jobs:
      - job: Test
        steps:
          - script: npm test

  - stage: Staging
    dependsOn: CI
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: staging
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to staging"

  - stage: Production
    dependsOn: Staging
    condition: succeeded()
    jobs:
      - deployment: Deploy
        environment: production    # manual approval gate here
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to production"
```

---

## Service Connections

```bash
# List service connections
az devops service-endpoint list --output table

# Create an Azure Resource Manager service connection
az devops service-endpoint azurerm create \
  --azure-rm-service-principal-id <SP_APP_ID> \
  --azure-rm-subscription-id <SUB_ID> \
  --azure-rm-subscription-name "My Subscription" \
  --azure-rm-tenant-id <TENANT_ID> \
  --name my-azure-connection

# Show endpoint
az devops service-endpoint show --id <ENDPOINT_ID>

# Delete service connection
az devops service-endpoint delete --id <ENDPOINT_ID> --yes
```

In YAML, reference a service connection as:

```yaml
- task: AzureCLI@2
  inputs:
    azureSubscription: my-azure-connection    # name of the connection
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: az group list
```

---

## Variable Groups & Secrets

### CLI

```bash
# Create a variable group
az pipelines variable-group create \
  --name production-vars \
  --authorize true \
  --variables NODE_ENV=production PORT=3000

# List variable groups
az pipelines variable-group list --output table

# Add a variable
az pipelines variable-group variable create \
  --group-id <GROUP_ID> \
  --name DB_URL \
  --value "postgres://user:pass@host/db" \
  --secret true

# Update a variable
az pipelines variable-group variable update \
  --group-id <GROUP_ID> \
  --name DB_URL \
  --value "postgres://new-host/db"

# Delete a variable
az pipelines variable-group variable delete \
  --group-id <GROUP_ID> \
  --name DB_URL --yes
```

### Using Variable Groups in YAML

```yaml
variables:
  - group: production-vars              # links the variable group
  - name: IMAGE_TAG
    value: $(Build.BuildId)

steps:
  - script: echo "$(NODE_ENV)"          # from group
  - script: echo "$(IMAGE_TAG)"         # locally defined
```

### Key Vault Integration

```yaml
variables:
  - group: production-vars              # variable group linked to Key Vault

# In the variable group UI:
# Link to Key Vault → all Key Vault secrets become pipeline variables
```

---

## Azure Artifacts

```bash
# List feeds
az artifacts universal packages list \
  --organization https://dev.azure.com/my-org \
  --feed my-feed \
  --package-name mypackage

# Create a feed
# (UI: Artifacts → Create Feed)

# Publish a Universal Package
az artifacts universal publish \
  --organization https://dev.azure.com/my-org \
  --feed my-feed \
  --name mypackage \
  --version 1.0.0 \
  --description "My package" \
  --path ./dist

# Download a Universal Package
az artifacts universal download \
  --organization https://dev.azure.com/my-org \
  --feed my-feed \
  --name mypackage \
  --version 1.0.0 \
  --path ./downloaded
```

### npm Feed

```bash
# Authenticate npm to Azure Artifacts
npmrc_file="$(npm config get userconfig)"
echo "registry=https://pkgs.dev.azure.com/my-org/my-project/_packaging/my-feed/npm/registry/" >> $npmrc_file
vsts-npm-auth -config $npmrc_file

# Publish a package
npm publish

# Install from feed
npm install @my-scope/my-package
```

### In a pipeline

```yaml
- task: npmAuthenticate@0
  inputs:
    workingFile: .npmrc

- script: npm publish
  displayName: Publish to Artifacts feed
```

---

## Environments & Approvals

```bash
# List environments
az devops environment list --output table

# Create environment
az devops environment create --name staging
az devops environment create --name production
```

### Adding manual approval via UI (YAML trigger)

```yaml
# In azure-pipelines.yml — reference the environment name
jobs:
  - deployment: Deploy
    environment: production     # must match the environment name in Azure DevOps
    strategy:
      runOnce:
        deploy:
          steps:
            - script: echo "Deploying..."
```

In the **Azure DevOps portal → Environments → production → Approvals and checks**:
- Add `Approvals` check → select required approvers
- Add `Business Hours` check (optional)
- Add `Invoke Azure Function` / `Query Azure Monitor Alerts` for automated gates

---

## Self-Hosted Agents

```bash
# Download agent
wget https://vstsagentpackage.azureedge.net/agent/3.x.x/vsts-agent-linux-x64-3.x.x.tar.gz
mkdir myagent && tar -xzf vsts-agent-linux-x64-*.tar.gz -C myagent
cd myagent

# Configure agent
./config.sh \
  --url https://dev.azure.com/my-org \
  --auth PAT \
  --token <PERSONAL_ACCESS_TOKEN> \
  --pool default \
  --agent myagent-01 \
  --acceptTeeEula

# Run as a service
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status

# Remove agent
sudo ./svc.sh stop
sudo ./svc.sh uninstall
./config.sh remove
```

### Using agent pool in YAML

```yaml
pool:
  name: my-self-hosted-pool    # self-hosted pool name
  demands:
    - docker                   # agent capability requirement
```

### Run agent in Docker

```yaml
# docker-compose.yml for self-hosted agent
services:
  azure-agent:
    image: mcr.microsoft.com/azure-pipelines/vsts-agent:ubuntu-22.04
    environment:
      AZP_URL: https://dev.azure.com/my-org
      AZP_TOKEN: ${AZP_TOKEN}
      AZP_AGENT_NAME: docker-agent-01
      AZP_POOL: default
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

---

## Advanced Scenarios

### Scenario 1 — Matrix strategy (test across multiple Node versions)

```yaml
jobs:
  - job: Test
    strategy:
      matrix:
        node18:
          NODE_VERSION: '18.x'
        node20:
          NODE_VERSION: '20.x'
        node22:
          NODE_VERSION: '22.x'
      maxParallel: 3
    steps:
      - task: NodeTool@0
        inputs:
          versionSpec: $(NODE_VERSION)
      - script: npm ci && npm test
```

### Scenario 2 — Trigger pipeline on another pipeline completing

```yaml
resources:
  pipelines:
    - pipeline: build-pipeline
      source: MyApp-CI
      trigger:
        branches:
          include:
            - main

trigger: none     # only trigger from upstream pipeline
```

### Scenario 3 — Rollback on failed deployment

```yaml
- deployment: Deploy
  environment: production
  strategy:
    runOnce:
      deploy:
        steps:
          - script: ./deploy.sh
      on:
        failure:
          steps:
            - script: ./rollback.sh
              displayName: Rollback on failure
```

### Scenario 4 — Run pipeline steps only when files change

```yaml
trigger:
  paths:
    include:
      - src/**
      - package.json
    exclude:
      - docs/**
      - '**/*.md'
```

### Scenario 5 — Scheduled nightly pipeline

```yaml
schedules:
  - cron: "0 2 * * *"          # 02:00 UTC every day
    displayName: Nightly build
    branches:
      include:
        - main
    always: true               # run even if no code changes

trigger: none
```

### Scenario 6 — Publish and consume pipeline artifacts between stages

```yaml
# Stage 1 — publish
- task: PublishPipelineArtifact@1
  inputs:
    targetPath: $(Build.ArtifactStagingDirectory)/dist
    artifact: webapp

# Stage 2 — consume
- task: DownloadPipelineArtifact@2
  inputs:
    artifact: webapp
    path: $(Pipeline.Workspace)/webapp
```

### Scenario 7 — Semantic versioning with GitVersion

```yaml
steps:
  - task: gitversion/setup@0
    inputs:
      versionSpec: '5.x'

  - task: gitversion/execute@0

  - script: echo "Version $(GitVersion.SemVer)"

  - task: Docker@2
    inputs:
      command: buildAndPush
      tags: |
        $(GitVersion.SemVer)
        latest
```

### Scenario 8 — Post build status to Slack

```yaml
- bash: |
    STATUS="✅ Success"
    [[ "$AGENT_JOBSTATUS" == "Failed" ]] && STATUS="❌ Failed"
    curl -X POST "$SLACK_WEBHOOK" \
      -H 'Content-type: application/json' \
      --data "{\"text\":\"$STATUS: $(Build.DefinitionName) #$(Build.BuildNumber)\"}"
  condition: always()
  env:
    SLACK_WEBHOOK: $(SLACK_WEBHOOK_URL)
  displayName: Notify Slack
```

---

## Quick Reference Card

| Task                                | Command / Reference                                              |
|-------------------------------------|------------------------------------------------------------------|
| Set default org + project           | `az devops configure --defaults organization=... project=...`   |
| List pipelines                      | `az pipelines list --output table`                              |
| Run a pipeline                      | `az pipelines run --name my-pipeline`                           |
| Create PR                           | `az repos pr create --source-branch feature/x --target-branch main` |
| List work items                     | `az boards work-item query --wiql "SELECT ..."`                 |
| Create variable group               | `az pipelines variable-group create --name group --variables K=V` |
| List environments                   | `az devops environment list`                                    |
| Trigger: branch filter              | `trigger: branches: include: [main, release/*]`                 |
| Skip CI on commit                   | Commit message: `[skip ci]`                                     |
| Cache node_modules in pipeline      | `task: Cache@2` with `key: npm \| package-lock.json`            |

---

> Pin pipeline task versions (e.g., `Docker@2`) to avoid unexpected breaking changes on updates.
