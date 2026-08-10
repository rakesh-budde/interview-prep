# SECTION 19: HANDS-ON LABS

> Individual per-topic labs already appear at the end of each section (1.16, 2.17, 3.17, 4-14 embedded labs). This section provides larger, portfolio-worthy **project-based** labs spanning multiple topics, organized by difficulty.

## 19.1 Beginner Projects

1. **Landing Zone Starter Kit:** Management Groups + Policy initiative (mandatory tags, approved regions) + a single subscription onboarded via a basic Terraform module. *(Sections 1, 8)*
2. **Public AKS "Hello World" Platform:** Deploy a public AKS cluster, expose a sample app via a LoadBalancer Service, add Container Insights for basic observability. *(Sections 6, 11)*
3. **Secure Storage Baseline:** Storage Account with a Private Endpoint, public access disabled, lifecycle management policy, and Soft Delete enabled. *(Section 5)*
4. **Basic CI/CD Pipeline:** GitHub Actions workflow using OIDC federation (no stored secrets) to deploy a Bicep template to a resource group. *(Sections 2, 10)*

## 19.2 Intermediate Projects

5. **Private AKS with Workload Identity:** Private cluster, Azure CNI Overlay, Workload Identity Federation for a pod reading a Key Vault secret with zero stored credentials, KEDA scaling a Deployment on Service Bus queue depth. *(Sections 2, 6, 14)*
6. **Hub-Spoke Network with Centralized Egress:** Hub VNet with Azure Firewall (FQDN allow-listing), spoke VNet with forced-tunneling UDRs, Private Endpoint for a Storage Account, and Network Watcher-based connectivity verification. *(Section 3)*
7. **Multi-Environment Terraform Module:** A single parameterized module deploying Dev/Staging/Prod with separate state backends, `prevent_destroy` on stateful resources, and a drift-detection CI job running `terraform plan` on a schedule. *(Section 8)*
8. **Observability Stack for a Microservices App:** OpenTelemetry-instrumented sample app on AKS, Managed Prometheus + Managed Grafana dashboards, and a multi-burn-rate SLO alert. *(Section 11)*

## 19.3 Advanced Projects

9. **Multi-Tenant AKS Platform:** Namespace-per-team isolation, Azure RBAC for Kubernetes Authorization, default-deny NetworkPolicies with explicit DNS/API-server egress rules, ResourceQuotas, and Gatekeeper constraints (no privileged pods, approved registries only). *(Section 6)*
10. **Event-Driven Order Processing System:** Service Bus Topics with Sessions for a trip/order lifecycle saga (with compensating transactions), Event Hub for high-volume telemetry ingestion, and KEDA-scaled consumers. *(Section 14, System Design Section 15.7)*
11. **Zero-Standing-Privilege Access Model:** PIM-eligible (not standing) role assignments for all privileged roles, Conditional Access requiring phishing-resistant MFA for activation, and a Sentinel analytics rule alerting on anomalous PIM activation patterns. *(Sections 2, 12)*
12. **Ransomware-Resilience Drill:** Immutable Azure Backup policies, CMK with a documented/tested key-revocation runbook, and a full simulated recovery drill measuring actual RTO against a target. *(Section 12)*

## 19.4 Expert Projects

13. **Full PCI-Style Segmented Architecture:** Multi-region hub-spoke with Firewall Premium (TLS inspection), a zero-internet-egress "cardholder data" spoke with Private Endpoints for every dependency, and a public spoke behind App Gateway + WAF — validated with a documented threat model proving no path bypasses the firewall. *(Sections 3, 12, System Design 15.6)*
14. **Multi-Region Active-Active AKS Platform:** Independent regional AKS clusters, GitOps (Flux/Argo CD) deploying identical manifests to all regions, Cosmos DB multi-region write for the data tier, Front Door global load balancing with automated failover drills measuring real RTO. *(System Design 15.10)*
15. **Complete Internal Developer Platform:** Self-service AKS provisioning via a parameterized Terraform module + pipeline trigger, per-team Key Vault/RBAC isolation, KEDA-scaled ephemeral build agents, and org-wide policy-as-code guardrails (mandatory security scan stage, approved base images) enforced via shared pipeline templates. *(System Design 15.8, Sections 8-10)*
16. **AI/LLM Platform with RAG:** Azure OpenAI Service behind API Management (rate limiting, content filtering), Azure AI Search for vector-based retrieval augmentation, semantic response caching in Redis, and Private Endpoints keeping all traffic off the public internet. *(System Design 15.11)*

