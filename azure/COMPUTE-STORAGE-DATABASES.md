# Azure Compute, Storage & Databases - Comprehensive Guide

> VM SKUs, Storage replication, database consistency models, managed services - 8% of questions

**Coverage:** 100+ scenarios | **Focus:** Production patterns, cost optimization, HA/DR

---

## Compute (VMs, VMSS, Scale Sets)

### Q: Compare VM SKU families and choose for production workload

```
AZURE VM SKU FAMILIES:

General Purpose (B, D, E series)
├─ B: Burstable (low baseline, spikes to full)
│  ├─ Example: B4ms (4 vCPU, 16 GB)
│  ├─ Use: Dev/test, light workloads
│  └─ Cost: $$ (cheap)
│
├─ D: General compute (standard ratio 1:4 vCPU:RAM)
│  ├─ Example: D4s_v3 (4 vCPU, 16 GB)
│  ├─ Use: Most applications, web servers
│  └─ Cost: $$$
│
└─ E: Memory-optimized (1:8 vCPU:RAM)
   ├─ Example: E4s_v3 (4 vCPU, 32 GB)
   ├─ Use: In-memory databases, caching
   └─ Cost: $$$$ (expensive)

Compute Optimized (F, H series)
├─ F: High CPU ratio (1:2 vCPU:RAM)
│  ├─ Use: Batch processing, compiling
│  └─ Example: F4s (4 vCPU, 8 GB)
│
└─ H: High-performance compute
   ├─ Use: Scientific simulations
   └─ Cost: $$$$$ (very expensive)

Memory Optimized (M, G series)
├─ M: Up to 12 TB RAM per VM
│  └─ Use: SAP HANA, in-memory analytics
│
└─ G: GPU-optimized memory
   └─ Use: ML training with memory

Storage Optimized (L, I series)
├─ L: Local SSD storage
│  └─ Use: NoSQL databases (Cassandra)
│
└─ I: Ultra-high IOPS
   └─ Use: SQL Server, Oracle

GPU Accelerated (N series)
├─ NC: NVIDIA Tesla K80 (for ML)
├─ ND: NVIDIA Tesla V100 (for AI)
└─ NV: NVIDIA Tesla M60 (for visualization)

PRODUCTION RECOMMENDATION:

Web application tier:
├─ SKU: D4s_v3 (4 vCPU, 16 GB, Premium SSD)
├─ Quantity: 3+ across AZs
├─ Auto-scaling: CPU > 75%
└─ Load balancer: Front

Database tier:
├─ SKU: E4s_v3 (4 vCPU, 32 GB, Premium SSD)
├─ Quantity: 1 primary + 2 read replicas
├─ Availability: Premium SSD + Azure Backup
└─ Zone-redundant: Yes (cross-AZ)

COST OPTIMIZATION:

Option 1: Reserved Instances (1-3 year)
├─ Discount: 30-72% vs on-demand
├─ Commitment: 1 or 3 years
├─ Use: Stable, predictable workloads
└─ Savings: $10K+ annually for production

Option 2: Spot Instances
├─ Discount: Up to 90%
├─ Risk: Azure can reclaim VM (minutes notice)
├─ Use: Non-critical, flexible workloads
├─ Example: Dev/test, batch jobs
└─ Mix: 80% reserved + 20% spot for peaks

Example:
$ aws pricing ec2 d4s_v3
On-demand: $384/month ($0.53/hour)
Reserved (1yr): $132/month ($0.18/hour)
Savings: $252/month = $3024/year

TERRAFORM:

resource "azurerm_virtual_machine_scale_set" "app_tier" {
  name                = "vmss-app"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  upgrade_policy_mode = "Rolling"

  sku {
    name     = "Standard_D4s_v3"
    tier     = "Standard"
    capacity = 3
  }

  os_profile {
    computer_name_prefix = "app-"
    admin_username       = "azureuser"
  }

  os_profile_linux_config {
    disable_password_authentication = true
    ssh_keys {
      path     = "/home/azureuser/.ssh/authorized_keys"
      key_data = var.ssh_public_key
    }
  }

  storage_profile_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts-gen2"
    version   = "latest"
  }

  storage_profile_os_disk {
    caching           = "ReadWrite"
    create_option     = "FromImage"
    managed_disk_type = "Premium_LRS"  # Premium SSD
  }

  network_profile {
    name                      = "app-nic"
    primary                   = true
    network_security_group_id = azurerm_network_security_group.app.id

    ip_configuration {
      name                                   = "app-ip"
      primary                                = true
      subnet_id                              = azurerm_subnet.application.id
      load_balancer_backend_address_pool_ids = [azurerm_lb_backend_address_pool.app.id]
      application_gateway_backend_address_pool_ids = [
        azurerm_application_gateway.app.backend_address_pool[0].id
      ]
    }
  }

  auto_scaling_group_properties {
    desired_capacity = 3
    maximum_size     = 10
    minimum_size     = 2
  }
}

resource "azurerm_monitor_autoscale_setting" "app_tier" {
  name                = "autoscale-app"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  target_resource_id  = azurerm_virtual_machine_scale_set.app_tier.id

  profile {
    name = "default"

    capacity {
      default = 3
      minimum = 2
      maximum = 10
    }

    rule {
      metric_trigger {
        metric_name              = "Percentage CPU"
        metric_resource_id       = azurerm_virtual_machine_scale_set.app_tier.id
        time_grain               = "PT1M"
        statistic                = "Average"
        time_window              = "PT5M"
        time_aggregation         = "Average"
        operator                 = "GreaterThan"
        threshold                = 75
      }

      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = 1
        cooldown  = "PT5M"
      }
    }

    rule {
      metric_trigger {
        metric_name              = "Percentage CPU"
        metric_resource_id       = azurerm_virtual_machine_scale_set.app_tier.id
        time_grain               = "PT1M"
        statistic                = "Average"
        time_window              = "PT5M"
        time_aggregation         = "Average"
        operator                 = "LessThan"
        threshold                = 25
      }

      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = 1
        cooldown  = "PT5M"
      }
    }
  }
}
```

