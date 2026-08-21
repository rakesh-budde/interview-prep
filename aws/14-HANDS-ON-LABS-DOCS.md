# Sections 19–20: Hands-On Labs · Documentation Index

> Part of the [AWS Interview Preparation Roadmap](./README.md). Covers **Section 19: Hands-On Labs** (Beginner→Expert) and **Section 20: Documentation Index**.

---

# SECTION 19: HANDS-ON LABS

> Build these in a sandbox account. **Always tear down** (destroy) afterward to avoid charges, and set a Budget alert. Prefer Terraform so labs are reproducible.

## 19.1 Beginner

### B1 — Networking: Three-Tier VPC
Build a 3-AZ VPC (public/private/data), IGW, per-AZ NAT, route tables, SGs. Deploy an EC2 web app behind an ALB.
- **Skills:** subnets, routing, SG vs NACL, ALB.
- **Success:** app reachable via ALB; private instances egress via NAT only.

### B2 — IAM: Roles & Instance Profiles
Create an IAM role granting read-only S3; attach to EC2; verify with `aws s3 ls` (no keys on the box).
- **Skills:** roles, instance profiles, least privilege, IMDSv2.

### B3 — Storage: S3 Lifecycle + Versioning
Enable versioning; add lifecycle to transition to IA/Glacier and expire old versions; enable default encryption.
- **Skills:** storage classes, lifecycle, encryption.

### B4 — Compute: Auto Scaling
Create an ASG with target-tracking on CPU behind an ALB; load-test to trigger scale-out/in.
- **Skills:** launch templates, ASG policies, health checks.

## 19.2 Intermediate

### I1 — EKS Cluster with IRSA (Terraform)
Provision EKS via `terraform-aws-modules/eks`; enable OIDC; give a pod scoped S3 access via IRSA; verify `aws sts get-caller-identity` in-pod.
- **Skills:** EKS, IRSA, OIDC, Terraform.

### I2 — CI/CD: GitHub Actions → ECS (keyless)
IAM OIDC provider + deploy role scoped to `main`; build image → ECR; deploy to ECS Fargate behind ALB; blue/green with CodeDeploy + alarm rollback.
- **Skills:** OIDC, ECR, ECS, deployment strategies.

### I3 — Terraform Backend & Modules
S3 + DynamoDB backend with KMS; build a reusable VPC module; consume in two environments; add checkov to CI.
- **Skills:** remote state, locking, modules, policy-as-code.

### I4 — Observability Stack
Instrument an app with ADOT (OTel): traces→X-Ray, metrics→AMP, dashboards in AMG; define an SLO + burn-rate alarm→SNS.
- **Skills:** OpenTelemetry, AMP/AMG, SLOs.

### I5 — Event-Driven: Fan-Out + DLQ
SNS→(2×SQS)→Lambda consumers with a DLQ; force a failure; redrive from DLQ; make consumers idempotent.
- **Skills:** SNS/SQS, DLQ, idempotency.

## 19.3 Advanced

### A1 — Karpenter + Spot on EKS
Install Karpenter; deploy a workload that triggers JIT Spot node provisioning; enable consolidation; add interruption handling.
- **Skills:** Karpenter, Spot, bin-packing, cost.

### A2 — Multi-Account Landing Zone
Set up Organizations + Control Tower; add an OU + SCP denying `s3:DeleteBucket` and region restrictions; centralize CloudTrail/Config in a log-archive account.
- **Skills:** Organizations, SCPs, governance.

### A3 — Hub-and-Spoke with Transit Gateway
Connect 3 VPCs via TGW; central egress VPC with AWS Network Firewall; force `0.0.0.0/0` through inspection; verify with Flow Logs.
- **Skills:** TGW, centralized egress, firewall.

### A4 — DynamoDB Single-Table Design
Model a chat/social app single-table with GSIs; reproduce a hot partition; fix with write sharding; add Streams + TTL.
- **Skills:** NoSQL modeling, partitioning, streams.

