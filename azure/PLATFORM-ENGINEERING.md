# Platform Engineering & Internal Developer Platforms

> Building self-service platforms, golden paths, DevEx - 5% of questions

**Coverage:** 60+ scenarios | **Focus:** Developer experience, automation, standardization

---

## Platform Engineering Fundamentals

### Q: Design internal developer platform for 200 engineers

**Requirements:**
- 200 software engineers (frontend, backend, data, mobile)
- Multiple teams: web, mobile, data, payments, ML
- ~100 microservices
- Different languages: Go, Python, Node.js, Java
- Minimal DevOps knowledge needed

**Solution:**

```
INTERNAL DEVELOPER PLATFORM (IDP):

Core Components:
├─ 1. Standardized Deployments
│  ├─ "Deploy" command → automatic AKS deployment
│  ├─ No kubectl knowledge needed
│  ├─ Opinionated defaults (but overridable)
│  └─ Zero-trust security baseline
│
├─ 2. Golden Paths
│  ├─ Web service template (Go best practices)
│  ├─ API service template (Python best practices)
│  ├─ Data pipeline template (Kubernetes Airflow)
│  ├─ Frontend template (Next.js, Terraform config)
│  └─ Developers choose template, fill in details
│
├─ 3. Self-Service Capabilities
│  ├─ Create database (Terraform apply via UI)
│  ├─ Create storage bucket (auto permission setup)
│  ├─ Create CI/CD pipeline (from template)
│  ├─ Request secrets in Key Vault
│  └─ Provision monitoring/alerts
│
├─ 4. Security Guardrails
│  ├─ Pod Security Standards enforced
│  ├─ Network policies auto-generated
│  ├─ RBAC pre-configured for team
│  ├─ Secrets not accessible to developers
│  └─ Audit logging automatic
│
└─ 5. Developer Portal
   ├─ Service registry (all services, owners, dependencies)
   ├─ Runbooks (how to scale, how to debug)
   ├─ Dashboards (team's services at a glance)
   ├─ Cost per service (transparency)
   ├─ On-call schedules
   └─ Incident postmortems

ARCHITECTURE:

┌──────────────────────────────────────────────┐
│        Developer Portal (Web UI)              │
│                                              │
│  [Deploy] [Create Database] [View Logs]     │
│  [Request Secrets] [Scale Service]          │
└────────────────┬─────────────────────────────┘
                 │
                 ├─ Internal API
                 │  ├─ /services (list services)
                 │  ├─ /deploy (trigger deployment)
                 │  ├─ /database/create
                 │  └─ /logs/stream
                 │
                 ├─ GitHub Actions
                 │  ├─ Test on PR
                 │  ├─ Build on merge
                 │  └─ Deploy to staging
                 │
                 ├─ CI/CD Pipeline
                 │  ├─ Run tests
                 │  ├─ Build container
                 │  ├─ Push to ACR
                 │  └─ Deploy to AKS
                 │
                 ├─ Terraform
                 │  ├─ Create resources
                 │  ├─ Manage configuration
                 │  └─ Scale infrastructure
                 │
                 └─ Kubernetes (AKS)
                    ├─ Run containers
                    ├─ Auto-scale pods
                    ├─ Manage networking
                    └─ Handle failures

DEVELOPER WORKFLOW (Simplified):

Day 1: Developer writes web service

$ git clone company-templates
$ template new web-service
  → Generates: 
    ├─ main.go (Hello World)
    ├─ Dockerfile
    ├─ k8s/deployment.yaml
    ├─ k8s/service.yaml
    └─ azure-pipelines.yml

$ git commit -m "Init service"
$ git push

GitHub Actions (automatic):
├─ Run tests: `go test ./...` ✓
├─ Build: `docker build .` ✓
├─ Push: ACR ✓
└─ Deploy to staging cluster ✓

Developer can test in staging (automated)

Day 2: Deploy to production (1-click)

[Staging] Service runs: api.staging.company.com
Developer clicks: [Deploy to Prod]

System automatically:
├─ Creates prod k8s namespace (if needed)
├─ Configures RBAC for team
├─ Sets up monitoring/alerts
├─ Creates DNS entry (api.prod.company.com)
├─ Enables auto-scaling (2-10 replicas)
├─ Blue-green deploys (zero downtime)
└─ Runs smoke tests

Result: Service running in production ✓

GOLDEN PATH EXAMPLE (Web Service Template):

```go
// main.go generated from template
package main

