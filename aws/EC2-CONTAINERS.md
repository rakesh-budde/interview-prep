# AWS EC2 and Containers (ECS/EKS) - Comprehensive Interview Guide

> Deep dive into EC2 fundamentals, ECS, and EKS with extreme focus on Kubernetes architecture and troubleshooting

**Estimated Reading Time:** 180 minutes | **Interview Coverage:** 200+ questions

---

## Table of Contents

- [EC2 Fundamentals](#ec2-fundamentals)
- [Instance Types and Sizing](#instance-types-and-sizing)
- [EBS Storage](#ebs-storage)
- [Auto Scaling and Launch Templates](#auto-scaling-and-launch-templates)
- [Spot Instances and Cost Optimization](#spot-instances-and-cost-optimization)
- [ECS Fundamentals](#ecs-fundamentals)
- [EKS Deep Dive](#eks-deep-dive)
- [Kubernetes Architecture](#kubernetes-architecture)
- [EKS Networking (CNI)](#eks-networking-cni)
- [Pod Identity and IRSA](#pod-identity-and-irsa)
- [EKS Troubleshooting](#eks-troubleshooting)
- [Interview Questions](#interview-questions)

---

## EC2 Fundamentals

### EC2 Instance Lifecycle

**Q: Explain EC2 instance states and transitions.**

**A (Intermediate):**

```
┌─────────────┐
│   pending   │  (Instance launching, initializing)
└──────┬──────┘
       │ (boot complete)
       ▼
┌─────────────┐
│   running   │◄─── (User can connect)
└──────┬──────┘
       │
   ┌───┴──────────────┬──────────────┐
   │                  │              │
   │              (stop)         (terminate)
   │                  │              │
   ▼                  ▼              ▼
┌─────────────┐  ┌──────────┐  ┌─────────────┐
│  stopping   │  │ shutting │  │  terminated │
└──────┬──────┘  │  down    │  │ (deleted)   │
       │         └────┬─────┘  └─────────────┘
       │              │
       ▼              ▼
   ┌──────────┐  ┌──────────┐
   │ stopped  │  │terminated│
   └──────────┘  └──────────┘
       │
       └────────────►(reboot)
              ▲
              │
       (restart after stop)
```

**Key Differences:**

```
Operation  │ Data    │ EBS Volumes │ Elastic IP │ Cost    │ Recoverable
───────────┼─────────┼─────────────┼────────────┼─────────┼──────────────
Stop       │ Kept    │ Attached    │ Retained   │ None    │ YES - restart
Terminate  │ Lost    │ Deleted     │ Released   │ None    │ NO - gone
Reboot     │ Kept    │ Attached    │ Retained   │ Charged │ Instance restarts
```

**Interview Question:** "What happens to EBS volumes when you terminate an EC2 instance?"

**Answer:** 
- By default, root volume is deleted (DeleteOnTermination=true)
- Additional volumes can be kept if DeleteOnTermination=false
- Snapshots can preserve data permanently

---

### Instance Metadata Service (IMDSv2)

**Q: Compare IMDSv1 and IMDSv2 from security perspective.**

**A (Advanced):**

```
IMDSv1 (Vulnerable - DEPRECATED):
EC2 Instance
    │
    └─► HTTP GET http://169.254.169.254/latest/meta-data/
        (IMMEDIATE response with credentials)

Attack Vector (SSRF):
Application vulnerability
    │
    └─► curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
        └─► Attacker gets IAM role credentials!

IMDSv2 (Secure - REQUIRED):
EC2 Instance
    │
    ├─► Step 1: GET http://169.254.169.254/latest/api/token
    │   X-aws-ec2-metadata-token-ttl-seconds: 21600
    │
    └─► (Returns token valid for 6 hours)
        │
        └─► Step 2: GET http://169.254.169.254/latest/meta-data/
            X-aws-ec2-metadata-token: <token>
            └─► Response with metadata

Security Benefit:
- Requires two HTTP calls (harder for SSRF)
- Token-based access
- Can't directly hit metadata endpoint
- Attacker needs valid token
```

**Implementation:**

```python
import boto3
import requests

# IMDSv2 - recommended
def get_credentials_v2():
    # Step 1: Get token
    token_response = requests.put(
        'http://169.254.169.254/latest/api/token',
        headers={'X-aws-ec2-metadata-token-ttl-seconds': '21600'},
        timeout=1
    )
    token = token_response.text
    
    # Step 2: Use token to get credentials
    metadata_response = requests.get(
        'http://169.254.169.254/latest/meta-data/iam/security-credentials/my-role',
        headers={'X-aws-ec2-metadata-token': token},
        timeout=1
    )
    
    return metadata_response.json()

# Enforce IMDSv2 only
ec2 = boto3.client('ec2')

ec2.modify_instance_metadata_options(
    InstanceId='i-1234567890abcdef0',
    HttpTokens='required',  # Require token (IMDSv2)
    HttpPutResponseHopLimit=1  # Token valid for 1 hop
)
```

**FAANG Interview Tip:** "We enforce IMDSv2 across all EC2 instances via SCP for security compliance."

---

### Nitro System Deep Dive

**Q: Explain AWS Nitro System and its benefits for performance.**

**A (Advanced):**

**What is Nitro?**
AWS custom-built hypervisor and system architecture replacing previous Xen-based systems.

**Components:**
```
┌────────────────────────────────────────────┐
│           EC2 Instance                     │
│  ┌──────────────────────────────────────┐  │
│  │  Guest Operating System              │  │
│  │  Application                         │  │
│  └──────────────────────────────────────┘  │
└──────────────────┬───────────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼                     ▼
    ┌────────┐           ┌───────────┐
    │ Nitro  │           │ Nitro     │
    │ System │           │ Security  │
    │        │           │ Module    │
    │ (Light)│           │ (HSM-like)│
    │ Hyper- │           │           │
    │ visor  │           │ Encrypts  │
    │        │           │ and       │
    │ Only   │           │ attests   │
    │ 50ms   │           │ all I/O   │
    └────────┘           └───────────┘

Benefits:
✓ Higher performance (less hypervisor overhead)
✓ Direct NIC/storage access (low latency)
✓ Consistent performance (predictable)
✓ Security hardening (isolated components)
✓ EBS and network performance up to 100 Gbps
```

**Performance Impact:**

```
Feature              │ Xen (Old)    │ Nitro (New)
─────────────────────┼──────────────┼──────────────
Hypervisor Overhead  │ 10-15%       │ ~1%
EBS Throughput       │ 25 Gbps      │ 100 Gbps
Network Throughput   │ 25 Gbps      │ 100 Gbps
Boot Time            │ 30-60s       │ 5-10s
I/O Latency          │ ~1ms         │ <0.1ms

Example (c5.large instance):
- Xen: Lose 10% CPU to hypervisor
- Nitro: Only 1% overhead
- User application gets 99% vs 90% CPU
```

**Nitro Enclaves (Secure Computation):**

```
Use case: Process highly sensitive data (PII, encryption keys)

Enclave = Isolated microVM within EC2 instance
├─ Dedicated CPU cores
├─ Dedicated memory
├─ No persistent storage (erased on stop)
├─ Only attested connections allowed
└─ Cannot be accessed by EC2 host

Example: Process credit cards for PCI compliance
Application → Nitro Enclave → Tokenize card → Return token
Host never sees actual card number!
```

---

## Instance Types and Sizing

### Instance Family Selection

**Q: How do you choose the right instance type for your workload?**

**A (Intermediate):**

```
┌─────────────────────────────────────────────────────────────┐
│            INSTANCE TYPE SELECTION MATRIX                  │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Compute Optimized (C)          Memory Optimized (R, X)    │
│  └─ ML training                 └─ Databases                │
│  └─ Batch processing            └─ In-memory caches         │
│  └─ Compilers                   └─ Real-time analytics      │
│                                                              │
│  Storage Optimized (I, D, H)    General Purpose (T, M)    │
│  └─ NoSQL databases              └─ Web servers              │
│  └─ Data warehousing            └─ Small databases          │
│  └─ Elasticsearch                └─ Dev/test                │
│                                                              │
│  GPU Accelerated (P, G)         Bare Metal (metal)         │
│  └─ Deep learning                └─ High-frequency trading   │
│  └─ Video encoding              └─ SAP HANA                │
│  └─ HPC                         └─ Database licensing       │
│                                                              │
│  ARM-based (Graviton - A, T4)   Burstable (T2/T3)         │
│  └─ Cost-optimized              └─ Dev/test                │
│  └─ Open-source workloads       └─ Small spikes            │
│  └─ Web tier                    └─ Credit-based burst      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Sizing Methodology:**

```
1. Baseline Performance (CloudWatch):
   - Average CPU: ____%
   - Peak CPU: ____%
   - Memory: ____GB
   - Network: ____Gbps
   
2. Instance Selection:
   - Current: t3.medium (1 vCPU, 4GB RAM)
   - Peak CPU: 85% → need more compute → c5.large
   - But memory only 2GB used → don't overspend on memory
   - Solution: c5.large (2 vCPU, 4GB RAM) = 2x cheaper than r5.large

3. Validation:
   - Can t3.large handle baseline? YES
   - Can it burst to peak? Check T3 burst credits
   - Cost comparison (monthly):
     * t3.large: $31/month
     * c5.large: $64/month
     * r5.large: $87/month
     
4. Final Choice: t3.large (burstable sufficient, cheapest)
```

**Example - Right-sizing an under-utilized RDS**

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

# Get metrics for past 30 days
response = cloudwatch.get_metric_statistics(
    Namespace='AWS/RDS',
    MetricName='CPUUtilization',
    Dimensions=[
        {
            'Name': 'DBInstanceIdentifier',
            'Value': 'production-db'
        }
    ],
    StartTime=datetime.datetime.now() - datetime.timedelta(days=30),
    EndTime=datetime.datetime.now(),
    Period=3600,
    Statistics=['Average', 'Maximum']
)

datapoints = response['Datapoints']
avg_cpu = sum(dp['Average'] for dp in datapoints) / len(datapoints)
max_cpu = max(dp['Maximum'] for dp in datapoints)

print(f"Average CPU: {avg_cpu:.1f}%")
print(f"Peak CPU: {max_cpu:.1f}%")

# Recommendation
if avg_cpu < 10 and max_cpu < 30:
    print("✓ DOWNSIZE: db.t3.large → db.t3.medium (SAVE 50%)")
elif avg_cpu > 80 or max_cpu > 90:
    print("✗ UPSIZE: db.t3.medium → db.t3.large")
else:
    print("✓ OPTIMAL: Current sizing appropriate")
```

---

## EBS Storage

### EBS Volume Types

**Q: When would you use io2 vs gp3 vs st1?**

**A (Intermediate):**

```
┌──────────────────────────────────────────────────────────┐
│                    EBS VOLUME TYPES                      │
├──────────────────────────────────────────────────────────┤
│                                                            │
│  GP3 (General Purpose - SSD)                             │
│  ├─ Baseline: 3,000 IOPS, 125 MB/s                       │
│  ├─ Burst: up to 16,000 IOPS, 1,000 MB/s                │
│  ├─ Cost: ~$0.10/month per GB                           │
│  ├─ Use: 90% of workloads (default choice)             │
│  ├─ Example: Web servers, small databases              │
│  └─ Buy separate IOPS & throughput (not volume-tied)   │
│                                                            │
│  IO2 (High Performance - SSD)                           │
│  ├─ Max: 64,000 IOPS, 1,000 MB/s                        │
│  ├─ 99.999% durability (vs 99.8% for gp3)             │
│  ├─ Cost: ~$0.25/month per GB + IOPS cost             │
│  ├─ Use: Databases (PostgreSQL, MySQL)                 │
│  ├─ Example: Multi-AZ databases requiring high IOPs   │
│  └─ High consistency SLAs                              │
│                                                            │
│  ST1 (Throughput Optimized - HDD)                      │
│  ├─ Throughput: up to 500 MB/s                         │
│  ├─ Cost: ~$0.045/month per GB (cheapest)            │
│  ├─ Use: Sequential workloads                         │
│  ├─ Example: Data warehouses, log processing          │
│  └─ NOT suitable for random I/O (databases)           │
│                                                            │
│  SC1 (Cold Storage - HDD)                              │
│  ├─ Throughput: 250 MB/s                              │
│  ├─ Cost: ~$0.025/month per GB (cheapest!)           │
│  ├─ Use: Infrequent access                            │
│  └─ Example: Archives, logs, infrequent backups       │
│                                                            │
└──────────────────────────────────────────────────────────┘
```

**Decision Matrix:**

```
Workload Type           │ Recommended │ Rationale
────────────────────────┼─────────────┼──────────────────────
Web server content      │ GP3         │ Balanced cost/performance
Mobile app backend      │ GP3         │ Burstable, cost-effective
Production database     │ IO2         │ High IOPS, durability
Data warehouse          │ ST1         │ High throughput sequential
Archive/infrequent logs │ SC1         │ Lowest cost
```

---

## Auto Scaling and Launch Templates

### Launch Template vs Launch Configuration

**Q: Why would you use Launch Template over Launch Configuration?**

**A (Intermediate):**

```
                 Launch Configuration  │  Launch Template
─────────────────────────────────────────────────────────
Versioning              │  NOT supported        │  Supported ✓
T2 Unlimited            │  NOT supported        │  Supported ✓
Spot + On-demand mix    │  NOT supported        │  Supported ✓
Placement groups        │  NOT supported        │  Supported ✓
GPU instances           │  NOT supported        │  Supported ✓
Tenancy (dedicated)     │  NOT supported        │  Supported ✓
Lifecycle               │  Immutable (delete)   │  Mutable ✓
                        │                        │  (create versions)

Modern AWS recommendation: ALWAYS use Launch Template
Launch Config: Legacy, maintained for backward compatibility
```

**Launch Template Example:**

```python
import boto3

ec2 = boto3.client('ec2')

# Create launch template
template_response = ec2.create_launch_template(
    LaunchTemplateName='web-server-template',
    LaunchTemplateData={
        'ImageId': 'ami-0c55b159cbfafe1f0',  # Amazon Linux 2
        'InstanceType': 't3.medium',
        'KeyName': 'my-key-pair',
        'SecurityGroupIds': ['sg-12345678'],
        'BlockDeviceMappings': [
            {
                'DeviceName': '/dev/xvda',
                'Ebs': {
                    'VolumeSize': 100,
                    'VolumeType': 'gp3',
                    'DeleteOnTermination': True,
                    'Encrypted': True
                }
            }
        ],
        'IamInstanceProfile': {
            'Arn': 'arn:aws:iam::123456789012:instance-profile/ec2-app-role'
        },
        'UserData': '''
            #!/bin/bash
            yum update -y
            yum install -y docker
            systemctl start docker
            docker pull myapp:latest
            docker run -d -p 80:8080 myapp:latest
        ''',
        'TagSpecifications': [
            {
                'ResourceType': 'instance',
                'Tags': [
                    {'Key': 'Name', 'Value': 'web-server'},
                    {'Key': 'Environment', 'Value': 'production'}
                ]
            }
        ],
        'MetadataOptions': {
            'HttpTokens': 'required',  # IMDSv2
            'HttpPutResponseHopLimit': 1
        },
        'Monitoring': {
            'Enabled': True  # Enable detailed monitoring
        }
    }
)

template_id = template_response['LaunchTemplate']['LaunchTemplateId']
```

### Auto Scaling Groups

**Q: Design Auto Scaling Group for an e-commerce website with variable load.**

**A (Advanced):**

```
Architecture:

Internet
    │
Route53 (DNS)
    │
    ▼
┌─────────────────────────────────────┐
│   Application Load Balancer         │
│   ├─ Health check: /health          │
│   ├─ Listener: 80, 443              │
│   └─ Target Group: web-servers      │
└────────────┬────────────────────────┘
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
┌─────────────┐   ┌─────────────┐
│ AZ-a        │   │ AZ-b        │
│             │   │             │
│ ┌─────────┐ │   │ ┌─────────┐ │
│ │ EC2-1   │ │   │ │ EC2-4   │ │
│ └─────────┘ │   │ └─────────┘ │
│ ┌─────────┐ │   │ ┌─────────┐ │
│ │ EC2-2   │ │   │ │ EC2-5   │ │
│ └─────────┘ │   │ └─────────┘ │
│ ┌─────────┐ │   │ ┌─────────┐ │
│ │ EC2-3   │ │   │ │ EC2-6   │ │
│ └─────────┘ │   │ └─────────┘ │
└─────────────┘   └─────────────┘

Auto Scaling Group Configuration:
```

```python
import boto3

autoscaling = boto3.client('autoscaling')

# Create Auto Scaling Group
autoscaling.create_auto_scaling_group(
    AutoScalingGroupName='web-servers-asg',
    LaunchTemplate={
        'LaunchTemplateId': 'lt-12345678',
        'Version': '$Latest'
    },
    MinSize=2,           # Minimum 2 instances
    MaxSize=20,          # Maximum 20 instances
    DesiredCapacity=4,   # Start with 4
    VPCZoneIdentifier='subnet-12345678,subnet-87654321',  # Multi-AZ
    TargetGroupARNs=['arn:aws:elasticloadbalancing:...'],
    HealthCheckType='ELB',              # Use ALB health checks
    HealthCheckGracePeriod=300,         # Wait 5 min after launch
    DefaultCooldown=300,                # Wait 5 min between scale actions
    TerminationPolicies=['OldestLaunchTemplate', 'Default']
)

# Create scaling policies
scale_up_policy = autoscaling.put_scaling_policy(
    AutoScalingGroupName='web-servers-asg',
    PolicyName='scale-up',
    PolicyType='TargetTrackingScaling',
    TargetTrackingConfiguration={
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'ASGAverageCPUUtilization'
        },
        'TargetValue': 70.0,  # Scale up if CPU > 70%
        'ScaleOutCooldown': 60,
        'ScaleInCooldown': 300
    }
)

scale_down_policy = autoscaling.put_scaling_policy(
    AutoScalingGroupName='web-servers-asg',
    PolicyName='scale-down',
    PolicyType='TargetTrackingScaling',
    TargetTrackingConfiguration={
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'ASGAverageCPUUtilization'
        },
        'TargetValue': 30.0,  # Scale down if CPU < 30%
        'ScaleOutCooldown': 60,
        'ScaleInCooldown': 600  # Longer to avoid thrashing
    }
)

# Set maximum instance lifetime (forces rotation)
autoscaling.update_auto_scaling_group(
    AutoScalingGroupName='web-servers-asg',
    MaxInstanceLifetime=2592000  # 30 days - instance auto-terminates
)
```

**Scaling Strategies:**

```
Strategy              │ When       │ Metric           │ Use Case
──────────────────────┼────────────┼──────────────────┼──────────────
Target Tracking      │ Proactive  │ CPUUtil, ALB RQ  │ Most common
Step Scaling         │ Aggressive │ Custom metrics   │ Complex rules
Scheduled Actions    │ Predictable│ Time-based       │ Known patterns

Example: E-commerce site
├─ 6-10 AM: Scale to 5 (morning traffic)
├─ 10 AM-3 PM: Scale to 10 (office browsing)
├─ 3-9 PM: Scale to 15 (peak shopping)
└─ 9 PM-6 AM: Scale to 2 (nighttime)
```

---

## Spot Instances and Cost Optimization

### Spot Instances with Interruption Handling

**Q: Design a cost-optimized application using Spot Instances that handles interruptions gracefully.**

**A (Advanced):**

```
Spot Instance Cost: 90% cheaper than on-demand
Risk: 2-5 minute interruption notice when AWS needs capacity

Solution: Mix on-demand + spot for resilience

Architecture:
┌─────────────────────────────────────────────┐
│   Auto Scaling Group (Mixed Instances)      │
│                                              │
│  Desired: 4 instances                       │
│  ├─ 1 on-demand (always available)         │
│  └─ 3 spot (cheaper, may be interrupted)   │
│                                              │
│  If spot interrupted:                       │
│  ├─ 2-min warning (EventBridge event)      │
│  ├─ Drain connections (deregister from ALB)│
│  └─ ASG launches replacement on-demand     │
│                                              │
│  Cost Savings: ~70% (3 spot + 1 on-demand)│
└─────────────────────────────────────────────┘
```

**Implementation:**

```python
import boto3
import json

autoscaling = boto3.client('autoscaling')
events = boto3.client('events')
lambda_client = boto3.client('lambda')

# 1. Create ASG with mixed instances
autoscaling.create_auto_scaling_group(
    AutoScalingGroupName='cost-optimized-asg',
    MixedInstancesPolicy={
        'LaunchTemplate': {
            'LaunchTemplateSpecification': {
                'LaunchTemplateId': 'lt-12345678',
                'Version': '$Latest'
            },
            'Overrides': [
                {'InstanceType': 't3.medium'},
                {'InstanceType': 't3a.medium'},  # Alternative
                {'InstanceType': 'm5.large'},
                {'InstanceType': 'm5a.large'}
            ]
        },
        'InstancesDistribution': {
            'OnDemandBaseCapacity': 1,    # Always 1 on-demand
            'OnDemandPercentageAboveBaseCapacity': 20,  # 20% on-demand, 80% spot
            'SpotAllocationStrategy': 'capacity-optimized'  # Diverse pools
        }
    },
    MinSize=1,
    MaxSize=10,
    DesiredCapacity=4,
    VPCZoneIdentifier='subnet-12345678,subnet-87654321'
)

# 2. Set up EventBridge rule for spot interruption warnings
events.put_rule(
    Name='spot-interruption-handler',
    EventPattern=json.dumps({
        'source': ['aws.ec2'],
        'detail-type': ['EC2 Instance State-change Notification'],
        'detail': {
            'state': ['running'],
            'instance-lifecycle': ['spot']
        }
    }),
    State='ENABLED'
)

# 3. Lambda function to handle interruption
lambda_code = '''
import boto3
import json

elb = boto3.client('elbv2')
autoscaling = boto3.client('autoscaling')

def lambda_handler(event, context):
    instance_id = event['detail']['instance-id']
    
    # Get target groups for this instance
    tg_response = elb.describe_target_groups()
    
    for tg in tg_response['TargetGroups']:
        # Deregister instance from target group
        elb.deregister_targets(
            TargetGroupArn=tg['TargetGroupArn'],
            Targets=[{'Id': instance_id}]
        )
        
        # Set to connection draining (complete existing requests)
        elb.modify_target_group_attributes(
            TargetGroupArn=tg['TargetGroupArn'],
            Attributes=[
                {
                    'Key': 'deregistration_delay.timeout_seconds',
                    'Value': '300'  # 5 minutes to drain
                }
            ]
        )
    
    return {'statusCode': 200, 'message': 'Instance drained'}
'''

# 4. Cost tracking
cost_explorer = boto3.client('ce')

response = cost_explorer.get_cost_and_usage(
    TimePeriod={
        'Start': '2024-08-01',
        'End': '2024-08-31'
    },
    Granularity='MONTHLY',
    Metrics=['UnblendedCost'],
    Filter={
        'Dimensions': {
            'Key': 'PURCHASE_TYPE',
            'Values': ['On Demand', 'Spot Instances']
        }
    }
)

# Analyze cost savings
for result in response['ResultsByTime']:
    od_cost = spot_cost = 0
    for group in result['Groups']:
        cost = float(group['Metrics']['UnblendedCost']['Amount'])
        if 'On Demand' in group['Keys']:
            od_cost = cost
        elif 'Spot' in group['Keys']:
            spot_cost = cost
    
    total = od_cost + spot_cost
    savings = (spot_cost / total) * 100 if total > 0 else 0
    print(f"On-Demand: ${od_cost} | Spot: ${spot_cost} | Spot %: {savings:.1f}%")
```

**Spot Interruption Rates (as of 2024):**

```
Instance Type         │ Interruption Rate
──────────────────────┼──────────────────
t3.medium             │ 0.5-2%
m5.large              │ 1-3%
c5.large              │ 2-5%
GPU (p3.2xlarge)      │ 5-15% (higher)

Best Practices:
1. Use capacity-optimized allocation (spreads across types)
2. Always have some on-demand as baseline
3. Set connection draining on ALB
4. Use EventBridge to catch interruptions early
5. Keep startup time < 2 minutes (within warning window)
6. Use automated rollback if health checks fail
```

---

## ECS Fundamentals

### ECS vs Kubernetes

**Q: When would you use ECS instead of EKS?**

**A (Intermediate):**

```
                  ECS              │  Kubernetes (EKS)
──────────────────────────────────────────────────────
Learning curve   │ Easy            │ Steep
AWS integration  │ Native ✓        │ Manual setup
EC2 management   │ AWS manages     │ DIY or use managed
Pricing          │ Simple          │ Complex
Best for:        │ AWS-only        │ Multi-cloud
                 │ Simple apps     │ Complex apps
                 │ Startups        │ Enterprise

ECS Use Cases:
✓ AWS-only SaaS
✓ Simple microservices (5-20 services)
✓ Fargate (serverless containers)
✓ Cost-conscious (simple pricing)

Kubernetes Use Cases:
✓ Multi-cloud strategy
✓ Complex microservices (100+ services)
✓ Existing K8s investments
✓ Advanced features (operators, custom resources)
```

### ECS Task Definition

**Q: Design an ECS Task Definition for a production API.**

**A:**

```json
{
  "family": "my-api-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "api-container",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-api:1.0.0",
      "portMappings": [
        {
          "containerPort": 8080,
          "hostPort": 8080,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "environment": [
        {
          "name": "LOG_LEVEL",
          "value": "INFO"
        },
        {
          "name": "ENV",
          "value": "production"
        }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-api",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

---

## EKS Deep Dive

### EKS Architecture

**Q: Explain EKS control plane and data plane separation.**

**A (Advanced - FAANG Interview):**

```
┌──────────────────────────────────────────────────────────────┐
│              EKS Cluster Architecture                        │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         AWS-MANAGED CONTROL PLANE (EKS Service)       │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │ API Server (kube-apiserver)                     │  │  │
│  │  │ - Authentication, authorization                │  │  │
│  │  │ - RESTful API for all operations               │  │  │
│  │  │ - 99.95% SLA guaranteed                        │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  │                                                          │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │ etcd (Distributed Key-Value Store)             │  │  │
│  │  │ - Stores all cluster state                      │  │  │
│  │  │ - Backed up to S3                               │  │  │
│  │  │ - Encrypted at rest                             │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  │                                                          │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │ Controller Manager (kube-controller-manager)   │  │  │
│  │  │ - Deployment controller                        │  │  │
│  │  │ - StatefulSet controller                       │  │  │
│  │  │ - DaemonSet controller                         │  │  │
│  │  │ - Service controller (creates CLBs, ALBs)     │  │  │
│  │  │ - Namespace controller                        │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  │                                                          │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │ Scheduler (kube-scheduler)                      │  │  │
│  │  │ - Assigns pods to nodes                         │  │  │
│  │  │ - Respects node selectors, affinities, taints  │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  │                                                          │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │ Add-ons                                         │  │  │
│  │  │ - CoreDNS (service discovery)                  │  │  │
│  │  │ - kube-proxy (network proxy)                   │  │  │
│  │  │ - CNI plugin (VPC CNI, Calico, etc.)          │  │  │
│  │  │ - kube-proxy                                   │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
│                           │                                    │
│                           │ (API calls via IAM auth)          │
│                           ▼                                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         CUSTOMER-MANAGED DATA PLANE (Your VPC)        │  │
│  │                                                        │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │  │
│  │  │  Node-1      │  │  Node-2      │  │  Node-3      │ │  │
│  │  │ (t3.large)   │  │ (t3.large)   │  │ (t3.large)   │ │  │
│  │  │              │  │              │  │              │ │  │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │ │  │
│  │  │ │kubelet   │ │  │ │kubelet   │ │  │ │kubelet   │ │ │  │
│  │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │ │  │
│  │  │              │  │              │  │              │ │  │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │ │  │
│  │  │ │kube-proxy│ │  │ │kube-proxy│ │  │ │kube-proxy│ │ │  │
│  │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │ │  │
│  │  │              │  │              │  │              │ │  │
│  │  │ Pod: api-1   │  │ Pod: api-2   │  │ Pod: api-3   │ │  │
│  │  │ Pod: db-1    │  │ Pod: cache-1 │  │              │ │  │
│  │  └──────────────┘  │ └──────────┐ │  │ └──────────────┘ │  │
│  │                                  │  │                   │  │
│  └──────────────────────────────────┼──┼───────────────────┘  │
│                                      │  │                      │
│  ┌──────────────────────────────────┴──┴───────────────────┐  │
│  │  Shared Services                                        │  │
│  │  - VPC CNI (pod-to-pod networking)                     │  │
│  │  - IAM Roles for Service Accounts (IRSA)              │  │
│  │  - AWS Load Balancer Controller                       │  │
│  │  - Cluster Autoscaler / Karpenter                     │  │
│  │  - Container Insights (monitoring)                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

**Key Components:**

```
Control Plane (AWS-Managed):
├─ NOT charged extra (included in EKS cost)
├─ Highly available across 3 AZs
├─ Automatic patching
├─ Encrypted by default
└─ You don't manage these

Data Plane (Your Responsibility):
├─ EC2 instances or Fargate
├─ You manage patching (if EC2)
├─ Pay for compute costs (EC2/Fargate)
├─ Configure security groups
└─ Configure IAM roles
```

### Kubernetes Core Concepts for Interviews

**Q: Explain Pods, Deployments, Services, and StatefulSets with production examples.**

**A (Advanced):**

```yaml
# Pod = Smallest deployable unit (not usually created directly)
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
  - name: logging-sidecar
    image: logging-agent:1.0
    # Sidecar patterns: observability, security, networking

---

# Deployment = Stateless application (most common)
# Manages ReplicaSets, handles rolling updates
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
      tier: backend
  template:
    metadata:
      labels:
        app: api
        tier: backend
    spec:
      containers:
      - name: api
        image: myapp:1.0.0
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - api
              topologyKey: kubernetes.io/hostname

---

# Service = Network abstraction (stable DNS name)
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: api
    tier: backend
  sessionAffinity: ClientIP  # Sticky sessions

---

# StatefulSet = Stateful application (databases)
# Maintains stable pod identity, ordered deployment
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-cluster
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: ebs-sc
      resources:
        requests:
          storage: 100Gi
```

**Comparison:**

```
                Deployment      StatefulSet
────────────────────────────────────────
Pod naming      Random          Predictable (mysql-0, mysql-1, mysql-2)
Storage         Shared/ephemeral Persistent per pod
Order           Parallel        Sequential
Use case        Stateless apps   Databases, message queues
Update strategy Rolling         OrderedReady
Replicas        Interchangeable  Identified
```

---

## Kubernetes Networking (CNI Deep Dive)

**Q: Explain how VPC CNI provides pod-to-pod networking in EKS.**

**A (FAANG Deep Dive):**

```
VPC CNI = Container Network Interface plugin
Integrates Kubernetes pods directly with AWS VPC networking

Architecture:

┌─────────────────────────────────────────────────┐
│            AWS VPC (10.0.0.0/16)               │
│                                                  │
│  ┌──────────────────────────────────────────┐  │
│  │   EC2 Node (10.0.1.100)                 │  │
│  │   ├─ Primary ENI (10.0.1.100)          │  │
│  │   │  └─ Pods get IPs from node's subnet│  │
│  │   │                                     │  │
│  │   ├─ Secondary ENI (10.0.1.101)       │  │
│  │   │  └─ Pod: pod-1 (10.0.1.50)        │  │
│  │   │  └─ Pod: pod-2 (10.0.1.51)        │  │
│  │   │                                    │  │
│  │   └─ Secondary ENI (10.0.1.102)      │  │
│  │      └─ Pod: pod-3 (10.0.1.52)       │  │
│  │      └─ Pod: pod-4 (10.0.1.53)       │  │
│  │                                       │  │
│  └──────────────────────────────────────┘  │
│                                             │
│  ┌──────────────────────────────────────┐  │
│  │   EC2 Node (10.0.2.100)             │  │
│  │   ├─ Primary ENI (10.0.2.100)      │  │
│  │   └─ Secondary ENI (10.0.2.101)   │  │
│  │      └─ Pod: pod-5 (10.0.2.50)    │  │
│  │      └─ Pod: pod-6 (10.0.2.51)    │  │
│  │                                   │  │
│  └──────────────────────────────────┘  │
│                                         │
└─────────────────────────────────────────────┘

Key Points:
✓ Pods get REAL VPC IP addresses (not overlay network)
✓ No encapsulation overhead (faster than Calico)
✓ Security Groups apply to pods
✓ VPC Flow Logs capture pod traffic
✓ Native VPC routing

Pod-to-Pod Communication:
pod-1 (10.0.1.50) → pod-5 (10.0.2.50)
1. pod-1 sends packet to 10.0.2.50
2. VPC router sees destination 10.0.2.50 in VPC CIDR
3. Routes to 10.0.2.100 (node ENI) using VPC routing table
4. Kernel on node routes to pod via veth pair
5. Pod-5 receives packet
```

**VPC CNI Configuration:**

```python
import boto3

eks = boto3.client('eks')

# Get cluster
cluster = eks.describe_cluster(name='my-cluster')

# Check VPC CNI version
addons = eks.describe_addon(
    clusterName='my-cluster',
    addonName='vpc-cni'
)

print(f"VPC CNI Version: {addons['addon']['addonVersion']}")
print(f"Status: {addons['addon']['addonHealth']['issues']}")

# Update VPC CNI
eks.update_addon(
    clusterName='my-cluster',
    addonName='vpc-cni',
    addonVersion='v1.14.1-eksbuild.1'
)
```

**Advanced CNI Settings:**

```yaml
# ConfigMap for VPC CNI
apiVersion: v1
kind: ConfigMap
metadata:
  name: amazon-vpc-cni
  namespace: kube-system
data:
  # Warm IP targets = pre-allocated IPs for faster pod launch
  WARM_IP_TARGET: "10"
  
  # Minimum IP target = minimum free IPs to maintain
  MINIMUM_IP_TARGET: "5"
  
  # Warm ENI target = keep this many secondary ENIs warm
  WARM_ENI_TARGET: "1"
  
  # Enable prefix delegation (more pods per node)
  ENABLE_PREFIX_DELEGATION: "true"
  
  # IPv6 support
  ENABLE_WINDOWS_IPAM: "false"

---
# Pod example
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: default
spec:
  securityContext:
    fsGroup: 1000
  containers:
  - name: app
    image: myapp:1.0
    securityContext:
      runAsUser: 1000
    # Pod gets IP from node's subnet via VPC CNI
```

---

## Pod Identity and IRSA

**Q: Compare IRSA and newer Pod Identity. When would you use each?**

**A (Advanced):**

```
IRSA (IAM Roles for Service Accounts) - Original approach:
├─ Service Account → OIDC Provider → IAM Role
├─ Setup: Create OIDC provider, IAM role, trust relationship
├─ Complexity: Medium (requires CloudFormation)
├─ Security: Good (token-based, time-limited)

Pod Identity (Newer AWS approach) - Simplified:
├─ Pod Identity Association → IAM Role
├─ Setup: One command (kubectl or AWS CLI)
├─ Complexity: Low (AWS handles it)
├─ Security: Excellent (agent-based, no token exposure)
├─ Recommended for: New clusters

Both work, but Pod Identity is the modern way
```

**IRSA Setup (for interview knowledge):**

```python
import boto3
import json

iam = boto3.client('iam')
eks = boto3.client('eks')

# 1. Get OIDC provider URL
cluster = eks.describe_cluster(name='my-cluster')
oidc_url = cluster['cluster']['identity']['oidc']['issuer']
# Returns: https://oidc.eks.region.amazonaws.com/id/EXAMPLEEXAMPLEEXAMPLE

# 2. Create IAM role
trust_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": f"arn:aws:iam::123456789012:oidc-provider/oidc.eks.region.amazonaws.com/id/EXAMPLEEXAMPLEEXAMPLE"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.region.amazonaws.com/id/EXAMPLEEXAMPLEEXAMPLE:sub": "system:serviceaccount:default:my-app"
                }
            }
        }
    ]
}

iam.create_role(
    RoleName='my-app-role',
    AssumeRolePolicyDocument=json.dumps(trust_policy)
)

# 3. Attach policy
iam.attach_role_policy(
    RoleName='my-app-role',
    PolicyArn='arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess'
)

# 4. Create Kubernetes ServiceAccount
# kubectl annotate serviceaccount my-app -n default \
#   eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/my-app-role

# 5. Pod uses credentials automatically via projected volume
```

**Pod Identity Setup (Newer, Recommended):**

```python
# 1. Create Pod Identity Association (one command!)
eks.create_pod_identity_association(
    clusterName='my-cluster',
    namespace='default',
    serviceAccount='my-app',
    roleArn='arn:aws:iam::123456789012:role/my-app-role'
)

# Done! Pod automatically gets credentials via local agent
# No OIDC, no web identity, no complex setup
```

---

## EKS Troubleshooting

### Common Pod Issues

**Q: Pod is stuck in "Pending" state. How do you debug?**

**A (Advanced - Production Scenario):**

```
Symptoms:
- Pod created 10 minutes ago
- Still in Pending state
- No "Running" status

Diagnostic Steps:

1. Check pod status
$ kubectl get pods -A
$ kubectl describe pod <pod-name> -n <namespace>

Look for Events section:
└─ "FailedScheduling" = Scheduler couldn't find node
└─ "Unschedulable" = Resource constraints

2. Check node resources
$ kubectl get nodes
$ kubectl top nodes

Possible causes:
├─ Not enough CPU (request > available)
├─ Not enough memory
├─ No nodes available in AZ
├─ Node selector doesn't match
└─ Taints prevent scheduling

3. Check resource requests
$ kubectl describe pod <pod-name> | grep -A5 Requests

Example issue:
Pod requests: 2000m CPU, but largest node has only 1000m available
Solution: 
  a) Reduce pod CPU request
  b) Add more nodes (ASG scaling)
  c) Use Karpenter for automatic node provisioning

4. Check node taints
$ kubectl describe node <node-name> | grep Taints

Example: node-role.kubernetes.io/gpu:NoSchedule
Pod without gpu toleration can't schedule here

5. Check node affinity
$ kubectl describe pod <pod-name> | grep -A10 Node-Selectors

If pod has strict node selector and no matching nodes exist,
pod stays pending forever

Solution:
$ kubectl label node <node-name> key=value
Or remove node selector constraint
```

**Debugging Script:**

```bash
#!/bin/bash

POD_NAME=$1
NAMESPACE=${2:-default}

echo "=== Pod Status ==="
kubectl get pod $POD_NAME -n $NAMESPACE -o wide

echo "=== Pod Events ==="
kubectl describe pod $POD_NAME -n $NAMESPACE | tail -20

echo "=== Node Resources ==="
kubectl top nodes

echo "=== Pod Resource Requests ==="
kubectl get pod $POD_NAME -n $NAMESPACE -o jsonpath='{.spec.containers[*].resources}'

echo "=== Check if Nodes Available ==="
kubectl get nodes --show-labels

echo "=== Check Cluster Autoscaler Logs ==="
kubectl logs -n kube-system deployment/cluster-autoscaler | tail -10
```

### Node Issues

**Q: EC2 node shows "NotReady" status. How do you recover?**

**A (Production Troubleshooting):**

```
Symptoms:
$ kubectl get nodes
NAME           STATUS      ROLES    AGE
node-1         NotReady    <none>   5h

Event: "Node node-1 has become NotReady: kubelet has not posted since 5 minutes"

Causes & Solutions:

1. Node crashed or rebooted
   Solution: Scale node out and in (ASG will replace)
   $ aws autoscaling set-desired-capacity --auto-scaling-group-name asg-1 --desired-capacity 3

2. Kubelet service stopped
   Solution: SSH to node and restart kubelet
   $ sudo systemctl restart kubelet
   
3. Disk pressure (node disk full)
   $ kubectl describe node node-1 | grep -i disk
   
   Solution:
   - Clean docker images: docker system prune
   - Increase EBS volume size
   - Enable docker cleanup policy

4. Memory pressure (OOM)
   $ kubectl describe node node-1 | grep -i memory
   
   Check which pod is consuming memory:
   $ kubectl top pods -A --sort-by=memory

5. Network connectivity issue
   $ kubectl get node node-1 -o wide
   
   Check if node can reach API server:
   $ curl -k https://<api-server>:443
   
   Check security groups and NACLs

6. Kubelet can't pull image
   Solution: Check ECR access, IAM permissions

Permanent fix (drain and terminate):
$ kubectl drain node-1 --ignore-daemonsets --delete-empty-dir-data
$ kubectl delete node node-1
# ASG will launch replacement
```

---

## Interview Questions

### Q1: Design EKS cluster for SaaS multi-tenant platform

**A:** (See system design section)

### Q2: Pod to database connection timeout. Debugging flow?

**A:**
1. Check pod logs: `kubectl logs <pod>`
2. Verify pod network: `kubectl exec <pod> -- ip route`
3. Test DNS: `kubectl exec <pod> -- nslookup <db-service>`
4. Check security groups: `aws ec2 describe-security-groups`
5. Verify NACL rules
6. Check RDS security group allows pod CIDR
7. Test connectivity from pod: `kubectl exec <pod> -- telnet <db-host> 3306`

### Q3: 50% of pods are running, others failing. What's happening?

**A:** Likely node failure or insufficient resources. 
- Check node status: `kubectl get nodes`
- Check events: `kubectl describe node <node>`
- Check ASG: Is it scaling up?
- Check Cluster Autoscaler logs

---

