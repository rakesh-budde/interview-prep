# Workloads Lifecycle — Pods, Deployments, StatefulSets, DaemonSets

> Container lifecycle internals, probe mechanics, deployment strategies

---

## 5.1 Pod Lifecycle

### Pod Phases & Container States

```
POD PHASES (pod.status.phase — coarse, rarely enough to debug alone):
├─ Pending: accepted by API server, not yet fully scheduled/running
│  (could be unscheduled, or scheduled but images still pulling)
├─ Running: bound to a node, at least one container running (or
│  starting/restarting)
├─ Succeeded: ALL containers terminated successfully (exit 0),
│  won't restart (relevant for Jobs)
├─ Failed: ALL containers terminated, at least one non-zero exit,
│  won't restart (RestartPolicy: Never scenario)
└─ Unknown: node communication lost, kubelet can't report status
   (usually node NotReady/network partition)

CONTAINER STATES (per-container, MORE useful for debugging):
├─ Waiting: not running yet — reason field tells you WHY:
│  (ContainerCreating, ImagePullBackOff, CrashLoopBackOff,
│   CreateContainerConfigError, InvalidImageName, ...)
├─ Running: started successfully, startedAt timestamp recorded
└─ Terminated: stopped — reason (Completed/Error/OOMKilled),
   exitCode, and importantly `lastState` retains info about the
   PREVIOUS termination even after a restart (crucial for debugging
   CrashLoopBackOff — check lastState.terminated.exitCode/reason)

RESTART POLICY (pod-level, applies to ALL containers):
├─ Always (default, used by Deployments): always restart, backoff
│  increases exponentially (10s, 20s, 40s... capped at 5min) —
│  this backoff IS the "CrashLoopBackOff" state name
├─ OnFailure: restart only on non-zero exit (used by Jobs)
└─ Never: never restart (used by one-shot Jobs/Pods, debugging pods)
```

### Probes

```
LIVENESS PROBE: "is this container alive/healthy?"
├─ Failure → kubelet KILLS and RESTARTS the container (respecting
│  restartPolicy) — this is a DESTRUCTIVE action, use carefully
├─ Use for: detecting deadlocks/hangs where the process is running
│  but not functioning (e.g. stuck in an infinite loop, deadlocked
│  thread) — a symptom that only a restart can fix
└─ ⚠️ DON'T make liveness depend on external dependencies (database,
   downstream API) — if the DB is briefly down, you DON'T want to
   kill+restart every pod (won't fix the DB, just causes a self-
   inflicted outage/thundering herd on recovery)

READINESS PROBE: "should this container receive traffic right now?"
├─ Failure → pod removed from Service Endpoints (no traffic routed),
│  but container is NOT restarted — it stays running, just excluded
├─ Use for: temporary unavailability (still warming cache, or
│  downstream dependency briefly down, or graceful degradation
│  under high load) — exactly appropriate for the DB-down scenario
│  liveness should NOT use
└─ Also gates Deployment rollout progress (new pods must pass
   readiness before old ones scale down) and Service traffic routing

STARTUP PROBE (added 1.16, stable 1.20): "has this SLOW-STARTING
container finished its initial startup?"
├─ Liveness/readiness probes are DISABLED until startup probe
│  succeeds (prevents liveness from killing a container that's
│  just slow to start, e.g. a JVM app with a long warm-up/JIT phase)
├─ Once startup succeeds, normal liveness/readiness probing begins
└─ failureThreshold * periodSeconds gives you a generous total
   startup budget without weakening the (tighter) liveness timing
   once actually running

PROBE TYPES:
├─ httpGet: HTTP GET, success = 200-399 status code
├─ tcpSocket: success = TCP connection established
├─ exec: success = command exits with code 0
└─ grpc (stable 1.27+): native gRPC health checking protocol support
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: app
    image: myapp:latest
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30      # 30 * 10s = 5 min startup budget
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0    # startupProbe already covers warm-up
      periodSeconds: 10
      failureThreshold: 3       # 3 consecutive failures → restart
      timeoutSeconds: 2
    readinessProbe:
      httpGet:
        path: /ready            # checks DB connectivity etc — separate
        port: 8080               # endpoint from /healthz (liveness)
      periodSeconds: 5
      failureThreshold: 2
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 15"]  # drain time before SIGTERM
    terminationGracePeriodSeconds: 30
```

