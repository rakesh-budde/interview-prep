# Section 17: Troubleshooting

Systematic troubleshooting in Kubernetes requires understanding which component is responsible for which part of the lifecycle, then narrowing the scope with targeted commands. This section covers common failure classes with step-by-step investigation flows.

## Subtopic Index

- [Troubleshooting Framework](#troubleshooting-framework)
- [Pod Issues](#pod-issues)
- [Node Issues](#node-issues)
- [Storage Issues](#storage-issues)
- [Network Issues](#network-issues)
- [DNS Issues](#dns-issues)
- [Scheduler Issues](#scheduler-issues)
- [API Server Issues](#api-server-issues)
- [etcd Issues](#etcd-issues)
- [Performance Issues](#performance-issues)

---

## Troubleshooting Framework

Always start with: **What changed?** Most failures happen at or shortly after a deployment, configuration change, cert rotation, or infrastructure event.

**Narrowing approach**:
1. Identify the blast radius (one pod? one node? one namespace? cluster-wide?).
2. Check Events first — they summarize what Kubernetes observed.
3. Check logs next — container stdout/stderr + component logs.
4. Check status/conditions on the affected object.
5. Cross-reference with recent changes (Git history, deploy history, cloud events).

```bash
# First commands for any mystery failure
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -30
kubectl describe <resource> <name> -n <namespace>
kubectl get pods -n <namespace> -o wide     # reveals node, IP, age, restarts
```

---

## Pod Issues

### CrashLoopBackOff
**Symptom**: pod restarts repeatedly, backoff increasing.
**Causes**: application crash, bad entrypoint, missing env/secret/configmap, failing probe.
```bash
kubectl logs <pod> --previous              # previous container's logs
kubectl describe pod <pod> | grep -A5 "Last State:"
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].state.terminated}'
```
Exit code 137 = OOMKilled or SIGKILL. Exit code 1 = application error. Exit code 126/127 = bad entrypoint/command.

### OOMKilled
```bash
kubectl describe pod <pod> | grep OOMKilled
kubectl top pod <pod>
# Check cgroup memory.events on the node
cat /sys/fs/cgroup/kubepods/burstable/pod<uid>/<container>/memory.events
```
Fix: increase memory limits; fix memory leak; use cgroups v2 `memory.oom.group` to kill all containers together.

### ErrImagePull / ImagePullBackOff
```bash
kubectl describe pod <pod> | grep -A5 Events
# "Failed to pull image": wrong image name, tag, or registry auth
kubectl get secret -n <namespace> | grep docker    # imagePullSecrets
crictl pull <image>  # test on the node
```

### ContainerCreating (stuck)
```bash
kubectl describe pod <pod>   # look for CNI, CSI, or runtime errors
journalctl -u kubelet | grep -E 'error|cni|volume' | tail -30
kubectl get volumeattachment  # stuck PVC attachment?
```

---

## Node Issues

### Node NotReady
```bash
kubectl describe node <node> | grep -A20 Conditions
kubectl -n kube-node-lease get lease <node> -o yaml   # check renewTime

# On the node:
systemctl status kubelet
journalctl -u kubelet | tail -50
systemctl status containerd
crictl info
df -h /var/lib/containerd    # disk pressure?
df -i /var/lib/kubelet       # inode exhaustion?
```

### DiskPressure / MemoryPressure
```bash
df -h && df -i                           # check both space and inodes
free -h                                   # memory available
crictl images | awk '{sum+=$3} END {print sum}' # image disk usage
crictl rmi --prune                        # remove unused images (safe)
```

### PLEG Unhealthy
```bash
journalctl -u kubelet | grep "PLEG is not healthy"
crictl ps -a | wc -l                     # too many containers?
systemctl restart containerd             # restart runtime if stuck
```

---

## Storage Issues

### PVC Stuck Pending
```bash
kubectl describe pvc <name>              # Events show reason
kubectl get sc                           # StorageClass exists?
kubectl get storageclass <sc> -o yaml | grep volumeBindingMode
# If WaitForFirstConsumer: need pod to be scheduled first
kubectl -n kube-system logs -l app=ebs-csi-controller -c csi-provisioner | tail -30
```

### Volume Attachment Failure
```bash
kubectl get volumeattachment | grep <pv-name>
kubectl describe volumeattachment <name>
# Stuck attachment from dead node:
kubectl delete volumeattachment <name>   # force cleanup
```

### Volume Mount Failure (pod stuck ContainerCreating)
```bash
kubectl describe pod <pod> | grep -i "mount\|volume\|attach"
journalctl -u kubelet | grep -E 'NodePublish|NodeStage|error' | tail -20
ls /var/lib/kubelet/pods/<uid>/volumes/  # check mount paths
dmesg | grep "I/O error"                # disk errors?
```

---

## Network Issues

### Pod Can't Reach Service
```bash
# 1. Verify service and endpoints exist
kubectl get svc <name> -n <ns>
kubectl get endpointslice -l kubernetes.io/service-name=<name> -n <ns>

# 2. Try by IP to separate DNS from connectivity
kubectl exec <pod> -- curl -v http://<cluster-ip>:<port>

# 3. Check kube-proxy rules
SVC_IP=$(kubectl get svc <name> -o jsonpath='{.spec.clusterIP}')
iptables-save | grep $SVC_IP             # or: ipvsadm -Ln | grep $SVC_IP

# 4. NetworkPolicy blocking?
kubectl get netpol -n <ns>
kubectl exec <pod> -- nc -zv <pod-ip> <port>   # direct pod IP
```

### Pod Can't Reach External IPs
```bash
kubectl exec <pod> -- curl https://1.1.1.1
kubectl exec <pod> -- ping 1.1.1.1     # NAT working?
# Check node's NAT/masquerade rule
iptables -t nat -L POSTROUTING -n | grep MASQUERADE
# No route to host? Check node network
ip route show                           # on the node
```

---

## DNS Issues

```bash
# 1. Basic DNS test
kubectl exec <pod> -- nslookup kubernetes.default

# 2. CoreDNS running?
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns | tail -20

# 3. NetworkPolicy blocking DNS egress?
kubectl get netpol -n <pod-namespace>

# 4. ndots causing NXDOMAIN storm?
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl -n kube-system top pod -l k8s-app=kube-dns   # CoreDNS CPU high?
```

---

## Scheduler Issues

```bash
# Check scheduler is running
kubectl -n kube-system get pods -l component=kube-scheduler

# Pending pods and why
kubectl get pods -A --field-selector=status.phase=Pending
kubectl describe pod <pending-pod> | grep -A20 Events
# Look for: Insufficient cpu/memory, taint, affinity, unbound PVC, quota

# Scheduler queue depths
kubectl get --raw='/metrics' | grep scheduler_pending_pods

# Scheduler leader
kubectl -n kube-system get lease kube-scheduler -o yaml
```

---

## API Server Issues

```bash
# Health checks
kubectl get --raw='/healthz'
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez?verbose'

# Latency metrics
kubectl get --raw='/metrics' | grep 'apiserver_request_duration_seconds' | grep 'le="1"'

# etcd latency (often root cause)
kubectl get --raw='/metrics' | grep 'etcd_request_duration_seconds'

# Webhook latency (second most common root cause)
kubectl get --raw='/metrics' | grep 'apiserver_admission_webhook_admission_duration'

# API Priority and Fairness (throttling)
kubectl get --raw='/metrics' | grep 'apiserver_flowcontrol_current_inqueue_requests'

# Self-managed: check apiserver logs
kubectl -n kube-system logs kube-apiserver-<node> | grep -E 'error|timeout|slow' | tail -30
```

---

## etcd Issues

```bash
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Cluster health
etcdctl endpoint health --cluster --write-out=table
etcdctl endpoint status --cluster --write-out=table

# WAL fsync latency (most important metric)
etcdctl endpoint status --write-out=json | python3 -m json.tool | grep -E 'dbSize|leader'
# Via Prometheus: etcd_disk_wal_fsync_duration_seconds p99 > 10ms = problem

# DB size (compaction needed?)
etcdctl endpoint status --write-out=table   # check DB SIZE column

# Quorum lost: check member list
etcdctl member list

# Leader election logs
journalctl -u etcd | grep -E 'elected|leader|term' | tail -20
```

---

## Performance Issues

### High Latency
Identify the layer: LB → Ingress → Service → Pod → Database.
```bash
kubectl exec <pod> -- curl -w "@curl-format.txt" http://backend-service/api  # timing breakdown
# curl-format.txt: "%{time_namelookup} %{time_connect} %{time_appconnect} %{time_pretransfer} %{time_starttransfer} %{time_total}"

# CPU throttling?
kubectl top pod <pod>
kubectl get --raw='/metrics' | grep container_cpu_cfs_throttled   # on metrics-server exposed

# Memory pressure?
kubectl top node
```

### High Memory Usage
```bash
kubectl top pod -A --sort-by=memory | head -20
kubectl describe node <node> | grep -A20 "Allocated resources:"
# Eviction threshold approaching?
kubectl describe node <node> | grep -A5 Conditions
```

### Slow Node
```bash
# Check node conditions
kubectl describe node <node>
# High iowait?
iostat -x 1 5
# High CPU steal?
top -b -n1 | grep Cpu
# Disk full?
df -h && df -i
```

---

## Interview Questions

### Conceptual / Internals (8 questions)

**1. A pod is Pending. How do you diagnose and what are the 5 most common root causes?**
`kubectl describe pod <name>` → Events. Common causes: (1) Insufficient node resources (CPU/memory requests too high). (2) Taint not tolerated (node has taint, pod has no matching toleration). (3) Node affinity/selector mismatch (pod requires a label that no node has). (4) PVC unbound (WaitForFirstConsumer — no pod scheduled yet, OR provisioner error). (5) Namespace ResourceQuota exceeded (admission denied but Events say "exceeded quota"). Also check: wrong schedulerName, all nodes cordoned, topology spread constraint unsatisfiable.

**2. Walk through debugging a CrashLoopBackOff container where logs are empty.**
Empty logs means the process crashed before writing anything (or wrote to stderr but it's not captured). Steps: (1) `kubectl describe pod` — check last exit code. 139=SIGSEGV, 137=SIGKILL/OOM, 126=not executable. (2) Override entrypoint to `sleep 3600` to keep the container alive: `kubectl set image deployment/myapp myapp=myapp:v1 && kubectl patch deployment myapp --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/command","value":["sleep","3600"]}]'`. (3) Exec in and try running the original command manually. (4) Check the image on a local machine. (5) If exit 137: memory limit too low.

**3. How do you debug a connection that works between pods on the same node but fails cross-node?**
Same-node works → kube-proxy rules are fine (no DNAT issue). Cross-node fails → CNI routing issue. Debug: (1) `kubectl exec <pod-node-1> -- ping <pod-ip-node-2>` — if ping fails, routing/CNI is broken. (2) `ip route get <pod-ip>` on node-1 — shows which interface handles it. (3) `tcpdump -i <cni-interface> host <pod-ip>` on node-1 — do packets leave node-1? (4) `tcpdump -i <cni-interface> host <pod-ip>` on node-2 — do packets arrive? If they leave but don't arrive: routing table issue, security group blocking, or encapsulation mismatch.

**4. An Ingress route returns 502. How do you identify whether it's the Ingress controller or the backend?**
502 = bad gateway — the Ingress controller reached the backend but got an error response (or the backend closed the connection). (1) Bypass Ingress: `kubectl exec <ingress-pod> -- curl http://<backend-service>:<port>/<path>` — if this fails, it's the backend. (2) Check Ingress controller access logs: `kubectl logs <ingress-pod> | grep 502`. The log shows the backend IP/port and the error (connection refused, timeout, etc.). (3) Check if the backend pod is running and ready: `kubectl get pod -l app=backend`. (4) Check if the Service endpoints exist: `kubectl get endpoints <backend-service>`.

**5. etcd is reporting high disk latency. What do you check and what are your immediate actions?**
Check `etcd_disk_wal_fsync_duration_seconds` p99. If >10ms: (1) Check the disk: `iostat -x 1 sda` — is it the etcd disk? High `%util` and `await` = I/O bound. (2) What else uses the disk? `iotop -b -n1 | head -20`. (3) Is it an HDD? Move to SSD immediately. (4) Is it a cloud disk being throttled? Check cloud provider metrics for I/O credit balance (EBS burst credits). Immediate actions: reduce load on etcd disk (separate WAL directory to a faster disk, stop non-critical writes), increase election timeout to prevent false leader elections while investigating.

**6. How do you diagnose high apiserver latency?**
Check which phase is slow: (1) Admission webhook latency: `apiserver_admission_webhook_admission_duration_seconds` — if a webhook is slow, it shows here. (2) etcd latency: `etcd_request_duration_seconds` — slow etcd propagates to all writes. (3) APF throttling: `apiserver_flowcontrol_current_inqueue_requests` — high queue means throttled. (4) Watch fan-out: `apiserver_watch_cache_capacity_*` — too many watchers. (5) CPU saturation on apiserver pods. Narrow: a single slow webhook can make all pod creates slow (if the webhook rule is broad).

**7. A node shows MemoryPressure but free memory looks fine. Explain.**
The kubelet's eviction threshold compares `memory.available` (from `/proc/meminfo`) against the hard threshold. Linux counts as "available" only MemFree + reclaimable buffers/cache. If a workload has created many hugetlb mappings, large mmap regions, or if the cgroup memory accounting differs from the system view, the kubelet may see pressure before `free -h` shows it. Also: the kubelet uses `--eviction-hard=memory.available<100Mi` and `--system-reserved` calculations. Verify with `kubectl describe node <node> | grep -A10 Conditions` and check both `MemAvailable` in `/proc/meminfo` AND the cgroup accounting.

**8. You're paged: cluster is completely unreachable (kubectl times out). What do you do first?**
(1) Check if the apiserver is up: `curl -sk https://<control-plane-ip>:6443/healthz`. If no response: apiserver is down. (2) Check cloud provider — is the control plane healthy (for managed clusters)? Check cloud console/status page. (3) For self-managed: SSH to a control plane node. Check apiserver, etcd, and kubelet status. (4) If etcd quorum lost: follow the etcd recovery runbook. (5) If apiserver OOMKilled: increase memory on control plane nodes. (6) If cert expired: renew certs with kubeadm. (7) Log the timeline for postmortem. Do not blindly restart components without understanding the cause.

### Scenario Questions (6 questions)

**9. Intermittent 5xx errors on a service that correlates with deployments. Diagnose.**
During rolling updates, the old pods are removed before all iptables rules update everywhere. Check: (1) Do errors correlate with specific pod IP removals? (use `kubectl get events | grep endpoint`). (2) Is there a preStop sleep? `kubectl get pod <pod> -o yaml | grep preStop`. (3) Is the app handling SIGTERM gracefully? `kubectl logs <pod> --previous | grep -i shutdown`. (4) Is the readiness probe gate too permissive (pod marked ready before actually ready)? Fix sequence: add `preStop: sleep 10`, ensure graceful shutdown in app, tune readiness probe.

**10. A Job ran successfully in staging but fails with "OOMKilled" in production. Memory limits are identical. Debug.**
Check: (1) Same data volume? Production may have larger payloads. (2) Same JVM/runtime settings? JVM may use more memory in production (more loaded classes, larger heap, GC overhead). (3) Any sidecar containers in production consuming memory? `kubectl get pod <job-pod> -o jsonpath='{.spec.containers[*].name}'`. (4) Are the node's `--system-reserved` and `--kube-reserved` different? More reserved = less allocatable memory. (5) Is the node under pressure from other workloads? `kubectl describe node <node> | grep -A20 "Allocated resources:"`. Diagnosis: increase limits by 2x, monitor with Grafana.

**11. kubectl apply works but pods don't start. No events visible.**
No events from the scheduler = scheduler isn't trying to schedule the pod. Reasons: (1) Wrong `schedulerName` in the pod spec. (2) Namespace ResourceQuota reached — admission rejected the pod but it's showing in the list (check `kubectl get pod <pod> -o yaml | grep status`). Actually Pending means it was admitted but not scheduled. (3) All nodes are cordoned. (4) Scheduler is down. Check: `kubectl -n kube-system get pods -l component=kube-scheduler`.

### FAANG Deep Dive (6 questions)

**12. Design a systematic troubleshooting methodology for unknown latency degradation in a microservices cluster.**
Layer-by-layer elimination: (1) **Is it all services or one?** Check error budget dashboards. One service → isolated issue. All services → infrastructure issue. (2) **Infrastructure**: node CPU/memory/disk pressure, network saturation, DNS latency, etcd latency. (3) **Control plane**: apiserver latency causing slow endpoint updates, slowing kube-proxy propagation. (4) **Service specific**: distributed traces to find slow span; RED metrics (rate/error/duration) to identify the bottleneck service; resource metrics (CPU throttling?) for that service. (5) **Database**: slow query logs, connection pool saturation. (6) **External dependencies**: third-party API latency (correlate with trace timing). Instrument: OTel traces mandatory, SLO dashboards, USE metrics per node.

**13. A large cluster has intermittent API call failures that correlate with controller-manager restarts. Explain the mechanism.**
When the controller-manager restarts, it loses all informer caches. On startup, it issues LIST requests for every resource type it watches (Deployments, ReplicaSets, Pods, etc.). In a large cluster, this is thousands of LIST calls simultaneously — a thundering herd on the apiserver. Simultaneously, the apiserver's etcd watches may expire (if the restart triggered a watch cache flush). etcd also sees a burst of LIST queries. APF (API Priority and Fairness) queues the excess requests. Low-priority clients (user kubectl, other controllers) receive 429 responses. The burst lasts 30–60 seconds as caches fill. Fix: increase apiserver APF concurrency for `workload-high` priority level; ensure controller restarts are infrequent; use startup probes on the controller so it's ready before accepting workloads.

**14. Explain how you would diagnose a cluster where pods are being OOMKilled but node memory appears plentiful.**
The disconnection between container OOMKills and node memory is explained by cgroup-level memory limits, not node-level limits. The pod's memory limit (enforced by the cgroup) can be exceeded even when the node has free memory. Scenarios: (1) Memory limit set too low for the workload. Metrics: `container_memory_working_set_bytes` approaching `container_spec_memory_limit_bytes` before kill. (2) Memory fragmentation: RSS is lower than working_set because of fragmented page allocations. (3) JVM off-heap allocations (direct buffers, thread stacks) not counted in `Xmx`. (4) Multiple containers in a pod competing for the pod-level cgroup limit. Diagnose: graph `container_memory_working_set_bytes / container_spec_memory_limit_bytes` over time. Look for steady growth (leak) vs spikes (burst).

---

## Production Incidents Summary

The top-10 Kubernetes production issues by frequency:

1. **CrashLoopBackOff** — app crash, bad config, probe failure
2. **Pending pods** — resource exhaustion, taints, PVC issues
3. **DNS failures** — CoreDNS OOM, NetworkPolicy blocking port 53
4. **Service unreachable** — kube-proxy rules stale, no endpoints
5. **Node NotReady** — kubelet/runtime failure, disk pressure
6. **OOMKilled** — memory limit too low, memory leak
7. **Rolling update stuck** — readiness probe failing on new pods
8. **Certificate expiration** — apiserver/etcd/kubelet certs
9. **etcd disk pressure** — WAL fsync slow, compaction needed
10. **Ingress 502/503** — backend unhealthy, misconfigured route

For each: use the framework (Events → Logs → Status → What changed) to systematically isolate the layer.
