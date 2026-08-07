# AWS Interview Prep - Learning Resources & Study Plans

> 90-Day roadmap, certification path, top questions, and essential resources

---

## 90-Day Interview Preparation Roadmap

### Month 1: AWS Fundamentals + Specialization (Weeks 1-4)

**Week 1: AWS Foundations**
```
Goal: Understand AWS architecture and global infrastructure

Monday-Tuesday: AWS Fundamentals
├─ Regions, AZs, edge locations
├─ Service categories (compute, storage, database, networking)
└─ Shared responsibility model

Wednesday-Thursday: Core Services Deep Dive
├─ EC2 (instances, types, lifecycle)
├─ VPC (subnets, routing, security)
└─ S3 (basic operations, storage classes)

Friday: Practice & Review
├─ Create 3 AWS resources manually (EC2, VPC, S3)
├─ Write terraform for same resources
└─ Document what you learned

Time: 25 hours
Completion check: Explain AWS architecture to non-technical person
```

**Week 2: Networking (Your Specialization)**
```
Goal: Master VPC design for FAANG interviews

Monday-Tuesday: VPC Deep Dive
├─ CIDR planning (5+ scenarios)
├─ Multi-AZ subnet design
├─ Route table evaluation logic

Wednesday-Thursday: Advanced Networking
├─ VPC peering (cross-account, cross-region)
├─ Transit Gateway (hub-and-spoke)
├─ Route53 failover patterns

Friday: Hands-on Lab
├─ Build 2-region VPC with Transit Gateway
├─ Test failover scenarios
├─ Document architecture diagrams

Time: 25 hours
Resource: Use [NETWORKING.md](NETWORKING.md)
Completion check: Design multi-region VPC from scratch
```

**Week 3: Security & IAM**
```
Goal: Master identity and security patterns

Monday-Tuesday: IAM Deep Dive
├─ Policy evaluation logic (deny wins)
├─ Cross-account access patterns
├─ Permission boundaries & SCPs

Wednesday-Thursday: Encryption & Secrets
├─ KMS (customer managed keys, encryption)
├─ Secrets Manager (rotation, multi-region)
├─ TLS/SSL (ACM certificates)

Friday: Lab & Review
├─ Create cross-account role with external ID
├─ Implement KMS envelope encryption
├─ Set up Secrets rotation

Time: 20 hours
Resource: Use [IAM-SECURITY.md](IAM-SECURITY.md)
Completion check: Design multi-tenant SaaS security model
```

**Week 4: Compute & Containers**
```
Goal: Understand EC2, ECS, and EKS deeply

Monday-Tuesday: EC2 Mastery
├─ Instance lifecycle state machine
├─ EBS volume types (when to use each)
├─ Auto Scaling Groups with mixed instances
├─ Spot instances + interruption handling

Wednesday: ECS Fundamentals
├─ Task definition vs service
├─ Fargate vs EC2 launch type
├─ ECS Anywhere for hybrid

Thursday-Friday: EKS Deep Dive (2 days)
├─ Control plane vs data plane
├─ VPC CNI networking
├─ Pod Identity (new) vs IRSA (old)
├─ Karpenter vs Cluster Autoscaler

Time: 25 hours
Resource: Use [EC2-CONTAINERS.md](EC2-CONTAINERS.md)
Completion check: Design Kubernetes cluster for production load

**Month 1 Summary:**
- Core AWS services understood
- Can design multi-region networks
- Can architect secure multi-tenant systems
- Can deploy containerized apps
- Can troubleshoot networking issues
```

### Month 2: Data, DevOps, & Design (Weeks 5-8)