### Graceful Shutdown Sequence

```
POD DELETION TIMELINE:

t=0:   kubectl delete pod (or scale-down, or rolling update)
       → Pod object gets deletionTimestamp set (NOT immediately
         removed from etcd — this is a "soft delete" marker)
       → Pod is IMMEDIATELY removed from Service Endpoints (so
         no NEW traffic is routed here, even before the container
         actually stops — this ordering is important!)
       → kubelet notified, begins termination sequence

t=0:   preStop hook executed (if defined) — BLOCKS before SIGTERM
       is sent. Common pattern: `sleep 5-15s` to allow in-flight
       requests/load-balancer convergence to finish routing away
       from this pod BEFORE the process actually receives SIGTERM
       (closes the gap between "removed from Endpoints" and
        "iptables/LB actually stopped sending new connections here")

t=preStop_done: SIGTERM sent to PID 1 in the container
       → Application should catch this, stop accepting NEW
         connections, finish in-flight requests, then exit cleanly

t=preStop_done + terminationGracePeriodSeconds (default 30s):
       → If container hasn't exited yet, kubelet sends SIGKILL
         (immediate, un-catchable termination) — any in-flight
         work is abruptly lost at this point

COMMON BUG: app doesn't handle SIGTERM at all (default behavior in
many languages/frameworks is to ignore it or not gracefully drain)
→ full terminationGracePeriodSeconds is wasted waiting, then SIGKILL
  abruptly cuts active connections → visible as request errors during
  every rolling deployment. FIX: implement a SIGTERM handler that
  stops the readiness probe from passing immediately (extra safety
  net) and drains connections before exiting.
```

### Interview Questions — Pod Lifecycle

**Q1: Pod is CrashLoopBackOff. Debug systematically.**
> `kubectl describe pod` → check `Last State: Terminated` reason/exitCode (137=SIGKILL/OOM, 1=generic app error, 139=segfault). `kubectl logs <pod> --previous` to see the CRASHED container's logs (not the current restarted attempt's, which may have no output yet). Check for OOMKilled specifically in the reason field → resource limits issue. Check `kubectl get events --sort-by=.lastTimestamp` for broader context (node pressure, image pull issues masquerading as crashes). If logs are empty, check the container's actual ENTRYPOINT/CMD is correct (a common cause: entrypoint script exits 0 immediately because of a typo, looking like a "crash" with no error).

**Q2: Design health checks for a database-dependent application.**
> Liveness probe: simple, in-process check ONLY (e.g., "is my HTTP server thread alive and responding at all") — NEVER check the database here, to avoid killing/restarting the app when the DB (not the app) is the problem. Readiness probe: DOES check DB connectivity (e.g., a lightweight `SELECT 1` with a short timeout) — if DB is down, mark not-ready (removed from traffic) without restarting, allowing automatic recovery the moment DB comes back (readiness re-passes, traffic resumes) with zero pod restarts needed.

**Q3: Explain the difference between liveness and readiness probes.**
> Liveness failure = destructive (kill + restart the container) — for detecting unrecoverable internal hangs. Readiness failure = non-destructive (just remove from Service endpoints, keep running) — for detecting temporary inability to serve traffic. Using liveness where readiness is appropriate (checking external dependencies) causes unnecessary restarts and can create outage-amplifying restart storms when a shared dependency (like a database) has a brief blip.

**Q4: A rolling deployment causes brief 502s for users. How do you fix it with zero code changes to the app?**
> Add a `preStop` hook with a short sleep (e.g. 5-10s) to bridge the gap between "removed from Service endpoints" (near-instant) and "load balancer/kube-proxy/ingress controller actually stops sending new connections" (can lag by a few seconds due to distributed propagation delay, especially with cloud LBs or Ingress controllers with their own reload cycles). This buys time without touching application code. Complement with a readiness probe that's quick to fail so unhealthy pods are pulled fast, and ensure `terminationGracePeriodSeconds` is generous enough for in-flight requests (check your app's typical request duration p99).

