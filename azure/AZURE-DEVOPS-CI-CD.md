# Azure DevOps & CI/CD Pipelines - Production Patterns

> Azure Pipelines mastery, GitOps, and deployment strategies - 10% of questions

**Coverage:** 100+ scenarios | **Focus:** YAML pipelines, multi-stage deployments, approval gates

---

## CI/CD Pipeline Fundamentals

### Q: Design multi-stage YAML pipeline for AKS deployment

```
PIPELINE FLOW:

Source: Git repository
  │
  ├─ Trigger: Push to main branch
  │  └─ Trigger pipeline
  │
  ├─ Stage 1: BUILD
  │  ├─ Checkout code
  │  ├─ Run unit tests (pytest, jest, etc.)
  │  ├─ Build application (docker build)
  │  ├─ Push to Container Registry (ACR)
  │  └─ Publish test results
  │
  ├─ Stage 2: DEPLOY-DEV
  │  ├─ Get image from ACR
  │  ├─ Deploy to dev AKS cluster
  │  ├─ Run integration tests
  │  └─ No approval required (dev is fast-moving)
  │
  ├─ Stage 3: DEPLOY-STAGING
  │  ├─ Requires manual approval
  │  ├─ QA validates in staging
  │  └─ Performance testing
  │
  ├─ Stage 4: DEPLOY-PRODUCTION
  │  ├─ Requires approval from 2 senior engineers
  │  ├─ Blue-green deployment (zero downtime)
  │  ├─ Health checks
  │  ├─ Smoke tests
  │  └─ Gradual rollout (Canary: 10% → 50% → 100%)
  │
  └─ Final: ROLLBACK (if failures)
     ├─ Automatic or manual
     └─ Restore previous version

YAML PIPELINE STRUCTURE:

trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - app/**
      - .pipelines/**
      - azure-pipelines.yml

pr:
  branches:
    include:
      - main
  paths:
    include:
      - app/**

variables:
  dockerRegistryServiceConnection: 'azure-registry'
  imageRepository: 'myapp'
  containerRegistry: 'prodacr.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'
  REGISTRY_USERNAME: $(REGISTRY_USERNAME)
  REGISTRY_PASSWORD: $(REGISTRY_PASSWORD)

stages:
  - stage: Build
    displayName: 'Build & Test'
    jobs:
      - job: BuildJob
        displayName: 'Build'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: UsePythonVersion@0
            inputs:
              versionSpec: '3.9'

          - script: |
              pip install -r requirements.txt
              pytest tests/ --junitxml=junit/test-results.xml
            displayName: 'Run tests'

          - task: PublishTestResults@2
            condition: succeededOrFailed()
            inputs:
              testResultsFiles: '**/test-results.xml'
              testRunTitle: 'Python Tests'

          - task: Docker@2
            displayName: 'Build image'
            inputs:
              command: build
              repository: $(imageRepository)
              dockerfile: $(dockerfilePath)
              containerRegistry: $(dockerRegistryServiceConnection)
              tags: |
                $(tag)
                latest

          - task: Docker@2
            displayName: 'Push to registry'
            inputs:
              command: push
              repository: $(imageRepository)
              containerRegistry: $(dockerRegistryServiceConnection)
              tags: |
                $(tag)
                latest

  - stage: Deploy_Dev
    displayName: 'Deploy to Dev'
    dependsOn: Build
    condition: succeeded()
    jobs:
      - deployment: Deploy
        displayName: 'Deploy to Dev AKS'
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'dev'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  displayName: 'Deploy manifest'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'aks-dev'
                    namespace: 'production'
                    manifests: |
                      $(Pipeline.Workspace)/manifests/deployment.yml
                      $(Pipeline.Workspace)/manifests/service.yml
                    containers: |
                      $(containerRegistry)/$(imageRepository):$(tag)

                - script: |
                    kubectl rollout status deployment/myapp -n production --timeout=5m
                  displayName: 'Wait for rollout'

  - stage: Deploy_Staging
    displayName: 'Deploy to Staging'
    dependsOn: Deploy_Dev
    condition: succeeded()
    jobs:
      - deployment: Approval
        displayName: 'Await approval'
        pool: server
        environment: 'staging'

      - deployment: Deploy
        displayName: 'Deploy to Staging'
        dependsOn: Approval
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  displayName: 'Deploy'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'aks-staging'
                    namespace: 'production'
                    manifests: |
                      $(Pipeline.Workspace)/manifests/deployment.yml

  - stage: Deploy_Production
    displayName: 'Deploy to Production'
    dependsOn: Deploy_Staging
    condition: succeeded()
    jobs:
      - deployment: ApprovalProd
        displayName: 'Await 2 approvals'
        pool: server
        environment: 'production'
        strategy:
          runOnce:
            preDeploy:
              steps:
                - script: echo 'Waiting for approvals from 2 senior engineers'

      - deployment: Deploy
        displayName: 'Blue-Green Deploy'
        dependsOn: ApprovalProd
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: |
                    # Blue-green deployment
                    kubectl set image deployment/myapp-blue \
                      myapp=$(containerRegistry)/$(imageRepository):$(tag) \
                      -n production
                    
                    # Wait for ready
                    kubectl rollout status deployment/myapp-blue \
                      -n production --timeout=10m
                    
                    # Switch traffic (via service selector)
                    kubectl patch service myapp \
                      -p '{"spec":{"selector":{"version":"blue"}}}' \
                      -n production
                  displayName: 'Blue-Green Switch'

                - script: |
                    # Smoke tests
                    APP_IP=$(kubectl get svc myapp -n production \
                      -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                    
                    curl -f http://${APP_IP}/health
                    curl -f http://${APP_IP}/api/status
                  displayName: 'Run smoke tests'

DEPLOYMENT STRATEGIES:

1. Blue-Green (Zero Downtime)
   ├─ Blue: Current version (100% traffic)
   ├─ Green: New version (0% traffic)
   ├─ Test green thoroughly
   ├─ Switch traffic from blue to green (instant)
   ├─ Keep blue as rollback
   └─ Rollback: Switch back to blue

2. Canary (Gradual Rollout)
   ├─ Current: 100% traffic
   ├─ New: 0% traffic
   ├─ Increment new: 5% → 10% → 25% → 50% → 100%
   ├─ Monitor metrics at each step
   ├─ If error rate spikes: Stop, rollback
   └─ Advantages: Catch issues with % of users

3. Rolling Update (Kubernetes Default)
   ├─ Update pods one by one
   ├─ 25% new, 75% old
   ├─ 50% new, 50% old
   ├─ 75% new, 25% old
   ├─ 100% new
   ├─ Advantage: Resource efficiency
   └─ Disadvantage: Mixed versions briefly

TERRAFORM FOR CANARY:

resource "azurerm_kubernetes_cluster_network_profile" "main" {
  # Istio for traffic management
  service_mesh = "Istio"
}

# Use Istio VirtualService for canary:
resource "kubernetes_manifest" "canary" {
  manifest = {
    apiVersion = "networking.istio.io/v1beta1"
    kind       = "VirtualService"
    metadata = {
      name      = "myapp"
      namespace = "production"
    }
    spec = {
      hosts = ["myapp.prod.svc.cluster.local"]
      http = [
        {
          match = [{ sourceLabels = { version = "canary" } }]
          route = [{ destination = { host = "myapp", subset = "v2" } }]
        },
        {
          route = [
            {
              destination = { host = "myapp", subset = "v1" }
              weight      = 90
            },
            {
              destination = { host = "myapp", subset = "v2" }
              weight      = 10  # 10% canary
            }
          ]
        }
      ]
    }
  }
}
```

