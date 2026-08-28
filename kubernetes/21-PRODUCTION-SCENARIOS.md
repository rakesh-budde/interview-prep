# Section 21: Production Scenarios

100 real-world Kubernetes production scenarios, organized by domain. Each entry covers symptom, investigation commands, root cause, fix, and prevention.

## Subtopic Index

- [Pod Scenarios (1–20)](#pod-scenarios-120)
- [Node Scenarios (21–35)](#node-scenarios-2135)
- [Networking Scenarios (36–55)](#networking-scenarios-3655)
- [Storage Scenarios (56–65)](#storage-scenarios-5665)
- [Control Plane Scenarios (66–80)](#control-plane-scenarios-6680)
- [Security Scenarios (81–88)](#security-scenarios-8188)
- [Performance Scenarios (89–95)](#performance-scenarios-8995)
- [Cluster-Wide Scenarios (96–100)](#cluster-wide-scenarios-96100)

---

## Pod Scenarios (1–20)

**1. Pod stuck Pending — no resources**
Symptom: pod `Pending` indefinitely. `kubectl describe pod` shows `Insufficient cpu`. Investigation: `kubectl describe nodes | grep -A5 "Allocated resources"`. Root cause: all node CPU capacity is reserved by existing pods. Fix: right-size requests, add nodes, or use Karpenter. Prevention: set resource budgets per namespace, alert on node utilization >80%.

**2. CrashLoopBackOff — config missing**
Symptom: restart count climbing, exit code 1. `kubectl logs --previous` shows `Error: DATABASE_URL not set`. Root cause: Secret referenced in env is missing. Fix: create the secret, restart. Prevention: use ValidatingAdmissionPolicy to require referenced secrets exist before pod creation.

**3. OOMKilled — JVM heap**
Symptom: exit code 137, `OOMKilled` in container status. Root cause: JVM Xmx set lower than actual memory need; off-heap (metaspace, direct buffers) exceeded the limit. Fix: increase memory limit, add `-XX:+UseContainerSupport`. Prevention: monitor `container_memory_working_set_bytes / limits`.

**4. Init container deadlock**
Symptom: pod stuck in `Init:0/2` for hours. Investigation: `kubectl logs <pod> -c init-wait`. Root cause: init container waits for service X; service X's pod has an init container waiting for a service in this pod. Circular dependency. Fix: break the cycle by deploying one service without its dependency first. Prevention: map service dependencies; avoid circular init waits.

**5. Image pull backoff — ECR token expiry**
Symptom: `ImagePullBackOff`, event: `401 Unauthorized`. Root cause: the ECR authorization token (12h TTL) expired; the node's pull credential is stale. Fix: restart the failing pod (triggers a fresh token fetch via IRSA/node role). On EKS: ensure the node role has `ecr:GetAuthorizationToken`. Prevention: use IRSA with the AWS ECR credential helper; it refreshes tokens automatically.

**6. Pod evicted — DiskPressure**
Symptom: pods evicted from a node, event `The node was low on resource: ephemeral-storage`. Root cause: a container wrote large files to its writable layer (not a volume), filling node disk. Fix: delete the evicted pods; free disk; add `ephemeral-storage` requests/limits to pods; write logs/data to mounted volumes. Prevention: `limits.ephemeral-storage`, log rotation, PVC for high-write paths.

**7. ReadinessProbe too aggressive**
Symptom: pod continuously goes in and out of Service endpoints. `kubectl describe pod` shows frequent `Readiness probe failed`. Root cause: probe timeout too short for a GC pause in the JVM. Fix: increase `timeoutSeconds` or `failureThreshold`. Prevention: probe tuning in staging under realistic load; monitor `kube_pod_container_status_ready` transitions.

**8. PreStop hook ignored**
Symptom: 502 errors spike during rolling deploy. Investigation: correlate errors with specific pod IP departures; the pod starts refusing connections before iptables rules update. Root cause: no preStop hook; pod exits immediately on SIGTERM. Fix: add `preStop: exec: command: ["sleep","10"]`. Prevention: include preStop in Deployment template standards.

**9. Pod stuck Terminating — finalizer**
Symptom: pod `Terminating` for 2 hours; `kubectl delete --force` has no effect on the namespace. Investigation: `kubectl get pod <pod> -o yaml | grep finalizers`. Root cause: a custom controller added a finalizer but the controller crashed and never removed it. Fix: manually patch out the finalizer: `kubectl patch pod <pod> -p '{"metadata":{"finalizers":[]}}'`. Prevention: always implement finalizer cleanup with a timeout; alert on stuck termination >5 minutes.

**10. Liveness kills healthy pod**
Symptom: pods repeatedly restarted every few minutes despite no application crash. Investigation: `kubectl logs --previous` shows no error; Events show `Liveness probe failed: Get ... context deadline exceeded`. Root cause: liveness probe calls an external dependency (database); a transient DB timeout causes the container to be killed. Fix: remove DB health check from liveness probe; use a lightweight in-process health endpoint instead. Prevention: liveness should only check internal application health.

**11. StatefulSet pod not rescheduled after node failure**
Symptom: `mydb-1` stuck Terminating after its node failed; new `mydb-1` never starts. Root cause: `mydb-1` still has `spec.nodeName` pointing to the dead node and is still Terminating from the old node. Fix: force-delete the stuck pod: `kubectl delete pod mydb-1 --force --grace-period=0`. Prevention: configure short node taint tolerations for StatefulSet pods when quick recovery matters.

**12. Job created thousands of pods**
Symptom: etcd memory spiking; cluster API latency high. Investigation: `kubectl get pods -A | wc -l` shows 10,000 pods. Root cause: CronJob with `concurrencyPolicy: Allow` and a long-running job created 10,000 overlapping instances. Fix: set `concurrencyPolicy: Forbid`, delete old jobs. Prevention: always set `concurrencyPolicy` and `ttlSecondsAfterFinished`.

**13. Container running as root**
Symptom: security audit flags container running as UID 0. Root cause: Dockerfile has no `USER` instruction; default is root. Fix: add `USER 10001` to Dockerfile; set `runAsNonRoot: true`, `runAsUser: 10001` in securityContext. Prevention: PSA Restricted mode + OPA/Kyverno policy requiring non-root.

**14. Secret update not reflected in pod**
Symptom: after rotating a secret, pods still use the old value. Root cause: secret is mounted as environment variable, not a volume file. Environment variables are set at pod creation and never updated. Fix: restart the pods; for future, mount secrets as volume files (kubelet rotates them within ~60s). Prevention: use mounted volumes for secrets that need rotation; avoid env-var secrets for rotation-sensitive credentials.

**15. Zombie processes accumulating**
Symptom: node PID count growing; eventually triggers PIDPressure. Investigation: `kubectl exec <pod> -- ps aux | grep defunct`. Root cause: PID 1 is not reaping child zombies (no `wait()` called). Fix: use `tini` as PID 1: `ENTRYPOINT ["/tini", "--", "myapp"]`. Prevention: mandate tini in Docker base images; check for zombie accumulation in staging.

**16. Pod IP recycled — stale connection**
Symptom: after a pod restart, a downstream service gets intermittent connection errors. Root cause: the downstream cached the old pod IP (from a DNS lookup or direct connection). The new pod got the same IP from the CNI pool and is receiving requests intended for the old pod. Fix: use Services (not pod IPs) for all communication; implement proper retry/reconnect. Prevention: never communicate by pod IP; always use Service DNS names.

**17. ResourceQuota silently blocking pod creation**
Symptom: deployment updated; HPA triggers scale-out; no new pods appear; no error in deployment events. Investigation: `kubectl get events -n production | grep FailedCreate`. Shows `exceeded quota: requests.cpu: requested 200m, used 9800m, limited 10000m`. Root cause: namespace quota exhausted. Fix: increase quota or reduce requests. Prevention: alert on quota at 80%; expose quota usage in FinOps dashboards.

**18. Readiness gate never satisfied**
Symptom: pod Running and container healthy but never added to Service endpoints. Investigation: `kubectl get pod <pod> -o yaml | grep readinessGates`. Root cause: AWS ALB Controller readiness gate (`target-health.alb.k8s.aws/...`) is configured but the ALB target is not registered (e.g., wrong annotation on Service). Fix: fix the ALB annotation or remove the readiness gate. Prevention: test readiness gate behavior in staging with synthetic traffic.

**19. emptyDir fills from log writes**
Symptom: pod evicted for ephemeral-storage; container was writing logs to `/tmp` (an emptyDir). Root cause: application writes verbose logs to in-container paths; emptyDir counts against ephemeral storage. Fix: mount a dedicated emptyDir with `sizeLimit` at `/tmp`; reduce log verbosity. Prevention: set `limits.ephemeral-storage` on all pods; include log path in capacity planning.

**20. Multiple pods consuming same RWO PVC**
Symptom: data corruption on a database; two pods on the same node mounted the same PVC simultaneously. Root cause: `ReadWriteOnce` allows multiple pods on the same node; a scale-up created a second replica that scheduled to the same node. Fix: use `ReadWriteOncePod` (k8s 1.22+); set `replicas: 1` for the database. Prevention: RWOP access mode for single-writer databases; StatefulSet instead of Deployment.

---

## Node Scenarios (21–35)

**21. Node NotReady — kubelet OOMKilled**
Symptom: node NotReady; all pods on it show Unknown. Investigation: node event log shows kubelet process was killed. Root cause: kubelet's own memory usage exceeded the node's memory; no system-reserved carve-out. Fix: set `--system-reserved=memory=500Mi --kube-reserved=memory=1Gi` in kubelet config. Prevention: always set reserved resources; monitor kubelet memory.

**22. Node NotReady — containerd deadlock**
Symptom: node NotReady; ssh to node shows kubelet running; `crictl ps` hangs. Root cause: containerd deadlock on a corrupted image layer. Fix: `systemctl restart containerd`. Prevention: monitor containerd health; set PLEG relist alarm; consider eBPF-based health checks.

**23. DiskPressure from inode exhaustion**
Symptom: DiskPressure taint but `df -h` shows only 40% disk used. Root cause: `df -i` shows 99% inodes used; thousands of small files from OverlayFS copy-ups or log files. Fix: `crictl rmi --prune`; clear old log files; add inode monitoring. Prevention: use larger inode count at node provisioning (`mkfs.ext4 -N`); use xfs (dynamic inodes).

**24. Node stuck cordon/uncordon loop**
Symptom: node repeatedly gets cordoned and uncordoned by an autoscaler. Root cause: CA detects the node as under-utilized, attempts to drain; a DaemonSet pod with `safe-to-evict: "false"` blocks drain; CA gives up and retries. Fix: annotate the DaemonSet pod as safe to evict, or add a `cluster-autoscaler.kubernetes.io/safe-to-evict: "true"` annotation. Prevention: review DaemonSet annotations for safe-to-evict; test CA drain in staging.

**25. Kernel OOM kills kubelet**
Symptom: `systemd` restarts kubelet repeatedly; `dmesg` shows `Out of memory: Kill process <kubelet-pid>`. Root cause: a container memory burst plus OS overhead exceeded the node's total RAM; kubelet was chosen as the victim. Fix: add memory requests to all pods; set proper resource reservations. Prevention: enforce memory limits; use cgroupv2 `memory.oom.group` for better container-level OOM kills.

**26. Node time drift breaks certificates**
Symptom: API calls start failing with `x509: certificate has expired`; but the certificate's actual expiry is still in the future. Root cause: node clock has drifted 10+ minutes; TLS verification uses system clock to check certificate validity. Fix: restart `chronyd` or `ntpd` on the affected node; sync clocks. Prevention: monitor `node_timex_offset_seconds`; alert on clock skew >1 second.

**27. Node pool mix causes scheduling imbalance**
Symptom: one node pool at 90% utilization; another at 10%; pods queue even though "there is capacity." Root cause: taints/node-affinity rules restrict pods to the full pool; CA can't rebalance across pools. Fix: review taint/affinity rules; consider removing unnecessary restrictions; use topology spread constraints. Prevention: avoid implicit node pool affinity without explicit business reason.

**28. GPU node driver failure**
Symptom: ML training pods stuck Pending: `0/1 nodes are available: 1 Insufficient nvidia.com/gpu`. Root cause: NVIDIA Device Plugin DaemonSet pod on the GPU node is in CrashLoopBackOff; NVML library error after a kernel update. Fix: reinstall NVIDIA drivers; restart device plugin pod. Prevention: pin kernel version on GPU nodes; test driver compatibility before node OS updates.

**29. Node upgrade wipes custom kubelet flags**
Symptom: after a node pool upgrade on AKS, pods start being OOMKilled more aggressively. Root cause: custom `kubelet-config` with `evictionHard` thresholds was not preserved through the upgrade. Fix: re-apply custom kubelet configuration; restart kubelet. Prevention: store kubelet config in IaC; validate post-upgrade using integration tests.

**30. Zombie node — cloud instance terminated but node object persists**
Symptom: `kubectl get nodes` shows a node in Unknown state for 30+ minutes; pods on it are stuck. Root cause: EC2 instance was terminated by the cloud provider without going through a proper Kubernetes drain. The cloud-controller-manager (or lack of it) hasn't removed the Node object. Fix: `kubectl delete node <zombie-node>`. Prevention: ensure cloud-controller-manager is running; use spot interruption handlers.

**31. Node NotReady — network plugin failure**
Symptom: node condition `NetworkUnavailable: True`; all new pods stuck ContainerCreating. Root cause: Calico agent pod on that node crashed; CNI config file missing or corrupt. Fix: restart the Calico DaemonSet pod; if config is corrupt, reinstall on that node. Prevention: DaemonSet health monitoring; alert on `NetworkUnavailable`.

**32. Node high CPU from kube-proxy iptables**
Symptom: node at 100% CPU; `top` shows `iptables-restore` using 30%. Root cause: large cluster with 5,000+ Services; every endpoint change triggers full iptables-restore (O(n²)). Fix: switch kube-proxy to IPVS mode; or migrate to Cilium (eBPF O(1) updates). Prevention: plan for IPVS/eBPF dataplane before exceeding ~500 Services.

**33. Node pool autoscaling creates wrong instance type**
Symptom: Karpenter launches `t3.micro` (too small) for a GPU workload. Root cause: NodePool requirements didn't exclude small instance types; Karpenter selected the cheapest fitting option. Fix: add `karpenter.k8s.aws/instance-family: NotIn: [t3]` to NodePool requirements. Prevention: define explicit instance type requirements in NodePools.

**34. Node affinity misconfigured — all pods on one node**
Symptom: Deployment with 10 replicas; all 10 on node-1; node-1 fails → all 10 pods fail. Root cause: node affinity `preferredDuringScheduling` strongly prefers node-1 (weight=100); all pods cluster there. Fix: use TopologySpreadConstraints instead. Prevention: test pod distribution after deploy; alert on pods-per-node imbalance.

**35. Spot instance interruption cascade**
Symptom: 40% of pods evicted simultaneously during peak traffic. Root cause: Spot instance interruption of a large pool; no on-demand baseline; 40% capacity lost instantly. Fix: restore traffic by scaling on-demand. Prevention: always maintain an on-demand baseline (at least 20-30% of capacity); use instance diversification; test interruption handling.

---

## Networking Scenarios (36–55)

**36. Service unreachable — kube-proxy not running**
Symptom: pods can ping each other (pod IPs) but ClusterIPs are unreachable. Investigation: check kube-proxy DaemonSet: `kubectl -n kube-system get pods -l k8s-app=kube-proxy`. Root cause: kube-proxy DaemonSet was accidentally deleted. Fix: reapply kube-proxy DaemonSet. Prevention: protect system DaemonSets with PodDisruptionBudget and RBAC preventing accidental deletion.

**37. Intermittent 10% packet loss between pods**
Symptom: cross-node service calls have ~10% error rate; same-node calls succeed. Investigation: `mtr <pod-ip>` from another node shows packet loss at the node hop. Root cause: NIC MTU mismatch — node MTU is 1500 but VXLAN overhead requires MTU adjustment to 1450; oversized packets are silently dropped. Fix: set CNI MTU to 1450 (Flannel/Calico). Prevention: validate MTU settings at cluster setup; test with large payloads.

**38. DNS timeouts under high load**
Symptom: sporadic 5-second timeouts on service calls; traces show DNS resolution taking 5s. Investigation: CoreDNS CPU at 100%; high NXDOMAIN rate from `ndots:5` storm. Root cause: high-QPS microservice making many external API calls; ndots search domain generates 4 DNS queries per call. Fix: set `ndots: 1` on the affected deployment; add NodeLocal DNSCache. Prevention: NodeLocal DNSCache cluster-wide; alert on CoreDNS CPU >80%.

**39. Cross-namespace NetworkPolicy blocks traffic**
Symptom: service A in namespace `frontend` cannot reach service B in namespace `backend`. Investigation: `kubectl exec <frontend-pod> -- nc -zv backend-svc.backend 80` fails. Root cause: default-deny policy in `backend` namespace has no rule allowing ingress from `frontend`. Fix: add an ingress rule allowing from `frontend` namespace. Prevention: test cross-namespace connectivity after policy changes; use Cilium network flow logs for visibility.

**40. Ingress returns 404 for valid path**
Symptom: `curl -H "Host: api.example.com" http://<ingress-ip>/v2/endpoint` returns 404. Investigation: check Ingress: path is `/v2` but pathType is `Exact`, not `Prefix`. Root cause: `pathType: Exact` matches `/v2` only, not `/v2/endpoint`. Fix: change to `pathType: Prefix`. Prevention: test all Ingress paths as part of deploy pipeline.

**41. LoadBalancer stuck in pending**
Symptom: Service `EXTERNAL-IP: <pending>` for 20 minutes. Root cause: cloud-controller-manager is not running; or IAM permissions missing for LB creation; or ELB subnet not tagged. Investigation: CCM logs, AWS CloudTrail. Fix: add subnet tag `kubernetes.io/role/elb: 1`; fix IAM. Prevention: validate LB-relevant tags and IAM in IaC; test LB provisioning in staging.

**42. CNI IP pool exhausted**
Symptom: `ContainerCreating` on all new pods; event: `failed to allocate for range 0: no IP addresses available`. Root cause: small CIDR per node (e.g., /28 = 14 IPs) fully consumed. Fix: expand CIDR (requires node replacement); enable prefix delegation (EKS VPC CNI); configure more secondary IPs per node. Prevention: plan pod CIDR with 2× headroom; monitor IP utilization.

**43. Service returns traffic to wrong version**
Symptom: after a deploy, 50% of traffic still hits old version. Root cause: two Deployments have the same pod labels; Service selector matches both. Fix: add a `version` label to each Deployment's pod template; update Service selector. Prevention: use Helm chart with version in labels; GitOps with PR review.

**44. Egress gateway breaks after node replacement**
Symptom: traffic from a specific pod group loses its stable source IP. Root cause: Cilium EgressGateway policy assigns the gateway node by label; the node was replaced and the new node doesn't have the label. Fix: apply the gateway label to the new node. Prevention: manage node labels in IaC; monitor EgressGateway health.

**45. NodePort unreachable from external LB**
Symptom: cloud LB health checks fail on all nodes for the NodePort. Root cause: a firewall/security group rule doesn't allow the LB health check source IP to reach the NodePort. Fix: add security group rule allowing LB IPs to node NodePort range. Prevention: automate security group management with AWS LBC; validate after LB creation.

**46. Pod-to-pod traffic silently dropped post-upgrade**
Symptom: after a Calico upgrade, some pod pairs have intermittent connectivity loss. Root cause: Calico policy re-evaluation bug in the new version; policy compilation took 10 minutes causing traffic drops. Fix: rollback Calico to previous version; apply known-good policy. Prevention: canary Calico upgrades on non-production nodes; test east-west connectivity post-upgrade.

**47. mTLS breaks after certificate rotation**
Symptom: after Istio cert rotation, ~2% of inter-service calls fail with `CERTIFICATE_VERIFY_FAILED`. Root cause: stale connections in connection pools that were established with the old certificate don't renegotiate. Fix: restart Envoy sidecars to force new connections. Prevention: configure connection lifetime limits on all workloads; monitor Istio cert expiry; test rotation in staging.

**48. Headless Service DNS returns stale IPs**
Symptom: application connects to a crashed pod using an old IP from DNS cache. Root cause: client caches the DNS response (TTL 30s) and connects to the stale IP after the pod restarted. Fix: use a proper retry + backoff strategy; reduce DNS cache TTL for headless services. Prevention: implement client-side connection health checks; use service mesh for connection management.

**49. AWS NLB health check fails for gRPC**
Symptom: NLB marks all targets unhealthy; the gRPC service is actually healthy. Root cause: NLB health check uses TCP (default) but the app responds to health checks only on gRPC-over-HTTP/2; TCP connect succeeds but gets RST before HTTP/2 negotiation. Fix: use `service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: HTTPS` with a gRPC health check. Prevention: test NLB health checks for each protocol variant.

**50. Cross-AZ traffic costs spike**
Symptom: AWS data transfer bill tripled. Investigation: CloudWatch shows cross-AZ bytes from Kubernetes pods. Root cause: kube-proxy routes traffic to backends in any AZ; pods in AZ-a connect to service backends in AZ-b and AZ-c. Fix: enable topology-aware routing (`spec.internalTrafficPolicy: Local` or EndpointSlice hints); deploy per-AZ NAT gateways. Prevention: monitor cross-AZ bytes by namespace; use topology hints from day one.

**51. Network Policy default-deny blocks control plane health checks**
Symptom: nodes occasionally go NotReady despite being healthy. Root cause: a broad default-deny egress policy was applied to all namespaces including `kube-system`; kubelet lease renewal calls are blocked. Fix: add egress allow to apiserver IP for `kube-system` namespace. Prevention: test NetworkPolicy in staging; exclude `kube-system` from default-deny.

**52. Service mesh circuit breaker opens indefinitely**
Symptom: service B stops receiving traffic even after it recovers. Root cause: Istio outlier detection ejected service B endpoints; `baseEjectionTime × maxEjectionPercent` caused all endpoints to be ejected simultaneously; auto-reset timeout hadn't triggered. Fix: reduce `maxEjectionPercent` to 50%; tune `baseEjectionTime`. Prevention: test circuit breaker behavior under partial failure scenarios.

**53. Flannel vxlan traffic blocked by cloud security groups**
Symptom: pods on different nodes can't communicate on new cloud setup. Root cause: VXLAN uses UDP port 8472; cloud security groups only allow TCP. Fix: add UDP 8472 to inter-node security group rules. Prevention: document required CNI ports in infrastructure playbooks; automate SG rules in IaC.

**54. DNS resolution succeeds but connection refused**
Symptom: `nslookup my-service` resolves correctly; `curl http://my-service` returns `Connection refused`. Root cause: Service exists with a ClusterIP; but EndpointSlice has no ready endpoints (all backing pods failing readiness). Fix: fix the underlying pod readiness issue. Prevention: monitor `kube_service_spec_external_traffic_policy` and `kube_endpoint_ready`.

**55. BGP route flapping causes pod communication loss**
Symptom: in a Calico BGP cluster, pods have intermittent 1–2s communication blackouts every 30 minutes. Root cause: BGP keepalive timeout set too low; brief network jitter causes BGP session teardown and route withdrawal; routes flap. Fix: increase BGP holdtime and keepalive intervals. Prevention: use IPIP/VXLAN overlay instead of BGP in cloud environments; BGP requires stable L2/L3 adjacency.

---

## Storage Scenarios (56–65)

**56. PVC stuck Pending — StorageClass deleted**
Symptom: new PVCs all stuck Pending. Root cause: the default StorageClass was accidentally deleted. Fix: recreate the StorageClass. Prevention: protect StorageClasses with ResourceLock or admission policy preventing deletion.

**57. Volume attachment stuck after zone mismatch**
Symptom: pod ContainerCreating; VolumeAttachment stuck. Root cause: PVC was provisioned with `Immediate` binding mode in us-east-1a; pod was scheduled to us-east-1b. Fix: delete PVC; recreate StorageClass with `WaitForFirstConsumer`. Prevention: always use `WaitForFirstConsumer` for AZ-scoped block volumes.

**58. Database data corruption — two writers**
Symptom: PostgreSQL reports `invalid page in block`. Root cause: two PostgreSQL pods on the same node both mount the same `ReadWriteOnce` PVC simultaneously during a rolling deploy. Fix: use `ReadWriteOncePod`; set `replicas: 1` with strict PDB. Prevention: RWOP access mode; never use Deployment for single-writer databases.

**59. fsGroup causing 5-minute pod startup**
Symptom: pod startup takes 5+ minutes; no errors. Investigation: `kubectl describe pod` shows "Waiting for volume to be ready". Root cause: a PVC with 10M files triggers recursive `chown` for `fsGroup`. Fix: add `fsGroupChangePolicy: OnRootMismatch`. Prevention: set `OnRootMismatch` as default in Kyverno policy; avoid large file count volumes.

**60. CSI driver deleted with PVs still attached**
Symptom: pods can't start; CSI node plugin missing; NodePublishVolume fails. Root cause: CSI driver DaemonSet was accidentally deleted. Existing mounts may still work; new mounts fail. Fix: reinstall CSI driver. Prevention: protect CSI DaemonSets with RBAC; use GitOps for infrastructure components.

**61. etcd backup stored on the same disk**
Symptom: During a disk failure, both etcd data and the backup are lost. Root cause: `etcdctl snapshot save /var/lib/etcd/backup.db` — backup on the same device as data. Fix: restore from an off-disk backup (S3/Azure Blob). Prevention: always ship backups off-node; `etcdctl snapshot save s3://bucket/backup.db`.

**62. Volume expansion stuck at FileSystemResizePending**
Symptom: PVC shows new size but pod's filesystem shows old size. Root cause: block device was expanded but filesystem wasn't; pod is running and CSI driver doesn't support online resize. Fix: restart the pod to trigger NodeExpandVolume. Prevention: add `allowVolumeExpansion: true` to StorageClass; test expansion procedure.

**63. StatefulSet PVC leaked after deletion**
Symptom: 500 orphaned EBS volumes after decommissioning a StatefulSet. Root cause: StatefulSet was deleted; PVCs with `Retain` policy weren't cleaned up. Fix: script to identify and delete orphaned PVCs and PVs: `kubectl get pv | grep Released`. Prevention: use `persistentVolumeClaimRetentionPolicy: Delete` for non-critical StatefulSets; automate PVC cleanup.

**64. NFS mount hanging entire node**
Symptom: node NotReady; all pods Terminating; the node is completely unresponsive for operations. Root cause: NFS server unreachable; a pod with an NFS volume (hard mount, no timeout) causes the kernel to block indefinitely on NFS operations, hanging all threads. Fix: force-unmount the NFS share; recover the NFS server. Prevention: use `soft,timeo=30` NFS mount options; prefer EFS/Ceph over raw NFS; alert on NFS server health.

**65. PVC data intact but pod can't mount**
Symptom: pod shows `MountVolume.SetUp failed: rpc error: code = Internal desc = corruptfile`. Root cause: filesystem on the EBS volume is corrupted (abrupt instance termination without proper umount). Fix: detach EBS volume; attach to a maintenance EC2; run `fsck -f /dev/nvme1n1`; reattach. Prevention: use databases with write-ahead logging; journal filesystems (ext4/xfs); test with sudden node loss.

---

## Control Plane Scenarios (66–80)

**66. etcd quorum loss — 2 of 3 members down**
Symptom: kubectl hangs; apiserver logs `etcdserver: request timed out`. Root cause: 2 etcd members failed simultaneously. Fix: recover lost members using last snapshot + WAL replay; if data lost, restore from snapshot backup with `etcdctl snapshot restore`. Prevention: 5-member etcd for critical clusters; automated backup every 5 minutes; monitor member health.

**67. apiserver certificate expired**
Symptom: `kubectl` returns `x509: certificate has expired or is not yet valid`. Root cause: apiserver serving cert expired; kubeadm cert rotation was not performed. Fix: `kubeadm certs renew all`; restart control plane components. Prevention: monitor cert expiry with `kubeadm certs check-expiration`; alert 30 days before expiry; automate renewal.

**68. Admission webhook takes down entire cluster**
Symptom: no new pods can start; all pod creates fail with webhook error. Root cause: a newly deployed MutatingAdmissionWebhook with `failurePolicy: Fail` and broad rules (all pods) has an unhealthy backend. Fix: delete the webhook: `kubectl delete mutatingwebhookconfiguration <name>`. Prevention: `failurePolicy: Ignore` for non-critical webhooks; narrow `namespaceSelector`; health checks before broad rollout.

**69. apiserver OOM — large object in etcd**
Symptom: apiserver pods restart frequently; OOMKilled. Root cause: a team stored a 50MB binary in a ConfigMap; apiserver's watch cache holds all copies of it; at 10 watchers it's 500MB of memory. Fix: delete the ConfigMap; use object storage for large binaries. Prevention: admission policy blocking ConfigMap/Secret size >1MB; monitor etcd object sizes.

**70. kube-controller-manager leader election stuck**
Symptom: Deployments not reconciling; ReplicaSets not scaling; leader election Lease not renewing. Root cause: all three kube-controller-manager replicas are failing to acquire the Lease due to a NetworkPolicy accidentally blocking access to the apiserver from kube-system. Fix: fix NetworkPolicy. Prevention: never apply NetworkPolicy to kube-system without extensive testing; monitor leader election Lease age.

**71. Scheduler backlog — thousands of pending pods**
Symptom: 3,000 pods Pending; scheduler CPU at 100%. Root cause: a deployment of 3,000 jobs was submitted simultaneously; each pod has complex pod-affinity rules that make scheduling O(pods²). Fix: reduce pod-affinity complexity; use TopologySpreadConstraints; increase scheduler worker count. Prevention: load test scheduling with realistic manifests; avoid expensive pod-affinity rules.

**72. etcd compaction causes relist storm**
Symptom: every 5 minutes, apiserver latency spikes; controllers restart; NXDOMAIN DNS surge. Root cause: etcd compaction advances the revision past all open watch revisions; every informer gets 410 Gone; all controllers relist simultaneously. Fix: tune `--etcd-compaction-interval` to a longer interval; accept higher memory usage for less-frequent compaction. Prevention: monitor informer relist rate; distribute compaction timing.

**73. Control plane out of disk space**
Symptom: apiserver fails to start; logs show `no space left on device`. Root cause: audit log directory filled the control plane disk; no log rotation configured. Fix: clear old audit logs; configure rotation: `--audit-log-maxsize=100 --audit-log-maxbackup=5`. Prevention: always set audit log rotation parameters; monitor disk usage on control plane nodes.

**74. etcd too large — DB quota exceeded**
Symptom: all API writes fail with `etcdserver: mvcc: database space exceeded`; alarm set. Root cause: etcd DB grew to 8GB limit (default quota) from accumulated event objects and old revisions. Fix: `etcdctl compact <rev>`; `etcdctl defrag`; `etcdctl alarm disarm`. Prevention: set `--quota-backend-bytes=8589934592`; monitor `etcd_mvcc_db_total_size_in_bytes`; alert at 6GB.

**75. scheduler extender crashes cluster scheduling**
Symptom: all pods Pending; scheduler logs show extender errors. Root cause: a custom scheduler extender is returning invalid JSON; the scheduler falls back to all-reject behavior. Fix: remove the extender from scheduler config; restart scheduler. Prevention: test extender response format; add circuit breaker in extender usage.

**76. api-server too many open files**
Symptom: apiserver errors: `accept4: too many open files`. Root cause: default `ulimit -n` (1024) exhausted by thousands of watch connections. Fix: increase file descriptor limit: `ulimit -n 65536` in systemd service. Prevention: set `LimitNOFILE=65536` in apiserver systemd unit; monitor open file descriptor count.

**77. controller rate limiter causes slow deployments**
Symptom: Deployment rollout takes 2 hours for 500 pods. Root cause: default Deployment controller workers (5) with rate limiting; each pod create is delayed by the rate limiter when many errors occur. Fix: increase `--concurrent-deployment-syncs` in controller-manager flags. Prevention: tune controller concurrency based on cluster size; monitor reconcile queue depth.

**78. API priority and fairness blocks system calls**
Symptom: kubelet heartbeat failures; nodes going NotReady despite no actual failure. Root cause: a misbehaving controller using cluster-admin floods the apiserver; APF is not protecting the node-heartbeat priority level. Fix: create a FlowSchema giving node leases the highest priority; reduce the misbehaving controller's API rate. Prevention: customize APF FlowSchemas to protect critical system operations.

**79. OIDC provider downtime breaks kubectl**
Symptom: all `kubectl` commands fail with `401 Unauthorized` for all users. Root cause: the OIDC provider (e.g., Okta) has an outage; all OIDC-authenticated users can't authenticate. Fix: use an emergency break-glass certificate-based kubeconfig stored in a vault. Prevention: always maintain a certificate-based break-glass credential; test break-glass procedure quarterly.

**80. Multiple apiservers with split-brain watch cache**
Symptom: `kubectl get pods` returns different results on consecutive calls. Root cause: two apiserver replicas have inconsistent watch caches (one has stale data) due to network partition between apiservers and etcd. Fix: verify etcd quorum; identify and restart the stale apiserver. Prevention: monitor watch cache staleness; ensure apiserver replicas all have network path to etcd.

---

## Security Scenarios (81–88)

**81. ServiceAccount token used to exfiltrate secrets**
Symptom: CloudTrail shows unusual Kubernetes API calls from an unexpected source IP. Root cause: a pod's auto-mounted SA token was extracted by a container breakout. Fix: disable the SA token from the compromised pod; rotate secrets it could access. Prevention: disable auto-mount (`automountServiceAccountToken: false`); use IRSA/Workload Identity; audit SA token usage.

**82. Root container modifies host path**
Symptom: production nodes showing unexpected configuration changes. Root cause: a pod with `hostPath: /etc` mount (dev leftovers) and running as root modified node configs. Fix: delete the offending pod; restore node config. Prevention: PSA Restricted mode blocks hostPath; OPA/Kyverno policy denying dangerous hostPaths.

**83. Cluster-admin binding from CI pipeline**
Symptom: security scan finds a CronJob SA with cluster-admin rights deployed last week. Root cause: CI pipeline used a template with overly permissive SA; no RBAC review in PR process. Fix: remove the binding; replace with minimal-rights SA. Prevention: RBAC review in PR pipelines; Kyverno policy blocking cluster-admin for non-system subjects.

**84. Unsigned image deployed**
Symptom: Kyverno policy-report shows violation for a recent pod. Root cause: developer pushed a hotfix using an untagged/unsigned image; CI pipeline was bypassed. Fix: rebuild and sign the image; restart the pod with the signed image. Prevention: webhook enforcing signed images; prevent bypass of CI pipeline with branch protection.

**85. etcd data exposed in backup**
Symptom: compliance audit finds etcd backup file on a public S3 bucket. Root cause: backup script used wrong S3 bucket name; no encryption on the backup. Fix: move/delete exposed file; rotate all secrets in etcd. Prevention: encrypt etcd backups; restrict S3 bucket policies; test backup restoration procedure (not just backup creation).

**86. Falco alert: crypto-mining**
Symptom: Falco fires `Detect outbound connections to common miner pool ports`. Root cause: attacker exploited a web shell in an improperly secured web app pod; installed a miner. Fix: isolate the pod; kill the miner process; patch the web app. Prevention: NetworkPolicy blocking crypto-miner ports; Falco rules; resource limits (CPU cap limits miner profitability).

**87. PodSecurityPolicy bypass via ephemeral container**
Symptom: security audit shows privileged access was obtained via an ephemeral debug container. Root cause: old cluster with PSP enabled; ephemeral containers didn't go through PSP admission. Fix: upgrade to PSA; test ephemeral container security. Prevention: use PSA which covers ephemeral containers; restrict `pods/ephemeralcontainers` subresource RBAC.

**88. Certificate pinning breaks rolling upgrade**
Symptom: during a cert rotation, 50% of clients fail with cert validation errors. Root cause: mobile clients had certificate-pinned the old cert; new cert is valid but pinned clients reject it. Fix: client-side cert pinning removal; gradual rollout with both old and new cert served. Prevention: avoid cert pinning; use CA pinning instead of leaf cert pinning.

---

## Performance Scenarios (89–95)

**89. High p99 latency correlated with GC**
Symptom: p99 request latency spikes every 2 minutes for exactly 200ms. Root cause: JVM full GC pause; correlated with high heap allocation rate. Fix: tune GC (G1GC with smaller region size); increase heap; reduce allocation rate. Prevention: monitor JVM GC metrics (`jvm_gc_pause_seconds`); baseline GC behavior in staging.

**90. CPU throttling causing latency not OOM**
Symptom: high p99 latency; CPU utilization only 30% on `kubectl top`. Root cause: CPU limit = 500m; the application has burst activity consuming 2 CPUs for 50ms; quota exhausted; throttled for rest of the 100ms period. Fix: increase CPU limit or reduce period. Prevention: monitor `container_cpu_cfs_throttled_seconds_total`; alert on throttled fraction >20%.

**91. etcd slow writes degrading whole cluster**
Symptom: all API calls have high latency (>2s). Root cause: etcd WAL fsync taking 150ms on an HDD. Fix: migrate etcd to SSD immediately; interim: reduce etcd write rate. Prevention: SSDs mandatory for etcd; monitor `etcd_disk_wal_fsync_duration_seconds_bucket` p99 <10ms.

**92. kube-dns timeout storm**
Symptom: application throughput drops 50%; DNS queries timing out. Root cause: a new Deployment with 100 pods making 1000 DNS queries/s each; total 100,000 q/s overwhelming CoreDNS (capacity ~50,000 q/s). Fix: scale CoreDNS to 8 replicas; deploy NodeLocal DNSCache; reduce ndots. Prevention: load test DNS capacity before large deployments.

**93. Memory pressure from large watch caches**
Symptom: apiserver memory growing indefinitely; eventually OOMKilled. Root cause: cluster has 10M objects (Event objects from a noisy controller); apiserver watch cache holds all of them in memory. Fix: reduce Event TTL; fix the noisy controller; limit watch cache size. Prevention: monitor apiserver memory growth; set EventTTL on apiserver; alert on excessive object counts.

**94. Slow rolling deploy from registry throttling**
Symptom: a 500-pod rolling deploy takes 4 hours; image pulls take 5 minutes each. Root cause: all 500 nodes simultaneously pulling the same large image from DockerHub; hit rate limit (100 pulls/6h for anonymous). Fix: configure a registry mirror or pull-through cache; pre-pull on nodes. Prevention: use ECR/ACR/GCR as registry with no rate limits; regional registry mirror.

**95. Pod anti-affinity makes scheduling O(n²)**
Symptom: cluster of 1000 pods takes 10 minutes to schedule a 100-pod batch. Root cause: every pod has `requiredDuringScheduling podAntiAffinity` checking all 1000 other pods. Fix: replace with `TopologySpreadConstraints`. Prevention: load test scheduling with representative pod counts; avoid pod-affinity for large workloads.

---

## Cluster-Wide Scenarios (96–100)

**96. Mass eviction from misconfigured PodDisruptionBudget**
Symptom: a cluster upgrade triggers simultaneous eviction of all pods from every node. Root cause: PDB was set as `minAvailable: 0` (should be 1); node drain evicted all pods without restriction. Fix: correct PDB to `minAvailable: 1`; investigate data loss from evictions. Prevention: validate PDB values in CI; test drain behavior in staging.

**97. GitOps sync wipes production namespace**
Symptom: all Services, Deployments, and ConfigMaps in production suddenly deleted. Root cause: ArgoCD prune was enabled; a Git commit accidentally deleted manifests; ArgoCD applied the deletion to the cluster. Fix: immediately restore from git history; reapply manifests; verify no data loss. Prevention: require PR review for manifest deletions; use ArgoCD sync-windows to prevent auto-sync during risky periods; `argocd app set --sync-option Prune=false` for production.

**98. Karpenter disrupts all nodes simultaneously**
Symptom: entire cluster experiences 5-minute outage. Root cause: Karpenter consolidation decided to replace all nodes simultaneously; no PDBs or disruption budgets configured. Fix: restart all pods; set Karpenter `disruption.maxNodeConsolidationNodeGroupConcurrency`. Prevention: configure PDBs for all workloads; set Karpenter disruption budgets; test consolidation in staging.

**99. Certificate rotation outage**
Symptom: cluster-wide connectivity loss lasting 15 minutes. Root cause: a certificate rotation script rotated all certificates simultaneously; brief window where some components had new certs and others had old, causing mTLS handshake failures across the entire control plane. Fix: wait for rotation to complete; restart components in order. Prevention: rotate certificates one-by-one with health checks between steps; have a rollback procedure.

**100. Namespace stuck Terminating**
Symptom: a namespace has been Terminating for 3 days. Root cause: a finalizer from an uninstalled operator is still on the namespace. Fix: list finalizers: `kubectl get namespace <ns> -o json | grep finalizers`. Patch: `kubectl patch namespace <ns> -p '{"spec":{"finalizers":[]}}'  --type=merge`. Prevention: when uninstalling operators, run cleanup step that removes finalizers; use operator lifecycle manager (OLM).

---

## Key Diagnostic Commands Reference

```bash
# Global health overview
kubectl get componentstatuses
kubectl get nodes -o wide
kubectl get pods -A | grep -v Running | grep -v Completed
kubectl get events -A --sort-by='.lastTimestamp' | tail -50

# Quick pod triage
kubectl describe pod <pod>                    # Events, conditions, last state
kubectl logs <pod> --previous --tail=100       # Previous container logs
kubectl top pod <pod>                          # Resource usage

# Node investigation
kubectl describe node <node>                  # Conditions, taints, allocated
kubectl -n kube-node-lease get lease <node>   # Heartbeat freshness
crictl ps -a                                  # All containers on node
journalctl -u kubelet --since "5m ago"         # Kubelet logs

# Networking
kubectl exec <pod> -- nslookup kubernetes.default
kubectl exec <pod> -- nc -zv <svc> <port>
iptables-save | grep <ClusterIP>
conntrack -C && cat /proc/sys/net/netfilter/nf_conntrack_max

# etcd health
etcdctl endpoint health --cluster --write-out=table
etcdctl endpoint status --cluster --write-out=table
kubectl get --raw='/metrics' | grep etcd_disk_wal_fsync
```
