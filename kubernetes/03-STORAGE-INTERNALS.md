# Storage Internals (5% of Interview Weight)

> CSI architecture, volume lifecycle, StorageClass patterns, snapshots

---

## 3.1 CSI (Container Storage Interface)

### CSI Driver Architecture

```
CSI decouples storage vendor logic from core Kubernetes (pre-CSI,
volume plugins were compiled INTO kubelet/kube-controller-manager —
"in-tree" — meaning a bug in an AWS EBS driver required a whole
Kubernetes release to fix. CSI drivers are external, out-of-tree).

CSI DRIVER COMPONENTS (typically deployed as):

1. Controller plugin (Deployment, 1+ replicas, leader-elected)
   Implements CreateVolume, DeleteVolume, ControllerPublishVolume
   (= cloud-level "attach" volume to a node), CreateSnapshot, etc.
   Talks to the cloud storage API (AWS EBS API, Azure Disk API, etc.)

2. Node plugin (DaemonSet, one per node)
   Implements NodeStageVolume (format + mount to a staging path),
   NodePublishVolume (bind-mount from staging to the pod's actual
   volume path). Talks to the LOCAL node's OS (mount, mkfs, etc.)

Sidecar containers (standard, provided by Kubernetes, wrap the
CSI driver's gRPC socket with Kubernetes API awareness):
├─ external-provisioner: watches PVCs, calls CreateVolume via CSI,
│  creates the bound PV object
├─ external-attacher: watches VolumeAttachment objects, calls
│  ControllerPublishVolume/Unpublish
├─ external-resizer: watches PVC size changes, calls
│  ControllerExpandVolume
├─ external-snapshotter: watches VolumeSnapshot objects, calls
│  CreateSnapshot/DeleteSnapshot
└─ node-driver-registrar: registers the node plugin with kubelet
   via the kubelet plugin registration mechanism (unix socket in
   /var/lib/kubelet/plugins_registry/)

Communication: kubelet ↔ CSI node plugin over a local UNIX domain
socket (gRPC) — NOT over the network, hence "node plugin" must run
on every node as a DaemonSet.
```

### Volume Lifecycle

```
1. USER creates PVC (PersistentVolumeClaim) requesting 10Gi, storageClass=fast-ssd
        │
        ▼
2. external-provisioner sidecar (watching PVCs) sees unbound PVC
   matching its StorageClass's provisioner name
        │
        ▼
3. Calls CSI CreateVolume RPC → cloud API creates a real disk
   (e.g. AWS EBS volume, Azure Managed Disk)
        │
        ▼
4. PV (PersistentVolume) object created, bound to the PVC
   (spec.csi.volumeHandle = cloud disk ID)
        │
        ▼
5. Pod referencing the PVC is scheduled (VolumeBinding scheduler
   plugin ensures pod lands on a node where the volume's topology
   constraints, e.g. availability zone, can be satisfied)
        │
        ▼
6. external-attacher sees a VolumeAttachment object needed → calls
   CSI ControllerPublishVolume → cloud API ATTACHES disk to the
   target NODE (e.g. `aws ec2 attach-volume`)
        │
        ▼
7. kubelet on that node calls CSI NodeStageVolume → mounts the raw
   block device to a staging directory, formats it if first use
   (mkfs.ext4/xfs, respecting the StorageClass's fsType)
        │
        ▼
8. kubelet calls CSI NodePublishVolume → bind-mounts from staging
   path into the actual pod's volume mount path (allows multiple
   pods, in ReadWriteMany cases, to share the same staged volume)
        │
        ▼
9. Container starts with the volume mounted and ready

DELETION reverses this: NodeUnpublish → NodeUnstage → ControllerUnpublish
(detach) → DeleteVolume (only if reclaimPolicy=Delete; Retain keeps
the underlying cloud disk even after PVC deletion, requiring manual
cleanup — used for critical data where accidental PVC deletion must
not destroy the actual disk).
```

### StorageClass Configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: disk.csi.azure.com     # or ebs.csi.aws.com, pd.csi.storage.gke.io
parameters:
  skuName: Premium_LRS
  cachingMode: ReadOnly
reclaimPolicy: Delete                # Delete | Retain
volumeBindingMode: WaitForFirstConsumer  # vs Immediate
allowVolumeExpansion: true
mountOptions:
  - noatime
```

```
volumeBindingMode explained:
├─ Immediate: PV provisioned as soon as PVC is created — RISK: if
│  the disk is created in AZ-1 but the pod later gets scheduled to
│  AZ-2 (scheduler doesn't know about the volume's zone yet), the
│  pod will be stuck Pending forever (volume not attachable cross-AZ)
└─ WaitForFirstConsumer (recommended default for zonal storage):
   PV provisioning is DELAYED until a pod actually claims the PVC
   and is scheduled — the scheduler picks a node FIRST (based on
   all other constraints), THEN the volume is provisioned in that
   node's exact zone, guaranteeing topology compatibility.
