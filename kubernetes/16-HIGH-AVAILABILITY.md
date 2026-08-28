# Section 16: High Availability

High Availability in Kubernetes means the cluster and the workloads it runs continue serving despite failures at any layer — individual pods, nodes, availability zones, or regions.

## Subtopic Index

- [Control Plane HA](#control-plane-ha)
- [etcd HA](#etcd-ha)
- [Worker Node HA](#worker-node-ha)
- [Multi-AZ Architecture](#multi-az-architecture)
- [Multi-Region Architecture](#multi-region-architecture)
- [Disaster Recovery](#disaster-recovery)
- [Backup Strategy](#backup-strategy)
- [Pod Disruption Budgets](#pod-disruption-budgets)

---

## Control Plane HA

A highly available control plane runs multiple apiserver, scheduler, and controller-manager replicas across different nodes/AZs.

**apiserver HA**: stateless — run 2–5 replicas behind a load balancer. All replicas serve requests from shared etcd state. Requests go to any replica. In managed clusters (EKS, AKS, GKE), this is handled by the provider.

**scheduler and controller-manager HA**: use leader election (Lease objects). Only one active replica runs at a time; others are hot standbys. Failover happens within `leaseDurationSeconds` (default 15s) of the leader failing.

**Self-managed cluster HA pattern (kubeadm)**:
1. 3 control plane nodes, each running apiserver+scheduler+controller-manager+etcd.
2. External or cloud load balancer in front of all apiserver replicas.
3. Each node in a different AZ.
4. Tolerates 1 control plane node failure while maintaining etcd quorum and apiserver availability.

```bash
# Check apiserver replicas (managed)
kubectl -n kube-system get pods -l component=kube-apiserver

# Check leader election state
kubectl -n kube-system get lease kube-controller-manager -o yaml
kubectl -n kube-system get lease kube-scheduler -o yaml

# Simulate apiserver loss (on a test cluster)
# docker pause <apiserver-container>
# Observe: kubectl commands fail, running pods continue
```

---

## etcd HA

Covered in depth in Section 4. Key HA points:

3-node etcd cluster: survives 1 simultaneous failure. 5-node: survives 2. Never run 2 or 4 nodes (no fault tolerance advantage, just more complexity).

Run etcd on dedicated disks (not shared with OS). Use SSDs. Separate etcd member per AZ. Monitor `etcd_disk_wal_fsync_duration_seconds` p99 — >10ms is a warning.

Backup automatically every 5–30 minutes. Test restores quarterly. Store backups in a different region from the cluster.

---

## Worker Node HA

**Multi-AZ node groups**: spread worker nodes across at least 3 AZs. A single AZ failure should not impact cluster capacity significantly.

**Node redundancy**: never run critical workloads with `replicas: 1`. Always set `minAvailable: 1` PDB at minimum. For 3 replicas, set `maxUnavailable: 1`.

**Topology Spread Constraints**: ensure pods are spread across AZs and nodes:
```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels: {app: payments}
```

**Graceful node maintenance**: `kubectl drain <node>` respects PDBs. Always cordon first (`kubectl cordon`) to prevent new scheduling, then drain.

---

## Multi-AZ Architecture

Multi-AZ is the baseline for production availability. Design principles:

1. **Run ≥ 2 replicas** per service, spread across AZs via TopologySpreadConstraints.
2. **Per-AZ NAT Gateways**: avoid cross-AZ traffic charges and SPOF.
3. **Zone-aware volume provisioning**: `volumeBindingMode: WaitForFirstConsumer`.
4. **Zone-aware Service routing**: `topology.kubernetes.io/zone` hints in EndpointSlices; kube-proxy prefers same-zone endpoints.
5. **Test AZ failure**: periodically simulate: cordon all nodes in one AZ, verify services remain healthy on remaining AZs.

For a 3-AZ cluster with N replicas: losing one AZ means ~1/3 of pods are evicted (toleration seconds expire). The remaining 2/3 on 2 AZs must handle full load. Plan for 150% of normal capacity spread across 3 AZs so a single-AZ failure leaves you at 100% capacity on 2 AZs.

---

## Multi-Region Architecture

Multi-region provides DR and potentially lower global latency but adds significant complexity: data replication, consistent deployments, global routing.

**Active-Passive**: one primary region serves all traffic. A standby region is pre-provisioned but not serving. On failure: update DNS to route to standby, promote standby DB to primary. RTO: minutes (DNS TTL + manual promotion). RPO: time since last DB replication/backup.

**Active-Active**: multiple regions serve traffic simultaneously. Global routing (Route 53, Cloudflare) with latency-based or geo routing. Each region has its own database with bidirectional replication (DynamoDB Global Tables, CockroachDB, Vitess). Conflicts from concurrent writes must be handled by the application.

Multi-region Kubernetes patterns:
- Identical cluster configuration via GitOps (ArgoCD ApplicationSet across multiple clusters).
- Data plane: DynamoDB Global Tables / Aurora Global / Kafka MirrorMaker.
- Routing: Route 53 with health checks and weighted routing.
- Each cluster is independent — cross-cluster service discovery via DNS or a service mesh federation.

---

## Disaster Recovery

DR plans must be documented, tested, and automated. Key metrics:
- **RTO (Recovery Time Objective)**: how long can you be down?
- **RPO (Recovery Point Objective)**: how much data loss is acceptable?

**etcd-based cluster recovery** (worst case):
1. Restore etcd from latest snapshot to all 3 members simultaneously.
2. Start control plane components.
3. Nodes re-register; workloads resume on surviving nodes.
4. Total RTO: 30–60 minutes depending on snapshot recency and cluster size.

**Faster recovery with GitOps**:
1. Spin up new cluster from IaC (Terraform/Pulumi) — 10–15 minutes.
2. Apply GitOps manifests via ArgoCD — 5–10 minutes.
3. Restore data from backup/replica.
4. Update DNS to new cluster.

**Game days**: simulate failure scenarios periodically (AZ loss, node failures, network partitions) in staging. Validate runbooks, fix gaps before a real incident.

---

## Backup Strategy

**etcd snapshot**: captures all Kubernetes API objects. Does NOT capture application data in volumes. Schedule: every 5–30 minutes. Retention: hourly for 24h, daily for 7d, weekly for 4w. Test: restore into a non-production cluster quarterly.

**Volume backups (Velero)**: snapshots PersistentVolumes (via CSI snapshots) and serializes all Kubernetes objects (as YAML). Can restore individual namespaces or full clusters. For application consistency: use pre/post backup hooks.

```yaml
# Velero Schedule
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: cluster-backup
  namespace: velero
spec:
  schedule: "0 */6 * * *"   # every 6 hours
  template:
    includedNamespaces: ["*"]
    storageLocation: default
    volumeSnapshotLocations: ["default"]
    ttl: "720h"               # 30 days retention
```

---

## Pod Disruption Budgets

PDB limits the number of pods that can be simultaneously unavailable during voluntary disruptions (node drain, rolling update, Karpenter consolidation):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payments-pdb
spec:
  selector:
    matchLabels: {app: payments}
  maxUnavailable: 1     # at most 1 pod unavailable at a time
  # OR
  minAvailable: 3       # at least 3 pods must be available
```

PDB interacts with: `kubectl drain` (respects PDB), Cluster Autoscaler scale-down (respects PDB), Karpenter disruption (respects PDB), StatefulSet rolling updates (checks PDB before deleting each pod).

A PDB of `minAvailable: 100%` (or `maxUnavailable: 0`) blocks ALL voluntary evictions — use only if needed, as it prevents node maintenance.

---

## Interview Questions

### Conceptual / Internals (8 questions)

**1. How many control plane nodes do you need for HA and why not 2?**
3 nodes minimum. etcd requires a quorum majority to accept writes: 2 nodes gives a quorum of 2 — if 1 fails, the remaining 1 has no quorum and the cluster stops accepting writes. 3 nodes give a quorum of 2 — 1 node can fail while 2 remain to maintain quorum. 4 nodes still only tolerates 1 failure (quorum = 3 of 4) while adding operational complexity. 5 nodes tolerate 2 failures. Use 3 for most production clusters; 5 for clusters requiring tolerance of 2 simultaneous failures.

**2. What is the difference between HA (high availability) and DR (disaster recovery)?**
HA: continuous availability despite component failures — automatic failover, no significant downtime. Achieved via redundancy (multiple replicas, multi-AZ), health checks, and automatic recovery. DR: recovery after a catastrophic event (entire region loss, data corruption, ransomware) — typically involves some downtime and possibly some data loss. Achieved via backups, replicas in another region, and tested runbooks. HA operates with RTO/RPO in seconds to minutes; DR operates with RTO/RPO in minutes to hours. Both are required — HA doesn't replace DR.

**3. Explain how a Topology Spread Constraint ensures AZ distribution and what happens when an AZ fails.**
TSC with `topologyKey: topology.kubernetes.io/zone` counts pods per zone and rejects scheduling to a zone that would exceed `maxSkew` beyond the least-loaded zone. This spreads pods evenly across zones at deployment time. When an AZ fails: the node lifecycle controller taints nodes `not-ready:NoExecute`. Pods tolerate this for their tolerance window (default 300s), then are evicted. The scheduler reschedules evicted pods to remaining zones. With `whenUnsatisfiable: ScheduleAnyway`, the scheduler proceeds even if it can't maintain the spread — crucial for failover. With `DoNotSchedule`, pods stay pending until the AZ recovers — wrong for failover.

**4. How does a PodDisruptionBudget interact with kubectl drain?**
`kubectl drain` evicts pods from the node. Before evicting each pod, it checks the PDB: `currentUnavailable + 1 <= maxUnavailable`. If evicting this pod would violate the PDB (too many already unavailable), the drain waits. It retries periodically until: (a) another pod becomes available (e.g., replacement scheduled on another node), (b) the timeout is reached (`--timeout` flag, default unlimited), or (c) you `--force` (which can violate PDB). This means a drain can block indefinitely if PDB constraints can't be satisfied — e.g., if there are no available nodes for the replacement pods.

**5. How does static stability apply to Kubernetes and why is it important for HA?**
Static stability: the data plane (running workloads) continues operating even when the control plane is unavailable or degraded. Running pods don't stop when the apiserver is down. kube-proxy's iptables rules persist. CNI routes remain. CSI mounts remain. The control plane is needed only for changes (new pods, scaling, updates). Design for static stability: (1) pre-provision sufficient capacity so AZ failure doesn't require control-plane-dependent scaling; (2) pre-cache secrets/configs (avoid runtime API calls in hot path); (3) use short-lived credentials that have long enough TTLs to survive a 1-hour control plane outage.

**6. What is the trade-off between RPO and cost in an etcd backup strategy?**
Lower RPO = more frequent backups = more storage, more compute, more network for transfers. A 5-minute RPO requires 12 backups/hour — if each is 1GB, that's 12GB/hour of backup storage writes. Over 30 days with 7-day retention: ~8TB. A 1-hour RPO reduces this 12x. For most Kubernetes clusters, the Kubernetes object state (etcd contents) is reproducible from GitOps — the critical RPO is for application data in volumes (Velero PV snapshots), not etcd. Set etcd RPO at 30 minutes; set volume backup RPO at whatever the business requires (often 1-4 hours for most workloads).

**7. How does multi-region active-active differ from active-passive and when would you choose each?**
Active-passive: all production traffic goes to Region A. Region B is a warm standby — pre-provisioned but serving 0 traffic. Failover: update DNS TTL to 0 before expected maintenance, then flip DNS on failure. Simpler (no conflict resolution), lower cost (standby can be smaller). Downtime during failover: DNS TTL (60–300s) + health check propagation (30–60s). Active-active: both regions serve production traffic simultaneously. Required when: single-region latency is unacceptable for global users, or single-region capacity is insufficient. Requires: conflict-free or conflict-resolved data replication, consistent secret management across regions, and global load balancing. Much more complex. Choose active-passive when one region can handle full load and latency allows. Choose active-active when you need global distribution or full-capacity failover.

**8. What does a Kubernetes "game day" look like and why is it necessary?**
A game day is a controlled failure exercise to validate that HA and DR mechanisms work as expected. Typical exercises: (1) Cordon all nodes in AZ-1, verify services serve from AZ-2 and AZ-3. (2) Delete the kube-controller-manager leader pod, verify another replica takes over within 15s. (3) Restore from an etcd backup into a non-production cluster, verify all objects are present and workloads start. (4) Simulate OOMKilled pods, verify alerting fires and recovery completes. (5) Verify Velero restore of a namespace. Game days are necessary because DR procedures documented but never tested often fail: DNS TTLs are wrong, IAM permissions for the DR account expired, the restore procedure has undocumented steps, or the backup is corrupt.

### Scenario Questions (6 questions)

**9. An AZ goes down. Walk through what happens to your 3-replica Deployment spread across 3 AZs.**
AZ-1 nodes lose network connectivity to apiserver. After 40s without lease renewal, nodes get `not-ready:NoExecute` taint. Pods have default `tolerationSeconds: 300` for this taint. After 300s (~5 minutes from failure), pods on AZ-1 are evicted. The Deployment's ReplicaSet controller sees 1 pod unavailable, creates a replacement. The scheduler assigns it to AZ-2 or AZ-3 (TSC ensures spread across remaining AZs). The PDB (`maxUnavailable: 1`) is temporarily violated during the 300s window — this is expected. After 6–8 minutes total, all 3 replicas are running on AZ-2 and AZ-3. Traffic during this window: if using LoadBalancer with `externalTrafficPolicy: Local`, health checks detect AZ-1 nodes have no pods and stop routing to them within 30–60s.

**10. A disaster scenario: the entire cluster's etcd data is corrupted. What is your recovery process?**
1. Identify the last good backup: `ls -lt /backups/etcd/ | head -5`.
2. Verify snapshot integrity: `etcdctl snapshot status <backup-file>`.
3. Stop all control plane components (apiserver, scheduler, controller-manager) on all nodes.
4. On each control plane node, restore from the same snapshot: `etcdctl snapshot restore <backup> --name etcd-N --initial-cluster ... --data-dir /var/lib/etcd-restore`.
5. Move restored data into place, restart etcd on all nodes.
6. Verify etcd health: `etcdctl endpoint health --cluster`.
7. Restart apiserver. Verify: `kubectl get nodes`, `kubectl get pods -A`.
8. Assess data loss: compare API objects to last known good state. Redeploy workloads created after the last backup via GitOps.

**11. You need zero-downtime maintenance on a 3-node control plane. What is your process?**
One node at a time, respecting etcd quorum. Step 1: identify the current etcd leader and move it away from the node you're maintaining: `etcdctl move-leader <follower-id>`. Step 2: cordon the node (`kubectl cordon node-1`) — new pods won't schedule here. Step 3: drain the node (`kubectl drain node-1 --ignore-daemonsets`). Step 4: perform maintenance. Step 5: uncordon (`kubectl uncordon node-1`). Step 6: verify control plane health (`kubectl get componentstatuses`, etcd endpoint health). Step 7: repeat for node-2 and node-3. Never maintain multiple control plane nodes simultaneously — you'd lose etcd quorum.

### FAANG Deep Dive (6 questions)

**12. How would you design a Kubernetes cluster that maintains 99.99% uptime (52 minutes downtime/year)?**
99.99% leaves 52 minutes/year for unplanned downtime. Approach: (1) Multi-AZ control plane (3 AZs) on dedicated nodes with etcd SSD and separate from worker traffic. (2) Multi-AZ worker nodes with TSC ensuring pod spread. (3) All workloads have `minReplicas >= 2`, PDB `maxUnavailable: 1`. (4) Pre-scale capacity for AZ-loss (3 AZs × 150% per-AZ capacity). (5) Automated failover: health checks, Route 53 TTL = 30s, readiness probes, preStop hooks. (6) Static stability: no runtime control-plane dependencies in request hot paths. (7) Automated remediation: runbooks for the top 10 failure scenarios. (8) Weekly game days for AZ failure simulation. (9) Immutable infrastructure: all changes via GitOps, no manual mutations. (10) SLO-based alerting catching degradation before it becomes downtime.

---

## Hands-On Labs

### Lab 1: AZ Failure Simulation
In a multi-AZ cluster, cordon all nodes in one AZ. Deploy a workload and verify it remains available on remaining AZs. Verify TSC prevents over-concentration.

### Lab 2: etcd Backup and Restore
Take a manual etcd snapshot. Create test resources. Delete them. Restore from snapshot. Verify deleted resources are recovered.

### Lab 3: PDB Testing
Create a Deployment with 3 replicas and PDB `maxUnavailable: 1`. Attempt `kubectl drain` on a node. Observe the drain respects the PDB.

---

## Production Incidents

### Incident 1: Single-AZ Node Group Caused Full Outage
All worker nodes were in one AZ (cost optimization gone wrong). The AZ experienced a cloud provider issue for 45 minutes. Complete service outage. **Prevention**: always use multi-AZ node groups, enforce with admission policy.

### Incident 2: DR Procedure Failed During Real Incident
The DR runbook hadn't been tested in 8 months. The S3 bucket for etcd backups had a misconfigured lifecycle policy that deleted backups after 7 days. The incident occurred 8 days after the last tested backup. **Prevention**: test DR monthly; monitor backup health as a metric; alert on backup age > 1 hour.
