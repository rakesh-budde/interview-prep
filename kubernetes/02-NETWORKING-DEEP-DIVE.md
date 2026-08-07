# Networking Deep Dive (20% of Interview Weight)

> CNI internals, kube-proxy modes, Service types, NetworkPolicy, CoreDNS — all at packet-level depth

---

## Table of Contents
- [2.1 CNI (Container Network Interface)](#21-cni-container-network-interface)
- [2.2 Service Networking](#22-service-networking)
- [2.3 Network Policies](#23-network-policies)
- [2.4 DNS (CoreDNS)](#24-dns-coredns)

---

## 2.1 CNI (Container Network Interface)

### CNI Specification & Plugin Lifecycle

```
CNI is a SPEC, not a specific tool: JSON config in + JSON result out,
plugin binary invoked by the container runtime via CRI → CNI shim.

POD CREATION NETWORK SETUP FLOW:

1. kubelet asks CRI (containerd) to create pod sandbox
2. containerd creates network namespace (netns) for the pod
3. containerd invokes CNI plugin binary with command=ADD, passing:
   ├─ CNI_NETNS: path to the new network namespace
   ├─ CNI_IFNAME: e.g. "eth0"
   ├─ CNI_ARGS: K8S_POD_NAME, K8S_POD_NAMESPACE, etc.
   └─ stdin: JSON config from /etc/cni/net.d/10-calico.conflist

4. CNI plugin (e.g. calico-ipam):
   a. Creates veth pair (one end in netns as eth0, other end on host)
   b. Requests IP from IPAM plugin (calico-ipam, host-local, etc.)
   c. Assigns IP to the pod-side veth interface
   d. Sets up routes (pod default route → host veth)
   e. Configures host-side routing/iptables/BGP so packets reach the pod
   f. Returns JSON result: {"ips": [...], "routes": [...], "dns": [...]}

5. Command=DEL invoked on pod deletion: reverses everything (removes
   veth, releases IP back to IPAM pool)

6. Command=CHECK (newer): verifies existing setup is still correct
   (used by kubelet periodically to detect drift)

CNI CHAINING: Multiple plugins can run in sequence via a "conflist"
(e.g., main CNI for IP assignment + "portmap" plugin for hostPort
support + "bandwidth" plugin for traffic shaping) — each plugin in
the chain gets the previous plugin's result as additional input.
```

### IPAM (IP Address Management)

```
IPAM MODELS:

1. host-local: each node gets a slice of the overall Pod CIDR
   (e.g., cluster CIDR 10.244.0.0/16, each node gets a /24 =
   10.244.5.0/24) — simple, but wastes IPs if pods-per-node is low
   relative to the /24 allocation.

2. Cloud-native (AWS VPC CNI, Azure CNI): pods get REAL IPs from the
   VPC/VNet address space directly (no overlay/NAT) — better
   performance, but consumes VPC IP space fast; often needs secondary
   ENIs/NICs attached to nodes to get enough IPs per node.

3. Calico IPAM: uses IP pools with configurable block sizes,
   supports cross-subnet routing via BGP, more flexible allocation
   than static /24-per-node.

Example calculation for Azure CNI:
Standard_D4s_v3 supports 8 IP configurations per NIC (across 2 NICs)
→ max ~30 pods/node (Azure CNI classic) unless using CNI Overlay or
"Azure CNI Powered by Cilium" — which decouples node IP capacity from
pod scaling limits.
```

### CNI Plugin Comparison

```
┌──────────────────────────────────────────────────────────────────┐
│               CNI PLUGIN DEEP COMPARISON                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                     │
│ FLANNEL (VXLAN overlay - simplest)                                 │
│ ├─ Encapsulates pod traffic in VXLAN (UDP port 8472)                │
│ ├─ Every packet: original IP packet wrapped in outer UDP/IP          │
│ ├─ Overhead: ~50 bytes/packet, MTU must account for encapsulation   │
│ │  (typically 1450 instead of 1500)                                  │
│ ├─ No NetworkPolicy support natively                                │
│ └─ Use: simple/dev clusters, when you don't need policies            │
│                                                                     │
│ CALICO (BGP routing OR VXLAN, + iptables/eBPF dataplane)            │
│ ├─ Default mode: pure L3 routing via BGP (no encapsulation, native   │
│ │  routing) when nodes are on the same L2 network                     │
│ ├─ IPIP/VXLAN mode when crossing L3 boundaries (e.g. across subnets) │
│ ├─ Felix agent programs iptables/ipsets (or eBPF in newer versions)   │
│ │  to enforce NetworkPolicy                                           │
│ ├─ BGP peering can integrate with physical network routers (on-prem) │
│ ├─ calico-node DaemonSet: one per node, runs Felix + BIRD (BGP)       │
│ └─ Use: production default choice, strong policy engine, on-prem      │
│                                                                     │
│ CILIUM (eBPF-based - modern, highest performance)                   │
│ ├─ Replaces iptables/kube-proxy entirely with eBPF programs           │
│ │  attached to kernel hooks (no per-packet userspace traversal)       │
│ ├─ L3-L7 policies (can filter by HTTP method/path, gRPC method,       │
│ │  Kafka topic — not just IP/port like Calico/NetworkPolicy std)      │
│ ├─ Hubble: built-in observability (flow logs, service map) via eBPF   │
│ ├─ Direct routing or VXLAN/Geneve overlay modes                       │
│ ├─ WireGuard/IPsec transparent encryption between nodes                │
│ └─ Use: performance-critical, deep observability needs, modern stacks │
│                                                                     │
│ AWS VPC CNI / AZURE CNI (cloud-native)                               │
│ ├─ Pods get real VPC/VNet IPs (no overlay) — best raw performance,    │
│ │  can be reached directly by other VPC resources                     │
│ ├─ Limited by ENI/NIC IP capacity per instance type                  │
│ └─ Use: when direct VPC integration needed (e.g. security groups     │
│    applied per-pod, PrivateLink/PrivateEndpoint targeting pods)       │
└──────────────────────────────────────────────────────────────────┘
```

### eBPF vs iptables — Why It Matters

```
IPTABLES-BASED DATAPLANE PROBLEM AT SCALE:
├─ Every Service = set of iptables rules (SNAT/DNAT chains)
├─ Rules evaluated SEQUENTIALLY (linear scan) for each packet
├─ 10,000 Services → potentially 40,000+ iptables rules
├─ Packet matching becomes O(n) — real measured latency increase
│  from single-digit microseconds to milliseconds at high rule counts
└─ Updates (Service add/remove) require re-writing large rule sets
   atomically (iptables-restore) — can cause brief packet loss/CPU spikes

EBPF-BASED DATAPLANE (Cilium) SOLUTION:
├─ Uses HASH MAPS in kernel (BPF maps) for O(1) lookup regardless of
│  Service count
├─ Programs attached at multiple hook points: XDP (earliest, before
│  the network stack even builds sk_buff — fastest), tc (traffic
│  control ingress/egress), socket layer (for pod-local optimization)
├─ Socket-level load balancing: for pod→ClusterIP traffic, Cilium can
│  rewrite the destination AT CONNECT() TIME (socket layer), so the
│  packet never even needs Service NAT — connects directly pod-to-pod
├─ Massively reduces per-packet overhead at scale
└─ This is why "kube-proxy replacement" mode in Cilium is recommended
   for large/high-throughput clusters
```

### Interview Questions — CNI

**Q1: Compare Calico vs Cilium — when would you choose each?**
> Calico: mature, BGP-based routing (great for on-prem integration with physical routers), strong standard NetworkPolicy support, iptables or eBPF dataplane option. Cilium: eBPF-native from the ground up, L3-L7 policy (HTTP/gRPC/Kafka aware), built-in Hubble observability, best raw performance at scale (10k+ services), WireGuard encryption. Choose Cilium for greenfield cloud-native at scale needing deep observability; choose Calico for on-prem/hybrid needing BGP integration or where team familiarity/maturity matters more.

**Q2: Pod can't reach another pod. Walk through network debugging.**
> `kubectl exec` into source pod, `ping`/`curl` destination pod IP directly (bypass Service). If that fails: check CNI plugin health (`kubectl get pods -n kube-system -l k8s-app=calico-node`), check `ip route` on both nodes for pod CIDR routes, check `iptables -L -n -v` for DROP rules (NetworkPolicy enforcement), check MTU mismatch (common with overlay networks — symptoms are small packets work but large ones hang, classic sign of fragmentation issues), verify security groups/NSGs at cloud level aren't blocking inter-node traffic.

**Q3: Design CNI for 10,000 node cluster with multiple availability zones.**
> Prefer Cilium in native routing mode (no overlay overhead) if nodes share L3 reachability, or BGP-based Calico peering with top-of-rack routers per AZ. Use IP pools scoped per-AZ/subnet to keep routing tables manageable and enable topology-aware routing (avoid cross-AZ traffic costs/latency — combine with Service `internalTrafficPolicy: Local` or topology-aware hints). Enable eBPF dataplane for kube-proxy replacement to avoid iptables scaling issues at this size. Plan IP address space carefully upfront — 10,000 nodes × ~110 pods/node = 1.1M+ addresses needed.

**Q4: Explain how eBPF improves Kubernetes networking.**
> Replaces sequential iptables rule matching with O(1) BPF map lookups attached directly to kernel hooks (XDP/tc/socket layer), eliminating per-packet linear scans that degrade with Service count. Also enables socket-level load balancing (rewriting destination at connect() time rather than per-packet NAT) and rich L7-aware policy/observability without a sidecar proxy.

**Q5: What is CNI chaining and give a real example?**
> Multiple CNI plugins execute in sequence on the same pod, each contributing to or modifying the network setup. Example: Calico (primary — IP assignment, routing, NetworkPolicy) chained with `portmap` (implements hostPort mapping, since main CNI plugins often don't) chained with `bandwidth` (traffic shaping via tbf qdisc). Each subsequent plugin receives the prior plugin's CNI result as part of its own input.

**Q6: Why does MTU matter for overlay networks, and how do you debug MTU issues?**
> VXLAN/IPIP encapsulation adds header overhead (~50 bytes for VXLAN), so the pod interface MTU must be set LOWER than the host's MTU (e.g., 1450 vs 1500) to avoid fragmentation. Symptom: small packets (pings, DNS) work fine, but larger payloads (HTTP responses, TLS handshakes) hang or reset — classic "works for ping, fails for real traffic" bug. Debug with `ping -M do -s 1472 <target>` (DF flag, various sizes) to find the actual working MTU, then fix CNI config's `mtu` field.

---

## 2.2 Service Networking

### kube-proxy Modes

```
IPTABLES MODE (historical default):

Client → ClusterIP:Port
        │
        ▼ (PREROUTING chain, KUBE-SERVICES)
┌──────────────────────────────────────────────┐
│ iptables DNAT rule (probabilistic, for each    │
│ backend pod, using "statistic" match module):   │
│                                                  │
│ -m statistic --mode random --probability 0.33  │
│   → DNAT to pod1:port                            │
│ -m statistic --mode random --probability 0.50  │
│   → DNAT to pod2:port  (of remaining 67%)        │
│ (else) → DNAT to pod3:port                       │
└──────────────────────────────────────────────┘
        │
        ▼
Connection tracking (conntrack) remembers this flow →
subsequent packets in the SAME connection go to the SAME pod
(no per-packet re-randomization, only per NEW connection)

Downsides: O(n) rule evaluation, "random" isn't true load balancing
(no least-connections), rule updates require full iptables-restore.

IPVS MODE (better at scale):

Uses Linux IPVS (IP Virtual Server, same tech behind many L4 load
balancers) — kernel hash table lookup, O(1) regardless of Service
count. Supports real load balancing algorithms: round-robin (rr),
least connection (lc), destination hashing (dh), source hashing (sh).
Requires ipvsadm kernel modules loaded. Recommended for clusters
with 1,000+ Services where iptables mode shows CPU/latency issues.

    ipvsadm -Ln                    # list IPVS virtual servers
    ipvsadm -Ln --stats            # traffic stats

EBPF MODE (Cilium kube-proxy replacement):

No iptables/IPVS at all. eBPF programs at socket layer + tc hooks
directly implement Service load balancing. Socket-level LB means
client connecting to a ClusterIP can have the destination rewritten
at connect() time — the packet is built with the REAL pod IP from
the start, never touching a separate NAT layer. Best performance,
best observability (Hubble shows actual Service-to-pod flow mapping).
```

### Complete Packet Flow: Client → ClusterIP → Pod

```
1. Client pod does: curl http://my-service.default.svc.cluster.local
2. CoreDNS resolves my-service → ClusterIP (e.g. 10.96.0.55)
3. Client's TCP SYN sent to 10.96.0.55:80
4. Packet leaves client's veth, enters host network namespace
5. netfilter PREROUTING hook: KUBE-SERVICES chain matches
   destination 10.96.0.55:80 → jumps to KUBE-SVC-XXXX chain
6. KUBE-SVC-XXXX has one KUBE-SEP-YYYY rule per healthy endpoint
   (pod IP:port), picked via statistic/random match
7. DNAT applied: destination rewritten from 10.96.0.55:80 to
   actual pod IP e.g. 10.244.2.15:8080
8. conntrack table records this NAT mapping (for return traffic +
   session stickiness for subsequent packets)
9. Packet routed (via CNI's routing setup — could be direct route,
   BGP-advertised, or VXLAN-encapsulated) to the node hosting that pod
10. On destination node: packet delivered to pod's veth → pod's eth0
11. Pod's application receives connection, appearing to originate
    from... the ORIGINAL client IP unless SNAT was needed (SNAT
    happens if client and backend are the same pod - hairpin - or
    if externalTrafficPolicy requires it)
12. Response follows reverse path, conntrack un-DNATs automatically
```

### Service Types

```
ClusterIP (default): virtual IP, only reachable within cluster
NodePort: ClusterIP + opens a port (30000-32767) on EVERY node;
  external traffic to <any-node-ip>:<nodePort> gets routed (via
  iptables) to a pod, possibly on a DIFFERENT node (extra hop,
  unless externalTrafficPolicy: Local)
LoadBalancer: NodePort + cloud provider provisions an external LB
  (ELB/ALB, Azure LB, GCP LB) pointing at node IPs:nodePort
ExternalName: pure DNS CNAME redirect, no proxying, no selector
Headless (clusterIP: None): no VIP; DNS returns ALL pod IPs directly
  (A records for each ready pod) — used by StatefulSets for direct
  pod addressing

externalTrafficPolicy: Cluster (default) vs Local
├─ Cluster: any node can route to any pod (extra hop possible),
│  but source IP is SNAT'd (backend sees node IP, not real client IP)
└─ Local: only routes to pods on the SAME node that received traffic
   (no extra hop, preserves source IP) — BUT if that node has no
   healthy pod, traffic is dropped (no failover to other nodes) —
   requires even pod distribution across nodes to avoid imbalance
```

### Interview Questions — Service Networking

**Q1: Explain the complete packet flow from client to pod via ClusterIP.**
> See diagram above — DNS resolves to VIP, netfilter PREROUTING DNATs to a specific endpoint pod IP based on iptables statistic match (or IPVS hash / eBPF socket rewrite), conntrack tracks the flow for consistent routing and automatic un-NAT on the return path, packet is routed to the destination node via the CNI's routing mechanism.

**Q2: Service returns 5xx intermittently. Debug it.**
> Check `kubectl get endpoints <svc>` — are ALL expected pods listed as ready endpoints, or is a subset flapping (readinessProbe failing intermittently under load)? Check for a single bad pod skewing error rate (uneven load balancing can mean the same "unlucky" ~1/3 of requests always hit the bad pod). Check conntrack table exhaustion (`conntrack -L | wc -l` vs `nf_conntrack_max`) — a full conntrack table silently drops new connections. Check `kube-proxy` logs for repeated iptables-restore failures.

**Q3: Design service mesh for 500 microservices.**
> Istio/Linkerd sidecar injection for mTLS between all services (zero-trust), centralized traffic management (retries, circuit breaking, canary via VirtualService/TrafficSplit), and rich L7 observability (per-route latency, success rate) without app code changes. Discuss sidecar resource overhead at 500-service scale (consider Cilium's sidecar-less "ambient mesh" mode as an alternative to reduce per-pod proxy overhead) and mTLS certificate rotation via the mesh's built-in CA (e.g. Istiod).

**Q4: Compare kube-proxy iptables vs IPVS for 10,000 services.**
> iptables: O(n) linear rule matching per packet — with 10k Services (each with a chain of rules per endpoint), packet processing latency increases materially, and every Service change requires a full atomic iptables-restore of a huge ruleset (CPU spike, brief risk). IPVS: O(1) hash table lookup regardless of Service count, supports real load-balancing algorithms (least-connection, etc.), incremental updates without full ruleset rewrite — clearly the better choice at this scale (or better yet, Cilium eBPF mode).

**Q5: Why does `externalTrafficPolicy: Local` sometimes cause uneven load?**
> It forces traffic to stay on the node that received it (no extra hop, preserves source IP), but if pod replicas aren't evenly spread across nodes, nodes with more replicas get proportionally more traffic per pod is fine, but nodes with a pod that's NOT ready get zero traffic routed there even if the external LB is still sending traffic to that node's IP (LB health checks per-node help mitigate — cloud LBs typically probe the `healthCheckNodePort`).

**Q6: What's the purpose of conntrack and what happens when its table fills up?**
> Conntrack tracks connection state (5-tuple: src/dst IP+port, protocol) so kube-proxy's NAT rules only need to apply on the FIRST packet of a connection — subsequent packets are handled by conntrack's existing mapping, not re-evaluated against iptables rules (this is a HUGE performance win). When the table fills (`nf_conntrack_max` exceeded, common with high connection churn or SNAT-heavy workloads), new connections are silently dropped — a classic hard-to-diagnose production issue. Fix: increase `net.netfilter.nf_conntrack_max` sysctl, investigate connection churn (are you creating new connections instead of reusing keep-alive?).

---

## 2.3 Network Policies

### Policy Specification & Implementation

```
IMPORTANT: NetworkPolicy is just an API object — it does NOTHING
without a CNI plugin that implements enforcement (Calico, Cilium,
Weave). Flannel alone does NOT enforce NetworkPolicy.

DEFAULT BEHAVIOR: no NetworkPolicy = all traffic allowed (flat network)
Once ANY NetworkPolicy selects a pod, that pod's traffic (for the
matched direction — ingress or egress) becomes DEFAULT DENY except
what's explicitly allowed by policies selecting it. Policies are
ADDITIVE (union of all matching policies), never subtractive.
```

```yaml
# Default deny all ingress (recommended baseline for every namespace)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}        # selects ALL pods in namespace
  policyTypes:
  - Ingress
---
# Default deny all egress too (stricter zero-trust baseline)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
---
# Allow specific: frontend → backend on port 8080 only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
---
# CRITICAL: allow DNS egress (forgotten constantly, breaks everything)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
# CIDR-based egress (allow only to a specific external API range)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24
        except:
        - 203.0.113.100/32   # exclude a specific known-bad IP
    ports:
    - protocol: TCP
      port: 443
```

### Interview Questions — Network Policies

**Q1: Implement zero-trust networking in Kubernetes.**
> Start with default-deny-ingress AND default-deny-egress in every namespace. Then explicitly allow: (1) DNS egress to kube-system (always needed), (2) specific pod-to-pod communication paths using podSelector-based rules with named ports, (3) egress to specific external CIDRs for third-party APIs, (4) ingress from your ingress-controller namespace only for internet-facing services. Use a policy engine like Cilium for L7-aware rules (e.g. only allow GET on /api/health, not full port access) for defense in depth beyond L3/L4.

**Q2: Application can't connect to database after NetworkPolicy change.**
> Check if a new default-deny was added without corresponding allow rules for the app→db path. Verify label selectors match EXACTLY (case-sensitive, common mistake: `app=db` policy vs pod labeled `app: database`). Check namespaceSelector requires the target namespace to have the referenced label (e.g. `kubernetes.io/metadata.name` — auto-added by K8s 1.21+, but older namespaces need manual labeling). Test with `kubectl exec` + `nc -zv <db-ip> <port>` from the app pod, and inspect the CNI's own policy debug tools (`calicoctl get networkpolicy` or `cilium policy trace`).

**Q3: Design network policy for PCI-DSS compliance.**
> Full default-deny baseline (ingress+egress) at namespace level. Explicit allow-lists documenting every permitted flow (auditable). Segment cardholder-data-environment (CDE) namespace with NO direct egress to internet — force traffic through an explicit egress proxy/NAT gateway with logging. Use L7 policies (Cilium) to restrict database access to specific SQL operations if possible. Combine with audit logging of all policy decisions and periodic policy review (drift detection) as compliance evidence.

**Q4: How would you debug "traffic works, but only sometimes" after applying a NetworkPolicy?**
> Likely multiple replicas of the source/destination — check if ALL destination pods have matching labels (one pod might be missing a label from a bad rollout), or if the NetworkPolicy's ingress `from` selector matches only some source pod replicas. Also check for policy evaluation order edge cases with multiple overlapping policies (remember: policies are additive/OR'd together, but a common bug is thinking one policy's `ports` restricts another policy's already-broader allow).

---

## 2.4 DNS (CoreDNS)

### DNS Architecture

```
CoreDNS runs as a Deployment (typically 2+ replicas) in kube-system,
exposed via a Service (kube-dns, ClusterIP typically 10.96.0.10).
Every pod's /etc/resolv.conf points nameserver to this ClusterIP
(injected automatically by kubelet based on dnsPolicy).

Corefile (CoreDNS config, itself a ConfigMap):
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {         # forward non-cluster queries upstream
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}

DNS RECORD TYPES GENERATED:
├─ Service (ClusterIP): A record  my-svc.my-ns.svc.cluster.local → VIP
├─ Service (Headless):  A records for EACH ready pod IP (no VIP)
├─ Pod (if enabled):    A record  10-244-2-15.my-ns.pod.cluster.local
├─ SRV records:         _http._tcp.my-svc.my-ns.svc.cluster.local
│                       (useful for clients doing SRV-based discovery,
│                        returns port + target hostname)
└─ StatefulSet pods (via headless svc): stable per-pod A record
   my-pod-0.my-svc.my-ns.svc.cluster.local → that specific pod's IP

DNS SEARCH PATH (from pod's /etc/resolv.conf, dnsPolicy=ClusterFirst):
search my-ns.svc.cluster.local svc.cluster.local cluster.local
→ this is WHY a short name like "my-service" resolves: the resolver
  tries each search suffix until one succeeds — but this also means
  every "not found" name (e.g. a typo, or a legitimately external
  name with dots) generates 3-4 EXTRA failed lookups before finally
  trying it as a plain FQDN — a hidden source of DNS latency/load
  at high query rates ("the ndots:5 problem").
```

### The ndots:5 Problem (Famous Production Issue)

```
Default pod resolv.conf: options ndots:5

MEANING: if a queried name has FEWER than 5 dots, resolver tries
ALL search suffixes FIRST before trying it as absolute.

Example: querying "myapp.default.svc.cluster.local" (5 dots exactly,
so ndots:5 rule says: try search paths first anyway since dots<5 is
false... wait, 5 is NOT < 5, so this one goes direct) — but querying
an EXTERNAL name like "api.stripe.com" (2 dots, well under 5):
1. Try api.stripe.com.my-ns.svc.cluster.local  (NXDOMAIN)
2. Try api.stripe.com.svc.cluster.local         (NXDOMAIN)
3. Try api.stripe.com.cluster.local              (NXDOMAIN)
4. Try api.stripe.com.<node's search domain>      (maybe NXDOMAIN)
5. Finally try api.stripe.com. as absolute        (SUCCESS)

= 5x the DNS queries for EVERY external hostname lookup, at scale
this multiplies CoreDNS load 5x and adds latency (each failed
lookup round-trip before the real one succeeds).

FIXES:
1. Use fully-qualified names with trailing dot: "api.stripe.com."
   (bypasses search path entirely) in application config
2. Set pod-level dnsConfig with ndots:2 or similar tuned value
3. Scale CoreDNS replicas + enable caching (cache plugin, already
   in Corefile above) to absorb the extra load
4. Use NodeLocal DNSCache (see below) to cache locally per-node
```

### NodeLocal DNSCache (Scaling CoreDNS)

```
PROBLEM AT SCALE: thousands of pods all querying a handful of
CoreDNS replicas via a single ClusterIP causes:
├─ conntrack table pressure (UDP DNS "connections" tracked)
├─ Uneven load balancing across CoreDNS pods (same iptables
│  randomness issue as any Service)
└─ Latency spikes under high query volume

SOLUTION: NodeLocalDNS runs a DNS caching agent as a DaemonSet
(one per node), pods query this LOCAL cache first (via a link-local
IP like 169.254.20.10, avoiding iptables/Service NAT entirely for
cache hits), only cache MISSES go to real CoreDNS. This:
├─ Eliminates conntrack entries for cached queries (huge win)
├─ Removes iptables DNAT overhead for cache hits
└─ Provides consistent low latency even during CoreDNS pod restarts
   (local cache continues serving cached entries)
```

### Interview Questions — DNS

**Q1: DNS resolution is slow (>100ms). Debug and fix.**
> Check CoreDNS pod CPU/memory (`kubectl top pods -n kube-system -l k8s-app=kube-dns`) — often under-provisioned for cluster size. Check for the ndots:5 problem causing 5x query amplification for external names. Check CoreDNS logs for upstream forward failures/timeouts (`forward` plugin issues reaching outside DNS). Consider deploying NodeLocal DNSCache to eliminate the network hop + conntrack overhead entirely. Check `kubectl get endpoints kube-dns -n kube-system` — enough healthy CoreDNS replicas for the query volume?

**Q2: Explain DNS-based service discovery in Kubernetes.**
> Every Service gets an A record `<svc>.<ns>.svc.cluster.local` resolving to its ClusterIP (or all pod IPs for headless Services). CoreDNS generates these dynamically by watching Service/Endpoints objects via the Kubernetes API (using the `kubernetes` plugin) — it's not static config, it's live and reflects the current cluster state. Pods use this instead of hardcoded IPs, enabling service discovery that survives pod rescheduling/scaling.

**Q3: Pod can't resolve external domain names.**
> Verify pod's `/etc/resolv.conf` has correct nameserver (should be CoreDNS ClusterIP, unless `dnsPolicy: Default` was mistakenly set, which uses the NODE's resolv.conf instead — a common misconfiguration). Check CoreDNS's `forward` plugin target (`/etc/resolv.conf` on the CoreDNS pod itself, or explicit upstream IPs) is reachable — test from within a CoreDNS pod. Check NetworkPolicy isn't blocking egress DNS (UDP/TCP 53) from application namespace to kube-system. Check for the ndots issue causing timeouts perceived as failures under load.

**Q4: How would you scale CoreDNS for a 5,000-node cluster?**
> Increase CoreDNS replica count (with pod anti-affinity to spread across nodes), tune the `cache` plugin TTL upward for better hit rates, deploy NodeLocalDNSCache to massively cut the query volume actually reaching CoreDNS replicas, use `autoscaler` (cluster-proportional-autoscaler) to scale CoreDNS replica count proportionally to cluster node/core count automatically, and set appropriate resource requests/limits (CoreDNS is usually CPU-bound under load, not memory-bound).
