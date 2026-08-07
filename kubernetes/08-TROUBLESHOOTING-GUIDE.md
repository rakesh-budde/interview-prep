# Troubleshooting Guide (15% of Interview Weight)

> 20 production incident scenarios: symptoms, debugging commands, root cause, fix

---

## 1. Pod Stuck in Pending

**Symptoms:** `kubectl get pods` shows `Pending`, no node assigned, pod age growing.

```bash
kubectl describe pod <pod> -n <ns>          # check Events section
kubectl get nodes -o wide                    # node status/capacity
kubectl describe nodes | grep -A5 "Allocated resources"
kubectl get pvc -n <ns>                      # if pod uses storage
```

**Root causes & fixes:**
| Cause | Signal | Fix |
|---|---|---|
| Insufficient CPU/memory on all nodes | Event: "Insufficient cpu" | Add nodes, reduce requests, enable Cluster Autoscaler |
| Node affinity/selector unsatisfiable | Event: "node(s) didn't match node selector" | Fix label selector or label nodes correctly |
| Taints not tolerated | Event: "node(s) had taints that pod didn't tolerate" | Add toleration or remove taint |
| PVC not bound | PVC status Pending | Fix StorageClass, check `WaitForFirstConsumer` (normal) |
| PriorityClass/preemption not enough | Multiple lower-priority pods still present | Verify preemption policy, check PDBs blocking eviction |

---

## 2. Pod Stuck in Terminating

**Symptoms:** Pod shows `Terminating` for many minutes, doesn't disappear.

```bash
kubectl get pod <pod> -n <ns> -o yaml | grep -A5 finalizers
kubectl get pod <pod> -n <ns> -o jsonpath='{.metadata.deletionTimestamp}'
kubectl describe pod <pod> -n <ns>
```

