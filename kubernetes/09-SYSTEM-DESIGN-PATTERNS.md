# System Design Patterns (10% of Interview Weight)

> 10 large-scale Kubernetes design problems for senior/principal engineer interviews. Practice each in 45 minutes, whiteboard-style.

---

## 1. Design a Kubernetes Cluster for 10,000 Nodes

**Control Plane Sizing:**
```
At this scale, a SINGLE control plane (even HA 3-node) is
insufficient — real-world guidance (and Kubernetes' own tested
scalability limits, ~5000 nodes/150k pods per cluster) means:

├─ Split into MULTIPLE clusters (e.g. 10 clusters of 1000 nodes each)
│  rather than one mega-cluster — reduces blast radius, keeps etcd/
│  API server within tested scaling envelopes
├─ Per-cluster control plane: 5-node etcd (higher fault tolerance
│  given the criticality), API servers scaled horizontally behind
│  a load balancer (stateless, easy to scale), dedicated high-spec
│  nodes for etcd (fast NVMe, isolated from noisy neighbors)
├─ etcd: separate physical/dedicated nodes, NEVER co-located with
│  general workloads; monitor fsync latency and DB size aggressively
└─ API server: tune --max-requests-inflight, enable APF properly,
   consider API Server request routing/sharding via multiple
   read-replica-style apiserver instances if read-heavy
```

**Network Architecture:**
```
├─ CNI: Cilium (eBPF) in native routing mode or Calico with BGP —
│  avoid iptables-based kube-proxy at this scale (rule count
│  explosion)
├─ IP address planning: 10,000 nodes × ~100 pods/node = 1M+ pod IPs
│  needed — plan CIDR carefully (e.g. /8 cluster CIDR), consider
│  cloud-native CNI IP exhaustion at the VPC/VNet level
└─ Multi-cluster federation (if truly need cross-cluster
   communication): service mesh with multi-cluster support
   (Istio multi-cluster, or a global load balancer routing between
   cluster ingresses)
```

**Monitoring at Scale:**
```
├─ Prometheus sharded per cluster/team, federated via Thanos/Mimir
│  for a global query view
├─ Careful cardinality management (10k nodes × many pods = massive
│  potential time series count)
└─ Centralized logging via Loki (label-indexed, cheaper at this
   volume than full-text Elasticsearch)
```

**Key interview talking points:** acknowledge Kubernetes' tested scale limits, prefer MULTIPLE right-sized clusters over one giant cluster (operational blast radius, upgrade risk, blast-radius containment), dedicated etcd nodes with strong disk/network SLAs, eBPF-based networking to avoid iptables scaling cliffs.

---

## 2. Design a Multi-Tenant Kubernetes Platform

```
ISOLATION LAYERS (defense in depth, not just one mechanism):

1. Namespace-per-tenant (logical isolation, cheapest)
   ├─ ResourceQuota + LimitRange per namespace
   ├─ RBAC: Role+RoleBinding scoped to tenant's namespace only,
   │  NO ClusterRoleBindings for tenant users
   └─ NetworkPolicy: default-deny cross-namespace, explicit allow
      only for legitimate shared services (e.g. shared ingress)

2. Node-level isolation (stronger, for untrusted/regulated tenants)
   ├─ Dedicated node pools per tenant tier (taints + tolerations +
   │  node affinity) — prevents noisy-neighbor CPU/memory contention
   │  even within quota limits (quota caps total, doesn't prevent
   │  scheduling contention among a tenant's own pods vs others on
   │  shared nodes)
   └─ For HIGHLY sensitive tenants: dedicated clusters entirely
      (the strongest isolation, highest cost/operational overhead)

3. Runtime isolation (for genuinely untrusted code, e.g. a PaaS
   running arbitrary customer code)
   ├─ gVisor or Kata Containers instead of standard runc — stronger
      sandbox boundary against container escape
   └─ Seccomp/AppArmor RESTRICTED profiles enforced via PSA

4. Admission control
   └─ OPA Gatekeeper/Kyverno policies enforcing: mandatory resource
      limits, mandatory NetworkPolicy presence, no privileged pods,
      approved image registries only

5. Cost visibility/chargeback
   └─ Kubecost or OpenCost tagging resource usage per namespace/
      tenant label for accurate billing attribution
```

