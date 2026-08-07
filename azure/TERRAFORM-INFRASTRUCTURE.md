# Terraform for Azure Infrastructure - Production Patterns

> Infrastructure-as-Code mastery for AKS and multi-region deployments - 10% of questions

**Coverage:** 120+ scenarios | **Focus:** Production patterns, state management, drift detection

---

## Core Terraform Concepts

### Q: Design Terraform structure for multi-region AKS deployment

```
TERRAFORM PROJECT STRUCTURE (Recommended):

terraform/
├─ shared/                          # Shared resources
│  ├─ main.tf                      # Provider config, state
│  ├─ variables.tf                 # Input variables
│  ├─ outputs.tf                   # Outputs for other modules
│  ├─ acr.tf                       # Container Registry
│  ├─ keyvault.tf                  # Key Vault
│  ├─ storage.tf                   # Storage accounts
│  └─ terraform.tfvars             # ⚠️ Secret values (gitignore!)
│
├─ networking/                      # Network resources
│  ├─ main.tf                      # VNets, subnets
│  ├─ variables.tf
│  ├─ outputs.tf
│  ├─ vnet.tf                      # Virtual networks
│  ├─ nsg.tf                       # Network security groups
│  ├─ firewall.tf                  # Azure Firewall
│  ├─ dns.tf                       # DNS zones
│  └─ routes.tf                    # Route tables
│
├─ aks-east/                        # AKS in East US
│  ├─ main.tf                      # Cluster config
│  ├─ variables.tf
│  ├─ outputs.tf
│  ├─ cluster.tf                   # AKS cluster
│  ├─ node-pools.tf                # Node pool definitions
│  └─ addons.tf                    # Monitoring, ingress, etc.
│
├─ aks-west/                        # AKS in West US
│  ├─ main.tf
│  ├─ variables.tf
│  ├─ outputs.tf
│  └─ cluster.tf
│
├─ monitoring/                      # Observability stack
│  ├─ main.tf
│  ├─ logs.tf                      # Log Analytics workspace
│  ├─ alerts.tf                    # Alert rules
│  └─ dashboards.tf                # Azure dashboards
│
├─ environments/                    # Environment configs
│  ├─ dev.tfvars
│  ├─ staging.tfvars
│  ├─ prod.tfvars
│  └─ README.md
│
├─ modules/                         # Reusable modules
│  ├─ aks-cluster/                 # Generic AKS module
│  │  ├─ main.tf
│  │  ├─ variables.tf
│  │  ├─ outputs.tf
│  │  └─ versions.tf
│  │
│  ├─ vnet/                        # Generic VNet module
│  │  ├─ main.tf
│  │  ├─ variables.tf
│  │  └─ outputs.tf
│  │
│  └─ monitoring/                  # Generic monitoring module
│     ├─ main.tf
│     ├─ variables.tf
│     └─ outputs.tf
│
├─ .gitignore                       # Ignore sensitive files
│  ├─ *.tfvars (secrets)
│  ├─ .terraform/
│  ├─ *.tfstate* (state files)
│  └─ crash.log
│
├─ backend.tf                       # Remote state config
├─ providers.tf                     # Provider versions
├─ versions.tf                      # Terraform version constraints
└─ README.md                        # Documentation

STATE MANAGEMENT (Critical!):

Local State (Development Only):
├─ File: terraform.tfstate (local disk)
├─ Risk: Easy to lose/corrupt
├─ Not suitable for teams
└─ Use for: Local testing only

Remote State (Production):
├─ Backend: Azure Storage
├─ File: prod.tfstate (in Storage Account)
├─ Features:
│  ├─ Team sharing
│  ├─ Automatic locking (prevents concurrent edits)
│  ├─ Versioning
│  ├─ Backup retention
│  └─ Encryption at rest
│
├─ Benefits:
│  ├─ Single source of truth
│  ├─ CI/CD integration
│  └─ Disaster recovery

REMOTE STATE SETUP (Terraform):

# Backend configuration
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "tfstateprod001"
    container_name       = "tfstate"
    key                  = "prod.tfstate"
    # Note: Can also use -backend-config flag during init
  }
}

# OR Backend configuration (local file, for dev):
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}

MULTI-ENVIRONMENT SETUP:

Directory structure:
├─ environments/
│  ├─ dev/
│  │  ├─ terraform.tfvars
│  │  ├─ backend.tf (dev.tfstate)
│  │  └─ main.tf
│  │
│  ├─ prod/
│  │  ├─ terraform.tfvars
│  │  ├─ backend.tf (prod.tfstate)
│  │  └─ main.tf
│  │
│  └─ modules/  (shared)
│     ├─ aks/
│     ├─ networking/
│     └─ monitoring/

Commands per environment:
$ cd environments/dev
$ terraform init
$ terraform plan -out=plan.tfplan
$ terraform apply plan.tfplan

This ensures dev and prod have separate state files.

EXAMPLE: Multi-Region AKS

# root/variables.tf
variable "regions" {
  default = {
    primary   = "eastus"
    secondary = "westus"
  }
}

variable "environment" {
  default = "production"
}

# root/main.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }

  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "tfstateaks001"
    container_name       = "tfstate"
    key                  = "prod-multiregion.tfstate"
  }
}

provider "azurerm" {
  features {
    virtual_machine {
      delete_os_disk_on_delete            = true
      graceful_shutdown                   = true
      skip_shutdown_and_force_delete       = false
    }
    key_vault {
      purge_soft_delete_on_destroy = true
    }
  }

  subscription_id = var.subscription_id
}

# root/main.tf continued - Deploy AKS in each region
module "aks_primary" {
  source = "./modules/aks-cluster"
  
  cluster_name        = "aks-prod-east"
  location            = var.regions.primary
  resource_group_name = azurerm_resource_group.primary.name
  
  cluster_version  = "1.28.0"
  node_count       = 3
  vm_size          = "Standard_D4s_v3"
}

module "aks_secondary" {
  source = "./modules/aks-cluster"
  
  cluster_name        = "aks-prod-west"
  location            = var.regions.secondary
  resource_group_name = azurerm_resource_group.secondary.name
  
  cluster_version  = "1.28.0"
  node_count       = 3
  vm_size          = "Standard_D4s_v3"
}

# Outputs for use by other teams
output "primary_cluster_id" {
  value = module.aks_primary.cluster_id
}

output "secondary_cluster_id" {
  value = module.aks_secondary.cluster_id
}
```