**Q5: What does `lastState.terminated.reason: OOMKilled` combined with `exitCode: 137` tell you?**
> 137 = 128 + 9 (SIGKILL). Combined with reason OOMKilled, this confirms the Linux kernel's OOM killer terminated the container because it exceeded its memory limit (cgroup memory.limit_in_bytes) — NOT an application crash. Fix: profile actual memory usage (`kubectl top pod`, or in-app memory profiling for leaks), then either raise the memory limit appropriately or fix a leak; don't just blindly raise limits without understanding if it's a genuine leak vs. legitimate higher baseline usage.

---

## 5.2 Controllers — Deployment, StatefulSet, DaemonSet

### Deployment Strategies

```
ROLLINGUPDATE (default) — see 01-CONTROL-PLANE-INTERNALS.md for the
maxSurge/maxUnavailable math in detail.

RECREATE strategy: scale OLD ReplicaSet to 0 FIRST, THEN scale NEW
ReplicaSet up — causes a full outage window, but guarantees no two
versions run simultaneously (needed for apps that can't tolerate
mixed-version co-existence, e.g. certain DB schema migration patterns,
or singleton apps using exclusive resources).

spec:
  strategy:
    type: Recreate

BLUE-GREEN (not native — implemented via label/selector switching):
├─ Deploy "green" (new version) as a SEPARATE Deployment, 0% traffic
├─ Test green thoroughly (internal testing, synthetic traffic)
├─ Switch traffic: update Service's selector from
│  `version: blue` to `version: green` — INSTANT full cutover
│  (not gradual — this is the key blue-green characteristic)
└─ Keep blue running for instant rollback (just switch selector back)

CANARY (not native — via multiple ReplicaSets/Deployments +
weighted traffic, OR properly via Istio/Argo Rollouts):
├─ Naive K8s-only approach: 2 Deployments sharing the same Service
│  selector labels, differing replica COUNTS (e.g. 9 stable + 1
│  canary = ~10% traffic) — crude, traffic split is proportional to
│  pod count only, not truly controllable percentages
└─ Proper approach: Argo Rollouts or Istio VirtualService — precise
   traffic percentage control (weight-based), automated analysis
   (Prometheus query-based promotion/rollback), progressive delivery
   with defined steps (5% → 25% → 50% → 100%, pausing for validation
   at each step)
```

### StatefulSet Deep Patterns

```
ORDERED POD MANAGEMENT (podManagementPolicy: OrderedReady, default):
Creation: pod-0 created, MUST become Ready before pod-1 is created,
  which must become Ready before pod-2... (strictly sequential)
Deletion/Scale-down: REVERSE order — pod-2 deleted first, then
  pod-1, then pod-0 (important for apps like Kafka/Cassandra where
  the LOWEST ordinal is often treated as a bootstrap/seed node that
  should be the last to go)

podManagementPolicy: Parallel — all pods created/deleted
  SIMULTANEOUSLY, no ordering guarantee — appropriate for stateful
  apps that handle their own cluster membership/quorum logic
  (Cassandra, Elasticsearch) where you WANT faster scale-up and
  don't need K8s-enforced ordering.

UPDATE STRATEGIES (spec.updateStrategy):
├─ RollingUpdate (default): updates pods in REVERSE ordinal order
│  (N down to 0), one at a time, waiting for each to be Ready
│  before proceeding — same rationale as deletion ordering
├─ partition: N  → only pods with ordinal >= N get updated, pods
│  BELOW the partition are left on the old version — this enables
│  CONTROLLED CANARY updates for StatefulSets (e.g. partition: 9
│  on a 10-replica set updates ONLY pod-9, the other 9 stay old
│  until you lower the partition further)
└─ OnDelete: pods are NEVER automatically updated — you must
   manually delete each pod for it to be recreated with the new
   spec (maximum manual control, used for very sensitive stateful
   systems where automated rolling updates are too risky)

STABLE STORAGE: volumeClaimTemplates create ONE PVC per pod ordinal
(named <template>-<statefulset>-<ordinal>), and this PVC is NEVER
automatically deleted when the pod is deleted/rescheduled — it's
reattached to whatever pod gets that ordinal next. Even scaling down
to 0 and back up to N restores the SAME PVCs (data survives). Actually
DELETING the PVCs requires manual action (`kubectl delete pvc`) —
this is intentional (protects against accidental data loss from a
simple scale-down operation).
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None            # headless - required for StatefulSet
  selector:
    app: postgres
  ports:
  - port: 5432
```

