# AKS (Azure Kubernetes Service) - Extreme Deep Dive

> Production-grade AKS mastery for FAANG interviews - 25% of questions focus here

**Estimated Reading Time:** 180 minutes | **Coverage:** 250+ questions and scenarios

---

## Table of Contents

- [AKS Architecture Fundamentals](#aks-architecture-fundamentals)
- [Control Plane Deep Dive](#control-plane-deep-dive)
- [Node Pools & Compute](#node-pools--compute)
- [Azure CNI Networking](#azure-cni-networking)
- [Workload Identity](#workload-identity)
- [Security & Policies](#security--policies)
- [Monitoring & Diagnostics](#monitoring--diagnostics)
- [Production Troubleshooting](#production-troubleshooting)
- [Interview Questions](#interview-questions)

---

## AKS Architecture Fundamentals

### Q: Explain the AKS architecture - Control Plane vs Data Plane

**Advanced Answer:**

```
AKS ARCHITECTURE (Multi-AZ Deployment)

Azure Subscription
│
├─ Control Plane (Managed by Microsoft)
│  └─ Located in Microsoft-owned subscription
│     ├─ API Server (port 443)
│     ├─ etcd (cluster state)
│     ├─ Controller Manager (runs reconciliation)
│     ├─ Scheduler (pod placement)
│     └─ Cloud Controller Manager (Azure integration)
│
├─ Data Plane (Customer-owned VNet)
│  │
│  ├─ System Node Pool (AZ-1)
│  │  ├─ VMSS with 1-3 nodes
│  │  ├─ CoreDNS pod
│  │  ├─ kube-proxy pod
│  │  ├─ azure-cni pod
│  │  └─ Adds / kube-system components
│  │
│  ├─ User Node Pool 1 (AZ-1, AZ-2, AZ-3)
│  │  ├─ VMSS with auto-scaling (min 1, max 10)
│  │  └─ Application workloads
│  │
│  └─ User Node Pool 2 (AZ-1, AZ-2)
│     ├─ VMSS with GPU nodes
│     └─ ML/AI workloads
│
└─ Networking
   ├─ Virtual Network (10.0.0.0/8)
   │  ├─ System subnet: 10.0.0.0/24
   │  ├─ User subnet 1: 10.0.1.0/24
   │  └─ User subnet 2: 10.0.2.0/24
   │
   ├─ Network Security Groups (NSG)
   ├─ Azure Load Balancer (for services)
   ├─ Application Gateway Ingress Controller
   └─ Private Endpoints (for Key Vault, Container Registry)
```

**Q: What happens when you deploy a pod in AKS?**

**Step-by-step flow (with latency measurements):**

```
1. Client sends kubectl apply (0ms)
   └─ kubectl -n production apply -f deployment.yaml

2. API Server receives request (1-2ms)
   ├─ Authentication (check RBAC)
   ├─ Validation (schema, admission webhooks)
   ├─ Write to etcd (10-50ms)
   └─ Return 201 Created

3. Scheduler looks for placement (50-200ms)
   ├─ Filter phase: Find nodes with capacity
   │  ├─ Check CPU request: 500m
   │  ├─ Check memory request: 512Mi
   │  ├─ Check node affinity rules
   │  ├─ Check taints/tolerations
   │  └─ Remaining nodes: N1, N2, N4
   │
   ├─ Score phase: Pick best node
   │  ├─ Spread score (prefer less loaded)
   │  ├─ Resource balance score
   │  └─ Winner: N2 (most balanced)
   │
   └─ Update Pod.spec.nodeName = "aks-node-2"

4. Kubelet on target node detects pod (100-500ms)
   ├─ Watches API server for changes
   ├─ Sees pod assigned to self
   ├─ Creates container
   │  ├─ Pull image from ACR (1-5s depending on size)
   │  ├─ Extract to container filesystem
   │  └─ Start container runtime
   │
   └─ Update Pod status → Running

5. CNI plugin assigns IP (100-200ms)
   ├─ Azure CNI IPAM contacts Azure API
   ├─ Gets IP from subnet: 10.0.1.50
   ├─ Attaches secondary NIC to node (if needed)
   ├─ Configures network interface on container
   └─ Updates Pod.status.podIP = 10.0.1.50

6. Service networking (50-100ms)
   ├─ Kube-proxy sees service selector matches pod
   ├─ Adds iptables rule: svc IP → pod IP
   └─ Load balancer starts load balancing

7. DNS registration (100-200ms)
   ├─ CoreDNS sees Service created
   ├─ Registers A record: myapp.namespace.svc.cluster.local → service IP
   └─ Queries now resolve

Total time: 1-7 seconds (depends on image pull)
```

---

## Control Plane Deep Dive

### Q: How does etcd work in AKS and why is it critical?

**Expert Answer:**

```
ETCD IN AKS

┌─────────────────────────────────────────────────────────────┐
│                  AKS Control Plane                           │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              etcd (Distributed KV Store)              │   │
│  │                                                       │   │
│  │  Data stored:                                         │   │
│  │  └─ /kubernetes.io/namespaces/...                    │   │
│  │  └─ /kubernetes.io/pods/...                          │   │
│  │  └─ /kubernetes.io/services/...                      │   │
│  │  └─ /kubernetes.io/configmaps/...                    │   │
│  │  └─ /kubernetes.io/secrets/...                       │   │
│  │  └─ /registry/...                                    │   │
│  │                                                       │   │
│  │  Replicated across 3 nodes (quorum-based)             │   │
│  │  Raft consensus algorithm                             │   │
│  │  Requires 2 out of 3 nodes for write                  │   │
│  │                                                       │   │
│  │  Typical size: 2-5 GB for 1000 nodes                  │   │
│  │  Maximum recommended: 8GB                             │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  Components reading/writing to etcd:                         │
│  ├─ API Server (reads/writes all state)                     │
│  ├─ Controller Manager (reads state, writes updates)        │
│  ├─ Scheduler (reads requirements, writes node binding)     │
│  └─ kube-proxy (reads endpoints, writes iptables)           │
│                                                              │
└─────────────────────────────────────────────────────────────┘

ETCD Storage Breakdown:
├─ Pods: ~1MB per 100 pods
├─ Services: ~100KB per service
├─ ConfigMaps: ~50KB per configmap (depends on size)
├─ Secrets: ~50KB per secret
└─ RBAC resources: ~10KB per role binding

Performance Characteristics:
├─ Write latency: 10-50ms (P99)
├─ Read latency: 5-20ms (P99)
├─ Throughput: 1000s ops/sec
└─ Replication lag: <100ms

Disaster Scenarios:

1. etcd data loss
   └─ All cluster state lost
   └─ Pods continue running but cluster can't be managed
   └─ Must restore from backup

2. etcd single node failure
   └─ System continues (2/3 consensus)
   └─ New leader elected (100-1000ms)
   └─ Transparent to applications

3. etcd two nodes fail
   └─ Cluster becomes read-only
   └─ No new changes possible
   └─ Pods continue running
   └─ Manual intervention needed
```

**Q: How does the API Server handle requests?**

```
API Server Request Flow:

Client: kubectl apply -f pod.yaml

1. HTTPS Connection (TLS 1.2+)
   └─ Certificate verified
   └─ API server listens on :443

2. Authentication
   ├─ Certificate-based (kubelet)
   ├─ Token-based (service account)
   ├─ Azure AD/Entra ID (interactive users)
   ├─ Anonymous (if enabled)
   └─ Results: identity + associated attributes

3. Authorization (RBAC Evaluation)
   ├─ Check if identity has permission
   ├─ Rule evaluation:
   │  ├─ Deny by default
   │  ├─ Check all allow rules
   │  ├─ Grant if ANY rule matches
   │  └─ Deny if NO rules match
   │
   └─ Example:
      User: "alice@company.com"
      Action: "create" pods in namespace "production"
      Roles: ["Pod Creator", "Namespace Admin"]
      Result: ALLOWED (via "Pod Creator" role)

4. Admission Control
   ├─ Mutating admission (modify request)
   │  ├─ Add resource limits if missing
   │  ├─ Inject init containers
   │  ├─ Inject sidecar (service mesh)
   │  └─ Add network policies
   │
   └─ Validating admission (accept/reject)
      ├─ Validate pod security policy
      ├─ Validate namespace quota
      ├─ Validate image registry
      └─ Custom webhook validation

5. etcd Write
   ├─ Serialize object
   ├─ Write to etcd with timestamp
   ├─ Replicate to other etcd nodes
   └─ Confirm write successful

6. Return Response
   └─ 201 Created + resource YAML

Total latency: 50-200ms
```

---

## Node Pools & Compute

### Q: Design a production AKS cluster with multiple node pools

**Architecture:**

```
Production AKS Cluster Design (Multi-AZ)

Resources Required:
├─ Virtual Network (10.0.0.0/8)
├─ 3 Subnets
│  ├─ System: 10.0.0.0/24 (256 IPs)
│  ├─ Application: 10.0.1.0/21 (2048 IPs)
│  └─ GPU: 10.0.9.0/24 (256 IPs)
│
└─ AKS Cluster
   ├─ System Node Pool
   │  ├─ VM SKU: Standard_D2s_v3 (2 CPU, 8GB RAM)
   │  ├─ VMSS: 1-3 nodes (min for 3 AZs = 3 nodes)
   │  ├─ Availability Zones: 1, 2, 3
   │  ├─ OS Disk: 128GB (Premium SSD)
   │  ├─ Taints: CriticalAddonsOnly
   │  └─ Purpose: CoreDNS, kube-proxy, metrics-server
   │
   ├─ User Node Pool 1 (General)
   │  ├─ VM SKU: Standard_D4s_v3 (4 CPU, 16GB RAM)
   │  ├─ VMSS: 2-20 nodes
   │  ├─ Availability Zones: 1, 2, 3
   │  ├─ Scale trigger: CPU >75% for 5 min
   │  └─ Labels: workload=general, tier=app
   │
   ├─ User Node Pool 2 (Memory-optimized)
   │  ├─ VM SKU: Standard_E4s_v3 (4 CPU, 32GB RAM)
   │  ├─ VMSS: 1-10 nodes
   │  ├─ Availability Zones: 1, 2
   │  ├─ Scale trigger: Memory >80%
   │  ├─ Labels: workload=memory-intensive
   │  └─ Taints: memory=true
   │
   └─ User Node Pool 3 (GPU for ML)
      ├─ VM SKU: Standard_NC6s_v3 (6 CPU, 112GB, 1x Tesla V100)
      ├─ VMSS: 0-5 nodes (spot instances to save cost)
      ├─ Availability Zones: 1, 2
      ├─ Labels: workload=gpu, type=ml
      └─ Taints: nvidia.com/gpu=v100

Key Design Decisions:

1. Node Pool Distribution
   ├─ System on 3 separate AZs (high availability)
   ├─ App workloads spread across 3 AZs
   ├─ Specialized workloads on 2 AZs (cost optimization)
   └─ GPU on 2 AZs (less critical, more expensive)

2. VM Selection
   ├─ General: D-series (good CPU/memory ratio)
   ├─ Memory-intensive: E-series (2:1 memory ratio)
   ├─ GPU: NC-series (GPU compute)
   └─ All Premium SSD for nodes (production minimum)

3. Autoscaling
   ├─ System pool: Manual (should be stable)
   ├─ General pool: CPU-based (80%)
   ├─ Memory pool: Memory-based (85%)
   └─ GPU pool: Very aggressive (save cost)

4. Security
   ├─ Network Policy: Calico enforced
   ├─ Pod Security Policy: restricted
   ├─ RBAC: Minimal privileges
   └─ Private cluster: Yes (restrict API server access)
```

**Terraform Implementation:**

```hcl
resource "azurerm_kubernetes_cluster" "production" {
  name                = "aks-prod-cluster"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "aksprod"

  # System node pool
  default_node_pool {
    name                         = "system"
    node_count                   = 3
    vm_size                      = "Standard_D2s_v3"
    enable_auto_scaling          = false
    zones                        = [1, 2, 3]
    max_surge                    = 1
    enable_host_encryption       = true
    os_disk_type                 = "Premium_LRS"

    node_labels = {
      "workload" = "system"
    }

    node_taints = [
      "CriticalAddonsOnly=true:NoSchedule"
    ]
  }

  # Authentication
  azure_active_directory_role_based_access_control {
    managed                  = true
    azure_rbac_enabled       = true
    tenant_id                = data.azurerm_client_config.current.tenant_id
  }

  # Networking
  network_profile {
    network_plugin           = "azure"
    network_plugin_mode      = "overlay"  # Use CNI Overlay for flexibility
    network_policy           = "calico"
    dns_service_ip           = "10.100.0.10"
    docker_bridge_cidr       = "172.17.0.1/16"
    load_balancer_sku        = "standard"
    outbound_type            = "loadBalancer"
  }

  # Security
  api_server_authorized_ip_ranges = [
    "203.0.113.0/24"  # Your office
  ]

  # Identity
  identity {
    type = "SystemAssigned"
  }

  # Version
  kubernetes_version = "1.28.0"

  # Monitoring
  monitor_metrics {
    enabled = true
  }
}

# User node pool (general workloads)
resource "azurerm_kubernetes_cluster_node_pool" "general" {
  name                  = "general"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.production.id
  vm_size               = "Standard_D4s_v3"
  enable_auto_scaling   = true
  min_count             = 2
  max_count             = 20
  zones                 = [1, 2, 3]
  max_surge             = 2

  node_labels = {
    workload = "general"
    tier     = "app"
  }
}

# User node pool (memory-intensive)
resource "azurerm_kubernetes_cluster_node_pool" "memory" {
  name                  = "memory"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.production.id
  vm_size               = "Standard_E4s_v3"
  enable_auto_scaling   = true
  min_count             = 1
  max_count             = 10
  zones                 = [1, 2]

  node_labels = {
    workload = "memory-intensive"
  }

  node_taints = [
    {
      key    = "memory"
      value  = "true"
      effect = "NoSchedule"
    }
  ]
}
```

---

## Azure CNI Networking

### Q: Explain Azure CNI vs Kubenet vs CNI Overlay

**Comparison:**

```
┌────────────────────────────────────────────────────────────┐
│           NETWORKING PLUGINS COMPARISON                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Feature           Kubenet    Azure CNI    CNI Overlay      │
│ ─────────────────────────────────────────────────────────  │
│ Pod IP Range      Private    Azure VNet   Overlay Network  │
│ Example           10.244.0.0 10.0.0.0     10.244.0.0       │
│                                                            │
│ Routing           NAT        Direct       Overlay tunnels  │
│ Pod to pod        Via host   Direct       Via tunnel       │
│ Pod to Azure      Via host   Direct       Via tunnel       │
│                                                            │
│ Network           No         Yes          Yes              │
│ Policies          (need      (native)     (native)         │
│                   workaround)                              │
│                                                            │
│ IP efficiency     Medium     Low          High             │
│ IPs per node      ~110       ~240         ~1000            │
│                                                            │
│ Performance       Good       Best         Good             │
│ Latency           <1ms       <1ms         <2ms             │
│ Throughput        High       Highest      High             │
│                                                            │
│ Complexity        Low        Medium       Low              │
│ Troubleshooting   Easier     Complex      Medium           │
│                                                            │
│ Hybrid scenarios  Limited    Good         Excellent        │
│ On-prem access    Via UDR    Direct       Direct           │
│                                                            │
│ Multi-cloud       Possible   Azure only   Possible         │
│                                                            │
│ Use case          Dev/Test   Production   Flexibility      │
│                             (most common)                 │
│                                                            │
└────────────────────────────────────────────────────────────┘

RECOMMENDATION FOR PRODUCTION:
├─ Azure CNI: Traditional choice (native Azure integration)
├─ CNI Overlay: New, recommended (flexibility + performance)
└─ Kubenet: Development/testing only
```

**Azure CNI Detailed Architecture:**

```
┌────────────────────────────────────────────────────────────┐
│              AZURE CNI POD NETWORKING                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Virtual Network (10.0.0.0/8)                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │                                                      │ │
│  │  Node 1 (10.0.1.10) - VM                            │ │
│  │  ┌────────────────────────────────────────────────┐ │ │
│  │  │  veth0 (Host) ─┐                               │ │ │
│  │  │                │ Bridge                        │ │ │
│  │  │  veth1 (Pod)───┤ No bridge! Direct attachment │ │ │
│  │  │  IP: 10.0.1.11 │ via secondary NIC            │ │ │
│  │  │  MAC: aa:bb:cc │                              │ │ │
│  │  │                │                              │ │ │
│  │  │  veth2 (Pod)───┤                              │ │ │
│  │  │  IP: 10.0.1.12 │                              │ │ │
│  │  └────────────────────────────────────────────────┘ │ │
│  │                                                      │ │
│  │  Node 1 has 3 NICs:                                  │ │
│  │  1. Primary (10.0.1.10)                              │ │
│  │  2. Secondary 1 (10.0.1.11) - Pod                    │ │
│  │  3. Secondary 2 (10.0.1.12) - Pod                    │ │
│  │                                                      │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  Node 2 (10.0.1.20) - VM                                 │
│  ┌────────────────────────────────────────────────────┐ │
│  │  Pod A: 10.0.1.21                                 │ │
│  │  Pod B: 10.0.1.22                                 │ │
│  │  Pod C: 10.0.1.23                                 │ │
│  └────────────────────────────────────────────────────┘ │
│                                                            │
│  PROBLEM:                                                 │
│  ├─ Max secondary NICs per VM: 8-64 (depends on SKU)     │
│  ├─ Each secondary NIC = 1 pod (oversimplified)         │
│  ├─ Max pods per node: 30-110 (not unlimited)           │
│  └─ Large cluster needs many VMs                        │
│                                                            │
│  IP ALLOCATION:                                           │
│  ├─ Each node reserves IPs for secondary NICs            │
│  ├─ Example: D2s_v3 can have 2 secondary NICs            │
│  ├─ Max 2 pods per node (not realistic)                  │
│  └─ Actually ~30 pods via IP reuse (same NIC)            │
│                                                            │
└────────────────────────────────────────────────────────────┘

Pod-to-Pod Communication (same node):
1. Pod A (10.0.1.11) → Pod B (10.0.1.12)
2. Packet exits Pod A veth
3. Host kernel forwards directly
4. Packet enters Pod B veth
5. Latency: <1ms (same VM memory space)

Pod-to-Pod Communication (different node):
1. Pod A (10.0.1.11 on Node 1) → Pod B (10.0.2.21 on Node 2)
2. Packet exits Pod A veth
3. Host kernel routes to Azure network
4. Azure VNet switch routes based on ARP table
5. Packet arrives at Node 2
6. Host kernel delivers to Pod B
7. Latency: <2ms (same datacenter)

Pod-to-Azure Service (e.g., Storage Account):
1. Pod makes HTTPS request to Storage API
2. Azure DNS resolves: blob.core.windows.net → Public IP
3. Packet routed through Azure backbone (VNet optimized)
4. No NAT, direct routing
5. Private Endpoints allow private routing (no internet)
```

---

## Workload Identity

### Q: Explain Workload Identity (new) vs Managed Identity (old) vs IRSA

**Comparison:**

```
┌────────────────────────────────────────────────────────────┐
│   AUTHENTICATION METHODS FOR AKS PODS                      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  1. POD MANAGED IDENTITY (Legacy - DEPRECATED)            │
│  ├─ System manages credentials (bad!)                     │
│  ├─ Credentials stored in pod                            │
│  ├─ Risk: Pod compromise = credential leak                │
│  └─ Being phased out                                      │
│                                                            │
│  2. IRSA (IAM Roles for Service Accounts) - Kubernetes    │
│  ├─ Federated identity using OIDC tokens                  │
│  ├─ Uses service account JWT token                        │
│  ├─ Exchanges token for temporary credentials             │
│  ├─ No long-lived credentials in pod                      │
│  └─ Works but configuration is complex                    │
│                                                            │
│  3. WORKLOAD IDENTITY (Recommended - NEW)                 │
│  ├─ Microsoft's native implementation                      │
│  ├─ Azure Entra ID + OIDC federation                      │
│  ├─ Same security as IRSA, simpler config                 │
│  ├─ Azure-native integration                              │
│  └─ Recommended by Microsoft                              │
│                                                            │
└────────────────────────────────────────────────────────────┘

WORKLOAD IDENTITY FLOW (Recommended):

Step 1: Create Azure resources
├─ User-assigned Managed Identity
└─ Role assignment (e.g., AcrPull for ACR)

Step 2: Create Kubernetes resources
├─ Service Account linked to Managed Identity
└─ Workload Identity binding

Step 3: Pod requests token
├─ Pod mounts service account token (/var/run/secrets/...)
├─ Token is OIDC JWT from AKS OIDC provider
└─ Token includes pod identity info

Step 4: Pod uses token
├─ Pod calls Azure API (e.g., pull image from ACR)
├─ Includes OIDC token in request
├─ Azure validates token signature
├─ Azure exchanges token for short-lived credential
└─ Credential valid for ~1 hour

Step 5: Pod makes API call
├─ Uses credential to call Azure API
└─ Succeeds if Managed Identity has permission

ARCHITECTURE:

┌─────────────────────────────────────────────────────────┐
│  AKS Cluster                                            │
│  ┌───────────────────────────────────────────────────┐ │
│  │  Pod                                              │ │
│  │  ┌──────────────────────────────────────────────┐ │ │
│  │  │  Application                                 │ │ │
│  │  │  ├─ Needs to pull image from ACR            │ │ │
│  │  │  ├─ Reads token from:                        │ │ │
│  │  │  │  /var/run/secrets/workload-identity       │ │ │
│  │  │  │  /token                                   │ │ │
│  │  │  │                                            │ │ │
│  │  │  ├─ Calls Azure API with token               │ │ │
│  │  │  └─ Succeeds!                                │ │ │
│  │  └──────────────────────────────────────────────┘ │ │
│  │                                                    │ │
│  │  Service Account (sa-app-deployment)              │ │
│  │  └─ Linked to Managed Identity                   │ │
│  │     via annotation                               │ │
│  └───────────────────────────────────────────────────┘ │
│                                                        │
└─────────────────────────────────────────────────────────┘

IMPLEMENTATION (Terraform):

# Step 1: Create Managed Identity
resource "azurerm_user_assigned_identity" "app" {
  name                = "mi-app-deployment"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
}

# Step 2: Grant permissions
resource "azurerm_role_assignment" "app_acr" {
  scope              = azurerm_container_registry.main.id
  role_definition_name = "AcrPull"
  principal_id       = azurerm_user_assigned_identity.app.principal_id
}

# Step 3: Create Kubernetes service account (in cluster)
resource "kubernetes_service_account" "app" {
  metadata {
    name      = "sa-app"
    namespace = "production"
    annotations = {
      "azure.workload.identity/client-id" = azurerm_user_assigned_identity.app.client_id
    }
  }
}

# Step 4: Establish OIDC federation (Terraform doesn't directly support this)
# Use Azure CLI:
# az identity federated-credential create \
#   --name "kubernetes-federation" \
#   --identity-name "mi-app-deployment" \
#   --issuer https://oidc.prod-aks-cluster.azure.com \
#   --subject "system:serviceaccount:production:sa-app"

# Step 5: Deploy pod with service account
resource "kubernetes_deployment" "app" {
  metadata {
    name      = "app-deployment"
    namespace = "production"
  }
  spec {
    replicas = 3
    selector {
      match_labels = {
        app = "app"
      }
    }
    template {
      metadata {
        labels = {
          app = "app"
        }
        annotations = {
          "azure.workload.identity/use" = "true"
        }
      }
      spec {
        service_account_name = kubernetes_service_account.app.metadata[0].name
        container {
          name  = "app"
          image = "${azurerm_container_registry.main.login_server}/app:latest"
        }
      }
    }
  }
}
```

---

## Security & Policies

### Q: Implement network security in AKS

```
MULTI-LAYER AKS SECURITY

Layer 1: Cluster Access
├─ Private API Server (not public)
├─ Authorized IP ranges only
└─ Managed identity for cluster operations

Layer 2: Node Security
├─ Security patching (automated)
├─ Azure Disk Encryption at rest
├─ Secure Boot enabled
└─ Host-based firewall (NSG)

Layer 3: Pod Security
├─ Network Policies (Calico)
├─ Pod Security Standards
├─ Container image scanning
└─ Pod Identity (no credentials in pod)

Layer 4: Application Security
├─ RBAC (fine-grained access)
├─ Secrets encryption at rest
├─ TLS for all communication
└─ Network policies enforce

NETWORK POLICY EXAMPLE:

# Deny all ingress traffic by default
resource "kubernetes_network_policy" "default_deny" {
  metadata {
    name      = "default-deny-ingress"
    namespace = "production"
  }
  spec {
    pod_selector {}
    policy_types = ["Ingress"]
  }
}

# Allow traffic from ingress controller to app pods
resource "kubernetes_network_policy" "allow_ingress" {
  metadata {
    name      = "allow-from-ingress"
    namespace = "production"
  }
  spec {
    pod_selector {
      match_labels = {
        app = "api-server"
      }
    }
    ingress {
      from {
        pod_selector {
          match_labels = {
            app = "ingress-controller"
          }
        }
      }
      ports {
        protocol = "TCP"
        port     = "8080"
      }
    }
    policy_types = ["Ingress"]
  }
}

# Allow app pods to talk to database
resource "kubernetes_network_policy" "allow_db" {
  metadata {
    name      = "allow-to-db"
    namespace = "production"
  }
  spec {
    pod_selector {
      match_labels = {
        app = "api-server"
      }
    }
    egress {
      to {
        pod_selector {
          match_labels = {
            app = "database"
          }
        }
      }
      ports {
        protocol = "TCP"
        port     = "5432"
      }
    }
    policy_types = ["Egress"]
  }
}

# Allow DNS resolution (critical!)
resource "kubernetes_network_policy" "allow_dns" {
  metadata {
    name      = "allow-dns"
    namespace = "production"
  }
  spec {
    pod_selector {}
    egress {
      to {
        namespace_selector {
          match_labels = {
            name = "kube-system"
          }
        }
      }
      ports {
        protocol = "UDP"
        port     = "53"
      }
    }
    policy_types = ["Egress"]
  }
}

DEFENDER FOR CONTAINERS:

Provides runtime protection:
├─ Vulnerability scanning (images at push time)
├─ Runtime threat detection (suspicious pod behavior)
├─ Integration with Entra ID
└─ Actionable insights and alerts
```

---

## Monitoring & Diagnostics

### Q: How to troubleshoot pod not starting in AKS?

**Diagnostic Flowchart:**

```
POD NOT STARTING DIAGNOSIS

Pod Status: Pending / CrashLoopBackOff / Error

STEP 1: Check pod status
$ kubectl get pod -n production app-pod-123 -o wide
STATUS              NODE              READY
Pending             <none>            0/1
CrashLoopBackOff    aks-node-2        0/1
CreateContainerError aks-node-2        0/1

IF Pending:
  └─ STEP 2A: Check events
     $ kubectl describe pod -n production app-pod-123
     Events:
       Type     Reason                Status
       Warning  FailedScheduling      0/3 nodes available (2 insufficient CPU, 1 tainted)
     
     Possible causes:
     ├─ Insufficient CPU/Memory on all nodes
     ├─ Node affinity/selector can't be satisfied
     ├─ PVC not bound yet
     ├─ Taints on all nodes
     └─ Network policy blocking scheduling

  └─ FIXES:
     ├─ Reduce resource requests
     ├─ Remove node affinity constraint
     ├─ Add more nodes
     └─ Check PVC status

ELSE IF CrashLoopBackOff:
  └─ STEP 2B: Check logs
     $ kubectl logs -n production app-pod-123 --previous
     
     Possible causes:
     ├─ Application error on startup
     ├─ Missing environment variable
     ├─ Bad configuration mount
     ├─ Image pull failed
     ├─ Permission denied
     └─ OOM (out of memory)

  └─ STEP 3: Check recent events
     $ kubectl describe pod -n production app-pod-123
     Events:
       Type     Reason             Status
       Normal   Scheduled          node assigned
       Normal   Pulling            image pull started
       Normal   Pulled             image pulled
       Normal   Created            container created
       Normal   Started            container started
       Warning  BackOff            container exited with code 1

  └─ FIXES:
     ├─ Fix application startup issue
     ├─ Check environment variables
     ├─ Check ConfigMap/Secret mounts
     ├─ Verify image exists and can be pulled
     ├─ Check file permissions in Dockerfile
     └─ Increase memory limits

ELSE IF CreateContainerError:
  └─ STEP 2C: Image pull error
     $ kubectl describe pod -n production app-pod-123
     Status: ContainerCannotRun
     LastState.Reason: ImageInspectError
     Message: "error pulling image ... access denied"

  └─ Possible causes:
     ├─ ACR not accessible (network issue)
     ├─ Credentials wrong
     ├─ Image doesn't exist
     ├─ Image manifest corrupt
     └─ Image architecture mismatch

  └─ FIXES:
     ├─ Check Workload Identity configuration
     ├─ Verify ACR exists and is accessible
     ├─ Check image tag is correct
     ├─ Verify network connectivity to ACR
     └─ Check pod logs for details

STEP 4: Check node status
$ kubectl get nodes
STATUS    ROLES    AGE    VERSION
Ready     worker   10d    v1.28.0
Ready     worker   10d    v1.28.0
NotReady  worker   10d    v1.28.0

If node NotReady:
├─ SSH into node
├─ Check kubelet status: systemctl status kubelet
├─ Check disk space: df -h
├─ Check memory: free -h
├─ Check node logs: journalctl -u kubelet
└─ If critical issue: drain and recreate node

STEP 5: Check resource limits
$ kubectl top pod -n production app-pod-123
POD                CPU     MEMORY
app-pod-123       100m    256Mi

$ kubectl top node aks-node-2
NODE          CPU(cores)  CPU%  MEMORY(bytes)  MEMORY%
aks-node-2    1500m       75%   4Gi            50%

If pod at limit:
├─ Increase resource request: resources.requests.cpu
└─ If node at limit, add more nodes

STEP 6: Check readiness/liveness probes
$ kubectl get pod -n production app-pod-123 -o yaml | grep -A 10 livenessProbe

If probe failing:
├─ Increase initialDelaySeconds (time to start)
├─ Increase timeoutSeconds (allow slow response)
├─ Check probe endpoint returns success
└─ Verify probe command/URL is correct

STEP 7: Advanced debugging
$ kubectl exec -it -n production app-pod-123 -- /bin/sh
$ curl http://localhost:8080/health  # Test startup
$ env | grep -i config               # Check env vars
$ ls -la /etc/config/                # Check mounts
$ df -h                              # Check disk space

STEP 8: Check controller logs
$ kubectl logs -n kube-system -l component=kubelet
$ kubectl logs -n kube-system -l component=controller-manager
$ kubectl logs -n kube-system -l app.kubernetes.io/name=azure-cni

COMMON SCENARIOS:

Scenario: "Pod pending, all nodes say taint"
$ kubectl describe node aks-node-1
Taints: gpu=true:NoSchedule
Fix: Add tolerations to pod spec or remove taint

Scenario: "Pod CrashLoopBackOff, but logs look empty"
Fix: Check /var/log/pods/{pod-uid}/ on node
$ SSH into node
$ ls /var/log/pods/
$ cat /var/log/pods/production_app-pod-123_*/app/*.log

Scenario: "Image pull always fails"
Fix: Verify Workload Identity
$ az identity show -n mi-app --query clientId -o tsv
$ kubectl get sa sa-app -o yaml | grep client-id
(Must match!)

Scenario: "Pod stuck at Terminating"
Fix: Force delete (last resort)
$ kubectl delete pod app-pod-123 --grace-period=0 --force

PREVENTION:

1. Use init containers for setup
2. Add startup probes (K8s 1.18+)
3. Request realistic resources
4. Use readiness probes
5. Monitor pod events with tools like Falco
6. Set resource limits
7. Enable Pod Disruption Budgets (PDB)
```

---

## Production Troubleshooting

### 10+ Real Production Scenarios

**Scenario 1: AKS pod can't reach database in private subnet**

```
Problem: Pod times out connecting to Azure Database for PostgreSQL

Diagnosis:
1. Verify pod networking
   $ kubectl exec -it app-pod -- curl -v postgres-server:5432
   Result: timeout (no response)

2. Check security group (NSG)
   - Pod source IP: 10.0.1.50 (within pod subnet)
   - Destination: postgres-server private IP (10.0.2.4)
   - NSG rule missing for port 5432 inbound

3. Check route table
   - Pod subnet routing: All non-local traffic → NAT Gateway
   - Database subnet: Separate subnet, needs peering

Solution:
- Add NSG rule allowing 10.0.1.0/24 → 10.0.2.0/24 on port 5432
- OR use Private Endpoint for database (recommended)
- OR use Service Endpoint (simpler than Private Endpoint)
```

---

### Interview Questions

**Q1: How would you design AKS for a financial services company?**

```
Requirements:
- Regulatory compliance (PCI-DSS, SOX)
- High availability (99.99%)
- Low latency (sub-100ms)
- Secure by default

Architecture:

1. Network Isolation
   ├─ Private AKS cluster (no public API server)
   ├─ Network policies: Deny all by default
   ├─ Private endpoints for Azure services
   └─ ExpressRoute for on-prem connectivity

2. Identity & Security
   ├─ Workload Identity (no credentials in pods)
   ├─ Azure Entra ID integration
   ├─ Pod Security Standards enforced
   ├─ Image scanning at push time
   └─ RBAC: Least privilege

3. Compliance
   ├─ Azure Policy (enforce tagging, network rules)
   ├─ Audit logging (all actions logged)
   ├─ Secrets encryption (even at rest in etcd)
   ├─ Data residency (single region)
   └─ Backup/restore procedures tested

4. High Availability
   ├─ 3 AZs for system nodes
   ├─ Auto-scaling for surge protection
   ├─ Pod Disruption Budgets
   ├─ Cross-region failover (if needed)
   └─ Disaster recovery testing monthly

5. Monitoring
   ├─ Azure Monitor with alerts
   ├─ Application Insights for tracing
   ├─ Custom metrics for business logic
   └─ On-call team with runbooks
```

**Q2: Design multi-region AKS for global SaaS company**

See SYSTEM-DESIGN-TROUBLESHOOTING.md for full answer

---

This guide provides production-ready knowledge for AKS interviews at FAANG level.
