# Kubernetes Interview Questions - Complete Guide

> **500+ Kubernetes Interview Questions for Senior DevOps Engineer, SRE, and Platform Engineer roles at FAANG companies**

---

## 📋 Table of Contents

- [Architecture Deep Dive](#architecture-deep-dive)
- [Control Plane Components](#control-plane-components)
- [Workloads](#workloads)
- [Networking](#networking)
- [Storage](#storage)
- [Security](#security)
- [Observability](#observability)
- [Troubleshooting Scenarios](#troubleshooting-scenarios)
- [Design Problems](#design-problems)
- [Production Operations](#production-operations)

---

## Architecture Deep Dive

### 🟢 Basic Questions

#### Q1: Explain the Kubernetes architecture and the role of each component.

**Basic Answer:**
Kubernetes has a control plane (API server, etcd, scheduler, controller manager) that manages the cluster and worker nodes (kubelet, kube-proxy, container runtime) that run applications.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     KUBERNETES ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      CONTROL PLANE                               │    │
│  │                                                                  │    │
│  │  ┌──────────────────────────────────────────────────────────┐   │    │
│  │  │                    API SERVER                             │   │    │
│  │  │  • REST API endpoint for all operations                   │   │    │
│  │  │  • Authentication, Authorization, Admission Control       │   │    │
│  │  │  • Only component that talks to etcd                      │   │    │
│  │  └──────────────────────────────────────────────────────────┘   │    │
│  │           │              │              │              │        │    │
│  │           ▼              ▼              ▼              ▼        │    │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │    │
│  │  │    etcd    │  │ Scheduler  │  │ Controller │  │  Cloud   │  │    │
│  │  │            │  │            │  │  Manager   │  │Controller│  │    │
│  │  │ • Key-value│  │ • Watches  │  │            │  │          │  │    │
│  │  │   store    │  │   unschd   │  │ • Node     │  │ • Node   │  │    │
│  │  │ • Cluster  │  │   pods     │  │ • Replica  │  │ • Route  │  │    │
│  │  │   state    │  │ • Selects  │  │ • Endpoint │  │ • Service│  │    │
│  │  │ • Raft     │  │   best     │  │ • Service  │  │ • Volume │  │    │
│  │  │   consensus│  │   node     │  │   Account  │  │          │  │    │
│  │  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                     │
│                                    │ kubelet registers                   │
│                                    │ API server sends workloads          │
│                                    ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                       WORKER NODES                               │    │
│  │                                                                  │    │
│  │  ┌──────────────────┐  ┌──────────────────┐                     │    │
│  │  │      NODE 1      │  │      NODE 2      │                     │    │
│  │  │                  │  │                  │                     │    │
│  │  │  ┌────────────┐  │  │  ┌────────────┐  │                     │    │
│  │  │  │  kubelet   │  │  │  │  kubelet   │  │                     │    │
│  │  │  │ • Node agent│  │  │  │ • Node agent│  │                     │    │
│  │  │  │ • Pod      │  │  │  │ • Pod      │  │                     │    │
│  │  │  │   lifecycle │  │  │  │   lifecycle │  │                     │    │
│  │  │  └────────────┘  │  │  └────────────┘  │                     │    │
│  │  │                  │  │                  │                     │    │
│  │  │  ┌────────────┐  │  │  ┌────────────┐  │                     │    │
│  │  │  │ kube-proxy │  │  │  │ kube-proxy │  │                     │    │
│  │  │  │ • Network  │  │  │  │ • Network  │  │                     │    │
│  │  │  │   rules    │  │  │  │   rules    │  │                     │    │
│  │  │  │ • iptables/│  │  │  │ • iptables/│  │                     │    │
│  │  │  │   IPVS     │  │  │  │   IPVS     │  │                     │    │
│  │  │  └────────────┘  │  │  └────────────┘  │                     │    │
│  │  │                  │  │                  │                     │    │
│  │  │  ┌────────────┐  │  │  ┌────────────┐  │                     │    │
│  │  │  │ Container  │  │  │  │ Container  │  │                     │    │
│  │  │  │  Runtime   │  │  │  │  Runtime   │  │                     │    │
│  │  │  │ containerd │  │  │  │ containerd │  │                     │    │
│  │  │  └────────────┘  │  │  └────────────┘  │                     │    │
│  │  │                  │  │                  │                     │    │
│  │  │  ┌────┐ ┌────┐   │  │  ┌────┐ ┌────┐   │                     │    │
│  │  │  │Pod1│ │Pod2│   │  │  │Pod3│ │Pod4│   │                     │    │
│  │  │  └────┘ └────┘   │  │  └────┘ └────┘   │                     │    │
│  │  └──────────────────┘  └──────────────────┘                     │    │
│  │                                                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

Component Communication Flow:
```
┌─────────────────────────────────────────────────────────────────┐
│                    REQUEST FLOW: kubectl apply                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. kubectl sends request to API Server                         │
│     └─▶ Authentication (certificates, tokens, OIDC)             │
│     └─▶ Authorization (RBAC)                                    │
│     └─▶ Admission Controllers (mutating, validating)            │
│                                                                  │
│  2. API Server writes to etcd                                   │
│     └─▶ Object stored with resourceVersion                      │
│                                                                  │
│  3. Controllers watch for changes (control loop)                │
│     └─▶ Deployment Controller creates ReplicaSet                │
│     └─▶ ReplicaSet Controller creates Pods                      │
│                                                                  │
│  4. Scheduler watches for unscheduled pods                      │
│     └─▶ Filtering: Node resources, taints, affinity             │
│     └─▶ Scoring: Spread, resource balance                       │
│     └─▶ Binding: Updates pod.spec.nodeName                      │
│                                                                  │
│  5. Kubelet on target node                                      │
│     └─▶ Watches API server for pods assigned to its node        │
│     └─▶ Pulls container images                                  │
│     └─▶ Creates containers via CRI (Container Runtime Interface)│
│     └─▶ Sets up networking via CNI                              │
│     └─▶ Mounts volumes via CSI                                  │
│     └─▶ Reports status back to API server                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Interview Tips:**
- Know the exact role of each component
- Understand the watch mechanism and control loops
- Be able to trace a request from kubectl to running pod

---

#### Q2: Explain etcd and its importance in Kubernetes. How would you handle etcd failures?

**Basic Answer:**
etcd is a distributed key-value store that holds all cluster state and configuration. It uses the Raft consensus algorithm to maintain consistency across nodes.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ETCD IN KUBERNETES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  What etcd stores:                                              │
│  ├── /registry/pods/                                            │
│  ├── /registry/services/                                        │
│  ├── /registry/deployments/                                     │
│  ├── /registry/secrets/                                         │
│  ├── /registry/configmaps/                                      │
│  └── /registry/... (all cluster objects)                        │
│                                                                  │
│  RAFT CONSENSUS:                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Leader Election:                                        │    │
│  │  • One leader handles all writes                        │    │
│  │  • Followers replicate from leader                      │    │
│  │  • Election if leader fails (heartbeat timeout)         │    │
│  │                                                          │    │
│  │  Write Process:                                          │    │
│  │  1. Client sends write to leader                        │    │
│  │  2. Leader appends to local log                         │    │
│  │  3. Leader replicates to followers                      │    │
│  │  4. Majority acknowledges (quorum)                      │    │
│  │  5. Leader commits and responds                         │    │
│  │                                                          │    │
│  │  Quorum: (n/2) + 1 nodes must agree                     │    │
│  │  • 3 nodes: can tolerate 1 failure                      │    │
│  │  • 5 nodes: can tolerate 2 failures                     │    │
│  │  • 7 nodes: can tolerate 3 failures                     │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  BACKUP AND RECOVERY:                                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # Backup                                                │    │
│  │  ETCDCTL_API=3 etcdctl snapshot save /backup/snap.db \  │    │
│  │    --endpoints=https://127.0.0.1:2379 \                 │    │
│  │    --cacert=/etc/kubernetes/pki/etcd/ca.crt \           │    │
│  │    --cert=/etc/kubernetes/pki/etcd/server.crt \         │    │
│  │    --key=/etc/kubernetes/pki/etcd/server.key            │    │
│  │                                                          │    │
│  │  # Verify backup                                         │    │
│  │  etcdctl snapshot status /backup/snap.db -w table       │    │
│  │                                                          │    │
│  │  # Restore                                               │    │
│  │  etcdctl snapshot restore /backup/snap.db \             │    │
│  │    --data-dir=/var/lib/etcd-restored \                  │    │
│  │    --name=master-1 \                                    │    │
│  │    --initial-cluster=master-1=https://10.0.0.1:2380     │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

etcd Failure Scenarios and Resolution:

| Scenario | Impact | Resolution |
|----------|--------|------------|
| Single member failure (3-node) | No impact (quorum maintained) | Replace member |
| Two member failure (3-node) | Cluster read-only | Restore from backup |
| Leader failure | Brief unavailability during election | Automatic failover |
| Data corruption | Cluster instability | Restore from backup |
| Network partition | Split-brain risk | Proper network design |

Production etcd best practices:
```yaml
# etcd configuration
name: etcd-1
data-dir: /var/lib/etcd
listen-peer-urls: https://10.0.0.1:2380
listen-client-urls: https://10.0.0.1:2379,https://127.0.0.1:2379
advertise-client-urls: https://10.0.0.1:2379
initial-advertise-peer-urls: https://10.0.0.1:2380
initial-cluster: etcd-1=https://10.0.0.1:2380,etcd-2=https://10.0.0.2:2380,etcd-3=https://10.0.0.3:2380
initial-cluster-token: etcd-cluster-1
initial-cluster-state: new

# Resource recommendations
# - SSD storage (NVMe preferred)
# - Dedicated disk for etcd
# - Low-latency network between members
# - 8GB RAM minimum for production
# - Regular compaction and defragmentation

# Monitoring metrics
# - etcd_server_has_leader
# - etcd_disk_wal_fsync_duration_seconds
# - etcd_network_peer_round_trip_time_seconds
# - etcd_mvcc_db_total_size_in_bytes
```

---

### 🟡 Intermediate Questions

#### Q3: Explain the Kubernetes Scheduler and its algorithm.

**Basic Answer:**
The scheduler watches for newly created pods without an assigned node and selects the best node based on resource requirements, constraints, and policies.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    SCHEDULER ALGORITHM                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  SCHEDULING CYCLE                                        │    │
│  │                                                          │    │
│  │  1. FILTERING (Predicates)                               │    │
│  │     Remove nodes that don't meet requirements            │    │
│  │     ┌─────────────────────────────────────────────────┐  │    │
│  │     │ • PodFitsResources: CPU, memory available       │  │    │
│  │     │ • PodFitsHostPorts: Port not in use             │  │    │
│  │     │ • NodeSelector: Labels match                    │  │    │
│  │     │ • NodeAffinity: Affinity rules match            │  │    │
│  │     │ • TaintToleration: Pod tolerates taints         │  │    │
│  │     │ • PodTopologySpread: Distribution constraints   │  │    │
│  │     └─────────────────────────────────────────────────┘  │    │
│  │                         │                                │    │
│  │                         ▼                                │    │
│  │  2. SCORING (Priorities)                                 │    │
│  │     Rank remaining nodes (0-100)                         │    │
│  │     ┌─────────────────────────────────────────────────┐  │    │
│  │     │ • LeastRequestedPriority: Prefer less utilized  │  │    │
│  │     │ • BalancedResourceAllocation: Even CPU/memory   │  │    │
│  │     │ • NodeAffinityPriority: Prefer affinity match   │  │    │
│  │     │ • ImageLocalityPriority: Image already present  │  │    │
│  │     │ • InterPodAffinityPriority: Pod affinity        │  │    │
│  │     │ • TaintTolerationPriority: Better toleration    │  │    │
│  │     └─────────────────────────────────────────────────┘  │    │
│  │                         │                                │    │
│  │                         ▼                                │    │
│  │  3. BINDING                                              │    │
│  │     Update pod.spec.nodeName via API server              │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  CUSTOM SCHEDULING:                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # Node Affinity                                         │    │
│  │  affinity:                                               │    │
│  │    nodeAffinity:                                         │    │
│  │      requiredDuringSchedulingIgnoredDuringExecution:     │    │
│  │        nodeSelectorTerms:                                │    │
│  │        - matchExpressions:                               │    │
│  │          - key: topology.kubernetes.io/zone              │    │
│  │            operator: In                                  │    │
│  │            values: [us-east-1a, us-east-1b]              │    │
│  │                                                          │    │
│  │  # Pod Anti-Affinity (spread across nodes)               │    │
│  │  affinity:                                               │    │
│  │    podAntiAffinity:                                      │    │
│  │      requiredDuringSchedulingIgnoredDuringExecution:     │    │
│  │      - labelSelector:                                    │    │
│  │          matchLabels:                                    │    │
│  │            app: web                                      │    │
│  │        topologyKey: kubernetes.io/hostname               │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

Scheduler Profiles (Multi-scheduler):
```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    plugins:
      score:
        enabled:
          - name: NodeResourcesBalancedAllocation
            weight: 1
          - name: ImageLocality
            weight: 1
  - schedulerName: high-priority-scheduler
    plugins:
      preFilter:
        disabled:
          - name: "*"
      filter:
        enabled:
          - name: NodeResourcesFit
      score:
        enabled:
          - name: NodeResourcesLeastAllocated
            weight: 10
```

Pod Topology Spread Constraints:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: my-app
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: my-app
```

---

## Workloads

### 🟢 Basic Questions

#### Q4: Explain the difference between Deployments, StatefulSets, DaemonSets, and Jobs.

**Basic Answer:**
- **Deployment**: Stateless applications, rolling updates
- **StatefulSet**: Stateful applications, stable network identity, ordered operations
- **DaemonSet**: Run on every node (logging, monitoring)
- **Job/CronJob**: Batch processing, one-time or scheduled tasks

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    WORKLOAD COMPARISON                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DEPLOYMENT                          STATEFULSET                │
│  ┌────────────────────────────┐     ┌────────────────────────┐  │
│  │ ReplicaSet manages pods    │     │ Stable network identity│  │
│  │                            │     │                        │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐  │     │  web-0  web-1  web-2   │  │
│  │  │web-x│ │web-y│ │web-z│  │     │    │      │      │     │  │
│  │  └─────┘ └─────┘ └─────┘  │     │  pvc-0  pvc-1  pvc-2   │  │
│  │                            │     │                        │  │
│  │ Random pod names           │     │ Predictable pod names  │  │
│  │ Shared storage (optional)  │     │ Unique storage per pod │  │
│  │ Parallel scaling           │     │ Ordered scaling        │  │
│  │ Any order deletion         │     │ Reverse order deletion │  │
│  └────────────────────────────┘     └────────────────────────┘  │
│                                                                  │
│  DAEMONSET                           JOB / CRONJOB              │
│  ┌────────────────────────────┐     ┌────────────────────────┐  │
│  │ One pod per node           │     │ Run to completion      │  │
│  │                            │     │                        │  │
│  │  Node1  Node2  Node3       │     │  Job: One-time task    │  │
│  │  ┌───┐  ┌───┐  ┌───┐       │     │  ┌─────┐               │  │
│  │  │log│  │log│  │log│       │     │  │batch│ → Complete    │  │
│  │  └───┘  └───┘  └───┘       │     │  └─────┘               │  │
│  │                            │     │                        │  │
│  │ Use cases:                 │     │  CronJob: Scheduled    │  │
│  │ • Log collectors           │     │  ┌─────┐               │  │
│  │ • Node monitoring          │     │  │ */5 │ → Run every   │  │
│  │ • Storage daemons          │     │  │ min │   5 minutes   │  │
│  │ • Network plugins          │     │  └─────┘               │  │
│  └────────────────────────────┘     └────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

StatefulSet Deep Dive:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql  # Required for stable network identity
  replicas: 3
  podManagementPolicy: OrderedReady  # or Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # Only update pods with ordinal >= partition
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi

# DNS entries created:
# mysql-0.mysql.namespace.svc.cluster.local
# mysql-1.mysql.namespace.svc.cluster.local
# mysql-2.mysql.namespace.svc.cluster.local
```

---

### 🟡 Intermediate Questions

#### Q5: Explain Pod lifecycle, probes, and how Kubernetes handles pod failures.

**Basic Answer:**
Pods go through Pending, Running, Succeeded/Failed states. Kubernetes uses liveness, readiness, and startup probes to manage container health and traffic routing.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    POD LIFECYCLE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  POD PHASES:                                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Pending ──▶ Running ──▶ Succeeded                       │    │
│  │     │           │            │                           │    │
│  │     │           │            └──▶ Pod completed          │    │
│  │     │           │                                        │    │
│  │     │           └──▶ Failed (container exits non-zero)   │    │
│  │     │                                                    │    │
│  │     └──▶ Failed (image pull, resource unavailable)       │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  CONTAINER STATES:                                              │
│  • Waiting: Pulling image, starting                             │
│  • Running: Executing                                           │
│  • Terminated: Finished (exit code)                             │
│                                                                  │
│  PROBES:                                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Startup Probe (runs first)                              │    │
│  │  ├── Purpose: Allow slow-starting containers             │    │
│  │  ├── Failure: Container killed, restart policy applies   │    │
│  │  └── Success: Other probes begin                         │    │
│  │                                                          │    │
│  │  Liveness Probe (after startup)                          │    │
│  │  ├── Purpose: Detect deadlocks, hangs                    │    │
│  │  ├── Failure: Container restarted                        │    │
│  │  └── Success: Container considered healthy               │    │
│  │                                                          │    │
│  │  Readiness Probe (after startup)                         │    │
│  │  ├── Purpose: Control traffic routing                    │    │
│  │  ├── Failure: Removed from Service endpoints             │    │
│  │  └── Success: Added to Service endpoints                 │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  PROBE TYPES:                                                   │
│  • exec: Run command in container                               │
│  • httpGet: HTTP GET request                                    │
│  • tcpSocket: TCP connection check                              │
│  • grpc: gRPC health check                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Probe Configuration Example:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
    - name: app
      image: myapp:1.0
      ports:
        - containerPort: 8080
      
      # Startup probe - for slow starting apps
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        failureThreshold: 30    # 30 * 10s = 5 minutes to start
        periodSeconds: 10
      
      # Liveness probe - restart if unhealthy
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 0  # Startup probe handles delay
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3     # 3 failures to restart
      
      # Readiness probe - control traffic
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 3
        successThreshold: 1
```

**Expert Answer:**

Graceful Shutdown Flow:
```
┌─────────────────────────────────────────────────────────────────┐
│                    POD TERMINATION FLOW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Pod marked for deletion                                     │
│     └── terminationGracePeriodSeconds starts (default 30s)      │
│                                                                  │
│  2. Simultaneously:                                             │
│     ├── preStop hook executes (if defined)                      │
│     ├── Pod removed from Service endpoints                      │
│     └── SIGTERM sent to containers                              │
│                                                                  │
│  3. Application handles SIGTERM                                 │
│     ├── Stop accepting new requests                             │
│     ├── Complete in-flight requests                             │
│     └── Clean up resources                                      │
│                                                                  │
│  4. Grace period expires or containers exit                     │
│     └── SIGKILL sent to remaining processes                     │
│                                                                  │
│  Best Practice Configuration:                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  spec:                                                   │    │
│  │    terminationGracePeriodSeconds: 60                     │    │
│  │    containers:                                           │    │
│  │      - name: app                                         │    │
│  │        lifecycle:                                        │    │
│  │          preStop:                                        │    │
│  │            exec:                                         │    │
│  │              command:                                    │    │
│  │                - /bin/sh                                 │    │
│  │                - -c                                      │    │
│  │                - sleep 15 && kill -SIGTERM 1             │    │
│  │            # Delay allows endpoint removal propagation   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Networking

### 🟢 Basic Questions

#### Q6: Explain Kubernetes networking model and Service types.

**Basic Answer:**
Kubernetes requires that all pods can communicate with all other pods without NAT. Service types are ClusterIP (internal), NodePort (external via node port), LoadBalancer (cloud LB), and ExternalName (DNS alias).

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    KUBERNETES NETWORKING MODEL                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  FUNDAMENTAL REQUIREMENTS:                                       │
│  1. All pods can communicate without NAT                        │
│  2. All nodes can communicate with all pods without NAT         │
│  3. Pod sees its own IP as others see it                        │
│                                                                  │
│  SERVICE TYPES:                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  ClusterIP (Default)                                     │    │
│  │  ┌─────────────────────────────────────────────────┐     │    │
│  │  │  Internal IP accessible only within cluster      │     │    │
│  │  │                                                  │     │    │
│  │  │  Client ──▶ ClusterIP:80 ──▶ Pod1               │     │    │
│  │  │         (10.96.0.1)          Pod2               │     │    │
│  │  │                              Pod3               │     │    │
│  │  └─────────────────────────────────────────────────┘     │    │
│  │                                                          │    │
│  │  NodePort                                                │    │
│  │  ┌─────────────────────────────────────────────────┐     │    │
│  │  │  Exposes service on each node's IP              │     │    │
│  │  │                                                  │     │    │
│  │  │  External ──▶ NodeIP:30007 ──▶ ClusterIP ──▶ Pods│     │    │
│  │  │                                                  │     │    │
│  │  │  Port range: 30000-32767                        │     │    │
│  │  └─────────────────────────────────────────────────┘     │    │
│  │                                                          │    │
│  │  LoadBalancer                                            │    │
│  │  ┌─────────────────────────────────────────────────┐     │    │
│  │  │  Cloud provider creates external load balancer  │     │    │
│  │  │                                                  │     │    │
│  │  │  External ──▶ Cloud LB ──▶ NodePort ──▶ Pods    │     │    │
│  │  │                                                  │     │    │
│  │  └─────────────────────────────────────────────────┘     │    │
│  │                                                          │    │
│  │  ExternalName                                            │    │
│  │  ┌─────────────────────────────────────────────────┐     │    │
│  │  │  DNS CNAME record to external service           │     │    │
│  │  │                                                  │     │    │
│  │  │  my-svc.ns ──▶ CNAME ──▶ db.external.com        │     │    │
│  │  └─────────────────────────────────────────────────┘     │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

CNI (Container Network Interface) Options:

| CNI | Features | Use Case |
|-----|----------|----------|
| Calico | Network policies, BGP, eBPF | Production, security-focused |
| Cilium | eBPF-based, L7 policies, observability | Advanced networking |
| Flannel | Simple overlay (VXLAN) | Simple deployments |
| Weave | Encrypted mesh | Multi-cloud |
| AWS VPC CNI | Native AWS IPs | EKS |
| Azure CNI | Native Azure IPs | AKS |

Network Policy Example:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-allow
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: frontend
          podSelector:
            matchLabels:
              app: web
        - ipBlock:
            cidr: 10.0.0.0/8
            except:
              - 10.0.1.0/24
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
```

---

## Troubleshooting Scenarios

### Scenario 1: Pod Stuck in CrashLoopBackOff

**Problem:** Pods repeatedly crash and restart.

**Diagnostic Steps:**

```
┌─────────────────────────────────────────────────────────────────┐
│              CRASHLOOPBACKOFF TROUBLESHOOTING                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Step 1: Check pod events and status                            │
│  kubectl describe pod <pod-name> -n <namespace>                 │
│  └── Look for: Exit codes, OOM, failed probes                   │
│                                                                  │
│  Step 2: Check container logs                                   │
│  kubectl logs <pod-name> -n <namespace> --previous              │
│  └── --previous shows logs from crashed container               │
│                                                                  │
│  Step 3: Analyze exit codes                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Exit Code 0   = Successful termination                 │    │
│  │  Exit Code 1   = General errors                         │    │
│  │  Exit Code 137 = SIGKILL (OOMKilled or killed)         │    │
│  │  Exit Code 139 = SIGSEGV (Segmentation fault)          │    │
│  │  Exit Code 143 = SIGTERM (Graceful shutdown)           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Step 4: Check resource constraints                             │
│  kubectl top pod <pod-name> -n <namespace>                      │
│  └── Compare with resource limits                               │
│                                                                  │
│  Common Causes:                                                  │
│  ├── Application error (check logs)                             │
│  ├── Missing dependencies (configmap, secret, service)          │
│  ├── Resource limits too low (OOMKilled)                        │
│  ├── Liveness probe misconfigured                               │
│  ├── Incorrect command/args                                     │
│  └── Image issues (missing entrypoint)                          │
│                                                                  │
│  Debug with ephemeral container:                                │
│  kubectl debug <pod-name> -it --image=busybox --target=<container>│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Scenario 2: Pod Stuck in Pending State

```
┌─────────────────────────────────────────────────────────────────┐
│                  PENDING POD TROUBLESHOOTING                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  kubectl describe pod <pod-name> | grep -A 10 Events            │
│                                                                  │
│  Common Causes:                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  "Insufficient cpu/memory"                               │    │
│  │  └── Scale cluster or reduce resource requests           │    │
│  │                                                          │    │
│  │  "node(s) didn't match node selector"                    │    │
│  │  └── Check nodeSelector labels vs node labels            │    │
│  │                                                          │    │
│  │  "node(s) had taints that pod didn't tolerate"           │    │
│  │  └── Add tolerations to pod spec                         │    │
│  │                                                          │    │
│  │  "persistentvolumeclaim not found/bound"                 │    │
│  │  └── Check PVC status, storage class, PV availability    │    │
│  │                                                          │    │
│  │  "no nodes available to schedule"                        │    │
│  │  └── All nodes unschedulable (cordoned, tainted)         │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Debug Commands:                                                │
│  kubectl get nodes -o wide                                      │
│  kubectl describe node <node-name>                              │
│  kubectl get pvc -n <namespace>                                 │
│  kubectl get events -n <namespace> --sort-by='.lastTimestamp'   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Scenario 3: Service Not Reaching Pods

```
┌─────────────────────────────────────────────────────────────────┐
│              SERVICE CONNECTIVITY TROUBLESHOOTING                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Step 1: Verify endpoints                                       │
│  kubectl get endpoints <service-name> -n <namespace>            │
│  └── Should show pod IPs. Empty = selector mismatch             │
│                                                                  │
│  Step 2: Check service selector matches pod labels              │
│  kubectl get svc <service-name> -o yaml                         │
│  kubectl get pods -l <selector> -n <namespace>                  │
│                                                                  │
│  Step 3: Test connectivity from within cluster                  │
│  kubectl run debug --image=nicolaka/netshoot -it --rm -- bash   │
│  └── curl http://<service-name>.<namespace>:<port>              │
│  └── nslookup <service-name>.<namespace>                        │
│                                                                  │
│  Step 4: Check pod readiness                                    │
│  kubectl get pods -n <namespace>                                │
│  └── Not Ready pods excluded from endpoints                     │
│                                                                  │
│  Step 5: Verify network policies                                │
│  kubectl get networkpolicies -n <namespace>                     │
│  └── May be blocking traffic                                    │
│                                                                  │
│  Step 6: Check kube-proxy                                       │
│  kubectl logs -n kube-system -l k8s-app=kube-proxy              │
│  └── iptables/IPVS rules creation                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Design Problems

### Design 1: Multi-Tenant Kubernetes Platform

**Requirements:**
- 50+ development teams
- Strong isolation between tenants
- Shared platform services
- Cost allocation
- Self-service capabilities

**Solution:**

```
┌─────────────────────────────────────────────────────────────────┐
│              MULTI-TENANT KUBERNETES DESIGN                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ISOLATION MODEL: Namespace per tenant                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Cluster                                                 │    │
│  │  ├── Namespace: team-a                                   │    │
│  │  │   ├── ResourceQuota (CPU, memory, pods)              │    │
│  │  │   ├── LimitRange (default limits)                    │    │
│  │  │   ├── NetworkPolicy (isolate by default)             │    │
│  │  │   └── RBAC (team-a-admin, team-a-developer)          │    │
│  │  │                                                       │    │
│  │  ├── Namespace: team-b                                   │    │
│  │  │   └── (same structure)                                │    │
│  │  │                                                       │    │
│  │  └── Namespace: platform-services                        │    │
│  │      ├── Ingress Controller                              │    │
│  │      ├── Monitoring (Prometheus)                         │    │
│  │      ├── Logging (Fluentd)                               │    │
│  │      └── Cert-Manager                                    │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  NETWORK ISOLATION:                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  # Default deny all ingress                              │    │
│  │  apiVersion: networking.k8s.io/v1                        │    │
│  │  kind: NetworkPolicy                                     │    │
│  │  metadata:                                               │    │
│  │    name: default-deny-ingress                            │    │
│  │  spec:                                                   │    │
│  │    podSelector: {}                                       │    │
│  │    policyTypes:                                          │    │
│  │      - Ingress                                           │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  RESOURCE QUOTAS:                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  apiVersion: v1                                          │    │
│  │  kind: ResourceQuota                                     │    │
│  │  metadata:                                               │    │
│  │    name: team-quota                                      │    │
│  │  spec:                                                   │    │
│  │    hard:                                                 │    │
│  │      requests.cpu: "20"                                  │    │
│  │      requests.memory: 40Gi                               │    │
│  │      limits.cpu: "40"                                    │    │
│  │      limits.memory: 80Gi                                 │    │
│  │      pods: "100"                                         │    │
│  │      services.loadbalancers: "2"                         │    │
│  │      persistentvolumeclaims: "20"                        │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📚 Documentation Links

### Official Documentation
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

### Certification Resources
- [CKA Curriculum](https://github.com/cncf/curriculum)
- [CKAD Exercises](https://github.com/dgkanatsios/CKAD-exercises)
- [CKS Study Guide](https://github.com/walidshaari/Certified-Kubernetes-Security-Specialist)

### Community Resources
- [Kubernetes Blog](https://kubernetes.io/blog/)
- [CNCF Landscape](https://landscape.cncf.io/)
- [Kubernetes Patterns Book](https://k8spatterns.io/)

---

**[← Back to Main README](../README.md)** | **[Next: Docker →](../docker/README.md)**
