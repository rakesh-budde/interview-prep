# Scaling & Performance (5% of Interview Weight)

> HPA/VPA/Cluster Autoscaler internals, resource model, QoS classes

---

## 6.1 Autoscaling

### Horizontal Pod Autoscaler (HPA)

```
ALGORITHM (control loop, evaluated every --horizontal-pod-autoscaler-
sync-period, default 15s):

desiredReplicas = ceil(currentReplicas * (currentMetricValue / desiredMetricValue))

Example: 4 replicas, current avg CPU = 150m, target = 100m (per pod)
desiredReplicas = ceil(4 * (150/100)) = ceil(6.0) = 6 replicas

METRICS SOURCES:
├─ resource (CPU/memory): via metrics-server (aggregates from kubelet
│  cAdvisor stats, exposed via metrics.k8s.io aggregated API)
├─ Pods (custom, e.g. requests-per-second per pod): via Custom Metrics
│  API, typically backed by prometheus-adapter translating PromQL
│  queries into the API server's custom.metrics.k8s.io endpoint
├─ Object (e.g. Ingress requests, queue depth of an external object):
│  same custom metrics pipeline, but keyed to a specific object not
│  averaged per-pod
└─ External (metric NOT associated with any K8s object at all, e.g.
   an SQS queue depth, or a cloud pub/sub backlog): external.metrics.k8s.io
   API, via prometheus-adapter or cloud-specific adapters

STABILIZATION WINDOW (prevents flapping):
├─ scaleUp stabilizationWindowSeconds (default 0 — scale up FAST)
├─ scaleDown stabilizationWindowSeconds (default 300s = 5min — scale
│  down SLOWLY) — HPA looks at the MAX recommended replica count
│  over the whole stabilization window before actually scaling down,
│  preventing a brief metric dip from causing premature scale-in
│  followed immediately by having to scale back up (thrashing)

behavior:
  scaleDown:
    stabilizationWindowSeconds: 300
    policies:
    - type: Percent
      value: 10          # scale down by at most 10% of current per period
      periodSeconds: 60
  scaleUp:
    stabilizationWindowSeconds: 0
    policies:
    - type: Percent
      value: 100         # can double replica count per period
      periodSeconds: 15
    - type: Pods
      value: 4           # or add up to 4 pods per period
      periodSeconds: 15
    selectPolicy: Max      # use whichever policy allows MORE scale-up
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

### Vertical Pod Autoscaler (VPA)

```
VPA recommends (or auto-applies) resource REQUESTS/LIMITS based on
observed historical usage — solves the "guessing game" of setting
requests correctly by hand.

Components:
├─ Recommender: analyzes historical usage (via metrics-server/
│  Prometheus history), computes recommended requests using a
│  decaying-weight histogram algorithm (recent usage weighted more
│  heavily, but retains longer-term percentile data too)
├─ Updater: if updateMode=Auto/Recreate, evicts pods whose current
│  requests deviate significantly from the recommendation (forcing
│  a recreate with new values applied by...)
└─ Admission webhook: intercepts pod creation, mutates resource
   requests/limits to match the current recommendation

updateMode options:
├─ Off: recommendation-only (view via `kubectl describe vpa`, apply
│  manually) — SAFEST, always start here to observe before automating
├─ Initial: only applied when pod is FIRST created, never updates a
│  running pod
├─ Recreate: evicts + recreates pods to apply new recommendations
│  (causes disruption — combine with PDB)
└─ Auto: same as Recreate currently (in-place resize without restart
   is a newer alpha/beta feature in recent Kubernetes versions,
   `InPlacePodVerticalScaling`, not GA as of most production clusters)

