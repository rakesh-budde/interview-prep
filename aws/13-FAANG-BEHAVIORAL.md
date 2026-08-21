# Sections 17–18: FAANG Interview Rounds · Behavioral & Leadership

> Part of the [AWS Interview Preparation Roadmap](./README.md). Covers **Section 17: FAANG Interview Round Preparation** and **Section 18: Behavioral & Leadership (STAR)**.

---

# SECTION 17: FAANG INTERVIEW ROUND PREPARATION

## 17.1 The Rounds (what each screens for)

| Round | Screens for | How to win it |
|-------|-------------|---------------|
| **Recruiter** | Fit, motivation, comp/logistics | Crisp story, quantified impact, clear role interest |
| **Hiring Manager** | Scope, ownership, team fit | Business impact, collaboration, growth trajectory |
| **Technical Screening** | Core AWS/Linux/scripting | Explain *why*, not just *what*; think aloud |
| **AWS Deep Dive** | Service internals & trade-offs | Whiteboard internals (IAM eval, VPC packet flow, S3 durability) |
| **Kubernetes Deep Dive** | EKS/K8s mastery | Control/data plane, IRSA, CNI, troubleshooting flow |
| **System Design** | Scalable architecture | Requirements→estimate→design→scale→failure |
| **Troubleshooting** | Debugging methodology | "What changed?" + hypothesis tree + commands |
| **Leadership** | Influence, ownership | STAR with metrics; drove outcomes across teams |
| **Behavioral** | Values/culture fit | STAR, honest, reflective, "what I'd do differently" |

## 17.2 Interviewer Expectations by Level

- **Mid (3–4 yrs):** solid fundamentals, can operate services, debug with guidance.
- **Senior (5–7 yrs):** designs systems, owns reliability, mentors, reasons about trade-offs and cost.
- **Staff+ (8+):** platform-level thinking, org impact, sets standards, drives cross-team initiatives.

At 5 years targeting FAANG, aim for **Senior**: depth in EKS/networking/IAM, clear trade-off reasoning, quantified production impact, and calm structured problem-solving.

## 17.3 Top 100 Frequently Asked Questions (grouped)

**Fundamentals (1–12):** Region vs AZ; Shared Responsibility; why multi-account; SCP behavior; Well-Architected pillars; static stability; eventual consistency; ARN structure; Control Tower; tagging strategy; Service Quotas; SigV4.

**IAM/Security (13–28):** user vs role; cross-account access; permission boundary; policy evaluation order; IRSA vs Pod Identity; STS AssumeRole; ABAC; confused deputy/ExternalId; envelope encryption; KMS key policy vs IAM; Secrets Manager vs Parameter Store; MFA enforcement; least privilege in practice; GuardDuty vs Inspector vs Macie; WAF vs Shield; data perimeter.

**Networking (29–48):** SG vs NACL; public vs private subnet; per-AZ NAT; VPC peering transitivity; gateway vs interface endpoint; ALB vs NLB vs GWLB; Route 53 failover; CloudFront vs Global Accelerator; Transit Gateway; PrivateLink internals; hybrid DNS; VPC CNI IP assignment; centralized egress; Direct Connect vs VPN; ephemeral port NACL pitfall; packet flow user→pod.

**Compute/Containers (49–64):** EC2 vs Lambda vs Fargate; Spot strategy; ASG scaling; placement groups; Nitro; IMDSv2; Graviton; container vs VM; namespaces/cgroups; multi-stage builds; ECS vs EKS; capacity providers; Karpenter vs Cluster Autoscaler; HPA vs VPA; PodDisruptionBudget; rolling vs blue/green vs canary.

**Storage/DB (65–80):** S3 storage classes; 11 nines; S3 consistency; lifecycle; EBS vs EFS vs S3; Multi-AZ vs read replica; Aurora storage/quorum; DynamoDB hot partition; DynamoDB consistency; Global Tables; ElastiCache patterns; CAP/PACELC; RDS Proxy; sharding; Redshift vs Athena; encryption at rest.

**CI/CD & IaC (81–90):** pipeline stages; deployment strategies; auto-rollback; OIDC keyless deploy; Terraform state/locking; count vs for_each; drift; Terraform vs CloudFormation vs CDK; secrets in CI; supply-chain hardening.

