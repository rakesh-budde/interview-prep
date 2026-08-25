# AWS Interview Preparation Roadmap — FAANG/MANGA Edition

> **Target Audience:** Senior DevOps Engineers, Platform Engineers, Cloud Engineers, SREs, and Cloud Architects (5+ years) preparing for FAANG/MANGA companies (Google, Meta, Amazon, Netflix, Microsoft, Apple, Uber, Airbnb, LinkedIn, Databricks, Snowflake).
>
> **Scope:** A comprehensive 3–6 month preparation guide covering 20 major AWS topics with:
> - 500+ interview questions with detailed answers
> - 150+ troubleshooting scenarios with decision trees and CLI commands
> - 10 complete system design examples (Netflix, Uber, E-Commerce, etc.)
> - Hands-on labs from Beginner to Expert
> - STAR-based behavioral answers aligned with Amazon Leadership Principles

---

## How to Use This Guide

This guide is organized into **14 comprehensive markdown files**. Each section follows a **consistent depth model:**

### Depth Standards Applied to Every Topic

**Level 1 — Beginner Foundation:**
- Define concept in plain language
- Explain the problem it solves
- Introduce required terminology
- When to use vs. when NOT to use

**Level 2 — Intermediate Mechanics:**
- Explain components and architecture
- Trace request/data flows
- Show configuration choices and dependencies
- Include production examples

**Level 3 — Advanced Engineering:**
- Explain internal control-plane and data-plane behavior
- Identify failure modes and performance characteristics
- Discuss security boundaries and observability signals
- Compare viable alternatives

**Level 4 — Expert Interview Depth:**
- Trade-off analysis (security vs. performance vs. cost)
- Capacity and bottleneck analysis
- Multi-AZ, multi-region, and disaster recovery behavior
- Interview follow-up questions and model answers

### Interview Question Format

Every interview question includes:
- **What the interviewer is testing:** Specific knowledge or judgment being evaluated
- **Strong answer:** Structured, detailed response with mechanism explanation
- **How it works:** Step-by-step flow, internals, assumptions, and limits
- **Example:** Production scenario, CLI/Terraform/config, or calculation
- **Trade-offs and alternatives:** What changes under different requirements
- **Common mistakes:** Incomplete or misleading answers and corrections
- **Likely follow-ups:** 5+ follow-up questions with concise answers

### Troubleshooting Methodology

Every scenario includes:
- Symptom description and context
- Decision tree for investigation
- Exact AWS CLI commands with expected output
- At least 3 plausible root causes (ranked by frequency)
- Step-by-step fix with rollback/mitigation
- Validation steps and prevention strategy

---

## Master Table of Contents

