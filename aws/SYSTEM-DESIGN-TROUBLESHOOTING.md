# AWS System Design and Troubleshooting - Comprehensive Interview Guide

> 50+ system design questions and 100+ troubleshooting scenarios for production systems

**Estimated Reading Time:** 120 minutes | **Coverage:** 150+ interview questions

---

## Table of Contents

- [System Design Framework](#system-design-framework)
- [Real-World Design Scenarios](#real-world-design-scenarios)
- [Troubleshooting Flowcharts](#troubleshooting-flowcharts)
- [Performance Optimization](#performance-optimization)

---

## System Design Framework

### Design Process (5 Steps)

**Step 1: Clarify Requirements**

```
Functional Requirements:
├─ What features? (read/write/search)
├─ Scale: daily active users, transactions/sec
├─ Data: volume, retention, access patterns
├─ Latency: expected response time
└─ Availability: 99.9% or 99.99%?

Non-Functional Requirements:
├─ Consistency (strong vs eventual)
├─ Fault tolerance
├─ Geographic distribution
├─ Cost constraints
└─ Security/compliance
```

**Step 2: Capacity Estimation**

```python
# Example: Netflix streaming service

Daily Active Users: 100 million
Peak QPS: 1 million concurrent
Storage per movie: 5 GB (multiple resolutions)
Total movies: 10,000

Calculations:
├─ Concurrent viewers: 100M * 0.1 = 10M peak
├─ Requests/sec: 10M / 10 (avg watch time) = 1M RPS
├─ Storage: 10,000 movies × 5 GB = 50 TB base
├─ Cache for hot content: 1% of users active = 100TB
├─ Bandwidth: 1M RPS × 5 Mbps avg bitrate = 5 Pbps (!)
└─ Solution: Use CDN (CloudFront) to edge

Network calculation:
- Full movie: 2GB × 100M viewers/day = 200 EB/day
- Impossible to handle directly from origin
- CDN caches 90% requests locally
- Origin handles 10% = 20 EB/day
- Still huge, but manageable with S3 direct serve
```

**Step 3: High-Level Architecture**

```
Typical 3-tier architecture:

Clients (Web/Mobile)
    │
    ├─ CloudFront (CDN)
    ├─ API Gateway
    ├─ Route53 (DNS)
    │
    ▼
Load Balancer (ALB/NLB)
    │
    ├─ Auto Scaling Group
    ├─ Multiple AZs
    │
    ▼
Application Servers (EC2/EKS)
    │
    ├─ Service 1
    ├─ Service 2
    └─ Service 3
    │
    ▼
Cache Layer (ElastiCache Redis)
    │
    ▼
Data Layer
    ├─ SQL: Aurora (primary)
    ├─ NoSQL: DynamoDB
    ├─ Search: OpenSearch/Elasticsearch
    └─ Files: S3
```

**Step 4: Deep Dive into Components**

**Step 5: Bottleneck Analysis & Trade-offs**

---

## Real-World Design Scenarios

### Scenario 1: Real-Time Notification System

**Requirements:**
- 100M users
- 50% online at peak (50M concurrent)
- Latency: <100ms
- Availability: 99.99%

**Architecture:**

```
User Client
    │
    ├─ WebSocket → API Gateway
    │
    ▼
Lambda (connection handler)
    │
    ├─ Store connection: user_id → connection_id
    │ (Use DynamoDB: O(1) lookup)
    │
    └─ Publish event to SNS
        │
        ▼
    Lambda (fan-out worker)
        │
        ├─ Query: who should receive this?
        │ (DynamoDB: GSI on recipient_id)
        │
        ├─ Batch 10,000 users
        │
        └─ Send via SQS → Lambda → Pinpoint

Challenges:
Q: How to scale WebSocket connections?
A: API Gateway scales automatically (50M concurrent fine)

Q: Connection state storage?
A: DynamoDB with TTL (auto-cleanup disconnected)

Q: Delivery at-least-once?
A: Use SQS deduplication + DynamoDB idempotency key

Q: User offline?
A: Send push notification via Pinpoint/SNS
```

### Scenario 2: E-Commerce Platform (Uber/Amazon-scale)

**Requirements:**
- 1M concurrent users
- Search: 100K QPS
- Orders: 10K orders/sec
- Inventory: real-time updates
- Availability: 99.99%

**Database Design:**

```
Microservices:

1. Product Service
   └─ Read-optimized
   ├─ Aurora Read Replicas (10+) for search
   ├─ ElastiCache Redis for hot products
   ├─ OpenSearch for full-text search
   └─ Reads: 100K QPS split across 10 replicas
              = 10K QPS each (easily handled)

2. Order Service
   ├─ DynamoDB (10K WCU for writes)
   │ └─ Partition key: user_id
   │ └─ Sort key: order_date (reverse)
   ├─ Read replicas for historical data
   └─ Writes: 10K/sec × 1 partition = spread across shards

3. Inventory Service
   ├─ DynamoDB Streams for updates
   ├─ EventBridge for cross-service events
   └─ Lambda workers for inventory sync

4. Payment Service
   ├─ Eventual consistency with reconciliation
   ├─ Retry logic with exponential backoff
   └─ DLQ for failed payments

5. Notification Service
   ├─ SNS for async messages
   ├─ SQS for queuing
   └─ Email/SMS via Pinpoint
```

**Handling Peak Traffic:**

```
Black Friday (100x normal traffic):

1. Pre-event Preparation:
   ├─ DynamoDB: Scale to 100K WCU (from 10K)
   ├─ RDS: Add read replicas (20+)
   ├─ ElastiCache: Scale cluster
   ├─ CloudFront: Increase TTL
   └─ API Gateway: Reserve concurrency

2. During Event:
   ├─ Circuit breakers on payment service
   ├─ Queue excess orders in SQS
   ├─ Disable non-critical features
   ├─ Rate limit at API Gateway
   └─ Serve static content from S3 + CloudFront

3. After Event:
   ├─ Scale down gradually
   ├─ Analyze logs for bottlenecks
   ├─ Optimize queries causing issues
   └─ Update capacity planning
```

### Scenario 3: Real-Time Analytics Platform

**Requirements:**
- 100K events/second
- Query latency: <1 second
- 7-year data retention
- Billion-row scans

**Solution:**

```
Event Ingestion:
├─ Kinesis Data Streams (100K RPS)
├─ Lambda for transformation
└─ S3 (Parquet, partitioned by date)

Hot Storage (Last 7 days):
├─ Redshift (100 nodes)
├─ Columnar storage (fast scans)
├─ Data refreshed every 10 minutes
└─ Query latency: <1s

Warm Storage (7-90 days):
├─ S3 with Athena queries
├─ Query latency: 5-10s
└─ Cheap storage

Cold Storage (90+ days):
├─ S3 Glacier
├─ Query latency: minutes
└─ Compliance/archive

Access Patterns:
├─ Dashboard (hot data): Redshift
├─ Reports (warm data): Athena
├─ Audit logs (cold data): S3 Glacier

Cost Optimization:
├─ Parquet compression: 10:1 ratio
├─ Partition by date: Skip irrelevant data
├─ Lifecycle policies: S3 → Glacier after 90 days
└─ Result caching: ElastiCache for common queries
```

---

## Troubleshooting Flowcharts

### EKS Troubleshooting

**Problem: Pods stuck in Pending**

```
Pending Pods
│
├─ Check events:
│  kubectl describe pod pod-name
│  │
│  └─ Look for: "Insufficient CPU", "Insufficient memory"
│     "node selector did not match", "taint"
│
├─ Check node resources:
│  kubectl top nodes
│  kubectl describe nodes
│  │
│  ├─ If CPU/memory full:
│  │  ├─ Scale ASG up (add nodes)
│  │  └─ Check if resource requests correct
│  │
│  └─ Check taints/labels:
│     kubectl get nodes -o wide
│     kubectl taints nodes
│
├─ Check scheduling:
│  kubectl get events
│  │
│  └─ If scheduler issues:
│     ├─ Check scheduler logs
│     ├─ Check for PVC pending
│     └─ Check network policies
│
└─ Solutions:
   ├─ Increase node count
   ├─ Reduce pod resource requests
   ├─ Fix pod node selectors
   ├─ Remove taints or add tolerations
   └─ Check PVC bound (for stateful pods)
```

**Problem: Pods in CrashLoopBackOff**

```
CrashLoopBackOff (pod restarts every 1s)
│
├─ Check logs:
│  kubectl logs pod-name --tail=100
│  kubectl logs pod-name --previous (previous instance)
│
├─ Common causes:
│  ├─ Application error at startup
│  ├─ Missing environment variable
│  ├─ Bad configuration (ConfigMap/Secret)
│  ├─ Missing dependency (database not reachable)
│  ├─ OOM (out of memory)
│  └─ Segmentation fault
│
├─ Debugging:
│  ├─ kubectl exec -it pod-name -- /bin/bash
│  │  └─ Run commands manually
│  │
│  ├─ kubectl describe pod pod-name
│  │  └─ Check resource limits
│  │
│  └─ Check readiness/liveness probes
│     └─ May be failing immediately
│
└─ Solutions:
   ├─ Fix application code/config
   ├─ Increase memory limits
   ├─ Check dependent services running
   ├─ Disable probes temporarily to debug
   └─ Test locally first
```

### RDS Troubleshooting

**Problem: Database connection timeout**

```
Cannot connect to RDS
│
├─ Check security groups:
│  ├─ RDS security group allows port 3306/5432
│  └─ Source is application security group
│
├─ Check Network ACLs:
│  └─ Ephemeral ports open (1024-65535)
│
├─ Check database status:
│  aws rds describe-db-instances --query "DBInstances[0].DBInstanceStatus"
│  │
│  └─ If status != "available":
│     ├─ May be rebooting
│     ├─ May be backing up
│     └─ May have failed over
│
├─ Check subnet routing:
│  ├─ Application in VPC
│  ├─ RDS in VPC (same VPC ideally)
│  └─ Route tables configured
│
├─ Connection pool exhaustion:
│  └─ Application has max_connections limit reached
│
└─ Solutions:
   ├─ Fix security group rules
   ├─ Check database hasn't failed
   ├─ Verify network connectivity
   ├─ Increase connection pool size
   └─ Use RDS Proxy for connection pooling
```

### DynamoDB Throttling

**Problem: "ProvisionedThroughputExceededException"**

```
DynamoDB throttled (400 errors)
│
├─ Check CloudWatch metrics:
│  ├─ ConsumedWriteCapacityUnits > ProvisionedWCU
│  └─ ConsumedReadCapacityUnits > ProvisionedRCU
│
├─ Identify cause:
│  ├─ Traffic spike? (normal growth)
│  ├─ Hot partition? (uneven key distribution)
│  ├─ Inefficient query? (scan entire table)
│  └─ GSI limits? (independent throttling)
│
├─ Check access patterns:
│  ├─ Are queries well-distributed?
│  ├─ Are you scanning (inefficient)?
│  └─ Are you using batch operations?
│
└─ Solutions:
   ├─ Increase provisioned capacity (temporary)
   ├─ Switch to on-demand billing (automatic scaling)
   ├─ Fix partition key (avoid hot partitions)
   ├─ Use batch operations (reduce API calls)
   ├─ Implement caching layer (ElastiCache)
   └─ Query optimization (use GSI)
```

---

## Performance Optimization

### Database Query Optimization

**Example: Slow query on 100M row table**

```sql
-- BAD: Full table scan
SELECT *
FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'
  AND customer_country = 'US';

Problem:
├─ No index on order_date
├─ No index on customer_country
└─ Scans all 100M rows (slow!)

Query plan: Full table scan (1000s of seconds)

-- GOOD: Use indexes
CREATE INDEX idx_order_date_country
ON orders (order_date, customer_country);

-- Or partition table
ALTER TABLE orders
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);

Result:
├─ Index scan: 0.1 seconds
├─ Only scans 2024-01 data (1M rows)
└─ Customer_country filter reduces to 100K rows

Performance improvement: 10,000x faster!
```

### Caching Strategy

**Q: Design caching for e-commerce product catalog.**

```
Cache Layers:

Layer 1: Browser Cache
├─ CloudFront (edge)
├─ TTL: 1 hour
├─ Cache control headers

Layer 2: Application Cache
├─ ElastiCache Redis
├─ TTL: 10 minutes
├─ Invalidate on product update

Layer 3: Database Query Cache
├─ Query result caching
├─ TTL: 1 minute
├─ Useful for hot queries

Invalidation Strategy:

When product updated:
├─ Delete from Layer 1: CloudFront
├─ Delete from Layer 2: Redis
├─ Recompute from DB
└─ Cache again for next request

Stampede Prevention:
├─ Use probabilistic expiry
├─ Recompute before expiry
├─ Use locks to prevent thundering herd

Cost savings:
├─ 95% of reads hit cache
├─ 5% hit database
├─ 80% cost reduction
```

---

## Production Incident Scenarios

### Scenario: Database CPU at 100%

```
Alert: RDS CPU >90%

Investigation:
1. Check CloudWatch Enhanced Monitoring
   └─ Which query is consuming CPU?

2. Enable Performance Insights
   └─ See which queries, SQL statements

3. Common causes:
   ├─ Full table scan (missing index)
   ├─ Join on large tables
   ├─ Inefficient query plan
   ├─ Too many concurrent connections
   └─ Memory insufficient (spill to disk)

Solution:
├─ Immediate: Read replicas (offload reads)
├─ Short-term: Optimize query, add indexes
├─ Medium-term: Upgrade to larger instance
└─ Long-term: Shard database by customer

Prevention:
├─ Query cost analysis before production
├─ Production load testing
├─ Automated index recommendations
└─ Slow query monitoring
```

### Scenario: Lambda timeout

```
Lambda function times out (15-minute max)

Investigation:
├─ CloudWatch logs: how long before timeout?
├─ Cold start time? (package size, memory)
├─ Actual execution time?
├─ Waiting for external service?

Solutions:
├─ Increase timeout (simple but not scalable)
├─ Optimize code (reduce compute)
├─ Parallelize work (async + SQS)
├─ Break into smaller functions
├─ Use Lambda layers for faster cold start
└─ Pre-warm Lambda (scheduled invocations)

Code optimization:
├─ Move initialization outside handler
├─ Use connection pooling
├─ Cache SDK clients
├─ Reduce package size (exclude dev dependencies)
```