**Week 5: Databases & Storage**
```
Goal: Master storage selection and database patterns

Monday-Tuesday: Relational Databases
├─ RDS Multi-AZ (failover, timing)
├─ Aurora architecture (global database)
├─ Read replicas vs multi-AZ

Wednesday-Thursday: NoSQL & Cache
├─ DynamoDB (partitioning, scaling, streams)
├─ ElastiCache (Redis vs Memcached)
├─ When to choose each

Friday: S3 Deep Dive + Lab
├─ S3 internals (11 nines durability)
├─ Lifecycle policies
├─ Cross-region replication
├─ Hands-on: Design tiered storage

Time: 20 hours
Resource: Use [STORAGE-DATABASES.md](STORAGE-DATABASES.md)
Completion check: Design database for 1M users, 100K QPS

**Week 6: CI/CD & Infrastructure**
```
Goal: Master deployment pipelines and IaC

Monday-Tuesday: CI/CD Pipelines
├─ CodePipeline stages
├─ Deployment strategies (blue-green, canary, rolling)
├─ Manual approval gates

Wednesday: Infrastructure as Code
├─ CloudFormation (templates, stacks)
├─ CDK (Python/TypeScript)
├─ Terraform (state, modules)

Thursday-Friday: GitOps & Advanced
├─ ArgoCD (declarative deployments)
├─ Drift detection
├─ Multi-environment strategies

Time: 20 hours
Resource: Use [CI-CD-INFRASTRUCTURE.md](CI-CD-INFRASTRUCTURE.md)
Completion check: Build full CodePipeline from code to production
```

**Week 7: Monitoring & Serverless**
```
Goal: Understand observability and serverless architecture

Monday-Tuesday: CloudWatch & Monitoring
├─ Metrics, logs, alarms
├─ CloudWatch Insights queries
├─ SLI/SLO/SLA concepts

Wednesday: X-Ray & Tracing
├─ Distributed tracing patterns
├─ Service maps
├─ Performance analysis

Thursday-Friday: Serverless Architecture
├─ Lambda functions (optimization, concurrency)
├─ API Gateway patterns
├─ EventBridge routing
├─ SQS, SNS, Kinesis comparison

Time: 20 hours
Resource: Use [MONITORING-SERVERLESS.md](MONITORING-SERVERLESS.md)
Completion check: Build event-driven microservices
```

**Week 8: System Design Interviews**
```
Goal: Practice large-scale architecture design

Monday-Tuesday: Design Framework
├─ Capacity estimation
├─ Component selection
├─ Trade-off analysis
└─ Practice: Netflix streaming (45 min)

Wednesday-Thursday: 2 Design Scenarios
├─ Scenario 1: Real-time notifications (50M users)
├─ Scenario 2: E-commerce platform (1M concurrent)

Friday: Mock Interview
├─ Do complete 60-minute design interview
├─ Have colleague evaluate
├─ Record and review

Time: 20 hours
Resource: Use [SYSTEM-DESIGN-TROUBLESHOOTING.md](SYSTEM-DESIGN-TROUBLESHOOTING.md)
Completion check: Design any system from scratch in 45 minutes
```

### Month 3: Troubleshooting, Behavioral, & Practice (Weeks 9-12)

**Week 9: Production Troubleshooting**
```
Goal: Know how to debug real production issues

Monday-Tuesday: Networking Troubleshooting
├─ "Pods can't reach database" → diagnosis
├─ "VPN connection drops" → root cause
├─ "DNS not resolving" → debugging

Wednesday-Thursday: Application Troubleshooting
├─ "Lambda timeout in production"
├─ "DynamoDB throttled"
├─ "RDS CPU at 100%"

Friday: Lab
├─ Simulate 10 production failures
├─ Debug each one (30 min per issue)
├─ Document root causes

Time: 20 hours
Resource: Use [SYSTEM-DESIGN-TROUBLESHOOTING.md](SYSTEM-DESIGN-TROUBLESHOOTING.md)
Completion check: Debug any production issue in 15 minutes
```

**Week 10: Behavioral & Leadership**
```
Goal: Prepare STAR format answers and leadership stories

Monday-Tuesday: Craft Your Stories
├─ 5 "tell me about a time" stories
├─ 3 "describe a decision" stories
├─ 2 "conflict resolution" stories

