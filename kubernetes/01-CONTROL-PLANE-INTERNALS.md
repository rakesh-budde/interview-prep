# Control Plane Internals — Deep Dive (25% of Interview Weight)

> System-internals level coverage of API Server, etcd, Scheduler, and Controller Manager

---

## Table of Contents
- [1.1 API Server Deep Dive](#11-api-server-deep-dive)
- [1.2 etcd Deep Dive](#12-etcd-deep-dive)
- [1.3 Scheduler Deep Dive](#13-scheduler-deep-dive)
- [1.4 Controller Manager Deep Dive](#14-controller-manager-deep-dive)

---

## 1.1 API Server Deep Dive

### Complete Request Lifecycle

```
kubectl apply -f deployment.yaml
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│ 1. TRANSPORT: HTTPS (TLS 1.2/1.3), client cert or bearer token     │
└──────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. AUTHENTICATION (who are you?)                                   │
│    Chain of authenticators, tried in order until one succeeds:     │
│    a) X.509 client certificates (CN = username, O = group)         │
│    b) Static bearer tokens (deprecated)                             │
│    c) Bootstrap tokens                                              │
│    d) ServiceAccount tokens (JWT, signed by cluster CA)             │
│    e) OIDC tokens (external IdP: Okta/Azure AD/Google)             │
│    f) Webhook token authentication (external auth service)          │
│    Result: user=alice, groups=[dev-team, system:authenticated]     │
└──────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. AUTHORIZATION (are you allowed?)                                 │
│    Modes (configured via --authorization-mode, evaluated in order): │
│    a) Node authorizer (kubelets can only access their own objects)  │
│    b) ABAC (deprecated, file-based policy)                          │
│    c) RBAC (Role/ClusterRole + Binding — the standard)              │
│    d) Webhook (external authz service, e.g. OPA)                    │
│    RBAC decision: iterate all RoleBindings/ClusterRoleBindings      │
│    bound to alice's identity/groups; if ANY rule allows → ALLOW.   │
│    Default is implicit DENY (no explicit deny rules exist).         │
└──────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│ 4. ADMISSION CONTROL (mutate + validate before persisting)          │
│                                                                     │
│   Mutating Admission Chain (order matters):                        │
│   ├─ NamespaceLifecycle, LimitRanger                                │
│   ├─ ServiceAccount (injects default SA + token volume)             │
│   ├─ DefaultStorageClass                                            │
│   ├─ PodSecurity (mutation only in warn/audit mode)                 │
│   └─ MutatingAdmissionWebhook (custom webhooks, e.g. Istio sidecar  │
│      injector, OPA Gatekeeper mutations)                            │
│                                                                     │
│   Validating Admission Chain (runs AFTER mutating, on final object): │
│   ├─ ResourceQuota                                                  │
│   ├─ PodSecurity (enforce mode — reject if violates restricted/     │
│      baseline profile)                                               │
│   └─ ValidatingAdmissionWebhook (custom policy, e.g. Gatekeeper     │
│      constraints, Kyverno policies)                                  │
│                                                                     │
│   Each webhook call: HTTP POST AdmissionReview JSON, timeout        │
│   default 10s, failurePolicy=Fail|Ignore determines behavior on     │
│   webhook unavailability.                                            │
└──────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│ 5. SCHEMA VALIDATION (OpenAPI v3 schema from CRD/built-in types)    │
└──────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│ 6. PERSIST TO ETCD                                                  │
│    - Object serialized to protobuf (internal storage format)       │
│    - Written with optimistic concurrency (resourceVersion check)   │
│    - etcd replicates via Raft to quorum before ack                 │
└──────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│ 7. WATCH NOTIFICATION                                               │
│    API server pushes watch event to all watchers (controllers,     │
│    schedulers, kubelets) via long-lived HTTP/gRPC streams           │
└──────────────────────────────────────────────────────────────────┘

Total latency: 50-200ms end-to-end (excluding scheduling + kubelet)
```

### Watch Mechanism & Informer Pattern

```
PROBLEM: 10,000 clients (controllers, kubelets) need to know about
every change to every object — polling would overwhelm etcd/API server.

SOLUTION: Watch + Informer pattern

┌─────────────────────────────────────────────────────────────┐
│  API Server: maintains in-memory "watch cache" per resource   │
│  type (backed by etcd, refreshed via etcd watch)              │
└─────────────────────────────────────────────────────────────┘
              │
              │ HTTP GET /api/v1/pods?watch=true&resourceVersion=12345
              ▼
┌─────────────────────────────────────────────────────────────┐
│  Client (Informer)                                             │
│  1. LIST: Get full snapshot at resourceVersion=12345           │
│  2. WATCH: Open long-lived connection from RV=12345            │
│  3. Receive incremental events: ADDED/MODIFIED/DELETED/BOOKMARK│
│  4. Populate local cache (thread-safe store, indexed)          │
│  5. If watch drops (410 Gone = RV too old/compacted):           │
│     → RELIST from scratch                                      │
└─────────────────────────────────────────────────────────────┘

SharedInformer Architecture (client-go):
├─ Reflector: performs List+Watch, pushes to DeltaFIFO queue
├─ DeltaFIFO: ordered queue of changes (compresses consecutive ops)
├─ Indexer: thread-safe local cache (like a mini read replica of etcd)
├─ Controller loop: pops DeltaFIFO, updates Indexer, calls handlers
└─ Workqueue: handler enqueues key (namespace/name), NOT the object
   (deduplicates + rate-limits; worker re-fetches from Indexer when
    processing — avoids stale-object bugs)

WHY THIS MATTERS: Because all controllers share informers via
informer factories, N controllers watching Pods = 1 watch connection
to API server, not N. This is why SharedInformerFactory exists.
```

### Admission Webhooks (Mutating vs Validating)

```yaml
# Mutating webhook - runs FIRST, can modify the object
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-defaulter
webhooks:
  - name: defaulter.example.com
    clientConfig:
      service:
        name: webhook-service
        namespace: webhook-system
        path: "/mutate"
      caBundle: <base64-ca-cert>
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail          # Fail = reject request if webhook down
    timeoutSeconds: 5
---
# Validating webhook - runs AFTER all mutations, cannot modify
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: resource-quota-enforcer
webhooks:
  - name: quota.example.com
    clientConfig:
      service:
        name: webhook-service
        namespace: webhook-system
        path: "/validate"
      caBundle: <base64-ca-cert>
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1"]
    sideEffects: None
    failurePolicy: Fail
```

```go
// Minimal admission webhook server (Go)
func mutateHandler(w http.ResponseWriter, r *http.Request) {
    var review admissionv1.AdmissionReview
    body, _ := io.ReadAll(r.Body)
    json.Unmarshal(body, &review)

    pod := corev1.Pod{}
    json.Unmarshal(review.Request.Object.Raw, &pod)

    // Example: enforce resource requests if missing
    patches := []map[string]interface{}{}
    for i, c := range pod.Spec.Containers {
        if c.Resources.Requests == nil {
            patches = append(patches, map[string]interface{}{
                "op":   "add",
                "path": fmt.Sprintf("/spec/containers/%d/resources/requests", i),
                "value": map[string]string{"cpu": "100m", "memory": "128Mi"},
            })
        }
    }
    patchBytes, _ := json.Marshal(patches)
    patchType := admissionv1.PatchTypeJSONPatch

    review.Response = &admissionv1.AdmissionResponse{
        UID:       review.Request.UID,
        Allowed:   true,
        Patch:     patchBytes,
        PatchType: &patchType,
    }
    resp, _ := json.Marshal(review)
    w.Write(resp)
}
```

### API Priority and Fairness (APF)

```
PROBLEM: A misbehaving controller flooding the API server with
List requests can starve critical traffic (kubelet heartbeats,
node status updates) → cascading cluster failure.

SOLUTION: API Priority and Fairness (stable since 1.20)

┌────────────────────────────────────────────────────────────┐
│ PriorityLevelConfiguration: defines concurrency share         │
│ ├─ system (kubelet, controller-manager): highest priority     │
│ ├─ leader-election                                             │
│ ├─ workload-high                                                │
│ ├─ workload-low                                                 │
│ └─ catch-all (default bucket)                                 │
│                                                                │
│ FlowSchema: maps a request to a PriorityLevel + queue           │
│ ├─ matchingPrecedence: lower number = higher precedence         │
│ ├─ distinguisher: e.g. by user, by namespace (fair queuing)     │
│ └─ Each PriorityLevel has N queues; requests hash to a queue     │
│    by distinguisher → prevents one tenant from starving others  │
│                                                                │
│ Result: requests queued, executed with fair round-robin        │
│ within priority level; excess requests get 429 (Retry-After)   │
└────────────────────────────────────────────────────────────┘
```

```bash
# Inspect current APF config
kubectl get flowschemas
kubectl get prioritylevelconfigurations

# Check for rejected/queued requests (indicates overload)
kubectl get --raw /metrics | grep apiserver_flowcontrol_rejected_requests_total
kubectl get --raw /metrics | grep apiserver_flowcontrol_request_wait_duration_seconds
```

### Interview Questions — API Server

**Q1: Walk me through what happens when you run `kubectl apply -f deployment.yaml`.**
> See the request lifecycle diagram above. Key points to hit: client-side (kubectl computes 3-way merge patch using last-applied-configuration annotation), AuthN → AuthZ → Admission (mutating then validating) → schema validation → etcd write (with optimistic concurrency via resourceVersion) → watch notification to Deployment controller → ReplicaSet created → Scheduler assigns pod to node → kubelet pulls image and starts container.

**Q2: How does the API server handle 10,000 concurrent watch connections?**
> Each watch is a long-lived HTTP connection backed by a shared "watch cache" (in-memory ring buffer per resource type, size configurable via `--watch-cache-sizes`), NOT a direct etcd watch per client. The API server maintains ONE etcd watch per resource type and fans out events to all client watches from its in-memory cache. This scales because etcd load stays constant regardless of client count. Bookmark events periodically update resourceVersion without data, letting clients resume cheaply after disconnect.

**Q3: Design an admission webhook that enforces resource quotas.**
> ValidatingWebhookConfiguration matching CREATE/UPDATE on pods; webhook computes sum of container resource requests, looks up namespace ResourceQuota object, rejects with 4xx AdmissionResponse if exceeded. Discuss failurePolicy trade-off (Fail = safe but risks cluster-wide outage if webhook down; Ignore = available but bypassable) and namespaceSelector to exclude kube-system.

**Q4: Explain the difference between optimistic and pessimistic concurrency in Kubernetes.**
> Kubernetes uses optimistic concurrency exclusively — every object has `resourceVersion`. A client must include the RV it read when updating; if etcd's current RV differs (someone else updated first), the write is rejected with 409 Conflict, and the client must re-GET and retry (client-go's `RetryOnConflict` helper). There's no locking/blocking (pessimistic) because it wouldn't scale to thousands of controllers.

**Q5: What's the difference between `kubectl apply` and `kubectl replace`?**
> `apply` computes a 3-way strategic merge patch (last-applied-config annotation, current live object, new local file) enabling additive updates and field ownership tracking (Server-Side Apply, `fieldManager`). `replace` does a full PUT, overwriting the entire object — any fields not in your local file are removed.

**Q6: Explain Server-Side Apply and field ownership conflicts.**
> Since 1.16 (GA 1.22), the API server tracks which "manager" (client) owns which field via `managedFields`. Two controllers writing the same field trigger a 409 Conflict (unless `force=true`). This solves the old problem of `kubectl apply` clobbering fields set by HPA/other controllers (e.g., `replicas`).

**Q7: How would you debug "unable to connect to API server" from kubelet?**
> Check kubelet logs for TLS handshake errors (cert expiry — kubeadm certs expire 1yr by default), check `--kubeconfig` on the node, verify API server's `--bind-address` and load balancer health, check network policy/firewall between node and control plane, confirm clock skew (<5min, else cert validation fails).

**Q8: Explain the aggregation layer and why metrics-server uses it.**
> The `kube-aggregator` lets you extend the Kubernetes API surface without patching the core API server. `APIService` objects register additional API groups (e.g., `metrics.k8s.io`) that proxy to a separate Deployment (metrics-server). The main API server forwards matching requests to the aggregated API server over mTLS, checking `--requestheader-*` flags for identity propagation.

**Q9: What is the difference between `--authorization-mode=Node,RBAC` ordering?**
> Modes are tried in order; Node authorizer specifically restricts kubelets to read/write only objects related to their own node (their pods, their node object, related secrets/configmaps) — preventing a compromised kubelet from reading arbitrary secrets. RBAC handles everything else. Order matters because Node authorizer short-circuits for kubelet identities before falling through to RBAC for other principals.

**Q10: How do you rate-limit a noisy client without APF?**
> Pre-1.20 clusters used `--max-requests-inflight` and `--max-mutating-requests-inflight` (global, not fair). This caused the "noisy neighbor" problem where one client could exhaust the whole budget. APF (1.20+) is the correct modern answer — always mention it's the current best practice.

---

## 1.2 etcd Deep Dive

### Raft Consensus Protocol

```
RAFT LEADER ELECTION

Initial state: 3 nodes, all Followers, election timeout random(150-300ms)

┌────────┐      ┌────────┐      ┌────────┐
│ Node A │      │ Node B │      │ Node C │
│Follower│      │Follower│      │Follower│
└────────┘      └────────┘      └────────┘

t=0: Node B's election timer expires first (randomized to avoid ties)
     Node B → Candidate, increments term (term=1), votes for itself
     Sends RequestVote RPC to A and C

t=1: A and C receive RequestVote (term=1 > their term=0)
     They haven't voted this term → grant vote, reset own timers

t=2: Node B receives majority (2 of 3 votes, itself + A) → becomes Leader
     Node B starts sending periodic heartbeats (AppendEntries, empty)
     every ~50-100ms to maintain authority

If Leader crashes:
     Followers stop receiving heartbeats within election timeout
     → new election starts (term increments again)

WHY ODD NUMBERS (3/5/7)?
Quorum = floor(N/2)+1
├─ N=3: quorum=2, tolerates 1 node failure
├─ N=5: quorum=3, tolerates 2 node failures (better fault tolerance,
│       but more replication overhead — 5 way write ack)
├─ N=4: quorum=3, SAME fault tolerance as N=3 but WORSE performance
│       (never use even numbers — wasted resources, same tolerance
│        as N-1)
└─ N=7: quorum=4, tolerates 3 failures (used only for very large,
        critical clusters — high write latency cost)
```

### Log Replication

```
CLIENT WRITE (e.g., API server PUT /pods/nginx)
        │
        ▼
┌──────────────────────────────────────────────────────┐
│ 1. Client sends write to Leader (etcd redirects if     │
│    sent to Follower)                                    │
│ 2. Leader appends entry to its LOCAL log (uncommitted)  │
│ 3. Leader sends AppendEntries RPC to all Followers      │
│    (parallel, with previous log index+term for          │
│    consistency check)                                    │
│ 4. Each Follower appends to its local log, ACKs          │
│ 5. Leader waits for ACK from QUORUM (majority, not all)  │
│ 6. Once quorum ACKs → entry is COMMITTED                 │
│ 7. Leader applies to its state machine (BoltDB), returns │
│    success to client                                      │
│ 8. Leader includes new commit index in NEXT heartbeat    │
│    → Followers apply the entry to their own state machine│
└──────────────────────────────────────────────────────┘

KEY INSIGHT: Commit requires QUORUM ack, not ALL nodes. This is
why a 3-node cluster survives 1 node down — the other 2 form quorum.
If 2 of 3 are down, NO quorum → etcd rejects ALL writes (read-only
degraded state, or fully unavailable depending on which 2).
```

### How Kubernetes Objects Are Stored

```
etcd key structure (default prefix /registry):

/registry/pods/<namespace>/<name>
/registry/deployments/<namespace>/<name>
/registry/services/<namespace>/<name>
/registry/secrets/<namespace>/<name>
/registry/configmaps/<namespace>/<name>
/registry/nodes/<name>
/registry/events/<namespace>/<name>          # TTL'd, high churn

Value: protobuf-encoded object (NOT JSON — smaller, faster)
       (JSON only used over HTTP API; internal storage = protobuf)

# Direct etcd inspection (bypass API server - for debugging only)
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/pods/default/nginx --prefix

# Count objects by type (find what's bloating etcd)
etcdctl get /registry/pods --prefix --keys-only | wc -l
etcdctl get /registry/events --prefix --keys-only | wc -l
```

### MVCC and Watch

```
etcd uses Multi-Version Concurrency Control (MVCC):
├─ Every write creates a NEW revision (monotonically increasing,
│  cluster-wide, NOT per-key)
├─ Old revisions retained until COMPACTION removes them
├─ Kubernetes "resourceVersion" == etcd's mod revision
├─ Watches work by replaying the revision log from a starting point
│  → if that revision was already compacted, watch fails with
│    "401: too old resource version" → client must relist
│
COMPACTION: Without it, etcd's DB grows unbounded (every historical
version of every key retained forever) → causes the classic
"etcd db size alert" issue.

Compaction settings (kube-apiserver flag passed to etcd, or via
etcd's own --auto-compaction-mode=revision --auto-compaction-retention=1000)

DEFRAGMENTATION: Compaction only marks space as free internally;
the DB file on disk doesn't shrink until you defragment.
    etcdctl defrag --endpoints=https://127.0.0.1:2379 ...
⚠️ Defrag blocks writes on that member — do it one member at a time,
   during low-traffic windows, never all members simultaneously.
```

### Backup & Disaster Recovery

```bash
# SNAPSHOT BACKUP (the standard method)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot integrity
etcdctl snapshot status /backup/etcd-snapshot-*.db -w table

# RESTORE (creates new data-dir; requires updating etcd manifest)
etcdctl snapshot restore /backup/etcd-snapshot-*.db \
  --name etcd-1 \
  --initial-cluster etcd-1=https://10.0.0.1:2380,etcd-2=https://10.0.0.2:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls https://10.0.0.1:2380 \
  --data-dir /var/lib/etcd-restored

# Then update /etc/kubernetes/manifests/etcd.yaml to point
# --data-dir to the restored path, and restart kubelet on that node.
```

```
DISASTER SCENARIOS:

1. One member down (3-node cluster, 1 down)
   → Cluster fully operational, quorum maintained (2/3)
   → Fix: replace member, `etcdctl member remove` + `member add`

2. Two members down (3-node cluster, 2 down = only 1 left)
   → NO quorum → etcd rejects ALL writes, cluster effectively down
   → API server: writes fail, reads may work from stale cache briefly
   → Recovery: restore from latest snapshot backup, OR if you still
     have access to the surviving member's data-dir, use
     `etcdctl snapshot restore` with --force-new-cluster on that
     member's data to bootstrap a new single-node cluster, then
     add members back one at a time.

3. Complete data loss (all 3 members' disks lost)
   → MUST restore from external snapshot backup (this is why offsite,
     scheduled backups every 1hr+ are non-negotiable for production)
   → RPO = time since last successful snapshot
   → After restore: cluster state reverts to backup time — any pods/
     deployments created after backup are GONE from etcd, but their
     actual containers may still be running on nodes until GC'd.
```

### Interview Questions — etcd

**Q1: Your etcd cluster has 2 of 3 nodes down. What happens? How do you recover?**
> No quorum (need 2 of 3, only 1 remains) → cluster stops accepting writes; API server becomes read-only/degraded (some GETs may still be served from local watch cache, but anything requiring a fresh read or any write fails). Recovery: bring back at least 1 of the failed nodes if data is intact (fastest path — no data loss); if disks are lost, restore the healthiest surviving member using `--force-new-cluster`, verify data, then re-join other members as fresh members (not restoring their old data, to avoid split history).

**Q2: etcd is consuming 100GB disk. How do you investigate and fix?**
> Check `etcdctl endpoint status -w table` for `DB SIZE` vs `DB SIZE IN USE` — big gap = fragmentation, needs `etcdctl defrag`. If DB SIZE IN USE itself is huge, check for compaction not running (`--auto-compaction-retention` unset or too high), or find noisy resource types (`events` are the #1 culprit — high churn, short TTL but if compaction lags they pile up). Also check for excessive ConfigMap/Secret churn from a broken controller doing rapid updates.

**Q3: Design etcd backup strategy for 99.99% availability.**
> Automated snapshot every 1 hour to offsite storage (S3/Azure Blob) with a CronJob, retain 7 days rolling + weekly long-term. Test restores monthly on a scratch cluster (untested backups are not backups). Combine with etcd's own replication (3 or 5 members across AZs) for the "normal" failure case, and snapshots for catastrophic/human-error cases (accidental `kubectl delete namespace kube-system` isn't fixed by replication — the delete replicates too!). Monitor `etcd_disk_wal_fsync_duration_seconds` and `etcd_server_has_leader` as SLIs.

**Q4: Explain etcd MVCC and how it affects Kubernetes watches.**
> See MVCC section above. Key interview point: "too old resource version" errors happen because compaction removed history a watch needed to resume from — controllers must handle this by doing a full LIST (relist) instead of crashing, which is exactly what client-go's Reflector does automatically.

**Q5: Why should etcd run on dedicated fast disks (SSD, low latency)?**
> Every write must be fsync'd to disk WAL before being ack'd (durability requirement of Raft) — `etcd_disk_wal_fsync_duration_seconds` P99 should be <10ms; on slow/shared disks this balloons, causing heartbeat timeouts → false leader elections → cluster instability. This is why cloud providers recommend local NVMe/Premium SSD, not network-attached spinning disks, for etcd.

**Q6: What happens during defragmentation — is it safe to run in production?**
> Defrag rewrites the member's local bbolt file to reclaim fragmented space, but it BLOCKS that member from serving requests during the operation (can take seconds to minutes depending on DB size). Never defrag all members simultaneously — do it one at a time, verify member health before moving to next, ideally during a maintenance window with quorum from other 2 members maintaining availability.

**Q7: How do you monitor etcd health?**
> Key metrics: `etcd_server_has_leader` (should be 1), `etcd_server_leader_changes_seen_total` (spikes = instability), `etcd_disk_wal_fsync_duration_seconds`, `etcd_network_peer_round_trip_time_seconds`, `etcd_mvcc_db_total_size_in_bytes` vs `_in_use`. Alert on leader changes >0 in 10 min window and fsync p99 >100ms.

---

## 1.3 Scheduler Deep Dive

### Scheduling Algorithm: Filter → Score → Bind

```
POD CREATED (unscheduled, spec.nodeName empty)
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│ SCHEDULING CYCLE (per pod, serialized in default scheduler) │
│                                                              │
│ PHASE 1: FILTERING (find feasible nodes)                    │
│  Plugins run in sequence, any rejection removes the node:    │
│  ├─ PodFitsResources: node has enough allocatable CPU/mem/   │
│  │  ephemeral-storage/extended resources (GPU) left?          │
│  ├─ NodeAffinity: required node affinity rules match?         │
│  ├─ PodAffinity/AntiAffinity: co-location rules satisfied?    │
│  ├─ TaintToleration: pod tolerates all node taints?           │
│  ├─ NodeUnschedulable: node not cordoned?                     │
│  ├─ VolumeBinding: PVC can bind/is bound to a volume reachable │
│  │  from this node's topology (zone)?                          │
│  └─ InterPodAffinity, PodTopologySpread, etc.                 │
│                                                                │
│  Result: list of feasible nodes (could be empty → Pending)    │
│                                                                │
│ PHASE 2: SCORING (rank feasible nodes 0-100)                  │
│  Plugins run in parallel, weighted sum determines final score: │
│  ├─ NodeResourcesFit (default: LeastAllocated - spreads load,  │
│  │  or MostAllocated - bin-packs for cost savings)              │
│  ├─ ImageLocality: node already has the image? (faster start)  │
│  ├─ InterPodAffinity: soft affinity/anti-affinity preferences  │
│  ├─ NodeAffinity: soft (preferred) node affinity                │
│  ├─ PodTopologySpread: even distribution across zones/nodes     │
│  └─ TaintToleration: soft toleration preferences                │
│                                                                │
│  Winner = highest total weighted score (ties broken randomly)  │
│                                                                │
│ PHASE 3: BINDING                                               │
│  Scheduler writes pod.spec.nodeName via a Binding API call      │
│  (this is itself just another apply that goes through the       │
│   whole API server admission chain)                              │
└──────────────────────────────────────────────────────────────────┘

Total: 1-100ms per pod for small clusters; scheduler uses a cache
+ percentageOfNodesToScore optimization for 1000+ node clusters
(doesn't evaluate ALL nodes, samples a percentage once enough
feasible nodes are found — trades optimality for scale).
```

### Resource Calculation

```
Node Allocatable = Node Capacity - kube-reserved - system-reserved - eviction-threshold

Example (Standard_D4s_v3: 4 vCPU, 16GB RAM):
Capacity:        cpu=4000m,  memory=16Gi
kube-reserved:   cpu=100m,   memory=1Gi    (kubelet, container runtime)
system-reserved: cpu=100m,   memory=512Mi  (sshd, systemd, OS)
eviction-hard:   memory.available<500Mi     (reserved buffer)
─────────────────────────────────────────────────────────
Allocatable:     cpu=3800m,  memory=14Gi (approx)

Scheduler uses SUM of pod.spec.containers[].resources.requests
(NOT limits) when filtering — this is why "requests" is the
scheduling-relevant number, and why overcommitting via generous
limits but small requests packs more pods per node (Burstable QoS).
```

### Affinity, Anti-Affinity, Taints & Tolerations

```yaml
# Node affinity - hard requirement + soft preference
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: accelerator
            operator: In
            values: ["nvidia-tesla-v100"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: ["us-east-1a"]
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["gpu-training"]
        topologyKey: "kubernetes.io/hostname"  # never co-locate on same node
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  containers:
  - name: trainer
    image: ml-trainer:latest
    resources:
      limits:
        nvidia.com/gpu: 1
---
# Pod Topology Spread Constraints (even distribution across zones)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 9
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule   # or ScheduleAnyway
        labelSelector:
          matchLabels:
            app: web
```

### Scheduler Preemption

```
When a high-priority pod cannot be scheduled (no feasible node):

1. Scheduler checks: is there a node where evicting SOME lower-
   priority pods would make room? (PriorityClass.value determines
   ranking)
2. If found: victim pods are chosen (minimal set needed), a
   "nominated node" is set on the pending pod
   (status.nominatedNodeName)
3. Victims get a graceful termination (respecting terminationGracePeriodSeconds)
4. Once victims are gone, the preempting pod is scheduled to that node

PriorityClass example:
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority  # or "Never" (won't preempt, just waits)
description: "Critical production workloads"

⚠️ Preemption does NOT guarantee the preempting pod lands on THAT
node — another pod could win the race. It's best-effort.
```

### Custom Schedulers & Extenders

```
OPTIONS FOR CUSTOM SCHEDULING LOGIC:

1. Second scheduler (schedulerName field)
   - Deploy a full second kube-scheduler binary/pod with a different
     --scheduler-name; pods set spec.schedulerName to opt in.
   - Full control but must reimplement everything you need.

2. Scheduler Extender (legacy, HTTP webhook)
   - Configure via --policy-config-file, scheduler calls out to an
     external HTTP service during Filter/Prioritize/Bind phases.
   - Simpler than a second scheduler but adds network hop latency
     to every scheduling decision. Being phased out in favor of...

3. Scheduler Framework Plugins (modern, in-process, Go)
   - Compile custom plugins into a scheduler binary implementing
     interfaces like Filter, Score, Reserve, Permit, PreBind, Bind.
   - Fastest (in-process, no network hop), used by real production
     systems (e.g., Kubernetes' own `default-scheduler` is built this
     way; Volcano/YuniKorn gang-schedulers extend this framework).

4. Descheduler (CronJob-based, post-hoc rebalancing)
   - Runs periodically, evicts pods violating policies (e.g.
     LowNodeUtilization, pod affinity violated after node label
     change) so they get RE-scheduled by the normal scheduler.
   - Doesn't schedule directly, just triggers re-scheduling.

GANG SCHEDULING (for ML - all-or-nothing):
Default scheduler schedules pods ONE AT A TIME — for a distributed
training job needing 8 pods with 1 GPU each simultaneously, this can
deadlock (4 scheduled, then cluster fills with other pods, remaining
4 pods wait forever, wasting the 4 already-scheduled GPUs).
Solution: Volcano or Kueue — implement "PodGroup" semantics where
the group is scheduled atomically (all pods placed, or none).
```

### Interview Questions — Scheduler

**Q1: Pod is Pending for 10 minutes. Walk through your debugging process.**
> `kubectl describe pod` → check Events for "FailedScheduling" and the specific predicate reason (insufficient cpu/memory, node affinity, taints). Then `kubectl get nodes -o wide` + `kubectl describe nodes` to check allocatable vs allocated resources, taints. Check for PVC binding issues (VolumeBinding filter) if the pod uses storage. Check PriorityClass — is a higher priority pod supposed to preempt but preemption is disabled? Check scheduler logs (`kubectl logs -n kube-system kube-scheduler-xxx`) for detailed filter rejection reasons.

**Q2: Design scheduling for ML training jobs requiring 8 GPUs together (gang scheduling).**
> Default scheduler can't do all-or-nothing. Use Volcano or Kueue: define a PodGroup/Job with `minMember: 8`; the gang scheduler holds partial allocations in a "pending" state until all 8 can be placed simultaneously, preventing deadlock/resource fragmentation. Combine with node affinity for GPU node pools and topology spread to keep pods on nodes with high-bandwidth interconnect (e.g., same NVLink domain).

**Q3: How would you implement bin-packing vs spreading across nodes?**
> Bin-packing (cost optimization, fewer nodes active → scale down more aggressively): use `NodeResourcesFit` scoring strategy `MostAllocated`. Spreading (resilience, blast-radius reduction): use `LeastAllocated` (default) plus `PodTopologySpread` with zone/hostname topology keys and `maxSkew: 1`. Trade-off: bin-packing risks noisy-neighbor and reduces failure-domain isolation; spreading costs more (more nodes needed to keep headroom).

**Q4: Explain scheduler preemption — when and how does it work?**
> Only triggers when a pod can't be scheduled AND has a PriorityClass higher than victim candidates. Scheduler computes minimal victim set on a candidate node, evicts them gracefully, sets `nominatedNodeName`. Not guaranteed — another pod can grab the freed resources first. `preemptionPolicy: Never` on a PriorityClass disables preemption for pods using it (they just wait in queue, "polite" high priority).

**Q5: Node has 4 CPU allocatable, currently 3.5 CPU requested by existing pods. Why won't a pod requesting 200m schedule?**
> Check pod's actual full resource request (all containers + init containers use max(sum of containers, max of any single init container) logic), also check for DaemonSet pod overhead not yet accounted, or the node might have a taint the pod doesn't tolerate, or the 500Mi eviction threshold reservation is being counted against allocatable — verify with `kubectl describe node` "Allocated resources" section directly, not just capacity.

**Q6: How does `percentageOfNodesToScore` help scheduler scale to 5,000 nodes?**
> Instead of scoring EVERY feasible node (expensive at scale), the scheduler stops evaluating once it's found enough good candidates (default formula scales down the percentage as cluster size grows, e.g., ~10% for huge clusters). This trades finding the theoretically optimal node for bounded scheduling latency — acceptable because differences between "good enough" nodes are usually marginal.

**Q7: Explain `requiredDuringSchedulingIgnoredDuringExecution` — what does "IgnoredDuringExecution" mean?**
> The affinity rule is only checked AT SCHEDULING TIME. If node labels change after the pod is running (making it no longer satisfy the affinity rule), Kubernetes does NOT evict the running pod — it "ignores" the violation during execution. This is why label changes don't cause mass pod evictions, but also means a Descheduler is needed if you want to actively re-balance.

---

## 1.4 Controller Manager Deep Dive

### Controller Pattern (Reconciliation Loop)

```
LEVEL-TRIGGERED RECONCILIATION (not edge-triggered!)

┌──────────────────────────────────────────────────────────┐
│  for {                                                       │
│      desired := getDesiredState()   // from etcd via Informer│
│      current := getCurrentState()   // from etcd via Informer│
│      if desired != current {                                 │
│          reconcile(desired, current)  // take corrective action│
│      }                                                        │
│      wait(resyncPeriod or triggered by watch event)          │
│  }                                                             │
└──────────────────────────────────────────────────────────────┘

WHY LEVEL-TRIGGERED (not edge/event-triggered)?
If a controller ONLY reacted to individual events ("pod deleted" →
"create replacement"), a missed/dropped event = permanent
inconsistency. Level-triggered means EVERY reconcile recomputes
the FULL diff between desired and actual state, self-healing any
missed events on the next periodic resync (default informer
resyncPeriod ~ every 10 hours, PLUS immediate trigger on watch
events). This is why Kubernetes controllers are robust to
temporary API server unavailability or missed messages.

WORKQUEUE PATTERN:
Informer event handler does NOT do the work directly — it just
enqueues the object's key (namespace/name):
    informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            key, _ := cache.MetaNamespaceKeyFunc(obj)
            workqueue.Add(key)
        },
    })
Workers pop keys off the queue, RE-FETCH current object from the
Indexer/lister (not the possibly-stale object from the event),
then reconcile. This decouples event rate from processing rate and
enables:
├─ Deduplication (rapid updates to the same object = 1 queue entry)
├─ Rate limiting (exponential backoff per key on repeated failures)
└─ Parallelism (multiple workers, safe because reconcile is idempotent)
```

### Leader Election (HA Controller Manager)

```
Multiple controller-manager replicas run for HA, but only ONE must
be ACTIVE (otherwise duplicate reconciliation/race conditions).

Mechanism: Lease API object (coordination.k8s.io/v1)
├─ Each replica tries to acquire/renew a Lease object
│  (holderIdentity, leaseDurationSeconds ~15s, renewTime)
├─ Leader renews lease every ~2s (renewDeadline ~10s)
├─ If leader crashes, lease expires after leaseDuration → another
│  replica acquires it (becomes new leader)
└─ Non-leader replicas sit idle (or serve read-only/health endpoints)

kubectl get lease -n kube-system kube-controller-manager -o yaml
kubectl get lease -n kube-system kube-scheduler -o yaml
```

### Built-in Controllers

```
DEPLOYMENT CONTROLLER (RollingUpdate algorithm):

Deployment (desired: 5 replicas, image v2) manages ReplicaSets:
├─ ReplicaSet-v1 (old, currently 5/5 running)
└─ ReplicaSet-v2 (new, currently 0/5) ← created on update

RollingUpdate with maxSurge=25%, maxUnavailable=25% (defaults):
maxSurge    = ceil(5 * 0.25) = 2  (can have up to 5+2=7 pods total)
maxUnavailable = floor(5 * 0.25) = 1  (at least 5-1=4 must be ready)

Step-by-step:
1. Scale RS-v2 up by maxSurge (2): RS-v1=5, RS-v2=2 (total=7, ok ≤7)
2. Wait for RS-v2 pods to become Ready (respects readinessProbe!)
3. Scale RS-v1 down by maxUnavailable (1): RS-v1=4, RS-v2=2 (total=6)
4. Repeat: scale RS-v2 up, wait ready, scale RS-v1 down...
5. Continue until RS-v1=0, RS-v2=5

This is why a broken readinessProbe on the new version HALTS the
rollout indefinitely (new pods never become Ready, so old pods are
never scaled down) — a built-in safety mechanism against bad
rollouts, visible as "stuck at X% rollout."

STATEFULSET CONTROLLER (ordered, stable identity):
├─ Pods named <name>-0, <name>-1, <name>-2 (NOT random suffixes)
├─ Creation order: 0, then 1 (only after 0 is Ready), then 2...
├─ Deletion order: REVERSE (2, then 1, then 0)
├─ Each pod gets a stable network identity via headless Service:
│  <name>-0.<service>.<namespace>.svc.cluster.local
├─ Each pod gets its OWN PVC (via volumeClaimTemplates), which
│  persists across pod rescheduling (same PVC reattached, NOT a
│  fresh volume) — this is how StatefulSet gives "stable storage"
└─ podManagementPolicy: OrderedReady (default) vs Parallel (faster,
   but loses ordering guarantees — ok for stateless-ish clustered
   apps like Cassandra which handle concurrent joins themselves)

JOB/CRONJOB CONTROLLER:
├─ Job tracks .status.succeeded / .status.failed against
│  .spec.completions and .spec.parallelism
├─ backoffLimit: number of retries before marking Job Failed
│  (exponential backoff between retries: 10s, 20s, 40s... capped
│   at 6 minutes)
├─ activeDeadlineSeconds: hard timeout for the whole Job
└─ CronJob controller creates Job objects on schedule; concurrencyPolicy
   (Allow/Forbid/Replace) governs overlap behavior;
   startingDeadlineSeconds handles missed schedules (e.g. controller
   was down) — if exceeded, that run is skipped, counted as a "miss."
```

### Writing a Custom Controller (client-go)

```go
package main

import (
    "context"
    "fmt"
    "time"

    corev1 "k8s.io/api/core/v1"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/tools/leaderelection"
    "k8s.io/client-go/tools/leaderelection/resourcelock"
    "k8s.io/client-go/util/workqueue"
)

type Controller struct {
    clientset kubernetes.Interface
    queue     workqueue.RateLimitingInterface
    informer  cache.SharedIndexInformer
}

func NewController(clientset kubernetes.Interface) *Controller {
    factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
    podInformer := factory.Core().V1().Pods().Informer()
    queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

    c := &Controller{clientset: clientset, queue: queue, informer: podInformer}

    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            key, err := cache.MetaNamespaceKeyFunc(obj)
            if err == nil {
                queue.Add(key)
            }
        },
        UpdateFunc: func(old, new interface{}) {
            key, err := cache.MetaNamespaceKeyFunc(new)
            if err == nil {
                queue.Add(key)
            }
        },
        DeleteFunc: func(obj interface{}) {
            key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
            if err == nil {
                queue.Add(key)
            }
        },
    })
    return c
}

func (c *Controller) Run(stopCh <-chan struct{}) {
    defer c.queue.ShutDown()
    go c.informer.Run(stopCh)
    cache.WaitForCacheSync(stopCh, c.informer.HasSynced)
    go c.worker()
    <-stopCh
}

func (c *Controller) worker() {
    for c.processNextItem() {
    }
}

func (c *Controller) processNextItem() bool {
    key, quit := c.queue.Get()
    if quit {
        return false
    }
    defer c.queue.Done(key)

    err := c.reconcile(key.(string))
    if err != nil {
        c.queue.AddRateLimited(key) // exponential backoff retry
    } else {
        c.queue.Forget(key)
    }
    return true
}

func (c *Controller) reconcile(key string) error {
    obj, exists, err := c.informer.GetIndexer().GetByKey(key)
    if err != nil {
        return err
    }
    if !exists {
        fmt.Printf("Pod %s deleted\n", key)
        return nil
    }
    pod := obj.(*corev1.Pod)
    fmt.Printf("Reconciling pod %s, phase=%s\n", pod.Name, pod.Status.Phase)
    // ... business logic here (idempotent!) ...
    return nil
}

// Leader election wrapper for HA deployment
func runWithLeaderElection(clientset kubernetes.Interface, controller *Controller) {
    lock := &resourcelock.LeaseLock{
        LeaseMeta: metav1.ObjectMeta{Name: "my-controller-lock", Namespace: "default"},
        Client:    clientset.CoordinationV1(),
        LockConfig: resourcelock.ResourceLockConfig{Identity: "pod-hostname"},
    }
    leaderelection.RunOrDie(context.Background(), leaderelection.LeaderElectionConfig{
        Lock:            lock,
        LeaseDuration:   15 * time.Second,
        RenewDeadline:   10 * time.Second,
        RetryPeriod:     2 * time.Second,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                stopCh := make(chan struct{})
                controller.Run(stopCh)
            },
            OnStoppedLeading: func() { fmt.Println("lost leadership, exiting") },
        },
    })
}
```

### Interview Questions — Controller Manager

**Q1: A Deployment is stuck at 50% rollout. Debug it.**
> `kubectl rollout status deployment/x` and `kubectl get rs` to see old vs new ReplicaSet counts. `kubectl describe pod <new-pod>` — almost always a failing readinessProbe (app not truly healthy) or ImagePullBackOff on the new image blocking `maxUnavailable` from proceeding further. Check `kubectl rollout history` and consider `kubectl rollout undo` if it's a bad release. Also check PodDisruptionBudget — if minAvailable is set too strict, it can block the old ReplicaSet from scaling down.

**Q2: Explain how StatefulSet maintains pod identity during updates.**
> Pod name is deterministic (`<name>-N`) and stable across restarts/reschedules. Each ordinal has a dedicated PVC created from `volumeClaimTemplates` (named `<claim>-<name>-N`) that is NOT deleted when the pod is deleted — it's reattached to the recreated pod with the same ordinal. Network identity via a headless Service gives a stable stable DNS name per pod. Updates (RollingUpdate strategy) proceed in reverse ordinal order (N down to 0) by default, one at a time.

**Q3: Design a controller that auto-scales based on custom metrics.**
> Either use the existing HPA with a Custom Metrics API (Prometheus Adapter exposing `external.metrics.k8s.io`), OR write a custom controller: Informer watches your CRD (e.g., `ScaledObject`), reconcile loop queries your metric source (Prometheus query, queue depth, etc.), computes desired replicas, calls `scale` subresource via `clientset.AppsV1().Deployments(ns).UpdateScale()`. Mention KEDA as the production-grade version of this pattern (event-driven autoscaling to/from zero).

**Q4: What happens if controller-manager crashes during a rollout?**
> Because reconciliation is level-triggered and idempotent, a new leader (via leader election) picks up EXACTLY where the desired vs actual state comparison left off — no special "resume" logic needed. On informer resync, the new leader recomputes the full diff and continues scaling RS-v1 down / RS-v2 up until desired state matches. Brief pause in rollout progress (bounded by leaseDuration ~15s) but no corruption or duplicate work due to idempotency.

**Q5: Why do controllers re-fetch the object from the Lister instead of using the object passed to the event handler?**
> The object at enqueue time can be stale by the time a worker actually processes it (queue may have items queued for a while, or dedup collapsed multiple updates into one queue entry). Re-fetching from the Indexer/Lister (in-memory cache, cheap, no API call) ensures reconciliation always acts on the latest known state, preventing "lost update" bugs.

**Q6: How would you detect and prevent a "hot loop" where a controller keeps re-triggering itself?**
> Common cause: controller's own writes to status/annotations trigger its own Update watch event, causing infinite reconcile loops. Fix: use `Status().Update()` subresource writes (doesn't bump generation the same way spec does, watch handlers can filter on `ResourceVersion`/`Generation` changes), compare old vs new deep-equal before writing (no-op if nothing changed), and use rate-limited workqueues so even a loop is throttled rather than hammering the API server.