---

## Advanced Patterns

### Q: Implement drift detection and policy enforcement

```
DRIFT DETECTION:

Problem: Manual changes made outside Terraform
├─ Someone SSHes into VM and edits config
├─ Azure Portal change (NSG rule modified)
├─ Script made a change
└─ Terraform doesn't know about it

Detection:
$ terraform plan
├─ Compares current state (tfstate file)
├─ Against actual Azure resources (by reading API)
├─ Reports differences (drift)
└─ Example: "NSG rule changed outside Terraform"

Solution - Automated Drift Detection:

# Schedule periodic plan in CI/CD
# Run daily: terraform plan → check for drift
# Alert if drift detected → investigate

resource "azurerm_resource_policy_assignment" "no_manual_changes" {
  name        = "prevent-manual-changes"
  policy_id   = azurerm_policy_definition.audit_changes.id
  scope       = azurerm_subscription.current.id

  # This policy requires all changes to come from Terraform
  # via managed identities in CI/CD
  parameters = jsonencode({
    listOfResourceTypesNotAllowed = {
      value = []  # All types can be created only by approved pipelines
    }
  })
}

POLICY AS CODE:

Azure Policy enforcement:
├─ No resources created without Terraform tags
├─ No public IPs on databases
├─ NSGs required on all subnets
├─ Encryption required on storage
├─ Budget caps enforced per resource group

Example:

resource "azurerm_policy_definition" "require_tags" {
  name        = "require-environment-tag"
  policy_type = "Custom"
  mode        = "All"

  policy_rule = jsonencode({
    if = {
      field = "tags"
      exists = "false"
    }
    then = {
      effect = "Deny"
    }
  })
}

resource "azurerm_policy_assignment" "require_tags" {
  name                = "require-tags"
  scope               = azurerm_resource_group.main.id
  policy_definition_id = azurerm_policy_definition.require_tags.id
}

Effect on user:
$ az vm create --resource-group prod \
              --name myvm \
              --image UbuntuLTS
Error: Policy violation - tags required
$ az vm create --resource-group prod \
              --name myvm \
              --image UbuntuLTS \
              --tags environment=prod
Success!

COMMON TERRAFORM MISTAKES:

❌ Mistake 1: Checking state files into Git
├─ Contains: Passwords, keys, secrets
├─ Visible to anyone with repo access
└─ Fix: Use .gitignore for *.tfstate*

❌ Mistake 2: Hard-coding values
├─ Example: subscription_id = "abc-123"
├─ Not portable across environments
└─ Fix: Use variables.tf

❌ Mistake 3: Not using remote state
├─ State on local disk
├─ Team members have different state
├─ Conflicts and corruption
└─ Fix: Always use backend (Azure Storage)

❌ Mistake 4: Destroying production accidentally
├─ $ terraform destroy (in prod directory!)
├─ All resources deleted
├─ Hours of recovery
└─ Fix: Use -prevent-destroy lifecycle rule

✅ Correct:
resource "azurerm_kubernetes_cluster" "prod" {
  name = "aks-prod"
  # ... config ...

  lifecycle {
    prevent_destroy = true  # Protect against accidents
  }
}

Then even `terraform destroy` will fail

STATE LOCKING:

Problem: Two people apply changes simultaneously
├─ Person A: Apply change 1
├─ Person B: Apply change 2 (before A finishes)
├─ Conflict: State corrupted
└─ Data inconsistent

Solution - State Locking:
├─ Person A acquires lock on state file
├─ Person B tries to acquire lock → WAIT
├─ Person A finishes, releases lock
├─ Person B acquires lock, proceeds
└─ No conflicts

Azure Storage automatically supports locking
(No config needed with azurerm backend)

DISASTER RECOVERY:

Scenario: Accidentally destroyed database
├─ $ terraform destroy (oops!)
└─ 200GB database gone

Recovery options:
├─ Option 1: Restore from backup (1-4 hours)
├─ Option 2: Recreate with terraform apply
│  ├─ Must restore data from backup
│  └─ Longer RTO
│
└─ Best: Backup your backups + Terraform configs

Prevention:
├─ Backup automation
├─ Backup validation (test restores monthly)
├─ Prevent destroy (lifecycle rule)
├─ Approval gates for production changes
└─ Private storage for state files

MULTI-REGION FAILOVER:

resource "azurerm_traffic_manager_profile" "main" {
  name                = "tm-prod"
  resource_group_name = azurerm_resource_group.main.name
  traffic_routing_method = "Priority"

  dns_config {
    relative_name = "tm-prod"
    ttl           = 300
  }

  monitor_config {
    protocol       = "HTTPS"
    port           = 443
    path           = "/health"
    interval_in_seconds = 30
    tolerated_number_of_failures = 3
    timeout_in_seconds = 10
  }
}

resource "azurerm_traffic_manager_endpoint" "primary" {
  name                = "primary-endpoint"
  profile_name        = azurerm_traffic_manager_profile.main.name
  resource_group_name = azurerm_resource_group.main.name
  type                = "azureEndpoints"
  target              = azurerm_public_ip.primary.ip_address
  endpoint_status     = "Enabled"
  priority            = 1  # Primary
}

resource "azurerm_traffic_manager_endpoint" "secondary" {
  name                = "secondary-endpoint"
  profile_name        = azurerm_traffic_manager_profile.main.name
  resource_group_name = azurerm_resource_group.main.name
  type                = "azureEndpoints"
  target              = azurerm_public_ip.secondary.ip_address
  endpoint_status     = "Enabled"
  priority            = 2  # Failover
}

Result:
├─ Primary healthy → Traffic to primary
├─ Primary down → Traffic to secondary (automatic)
└─ TTL: 300 sec (5 min failover time)
```

