# AWS CI/CD and Infrastructure as Code - Comprehensive Interview Guide

> Deep dive into CodePipeline, CloudFormation, CDK, Terraform, and GitOps patterns

**Estimated Reading Time:** 90 minutes | **Coverage:** 120+ interview questions

---

## Table of Contents

- [CI/CD Fundamentals](#cicd-fundamentals)
- [AWS CodePipeline](#aws-codepipeline)
- [CloudFormation Deep Dive](#cloudformation-deep-dive)
- [CDK vs Terraform](#cdk-vs-terraform)
- [GitOps with ArgoCD and FluxCD](#gitops-with-argocd-and-fluxcd)
- [Interview Questions](#interview-questions)

---

## CI/CD Fundamentals

### CI/CD Pipeline Stages

**Q: Design a production-grade CI/CD pipeline for microservices.**

**A (Advanced Architecture):**

```
Git Push (main branch)
    │
    ▼
┌─────────────────────────────────┐
│   Stage 1: Source              │
│   ├─ GitHub/CodeCommit webhook  │
│   └─ Trigger pipeline           │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│   Stage 2: Build                │
│   ├─ CodeBuild compiles code    │
│   ├─ Runs unit tests            │
│   ├─ Builds Docker image        │
│   └─ Push to ECR                │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│   Stage 3: Test                 │
│   ├─ Integration tests          │
│   ├─ Security scanning (SAST)   │
│   ├─ Dependency check           │
│   └─ Code quality gates         │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│   Stage 4: Deploy to Dev        │
│   ├─ Manual approval (optional) │
│   ├─ Blue-green deployment      │
│   └─ Smoke tests                │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│   Stage 5: Deploy to Staging    │
│   ├─ Manual approval required   │
│   ├─ E2E tests                  │
│   └─ Performance tests          │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│   Stage 6: Deploy to Prod       │
│   ├─ Canary deployment (10%)    │
│   ├─ Health checks              │
│   └─ Gradual rollout (90%)      │
└──────────────┬──────────────────┘
               │
               ▼
        Production Live

Success Metrics:
- Build time: < 5 minutes
- Test time: < 10 minutes
- Deploy time: < 5 minutes
- Failure rate: < 1%
```

### Deployment Strategies

**Q: Compare blue-green, canary, and rolling deployments.**

**A (Production Knowledge):**

```
Blue-Green Deployment:
├─ Blue (v1.0): Currently live, receiving 100% traffic
├─ Green (v1.1): New version, deployed but NO traffic
├─ Switch: Instantly move ALL traffic to green
├─ Rollback: Instant (switch back to blue)
├─ Downtime: Zero (no traffic loss)
├─ Resources: 2x (need capacity for both)
└─ Best for: Quick rollback requirement

Blue:  v1.0 ─── 100% traffic
Green: v1.1 ─── 0% traffic
       Instant switch →
Blue:  v1.0 ─── 0% traffic
Green: v1.1 ─── 100% traffic

Canary Deployment:
├─ v1.0: 90% traffic (stable)
├─ v1.1: 10% traffic (new)
├─ Monitor: Metrics for both versions
├─ If v1.1 healthy: Gradually shift traffic
├─ Rollback: Automatic if v1.1 errors high
├─ Downtime: Zero (controlled traffic shift)
└─ Best for: Early detection of issues

Start:  v1.0 ─── 90% traffic
        v1.1 ─── 10% traffic
After 5min:
        v1.0 ─── 50% traffic
        v1.1 ─── 50% traffic
After 10min:
        v1.0 ─── 0% traffic
        v1.1 ─── 100% traffic

Rolling Deployment:
├─ v1.0 (pod 1-4): Running
├─ Replace pod 1: v1.1
├─ Wait: Health check passes
├─ Replace pod 2: v1.1
├─ Continue until all replaced
├─ Downtime: Minimal (some pods always running)
├─ Resources: Same (no extra capacity needed)
└─ Best for: Cost-conscious deployments

v1.0 → v1.1
v1.0 → v1.1
v1.0 → v1.0 (still running old)
v1.0 → v1.0
(Service available throughout)
```

---

## AWS CodePipeline

### CodePipeline Architecture

**Q: Design a CodePipeline that deploys to EKS with approval gates.**

**A:**

```python
import boto3

codepipeline = boto3.client('codepipeline')

# Create pipeline
codepipeline.create_pipeline(
    pipeline={
        'name': 'microservice-pipeline',
        'roleArn': 'arn:aws:iam::123456789012:role/codepipeline-role',
        'artifactStore': {
            'type': 'S3',
            'location': 'pipeline-artifacts-bucket'
        },
        'stages': [
            # Stage 1: Source
            {
                'name': 'Source',
                'actions': [
                    {
                        'name': 'SourceAction',
                        'actionTypeId': {
                            'category': 'Source',
                            'owner': 'ThirdParty',
                            'provider': 'GitHub',
                            'version': '1'
                        },
                        'configuration': {
                            'Owner': 'mycompany',
                            'Repo': 'microservice',
                            'Branch': 'main',
                            'OAuthToken': 'github-token'
                        },
                        'outputArtifacts': [
                            {'name': 'SourceOutput'}
                        ]
                    }
                ]
            },
            
            # Stage 2: Build
            {
                'name': 'Build',
                'actions': [
                    {
                        'name': 'BuildAction',
                        'actionTypeId': {
                            'category': 'Build',
                            'owner': 'AWS',
                            'provider': 'CodeBuild',
                            'version': '1'
                        },
                        'configuration': {
                            'ProjectName': 'microservice-build'
                        },
                        'inputArtifacts': [
                            {'name': 'SourceOutput'}
                        ],
                        'outputArtifacts': [
                            {'name': 'BuildOutput'}
                        ]
                    }
                ]
            },
            
            # Stage 3: Deploy to Dev (Automatic)
            {
                'name': 'DeployDev',
                'actions': [
                    {
                        'name': 'DeployDevAction',
                        'actionTypeId': {
                            'category': 'Deploy',
                            'owner': 'AWS',
                            'provider': 'CloudFormation',
                            'version': '1'
                        },
                        'configuration': {
                            'ActionMode': 'CHANGE_SET_EXECUTE',
                            'StackName': 'microservice-dev',
                            'ChangeSetName': 'microservice-dev-changeset',
                            'TemplatePath': 'BuildOutput::packaged.yaml',
                            'Capabilities': 'CAPABILITY_IAM,CAPABILITY_NAMED_IAM',
                            'ParameterOverrides': '{"Environment":"dev"}'
                        },
                        'inputArtifacts': [
                            {'name': 'BuildOutput'}
                        ]
                    }
                ]
            },
            
            # Stage 4: Manual Approval for Staging
            {
                'name': 'ApprovalStaging',
                'actions': [
                    {
                        'name': 'ApprovalAction',
                        'actionTypeId': {
                            'category': 'Approval',
                            'owner': 'AWS',
                            'provider': 'Manual',
                            'version': '1'
                        },
                        'configuration': {
                            'CustomData': 'Approve deployment to staging environment?',
                            'NotificationArn': 'arn:aws:sns:region:account:approvals-topic'
                        }
                    }
                ]
            },
            
            # Stage 5: Deploy to Staging (After Approval)
            {
                'name': 'DeployStaging',
                'actions': [
                    {
                        'name': 'DeployEKS',
                        'actionTypeId': {
                            'category': 'Deploy',
                            'owner': 'AWS',
                            'provider': 'CloudFormation',
                            'version': '1'
                        },
                        'configuration': {
                            'ActionMode': 'CHANGE_SET_EXECUTE',
                            'StackName': 'microservice-staging',
                            'ChangeSetName': 'microservice-staging-changeset',
                            'TemplatePath': 'BuildOutput::packaged.yaml',
                            'Capabilities': 'CAPABILITY_IAM,CAPABILITY_NAMED_IAM',
                            'ParameterOverrides': '{"Environment":"staging"}'
                        },
                        'inputArtifacts': [
                            {'name': 'BuildOutput'}
                        ]
                    }
                ]
            },
            
            # Stage 6: Manual Approval for Production
            {
                'name': 'ApprovalProd',
                'actions': [
                    {
                        'name': 'ApprovalAction',
                        'actionTypeId': {
                            'category': 'Approval',
                            'owner': 'AWS',
                            'provider': 'Manual',
                            'version': '1'
                        },
                        'configuration': {
                            'CustomData': 'Approve deployment to PRODUCTION? This affects 100% of users!',
                            'NotificationArn': 'arn:aws:sns:region:account:approvals-topic',
                            'ExternalEntityLink': 'https://monitoring.company.com/staging-metrics'
                        }
                    }
                ]
            },
            
            # Stage 7: Deploy to Production (Canary)
            {
                'name': 'DeployProd',
                'actions': [
                    {
                        'name': 'DeployCanary',
                        'actionTypeId': {
                            'category': 'Deploy',
                            'owner': 'AWS',
                            'provider': 'ServiceCatalog',
                            'version': '1'
                        },
                        'configuration': {
                            'ProductId': 'prod-microservice',
                            'ProvisioningArtifactId': 'artifact-123',
                            'ProvisioningParameters': '{"DesiredCanaryPercentage":"10"}'
                        },
                        'inputArtifacts': [
                            {'name': 'BuildOutput'}
                        ]
                    }
                ]
            }
        ]
    }
)
```

---

## CloudFormation Deep Dive

### Template Structure

**Q: Design CloudFormation template for production VPC with multi-AZ setup.**

**A (Advanced):**

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Production VPC with multi-AZ setup'

Parameters:
  Environment:
    Type: String
    Default: production
    AllowedValues: [dev, staging, production]
  
  VpcCIDR:
    Type: String
    Default: '10.0.0.0/16'
    Description: VPC CIDR block

Conditions:
  IsProduction: !Equals [!Ref Environment, 'production']
  IsStaging: !Equals [!Ref Environment, 'staging']

Resources:
  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc'

  # Internet Gateway
  IGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-igw'

  AttachIGW:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref VPC
      InternetGatewayId: !Ref IGW

  # Public Subnets
  PublicSubnetAZ1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.1.0/24'
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-az1'

  PublicSubnetAZ2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.11.0/24'
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-az2'

  # Private Subnets
  PrivateSubnetAZ1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.2.0/24'
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-az1'

  PrivateSubnetAZ2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: '10.0.12.0/24'
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-az2'

  # NAT Gateway for HA
  EIPNATZ1:
    Type: AWS::EC2::EIP
    DependsOn: AttachIGW
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-eip-nat-az1'

  NATGatewayAZ1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt EIPNATZ1.AllocationId
      SubnetId: !Ref PublicSubnetAZ1
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-az1'

  EIPNATZ2:
    Type: AWS::EC2::EIP
    Condition: IsProduction
    DependsOn: AttachIGW
    Properties:
      Domain: vpc

  NATGatewayAZ2:
    Type: AWS::EC2::NatGateway
    Condition: IsProduction
    Properties:
      AllocationId: !If [IsProduction, !GetAtt EIPNATZ2.AllocationId, !Ref AWS::NoValue]
      SubnetId: !Ref PublicSubnetAZ2

  # Route Tables
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-rt'

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachIGW
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId: !Ref IGW

  PublicSubnetAZ1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetAZ1
      RouteTableId: !Ref PublicRouteTable

  PublicSubnetAZ2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnetAZ2
      RouteTableId: !Ref PublicRouteTable

  # Private Route Table
  PrivateRouteTableAZ1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-rt-az1'

  PrivateRouteAZ1:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTableAZ1
      DestinationCidrBlock: '0.0.0.0/0'
      NatGatewayId: !Ref NATGatewayAZ1

  PrivateSubnetAZ1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnetAZ1
      RouteTableId: !Ref PrivateRouteTableAZ1

  PrivateRouteTableAZ2:
    Type: AWS::EC2::RouteTable
    Condition: IsProduction
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-rt-az2'

  PrivateRouteAZ2:
    Type: AWS::EC2::Route
    Condition: IsProduction
    Properties:
      RouteTableId: !Ref PrivateRouteTableAZ2
      DestinationCidrBlock: '0.0.0.0/0'
      NatGatewayId: !If [IsProduction, !Ref NATGatewayAZ2, !Ref AWS::NoValue]

  PrivateSubnetAZ2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Condition: IsProduction
    Properties:
      SubnetId: !Ref PrivateSubnetAZ2
      RouteTableId: !If [IsProduction, !Ref PrivateRouteTableAZ2, !Ref AWS::NoValue]

  # Network ACLs
  PrivateNACL:
    Type: AWS::EC2::NetworkAcl
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-nacl'

  # Allow inbound from VPC
  PrivateNACLInboundVPC:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref PrivateNACL
      RuleNumber: 100
      Protocol: -1  # All protocols
      RuleAction: allow
      CidrBlock: !Ref VpcCIDR

  # Allow inbound ephemeral ports (responses)
  PrivateNACLInboundEphemeral:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref PrivateNACL
      RuleNumber: 110
      Protocol: 6  # TCP
      RuleAction: allow
      CidrBlock: '0.0.0.0/0'
      PortRange:
        FromPort: 1024
        ToPort: 65535

  # Allow outbound to internet
  PrivateNACLOutbound:
    Type: AWS::EC2::NetworkAclEntry
    Properties:
      NetworkAclId: !Ref PrivateNACL
      RuleNumber: 100
      Protocol: -1
      Egress: true
      RuleAction: allow
      CidrBlock: '0.0.0.0/0'

Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${Environment}-VpcId'

  PublicSubnets:
    Description: Public subnet IDs
    Value: !Join [',', [!Ref PublicSubnetAZ1, !Ref PublicSubnetAZ2]]
    Export:
      Name: !Sub '${Environment}-PublicSubnets'

  PrivateSubnets:
    Description: Private subnet IDs
    Value: !If
      - IsProduction
      - !Join [',', [!Ref PrivateSubnetAZ1, !Ref PrivateSubnetAZ2]]
      - !Ref PrivateSubnetAZ1
    Export:
      Name: !Sub '${Environment}-PrivateSubnets'
```

**CloudFormation Best Practices:**

```yaml
# Use Parameters for flexibility
Parameters:
  InstanceType:
    Type: String
    Default: t3.medium
    AllowedValues: [t3.small, t3.medium, t3.large]

# Use Conditions for environment-specific logic
Conditions:
  IsProduction: !Equals [!Ref Environment, 'production']
  IsLargeDeploy: !Or
    - !Condition IsProduction
    - !Equals [!Ref InstanceType, 't3.large']

# Use Mappings for region/AZ data
Mappings:
  AmiMap:
    us-east-1:
      ami: ami-12345678
    us-west-2:
      ami: ami-87654321

# Export outputs for cross-stack references
Outputs:
  VpcId:
    Export:
      Name: !Sub '${Environment}-VpcId'
```

---

## CDK vs Terraform

### CDK (AWS-Specific)

**Q: When would you use CDK vs Terraform?**

**A:**

```
CDK (Cloud Development Kit):

Pros:
✓ Write IaC in Python/TypeScript/Java
✓ Full AWS API access
✓ Loops, conditionals, OOP patterns
✓ Faster development for AWS-only
✓ Powerful abstractions (Constructs)

Cons:
✗ AWS-only (not multi-cloud)
✗ Smaller community than Terraform
✗ Generates CloudFormation (indirect)
✗ Version lock-in

Best for:
- AWS-only shops
- Complex logic
- Team knows programming languages
- Rapid iteration

CDK Example:
```

```python
from aws_cdk import (
    aws_ec2 as ec2,
    aws_ecs as ecs,
    aws_efs as efs,
    core
)

class MyAppStack(core.Stack):
    def __init__(self, scope: core.Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)
        
        # Create VPC
        vpc = ec2.Vpc(self, "VPC", max_azs=3)
        
        # Create ECS cluster
        cluster = ecs.Cluster(self, "Cluster", vpc=vpc)
        
        # Add capacity
        cluster.add_capacity(
            "DefaultAutoScalingGroup",
            instance_type=ec2.InstanceType("t3.medium"),
            desired_capacity=3
        )
```

### Terraform (Multi-Cloud)

**Pros:**
- Multi-cloud (AWS, Azure, GCP)
- Largest community
- Mature ecosystem
- State management
- Plan before apply (safe)

**Cons:**
- More verbose
- HCL syntax learning curve
- State file management complexity
- Slower for simple deploys

**Best for:**
- Multi-cloud strategy
- Large teams
- Mature infrastructure

```hcl
# Terraform example
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "private" {
  count            = 3
  vpc_id           = aws_vpc.main.id
  cidr_block       = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-subnet-${count.index + 1}"
  }
}

# Reference other resources
resource "aws_instance" "app" {
  count           = 3
  ami             = data.aws_ami.amazon_linux.id
  instance_type   = "t3.medium"
  subnet_id       = aws_subnet.private[count.index].id

  tags = {
    Name = "app-server-${count.index + 1}"
  }
}
```

---

## GitOps with ArgoCD and FluxCD

### ArgoCD Architecture

**Q: Design GitOps workflow with ArgoCD for multi-environment deployments.**

**A:**

```
Git Repository (Source of Truth):
├─ main branch (production)
├─ staging branch
└─ dev branch

Each branch contains Kubernetes manifests

ArgoCD monitors Git repo
    │
    ├─ Detects changes
    │
    ├─ Pulls manifests
    │
    └─ Applies to cluster

Continuous Reconciliation:
Git State ≠ Cluster State → ArgoCD syncs automatically
```

**Installation and Setup:**

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 3. Get default password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# 4. Create Application CRD
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mycompany/myapp
    targetRevision: HEAD
    path: k8s/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

**Interview Advantage:**
"We use ArgoCD for GitOps. Every Kubernetes change goes through Git. This gives us audit trail, easy rollback, and environment parity."

---

## Interview Questions

### Q1: CloudFormation drift detection

**A:** Detect when actual resources differ from template.
```bash
aws cloudformation detect-stack-drift --stack-name my-stack
aws cloudformation get-stack-drift-detection-status --stack-drift-detection-id drift-id
```

### Q2: Terraform state corruption

**A:** Backup, lock state, plan before apply, use remote state (S3+DynamoDB)

### Q3: CodePipeline failed deploy, rollback?

**A:** 
- Automatic: Use blue-green or canary with health checks
- Manual: CodePipeline retry or invoke different stage
- Database: Use RDS automated backups