**Key interview talking points:** isolation is a SPECTRUM (namespace < node pool < cluster), match the isolation level to the tenant's trust level and compliance needs, always combine RBAC + NetworkPolicy + ResourceQuota (no single control is sufficient alone), mention concrete tools (Gatekeeper/Kyverno, Kubecost) to show hands-on awareness.

---

## 3. Design a GitOps Pipeline for 100 Microservices

```
ARCHITECTURE:

Git repos (source of truth):
├─ Option A: monorepo (all 100 services' manifests in one repo,
│  organized by directory) — simpler tooling, but large blast
│  radius for a bad merge, slower CI on big repos
└─ Option B: repo-per-service (or per-team) + a separate
   "environment/config" repo referencing versions — better
   isolation, more repos to manage, common at this scale

ArgoCD (or Flux) architecture:
├─ App-of-Apps pattern: one root Argo Application manages N child
│  Application objects (one per microservice), enabling a SINGLE
│  place to see/manage the whole fleet's sync status
├─ ApplicationSet controller: auto-generates Application objects
│  from a template + a generator (e.g. list of services, or a Git
│  directory listing) — critical at 100-service scale to avoid
│  manually maintaining 100 Application YAMLs
└─ Multiple ArgoCD instances or app-of-apps segmented by team/tier
   if a single ArgoCD becomes a bottleneck/blast-radius concern

PROMOTION STRATEGY (dev → staging → prod):
├─ Separate Kustomize overlays or Helm values files per environment
│  referencing the SAME base manifest (DRY, avoid environment drift)
├─ Image tag/digest promotion: CI builds once, pushes an immutable
│  digest; promotion between environments is just updating the
│  digest reference in the target environment's config repo (via
│  automated PR, e.g. using Argo CD Image Updater or a CI job)
└─ Progressive delivery: Argo Rollouts for canary/blue-green with
   automated analysis (Prometheus query-based promotion gates)
   before full rollout to prod

ROLLBACK: Git revert of the environment config repo commit → ArgoCD
auto-syncs back to previous state (this is the core GitOps rollback
superpower — rollback is just a git operation, fully auditable).

DRIFT DETECTION: ArgoCD continuously compares live cluster state to
Git (self-healing mode auto-corrects manual kubectl changes, or
alert-only mode for visibility without auto-revert, depending on
team's operational maturity/trust level).
```

**Key interview talking points:** App-of-Apps/ApplicationSet for managing scale, immutable image digests (not floating tags) for reliable promotion, Git revert as the rollback mechanism (huge audit/compliance advantage over manual kubectl rollback), self-healing vs alert-only drift handling trade-off.

---

## 4. Design Disaster Recovery for Stateful Applications

```
BACKUP STRATEGY (layered):

1. Application-level backups (most portable, most restorable):
   ├─ Database-native backup tools (pg_dump/pg_basebackup, mysqldump,
   │  mongodump) scheduled via CronJob, shipped to object storage
   └─ Point-in-time recovery via WAL/binlog archiving for databases
      needing sub-hour RPO

2. Volume-level snapshots (faster, but less portable):
   ├─ VolumeSnapshot (CSI) — near-instant, storage-layer snapshots
   └─ Velero: backs up BOTH Kubernetes object manifests AND
      (via CSI snapshot integration) the underlying PV data,
      to object storage (S3/Blob) — enables full NAMESPACE or
      CLUSTER restore, not just data restore

3. etcd backup (cluster CONFIGURATION disaster recovery, separate
   concern from application DATA disaster recovery):
   └─ Scheduled snapshot save, tested restore procedure (see
      Control Plane guide)

CROSS-REGION REPLICATION:
├─ Database: native replication to a standby in the secondary region
   (async replication acceptable RPO of minutes; sync replication
   for near-zero RPO but at a latency/availability cost)
├─ Object storage: cross-region replication (S3 CRR, Azure GRS)
   for backup artifacts themselves
└─ Velero backups shipped to a MULTI-region or cross-cloud storage
   bucket (protects against the backup target itself being in the
   failed region)

RTO/RPO TARGETS (example, tiered by criticality):
├─ Tier 1 (payment processing): RPO<1min (sync replication),
   RTO<5min (automated failover, pre-warmed standby cluster)
├─ Tier 2 (internal tools): RPO<1hr (async replication),
   RTO<1hr (manual failover acceptable, documented runbook)
└─ Tier 3 (dev/test): RPO<24hr (nightly backup), RTO<1day

TESTING: DR is worthless if untested — schedule quarterly (minimum)
"game day" restore drills to a SCRATCH environment, measuring actual
RTO against the target, and treat any gap as an action item.
```