---

## GitOps Patterns

### Q: Implement GitOps for AKS with ArgoCD

```
GITOPS PRINCIPLE:

"Git is source of truth"
├─ Desired state: Defined in Git repo
├─ ArgoCD: Continuously syncs cluster to Git
├─ Any drift: Automatically corrected
└─ All changes: Auditable, reversible

WORKFLOW:

1. Developer commits deployment manifest
   $ git commit -m "Deploy app v2.0"
   └─ Pushed to main branch

2. ArgoCD detects change
   ├─ Polls Git repo (every 3 min)
   ├─ Sees new manifest
   └─ Deployment definition changed

3. ArgoCD reconciles
   ├─ Compares desired (Git) vs actual (cluster)
   ├─ Creates/updates deployment
   ├─ Scales replicas
   └─ Updates image version

4. ArgoCD monitors
   ├─ Continuously watches cluster
   ├─ If someone manually deletes pod:
   │  └─ ArgoCD recreates it (within 3 min)
   ├─ If image pulled old version:
   │  └─ ArgoCD fixes it
   └─ Keeps cluster in sync with Git

ARGOCD SETUP:

# Add ArgoCD Helm repo
resource "helm_release" "argocd" {
  name             = "argocd"
  repository       = "https://argoproj.github.io/argo-helm"
  chart            = "argo-cd"
  namespace        = "argocd"
  create_namespace = true

  values = [
    file("${path.module}/argocd-values.yaml")
  ]
}

# ArgoCD Application (Git as source)
resource "kubernetes_manifest" "argocd_app" {
  manifest = {
    apiVersion = "argoproj.io/v1alpha1"
    kind       = "Application"
    metadata = {
      name      = "myapp"
      namespace = "argocd"
    }
    spec = {
      project = "default"
      source = {
        repoURL        = "https://github.com/myorg/app-configs"
        targetRevision = "main"
        path           = "k8s/production"
      }
      destination = {
        server    = "https://kubernetes.default.svc"
        namespace = "production"
      }
      syncPolicy = {
        automated = {
          prune   = true      # Delete resources removed from Git
          selfHeal = true     # Correct drift automatically
          allowEmpty = false
        }
        syncOptions = [
          "CreateNamespace=true"
        ]
      }
    }
  }
}

Git Repo Structure:
├─ k8s/
│  ├─ production/
│  │  ├─ kustomization.yaml
│  │  ├─ deployment.yaml
│  │  ├─ service.yaml
│  │  └─ configmap.yaml
│  │
│  └─ staging/
│     ├─ kustomization.yaml
│     └─ deployment.yaml
│
├─ helm/
│  └─ charts/
│     └─ myapp/
│        ├─ Chart.yaml
│        ├─ values.yaml
│        └─ templates/
│
└─ README.md

DEPLOYMENT UPDATE VIA GIT:

Developer wants to deploy v2.0 to production:

1. Modify manifest in repo
   ```yaml
   # k8s/production/deployment.yaml
   image: myapp:v1.9 → v2.0  # Update
   replicas: 3 → 4           # Scale up
   ```

2. Commit and push
   $ git commit -m "Deploy myapp v2.0"
   $ git push origin main

3. ArgoCD detects change
   ├─ Webhook notification (instant)
   └─ Or polling detects change (within 3 min)

4. ArgoCD reconciles
   ├─ Updates deployment.yaml in cluster
   ├─ Kubernetes starts new pods with v2.0
   ├─ Old pods are terminated
   └─ Deployment complete (rolling update)

5. Rollback (if needed)
   ```yaml
   # Revert Git commit
   image: myapp:v2.0 → v1.9
   ```
   ArgoCD automatically reverts cluster

SECURITY:

GitOps secures deployments:
├─ No direct kubectl access needed
├─ All changes in Git = Audit trail
├─ RBAC on Git repo (who can commit)
├─ ArgoCD uses service account (limited perms)
├─ No secrets in Git (use Secret Store CSI)
└─ Signed commits (verify author)
```