Wednesday-Thursday: Practice Out Loud
├─ Record yourself telling each story
├─ Time each story (should be 3-4 minutes)
├─ Practice under pressure (have friend interrupt)

Friday: Mock Behavioral Interview
├─ Full 1-hour behavioral interview
├─ Have someone interview you
├─ Get feedback

Time: 15 hours
Resource: Use [BEHAVIORAL-LEADERSHIP.md](BEHAVIORAL-LEADERSHIP.md)
Completion check: Tell compelling leadership story (timed)
```

**Week 11: Specialized Deep Dives**
```
Goal: Answer ANY follow-up question about any service

Pick YOUR weakest area and deep dive:

Option A: Kubernetes/EKS Deep Dive
├─ Read: Advanced EKS patterns
├─ Lab: Build prod-grade EKS cluster
├─ Practice: 10 EKS troubleshooting scenarios

Option B: Database Optimization
├─ Read: Advanced query tuning
├─ Lab: Optimize slow queries
├─ Practice: 10 database performance scenarios

Option C: Serverless at Scale
├─ Read: Lambda best practices
├─ Lab: Build event-driven system
├─ Practice: 10 serverless failure scenarios

Time: 20 hours
Completion check: Become expert in one area
```

**Week 12: Mock Interviews & Final Review**
```
Goal: Be ready for real interviews

Monday-Tuesday: System Design Mock (2x)
├─ Interview 1: Netflix streaming (60 min)
├─ Interview 2: Uber real-time (60 min)
├─ Get feedback on both

Wednesday-Thursday: Behavioral Mocks (2x)
├─ Interview 1: Leadership & decisions
├─ Interview 2: Conflict & learning
├─ Get feedback on storytelling

Friday: Technical Deep Dive Mock
├─ 60 minute technical interview
├─ Deep questions on any service
├─ Test breadth + depth

Final Review:
├─ Review all notes from Month 1-3
├─ Identify weak areas
├─ Do 1 final practice per weak area
└─ Rest day before real interview

Time: 20 hours
Completion check: Pass all mock interviews with 85%+ score
```

---

## AWS Certification Roadmap

### If interviewing at AWS (different path):
```
For AWS interviews specifically:

1. AWS Certified Solutions Architect Pro (4-8 weeks)
   ├─ Covers design, multi-region, disaster recovery
   ├─ Exam cost: $300
   └─ Useful for: Architecture design interviews

2. AWS Certified DevOps Engineer Professional (4-8 weeks)
   ├─ Covers CI/CD, IaC, monitoring
   ├─ Exam cost: $300
   └─ Useful for: DevOps engineer interviews

3. AWS Certified Security Specialty (2-4 weeks)
   ├─ Quick boost for security knowledge
   ├─ Exam cost: $300
   └─ Not required, but nice to have

Recommendation:
├─ If pure AWS job: Pursue Solutions Architect Pro
├─ If DevOps job: Pursue DevOps Engineer Pro
├─ Don't need both for FAANG (they care about skills, not cert)
└─ Certifications are nice but not required
```

### For non-AWS jobs (FAANG general):
```
Certifications NOT required for Google/Meta/Netflix interviews
├─ They test skills directly
├─ Certifications don't guarantee skills
├─ Some teams might not even care