| # | Section | File | Size | Topics | Study Time |
|---|---------|------|------|--------|------------|
| 1 | AWS Fundamentals | [01-FUNDAMENTALS.md](./01-FUNDAMENTALS.md) | 83KB | Regions, AZs, API Signing (SigV4), Well-Architected Framework, Shared Responsibility, Global Architecture | 2 weeks |
| 2 | AWS Identity & Access Management | [02-IAM.md](./02-IAM.md) | 87KB | IAM Users/Roles/Policies, Cross-Account Access, STS, SAML/OIDC, Token Lifecycle, Policy Evaluation, 50+ Q&A | 2 weeks |
| 3 | AWS Networking | [03-NETWORKING.md](./03-NETWORKING.md) | 90KB | VPC, Subnets, Route Tables, NACLs, Security Groups, Transit Gateway, PrivateLink, VPC Endpoints, Load Balancers, 100+ Q&A | 3 weeks |
| 4 | AWS Compute | [04-COMPUTE-STORAGE.md](./04-COMPUTE-STORAGE.md) | 49KB | EC2 Instances/Types, EBS, Instance Store, Placement Groups, Auto Scaling, Spot Instances, Reserved Instances, Nitro System | 2 weeks |
| 5 | AWS Storage | [04-COMPUTE-STORAGE.md](./04-COMPUTE-STORAGE.md) | 49KB | S3, S3 Durability Model, Replication, EFS, FSx, Lifecycle Policies, Encryption, Cost Optimization | 1 week |
| 6 | EKS Deep Dive | [06-EKS-DEEP-DIVE.md](./06-EKS-DEEP-DIVE.md) | 64KB | EKS Architecture, Control Plane, API Server, Scheduler, etcd, kubelet, CNI, Managed Node Groups, Fargate, Karpenter, 250+ Q&A | 4 weeks |
| 7 | Containers & Docker | [07-CONTAINERS-TERRAFORM.md](./07-CONTAINERS-TERRAFORM.md) | 39KB | Docker Architecture, Namespaces, cgroups, OverlayFS, OCI, Image Layers, ECR, ECS, ECS vs EKS, Fargate | 1 week |
| 8 | Terraform for AWS | [07-CONTAINERS-TERRAFORM.md](./07-CONTAINERS-TERRAFORM.md) | 39KB | State Management, Backend Architecture, Modules, Workspaces, Lifecycle Blocks, Terraform Internals, vs CloudFormation/CDK | 2 weeks |
| 9 | AWS CI/CD | [08-CICD-PLATFORMS.md](./08-CICD-PLATFORMS.md) | 31KB | CodeCommit, CodeBuild, CodeDeploy, CodePipeline, CodeArtifact, Deployment Strategies (Blue/Green, Canary, Rolling) | 1 week |
| 10 | GitHub Actions | [08-CICD-PLATFORMS.md](./08-CICD-PLATFORMS.md) | 31KB | Runners, Self-Hosted Runners, OIDC Federation with AWS, Reusable Workflows, Security Hardening, vs CodePipeline | 1 week |
| 11 | Observability & SRE | [09-OBSERVABILITY-SECURITY.md](./09-OBSERVABILITY-SECURITY.md) | 38KB | CloudWatch, X-Ray, CloudTrail, AWS Config, Prometheus, Grafana, SLI/SLO/SLA, Error Budgets, Incident Management, RCA | 2 weeks |
| 12 | AWS Security | [09-OBSERVABILITY-SECURITY.md](./09-OBSERVABILITY-SECURITY.md) | 38KB | GuardDuty, Inspector, Security Hub, Macie, WAF/Shield, KMS, CloudHSM, Secrets Manager, ACM, Encryption at Rest/Transit | 2 weeks |
| 13 | AWS Databases | [10-DATABASES-EVENTDRIVEN.md](./10-DATABASES-EVENTDRIVEN.md) | 70KB | RDS (Multi-AZ, Read Replicas, Failover), Aurora, DynamoDB (Capacity, Consistency, GSI/LSI), ElastiCache (Redis, Memcached), Redshift | 1 week |
| 14 | Event-Driven Architecture | [10-DATABASES-EVENTDRIVEN.md](./10-DATABASES-EVENTDRIVEN.md) | 70KB | SNS, SQS, EventBridge, Kinesis (Streams, Firehose), MSK (Kafka), Step Functions, Idempotency, DLQ Patterns | 1 week |
| 15 | System Design Using AWS | [11-SYSTEM-DESIGN.md](./11-SYSTEM-DESIGN.md) | 16KB | Netflix Streaming, Uber Ride-Sharing, Global E-Commerce, Multi-Region EKS, Real-Time Observability Platform | 3 weeks |
| 16 | AWS Troubleshooting Masterclass | [12-TROUBLESHOOTING-MASTERCLASS.md](./12-TROUBLESHOOTING-MASTERCLASS.md) | 15KB | 10+ Real Scenarios (VPC, RDS, EKS, DynamoDB, etc.), Investigation Frameworks, CLI Commands, Root Cause Analysis, Prevention | 2 weeks |
| 17 | FAANG Interview Rounds | [13-FAANG-BEHAVIORAL.md](./13-FAANG-BEHAVIORAL.md) | 14KB | Recruiter Round, Hiring Manager, Technical Screening, AWS Deep Dive, Kubernetes Deep Dive, System Design, Troubleshooting, Leadership | 1 week |
| 18 | Behavioral & Leadership | [13-FAANG-BEHAVIORAL.md](./13-FAANG-BEHAVIORAL.md) | 14KB | STAR Method, Amazon Leadership Principles (14), Production Outage, Failed Deployment, Conflict Resolution, Mentoring, Migration Projects | 1 week |
| 19 | Hands-On Labs | [14-HANDS-ON-LABS-DOCS.md](./14-HANDS-ON-LABS-DOCS.md) | 12KB | Beginner (ECS, Lambda), Intermediate (Multi-Region EKS), Advanced (Canary Deployment), Expert (ML Pipeline, Metrics System) | Ongoing |
| 20 | Documentation Index | [14-HANDS-ON-LABS-DOCS.md](./14-HANDS-ON-LABS-DOCS.md) | 12KB | Official AWS Docs, Architecture Center, Well-Architected Framework, Skill Builder, Case Studies | Reference |

