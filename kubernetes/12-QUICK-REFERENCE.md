# Quick Reference — Commands, Cheatsheets, Diagrams

> Night-before-interview rapid recall sheet

---

## Essential kubectl Commands

```bash
# Debugging pods
kubectl get pods -A -o wide
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> [-c container] [--previous] [-f]
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl get events -n <ns> --sort-by=.lastTimestamp
kubectl top pod <pod> -n <ns>
kubectl top node

# Debugging scheduling
kubectl describe node <node> | grep -A5 "Allocated resources"
kubectl get pods -n <ns> -o wide --field-selector spec.nodeName=<node>

# RBAC
kubectl auth can-i <verb> <resource> --as <user> -n <ns>
kubectl auth can-i --list --as <user> -n <ns>
kubectl get rolebindings,clusterrolebindings -A -o wide

# Networking
kubectl get endpoints <svc> -n <ns>
kubectl get networkpolicy -n <ns> -o yaml
kubectl get svc -n <ns> -o wide

# Rollouts
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout history deployment/<name> -n <ns>
kubectl rollout undo deployment/<name> -n <ns> [--to-revision=N]
kubectl rollout restart deployment/<name> -n <ns>

# etcd
etcdctl snapshot save /path/to/snapshot.db
etcdctl snapshot status /path/to/snapshot.db -w table
etcdctl endpoint status --cluster -w table
etcdctl endpoint health --cluster
etcdctl member list

# Certificates (kubeadm)
kubeadm certs check-expiration
kubeadm certs renew all

# Draining nodes safely
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>
```

---

## Exit Code Quick Reference

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Generic application error |
| 137 | SIGKILL (128+9) — often OOMKilled |
| 139 | SIGSEGV (128+11) — segfault |
| 143 | SIGTERM (128+15) — graceful shutdown signal received |

---

## Pod Status Quick Diagnosis

| Status | First Command | Likely Cause |
|---|---|---|
| Pending | `kubectl describe pod` → Events | Resources, affinity, taints, PVC |
| CrashLoopBackOff | `kubectl logs --previous` | App error, OOM, bad config |
| ImagePullBackOff | `kubectl describe pod` | Registry auth, wrong tag |
| Evicted | `kubectl get pod -o yaml \| grep reason` | Node pressure |
| Terminating (stuck) | `kubectl get pod -o yaml \| grep finalizers` | Finalizer, SIGTERM ignored |
| ContainerCreating (stuck) | `kubectl describe pod` | Volume mount, CNI issue |

---

## RollingUpdate Math

```
maxSurge = ceil(replicas * maxSurge%)
maxUnavailable = floor(replicas * maxUnavailable%)

Example: 10 replicas, maxSurge=25%, maxUnavailable=25%
maxSurge = ceil(10*0.25) = 3   → up to 13 pods total during rollout
maxUnavailable = floor(10*0.25) = 2  → at least 8 must stay Ready
```

---

## QoS Class Determination

```
Guaranteed: EVERY container has requests == limits for BOTH cpu+memory
BestEffort: NO requests/limits set on ANY container
Burstable:  everything else (the common case)

Eviction order under pressure: BestEffort → Burstable (most over-request first) → Guaranteed (last)
```

---

## RBAC Cheat Sheet

```
Role          → namespaced permissions
ClusterRole   → cluster-wide permissions (or reusable across namespaces via RoleBinding)
RoleBinding        → grants Role/ClusterRole, scoped to ONE namespace
ClusterRoleBinding → grants ClusterRole, cluster-wide

Rule matching: additive only (union of all applicable bindings), no explicit DENY
```

---

## NetworkPolicy Cheat Sheet

```
No policy selecting a pod = all traffic allowed (flat network)
ANY policy selecting a pod (for a direction) = default deny for THAT direction,
  except what's explicitly allowed (union of ALL matching policies)

ALWAYS remember: allow DNS egress (UDP/TCP 53 to kube-system) or everything breaks
```