### DaemonSet

```
Ensures EXACTLY ONE pod per node (or per matching node, via
nodeSelector) — used for node-level agents: CNI plugins, log
collectors (Fluent Bit), monitoring agents (node-exporter), CSI
node drivers.

├─ Automatically schedules onto NEW nodes as they join (no manual
│  intervention needed for cluster scale-out)
├─ Bypasses normal scheduler resource fit checks by default in some
│  configurations for critical system DaemonSets (tolerations for
│  ALL taints, including NoSchedule for uninitialized/not-ready nodes)
│  — necessary because CNI DaemonSet pods must run BEFORE a node is
│  network-ready, a chicken-and-egg problem solved via specific
│  toleration for `node.kubernetes.io/not-ready`
└─ RollingUpdate strategy updates DaemonSet pods node-by-node,
   respecting `maxUnavailable` (how many nodes can be without this
   DaemonSet's pod simultaneously during rollout)
```

### Interview Questions — Controllers

**Q1: Design zero-downtime deployment strategy.**
> RollingUpdate with `maxUnavailable: 0` (never drop below desired replica count — requires `maxSurge` > 0 to make progress at all) combined with correctly tuned readinessProbe (new pods must genuinely be ready before old ones are removed) and a preStop hook + adequate terminationGracePeriodSeconds for connection draining. Add a PodDisruptionBudget (`minAvailable`) to protect against voluntary disruptions (node drains during upgrades) compounding with the rollout simultaneously. For higher confidence, layer in canary analysis (Argo Rollouts) before full rollout.

**Q2: StatefulSet pod stuck in Terminating. Debug it.**
> Check for a `finalizer` still present (`kubectl get pod <name> -o yaml | grep finalizers`) blocking actual deletion until some controller removes it (common with storage-related finalizers if the CSI driver's unmount is hanging). Check if the container process itself is ignoring SIGTERM (stuck in D-state/uninterruptible sleep, e.g. blocked on slow disk I/O) — `kubectl delete pod --grace-period=0 --force` is a LAST RESORT (can cause data corruption for stateful apps mid-write, and leaves orphaned resources) — always prefer investigating why graceful termination is hanging first. Check the StatefulSet's ordering — if a HIGHER ordinal pod is also stuck, the lower one may be legitimately WAITING for it per OrderedReady deletion semantics.

**Q3: When would you use StatefulSet vs Deployment?**
> StatefulSet: needs stable network identity per replica (DNS name), stable dedicated storage per replica surviving rescheduling, and/or ordered startup/shutdown (leader election bootstrap sequences, quorum-based systems like etcd/Kafka/Cassandra/PostgreSQL primary-replica). Deployment: stateless, interchangeable replicas where any pod can serve any request identically, no need for stable identity or per-replica storage — the vast majority of application workloads (web servers, API services, workers).

**Q4: How does StatefulSet `partition` enable canary-style updates?**
> Setting `updateStrategy.rollingUpdate.partition: N` tells the controller to only update pods with ordinal >= N, leaving lower-numbered pods untouched on the old version. E.g., with 10 replicas and `partition: 9`, only `pod-9` gets the new image — you can validate that single canary pod's behavior, then progressively lower the partition (9→7→5→0) to roll out further once confident, giving StatefulSets a form of controlled progressive delivery despite lacking native canary support.

**Q5: A DaemonSet pod isn't scheduled on a specific node — why?**
> Check node taints the DaemonSet doesn't tolerate (DaemonSets need EXPLICIT tolerations for custom taints — they don't automatically tolerate everything except the built-in system ones like not-ready/unreachable which get auto-added). Check `nodeSelector`/`affinity` on the DaemonSet template excludes that node's labels. Check the node itself is Ready and not cordoned (`kubectl get nodes`). Check resource pressure — even DaemonSets respect resource requests/limits and won't schedule (or will be evicted) if the node genuinely lacks capacity, unless given a high enough PriorityClass to preempt other pods.
