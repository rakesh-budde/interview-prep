# Learning Resources & Study Roadmap

> 90-day study plan, top 50 questions, certifications, and resources

---

## 90-Day Study Roadmap

### Week 1-2: AKS Fundamentals (25% weight)
```
Day 1-3: AKS Architecture
├─ Read: AKS-KUBERNETES.md (control plane section)
├─ Hands-on: Deploy AKS cluster via Terraform
├─ Practice: Explain pod deployment flow (whiteboard)
└─ Quiz: 20 AKS architecture questions

Day 4-7: Networking & Identity
├─ Read: Azure CNI vs Kubenet vs Overlay
├─ Hands-on: Configure network policies (Calico)
├─ Read: Workload Identity deep dive
├─ Practice: Design AKS networking (interview style)
└─ Quiz: 15 networking questions

Day 8-10: Node Pools & Scaling
├─ Read: Node pool design patterns
├─ Hands-on: Configure cluster autoscaler
├─ Practice: Size cluster for 1000 pods
└─ Quiz: 10 scaling questions

Day 11-14: Troubleshooting
├─ Lab: Pod stuck in Pending (diagnose & fix)
├─ Lab: CrashLoopBackOff (various causes)
├─ Lab: Image pull failures
├─ Lab: Network connectivity issues
└─ Practice: Walk through troubleshooting flowchart
```

### Week 3-4: Azure Networking (20% weight)
```
Day 15-18: VNet & Subnets
├─ Read: NETWORKING.md (VNet section)
├─ Hands-on: Design 3-tier VNet with Terraform
├─ Practice: CIDR planning exercise
└─ Quiz: 15 VNet questions

Day 19-22: Security & Connectivity
├─ Read: NSG rule evaluation
├─ Hands-on: Configure NSG for web→app→db
├─ Read: ExpressRoute vs VPN
├─ Practice: Design hybrid connectivity
└─ Quiz: 15 security questions

Day 23-28: Advanced Networking
├─ Read: Private Endpoints vs Service Endpoints
├─ Hands-on: Configure Private Endpoint for Storage
├─ Read: Azure Firewall & Application Gateway
├─ Lab: Troubleshoot "can't reach database" scenario
└─ Practice: Multi-region network design
```

### Week 5-6: Identity & Security (15% weight)
```
Day 29-32: Entra ID Fundamentals
├─ Read: IDENTITY-SECURITY.md
├─ Hands-on: Configure service principal
├─ Practice: Explain OIDC token flow
└─ Quiz: 15 identity questions

Day 33-38: RBAC & Key Vault
├─ Read: RBAC evaluation logic
├─ Hands-on: Design RBAC for multi-team org
├─ Read: Key Vault best practices
├─ Lab: Configure Workload Identity + Key Vault
└─ Practice: Security hardening for regulated industry

Day 39-42: Compliance & Defender
├─ Read: Defender for Containers
├─ Read: Azure Policy enforcement
├─ Practice: Design for PCI-DSS compliance
└─ Quiz: 10 compliance questions
```

### Week 7-8: Terraform & CI/CD (20% weight)
```
Day 43-49: Terraform Mastery
├─ Read: TERRAFORM-INFRASTRUCTURE.md
├─ Hands-on: Multi-region AKS deployment
├─ Lab: State management & drift detection
├─ Practice: Design Terraform for 20-customer SaaS
└─ Quiz: 20 Terraform questions

Day 50-56: CI/CD Pipelines
├─ Read: AZURE-DEVOPS-CI-CD.md
├─ Hands-on: Multi-stage YAML pipeline
├─ Lab: Blue-green deployment
├─ Practice: Design GitOps workflow with ArgoCD
└─ Quiz: 15 CI/CD questions
```

### Week 9-10: Supporting Topics (15% weight)
```
Day 57-63: Compute, Storage, Databases
├─ Read: COMPUTE-STORAGE-DATABASES.md
├─ Practice: Choose VM SKU for workload
├─ Practice: Storage replication strategy
├─ Quiz: 20 mixed questions

Day 64-70: Observability & Platform Engineering
├─ Read: OBSERVABILITY-MONITORING.md
├─ Read: PLATFORM-ENGINEERING.md
├─ Lab: Configure alerting for AKS
├─ Practice: Design SLI/SLO for SaaS
└─ Quiz: 15 monitoring questions
```

### Week 11-12: System Design & Behavioral
```
Day 71-77: System Design
├─ Read: SYSTEM-DESIGN-TROUBLESHOOTING.md
├─ Practice: Multi-region AKS design (45 min)
├─ Practice: Financial services AKS design
├─ Practice: Troubleshooting scenarios (10+)
└─ Mock interview: Full system design

Day 78-84: Behavioral Preparation
├─ Read: BEHAVIORAL-LEADERSHIP.md
├─ Write: 10 STAR stories (your experience)
├─ Practice: Mock behavioral interview
├─ Review: Leadership principles
└─ Final review: All guides
```