**Root causes & fixes:**
- **Finalizer stuck** (e.g., a CSI volume finalizer waiting for unmount that's hanging): identify which controller owns the finalizer, check its logs; only manually patch out the finalizer as a last resort (`kubectl patch pod <pod> -p '{"metadata":{"finalizers":null}}'`) — risk of orphaned resources.
- **Process ignoring SIGTERM / stuck in D-state (uninterruptible I/O wait)**: check `kubectl exec` (won't work if terminating) or node-level `ps aux` for the container's PID state; slow disk I/O is common culprit.
- **kubelet on the node is down/unresponsive**: `kubectl get nodes` shows NotReady; pod object can't be finalized until kubelet reports back. Fix node health first.
- **Force delete (last resort, data-loss risk for stateful workloads):** `kubectl delete pod <pod> --grace-period=0 --force`

---

## 3. CrashLoopBackOff

**Symptoms:** Restart count climbing, exponential backoff delay between restarts.

```bash
kubectl logs <pod> -n <ns> --previous     # logs from the CRASHED attempt
kubectl describe pod <pod> -n <ns>        # Last State: Terminated reason/exitCode
```

**Diagnosis by exit code:**
| Exit Code | Meaning | Investigation |
|---|---|---|
| 0 | Clean exit, but restartPolicy=Always keeps restarting | Check ENTRYPOINT/CMD isn't a one-shot script |
| 1 | Generic application error | Check `--previous` logs for stack trace |
| 137 | SIGKILL (often OOMKilled) | Check `reason: OOMKilled`, raise memory limit or fix leak |
| 139 | Segfault | Application/native library bug, check for arch mismatch (arm64 vs amd64) |
| 143 | SIGTERM (graceful, but still restarting) | Check app handles shutdown properly, may be liveness probe killing it |

**Root causes:** missing environment variables/config, bad ConfigMap/Secret mount, application dependency (DB) unreachable at startup with no retry logic, wrong image tag deployed, liveness probe misconfigured (killing an otherwise healthy but slow-starting app).

---

## 4. ImagePullBackOff

**Symptoms:** Container stuck in `Waiting`, reason `ImagePullBackOff` or `ErrImagePull`.

```bash
kubectl describe pod <pod> -n <ns>          # exact pull error message
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

**Root causes & fixes:**
- **Wrong image name/tag** (typo, or tag doesn't exist) → verify with `docker pull`/`crane` manually.
- **Private registry auth missing**: check `imagePullSecrets` on the pod/ServiceAccount, verify secret isn't expired (ACR/ECR tokens can expire).
- **Network/DNS issue reaching registry**: check NetworkPolicy, node egress to registry endpoint, private endpoint/firewall rules.
- **Rate limiting** (Docker Hub anonymous pull limits): use an authenticated pull or a pull-through cache/mirror.
- **Image architecture mismatch**: multi-arch cluster (arm64 nodes) pulling an amd64-only image.

---

## 5. OOMKilled

**Symptoms:** `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`.

```bash
kubectl describe pod <pod> -n <ns>
kubectl top pod <pod> -n <ns>
# Prometheus: container_memory_working_set_bytes over time
```

**Root cause diagnosis:** graph memory usage over time — steadily climbing (never plateaus) = leak, requires profiling (pprof/heap dump); plateaus at a stable-but-high level = legitimately needs more memory, just raise the limit. Check for a recent deploy correlating with onset (new dependency, config change, traffic increase).

---

## 6. Evicted Pods

**Symptoms:** Pod shows `status.phase: Failed`, `reason: Evicted`.

```bash
kubectl get pod <pod> -n <ns> -o jsonpath='{.status.reason}{"\n"}{.status.message}'
kubectl describe node <node>                 # check Conditions: MemoryPressure/DiskPressure
```

**Root causes & fixes:** node-level resource pressure (kubelet's node-pressure eviction manager reclaiming resources) — check `MemoryPressure`, `DiskPressure`, `PIDPressure` node conditions. BestEffort/Burstable pods exceeding requests evicted first. Fix: right-size requests to reflect real usage, add more node capacity, clean up disk (unused images via `crictl rmi`, excessive log volume), set proper eviction thresholds (`--eviction-hard`).

---

## 7. Service Not Reachable

**Symptoms:** `curl http://my-svc` times out or connection refused from within cluster.

```bash
kubectl get endpoints <svc> -n <ns>          # any endpoints listed? all pods ready?
kubectl get svc <svc> -n <ns> -o yaml        # selector matches pod labels?
kubectl exec -it <client-pod> -- curl -v <pod-ip>:<port>   # bypass Service, test pod directly
kubectl get networkpolicy -n <ns>
```

**Root causes & fixes:** Service selector doesn't match any pod labels (typo/case mismatch — most common cause), all backing pods failing readinessProbe (zero healthy endpoints), NetworkPolicy blocking traffic (check default-deny + missing allow rule), port mismatch (`targetPort` vs container's actual listening port), kube-proxy not running/healthy on the node (`kubectl get pods -n kube-system -l k8s-app=kube-proxy`).

---

## 8. DNS Resolution Failures

**Symptoms:** `nslookup`/`curl` to service names fails; external domains fail too.

```bash
kubectl exec -it <pod> -- nslookup my-service
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl get endpoints kube-dns -n kube-system
```

**Root causes & fixes:** CoreDNS pods unhealthy/overloaded (check CPU/memory, scale replicas), `dnsPolicy` misconfigured (should be `ClusterFirst`, not accidentally `Default` which uses node's resolv.conf), NetworkPolicy blocking UDP/TCP 53 egress to kube-system, ndots:5 amplification causing timeouts under load (see Networking guide), upstream `forward` plugin can't reach external DNS (check CoreDNS's own resolv.conf/upstream config).

---

## 9. Slow Pod Startup

**Symptoms:** Pod takes minutes to become Ready.

```bash
kubectl describe pod <pod> -n <ns>           # check timestamps: Scheduled → Pulling → Pulled → Started
kubectl get events -n <ns> --sort-by=.lastTimestamp
```

**Root causes & fixes:** large image size (optimize with multi-stage builds, smaller base images, layer caching, or a registry pull-through cache closer to nodes), slow init containers (parallelize or optimize what they do), missing `startupProbe` causing liveness to kill a slow-starting app before it's ready (add startupProbe with generous `failureThreshold`), image not cached on the node (first pull always slower — consider pre-pulling critical images via DaemonSet).

---

## 10. Node NotReady

**Symptoms:** `kubectl get nodes` shows `NotReady`, pods on it show `Unknown` or get evicted.

```bash
kubectl describe node <node>                 # Conditions section
# SSH to node (if accessible):
systemctl status kubelet
journalctl -u kubelet -f
df -h    # disk pressure?
free -h  # memory pressure?
```

**Root causes & fixes:** kubelet crashed/not running (check systemd status, restart), network partition between node and control plane (check security groups/NSGs, VPN/ExpressRoute health), disk pressure (kubelet stops reporting Ready if disk usage exceeds threshold — clean up or expand disk), certificate expiration (kubelet's client cert to API server expired — common on clusters that skip cert-rotation maintenance), container runtime (containerd) crashed/hung.

---

## 11. etcd Leader Election Loops

**Symptoms:** Frequent `etcd_server_leader_changes_seen_total` increments, API server latency spikes/timeouts.

```bash
etcdctl endpoint status --cluster -w table
etcdctl member list
# Check network latency between etcd members
```

**Root causes & fixes:** network latency/instability between etcd members (heartbeat timeouts causing false leader-lost detection) — check inter-node network health, especially across AZs; disk I/O too slow causing fsync delays that look like an unresponsive leader (`etcd_disk_wal_fsync_duration_seconds` high) — move to faster disks; CPU starvation on etcd nodes (co-located workloads competing for CPU) — dedicate nodes to etcd in production; clock skew between members affecting election timeout calculations.

---

## 12. API Server Timeouts

**Symptoms:** `kubectl` commands hang or return timeout errors; controllers report watch failures.

```bash
kubectl get --raw /healthz
kubectl get --raw /metrics | grep apiserver_request_duration_seconds
kubectl top pods -n kube-system -l component=kube-apiserver
```

**Root causes & fixes:** etcd itself is slow/overloaded (API server is only as fast as etcd underneath — check etcd health first), too many concurrent requests (check APF rejected/queued request metrics — a noisy client without proper backoff), API server resource-starved (check its own CPU/memory), large list/watch requests (e.g., a controller doing full LIST of every Secret cluster-wide repeatedly) — identify culprit via audit logs, add pagination/field selectors.

---

## 13. Certificate Expiration (kubeadm Clusters)

**Symptoms:** Sudden `x509: certificate has expired` errors across the cluster, often exactly 1 year after cluster creation (default kubeadm cert validity).

```bash
kubeadm certs check-expiration
```

**Root causes & fixes:** kubeadm-managed clusters generate certs with 1-year validity by default; without automated renewal, they silently approach expiry until a hard failure. Fix: `kubeadm certs renew all` (requires control-plane node access, then restart static pods by moving/restoring their manifests to force kubelet to recreate them). PREVENTION: automate renewal via a scheduled job or migrate to a managed Kubernetes offering (EKS/AKS/GKE) where the cloud provider handles control-plane cert rotation entirely.

---

## 14. PVC Stuck in Pending

**Symptoms:** `kubectl get pvc` shows `Pending` indefinitely.

```bash
kubectl describe pvc <pvc> -n <ns>
kubectl get storageclass
kubectl logs -n kube-system -l app=csi-provisioner  # or wherever your CSI controller runs
```

**Root causes & fixes:** no StorageClass exists/specified and no cluster default set; `WaitForFirstConsumer` binding mode (this is NORMAL — PVC stays Pending until a pod actually uses it, not a bug); cloud quota exhausted (max volumes per account/region); requested access mode unsupported by the provisioner (e.g., RWX on block-storage-only StorageClass); CSI controller pod itself unhealthy (check its logs for the actual cloud API error).

---

## 15. Ingress Not Routing

**Symptoms:** External requests to Ingress host return 404/502/503.

```bash
kubectl describe ingress <name> -n <ns>
kubectl get pods -n <ingress-controller-namespace>
kubectl logs -n <ingress-ns> -l app.kubernetes.io/name=ingress-nginx
```

**Root causes & fixes:** Ingress controller not installed/running (verify pods healthy); host/path rules don't match request exactly (check for trailing slash / rewrite-target annotation issues); backend Service has no healthy endpoints (same root cause chain as scenario #7); TLS secret missing/misconfigured for HTTPS Ingress (check `secretName` exists in the SAME namespace as the Ingress); IngressClass mismatch (multiple ingress controllers in cluster, Ingress object doesn't specify the right `ingressClassName`).

---

## 16. HPA Not Scaling

**Symptoms:** Load increasing but replica count stays flat.

```bash
kubectl describe hpa <name> -n <ns>          # current/target metrics + conditions
kubectl top pods -n <ns>                     # confirm metrics-server works at all
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml   # metrics-server aggregation health
```

**Root causes & fixes:** metrics-server not installed/unhealthy (`kubectl top` fails entirely — install/fix it first); pods have NO resource requests set (utilization % can't be computed — HPA shows "unknown"); custom metrics adapter (prometheus-adapter) misconfigured for custom/external metrics; already at `maxReplicas` ceiling (raise it if truly needed); stabilization window delaying an EXPECTED scale-down (not a bug).

---

## 17. StatefulSet Stuck During Update

**Symptoms:** Rolling update halts partway, some pods old version, some new, no progress.

```bash
kubectl rollout status statefulset/<name> -n <ns>
kubectl describe pod <new-version-pod> -n <ns>
kubectl get pdb -n <ns>
```

**Root causes & fixes:** new pod failing readinessProbe (update won't proceed past a non-Ready pod per OrderedReady semantics — same as Deployment rollout stall logic); PodDisruptionBudget too restrictive combined with a manual/voluntary disruption competing for the same "budget"; `partition` value in `updateStrategy` set higher than expected (intentionally pausing rollout at that ordinal — verify this isn't just working as designed for a canary pattern).

---

## 18. Job Never Completes

**Symptoms:** Job's pods keep failing and retrying, or Job shows `Active` indefinitely.

```bash
kubectl describe job <name> -n <ns>
kubectl get pods -n <ns> -l job-name=<name>
kubectl logs <pod> -n <ns>
```

**Root causes & fixes:** `backoffLimit` too high combined with a persistently failing pod (check pod logs for the actual application error causing every attempt to fail — fix the underlying bug, don't just adjust backoffLimit); `activeDeadlineSeconds` not set, so a genuinely hung job runs forever consuming resources (add a deadline); `parallelism`/`completions` misconfigured causing the Job to never reach its target completion count; a stuck pod in Terminating (see scenario #2) preventing the Job controller from creating replacement pods.

---

## 19. Secret Not Mounted / Not Found

**Symptoms:** Pod fails to start with `CreateContainerConfigError`, or app can't find expected env var/file.

```bash
kubectl describe pod <pod> -n <ns>            # exact "secret not found" error
kubectl get secret <name> -n <ns>
kubectl get serviceaccount <sa> -n <ns> -o yaml
```

**Root causes & fixes:** Secret doesn't exist in the SAME namespace as the pod (Secrets are namespace-scoped, can't reference cross-namespace directly); typo in Secret/key name referenced by `secretKeyRef`/volume; ServiceAccount lacks RBAC to read the Secret (rare, usually only relevant for custom automation, not standard pod-mounted secrets which kubelet handles with its own node-level permissions); External Secrets Operator hasn't synced yet (`ExternalSecret` status shows sync error — check ESO controller logs and the SecretStore's connectivity to the backend).

---

## 20. Network Policy Blocking Traffic

**Symptoms:** Connectivity worked before, breaks immediately after a NetworkPolicy change.

```bash
kubectl get networkpolicy -n <ns> -o yaml
kubectl exec -it <src-pod> -- nc -zv <dst-ip> <port>
# Calico: calicoctl get networkpolicy -o wide
# Cilium: cilium policy trace --src-k8s-pod=<ns>:<pod> --dst-k8s-pod=<ns>:<pod>
```

**Root causes & fixes:** new default-deny policy added without a corresponding explicit allow for a needed flow (classic "forgot to allow DNS" — always check egress to kube-system:53 first); label selector typo/case-mismatch between policy and target pods; `namespaceSelector` requires the namespace to carry a matching label (auto-added `kubernetes.io/metadata.name` since 1.21+, but custom labels need manual application); policies are ADDITIVE only — a common misconception is that a narrower policy can "override" a broader one; actually the union of all matching policies applies, so unexpected broad access might come from ANOTHER policy you forgot about, not the one you just changed.

---

## General Troubleshooting Methodology (Interview Framework)

When asked "how do you approach debugging in Kubernetes," structure your answer as:

```
1. SYMPTOM → Identify exact failure mode (kubectl get, describe, events)
2. SCOPE → Is it one pod, one node, one namespace, or cluster-wide?
3. TIMELINE → What changed recently? (deployment, config, node pool scaling,
   certificate expiry schedule, traffic pattern change)
4. LAYER → Application → Pod → Node → Network → Control Plane
   (work outward from the symptom, don't jump straight to etcd)
5. EVIDENCE → Logs (app + system), events, metrics (historical graphs
   showing WHEN it started, not just current state)
6. HYPOTHESIS → Form a specific, testable theory
7. VERIFY → Confirm via a targeted command/test BEFORE making changes
8. FIX → Smallest safe change, not a shotgun blast of unrelated changes
9. PREVENT → Add monitoring/alerting/automated test so it's caught
   earlier next time
```
