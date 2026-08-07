# AWS Monitoring, Serverless, and Best Practices - Comprehensive Interview Guide

> CloudWatch, X-Ray, Lambda, API Gateway, SQS, SNS with production scenarios

**Estimated Reading Time:** 90 minutes | **Coverage:** 100+ interview questions

---

## Table of Contents

- [CloudWatch Monitoring](#cloudwatch-monitoring)
- [X-Ray Distributed Tracing](#x-ray-distributed-tracing)
- [Serverless Architecture](#serverless-architecture)
- [Message Queues and Events](#message-queues-and-events)
- [Disaster Recovery](#disaster-recovery)
- [Production Best Practices](#production-best-practices)

---

## CloudWatch Monitoring

### Metrics and Alarms

**Q: Design monitoring for production microservices.**

**A:**

```python
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch')

# 1. Create custom metrics
def log_custom_metric(metric_name, value, unit='Count'):
    cloudwatch.put_metric_data(
        Namespace='MyApplication',
        MetricData=[
            {
                'MetricName': metric_name,
                'Value': value,
                'Unit': unit,
                'Timestamp': datetime.utcnow(),
                'Dimensions': [
                    {'Name': 'Environment', 'Value': 'production'},
                    {'Name': 'Service', 'Value': 'api-server'}
                ]
            }
        ]
    )

# 2. Create alarm for high CPU
cloudwatch.put_metric_alarm(
    AlarmName='EC2-HighCPU',
    ComparisonOperator='GreaterThanThreshold',
    EvaluationPeriods=2,  # 2 consecutive periods
    MetricName='CPUUtilization',
    Namespace='AWS/EC2',
    Period=300,  # 5 minutes
    Statistic='Average',
    Threshold=80.0,
    ActionsEnabled=True,
    AlarmActions=[
        'arn:aws:sns:region:account:alerts-topic'
    ],
    Dimensions=[
        {'Name': 'InstanceId', 'Value': 'i-1234567890abcdef0'}
    ]
)

# 3. Create composite alarm (multiple conditions)
cloudwatch.put_composite_alarm(
    AlarmName='HighErrorRateAndHighLatency',
    AlarmDescription='Alert if both error rate >5% AND p99 latency >1s',
    ActionsEnabled=True,
    AlarmActions=['arn:aws:sns:region:account:critical-alerts'],
    AlarmRule=(
        '(ALARM(ErrorRateHigh) AND ALARM(LatencyHigh)) '
        'OR ALARM(DiskSpaceAlarm)'
    )
)

# 4. Query metrics with CloudWatch Insights
# Log query language for analyzing logs
log_query = '''
fields @timestamp, @message, @duration
| stats avg(@duration), max(@duration), pct(@duration, 95)
| filter @message like /ERROR/
'''
```

**Key Metrics to Monitor:**

```
Application Layer:
├─ Request count (by endpoint)
├─ Error rate (4xx, 5xx)
├─ Latency (p50, p95, p99)
├─ Throughput (requests/sec)
└─ Business metrics (conversions, revenue)

Infrastructure Layer:
├─ CPU utilization (>80% bad)
├─ Memory utilization
├─ Disk space
├─ Network (in/out, dropped packets)
├─ IO throughput
└─ Connection count

Database Layer:
├─ Query latency (by query type)
├─ Connections active
├─ Transaction rate
├─ Replication lag
├─ Storage usage
└─ IOPS used vs provisioned

Alert Thresholds:
├─ Warning: 70% of limit (gradual action)
├─ Critical: 85% of limit (immediate action)
└─ Blocking: 95% of limit (emergency response)
```

### CloudWatch Logs and Log Groups

**Q: Design log aggregation and analysis.**

**A:**

```python
import boto3
import json
from datetime import datetime

logs = boto3.client('logs')

# 1. Create log group with retention
logs.create_log_group(logGroupName='/aws/lambda/my-function')

logs.put_retention_policy(
    logGroupName='/aws/lambda/my-function',
    retentionInDays=30  # Auto-delete after 30 days
)

# 2. Create subscription filter (stream logs to another service)
logs.put_subscription_filter(
    logGroupName='/aws/lambda/my-function',
    filterName='ErrorFilter',
    filterPattern='[ERROR]',  # Only match error logs
    destinationArn='arn:aws:lambda:region:account:function:process-errors'
)

# 3. Log event with context
def lambda_handler(event, context):
    # Structured logging (JSON for easy parsing)
    log_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'request_id': context.request_id,
        'function_name': context.function_name,
        'level': 'INFO',
        'message': 'Processing event',
        'event_data': event,
        'duration_ms': 0
    }
    
    print(json.dumps(log_entry))  # CloudWatch captures stdout
    
    # Simulate work
    result = process_event(event)
    
    log_entry['level'] = 'INFO'
    log_entry['message'] = 'Event processed successfully'
    log_entry['result'] = result
    print(json.dumps(log_entry))
    
    return result

# 4. Query logs with CloudWatch Insights
response = logs.start_query(
    logGroupName='/aws/lambda/my-function',
    startTime=int((datetime.utcnow() - timedelta(hours=1)).timestamp()),
    endTime=int(datetime.utcnow().timestamp()),
    queryString='''
        fields @timestamp, @message, @duration
        | filter @message like /ERROR/
        | stats count() as error_count, 
                avg(@duration) as avg_duration,
                max(@duration) as max_duration
        by bin(@timestamp, 1m)
    '''
)

# 5. Filter patterns for common scenarios
patterns = {
    'errors': '[ERROR]',
    'warnings': '[WARN]',
    'http_errors': '{ $.statusCode >= 400 }',
    'latency': '{ $.duration > 1000 }',
    'all_fields': '{ $.field1 = * && $.field2 = * }'
}
```

---

## X-Ray Distributed Tracing

### End-to-End Request Tracing

**Q: Design X-Ray tracing for microservices.**

**A:**

```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all
import boto3

# Patch all AWS SDK calls
patch_all()

# Configure X-Ray
xray_recorder.configure(
    service='OrderService',
    context_missing='LOG_ERROR'
)

@xray_recorder.capture('process_order')
def process_order(order_id):
    """
    Automatically creates X-Ray segment
    Shows in console:
    ├─ Function name: process_order
    ├─ Duration
    ├─ Error status
    └─ Nested calls
    """
    
    # Add metadata
    xray_recorder.put_metadata('order_id', order_id)
    
    # Subsegment for database call
    with xray_recorder.capture('get_order_from_db'):
        order = get_order(order_id)
    
    # Subsegment for payment processing
    with xray_recorder.capture('process_payment'):
        payment_result = process_payment(order)
        xray_recorder.put_annotation('payment_status', payment_result['status'])
    
    # Subsegment for notification
    with xray_recorder.capture('send_notification'):
        send_order_confirmation(order)
    
    return order

# Lambda handler
def lambda_handler(event, context):
    order_id = event['order_id']
    
    try:
        result = process_order(order_id)
        return {'statusCode': 200, 'body': result}
    except Exception as e:
        xray_recorder.put_annotation('error_type', type(e).__name__)
        xray_recorder.put_metadata('error_details', str(e))
        raise
```

**X-Ray Service Map:**

```
Shows dependencies between services:

Client
  │
  ├─ API Gateway (50ms)
  │
  ├─ Lambda: OrderService (200ms)
  │   ├─ DynamoDB: Orders table (30ms)
  │   ├─ Lambda: PaymentService (100ms)
  │   │   └─ S3: stripe-keys (5ms)
  │   └─ SNS: notifications (10ms)
  │
  └─ Response (200ms)

Benefits:
├─ Visualize call flow
├─ Identify bottlenecks (which service is slow?)
├─ Debug errors (trace request through services)
├─ Understand dependencies
└─ Measure latency at each step
```

---

## Serverless Architecture

### Lambda Best Practices

**Q: Design production Lambda functions.**

**A:**

```python
import json
import logging
import time
from datetime import datetime
import boto3
from functools import lru_cache
from aws_lambda_powertools import Logger, Metrics, Tracer

# Initialize once (outside handler)
logger = Logger()
metrics = Metrics()
tracer = Tracer()

dynamodb = boto3.resource('dynamodb')
s3 = boto3.client('s3')

# Cache expensive operations
@lru_cache(maxsize=128)
def get_config():
    """Cached configuration - avoids repeated API calls"""
    return {
        'max_retries': 3,
        'timeout': 30,
        'batch_size': 100
    }

# Lambda handler best practice
@logger.inject_lambda_context  # Adds request_id automatically
@tracer.capture_lambda_handler
@metrics.log_cold_start_metric
def lambda_handler(event, context):
    """
    Production-grade Lambda handler
    
    Guidelines:
    ✓ Keep handler focused
    ✓ Initialize connections outside
    ✓ Use environment variables
    ✓ Implement idempotency
    ✓ Handle errors gracefully
    ✓ Monitor with X-Ray/CloudWatch
    """
    
    try:
        # Extract and validate input
        order_id = event.get('order_id')
        if not order_id:
            logger.info("Missing order_id", order_id=order_id)
            return {'statusCode': 400, 'error': 'Missing order_id'}
        
        # Log with context
        logger.info(f"Processing order", order_id=order_id)
        metrics.add_metric(name="OrderProcessed", unit="Count", value=1)
        
        # Business logic
        result = process_order(order_id)
        
        # Log success
        logger.info(f"Order processed successfully", 
                   order_id=order_id, 
                   result=result)
        metrics.add_metric(name="ProcessingSuccess", unit="Count", value=1)
        
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
        
    except ValueError as e:
        logger.warning(f"Invalid input: {e}", order_id=order_id)
        metrics.add_metric(name="ProcessingError", unit="Count", value=1)
        return {'statusCode': 400, 'error': str(e)}
        
    except Exception as e:
        logger.exception(f"Unexpected error: {e}")
        metrics.add_metric(name="ProcessingFailure", unit="Count", value=1)
        # Re-raise so Lambda retries
        raise

# Idempotent write
def save_order_idempotent(order_id, order_data):
    """
    Ensure function is idempotent (safe to retry)
    
    Solution: use idempotency key
    """
    idempotency_key = f"order#{order_id}"
    
    table = dynamodb.Table('Orders')
    
    try:
        response = table.put_item(
            Item={
                'order_id': order_id,
                'idempotency_key': idempotency_key,
                'data': order_data,
                'timestamp': int(time.time())
            },
            ConditionExpression='attribute_not_exists(idempotency_key)',
            # Fail if already exists
        )
        logger.info("Order saved", order_id=order_id)
        return response
        
    except table.meta.client.exceptions.ConditionalCheckFailedException:
        logger.info("Order already exists (idempotent retry)", 
                   order_id=order_id)
        # Return cached result
        existing = table.get_item(Key={'order_id': order_id})
        return existing['Item']

# Context properties
def demonstrate_lambda_context(event, context):
    logger.info("Lambda context info:", extra={
        'function_name': context.function_name,
        'request_id': context.request_id,
        'aws_request_id': context.aws_request_id,
        'invoked_function_arn': context.invoked_function_arn,
        'remaining_time_ms': context.get_remaining_time_in_millis(),
        'memory_limit': context.memory_limit_in_mb
    })
```

### Lambda Concurrency and Scaling

**Q: Lambda function throttled. How to debug?**

**A:**

```
Lambda Scaling:

Concurrent executions = # of requests being processed simultaneously

Limits (per region, per account):
├─ Default: 1000 concurrent executions
├─ Reserved concurrency: Dedicate specific count to function
└─ Provisioned concurrency: Pre-warm containers

Throttling (429 errors):

Scenario 1: Exceeded account limit
├─ All Lambda functions share 1000 limit
├─ If total > 1000, new invocations throttled
└─ Solution: Request limit increase or optimize

Scenario 2: Exceeded function reserved limit
├─ Function set to 100 concurrent
├─ 101st invocation gets throttled
└─ Solution: Increase reserved concurrency

Scenario 3: Burst capacity exhausted
├─ Initial burst: 500-3000 concurrent (region dependent)
├─ After burst, scales at 500/minute
├─ Spike above burst = throttled
└─ Solution: Provisioned concurrency

Debugging:
├─ CloudWatch Logs: "LimitExceededException"
├─ Metric: Throttles
├─ Metric: Duration increase (queueing)
└─ Solutions:
   ├─ Increase reserved/provisioned concurrency
   ├─ Optimize function (run faster)
   ├─ Use SQS to queue and throttle gracefully
   └─ Use API Gateway request throttling
```

---

## Message Queues and Events

### SQS vs SNS vs EventBridge

**Q: Choose between SQS, SNS, and EventBridge.**

**A:**

```
┌─────────────────────────────────────────────────────────────┐
│              MESSAGE SERVICE COMPARISON                      │
├─────────────────────────────────────────────────────────────┤
│                                                                │
│ SQS (Simple Queue Service)                                   │
│ ├─ Type: Queue (FIFO or Standard)                           │
│ ├─ Delivery: One consumer per message                       │
│ ├─ Use: Decouple services, load leveling                    │
│ ├─ Latency: 0-30 seconds                                    │
│ ├─ Order: FIFO guaranteed, Standard ~order                 │
│ ├─ Scaling: Auto-scale based on queue depth                │
│ ├─ Pricing: Per million messages                            │
│ └─ Example: Order → SQS → Lambda processor                 │
│                                                                │
│ SNS (Simple Notification Service)                            │
│ ├─ Type: Publish/Subscribe (push)                           │
│ ├─ Delivery: Multiple subscribers                           │
│ ├─ Use: Notifications, broadcasts                           │
│ ├─ Latency: <1 second                                       │
│ ├─ Order: No guarantee                                      │
│ ├─ Scaling: Infinite subscribers                            │
│ ├─ Pricing: Per million publishes                           │
│ └─ Example: Order created → SNS → Email + SMS + Lambda     │
│                                                                │
│ EventBridge                                                   │
│ ├─ Type: Event Router (rules-based)                         │
│ ├─ Delivery: Route to multiple targets based on rules       │
│ ├─ Use: Event-driven architecture, complex routing          │
│ ├─ Latency: <1 second                                       │
│ ├─ Order: Per event source                                  │
│ ├─ Scaling: Unlimited                                       │
│ ├─ Pricing: Per million events                              │
│ └─ Example: Lambda → EventBridge → filter → route           │
│                                                                │
└─────────────────────────────────────────────────────────────┘

Decision Tree:

Need to guarantee order?
├─ Yes → SQS FIFO
└─ No → SNS or EventBridge

Multiple consumers for same event?
├─ Yes → SNS or EventBridge
└─ No → SQS

Complex routing logic?
├─ Yes → EventBridge
└─ No → SNS

Can afford message loss (best effort)?
├─ Yes → SNS
└─ No (need guaranteed delivery) → SQS
```

### Event-Driven Architecture

**Q: Design order processing with event-driven architecture.**

**A:**

```
Order Creation Flow:

User creates order
│
▼
POST /orders (API)
│
▼
Lambda: CreateOrder
├─ Validate input
├─ Save to DynamoDB
├─ Publish event: "OrderCreated"
└─ Return order_id

OrderCreated event → EventBridge
│
├─ Rule 1: Send confirmation email
│  └─ Lambda: SendEmailConfirmation
│
├─ Rule 2: Process payment
│  └─ Lambda: ProcessPayment
│
├─ Rule 3: Update inventory
│  └─ Lambda: UpdateInventory
│
├─ Rule 4: Notify warehouse
│  └─ SNS topic: WarehouseNotifications
│
└─ Rule 5: Log for analytics
   └─ S3: events bucket

Benefits:
✓ Loosely coupled (services don't know about each other)
✓ Easy to add new handlers (add new EventBridge rule)
✓ Scalable (each handler scales independently)
✓ Maintainable (change one service without affecting others)
```

---

## Disaster Recovery

### Backup and Recovery

**Q: Design RDS backup and recovery strategy.**

**A:**

```
Backup Strategy:

1. Automated Backups (Daily)
   ├─ Retention: 35 days (configurable)
   ├─ Frequency: Daily at maintenance window
   ├─ Location: AWS-managed
   └─ Point-in-time recovery: Any time in retention period

2. Manual Snapshots
   ├─ Triggered manually before risky changes
   ├─ Retention: Until manually deleted
   └─ Use case: Preserve state before migration

3. Read Replica (Different Region)
   ├─ Continuous replication (RPO ~1 second)
   ├─ Can promote to standalone
   ├─ Useful for warm DR
   └─ Cost: Read replica instance charge

4. Automated Backup to S3
   ├─ Use AWS Database Migration Service
   ├─ Daily backups to S3 (cheaper storage)
   ├─ Retention: Years (for compliance)
   └─ Restore: Via Parquet files + Athena

Recovery Scenarios:

Accidental deletion (recent):
├─ Use point-in-time restore
├─ Restore to a different DB instance
├─ Verify data
└─ Promote when ready

Region failure:
├─ Promote read replica to primary
├─ Update DNS/connection strings
├─ Failover time: 2-5 minutes
└─ No data loss (continuous replication)

Corruption detected (1 week old):
├─ Restore from snapshot (1 week old)
├─ Verify data
├─ Decide: Use restored data or retry from backup before corruption
└─ Implement data validation to detect earlier

RPO/RTO Goals:

RPO (Recovery Point Objective): Data loss acceptable?
├─ 5-minute RPO: Lose ≤5 minutes of data
├─ Solution: Backups every 5 minutes or read replicas

RTO (Recovery Time Objective): How fast must we recover?
├─ 15-minute RTO: Back online within 15 minutes
├─ Solution: Read replica + automated failover
```

---

## Production Best Practices

### Security Best Practices

```yaml
Identity & Access:
  ✓ Use IAM roles (not access keys)
  ✓ Principle of least privilege
  ✓ Enable MFA for human users
  ✓ Rotate credentials regularly
  ✓ Use temporary credentials (STS)
  ✓ Cross-account roles for multi-account setup

Data Protection:
  ✓ Encryption at rest (S3, RDS, EBS)
  ✓ Encryption in transit (HTTPS, VPN)
  ✓ Use KMS for key management
  ✓ Enable versioning (audit trail)
  ✓ Regular backups (tested recovery)

Network Security:
  ✓ Principle of zero trust
  ✓ Security groups (stateful firewall)
  ✓ NACLs (stateless firewall)
  ✓ VPC endpoints for private access
  ✓ No public internet exposure
  ✓ WAF for DDoS/web attacks

Compliance:
  ✓ CloudTrail logging (audit)
  ✓ Config monitoring (drift detection)
  ✓ Data residency (specific regions)
  ✓ HIPAA/PCI/SOC2 requirements
  ✓ Regular security audits
```

### Cost Optimization

```
Reserved Instances:
├─ Commit to 1 or 3 years
├─ 30-60% discount vs on-demand
├─ Best for: Baseline load (never fluctuates)
└─ Example: ECS prod always needs 5 nodes

Spot Instances:
├─ 70% discount (but can be interrupted)
├─ Best for: Batch jobs, CI/CD, non-critical
└─ Mix: 1 on-demand + 3 spot (cost savings)

Savings Plans:
├─ Commit to hourly spend ($1/hour)
├─ 25-45% discount
├─ More flexible than RI
└─ Best for: Variable workloads

Right-sizing:
├─ Don't over-provision
├─ Monitor actual utilization
├─ Use CloudWatch metrics
└─ Example: t3.large sitting idle → downsize

Data transfer costs:
├─ Out of region: expensive ($0.02/GB)
├─ Within region: free
├─ Use CloudFront to reduce egress
└─ Example: Saved $50k/month with CDN
```