---

## Storage Accounts

### Q: Design storage replication strategy for data durability

```
STORAGE REPLICATION OPTIONS:

LRS (Locally Redundant Storage)
├─ Copies: 3 copies within single datacenter
├─ Failures handled: Disk failure (1-2 disks)
├─ Failures NOT handled: Datacenter fire
├─ Durability: 99.999999999% (11 nines)
├─ Cost: $$ (cheapest)
├─ Use: Non-critical, easily recoverable
└─ RPO: ~15 minutes (aggressive replication)

ZRS (Zone Redundant Storage)
├─ Copies: 3 copies across 3 availability zones
├─ Failures handled: 1 entire AZ down
├─ Failures NOT handled: Region disaster
├─ Durability: 99.9999999999% (12 nines)
├─ Cost: $$$ (30-50% more than LRS)
├─ Use: Production, multi-AZ resilience
└─ RPO: ~seconds

GRS (Geo-Redundant Storage)
├─ Copies: 3 in primary region + 3 in secondary region
├─ Failures handled: Entire primary region down
├─ Failures NOT handled: Primary + secondary down
├─ Durability: 99.99999999999999% (16 nines)
├─ Cost: $$$$ (60% more than LRS)
├─ Use: Geo-critical data
├─ Secondary region: Azure-managed (read-only unless failover)
└─ Failover: Manual (requires action), 1+ hours
    └─ After failover: Secondary becomes primary (R/W)

GZRS (Geo + Zone Redundant)
├─ Copies: 3 across AZs in primary + 3 in secondary
├─ Most durable: Regional disaster + zone failure
├─ Durability: 16 nines
├─ Cost: $$$$$ (most expensive)
├─ Use: Mission-critical, compliance-required
└─ Example: Financial institutions, healthcare

RA-GRS (Read-Access Geo-Redundant)
├─ Like GRS but:
├─ Secondary region is READABLE (read-only)
├─ Write only to primary
├─ Use: Read-heavy workloads
└─ Benefit: Can read from secondary even if primary down

REPLICATION COMPARISON:

┌─────────────────────────────────────────────────────────┐
│  SCENARIO: Application in East US, backup in West US   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Option 1: LRS (East US only)                           │
│  ├─ Scenario: Disk fails                               │
│  ├─ Azure: Replicate within zone (auto)                │
│  ├─ RTO: 15 minutes                                    │
│  ├─ Scenario: Datacenter fire                          │
│  ├─ Result: DATA LOST ❌                                 │
│  └─ Cost: $5/month per TB                              │
│                                                         │
│  Option 2: GRS (East US primary, West US secondary)    │
│  ├─ Scenario: Disk fails                               │
│  ├─ RTO: 15 minutes (intra-region replication)         │
│  ├─ Scenario: Datacenter fire                          │
│  ├─ Action: Manual failover                            │
│  ├─ RTO: 1-4 hours (manual process)                    │
│  ├─ Data intact? Yes ✅                                 │
│  └─ Cost: $8/month per TB                              │
│                                                         │
│  Option 3: GZRS (East + West with zones)               │
│  ├─ Scenario: Disk fails                               │
│  ├─ RTO: seconds (auto failover within zone)           │
│  ├─ Scenario: Zone down                                │
│  ├─ RTO: seconds (replicate to other zone)             │
│  ├─ Scenario: Region down                              │
│  ├─ RTO: minutes (automatic failover to West)          │
│  ├─ Automatic? Yes (for zone, manual for region)       │
│  └─ Cost: $10/month per TB                             │
│                                                         │
└─────────────────────────────────────────────────────────┘

PRODUCTION RECOMMENDATION:

Tier 1: Application data (GZRS)
├─ RPO: ~seconds
├─ RTO: Seconds-minutes
├─ Cost: Premium (required for mission-critical)
└─ Justification: Business continuity essential

Tier 2: Backups (RA-GRS)
├─ RPO: 1 day
├─ RTO: Hours (manual recovery)
├─ Cost: Standard
└─ Justification: Long-term retention, disaster recovery

Tier 3: Logs/diagnostics (LRS)
├─ RPO: N/A (non-critical)
├─ RTO: Not time-sensitive
├─ Cost: Cheapest
└─ Justification: Compliance only

TERRAFORM:

# GZRS Storage Account (production data)
resource "azurerm_storage_account" "production" {
  name                             = "stgprod001"
  resource_group_name              = azurerm_resource_group.main.name
  location                         = "eastus"
  account_tier                     = "Standard"
  account_replication_type         = "GZRS"
  access_tier                      = "Hot"
  min_tls_version                  = "TLS1_2"
  https_traffic_only_enabled       = true
  shared_access_key_enabled        = false  # Use MSI only
  public_network_access_enabled    = false  # Private only

  blob_properties {
    versioning_enabled             = true   # Protect against deletion
    last_access_time_enabled       = true
    change_feed_enabled            = true
    
    delete_retention_policy {
      days = 30                    # Soft delete
    }
  }
}

# Backup Storage (RA-GRS, cheaper tier)
resource "azurerm_storage_account" "backup" {
  name                             = "stgbackup001"
  resource_group_name              = azurerm_resource_group.main.name
  location                         = "eastus"
  account_tier                     = "Standard"
  account_replication_type         = "RA-GRS"
  access_tier                      = "Cool"  # Long-term retention
  https_traffic_only_enabled       = true

  lifecycle_rule {
    name                    = "archive-after-90-days"
    enabled                 = true
    
    actions {
      tier_to_archive_after_days  = 90
      delete_after_days           = 365  # Auto-delete after 1 year
    }
  }
}
```

