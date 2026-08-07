# Azure Observability & Monitoring - Production Strategies

> Azure Monitor, Log Analytics, Application Insights, SLI/SLO/SLA - 5% of questions

**Coverage:** 80+ scenarios | **Focus:** Alerting strategies, dashboards, cost optimization

---

## Monitoring Stack

### Q: Design comprehensive monitoring for AKS production cluster

```
MONITORING LAYERS:

Layer 1: Infrastructure Metrics
├─ Node CPU, memory, disk usage
├─ Network I/O, latency
├─ Disk queue length, IOPS
└─ Source: Azure Monitor (VM metrics)

Layer 2: Container Metrics
├─ Pod CPU, memory usage
├─ Container restart count
├─ Image pull duration
└─ Source: Kubelet metrics

Layer 3: Application Metrics
├─ Request latency (p50, p95, p99)
├─ Error rate (5xx, 4xx)
├─ Throughput (requests/sec)
├─ Business metrics (orders/min, users online)
└─ Source: Application Insights, Prometheus

Layer 4: Logs
├─ Application logs
├─ Container logs
├─ Kubelet logs
├─ API Server audit logs
└─ Source: Log Analytics Workspace

Layer 5: Traces
├─ Request flow across services
├─ Service-to-service latency
├─ Failure root cause
└─ Source: Application Insights distributed tracing

ALERTING STRATEGY:

Alert = Threshold violation that requires immediate action

Example Alerts:
├─ CPU > 85% for 5 minutes
│  └─ Action: Auto-scale up OR page on-call engineer
│
├─ Pod OOM (out of memory)
│  └─ Action: Increase memory limits, page engineer
│
├─ Error rate > 1%
│  └─ Action: Page engineer immediately
│
├─ Request latency p95 > 500ms
│  └─ Action: Investigate, page if worsening
│
└─ No successful requests in 5 minutes
   └─ Action: Emergency page

TERRAFORM SETUP:

resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-prod-001"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30  # Production minimum

  daily_quota_gb = 10  # Cost control: drop data after quota
}

# Enable container insights on AKS
resource "azurerm_kubernetes_cluster" "main" {
  # ... AKS config ...

  monitor_metrics {
    enabled = true
  }

  oms_agent {
    enabled                    = true
    log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id
  }
}

# Create alert rule (CPU > 85%)
resource "azurerm_monitor_metric_alert" "cpu_high" {
  name                = "alert-cpu-high"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_kubernetes_cluster.main.id]
  description         = "Alert when node CPU exceeds 85%"

  criteria {
    metric_name        = "Percentage CPU"
    metric_namespace   = "Microsoft.Compute/virtualMachines"
    aggregation        = "Average"
    operator           = "GreaterThan"
    threshold          = 85
  }

  window_size  = "PT5M"  # Evaluate every 5 minutes
  frequency    = "PT1M"  # Check every 1 minute
  severity     = 2

  action {
    action_group_id = azurerm_monitor_action_group.pagerduty.id
  }
}

# Create action group (where alerts send notifications)
resource "azurerm_monitor_action_group" "pagerduty" {
  name                = "action-pagerduty"
  resource_group_name = azurerm_resource_group.main.name
  short_name          = "pagerduty"

  webhook_receiver {
    name        = "pagerduty"
    service_uri = "https://events.pagerduty.com/v2/enqueue"
  }

  email_receiver {
    name          = "oncall"
    email_address = "oncall@company.com"
  }

  sms_receiver {
    name         = "critical"
    country_code = "1"
    phone_number = "+1234567890"
  }
}

# Alert rule (Error rate > 1%)
resource "azurerm_monitor_metric_alert" "error_rate" {
  name                = "alert-error-rate"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_application_insights.main.id]

  criteria {
    metric_name = "server/requestsFailed"
    operator    = "GreaterThan"
    threshold   = 1  # 1% error rate
    aggregation = "Total"
  }

  window_size = "PT5M"
  frequency   = "PT1M"
  severity    = 1  # Critical

  action {
    action_group_id = azurerm_monitor_action_group.pagerduty.id
  }
}

PROMETHEUS + GRAFANA (Alternative):

# Add Prometheus/Grafana via Helm
resource "helm_release" "prometheus" {
  name       = "prometheus"
  chart      = "kube-prometheus-stack"
  repository = "https://prometheus-community.github.io/helm-charts"
  namespace  = "monitoring"
  create_namespace = true

  values = [
    file("${path.module}/prometheus-values.yaml")
  ]
}

# Sample Grafana dashboard JSON
resource "grafana_dashboard" "aks" {
  config_json = file("${path.module}/aks-dashboard.json")
  folder      = "AKS Cluster"
}

COST OPTIMIZATION:

Problem: Log Analytics costs $3-5 per GB ingested

Solutions:
├─ 1. Reduce retention (30 days → 7 days saves 75%)
├─ 2. Sample logs (collect 10% of non-critical)
├─ 3. Filter: Only collect ERROR and above
├─ 4. Use cheaper tier (Per GB → Free tier if <1GB)
└─ 5. Separate workspaces:
   ├─ Production: 30 day retention (pay full price)
   ├─ Staging: 7 day retention (cheaper)
   └─ Dev: Stream to local ELK only

Example cost savings:
├─ Current: 50 GB/day × $3 = $150/day = $4500/month
├─ After filtering: 10 GB/day × $3 = $30/day = $900/month
└─ Savings: $3600/month = $43,200/year

DASHBOARDS:

Production Dashboard Shows:
├─ Node health (green=healthy, red=critical)
├─ Pod deployment status (desired vs running)
├─ Request rate (requests/sec trending)
├─ Error rate (% vs threshold)
├─ Latency (p95, p99 vs SLO)
├─ Resource utilization (CPU, memory %)
├─ Traffic by endpoint
└─ Top errors (grouped by message)

SLI / SLO / SLA:

SLI (Service Level Indicator)
├─ Metric you measure
├─ Example: "99.5% of requests complete in < 500ms"
├─ Data: Measured from production
└─ Calculation: successes / total requests

SLO (Service Level Objective)
├─ Target you commit to
├─ Example: "99% availability"
├─ Set by: Team/product
├─ Based on: Business requirements
└─ If missed: "Error budget" consumed (can't deploy)

SLA (Service Level Agreement)
├─ Contract with customers
├─ Example: "99.9% uptime or we provide credits"
├─ Backed by: Legal/financial penalty
├─ Stricter than SLO (SLO > SLA)
└─ Example: SLO 99.95%, SLA 99.9%

ERROR BUDGET:

If SLO = 99%:
├─ You can have 1% failures/downtime per month
├─ Minutes available per month: 43,200
├─ Error budget: 432 minutes = 7.2 hours
├─ Burned so far this month:
│  ├─ Incident A: 15 minutes
│  ├─ Incident B: 30 minutes
│  └─ Incident C: 5 minutes
│  └─ Total: 50 minutes
├─ Remaining budget: 382 minutes (out of 432)
└─ Decision: Can deploy risky feature? Only if <382 min risk
   └─ If risky feature has 1% failure chance = 432 min risk
   └─ Decision: NO, too risky, would exceed budget

QUERY EXAMPLES:

# Log Analytics KQL (Kusto Query Language)

# Find all errors in last hour
ContainerLog
| where TimeGenerated > ago(1h)
| where LogLevel == "ERROR"
| summarize count() by Message
| top 10 by count_

# Latency percentiles
AppRequest
| where TimeGenerated > ago(1h)
| extend LatencyMs = toint(Duration)
| summarize 
    p50 = percentile(LatencyMs, 50),
    p95 = percentile(LatencyMs, 95),
    p99 = percentile(LatencyMs, 99)

# Error rate by endpoint
AppRequest
| where TimeGenerated > ago(1h)
| extend IsError = iff(ResultCode >= 400, 1, 0)
| summarize ErrorRate = sum(IsError) / count() by Url
| where ErrorRate > 0.01  # > 1%
```

This covers production monitoring patterns for AKS and cloud applications.