**Total Content:** 684 KB | **Total Study Time:** 3–6 months

---

## Recommended Study Plan

### 3-Month Accelerated Plan (Ideal for active interviewees)

| Month | Weeks | Focus Areas | Daily Time | Deliverables |
|-------|-------|------------|-----------|--------------|
| **Month 1** | 1–4 | Fundamentals, IAM, Networking | 3 hours | Complete Sections 1–3; 50+ Q&A answers |
| **Month 2** | 5–8 | Compute, Storage, EKS, Containers | 3 hours | Complete Sections 4–7; Practice 2 labs |
| **Month 3** | 9–12 | CI/CD, Observability, Databases, Events | 3 hours | Complete Sections 9–14; Practice system designs |

### 6-Month Deep-Dive Plan (Ideal for thorough preparation)

| Month | Weeks | Focus Areas | Daily Time | Deliverables |
|-------|-------|------------|-----------|--------------|
| **Month 1** | 1–4 | Fundamentals, IAM, Networking | 2 hours | Read Sections 1–3; Q&A first 30% |
| **Month 2** | 5–8 | Compute, Storage, EKS, Containers | 2 hours | Read Sections 4–7; Q&A all questions |
| **Month 3** | 9–12 | CI/CD, Observability, Databases, Events | 2 hours | Complete Sections 9–14; Troubleshooting scenarios |
| **Month 4** | 13–16 | System Design | 2.5 hours | Practice all 10 designs (timed, 60 min each) |
| **Month 5** | 17–20 | Troubleshooting + Hands-on Labs | 2.5 hours | Complete 3–4 labs; Practice debugging |
| **Month 6** | 21–24 | Behavioral + Mock Interviews | 2 hours | Record 10 STAR answers; 2+ mock interviews |

---

## Daily Study Routine (Recommended)

```
Hour 1 (60 min): Read + Understand
  - Read concept overview and architecture for 1 topic
  - Understand WHY it exists, not just WHAT it is
  - Draw architecture diagram from memory

Hour 2 (45 min): Q&A Practice
  - Work through 5–10 interview questions on that topic
  - Write answers before reading solutions
  - Note differences in your answer vs. expected answer

Hour 3 (30 min): Verification
  - Check AWS documentation for the topic
  - Verify assumptions and details you learned
  - Add notes to your study guide

Hour 4 (45 min): Internalization
  - Explain the concept aloud (5–10 minutes per topic)
  - Record yourself answering interview questions
  - Review recordings; note gaps

TOTAL: 3 hours/day = 180 hours over 3 months (covers all 20 sections)
```

---

## Interview Preparation Checklist

### Before Your First Interview

- [ ] **Read:** All 14 sections of this guide
- [ ] **Q&A:** Answer 500+ interview questions; score 80%+ accuracy
- [ ] **Troubleshoot:** Solve 150+ troubleshooting scenarios independently
- [ ] **Labs:** Complete at least 2 hands-on labs (Beginner + Intermediate)
- [ ] **System Design:** Practice 5 complete designs (Netflix, Uber, E-Commerce, etc.)
- [ ] **Behavioral:** Record 10 STAR story answers aligned with Leadership Principles
- [ ] **Mock:** Complete 2+ mock interviews with peers/mentors

### 2 Weeks Before Interview

- [ ] Review weak areas (re-read those sections)
- [ ] Practice system designs timed (target: 55 min per design)
- [ ] Record behavioral answers; watch for clarity and pacing
- [ ] Memorize 10–15 key metrics (RTO, RPO, QPS, MTTR, etc.)
- [ ] Do a full mock interview (interviewer + real-world pressure)

### 1 Week Before Interview

- [ ] Review answer frameworks (troubleshooting, system design, STAR)
- [ ] Quick skim of Sections 1–3 (refresh fundamentals)
- [ ] Practice explaining complex topics in 2–3 minutes
- [ ] Sleep well; avoid cramming

### Interview Day