---

## Service Types Cheat Sheet

| Type | Use Case | Reachable From |
|---|---|---|
| ClusterIP | Internal only | Inside cluster |
| NodePort | Dev/test external access | `<node-ip>:<30000-32767>` |
| LoadBalancer | Production external access | Internet (via cloud LB) |
| ExternalName | DNS alias to external service | CNAME only, no proxying |
| Headless (`clusterIP: None`) | Direct pod addressing (StatefulSet) | DNS returns all pod IPs |

---

## CNI Plugin Decision Matrix

| Requirement | Recommended CNI |
|---|---|
| On-prem, need BGP integration | Calico |
| Cloud-native, max performance/observability | Cilium |
| Simple dev/test cluster | Flannel |
| Need pods with real VPC IPs | AWS VPC CNI / Azure CNI |
| Need L7-aware NetworkPolicy | Cilium |

---

## Probe Types Cheat Sheet

```
startupProbe:  gates liveness/readiness until slow-start finishes
livenessProbe: failure → RESTART container (use for internal deadlock only)
readinessProbe: failure → REMOVE from Service endpoints (use for dependency checks)

NEVER put external dependency checks (DB, downstream API) in livenessProbe
```

---

## Key Metrics to Know

| Metric | What It Tells You |
|---|---|
| `etcd_disk_wal_fsync_duration_seconds` | etcd disk write latency (P99 <10ms healthy) |
| `etcd_server_leader_changes_seen_total` | etcd instability (should be ~0) |
| `apiserver_flowcontrol_rejected_requests_total` | API server overload (APF) |
| `container_cpu_cfs_throttled_periods_total` | CPU throttling despite node headroom |
| `container_memory_working_set_bytes` | Actual memory usage (vs OOM investigation) |
| `kube_pod_status_phase` | Pod phase distribution |

---

## Raft/etcd Quorum Table

| Cluster Size | Quorum | Fault Tolerance |
|---|---|---|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

(Always use odd numbers — even numbers waste resources for no added fault tolerance)

---

## Scheduler Phases

```
FILTER (eliminate infeasible nodes) → SCORE (rank feasible nodes 0-100) → BIND

Filter plugins: PodFitsResources, NodeAffinity, TaintToleration, VolumeBinding, PodAffinity
Score plugins: NodeResourcesFit, ImageLocality, InterPodAffinity, PodTopologySpread
```

---

## Admission Control Order

```
Mutating webhooks (can modify) → Schema validation → Validating webhooks (accept/reject) → etcd write
```

---

## StatefulSet vs Deployment Decision

```
Use StatefulSet when you need:
├─ Stable network identity (pod-0, pod-1, ...)
├─ Stable per-replica storage (survives rescheduling)
└─ Ordered startup/shutdown

Use Deployment for everything else (stateless, interchangeable replicas)
```

---

## The 8-Second Elevator Pitch Answers

**"What happens on kubectl apply?"**
> AuthN → AuthZ (RBAC) → Admission (mutate then validate) → schema validation → etcd write (optimistic concurrency via resourceVersion) → watch notification → controller reconciles → scheduler binds → kubelet starts container.

**"Pod is Pending, what do you check?"**
> `describe pod` for FailedScheduling event reason: resources, affinity, taints, or PVC binding.

**"CrashLoopBackOff?"**
> `logs --previous` + `describe` for exit code (137=OOM, 1=app error) — fix root cause, not just restart policy.

**"How does etcd stay consistent?"**
> Raft consensus — leader replicates log entries, commits on quorum ACK (majority), followers apply after commit index advances.

**"Liveness vs readiness?"**
> Liveness = restart on failure (internal deadlock only). Readiness = remove from Service on failure (external dependency checks go here).

**"How do Services route traffic?"**
> kube-proxy programs iptables/IPVS/eBPF rules that DNAT ClusterIP:port to a healthy backend pod IP:port, tracked by conntrack for connection consistency.
