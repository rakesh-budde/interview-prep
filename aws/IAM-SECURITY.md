# AWS IAM and Security - Comprehensive Guide

> Deep dive into Identity and Access Management, Secrets Management, Encryption, and Security Services

---

## Table of Contents

- [IAM Fundamentals](#iam-fundamentals)
- [Policy Evaluation and Permissions](#policy-evaluation-and-permissions)
- [Advanced IAM Patterns](#advanced-iam-patterns)
- [Secrets Management](#secrets-management)
- [Encryption (KMS, CloudHSM)](#encryption)
- [Network Security](#network-security)
- [Detective Controls](#detective-controls)
- [Troubleshooting IAM](#troubleshooting-iam)
- [Interview Questions](#interview-questions)

---

## IAM Fundamentals

### Users vs Roles vs Groups

**Questions:**

**Q: What's the difference between IAM Users and IAM Roles?**

**A (Intermediate):**

```
┌─────────────────────────────┬──────────────────────────────┐
│ IAM User                    │ IAM Role                     │
├─────────────────────────────┼──────────────────────────────┤
│ Long-term credentials       │ Temporary credentials (STS)  │
│ - Access Key ID             │ - AccessKeyId                │
│ - Secret Access Key         │ - SecretAccessKey            │
│ - Password (console)        │ - SessionToken               │
│ - MFA devices               │ - Expires (15 min - 1 hour)  │
│                             │                              │
│ Cannot be assumed           │ Can be assumed by:           │
│ Individual human only       │ - EC2 instances              │
│                             │ - Lambda functions           │
│ NOT for services            │ - Other AWS accounts         │
│ NOT for applications        │ - SAML users                 │
│                             │ - Web identity (Google, FB)  │
│                             │                              │
│ NEVER share credentials     │ Safely passes between        │
│ Each person = 1 user        │ Services securely            │
│                             │                              │
│ Use: Individual developers  │ Use: Production applications │
```

**Q: When would you use IAM Groups?**

**A:** Groups simplify permission management for multiple users with same role:

```python
import boto3

iam = boto3.client('iam')

# Create groups for different teams
iam.create_group(GroupName='backend-team')
iam.create_group(GroupName='devops-team')
iam.create_group(GroupName='qa-team')

# Attach policies to groups (not individual users)
iam.attach_group_policy(
    GroupName='backend-team',
    PolicyArn='arn:aws:iam::aws:policy/AmazonEC2FullAccess'
)

# Add users to groups
iam.add_user_to_group(
    GroupName='backend-team',
    UserName='alice'
)

iam.add_user_to_group(
    GroupName='backend-team',
    UserName='bob'
)

# Now alice and bob have same permissions
# If you update backend-team policies, changes apply to all members
```

**Best Practice:**
- Don't attach policies directly to users
- Create groups by team/role
- Attach policies to groups
- Easier to audit and maintain

---

### IAM Policies Deep Dive

**Q: Walk me through IAM policy evaluation logic. What happens if a user has allow but also has deny?**

**A (Advanced - FAANG Interview):**

```
IAM Policy Evaluation Flow:

User makes request to AWS service
        ↓
Step 1: Check for EXPLICIT DENY
        ├─ In identity-based policies? → DENY ❌
        ├─ In permission boundaries? → DENY ❌
        ├─ In resource-based policies? → DENY ❌
        └─ In SCPs? → DENY ❌
        
If ANY Deny found → DENY (evaluation stops)
        ↓
Step 2: No explicit deny, check ALLOW

        2a. Check Permission Boundary (if set)
            ├─ Request allowed by boundary?
            └─ NO → DENY ❌
            
        2b. Check Identity-based Policy
            ├─ Has explicit ALLOW?
            └─ NO → DENY ❌
            
        2c. Check Resource-based Policy (S3, SQS, etc.)
            ├─ Has explicit ALLOW for principal?
            └─ Continues...
            
Step 3: Check Service Control Policies (SCP)
        ├─ SCP allows the action?
        └─ NO → DENY ❌
        
        ↓
ALLOW ✓ (all checks passed)
```

**Concrete Example:**

```python
# Scenario: Developer alice tries to delete S3 bucket

# Developer IAM Policy (identity-based)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}

# S3 Bucket Policy (resource-based) - doesn't mention alice
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/bob"
      },
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}

# Permission Boundary (limits max permissions)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "*"
    }
  ]
}

# SCP in Organization
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "s3:DeleteBucket",
      "Resource": "*"
    }
  ]
}

Result: DENY ❌

Why? Multiple reasons:
1. SCP explicitly denies s3:DeleteBucket
2. Permission boundary doesn't allow DeleteBucket
3. Bucket policy doesn't grant access (only Bob can access)
```

**Q: What does "principal" mean in IAM policies?**

**A:** Principal is who can perform the action:

```json
{
  "Principal": {
    "AWS": "arn:aws:iam::123456789012:user/alice",
    "Service": "ec2.amazonaws.com",
    "Federated": "arn:aws:iam::123456789012:saml-provider/ExampleProvider"
  }
}
```

Types:
- **AWS:** IAM users, roles, root account
- **Service:** AWS services (EC2, Lambda, RDS)
- **Federated:** External identity providers
- **Wildcard (*):** Anyone (public access)

---

### Policy Examples and Anti-Patterns

**Q: Show me examples of good and bad IAM policies.**

**A:**

**❌ BAD - Over-permissive:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```
Why bad: Gives admin access to everything. Violates least privilege.

**❌ BAD - Uses wildcard for actions:**
```json
{
  "Effect": "Allow",
  "Action": "iam:*",
  "Resource": "*"
}
```
Why bad: Allows IAM modifications (users can escalate privileges).

**✅ GOOD - Least privilege for developer:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ViewEC2Instances",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeInstanceTypes"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ManageDevelopmentInstances",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "arn:aws:ec2:*:123456789012:instance/*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Environment": "development"
        }
      }
    },
    {
      "Sid": "DenyDangerousActions",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "ec2:DeleteVolume",
        "ec2:ModifyInstanceAttribute"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowS3ReadOnlyForAppData",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::app-data-bucket",
        "arn:aws:s3:::app-data-bucket/*"
      ]
    }
  ]
}
```

Why good:
1. Specific actions listed (not wildcards)
2. Resource ARNs specified (not *)
3. Conditions limit to development environment
4. Explicit denies prevent dangerous ops
5. Separate statements for clarity

---

## Advanced IAM Patterns

### Cross-Account Access

**Q: How would you set up cross-account access for disaster recovery? Account A is production, Account B is DR.**

**A (Intermediate):**

**Scenario:** DR account needs to assume role to provision resources in Prod account.

**Step 1: Production Account (Account A) - Create Role**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT-B:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-dr-external-id-xyz123"
        }
      }
    }
  ]
}
```

**Important:** ExternalId prevents the "confused deputy problem":
- Without it: if DR account is compromised, attacker could assume prod role
- With it: attacker needs to know external ID too

**Step 2: Production Account - Attach Permissions**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestoreEC2Instances",
      "Effect": "Allow",
      "Action": [
        "ec2:RunInstances",
        "ec2:CreateVolume",
        "ec2:CreateSecurityGroup",
        "ec2:ModifySecurityGroupRules"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RestoreRDSDatabase",
      "Effect": "Allow",
      "Action": [
        "rds:RestoreDBInstanceFromDBSnapshot",
        "rds:ModifyDBInstance",
        "rds:AddTagsToResource"
      ],
      "Resource": "arn:aws:rds:*:ACCOUNT-A:*"
    },
    {
      "Sid": "ReadProductionSnapshots",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeSnapshots",
        "rds:DescribeDBSnapshots"
      ],
      "Resource": "*"
    }
  ]
}
```

**Step 3: DR Account (Account B) - Assume the Role**

```python
import boto3

sts = boto3.client('sts')

# Assume role from Prod account
response = sts.assume_role(
    RoleArn='arn:aws:iam::ACCOUNT-A:role/DR-Admin-Role',
    RoleSessionName='dr-restore-session',
    ExternalId='unique-dr-external-id-xyz123',
    DurationSeconds=3600
)

# Get temporary credentials
credentials = response['Credentials']
access_key = credentials['AccessKeyId']
secret_key = credentials['SecretAccessKey']
session_token = credentials['SessionToken']

# Use credentials to restore resources in Prod account
ec2_prod = boto3.client(
    'ec2',
    region_name='us-east-1',
    aws_access_key_id=access_key,
    aws_secret_access_key=secret_key,
    aws_session_token=session_token
)

# Now can restore instances
instances = ec2_prod.run_instances(
    ImageId='ami-12345678',
    MinCount=1,
    MaxCount=1,
    InstanceType='t3.medium'
)
```

**Q: What if we need MFA for critical operations?**

**A:** Add MFA requirement to role assumption:

```python
sts = boto3.client('sts')

try:
    response = sts.assume_role(
        RoleArn='arn:aws:iam::ACCOUNT-A:role/DR-Admin-Role',
        RoleSessionName='dr-restore-session',
        ExternalId='unique-dr-external-id-xyz123',
        SerialNumber='arn:aws:iam::ACCOUNT-B:mfa/dr-automation',
        TokenCode='123456'  # 6-digit code from MFA device
    )
except sts.exceptions.AccessDenied as e:
    print(f"MFA required or invalid: {e}")
```

---

### Permission Boundaries

**Q: A manager needs to create policies for their team but shouldn't grant themselves admin access. How?**

**A:** Use permission boundaries:

```python
import boto3

iam = boto3.client('iam')

# Define Permission Boundary
permission_boundary = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:*",
                "s3:*",
                "rds:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Deny",
            "Action": [
                "iam:*",
                "organizations:*",
                "account:*",
                "billing:*"
            ],
            "Resource": "*"
        }
    ]
}

# Create permission boundary policy
iam.create_policy(
    PolicyName='manager-permission-boundary',
    PolicyDocument=json.dumps(permission_boundary)
)

# Create manager role
iam.create_role(
    RoleName='team-manager',
    AssumeRolePolicyDocument=json.dumps({
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {"AWS": "arn:aws:iam::123456789012:user/alice"},
                "Action": "sts:AssumeRole"
            }
        ]
    })
)

# Attach permission boundary to manager role
iam.put_role_permissions_boundary(
    RoleName='team-manager',
    PermissionsBoundary='arn:aws:iam::123456789012:policy/manager-permission-boundary'
)

# Attach full policies to role (within boundary limit)
iam.attach_role_policy(
    RoleName='team-manager',
    PolicyArn='arn:aws:iam::aws:policy/PowerUserAccess'
)

# Now even if alice attaches admin policy to herself,
# permission boundary prevents IAM and billing access
```

**How it works:**
```
Permission Boundary (LIMIT):
  Allow: EC2, S3, RDS
  Deny: IAM, Billing, Organizations

Identity Policy (GRANT):
  Allow: PowerUserAccess (everything)

Result: Intersection
  ✓ EC2, S3, RDS access
  ✗ IAM, Billing, Organizations (boundary blocks)
```

---

### Service Control Policies (SCPs)

**Q: How would you use SCPs to enforce compliance across 100 AWS accounts?**

**A (Advanced):**

**Scenario:** SaaS company with 100 customer accounts. Need to:
1. Ensure all data is in US regions only
2. Require encryption for all data
3. Prevent accidental deletions
4. Audit all API calls

```python
import boto3
import json

organizations = boto3.client('organizations')

# SCP 1: Restrict regions to US only
scp_restrict_regions = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Action": "*",
            "Resource": "*",
            "Condition": {
                "StringNotEquals": {
                    "aws:RequestedRegion": [
                        "us-east-1",
                        "us-west-2",
                        "us-gov-west-1"
                    ]
                }
            }
        }
    ]
}

# SCP 2: Require S3 encryption
scp_require_s3_encryption = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Action": [
                "s3:PutObject",
                "s3:PutBucketPolicy"
            ],
            "Resource": "*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "aws:kms"
                }
            }
        }
    ]
}

# SCP 3: Require CloudTrail
scp_require_cloudtrail = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Action": [
                "cloudtrail:StopLogging",
                "cloudtrail:DeleteTrail",
                "cloudtrail:PutEventSelectors"
            ],
            "Resource": "*"
        }
    ]
}

# SCP 4: Protect production resources
scp_protect_prod = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Action": [
                "ec2:TerminateInstances",
                "rds:DeleteDBInstance",
                "s3:DeleteBucket"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:PrincipalOrgID": "o-xxxxx"
                }
            }
        }
    ]
}

# Apply SCPs to OU
organization_unit = "ou-customer-prod"

for scp_name, scp_policy in [
    ("restrict-regions", scp_restrict_regions),
    ("require-s3-encryption", scp_require_s3_encryption),
    ("require-cloudtrail", scp_require_cloudtrail),
    ("protect-prod", scp_protect_prod)
]:
    response = organizations.create_policy(
        Content=json.dumps(scp_policy),
        Description=scp_name,
        Name=scp_name,
        Type='SERVICE_CONTROL_POLICY'
    )
    
    policy_id = response['Policy']['PolicySummary']['Id']
    
    organizations.attach_policy(
        PolicyId=policy_id,
        TargetId=organization_unit
    )
```

**Key Points:**
- SCPs are inherited down the OU hierarchy
- Root account not affected by SCPs
- SCPs are "permission filter" (deny only)
- Combined with identity policies for final permissions
- Evaluated after all other policy types

---

## Secrets Management

### AWS Secrets Manager

**Q: Explain automatic secret rotation and what happens if rotation fails.**

**A (Intermediate):**

```python
import boto3
import json
import psycopg2
import random
import string

secrets_client = boto3.client('secretsmanager')

# Setup automatic rotation for RDS password
response = secrets_client.create_secret(
    Name='prod/mysql/admin-password',
    SecretString=json.dumps({
        'username': 'admin',
        'password': 'Initial@Password123',
        'host': 'mydb.c9akciq32.us-east-1.rds.amazonaws.com',
        'port': 3306,
        'dbname': 'mydb'
    })
)

# Enable automatic rotation
secrets_client.rotate_secret(
    SecretId='prod/mysql/admin-password',
    RotationRules={
        'AutomaticallyAfterDays': 30,
        'Duration': 3600,  # 1 hour to complete rotation
        'ScheduleExpression': 'rate(30 days)'
    },
    RotationLambdaARN='arn:aws:lambda:us-east-1:123456789012:function/SecretsManager-Rotation'
)

# Lambda function that performs rotation
def rotate_secret_lambda(event, context):
    service_client = boto3.client('secretsmanager')
    
    # Event contains:
    # - SecretId: which secret to rotate
    # - ClientRequestToken: unique rotation ID
    # - Step: 'create', 'set', 'test', 'finish'
    
    secret_id = event['SecretId']
    client_request_token = event['ClientRequestToken']
    step = event['Step']
    
    # Get current secret
    current_secret = service_client.get_secret_value(SecretId=secret_id)
    secret_dict = json.loads(current_secret['SecretString'])
    
    if step == 'create':
        # Generate new password
        new_password = ''.join(random.choices(
            string.ascii_letters + string.digits + '!@#$%', 
            k=32
        ))
        
        # Store as pending version
        service_client.put_secret_value(
            SecretId=secret_id,
            ClientRequestToken=client_request_token,
            SecretString=json.dumps({
                **secret_dict,
                'password': new_password
            }),
            VersionStages=['AWSPENDING']
        )
        
    elif step == 'set':
        # Update password in actual database
        try:
            conn = psycopg2.connect(
                host=secret_dict['host'],
                user=secret_dict['username'],
                password=secret_dict['password'],
                database=secret_dict['dbname']
            )
            cursor = conn.cursor()
            
            pending_secret = service_client.get_secret_value(
                SecretId=secret_id,
                VersionId=client_request_token,
                VersionStage='AWSPENDING'
            )
            pending_dict = json.loads(pending_secret['SecretString'])
            
            # Update password in database
            cursor.execute(
                f"ALTER USER {secret_dict['username']} WITH PASSWORD %s",
                (pending_dict['password'],)
            )
            conn.commit()
            cursor.close()
            conn.close()
            
        except Exception as e:
            print(f"Failed to update password: {e}")
            raise
            
    elif step == 'test':
        # Test new password works
        try:
            pending_secret = service_client.get_secret_value(
                SecretId=secret_id,
                VersionId=client_request_token,
                VersionStage='AWSPENDING'
            )
            test_dict = json.loads(pending_secret['SecretString'])
            
            conn = psycopg2.connect(
                host=test_dict['host'],
                user=test_dict['username'],
                password=test_dict['password'],
                database=test_dict['dbname']
            )
            conn.close()
            
        except Exception as e:
            print(f"Failed to verify new password: {e}")
            raise
            
    elif step == 'finish':
        # Finalize rotation
        service_client.update_secret_version_stage(
            SecretId=secret_id,
            VersionStage='AWSCURRENT',
            MoveToVersionId=client_request_token,
            RemoveFromVersionId=current_secret['VersionId']
        )
    
    return {"statusCode": 200}
```

**What if rotation fails?**

```
Normal Rotation Timeline:
[Create] → [Set] → [Test] → [Finish]

If [Set] fails (DB connection timeout):
├─ New version stays in AWSPENDING state
├─ Current version (AWSCURRENT) remains unchanged
├─ Applications continue using old password
├─ Rotation retried in 24 hours (with exponential backoff)
└─ CloudWatch alarm triggers for failure

Best Practices for Rotation:
1. Have detailed error logs in Lambda
2. Set up SNS notifications for failures
3. Manual remediation playbook
4. Test rotation before enabling auto-rotation
5. Schedule for low-traffic windows
```

### AWS Systems Manager Parameter Store

**Q: When would you use Parameter Store vs Secrets Manager?**

**A:**

| Aspect | Parameter Store | Secrets Manager |
|--------|-----------------|-----------------|
| **Cost** | Free (standard) / $0.04 per parameter (advanced) | $0.40 per secret/month |
| **Automatic Rotation** | No | Yes |
| **CloudFormation Support** | Yes | Limited |
| **Use Case** | Configuration values, feature flags | Sensitive secrets, credentials |
| **Encryption** | KMS supported | Always encrypted |
| **Audit Trail** | CloudTrail | CloudTrail + detailed |

**When to use Parameter Store:**
- Database host (not sensitive)
- Feature flags
- Configuration that changes
- Non-sensitive environment variables

**When to use Secrets Manager:**
- Passwords
- API keys
- OAuth tokens
- Connection strings with credentials

**Example:**
```python
import boto3

ssm = boto3.client('ssm')

# Store non-sensitive configuration
ssm.put_parameter(
    Name='/myapp/database/host',
    Value='mydb.c9akciq32.us-east-1.rds.amazonaws.com',
    Type='String',
    Tags=[
        {'Key': 'Environment', 'Value': 'production'},
        {'Key': 'Component', 'Value': 'database'}
    ]
)

# Store sensitive configuration (encrypted)
ssm.put_parameter(
    Name='/myapp/api/key',
    Value='sk_live_abc123xyz789',
    Type='SecureString',  # Automatically encrypted with KMS
    KeyId='arn:aws:kms:us-east-1:123456789012:key/12345678'
)

# Retrieve with hierarchy
params = ssm.get_parameters_by_path(
    Path='/myapp/database/',
    Recursive=True,
    WithDecryption=True
)

for param in params['Parameters']:
    print(f"{param['Name']}: {param['Value']}")
```

---

## Encryption

### KMS Deep Dive

**Q: Explain how KMS works and the difference between customer-managed and AWS-managed keys.**

**A (Advanced):**

**AWS-Managed Keys:**
- Created and managed by AWS
- Automatic rotation every year
- No key management overhead
- Free to use
- Can't control key policy
- Use: Default encryption for most services

**Customer-Managed Keys (CMK):**
- You create and manage
- Manual or automatic rotation (annually)
- Full control over key policies
- Cost: ~$1/month per key
- Audit trail in CloudTrail
- Use: Highly sensitive data, compliance requirements

```python
import boto3
import json

kms_client = boto3.client('kms')

# Create customer-managed key
response = kms_client.create_key(
    Description='Master key for production data',
    KeyUsage='ENCRYPT_DECRYPT',
    Origin='AWS_KMS',
    MultiRegion=False
)

key_id = response['KeyMetadata']['KeyId']
print(f"Created key: {key_id}")

# Create alias for easier reference
kms_client.create_alias(
    AliasName='alias/prod-master-key',
    TargetKeyId=key_id
)

# Get key metadata
response = kms_client.describe_key(KeyId=key_id)
key_metadata = response['KeyMetadata']
print(f"Key Status: {key_metadata['KeyState']}")
print(f"Creation Date: {key_metadata['CreationDate']}")

# Enable automatic rotation
kms_client.enable_key_rotation(KeyId=key_id)

# Check rotation status
response = kms_client.get_key_rotation_status(KeyId=key_id)
print(f"Rotation Enabled: {response['KeyRotationEnabled']}")
```

**Encryption Flow:**

```
┌─────────────────────────────────────┐
│  Application wants to encrypt data  │
└─────────────────────────────────────┘
        ↓
    KMS GenerateDataKey
        ↓
KMS returns:
├─ Plaintext Data Key (256-bit)
└─ Encrypted Data Key
        ↓
Application:
├─ Encrypts data with plaintext key
├─ Stores: encrypted_data_key + encrypted_data
└─ Clears plaintext key from memory
        ↓
Later, to decrypt:
├─ Call KMS Decrypt with encrypted_data_key
├─ Get plaintext key
├─ Decrypt data
└─ Clear plaintext key
```

**Envelope Encryption Example:**

```python
from cryptography.fernet import Fernet
import base64

# Encrypt 1GB file with envelope encryption
def encrypt_large_file(kms_client, file_path, output_path, kms_key_id):
    # Step 1: Generate data key from KMS
    response = kms_client.generate_data_key(
        KeyId=kms_key_id,
        KeySpec='AES_256'
    )
    
    plaintext_key = response['Plaintext']
    encrypted_key = response['EncryptedDataKey']
    
    # Step 2: Encrypt file with plaintext key
    cipher = Fernet(base64.b64encode(plaintext_key[:32]))
    
    with open(file_path, 'rb') as f_in:
        with open(output_path, 'wb') as f_out:
            # Write encrypted key header
            f_out.write(b'ENC:')
            f_out.write(len(encrypted_key).to_bytes(4, 'big'))
            f_out.write(encrypted_key)
            
            # Encrypt and write file
            while True:
                chunk = f_in.read(1024 * 1024)  # 1MB chunks
                if not chunk:
                    break
                encrypted_chunk = cipher.encrypt(chunk)
                f_out.write(encrypted_chunk)
    
    return output_path

# Decrypt large file
def decrypt_large_file(kms_client, encrypted_path, output_path):
    with open(encrypted_path, 'rb') as f_in:
        # Read encrypted key
        header = f_in.read(4)
        key_length = int.from_bytes(f_in.read(4), 'big')
        encrypted_key = f_in.read(key_length)
        
        # Decrypt data key
        response = kms_client.decrypt(CiphertextBlob=encrypted_key)
        plaintext_key = response['Plaintext']
        
        # Decrypt file
        cipher = Fernet(base64.b64encode(plaintext_key[:32]))
        
        with open(output_path, 'wb') as f_out:
            while True:
                chunk = f_in.read(1024 * 1024)
                if not chunk:
                    break
                decrypted_chunk = cipher.decrypt(chunk)
                f_out.write(decrypted_chunk)
```

**Benefits of Envelope Encryption:**
- No need to send data to KMS for each 4KB chunk
- Avoid KMS throttling (1000-4000 requests/second)
- Much faster for large datasets
- Cost-effective

---

## Network Security

### Security Groups vs NACLs

**Q: Explain the difference between Security Groups and NACLs. When would you use each?**

**A (Intermediate):**

```
┌──────────────────────────────────────────────────────────────┐
│                     VPC                                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    Subnet                              │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │  NACL (Network ACL) - Stateless firewall       │  │ │
│  │  │  Processes rules in order (1-32766)            │  │ │
│  │  │  ALL traffic passes through              │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  │                     ↓                                  │ │
│  │  ┌────────────────────────────────────────────────┐   │ │
│  │  │            EC2 Instance                        │   │ │
│  │  │  ┌──────────────────────────────────────────┐  │   │ │
│  │  │  │ Security Group - Stateful firewall      │  │   │ │
│  │  │  │ No order (all rules evaluated)          │  │   │ │
│  │  │  │ Only ALLOW rules (implicit deny)        │  │   │ │
│  │  │  └──────────────────────────────────────────┘  │   │ │
│  │  │            Application                         │   │ │
│  │  └────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
```

| Aspect | Security Group | NACL |
|--------|---|---|
| **Scope** | Instance | Subnet |
| **Stateful** | Yes | No |
| **Rule Order** | No order | Numbered (1-32766) |
| **Effect Types** | Allow only | Allow & Deny |
| **Default** | Deny inbound, allow outbound | Allow all |
| **When to use** | Primary security layer | Explicit denies, subnet-level rules |

**Stateful Example:**
```
Security Group:
Outbound: EC2 → 8.8.8.8:53 (DNS)
           ↓ (SG tracks this connection)
Inbound:  8.8.8.8:53 → EC2 (automatically allowed)
          No rule needed ✓

NACL:
Outbound: EC2 → 8.8.8.8:53 (needs explicit ALLOW rule)
Inbound:  8.8.8.8:53 → EC2 (needs separate ALLOW rule)
          Return traffic NOT automatically tracked
```

**When to use each:**

**Security Groups:**
- Protect individual instances
- Application-specific rules
- Simpler management (no ordering)
- Default deny inbound

**NACLs:**
- Explicit denies (Security Groups can't deny)
- Subnet-wide rules
- DDoS mitigation at subnet edge
- Port scanning protection

**Example - Block port scanning:**
```python
import boto3

ec2 = boto3.client('ec2')

# Create NACL
nacl_response = ec2.create_network_acl(VpcId='vpc-12345678')
nacl_id = nacl_response['NetworkAcl']['NetworkAclId']

# Block port scanners (nmap, etc.)
ec2.create_network_acl_entry(
    NetworkAclId=nacl_id,
    RuleNumber=100,
    Protocol='tcp',
    RuleAction='deny',
    CidrBlock='0.0.0.0/0',
    PortRange={'From': 0, 'To': 1024},
    EgressIn=False  # Inbound rule
)

# Allow SSH from office
ec2.create_network_acl_entry(
    NetworkAclId=nacl_id,
    RuleNumber=110,
    Protocol='tcp',
    RuleAction='allow',
    CidrBlock='203.0.113.0/24',  # Office CIDR
    PortRange={'From': 22, 'To': 22},
    EgressIn=False
)

# Allow HTTP/HTTPS
ec2.create_network_acl_entry(
    NetworkAclId=nacl_id,
    RuleNumber=120,
    Protocol='tcp',
    RuleAction='allow',
    CidrBlock='0.0.0.0/0',
    PortRange={'From': 80, 'To': 80},
    EgressIn=False
)

ec2.create_network_acl_entry(
    NetworkAclId=nacl_id,
    RuleNumber=130,
    Protocol='tcp',
    RuleAction='allow',
    CidrBlock='0.0.0.0/0',
    PortRange={'From': 443, 'To': 443},
    EgressIn=False
)

# Allow ephemeral ports for return traffic
ec2.create_network_acl_entry(
    NetworkAclId=nacl_id,
    RuleNumber=140,
    Protocol='tcp',
    RuleAction='allow',
    CidrBlock='0.0.0.0/0',
    PortRange={'From': 1024, 'To': 65535},
    EgressIn=False
)

# Default deny (32767 is last rule)
```

---

## Detective Controls

### GuardDuty

**Q: How would you respond to a GuardDuty finding of EC2 involvement in a botnet?**

**A (Advanced - incident response):**

```python
import boto3
import json

guardduty = boto3.client('guardduty3')
ec2 = boto3.client('ec2')
sns = boto3.client('sns')

# Finding example
finding = {
    "type": "CryptoCurrency:EC2/BitcoinTool.B",
    "severity": 8.0,
    "description": "EC2 instance performing Bitcoin mining",
    "resource": {
        "instanceDetails": {
            "instanceId": "i-1234567890abcdef0",
            "networkInterfaces": [{
                "ipv4Addresses": ["10.0.0.50"]
            }]
        }
    }
}

# RESPONSE PROCEDURE:
# Step 1: Isolate instance
print(f"[1] Isolating instance {finding['resource']['instanceDetails']['instanceId']}")

instance_id = finding['resource']['instanceDetails']['instanceId']

# Get security group
instance = ec2.describe_instances(InstanceIds=[instance_id])['Reservations'][0]['Instances'][0]
sg_id = instance['SecurityGroups'][0]['GroupId']

# Create isolation security group
isolation_sg = ec2.create_security_group(
    GroupName='isolation-sg',
    Description='Isolation for compromised instances',
    VpcId=instance['VpcId']
)
isolation_sg_id = isolation_sg['GroupId']

# No inbound/outbound allowed
ec2.revoke_security_group_egress(
    GroupId=isolation_sg_id,
    IpPermissions=[{
        'IpProtocol': '-1',
        'CidrIp': '0.0.0.0/0'
    }]
)

# Apply isolation SG
ec2.modify_instance_attribute(
    InstanceId=instance_id,
    Groups=[isolation_sg_id]
)

# Step 2: Snapshot for forensics
print(f"[2] Creating snapshot for forensics")
volumes = [bd['Ebs']['VolumeId'] for bd in instance['BlockDeviceMappings']]
for vol_id in volumes:
    snapshot = ec2.create_snapshot(
        VolumeId=vol_id,
        Description=f'Forensic snapshot of {instance_id}'
    )
    print(f"Snapshot created: {snapshot['SnapshotId']}")

# Step 3: Collect logs
print(f"[3] Collecting CloudTrail logs")
ct = boto3.client('cloudtrail')
events = ct.lookup_events(
    LookupAttributes=[
        {
            'AttributeKey': 'ResourceName',
            'AttributeValue': instance_id
        }
    ],
    MaxResults=50
)

# Step 4: Alert team
sns.publish(
    TopicArn='arn:aws:sns:us-east-1:123456789012:security-incidents',
    Subject=f'CRITICAL: Security Incident - Instance {instance_id}',
    Message=f"""
GuardDuty Alert - Immediate Action Required

Finding Type: {finding['type']}
Severity: {finding['severity']}/10
Instance: {instance_id}
IP Address: {instance['PrivateIpAddress']}

Actions Taken:
1. Instance isolated (security group restrictions applied)
2. Forensic snapshots created
3. CloudTrail events collected

Next Steps:
1. Investigate CloudTrail logs
2. Review instance metadata/userdata
3. Check for lateral movement
4. Determine breach scope
5. Remediation plan
"""
)

# Step 5: Terminate after forensics complete (after 24-72 hours)
print("[4] Instance will be terminated after forensic analysis")
```

**Complete Response Playbook:**

```
GuardDuty Finding ↓
[ALERT] Get finding details
↓
[TRIAGE] Assess severity
├─ Tier 1 (Critical): Immediate isolation
├─ Tier 2 (High): Isolate + investigate
└─ Tier 3 (Medium): Investigate + monitor
↓
[CONTAINMENT] Isolate instance
├─ Apply restrictive security group
├─ Disable SSM Session Manager
└─ Prepare for analysis
↓
[INVESTIGATION] Collect evidence
├─ EBS snapshots
├─ CloudTrail logs
├─ VPC Flow Logs
├─ CloudWatch logs
└─ SSM Session Manager history
↓
[ANALYSIS] Determine root cause
├─ Credential compromise?
├─ Misconfiguration?
├─ Vulnerable application?
└─ Supply chain?
↓
[REMEDIATION] Fix root cause
├─ Rotate credentials
├─ Patch vulnerabilities
├─ Update security groups
└─ Enhanced monitoring
↓
[ERADICATION] Remove threat
├─ Terminate compromised instance
├─ Re-image other instances
└─ Restore from known-good backup
↓
[RECOVERY] Verify normal ops
└─ Monitor metrics and logs
```

---

## Interview Questions

### Q1: Walk me through creating a least-privilege policy for a developer

**A:** [See IAM Policies examples above]

### Q2: How would you audit IAM permissions across 500 accounts?

**A:**
```python
import boto3
from concurrent.futures import ThreadPoolExecutor

# Use AWS Config aggregator to centrally view compliance
config = boto3.client('config')

# Create aggregator
config.put_configuration_aggregator(
    ConfigurationAggregatorName='organization-aggregator',
    AccountAggregationSources=[
        {
            'AllAwsRegions': True,
            'AccountIds': ['111111111111', '222222222222', ...]  # 500 accounts
        }
    ]
)

# Run audit across all accounts
# Use AWS Config Rules to check IAM compliance
rules = [
    'iam-policy-no-statements-with-admin-access',
    'iam-user-mfa-enabled',
    'iam-password-policy',
    'root-account-mfa-enabled',
    'access-keys-rotated'
]

# Get compliance data aggregated
response = config.get_aggregate_compliance_details_by_config_rule(
    ConfigurationAggregatorName='organization-aggregator',
    ConfigRuleName='iam-policy-no-statements-with-admin-access'
)

# Non-compliant accounts
for detail in response['AggregateEvaluationResults']:
    if detail['EvaluationResultIdentifier']['EvaluationResultQualifier']['ComplianceType'] == 'NON_COMPLIANT':
        print(f"Non-compliant account: {detail['AwsAccountId']}")
```

### Q3: A developer accidentally deleted a production RDS password. How do you recover?

**A:**
- Rotate the password using Secrets Manager
- If no rotation Lambda: manually update in AWS Systems Manager Parameter Store
- Update application connection strings
- Restart applications
- Monitor for connection errors

---

