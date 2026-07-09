# Microsoft Azure Interview Questions - Complete Guide

> **500+ Azure Interview Questions for Senior DevOps Engineer, SRE, and Platform Engineer roles at FAANG companies**

---

## 📋 Table of Contents

- [Quick Reference](#quick-reference)
- [Azure Fundamentals](#azure-fundamentals)
- [Identity & Access Management](#identity--access-management)
- [Networking](#networking)
- [Compute Services](#compute-services)
- [Storage Services](#storage-services)
- [Database Services](#database-services)
- [Containers & Kubernetes](#containers--kubernetes)
- [Monitoring & Observability](#monitoring--observability)
- [Security Services](#security-services)
- [Governance & Compliance](#governance--compliance)
- [Troubleshooting Scenarios](#troubleshooting-scenarios)
- [Architecture Design Questions](#architecture-design-questions)

---

## Quick Reference

### Azure Global Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                   AZURE GLOBAL INFRASTRUCTURE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │    Geography    │  │    Geography    │  │    Geography    │  │
│  │   (Americas)    │  │    (Europe)     │  │  (Asia Pacific) │  │
│  │                 │  │                 │  │                 │  │
│  │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │  │
│  │  │  Region   │  │  │  │  Region   │  │  │  │  Region   │  │  │
│  │  │ East US   │  │  │  │West Europe│  │  │  │East Asia  │  │  │
│  │  │           │  │  │  │           │  │  │  │           │  │  │
│  │  │  ┌─────┐  │  │  │  │  ┌─────┐  │  │  │  │  ┌─────┐  │  │  │
│  │  │  │ AZ1 │  │  │  │  │  │ AZ1 │  │  │  │  │  │ AZ1 │  │  │  │
│  │  │  │ AZ2 │  │  │  │  │  │ AZ2 │  │  │  │  │  │ AZ2 │  │  │  │
│  │  │  │ AZ3 │  │  │  │  │  │ AZ3 │  │  │  │  │  │ AZ3 │  │  │  │
│  │  │  └─────┘  │  │  │  │  └─────┘  │  │  │  │  └─────┘  │  │  │
│  │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                  │
│  60+ Regions | 300+ Availability Zones | 190+ Edge Locations    │
└─────────────────────────────────────────────────────────────────┘
```

### Azure Resource Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                  AZURE RESOURCE HIERARCHY                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                 Tenant (Azure AD / Entra ID)             │    │
│  │                    Identity boundary                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  Management Groups                        │    │
│  │              Organize subscriptions                       │    │
│  │              Apply policies at scale                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│           ┌────────────────┼────────────────┐                   │
│           ▼                ▼                ▼                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │Subscription │  │Subscription │  │Subscription │             │
│  │   (Prod)    │  │    (Dev)    │  │    (Test)   │             │
│  │             │  │             │  │             │             │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │             │
│  │ │Resource │ │  │ │Resource │ │  │ │Resource │ │             │
│  │ │ Groups  │ │  │ │ Groups  │ │  │ │ Groups  │ │             │
│  │ │         │ │  │ │         │ │  │ │         │ │             │
│  │ │Resources│ │  │ │Resources│ │  │ │Resources│ │             │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Azure Fundamentals

### 🟢 Basic Questions

#### Q1: Explain the difference between Azure Regions, Availability Zones, and Availability Sets.

**Basic Answer:**
- **Region**: Physical location with one or more data centers
- **Availability Zone**: Separate data centers within a region with independent power, cooling, and networking
- **Availability Set**: Logical grouping of VMs to distribute across fault domains and update domains

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│              AZURE AVAILABILITY CONCEPTS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  REGION: East US                                                │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  AZ 1              AZ 2              AZ 3                │    │
│  │  ┌──────────┐     ┌──────────┐     ┌──────────┐         │    │
│  │  │Datacenter│     │Datacenter│     │Datacenter│         │    │
│  │  │          │     │          │     │          │         │    │
│  │  │ Power    │     │ Power    │     │ Power    │         │    │
│  │  │ Cooling  │     │ Cooling  │     │ Cooling  │         │    │
│  │  │ Network  │     │ Network  │     │ Network  │         │    │
│  │  └──────────┘     └──────────┘     └──────────┘         │    │
│  │       │                │                │                │    │
│  │       └────── High-speed, low-latency links ─────────┘   │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  AVAILABILITY SET (Within single datacenter)                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Fault Domain 0    Fault Domain 1    Fault Domain 2     │    │
│  │  ┌──────────┐     ┌──────────┐     ┌──────────┐         │    │
│  │  │ Rack 1   │     │ Rack 2   │     │ Rack 3   │         │    │
│  │  │ VM1, VM4 │     │ VM2, VM5 │     │ VM3, VM6 │         │    │
│  │  └──────────┘     └──────────┘     └──────────┘         │    │
│  │                                                          │    │
│  │  Update Domains: Spread VMs for planned maintenance      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  SLA Comparison:                                                │
│  ├── Single VM (Premium SSD): 99.9%                            │
│  ├── Availability Set: 99.95%                                  │
│  └── Availability Zones: 99.99%                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

When to use each:

| Scenario | Recommendation | SLA |
|----------|----------------|-----|
| Development/Test | Single VM | 99.9% |
| Legacy apps (no zone support) | Availability Set | 99.95% |
| Production workloads | Availability Zones | 99.99% |
| Global applications | Multiple Regions | 99.99%+ |

Zone-redundant services:
- Azure SQL Database (zone-redundant)
- Azure Storage (ZRS)
- Azure Load Balancer (Standard)
- Azure VPN Gateway (zone-redundant SKU)
- Azure Application Gateway v2

**Interview Tips:**
- Know which services support Availability Zones
- Understand the trade-off between availability and latency
- Discuss cross-region disaster recovery patterns

---

#### Q2: What is Azure Resource Manager (ARM) and how does it work?

**Basic Answer:**
Azure Resource Manager is the deployment and management service for Azure. It provides a consistent management layer for creating, updating, and deleting resources through templates, REST APIs, SDKs, and CLI.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│               AZURE RESOURCE MANAGER ARCHITECTURE                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Management Interfaces                       │    │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ │    │
│  │  │ Portal │ │  CLI   │ │PowerShl│ │ SDK    │ │REST API│ │    │
│  │  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │            Azure Resource Manager (ARM)                  │    │
│  │                                                          │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │    │
│  │  │Authentication│  │Authorization │  │  Validation  │   │    │
│  │  │   (Entra ID) │  │    (RBAC)    │  │   & Audit    │   │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │    │
│  │                                                          │    │
│  │  ┌──────────────────────────────────────────────────┐   │    │
│  │  │           Template Processing Engine              │   │    │
│  │  │  - Dependency resolution                         │   │    │
│  │  │  - Parallel deployment                           │   │    │
│  │  │  - Idempotent operations                         │   │    │
│  │  └──────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│          ┌─────────────────┼─────────────────┐                  │
│          ▼                 ▼                 ▼                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Compute    │  │   Storage    │  │   Network    │          │
│  │   Provider   │  │   Provider   │  │   Provider   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

ARM Template vs Bicep vs Terraform:

| Feature | ARM Template | Bicep | Terraform |
|---------|--------------|-------|-----------|
| Format | JSON | DSL | HCL |
| Learning Curve | Steep | Moderate | Moderate |
| Multi-cloud | No | No | Yes |
| State Management | Azure | Azure | External |
| Modularity | Linked templates | Modules | Modules |

**Expert Answer:**

ARM deployment modes:
- **Incremental** (default): Only adds/modifies resources
- **Complete**: Removes resources not in template (dangerous!)

Best practices:
```json
// ARM Template structure
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": { },
    "variables": { },
    "resources": [ ],
    "outputs": { }
}
```

Deployment scopes:
- Resource Group (most common)
- Subscription (policies, RBAC)
- Management Group (organization-wide)
- Tenant (Azure AD resources)

---

## Identity & Access Management

### 🟢 Basic Questions

#### Q3: What is Microsoft Entra ID (formerly Azure AD) and how does it differ from on-premises Active Directory?

**Basic Answer:**
Microsoft Entra ID is a cloud-based identity and access management service. Unlike on-premises AD which uses LDAP and Kerberos, Entra ID uses REST APIs, SAML, OIDC, and OAuth 2.0 for modern authentication.

**Advanced Answer:**

| Feature | On-Premises AD | Microsoft Entra ID |
|---------|----------------|-------------------|
| Protocol | LDAP, Kerberos | SAML, OIDC, OAuth 2.0 |
| Structure | Hierarchical (OUs, domains) | Flat (users, groups) |
| Join Type | Domain Join | Azure AD Join, Hybrid Join |
| Primary Use | On-prem resources | Cloud apps, SaaS |
| Authentication | NTLM, Kerberos | Modern auth, MFA |
| Group Policy | Supported | Intune MDM |

```
┌─────────────────────────────────────────────────────────────────┐
│              HYBRID IDENTITY ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  On-Premises                          Cloud                      │
│  ┌────────────────────┐              ┌────────────────────┐     │
│  │                    │              │                    │     │
│  │  Active Directory  │◄────────────▶│  Microsoft Entra   │     │
│  │  Domain Services   │  Azure AD    │  ID                │     │
│  │                    │  Connect     │                    │     │
│  │  ┌──────────────┐  │  (Sync)      │  ┌──────────────┐  │     │
│  │  │    Users     │  │              │  │ Cloud Users  │  │     │
│  │  │   Groups     │  │              │  │   Groups     │  │     │
│  │  │  Computers   │  │              │  │    Apps      │  │     │
│  │  └──────────────┘  │              │  └──────────────┘  │     │
│  │                    │              │                    │     │
│  │  Authentication:   │              │  Authentication:   │     │
│  │  - Kerberos        │              │  - SAML 2.0        │     │
│  │  - NTLM            │              │  - OIDC            │     │
│  │                    │              │  - OAuth 2.0       │     │
│  └────────────────────┘              └────────────────────┘     │
│                                                                  │
│  Sync Options:                                                   │
│  1. Password Hash Sync (PHS) - Recommended                      │
│  2. Pass-through Authentication (PTA)                           │
│  3. Federation (ADFS)                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

Entra ID License Tiers:

| Feature | Free | P1 | P2 |
|---------|------|----|----|
| SSO | ✓ | ✓ | ✓ |
| MFA | ✓ | ✓ | ✓ |
| Conditional Access | | ✓ | ✓ |
| PIM | | | ✓ |
| Identity Protection | | | ✓ |
| Access Reviews | | | ✓ |

Conditional Access example:
```
IF:
  - User is in "Sales" group
  - Accessing "Salesforce" application
  - From untrusted location
  - Device is not compliant
THEN:
  - Require MFA
  - Block legacy authentication
  - Require compliant device
```

---

### 🟡 Intermediate Questions

#### Q4: Explain Azure RBAC (Role-Based Access Control) and how to implement least privilege access.

**Basic Answer:**
Azure RBAC provides fine-grained access management using roles assigned to security principals (users, groups, service principals) at a specific scope (management group, subscription, resource group, resource).

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    AZURE RBAC COMPONENTS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Security Principal  +  Role Definition  +  Scope  =  Access    │
│                                                                  │
│  WHO?                   WHAT?              WHERE?                │
│  ┌─────────────────┐   ┌─────────────────┐   ┌──────────────┐   │
│  │ • User          │   │ • Actions       │   │ • Mgmt Group │   │
│  │ • Group         │ + │ • NotActions    │ + │ • Subscription│ = │
│  │ • Service Princ │   │ • DataActions   │   │ • Res Group  │   │
│  │ • Managed ID    │   │ • NotDataActions│   │ • Resource   │   │
│  └─────────────────┘   └─────────────────┘   └──────────────┘   │
│                                                                  │
│  Built-in Roles (Examples):                                      │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Owner              - Full access + assign roles        │     │
│  │ Contributor        - Full access, no role assignment   │     │
│  │ Reader             - View all resources                │     │
│  │ User Access Admin  - Manage user access only           │     │
│  │ Storage Blob Reader- Read blob data                    │     │
│  │ AKS Cluster Admin  - Full AKS cluster access           │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
│  Role Inheritance:                                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Management Group (Role: Reader)                        │    │
│  │       │                                                  │    │
│  │       ▼                                                  │    │
│  │  Subscription (inherits Reader)                          │    │
│  │       │                                                  │    │
│  │       ▼                                                  │    │
│  │  Resource Group (inherits Reader + Contributor assigned) │    │
│  │       │                                                  │    │
│  │       ▼                                                  │    │
│  │  Resource (inherits all above)                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Custom Role Example:
```json
{
    "Name": "VM Operator",
    "Description": "Can start, stop, and restart VMs",
    "Actions": [
        "Microsoft.Compute/virtualMachines/start/action",
        "Microsoft.Compute/virtualMachines/restart/action",
        "Microsoft.Compute/virtualMachines/deallocate/action",
        "Microsoft.Compute/virtualMachines/read"
    ],
    "NotActions": [],
    "DataActions": [],
    "NotDataActions": [],
    "AssignableScopes": [
        "/subscriptions/{subscription-id}"
    ]
}
```

**Expert Answer:**

Least Privilege Implementation Strategy:

```
┌─────────────────────────────────────────────────────────────────┐
│              LEAST PRIVILEGE IMPLEMENTATION                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. AUDIT CURRENT ACCESS                                        │
│     az role assignment list --all --output table                │
│     Use Azure AD Access Reviews for periodic validation          │
│                                                                  │
│  2. IMPLEMENT ROLE HIERARCHY                                     │
│     ┌─────────────────────────────────────────────────────┐     │
│     │  Management Group: Reader (Security team)           │     │
│     │       │                                             │     │
│     │       ├── Prod Subscription: Limited Contributor    │     │
│     │       │       └── Prod RG: Custom App Operator     │     │
│     │       │                                             │     │
│     │       └── Dev Subscription: Contributor             │     │
│     │               └── Dev RG: Full access               │     │
│     └─────────────────────────────────────────────────────┘     │
│                                                                  │
│  3. USE PIM FOR PRIVILEGED ACCESS                               │
│     - Just-in-time access                                       │
│     - Time-bound assignments                                    │
│     - Approval workflows                                        │
│     - Audit trail                                               │
│                                                                  │
│  4. PREFER MANAGED IDENTITIES                                   │
│     - System-assigned for single resource                       │
│     - User-assigned for shared identity                         │
│     - No credential management                                  │
│                                                                  │
│  5. REGULAR ACCESS REVIEWS                                      │
│     - Quarterly reviews for all privileged roles                │
│     - Remove stale assignments                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

#### Q5: What is Privileged Identity Management (PIM) and how would you implement it?

**Basic Answer:**
PIM is an Entra ID P2 feature that provides time-based and approval-based role activation to reduce the risk of excessive, unnecessary, or misused access permissions.

**Advanced Answer:**

PIM Key Features:
- Just-in-time privileged access
- Time-bound access with start and end dates
- Approval workflow for role activation
- Multi-factor authentication enforcement
- Notifications and alerts
- Access reviews

```
┌─────────────────────────────────────────────────────────────────┐
│                    PIM ACTIVATION FLOW                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  User (Eligible)                                                │
│       │                                                          │
│       │ 1. Request activation                                    │
│       ▼                                                          │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  PIM Workflow                            │    │
│  │                                                          │    │
│  │  2. MFA Challenge    3. Justification    4. Approval     │    │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │    │
│  │  │   Verify    │───▶│   Reason    │───▶│  Manager/   │  │    │
│  │  │  Identity   │    │  Required   │    │   Admin     │  │    │
│  │  └─────────────┘    └─────────────┘    └─────────────┘  │    │
│  │                                              │           │    │
│  └──────────────────────────────────────────────┼──────────┘    │
│                                                  │               │
│                                                  ▼               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Time-Limited Role Assignment                │    │
│  │                                                          │    │
│  │  Duration: 8 hours (configurable)                        │    │
│  │  Scope: Specific subscription/resource group             │    │
│  │  Role: Contributor (not Owner)                           │    │
│  │                                                          │    │
│  │  Auto-expires after duration                             │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Audit Trail:                                                    │
│  - Who requested                                                │
│  - Who approved                                                 │
│  - What actions taken                                           │
│  - When it expired                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

PIM Configuration Best Practices:

```yaml
# PIM Role Settings (example for Contributor)
Settings:
  Activation:
    MaximumDuration: 8 hours
    RequireMFA: true
    RequireJustification: true
    RequireTicketInfo: true
    RequireApproval: true
    Approvers:
      - SecurityTeam@company.com
      - Manager
  
  Assignment:
    AllowPermanentEligible: false
    ExpireEligibleAfter: 365 days
    AllowPermanentActive: false
    ExpireActiveAfter: 8 hours
  
  Notification:
    SendEmailOnActivation: true
    SendEmailToApprovers: true
    SendEmailToAdmins: true

# Azure Role PIM vs Entra ID Role PIM
AzureRoles:
  - Scope: Azure resources
  - Examples: Contributor, Owner, Custom roles
  - Managed in: PIM > Azure resources

EntraIDRoles:
  - Scope: Entra ID
  - Examples: Global Admin, User Admin
  - Managed in: PIM > Entra ID roles
```

---

## Networking

### 🟢 Basic Questions

#### Q6: Explain Azure Virtual Network (VNet) and its key components.

**Basic Answer:**
Azure VNet is a logically isolated network in Azure that enables resources to securely communicate with each other, the internet, and on-premises networks. Key components include subnets, NSGs, route tables, and peering.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    AZURE VNET ARCHITECTURE                       │
│                    CIDR: 10.0.0.0/16                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                     Subnet Layout                        │    │
│  │                                                          │    │
│  │  ┌──────────────────┐  ┌──────────────────┐             │    │
│  │  │ Web Subnet       │  │ App Subnet       │             │    │
│  │  │ 10.0.1.0/24      │  │ 10.0.2.0/24      │             │    │
│  │  │                  │  │                  │             │    │
│  │  │ NSG: Web-NSG     │  │ NSG: App-NSG     │             │    │
│  │  │ - Allow 80,443   │  │ - Allow from Web │             │    │
│  │  │ - Deny all else  │  │ - Deny all else  │             │    │
│  │  └──────────────────┘  └──────────────────┘             │    │
│  │                                                          │    │
│  │  ┌──────────────────┐  ┌──────────────────┐             │    │
│  │  │ DB Subnet        │  │ Gateway Subnet   │             │    │
│  │  │ 10.0.3.0/24      │  │ 10.0.255.0/27    │             │    │
│  │  │                  │  │                  │             │    │
│  │  │ NSG: DB-NSG      │  │ VPN/ExpressRoute │             │    │
│  │  │ - Allow from App │  │ Gateway          │             │    │
│  │  │ - Deny all else  │  │                  │             │    │
│  │  └──────────────────┘  └──────────────────┘             │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Reserved IPs per subnet (5 total):                             │
│  .0 = Network address                                           │
│  .1 = Default gateway                                           │
│  .2, .3 = Azure DNS                                             │
│  .255 = Broadcast                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

VNet Design Considerations:

| Consideration | Recommendation |
|--------------|----------------|
| Address Space | Plan for growth, avoid overlaps for peering |
| Subnets | Separate by tier (web, app, db) and security zone |
| NSGs | Apply at subnet level, use ASGs for grouping |
| Service Endpoints | Enable for PaaS services (Storage, SQL) |
| Private Endpoints | Use for private PaaS connectivity |
| DNS | Use Azure DNS Private Zones for internal resolution |

---

### 🟡 Intermediate Questions

#### Q7: Compare Azure Load Balancer, Application Gateway, Front Door, and Traffic Manager.

**Basic Answer:**
- **Load Balancer**: Layer 4 (TCP/UDP), regional
- **Application Gateway**: Layer 7 (HTTP/HTTPS), regional, WAF
- **Front Door**: Layer 7, global, CDN, WAF
- **Traffic Manager**: DNS-based, global routing

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│              AZURE LOAD BALANCING OPTIONS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   GLOBAL (Cross-Region)                  │    │
│  │                                                          │    │
│  │  Traffic Manager (DNS)     Front Door (HTTP/HTTPS)      │    │
│  │  ┌─────────────────┐      ┌─────────────────────────┐   │    │
│  │  │ • DNS routing   │      │ • Layer 7 proxy         │   │    │
│  │  │ • No data path  │      │ • SSL offload           │   │    │
│  │  │ • Any protocol  │      │ • WAF included          │   │    │
│  │  │ • Failover only │      │ • URL routing           │   │    │
│  │  │                 │      │ • Caching/CDN           │   │    │
│  │  │ Use for:        │      │ • Session affinity      │   │    │
│  │  │ Non-HTTP global │      │                         │   │    │
│  │  │ load balancing  │      │ Use for:                │   │    │
│  │  │                 │      │ Global HTTP/HTTPS apps  │   │    │
│  │  └─────────────────┘      └─────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                   REGIONAL (Single Region)               │    │
│  │                                                          │    │
│  │  Load Balancer (L4)        Application Gateway (L7)     │    │
│  │  ┌─────────────────┐      ┌─────────────────────────┐   │    │
│  │  │ • TCP/UDP only  │      │ • HTTP/HTTPS only       │   │    │
│  │  │ • High perf     │      │ • SSL termination       │   │    │
│  │  │ • Port forward  │      │ • WAF (optional)        │   │    │
│  │  │ • Zone-redundant│      │ • URL routing           │   │    │
│  │  │                 │      │ • Rewrite rules         │   │    │
│  │  │ Use for:        │      │ • Autoscaling           │   │    │
│  │  │ Non-HTTP traffic│      │                         │   │    │
│  │  │ Internal LB     │      │ Use for:                │   │    │
│  │  │                 │      │ Web app load balancing  │   │    │
│  │  └─────────────────┘      └─────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Decision Matrix:

| Requirement | Best Choice |
|-------------|-------------|
| Global HTTP/HTTPS with CDN | Front Door |
| Global non-HTTP failover | Traffic Manager |
| Regional HTTP with WAF | Application Gateway |
| Regional TCP/UDP | Load Balancer (Standard) |
| Internal load balancing | Internal Load Balancer |

---

## Containers & Kubernetes

### 🟢 Basic Questions

#### Q8: What is Azure Kubernetes Service (AKS) and what are its key features?

**Basic Answer:**
AKS is a managed Kubernetes service that simplifies deploying, managing, and scaling containerized applications. Azure manages the control plane while you manage the worker nodes.

**Advanced Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    AKS ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Azure-Managed Control Plane                 │    │
│  │                    (FREE)                                │    │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────────────┐   │    │
│  │  │ API Server │ │    etcd    │ │ Controller Manager │   │    │
│  │  └────────────┘ └────────────┘ └────────────────────┘   │    │
│  │  ┌────────────┐ ┌────────────┐                          │    │
│  │  │ Scheduler  │ │ Cloud Ctrl │                          │    │
│  │  └────────────┘ └────────────┘                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Customer-Managed Node Pools                 │    │
│  │                   (You pay for VMs)                      │    │
│  │                                                          │    │
│  │  System Pool (required)      User Pool (optional)        │    │
│  │  ┌─────────────────────┐    ┌─────────────────────┐     │    │
│  │  │ • CoreDNS           │    │ • Application pods  │     │    │
│  │  │ • Metrics Server    │    │ • Can use Spot VMs  │     │    │
│  │  │ • Tunnelfront       │    │ • Auto-scaling      │     │    │
│  │  │ • kube-proxy        │    │ • Taint/tolerations │     │    │
│  │  └─────────────────────┘    └─────────────────────┘     │    │
│  │                                                          │    │
│  │  Node configuration:                                     │    │
│  │  • VM Size: Standard_DS2_v2 (default)                   │    │
│  │  • OS: Ubuntu, Windows, Azure Linux                     │    │
│  │  • Disk: Ephemeral OS disk recommended                  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Key Features:                                                   │
│  • Azure AD integration for RBAC                                │
│  • Azure CNI or Kubenet networking                              │
│  • Azure Policy for Kubernetes                                  │
│  • Cluster autoscaler                                           │
│  • Azure Monitor for containers                                 │
│  • GitOps with Flux                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Expert Answer:**

AKS Networking Options:

| Feature | Kubenet | Azure CNI | Azure CNI Overlay |
|---------|---------|-----------|-------------------|
| Pod IPs | NAT'd | VNet IPs | Overlay network |
| IP consumption | Low | High | Low |
| Network policies | Calico | Azure + Calico | Azure + Calico |
| Performance | Good | Best | Good |
| Windows nodes | No | Yes | Yes |

AKS Security Best Practices:
```yaml
# Enable Azure Policy
az aks enable-addons --addons azure-policy --name myAKSCluster --resource-group myResourceGroup

# Use managed identity
az aks update --enable-managed-identity --name myAKSCluster --resource-group myResourceGroup

# Enable Defender for Containers
# (via Azure Portal or ARM template)

# Implement Pod Security Standards
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
```

---

## Troubleshooting Scenarios

### Scenario 1: AKS Pod Stuck in Pending State

**Problem:** Pods are stuck in Pending state and not being scheduled.

**Diagnostic Steps:**

```
┌─────────────────────────────────────────────────────────────────┐
│              AKS POD PENDING TROUBLESHOOTING                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Step 1: Check pod events                                       │
│  kubectl describe pod <pod-name> -n <namespace>                 │
│  Look for: FailedScheduling events                              │
│                                                                  │
│  Step 2: Check node resources                                   │
│  kubectl describe nodes | grep -A 5 "Allocated resources"       │
│                                                                  │
│  Common Causes:                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  1. Insufficient Resources                               │    │
│  │     Event: "Insufficient cpu" or "Insufficient memory"   │    │
│  │     Fix: Scale node pool or reduce resource requests     │    │
│  │                                                          │    │
│  │  2. Node Selector/Affinity Not Satisfied                 │    │
│  │     Event: "node(s) didn't match node selector"          │    │
│  │     Fix: Check nodeSelector and affinity rules           │    │
│  │                                                          │    │
│  │  3. Taints Not Tolerated                                 │    │
│  │     Event: "node(s) had taints that pod didn't tolerate" │    │
│  │     Fix: Add tolerations to pod spec                     │    │
│  │                                                          │    │
│  │  4. PVC Not Bound                                        │    │
│  │     Event: "persistentvolumeclaim not found"             │    │
│  │     Fix: Check PVC status and storage class              │    │
│  │                                                          │    │
│  │  5. Image Pull Issues                                    │    │
│  │     Status: ImagePullBackOff                             │    │
│  │     Fix: Check ACR access, image name, credentials       │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Resolution Commands:                                            │
│  # Scale cluster                                                │
│  az aks scale --resource-group myRG --name myAKS --node-count 5 │
│                                                                  │
│  # Enable cluster autoscaler                                    │
│  az aks update --resource-group myRG --name myAKS \             │
│    --enable-cluster-autoscaler --min-count 1 --max-count 10     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Architecture Design Questions

### Design 1: Azure Landing Zone for Enterprise

**Requirements:**
- 50+ application teams
- Compliance: GDPR, SOC2
- Multi-region deployment
- Centralized governance

**Solution:**

```
┌─────────────────────────────────────────────────────────────────┐
│              AZURE LANDING ZONE ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  ROOT MANAGEMENT GROUP                   │    │
│  │              (Tenant-wide policies)                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                            │                                     │
│      ┌────────────────────┬┴───────────────────┐                │
│      ▼                    ▼                    ▼                │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐         │
│  │  Platform   │    │  Workloads  │    │  Sandboxes  │         │
│  │     MG      │    │     MG      │    │     MG      │         │
│  └─────────────┘    └─────────────┘    └─────────────┘         │
│        │                  │                                      │
│  ┌─────┴─────┐      ┌─────┴─────┐                               │
│  │           │      │           │                               │
│  ▼           ▼      ▼           ▼                               │
│ Identity  Networking  Production  Non-Production                │
│                                                                  │
│  PLATFORM SUBSCRIPTIONS:                                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Identity Subscription:                                  │    │
│  │  - Azure AD Connect servers                              │    │
│  │  - Domain Controllers (if needed)                        │    │
│  │  - Azure AD DS (if needed)                               │    │
│  │                                                          │    │
│  │  Connectivity Subscription:                              │    │
│  │  - Hub VNet                                              │    │
│  │  - Azure Firewall                                        │    │
│  │  - ExpressRoute/VPN Gateways                             │    │
│  │  - DNS zones                                             │    │
│  │                                                          │    │
│  │  Management Subscription:                                │    │
│  │  - Log Analytics workspace                               │    │
│  │  - Automation accounts                                   │    │
│  │  - Azure Monitor                                         │    │
│  │  - Security Center                                       │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  HUB-SPOKE NETWORK TOPOLOGY:                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │        ┌─────────────────────────────────┐              │    │
│  │        │           HUB VNET              │              │    │
│  │        │     (Connectivity Sub)          │              │    │
│  │        │                                 │              │    │
│  │        │  ┌─────────┐  ┌─────────┐      │              │    │
│  │        │  │Azure FW │  │VPN/ER GW│      │              │    │
│  │        │  └─────────┘  └─────────┘      │              │    │
│  │        │                                 │              │    │
│  │        └──────────────┬──────────────────┘              │    │
│  │                       │                                  │    │
│  │    ┌──────────────────┼──────────────────┐              │    │
│  │    │                  │                  │              │    │
│  │    ▼                  ▼                  ▼              │    │
│  │  Spoke 1           Spoke 2           Spoke 3            │    │
│  │  (App Team 1)      (App Team 2)      (App Team 3)       │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📚 Documentation Links

### Official Microsoft Documentation
- [Azure Architecture Center](https://docs.microsoft.com/azure/architecture/)
- [Azure Well-Architected Framework](https://docs.microsoft.com/azure/architecture/framework/)
- [Azure Landing Zones](https://docs.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/)
- [AKS Documentation](https://docs.microsoft.com/azure/aks/)
- [Azure Networking](https://docs.microsoft.com/azure/networking/)

### Microsoft Learn
- [AZ-104: Azure Administrator](https://docs.microsoft.com/learn/certifications/azure-administrator/)
- [AZ-305: Azure Solutions Architect](https://docs.microsoft.com/learn/certifications/azure-solutions-architect/)
- [AZ-400: Azure DevOps Engineer](https://docs.microsoft.com/learn/certifications/devops-engineer/)

### Azure Whitepapers
- [Security Best Practices](https://docs.microsoft.com/azure/security/fundamentals/best-practices-and-patterns)
- [Networking Best Practices](https://docs.microsoft.com/azure/cloud-adoption-framework/ready/azure-best-practices/plan-for-traffic-inspection)

---

**[← Back to Main README](../README.md)** | **[Next: Kubernetes →](../kubernetes/README.md)**
