# Kubernetes Interview Preparation — FAANG/MANGA Level (>50 LPA)

> **500+ questions, system internals, hands-on labs, and troubleshooting scenarios** for Senior DevOps Engineers, SREs, and Platform Engineers targeting Google, Amazon, Meta, Netflix, Microsoft, and similar high-paying roles.

---

## 📚 Study Guides (Weightage-Based)

| # | Guide | Weight | Questions | Focus |
|---|-------|--------|-----------|-------|
| 1 | [Control Plane Internals](01-CONTROL-PLANE-INTERNALS.md) | **25%** | 125+ | API Server, etcd/Raft, Scheduler, Controller Manager |
| 2 | [Networking Deep Dive](02-NETWORKING-DEEP-DIVE.md) | **20%** | 100+ | CNI, kube-proxy, Services, NetworkPolicy, CoreDNS |
| 3 | [Security Hardening](04-SECURITY-HARDENING.md) | **15%** | 75+ | AuthN/RBAC, Pod Security, Secrets, Runtime security |
| 4 | [Troubleshooting Guide](08-TROUBLESHOOTING-GUIDE.md) | **15%** | 75+ | 20 production incidents, root cause, fixes |
| 5 | [System Design Patterns](09-SYSTEM-DESIGN-PATTERNS.md) | **10%** | 50+ | 10 large-scale design problems |
| 6 | [Storage Internals](03-STORAGE-INTERNALS.md) | **5%** | 25+ | CSI, PV/PVC, StorageClass, snapshots |
| 7 | [Scaling & Performance](06-SCALING-PERFORMANCE.md) | **5%** | 25+ | HPA/VPA/CA, QoS, resource model |
| 8 | [Observability Stack](07-OBSERVABILITY-STACK.md) | **5%** | 25+ | Logging, Prometheus, tracing |
| — | [Workloads Lifecycle](05-WORKLOADS-LIFECYCLE.md) | — | 40+ | Pods, probes, Deployment/StatefulSet/DaemonSet |
| — | [Hands-On Labs](10-HANDS-ON-LABS.md) | — | 30+ labs | Runnable exercises (kind/minikube/kubeadm) |
| — | [Quick Reference](12-QUICK-REFERENCE.md) | — | — | Commands, cheatsheets, diagrams |

**Total: 500+ questions | ~6000+ lines of production-grade content**

---

## 🗓️ 8-Week Study Plan

```
Week 1-2: Control Plane Internals (25%)
├─ Day 1-3:  API Server request flow, admission control, watch/informers
├─ Day 4-7:  etcd Raft consensus, backup/restore, disaster recovery
├─ Day 8-10: Scheduler algorithm (filter/score), affinity, taints
└─ Day 11-14: Controller Manager, informer pattern, custom controllers

Week 3-4: Networking (20%)
├─ Day 15-18: CNI specification, Calico vs Cilium vs Flannel
├─ Day 19-22: kube-proxy (iptables/IPVS/eBPF), Service types
├─ Day 23-25: NetworkPolicy, zero-trust patterns
└─ Day 26-28: CoreDNS, DNS troubleshooting

Week 5: Security (15%)
├─ Day 29-31: AuthN (certs, tokens, OIDC), RBAC evaluation
├─ Day 32-33: Pod Security Standards, seccomp/AppArmor
└─ Day 34-35: Secrets encryption, Vault/External Secrets Operator

Week 6: Storage, Scaling, Observability (15%)
├─ Day 36-38: CSI, StorageClass, PVC troubleshooting
├─ Day 39-41: HPA/VPA/Cluster Autoscaler, QoS classes
└─ Day 42: Prometheus/Grafana, logging, tracing

Week 7: Troubleshooting & Workloads
├─ Day 43-46: All 20 troubleshooting scenarios (hands-on)
└─ Day 47-49: Pod lifecycle, Deployment strategies, StatefulSet

Week 8: System Design & Mock Interviews
├─ Day 50-53: All 10 system design problems (whiteboard practice)
├─ Day 54-56: Hands-on labs (kind/kubeadm cluster build)
└─ Day 57-56: Mock interviews, weak-area review
```

---

## 🎯 How to Use This Guide

1. **Read theory** in each numbered guide — all content is at *system internals* depth (not just "what" but "how" and "why").
2. **Run the hands-on labs** in [10-HANDS-ON-LABS.md](10-HANDS-ON-LABS.md) on kind/minikube/kubeadm — don't just read, execute.
3. **Practice troubleshooting** in [08-TROUBLESHOOTING-GUIDE.md](08-TROUBLESHOOTING-GUIDE.md) by intentionally breaking your lab cluster and fixing it using only the symptoms.
4. **Whiteboard the system designs** in [09-SYSTEM-DESIGN-PATTERNS.md](09-SYSTEM-DESIGN-PATTERNS.md) out loud, timing yourself to 45 minutes each.
5. **Use** [12-QUICK-REFERENCE.md](12-QUICK-REFERENCE.md) the night before your interview for rapid recall.

---

## 🏗️ Kubernetes Architecture (Reference Diagram)

```
┌───────────────────────────────────────────────────────────────────────┐
│                         CONTROL PLANE (HA: 3 nodes)                    │
│                                                                        │
│   ┌────────────┐   ┌──────────┐   ┌──────────────────┐   ┌─────────┐  │
│   │ API Server │◄──┤   etcd   │   │ Scheduler         │   │ Cloud   │  │
│   │ (stateless)│──►│ (Raft,   │   │ (filter+score)    │   │Controller│  │
│   │            │   │  3/5/7)  │   │                    │   │ Manager │  │
│   └─────┬──────┘   └──────────┘   └──────────────────┘   └─────────┘  │
│         │                          ┌──────────────────┐               │
│         │                          │ Controller Manager│               │
│         │                          │ (Deploy/RS/SS/...)│               │
│         │                          └──────────────────┘               │
└─────────┼──────────────────────────────────────────────────────────────┘
          │  watch / list / apply (HTTPS + mTLS)
┌─────────┼──────────────────────────────────────────────────────────────┐
│         ▼                     WORKER NODES                             │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Node                                                            │   │
│  │  ┌─────────┐  ┌────────────┐  ┌─────────────┐  ┌─────────────┐ │   │
│  │  │ kubelet │  │ kube-proxy │  │ containerd  │  │ CNI plugin  │ │   │
│  │  │(PLEG,   │  │(iptables/  │  │(CRI, runc,  │  │(Calico/     │ │   │
│  │  │ cAdvisor│  │ IPVS/eBPF) │  │ containerd- │  │ Cilium/     │ │   │
│  │  │ probes) │  │            │  │ shim)       │  │ Flannel)    │ │   │
│  │  └─────────┘  └────────────┘  └─────────────┘  └─────────────┘ │   │
│  │       Pod A          Pod B          Pod C                       │   │
│  └────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

Start with [01-CONTROL-PLANE-INTERNALS.md](01-CONTROL-PLANE-INTERNALS.md) →
