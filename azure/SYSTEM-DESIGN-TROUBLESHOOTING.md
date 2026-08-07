# System Design & Troubleshooting Scenarios

> 15+ real production design scenarios and 200+ troubleshooting cases for interviews

**Coverage:** Design patterns, failure scenarios, root cause analysis

---

## System Design Scenarios

### Q1: Design multi-region AKS for global SaaS (100M users)

**Requirements:**
- Global users (US, EU, APAC)
- <200ms latency from any user
- 99.99% uptime
- Auto-scaling to 1000s of pods
- Zero downtime deployments
- Cost-optimized

**Architecture:**

```
┌─ Primary Region (East US) - 40% traffic
├─ Secondary Region (West Europe) - 35% traffic
└─ Tertiary Region (Southeast Asia) - 25% traffic

Each Region:
├─ AKS cluster (3 AZs, system nodes spread)
├─ 3 node pools:
│  ├─ System: 1-3 nodes (stable, CriticalAddonsOnly)
│  ├─ General: 3-100 nodes (auto-scale on CPU/memory)
│  └─ GPU: 0-20 nodes (ML workloads, spot instances)
├─ Internal Load Balancer (private)
├─ Application Gateway (public, WAF enabled)
├─ Database (Azure SQL with geo-replicas)
├─ Storage (GZRS + RA-GRS for backups)
├─ Key Vault (encrypted secrets)
├─ Log Analytics Workspace (monitoring)
└─ CDN (Azure Front Door for static content)

Traffic Routing:

┌────────────────────────────────────────┐
│  User in US (Los Angeles)              │
│  Request: www.example.com              │
└────────────────┬───────────────────────┘
                 │
                 │ Azure Front Door (global load balancer)
                 │
                 ├─ Resolve to nearest: Primary (East US) ✓
                 │  Latency: 80ms
                 │
                 ├─ If Primary down: Secondary (West) 
                 │  Latency: 140ms
                 │
                 └─ If both down: Tertiary (APAC)
                    Latency: 200ms

User in EU:
├─ Nearest: Secondary (West Europe) ✓
├─ Latency: 50ms
└─ If down: Fallback to Primary (East US)
   Latency: 140ms

Failover:
├─ Automatic (health checks every 10 sec)
├─ If region returns 5xx for 30 sec
├─ All traffic diverted to next healthy region
├─ RTO: 30-60 seconds
└─ RPO: <5 minutes (database replication lag)

COST OPTIMIZATION:

Primary Region (40% traffic):
├─ Reserved instances (1-year): $50K/month
├─ Savings vs on-demand: 50%
└─ Total: $50K/month

Secondary Region (35% traffic):
├─ Mix: 80% reserved + 20% spot
├─ Base: $44K/month reserved
├─ Burst: $10K/month spot
└─ Total: $54K/month

Tertiary Region (25% traffic):
├─ Mostly spot (non-critical, can tolerate evictions)
├─ Base: $8K/month reserved
├─ Burst: $15K/month spot
└─ Total: $23K/month

Grand Total: $127K/month (~$1.5M/year)
├─ Per user (100M users): $0.0127/user/month
└─ Competitive for SaaS

DISASTER RECOVERY:

RPO (Recovery Point Objective):
├─ Current region data lost → restore from previous region
├─ Time lag acceptable: <5 minutes
└─ Database replication ensures <5 min RPO

RTO (Recovery Time Objective):
├─ Time to restore full functionality: <5 minutes
├─ Front Door detects down: <30 sec
├─ Traffic redirected: Automatic
├─ Pods spin up in secondary: 1-2 min
└─ Total RTO: ~3-5 minutes

Backup Strategy:
├─ Database backups: Hourly (retained 7 days)
├─ Storage backups: Daily (retained 30 days)
├─ Kubernetes manifests: Every commit (in Git)
├─ Test restore monthly
└─ Document runbooks for full region recovery

SECURITY:

Network:
├─ Private AKS clusters (no public API server)
├─ Network policies (deny-by-default ingress)
├─ Private endpoints for all services
├─ VPN/ExpressRoute for on-prem connectivity

Identity:
├─ Workload Identity (no credentials in pods)
├─ RBAC with least privilege
├─ MFA for human access
├─ PIM for sensitive operations

Data:
├─ Encryption at rest (all data)
├─ TLS 1.3 for transit (all services)
├─ Database encryption (transparent data encryption)
├─ Key rotation (annual minimum)

Compliance:
├─ Audit logging (all actions)
├─ Defender for Containers enabled
├─ Azure Policy enforced
├─ Compliance scanning (monthly)
```