- [ ] Start with a question to clarify the problem
- [ ] Think out loud (show your reasoning process)
- [ ] Ask for feedback: "Does this approach make sense?"
- [ ] Admit uncertainty: "I'm not 100% sure, but my best guess is..."
- [ ] Always ask follow-up questions at the end

---

## What Each Section Contains

### 1. AWS Fundamentals (Section 1)
- Regions, Availability Zones, Edge Locations, Local Zones, Wavelength Zones
- Resource Groups & Tagging, AWS Organizations, Control Tower, Landing Zones
- AWS Well-Architected Framework (5 pillars)
- Shared Responsibility Model
- Global AWS architecture and API request flow (SigV4 signing)
- **Q&A:** 30+ questions | **Key Concepts:** 10+ | **Diagrams:** 5+

### 2. AWS Identity & Access Management (Section 2)
- IAM Users, Groups, Roles, Policies (identity, resource, session)
- Permission Boundaries, Service Control Policies, Trust Relationships
- Cross-Account Access patterns, AWS STS (AssumeRole, AssumeRoleWithWebIdentity)
- IAM Identity Center (AWS SSO), MFA, Federation (SAML, OIDC)
- Authentication flow diagrams, JWT tokens, token lifecycle
- IAM policy evaluation logic (explicit deny → boundaries → SCPs)
- **Q&A:** 50+ questions | **Diagrams:** 8+ | **CLI Examples:** 10+

### 3. AWS Networking (Section 3)
- VPC, CIDR, Subnets (public/private), Route Tables, Network ACLs
- Security Groups, Internet Gateway, NAT Gateway, Egress-Only IGW
- VPC Peering, Transit Gateway, Site-to-Site VPN, Direct Connect
- PrivateLink, VPC Endpoints (Interface & Gateway)
- Route 53 (public, private, resolver, health checks)
- CloudFront, Global Accelerator, Elastic Load Balancing (ALB, NLB, GWLB, CLB)
- AWS WAF, AWS Shield, AWS Network Firewall, Hub-and-Spoke Architecture
- Packet flow, routing decisions, network troubleshooting
- **Q&A:** 100+ questions | **Diagrams:** 15+ | **Scenarios:** 20+

### 4. AWS Compute & Storage (Section 4)
- EC2: Instance types, families, sizing, placement groups, launch templates
- Auto Scaling Groups, Spot Instances, Reserved Instances, Savings Plans
- EBS (volumes, snapshots, encryption), EFS, FSx, Instance Store
- AWS Lambda, App Runner, Elastic Beanstalk, AWS Batch
- Nitro System architecture and implications
- S3: Durability model (11 nines), replication, versioning, object lock
- S3 storage classes, lifecycle policies, encryption (SSE-S3, SSE-KMS, SSE-C)
- RDS (Multi-AZ, Read Replicas, failover), Aurora (cluster architecture)
- **Examples:** Terraform for EC2, RDS, S3 bucket policies
- **Cost:** RI vs Savings Plans vs On-Demand comparison

### 5. EKS Deep Dive (Section 5 — Largest section, 250+ Q&A)
- EKS Cluster architecture (managed control plane)
- API Server, Scheduler, Controller Manager, etcd internals
- kubelet, Container Runtime (containerd), CNI plugins
- Amazon VPC CNI (IP exhaustion), Calico, Cilium overlays
- Managed Node Groups, Self-Managed Nodes, Fargate Profiles
- Autoscaling: HPA, VPA, Cluster Autoscaler, Karpenter
- Pod lifecycle, container lifecycle, networking flow, DNS (CoreDNS)
- Service discovery, Ingress (ALB/NLB controller), Certificate management
- IAM Roles for Service Accounts (IRSA), EKS Pod Identity, Kubernetes RBAC
- aws-auth ConfigMap, Access Entries, Network Policies
- Secrets Management (Secrets Manager, External Secrets Operator)
- Troubleshooting: Pending pods, CrashLoopBackOff, OOMKilled, Node Not Ready, DNS failures, API server issues, IP exhaustion
- **Q&A:** 250+ | **Diagrams:** 20+ | **Scenarios:** 30+

### 6. Containers & Docker (Section 6)
- Docker architecture (client-server model)
- Storage drivers (overlay2, devicemapper), namespaces, cgroups
- OverlayFS, OCI (Open Container Initiative)
- Image layers, registries (ECR), Docker networking
- ECS (Fargate vs EC2 launch type), capacity providers
- Container internals, runtime architecture
- Troubleshooting: Image pull errors, network issues, resource constraints
- **Examples:** Dockerfile, docker-compose, ECR push/pull

