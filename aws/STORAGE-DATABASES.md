# AWS Storage and Databases - Comprehensive Interview Guide

> Deep dive into S3, EBS, EFS, FSx, RDS, Aurora, DynamoDB, ElastiCache, Redshift with production scenarios

**Estimated Reading Time:** 150 minutes | **Coverage:** 180+ interview questions

---

## Table of Contents

- [S3 Deep Dive](#s3-deep-dive)
- [EBS and EFS](#ebs-and-efs)
- [FSx](#fsx)
- [RDS and Aurora](#rds-and-aurora)
- [DynamoDB](#dynamodb)
- [ElastiCache](#elasticache)
- [Redshift](#redshift)
- [Database Selection Guide](#database-selection-guide)
- [Interview Questions](#interview-questions)

---

## S3 Deep Dive

### S3 Architecture and Internals

**Q: Explain how S3 achieves 11 nines (99.999999999%) durability.**

**A (Advanced - FAANG Interview):**

```
S3 Durability Model:

1. Data stored across multiple AZs
2. Each object replicated 3x minimum
3. Different storage devices and failure domains
4. Continuous monitoring and healing

Durability Calculation:
- Single object failure rate: 0.0000001% per year
- 11 nines = 99.999999999%
- Annual probability of losing object: 1 in 100,000,000,000

Comparison:
S3: 11 nines (99.999999999%) = 1 object lost per 100 billion
RDS: 9 nines (99.9999999%) = significant difference
DynamoDB: 11 nines (like S3)

Real-world example:
If you store 1 billion objects:
- S3: Expected to lose 0.00001 objects/year (essentially never)
- RDS: Expected to lose 0.1 objects/year
```

### S3 Request Rate Performance

**Q: What are S3's request rate limits and how do you optimize for high throughput?**

**A (Advanced):**

```
Request Rate Limits (per partition key):

Before 2018: Hard limit
├─ 100 PUT/COPY/POST/DELETE per second
├─ 300 GET/HEAD per second
└─ Workaround: Use random prefix (parallelization)

After 2018: Automatic scaling
├─ Scales to very high request rates automatically
├─ No fixed limits
└─ Spikes handled transparently

Optimization Strategies:

1. Parallel uploads (multipart)
S3.upload_file() uses multipart automatically
└─ Default: 5 parts, 5 threads
└─ Can configure: mb_chunk_size, max_concurrency

2. Request routing
Good:
└─ s3://mybucket/object1
└─ s3://mybucket/object2

Bad (sequential same key):
└─ s3://mybucket/log
└─ s3://mybucket/log (append operation)
└─ Serialized to single partition

3. CloudFront caching
- First access: 500ms (S3 latency)
- Cached accesses: 10-20ms (from edge)

Benchmark:
- Single connection: 50-100 Mbps
- 10 parallel connections: 500-1000 Mbps
- 100 parallel connections: Near S3 theoretical max
```

### S3 Storage Classes and Lifecycle

**Q: Design S3 storage strategy for data retention and cost optimization.**

**A (Production Scenario):**

```
Scenario: Application generates 1TB logs daily, 3-year retention

Cost Analysis:
Naive approach (keep everything in S3 Standard):
- Storage: 1TB/day × 365 days × 3 years = 1095TB
- Cost: 1095TB × $0.023/month = $25,185/month = $302k/year

Optimized approach (tiered storage):
```

```python
import boto3
import json

s3 = boto3.client('s3')

# Lifecycle policy
lifecycle_config = {
    'Rules': [
        {
            'Id': 'log-archival-policy',
            'Filter': {'Prefix': 'logs/'},
            'Status': 'Enabled',
            'Transitions': [
                {
                    'Days': 30,
                    'StorageClass': 'STANDARD_IA'  # After 30 days
                },
                {
                    'Days': 90,
                    'StorageClass': 'GLACIER_IR'  # After 90 days
                },
                {
                    'Days': 180,
                    'StorageClass': 'DEEP_ARCHIVE'  # After 6 months
                }
            ],
            'Expiration': {
                'Days': 1095  # Delete after 3 years
            },
            'NoncurrentVersionTransitions': [
                {
                    'NoncurrentDays': 1,
                    'StorageClass': 'GLACIER'
                }
            ],
            'NoncurrentVersionExpiration': {
                'NoncurrentDays': 30
            }
        }
    ]
}

s3.put_bucket_lifecycle_configuration(
    Bucket='my-logs-bucket',
    LifecycleConfiguration=lifecycle_config
)

# Result cost analysis:
# - First 30 days (Standard): 1TB × $0.023 = $23/month
# - Days 30-90 (Standard-IA): 2TB × $0.0125 = $25/month
# - Days 90-180 (Glacier): 3TB × $0.004 = $12/month
# - Days 180-1095 (Deep Archive): 915TB × $0.00099 = $906/month

# Average: ~$240/month = $2,880/year (vs $302k naive)
# Savings: 99% reduction!
```

**Storage Class Comparison:**

```
                  Standard  IA      Glacier  Deep Archive
────────────────────────────────────────────────────────
Per GB/month     $0.023    $0.0125 $0.004   $0.00099
Retrieval time   Instant   Minutes Hours    12+ hours
Min. duration    N/A       30 days 90 days  180 days
Use case         Active    Archive Old data Compliance
```

### S3 Security and Encryption

**Q: Design S3 bucket security for a SaaS platform storing customer data.**

**A (Advanced):**

```python
import boto3
import json

s3 = boto3.client('s3')
kms = boto3.client('kms')

# 1. Create KMS key for encryption
kms_key = kms.create_key(
    Description='S3 encryption key for customer data'
)
key_id = kms_key['KeyMetadata']['KeyId']

# 2. Create S3 bucket with security settings
s3.create_bucket(
    Bucket='customer-data-secure',
    CreateBucketConfiguration={'LocationConstraint': 'us-east-1'}
)

# 3. Enable versioning (protection against accidental deletion)
s3.put_bucket_versioning(
    Bucket='customer-data-secure',
    VersioningConfiguration={'Status': 'Enabled'}
)

# 4. Block all public access
s3.put_public_access_block(
    Bucket='customer-data-secure',
    PublicAccessBlockConfiguration={
        'BlockPublicAcls': True,
        'IgnorePublicAcls': True,
        'BlockPublicPolicy': True,
        'RestrictPublicBuckets': True
    }
)

# 5. Enable encryption by default
s3.put_bucket_encryption(
    Bucket='customer-data-secure',
    ServerSideEncryptionConfiguration={
        'Rules': [
            {
                'ApplyServerSideEncryptionByDefault': {
                    'SSEAlgorithm': 'aws:kms',
                    'KMSMasterKeyID': key_id
                },
                'BucketKeyEnabled': True  # Reduced KMS costs
            }
        ]
    }
)

# 6. Enforce SSL/TLS only
bucket_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyUnencryptedObjectUploads",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::customer-data-secure/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "aws:kms"
                }
            }
        },
        {
            "Sid": "DenyInsecureTransport",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::customer-data-secure",
                "arn:aws:s3:::customer-data-secure/*"
            ],
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        }
    ]
}

s3.put_bucket_policy(
    Bucket='customer-data-secure',
    Policy=json.dumps(bucket_policy)
)

# 7. Enable MFA Delete (extra protection)
s3.put_bucket_versioning(
    Bucket='customer-data-secure',
    VersioningConfiguration={
        'Status': 'Enabled',
        'MFADelete': 'Enabled'  # Requires MFA for permanent delete
    },
    MFA='arn:aws:iam::123456789012:mfa/root-account'
)

# 8. Enable CloudTrail logging
s3.put_bucket_logging(
    Bucket='customer-data-secure',
    BucketLoggingStatus={
        'LoggingEnabled': {
            'TargetBucket': 'access-logs-bucket',
            'TargetPrefix': 'customer-data/'
        }
    }
)

# 9. Enable Object Lock (immutability)
# Must be enabled at bucket creation, not after
# s3.create_bucket(..., ObjectLockEnabledForBucket=True)

# 10. Server-side access control
s3.put_object_acl(
    Bucket='customer-data-secure',
    Key='data.txt',
    ACL='private'  # Not 'public-read' or 'public-read-write'
)
```

**Security Checklist:**

```
Encryption:
☑ Server-side encryption (KMS)
☑ Bucket key enabled (cost optimization)
☑ Enforce encryption in bucket policy

Access Control:
☑ Block all public access
☑ Bucket policy restricts to IAM users
☑ IAM roles for applications (not access keys)

Transport:
☑ Enforce HTTPS only (deny HTTP)
☑ VPC endpoint for private access
☑ CloudFront for distribution

Durability & Resilience:
☑ Versioning enabled
☑ MFA delete for critical data
☑ Cross-region replication
☑ Backup strategy

Monitoring:
☑ CloudTrail logging enabled
☑ Access logs enabled
☑ CloudWatch alarms for unusual activity
```

---

## RDS and Aurora

### RDS Multi-AZ Deep Dive

**Q: Explain RDS Multi-AZ failover. How long does it take? What's affected?**

**A (Production Scenario):**

```
Multi-AZ Setup:

Primary DB (AZ-a)
    │
    ├─ Synchronous replication
    │
    ▼
Standby DB (AZ-b) - NO TRAFFIC, STANDBY MODE

When Primary Fails:

1. Health check fails (within 60 seconds)
2. Route53 or AWS API updates endpoint DNS
3. Application reconnects to new endpoint (standby)
4. Connection pool refreshes
5. Traffic flows to AZ-b

Failover Time: 60-120 seconds
Downtime felt: 30-60 seconds (connection timeout + reconnect)

What stays the same:
✓ Endpoint URL (DNS updates automatically)
✓ Database identifier
✓ Database credentials

What's affected:
✗ Existing connections drop (must reconnect)
✗ In-flight transactions roll back
✗ Temporary traffic spike (all clients reconnect at once)
```

**Failover Test Code:**

```python
import boto3
import time

rds = boto3.client('rds')

# Simulate failure (safe for testing!)
print("Initiating RDS failover...")
rds.reboot_db_instance(
    DBInstanceIdentifier='production-db',
    ForceFailover=True  # Fail over to standby
)

# Monitor status
start_time = time.time()
while True:
    db = rds.describe_db_instances(
        DBInstanceIdentifier='production-db'
    )['DBInstances'][0]
    
    status = db['DBInstanceStatus']
    print(f"Status: {status} ({time.time() - start_time:.0f}s)")
    
    if status == 'available':
        print(f"Failover complete in {time.time() - start_time:.0f} seconds")
        break
    
    time.sleep(10)
```

### Aurora Global Database

**Q: Design Aurora Global Database for a multi-region SaaS platform.**

**A (Advanced Architecture):**

```
Architecture:

Region: us-east-1 (Primary)
┌─────────────────────────────────────┐
│   Aurora Primary Cluster            │
│   ├─ Writer: writer.us-east.rds     │
│   ├─ Reader: reader.us-east.rds     │
│   └─ 3 replicas (high availability) │
└─────────────────────────────────────┘
            │
            │ (Asynchronous replication)
            │ <1ms typical latency
            ▼
Region: eu-west-1 (Secondary - Read-Only)
┌─────────────────────────────────────┐
│   Aurora Secondary Cluster          │
│   ├─ NO writer endpoint             │
│   ├─ reader.eu-west.rds             │
│   └─ Read-only replicas             │
│   └─ Can promote if primary fails   │
└─────────────────────────────────────┘

Disaster Recovery:
If us-east-1 fails:
1. Detect primary unavailable
2. Promote eu-west-1 to standalone
3. Update DNS to eu-west-1 writer
4. Applications reconnect

Failover time: 1-2 minutes
RPO: Near-zero (continuous replication)
```

**Implementation:**

```python
import boto3

rds = boto3.client('rds')

# 1. Create primary cluster in us-east-1
rds.create_db_cluster(
    DBClusterIdentifier='global-db-primary',
    Engine='aurora-mysql',
    MasterUsername='admin',
    MasterUserPassword='SecurePassword123!',
    DatabaseName='production',
    AvailabilityZones=['us-east-1a', 'us-east-1b', 'us-east-1c'],
    StorageEncrypted=True,
    KmsKeyId='arn:aws:kms:us-east-1:123456789012:key/12345678',
    BackupRetentionPeriod=35,
    EnableIAMDatabaseAuthentication=True,
    EnableCloudwatchLogsExports=['error', 'general', 'slowquery']
)

# 2. Create global database
rds.create_global_cluster(
    GlobalClusterIdentifier='global-db',
    Engine='aurora-mysql',
    EngineVersion='8.0.mysql_aurora.3.02.0'
)

# 3. Modify primary cluster to be part of global DB
rds.modify_db_cluster(
    DBClusterIdentifier='global-db-primary',
    GlobalClusterIdentifier='global-db'
)

# 4. Create secondary cluster in eu-west-1
rds.create_db_cluster(
    DBClusterIdentifier='global-db-secondary',
    Engine='aurora-mysql',
    GlobalClusterIdentifier='global-db',
    AvailabilityZones=['eu-west-1a', 'eu-west-1b', 'eu-west-1c']
)

# 5. Add writer instance to primary
rds.create_db_instance(
    DBInstanceIdentifier='primary-writer-1',
    DBInstanceClass='db.r6g.xlarge',
    Engine='aurora-mysql',
    DBClusterIdentifier='global-db-primary',
    PubliclyAccessible=False,
    StorageEncrypted=True
)

# 6. Add reader instances
for i in range(2):
    rds.create_db_instance(
        DBInstanceIdentifier=f'primary-reader-{i+1}',
        DBInstanceClass='db.r6g.xlarge',
        Engine='aurora-mysql',
        DBClusterIdentifier='global-db-primary'
    )

# 7. Add reader instances to secondary
for i in range(3):
    rds.create_db_instance(
        DBInstanceIdentifier=f'secondary-reader-{i+1}',
        DBInstanceClass='db.r6g.xlarge',
        Engine='aurora-mysql',
        DBClusterIdentifier='global-db-secondary'
    )
```

---

## DynamoDB

### DynamoDB Partitioning and Scaling

**Q: Explain DynamoDB partitioning. How does hot partition affect performance?**

**A (Advanced - FAANG Interview):**

```
DynamoDB Partitioning:

Data distributed by Partition Key (hash key)

Partition Function:
Hash(PartitionKey) % NumPartitions = Partition

Example:
Partition Key: user_id

user_id=123 → Hash(123) % 10 = Partition 3
user_id=456 → Hash(456) % 10 = Partition 7
user_id=789 → Hash(789) % 10 = Partition 2

Partition Capacity:
- Each partition: 10 GB storage max
- Each partition: 1000 WCU (write capacity units)
- Each partition: 3000 RCU (read capacity units)

If you provision:
- Table size: 50 GB
- Required partitions: 50 GB / 10 GB = 5 partitions
- WCU provisioned: 500
- WCU per partition: 500 / 5 = 100 WCU each

Hot Partition Problem:

All writes to same partition key:
├─ user_id=1 (celebrity account)
├─ user_id=2
├─ user_id=3
└─ Heavy traffic to user_id=1

Effect:
- Partition with user_id=1 gets throttled first
- Other partitions sit idle
- Application gets 400 errors
- Other users with low traffic unaffected

Scenarios:
✗ BAD: Partition Key = "status" (only "active" or "inactive")
✓ GOOD: Partition Key = user_id (millions of values)

✗ BAD: Partition Key = country_code (high cardinality skew)
✓ GOOD: Add composite key: user_id + timestamp
```

**Designing to Avoid Hot Partitions:**

```python
import boto3
import hashlib

dynamodb = boto3.resource('dynamodb')

# Bad design
table_bad = dynamodb.Table('users')
# If most users are status='active', this becomes hot partition

# Good design with write sharding
class DynamoDBWithSharding:
    def __init__(self, table_name, shard_count=10):
        self.table = dynamodb.Table(table_name)
        self.shard_count = shard_count
    
    def put_item_sharded(self, user_id, data):
        """
        Distribute writes across multiple partitions
        using artificial shard key
        """
        shard_id = int(hashlib.md5(user_id.encode()).hexdigest(), 16) % self.shard_count
        
        # Use composite key: user_id + shard_id
        self.table.put_item(
            Item={
                'PK': f'USER#{user_id}#{shard_id}',  # Partition Key
                'SK': 'METADATA#active',              # Sort Key
                'data': data,
                'timestamp': int(time.time())
            }
        )
    
    def query_user(self, user_id):
        """
        Query across all shards for a user
        Queries in parallel
        """
        results = []
        for shard_id in range(self.shard_count):
            response = self.table.query(
                KeyConditionExpression='PK = :pk AND begins_with(SK, :sk)',
                ExpressionAttributeValues={
                    ':pk': f'USER#{user_id}#{shard_id}',
                    ':sk': 'METADATA#'
                }
            )
            results.extend(response.get('Items', []))
        
        return results

# Usage
sharded_db = DynamoDBWithSharding('users', shard_count=10)

# Writes distributed across 10 partitions automatically
sharded_db.put_item_sharded('user123', {'name': 'Alice', 'age': 30})

# Query returns data from all 10 "shards"
data = sharded_db.query_user('user123')
```

### DynamoDB Streams and TTL

**Q: Design event-driven architecture using DynamoDB Streams.**

**A:**

```
Architecture:

DynamoDB Table (Orders)
    │
    ├─ Put Item: {OrderId: 123, Status: PENDING}
    │
    ▼
DynamoDB Streams
    │
    ├─ STREAM_VIEW_TYPE: NEW_AND_OLD_IMAGES
    │  (Records changes: before and after)
    │
    └─ Records Sent To:
       ├─ Lambda (order processing)
       ├─ Kinesis (analytics)
       └─ SQS (notification queue)

Implementation:
```

```python
import boto3

dynamodb = boto3.resource('dynamodb')
lambda_client = boto3.client('lambda')

# 1. Enable DynamoDB Streams
table = dynamodb.Table('Orders')
table_description = dynamodb.meta.client.describe_table(TableName='Orders')

# Check if streams enabled
if 'StreamSpecification' not in table_description['Table']:
    dynamodb.meta.client.update_table(
        TableName='Orders',
        StreamSpecification={
            'StreamViewType': 'NEW_AND_OLD_IMAGES',
            'StreamEnabled': True
        }
    )

# 2. Lambda function triggered by Streams
lambda_code = '''
import json
import boto3

sns = boto3.client('sns')
ses = boto3.client('ses')

def lambda_handler(event, context):
    """
    Processes DynamoDB stream records
    Sends email notifications
    """
    for record in event['Records']:
        if record['eventName'] == 'MODIFY':
            new_image = record['dynamodb'].get('NewImage')
            old_image = record['dynamodb'].get('OldImage')
            
            # Check if status changed
            old_status = old_image.get('Status', {}).get('S')
            new_status = new_image.get('Status', {}).get('S')
            
            if old_status != new_status and new_status == 'SHIPPED':
                order_id = new_image['OrderId']['S']
                customer_email = new_image['Email']['S']
                
                # Send email
                ses.send_email(
                    Source='orders@company.com',
                    Destination={'ToAddresses': [customer_email]},
                    Message={
                        'Subject': {'Data': f'Order {order_id} shipped!'},
                        'Body': {'Text': {'Data': f'Your order is on the way!'}}
                    }
                )
                
                # Publish event
                sns.publish(
                    TopicArn='arn:aws:sns:region:account:order-events',
                    Message=json.dumps({
                        'OrderId': order_id,
                        'Status': new_status
                    })
                )
    
    return {'statusCode': 200}
'''

# 3. Create event source mapping (Lambda from Streams)
stream_arn = table_description['Table']['LatestStreamArn']
lambda_client.create_event_source_mapping(
    EventSourceArn=stream_arn,
    FunctionName='process-orders',
    Enabled=True,
    BatchSize=100,
    StartingPosition='LATEST',
    MaximumRetryAttempts=2
)

# 4. TTL (Time-To-Live) for automatic cleanup
dynamodb.meta.client.update_time_to_live(
    TableName='Orders',
    TimeToLiveSpecification={
        'Enabled': True,
        'AttributeName': 'ExpirationTime'  # Unix timestamp
    }
)

# When creating item:
table.put_item(
    Item={
        'OrderId': '123',
        'Status': 'PENDING',
        'ExpirationTime': int(time.time()) + 86400 * 30  # Delete after 30 days
    }
)
```

---

## Database Selection Guide

**Q: How do you choose between RDS, Aurora, DynamoDB, and Redshift?**

**A (Critical for Interviews):**

```
┌──────────────────────────────────────────────────────────────┐
│              DATABASE SELECTION MATRIX                       │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│ RDS (MySQL, PostgreSQL, SQL Server)                          │
│ ├─ Use: Structured data, transactions, ACID                 │
│ ├─ Scale: Up to ~65TB storage                               │
│ ├─ Cost: $$$$ (compute + storage)                           │
│ ├─ Examples: Web app backend, business apps                 │
│ └─ Scaling: Vertical (bigger instance) or read replicas    │
│                                                                │
│ Aurora (MySQL/PostgreSQL compatible)                         │
│ ├─ Use: High performance, scaling, auto-repair             │
│ ├─ Scale: Unlimited (auto-scaling storage)                 │
│ ├─ Cost: $$$ (better price/performance than RDS)           │
│ ├─ Examples: SaaS, e-commerce, analytics                   │
│ └─ Scaling: Horizontal (read replicas), global database    │
│                                                                │
│ DynamoDB (NoSQL)                                             │
│ ├─ Use: High throughput, variable load, no complex queries  │
│ ├─ Scale: Millions of requests/second                       │
│ ├─ Cost: Pay-per-request (good for variable load)          │
│ ├─ Examples: Real-time systems, IoT, user sessions         │
│ └─ Scaling: Automatic (provisioned or on-demand)           │
│                                                                │
│ Redshift (Data Warehouse)                                    │
│ ├─ Use: OLAP (analytics), petabyte-scale data               │
│ ├─ Scale: PB+ with massive parallelism                      │
│ ├─ Cost: $$ (per-hour for full cluster)                    │
│ ├─ Examples: Analytics, BI, historical analysis            │
│ └─ Scaling: Add nodes to cluster                            │
│                                                                │
│ ElastiCache (In-Memory)                                      │
│ ├─ Use: Caching, sessions, leaderboards, real-time         │
│ ├─ Scale: Millions of ops/sec                              │
│ ├─ Cost: $$ (memory-based)                                 │
│ ├─ Examples: Cache layer, pub/sub, queues                  │
│ └─ Scaling: Cluster mode, auto-failover                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘

Decision Tree:

Need transactions? → RDS/Aurora
  │
  ├─ < 10GB, < 1000 RPS → RDS
  └─ > 10GB or > 1000 RPS → Aurora

Massive scale (millions RPS)? → DynamoDB
  │
  ├─ Simple queries, high throughput → DynamoDB
  └─ Complex analytics → Redshift

Need to query historical data? → Redshift
│
Need fast access (cache)? → ElastiCache
│
Millions of transactions? → Aurora
```

---

## Interview Questions

### Q1: Design database for multi-tenant SaaS

**A:** (See system design section)

### Q2: RDS is slow. How to debug?

**A:**
1. Check CloudWatch metrics: CPU, memory, IOPS
2. Enable Performance Insights
3. Check slow query log
4. Analyze query execution plans
5. Check for lock contention
6. Consider read replicas for scaling

### Q3: DynamoDB table throttled. Solutions?

**A:**
1. Increase provisioned capacity
2. Switch to on-demand billing
3. Check for hot partitions (use write sharding)
4. Add Global Secondary Indexes
5. Use TTL to clean old data
6. Enable DAX for caching

