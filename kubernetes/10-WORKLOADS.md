# Section 10: Workloads

Workload resources are the primary way users deploy applications on Kubernetes. Each workload type — Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, Job, CronJob — makes different guarantees about pod identity, ordering, persistence, and scheduling. Choosing the wrong workload type for an application is a common architectural mistake that causes operational pain at scale. Understanding the internal mechanics of each, including how the controllers manage them and when each is appropriate, is essential for both design and troubleshooting interviews.

## Subtopic Index

- [Pod](#pod)
- [ReplicaSet](#replicaset)
- [Deployment](#deployment)
- [StatefulSet](#statefulset)
- [DaemonSet](#daemonset)
- [Job](#job)
- [CronJob](#cronjob)
- [Design Decisions — When to Use Each Workload](#design-decisions--when-to-use-each-workload)
- [Failure Scenarios and Recovery](#failure-scenarios-and-recovery)

---

## Pod

A Pod is the smallest deployable unit in Kubernetes. It represents one or more tightly coupled containers that share a network namespace, IPC namespace, and optionally a PID namespace. All containers in a pod share the same IP address, the same network port space, and communicate on `localhost`. They have independent filesystems except for explicitly shared volumes.

Pods are ephemeral by design. The scheduler places pods on nodes, the kubelet runs them, but Kubernetes has no mechanism to "heal" a pod that exits — that is the responsibility of higher-level controllers (Deployment, ReplicaSet, StatefulSet). A pod created directly with `kubectl run` or `kubectl create pod` that crashes will not be restarted by any controller; it simply enters a terminal state.

Pod lifecycle phases: **Pending** (accepted, waiting for scheduling or image pull), **Running** (at least one container is running), **Succeeded** (all containers exited with 0), **Failed** (at least one container exited non-zero, restart policy = Never/OnFailure exhausted), **Unknown** (node communication lost).

Pod conditions provide more granular state: `PodScheduled`, `Initialized` (all init containers completed), `ContainersReady` (all containers passing readiness), `Ready` (PodScheduled + Initialized + ContainersReady + all readiness gates).

A pod's `restartPolicy` controls how individual container crashes are handled within the pod: `Always` (restart regardless of exit code — default for long-running services), `OnFailure` (restart on non-zero exit — for batch), `Never` (no restart — for one-shot tasks). The restart uses exponential backoff (10s, 20s, 40s, up to 5 minutes) to prevent crash-loop storms — the visible result is `CrashLoopBackOff`.

Multi-container pods share networking and potentially volumes. Common patterns: sidecar (helper runs alongside the main container — log shipper, proxy, secret injector), init container (runs to completion before main containers start — used for initialization, dependency checking), ephemeral container (added to a running pod for debugging without modifying the pod spec permanently).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container
spec:
  restartPolicy: Always
  initContainers:
  - name: init-db-wait
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
  containers:
  - name: app
    image: myapp:v2
    ports: [{containerPort: 8080}]
    resources:
      requests: {cpu: "100m", memory: "128Mi"}
      limits: {cpu: "500m", memory: "256Mi"}
  - name: log-shipper
    image: fluent/fluent-bit:latest
    volumeMounts:
    - name: log-vol
      mountPath: /var/log/app
  volumes:
  - name: log-vol
    emptyDir: {}
```

### Key commands
```bash
# Run a one-off pod
kubectl run debug --image=busybox --restart=Never --rm -it -- sh

# Get detailed pod status including all conditions
kubectl get pod <pod> -o yaml | grep -A50 status

# Check restart count and last exit code
kubectl get pod <pod> -o jsonpath='{range .status.containerStatuses[*]}{.name}: restarts={.restartCount} lastExitCode={.lastState.terminated.exitCode}{"\n"}{end}'

# Force delete a stuck pod (no graceful shutdown)
kubectl delete pod <pod> --force --grace-period=0

# Add an ephemeral debug container to a running pod
kubectl debug -it <pod> --image=busybox --target=app-container
```

---

## ReplicaSet

A ReplicaSet ensures a specified number of pod replicas are running at any time. It owns pods via `spec.selector` (label selector) and `ownerReferences`. The RS controller continuously reconciles: if `readyReplicas < spec.replicas`, it creates pods; if `readyReplicas > spec.replicas`, it deletes pods.

ReplicaSets are rarely used directly — Deployments manage ReplicaSets. Direct RS usage is appropriate when you need pod count control without rollout management (e.g., canary deployments managed by a separate rollout tool).

The RS controller selects which pods to delete when scaling down. It prefers to delete: pods on nodes with the most pods (spreading survivors), youngest pods (to preserve longer-running instances), non-running pods over running pods. This heuristic attempts to maintain workload distribution after scale-down.

**Selector immutability**: once a ReplicaSet's `spec.selector` is created, it cannot be changed. Changing the pod template without changing the selector label means the RS "adopts" old pods with old specs and won't create new ones with the new spec — because the selector still matches the old pods. This is why Deployments use a pod-template-hash in the selector — each RS has a unique selector that matches only its own pods.

### Key commands
```bash
# See ReplicaSets (normally managed by Deployments)
kubectl get rs -l app=my-app --sort-by='.metadata.creationTimestamp'

# Check RS owner (which Deployment manages it)
kubectl get rs <rs-name> -o jsonpath='{.metadata.ownerReferences}'

# Manually scale an RS (bypasses Deployment reconciliation)
kubectl scale rs <rs-name> --replicas=5

# Inspect pod template hash used for RS uniqueness
kubectl get rs <rs-name> -o jsonpath='{.metadata.labels.pod-template-hash}'
```

---

## Deployment

A Deployment manages rollouts. It owns ReplicaSets, and each distinct pod template maps to exactly one ReplicaSet. The Deployment controller's job is to drive the cluster toward the desired state (the current RS at full replicas, old RSes at 0).

When `spec.template` changes (new image, new env var, new config), the Deployment controller:
1. Computes a hash of the new pod template.
2. Looks for an existing RS with that hash (supporting rollback to a previous version without creating a new RS).
3. If none found, creates a new RS.
4. Scales up the new RS and scales down the old RS according to `strategy.rollingUpdate`.

**Rollout strategy comparison:**

`RollingUpdate` (default): progressively replaces old pods with new ones. `maxSurge` (default 25%) allows temporarily exceeding desired replicas to speed up the rollout. `maxUnavailable` (default 25%) allows temporarily falling below desired replicas. At `maxSurge=1, maxUnavailable=0`: the rollout adds one new pod, waits for it to be Ready, then removes one old pod — perfectly zero-downtime but 1 pod slower.

`Recreate`: terminates ALL old pods before starting any new ones. Causes a brief outage. Use when two versions cannot coexist (database schema migration that's backward-incompatible, exclusive resource access).

**Revision history**: Deployments keep old RSes (at 0 replicas) for rollback, up to `spec.revisionHistoryLimit` (default 10). Each old RS represents a revision. `kubectl rollout undo` promotes the previous RS to the current one, scaling it up and the current RS down.

**Rollout status**: a Deployment is considered complete when `status.updatedReplicas == spec.replicas && status.readyReplicas == spec.replicas && status.observedGeneration >= metadata.generation`. `status.conditions` includes `Progressing` (True while rolling, False if stuck) and `Available` (True if `availableReplicas >= minAvailable`).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments
spec:
  replicas: 10
  selector:
    matchLabels: {app: payments}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # up to 12 pods during rollout
      maxUnavailable: 0    # always maintain 10 ready pods
  minReadySeconds: 30      # pod must be Ready for 30s before rollout proceeds
  progressDeadlineSeconds: 300  # fail rollout if no progress in 5min
  template:
    metadata:
      labels: {app: payments}
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        image: payments:v3
        readinessProbe:
          httpGet: {path: /ready, port: 8080}
          initialDelaySeconds: 10
          failureThreshold: 3
        lifecycle:
          preStop:
            exec: {command: ["sleep", "5"]}
```

### Key commands
```bash
# Watch rollout progress
kubectl rollout status deployment/payments --timeout=10m

# Pause a rollout (freeze at current state for canary inspection)
kubectl rollout pause deployment/payments
kubectl rollout resume deployment/payments

# Rollback to previous version
kubectl rollout undo deployment/payments
kubectl rollout undo deployment/payments --to-revision=3

# See revision history with change cause
kubectl rollout history deployment/payments
kubectl rollout history deployment/payments --revision=3

# Set change cause for history (annotate before apply)
kubectl annotate deployment payments kubernetes.io/change-cause="deploy payments v3"

# Force a rollout (restart all pods even if template unchanged)
kubectl rollout restart deployment/payments
```

---

## StatefulSet

StatefulSets provide three guarantees that Deployments do not: stable network identity, stable persistent storage, and ordered pod management. They are designed for clustered applications where each instance has a unique role and identity: databases (PostgreSQL primary/replica, Cassandra nodes), message brokers (Kafka brokers, ZooKeeper nodes), and distributed caches (Redis Cluster).

**Stable network identity**: each pod gets a predictable hostname `<sts-name>-<ordinal>`. With a Headless Service (`clusterIP: None`), each pod gets a DNS A record: `<pod-name>.<headless-service>.<namespace>.svc.cluster.local`. This DNS name is stable — the same ordinal always resolves to the same pod identity. After pod-0 crashes and is replaced, the new pod has the same hostname and DNS name (though different pod IP, since IPs are not stable).

**Stable persistent storage**: `volumeClaimTemplates` creates a unique PVC per pod: `data-mydb-0`, `data-mydb-1`, `data-mydb-2`. These PVCs are NOT deleted when the pod is deleted, scaled down, or even when the StatefulSet is deleted (by default). The same PVC is reused when the ordinal is recreated.

**Ordered management**: pods are created in order (0, 1, 2…) and deleted in reverse order (2, 1, 0). Pod N is not created until pod N-1 is Running and Ready. Pod N is not deleted until pod N+1 is fully terminated. This guarantees that during a rollout, the cluster never has two primaries or violates a majority quorum.

`podManagementPolicy: Parallel` relaxes ordering. All pods start and stop simultaneously — useful for stateless workloads that happen to need stable identities (e.g., a service requiring stable hostnames but not sequential initialization).

Update strategies: `RollingUpdate` (with optional `partition` for staged rollouts) and `OnDelete` (manual per-pod control). `partition: N` means only pods with ordinal ≥ N get the new template — lower ordinals keep the old template. Use this for manual canary: update pod-2 first, verify, update pod-1, verify, update pod-0.

### Key commands
```bash
# Watch StatefulSet rollout (ordered)
kubectl rollout status statefulset/mydb

# Inspect ordered pods
kubectl get pods -l app=mydb --sort-by='.metadata.name'

# Set partition for staged update (canary)
kubectl patch statefulset mydb -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

# List PVCs created by StatefulSet
kubectl get pvc -l app=mydb

# Exec into a specific ordinal
kubectl exec mydb-0 -- psql -U postgres -c 'SELECT pg_is_in_recovery()'
kubectl exec mydb-1 -- psql -U postgres -c 'SELECT pg_is_in_recovery()'
```

---

## DaemonSet

A DaemonSet runs exactly one pod on every node (or on nodes matching a selector). It is used for node-level infrastructure agents that must run alongside every workload: log collectors (Fluent Bit), monitoring agents (Datadog, node-exporter), network plugins (Calico, Cilium, Flannel), storage plugins (CSI node drivers), security agents (Falco).

The DaemonSet controller bypasses the scheduler for pod placement. When a new node joins, the controller creates a pod with `spec.nodeName: <new-node>` already set — the scheduler never sees the pod. The DaemonSet controller also sets tolerations for all standard node taints (NotReady, Unreachable, DiskPressure, MemoryPressure, Unschedulable) so DaemonSet pods run even on nodes that are tainted for normal workloads.

Update strategies: `RollingUpdate` (one pod at a time, respecting `maxUnavailable`) and `OnDelete` (pods are updated only when manually deleted). `RollingUpdate` with `maxUnavailable: 1` updates nodes sequentially — safe for node-level agents. `OnDelete` gives full control but requires manual intervention.

A DaemonSet with a `nodeSelector` or `nodeAffinity` runs only on matching nodes. This is used for GPU monitoring agents (only GPU nodes), storage agents (only nodes with specific storage), or workloads that only need to run on a specific tier.

DaemonSet pods do not have guaranteed scheduling priority. They are subject to node resource constraints — if a node has no allocatable resources left (used entirely by workloads), the DaemonSet pod fails to start (OOM or CPU throttle). Always set DaemonSet resource requests appropriately and account for them in node capacity planning.

### Key commands
```bash
# Check DaemonSet rollout status
kubectl rollout status daemonset/fluentbit -n logging

# See which nodes have the DaemonSet pod
kubectl get pods -l app=fluentbit -o wide -n logging

# Find nodes WITHOUT the DaemonSet pod (indicates problem)
comm -23 \
  <(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | sort) \
  <(kubectl get pods -n logging -l app=fluentbit -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort)

# OnDelete update: manually trigger update on one node
kubectl delete pod -l app=fluentbit --field-selector=spec.nodeName=node-1 -n logging

# Check DaemonSet desired vs current
kubectl get ds fluentbit -n logging
```

---

## Job

A Job creates one or more pods to run a task to completion. Unlike Deployments, Jobs don't maintain steady state — they create pods, track their success, and terminate. Jobs are used for: database migrations, data processing batch jobs, machine learning training runs, report generation, and one-time administrative tasks.

`spec.completions`: how many successful pod runs are required. `spec.parallelism`: how many pods may run simultaneously. `spec.backoffLimit`: how many pod failures are allowed before the Job is marked Failed. `spec.activeDeadlineSeconds`: maximum time the Job may run (kills remaining pods after this).

**Completion modes**: `NonIndexed` (default): pods are interchangeable, any pod success counts toward completions. `Indexed`: each pod gets a unique index (0 to completions-1) via `JOB_COMPLETION_INDEX` env var and a stable hostname. Useful for sharded batch jobs where each worker processes a specific data partition.

**Failure handling**: `backoffLimit` counts total pod failures. The Job is marked Failed when `totalFailed >= backoffLimit`. After the first failure, the next pod starts after an exponential backoff (10s, 20s, 40s…). `backoffLimitPerIndex` (k8s 1.29+) tracks failures per index, preventing one bad index from consuming the entire budget.

**Automatic cleanup**: completed Jobs stay in the cluster indefinitely unless `spec.ttlSecondsAfterFinished` is set. Without TTL, completed Jobs (and their pods) accumulate, consuming etcd space and API list performance. Set `ttlSecondsAfterFinished: 3600` to auto-delete completed Jobs after 1 hour.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  activeDeadlineSeconds: 600    # fail if not complete in 10 minutes
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never       # required for Jobs (OnFailure is also valid)
      containers:
      - name: migrate
        image: myapp:v3-migrate
        command: ["/app/migrate", "--up"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef: {name: db-creds, key: url}
```

### Key commands
```bash
# Watch job completion
kubectl get job db-migration -w

# Check job status
kubectl describe job db-migration | grep -E 'Completions|Duration|Conditions|Events'

# View job pods (including failed ones)
kubectl get pods -l job-name=db-migration --sort-by='.metadata.creationTimestamp'

# Get logs from the most recent job pod
kubectl logs -l job-name=db-migration --tail=100

# Check failed pod exit codes
kubectl get pods -l job-name=db-migration -o jsonpath='{range .items[*]}{.metadata.name}: exitCode={.status.containerStatuses[0].state.terminated.exitCode}{"\n"}{end}'

# Create a job from a CronJob template (one-time manual run)
kubectl create job manual-run --from=cronjob/my-cron
```

---

## CronJob

A CronJob creates Jobs on a cron schedule. It is a wrapper around Job that adds schedule management. Use cases: nightly backups, daily reports, hourly cache invalidation, periodic cleanup jobs.

`spec.schedule`: standard cron format (minute, hour, day-of-month, month, day-of-week). Extended with timezone support via `spec.timeZone` (k8s 1.27+). `spec.concurrencyPolicy`: `Allow` (multiple runs can overlap), `Forbid` (skip if previous run is still running), `Replace` (cancel the running job and start a new one). `spec.startingDeadlineSeconds`: if the job misses its scheduled time by more than this many seconds, it's skipped.

**Missed run handling**: if the CronJob controller was down (or the cluster was unavailable) and missed scheduled runs, it computes all missed times since `lastScheduleTime`. If > 100 missed runs are computed, NO job is created (to prevent overwhelming the cluster after a long outage). `startingDeadlineSeconds` limits the lookback window for missed runs.

`spec.successfulJobsHistoryLimit` (default 3) and `spec.failedJobsHistoryLimit` (default 1) control how many completed/failed Job objects are retained for inspection. Old Jobs and their pods are automatically deleted.

### Key commands
```bash
# List CronJobs and their schedule / last run
kubectl get cronjob -o custom-columns=NAME:.metadata.name,SCHEDULE:.spec.schedule,LAST:.status.lastScheduleTime,ACTIVE:.status.active

# Check CronJob history
kubectl get jobs -l <selector-from-cronjob> --sort-by='.metadata.creationTimestamp'

# Suspend a CronJob (no more jobs created)
kubectl patch cronjob my-cron -p '{"spec":{"suspend":true}}'
kubectl patch cronjob my-cron -p '{"spec":{"suspend":false}}'  # resume

# Set timezone (k8s 1.27+)
kubectl patch cronjob my-cron -p '{"spec":{"timeZone":"America/New_York"}}'

# Manually trigger a CronJob run
kubectl create job --from=cronjob/my-cron manual-$(date +%s)
```

---

## Design Decisions — When to Use Each Workload

This is a FAANG-level design question. The correct workload type depends on the application's requirements for identity, storage, ordering, scheduling, and lifecycle.

| Workload | Use when | Don't use when |
|---|---|---|
| **Pod** | One-off debugging, testing | Running production services (no self-healing) |
| **Deployment** | Stateless services, can run multiple identical instances, need rolling updates | App needs stable identity or per-instance storage |
| **StatefulSet** | Databases, clustered apps, stable hostname/DNS needed, per-instance persistent storage | Simple stateless services (unnecessarily complex) |
| **DaemonSet** | Node-level agents (logging, monitoring, network, storage), must run on every node | General application deployment |
| **Job** | One-time tasks, batch processing, data migrations, completions with finite end state | Long-running services |
| **CronJob** | Periodic tasks, scheduled reports, recurring batch | Event-driven tasks (use EventBridge/SQS trigger instead) |

**Deployment vs StatefulSet decision**:
- Does each instance need a stable unique name? → StatefulSet
- Does each instance have its own persistent data? → StatefulSet
- Does initialization order matter? → StatefulSet
- Are all instances interchangeable? → Deployment

**StatefulSet vs Deployment for a simple database**: A single-instance PostgreSQL can run as a Deployment with a single PVC — replicas=1 ensures one instance, the PVC provides persistent storage. Use StatefulSet only when you need multiple instances with different roles (primary/replica) requiring stable identity for replication configuration. A single-instance database as a Deployment is simpler and sufficient.

---

## Failure Scenarios and Recovery

**Deployment rollout stuck**: new pods are not becoming Ready (failing readiness probe, CrashLoopBackOff, insufficient resources). The rollout stops at `maxUnavailable` pods replaced. Diagnosis: `kubectl describe pod <new-pod>`. Fix the root cause (probe, image, resource). The rollout automatically continues when pods become Ready. Manual rollback: `kubectl rollout undo deployment/<name>`.

**StatefulSet pod crash during ordered rollout**: StatefulSet requires pod N to be Ready before pod N-1 updates. If pod-2 crashes after updating and can't become Ready, pod-1 and pod-0 are never updated — the rollout is stuck. Diagnosis: `kubectl describe pod <sts>-2`. Options: fix the issue in the new image, or rollback the StatefulSet image. With `partition`, you can manually control which pods update.

**DaemonSet pod OOMKilled**: a node-level agent crashes due to insufficient memory limits. The DaemonSet controller restarts it (backoff), consuming node CPU/memory resources in the crash loop. Diagnosis: `kubectl describe pod <ds-pod>` on the affected node. Fix: increase DaemonSet memory limits. Short-term: `kubectl delete pod <ds-pod>` to trigger restart without backoff delay.

**Job getting stuck**: a Job's pod is Running but not making progress (infinite loop in application, deadlock). `spec.activeDeadlineSeconds` will eventually kill it, but if not set, it runs forever. Manual fix: `kubectl delete job <name> --cascade=foreground` (deletes the Job and all pods).

**CronJob creating too many jobs**: `concurrencyPolicy: Allow` with a long-running job and a frequent schedule causes many concurrent Jobs. Memory and pod count increases. Fix: change to `concurrencyPolicy: Forbid` or `Replace`. Clean up accumulated Jobs: `kubectl delete jobs -l <selector> --field-selector status.successful=1`.

### Key commands
```bash
# General workload health check
kubectl get deploy,sts,ds,job,cronjob -A | grep -v Running

# Find all pods in CrashLoopBackOff
kubectl get pods -A | grep CrashLoopBackOff

# Find all pods not running
kubectl get pods -A --field-selector='status.phase!=Running,status.phase!=Succeeded'

# Check workload-level events
kubectl describe deployment <name> | grep -A20 Events
kubectl describe statefulset <name> | grep -A20 Events

# Annotate a deployment with change cause for history
kubectl annotate deployment payments kubernetes.io/change-cause="v3: add retry logic"
```

---

## Interview Questions

### Conceptual / Internals (8 questions)

**1. A Deployment and a StatefulSet both run 3 replicas of a database. What are the practical differences in how they behave and when would you choose each?**

Deployment: all 3 pods are interchangeable, get random names (`db-7d4f9c-xxx`), random pod IPs that change on restart, and share one PVC (if any) or each get the same template-specified PVC. Rolling updates may run 2 or 3 versions simultaneously. Good for: single-instance database with one PVC (replicas=1), or truly stateless services. StatefulSet: each pod has a stable ordinal name (`db-0`, `db-1`, `db-2`), stable DNS (`db-0.db-headless.ns.svc.cluster.local`), and a unique PVC (`data-db-0`, `data-db-1`). Pods are created/deleted in order. Good for: multi-instance databases where each instance has unique state (PostgreSQL primary/replica, Cassandra, Kafka). Choose StatefulSet when: pod identity matters for replication configuration, each pod has its own data, or initialization order is required. Choose Deployment for stateless services and single-instance databases with one PVC.

**2. Explain how Deployment rolling updates work at the controller level, and what exactly maxSurge and maxUnavailable control.**

The Deployment controller creates a new ReplicaSet with the new pod template hash. It then runs an interleaved scale-up/scale-down loop. `maxSurge` is the maximum number of extra pods (above `spec.replicas`) that may exist during the rollout — these are the "in-progress" new pods. `maxUnavailable` is the maximum number of desired pods that may be unavailable (not Ready) during the rollout. The controller: (1) scales up new RS by min(maxSurge, needed) until new RS has replicas; (2) scales down old RS by min(maxUnavailable, excess) until old RS is at 0. Each step requires waiting for pods to become Ready (or Unavailable, depending on direction). At `maxSurge=25%, maxUnavailable=25%` on a 10-replica Deployment: up to 12 pods exist (10+25%) and at least 7 are Ready (10-25%) during the rollout — reasonably fast with acceptable risk.

**3. Why does StatefulSet require a Headless Service, and what DNS records does it create?**

A Headless Service (`clusterIP: None`) tells CoreDNS to return individual pod IPs directly in DNS A records, rather than a single ClusterIP. Without a Headless Service, DNS for the StatefulSet's pods would require knowing their random pod names (or IPs). With a Headless Service named `mydb-headless`, CoreDNS creates: `mydb-0.mydb-headless.namespace.svc.cluster.local → <pod-0-IP>`, `mydb-1.mydb-headless.namespace.svc.cluster.local → <pod-1-IP>`, etc. These DNS names are stable — `mydb-0.*` always resolves to the pod with ordinal 0, even if its IP changes. This is what allows Cassandra seeds, Kafka brokers, and ZooKeeper nodes to configure their peer addresses statically in config files. The Headless Service also creates a DNS record for the service itself that returns all pod IPs: `mydb-headless.namespace.svc.cluster.local → [pod-0-IP, pod-1-IP, pod-2-IP]`.

**4. How does a DaemonSet ensure its pods run even on nodes tainted with NoSchedule?**

When the DaemonSet controller creates pods, it adds tolerations to the pod spec automatically for well-known system taints: `node.kubernetes.io/not-ready:NoExecute`, `node.kubernetes.io/unreachable:NoExecute`, `node.kubernetes.io/disk-pressure:NoSchedule`, `node.kubernetes.io/memory-pressure:NoSchedule`, `node.kubernetes.io/pid-pressure:NoSchedule`, `node.kubernetes.io/unschedulable:NoSchedule`, `node.kubernetes.io/network-unavailable:NoSchedule`. These are added regardless of what the user specified. For custom taints (e.g., `dedicated=gpu:NoSchedule`), the DaemonSet pod must explicitly include a matching toleration in `spec.template.spec.tolerations`. This is why some infrastructure DaemonSets use `operator: Exists` with no key — they tolerate ALL taints — ensuring they run everywhere.

**5. What happens to StatefulSet PVCs when the StatefulSet is scaled down from 3 to 1?**

By default, nothing — the PVCs for ordinals 1 and 2 (`data-mydb-1` and `data-mydb-2`) are NOT deleted. The pods for ordinals 1 and 2 are deleted (in reverse order: 2 first, then 1), but their PVCs remain in `Bound` state, attached to their PVs. This is intentional: the data must survive scale-down in case the workload is later scaled back up. The same PVCs are reused when ordinals 1 and 2 are recreated. The downside: the PVCs continue consuming cloud storage (and money) even when the pods are gone. `persistentVolumeClaimRetentionPolicy` (k8s 1.23+) allows configuring auto-deletion: `whenScaled: Delete` deletes PVCs for pods removed by scale-down. Use this for ephemeral data; never for databases with unique state per ordinal.

**6. A Job has `backoffLimit: 6` but fails after only 2 pod failures. What could cause this?**

Several possibilities: (1) The Job has `activeDeadlineSeconds` set — if the total Job runtime exceeds this, the Job is marked Failed and all pods are terminated, regardless of `backoffLimit`. (2) A pod failure policy (`spec.podFailurePolicy`) has a rule that matches the failure condition (e.g., exit code 1) with `action: FailJob` — this marks the Job failed immediately regardless of backoffLimit. (3) `backoffLimit` counts pod failures across all parallel pods. In a parallel job with `parallelism: 3`, 3 simultaneous pods each failing once = 3 failures counted. (4) One pod failed twice before others ran — the backoff for that pod index is counted twice. Check `kubectl describe job <name>` for `Failed` in conditions and Events for the reason.

**7. Explain the difference between `restartPolicy: Never` and `restartPolicy: OnFailure` for a Job.**

`restartPolicy: Never`: when a container exits (any exit code), the pod terminates. If the exit code is non-zero, the Job controller creates a *new pod* for that completion (up to `backoffLimit`). Failed pods remain visible for debugging. `restartPolicy: OnFailure`: when a container exits non-zero, the kubelet restarts the container *within the same pod* (with exponential backoff). The pod remains Running with an increasing restart count. Only when `backoffLimit` is reached is a new pod NOT created (the Job fails). For debugging job failures: `Never` is better (each failed pod is preserved). For resource efficiency (each retry is a new pod with new scheduling overhead): `OnFailure` is better. `restartPolicy: Always` is NOT valid for Jobs — it conflicts with a Job's finite-completion model.

**8. How does the CronJob controller decide whether to create a Job for a missed schedule when the controller was down?**

On each reconcile cycle (every ~10 seconds), the CronJob controller computes all scheduled times since `lastScheduleTime` that should have triggered a Job. If `startingDeadlineSeconds` is set, it only considers times within `now - startingDeadlineSeconds`. If more than 100 missed schedule times are computed, the controller logs an error and does NOT create any jobs — to prevent overwhelming the cluster after a long outage. For fewer than 100 missed times, it creates one Job per missed time if `concurrencyPolicy: Allow`, or one Job for the most recent missed time if `Forbid` or `Replace`. `lastScheduleTime` is updated to the time of the last created Job, so the calculation window shrinks. The 100-job limit is a safety valve for when the controller was down for a very long time with a frequent schedule.

---

### Scenario / Troubleshooting (6 questions)

**9. A Deployment rollout is stuck at 50% — half old pods, half new pods. Neither set is being replaced. Diagnose.**

`kubectl rollout status deployment/<name>` — stuck. `kubectl describe deployment <name>` — check `Conditions`. `ProgressDeadlineExceeded` means no progress for `progressDeadlineSeconds` (default 600). Check new pods: `kubectl get pods -l <selector>` — find new pods by their pod-template-hash. `kubectl describe pod <new-pod>` — if readiness probe is failing, the rollout won't advance (can't scale down old pods because `maxUnavailable` would be exceeded). Fix the readiness probe issue or the application startup. If the readiness probe is healthy but pods are not Ready: check `ContainerReady: False` condition and Events for probe failures, missing ConfigMap/Secret, or resource constraints.

**10. You need to run a one-time database migration before deploying a new application version. What's the correct Kubernetes pattern?**

Use an `initContainer` in the Deployment pod spec for the migration, OR use a separate Job that runs the migration before the Deployment rollout. The initContainer approach: the migration runs in every pod before the app container starts — fine for idempotent migrations but problematic for migrations that should run exactly once (concurrent pods would all run the migration simultaneously). The Job approach (preferred): create a Job that runs the migration, wait for it to succeed, then update the Deployment. Implement this as an ArgoCD `PreSync` hook or a Helm pre-upgrade hook. The hook runs the migration Job, waits for completion, then proceeds with the Deployment update. For non-idempotent migrations, the Job approach is safer. Add `ttlSecondsAfterFinished: 86400` to auto-clean the Job.

**11. A StatefulSet's rolling update is stuck — pod-2 was updated but pod-1 and pod-0 were not. Why and how do you fix it?**

StatefulSet rolling update requires each pod to be Running and Ready before updating the next (lower) ordinal. If pod-2 is updated but not Ready (failing readiness probe, CrashLoopBackOff), the controller stops and won't update pod-1. Diagnose: `kubectl describe pod <sts>-2` — check Events and container state. Fix the application or probe configuration. If the image is bad and you want to rollback: `kubectl patch statefulset <name> --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"<old-image>"}]'`. Setting `updateStrategy.rollingUpdate.partition` to 3 (above all ordinals) immediately stops further updates and can be used to "pause" the rollout.

**12. A CronJob is scheduled to run every minute but only runs every 5 minutes. Why?**

If the previous Job is still running when the next scheduled time arrives, `concurrencyPolicy: Forbid` (if set) causes the new run to be skipped. With every-minute schedule and a 4-minute job, 4 runs per 5 minutes are skipped. Fix: change to `Replace` (cancels running job and starts new) or `Allow` (runs concurrently). Another cause: `startingDeadlineSeconds` is too small — if the cron controller misses a window by even a few seconds (due to clock skew or controller delay), it marks the run as missed and skips. Set `startingDeadlineSeconds` to at least 60 seconds for a minute-frequency schedule. Verify: `kubectl describe cronjob <name>` — check `Last Schedule Time` and `Active` count.

**13. A Job is consuming resources but the expected task completed 3 hours ago. What happened?**

The Job's pod is still Running even though the task is done. Causes: (1) The application doesn't exit after completing its task — it runs forever (a long-polling loop, a web server started by mistake, or waiting for a signal). The Job controller only marks the Job complete when pods exit with code 0. (2) The Job has `activeDeadlineSeconds` not set, so it waits indefinitely. Fix: inspect the pod's process: `kubectl exec <job-pod> -- ps aux`. Kill the hanging process or fix the application to exit after task completion. Add `activeDeadlineSeconds` to prevent infinite hangs. Add `ttlSecondsAfterFinished` to auto-cleanup.

**14. After running `kubectl delete namespace staging`, staging workloads are gone but some pods are still Terminating after 30 minutes. Why won't they delete?**

Namespace deletion is cascading: it deletes all objects in the namespace in dependency order. Pods typically delete within `terminationGracePeriodSeconds` (30-60s). Stuck Terminating pods have a finalizer preventing deletion. Common culprits: (1) A service mesh (Istio) adds a finalizer to namespaces (`istio-injection`) waiting for its cleanup webhook. Check `kubectl describe namespace staging | grep Finalizers`. Remove stuck finalizers: `kubectl patch namespace staging --type=json -p='[{"op":"remove","path":"/metadata/finalizers"}]'`. (2) A pod has a long `terminationGracePeriodSeconds`. Force delete: `kubectl delete pod --all -n staging --force --grace-period=0`. (3) A custom controller has a finalizer on pods that is waiting for cleanup (e.g., CSI driver waiting to detach a volume). Check the pod's finalizers: `kubectl get pod <pod> -o jsonpath='{.metadata.finalizers}'`.

---

### FAANG-Level Deep Dive (6 questions)

**15. How does the Deployment controller use pod-template-hash to safely manage multiple concurrent ReplicaSets during a rollout?**

When a Deployment's pod template changes, the controller computes a hash of the pod template spec (using `hash.DeepHashObject` from the apimachinery library). This hash becomes a label `pod-template-hash: <hash>` added to both the ReplicaSet's labels and its pod template. The Deployment's selector includes `pod-template-hash` via a label requirement, making each ReplicaSet's selector unique — RS for v1 selects `pod-template-hash=abc123`, RS for v2 selects `pod-template-hash=xyz789`. This prevents RSes from adopting each other's pods. The Deployment controller uses `spec.revisionHistoryLimit` to keep a bounded number of old RSes. Each RS stores its revision number in `metadata.annotations["deployment.kubernetes.io/revision"]`. Rollback simply finds the RS with the target revision and scales it up (scaling the current RS down). If the new template matches an existing RS (e.g., rolling back to v1 which still exists), the existing RS is reused — no new RS is created.

**16. Explain the StatefulSet controller's guarantee that two pods with the same ordinal never run simultaneously, and how this is implemented.**

The StatefulSet controller maintains a strict invariant: before creating or updating pod N, it verifies that pod N is not currently Running or Pending. Before deleting pod N, it verifies pod N-1 does not exist (or has already terminated). The controller reads the current pod set via its pod informer, groups them by ordinal, and for each ordinal determines the action (create/update/delete). If an existing pod has `DeletionTimestamp` set (terminating), the controller waits — it doesn't create the replacement until the terminating pod is fully gone (no longer in the API). This is implemented by the `isMemberOf` check in the StatefulSet controller code. The guarantee relies on Kubernetes etcd-level linearizability: the controller reads current pod state from the cache (fed by etcd) and writes new pods only when ordinal N's slot is verified empty.

**17. How does the DaemonSet controller handle a node that joins the cluster while the DaemonSet is being updated (mid-rollout)?**

During a DaemonSet rolling update, some nodes have the old pod and some have the new pod. When a new node joins: the DaemonSet controller's node informer receives an ADDED event for the new node. The controller reconciles and creates a pod on the new node. For `updateStrategy: RollingUpdate`, it creates the pod with the **current** (new) template — the new node always gets the latest version. The `maxUnavailable` budget applies only to existing pods being updated, not to new pods on new nodes. So a new node joining during a rolling update always runs the new version immediately, which could mean the cluster has two different versions running simultaneously on different nodes (new node: new version, old nodes still being updated: old version). This is expected and intentional — DaemonSet updates are designed to be gradual across existing nodes.

**18. How does Kubernetes implement pod priority and preemption at the scheduler level, including the exact sequence of events?**

When a high-priority pod can't be scheduled (all nodes fail filter), the PostFilter plugin `DefaultPreemption` runs: (1) It identifies "victim" pods (lower priority than the pending pod) that could be evicted from nodes to make room. For each candidate node, it simulates removing lower-priority pods and checks if the pending pod would then fit. (2) It selects the node that minimizes disruption (fewest victims, lowest-priority victims, respects PodDisruptionBudgets — won't evict if it would violate a PDB). (3) It issues `DELETE` requests on the victim pods (graceful deletion, respecting terminationGracePeriodSeconds). (4) The pending pod is NOT immediately placed on the node. It is re-queued to `activeQ`. (5) As victim pods terminate and free resources, the next scheduling cycle for the pending pod will find the node feasible. (6) A "nominated node" annotation is added to the pending pod, telling the scheduler to consider that node first on retry — but not exclusively. Another pod may preempt the "reserved" spot before the high-priority pod gets there.

**19. What is the difference between a Job with `restartPolicy: OnFailure` and `backoffLimit: 3` vs `restartPolicy: Never` and `backoffLimit: 3`, and which is more efficient for a large batch workload?**

`OnFailure`: failed containers are restarted *in-place* by the kubelet within the same pod. The pod remains scheduled on the same node. Restart uses exponential backoff (10s → 20s → 40s). The pod maintains state between restarts (same node's filesystem, same ephemeral volumes). `Never`: on container failure, the pod terminates. The Job controller creates a *new pod*, which must be re-scheduled (new node, new image pull check, new CNI assignment, new ephemeral volumes). For large batch workloads: `OnFailure` is more efficient because it avoids scheduling overhead per retry. However, if the failure is node-specific (disk issue, GPU driver), `Never` allows the new pod to be scheduled to a different (healthy) node — `OnFailure` retries on the same potentially-broken node. Best practice: `OnFailure` for transient failures (network timeouts, database connection refused), `Never` for non-transient failures where you want the pod to move nodes.

**20. Design a zero-downtime canary deployment system using only native Kubernetes primitives (no Argo Rollouts or Flagger).**

Using two Deployments (stable and canary) with one shared Service: (1) `stable` Deployment: 10 replicas, label `app: payments, version: stable`. (2) `canary` Deployment: 1 replica, label `app: payments, version: canary`. (3) Service selector: `app: payments` — selects BOTH stable and canary pods. Traffic is split proportionally by pod count: 10:1 = ~9% canary. (4) Scale the canary up and stable down to shift traffic. At 5:5, traffic is 50/50. Advantages: pure Kubernetes, no extra tools. Disadvantages: traffic split is coarse (controlled by pod ratio, not exact percentage), you can't do header-based routing. Better alternative: use `TopologyAwareHints: Disabled` to ensure load is distributed randomly. For exact percentage splits (10%), use two Services + a weighted Ingress (NGINX ingress annotation or Gateway API `weight` field), not pod count. The Gateway API approach with HTTPRoute weights is the most correct native solution.

---

## Hands-On Labs

### Lab 1: Deployment Rollout and Rollback

**Objective:** Experience Deployment rollout, pause, and rollback.

**Tasks:**
1. Deploy `nginx:1.24` with 5 replicas, `maxSurge=1, maxUnavailable=0`.
2. Watch the rollout: `kubectl rollout status deployment/nginx -w` alongside `kubectl get pods -w`.
3. Update to `nginx:1.25`. Watch the rolling update — observe one new pod at a time.
4. Update to `nginx:1.99` (broken image). Observe the rollout stall.
5. Rollback: `kubectl rollout undo deployment/nginx`.
6. Check history: `kubectl rollout history deployment/nginx`.

### Lab 2: StatefulSet Identity and Ordered Operations

**Objective:** Observe StatefulSet ordering and stable identity.

**Tasks:**
1. Deploy a 3-replica StatefulSet with a Headless Service.
2. Watch pod creation: `kubectl get pods -w` — observe 0 → 1 → 2 sequential start.
3. Kill pod-1: `kubectl delete pod <sts>-1`. Observe it does NOT affect pod-2.
4. Exec into pod-0: `kubectl exec <sts>-0 -- hostname`. Confirm `<sts>-0`.
5. Resolve DNS: `kubectl exec <sts>-0 -- nslookup <sts>-1.<headless-svc>`.
6. Delete pod-0 and confirm it gets the same DNS name when recreated.

### Lab 3: Job Failure Modes

**Objective:** Understand Job failure policies and backoff.

**Tasks:**
1. Create a Job with a script that fails with exit code 1. Set `backoffLimit: 3`.
2. Watch pods created and failed: `kubectl get pods -l job-name=<job> -w`.
3. Observe exponential backoff between retries.
4. After Job fails, check `kubectl describe job <name>` for failure reason and pod count.
5. Create a Job with `restartPolicy: OnFailure`. Observe restarts within the same pod vs new pods.
6. Create a Job with `activeDeadlineSeconds: 30` and a sleep-forever command. Observe forced termination.

---

## Production Incidents

### Incident 1: StatefulSet Update Creates Split-Brain in Kafka Cluster

**Symptom:** After updating Kafka StatefulSet image version, 2 of 3 brokers are running the new version. Producers begin failing with "Incompatible magic byte error." Consumer lag spikes. The third broker (kafka-0, the controller) is running the old version.

**Investigation:** StatefulSet rolling update starts from the highest ordinal. kafka-2 and kafka-1 updated successfully. kafka-0 (the Kafka cluster controller, ordinal 0) is next. After kafka-1 updated, the readiness probe for kafka-1 failed because the new Kafka version changed the protocol slightly and kafka-0 (old version) rejected the new version's heartbeat. kafka-1 is not Ready, so the StatefulSet controller stops and won't update kafka-0. Result: split-brain — two brokers on v3.5, one broker on v3.4, incompatible protocol.

**Root cause:** Kafka's rolling update requires controller-first (kafka-0) upgrade because the controller determines protocol compatibility. Kubernetes StatefulSet updates highest-ordinal-first by default. Setting `partition: 1` first would update kafka-1 and kafka-2, but the correct sequence for Kafka is to update the controller last, which StatefulSet does naturally — except Kafka requires the controller to be updated *first*.

**Recovery:** Manually set `partition: 3` to stop automatic updates. Manually delete kafka-0 pod to force update of the controller first. Then reduce partition to 0 for remaining pods.

**Prevention:** Use `OnDelete` strategy for stateful systems with complex upgrade ordering requirements. Document the correct upgrade sequence. Test upgrade procedures in staging before production.

### Incident 2: CronJob Accumulation Causes etcd Pressure

**Symptom:** Over 6 months, etcd size grows from 500MB to 7.5GB. Cluster begins showing API latency spikes. `etcd_db_total_size_in_bytes` metric shows steady growth. No major cluster expansion occurred.

**Investigation:** `etcdctl get /registry --prefix --keys-only | grep jobs | wc -l` returns 47,000 Job objects. A data processing CronJob runs every 5 minutes, 24/7 = 288 jobs/day × 180 days = 51,840 jobs. Neither `ttlSecondsAfterFinished` nor `successfulJobsHistoryLimit` was set. Each completed Job retains its pods (logs). 51,840 Jobs × (Job object + pod object + pod log reference) = significant etcd volume.

**Root cause:** CronJob without history limits or TTL accumulates completed Jobs and pods indefinitely.

**Recovery:** Set `ttlSecondsAfterFinished: 3600` on the CronJob. Bulk-delete old completed Jobs: `kubectl delete jobs -l <selector> --field-selector status.successful=1`. etcd compaction and defragmentation to reclaim space.

**Prevention:** Enforce via ValidatingAdmissionPolicy: all CronJobs must have `ttlSecondsAfterFinished <= 86400` and `successfulJobsHistoryLimit <= 5`. Monitor `etcd_mvcc_db_total_size_in_bytes` and alert at 4GB (50% of 8GB quota).