---

## Interview Scenarios

**Q: Design CI/CD for microservices (10 services, independent releases)**

```
Requirements:
- Each service deploys independently
- Shared base pipeline
- Automatic versioning
- Canary deployments

Solution:

1. Shared Pipeline Template
   └─ Reusable steps for all services

2. Service-Specific Variables
   └─ Each service defines: registry, AKS cluster, namespace

3. Trigger: Service changes
   ├─ Service A commits → Deploy Service A only
   ├─ Service B commits → Deploy Service B only
   └─ No unnecessary deployments

Example:
# pipelines/service-pipeline.yml (template)
parameters:
  - name: serviceName
    type: string
  - name: servicePort
    type: number

stages:
  - stage: Build
    jobs:
      - job: BuildService
        steps:
          - task: Docker@2
            inputs:
              command: 'build'
              Dockerfile: 'services/${{ parameters.serviceName }}/Dockerfile'
              repository: '${{ parameters.serviceName }}'
              tags: '$(Build.BuildId)'

  - stage: Deploy
    jobs:
      - deployment: DeployService
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  inputs:
                    manifests: |
                      k8s/${{ parameters.serviceName }}/deployment.yml
                    containers: |
                      acr.azurecr.io/${{ parameters.serviceName }}:$(Build.BuildId)

# For each service, create azure-pipelines.yml
# services/payment-service/azure-pipelines.yml
extends:
  template: pipelines/service-pipeline.yml
  parameters:
    serviceName: payment-service
    servicePort: 8080

# services/auth-service/azure-pipelines.yml
extends:
  template: pipelines/service-pipeline.yml
  parameters:
    serviceName: auth-service
    servicePort: 9000
```

This covers production CI/CD patterns for AKS and multi-service deployments.