### 7. Terraform for AWS (Section 7)
- State management and remote backends (S3 + DynamoDB)
- State locking, state migration, backup strategies
- Modules, workspaces, local vs remote state
- Lifecycle blocks (create_before_destroy, ignore_changes)
- Terraform Cloud/Enterprise features
- Terraform internals: plan generation, dependency graph, apply order
- Comparison: Terraform vs CloudFormation vs CDK
- **Examples:** 10+ Terraform modules with best practices
- **Anti-patterns:** What not to do in Terraform

### 8. AWS CI/CD (Section 8)
- CodeCommit (Git repository), CodeBuild (build environment)
- CodeDeploy (EC2, on-premises, Lambda), CodePipeline orchestration
- CodeArtifact (artifact repository)
- Deployment strategies: Blue/Green, Canary, Rolling, All-at-once
- Source → Build → Test → Deploy pipeline
- Security practices, multi-stage pipelines, approval gates
- Comparison with GitHub Actions, Jenkins

### 9. GitHub Actions (Section 9)
- Runners (GitHub-hosted, self-hosted), run container images
- OIDC federation with AWS (temporary credentials, no long-lived secrets)
- Reusable workflows, matrix builds, conditional steps
- Security hardening (secrets, environment isolation)
- Comparison: GitHub Actions vs AWS CodePipeline (when to use each)
- **Examples:** Deploy to EKS, run Terraform plans

### 10. Observability & SRE (Section 10)
- CloudWatch: Metrics, Logs, Alarms, Insights, Dashboards
- CloudTrail: API logging, compliance, forensics
- AWS Config: Compliance checking, configuration snapshots
- X-Ray: Distributed tracing, service maps, error analysis
- Amazon Managed Prometheus (AMP), Amazon Managed Grafana (AMG)
- OpenTelemetry integration, custom metrics
- SRE concepts: SLI (Service Level Indicator), SLO (Objective), SLA (Agreement)
- Error budgets, incident management, RCA (Root Cause Analysis)
- RED method (Rate, Errors, Duration), USE method (Utilization, Saturation, Errors)
- **Examples:** CloudWatch Insights queries, alarm configurations

### 11. AWS Security (Section 11)
- GuardDuty (threat detection), Amazon Inspector (vulnerability scanning)
- Security Hub (centralized findings), Amazon Macie (data discovery)
- AWS WAF (web application firewall), AWS Shield (DDoS protection)
- AWS KMS (key management), CloudHSM (hardware security module)
- Secrets Manager (secret rotation), Parameter Store (config management)
- Certificate Manager (ACM), SSL/TLS certificate lifecycle
- Encryption at rest (S3, RDS, EBS with KMS) and in transit (TLS)
- Customer Managed Keys (CMK), envelope encryption, key rotation
- Network security (VPC isolation, security groups, NACLs)
- Threat modeling, security architecture patterns
- **Examples:** KMS policy, bucket policies, IAM trust relationships

### 12. AWS Databases (Section 12)
- **RDS:** Multi-AZ (synchronous replication, automatic failover), Read Replicas (async, cross-region)
- Aurora: Cluster architecture, distributed storage layer, backtrack feature
- DynamoDB: Partition keys, sort keys, GSI/LSI, eventual vs strong consistency
- Capacity modes (provisioned vs on-demand), throttling, hot partitions
- ElastiCache (Redis, Memcached): Cache-aside, write-through, write-behind patterns
- Redshift: Columnar storage, RA3 nodes, concurrency scaling
- **Examples:** Terraform for RDS, DynamoDB, ElastiCache
- **Troubleshooting:** Failover behavior, replication lag, throttling

### 13. Event-Driven Architecture (Section 13)
- **SNS** (pub-sub): Topics, subscriptions, fan-out patterns
- **SQS** (queuing): Standard vs FIFO, visibility timeout, DLQ
- **EventBridge** (event routing): Rules, pattern matching, cross-account events
- **Kinesis** (streaming): Data Streams vs Firehose, shards, consumer groups
- **MSK** (managed Kafka): Brokers, topics, partitions, Avro schemas
- **Step Functions**: State machines, error handling, retry policies
- Idempotency patterns, distributed transactions, DLQ strategies
- **Examples:** Lambda processing Kinesis records, EventBridge cross-region events