**Observability/SRE (91–100):** SLI/SLO/SLA; error budget; three pillars; RED vs USE; burn-rate alerting; symptom vs cause alerts; distributed tracing; incident command; blameless postmortem; reduce alert fatigue.

## 17.4 Strong vs Weak Answer Examples

**Q: "How do you give an app on EKS access to S3?"**
- **Weak:** *"Put the AWS keys in a Kubernetes secret and mount them."* (Long-lived creds, blast radius, no rotation — red flag.)
- **Strong:** *"IRSA or Pod Identity — a scoped IAM role bound to the pod's ServiceAccount, short-lived creds via `AssumeRoleWithWebIdentity`, no static secrets, auditable in CloudTrail."*

**Q: "Design a URL shortener."**
- **Weak:** jumps to tech, no requirements/estimation, single-server DB.
- **Strong:** clarifies scale/latency, estimates QPS/storage, DynamoDB + base62 keys + CloudFront cache, discusses hot keys, multi-AZ, and cost.

**Q: "Tell me about an outage you caused."**
- **Weak:** blames others / vague / no lesson.
- **Strong:** STAR, owns the mistake, quantifies impact, describes mitigation + systemic prevention + what changed.

## 17.5 Mock Interview Plan

- **Weeks 1–2:** technical screening + AWS deep dive (record yourself; refine).
- **Weeks 3–4:** 2 system designs/week timed to 40 min; get feedback.
- **Week 5:** EKS deep dive + troubleshooting live-debug drills.
- **Week 6:** behavioral/leadership STAR polish + full loop simulation.

## 17.6 Day-of Tactics

- Think aloud; state assumptions; manage time; drive the conversation.
- For design: requirements → estimate → draw → deep-dive the hard part → failure modes → cost.
- For troubleshooting: "what changed?", hypothesis tree, cheapest signal first.
- Ask clarifying questions; it's a dialogue, not a monologue.

---

# SECTION 18: BEHAVIORAL & LEADERSHIP (STAR)

## 18.1 The STAR Method

- **Situation:** context (brief).
- **Task:** your responsibility/goal.
- **Action:** what **you** did (most detail; use "I").
- **Result:** quantified outcome + what you learned.

Keep it ~2 minutes. Lead with impact. Be honest, reflective, specific.

## 18.2 Amazon Leadership Principles Map

| Principle | Story theme to prepare |
|-----------|------------------------|
| Customer Obsession | Fixed a customer-impacting reliability issue |
| Ownership | Took on something outside your remit |
| Invent & Simplify | Automated a painful manual process |
| Are Right, A Lot | Data-driven decision that paid off |
| Learn & Be Curious | Learned a new tech under pressure |
| Hire & Develop | Mentored an engineer to promotion |
| Insist on Highest Standards | Refused to ship something unsafe |
| Bias for Action | Made a reversible call fast in an incident |
| Frugality | Big cost optimization |
| Earn Trust | Owned a mistake transparently |
| Dive Deep | Root-caused a gnarly bug |
| Have Backbone; Disagree & Commit | Pushed back, then committed |
| Deliver Results | Delivered under a hard deadline |

## 18.3 Ready-to-Use STAR Stories

### 1. Major Production Outage (Ownership, Dive Deep, Earn Trust)
- **S:** Peak-hours outage; checkout API returning 5xx, revenue impact.
- **T:** I was on call and took Incident Commander role.
- **A:** Declared incident, checked recent changes (a deploy 20 min earlier), correlated with a spike in RDS connections from a new Lambda. Rolled back the deploy, added RDS Proxy as immediate mitigation, communicated status every 10 min.
- **R:** Restored service in ~25 min; wrote a blameless postmortem; added connection pooling + a canary + a burn-rate alarm. **Zero recurrence**; MTTR for similar issues dropped ~60%.
- **Lesson:** "What changed?" first; make mitigations systemic, not one-off.