### Week 13: Final Prep
```
Day 85-90: Review & Mock Interviews
├─ Review: Top 50 questions (below)
├─ Mock: Technical screen (45 min)
├─ Mock: System design (60 min)
├─ Mock: Behavioral (30 min)
├─ Rest day before interview
└─ Interview day: You're ready!
```

---

## Top 50 Interview Questions

### AKS & Kubernetes (15 questions)
1. Explain AKS architecture (control plane vs data plane)
2. What happens when you deploy a pod? (step-by-step)
3. Compare Azure CNI vs Kubenet vs CNI Overlay
4. How does Workload Identity work? (OIDC flow)
5. Design node pools for production AKS
6. How do you troubleshoot pod stuck in Pending?
7. Explain network policies and their enforcement
8. How does the Kubernetes scheduler work?
9. What is etcd and why is it critical?
10. How do you implement zero-downtime deployments?
11. Explain pod disruption budgets (PDB)
12. How does horizontal pod autoscaling work?
13. What are init containers and when to use them?
14. How do you handle secrets in AKS?
15. Design AKS for 99.99% availability

### Networking (10 questions)
16. Design VNet for multi-tier application
17. Explain NSG rule evaluation order
18. Compare Load Balancer vs Application Gateway
19. What are Private Endpoints? When to use?
20. Design hybrid connectivity (VPN vs ExpressRoute)
21. How does Azure DNS work? Private DNS zones?
22. Troubleshoot: Pod can't reach database
23. Explain Service Endpoints vs Private Endpoints
24. Design network security for AKS
25. How does Azure Firewall differ from NSG?

### Identity & Security (8 questions)
26. Explain Entra ID (Azure AD) architecture
27. How does RBAC work in Azure?
28. Compare Service Principal vs Managed Identity
29. Design Key Vault access for AKS pods
30. How do you implement least privilege?
31. Explain Defender for Containers
32. Design security for regulated industry (PCI-DSS)
33. How does token authentication work in Azure?

### Terraform & CI/CD (8 questions)
34. Design Terraform structure for multi-region
35. Explain Terraform state management
36. How do you handle drift detection?
37. Design multi-stage CI/CD pipeline
38. Compare blue-green vs canary deployments
39. How does GitOps work with ArgoCD?
40. Implement approval gates in pipelines
41. How do you manage secrets in CI/CD?

### System Design (5 questions)
42. Design multi-region AKS for global SaaS
43. Design AKS for financial services
44. Design disaster recovery for stateful app
45. Design internal developer platform
46. Troubleshoot cascading microservice failures

### Behavioral (4 questions)
47. Tell me about a critical incident you led
48. Describe a technical decision with significant impact
49. How do you handle pushback on recommendations?
50. Tell me about a time you had incomplete information

---

## Certifications

### Recommended Path
```
1. AZ-900: Azure Fundamentals (optional, skip if experienced)
2. AZ-104: Azure Administrator (core Azure knowledge)
3. AZ-305: Azure Solutions Architect Expert (design patterns)
4. AZ-400: DevOps Engineer Expert (CI/CD, automation)
5. CKA: Certified Kubernetes Administrator (Kubernetes deep dive)
6. CKS: Certified Kubernetes Security (security focus)
```

### Study Resources
- Microsoft Learn (free, official)
- A Cloud Guru / Pluralsight (video courses)
- Kubernetes documentation (kubernetes.io)
- Azure documentation (docs.microsoft.com)
- This interview prep guide (comprehensive)

---

## Practice Labs

### Free Labs
1. Azure Free Tier ($200 credit for 30 days)
2. Microsoft Learn Sandbox (free lab environments)
3. Katacoda Kubernetes scenarios (browser-based)
4. Play with Kubernetes (playground.kubernetes.io)

### Paid Labs
1. A Cloud Guru hands-on labs
2. Whizlabs practice tests
3. KodeKloud CKA/CKS courses

---

## Interview Day Checklist

```
Before Interview:
□ Test internet connection
□ Test camera and microphone
□ Have water nearby
□ Have notepad for system design
□ Review top 50 questions
□ Review your STAR stories
□ Get good sleep night before

During Interview:
□ Ask clarifying questions
□ Think out loud (show reasoning)
□ Draw diagrams for system design
□ Use STAR format for behavioral
□ Admit when you don't know (then reason through)
□ Ask about the role/team

After Interview:
□ Send thank-you email
□ Note questions you struggled with
□ Review areas for improvement
□ Prepare for next round
```

Good luck with your interview! 🚀