**Key interview talking points:** distinguish application-data DR from control-plane (etcd) DR — they're different problems with different tools. RPO/RTO should be TIERED by workload criticality, not a single blanket target. Emphasize TESTED restores (untested backup = no backup).

---

## 5. Design Kubernetes for Machine Learning Workloads

```
GPU SCHEDULING:
├─ NVIDIA device plugin exposes nvidia.com/gpu as an extended
│  resource — integer-only, no overcommit, exclusive per container
├─ Node pools dedicated to GPU instance types, tainted so only
│  GPU-requesting pods land there (avoid wasting expensive GPU
│  nodes on non-GPU workloads)
├─ Time-slicing or MIG (Multi-Instance GPU, on A100/H100) to allow
│  multiple smaller workloads to share a single physical GPU when
│  full-GPU allocation would be wasteful for lightweight inference
└─ Bin-packing strategy (MostAllocated scoring) for GPU nodes to
   minimize the number of expensive GPU nodes running at low
   utilization

DISTRIBUTED TRAINING (gang scheduling):
├─ Training job needs N pods (e.g. 8, one per GPU-worker) scheduled
   SIMULTANEOUSLY — default scheduler's one-pod-at-a-time approach
   risks partial allocation deadlock at cluster capacity limits
├─ Use Volcano or Kueue for gang/queue-based scheduling (PodGroup
   semantics: all-or-nothing placement)
├─ Topology-aware placement: keep worker pods on nodes with
   high-bandwidth interconnect (same NVLink/InfiniBand domain) via
   node affinity/topology labels — critical for training throughput,
   cross-rack GPU communication is a major bottleneck
└─ Kubeflow Training Operator (TFJob/PyTorchJob CRDs) to manage the
   distributed training job lifecycle declaratively

MODEL SERVING:
├─ KServe or Seldon Core for standardized model-serving CRDs
   (handles autoscaling including scale-to-zero for infrequently
   used models, canary rollout of new model versions, request
   batching for GPU efficiency)
├─ HPA on custom metrics (inference requests/sec, GPU utilization %
   via DCGM exporter → Prometheus) rather than plain CPU
└─ Separate node pools for training (bursty, batch, tolerant of
   preemption/spot) vs serving (steady-state, latency-sensitive,
   needs guaranteed capacity)

COST OPTIMIZATION: spot/preemptible GPU instances for training
(checkpointing required to survive preemption), on-demand/reserved
for production serving (can't tolerate preemption mid-request).
```

**Key interview talking points:** gang scheduling is the #1 differentiator vs standard web workloads, topology-awareness for interconnect matters as much as raw GPU count, distinguish training (batch, preemption-tolerant) vs serving (latency-sensitive, steady) infrastructure needs.

---

## 6. Design a Service Mesh for 500 Microservices