---

# SECTION 20: DOCUMENTATION INDEX

> Consolidated, curated list of official Microsoft Learn resources referenced throughout this guide, organized by topic for quick reference during study.

## 20.1 Fundamentals & Governance
- [Azure Regions and Availability Zones](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview)
- [Azure Resource Manager Overview](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/overview)
- [Azure Policy Overview](https://learn.microsoft.com/en-us/azure/governance/policy/overview)
- [Cloud Adoption Framework — Landing Zones](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/)
- [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/)
- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)

## 20.2 Identity
- [Microsoft Entra ID Fundamentals](https://learn.microsoft.com/en-us/entra/fundamentals/whatis)
- [Workload Identity Federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- [Privileged Identity Management](https://learn.microsoft.com/en-us/entra/id-governance/privileged-identity-management/pim-configure)
- [Conditional Access Overview](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview)

## 20.3 Networking
- [Virtual Network Overview](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview)
- [Azure Private Link](https://learn.microsoft.com/en-us/azure/private-link/private-link-overview)
- [Hub-Spoke Network Topology](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke)
- [Azure Firewall Overview](https://learn.microsoft.com/en-us/azure/firewall/overview)
- [ExpressRoute Overview](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-introduction)

## 20.4 Compute & Storage
- [Virtual Machines Overview](https://learn.microsoft.com/en-us/azure/virtual-machines/overview)
- [Azure Storage Redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [ADLS Gen2 Introduction](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)

## 20.5 AKS & Containers
- [AKS Architecture](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-start-here)
- [AKS Network Concepts](https://learn.microsoft.com/en-us/azure/aks/concepts-network)
- [AKS Workload Identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [Cluster Autoscaler on AKS](https://learn.microsoft.com/en-us/azure/aks/cluster-autoscaler)
- [KEDA on AKS](https://learn.microsoft.com/en-us/azure/aks/keda-about)
- [Troubleshoot AKS](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/welcome-azure-kubernetes)

## 20.6 IaC & CI/CD
- [Terraform azurerm Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [Azure Pipelines Documentation](https://learn.microsoft.com/en-us/azure/devops/pipelines/)
- [GitHub Actions OIDC with Azure](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure)

## 20.7 Observability & Security
- [Azure Monitor Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/overview)
- [Azure Monitor Managed Prometheus](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/prometheus-metrics-overview)
- [Microsoft Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-cloud-introduction)
- [Microsoft Sentinel Overview](https://learn.microsoft.com/en-us/azure/sentinel/overview)
- [Google SRE Book (SLOs)](https://sre.google/sre-book/service-level-objectives/)

## 20.8 Databases & Messaging
- [Cosmos DB Consistency Levels](https://learn.microsoft.com/en-us/azure/cosmos-db/consistency-levels)
- [Azure SQL Service Tiers](https://learn.microsoft.com/en-us/azure/azure-sql/database/service-tiers-general-purpose-business-critical)
- [Choosing an Azure Messaging Service](https://learn.microsoft.com/en-us/azure/event-grid/compare-messaging-services)

## 20.9 Reliability & Security Deep Reference
- [Azure Reliability Documentation](https://learn.microsoft.com/en-us/azure/reliability/)
- [Azure Security Documentation](https://learn.microsoft.com/en-us/security/)
- [Azure Well-Architected Framework — Reliability Pillar](https://learn.microsoft.com/en-us/azure/well-architected/reliability/)
- [Azure Well-Architected Framework — Security Pillar](https://learn.microsoft.com/en-us/azure/well-architected/security/)

---

*This concludes the 20-section Azure Interview Preparation Roadmap. Return to [README.md](./README.md) for the master index and study plan.*