⚠️ CRITICAL GOTCHA: VPA and HPA MUST NOT both control CPU/memory-based
scaling for the SAME workload simultaneously — they will fight each
other (VPA changes per-pod resources, HPA reacts to the resulting
utilization % change, adjusting replica count, which changes
aggregate load per pod, which VPA reacts to again...). If using both,
VPA should target only resources HPA does NOT scale on (e.g. VPA for
memory, HPA for a custom metric like requests/sec), or use VPA in
"Off" mode purely for recommendations while HPA does the actual
autoscaling action.
```

### Cluster Autoscaler

```
SCALE-UP TRIGGER: a pod is Pending because NO existing node has
enough allocatable resources to fit it (scheduler already tried and
failed) → Cluster Autoscaler (CA) simulates: "if I added a node from
node-group X, would this pod become schedulable?" → if yes for some
group, CA calls the cloud provider API to add a node (e.g. increase
VMSS/ASG desired count) → new node joins, kubelet registers, pod
gets scheduled normally.

Typical scale-up latency: 30s-3min (cloud VM provisioning time is
the dominant factor) — this is why for latency-sensitive bursty
workloads, some teams keep a small buffer of "warm" over-provisioned
capacity (e.g. via low-priority "placeholder" pods that get preempted
instantly when real workload pods need the room) rather than relying
purely on reactive scale-up.

SCALE-DOWN CONDITIONS (a node is a scale-down candidate if, for a
sustained default 10 minutes):
├─ Node utilization below threshold (default 50% for both CPU and
│  memory simultaneously — configurable via --scale-down-utilization-
│  threshold)
├─ ALL pods on the node COULD be rescheduled elsewhere (no pod
│  blocking removal — see exceptions below)
└─ No scale-down in progress already for this node group (avoid
   thrashing)