```
WHY A MESH AT THIS SCALE: manually implementing mTLS, retries,
circuit-breaking, and observability in EVERY one of 500 services'
application code is untenable (inconsistent implementations,
language-specific libraries needed for each of your tech stacks) —
a mesh provides these as infrastructure, transparent to app code.

ARCHITECTURE (Istio-style, sidecar model):
├─ Envoy sidecar injected into every pod (via mutating webhook) —
│  intercepts ALL inbound/outbound traffic transparently (iptables
│  redirect rules injected by init container)
├─ Istiod (control plane): pushes configuration (routing rules,
│  mTLS certs via built-in CA, telemetry config) to all sidecars
├─ mTLS: automatic mutual TLS between ALL services (zero-trust,
│  automatic cert rotation — huge security win, avoids each team
│  reimplementing TLS + cert rotation individually)
└─ Traffic management: VirtualService/DestinationRule enable canary
   (weighted traffic split), circuit breaking (outlier detection
   ejecting unhealthy backends), retries/timeouts with consistent
   policy across all services

AT 500-SERVICE SCALE, CONSIDER:
├─ Sidecar resource overhead: 500 services × N replicas × ~50-100MB
   memory + some CPU per Envoy sidecar — genuinely material cost;
   evaluate "ambient mesh" (Istio Ambient, ztunnel — ships mTLS/L4
   security as a per-NODE (not per-pod) component, avoiding sidecar
   proliferation, reserving heavier sidecar/waypoint proxies only
   for services actually needing L7 features)
├─ Control plane scaling: Istiod itself needs adequate resources
   and may need sharding/multiple revisions for very large meshes
├─ Multi-cluster mesh if services span multiple clusters (shared
   root CA, cross-cluster service discovery via a mesh-aware gateway)
└─ Observability integration: mesh-generated metrics/traces feeding
   the SAME Prometheus/Grafana/Tempo stack as application metrics
   (unified view, not two separate observability silos)
```

**Key interview talking points:** the mesh solves the "N teams reimplementing mTLS/retries/circuit-breaking N different ways" problem, sidecar overhead is a real cost at scale (mention ambient mesh as the modern mitigation), mTLS + automatic cert rotation is the single biggest security win, don't over-index on sidecar mesh if a simpler solution (e.g. just NetworkPolicy + app-level retries) suffices for a smaller/simpler system — mesh complexity should be justified by genuine need.

---

## 7. Design Kubernetes Security for a Regulated Industry (PCI-DSS/HIPAA)

```
COMPLIANCE REQUIREMENTS MAPPED TO K8S CONTROLS:

├─ Access control / least privilege → RBAC with periodic access
   reviews, no standing cluster-admin except break-glass (audited)
├─ Encryption at rest → etcd encryption via KMS provider, encrypted
   PVs (StorageClass with encryption enabled at the cloud layer)
├─ Encryption in transit → mTLS everywhere (service mesh or manual
   TLS), TLS for ingress termination, encrypted etcd peer/client
   communication
├─ Network segmentation → NetworkPolicy default-deny + explicit
   documented allow-list (auditable evidence of segmentation),
   dedicated namespace/cluster for cardholder-data-environment (CDE)
   with NO direct internet egress (force through an audited proxy)
├─ Audit logging → full Kubernetes audit log (--audit-log-path,
   audit policy capturing at minimum all writes + auth failures),
   shipped to an immutable/tamper-evident log store, retained per
   compliance requirement (often 1+ years)
├─ Vulnerability management → mandatory image scanning (Trivy/Grype)
   in CI blocking critical CVEs, admission-time verification that
   only scanned+signed images from approved registries can run
   (Kyverno/Connaisseur policy)
├─ Pod-level hardening → Pod Security Admission "restricted"
   enforced cluster-wide, kube-bench (CIS Benchmark) scans on a
   schedule with remediation SLAs
└─ Change management → GitOps (every change is a Git commit,
   reviewed via PR, fully auditable — directly satisfies "documented
   change control" requirements common in these frameworks)

SEGMENTATION EXAMPLE (PCI-DSS Cardholder Data Environment):
Dedicated namespace/cluster for CDE workloads, NetworkPolicy
allowing ONLY: (a) ingress from a specific, audited API gateway,
(b) egress to the specific payment processor's IP range on 443,
(c) DNS. Everything else default-deny. No shared node pools with
non-CDE workloads (reduces PCI scope — fewer systems "in scope"
for the annual audit = less audit burden).
```

