# Azure Networking Deep Dive - Comprehensive Interview Guide

> Production-grade Azure networking for FAANG interviews - 20% of questions

**Estimated Reading Time:** 150 minutes | **Coverage:** 200+ questions and scenarios

---

## Table of Contents

- [Virtual Networks & Subnets](#virtual-networks--subnets)
- [Network Security Groups](#network-security-groups)
- [Load Balancers & Gateways](#load-balancers--gateways)
- [Hybrid Connectivity](#hybrid-connectivity)
- [Private Endpoints & Service Endpoints](#private-endpoints--service-endpoints)
- [Azure DNS](#azure-dns)
- [Troubleshooting Flowcharts](#troubleshooting-flowcharts)
- [Interview Questions](#interview-questions)

---

## Virtual Networks & Subnets

### Q: Design a multi-tier production VNet with AKS, databases, and on-premises connectivity

**Architecture:**

```
Azure Region: East US
├─ Virtual Network (10.0.0.0/8)
│  ├─ Address Space: 10.0.0.0 - 10.255.255.255 (16 million IPs)
│  │
│  ├─ Subnet 1: System Pool (10.0.0.0/24)
│  │  ├─ Size: 256 IPs
│  │  ├─ Reserved: 5 (Azure reserves first 4, last 1)
│  │  ├─ Usable: 251 IPs
│  │  ├─ Nodes: 3 system pods
│  │  └─ Purpose: AKS system components
│  │
│  ├─ Subnet 2: Application Tier (10.0.1.0/21)
│  │  ├─ Size: 2048 IPs
│  │  ├─ Usable: 2043 IPs
│  │  ├─ Max pods: 240 (3 AZ × 80 pods/node)
│  │  ├─ Nodes: Up to 30 application nodes
│  │  └─ Auto-scaling: 2-30 nodes
│  │
│  ├─ Subnet 3: Database Tier (10.0.9.0/24)
│  │  ├─ Size: 256 IPs
│  │  ├─ Usable: 251 IPs
│  │  ├─ Azure Database: 3 IPs (primary + 2 replicas)
│  │  ├─ Private Endpoint: 1 IP
│  │  └─ Management: Reserved for future
│  │
│  ├─ Subnet 4: Gateway Subnet (10.0.10.0/24)
│  │  ├─ Size: 256 IPs
│  │  ├─ Purpose: VPN Gateway / ExpressRoute Gateway
│  │  ├─ Mandatory name: "GatewaySubnet"
│  │  └─ At least /27 required (minimum 32 IPs)
│  │
│  ├─ Subnet 5: Bastion (10.0.11.0/27)
│  │  ├─ Size: 32 IPs
│  │  ├─ Purpose: Azure Bastion for secure access
│  │  └─ Mandatory name: "AzureBastionSubnet"
│  │
│  └─ Subnet 6: Management (10.0.12.0/24)
│     ├─ Size: 256 IPs
│     ├─ Purpose: Management servers, jumpboxes
│     └─ NSG: Restrictive rules
│
├─ Network Security Groups (NSGs)
│  ├─ NSG-System (subnet-level rules)
│  ├─ NSG-Application
│  ├─ NSG-Database
│  ├─ NSG-Gateway (for VPN)
│  └─ NSG-Management
│
├─ Route Tables
│  ├─ RT-System (default routing)
│  ├─ RT-Application (default + custom routes)
│  ├─ RT-Database (restrict routing)
│  └─ RT-Management (restricted outbound)
│
└─ Connectivity
   ├─ VPN Gateway (Site-to-Site)
   │  ├─ Connect to on-premises datacenter
   │  ├─ Protocol: IPSec/IKEv2
   │  └─ Bandwidth: Up to 10Gbps
   │
   ├─ ExpressRoute (Dedicated circuit)
   │  ├─ Private connectivity to Azure
   │  ├─ SLA: 99.95%
   │  └─ Bandwidth: 50 Mbps - 100 Gbps
   │
   ├─ Application Gateway
   │  ├─ Listens: 10.0.12.0/24 (management subnet)
   │  ├─ Backend pool: 10.0.1.0/21 (app tier)
   │  └─ Public IP: 203.0.113.x (example)
   │
   └─ Azure Firewall
      ├─ Listens on: 10.0.14.0/24 (firewall subnet)
      ├─ Centralized logging
      └─ Application rules (FQDN-based)

CIDR Planning Best Practices:

1. Reserve /24 for each logical tier
   └─ Easy to remember and manage

2. Use /21 for high-scaling tiers
   └─ Supports 2000+ IPs for cloud-native scaling

3. Reserve 10% extra
   └─ Account for services, future growth

4. Never overlap
   └─ Document all CIDR ranges in a spreadsheet

5. Plan for expansion
   └─ If you outgrow 10.0.0.0/8, it's too late
   └─ Consider 10.0.0.0/8 + 172.16.0.0/12 from start
```

**Terraform Implementation:**

```hcl
# Create Virtual Network
resource "azurerm_virtual_network" "main" {
  name                = "vnet-production"
  address_space       = ["10.0.0.0/8"]
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  tags = {
    environment = "production"
    managed_by  = "terraform"
  }
}

# System subnet for AKS system components
resource "azurerm_subnet" "system" {
  name                 = "subnet-system"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.0.0/24"]

  service_endpoints = ["Microsoft.KeyVault", "Microsoft.Storage"]

  delegation {
    name = "delegation"
    service_delegation {
      name = "Microsoft.ContainerInstance/containerGroups"
    }
  }
}

# Application subnet for AKS nodes and pods
resource "azurerm_subnet" "application" {
  name                 = "subnet-application"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/21"]  # 2048 IPs

  service_endpoints = ["Microsoft.Sql", "Microsoft.Storage"]
}

# Database subnet
resource "azurerm_subnet" "database" {
  name                 = "subnet-database"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.9.0/24"]

  service_endpoints = ["Microsoft.Sql"]
}

# Gateway subnet (required for VPN/ExpressRoute)
resource "azurerm_subnet" "gateway" {
  name                 = "GatewaySubnet"  # Must be named exactly this
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.10.0/24"]  # Minimum /27
}
```

---

## Network Security Groups

### Q: Design NSG rules for multi-tier application (web → app → database)

**NSG Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│              NSG RULE EVALUATION (Layer 4/7)                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Inbound packet: Source 203.0.113.50:12345                 │
│                 Destination 10.0.1.10:80                   │
│                                                             │
│  NSG Rules (evaluated in order of priority):               │
│  ┌──────────────┬────┬─────┬──────────┬─────────────────┐  │
│  │ Priority     │ Dir│Type │ Protocol │ Source/Dest     │  │
│  ├──────────────┼────┼─────┼──────────┼─────────────────┤  │
│  │ 100          │In  │Allow│TCP       │80/443 from ALB  │  │
│  │ 110          │In  │Allow│TCP       │22 from jumpbox  │  │
│  │ 200          │In  │Allow│UDP       │53 DNS           │  │
│  │ 300          │In  │Allow│ICMP      │Any              │  │
│  │ 4096 (def)   │In  │Deny │Any       │Any (default)    │  │
│  └──────────────┴────┴─────┴──────────┴─────────────────┘  │
│                                                             │
│  MATCHING PROCESS:                                         │
│  1. Check rule 100: Source 203.0.113.50, Dest port 80    │
│     Match: YES → ALLOW (STOP)                             │
│                                                             │
│  RESULT: Packet ALLOWED                                   │
│                                                             │
│  If source was 203.0.113.100 (not in ALB range):          │
│  1. Check rule 100: Source not in range → SKIP            │
│  2. Check rule 110: Source not jumpbox → SKIP             │
│  3. Check rule 200: Protocol TCP, not UDP → SKIP          │
│  4. Check rule 300: Protocol TCP, not ICMP → SKIP         │
│  5. Check rule 4096 (default): Deny → DENY               │
│                                                             │
│  RESULT: Packet DENIED                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘

STATEFUL FIREWALL (Important!):

Inbound Rule: Allow TCP 80 from 203.0.113.0/24
├─ Opens port 80 for INBOUND traffic
├─ Automatically allows OUTBOUND response traffic
├─ No need for explicit egress rule for responses

Example Flow:
┌─────────────────────────────────────────┐
│ Client 203.0.113.50:12345               │
│ sends HTTP GET request                  │
└────────────────────┬────────────────────┘
                     │
                     │ Inbound (port 80) - Allowed by rule
                     ▼
        ┌────────────────────────────┐
        │ Azure VM 10.0.1.10:80      │
        │                            │
        │ Receives request           │
        │ Processes                  │
        └────────────┬───────────────┘
                     │
                     │ Outbound response - Automatically allowed
                     │ (due to connection tracking)
                     ▼
┌─────────────────────────────────────────┐
│ Client 203.0.113.50:12345               │
│ receives HTTP 200 response              │
└─────────────────────────────────────────┘

NSG Rules for Multi-Tier:

# NSG for Application Tier
# Inbound from Load Balancer
resource "azurerm_network_security_rule" "app_from_alb" {
  name                        = "allow-alb"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "80,443"
  source_address_prefix       = "10.0.12.0/24"  # ALB subnet
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.main.name
  network_security_group_name = azurerm_network_security_group.app.name
}

# Outbound to Database
resource "azurerm_network_security_rule" "app_to_db" {
  name                        = "allow-db"
  priority                    = 100
  direction                   = "Outbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "5432"  # PostgreSQL
  source_address_prefix       = "*"
  destination_address_prefix  = "10.0.9.0/24"  # DB subnet
  resource_group_name         = azurerm_resource_group.main.name
  network_security_group_name = azurerm_network_security_group.app.name
}

# Deny all other outbound (implicit allow everything else)
resource "azurerm_network_security_rule" "app_deny_all_out" {
  name                        = "deny-all-outbound"
  priority                    = 4096
  direction                   = "Outbound"
  access                      = "Deny"
  protocol                    = "*"
  source_port_range           = "*"
  destination_port_range      = "*"
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.main.name
  network_security_group_name = azurerm_network_security_group.app.name
}

# NSG for Database Tier
# Inbound only from Application tier
resource "azurerm_network_security_rule" "db_from_app" {
  name                        = "allow-app"
  priority                    = 100
  direction                   = "Inbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "5432"
  source_address_prefix       = "10.0.1.0/21"  # App subnet
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.main.name
  network_security_group_name = azurerm_network_security_group.db.name
}

# Deny all inbound
resource "azurerm_network_security_rule" "db_deny_all_in" {
  name                        = "deny-all-inbound"
  priority                    = 4096
  direction                   = "Inbound"
  access                      = "Deny"
  protocol                    = "*"
  source_port_range           = "*"
  destination_port_range      = "*"
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = azurerm_resource_group.main.name
  network_security_group_name = azurerm_network_security_group.db.name
}
```

---

## Load Balancers & Gateways

### Q: Compare Azure Load Balancer vs Application Gateway

**Comparison Matrix:**

```
┌────────────────────────────────────────────────────────────┐
│          LOAD BALANCER vs APPLICATION GATEWAY              │
├────────────────────────────────────────────────────────────┤
│                                                            │
│ Feature              Load Balancer   App Gateway           │
│ ─────────────────────────────────────────────────────────  │
│ Layer                L4 (Transport)  L7 (Application)      │
│ Protocols            TCP, UDP        HTTP, HTTPS, WebSocket│
│ Latency              < 1ms           < 5ms                 │
│ Throughput           10 Gbps         8 Gbps                │
│ Connection handling  Per flow        Per request           │
│                                                            │
│ Routing              IP + Port       FQDN, Path, Hostname  │
│ Example              10.0.1.10:8080  www.example.com/api   │
│                                      www.example.com/static│
│                                                            │
│ Session affinity     Source IP       Cookie-based          │
│ SSL termination      No              Yes (Secure Gateway)  │
│ Web App Firewall     No              Yes (with WAF)        │
│ URL rewriting        No              Yes                   │
│ Compression          No              Yes (gzip)            │
│                                                            │
│ Use case             Internal LB,    Web apps,             │
│                      Non-HTTP proto  HTTPS termination     │
│                                                            │
│ Availability         99.99%          99.95%                │
│ SLA Zone redundant   Yes             Yes (v2)              │
│                                                            │
└────────────────────────────────────────────────────────────┘

LOAD BALANCER ARCHITECTURE:

┌────────────────────────────────────────────────────────┐
│  Azure Standard Load Balancer (L4)                      │
│                                                        │
│  Public Frontend                                       │
│  └─ VIP: 203.0.113.10:80                              │
│                                                        │
│  Backend Pool                                          │
│  ├─ VM1: 10.0.1.11:80                                 │
│  ├─ VM2: 10.0.1.12:80                                 │
│  └─ VM3: 10.0.1.13:80                                 │
│                                                        │
│  Load Balancing Rule                                   │
│  ├─ Frontend: 203.0.113.10:80 (public)                │
│  ├─ Backend: Port 80 (on backend VMs)                 │
│  ├─ Protocol: TCP                                     │
│  ├─ Session affinity: Source IP (sticky)              │
│  └─ Health probe: Every 15 sec, timeout 30 sec        │
│                                                        │
│  NAT Rule (for RDP/SSH access)                        │
│  ├─ Frontend: 203.0.113.10:3389                       │
│  └─ Backend: VM1 10.0.1.11:3389 (direct mapping)      │
│                                                        │
└────────────────────────────────────────────────────────┘

APPLICATION GATEWAY ARCHITECTURE:

┌────────────────────────────────────────────────────────┐
│  Azure Application Gateway v2 (L7)                      │
│                                                        │
│  Public Frontend                                       │
│  └─ VIP: 203.0.113.20                                 │
│                                                        │
│  Listeners                                             │
│  ├─ Listener 1: HTTPS:443 (for www.example.com)       │
│  └─ Listener 2: HTTP:80 (redirects to HTTPS)          │
│                                                        │
│  Request Routing Rules (examines HTTP headers)         │
│  ├─ Rule 1: If path="/api" → Backend Pool "api"       │
│  ├─ Rule 2: If path="/static" → Backend Pool "cdn"    │
│  └─ Rule 3: Otherwise → Backend Pool "web"            │
│                                                        │
│  Backend Pools                                         │
│  ├─ "api": API servers 10.0.1.11:8080                 │
│  ├─ "web": Web servers 10.0.1.12:8080                 │
│  └─ "cdn": Static content servers 10.0.1.13:8080      │
│                                                        │
│  WAF Policy (Application Layer Firewall)               │
│  ├─ Blocks SQL injection                              │
│  ├─ Blocks XSS attacks                                │
│  ├─ Blocks malformed HTTP                             │
│  └─ Rate limiting                                     │
│                                                        │
└────────────────────────────────────────────────────────┘

LOAD BALANCER TERRAFORM:

resource "azurerm_lb" "main" {
  name                = "lb-production"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "Standard"
  sku_tier            = "Regional"

  frontend_ip_configuration {
    name                 = "frontend"
    public_ip_address_id = azurerm_public_ip.main.id
  }
}

resource "azurerm_lb_backend_address_pool" "main" {
  loadbalancer_id = azurerm_lb.main.id
  name            = "backend-pool"
}

resource "azurerm_lb_rule" "http" {
  loadbalancer_id            = azurerm_lb.main.id
  name                       = "http"
  protocol                   = "Tcp"
  frontend_port              = 80
  backend_port               = 80
  frontend_ip_configuration_name = "frontend"
  backend_address_pool_ids   = [azurerm_lb_backend_address_pool.main.id]
  probe_id                   = azurerm_lb_probe.http.id
  enable_tcp_reset           = true
  load_distribution          = "SourceIP"  # Sticky sessions
}

resource "azurerm_lb_probe" "http" {
  loadbalancer_id = azurerm_lb.main.id
  name            = "http-probe"
  protocol        = "Http"
  port            = 80
  request_path    = "/health"
  interval_in_seconds = 15
  number_of_probes    = 2
}
```

---

## Hybrid Connectivity (ExpressRoute & VPN)

### Q: Design hybrid connectivity for enterprise with on-premises datacenter

```
HYBRID NETWORK TOPOLOGY:

On-Premises Datacenter          Azure Cloud
┌─────────────────────┐        ┌──────────────────────┐
│ Corporate Network   │        │ Azure VNet           │
│ 172.16.0.0/12       │        │ 10.0.0.0/8          │
│                     │        │                      │
│ ┌───────────────┐   │        │  ┌──────────────┐   │
│ │ Workstations  │   │        │  │ AKS Cluster  │   │
│ │ Servers       │   │        │  │ Pods: ....   │   │
│ │ Databases     │   │        │  └──────────────┘   │
│ │ File Servers  │   │        │                      │
│ └───────────────┘   │        │  ┌──────────────┐   │
│                     │        │  │ Databases    │   │
│ ┌───────────────┐   │        │  │ (Private)    │   │
│ │ Firewalls     │   │        │  └──────────────┘   │
│ └───────────────┘   │        │                      │
│                     │        │  ┌──────────────┐   │
│                     │        │  │ Storage      │   │
└──────────┬──────────┘        │  │ (Private EP) │   │
           │                  │  └──────────────┘   │
           │  VPN/ExpressRoute│                      │
           │                  └──────────┬───────────┘
           │                             │
           ├─────────────────────────────┤
           │ Connectivity Options:       │
           │                             │
           ├─ Option 1: Site-to-Site VPN │
           │  Pros: Cheap, quick setup    │
           │  Cons: Variable latency      │
           │  Speed: 100 Mbps typical     │
           │                             │
           ├─ Option 2: ExpressRoute      │
           │  Pros: Consistent latency    │
           │  Cons: Lead time, expensive  │
           │  Speed: 1-100 Gbps           │
           │                             │
           └─ Option 3: Hybrid           │
              (Both for redundancy)      │
              Primary: ExpressRoute      │
              Failover: VPN              │

ROUTING IN HYBRID SETUP:

┌──────────────────────────────────────────────────────┐
│  On-Premises 172.16.0.0/12                           │
│  ├─ Workstation needs to reach Azure Database        │
│  │  (private IP: 10.0.9.10)                          │
│  │                                                   │
│  │  Workstation sends packet:                        │
│  │  Source: 172.16.5.25                              │
│  │  Destination: 10.0.9.10                           │
│  │                                                   │
│  │  Firewall sees:                                   │
│  │  10.0.9.10? Not local (172.16.x.x)                │
│  │  Check route table:                               │
│  │  10.0.0.0/8 → VPN Gateway                         │
│  │                                                   │
│  │  Firewall sends to VPN Gateway (IPSec tunnel)     │
│  │                                                   │
│  └─ Packet travels through encrypted tunnel          │
│     to Azure VPN Gateway                             │
│                                                      │
└──────────────────────────────────────────────────────┘

VPN GATEWAY SETUP:

resource "azurerm_public_ip" "vpn" {
  name                = "pip-vpn-gateway"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_vpn_gateway" "main" {
  name                = "vpn-gateway"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  type                = "Vpn"
  vpn_type            = "RouteBased"
  active_active       = true  # High availability
  enable_bgp          = true  # Dynamic routing

  ip_configuration {
    name                          = "vnetGatewayConfig"
    public_ip_address_id          = azurerm_public_ip.vpn.id
    private_ip_address_allocation = "Dynamic"
    subnet_id                     = azurerm_subnet.gateway.id
  }

  vpn_client_configuration {
    address_space = ["172.16.201.0/24"]  # VPN clients get IPs here

    root_certificate {
      name             = "P2SRootCert"
      public_cert_data = file("${path.module}/certs/P2SRootCert.cer")
    }

    revoked_certificate {
      name       = "P2SRevokedCert"
      thumbprint = "..."
    }
  }
}

resource "azurerm_local_network_gateway" "onprem" {
  name                = "lng-onprem"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  gateway_address     = "203.0.113.1"  # On-prem VPN endpoint
  address_space       = ["172.16.0.0/12"]  # On-prem network

  bgp_settings {
    asn              = 65000
    bgp_peering_address = "10.0.10.254"
    peer_weight      = 100
  }
}

resource "azurerm_vpn_gateway_connection" "onprem" {
  name                = "vpn-conn-onprem"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name

  type                       = "IPSec"
  virtual_network_gateway_id = azurerm_vpn_gateway.main.id
  local_network_gateway_id   = azurerm_local_network_gateway.onprem.id

  shared_key = "shared-secret-key"  # Pre-shared key (minimum 16 chars)

  enable_bgp = true

  ipsec_policy {
    sa_lifetime = 27000  # seconds (7.5 hours)
    sa_datasize = 102400000
    ike_encryption = "AES256"
    ike_integrity  = "SHA256"
    dh_group       = "DHGroup14"
    ipsec_encryption = "AES256"
    ipsec_integrity  = "SHA256"
    pfs_group        = "PFS2048"
  }

  traffic_selector_policy {
    local_address_cidrs  = ["172.16.0.0/12"]
    remote_address_cidrs = ["10.0.0.0/8"]
  }

  depends_on = [azurerm_vpn_gateway.main]
}
```

---

## Private Endpoints & Service Endpoints

### Q: Design storage account access using Private Endpoints

```
PROBLEM: Storage account publicly accessible
├─ Anyone on internet can access (if they have key)
└─ Data travels through public internet
   └─ Potential performance/security issue

SOLUTION: Private Endpoint
├─ Storage accessible only from Azure VNet
├─ Private IP assigned to storage service
├─ Traffic never leaves Azure backbone
└─ Network isolation

ARCHITECTURE:

Azure VNet (10.0.0.0/8)
┌─────────────────────────────────────┐
│  Subnet: 10.0.1.0/24                │
│  ┌──────────────────────────────┐   │
│  │ Pod/VM accessing storage     │   │
│  │ $ curl                        │   │
│  │ https://storageacct.blob.    │   │
│  │ core.windows.net/container/  │   │
│  │ blob.txt                      │   │
│  └────────────┬──────────────────┘   │
│               │                      │
│               │ DNS resolution       │
│               ▼                      │
│  Private DNS Zone                   │
│  ├─ privatelink.blob.core.windows.net
│  │  A record: storageacct → 10.0.15.4
│  │                                   │
│  └─ Returns PRIVATE IP               │
│     (not public IP)                  │
│               │                      │
│               │ Network traffic      │
│               ▼                      │
│  Private Endpoint NIC                │
│  ├─ Private IP: 10.0.15.4            │
│  ├─ Connected to subnet              │
│  ├─ No public IP                     │
│  └─ Linked to storage account        │
│                                      │
│  Storage Account (backend)           │
│  ├─ Public IP disabled               │
│  ├─ Only accessible via private EP   │
│  └─ NSG rules restrict access        │
│                                      │
└─────────────────────────────────────┘

ACCESS CONTROL:

Without Private Endpoint (risky):
├─ Storage accessible from anywhere
├─ Only auth: Storage account key
├─ If key leaked → data exposed
└─ No network-level access control

With Private Endpoint (secure):
├─ Storage accessible only from specific VNet
├─ Storage + key required
├─ Network-level isolation
└─ Even with leaked key, can't access from internet

TERRAFORM:

resource "azurerm_private_endpoint" "storage" {
  name                = "pep-storage"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  subnet_id           = azurerm_subnet.database.id

  private_service_connection {
    name                           = "storage-connection"
    is_manual_connection           = false
    private_connection_resource_id = azurerm_storage_account.main.id
    subresource_names              = ["blob"]
  }
}

# Create private DNS zone
resource "azurerm_private_dns_zone" "storage" {
  name                = "privatelink.blob.core.windows.net"
  resource_group_name = azurerm_resource_group.main.name
}

# Link VNet to private DNS zone
resource "azurerm_private_dns_zone_virtual_network_link" "storage" {
  name                  = "storage-vnet-link"
  resource_group_name   = azurerm_resource_group.main.name
  private_dns_zone_name = azurerm_private_dns_zone.storage.name
  virtual_network_id    = azurerm_virtual_network.main.id
}

# Create DNS A record (maps FQDN to private IP)
resource "azurerm_private_dns_a_record" "storage" {
  name                = azurerm_storage_account.main.name
  zone_name           = azurerm_private_dns_zone.storage.name
  resource_group_name = azurerm_resource_group.main.name
  ttl                 = 10
  records             = [
    azurerm_private_endpoint.storage.private_service_connection[0].private_ip_address
  ]
}
```

---

## Azure DNS

### Q: Design DNS for multi-region application with failover

```
MULTI-REGION DNS FAILOVER:

Primary Region: East US
├─ Application Gateway: 203.0.113.10
├─ Health Endpoint: /health → 200 OK
└─ Status: Healthy

Secondary Region: West US
├─ Application Gateway: 203.0.114.10
├─ Health Endpoint: /health → 200 OK (standby)
└─ Status: Healthy (but not primary)

Azure DNS Configuration:
└─ www.example.com

Query: User resolves www.example.com
└─ Azure DNS Health Probe checks:
   ├─ Primary (East US) → 200 OK → Return 203.0.113.10
   └─ User redirected to East US

If Primary Fails:
├─ User still tries East US
├─ Connection timeout (50 seconds TTL + DNS propogation)
├─ User retries DNS query
├─ Health probe detects primary down
├─ Azure DNS returns 203.0.114.10 (West US)
└─ Traffic flows to secondary

TERRAFORM:

resource "azurerm_dns_zone" "main" {
  name                = "example.com"
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_dns_a_record" "primary" {
  name                = "www"
  zone_name           = azurerm_dns_zone.main.name
  resource_group_name = azurerm_resource_group.main.name
  ttl                 = 300

  alias {
    name                   = azurerm_application_gateway.primary.name
    zone_id                = azurerm_application_gateway.primary.id
    evaluate_target_health = true  # Enable health check
  }
}

resource "azurerm_dns_a_record" "secondary" {
  name                = "www"
  zone_name           = azurerm_dns_zone.main.name
  resource_group_name = azurerm_resource_group.main.name
  ttl                 = 300
  set_identifier      = "secondary"

  alias {
    name                   = azurerm_application_gateway.secondary.name
    zone_id                = azurerm_application_gateway.secondary.id
    evaluate_target_health = true  # Enable health check
  }

  failover_routing_policy {
    failover = "Secondary"
  }
}
```

---

## Troubleshooting Flowcharts

### Connectivity Troubleshooting

```
"Can't connect to Azure Database from pod"

Step 1: Verify pod can reach gateway
$ kubectl exec -it pod -- curl 10.0.9.10:5432
$ kubectl exec -it pod -- nc -zv 10.0.9.10 5432

Step 2: Check DNS resolution
$ kubectl exec -it pod -- nslookup postgres.database.azure.com
Expected: Returns private IP (e.g., 10.0.9.10)

Step 3: Verify NSG rules
Application tier subnet NSG:
├─ Outbound rule 5432 to database subnet?
├─ Priority correct (lower number = higher priority)?
└─ Action = Allow?

Step 4: Check route tables
Application subnet routing:
├─ 10.0.9.0/24 → Direct (local routing)?
├─ Or via NAT Gateway?
└─ Verify no conflicting routes

Step 5: Verify private endpoint
$ az network private-endpoint show -n pep-db
├─ Status: Approved?
├─ Connected to correct resource group?
└─ Private IP assigned?

Step 6: Check firewall rules
Azure SQL Firewall:
├─ Source IP: Pod IP (10.0.1.x)?
└─ Service endpoint or private endpoint rules?

Root Cause: Missing NSG rule (most common)
Fix: Add outbound rule in app tier NSG
```

---

## Interview Questions

**Q1: Design global network for multi-region SaaS company**

(See SYSTEM-DESIGN-TROUBLESHOOTING.md for complete answer)

This covers production-ready Azure networking knowledge for FAANG interviews.