### 14. System Design Using AWS (Section 14)
- **Netflix-Scale Video Streaming:** Capacity estimation (10M concurrent), CDN caching, multi-region architecture
- **Uber Ride-Sharing:** Real-time matching, geospatial queries, surge pricing
- **Global E-Commerce Platform:** Multi-region, inventory, payments, fulfillment
- **Multi-Region EKS:** Global deployment, failover, consistency
- **Real-Time Observability Platform:** Metrics collection, storage, querying at scale
- Each design includes: requirements, capacity table, architecture diagram, request flows, failure scenarios, cost analysis, 5+ interviewer follow-ups

### 15. AWS Troubleshooting Masterclass (Section 15)
- **Framework:** 5-step diagnostic approach (clarify → scope → timeline → hypothesize → verify)
- **Scenarios:**
  - VPC endpoint connection timeout (route table, security group, S3 policy)
  - ALB targets healthy but traffic gets 502 errors (memory leak, connection pool exhaustion)
  - EC2 instance "running" but no traffic (application crashed on startup)
  - RDS CPU spike and slow queries (missing index, lock contention)
  - DynamoDB throttling during peak load (hot partition, insufficient capacity)
  - EKS pod stuck in Pending (resource constraints, node pressure)
  - Lambda timeout and incomplete execution (cold start, code efficiency)
  - Payment processing failures (transient network, retry backoff)
- Each scenario includes: symptom, investigation steps, decision tree, root cause ranking, fix, validation, prevention
- **Commands:** 50+ AWS CLI and kubectl debugging commands

### 16. FAANG Interview Rounds (Section 16)
- **Recruiter round:** Behavioral, career goals, company fit
- **Hiring manager round:** Technical depth, leadership potential
- **Technical screening:** Problem-solving, coding/architecture basics
- **AWS deep dive:** EKS, RDS, DynamoDB internals, troubleshooting
- **Kubernetes deep dive:** Pod scheduling, networking, storage
- **System design round:** Design a large-scale system in 60 minutes
- **Troubleshooting round:** Debug production issues, explain fix
- **Leadership round:** Mentoring, ownership, conflict resolution
- **Behavioral round:** STAR stories aligned with company values

### 17. Behavioral & Leadership (Section 17)
- **STAR Method:** Situation → Task → Action → Result structure
- **Amazon Leadership Principles:** 14 principles with examples
- **Scenarios:**
  - Major production outage (ownership, bias for action, dive deep)
  - Disagreement with manager (respectfully disagree and commit, have backbone)
  - Mentoring someone (hire and develop the best, customer obsession)
  - Technical failure (are right a lot, learn and be curious)
  - Building something from scratch (invent and simplify, think big)
- Each scenario includes: strong answer (5 min), weak answer, what interviewer is testing, follow-ups
- Tips for sounding authentic, admitting uncertainty, asking good questions

### 18. Hands-On Labs (Section 18)
- **Beginner Labs (2–3 hours):**
  - Deploy a 3-tier app to ECS Fargate
  - Build a Lambda-based REST API with DynamoDB
- **Intermediate Labs (5–8 hours):**
  - Multi-region EKS cluster with Route53 failover
  - Real-time canary deployment with GitOps (Flux, Flagger)
- **Advanced Labs (10+ hours):**
  - Distributed metrics collection system (Prometheus, Grafana)
  - Production-ready ML pipeline (SageMaker, A/B testing)
- All labs include: step-by-step Terraform/CLI commands, expected output, troubleshooting tips

---

## Common Mistakes to Avoid

❌ **Don't memorize answers.** Understand the mechanism instead. Interviewers test thinking, not memory.

❌ **Don't skip diagrams.** Draw architecture before reading the section. Forces deeper understanding.

❌ **Don't ignore trade-offs.** Every AWS service has trade-offs. Know them cold.

❌ **Don't focus only on theory.** Do labs. Write Terraform. Run commands. Hands-on experience matters.

❌ **Don't rush system design.** Take 5 minutes to clarify requirements. Capacity estimation proves you think at scale.

❌ **Don't practice alone.** Do mock interviews. Record yourself. Ask for feedback.

❌ **Don't forget observability.** Interviewers ask: "How do you know if this is working? How do you debug?"