```

### Access Modes

```
ReadWriteOnce (RWO): mounted read-write by a SINGLE node
  (NOT single pod! Multiple pods on the SAME node CAN share an RWO
   volume — a common interview trick question)
ReadOnlyMany (ROX): mounted read-only by many nodes simultaneously
ReadWriteMany (RWX): mounted read-write by many nodes simultaneously
  (requires a network filesystem: NFS, Azure Files, EFS, CephFS —
   block storage like EBS/Azure Disk/GCP PD CANNOT do RWX)
ReadWriteOncePod (RWOP, newer, 1.22+): stricter than RWO — guarantees
  only a SINGLE POD (not just single node) can mount it, useful for
  strict data-corruption-sensitive workloads.
```

### Interview Questions — Storage

**Q1: PVC is stuck in Pending. Debug it.**
> `kubectl describe pvc <name>` — check Events for the actual provisioner error. Common causes: (1) no StorageClass exists matching the requested name, or no default StorageClass and PVC didn't specify one; (2) `WaitForFirstConsumer` binding mode — PVC will show Pending until a pod actually uses it, this is NORMAL and not a bug; (3) cloud quota exhausted (e.g. max EBS volumes per account/region hit); (4) requested access mode not supported by the provisioner (e.g. asking for RWX on an EBS-backed StorageClass); (5) check `external-provisioner` sidecar logs in the CSI controller pod for the actual API error from the cloud provider.

**Q2: Design storage for StatefulSet with 1000 replicas.**
> Each replica gets its own PVC via `volumeClaimTemplates` (1000 separate volumes — plan for cloud API rate limits on volume creation, this can genuinely hit them at this scale, might need to batch/stagger rollout). Use `WaitForFirstConsumer` to ensure zone-correct provisioning as pods spread across AZs. Consider whether truly 1000 independent volumes are needed vs. a shared distributed storage backend (e.g., if this is a database like Cassandra, each node's local disk is correct; if it's shared config, a single RWX volume might be more appropriate). Set resource quotas per namespace to prevent runaway volume creation from a bad rollout.

**Q3: Explain volume attachment and mount process.**
> Two distinct CSI phases: (1) "Attach" — controller-level, cloud API attaches the block device to a target NODE (VM-level operation, e.g. `attach-volume`) — coordinated by `external-attacher` watching `VolumeAttachment` objects; (2) "Mount" — node-level, kubelet's CSI node plugin runs `NodeStageVolume` (format if needed, mount to a staging dir) then `NodePublishVolume` (bind-mount staging dir into the pod's actual container path). Attach happens once per node; Publish can happen multiple times if multiple pods on that node share the volume (RWX/shared RWO-same-node case).

**Q4: Compare ReadWriteOnce vs ReadWriteMany access modes.**
> RWO is single-NODE (not single-pod — a subtlety many get wrong), works with any block storage (EBS, Azure Disk, GCP PD) — the vast majority of use cases (databases, single-writer apps). RWX is multi-node concurrent read-write, requires a genuine network filesystem (NFS, EFS, Azure Files, CephFS, Portworx) since block devices fundamentally can't be safely written from multiple hosts without a clustered filesystem layer on top. Use RWX only when truly needed (shared uploads directory, ML training data shared across workers) since it's typically slower and more complex than RWO.

**Q5: How do volume snapshots work in Kubernetes?**
> `VolumeSnapshotClass` (analogous to StorageClass) defines the snapshot provisioner. User creates a `VolumeSnapshot` object referencing a source PVC; `external-snapshotter` sidecar calls CSI `CreateSnapshot` RPC, which triggers the cloud API's native snapshot mechanism (e.g., EBS snapshot). A `VolumeSnapshotContent` object is created representing the actual snapshot. To restore, create a new PVC with `dataSource` pointing to the VolumeSnapshot — the CSI driver provisions a new volume pre-populated from the snapshot data.

**Q6: A pod using an EBS-backed PVC is stuck because the node it's rescheduled to already has a volume attached from a crashed pod on a different node. How do you debug/fix?**
> Classic "multi-attach error" — EBS/Azure Disk can only attach to ONE node at a time (unless using a shared/multi-attach SKU). If the original node crashed ungracefully, Kubernetes may not have detected the pod as terminated yet (node NotReady, but pod object not yet marked Failed), so the `VolumeAttachment` for the old node isn't cleaned up. Kubernetes' own multi-attach detection has a wait timeout (~6 minutes, `pod-eviction-timeout`) before force-detaching. To fix immediately: verify old node is truly dead, then manually delete the stale VolumeAttachment object (careful — data corruption risk if the old node ISN'T actually dead and still has the disk mounted).