PODS THAT BLOCK SCALE-DOWN (by default):
├─ Pods with no controller (bare pods) — CA doesn't know how to
│  safely recreate them elsewhere
├─ Pods with local storage (emptyDir with data CA can't migrate)
├─ Pods with restrictive PodDisruptionBudgets that would be violated
├─ Pods annotated `cluster-autoscaler.kubernetes.io/safe-to-evict:
│  "false"` (explicit opt-out, e.g. for a pod doing critical
│  irreplaceable work)
└─ kube-system pods NOT managed by a DaemonSet (unless
   --skip-nodes-with-system-pods=false)

NODE GROUP / NODE POOL config: CA operates per node group (e.g. one
ASG per instance type/AZ combination) — --expander flag controls
WHICH group to scale when multiple groups could satisfy a pending
pod: `random`, `least-waste` (picks the group leaving least unused
capacity after the pod is placed), `priority` (explicit ranked
preference, e.g. prefer cheaper spot node group first), `price`
(cloud cost-aware selection).
```

### KEDA (Event-Driven Autoscaling)

```
KEDA extends HPA to scale based on EVENT SOURCE metrics (queue
depth, Kafka lag, cron schedule) AND crucially can scale TO/FROM
ZERO (native HPA cannot scale below 1 replica) — ideal for bursty,
intermittent workloads (batch processors, message consumers) where
paying for idle replicas 24/7 is wasteful.

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-worker-scaler
spec:
  scaleTargetRef:
    name: queue-worker
  minReplicaCount: 0            # scale to zero when queue empty!
  maxReplicaCount: 100
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: work-items
      messageCount: "5"          # target: 5 messages per replica
```

### Interview Questions — Autoscaling

**Q1: Design autoscaling for bursty traffic (10x spikes).**
> Combine HPA (fast scaleUp policy, no stabilization delay on scale-up) targeting a leading-indicator custom metric (requests/sec or queue depth — reacts FASTER than CPU which lags behind actual load by the time a pod is under enough load to show elevated CPU) with Cluster Autoscaler pre-configured with sufficient max node count headroom. For truly extreme/instant spikes, consider over-provisioning a buffer via low-priority placeholder pods (instantly preempted) to mask cloud VM provisioning latency (30s-3min), and use KEDA if the trigger is genuinely event/queue-based rather than HTTP traffic.

**Q2: HPA is not scaling. Debug it.**
> `kubectl describe hpa` shows current vs target metric AND any error conditions (e.g. "unable to get metrics" — metrics-server down or not installed, or custom metrics adapter misconfigured). Check `kubectl top pods` works at all (confirms metrics-server health independently). Check the Deployment's pods actually HAVE resource requests set (HPA utilization % is calculated against REQUESTS — if requests aren't set, HPA can't compute utilization and will show "unknown" metric value, a very common misconfiguration). Check `minReplicas`/`maxReplicas` bounds aren't already at the limit. Check stabilization window isn't just delaying an expected scale-down (not a bug, working as intended).

**Q3: Compare HPA vs KEDA for event-driven scaling.**
> HPA natively supports resource metrics and (via adapters) custom/external metrics, but cannot scale below 1 replica and typically needs the metric to be somewhat "per-pod averageable" (Utilization/AverageValue types) which fits HTTP-request-style workloads well. KEDA is purpose-built for EVENT sources (Kafka lag, SQS/ServiceBus queue depth, cron schedules, 50+ built-in scalers) and its key differentiator is scale-to-ZERO support (via an internal small polling component that keeps watching the event source even at 0 replicas, then triggers scale-up when events appear) — ideal for cost-sensitive, bursty, non-HTTP workloads. In practice, KEDA actually CREATES an HPA object under the hood for non-zero scaling — it's an enhancement layered on top of HPA, not a full replacement.

**Q4: Why shouldn't VPA and HPA target the same metric on the same workload?**
> They form a feedback loop / control conflict: VPA adjusts each pod's resource allocation based on observed usage, which changes the utilization PERCENTAGE HPA measures (same absolute usage against a different requests baseline = different % reading), causing HPA to react by changing replica count, which changes per-pod load, which VPA then reacts to again — resulting in oscillation/instability rather than convergence. Mitigation: use VPA in recommendation-only mode for CPU/memory while HPA handles actual replica scaling, or have VPA manage a DIFFERENT resource dimension HPA doesn't scale on.

---

## 6.2 Resource Management

### Resource Model

```
CPU: measured in "cores" or millicores (1000m = 1 full core)
├─ REQUESTS: guaranteed minimum, used by scheduler for bin-packing
│  decisions (sum of requests must fit in node allocatable)
├─ LIMITS: hard ceiling enforced via CFS (Completely Fair Scheduler)
│  quota/period mechanism in the kernel cgroup — NOT a kill, just
│  THROTTLING (process gets zero CPU time for the remainder of each
│  100ms period once it exceeds its quota within that period)
└─ CPU THROTTLING can happen even when a node has spare idle CPU
   capacity — it's a per-container HARD cap enforced regardless of
   what else is happening on the node, a very common source of
   "why is my app slow despite low node CPU usage" confusion. Debug
   via `container_cpu_cfs_throttled_periods_total` metric (cAdvisor/
   Prometheus) — if this is climbing, your limit is too restrictive
   for actual burst needs.

MEMORY: measured in bytes (Mi/Gi suffixes)
├─ REQUESTS: guaranteed, used for scheduling fit
├─ LIMITS: hard ceiling — exceeding it triggers the kernel OOM
│  killer to SIGKILL the process immediately (NOT throttled like
│  CPU — memory can't be "throttled", you either have it or you
│  don't, so exceeding = kill)
└─ Unlike CPU, there's no "borrowing spare capacity" grace for
   memory limits — hitting the limit is immediate termination,
   which is why memory limits need MORE conservative headroom
   than CPU limits typically do.

EXTENDED RESOURCES (e.g. nvidia.com/gpu): integer-only, NO
overcommit allowed (no "limits > requests" concept — request must
equal limit exactly for extended resources), allocated exclusively
per container (device plugin framework manages actual device
assignment to containers).
```

### QoS Classes

```
GUARANTEED: EVERY container in the pod has requests == limits for
  BOTH cpu AND memory (exact values specified, not just "set to
  something"). Highest priority — LAST to be OOM-killed/evicted
  under node pressure.

BURSTABLE: at least one container has requests set, but requests !=
  limits for at least one resource (or limits unset while requests
  set) — the common "reasonable middle ground" (e.g. requests=200m,
  limits=1000m — allows bursting above baseline when node has spare
  capacity, without guaranteeing it). Medium eviction priority.

BESTEFFORT: NO requests or limits set AT ALL, for any container in
  the pod. Lowest priority — FIRST to be OOM-killed/evicted under
  any node memory pressure, since the kernel/kubelet has no
  guarantee to honor for these at all.

EVICTION ORDER under node memory pressure (kubelet's node-pressure
eviction manager): BestEffort pods evicted first, then Burstable
pods EXCEEDING their requests (ranked by how far over they are —
the "most over-consuming relative to its request" burstable pod
goes first), Guaranteed pods evicted LAST (and only if the node
itself is critically starved even for guaranteed workloads — rare,
usually indicates genuine node-level resource exhaustion beyond
what was accounted for).

OOM SCORE ADJUSTMENT: kubelet sets Linux `oom_score_adj` per
container reflecting this QoS priority — BestEffort gets the
highest (most-likely-to-be-killed) score, Guaranteed gets the
lowest (least-likely) — this is literally how the kernel OOM killer
picks its victim when GLOBAL (not per-cgroup limit) memory pressure
occurs on the node.
```

```yaml
# Guaranteed QoS example
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"        # EXACTLY equal to requests
    memory: "512Mi"    # EXACTLY equal to requests
```

### ResourceQuota & LimitRange

```yaml
# Namespace-level cap (ResourceQuota) — prevents one team from
# exhausting the whole cluster
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    limits.cpu: "100"
    limits.memory: 200Gi
    pods: "200"
    persistentvolumeclaims: "20"
---
# Default/min/max per-container (LimitRange) — fills in requests
# for pods that don't specify them (prevents accidental BestEffort)
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "4"
      memory: "8Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
```

### Interview Questions — Resource Management

**Q1: Pod is being OOM killed. Investigate and fix.**
> `kubectl describe pod` shows `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`. Check actual memory usage pattern over time (`kubectl top pod`, or better, Prometheus/Grafana historical graph) to distinguish a genuine memory LEAK (usage climbs steadily, never plateaus — needs an application fix) from legitimately needing more memory for its workload (usage plateaus at a higher-than-expected but STABLE level — just raise the limit). Check for a recent deployment/config change correlating with onset. If it's a leak, use language-specific profiling (pprof for Go, heap dumps for JVM) to find the root cause rather than just repeatedly raising limits as a band-aid.

**Q2: Design resource allocation for a multi-tenant cluster.**
> Per-namespace (per-tenant) ResourceQuota capping total requests/limits and object counts, LimitRange providing sane per-container defaults (prevents an accidental BestEffort pod from a forgotten resources block, which could then be trivially evicted or, worse, contribute to noisy-neighbor CPU contention). Use Guaranteed QoS for critical/latency-sensitive shared infrastructure, Burstable for general application workloads (majority case), and explicitly steer genuinely low-priority/batch work to BestEffort only if eviction tolerance is acceptable. Consider separate node pools/taints per tenant tier if strict physical isolation (not just accounting) is required.

**Q3: Explain CPU throttling vs memory OOM.**
> CPU limit violations result in THROTTLING (CFS quota mechanism zeroes out remaining CPU time for the container for the rest of each ~100ms accounting period) — the process keeps running, just slower/stalled intermittently, which shows up as elevated latency/tail-latency rather than a crash. Memory limit violations result in IMMEDIATE KILL (OOM killer SIGKILL) because memory can't be "throttled" the way CPU time can — you either have the memory allocated or you don't. This asymmetry means CPU limits should generally be set more generously/loosely (burst headroom) while memory limits need careful, closer-to-actual-usage sizing since overshooting is catastrophic (restart) rather than just slow.

**Q4: Why might a pod be slow even when `kubectl top node` shows the node has spare CPU capacity?**
> CPU limits enforce a HARD per-container cap via cgroup CFS quota regardless of overall node utilization — a container can be throttled purely because IT individually exceeded its own quota within a 100ms window, even if 90% of the node's total CPU sits idle. This is a very common and counter-intuitive debugging trap; confirm via the `container_cpu_cfs_throttled_periods_total` / `container_cpu_cfs_periods_total` ratio in Prometheus — a high throttled-period ratio confirms this exact scenario, and the fix is raising (or removing) that specific container's CPU limit, not adding more nodes.
