# AWS Networking - Comprehensive Guide

> Deep dive into VPC, subnets, routing, security, connectivity, DNS, and advanced networking patterns

---

## Table of Contents

- [VPC Fundamentals](#vpc-fundamentals)
- [Subnets and Routing](#subnets-and-routing)
- [Security (Security Groups, NACLs)](#security-groups--nacls)
- [Internet Connectivity](#internet-connectivity)
- [VPC Peering](#vpc-peering)
- [Transit Gateway](#transit-gateway)
- [VPN and Direct Connect](#vpn-and-direct-connect)
- [PrivateLink and VPC Endpoints](#privatelink-and-vpc-endpoints)
- [Route53 and DNS](#route53-and-dns)
- [Hybrid Connectivity](#hybrid-connectivity)
- [Troubleshooting](#networking-troubleshooting)
- [Interview Questions](#interview-questions)

---

## VPC Fundamentals

### What is a VPC?

**Q: Explain VPC architecture and its key components.**

**A (Intermediate):**

VPC = Virtual Private Cloud = isolated network within AWS

Components:
```
┌─────────────────────────────────────────────────────────────┐
│                         VPC (10.0.0.0/16)                   │
│  ┌─────────────────────────────────────────────────────────┤
│  │                    Internet                              │
│  └─────────────────────────────────────────────────────────┤
│                    ↑                                         │
│         (Internet Gateway)                                  │
│                    ↓                                         │
│  ┌─────────────────────────────────────────────────────────┤
│  │              Route Table (Public)                        │
│  │  0.0.0.0/0 → IGW                                        │
│  └─────────────────────────────────────────────────────────┤
│                    ↓                                         │
│  ┌─────────────────────────────────────────────────────────┤
│  │   Public Subnet (10.0.1.0/24) - AZ-a                   │
│  │  ┌─────────────────────────────────────────────────────┤
│  │  │  ALB (Elastic IP)                                    │
│  │  │  ┌─────────────────────────────────────────────────┤
│  │  │  │  EC2 Instance (10.0.1.50)                       │
│  │  │  └─────────────────────────────────────────────────┤
│  │  └─────────────────────────────────────────────────────┤
│  └─────────────────────────────────────────────────────────┤
│                    ↓                                         │
│  ┌─────────────────────────────────────────────────────────┤
│  │              Route Table (Private)                       │
│  │  0.0.0.0/0 → NAT Gateway                               │
│  └─────────────────────────────────────────────────────────┤
│                    ↓                                         │
│  ┌─────────────────────────────────────────────────────────┤
│  │   Private Subnet (10.0.2.0/24) - AZ-a                  │
│  │  ┌─────────────────────────────────────────────────────┤
│  │  │  EC2 Instance (10.0.2.50)                           │
│  │  │  ┌─────────────────────────────────────────────────┤
│  │  │  │ RDS Instance (10.0.2.100)                       │
│  │  │  └─────────────────────────────────────────────────┤
│  │  └─────────────────────────────────────────────────────┤
│  └─────────────────────────────────────────────────────────┤
│                                                              │
│   VPC Endpoints (private connectivity to AWS services)     │
│   ├─ S3 Gateway Endpoint                                    │
│   ├─ DynamoDB Gateway Endpoint                              │
│   └─ EC2/CloudFormation Interface Endpoints                │
└─────────────────────────────────────────────────────────────┘
```

**Key Components:**
1. **CIDR Block:** IP range for VPC (10.0.0.0/16 = 65,536 addresses)
2. **Subnets:** Subdivide VPC into smaller networks
3. **Route Tables:** Define traffic routing
4. **Internet Gateway:** Connection to internet
5. **NAT Gateway:** Outbound internet for private subnets
6. **VPC Endpoints:** Private connectivity to AWS services
7. **Network ACLs:** Subnet-level firewall
8. **Security Groups:** Instance-level firewall

---

### CIDR Notation

**Q: Explain CIDR notation and calculate subnet sizes.**

**A:**

CIDR = Classless Inter-Domain Routing = 10.0.0.0/16

```
10.0.0.0/16
│   │   │ │ │
│   │   │ │ └─ Network mask (bits used for network)
│   │   │ └──── Host bits (32 - 16 = 16)
│   │   └─────── Last 8 bits for host (0-255)
│   └──────────── Second octet
└────────────── First octet
```

**Calculating sizes:**

```
/16: 2^(32-16) = 2^16 = 65,536 hosts
     Usable: 65,534 (minus network and broadcast)

/24: 2^(32-24) = 2^8 = 256 hosts
     Usable: 254

/25: 2^(32-25) = 2^7 = 128 hosts
     Usable: 126

/30: 2^(32-30) = 2^2 = 4 hosts
     Usable: 2 (for VPN tunnels, VPC peering)

/32: 2^(32-32) = 2^0 = 1 host (host route)
```

**Common VPC design:**

```
VPC: 10.0.0.0/16 (65,536 hosts)
├─ AZ-a:
│  ├─ Public Subnet: 10.0.1.0/24 (256 hosts)
│  └─ Private Subnet: 10.0.2.0/24 (256 hosts)
├─ AZ-b:
│  ├─ Public Subnet: 10.0.11.0/24
│  └─ Private Subnet: 10.0.12.0/24
└─ AZ-c:
   ├─ Public Subnet: 10.0.21.0/24
   └─ Private Subnet: 10.0.22.0/24

Total used: 6 × 256 = 1,536 hosts
Remaining: ~64,000 hosts for future growth
```

**Reserved IPs in each subnet:**

```
10.0.1.0/24 has:
- 10.0.1.0 (network address)
- 10.0.1.1 (gateway)
- 10.0.1.2 (DNS resolver)
- 10.0.1.3 (reserved by AWS)
- 10.0.1.4-254 (usable)
- 10.0.1.255 (broadcast)

Usable: 251 addresses (not 256!)
```

---

## Subnets and Routing

### Subnet Design

**Q: Design subnet architecture for a scalable application.**

**A (Advanced):**

```
Multi-AZ Application Architecture:

VPC: 10.0.0.0/16

Region: us-east-1
├─ AZ: us-east-1a
│  ├─ Public Subnet: 10.0.1.0/24
│  │  └─ ALB (load balancer)
│  │  └─ NAT Gateway (High Availability)
│  └─ Private Subnet: 10.0.2.0/24
│     └─ EC2 ASG (application servers)
│     └─ RDS Replica
│
├─ AZ: us-east-1b
│  ├─ Public Subnet: 10.0.11.0/24
│  │  └─ ALB (redundancy)
│  │  └─ NAT Gateway (redundancy)
│  └─ Private Subnet: 10.0.12.0/24
│     └─ EC2 ASG (application servers)
│     └─ RDS Replica
│
└─ AZ: us-east-1c
   ├─ Public Subnet: 10.0.21.0/24
   │  └─ ALB (redundancy)
   │  └─ NAT Gateway (redundancy)
   └─ Private Subnet: 10.0.22.0/24
      └─ EC2 ASG (application servers)
      └─ RDS Primary/Replica

Key Points:
✓ ALB in public subnet (internet-facing)
✓ EC2 in private subnet (no internet access)
✓ NAT Gateway per AZ (HA)
✓ RDS in private subnet with read replicas
✓ Each AZ independent for fault tolerance
```

### Route Tables

**Q: Explain route table evaluation and priority.**

**A:**

Routes are evaluated by **most specific match wins**:

```
Route Table for Private Subnet:

Destination     Target              Priority
─────────────   ──────────────      ────────
10.0.0.0/16     Local               1 (most specific)
10.1.0.0/24     PCX-12345678        2
10.0.0.0/8      TGW-12345678        3
0.0.0.0/0       NAT-12345678        4 (least specific)

When packet goes to 10.1.5.50:
├─ Matches 10.0.0.0/16? NO
├─ Matches 10.1.0.0/24? YES → Use PCX
└─ No need to check 10.0.0.0/8 or 0.0.0.0/0

When packet goes to 192.168.1.50:
├─ Matches 10.0.0.0/16? NO
├─ Matches 10.1.0.0/24? NO
├─ Matches 10.0.0.0/8? NO
└─ Matches 0.0.0.0/0? YES → Use NAT Gateway
```

**Example - Create route table:**

```python
import boto3

ec2 = boto3.client('ec2')

# Create private route table
rt_response = ec2.create_route_table(VpcId='vpc-12345678')
route_table_id = rt_response['RouteTable']['RouteTableId']

# Add routes
# 1. Local traffic (auto-added for VPC CIDR)
# 2. Internet through NAT
ec2.create_route(
    RouteTableId=route_table_id,
    DestinationCidrBlock='0.0.0.0/0',
    NatGatewayId='natgw-12345678'
)

# 3. Other VPCs through Transit Gateway
ec2.create_route(
    RouteTableId=route_table_id,
    DestinationCidrBlock='10.1.0.0/16',
    TransitGatewayId='tgw-12345678'
)

# 4. On-premise through VPN
ec2.create_route(
    RouteTableId=route_table_id,
    DestinationCidrBlock='192.168.0.0/16',
    VpnConnectionId='vpn-12345678'
)

# Associate with subnet
ec2.associate_route_table(
    RouteTableId=route_table_id,
    SubnetId='subnet-12345678'
)
```

---

## Internet Connectivity

### Internet Gateway (IGW)

**Q: Design internet connectivity for a web application. When would you use IGW vs NAT?**

**A (Intermediate):**

| Scenario | Solution | Why |
|----------|----------|-----|
| EC2 needs inbound internet | Public subnet + IGW + Elastic IP | Can receive connections |
| EC2 needs outbound internet only | Private subnet + NAT Gateway | No inbound access |
| Database in private subnet | No IGW/NAT needed | Shouldn't be internet-facing |
| Lambda in VPC needs internet | NAT Gateway for outbound | Lambda can't have public IP |

**Architecture:**

```
┌─────────────────────────────────────────────┐
│             Internet                        │
│            (0.0.0.0/0)                      │
└─────────────┬───────────────────────────────┘
              │ (port 80, 443)
┌─────────────┴───────────────────────────────┐
│  Internet Gateway (IGW)                     │
│  Translates public IPs ↔ internet traffic   │
└─────────────┬───────────────────────────────┘
              │
┌─────────────┴───────────────────────────────┐
│  Public Subnet (10.0.1.0/24)                │
│  Route: 0.0.0.0/0 → IGW                    │
│  ┌─────────────────────────────────────────┤
│  │ ALB (Elastic IP: 203.0.113.50)          │
│  │  ↓                                       │
│  │ EC2 (Private IP: 10.0.1.50)             │
│  │ (Can access internet and be accessed)   │
│  └─────────────────────────────────────────┤
└─────────────────────────────────────────────┘
              │
┌─────────────┴───────────────────────────────┐
│  NAT Gateway (Public Subnet)                │
│  Located in: us-east-1a                     │
│  Elastic IP: 203.0.113.100                  │
└─────────────┬───────────────────────────────┘
              │
┌─────────────┴───────────────────────────────┐
│  Private Subnet (10.0.2.0/24)               │
│  Route: 0.0.0.0/0 → NAT                    │
│  ┌─────────────────────────────────────────┤
│  │ EC2 (Private IP: 10.0.2.50)             │
│  │ (Can access internet, NOT accessed)     │
│  │                                         │
│  │ RDS (Private IP: 10.0.2.100)            │
│  │ (Cannot access internet)                │
│  └─────────────────────────────────────────┤
└─────────────────────────────────────────────┘
```

**Cost Consideration:**
- IGW: Free
- NAT Gateway: $0.045/hour + $0.045 per GB processed
- For high-traffic application: Consider using NAT Instance instead (on EC2)

---

### NAT Gateway vs NAT Instance

| Aspect | NAT Gateway | NAT Instance |
|--------|---|---|
| **Management** | Managed by AWS | You manage EC2 instance |
| **Availability** | Auto HA within AZ | Must configure HA yourself |
| **Performance** | 10 Gbps | Limited by instance size |
| **Cost** | Hourly + data transfer | EC2 instance cost |
| **Best for** | Production, high-traffic | Development, cost-sensitive |

**Example - NAT Instance with HA:**

```python
import boto3

ec2 = boto3.client('ec2')

# Create NAT instance in each AZ for HA
for az, subnet_id in [
    ('us-east-1a', 'subnet-12345678'),
    ('us-east-1b', 'subnet-87654321')
]:
    # Launch small t3.small instance (ami-ami-nat)
    response = ec2.run_instances(
        ImageId='ami-00ca5e4f2fa2c30d9',  # NAT AMI
        InstanceType='t3.small',
        SubnetId=subnet_id,
        SecurityGroupIds=['sg-nat-12345678']
    )
    
    instance_id = response['Instances'][0]['InstanceId']
    
    # Disable source/dest check (required for NAT)
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        SourceDestCheck={'Value': False}
    )
    
    # Create route to NAT instance
    ec2.create_route(
        RouteTableId=f'rtb-private-{az}',
        DestinationCidrBlock='0.0.0.0/0',
        InstanceId=instance_id
    )
```

---

## VPC Peering

**Q: Design VPC peering for a multi-region application.**

**A (Advanced):**

**Scenario:** Company with separate VPCs for Dev, Staging, Prod in same region. Need inter-VPC communication.

```
Setup:

VPC-Prod (10.0.0.0/16)
  ├─ Public: 10.0.1.0/24
  └─ Private: 10.0.2.0/24

VPC-Staging (10.1.0.0/16)
  ├─ Public: 10.1.1.0/24
  └─ Private: 10.1.2.0/24

VPC-Dev (10.2.0.0/16)
  ├─ Public: 10.2.1.0/24
  └─ Private: 10.2.2.0/24

Peering:
Prod ←→ Staging (prod-staging-pcx)
Prod ←→ Dev (prod-dev-pcx)
Staging ←→ Dev (staging-dev-pcx)
```

**Implementation:**

```python
import boto3

ec2 = boto3.client('ec2')

# Create peering connection
pcx_response = ec2.create_vpc_peering_connection(
    VpcId='vpc-prod-12345678',
    PeerVpcId='vpc-staging-87654321'
)

pcx_id = pcx_response['VpcPeeringConnection']['VpcPeeringConnectionId']

# Accept peering connection
ec2.accept_vpc_peering_connection(VpcPeeringConnectionId=pcx_id)

# Update route tables in Prod VPC
ec2.create_route(
    RouteTableId='rtb-prod-private',
    DestinationCidrBlock='10.1.0.0/16',  # Staging CIDR
    VpcPeeringConnectionId=pcx_id
)

# Update route tables in Staging VPC
ec2.create_route(
    RouteTableId='rtb-staging-private',
    DestinationCidrBlock='10.0.0.0/16',  # Prod CIDR
    VpcPeeringConnectionId=pcx_id
)

# Update security groups
ec2.authorize_security_group_ingress(
    GroupId='sg-prod-app',
    IpPermissions=[{
        'IpProtocol': 'tcp',
        'FromPort': 3306,
        'ToPort': 3306,
        'IpRanges': [{'CidrIp': '10.1.0.0/16'}]
    }]
)
```

**Limitations:**
- CIDR ranges cannot overlap
- Transitive peering NOT supported (A→B, B→C doesn't mean A→C)
- For complex architectures, use Transit Gateway instead

---

## Transit Gateway

**Q: When would you use Transit Gateway instead of VPC Peering?**

**A (Advanced):**

**Transit Gateway = Hub-and-spoke networking model**

```
Without Transit Gateway (Mesh):
A ←→ B
A ←→ C
B ←→ C

Requires: 3 peering connections
Scaling: For N VPCs, need N(N-1)/2 connections
Problem: Exponential complexity

With Transit Gateway (Hub-and-spoke):
    ┌─────────┐
    │   TGW   │
    └────┬────┘
    ┌────┼────┐
    │    │    │
    ▼    ▼    ▼
    A    B    C

Requires: 3 attachments
Scaling: For N VPCs, need N attachments
Benefit: Centralized routing, easy to manage
```

**Setup:**

```python
import boto3

ec2 = boto3.client('ec2')

# Create Transit Gateway
tgw_response = ec2.create_transit_gateway(
    Description='Production TGW',
    Options={
        'AmazonSideAsn': 64512,
        'DefaultRouteTableAssociation': 'enable',
        'DefaultRouteTablePropagation': 'enable'
    }
)

tgw_id = tgw_response['TransitGateway']['TransitGatewayId']

# Attach VPCs
for vpc_id, subnets in [
    ('vpc-prod', ['subnet-prod-1', 'subnet-prod-2']),
    ('vpc-staging', ['subnet-staging-1', 'subnet-staging-2']),
    ('vpc-dev', ['subnet-dev-1', 'subnet-dev-2'])
]:
    attachment = ec2.create_transit_gateway_vpc_attachment(
        TransitGatewayId=tgw_id,
        VpcId=vpc_id,
        SubnetIds=subnets
    )

# Create route in VPC
ec2.create_route(
    RouteTableId='rtb-prod-private',
    DestinationCidrBlock='10.1.0.0/16',
    TransitGatewayId=tgw_id
)
```

**Advantages:**
- Centralized routing
- Easy to add/remove VPCs
- Support for on-premise via VPN
- Support for Direct Connect
- Route filtering and monitoring
- Multicast support

---

## VPN and Direct Connect

### Site-to-Site VPN

**Q: Design secure connectivity from on-premise to AWS.**

**A:**

```
On-Premise Network              AWS VPC
192.168.0.0/16                  10.0.0.0/16
    │                               │
    │ Customer Gateway              │ Virtual Private Gateway
    │ (on-premise router)           │ (VPC side)
    │                               │
    └───────── IPSec Tunnel 1 ──────┤
    │                               │
    └───────── IPSec Tunnel 2 ──────┤
    │                               │
```

**Implementation:**

```python
import boto3

ec2 = boto3.client('ec2')

# 1. Create Virtual Private Gateway (VGW)
vgw_response = ec2.create_vpn_gateway(
    Type='ipsec.1',
    AmazonSideAsn=64512
)
vgw_id = vgw_response['VpnGateway']['VpnGatewayId']

# Attach to VPC
ec2.attach_vpn_gateway(
    VpcId='vpc-12345678',
    VpnGatewayId=vgw_id
)

# 2. Create Customer Gateway (on-premise side)
cgw_response = ec2.create_customer_gateway(
    Type='ipsec.1',
    PublicIp='203.0.113.50',  # On-premise router public IP
    BgpAsn=65000
)
cgw_id = cgw_response['CustomerGateway']['CustomerGatewayId']

# 3. Create VPN Connection
vpn_response = ec2.create_vpn_connection(
    Type='ipsec.1',
    CustomerGatewayId=cgw_id,
    VpnGatewayId=vgw_id,
    Options={
        'StaticRoutesOnly': False,  # Use dynamic routing (BGP)
        'TunnelOptions': [
            {
                'TunnelInsideCidr': '169.254.10.0/30',
                'PreSharedKey': 'your-pre-shared-key-here'
            },
            {
                'TunnelInsideCidr': '169.254.11.0/30',
                'PreSharedKey': 'your-pre-shared-key-here'
            }
        ]
    }
)

vpn_id = vpn_response['VpnConnection']['VpnConnectionId']

# 4. Add route to VPN
ec2.create_route(
    RouteTableId='rtb-private',
    DestinationCidrBlock='192.168.0.0/16',
    GatewayId=vgw_id
)

# 5. Enable VGW route propagation
ec2.enable_vgw_route_propagation(
    RouteTableId='rtb-private',
    GatewayId=vgw_id
)
```

### AWS Direct Connect

**Q: When would you use Direct Connect vs VPN?**

**A:**

| Aspect | Site-to-Site VPN | Direct Connect |
|--------|---|---|
| **Setup Time** | Minutes | Weeks (physical connection) |
| **Connection** | Internet (IPSec) | Dedicated fiber |
| **Bandwidth** | Up to 1.25 Gbps | Up to 400 Gbps |
| **Latency** | Variable (internet) | Consistent, low |
| **Cost** | Lower | Higher ($0.30/hour) |
| **Use Case** | Temporary, backup | Production, high-throughput |

**Architecture:**

```
On-Premise Data Center
    │
    │ (Dedicated Fiber)
    │ AWS Direct Connect LOA/CFA
    │ (Letter of Authorization)
    │
    ▼
AWS Direct Connect Location
(Telecom provider's facility)
    │
    │ (AWS backbone)
    │
    ▼
AWS Region
    │
    ├─ Private Virtual Interface (Private VPC)
    ├─ Public Virtual Interface (S3, DynamoDB)
    └─ Transit Virtual Interface (Multiple VPCs via TGW)
```

**Cost Optimization - Backup Connection:**

```
Primary: Direct Connect (high bandwidth)
Backup: Site-to-Site VPN (low cost)

If Direct Connect fails → traffic reroutes over VPN
```

---

## PrivateLink and VPC Endpoints

**Q: A customer needs to access your S3 bucket but you don't want data crossing the internet. How?**

**A (Intermediate):**

**VPC Endpoint Gateway (S3, DynamoDB):**

```
Without VPC Endpoint:
EC2 in Private Subnet
    ↓
NAT Gateway (public subnet)
    ↓
Internet Gateway
    ↓
Internet
    ↓
S3 Public Endpoint
    ↓ (return)
NAT Gateway
    ↓
EC2

Cost: Data transfer out of VPC ($0.02/GB)

With VPC Endpoint:
EC2 in Private Subnet
    ↓
VPC Endpoint (local)
    ↓
S3

Cost: FREE!
```

**Implementation:**

```python
import boto3

ec2 = boto3.client('ec2')
s3 = boto3.client('s3')

# Create VPC Endpoint for S3
endpoint_response = ec2.create_vpc_endpoint(
    VpcId='vpc-12345678',
    ServiceName='com.amazonaws.us-east-1.s3',
    RouteTableIds=['rtb-private-1', 'rtb-private-2'],
    PolicyDocument='''{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": "*",
                "Action": "s3:GetObject",
                "Resource": "arn:aws:s3:::my-bucket/*"
            }
        ]
    }'''
)

endpoint_id = endpoint_response['VpcEndpoint']['VpcEndpointId']

# Now EC2 can access S3 without NAT/IGW
s3.get_object(Bucket='my-bucket', Key='file.txt')
```

**VPC Endpoint Services (Interface Endpoints - EC2, ALB, etc.):**

```
Scenario: AWS Account A has API, Account B needs access

Account A (API Provider):
    ALB → TargetGroup → Instance
    └─ Create NLB in front (for endpoint service)
    └─ Create Endpoint Service

Account B (API Consumer):
    Create Interface Endpoint → Points to Account A's service
    Application → Interface Endpoint → NLB → ALB → API

Benefits:
✓ Private connectivity
✓ No internet exposure
✓ No NAT/IGW needed
✓ Cross-account/cross-region
```

---

## Route53 and DNS

**Q: Design Route53 for multi-region failover.**

**A (Advanced):**

**Scenario:** E-commerce site in us-east-1 and eu-west-1. If primary region fails, traffic should shift.

```
Route53 Configuration:

Primary Record (us-east-1):
├─ Type: A
├─ Name: api.example.com
├─ Value: 1.2.3.4 (ALB in us-east-1)
├─ Health Check: CloudWatch + HTTP checks
├─ SetId: us-east-1-primary
└─ Failover: Primary

Secondary Record (eu-west-1):
├─ Type: A
├─ Name: api.example.com
├─ Value: 5.6.7.8 (ALB in eu-west-1)
├─ Health Check: CloudWatch + HTTP checks
├─ SetId: eu-west-1-secondary
└─ Failover: Secondary

DNS Query Flow:
1. Client queries: api.example.com
2. Route53 checks primary health
   ├─ Healthy? → Return 1.2.3.4
   └─ Unhealthy? → Return 5.6.7.8
3. TTL: 60 seconds (fast failover)
```

**Implementation:**

```python
import boto3

route53 = boto3.client('route53')

# Get hosted zone
zones = route53.list_hosted_zones_by_name(DNSName='example.com')
zone_id = zones['HostedZones'][0]['Id']

# Create health check for primary
health_check_primary = route53.create_health_check(
    HealthCheckConfig={
        'Type': 'HTTP',
        'ResourcePath': '/health',
        'FullyQualifiedDomainName': 'api-us-east-1.example.com',
        'Port': 80,
        'RequestInterval': 30,
        'FailureThreshold': 3,
        'MeasureLatency': True
    }
)

# Create health check for secondary
health_check_secondary = route53.create_health_check(
    HealthCheckConfig={
        'Type': 'HTTP',
        'ResourcePath': '/health',
        'FullyQualifiedDomainName': 'api-eu-west-1.example.com',
        'Port': 80,
        'RequestInterval': 30,
        'FailureThreshold': 3,
        'MeasureLatency': True
    }
)

# Create failover records
changes = [
    {
        'Action': 'CREATE',
        'ResourceRecordSet': {
            'Name': 'api.example.com',
            'Type': 'A',
            'SetIdentifier': 'us-east-1-primary',
            'Failover': 'PRIMARY',
            'AliasTarget': {
                'HostedZoneId': 'Z35SXDOTRQ7X7K',  # us-east-1 ALB zone
                'DNSName': 'my-alb-123456.us-east-1.elb.amazonaws.com',
                'EvaluateTargetHealth': True
            },
            'HealthCheckId': health_check_primary['HealthCheck']['Id']
        }
    },
    {
        'Action': 'CREATE',
        'ResourceRecordSet': {
            'Name': 'api.example.com',
            'Type': 'A',
            'SetIdentifier': 'eu-west-1-secondary',
            'Failover': 'SECONDARY',
            'AliasTarget': {
                'HostedZoneId': 'Z32O12XQLNTSW2',  # eu-west-1 ALB zone
                'DNSName': 'my-alb-789012.eu-west-1.elb.amazonaws.com',
                'EvaluateTargetHealth': True
            },
            'HealthCheckId': health_check_secondary['HealthCheck']['Id']
        }
    }
]

route53.change_resource_record_sets(
    HostedZoneId=zone_id,
    ChangeBatch={'Changes': changes}
)
```

**Routing Policies:**

```
┌──────────────────────┬────────────────────────────────┐
│ Policy               │ Use Case                       │
├──────────────────────┼────────────────────────────────┤
│ Simple               │ Single resource                │
│ Weighted             │ Load balancing (A-60%, B-40%)  │
│ Latency-based        │ Route based on lowest latency  │
│ Failover             │ Active-passive failover        │
│ Geolocation          │ Route by geographic location   │
│ Geoproximity         │ Route by distance to location  │
│ Multi-value answer   │ Multiple random IPs            │
│ Traffic policy       │ Complex routing rules          │
└──────────────────────┴────────────────────────────────┘
```

---

## Networking Troubleshooting

### "Pod cannot reach database" - Troubleshooting Flow

**Scenario:** Kubernetes pod in us-east-1 cannot connect to RDS database.

**Diagnostic Steps:**

```
1. Is pod running?
   $ kubectl get pods -A
   $ kubectl describe pod <pod-name>

2. Can pod reach node?
   $ kubectl exec <pod> -- ip route
   Should show gateway to node

3. Can node reach RDS security group?
   From EC2 node:
   $ telnet <rds-endpoint> 3306
   
   If fails → Security group issue

4. Check RDS Security Group:
   $ aws ec2 describe-security-groups --group-ids sg-12345678
   
   Verify:
   ✓ Inbound rule allows port 3306
   ✓ Source includes EC2 security group or 0.0.0.0/0

5. Check Network ACL:
   $ aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=subnet-xyz
   
   Verify:
   ✓ Inbound allows port 3306
   ✓ Outbound allows ephemeral ports (1024-65535)

6. Check Route Table:
   $ aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=subnet-xyz
   
   Verify:
   ✓ RDS is in same VPC or connected via peering/TGW

7. Check RDS status:
   $ aws rds describe-db-instances --db-instance-identifier mydb
   
   Verify:
   ✓ Status: available
   ✓ Storage not full
   ✓ Endpoint accessible

8. Check application logs:
   $ kubectl logs <pod-name>
   
   Look for:
   ✓ Connection timeout (network issue)
   ✓ Authentication failed (credentials issue)
   ✓ Database not found (application issue)
```

**Common Issues and Solutions:**

```
Issue: Connection timeout
├─ Security group doesn't allow traffic
├─ NACL blocks port 3306
├─ RDS in different AZ and no route
└─ Solution: Review SG, NACL, route tables

Issue: Connection refused
├─ RDS is not running
├─ RDS port changed from default 3306
├─ RDS resource limit reached
└─ Solution: Restart RDS, check parameter groups

Issue: Access denied (after connection)
├─ Wrong username/password
├─ User doesn't have permissions
├─ Database/table doesn't exist
└─ Solution: Verify credentials, check IAM database auth
```

---

## Interview Questions

### Q1: Design VPC with 1000+ microservices

**A:** 
- Use Transit Gateway for central routing
- Separate VPC per team/service domain
- Centralized security monitoring
- Shared services VPC (logging, CI/CD)
- Multiple NAT Gateways per AZ for scale
- Route53 for service discovery

### Q2: High-performance connection to on-premise

**A:**
- AWS Direct Connect (primary) + VPN (backup)
- Direct Connect Gateway for multi-region
- BGP for dynamic routing
- Virtual Interfaces for isolation

### Q3: Application with PII must be accessible only from internal network

**A:**
- Private subnet (no IGW)
- Bastion host for access (not needed in modern AWS)
- VPC Endpoint for external APIs
- VPN for developer access
- All outbound restricted except VPC Endpoints
- NetFlow logs for audit