But if you want:
├─ AWS Solutions Architect Pro (most recognized)
├─ Study while doing this guide (kills 2 birds)
└─ Exam in Month 3 (after all practice)
```

---

## Top 50 AWS Interview Questions

### Easy (Warm-up, 5 min each)

1. What's the difference between a Region and Availability Zone?
2. Explain the AWS Shared Responsibility Model.
3. What are the 5 pillars of the Well-Architected Framework?
4. What's the difference between a Security Group and a NACL?
5. What is VPC?

### Medium (10 min each)

6. Design a 2-region VPC with failover.
7. Explain IAM policy evaluation logic.
8. What's the difference between RDS and Aurora?
9. When would you use DynamoDB vs RDS?
10. How does Auto Scaling Group work?

### Hard (20+ min each, design questions)

11. Design Netflix streaming platform for 100M users.
12. Design real-time notification system for 50M concurrent users.
13. Design e-commerce order processing system.
14. Design multi-region active-active system.
15. Your RDS primary failed, what do you do? (Production troubleshooting)

### Expert (30+ min, full system design)

16-50. [See SYSTEM-DESIGN-TROUBLESHOOTING.md for 40+ detailed design questions]

---

## Essential AWS Resources

### Documentation (Free)
- AWS Architecture Center: https://aws.amazon.com/architecture/
- AWS Well-Architected Framework: https://aws.amazon.com/architecture/well-architected/
- EKS Best Practices Guide: https://aws.github.io/aws-eks-best-practices/
- AWS Security Best Practices: https://docs.aws.amazon.com/security/

### Whitepapers (Free, highly recommended)
1. "AWS Well-Architected Framework"
2. "AWS Security Best Practices"
3. "Building Secure and Scalable Applications"
4. "Disaster Recovery of Workloads on AWS"
5. "AWS Cloud Cost Optimization"
6. "Multi-Region Active-Active Applications"
7. "AWS Microservices on AWS"

### Courses (Paid, high value)
- A Cloud Guru: AWS DevOps Engineer certification course ($50-100)
- Linux Academy: Advanced AWS course ($50-100)
- Udemy: "Ultimate AWS Solutions Architect" ($20 during sales)

### Hands-on Labs
- AWS Workshops (free): https://workshops.aws/
- Terraform AWS examples: https://github.com/hashicorp/terraform-aws-examples
- EKS Workshop: https://www.eksworkshop.com/

### Blogs (Follow regularly)
- AWS Blog: https://aws.amazon.com/blogs/aws/
- Netflix Technology Blog: https://netflixtechblog.com/
- Cloudflare Blog: https://blog.cloudflare.com/
- Stripe Engineering Blog: https://stripe.com/blog/engineering

### Books
- "AWS Certified Solutions Architect Study Guide" ($40)
- "Infrastructure as Code" by Kief Morris ($40)
- "The Phoenix Project" (understand DevOps mindset) ($20)

---

## Daily Study Schedule (Final Month)

### Morning (1 hour)
- Review one topic from guides (25 min)
- Read 1 recent blog/whitepaper (20 min)
- Quick 10-question quiz (15 min)

### Midday (30 min)
- Create 1 AWS resource manually
- Repeat 3x (deploy, delete, redeploy)
- Time yourself (improve speed)

### Afternoon (1 hour)
- Solve 2 troubleshooting scenarios
- Deep dive on 1 AWS service
- Write answer from scratch (no peeking)

### Evening (1 hour)
- Practice STAR story
- Mock interview question
- Record yourself + review

### Weekend
- Full mock interview (2-3 hours)
- Review weak areas (1 hour)
- Hands-on lab project (2-3 hours)

**Total:** 15-20 hours/week

---

## Day Before Interview Checklist

- [ ] Review top 10 most common questions
- [ ] Practice 2 system design scenarios
- [ ] Tell 2 behavioral stories out loud
- [ ] Review weak areas one more time
- [ ] Get good sleep (8+ hours)
- [ ] Eat healthy breakfast
- [ ] Test video/audio setup (if virtual)
- [ ] Have water bottle ready
- [ ] Arrive 15 min early
- [ ] Deep breaths before interview (you got this!)

---

## Post-Interview

### Regardless of outcome:
1. **Send thank you email** (within 2 hours)
   - Mention specific discussion points
   - Show genuine interest
   - Keep door open for next round

2. **Debrief notes** (same day)
   - What questions were asked?
   - How did you do?
   - What would you improve?
   - Study those areas before next interview

3. **Keep learning**
   - Whether you get offer or not
   - These skills apply to other jobs
   - Keep practicing system design
   - Stay current with AWS services

**Good luck! 🚀**