**Key interview talking points:** map EVERY control back to a specific compliance requirement (shows you understand WHY, not just HOW), GitOps as a change-management control is a strong differentiator to mention, minimizing "audit scope" (fewer systems touching regulated data) is a practical cost-reduction strategy interviewers appreciate.

---

## 8. Design a Kubernetes Cost Optimization Strategy

```
RIGHT-SIZING:
├─ VPA in recommendation mode (Off) to surface over-provisioned
   requests without auto-applying (safe first step, builds trust)
├─ Regularly review actual usage (container_cpu/memory_usage vs
   requests) via Kubecost/OpenCost or Prometheus — target the gap
   between requested and actually-used resources
└─ LimitRange defaults to prevent NEW workloads from being
   accidentally over-provisioned from day one

SPOT/PREEMPTIBLE INSTANCES:
├─ Use for fault-tolerant, interruptible workloads: batch jobs, CI
   runners, stateless web tier behind a load balancer (with enough
   replicas that losing one is a non-event), ML training with
   checkpointing
├─ NEVER for: stateful primary databases, single-replica critical
   services, anything without graceful handling of sudden termination
   (spot instances get a short, e.g. 30s-2min, termination warning —
   app/controller must react quickly)
└─ Mix: e.g. 70% reserved/on-demand baseline + 30% spot for burst
   capacity, using Cluster Autoscaler's --expander=priority to
   prefer spot node groups when a pod tolerates it

CLUSTER AUTOSCALING:
├─ Aggressive scale-down (lower --scale-down-unneeded-time if safe
   for the workload profile) to avoid paying for idle capacity
├─ Bin-packing (MostAllocated scheduler scoring) to consolidate
   workloads onto fewer nodes, enabling more nodes to become
   scale-down candidates
└─ Separate node pools per workload SHAPE (e.g. general CPU-heavy,
   memory-heavy, GPU) so autoscaler doesn't provision an expensive
   GPU node just because a tiny CPU-only pod is pending

RESERVED CAPACITY: for the STABLE baseline load (the minimum you
KNOW you'll always need), commit to 1-3yr reserved instances/savings
plans (30-70% discount vs on-demand) — combine with spot for the
variable/bursty portion above baseline.

VISIBILITY: Kubecost/OpenCost providing per-namespace/per-team cost
allocation (showback/chargeback) — visibility alone often drives
significant organic optimization once teams can SEE their own
resource costs, which is otherwise invisible to them in a shared
cluster.
```

**Key interview talking points:** right-sizing (fixing over-provisioned requests) is usually the SINGLE biggest lever, spot instances require workload-level resilience design (not just "turn on spot"), cost visibility/chargeback itself is a powerful (and cheap) optimization tool, reserved+spot mix matches financial commitment to actual load predictability.

---

## 9. Design a Zero-Downtime Kubernetes Upgrade Strategy

```
NODE UPGRADE PATTERNS:

Surge/blue-green node pool upgrade (recommended for managed K8s):
├─ Create a NEW node pool with the upgraded Kubernetes version
├─ Cordon + drain OLD nodes one at a time (or in small batches),
   forcing pods to reschedule onto the NEW node pool
├─ PodDisruptionBudget ensures the drain respects minimum
   availability per service during the migration
└─ Delete old node pool once fully drained and validated

In-place node upgrade (traditional kubeadm-style):
├─ Upgrade control plane FIRST (one node at a time if HA, following
   strict version skew policy: control plane can be at MOST 1 minor
   version ahead of kubelet — never upgrade kubelet ahead of API
   server)
├─ Then upgrade worker nodes one at a time: cordon → drain → upgrade
   kubelet/kubeadm on that node → uncordon
└─ Never upgrade all nodes simultaneously (guarantees an outage if
   the new version has an unexpected issue — always roll gradually)

PDB CONFIGURATION (critical prerequisite):
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2          # or maxUnavailable: 1
  selector:
    matchLabels:
      app: web-app
Without a PDB, `kubectl drain` can evict ALL replicas of a service
simultaneously if they all happen to be on the node being drained
at once — PDB is what makes drain SAFE for HA services.

TESTING PROCEDURE:
├─ Upgrade a staging/canary cluster FIRST, run full test suite +
   soak time before touching production
├─ Read the FULL changelog for deprecated/removed APIs between
   versions (use `kubectl deprecations` / Pluto tool to scan for
   manifests using soon-to-be-removed API versions BEFORE upgrading)
├─ Upgrade one AZ/node-pool at a time in production even after
   staging validation (staged rollout, not big-bang)
└─ Have a tested rollback plan (for control plane: this is HARD/
   risky — Kubernetes doesn't officially support downgrading; the
   real "rollback" for control plane issues is usually restoring
   from an etcd backup taken just before the upgrade)
```