### 2. Failed Deployment (Bias for Action, Highest Standards)
- **S:** A canary showed elevated latency during a prod rollout.
- **T:** Decide fast: continue or roll back.
- **A:** Trusted the SLO alarm, auto-rolled back, then reproduced in staging — a missing DB index. Added the index, backfilled, redeployed with canary.
- **R:** No customer-visible impact beyond the 3% canary; added index checks to CI.
- **Lesson:** reversible + fast beats hopeful; guardrails catch what reviews miss.

### 3. Conflict Resolution (Earn Trust, Backbone)
- **S:** Dev team wanted direct prod access; I owned security guardrails.
- **T:** Balance velocity vs least privilege.
- **A:** Listened to their pain (slow deploys), proposed SSM Session Manager + a self-service pipeline with break-glass instead of standing access. Ran a POC together.
- **R:** Deploys sped up, standing prod access removed, audit improved. Both teams bought in.
- **Lesson:** solve the underlying need, not the literal ask.

### 4. Technical Leadership (Invent & Simplify, Deliver Results)
- **S:** Fragmented Terraform, snowflake accounts, slow onboarding.
- **T:** Standardize infra delivery.
- **A:** Built a versioned module library + landing-zone factory + OIDC CI with policy-as-code; migrated teams incrementally.
- **R:** New-service infra from days → hours; drift incidents down sharply; consistent security baseline.
- **Lesson:** platforms scale teams; adoption needs migration paths, not mandates.

### 5. Mentoring (Hire & Develop)
- **S:** A junior engineer struggling with Kubernetes.
- **A:** Paired weekly, gave a scoped EKS project (IRSA + Karpenter), code reviews focused on reasoning.
- **R:** They led the next cluster upgrade and were promoted.
- **Lesson:** delegate real ownership with a safety net.

### 6. Cost Optimization (Frugality)
- **S:** Cloud bill trending 30% over budget.
- **A:** Compute Optimizer + Graviton migration, Savings Plans for baseline, Spot for stateless/batch, gp3, S3 lifecycle, killed idle resources, cross-AZ traffic reduction.
- **R:** ~35% reduction (six figures/yr) with no reliability regression.
- **Lesson:** measure first; tie every cut to a reliability check.

### 7. Security Incident (Customer Obsession, Dive Deep)
- **S:** GuardDuty flagged anomalous API calls from a CI credential.
- **A:** Disabled the key, rotated secrets, reviewed CloudTrail for blast radius, moved CI to OIDC short-lived roles, added SCP guardrails.
- **R:** Contained quickly, no data loss; eliminated the class of issue (no static keys).
- **Lesson:** contain fast, then remove the root cause category.

### 8. Migration Project (Deliver Results)
- **S:** Lift-and-shift EC2 monolith → EKS microservices.
- **A:** Strangler pattern, per-service pipelines, observability first, gradual traffic shift via ALB weights.
- **R:** Deploy frequency up 5×, MTTR down, autoscaling cut cost; zero big-bang risk.
- **Lesson:** incremental migration + observability beats big-bang.

### 9. Automation Initiative (Invent & Simplify)
- **S:** Manual, error-prone on-call toil for common alerts.
- **A:** Built EventBridge→Lambda auto-remediations + runbooks-as-code + self-healing for known failures.
- **R:** ~40% fewer pages, faster remediation, happier on-call.
- **Lesson:** automate the top 3 toil sources first.

## 18.4 Common Behavioral Questions

- Tell me about a time you failed / disagreed with your manager / missed a deadline.
- Your biggest technical achievement / hardest bug / most impactful project.
- A time you influenced without authority / handled ambiguity / made a risky call.
- How do you prioritize under conflicting demands?

## 18.5 Do / Don't

- **Do:** quantify, use "I", show reflection, keep it ~2 min, be honest.
- **Don't:** blame, ramble, be vague, claim zero mistakes, use "we" for your own work.

## 18.6 Documentation / Resources

- Amazon Leadership Principles: https://www.amazon.jobs/content/en/our-workplace/leadership-principles
- Google SRE (postmortem culture): https://sre.google/books/
- STAR method overview: https://www.themuse.com/advice/star-interview-method

---

> Next: **[Sections 19–20 — Hands-On Labs & Documentation Index](./14-HANDS-ON-LABS-DOCS.md)**.