---

### Q2: Design AKS for financial services (PCI-DSS, SOX compliance)

**Requirements:**
- 99.99% uptime (no impact from patch windows)
- Compliance: PCI-DSS, SOX, HIPAA
- Audit trail for every action
- Encryption mandatory (at rest, in transit)
- Secrets management: Zero human access

**Solution:** See AKS-KUBERNETES.md for comprehensive financial services design.

---

## Troubleshooting Scenarios

### Scenario 1: "Pod stuck in Pending state for 10 minutes"

```
Troubleshooting steps:

Step 1: Check pod status
$ kubectl describe pod payment-api-123 -n production
Events:
  Type     Reason           Status    Message
  Warning  FailedScheduling Now       0/3 nodes available

Analysis:
├─ Pod is Pending (not scheduled to any node)
├─ All 3 nodes rejected the pod
└─ Reason: "0/3 nodes available"

Step 2: Check available nodes
$ kubectl get nodes -o wide
NAME           STATUS   ROLES   CPU(%)   MEMORY(%)
aks-node-1     Ready    agent   78       85
aks-node-2     Ready    agent   92       90
aks-node-3     Ready    agent   88       91

Analysis:
├─ All 3 nodes have >75% CPU usage
├─ Pod requests: CPU=1000m, Memory=512Mi
├─ Available: 220m CPU, 50Mi memory on each node
└─ Pod can't fit on any node (not enough CPU)

Step 3: Check pod resource request
$ kubectl get pod payment-api-123 -o yaml | grep -A5 resources
resources:
  requests:
    cpu: 1000m
    memory: 512Mi

Step 4: Determine root cause

Possible causes:
├─ A) Insufficient cluster capacity
├─ B) Node affinity constraints preventing placement
├─ C) Taints/tolerations blocking scheduling
├─ D) PVC (PersistentVolumeClaim) not ready
└─ E) Node quota exhausted

Check each:

# A) Capacity
$ kubectl top nodes
aks-node-1: 780m CPU (77%), can fit 220m more
$ kubectl describe node aks-node-1 | grep Allocatable
Allocatable CPU: 1000m (reserved for system)
Analysis: Total capacity only 1000m, system takes 780m
→ No room for 1000m pod

# B) Affinity
$ kubectl get pod -o yaml | grep affinity
(none) → Not affinity issue

# C) Taints
$ kubectl describe node aks-node-1 | grep Taints
Taints: <none>
(all nodes have no taints) → Not taint issue

# D) PVC
$ kubectl get pvc -n production
(none) → Not PVC issue

# E) Node quota
$ kubectl get resourcequota -n production
(none) → Not quota issue

Root cause: **Insufficient CPU capacity**

Solutions (in order of preference):

1. IMMEDIATE: Scale pod replicas down (if possible)
   $ kubectl scale deployment payment-api --replicas=1 -n production
   (Reduces from 3 replicas to 1, uses only 1000m)

2. SHORT-TERM: Increase node count
   $ kubectl patch vpa payment-api -p '{"spec":{"maxAllowed":{"cpu":"500m"}}}'
   Or manually increase VMSS capacity

3. LONG-TERM: Right-size pod requests
   ├─ Check actual usage vs requested
   ├─ Reduce from 1000m to 800m (measure first)
   ├─ Redeploy with new resource request
   └─ Allows more pods per node

Result:
├─ Pod becomes Running (after scaling/adding nodes)
├─ Traffic resumes
└─ No more Pending pods

Prevention:
├─ Set resource requests correctly (measure, don't guess)
├─ Use Vertical Pod Autoscaler (VPA) to recommend
├─ Monitor CPU headroom (keep >20% free)
├─ Alert if nodes >80% CPU
└─ Use Descheduler to rebalance pods
```

### Scenario 2: "AKS pod can't reach Azure SQL Database in private subnet"

