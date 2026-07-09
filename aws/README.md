# AWS Interview Questions - Complete Guide

> **500+ AWS Interview Questions for Senior DevOps Engineer, SRE, and Platform Engineer roles at FAANG companies**

---

## 📋 Table of Contents

- [Quick Reference](#quick-reference)
- [IAM & Security](#iam--security)
- [Networking](#networking)
- [Compute](#compute)
- [Storage](#storage)
- [Database](#database)
- [Containers & Orchestration](#containers--orchestration)
- [Serverless](#serverless)
- [Monitoring & Observability](#monitoring--observability)
- [Security Services](#security-services)
- [Multi-Account Strategy](#multi-account-strategy)
- [Architecture Patterns](#architecture-patterns)
- [Troubleshooting Scenarios](#troubleshooting-scenarios)
- [Design Questions](#design-questions)

---

## Quick Reference

### AWS Global Infrastructure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS GLOBAL INFRASTRUCTURE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │
│  │   Region    │    │   Region    │    │   Region    │             │
│  │  us-east-1  │    │  eu-west-1  │    │ ap-south-1  │             │
│  │             │    │             │    │             │             │
│  │  ┌───────┐  │    │  ┌───────┐  │    │  ┌───────┐  │             │
│  │  │ AZ-a  │  │    │  │ AZ-a  │  │    │  │ AZ-a  │  │             │
│  │  │ AZ-b  │  │    │  │ AZ-b  │  │    │  │ AZ-b  │  │             │
│  │  │ AZ-c  │  │    │  │ AZ-c  │  │    │  │ AZ-c  │  │             │
│  │  └───────┘  │    │  └───────┘  │    │  └───────┘  │             │
│  └─────────────┘    └─────────────┘    └─────────────┘             │
│                                                                      │
│  Edge Locations: 400+ Points of Presence                            │
│  Local Zones: For latency-sensitive workloads                       │
│  Wavelength: For 5G edge computing                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Services by Category

| Category | Core Services |
|----------|---------------|
| Identity | IAM, Organizations, SSO, Control Tower |
| Networking | VPC, Route53, Direct Connect, Transit Gateway |
| Compute | EC2, Lambda, ECS, EKS, Fargate |
| Storage | S3, EBS, EFS, FSx, Glacier |
| Database | RDS, Aurora, DynamoDB, ElastiCache |
| Security | KMS, WAF, Shield, Secrets Manager |
| Monitoring | CloudWatch, CloudTrail, X-Ray |

---

## IAM & Security

### 🟢 Basic Questions

#### Q1: What is AWS IAM and what are its main components?

**Basic Answer:**
AWS Identity and Access Management (IAM) is a service that helps you securely control access to AWS resources. Main components include Users, Groups, Roles, and Policies.

**Advanced Answer:**
IAM provides fine-grained access control through:
- **Users**: Individual identities with long-term credentials
- **Groups**: Collections of users with shared permissions
- **Roles**: Temporary credentials for trusted entities (services, users, applications)
- **Policies**: JSON documents defining permissions (identity-based, resource-based, permission boundaries, SCPs)

Key features:
- Supports federation (SAML 2.0, OIDC)
- Integrates with AWS Organizations for centralized management
- Provides IAM Access Analyzer for policy validation
- Supports MFA for enhanced security

**Expert Answer:**
IAM operates on an eventual consistency model due to its global, distributed nature. Understanding this is crucial for:
- Policy propagation delays (typically seconds, but can be minutes)
- Cross-region operations require waiting for replication
- IAM uses signature version 4 for API authentication

Architecture considerations:
```
┌─────────────────────────────────────────────────────────────────┐
│                     IAM EVALUATION LOGIC                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Request → Explicit Deny? → YES → DENY                          │
│              │                                                   │
│              NO                                                  │
│              ↓                                                   │
│         SCP Allow? → NO → DENY                                   │
│              │                                                   │
│              YES                                                 │
│              ↓                                                   │
│    Permission Boundary Allow? → NO → DENY                        │
│              │                                                   │
│              YES                                                 │
│              ↓                                                   │
│    Identity/Resource Policy Allow? → NO → DENY                   │
│              │                                                   │
│              YES → ALLOW                                         │
└─────────────────────────────────────────────────────────────────┘
```

**Interview Tips:**
- Always mention the principle of least privilege
- Discuss IAM best practices: no root user for daily tasks, MFA everywhere
- Know the difference between identity-based and resource-based policies

**Follow-up Questions:**
- How does IAM policy evaluation work when multiple policies apply?
- What's the difference between IAM roles and users for cross-account access?
- How would you implement temporary elevated access?

---

#### Q2: Explain the difference between IAM Roles and Users. When would you use each?

**Basic Answer:**
Users have permanent credentials (username/password, access keys), while Roles provide temporary credentials. Use Users for individuals, Roles for services and cross-account access.

**Advanced Answer:**

| Aspect | IAM Users | IAM Roles |
|--------|-----------|-----------|
| Credentials | Long-term (access keys, passwords) | Temporary (STS tokens) |
| Use Case | Human users, CI/CD with no role support | EC2, Lambda, cross-account, federation |
| Security | Requires key rotation | Auto-rotating credentials |
| Audit | Tied to specific identity | Tracked via assumed role session |

Best practices:
- Use Roles for all AWS service-to-service communication
- Use Roles for cross-account access
- Use Users only when Roles aren't possible
- Implement role chaining for complex scenarios

**Expert Answer:**
Role assumption mechanics:
```python
# Role assumption flow
1. Principal calls sts:AssumeRole
2. STS validates trust policy
3. STS returns temporary credentials:
   - AccessKeyId
   - SecretAccessKey
   - SessionToken
   - Expiration (15 min - 12 hours)

# Trust policy example
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "Service": "ec2.amazonaws.com",
            "AWS": "arn:aws:iam::123456789012:root"
        },
        "Action": "sts:AssumeRole",
        "Condition": {
            "StringEquals": {
                "sts:ExternalId": "UniqueExternalId"
            }
        }
    }]
}
```

Role chaining limitations:
- Maximum session duration is 1 hour when chaining
- CloudTrail logs show all assumed roles
- Role session names help with auditing

**Interview Tips:**
- Emphasize security benefits of temporary credentials
- Mention confused deputy problem and ExternalId
- Discuss instance profiles for EC2

---

#### Q3: What are Service Control Policies (SCPs) and how do they differ from IAM policies?

**Basic Answer:**
SCPs are policies attached to AWS Organizations OUs or accounts that define the maximum permissions. Unlike IAM policies, SCPs don't grant permissions—they only restrict what IAM policies can grant.

**Advanced Answer:**

| Aspect | IAM Policies | SCPs |
|--------|--------------|------|
| Scope | Users, Groups, Roles | Accounts, OUs |
| Function | Grant permissions | Set permission guardrails |
| Evaluation | After SCPs | Before IAM policies |
| Management Account | Full effect | No effect |

SCP Use Cases:
1. Prevent disabling CloudTrail
2. Restrict regions
3. Require encryption
4. Prevent leaving organization

```json
// Example: Deny disabling CloudTrail
{
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "ProtectCloudTrail",
        "Effect": "Deny",
        "Action": [
            "cloudtrail:StopLogging",
            "cloudtrail:DeleteTrail"
        ],
        "Resource": "*"
    }]
}
```

**Expert Answer:**
SCP inheritance and effective permissions:
```
┌─────────────────────────────────────────────────────────────────┐
│                    SCP INHERITANCE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Root (Management Account) - SCPs don't apply                    │
│       │                                                          │
│       ├── OU: Production                                         │
│       │   │   SCP: DenyAllExceptApproved                        │
│       │   │                                                      │
│       │   ├── Account: Prod-1                                    │
│       │   │       Effective = Root ∩ OU-SCP ∩ Account-SCP       │
│       │   │                                                      │
│       │   └── Account: Prod-2                                    │
│       │           Effective = Root ∩ OU-SCP ∩ Account-SCP       │
│       │                                                          │
│       └── OU: Development                                        │
│           │   SCP: AllowAll                                      │
│           │                                                      │
│           └── Account: Dev-1                                     │
│                   More permissive                                │
└─────────────────────────────────────────────────────────────────┘
```

Key considerations:
- SCPs use intersection logic (most restrictive wins)
- Management account is exempt from SCPs
- Full AWS access SCP is attached by default
- SCPs don't grant permissions to the management account's root user

**Interview Tips:**
- Explain the relationship between SCPs and IAM policies
- Mention that SCPs affect all users including root (except management account)
- Discuss common SCP patterns for compliance

---

### 🟡 Intermediate Questions

#### Q4: How would you implement a secure cross-account access pattern?

**Basic Answer:**
Create an IAM role in the target account with a trust policy allowing the source account, then assume the role from the source account.

**Advanced Answer:**

**Pattern 1: Role Assumption**
```
Source Account (111111111111)     Target Account (222222222222)
┌─────────────────────────┐      ┌─────────────────────────┐
│                         │      │                         │
│  User/Role with policy: │      │  Role: CrossAccountRole │
│  sts:AssumeRole         │─────▶│  Trust: 111111111111    │
│                         │      │  Permissions: S3 Read   │
│                         │      │                         │
└─────────────────────────┘      └─────────────────────────┘
```

**Pattern 2: Resource-Based Policy**
```json
// S3 bucket policy allowing cross-account access
{
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "CrossAccountAccess",
        "Effect": "Allow",
        "Principal": {
            "AWS": "arn:aws:iam::111111111111:role/DataAccessRole"
        },
        "Action": ["s3:GetObject", "s3:ListBucket"],
        "Resource": [
            "arn:aws:s3:::my-bucket",
            "arn:aws:s3:::my-bucket/*"
        ]
    }]
}
```

**Expert Answer:**

Secure cross-account architecture:
```
┌─────────────────────────────────────────────────────────────────┐
│              CROSS-ACCOUNT ACCESS ARCHITECTURE                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Source Account                  Target Account                  │
│  ┌──────────────────┐           ┌──────────────────┐            │
│  │                  │           │                  │            │
│  │  ┌────────────┐  │           │  ┌────────────┐  │            │
│  │  │  IAM Role  │  │  Assume   │  │  IAM Role  │  │            │
│  │  │  with      │──┼───────────┼─▶│  Trust:    │  │            │
│  │  │  permission│  │           │  │  - Source  │  │            │
│  │  │  to assume │  │           │  │  - External│  │            │
│  │  └────────────┘  │           │  │    ID      │  │            │
│  │                  │           │  │  - MFA     │  │            │
│  │                  │           │  └────────────┘  │            │
│  │                  │           │        │         │            │
│  │                  │           │        ▼         │            │
│  │                  │           │  ┌────────────┐  │            │
│  │                  │           │  │  Resources │  │            │
│  │                  │           │  │  (S3, RDS) │  │            │
│  │                  │           │  └────────────┘  │            │
│  └──────────────────┘           └──────────────────┘            │
│                                                                  │
│  Security Controls:                                              │
│  1. External ID (prevents confused deputy)                       │
│  2. MFA requirement                                              │
│  3. Source IP/VPC restrictions                                   │
│  4. Time-based conditions                                        │
│  5. Session tags for ABAC                                        │
└─────────────────────────────────────────────────────────────────┘
```

Trust policy with security controls:
```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Principal": {
            "AWS": "arn:aws:iam::111111111111:role/SourceRole"
        },
        "Action": "sts:AssumeRole",
        "Condition": {
            "StringEquals": {
                "sts:ExternalId": "unique-external-id-12345"
            },
            "Bool": {
                "aws:MultiFactorAuthPresent": "true"
            },
            "IpAddress": {
                "aws:SourceIp": ["10.0.0.0/8", "192.168.1.0/24"]
            }
        }
    }]
}
```

**Interview Tips:**
- Always mention ExternalId for third-party access
- Discuss the confused deputy problem
- Mention AWS RAM for resource sharing

---

#### Q5: Explain IAM Permission Boundaries and their use cases.

**Basic Answer:**
Permission boundaries are advanced IAM features that set the maximum permissions an identity-based policy can grant. They're used to delegate permission management safely.

**Advanced Answer:**

Permission boundaries enable secure delegation:
```
┌─────────────────────────────────────────────────────────────────┐
│                  PERMISSION BOUNDARY EFFECT                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Identity Policy Permissions                                     │
│  ┌─────────────────────────────────────────┐                    │
│  │  S3:*                                   │                    │
│  │  EC2:*                                  │                    │
│  │  Lambda:*                               │                    │
│  │  RDS:*                                  │                    │
│  └─────────────────────────────────────────┘                    │
│                        ∩                                         │
│  Permission Boundary                                             │
│  ┌─────────────────────────────────────────┐                    │
│  │  S3:*                                   │                    │
│  │  Lambda:*                               │                    │
│  │  CloudWatch:*                           │                    │
│  └─────────────────────────────────────────┘                    │
│                        =                                         │
│  Effective Permissions                                           │
│  ┌─────────────────────────────────────────┐                    │
│  │  S3:*                                   │                    │
│  │  Lambda:*                               │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Use cases:
1. **Developer self-service**: Allow developers to create roles within boundaries
2. **Multi-tenant isolation**: Limit each tenant's maximum permissions
3. **Compliance**: Ensure certain permissions can never be granted

```json
// Permission boundary for developers
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowedServices",
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "lambda:*",
                "dynamodb:*",
                "logs:*",
                "cloudwatch:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DenyBoundaryChanges",
            "Effect": "Deny",
            "Action": [
                "iam:DeleteRolePermissionsBoundary",
                "iam:PutRolePermissionsBoundary"
            ],
            "Resource": "*"
        }
    ]
}
```

**Expert Answer:**

Delegation pattern with permission boundaries:
```
┌─────────────────────────────────────────────────────────────────┐
│              SAFE IAM DELEGATION PATTERN                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Central Admin creates:                                          │
│  1. Permission Boundary (DeveloperBoundary)                      │
│  2. Developer IAM Policy that allows:                            │
│     - iam:CreateRole (with boundary condition)                   │
│     - iam:AttachRolePolicy                                       │
│     - iam:CreatePolicy (limited)                                 │
│                                                                  │
│  Developer Policy Condition:                                     │
│  {                                                               │
│      "StringEquals": {                                           │
│          "iam:PermissionsBoundary":                              │
│              "arn:aws:iam::123456789012:policy/DeveloperBoundary"│
│      }                                                           │
│  }                                                               │
│                                                                  │
│  Result:                                                         │
│  - Developers can create roles                                   │
│  - All created roles automatically have the boundary             │
│  - Developers cannot escalate beyond boundary                    │
│  - Central team maintains guardrails                             │
└─────────────────────────────────────────────────────────────────┘
```

**Interview Tips:**
- Explain how boundaries prevent privilege escalation
- Discuss the delegation use case in detail
- Mention that boundaries only affect identity-based policies

---

### 🔴 Advanced Questions

#### Q6: How does AWS STS work internally, and what are the security implications?

**Basic Answer:**
AWS Security Token Service (STS) provides temporary credentials. It's used when assuming roles, for federation, and for temporary elevated access.

**Advanced Answer:**

STS Operations:
| Operation | Use Case | Duration |
|-----------|----------|----------|
| AssumeRole | Cross-account, service roles | 15 min - 12 hours |
| AssumeRoleWithSAML | SAML federation | 15 min - 12 hours |
| AssumeRoleWithWebIdentity | OIDC federation | 15 min - 12 hours |
| GetFederationToken | Custom federation | 15 min - 36 hours |
| GetSessionToken | MFA-protected API access | 15 min - 36 hours |

Token structure:
```
Temporary Credentials:
├── AccessKeyId: ASIA...  (starts with ASIA for temp creds)
├── SecretAccessKey: ****
├── SessionToken: ****  (must be included in requests)
└── Expiration: ISO8601 timestamp
```

**Expert Answer:**

STS internal architecture and security:
```
┌─────────────────────────────────────────────────────────────────┐
│                    STS TOKEN FLOW                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Request                                                      │
│  ┌─────────────┐                                                │
│  │   Client    │────────▶ STS Endpoint (Regional/Global)        │
│  └─────────────┘                                                │
│        │                                                         │
│        │ Request contains:                                       │
│        │ - RoleArn                                               │
│        │ - RoleSessionName                                       │
│        │ - ExternalId (optional)                                 │
│        │ - DurationSeconds                                       │
│        │ - Policy (optional session policy)                      │
│                                                                  │
│  2. Validation                                                   │
│  ┌─────────────┐                                                │
│  │    STS      │────────▶ Validates trust policy                │
│  └─────────────┘          Checks conditions                      │
│        │                   Applies session policy                │
│        │                                                         │
│  3. Token Generation                                             │
│        │                                                         │
│        ▼                                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Temporary Credentials                                   │    │
│  │  - Cryptographically signed                              │    │
│  │  - Contains encoded policy information                   │    │
│  │  - Self-contained (stateless validation)                 │    │
│  │  - Cannot be revoked (only expire)                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Security Implications:                                          │
│  - Tokens cannot be individually revoked                         │
│  - Use short durations for sensitive operations                  │
│  - Session policies further restrict (but can't expand)          │
│  - Regional endpoints reduce latency and improve availability    │
└─────────────────────────────────────────────────────────────────┘
```

Session policies for least privilege:
```python
import boto3

sts = boto3.client('sts')

# Assume role with inline session policy
response = sts.assume_role(
    RoleArn='arn:aws:iam::123456789012:role/AdminRole',
    RoleSessionName='RestrictedSession',
    Policy=json.dumps({
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Action": ["s3:GetObject"],
            "Resource": ["arn:aws:s3:::specific-bucket/*"]
        }]
    }),
    DurationSeconds=900  # 15 minutes
)
```

**Interview Tips:**
- Explain that STS tokens are self-contained and can't be revoked
- Discuss regional vs global STS endpoints
- Mention session policies for additional restrictions
- Know the confused deputy problem and how ExternalId prevents it

---

#### Q7: Describe attribute-based access control (ABAC) in AWS and compare it to RBAC.

**Basic Answer:**
ABAC uses tags and attributes to control access, while RBAC uses predefined roles. ABAC is more scalable as it doesn't require policy updates for new resources.

**Advanced Answer:**

**RBAC vs ABAC Comparison:**

| Aspect | RBAC | ABAC |
|--------|------|------|
| Scaling | Create new policies/roles | Use existing tag-based policies |
| Maintenance | Update policies for new resources | Tag new resources consistently |
| Flexibility | Role-based grouping | Attribute combinations |
| Complexity | Simpler to understand | Requires tag discipline |
| AWS Fit | Traditional approach | Native tag support |

**ABAC Implementation:**
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ABACExample",
            "Effect": "Allow",
            "Action": [
                "ec2:StartInstances",
                "ec2:StopInstances"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/Project": "${aws:PrincipalTag/Project}",
                    "aws:ResourceTag/Environment": "${aws:PrincipalTag/Environment}"
                }
            }
        }
    ]
}
```

**Expert Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    ABAC ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Principal (User/Role)            Resource (EC2, S3, etc.)      │
│  ┌────────────────────┐          ┌────────────────────┐         │
│  │ Tags:              │          │ Tags:              │         │
│  │ - Project: Alpha   │          │ - Project: Alpha   │         │
│  │ - Team: Platform   │          │ - Team: Platform   │         │
│  │ - CostCenter: 1234 │          │ - CostCenter: 1234 │         │
│  └────────────────────┘          └────────────────────┘         │
│           │                               │                      │
│           └───────────┬───────────────────┘                      │
│                       ▼                                          │
│          ┌────────────────────────────────┐                     │
│          │     Policy Condition           │                     │
│          │                                │                     │
│          │ aws:ResourceTag/Project ==    │                     │
│          │ aws:PrincipalTag/Project      │                     │
│          │                                │                     │
│          │ Result: ALLOW if tags match   │                     │
│          └────────────────────────────────┘                     │
│                                                                  │
│  ABAC Benefits:                                                  │
│  1. Single policy scales to thousands of resources               │
│  2. New resources automatically inherit access via tags          │
│  3. Easy to audit (check tags, not policy attachments)          │
│  4. Supports team-based isolation                                │
│  5. Works with session tags for dynamic access                   │
│                                                                  │
│  Session Tags Example (AssumeRole):                              │
│  sts:AssumeRole with:                                            │
│    - Tags: [{Key: Project, Value: Alpha}]                        │
│    - TransitiveTagKeys: [Project]                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Hybrid RBAC + ABAC approach:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "RBACBasePermissions",
            "Effect": "Allow",
            "Action": ["ec2:Describe*", "s3:List*"],
            "Resource": "*"
        },
        {
            "Sid": "ABACProjectAccess",
            "Effect": "Allow",
            "Action": ["ec2:*", "s3:*"],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/Project": "${aws:PrincipalTag/Project}"
                }
            }
        }
    ]
}
```

**Interview Tips:**
- Discuss tag governance requirements for ABAC
- Mention AWS Organizations tag policies for consistency
- Explain how ABAC reduces policy management overhead
- Know the limitations (services must support ABAC)

---

## Networking

### 🟢 Basic Questions

#### Q8: What is a VPC and what are its core components?

**Basic Answer:**
A Virtual Private Cloud (VPC) is a logically isolated virtual network in AWS. Core components include subnets, route tables, internet gateways, NAT gateways, security groups, and NACLs.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                         VPC ARCHITECTURE                         │
│                    CIDR: 10.0.0.0/16 (65,536 IPs)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 Availability Zone A                      │    │
│  │  ┌─────────────────┐    ┌─────────────────┐            │    │
│  │  │ Public Subnet   │    │ Private Subnet  │            │    │
│  │  │ 10.0.1.0/24     │    │ 10.0.10.0/24    │            │    │
│  │  │                 │    │                 │            │    │
│  │  │ ┌───────────┐   │    │ ┌───────────┐   │            │    │
│  │  │ │    EC2    │   │    │ │    EC2    │   │            │    │
│  │  │ │  (Web)    │   │    │ │   (App)   │   │            │    │
│  │  │ └───────────┘   │    │ └───────────┘   │            │    │
│  │  └─────────────────┘    └─────────────────┘            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 Availability Zone B                      │    │
│  │  ┌─────────────────┐    ┌─────────────────┐            │    │
│  │  │ Public Subnet   │    │ Private Subnet  │            │    │
│  │  │ 10.0.2.0/24     │    │ 10.0.20.0/24    │            │    │
│  │  │                 │    │                 │            │    │
│  │  │ ┌───────────┐   │    │ ┌───────────┐   │            │    │
│  │  │ │    EC2    │   │    │ │    RDS    │   │            │    │
│  │  │ │  (Web)    │   │    │ │  (MySQL)  │   │            │    │
│  │  │ └───────────┘   │    │ └───────────┘   │            │    │
│  │  └─────────────────┘    └─────────────────┘            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │   IGW    │  │   NAT    │  │  Route   │  │   SG/    │        │
│  │          │  │ Gateway  │  │  Tables  │  │  NACL    │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

**Component Details:**

| Component | Purpose | Scope |
|-----------|---------|-------|
| Subnet | IP address range segment | AZ-specific |
| Route Table | Traffic routing rules | Subnet association |
| Internet Gateway | Public internet access | VPC-wide |
| NAT Gateway | Outbound internet for private | AZ-specific |
| Security Group | Stateful firewall | Instance-level |
| NACL | Stateless firewall | Subnet-level |

**Expert Answer:**

VPC Reserved IPs per subnet:
```
For a /24 subnet (256 IPs):
- .0   Network address
- .1   VPC router
- .2   DNS server
- .3   Reserved for future use
- .255 Broadcast (not supported in VPC)

Available: 251 IPs
```

VPC Limits (default, can be increased):
- 5 VPCs per region
- 200 subnets per VPC
- 5 Elastic IPs per region
- 200 route tables per VPC

---

#### Q9: Explain the difference between Security Groups and NACLs.

**Basic Answer:**
Security Groups are stateful firewalls at the instance level that only allow rules. NACLs are stateless firewalls at the subnet level that support both allow and deny rules.

**Advanced Answer:**

| Feature | Security Groups | NACLs |
|---------|-----------------|-------|
| Level | Instance/ENI | Subnet |
| Statefulness | Stateful | Stateless |
| Rules | Allow only | Allow and Deny |
| Rule Processing | All rules evaluated | Rules processed in order |
| Default | Deny all inbound, Allow all outbound | Allow all |
| Association | Multiple SGs per instance | One NACL per subnet |

```
┌─────────────────────────────────────────────────────────────────┐
│                  TRAFFIC FLOW WITH SG AND NACL                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Internet                                                        │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  NACL (Subnet Level) - Inbound Rules                    │    │
│  │  Rule 100: Allow TCP 80 from 0.0.0.0/0                  │    │
│  │  Rule 110: Allow TCP 443 from 0.0.0.0/0                 │    │
│  │  Rule *: Deny all                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│      │                                                           │
│      ▼                                                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Security Group (Instance Level)                        │    │
│  │  Inbound: Allow TCP 80 from 0.0.0.0/0                   │    │
│  │  Inbound: Allow TCP 443 from 0.0.0.0/0                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│      │                                                           │
│      ▼                                                           │
│  ┌───────────┐                                                  │
│  │    EC2    │                                                  │
│  └───────────┘                                                  │
│      │                                                           │
│      ▼ (Response - SG stateful, NACL needs explicit rule)        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  NACL - Outbound Rules (Required for response)          │    │
│  │  Rule 100: Allow TCP 1024-65535 to 0.0.0.0/0            │    │
│  │  (Ephemeral ports for return traffic)                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

Use cases for NACLs vs Security Groups:
- **NACLs**: Block specific IP ranges, compliance requirements, defense in depth
- **Security Groups**: Application-level rules, dynamic scaling, reference other SGs

Best practice - Defense in depth:
```
┌─────────────────────────────────────────────────────────────────┐
│                    DEFENSE IN DEPTH                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Layer 1: NACL (Subnet Perimeter)                               │
│  - Block known bad IP ranges                                     │
│  - Allow necessary ports                                         │
│                                                                  │
│  Layer 2: Security Group (Instance)                              │
│  - Application-specific rules                                    │
│  - Reference other security groups                               │
│                                                                  │
│  Layer 3: Host Firewall (OS Level)                              │
│  - iptables, Windows Firewall                                    │
│  - Additional application controls                               │
│                                                                  │
│  Layer 4: Application (Code Level)                              │
│  - Authentication/Authorization                                  │
│  - Input validation                                              │
└─────────────────────────────────────────────────────────────────┘
```

---

### 🟡 Intermediate Questions

#### Q10: Explain VPC Peering vs Transit Gateway vs PrivateLink. When would you use each?

**Basic Answer:**
- VPC Peering: Direct connection between two VPCs
- Transit Gateway: Hub-and-spoke connectivity for multiple VPCs
- PrivateLink: Private access to services without exposing to internet

**Advanced Answer:**

| Feature | VPC Peering | Transit Gateway | PrivateLink |
|---------|-------------|-----------------|-------------|
| Topology | Point-to-point | Hub-and-spoke | Service endpoint |
| Transitive Routing | No | Yes | N/A |
| Cross-Region | Yes | Yes | No (within region) |
| Cross-Account | Yes | Yes | Yes |
| Bandwidth | No limit | 50 Gbps | Based on endpoint |
| Cost | Data transfer | Hourly + data | Hourly + data |

```
┌─────────────────────────────────────────────────────────────────┐
│                    CONNECTIVITY PATTERNS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  VPC PEERING (Point-to-Point)                                   │
│  ┌───────┐        ┌───────┐                                     │
│  │ VPC A │◄──────▶│ VPC B │     Full network access             │
│  └───────┘        └───────┘     Non-transitive                  │
│                                                                  │
│  TRANSIT GATEWAY (Hub-and-Spoke)                                │
│  ┌───────┐                                                      │
│  │ VPC A │──────┐                                               │
│  └───────┘      │    ┌─────────────┐                            │
│  ┌───────┐      ├───▶│   Transit   │     Transitive routing     │
│  │ VPC B │──────┤    │   Gateway   │     Centralized routing    │
│  └───────┘      │    └─────────────┘                            │
│  ┌───────┐      │                                               │
│  │ VPC C │──────┘                                               │
│  └───────┘                                                      │
│                                                                  │
│  PRIVATELINK (Service Endpoint)                                 │
│  ┌───────────────────┐        ┌───────────────────┐            │
│  │    Consumer VPC   │        │   Provider VPC    │            │
│  │                   │        │                   │            │
│  │  ┌─────────────┐  │        │  ┌─────────────┐  │            │
│  │  │  Interface  │◄─┼────────┼─▶│     NLB     │  │            │
│  │  │  Endpoint   │  │        │  │   Service   │  │            │
│  │  └─────────────┘  │        │  └─────────────┘  │            │
│  │                   │        │                   │            │
│  └───────────────────┘        └───────────────────┘            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

Decision matrix:
```
┌─────────────────────────────────────────────────────────────────┐
│                    WHEN TO USE WHAT                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Use VPC Peering when:                                          │
│  ✓ Only 2 VPCs need to communicate                              │
│  ✓ Need lowest latency (direct path)                            │
│  ✓ Simple setup, no transitive needs                            │
│  ✗ Many VPCs (n*(n-1)/2 connections)                            │
│                                                                  │
│  Use Transit Gateway when:                                       │
│  ✓ Multiple VPCs need full mesh connectivity                    │
│  ✓ Need transitive routing                                      │
│  ✓ VPN/Direct Connect aggregation                               │
│  ✓ Complex routing requirements                                  │
│  ✓ Cross-region connectivity                                    │
│                                                                  │
│  Use PrivateLink when:                                          │
│  ✓ Exposing a specific service, not network                     │
│  ✓ SaaS connectivity                                            │
│  ✓ Need to limit network exposure                               │
│  ✓ Consumer doesn't need full VPC access                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

#### Q11: How does Route 53 work and what routing policies are available?

**Basic Answer:**
Route 53 is AWS's DNS service offering domain registration, DNS routing, and health checking. Routing policies include Simple, Weighted, Latency, Failover, Geolocation, Geoproximity, and Multivalue Answer.

**Advanced Answer:**

Route 53 Routing Policies:

| Policy | Use Case | Health Checks |
|--------|----------|---------------|
| Simple | Single resource | No |
| Weighted | A/B testing, gradual migration | Optional |
| Latency | Route to lowest latency region | Optional |
| Failover | Active-passive failover | Required |
| Geolocation | Compliance, content localization | Optional |
| Geoproximity | Route based on distance + bias | Optional |
| Multivalue | Return multiple healthy records | Required |

```
┌─────────────────────────────────────────────────────────────────┐
│                    ROUTE 53 ROUTING                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  LATENCY-BASED ROUTING                                          │
│                                                                  │
│  User in Europe ──────────────────────────────────────▶         │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     Route 53                             │    │
│  │  Measures latency from user to each AWS region          │    │
│  │  Returns record with lowest latency                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│       │                                                          │
│       │  Lowest latency to eu-west-1                            │
│       ▼                                                          │
│  ┌─────────────┐                                                │
│  │  eu-west-1  │                                                │
│  │    ALB      │                                                │
│  └─────────────┘                                                │
│                                                                  │
│  FAILOVER ROUTING                                               │
│                                                                  │
│  User ──▶ Route 53 ──┬──▶ Primary (us-east-1)  [Healthy ✓]     │
│                      │                                          │
│                      └──▶ Secondary (us-west-2) [Standby]       │
│                                                                  │
│  If Primary fails:                                              │
│  User ──▶ Route 53 ──────▶ Secondary (us-west-2) [Active]       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

Health check types:
- **Endpoint**: HTTP, HTTPS, TCP to specific endpoint
- **Calculated**: Combine multiple health checks
- **CloudWatch Alarm**: Based on CloudWatch metric

Route 53 Resolver for hybrid DNS:
```
┌─────────────────────────────────────────────────────────────────┐
│                ROUTE 53 RESOLVER (HYBRID DNS)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  On-Premises ◄───────────────────────────────▶ AWS VPC         │
│                                                                  │
│  ┌─────────────────┐           ┌─────────────────────────────┐  │
│  │   On-Prem DNS   │           │      Route 53 Resolver      │  │
│  │   (AD, BIND)    │           │                             │  │
│  └─────────────────┘           │  Inbound Endpoint:          │  │
│         │                      │  - Receives queries from    │  │
│         │                      │    on-prem                  │  │
│         │   Forward for        │  - Resolves AWS resources   │  │
│         │   *.amazonaws.com    │                             │  │
│         ├─────────────────────▶│  Outbound Endpoint:         │  │
│         │                      │  - Forwards queries to      │  │
│         │                      │    on-prem                  │  │
│         │◀─────────────────────│  - For corp.internal        │  │
│         │   Forward for        │                             │  │
│         │   corp.internal      └─────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Interview Tips:**
- Understand alias vs CNAME records
- Know Route 53 health check options
- Explain how to implement global load balancing
- Discuss DNS caching implications

---

### 🔴 Advanced Questions

#### Q12: Design a multi-region, highly available network architecture for a global application.

**Basic Answer:**
Use Route 53 for global DNS routing, deploy resources in multiple regions with local load balancers, use Global Accelerator for improved performance, and implement cross-region database replication.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│              GLOBAL MULTI-REGION ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                        Global Edge                               │
│  ┌────────────────────────────────────────────────────────┐     │
│  │                    CloudFront                           │     │
│  │                 (Static Content)                        │     │
│  └────────────────────────────────────────────────────────┘     │
│                            │                                     │
│  ┌────────────────────────────────────────────────────────┐     │
│  │                  Global Accelerator                     │     │
│  │              (Anycast IP, TCP/UDP Proxy)               │     │
│  └────────────────────────────────────────────────────────┘     │
│              │                              │                    │
│              ▼                              ▼                    │
│  ┌──────────────────────┐    ┌──────────────────────┐          │
│  │     US-EAST-1        │    │     EU-WEST-1        │          │
│  │  ┌───────────────┐   │    │  ┌───────────────┐   │          │
│  │  │      ALB      │   │    │  │      ALB      │   │          │
│  │  └───────────────┘   │    │  └───────────────┘   │          │
│  │         │            │    │         │            │          │
│  │  ┌──────┴──────┐     │    │  ┌──────┴──────┐     │          │
│  │  │    EKS      │     │    │  │    EKS      │     │          │
│  │  │  Cluster    │     │    │  │  Cluster    │     │          │
│  │  └─────────────┘     │    │  └─────────────┘     │          │
│  │         │            │    │         │            │          │
│  │  ┌─────────────┐     │    │  ┌─────────────┐     │          │
│  │  │Aurora Global│     │    │  │Aurora Global│     │          │
│  │  │  (Primary)  │◄────┼────┼─▶│ (Replica)   │     │          │
│  │  └─────────────┘     │    │  └─────────────┘     │          │
│  │                      │    │                      │          │
│  │  ┌─────────────┐     │    │  ┌─────────────┐     │          │
│  │  │ElastiCache  │     │    │  │ElastiCache  │     │          │
│  │  │  (Redis)    │     │    │  │  (Redis)    │     │          │
│  │  └─────────────┘     │    │  └─────────────┘     │          │
│  └──────────────────────┘    └──────────────────────┘          │
│                                                                  │
│  Cross-Region Connectivity:                                      │
│  - Transit Gateway Inter-Region Peering                          │
│  - VPC Peering for specific traffic                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

Network design considerations:

1. **Edge Layer**
   - CloudFront for static content (400+ PoPs)
   - Global Accelerator for dynamic content (TCP/UDP optimization)
   - Route 53 for DNS-based routing

2. **Regional Layer**
   - Application Load Balancers in each region
   - EKS/ECS clusters with auto-scaling
   - Regional caching (ElastiCache)

3. **Data Layer**
   - Aurora Global Database (< 1 second replication)
   - DynamoDB Global Tables (eventual consistency)
   - S3 Cross-Region Replication

4. **Network Connectivity**
   - Transit Gateway for intra-region
   - Transit Gateway Inter-Region Peering
   - PrivateLink for service access

Failover strategy:
```
┌─────────────────────────────────────────────────────────────────┐
│                    FAILOVER MECHANISMS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Level 1: Instance Failover                                      │
│  - ALB health checks (seconds)                                   │
│  - EKS pod health checks                                        │
│                                                                  │
│  Level 2: AZ Failover                                           │
│  - Multi-AZ deployments                                         │
│  - ALB cross-zone load balancing                                │
│                                                                  │
│  Level 3: Regional Failover                                      │
│  - Route 53 health checks + failover routing                    │
│  - Global Accelerator endpoint groups                           │
│  - Aurora Global Database failover (RPO < 1s, RTO < 1 min)      │
│                                                                  │
│  Automation:                                                     │
│  - CloudWatch Alarms → EventBridge → Lambda                     │
│  - Automated DNS failover                                       │
│  - Database promotion                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Compute

### 🟢 Basic Questions

#### Q13: Explain EC2 instance types and how to choose the right one.

**Basic Answer:**
EC2 offers different instance families optimized for various workloads: General Purpose (T, M), Compute Optimized (C), Memory Optimized (R, X), Storage Optimized (I, D), and Accelerated Computing (P, G). Choose based on your workload's CPU, memory, and I/O requirements.

**Advanced Answer:**

Instance Family Overview:
```
┌─────────────────────────────────────────────────────────────────┐
│                    EC2 INSTANCE FAMILIES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  GENERAL PURPOSE (T3, M6i)                                       │
│  ├── Balanced compute, memory, networking                        │
│  ├── T3: Burstable, good for variable workloads                 │
│  └── M6i: Fixed performance, production workloads               │
│                                                                  │
│  COMPUTE OPTIMIZED (C6i, C7g)                                   │
│  ├── High CPU-to-memory ratio                                   │
│  ├── Batch processing, gaming, HPC                              │
│  └── C7g: Graviton3 (ARM), 25% better price/perf               │
│                                                                  │
│  MEMORY OPTIMIZED (R6i, X2idn)                                  │
│  ├── High memory-to-CPU ratio                                   │
│  ├── In-memory databases, caching                               │
│  └── X2: Up to 4TB RAM                                          │
│                                                                  │
│  STORAGE OPTIMIZED (I4i, D3)                                    │
│  ├── High sequential read/write                                 │
│  ├── I4i: NVMe SSD, high IOPS                                   │
│  └── D3: HDD, high throughput                                   │
│                                                                  │
│  ACCELERATED COMPUTING (P4d, G5)                                │
│  ├── P4d: ML training (NVIDIA A100)                             │
│  └── G5: Graphics, inference (NVIDIA A10G)                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Instance naming convention:
```
m6i.2xlarge
│││ │
││└─ Instance generation
│└── Family (m = general purpose)
└─── Additional capabilities (i = Intel, a = AMD, g = Graviton)
    
    xlarge = size (vCPU and memory scale)
```

**Expert Answer:**

Selection criteria with real examples:
```
┌─────────────────────────────────────────────────────────────────┐
│                    INSTANCE SELECTION GUIDE                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Workload: Web Application                                       │
│  ├── Start: t3.medium (burstable, cost-effective)               │
│  ├── Scale: m6i.large (predictable load)                        │
│  └── Optimize: c6g.large (compute-heavy, Graviton)              │
│                                                                  │
│  Workload: Database                                              │
│  ├── Small: r6i.large (memory optimized)                        │
│  ├── Medium: r6i.2xlarge                                        │
│  └── Large: x2idn.xlarge (high memory)                          │
│                                                                  │
│  Workload: Data Processing                                       │
│  ├── CPU-bound: c6i.xlarge                                      │
│  ├── Memory-bound: r6i.xlarge                                   │
│  └── I/O-bound: i4i.xlarge (NVMe storage)                       │
│                                                                  │
│  Cost Optimization:                                              │
│  1. Use Graviton (ARM) for 20-40% savings                       │
│  2. Right-size using CloudWatch metrics                         │
│  3. Reserved Instances for steady-state                         │
│  4. Spot for fault-tolerant workloads                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

#### Q14: What are the EC2 purchasing options and when to use each?

**Basic Answer:**
- **On-Demand**: Pay per hour/second, no commitment
- **Reserved Instances**: 1-3 year commitment, up to 72% discount
- **Spot**: Up to 90% discount, can be interrupted
- **Savings Plans**: Flexible commitment, discounts across services

**Advanced Answer:**

| Purchase Option | Discount | Commitment | Use Case |
|-----------------|----------|------------|----------|
| On-Demand | 0% | None | Variable, unpredictable workloads |
| Reserved (Standard) | Up to 72% | 1-3 years | Steady-state workloads |
| Reserved (Convertible) | Up to 66% | 1-3 years | Evolving requirements |
| Spot | Up to 90% | None | Fault-tolerant, flexible |
| Savings Plans (Compute) | Up to 66% | 1-3 years | Flexible across services |
| Savings Plans (EC2) | Up to 72% | 1-3 years | Specific to EC2 family |
| Dedicated Hosts | Varies | None/Reserved | Licensing, compliance |

```
┌─────────────────────────────────────────────────────────────────┐
│                    COST OPTIMIZATION STRATEGY                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Workload Analysis:                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Total Capacity Needed                                   │    │
│  │  ████████████████████████████████████████ 100%          │    │
│  │                                                          │    │
│  │  ├─ Baseline (always running)                            │    │
│  │  │  ████████████████████ 50%  → Reserved/Savings Plans  │    │
│  │  │                                                        │    │
│  │  ├─ Variable (predictable scaling)                        │    │
│  │  │  ████████████ 30%  → Mix of Spot + On-Demand          │    │
│  │  │                                                        │    │
│  │  └─ Burst (unpredictable)                                 │    │
│  │     ████████ 20%  → On-Demand                            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Spot Instance Strategies:                                       │
│  1. Diversify across AZs and instance types                     │
│  2. Use Spot Fleet with allocation strategies                   │
│  3. Implement graceful shutdown handling                        │
│  4. Use capacity-optimized allocation                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

Spot instance interruption handling:
```python
# Check for spot interruption notice (2-minute warning)
import requests

def check_spot_interruption():
    try:
        response = requests.get(
            'http://169.254.169.254/latest/meta-data/spot/termination-time',
            timeout=2
        )
        if response.status_code == 200:
            # Termination scheduled
            termination_time = response.text
            # Initiate graceful shutdown
            drain_connections()
            save_state()
            deregister_from_lb()
            return True
    except requests.exceptions.RequestException:
        # No interruption notice
        return False
```

---

[Continue with Sections 14-50 for remaining AWS content...]

---

## Troubleshooting Scenarios

### Scenario 1: EC2 Instance Not Reachable

**Problem:** A newly launched EC2 instance is not accessible via SSH.

**Diagnostic Steps:**

```
┌─────────────────────────────────────────────────────────────────┐
│              EC2 CONNECTIVITY TROUBLESHOOTING                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Step 1: Check Instance Status                                   │
│  aws ec2 describe-instance-status --instance-id i-xxx           │
│  ├── System Status: Should be "ok"                              │
│  └── Instance Status: Should be "ok"                            │
│                                                                  │
│  Step 2: Verify Security Group                                   │
│  aws ec2 describe-security-groups --group-ids sg-xxx            │
│  └── Check for inbound rule: TCP 22 from your IP                │
│                                                                  │
│  Step 3: Check NACL                                              │
│  aws ec2 describe-network-acls --filters "Name=..."             │
│  ├── Inbound: Allow TCP 22                                      │
│  └── Outbound: Allow ephemeral ports (1024-65535)               │
│                                                                  │
│  Step 4: Verify Route Table                                      │
│  ├── Public subnet: Route to IGW (0.0.0.0/0 → igw-xxx)          │
│  └── Private subnet: Route to NAT (0.0.0.0/0 → nat-xxx)         │
│                                                                  │
│  Step 5: Check Public IP                                        │
│  ├── Public subnet: Should have public IP or EIP                │
│  └── Private subnet: Need bastion or VPN                        │
│                                                                  │
│  Step 6: Verify Key Pair                                        │
│  └── ssh -i key.pem -v ec2-user@ip (verbose for debugging)      │
│                                                                  │
│  Step 7: Check System Logs (if all else fails)                  │
│  aws ec2 get-console-output --instance-id i-xxx                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Common Causes:**
1. Security group doesn't allow SSH (port 22)
2. Instance is in private subnet without NAT/bastion
3. NACL blocking traffic
4. No public IP assigned
5. Wrong key pair
6. Instance failed to boot (check system logs)

---

### Scenario 2: High Latency on Application

**Problem:** Users report intermittent high latency accessing the application.

**Diagnostic Approach:**

```
┌─────────────────────────────────────────────────────────────────┐
│              LATENCY TROUBLESHOOTING FLOW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. IDENTIFY THE LAYER                                          │
│     ├── Network (DNS, routing)                                  │
│     ├── Load Balancer                                           │
│     ├── Application                                             │
│     └── Database                                                │
│                                                                  │
│  2. CHECK CLOUDWATCH METRICS                                    │
│     ALB:                                                        │
│     ├── TargetResponseTime (time to backend)                    │
│     ├── RequestCount (traffic patterns)                         │
│     └── HTTPCode_Target_5XX (backend errors)                    │
│                                                                  │
│     EC2:                                                        │
│     ├── CPUUtilization (over 80%?)                              │
│     ├── NetworkIn/Out (saturation?)                             │
│     └── DiskReadOps (I/O wait?)                                 │
│                                                                  │
│     RDS:                                                        │
│     ├── DatabaseConnections (maxed out?)                        │
│     ├── ReadLatency/WriteLatency                                │
│     └── CPUUtilization                                          │
│                                                                  │
│  3. ENABLE X-RAY FOR DISTRIBUTED TRACING                        │
│     ├── Identify slow service calls                             │
│     ├── Find downstream dependencies                            │
│     └── Analyze service map                                     │
│                                                                  │
│  4. CHECK APPLICATION LOGS                                       │
│     ├── CloudWatch Logs Insights query                          │
│     └── Look for timeouts, connection errors                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**CloudWatch Logs Insights Query:**
```sql
fields @timestamp, @message
| filter @message like /(?i)(timeout|slow|latency|error)/
| sort @timestamp desc
| limit 100
```

---

## Architecture Design Questions

### Design 1: Multi-Account AWS Architecture for Enterprise

**Requirements:**
- 500+ developers across 10 business units
- Compliance requirements (SOC2, HIPAA)
- Cost allocation by business unit
- Centralized security and networking

**Solution:**

```
┌─────────────────────────────────────────────────────────────────┐
│              ENTERPRISE MULTI-ACCOUNT ARCHITECTURE               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  MANAGEMENT ACCOUNT                      │    │
│  │  AWS Organizations, SCPs, Consolidated Billing           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│  ┌─────────────────────────┴─────────────────────────┐          │
│  │                                                    │          │
│  ▼                                                    ▼          │
│  ┌─────────────────────┐              ┌─────────────────────┐   │
│  │   SECURITY OU       │              │   INFRASTRUCTURE OU │   │
│  │  ┌───────────────┐  │              │  ┌───────────────┐  │   │
│  │  │ Log Archive   │  │              │  │   Network     │  │   │
│  │  │ - CloudTrail  │  │              │  │   - TGW       │  │   │
│  │  │ - VPC Flow    │  │              │  │   - DNS       │  │   │
│  │  │ - Config      │  │              │  │   - VPN/DX    │  │   │
│  │  └───────────────┘  │              │  └───────────────┘  │   │
│  │  ┌───────────────┐  │              │  ┌───────────────┐  │   │
│  │  │ Security Hub  │  │              │  │   Shared      │  │   │
│  │  │ - GuardDuty   │  │              │  │   Services    │  │   │
│  │  │ - Inspector   │  │              │  │   - CI/CD     │  │   │
│  │  │ - IAM AA      │  │              │  │   - Artifacts │  │   │
│  │  └───────────────┘  │              │  └───────────────┘  │   │
│  └─────────────────────┘              └─────────────────────┘   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    WORKLOAD OUs                          │    │
│  │                                                          │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │    │
│  │  │ BU-1 (Dev)  │  │ BU-1 (Prod) │  │ BU-2 (Dev)  │ ...  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │    │
│  │                                                          │    │
│  │  Each workload account:                                  │    │
│  │  - VPC connected to Transit Gateway                      │    │
│  │  - Logs sent to Log Archive                              │    │
│  │  - Security findings to Security Hub                     │    │
│  │  - Cost tagged for allocation                            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Key Components:**

1. **AWS Control Tower**: Automated account provisioning with guardrails
2. **Service Control Policies**: Prevent disabling security controls
3. **Transit Gateway**: Centralized network connectivity
4. **Centralized Logging**: CloudTrail, VPC Flow Logs, Config
5. **Security Hub**: Aggregated security findings
6. **AWS SSO**: Federated identity management

---

## 📚 Documentation Links

### Official AWS Documentation
- [IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/)
- [VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [EC2 User Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/)
- [EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/)

### AWS Whitepapers
- [Security Best Practices](https://docs.aws.amazon.com/whitepapers/latest/introduction-aws-security/)
- [Multi-Account Strategy](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/)
- [Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/)

### AWS Architecture Center
- [Reference Architectures](https://aws.amazon.com/architecture/)
- [This Is My Architecture](https://aws.amazon.com/architecture/this-is-my-architecture/)

---

**[← Back to Main README](../README.md)** | **[Next: Azure →](../azure/README.md)**