### A5 — Security: KMS + Auto-Remediation
Create a CMK; encrypt S3/EBS; cross-account decrypt via key policy; GuardDuty finding→EventBridge→Lambda quarantine.
- **Skills:** envelope encryption, detection, response automation.

## 19.4 Expert

### E1 — Multi-Region Active-Active EKS
Two regional EKS clusters, Route 53 latency + failover (or Global Accelerator), DynamoDB Global Tables, GitOps (ArgoCD) to both; run a Region-evacuation game day.
- **Skills:** multi-Region, global routing, DR, static stability.

### E2 — Progressive Delivery at Scale
Argo Rollouts canary with automated metric analysis (Prometheus) + auto-rollback; wave-based rollout across cells.
- **Skills:** progressive delivery, SLO gating.

### E3 — Cost Optimization Program
Instrument Cost Explorer + Storage Lens + Compute Optimizer; migrate to Graviton; Savings Plans + Spot; document savings with reliability checks.
- **Skills:** FinOps, right-sizing, trade-offs.

### E4 — LLM Inference Platform
vLLM/TGI on EKS GPU/Inferentia nodes (Karpenter Spot), model artifacts in S3, RAG with OpenSearch/pgvector, token-based autoscaling, guardrails; compare with Bedrock.
- **Skills:** GPU scheduling, RAG, cost control.

## 19.5 Lab Discipline

- Tag every lab resource (`Project=interview-labs`); `terraform destroy` when done.
- Set an AWS Budget + anomaly alert on the sandbox account.
- Never use the root user; use an admin role with MFA.
- Keep each lab in its own state/directory for clean teardown.

---

# SECTION 20: DOCUMENTATION INDEX

> Official AWS docs, Architecture Center, Well-Architected, Reliability, and Security references per topic. Use official Microsoft-of-AWS sources: docs.aws.amazon.com, aws.amazon.com/architecture, AWS Skill Builder.

## 20.1 Foundational

- AWS Documentation home: https://docs.aws.amazon.com/
- AWS Architecture Center: https://aws.amazon.com/architecture/
- Well-Architected Framework: https://docs.aws.amazon.com/wellarchitected/latest/framework/
- AWS Builders' Library: https://aws.amazon.com/builders-library/
- AWS Skill Builder (training): https://skillbuilder.aws/
- AWS Global Infrastructure: https://aws.amazon.com/about-aws/global-infrastructure/

## 20.2 Governance & Fundamentals (Section 1)

- Organizations: https://docs.aws.amazon.com/organizations/latest/userguide/
- Control Tower: https://docs.aws.amazon.com/controltower/latest/userguide/
- AWS Config: https://docs.aws.amazon.com/config/latest/developerguide/
- Service Quotas: https://docs.aws.amazon.com/servicequotas/latest/userguide/

## 20.3 Identity (Section 2)

- IAM: https://docs.aws.amazon.com/IAM/latest/UserGuide/
- Policy evaluation logic: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html
- STS: https://docs.aws.amazon.com/STS/latest/APIReference/
- IAM Identity Center: https://docs.aws.amazon.com/singlesignon/latest/userguide/
- Security Pillar (WAF): https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/

## 20.4 Networking (Section 3)

- VPC: https://docs.aws.amazon.com/vpc/latest/userguide/
- Transit Gateway: https://docs.aws.amazon.com/vpc/latest/tgw/
- PrivateLink: https://docs.aws.amazon.com/vpc/latest/privatelink/
- Route 53: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/
- ELB: https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/

## 20.5 Compute & Storage (Sections 4–5)