❌ **Don't ignore cost.** FAANG companies care about efficiency. Know the cost impact of your design.

---

## Key Metrics to Memorize

- **Availability SLA:** 99.9% = 43 min/month downtime | 99.99% = 4 min/month | 99.999% = 26 sec/month
- **RTO/RPO:** RTO = Recovery Time Objective (downtime duration) | RPO = Recovery Point Objective (data loss)
- **Consistency:** Strong = latest write guaranteed | Eventual = might return stale (~1ms lag)
- **AWS Global Architecture:** 31 regions, 99 AZs, 400+ edge locations, 13 wavelength zones
- **DynamoDB:** 1 WCU = 1KB write/sec | 1 RCU = 4KB eventual read/sec | 2KB strong read/sec
- **RDS:** Multi-AZ sync latency ~5–10ms | Failover ~60–120s | Read replica async latency ~100ms
- **S3 Durability:** 99.999999999% (11 nines) = 1 object lost per 100,000,000 stored objects
- **CloudFront:** 350+ edge locations, cache hit ratio 90% typical
- **Lambda:** Cold start 100–1000ms, concurrent execution limit default 1000
- **EKS Pod Network Latency:** <1ms (same AZ) | 5–10ms (cross-AZ)

---

## How to Get Maximum Value

### If You Have 1 Month
1. Read Sections 1–3 (Fundamentals, IAM, Networking)
2. Read Sections 6 (EKS Deep Dive) — most relevant for DevOps roles
3. Practice 100 Q&A from all sections
4. Do 1 system design (Netflix)
5. Record 5 STAR behavioral answers

### If You Have 3 Months (Recommended)
1. Follow the 3-month plan above
2. Complete all 14 sections
3. Solve 500+ Q&A
4. Do 5 system designs (practice timed)
5. Complete 2 hands-on labs
6. Do 2 mock interviews

### If You Have 6 Months (Ideal)
1. Follow the 6-month plan above
2. Deep-dive into each section
3. Solve all Q&A multiple times
4. Do all troubleshooting scenarios
5. Complete 4 hands-on labs (different difficulty levels)
6. Do 5+ mock interviews

---

## Tips for Success

✅ **Start with fundamentals.** Don't jump to EKS if you don't understand IAM/Networking first.

✅ **Understand, don't memorize.** If you can explain "why," you'll remember "what."

✅ **Test your knowledge.** Answer questions before reading solutions. Note gaps.

✅ **Use AWS documentation.** Cross-reference everything. Become familiar with AWS docs structure.

✅ **Draw diagrams.** Architecture diagrams are worth 1000 words. Practice drawing from memory.

✅ **Practice out loud.** Record yourself. Watch/listen for clarity, pacing, confidence.

✅ **Get feedback.** Mock interviews with peers or mentors. Ask: "What could I improve?"

✅ **Stay current.** AWS launches new services/features constantly. Check release notes monthly.

✅ **Build projects.** Use Terraform to build things. Deploy to EKS. Run lambdas. Real experience matters.

✅ **Connect concepts.** How does EKS networking relate to VPC? How does IAM relate to Lambda execution? See the big picture.

---

## Contact & Contributions

This guide was created from first principles to address gaps in existing interview prep materials. It focuses on **depth**, **mechanisms**, and **real-world scenarios** rather than surface-level definitions.

**Contributing:** If you find errors or want to add content, please open an issue or submit a PR. This is a living document—it should evolve with AWS and interview questions.

**Last Updated:** August 25, 2026

---

## Quick Reference Links

- **AWS Documentation:** https://docs.aws.amazon.com/
- **AWS Architecture Center:** https://aws.amazon.com/architecture/
- **AWS Well-Architected Framework:** https://docs.aws.amazon.com/wellarchitected/
- **AWS Skill Builder:** https://skillbuilder.aws.com/
- **AWS Workshops:** https://workshops.aws/
- **Kubernetes Documentation:** https://kubernetes.io/docs/
- **Terraform AWS Provider:** https://registry.terraform.io/providers/hashicorp/aws/latest/docs
- **Amazon Leadership Principles:** https://www.amazon.jobs/en/principles
- **STAR Interview Method:** https://www.verywell.com/what-is-the-star-interview-response-technique-2061629

---

**You've got this! 🚀 Focus on understanding, not memorizing. Think about trade-offs. Ask good questions. Show your reasoning. Good luck with your interviews!**