**Key interview talking points:** version skew policy (control plane ahead of kubelets, never the reverse) is a common trap question, PDB is the specific mechanism that makes drains safe — always mention it, checking deprecated API usage BEFORE upgrading (Pluto/kubectl-deprecations) prevents the classic "upgrade breaks a CRD/controller using a removed API version" incident, control-plane rollback is fundamentally hard — the real safety net is a pre-upgrade etcd backup.

---

## 10. Design Kubernetes for Edge Computing

```
CHALLENGES UNIQUE TO EDGE:
├─ Resource-constrained devices (small ARM SBCs, industrial gateways
   — not standard cloud VM sizes)
├─ Intermittent/unreliable connectivity to a central control plane
   (unlike cloud, WAN links to remote sites can be flaky, high-
   latency, or periodically offline entirely)
└─ Potentially hundreds/thousands of small "clusters" (one per
   site/store/factory) rather than one large cluster

LIGHTWEIGHT DISTRIBUTIONS:
├─ K3s: single-binary, uses SQLite (or embedded etcd) instead of
   full etcd for smaller footprint, drops rarely-used in-tree
   features, ideal for single-node or small edge clusters
├─ MicroK8s / KubeEdge: similar lightweight goals, KubeEdge
   specifically designed for cloud-edge coordination with an
   architecture that tolerates edge-node disconnection
└─ Standard kubeadm is typically TOO heavy (etcd resource footprint
   alone often exceeds an edge device's total capacity)

INTERMITTENT CONNECTIVITY HANDLING:
├─ Edge node's local kubelet/agent must continue running currently-
   scheduled workloads even when disconnected from the "cloud"
   control plane (KubeEdge's EdgeCore explicitly designed for this
   — local autonomy, syncs state back when connectivity resumes)
├─ Avoid designs requiring constant control-plane round-trips for
   basic operation (e.g. avoid remote-only admission webhooks that
   would block ALL scheduling the moment connectivity drops)
└─ Local caching of images/configs so a reconnect-and-resume doesn't
   require re-pulling everything over a potentially slow WAN link

RESOURCE CONSTRAINTS:
├─ Minimal-footprint CNI (e.g. simpler overlay, avoid heavy eBPF
   feature sets not needed at a single small site)
├─ Aggressive resource requests/limits tuned for genuinely tiny
   nodes (a "small" cloud VM's defaults may not even fit)
└─ Centralized FLEET management (e.g. Rancher, Azure Arc, GKE
   fleet/Anthos, or a GitOps repo-per-site pattern) to manage
   configuration/deployment across hundreds of independent small
   edge clusters WITHOUT needing them to be one giant connected
   cluster — treat each site as an independent cluster, manage
   fleet-wide via a central control/observability plane that
   tolerates individual site disconnection gracefully
```

**Key interview talking points:** lightweight distributions (K3s/KubeEdge) exist specifically because standard kubeadm's etcd footprint doesn't fit edge hardware, design for LOCAL AUTONOMY (workloads keep running when disconnected) rather than assuming constant control-plane connectivity, fleet management tooling (not one giant cluster) is the right mental model for hundreds of edge sites.