---

## Interview Questions

**Q: Design Terraform for SaaS platform with 20 customers**

```
Requirements:
- Each customer has dedicated resources
- Shared infrastructure (load balancer, storage)
- Different configurations per customer
- Easy to add new customer

Solution - Multi-Tenant Pattern:

# customers.tfvars
customers = {
  acme = {
    name           = "ACME Corp"
    tier           = "enterprise"
    node_count     = 5
    storage_gb     = 500
    backup_days    = 30
  }
  startup = {
    name           = "Startup Inc"
    tier           = "standard"
    node_count     = 2
    storage_gb     = 50
    backup_days    = 7
  }
}

# main.tf
resource "azurerm_resource_group" "customer" {
  for_each = var.customers
  
  name     = "rg-${each.key}-prod"
  location = var.location
}

module "aks_customer" {
  for_each = var.customers
  
  source = "./modules/aks-multi-tenant"
  
  customer_name  = each.key
  resource_group = azurerm_resource_group.customer[each.key].name
  config         = each.value
}

output "customer_clusters" {
  value = {
    for key, module in module.aks_customer :
    key => module.cluster_id
  }
}

Benefits:
├─ Add customer: Add 1 line to customers.tfvars
├─ Deploy: terraform apply (creates all customer RGs)
└─ Isolation: Each customer separate RG
```

This covers production Terraform patterns for multi-region AKS deployments.