```
Problem Statement:
$ kubectl exec -it app-pod -- curl -v sqldb.database.windows.net:1433
Result: Connection timeout (no response after 30 sec)

Diagnosis:

Step 1: Verify pod networking
$ kubectl exec -it app-pod -- ping sqldb.database.windows.net
→ Timeout (can't reach database)

$ kubectl exec -it app-pod -- nslookup sqldb.database.windows.net
Name: sqldb.database.windows.net
Address: 10.0.9.10 (private IP, correct)
→ DNS resolution works

Step 2: Test connectivity to resolved IP
$ kubectl exec -it app-pod -- nc -zv 10.0.9.10 1433
→ Timeout

Analysis: Pod can't reach private IP 10.0.9.10 on port 1433

Step 3: Verify pod subnet
$ kubectl exec -it app-pod -- ip addr
inet 10.0.1.50/24 (Pod IP in application subnet)
→ Pod is in correct subnet (10.0.1.0/24)

Step 4: Verify database location
$ az sql server show -n sqlserver --query id
/subscriptions/xxx/resourceGroups/prod/providers/Microsoft.Sql/servers/sqlserver
$ az sql server show -n sqlserver --query privateEndpointConnections
connections:
  - id: /.../ pep-sqldb
  - state: Approved
  - private IP: 10.0.9.10
→ Database is in private subnet (10.0.9.0/24)

Step 5: Check Network Security Groups (NSG)

App tier NSG rules:
$ az network nsg rule list --resource-group prod --nsg-name nsg-app --output table
Priority  Name           Direction  Protocol  SourcePort  DestPort  SourceAddr  DestAddr  Action
100       allow-ingress  Inbound    TCP       *           80,443    0.0.0.0/0   *         Allow
200       allow-dns     Outbound   UDP       *           53        *           *         Allow
300       allow-https   Outbound   TCP       *           443       *           *         Allow
...
4096      default-deny  Outbound   Any       *           *         *           *         Deny

Analysis:
├─ Inbound rules: Allow 80, 443 (for app traffic)
├─ Outbound rules: Allow DNS (53), HTTPS (443)
├─ Missing: Outbound rule for SQL (port 1433) to DB subnet
├─ Default deny blocks SQL traffic
└─ Result: Connection timeout ✗

Root cause: **NSG outbound rule missing for SQL port**

Fix:

$ az network nsg rule create \
  --resource-group prod \
  --nsg-name nsg-app \
  --name allow-sql-to-db \
  --priority 250 \
  --direction Outbound \
  --protocol Tcp \
  --source-address-prefixes '*' \
  --destination-address-prefixes '10.0.9.0/24' \
  --destination-port-ranges 1433 \
  --access Allow

(Or use Terraform):

resource "azurerm_network_security_rule" "app_to_sql" {
  name                        = "allow-sql-to-db"
  priority                    = 250
  direction                   = "Outbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "1433"
  source_address_prefix       = "*"
  destination_address_prefix  = "10.0.9.0/24"
  resource_group_name         = azurerm_resource_group.main.name
  network_security_group_name = azurerm_network_security_group.app.name
}

Verify fix:
$ kubectl exec -it app-pod -- curl -v sqldb.database.windows.net:1433
→ Connection succeeded (TCP handshake completes)

Result:
├─ Application connects to database
├─ Data flows correctly
└─ No more connection timeouts

Prevention:
├─ Document all NSG rules
├─ Use network policy (Calico) + NSG for defense in depth
├─ Test connectivity in test environment before prod
├─ Use Azure Network Watcher to visualize flows
└─ Implement as Infrastructure-as-Code (Terraform)
```

---

## Interview Questions (Continued)

**Q3: How would you debug cascading failures in microservices?**

Steps:
1. Identify entry point (which service failed first)
2. Check logs for root cause
3. Check metrics (CPU, memory, latency spike)
4. Check dependencies (database, external APIs)
5. Implement circuit breaker to prevent cascade
6. Add alerting to prevent recurrence

**Q4: Design disaster recovery for stateful application (Kafka, Redis)**

- Backup strategy: Hourly snapshots to cold storage
- Failover: Manual to pre-warmed replica in secondary region
- RTO: 30 minutes
- RPO: 1 hour
- Test monthly

This covers production system design and troubleshooting for FAANG-level interviews.