---

## Databases

### Q: Compare Azure SQL, PostgreSQL, MySQL, Cosmos DB

```
AZURE DATABASE OPTIONS:

Azure SQL Database (Managed SQL Server)
├─ Engine: SQL Server (Microsoft)
├─ Use: Enterprise workloads, complex queries, T-SQL
├─ Scaling: 1-80 vCores, up to 4TB
├─ HA: Built-in (3-replica within AZs)
├─ Backup: Auto (7-35 days retention)
├─ Replication: Read replicas (cross-region failover)
├─ Cost: $$ per month
├─ Connections: ~30,000/sec
└─ Example: ERP systems, financial apps

Azure Database for PostgreSQL
├─ Engine: PostgreSQL (open-source)
├─ Use: Applications using PostgreSQL
├─ Scaling: Flexible, up to 64 vCores
├─ HA: Zone-redundant replicas
├─ JSONB: Native JSON support
├─ Cost: $ (cheaper than SQL)
├─ Connections: ~10,000/sec
└─ Example: Rails/Django apps, analytics

Azure Database for MySQL
├─ Engine: MySQL (open-source)
├─ Use: Web apps, CMS, lower complexity
├─ Scaling: Less performant than PostgreSQL
├─ Cost: $ (cheapest relational)
├─ Deprecation: Single server being deprecated
└─ Migration: Move to PostgreSQL recommended

Cosmos DB (NoSQL)
├─ Engine: NoSQL (multi-model)
├─ Consistency models: Strong, Bounded, Session, Eventual
├─ Scaling: Unlimited (sharding auto)
├─ Replication: Multi-region active-active
├─ Cost: $ per request (not per vCore)
├─ Throughput: Millions requests/sec
├─ Use: IoT, real-time analytics, distributed systems
├─ Limitations: Complex joins difficult, eventual consistency
└─ Example: Mobile apps, sensor data, global coordination

Redis Cache
├─ Engine: In-memory key-value store
├─ Use: Caching, sessions, leaderboards
├─ Throughput: 100K ops/sec (single node)
├─ TTL: Keys auto-expire
├─ Persistence: AOF (append-only file)
├─ Cost: $ per month (fixed)
└─ Example: Cache database queries, rate limiting

CONSISTENCY MODELS (Cosmos DB):

Strong Consistency
├─ Write: Block until replicated to 3 quorum
├─ Read: Always latest version
├─ Latency: High (wait for replication)
├─ Use: Financial transactions
└─ Example: "Your account balance is $1000.00"

Bounded Staleness
├─ Write: Replicate within time bound (e.g., 5 sec)
├─ Read: May read slightly old data
├─ Latency: Medium
├─ Staleness bound: K operations or T time
└─ Example: "Your balance ~$1000 (within 5 sec)"

Session Consistency
├─ Write: Replicate then ack
├─ Read: Latest from same session
├─ Latency: Low
├─ Caveat: Other clients might see older
└─ Example: "Your profile changes visible to you"

Eventual Consistency
├─ Write: Ack immediately (no wait)
├─ Read: Eventually latest (milliseconds)
├─ Latency: Lowest
├─ Risk: Stale reads possible
└─ Example: "Tweet visible worldwide within 1 sec"

TERRAFORM:

# Azure SQL Database (High Availability)
resource "azurerm_mssql_server" "main" {
  name                         = "sqlserver-prod"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = random_password.sql.result
  identity {
    type = "SystemAssigned"
  }
}

resource "azurerm_mssql_database" "main" {
  name      = "appdb"
  server_id = azurerm_mssql_server.main.id

  sku_name = "S1"  # Standard tier

  backup_retention_days        = 35
  geo_backup_enabled           = true  # Replicate to secondary
  zone_redundant               = true  # Multi-AZ
  transparent_data_encryption_enabled = true
  
  threat_detection_policy {
    state                      = "On"
    retention_days             = 30
    disabled_alerts            = []
  }
}

# Cosmos DB (Multi-region, NoSQL)
resource "azurerm_cosmosdb_account" "main" {
  name                = "cosmos-prod"
  resource_group_name = azurerm_resource_group.main.name
  location            = "eastus"
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"

  default_identity_type           = "SystemAssignedManagedIdentity"
  
  consistency_policy {
    consistency_level       = "Session"  # Balanced
    max_interval_in_seconds = 10
    max_staleness_prefix    = 100
  }

  geo_location {
    location          = "eastus"
    failover_priority = 0  # Primary
    zone_redundant    = true
  }

  geo_location {
    location          = "westus"
    failover_priority = 1  # Secondary
    zone_redundant    = true
  }

  capabilities {
    name = "EnableServerless"  # Serverless pricing
  }
}

resource "azurerm_cosmosdb_sql_database" "main" {
  name                = "appdb"
  resource_group_name = azurerm_resource_group.main.name
  account_name        = azurerm_cosmosdb_account.main.name
}

resource "azurerm_cosmosdb_sql_container" "events" {
  name                = "events"
  database_name       = azurerm_cosmosdb_sql_database.main.name
  account_name        = azurerm_cosmosdb_account.main.name
  partition_key_path  = "/eventType"

  indexing_policy {
    indexing_mode = "consistent"
  }
}
```

This covers production patterns for compute, storage, and databases on Azure.