- EC2: https://docs.aws.amazon.com/ec2/
- Nitro: https://aws.amazon.com/ec2/nitro/
- Auto Scaling: https://docs.aws.amazon.com/autoscaling/ec2/userguide/
- Lambda: https://docs.aws.amazon.com/lambda/latest/dg/
- S3: https://docs.aws.amazon.com/AmazonS3/latest/userguide/
- EBS/EFS/FSx: https://docs.aws.amazon.com/ebs/ · https://docs.aws.amazon.com/efs/ · https://docs.aws.amazon.com/fsx/

## 20.6 EKS & Containers (Sections 6–7)

- EKS User Guide: https://docs.aws.amazon.com/eks/latest/userguide/
- EKS Best Practices: https://aws.github.io/aws-eks-best-practices/
- VPC CNI: https://github.com/aws/amazon-vpc-cni-k8s
- IRSA: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
- Pod Identity: https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html
- Karpenter: https://karpenter.sh/docs/
- AWS Load Balancer Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/
- ECS/ECR/Fargate: https://docs.aws.amazon.com/AmazonECS/ · https://docs.aws.amazon.com/AmazonECR/

## 20.7 Terraform & CI/CD (Sections 8–10)

- Terraform AWS provider: https://registry.terraform.io/providers/hashicorp/aws/latest/docs
- Terraform S3 backend: https://developer.hashicorp.com/terraform/language/settings/backends/s3
- CodePipeline/Build/Deploy: https://docs.aws.amazon.com/codepipeline/ · https://docs.aws.amazon.com/codebuild/ · https://docs.aws.amazon.com/codedeploy/
- GitHub Actions OIDC with AWS: https://docs.github.com/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services

## 20.8 Observability & Security (Sections 11–12)

- CloudWatch: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/
- X-Ray: https://docs.aws.amazon.com/xray/latest/devguide/
- AMP / AMG: https://docs.aws.amazon.com/prometheus/ · https://docs.aws.amazon.com/grafana/
- KMS: https://docs.aws.amazon.com/kms/latest/developerguide/
- Secrets Manager: https://docs.aws.amazon.com/secretsmanager/latest/userguide/
- GuardDuty / Security Hub: https://docs.aws.amazon.com/guardduty/ · https://docs.aws.amazon.com/securityhub/

## 20.9 Databases & Event-Driven (Sections 13–14)

- RDS / Aurora: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/ · https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/
- DynamoDB: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/
- ElastiCache / Redshift: https://docs.aws.amazon.com/AmazonElastiCache/ · https://docs.aws.amazon.com/redshift/
- SQS / SNS / EventBridge: https://docs.aws.amazon.com/AWSSimpleQueueService/ · https://docs.aws.amazon.com/sns/ · https://docs.aws.amazon.com/eventbridge/
- Kinesis / MSK / Step Functions: https://docs.aws.amazon.com/streams/ · https://docs.aws.amazon.com/msk/ · https://docs.aws.amazon.com/step-functions/

## 20.10 Reliability & Resilience

- Reliability Pillar: https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/
- AWS Resilience Hub: https://docs.aws.amazon.com/resilience-hub/
- AWS Health Dashboard: https://docs.aws.amazon.com/health/latest/ug/
- Disaster Recovery whitepaper: https://docs.aws.amazon.com/whitepapers/latest/disaster-recovery-workloads-on-aws/

## 20.11 Certifications (map to study)

| Cert | Relevance |
|------|-----------|
| Solutions Architect – Associate/Professional | Architecture, trade-offs, system design |
| DevOps Engineer – Professional | CI/CD, IaC, monitoring, automation |
| Security – Specialty | IAM, KMS, detection, data perimeter |
| Advanced Networking – Specialty | VPC, TGW, hybrid, DNS |

## 20.12 Study Cadence (recap)

- Read the section → do the lab → answer the Q&A out loud → note weak spots.
- Weekly: one timed system design + one troubleshooting drill.
- Final weeks: full loop simulation + STAR polish.

---

> This completes **Sections 19–20** and the full 20-section AWS Interview Preparation Roadmap. Return to the **[README](./README.md)** for the master table of contents.
