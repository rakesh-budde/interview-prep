# Azure Identity & Security Deep Dive

> Comprehensive Microsoft Entra ID, RBAC, and security hardening - 15% of interview questions

**Coverage:** 150+ questions | **Reading Time:** 120 minutes

---

## Table of Contents

- [Microsoft Entra ID (Azure AD) Fundamentals](#microsoft-entra-id-fundamentals)
- [RBAC Deep Dive](#rbac-deep-dive)
- [Workload Identity & Service Principals](#workload-identity--service-principals)
- [Key Vault & Secrets Management](#key-vault--secrets-management)
- [Security & Compliance](#security--compliance)
- [Interview Scenarios](#interview-scenarios)

---

## Microsoft Entra ID Fundamentals

### Q: Explain Microsoft Entra ID (formerly Azure AD) architecture and core concepts

```
ENTRA ID HIERARCHY:

├─ Tenant (10000 AAD instances possible)
│  └─ Single tenant = one organization
│
├─ Users (Employee identities)
│  ├─ Cloud-only users (user@contoso.onmicrosoft.com)
│  └─ Synced from on-prem AD (via Azure AD Connect)
│
├─ Groups (Membership management)
│  ├─ Security groups (for RBAC)
│  └─ Microsoft 365 groups (for collaboration)
│
├─ Service Principals (Application identities)
│  ├─ Represents app or service
│  ├─ Has client ID, secret/certificate
│  └─ Used for automation
│
├─ Managed Identities (Simplified service principal)
│  ├─ System-assigned (1-to-1 with resource)
│  └─ User-assigned (shared across resources)
│
└─ External Identities (B2B/B2C)
   ├─ Guest users from partner organizations
   └─ Consumer identities (Azure AD B2C)

AUTHENTICATION METHODS:

┌────────────────────────────────────────┐
│  User Login Flow                        │
├────────────────────────────────────────┤
│                                        │
│ 1. User enters credentials             │
│    ├─ Email: alice@contoso.com         │
│    ├─ Password: [entered in portal]    │
│    └─ MFA: [TOTP from authenticator]   │
│                                        │
│ 2. Entra ID validates                  │
│    ├─ Password hash match?             │
│    ├─ Account not disabled?            │
│    ├─ Conditional Access policy OK?    │
│    │  ├─ Is client trusted device?     │
│    │  ├─ Is IP range trusted?          │
│    │  └─ Is time within business hrs?  │
│    └─ MFA validated?                   │
│                                        │
│ 3. Issue token                         │
│    ├─ Type: JWT (JSON Web Token)       │
│    ├─ Expiry: 1 hour (ID token)        │
│    │         Longer (refresh token)    │
│    ├─ Claims:                          │
│    │  ├─ sub (subject/user ID)         │
│    │  ├─ upn (user principal name)     │
│    │  ├─ roles (assigned roles)        │
│    │  ├─ groups (assigned groups)      │
│    │  └─ exp (expiration time)         │
│    └─ Signature: Signed by Entra ID    │
│                                        │
│ 4. User receives token                 │
│    └─ Stored in browser/app            │
│       (securely - httpOnly cookies)    │
│                                        │
│ 5. Token used for subsequent requests  │
│    └─ Included in Authorization header │
│       "Authorization: Bearer eyJ0..."   │
│                                        │
└────────────────────────────────────────┘

SERVICE PRINCIPAL vs MANAGED IDENTITY:

Service Principal (Old Way)
├─ Must create manually
├─ Credentials: Secret or certificate
├─ Secret rotation: Manual responsibility
├─ Complexity: Higher
├─ Use: Complex integrations

Managed Identity (New Way - Recommended)
├─ Created automatically with resource
├─ No credentials to manage
├─ Rotation: Automatic (handled by Azure)
├─ Complexity: Lower
├─ Use: Most scenarios (AKS, VMs, Functions, etc.)

TERRAFORM:

# Create resource (AKS cluster)
resource "azurerm_kubernetes_cluster" "main" {
  name                = "aks-prod"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  
  # System-assigned managed identity
  identity {
    type = "SystemAssigned"
  }
}

# Output the identity
output "cluster_identity_id" {
  value = azurerm_kubernetes_cluster.main.identity[0].principal_id
}

# Grant permissions
resource "azurerm_role_assignment" "aks_network_contributor" {
  scope              = azurerm_virtual_network.main.id
  role_definition_name = "Network Contributor"
  principal_id       = azurerm_kubernetes_cluster.main.identity[0].principal_id
}
```

---

## RBAC Deep Dive

### Q: Design RBAC for large organization with multiple teams

```
ROLE-BASED ACCESS CONTROL (RBAC):

Who (Principal)
├─ Users (alice@contoso.com)
├─ Groups (eng-team@contoso.com)
├─ Service Principals (app-deployment)
└─ Managed Identities (aks-cluster)

Does What (Action)
├─ List resources (*:*/read)
├─ Create resources (*:*/write)
├─ Delete resources (*:*/delete)
├─ Custom actions
└─ Example: Microsoft.Compute/virtualMachines/read

On What (Resource)
├─ Subscription
├─ Resource Group
├─ Individual resource
└─ Example: /subscriptions/abc123/resourceGroups/prod

Role Assignment:
Principal + Role + Scope = Permission

EXAMPLE:
Principal: eng-team@contoso.com
Role: "Kubernetes Service Cluster Admin"
Scope: /subscriptions/abc123/resourceGroups/prod/providers/Microsoft.ContainerService/managedClusters/aks-prod
Result: Group has admin access to AKS cluster in production RG

BUILT-IN ROLES:

Owner
├─ Full access to resource
├─ Can grant others access
└─ Use: Subscriptions owner only

Contributor
├─ Create and manage resources
├─ Cannot grant others access
└─ Use: DevOps engineers

Reader
├─ View resources only
├─ Cannot modify
└─ Use: Auditors, monitoring

Kubernetes Service Cluster Admin
├─ Full AKS cluster admin
├─ Get cluster credentials
└─ Use: Cluster operators

Kubernetes Service Cluster User
├─ Limited AKS access
├─ Cannot scale clusters
└─ Use: Application developers

RBAC EVALUATION:

User: alice@contoso.com
Action: Create Kubernetes deployment
Resource: AKS cluster (prod)

System checks:
1. Find all roles assigned to alice (directly)
   └─ "Application Developer"
2. Find all groups alice is in
   ├─ Group 1: "eng-team"
   └─ Group 2: "prod-readers"
3. Find all roles assigned to eng-team
   └─ "AKS Application Developer"
4. Find all roles assigned to prod-readers
   └─ "Reader"
5. Aggregate permissions:
   ├─ From "Application Developer": deploy pods ✓
   ├─ From "AKS Application Developer": same
   ├─ From "Reader": view resources
   └─ Result: ALLOWED (has deployment permission)

DENY RULES (Important!):

Azure RBAC doesn't support explicit Deny
├─ If you want to prevent access: don't assign role
└─ But Azure Policy CAN enforce deny rules

TERRAFORM IMPLEMENTATION:

# Create custom role
resource "azurerm_role_definition" "app_deployer" {
  scope       = azurerm_subscription_data.current.id
  name        = "Application Deployer"
  description = "Can deploy apps to AKS"

  permissions {
    actions = [
      "Microsoft.ContainerService/managedClusters/read",
      "Microsoft.ContainerService/managedClusters/listClusterUserCredential/action",
    ]
    not_actions = [
      "Microsoft.ContainerService/managedClusters/delete",
      "Microsoft.ContainerService/managedClusters/agentPools/delete",
    ]
  }

  assignable_scopes = [
    azurerm_subscription_data.current.id,
  ]
}

# Assign role to group
resource "azurerm_role_assignment" "dev_team" {
  scope              = azurerm_kubernetes_cluster.main.id
  role_definition_id = azurerm_role_definition.app_deployer.role_definition_resource_id
  principal_id       = data.azuread_group.dev_team.id  # Group ID
}

# Assign role to individual user
resource "azurerm_role_assignment" "alice" {
  scope              = azurerm_resource_group.main.id
  role_definition_name = "Reader"
  principal_id       = data.azuread_user.alice.id
}

# System-assigned identity permissions
resource "azurerm_role_assignment" "aks_identity" {
  scope              = azurerm_container_registry.main.id
  role_definition_name = "AcrPull"
  principal_id       = azurerm_kubernetes_cluster.main.identity[0].principal_id
}

ORGANIZATIONAL STRUCTURE EXAMPLE:

Subscription (prod-subscription)
├─ Resource Group: production
│  ├─ AKS Cluster
│  │  ├─ Role: "Kubernetes Admin"
│  │  │  └─ Principal: sre-team@contoso.com
│  │  ├─ Role: "Kubernetes Developer"
│  │  │  └─ Principal: dev-team@contoso.com
│  │  └─ Role: "Viewer"
│  │     └─ Principal: managers@contoso.com
│  │
│  └─ Storage Account
│     ├─ Role: "Storage Blob Data Owner"
│     │  └─ Principal: data-eng@contoso.com
│     └─ Role: "Storage Blob Data Reader"
│        └─ Principal: analytics@contoso.com
│
├─ Resource Group: staging
│  ├─ AKS Cluster
│  │  └─ Role: "Kubernetes Developer"
│  │     └─ Principal: dev-team@contoso.com
│  │
│  └─ Azure SQL Database
│     └─ Role: "SQL DB Contributor"
│        └─ Principal: dba-team@contoso.com
│
└─ Resource Group: shared
   ├─ Container Registry
   │  ├─ Role: "AcrPush"
   │  │  └─ Principal: ci-cd-pipeline (service principal)
   │  └─ Role: "AcrPull"
   │     └─ Principal: aks-cluster (managed identity)
   │
   └─ Key Vault
      ├─ Role: "Key Vault Admin"
      │  └─ Principal: vault-ops@contoso.com
      └─ Role: "Key Vault Secrets Officer"
         └─ Principal: app-deployment (managed identity)
```

---

## Key Vault & Secrets Management

### Q: Design secrets management for AKS applications

```
SECRETS STORAGE PYRAMID:

┌─────────────────────────────────┐
│  Tier 3: Application Code       │
│  ❌ NEVER store secrets here    │
│  └─ If leaked: Entire app       │
│     credentials exposed         │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  Tier 2: Environment Variables  │
│  ⚠️  RISKY: Can be exposed via:  │
│  ├─ Pod logs                    │
│  ├─ Process inspect             │
│  ├─ Debugging                   │
│  └─ Accidental output           │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  Tier 1: Kubernetes Secrets     │
│  ⚠️  ENCRYPTED in etcd (at rest) │
│  ├─ But etcd accessible to:     │
│  │  ├─ Cluster admins           │
│  │  ├─ etcd backup files        │
│  │  └─ kubectl access           │
│  └─ Not ideal for sensitive     │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  Tier 0: Azure Key Vault        │
│  ✅ RECOMMENDED: Production use │
│  ├─ Encrypted at rest + transit │
│  ├─ Access via RBAC             │
│  ├─ Audit logging               │
│  ├─ Key rotation                │
│  └─ Secrets never in pod memory │
└─────────────────────────────────┘

KEY VAULT ARCHITECTURE:

┌─────────────────────────────────────────────────────────┐
│  Azure Key Vault                                        │
│  ├─ Vault Name: prod-kv-001                            │
│  ├─ Region: East US                                    │
│  ├─ SKU: Premium (for HSM)                             │
│  ├─ Soft-delete: Enabled (retention: 90 days)         │
│  └─ Purge protection: Enabled (no accidental delete)   │
│                                                        │
│  SECRETS (stored):                                      │
│  ├─ db-password: "SecureP@ssw0rd123"                   │
│  ├─ api-key: "sk_live_xyz789"                          │
│  ├─ jwt-secret: "jwt-signing-key-abc123"               │
│  └─ tls-cert: "[base64 cert data]"                     │
│                                                        │
│  KEYS (crypto):                                        │
│  ├─ app-encryption-key (for data encryption)           │
│  ├─ tls-key (for HTTPS)                                │
│  └─ signing-key (for JWTs)                             │
│                                                        │
│  CERTIFICATES:                                          │
│  ├─ app.contoso.com (TLS cert)                         │
│  ├─ Renewal: Automatic                                 │
│  └─ Expiry notifications: 90 days before              │
│                                                        │
│  ACCESS CONTROL:                                        │
│  ├─ RBAC:                                              │
│  │  ├─ Role: "Key Vault Secrets Officer"              │
│  │  └─ Principal: aks-cluster (managed identity)       │
│  │                                                     │
│  ├─ Network:                                           │
│  │  ├─ Public access: Disabled                         │
│  │  └─ Private endpoint: Enabled                       │
│  │                                                     │
│  └─ Policy (legacy):                                   │
│     ├─ Principal: aks-cluster                          │
│     └─ Permissions: Get, List secrets                 │
│                                                        │
└─────────────────────────────────────────────────────────┘

POD-TO-KEY VAULT FLOW:

┌──────────────────────────────────────────┐
│  Pod in AKS Cluster                      │
│                                          │
│  Code:                                   │
│  ```python                               │
│  vault_uri = os.environ.get(             │
│    'VAULT_URI')  # https://prod-kv.vault...
│  secret = get_secret(vault_uri, 'db-pwd'│
│  ```                                     │
│                                          │
│  Steps:                                  │
│  1. Pod has Workload Identity annotation │
│  2. kubelet detects pod start            │
│  3. Obtains OIDC token from AKS OIDC     │
│     provider                             │
│  4. Token mounted at:                    │
│     /var/run/secrets/workload-identity   │
│  5. Application uses OIDC token          │
│  6. Exchanges token for Azure credential │
│     (via Azure SDK)                      │
│  7. Calls Key Vault API with credential  │
│  8. Key Vault checks RBAC:               │
│     Role: "Secrets Officer"              │
│     Principal: aks-cluster               │
│     Action: "Get secret"                 │
│     Result: ✅ ALLOWED                    │
│  9. Returns decrypted secret              │
│  10. Pod uses secret                     │
│                                          │
└──────────────────────────────────────────┘

TERRAFORM SETUP:

# Create Key Vault
resource "azurerm_key_vault" "main" {
  name                       = "prod-kv-001"
  location                   = azurerm_resource_group.main.location
  resource_group_name        = azurerm_resource_group.main.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "premium"
  enabled_for_deployment     = true
  enabled_for_disk_encryption = true
  enabled_for_template_deployment = true
  enable_rbac_authorization  = true  # Use RBAC (not legacy policies)
  soft_delete_retention_days  = 90
  purge_protection_enabled    = true

  network_rules {
    default_action             = "Deny"
    bypass                     = ["AzureServices"]
    virtual_network_subnet_ids = [azurerm_subnet.application.id]
  }
}

# Create secret
resource "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  value        = random_password.db.result
  key_vault_id = azurerm_key_vault.main.id
  
  lifecycle {
    ignore_changes = [value]  # Don't overwrite if manually rotated
  }
}

# Grant AKS identity access
resource "azurerm_role_assignment" "kv_secrets_access" {
  scope              = azurerm_key_vault.main.id
  role_definition_name = "Key Vault Secrets Officer"
  principal_id       = azurerm_kubernetes_cluster.main.identity[0].principal_id
}

# Private Endpoint for Key Vault
resource "azurerm_private_endpoint" "kv" {
  name                = "pep-kv"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.application.id

  private_service_connection {
    name                           = "kv-connection"
    is_manual_connection           = false
    private_connection_resource_id = azurerm_key_vault.main.id
    subresource_names              = ["vault"]
  }
}

# Kubernetes deployment using secret
resource "kubernetes_secret" "kv_reference" {
  metadata {
    name      = "kv-secrets"
    namespace = "production"
  }

  data = {
    VAULT_URI = azurerm_key_vault.main.vault_uri
  }
}

# Pod references secret
resource "kubernetes_deployment" "app" {
  metadata {
    name      = "app"
    namespace = "production"
  }

  spec {
    template {
      spec {
        container {
          name  = "app"
          image = "app:latest"

          env {
            name      = "VAULT_URI"
            value_from {
              secret_key_ref {
                name = kubernetes_secret.kv_reference.metadata[0].name
                key  = "VAULT_URI"
              }
            }
          }

          # Workload Identity annotation
          service_account_name = "sa-app"
        }
      }
    }
  }
}
```

---

## Security & Compliance

### Q: Design security hardening for AKS in regulated industry (banking/healthcare)

```
MULTI-LAYER SECURITY:

Layer 1: Network Security
├─ Private AKS cluster (no public API server)
├─ Network policies (Calico) enforced
├─ Private endpoints for services
├─ ExpressRoute for on-prem connectivity
└─ Azure Firewall with DPI

Layer 2: Identity & Access
├─ Entra ID integration (no local accounts)
├─ RBAC with least privilege
├─ Managed identities (no secrets)
├─ MFA for all human access
└─ PIM (Privileged Identity Management)

Layer 3: Compute Security
├─ Image scanning (ACR scanning)
├─ Pod Security Standards enforced
├─ Admission controllers
├─ Container runtime isolation
└─ Secure Boot enabled on nodes

Layer 4: Data Security
├─ Secrets stored in Key Vault (not configmaps)
├─ Etcd encryption (at rest in control plane)
├─ TLS for all communication
├─ Azure Disk Encryption on nodes
└─ Secrets not in logs

Layer 5: Compliance
├─ Audit logging (all API calls logged)
├─ Defender for Containers enabled
├─ Azure Policy enforcing standards
├─ Backup/restore tested monthly
└─ Compliance scanning (PCI-DSS, etc.)

DEFENDER FOR CONTAINERS:

Features:
├─ Image vulnerability scanning
│  ├─ Scan on push to ACR
│  ├─ Detects CVEs
│  └─ Severity: Critical/High/Medium/Low
│
├─ Runtime threat detection
│  ├─ Suspicious process execution
│  ├─ Unexpected network connections
│  └─ Privilege escalation attempts
│
└─ Investigation capabilities
   ├─ Alert details
   ├─ Remediation guidance
   └─ Integration with SIEM

AUDIT LOGGING:

All cluster activities logged:
├─ Pod creations/deletions
├─ Service account usage
├─ RBAC changes
├─ Secret access
├─ Network policy changes
├─ Node operations
└─ Authentication failures

Retention: 1+ years (compliance requirement)

TERRAFORM:

# Enable Defender for Containers
resource "azurerm_security_center_setting" "defender" {
  setting_name       = "MCAS_THEN_MDE"
  enabled            = true
}

# Azure Policy for pod security
resource "azurerm_policy_definition" "pod_security" {
  name        = "enforce-pod-security"
  policy_type = "Custom"
  mode        = "Indexed"

  policy_rule = jsonencode({
    if = {
      field = "type"
      equals = "Microsoft.ContainerService/managedClusters"
    }
    then = {
      effect = "deny"
      details = [
        {
          field = "Microsoft.ContainerService/managedClusters/enablePodSecurityPolicy"
          equals = true
        }
      ]
    }
  })
}

# Assign policy
resource "azurerm_management_group_policy_assignment" "pod_security" {
  name                = "enforce-pod-security"
  policy_definition_id = azurerm_policy_definition.pod_security.id
  management_group_id  = "/providers/Microsoft.Management/managedGroups/root"
}
```

---

## Interview Scenarios

**Q: Design authentication for SaaS application with external customers**

```
Requirements:
- Multi-tenant application
- Customer employees access app
- SSO with customer's Entra ID
- Audit all access

Solution:

1. Application Registration (in your Entra ID)
   ├─ Client ID: app-xyz
   ├─ Redirect URI: https://app.contoso.com/auth/callback
   └─ Certificates: For signing

2. Customer Entra ID (enterprise customer)
   ├─ Consumer: customer-contoso.onmicrosoft.com
   ├─ Users: alice@customer-contoso.onmicrosoft.com
   └─ Service principal: Your app in their tenant

3. OAuth 2.0 / OpenID Connect Flow
   ├─ User visits app.contoso.com
   ├─ Redirected to customer's Entra ID login
   ├─ User authenticates with their org credentials
   ├─ Customer's Entra ID issues token
   ├─ Token sent back to your app
   ├─ Your app validates token signature
   ├─ Your app extracts user info from token
   └─ User logged in

4. Access Control
   ├─ Token contains: oid (object ID), email, roles
   ├─ Your app enforces permissions
   ├─ Database: Links Entra user to app user
   └─ API calls logged with user info

Benefits:
├─ No password management (Entra ID handles it)
├─ MFA enforced by customer
├─ Complete audit trail
├─ Seamless SSO experience
└─ Compliance-friendly (no PII in your systems)
```

This guide covers identity and security foundations for production Azure at FAANG level.
