# Azure DevOps Interview Questions - Complete Guide

> **300+ Azure DevOps Interview Questions for Senior DevOps Engineer, SRE, and Platform Engineer roles at FAANG companies**

---

## 📋 Table of Contents

- [Azure DevOps Overview](#azure-devops-overview)
- [Azure Pipelines](#azure-pipelines)
- [YAML Pipelines](#yaml-pipelines)
- [Azure Repos](#azure-repos)
- [Azure Artifacts](#azure-artifacts)
- [Azure Boards](#azure-boards)
- [Security & Compliance](#security--compliance)
- [Enterprise Patterns](#enterprise-patterns)

---

## Azure DevOps Overview

### 🟢 Basic Questions

#### Q1: Explain Azure DevOps services and their purposes.

**Basic Answer:**
Azure DevOps provides five main services: Azure Boards (work tracking), Azure Repos (Git repos), Azure Pipelines (CI/CD), Azure Test Plans (testing), and Azure Artifacts (package management).

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    AZURE DEVOPS SERVICES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │    │
│  │  │    BOARDS    │  │    REPOS     │  │  PIPELINES   │   │    │
│  │  │              │  │              │  │              │   │    │
│  │  │ • Work Items │  │ • Git Repos  │  │ • Build      │   │    │
│  │  │ • Sprints    │  │ • PR Review  │  │ • Release    │   │    │
│  │  │ • Backlogs   │  │ • Branching  │  │ • YAML/UI    │   │    │
│  │  │ • Dashboards │  │ • Policies   │  │ • Agents     │   │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │    │
│  │                                                          │    │
│  │  ┌──────────────┐  ┌──────────────┐                     │    │
│  │  │  TEST PLANS  │  │  ARTIFACTS   │                     │    │
│  │  │              │  │              │                     │    │
│  │  │ • Manual     │  │ • NuGet      │                     │    │
│  │  │ • Automated  │  │ • npm        │                     │    │
│  │  │ • Exploratory│  │ • Maven      │                     │    │
│  │  │ • Load       │  │ • Python     │                     │    │
│  │  └──────────────┘  └──────────────┘                     │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ORGANIZATION HIERARCHY:                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Organization (dev.azure.com/myorg)                      │    │
│  │  │                                                       │    │
│  │  ├── Project 1                                           │    │
│  │  │   ├── Repos                                           │    │
│  │  │   ├── Pipelines                                       │    │
│  │  │   ├── Boards                                          │    │
│  │  │   └── Artifacts                                       │    │
│  │  │                                                       │    │
│  │  ├── Project 2                                           │    │
│  │  │   └── ...                                             │    │
│  │  │                                                       │    │
│  │  └── Shared Resources                                    │    │
│  │      ├── Agent Pools                                     │    │
│  │      ├── Service Connections                             │    │
│  │      ├── Variable Groups                                 │    │
│  │      └── Library (Secure Files)                          │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Azure Pipelines

### 🟡 Intermediate Questions

#### Q2: Explain the difference between Classic and YAML pipelines.

**Basic Answer:**
Classic pipelines use a UI-based editor and store configuration in Azure DevOps. YAML pipelines are code-defined, version-controlled with the source code, and support better review processes.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLASSIC vs YAML PIPELINES                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CLASSIC PIPELINES                 YAML PIPELINES               │
│  ─────────────────────            ─────────────────────         │
│  ✓ Visual designer                ✓ Code as configuration       │
│  ✓ Quick setup                    ✓ Version controlled          │
│  ✓ No code knowledge needed       ✓ PR review for changes       │
│  ✗ Not version controlled         ✓ Templates & reuse           │
│  ✗ Hard to review changes         ✓ Multi-stage pipelines       │
│  ✗ Limited reusability            ✓ Environment approvals       │
│                                                                  │
│  YAML PIPELINE STRUCTURE:                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Pipeline                                                │    │
│  │  │                                                       │    │
│  │  ├── trigger              # When to run                  │    │
│  │  │                                                       │    │
│  │  ├── variables            # Pipeline variables           │    │
│  │  │                                                       │    │
│  │  ├── stages               # Deployment stages            │    │
│  │  │   │                                                   │    │
│  │  │   ├── stage: Build                                    │    │
│  │  │   │   └── jobs                                        │    │
│  │  │   │       └── job: BuildJob                           │    │
│  │  │   │           └── steps                               │    │
│  │  │   │               ├── task: Build                     │    │
│  │  │   │               └── task: Test                      │    │
│  │  │   │                                                   │    │
│  │  │   ├── stage: Deploy_Dev                               │    │
│  │  │   │   └── deployment job                              │    │
│  │  │   │                                                   │    │
│  │  │   └── stage: Deploy_Prod                              │    │
│  │  │       └── deployment job (with approval)              │    │
│  │  │                                                       │    │
│  │  └── resources            # External resources           │    │
│  │      ├── repositories                                    │    │
│  │      ├── pipelines                                       │    │
│  │      └── containers                                      │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## YAML Pipelines

### 🔴 Advanced Questions

#### Q3: Design a multi-stage YAML pipeline with environments and approvals.

**Basic Answer:**
Use stages for different environments, deployment jobs with environment references, and configure approvals in environment settings.

**Advanced Answer:**

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    exclude:
      - docs/*
      - README.md

pr:
  branches:
    include:
      - main

variables:
  - group: global-variables
  - name: dockerRegistry
    value: 'myregistry.azurecr.io'

stages:
  # ==================== BUILD STAGE ====================
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: Build
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: Docker@2
            displayName: 'Build Docker Image'
            inputs:
              containerRegistry: 'acr-connection'
              repository: 'myapp'
              command: 'build'
              Dockerfile: '**/Dockerfile'
              tags: |
                $(Build.BuildId)
                latest

          - task: Docker@2
            displayName: 'Push Docker Image'
            inputs:
              containerRegistry: 'acr-connection'
              repository: 'myapp'
              command: 'push'
              tags: |
                $(Build.BuildId)
                latest

          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: '$(Build.SourcesDirectory)/k8s'
              artifact: 'manifests'

  # ==================== DEV STAGE ====================
  - stage: Deploy_Dev
    displayName: 'Deploy to Dev'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: DeployDev
        displayName: 'Deploy to Dev Environment'
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'dev'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: manifests

                - task: KubernetesManifest@0
                  displayName: 'Deploy to AKS'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'aks-dev'
                    namespace: 'myapp'
                    manifests: |
                      $(Pipeline.Workspace)/manifests/*.yaml
                    containers: |
                      $(dockerRegistry)/myapp:$(Build.BuildId)

  # ==================== STAGING STAGE ====================
  - stage: Deploy_Staging
    displayName: 'Deploy to Staging'
    dependsOn: Deploy_Dev
    condition: succeeded()
    jobs:
      - deployment: DeployStaging
        displayName: 'Deploy to Staging Environment'
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - download: current
                  artifact: manifests
                  
                - task: KubernetesManifest@0
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'aks-staging'
                    namespace: 'myapp'
                    manifests: |
                      $(Pipeline.Workspace)/manifests/*.yaml

  # ==================== PRODUCTION STAGE ====================
  - stage: Deploy_Production
    displayName: 'Deploy to Production'
    dependsOn: Deploy_Staging
    condition: succeeded()
    jobs:
      - deployment: DeployProd
        displayName: 'Deploy to Production'
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'production'  # Has approval gate configured
        strategy:
          canary:
            increments: [10, 50]
            deploy:
              steps:
                - download: current
                  artifact: manifests
                  
                - task: KubernetesManifest@0
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'aks-prod'
                    namespace: 'myapp'
                    manifests: |
                      $(Pipeline.Workspace)/manifests/*.yaml
                    strategy: canary
                    percentage: $(strategy.increment)
            
            postRouteTraffic:
              steps:
                - task: AzureCLI@2
                  displayName: 'Run smoke tests'
                  inputs:
                    azureSubscription: 'production-subscription'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Run smoke tests
                      curl -f https://myapp.prod.example.com/health
            
            on:
              failure:
                steps:
                  - task: KubernetesManifest@0
                    displayName: 'Rollback'
                    inputs:
                      action: 'reject'
                      kubernetesServiceConnection: 'aks-prod'
                      namespace: 'myapp'
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    ENVIRONMENT APPROVALS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Configure in: Pipelines > Environments > production > ...      │
│                                                                  │
│  APPROVAL SETTINGS:                                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Approvers:                                              │    │
│  │  • User: release-manager@company.com                     │    │
│  │  • Group: Production Approvers                           │    │
│  │                                                          │    │
│  │  Options:                                                │    │
│  │  □ Allow approvers to approve their own runs             │    │
│  │  ☑ Require a minimum number of approvers: 2              │    │
│  │                                                          │    │
│  │  Timeout: 72 hours                                       │    │
│  │                                                          │    │
│  │  Instructions: "Review deployment changes and verify     │    │
│  │  staging tests passed before approving production."      │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ADDITIONAL CHECKS:                                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  • Business Hours: Only deploy Mon-Fri 9am-4pm           │    │
│  │  • Branch Control: Only from main branch                 │    │
│  │  • Required Template: Must use approved template         │    │
│  │  • Invoke Azure Function: Custom validation              │    │
│  │  • Query Work Items: All linked items in Done state      │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Security & Compliance

### 🔴 Advanced Questions

#### Q4: How do you secure Azure DevOps pipelines?

**Basic Answer:**
Use service connections with least privilege, variable groups with secrets, branch policies, environment approvals, and audit logging.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    PIPELINE SECURITY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. SERVICE CONNECTIONS:                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  • Use Workload Identity Federation (no secrets!)        │    │
│  │  • Scope to specific resource groups                     │    │
│  │  • Grant pipeline-level, not project-level               │    │
│  │  • Require approval for production connections           │    │
│  │                                                          │    │
│  │  # Workload Identity setup:                              │    │
│  │  Azure AD App Registration                               │    │
│  │    └── Federated Credential                              │    │
│  │        Subject: sc://org/project/connection-name         │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  2. SECRETS MANAGEMENT:                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Variable Groups (linked to Key Vault):                  │    │
│  │  - name: prod-secrets                                    │    │
│  │    type: AzureKeyVault                                   │    │
│  │    azureSubscription: 'keyvault-connection'              │    │
│  │    keyVaultName: 'my-keyvault'                           │    │
│  │                                                          │    │
│  │  # In pipeline:                                          │    │
│  │  variables:                                              │    │
│  │    - group: prod-secrets  # Auto-fetches from Key Vault  │    │
│  │                                                          │    │
│  │  Best Practices:                                         │    │
│  │  • Never echo secrets in logs                            │    │
│  │  • Use secret variables (isSecret: true)                 │    │
│  │  • Rotate secrets regularly                              │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  3. BRANCH POLICIES:                                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Protect main branch:                                    │    │
│  │  ☑ Require minimum number of reviewers: 2                │    │
│  │  ☑ Check for linked work items                           │    │
│  │  ☑ Check for comment resolution                          │    │
│  │  ☑ Build validation (run CI on PR)                       │    │
│  │  ☑ Require merge strategy: Squash                        │    │
│  │  ☑ Automatically include reviewers (CODEOWNERS)          │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  4. PIPELINE PERMISSIONS:                                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # Restrict who can edit pipelines                       │    │
│  │  Pipeline Settings:                                      │    │
│  │  ☑ Limit job authorization scope to current project      │    │
│  │  ☑ Limit job authorization scope to referenced repos     │    │
│  │  ☑ Protect access to repositories in YAML pipelines      │    │
│  │                                                          │    │
│  │  # Pipeline-level permissions                            │    │
│  │  Readers: View pipeline                                  │    │
│  │  Contributors: Queue builds                              │    │
│  │  Administrators: Edit pipeline                           │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  5. AGENT SECURITY:                                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  • Use Microsoft-hosted agents (ephemeral)               │    │
│  │  • Self-hosted: Isolate in dedicated VNet                │    │
│  │  • Self-hosted: Run as non-root user                     │    │
│  │  • Self-hosted: Regular patching                         │    │
│  │  • Use agent pools with specific permissions             │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Enterprise Patterns

### ⚫ Expert Questions

#### Q5: Design a multi-team Azure DevOps structure with shared components.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│              ENTERPRISE AZURE DEVOPS ARCHITECTURE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ORGANIZATION STRUCTURE:                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Organization: company.visualstudio.com                  │    │
│  │  │                                                       │    │
│  │  ├── Platform Team Project                               │    │
│  │  │   ├── Shared Pipeline Templates                       │    │
│  │  │   ├── Shared Variable Groups                          │    │
│  │  │   ├── Centralized Agent Pools                         │    │
│  │  │   └── Common Build Tasks                              │    │
│  │  │                                                       │    │
│  │  ├── Product A Project                                   │    │
│  │  │   ├── Uses: @templates from Platform                  │    │
│  │  │   └── Team-specific pipelines                         │    │
│  │  │                                                       │    │
│  │  ├── Product B Project                                   │    │
│  │  │   └── ...                                             │    │
│  │  │                                                       │    │
│  │  └── Shared Services Project                             │    │
│  │      ├── Common libraries                                │    │
│  │      └── Internal packages                               │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  TEMPLATE REPOSITORY PATTERN:                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # Platform team's template repo                         │    │
│  │  templates/                                              │    │
│  │  ├── stages/                                             │    │
│  │  │   ├── build.yml                                       │    │
│  │  │   ├── deploy-aks.yml                                  │    │
│  │  │   └── security-scan.yml                               │    │
│  │  ├── jobs/                                               │    │
│  │  │   ├── docker-build.yml                                │    │
│  │  │   └── terraform-deploy.yml                            │    │
│  │  └── steps/                                              │    │
│  │      ├── npm-install.yml                                 │    │
│  │      └── sonar-scan.yml                                  │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  CONSUMING TEMPLATES:                                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # Product team's azure-pipelines.yml                    │    │
│  │  resources:                                              │    │
│  │    repositories:                                         │    │
│  │      - repository: templates                             │    │
│  │        type: git                                         │    │
│  │        name: Platform/pipeline-templates                 │    │
│  │        ref: refs/tags/v2.0.0  # Pin to version!          │    │
│  │                                                          │    │
│  │  stages:                                                 │    │
│  │    - template: stages/build.yml@templates                │    │
│  │      parameters:                                         │    │
│  │        buildConfiguration: Release                       │    │
│  │        runTests: true                                    │    │
│  │                                                          │    │
│  │    - template: stages/deploy-aks.yml@templates           │    │
│  │      parameters:                                         │    │
│  │        environment: production                           │    │
│  │        aksCluster: prod-aks                              │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  GOVERNANCE:                                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Required Template Check:                                │    │
│  │  • Enforce specific templates for production             │    │
│  │  • Version pinning required                              │    │
│  │                                                          │    │
│  │  Audit & Compliance:                                     │    │
│  │  • Stream audit logs to Azure Monitor                    │    │
│  │  • Track all pipeline changes                            │    │
│  │  • Compliance dashboard                                  │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📚 Documentation Links

- [Azure DevOps Documentation](https://docs.microsoft.com/en-us/azure/devops/)
- [YAML Schema Reference](https://docs.microsoft.com/en-us/azure/devops/pipelines/yaml-schema)
- [Pipeline Templates](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates)

---

**[← Back to Main README](../README.md)** | **[Next: Jenkins →](../jenkins/README.md)**