import (
  "fmt"
  "net/http"
  "os"
)

func handleHealth(w http.ResponseWriter, r *http.Request) {
  // Liveness/readiness probe
  w.Header().Set("Content-Type", "application/json")
  w.WriteHeader(http.StatusOK)
  fmt.Fprintf(w, `{"status":"healthy"}`)
}

func main() {
  http.HandleFunc("/health", handleHealth)
  http.HandleFunc("/", handleHome)
  
  port := os.Getenv("PORT")
  if port == "" {
    port = "8080"
  }
  
  fmt.Printf("Starting server on port %s\n", port)
  http.ListenAndServe(":"+port, nil)
}
```

```dockerfile
# Dockerfile (optimized for production)
FROM golang:1.20 AS builder
WORKDIR /build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o app main.go

FROM gcr.io/distroless/base:nonroot  # Minimal, secure base image
COPY --from=builder /build/app /app
USER nonroot
ENTRYPOINT ["/app"]
```

```yaml
# k8s/deployment.yaml (auto-generated)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{SERVICE_NAME}}
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: {{SERVICE_NAME}}
  template:
    metadata:
      labels:
        app: {{SERVICE_NAME}}
      annotations:
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: {{SERVICE_NAME}}
      containers:
      - name: {{SERVICE_NAME}}
        image: {{REGISTRY}}/{{SERVICE_NAME}}:{{BUILD_ID}}
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: SERVICE_NAME
          value: {{SERVICE_NAME}}
        - name: ENVIRONMENT
          value: production
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

PLATFORM BENEFITS:

For Developers:
├─ 15 min to deploy first service (vs 2 days manually)
├─ No Kubernetes/DevOps knowledge required
├─ Self-service: Request database, scale, debug
├─ Faster feedback (staging in 5 min)
├─ Clear runbooks (how to handle production issues)
└─ Focus on business logic, not infrastructure

For Operations:
├─ Standardization (all services follow same pattern)
├─ Compliance/security by default
├─ Visibility: Service registry, dependencies
├─ Reduced toil: Less manual deployments
├─ Knowledge codified: Runbooks, best practices
└─ Lower on-call burden (standardized services easier to debug)

For Company:
├─ Velocity: Deploy 10x faster
├─ Reliability: Standardized = fewer bugs
├─ Cost: Efficient resource usage, less manual work
├─ Scalability: Add new services/engineers easily
└─ Hiring: "We have great DX" attracts talent

COST:

Platform maintenance:
├─ 1 Platform Engineer: $180K/year
├─ + Cloud infrastructure: $50K/year
└─ Total: ~$230K/year

Benefit per engineer:
├─ Saves 30 min/day on deployment + debugging
├─ 200 engineers × 30 min × 250 working days
├─ = 2,500 person-days/year saved
├─ = $3M in reduced labor (@ $120K salary)
└─ ROI: 13x (saves $2.77M on $230K investment)

PLATFORM TOOLS (Example Stack):

├─ Service Registry: Backstage (Spotify)
├─ Templates: Cookiecutter + GitHub templates
├─ CI/CD: GitHub Actions + Azure Pipelines
├─ Infrastructure: Terraform
├─ Deployment: ArgoCD (GitOps)
├─ Observability: Datadog / New Relic
├─ Secrets: Azure Key Vault
├─ Compute: AKS
└─ Portal: Custom Node.js app OR Backstage
```

---

## Advanced Patterns

### Q: Implement cost optimization platform

**Features:**
- Auto-shutdown unused resources (dev/test)
- Right-size recommendations (VMs too large)
- Spot instance recommendations
- Reserved capacity forecasting
- Chargeback by team (cost attribution)
- Alert when team exceeds budget

**Result:**
- 30-40% reduction in cloud spend
- Better resource utilization
- Team accountability for costs

This covers platform engineering for DevEx and operational efficiency.
