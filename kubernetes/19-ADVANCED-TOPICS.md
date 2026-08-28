# Section 19: Advanced Topics

This section covers the ecosystem tools and architectural concepts that appear most frequently in senior-level Kubernetes interviews: Operators, CRDs, admission webhooks, service meshes, GitOps, eBPF, multi-cluster, and AI/LLM workloads.

## Subtopic Index

- [Operators and CRDs](#operators-and-crds)
- [Admission Webhooks](#admission-webhooks)
- [Service Mesh Deep Dive](#service-mesh-deep-dive)
- [GitOps — ArgoCD and FluxCD](#gitops--argocd-and-fluxcd)
- [eBPF in Kubernetes](#ebpf-in-kubernetes)
- [Cilium](#cilium)
- [Multi-Cluster Federation](#multi-cluster-federation)
- [KEDA](#keda)
- [AI/LLM Workloads on Kubernetes](#aillm-workloads-on-kubernetes)

---

## Operators and CRDs

**Custom Resource Definition (CRD)**: extends the Kubernetes API with new resource types. The CRD schema (OpenAPI v3) validates objects at admission. Resources are stored in etcd. All standard Kubernetes features work: RBAC, watching, listing, labeling.

**Custom Controller (Operator)**: watches CRDs and reconciles cluster state. Implements domain-specific operational logic that would otherwise require human intervention.

The operator maturity model (Operator Framework):
1. **Basic Install**: automated installation/config.
2. **Seamless Upgrades**: patch and minor version upgrades.
3. **Full Lifecycle**: app lifecycle management (backup, failure recovery).
4. **Deep Insights**: metrics, alerts, log processing, workload analysis.
5. **Auto Pilot**: horizontal/vertical scaling, auto config tuning, anomaly detection.

**Kubebuilder / controller-runtime**: Go framework for building operators. Generates scaffolding, CRD manifests, RBAC, and webhook configuration. The reconciliation loop is built on informers and work queues.

Production considerations:
- Use `server-side apply` in controllers to avoid field ownership conflicts.
- Implement `Conditions` in status for observability.
- Use finalizers for cleanup on deletion.
- Implement conversion webhooks for CRD version migration.

---

## Admission Webhooks

Admission webhooks intercept all API requests after auth but before persistence. They're the most powerful Kubernetes extension point, used for policy enforcement, mutation, and validation.

**MutatingAdmissionWebhook**: can modify objects. Runs before validation. Use cases: inject sidecars, set defaults, normalize labels, pin image tags to digests.

**ValidatingAdmissionWebhook**: can only accept/deny. Runs after mutation. Use cases: policy enforcement (require labels, limit image registries, block privileged containers).

**ValidatingAdmissionPolicy (GA 1.30)**: CEL-based validation in-process. No external webhook needed for simple rules. Zero availability dependency.

```yaml
# Webhook fails open vs fails closed
webhooks:
- name: sidecar-injector.example.com
  failurePolicy: Fail        # fails closed: block if webhook unavailable
  # vs
  failurePolicy: Ignore      # fails open: allow if webhook unavailable

  # Narrow scope to reduce blast radius
  namespaceSelector:
    matchLabels:
      inject-sidecar: "true"
  objectSelector:
    matchExpressions:
    - key: skip-injection
      operator: DoesNotExist

  timeoutSeconds: 5           # fail fast; slow webhook = cluster lag
  sideEffects: None           # required for dry-run support
```

Common pitfalls:
- **Broad rules + failurePolicy: Fail** = single webhook failure takes down the cluster.
- **Slow webhook** = every API call is slow (apiserver waits for webhook response).
- **TLS cert expiration** = all matching API calls fail immediately.
- **Not idempotent** = reinvocation policy causes double mutations.

---

## Service Mesh Deep Dive

Service meshes solve the "too much networking code in every microservice" problem. They move mTLS, load balancing, retries, circuit breaking, and observability from application code into sidecar proxies.

**Istio architecture** (covered in Section 10). Key advanced concepts:

**Traffic shaping**: VirtualService allows sophisticated routing:
```yaml
spec:
  http:
  - match:
    - headers:
        x-user-group:
          exact: beta-testers
    route:
    - destination:
        host: payments
        subset: v2
      weight: 100
  - route:
    - destination:
        host: payments
        subset: v1
      weight: 90
    - destination:
        host: payments
        subset: v2
      weight: 10
```

**Circuit breaking** (DestinationRule):
```yaml
trafficPolicy:
  outlierDetection:
    consecutive5xxErrors: 5
    interval: 10s
    baseEjectionTime: 30s
    maxEjectionPercent: 50
```
After 5 consecutive 5xx errors in 10s, the endpoint is ejected from the load-balancing pool for 30s.

**Ambient mesh** eliminates sidecars by using a per-node ztunnel for L4 and a per-namespace waypoint proxy for L7. Reduces resource usage by ~60% (no per-pod sidecar containers).

**Linkerd** is lighter-weight than Istio: uses a Rust-based micro-proxy (linkerd2-proxy), ~3ms latency overhead vs Istio's ~5ms. No xDS complexity; simpler operations. Less feature-rich (no advanced traffic management).

---

## GitOps — ArgoCD and FluxCD

GitOps uses Git as the single source of truth for desired state. A GitOps agent (ArgoCD, Flux) continuously reconciles the cluster to match the Git repository state.

**Pull-based model**: the cluster agent pulls from Git and applies changes. No inbound network access from CI to cluster — significantly reduces attack surface.

**ArgoCD**: UI-centric, strong RBAC, ApplicationSet for fleet management, sync waves for ordering, Argo Rollouts integration for progressive delivery.

**FluxCD**: CLI-focused, composable controllers (source-controller, kustomize-controller, helm-controller), image automation (auto-commit new image tags), multi-tenancy via Kustomization with service account impersonation.

**Key patterns**:
- **App of Apps**: one ArgoCD Application manages a set of child Applications. Useful for fleet bootstrapping.
- **ApplicationSet**: generates multiple ArgoCD Applications from a template + generator (Git directory, cluster list, JSON data).
- **Sync waves**: `argocd.argoproj.io/sync-wave: "-1"` runs before wave 0; used to apply CRDs before the workloads that use them.
- **SSO + RBAC**: ArgoCD integrates with OIDC for user authentication; RBAC controls which users can sync which apps.

**Secrets in GitOps**: Never commit plaintext secrets to Git. Options: Sealed Secrets (encrypt before commit, cluster has the private key), SOPS (age/KMS encrypted files), External Secrets (pull from Vault at runtime, not committed).

---

## eBPF in Kubernetes

eBPF (extended Berkeley Packet Filter) allows safe custom programs to run in the Linux kernel without kernel modules. In Kubernetes, eBPF enables:

1. **Observability**: Hubble (Cilium) captures every L3/L4/L7 packet with full flow metadata, no sidecar required.

2. **Security**: Tetragon and Falco use kprobes to detect malicious syscalls in real time. Tetragon can `SIGKILL` the process in-kernel before the syscall completes.

3. **Networking**: Cilium replaces iptables with BPF maps for O(1) Service lookups and XDP for early-drop DDoS mitigation.

4. **Profiling**: Parca (continuous profiling) uses eBPF perf events to sample CPU usage at the kernel level without modifying applications.

Key eBPF concepts: BPF verifier (safety guarantees), BPF map types (hash, array, ring buffer), BPF helper functions, attachment points (kprobe, tracepoint, TC, XDP), JIT compilation.

---

## Cilium

Cilium is the leading eBPF-based Kubernetes network and security platform. Key features:

- **kube-proxy replacement**: BPF maps for Service load balancing (O(1) vs O(n) iptables).
- **Identity-based NetworkPolicy**: policy based on pod labels/identity, not IPs. Policies survive pod restarts without rule updates.
- **L7 policy**: HTTP path, method, gRPC service — not just L3/L4.
- **Hubble**: distributed observability — flow logs, DNS queries, HTTP status codes — all without application changes.
- **Transparent encryption**: WireGuard or IPsec for all pod-to-pod traffic, managed by Cilium.
- **Egress gateway**: assign stable IPs to groups of pods for external services that require IP whitelisting.

```bash
# Check Cilium status
cilium status
cilium endpoint list

# Watch network flows
hubble observe --follow
hubble observe --namespace production --verdict DROPPED

# L7 Policy example
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-get-only
spec:
  endpointSelector:
    matchLabels: {app: payments}
  ingress:
  - fromEndpoints:
    - matchLabels: {app: frontend}
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: GET      # only allow GET, not POST/DELETE
          path: /api/.*
```

---

## Multi-Cluster Federation

Multi-cluster architectures run multiple independent Kubernetes clusters, each with its own control plane, but coordinated by a management layer.

**Patterns**:
- **Replicated**: same workloads in multiple clusters for HA and latency. Each cluster is independent.
- **Segmented**: different workloads in different clusters for isolation (compliance, blast radius). A payments cluster, a data science cluster, etc.
- **Federated**: a meta-control plane manages multiple clusters as a single logical cluster (Cluster API, vCluster fleet management).

**Multi-cluster tooling**:
- **Cluster API (CAPI)**: Kubernetes-native cluster lifecycle management (provision, upgrade, scale worker nodes). Provider plugins for AWS, Azure, GCP, vSphere.
- **ArgoCD ApplicationSet**: deploy applications to many clusters from one ArgoCD instance.
- **Flux multi-cluster**: each cluster has its own Flux installation; a "hub" cluster manages "tenant" cluster configurations.
- **vCluster**: virtual clusters that share a host cluster's nodes/network/storage but have isolated control planes. Cheap multi-tenancy or per-team clusters.

**Cross-cluster service discovery**: options include DNS (global DNS that resolves to the nearest cluster's ClusterIP), service mesh federation (Istio multi-cluster with shared root CA), or a gateway pattern (route cross-cluster traffic through a central gateway).

---

## KEDA

KEDA (Kubernetes Event-Driven Autoscaling) extends Kubernetes HPA to scale based on external events. Covered in detail in Section 15. Key additional concepts:

**ScaledJob** for batch: scales Jobs based on queue depth, launching one job per N queue messages, scaling down when queue is empty.

**KEDA external scaler**: write a custom gRPC scaler to expose any metric as an external scaling trigger. Useful for proprietary monitoring systems.

**KEDA + Spot**: KEDA scales out replicas based on events; Karpenter provisions Spot nodes. Combined: cost-optimized event-driven scaling where both the application layer and compute layer respond to demand.

---

## AI/LLM Workloads on Kubernetes

Running AI/ML and LLM inference on Kubernetes presents unique challenges: GPU scheduling, model storage, inference server autoscaling.

**GPU scheduling**: use `resources.limits: {nvidia.com/gpu: "1"}` with the NVIDIA Device Plugin DaemonSet. The device plugin advertises GPU capacity to the kubelet. Kubernetes schedules pods to nodes with available GPUs. For fractional GPU (multiple inference pods per GPU), use NVIDIA MPS or time-slicing.

**Karpenter for GPU nodes**: on-demand vs Spot GPU instances. Spot GPU prices are 70–80% lower but have higher interruption rates. For batch training: Spot with checkpointing. For real-time inference: On-Demand.

**Model storage**: models (10–100GB) need fast, shared access. Options:
- PVC backed by EFS/Azure Files (shared NFS, slow but simple)
- Init container pulls model to node-local storage (`emptyDir: {medium: ""}`)
- Model registry → object store → sidecar pulls on startup
- Custom CSI driver that mounts model as a read-only volume from object store

**Inference autoscaling**: standard CPU/memory HPA doesn't work well for LLMs — the bottleneck is GPU memory and token throughput. Use KEDA with custom metrics: `gpu_memory_used_fraction` or `inference_queue_depth`. Scale from 0 with KEDA when no requests arrive (expensive idle GPU = wasted cost).

**vLLM on Kubernetes**: deploy vLLM (OpenAI-compatible inference server) as a Deployment with GPU affinity:
```yaml
resources:
  limits:
    nvidia.com/gpu: "1"
    memory: 32Gi
  requests:
    nvidia.com/gpu: "1"
    memory: 32Gi
env:
- name: MODEL_PATH
  value: /models/llama-3-8b
volumeMounts:
- name: models
  mountPath: /models
  readOnly: true
```

---

## Interview Questions

### Conceptual / Internals (8 questions)

**1. What is the difference between Cluster API and vanilla kubeadm for provisioning clusters?**
kubeadm is a node-level tool — it configures a single node to be part of a Kubernetes cluster (generates certs, installs static pods, etc.). It doesn't manage cloud infrastructure. Cluster API (CAPI) is a Kubernetes-native management layer for cluster lifecycle. You create `Cluster`, `MachineDeployment`, and provider-specific `AWSMachineTemplate` objects in a management cluster. CAPI controllers call cloud APIs to provision VMs, then use kubeadm (or another bootstrapper) to configure them into a cluster. CAPI provides declarative, GitOps-compatible cluster lifecycle management with support for upgrades, scaling, and multi-cloud.

**2. Why is GitOps more secure than push-based CI/CD for Kubernetes deployments?**
Push-based: the CI server needs network access and credentials (KUBECONFIG or service account token) to call the Kubernetes API. If the CI server is compromised, the attacker has cluster access. Pull-based (GitOps): the cluster agent has Git read access and cluster write access, but no external entity calls into the cluster. The attack surface is: (1) the Git repository (protected by OIDC + branch protection), (2) the GitOps agent itself (runs in-cluster, minimal RBAC). A compromised CI server can't deploy to the cluster — it can only commit to Git. An adversary must compromise both Git AND the cluster to achieve deployment.

**3. How does Cilium's identity-based NetworkPolicy differ from Kubernetes standard NetworkPolicy?**
Kubernetes NetworkPolicy: enforced by the CNI, policy rules match on IP addresses and port numbers. When a pod restarts with a new IP, all policy rules must be updated. IP-based policy creates churn. Cilium NetworkPolicy: rules match on security identity — a numeric value derived from the pod's label set. When a pod restarts with the same labels, its identity is unchanged — policy rules need no update. Additionally, Cilium supports L7 rules (HTTP method/path, gRPC service) which are impossible with standard NetworkPolicy. Cilium's Hubble uses identities to annotate flow logs with service names, making them human-readable.

**4. Explain vCluster and when you'd use it instead of a real cluster.**
vCluster creates a lightweight virtual Kubernetes cluster running inside a namespace of a host (real) cluster. The virtual cluster has its own apiserver and etcd (lightweight k3s/k0s) but shares the host cluster's nodes, networking, and storage. Pods scheduled in the vCluster appear in the host cluster's namespace with mangled names. Use cases: (1) **Multi-tenancy**: give teams isolated cluster-level access (CRDs, RBAC, cluster-admin) without the cost of separate real clusters. (2) **CI**: spin up ephemeral test clusters per PR, deleted after testing. (3) **Development**: developers get full cluster-admin in their vCluster without affecting others. Trade-offs: vCluster adds API call overhead (host-cluster apiserver + vCluster apiserver), and some features (LoadBalancer Services, Node labels) require host-cluster integration.

**5. How does ArgoCD ApplicationSet enable fleet management?**
ApplicationSet uses generators to create many ArgoCD Application objects from a template. The Git directory generator creates one Application per directory in a Git repo — useful for environment directories (staging/production/dr). The cluster generator creates one Application per registered ArgoCD cluster — useful for deploying an add-on to every cluster. The matrix generator combines two generators — e.g., cluster × environment. ApplicationSet eliminates the N-copy-paste problem: you define one ApplicationSet, and ArgoCD creates and manages N Application objects automatically. When you add a new cluster to ArgoCD, the cluster generator automatically creates the Application for the new cluster.

**6. How would you implement zero-downtime LLM model updates (replacing a 70B parameter model)?**
Key challenges: (1) Model is large (140GB for 70B at float16) — can't download during prod traffic. (2) GPU memory is the bottleneck — can't run two models simultaneously on same node (unless using larger GPU or model parallelism). Strategy: (1) Pre-stage the new model: init container or a separate pre-loading job downloads the new model to a node-local path before the rollout begins. (2) Use a separate node pool for the new model: Karpenter launches new GPU nodes with the new model pre-loaded, while old nodes continue serving. (3) Switch traffic: HTTPRoute weights route 5% to new, validate quality (perplexity, latency), then 100%. (4) Terminate old nodes: after traffic shift, drain old nodes and let Karpenter terminate them.

**7. Explain how eBPF's verifier ensures safety of kernel BPF programs.**
The BPF verifier performs static analysis on the BPF bytecode before loading it into the kernel. It: (1) Traces all possible execution paths through the program. (2) Ensures every path terminates (no loops that can run forever — bounded loops only). (3) Verifies that every memory access is within bounds (no buffer overflows, no NULL pointer dereferences). (4) Checks that registers are initialized before use. (5) Enforces type safety on helper function arguments. The verification is a constraint-solving problem; for complex programs it can be computationally expensive and may reject valid programs that the verifier can't prove safe. The verifier makes BPF programs safe to run in kernel context without kernel crashes.

**8. How does Istio ambient mesh eliminate sidecars while preserving mTLS?**
Ambient mesh uses two components: (1) **ztunnel** (per-node): handles L4 traffic. A Rust-based minimal proxy that handles TCP tunnel establishment and mTLS between ztunnels on different nodes. Each pod's traffic is redirected to the node's ztunnel via iptables rules (similar to sidecar, but at the host namespace level). (2) **waypoint proxy** (per-namespace or per-service): an Envoy-based proxy that handles L7 — HTTP routing, retries, circuit breaking. Only deployed when L7 features are needed. Without waypoints, only L4/mTLS is available (ztunnel only). The key difference from sidecar: no per-pod memory overhead for Envoy (ztunnel is ~100KB vs Envoy's 50-200MB). Mutual TLS still occurs — ztunnel-to-ztunnel connections use HBONE (HTTP-based overlay tunneling) over mTLS.

### Scenario Questions (6 questions)

**9. Your team needs to build a platform where developers can request databases as code. Design the operator.**
Architecture: `DatabaseInstance` CRD. Controller with reconcile loop: (1) Creates a Secret with credentials (using random password generator). (2) Creates a StatefulSet with the appropriate DB image and PVC. (3) Creates a Headless Service for stable DNS. (4) Creates a ClusterIP Service for client access. (5) Updates `DatabaseInstance.status.connectionString`. Lifecycle: deletion triggers cleanup (PVC protected by finalizer, admin decision whether to delete PVC). Backup: watches for `DatabaseBackup` CRDs that trigger backup Jobs. The developer creates a `DatabaseInstance` YAML in their app's Git repo; the GitOps agent applies it; the operator provisions the database. No human operator intervention.

**10. ArgoCD is showing an application as OutOfSync even though you just synced. Debug.**
Causes: (1) Helm rendering is non-deterministic (templating with random secrets, timestamps) — each render produces different output. Fix: use `--set` with explicit values. (2) A controller is mutating the object after sync (e.g., adding labels/annotations, setting defaults). ArgoCD diffs the desired state (from Git) vs live state (from cluster). Fix: configure `ignoreDifferences` for those fields. (3) Server-side defaulting: the apiserver adds default values that aren't in the manifest. Fix: use `managedNamespaceMetadata` or ignore the defaulted fields. (4) Pruning not enabled: orphaned resources not managed by ArgoCD are counted as OutOfSync. Enable prune.

### FAANG Deep Dive (6 questions)

**11. How would you design a multi-tenant LLM inference platform where each tenant gets isolated resources but shares the underlying GPU infrastructure?**
Architecture: (1) **vCluster per tenant**: each tenant has their own virtual Kubernetes cluster (isolated API, RBAC, namespaces). Models and inference Deployments live in the vCluster. (2) **Shared GPU node pool**: host cluster has GPU nodes with Karpenter. vCluster pods appear in the host cluster's namespace; host cluster schedules them to GPU nodes. (3) **Resource quotas**: host cluster namespace-level ResourceQuota limits total GPU usage per tenant. (4) **Network isolation**: Cilium NetworkPolicy in host cluster prevents cross-tenant pod communication. (5) **Model caching**: shared read-only PVC (EFS) for common models; tenant-specific PVCs for custom models. (6) **Cost attribution**: labels on all resources for showback/chargeback via Kubecost.

**12. Describe how you would build a safe, self-service cluster provisioning system using Cluster API.**
Management cluster (separate from tenant clusters): (1) Install Cluster API + provider (CAPA for AWS). (2) Create `ClusterTemplate` ConfigMaps defining approved cluster configurations. (3) Build a self-service API (a CRD `ClusterRequest`) that developers submit. (4) A controller validates requests (team quota, allowed regions, approved instance types), creates `Cluster`+`MachineDeployment` objects from templates. (5) ArgoCD ApplicationSet deploys standard add-ons (ingress, monitoring, Falco) to each new cluster. (6) Cluster lifecycle events (upgrade, scale, delete) require PR approval via policy-as-code. (7) Cluster RBAC is bootstrapped via ArgoCD: team namespace admin is configured automatically. The developer experience: submit a PR with a `ClusterRequest` manifest; PR approval triggers provisioning; cluster is ready in 15 minutes.

---

## Hands-On Labs

### Lab 1: ArgoCD GitOps Setup
Install ArgoCD. Create a Git repo with a simple app manifest. Configure ArgoCD Application pointing to the repo. Make a change to the manifest, observe ArgoCD detecting drift and syncing.

### Lab 2: Cilium L7 NetworkPolicy
Install Cilium in a kind cluster. Deploy two services. Apply a CiliumNetworkPolicy allowing only GET requests. Verify POST is blocked.

### Lab 3: Operator with Kubebuilder
Use Kubebuilder to scaffold a controller. Implement a `ConfigSync` CRD that syncs ConfigMaps from a "source" namespace to target namespaces. Test with a simulated namespace creation.
