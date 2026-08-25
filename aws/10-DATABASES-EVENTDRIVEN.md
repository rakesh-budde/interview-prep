# SECTION 10: AWS DATABASES & EVENT-DRIVEN ARCHITECTURE

## TABLE OF CONTENTS
- [AWS Databases Deep Dive](#aws-databases-deep-dive)
- [Event-Driven Architecture](#event-driven-architecture)
- [Interview Questions](#interview-questions)
- [Troubleshooting Scenarios](#troubleshooting-scenarios)
- [System Design Examples](#system-design-examples)

---

## AWS DATABASES DEEP DIVE

### 1. RELATIONAL DATABASES (RDS & AURORA)

#### Concept Overview

**RDS (Relational Database Service)** is a managed relational database service supporting MySQL, PostgreSQL, SQL Server, Oracle, and MariaDB. **Aurora** is AWS's proprietary high-performance SQL engine compatible with MySQL and PostgreSQL.

**Problem solved:** Eliminates operational overhead of database administration—patching, backups, replication, and failover are automated. Allows developers to focus on application logic rather than infrastructure.

**When to use:** Applications requiring ACID compliance, complex joins, structured data with schema, or existing relational SQL expertise. Do NOT use for unstructured data, massive horizontal scaling (>100k writes/sec), or analytics-only workloads.

#### Beginner Foundation

**RDS Architecture:**
- **Multi-AZ Deployment:** Primary and synchronous standby in different AZ. Automatic failover on primary failure (~60–120 seconds). Standby is NOT available for reads during normal operation.
- **Read Replicas:** Asynchronous copies in same or different region. Can be promoted to independent DB. Used for read scaling, not high-availability.
- **Storage:** EBS volumes (gp3/io1) with automatic snapshots, point-in-time restore (PITR).

**Aurora Architecture:**
- **Cluster Architecture:** Single writer + 0–15 read replicas sharing a distributed storage layer across 3 AZs.
- **Storage:** Shared data volume spread across 3 AZs with 6-way replication internally (appears as single logical volume).
- **Performance:** Up to 5x faster writes than MySQL RDS, 3x faster reads.
- **Failover:** Reader promotes to writer automatically in ~30 seconds. No need for external failover logic.

**Key Terminology:**
- **IOPS:** Input/Output Operations Per Second. Provisioned IOPS = guaranteed baseline. Burstable = temporary spikes.
- **Throughput:** MB/s of data transfer.
- **Parameter Group:** Template for database config (timeout, memory, logging).
- **Option Group:** Extensions (Oracle RAC, SQL Server agents).

#### Intermediate Mechanics

**Multi-AZ Failover Flow:**

```
Mermaid:
graph LR
    App["Application"] -->|writes| Primary["Primary DB (AZ-A)"]
    App -->|reads| Primary
    Primary -->|sync replication| Standby["Standby DB (AZ-B)"]
    Standby -->|no reads| Silent["Silent Replica"]
    Monitor["RDS Monitor"] -->|health check| Primary
    Monitor -->|detects failure| Failover["Promotion Logic"]
    Failover -->|promote| Standby
    Failover -->|DNS update| DNSChange["Route53 changes CNAME"]
    DNSChange -->|app reconnects| Primary
```

**How it works:**
1. Application issues write to Primary endpoint (e.g., `mydb.xxx.rds.amazonaws.com`).
2. Primary writes to EBS volume and sends synchronous acknowledgment to application.
3. Standby receives the same write synchronously but does NOT acknowledge to application—it just persists.
4. If Primary fails (EC2 instance crash, network partition, storage failure), RDS detects via TCP health checks.
5. RDS waits ~60 seconds to confirm failure (avoid flapping), then promotes Standby.
6. Route53 updates the CNAME to point to the former Standby (now Primary).
7. Application reconnects—reads/writes resume. ~2–3 minutes of downtime typical.

**Read Replicas (Asynchronous):**
- Updates replicated with ~100ms lag (same region) or variable lag (cross-region).
- Reads can be distributed across replicas to reduce primary load.
- No automatic promotion—manual or Lambda-based promotion on primary failure.
- Source and replica can have different instance types, storage.
- Cost: Replica incurs storage + compute charges. No inter-AZ replication cost within region; cross-region replication charged per GB.

**Aurora Cluster Mechanics:**
- **Writer Endpoint:** Always points to primary. Application writes here.
- **Reader Endpoint:** Load-balances across all read replicas. Application reads here (or specific replica endpoint).
- **Scaling Read Replicas:** Add/remove replicas without downtime. Each replica gets a copy of data immediately from shared storage.
- **Failover:** If writer fails, RDS promotes one replica within ~30 seconds. Reader endpoint continues to work (re-balanced).
- **Backtrack:** Aurora MySQL-compatible only. Rewind DB to a past timestamp without restoring from snapshot. Useful for accidental DELETE. No extra cost if enabled within backup retention.

**Storage & Backup:**
- **Automated Backups:** Daily snapshots + transaction logs. Retention 1–35 days. Enables PITR to any second within retention window.
- **Manual Snapshots:** Keep indefinitely. Cross-region copy for disaster recovery.
- **Snapshot Restore:** Creates new DB instance. No change to original. RTO ~10 minutes (depends on DB size).
- **Binary Log Retention (MySQL/Aurora MySQL):** Incremental backups via binlog replication. Used for point-in-time recovery and read replica lag detection.

**Configuration Examples:**

**RDS Multi-AZ with Terraform:**
```hcl
resource "aws_db_instance" "postgres" {
  identifier           = "prod-postgres"
  engine               = "postgres"
  engine_version       = "15.4"
  instance_class       = "db.r6i.xlarge"  # Memory-optimized for prod
  allocated_storage    = 1000  # GB
  storage_type         = "gp3"
  iops                 = 5000  # Provisioned IOPS
  
  db_name              = "appdb"
  username             = "adminuser"
  password             = var.db_password  # Use AWS Secrets Manager
  
  multi_az             = true  # Enables standby in different AZ
  publicly_accessible  = false
  
  backup_retention_period = 30  # days
  backup_window           = "03:00-04:00"  # UTC
  maintenance_window      = "sun:04:00-sun:05:00"
  
  skip_final_snapshot = false
  final_snapshot_identifier = "prod-postgres-final-${formattime("YYYY-MM-DD-hhmm", timestamp())}"
  
  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  enabled_cloudwatch_logs_exports = ["postgresql"]
  
  tags = {
    Environment = "production"
    CriticalityTier = "high"
  }
}

resource "aws_db_subnet_group" "main" {
  name       = "prod-subnet-group"
  subnet_ids = [aws_subnet.private_1.id, aws_subnet.private_2.id, aws_subnet.private_3.id]
}
```

**Aurora Cluster with Terraform:**
```hcl
resource "aws_rds_cluster" "aurora_prod" {
  cluster_identifier      = "prod-aurora-mysql"
  engine                  = "aurora-mysql"
  engine_version          = "8.0.mysql_aurora.3.04.0"
  database_name           = "appdb"
  master_username         = "admin"
  master_password         = var.db_password
  
  db_subnet_group_name            = aws_db_subnet_group.main.name
  db_cluster_parameter_group_name = aws_rds_cluster_parameter_group.aurora.name
  
  backup_retention_period      = 35  # max for Aurora
  preferred_backup_window      = "03:00-04:00"
  preferred_maintenance_window = "sun:04:00-sun:05:00"
  
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
  
  enabled_cloudwatch_logs_exports = ["error", "general", "slowquery", "audit"]
  
  deletion_protection = true
  skip_final_snapshot = false
  final_snapshot_identifier_prefix = "prod-aurora-final"
  
  # Enable backtrack (MySQL only)
  backtrack_window = 72  # hours
  
  tags = {
    Environment = "production"
  }
}

resource "aws_rds_cluster_instance" "aurora_writer" {
  cluster_identifier = aws_rds_cluster.aurora_prod.id
  instance_class     = "db.r6g.xlarge"
  engine              = aws_rds_cluster.aurora_prod.engine
  engine_version      = aws_rds_cluster.aurora_prod.engine_version
  
  publicly_accessible = false
  
  performance_insights_enabled = true
  monitoring_interval          = 60  # Enhanced monitoring
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn
  
  identifier = "prod-aurora-mysql-writer-1"
}

resource "aws_rds_cluster_instance" "aurora_readers" {
  count              = 2  # 2 read replicas
  cluster_identifier = aws_rds_cluster.aurora_prod.id
  instance_class     = "db.r6g.large"  # Smaller than writer
  engine              = aws_rds_cluster.aurora_prod.engine
  engine_version      = aws_rds_cluster.aurora_prod.engine_version
  
  publicly_accessible = false
  
  identifier = "prod-aurora-mysql-reader-${count.index + 1}"
}
```

#### Advanced Engineering

**Consistency Models:**
- **Strong Consistency (Multi-AZ Primary):** Writes synchronously replicated to standby before ACK. RPO = 0 (no data loss). Slight write latency (~5–10ms extra for sync replication).
- **Eventual Consistency (Read Replicas):** Asynchronous replication. RPO = max replica lag (typically 100ms–1s). Writes fast, reads may see stale data briefly.
- **Aurora Global Database:** Cross-region read-only replicas. Secondary region promoted in ~1 minute on primary region failure. RPO ~1 second (async replication lag).

**Failure Modes & Limits:**
- **Max Connections:** Limited by instance class and memory. `max_connections` parameter. Exceed = connection refused errors. Monitor via CloudWatch `DatabaseConnections` metric.
- **Storage Full:** No more writes. Must increase allocated storage. RDS pauses autoscaling if insufficient space to grow (gp3 volumes).
- **High CPU:** Queries running too long, index missing, lock contention. Monitor `CPUUtilization`, `DatabaseLoad` from Performance Insights. Increase instance class or optimize queries.
- **Network Partition:** Multi-AZ promotes standby even if primary is alive but unreachable. Can cause two primaries briefly (split-brain). RDS mitigates via cluster membership protocol; application must handle reconnects.
- **Parameter Store Lag:** Parameter changes applied at next maintenance window unless `immediately_apply=true` (causes brief downtime).

**Observability & Metrics:**
- **CloudWatch Metrics:** `DatabaseConnections`, `CPUUtilization`, `FreeableMemory`, `ReadLatency`, `WriteLatency`, `DiskQueueDepth`.
- **Performance Insights:** Shows active sessions, wait events (I/O, locks, CPU). Identify query bottlenecks.
- **Enhanced Monitoring:** OS-level metrics (processes, file handles, network I/O). Available for MySQL 5.7+, PostgreSQL 9.6+, Aurora.
- **Slow Query Log:** Queries exceeding `long_query_time`. Enable via parameter group. Log to CloudWatch.
- **RDS Proxy Metrics:** Connection pool health, query latency, connection churn rate.

**RDS Proxy (Connection Pooling):**
- Sits between application and database. Multiplexes application connections over fewer database connections.
- Reduces connection overhead for serverless/microservices architectures (Lambda, ECS).
- Transparent to application (swap endpoint).
- Supports IAM authentication (no password in app config).
- Max idle timeout, connection borrow timeout configurable.
- Useful when app opens/closes connections frequently or has many concurrent clients.

**Cost Optimization:**
- **Reserved Instances:** 1-year or 3-year commitment, ~30–40% discount. Use for stable production loads.
- **Savings Plans:** Flexible across instance types/regions, ~30–35% discount.
- **On-Demand:** Full price, no commitment. Use for dev/test or variable load.
- **gp3 vs io1:** gp3 includes 3000 IOPS free, scales to 16000 IOPS. io1 charged per IOPS. gp3 cheaper for most workloads.
- **Read Replicas Pricing:** Charged for compute + storage + cross-region traffic (if applicable). Use only if read load justifies cost.

---

### 2. DYNAMODB (NoSQL DISTRIBUTED DATABASE)

#### Concept Overview

**DynamoDB** is a fully managed, serverless NoSQL database optimized for high-scale, low-latency workloads. Stores semi-structured JSON data in tables with partition keys and optional sort keys.

**Problem solved:** Provides consistency at scale without sharding complexity. Single digit millisecond latency at any scale. Automatic scaling, built-in encryption, PITR, global replication.

**When to use:** High-traffic APIs (>1000 req/sec), session stores, real-time analytics, IoT data ingestion, leaderboards. Do NOT use for complex joins, strong transactions across multiple items, or structured SQL queries.

#### Beginner Foundation

**Core Concepts:**
- **Table:** Collection of items. Must define Partition Key (PK) and optional Sort Key (SK).
- **Item:** Single record = one partition key value + optional sort key value + attributes (JSON).
- **Attributes:** Flexible schema. Each item can have different attributes (unlike RDBMS).
- **Partition Key (PK):** Hashes to partition. Determines which partition stores the item. Must be unique per item (if no SK) or unique per PK+SK combo.
- **Sort Key (SK):** Orders items within partition. Enables range queries. E.g., PK=UserID, SK=Timestamp.
- **Global Secondary Index (GSI):** Alternative key schema. Partition on different attribute. Eventual consistency.
- **Local Secondary Index (LSI):** Same PK, different SK. Strong consistency. Max 10GB per PK value.

**Capacity Modes:**
- **Provisioned:** Specify read/write capacity units (RCU/WCU). Billed per unit. Predictable cost, risk of throttling if exceeding capacity.
- **On-Demand:** Pay per request. No provisioning. Handles bursts automatically. More expensive at high sustained throughput.

**Consistency:**
- **Strong Read:** Latest write. ~2x cost of eventual read. Default for `GetItem`, `Query`.
- **Eventual Read:** May return stale data. Reads immediately after writes may miss them (~1ms lag). Cheaper, faster.

#### Intermediate Mechanics

**Request Flow & Partitioning:**

```
Mermaid:
graph LR
    App["Application"] -->|PutItem<br/>UserID=123| Routing["DynamoDB Routing"]
    Routing -->|hash(123)| Partition["Partition A"]
    Partition -->|store Item| Node1["Replica 1 (AZ-1)"]
    Partition -->|replicate| Node2["Replica 2 (AZ-2)"]
    Partition -->|replicate| Node3["Replica 3 (AZ-3)"]
    
    App2["App Query"] -->|Query PK=123| Routing2["Route to Partition A"]
    Routing2 -->|strong read| Node1
    Routing2 -->|eventual read| Node2
```

**Write Path:**
1. Application calls `PutItem` with item (PK + attributes).
2. DynamoDB SDK computes hash of PK, routes to correct partition.
3. Partition leader (primary replica) writes to log and memory.
4. Replicates synchronously to 2 other replicas across AZs.
5. When 2/3 replicas ack, returns success to application.
6. Cost: 1 WCU = 1KB write (or part thereof). PutItem consuming 2.5KB = 3 WCU.

**Read Path:**
- **Strong Read:** Contacts replica 1 only. Guarantees latest data. 1 RCU = 4KB (eventually) or 2KB (strongly).
- **Eventual Read:** Can contact any replica. Cheaper, faster, stale (~1ms).

**Query & Scan:**
- **Query:** Specify PK, optionally SK range. Returns all matching items. Max 1MB per request. Consumes RCU for data returned.
- **Scan:** Full table scan. Inefficient at scale. Consumes RCU for ALL items examined, even filtered out.
- **Filter Expression:** Applied AFTER query/scan, reduces returned items but STILL consumes RCU for examined items. Inefficient if filtering >90% of data—add GSI instead.
- **Pagination:** Scan/Query returns `LastEvaluatedKey` if result > 1MB. Use to fetch next page.

**Capacity & Throttling:**
- Provision RCU/WCU or use on-demand.
- If traffic exceeds provisioned capacity, requests return 400 `ProvisionedThroughputExceededException`.
- Burst capacity: DynamoDB allows brief spikes (30 seconds worth) before throttling. Sustained traffic > provisioned = throttling.
- Uneven partition loading (hot partition): Single partition can still be throttled if one PK is heavily queried (e.g., celebrity UserID). Solved by adding GSI or changing PK design.

**Terraform Example:**

```hcl
resource "aws_dynamodb_table" "user_sessions" {
  name           = "user-sessions"
  billing_mode   = "PAY_PER_REQUEST"  # On-demand
  hash_key       = "UserID"
  range_key      = "SessionID"
  
  attribute {
    name = "UserID"
    type = "S"  # String
  }
  
  attribute {
    name = "SessionID"
    type = "S"
  }
  
  attribute {
    name = "CreatedAt"
    type = "N"  # Number (timestamp)
  }
  
  # Global Secondary Index for time-based queries
  global_secondary_index {
    name            = "CreatedAtIndex"
    hash_key        = "CreatedAt"
    projection_type = "ALL"  # Return all attributes
    write_capacity_units  = 10
    read_capacity_units   = 10
  }
  
  ttl {
    attribute_name = "ExpirationTime"
    enabled        = true  # Auto-delete expired items
  }
  
  point_in_time_recovery_enabled = true
  encryption_at_rest_enabled = true
  
  stream_specification {
    stream_view_type = "NEW_AND_OLD_IMAGES"  # Capture changes
  }
  
  tags = {
    Environment = "production"
  }
}
```

**DynamoDB Streams & Lambda:**
- Streams capture changes (PutItem, UpdateItem, DeleteItem).
- Lambda triggered for each stream record.
- Use for: real-time indexing to Elasticsearch, cache invalidation, audit logging, cross-table consistency.

**Configuration CLI:**

```bash
# Create table
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions AttributeName=OrderID,AttributeType=S AttributeName=UserID,AttributeType=S \
  --key-schema AttributeName=OrderID,KeyType=HASH AttributeName=UserID,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1

# Put item
aws dynamodb put-item \
  --table-name Orders \
  --item '{"OrderID":{"S":"ORD-001"},"UserID":{"S":"user-123"},"Amount":{"N":"99.99"},"Status":{"S":"pending"}}' \
  --region us-east-1

# Query items
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression "OrderID = :oid" \
  --expression-attribute-values '{":oid":{"S":"ORD-001"}}' \
  --region us-east-1

# Get item metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedReadCapacityUnits \
  --dimensions Name=TableName,Value=Orders \
  --statistics Sum \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 300 \
  --region us-east-1
```

#### Advanced Engineering

**Consistency & CAP Theorem:**
- DynamoDB trades **Consistency** for **Availability** and **Partition tolerance** (AP model).
- Strong reads sacrifice latency/availability for consistency.
- Eventual reads sacrifice consistency for latency/availability.
- Global tables are eventually consistent across regions (~1s sync lag).

**Hot Partitions & Scaling Issues:**
- If one PK value (e.g., a popular user) gets most traffic, DynamoDB routes all requests to one partition.
- Single partition cannot scale beyond ~40k RCU (example limit).
- Solution: Use composite PK (e.g., `UserID#Date` instead of just `UserID`). Distributes load.
- Kinesis or event queue as buffer for write bursts (decouple app from DB).

**Transaction Support (DynamoDB Transactions):**
- `TransactWriteItems`: Atomically write to multiple items across partitions.
- `TransactGetItems`: Atomically read multiple items.
- Max 25 items per transaction, max 4MB per transaction.
- Latency: ~25–50ms (vs 5ms for single PutItem).
- Failures: All-or-nothing semantics. Validate conditions; roll back on failure.

**Backup & Disaster Recovery:**
- **On-Demand Backups:** Snapshot at point-in-time. Restore to new table with same or different name.
- **Point-in-Time Recovery (PITR):** Automatic backups kept for 35 days. Restore to any second within window.
- **Global Tables:** Multi-region replication. Read/write any region, sync eventually. RTO ~1 minute (promote replica on primary failure).

**Observability:**
- **CloudWatch Metrics:** `ConsumedReadCapacityUnits`, `ConsumedWriteCapacityUnits`, `UserErrors`, `ProvisionedThroughputExceeded`.
- **DynamoDB Streams:** Monitor stream consumer lag. High lag = processing slow, buildup.
- **AWS X-Ray:** Trace end-to-end request through app → DynamoDB. Identify slow queries.
- **DynamoDB Insights:** Recent console feature showing hot items, expensive queries.

---

### 3. ELASTICACHE (IN-MEMORY CACHE)

#### Concept Overview

**ElastiCache** is a managed in-memory data store supporting Redis and Memcached. Used to cache hot data, reduce database load, and enable real-time leaderboards and sessions.

**Problem solved:** Database queries are slow; cache results in memory for sub-millisecond retrieval. Offloads database reads.

**When to use:** Session stores, leaderboards, real-time analytics, rate limiting, full-page caching. Do NOT use as primary data store (no persistence by default).

#### Beginner Foundation

**Redis vs Memcached:**

| Aspect | Redis | Memcached |
|--------|-------|----------|
| **Data Types** | Strings, Lists, Sets, Sorted Sets, Hashes, Streams | Strings only |
| **Persistence** | RDB snapshots + AOF (append-only file) | None (memory-only) |
| **Replication** | Primary + Replicas, auto-failover | None (no HA) |
| **Lua Scripting** | Yes (atomic transactions) | No |
| **TTL/Expiration** | Yes | Yes |
| **Pub/Sub** | Yes (messaging) | No |
| **Memory Efficiency** | ~10–15% overhead | ~5–10% overhead |
| **Throughput** | ~100k ops/sec | ~1M ops/sec (simpler) |
| **Use Case** | Cache + Session + Leaderboards + Messaging | Simple cache, non-HA |

#### Intermediate Mechanics

**Redis Cluster Architecture:**

```
Mermaid:
graph LR
    App["Application"] -->|SET/GET| Cluster["Redis Cluster"]
    Cluster -->|3 Master nodes| M1["Master-1<br/>Shard 1"]
    Cluster -->|replicates to| R1["Replica-1"]
    M1 -->|hash slot 0-5460| Data1["Data Partition 1"]
    
    Cluster -->|3 Master nodes| M2["Master-2<br/>Shard 2"]
    Cluster -->|replicates to| R2["Replica-2"]
    M2 -->|hash slot 5461-10922| Data2["Data Partition 2"]
    
    Cluster -->|3 Master nodes| M3["Master-3<br/>Shard 3"]
    Cluster -->|replicates to| R3["Replica-3"]
    M3 -->|hash slot 10923-16383| Data3["Data Partition 3"]
```

**How it works:**
1. Application connects to Redis cluster endpoint.
2. Client library hashes the key, determines which shard (master) owns it.
3. Sends command to that shard.
4. Master processes write, replicates to its replica synchronously.
5. Returns response to app.
6. If master fails, replica promoted within ~30s. Resharding can add/remove shards.

**Cache Strategies:**

**Cache-Aside (Lazy Loading):**
```python
def get_user(user_id):
    # Check cache first
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # Cache miss, fetch from DB
    user = db.query(f"SELECT * FROM users WHERE id={user_id}")
    
    # Store in cache with TTL
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))  # 1 hour TTL
    return user
```
- Simple but can return stale data. Cache misses on cold start or expiry.

**Write-Through:**
```python
def update_user(user_id, data):
    # Update DB first
    db.update("users", data, where=f"id={user_id}")
    
    # Update cache
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
```
- Ensures cache consistency with DB. Slower writes.

**Write-Behind (Async Batch):**
```python
def update_user_async(user_id, data):
    # Update cache immediately (fast)
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
    
    # Queue DB write for batch processing
    queue.push({user_id, data})
    
# Batch writer (separate process)
def flush_to_db():
    while True:
        batch = queue.get_batch(size=100)
        db.batch_update(batch)
        time.sleep(5)
```
- Fast writes, risk of data loss if cache fails before DB sync.

**Terraform Example:**

```hcl
resource "aws_elasticache_replication_group" "redis_prod" {
  replication_group_description = "Production Redis Cluster"
  engine                         = "redis"
  engine_version                 = "7.0"
  node_type                      = "cache.r6g.xlarge"  # Memory-optimized
  num_cache_clusters             = 3  # 1 primary + 2 replicas (for high availability)
  
  automatic_failover_enabled = true
  
  port                 = 6379
  parameter_group_name = aws_elasticache_parameter_group.redis.name
  
  snapshot_retention_limit = 5  # days
  snapshot_window          = "03:00-05:00"  # UTC
  maintenance_window       = "sun:05:00-sun:07:00"
  
  security_group_ids = [aws_security_group.redis.id]
  
  transit_encryption_enabled = true
  auth_token                 = random_password.redis_auth.result  # Require password
  
  at_rest_encryption_enabled = true
  kms_key_id                 = aws_kms_key.elasticache.arn
  
  log_delivery_configuration {
    destination      = aws_cloudwatch_log_group.redis_slow.name
    destination_type = "cloudwatch-logs"
    log_format       = "json"
    log_type         = "slow-log"
  }
  
  tags = {
    Environment = "production"
  }
}

resource "aws_elasticache_parameter_group" "redis" {
  family      = "redis7"
  name        = "redis-prod-params"
  description = "Production Redis parameters"
  
  parameter {
    name  = "maxmemory-policy"
    value = "allkeys-lru"  # Evict LRU keys when full
  }
  
  parameter {
    name  = "timeout"
    value = "300"  # Client idle timeout (seconds)
  }
}
```

**CLI Monitoring:**

```bash
# Get cache node stats
aws elasticache describe-cache-nodes \
  --cache-cluster-id redis-prod-001 \
  --region us-east-1

# Get metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name CacheHits \
  --dimensions Name=CacheClusterId,Value=redis-prod-001 \
  --statistics Sum \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 300 \
  --region us-east-1

# Connect and test
redis-cli -h redis-prod.xxxxx.ng.0001.use1.cache.amazonaws.com -p 6379 --tls -a $AUTH_TOKEN ping
# Response: PONG (if connectivity works)
```

#### Advanced Engineering

**Redis Persistence:**
- **RDB (Snapshot):** Periodic full snapshot. Fast recovery, but can lose data between snapshots.
- **AOF (Append-Only File):** Log every command. Slower than RDB but better durability.
- **Hybrid:** RDB + AOF. On restart, load RDB then replay AOF.
- Trade-off: Durability vs performance. Production = AOF enabled. Dev = RDB only.

**Cluster Resharding:**
- Adding/removing shards redistributes hash slots across nodes.
- Online (no downtime) but moves data; can cause temporary slowdown.
- ~1GB/min throughput during resharding.

---

### 4. REDSHIFT (DATA WAREHOUSE)

#### Concept Overview

**Redshift** is a columnar, distributed data warehouse for analytics. Not a transactional DB.

**Problem solved:** Query terabytes of data in seconds. Structured for OLAP (analytics), not OLTP.

**When to use:** BI queries, log analysis, time-series analytics. Do NOT use for transactional workloads or <100GB data.

#### Beginner Foundation

**Redshift Cluster Architecture:**
- **Leader Node:** Coordinates query planning, execution.
- **Compute Nodes:** Execute queries in parallel. Each node has local SSD or managed storage.
- **Data Distribution:** By distribution key (hash), all rows, or range.
- **Replication:** Leader + compute nodes replicated across AZs.
- **Concurrency Scaling:** Auto-add nodes during heavy query load, scale down after.

**Columnar Storage:**
- Each column stored separately (vs row-store in RDBMS).
- Compression optimized per column. Queries reading few columns = fast.
- Example: 1 billion rows × 100 columns. Query on 2 columns = reads 2% of data (vs 100% in row-store).

---

## EVENT-DRIVEN ARCHITECTURE

### 1. AMAZON SNS (SIMPLE NOTIFICATION SERVICE)

#### Concept Overview

**SNS** is a pub-sub messaging service. Publisher sends message to topic; subscribers receive it.

**Problem solved:** Decouple components. One event triggers multiple downstream actions.

**When to use:** Notifications, alerts, fan-out to multiple consumers. Do NOT use for queuing (use SQS).

#### Beginner Foundation

**Architecture:**

```
Mermaid:
graph LR
    App["Application"] -->|Publish<br/>message| Topic["SNS Topic"]
    Topic -->|deliver| Sub1["Email Subscription"]
    Topic -->|deliver| Sub2["Lambda"]
    Topic -->|deliver| Sub3["SQS Queue"]
    Topic -->|deliver| Sub4["HTTP Endpoint"]
```

**How it works:**
1. Publisher sends message to SNS topic (ARN).
2. SNS delivers to all active subscriptions (email, Lambda, SQS, HTTP, etc.).
3. Subscribers process independently.
4. No queuing; messages not replayed if subscriber is down at publish time.

**Subscription Types:**
- **Email:** Manual confirmation, receives messages in inbox.
- **SMS:** Text message (charges apply).
- **SQS Queue:** Delivers to queue. Multiple consumers can dequeue.
- **Lambda:** Invokes function synchronously with message.
- **HTTP/HTTPS:** POST to webhook URL.
- **Application:** SNS pushes to mobile apps.

#### Intermediate Mechanics

**Message Filtering (SNS):**
```json
{
  "MessageStructure": "json",
  "Message": "{\"default\":\"Alert\",\"email\":\"Email body\",\"sqs\":\"SQS body\"}"
}
```
- Different message formats for different subscription types.
- Subscribers interpret `MessageStructure` to pick variant.

**Dead-Letter Queue (SNS → SQS):**
- SNS can deliver to SQS DLQ if SQS fails to accept message.
- Useful for retries and debugging.

**Terraform Example:**

```hcl
resource "aws_sns_topic" "alerts" {
  name = "app-alerts"
  
  tags = {
    Environment = "production"
  }
}

resource "aws_sns_topic_subscription" "email_alert" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = "ops@example.com"
}

resource "aws_sns_topic_subscription" "lambda_alert" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "lambda"
  endpoint  = aws_lambda_function.alert_handler.arn
}

# Allow SNS to invoke Lambda
resource "aws_lambda_permission" "allow_sns" {
  statement_id  = "AllowSNSInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.alert_handler.function_name
  principal     = "sns.amazonaws.com"
  source_arn    = aws_sns_topic.alerts.arn
}
```

---

### 2. AMAZON SQS (SIMPLE QUEUE SERVICE)

#### Concept Overview

**SQS** is a fully managed queue service. Producer sends message; consumers pull and delete.

**Problem solved:** Decouple producer from consumer. Handle burst traffic. Ensure messages are processed.

**When to use:** Background jobs, rate-limiting spikes, retries, batch processing. Do NOT use for real-time messaging (use SNS).

#### Beginner Foundation

**Queue Types:**

| Feature | Standard | FIFO |
|---------|----------|------|
| **Ordering** | Best-effort (may be out of order) | Guaranteed order (FIFO) |
| **Delivery** | At-least-once (duplicates possible) | Exactly-once (within 5-minute dedup window) |
| **Throughput** | Unlimited | 300 msg/sec (or 3000 with batching) |
| **Visibility Timeout** | 12 hours max | 12 hours max |
| **Cost** | Cheaper | ~50% more |

**Request Flow:**

```
Mermaid:
graph LR
    Producer["Producer"] -->|SendMessage| Queue["SQS Queue"]
    Queue -->|store| Messages["Messages<br/>Invisible until processed"]
    Consumer["Consumer"] -->|ReceiveMessage| Queue
    Queue -->|return message<br/>set VisibilityTimeout| Consumer
    Consumer -->|process| Process["Process message"]
    Consumer -->|DeleteMessage| Queue
    Queue -->|remove| Messages
```

**How it works:**
1. Producer sends message to queue URL.
2. Message stored in queue (3–5 copies across AZs for durability).
3. Consumer polls queue, receives up to 10 messages.
4. Message becomes "invisible" for `VisibilityTimeout` seconds (default 30s).
5. Consumer processes message, calls `DeleteMessage`.
6. If consumer crashes/timeout before delete, message reappears after timeout.
7. Other consumers can then process it (at-least-once semantics).

#### Intermediate Mechanics

**Configuration:**

```hcl
resource "aws_sqs_queue" "background_jobs" {
  name                       = "background-jobs"
  visibility_timeout_seconds = 300  # 5 minutes
  message_retention_seconds  = 1209600  # 14 days
  max_message_size           = 262144  # 256 KB
  
  # Delay messages before they appear to consumers
  delay_seconds = 0  # 0–900 seconds
  
  # Enable long polling (reduce empty polls)
  receive_wait_time_seconds = 10  # Waits up to 10s for message
  
  # FIFO configuration
  fifo_queue                  = false
  content_based_deduplication = false
  
  # Redrive policy (dead-letter queue)
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.dlq.arn
    maxReceiveCount     = 3  # After 3 failures, move to DLQ
  })
  
  tags = {
    Environment = "production"
  }
}

resource "aws_sqs_queue" "dlq" {
  name                      = "background-jobs-dlq"
  message_retention_seconds = 1209600  # 14 days (for debugging)
}
```

**Consumer Patterns:**

```python
import boto3
import json
import time

sqs = boto3.client('sqs')
queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789/background-jobs'

def long_poll_consumer():
    """Consume messages with long polling."""
    while True:
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=10,  # Long poll—wait up to 10s for message
            VisibilityTimeout=300
        )
        
        messages = response.get('Messages', [])
        if not messages:
            print("No messages, waiting...")
            continue
        
        for msg in messages:
            try:
                body = json.loads(msg['Body'])
                print(f"Processing: {body}")
                
                # Process message
                process_job(body)
                
                # Delete on success
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=msg['ReceiptHandle']
                )
            except Exception as e:
                print(f"Error: {e}")
                # Don't delete; message reappears after VisibilityTimeout
                # Consumer rejects it; eventually moved to DLQ after maxReceiveCount

def process_job(job):
    # Application logic
    pass

if __name__ == "__main__":
    long_poll_consumer()
```

---

### 3. AMAZON EVENTBRIDGE (EVENT ROUTING)

#### Concept Overview

**EventBridge** is a serverless event bus. Routes events from sources to targets based on rules.

**Problem solved:** Decouple event producers from consumers. Route events to multiple targets. Pattern matching.

**When to use:** Scheduled tasks, cross-service event routing, third-party SaaS integrations. Do NOT use for message queuing (use SQS/SNS).

#### Beginner Foundation

**Architecture:**

```
Mermaid:
graph LR
    Sources["Event Sources<br/>EC2, S3, Custom App"] -->|Put event| Bus["EventBridge Bus"]
    Bus -->|Rule 1: pattern match| Target1["Lambda"]
    Bus -->|Rule 2: pattern match| Target2["SNS Topic"]
    Bus -->|Rule 3: pattern match| Target3["SQS Queue"]
    Bus -->|Rule 4: schedule| Target4["Scheduled Task"]
```

**How it works:**
1. Event source publishes event to bus (or use `PutEvents` API).
2. EventBridge evaluates rules (JSON pattern matching).
3. For matching rules, sends event to targets (Lambda, SNS, SQS, HTTP, etc.).
4. Targets invoked asynchronously. Retries if they fail.
5. Dead-letter queue for failed deliveries.

#### Intermediate Mechanics

**Rule Pattern Matching:**

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["running"],
    "instance-type": ["t3.medium", "t3.large"]
  }
}
```
- Matches EC2 instance state changes to running for t3.medium/large instances.

**Terraform Example:**

```hcl
resource "aws_cloudwatch_event_rule" "ec2_state_change" {
  name        = "ec2-state-change-rule"
  description = "Trigger on EC2 state change"
  
  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance State-change Notification"]
    detail = {
      state = ["running", "stopped"]
    }
  })
}

resource "aws_cloudwatch_event_target" "lambda" {
  rule      = aws_cloudwatch_event_rule.ec2_state_change.name
  target_id = "EC2ChangeHandler"
  arn       = aws_lambda_function.ec2_handler.arn
  
  dead_letter_config {
    arn = aws_sqs_queue.eventbridge_dlq.arn
  }
}

# Allow EventBridge to invoke Lambda
resource "aws_lambda_permission" "allow_eventbridge" {
  statement_id  = "AllowEventBridgeInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.ec2_handler.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.ec2_state_change.arn
}

# Scheduled rule (cron)
resource "aws_cloudwatch_event_rule" "daily_cleanup" {
  name                = "daily-cleanup"
  description         = "Run cleanup job daily"
  schedule_expression = "cron(0 2 * * ? *)"  # 2 AM UTC daily
}

resource "aws_cloudwatch_event_target" "cleanup_lambda" {
  rule      = aws_cloudwatch_event_rule.daily_cleanup.name
  target_id = "CleanupTask"
  arn       = aws_lambda_function.cleanup.arn
}
```

---

### 4. AMAZON KINESIS (STREAMING DATA)

#### Concept Overview

**Kinesis** processes high-volume streaming data in real-time. Data Streams for ingestion; Firehose for delivery.

**Problem solved:** Ingest millions of events/sec, real-time analytics, dashboards.

**When to use:** IoT sensors, clickstreams, log aggregation, real-time fraud detection. Do NOT use for durable queuing (use SQS).

#### Beginner Foundation

**Kinesis Data Streams:**
- **Shard:** Unit of capacity. 1 shard = 1000 writes/sec, 1MB/sec, 2MB/sec reads.
- **Partition Key:** Determines which shard. Hash-based distribution.
- **Sequence Number:** Unique per shard. Used for exactly-once processing.
- **Record:** Data blob + partition key + sequence number.
- **Consumer:** Application reading from stream. Multiple consumers independently.

**Request Flow:**

```
Mermaid:
graph LR
    Producer["IoT Device"] -->|PutRecord<br/>key=device-1| Shard1["Shard 1"]
    Producer2["IoT Device 2"] -->|PutRecord<br/>key=device-2| Shard2["Shard 2"]
    Producer3["IoT Device 3"] -->|PutRecord<br/>key=device-3| Shard1
    
    Shard1 -->|store record| Stream["Kinesis Stream<br/>Replicated 3x"]
    Shard2 -->|store record| Stream
    
    Consumer1["Lambda Consumer"] -->|GetRecords| Shard1
    Consumer2["Lambda Consumer"] -->|GetRecords| Shard2
```

**Key Properties:**
- **Ordering:** Guaranteed within shard (by sequence number). Not across shards.
- **Retention:** 24 hours default, up to 365 days (extra cost).
- **Scaling:** Manual or auto-scaling based on traffic.
- **Cost:** Per shard per hour + data ingestion/retrieval charges.

#### Intermediate Mechanics

**Kinesis Firehose:**
- Simpler than Kinesis Streams. Delivers data to S3, Redshift, Elasticsearch, Splunk.
- Automatic buffering and compression.
- No manual shard management.
- ~5 minute delivery latency.

**Terraform Example:**

```hcl
resource "aws_kinesis_stream" "events" {
  name            = "app-events"
  retention_period = 24  # hours
  
  stream_mode_details {
    stream_mode = "ON_DEMAND"  # Auto-scaling, pay per record
  }
  
  tags = {
    Environment = "production"
  }
}

# Firehose to S3
resource "aws_kinesis_firehose_delivery_stream" "s3_delivery" {
  name            = "events-to-s3"
  destination     = "extended_s3"
  s3_configuration {
    role_arn   = aws_iam_role.firehose_role.arn
    bucket_arn = aws_s3_bucket.analytics.arn
    
    prefix              = "events/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/"
    error_output_prefix = "errors/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/!{firehose:error-output-type}"
    
    buffering_size     = 128  # MB
    buffering_interval = 300  # seconds
    
    compression_format = "GZIP"
  }
}
```

---

## INTERVIEW QUESTIONS

### Question: Explain the consistency guarantees of RDS Multi-AZ vs Read Replicas.

**What the interviewer is testing:** Understanding of synchronous vs asynchronous replication, RPO/RTO, and trade-offs.

**Strong answer:** RDS Multi-AZ provides **strong consistency (RPO=0)** via synchronous replication. Writes to the primary are copied to the standby before acknowledging to the application. This guarantees no data loss but adds ~5–10ms latency per write.

Read Replicas use **asynchronous replication**, so the replica may lag by 100ms to several seconds. Reads from replicas may return stale data. **RPO** is minutes (replica lag). Multi-AZ is for **high availability** (automatic failover, HA). Read Replicas are for **read scaling and DR** (manual promotion, different region possible).

**How it works:**
1. **Multi-AZ Write Path:** App sends write → Primary writes to log + EBS → Primary replicates to Standby's log synchronously → Standby acks → Primary acks to app. If Primary fails, RDS detects (~60s), promotes Standby. DNS updates. App reconnects.
2. **Read Replica Async Path:** Primary writes → app ack returns immediately → Primary asynchronously sends write to Replica. Replica applies at its own pace. Reads from Replica may see uncommitted writes from other replicas or miss very recent writes.

**Example:** Production ecommerce checkout. Use Multi-AZ for transactional DB (orders, payments) to prevent data loss. Use Read Replicas for reporting queries (analytics, leaderboards) to offload reads from primary.

**Trade-offs and alternatives:**
- **Cost:** Multi-AZ = double DB cost (standby always running). Read Replicas = additional cost only if reads are high (justify ROI).
- **Failover Speed:** Multi-AZ ~60–120s. Read Replica promotion = manual (~5–10 min) or Lambda-orchestrated.
- **Data Freshness:** Multi-AZ = strong. Read Replicas = stale. If app needs reads to see writes immediately, use Multi-AZ only.
- **Cross-Region:** Multi-AZ = same region only. Read Replicas = cross-region (for disaster recovery).

**Common mistakes:**
- Thinking Multi-AZ read replicas can serve reads (they can't—standby is idle).
- Using Read Replicas for HA (they won't auto-promote; promote is manual).
- Not accounting for failover lag; expecting instant switchover.

**Likely follow-ups:**
1. How does RDS handle split-brain when primary is alive but network is down? → RDS has cluster membership protocol; worse case is transient split-brain but membership ensures one primary wins.
2. Can you promote a Read Replica while the primary is running? → Yes, but data is not synced after promotion. App must handle stale reads or re-sync.
3. What's the cost difference between Multi-AZ and Read Replicas? → Multi-AZ ~50% more (extra compute). Read Replicas + data transfer extra.

---

### Question: How would you design a caching strategy for a high-traffic session store using DynamoDB and ElastiCache?

**What the interviewer is testing:** Knowledge of CAP theorem, consistency, caching patterns, failover.

**Strong answer:** Use a **hybrid cache** with ElastiCache (L1 hot cache) in front of DynamoDB (L2 durable source).

**Cache-Aside Pattern:**
1. App checks Redis for session (by session ID).
2. If hit, return immediately (sub-millisecond).
3. If miss, query DynamoDB (with strong read).
4. App writes session back to Redis with TTL (e.g., 1 hour).
5. DynamoDB also stores session (write-through), but only after Redis cache succeeds.

**How it works:**
- **Read Path:** Redis (1ms) → DynamoDB (strong read ~10ms).
- **Write Path:** Redis (immediate, short TTL) + DynamoDB (durable, longer retention).
- **Failover:** If Redis fails, app falls back to DynamoDB directly (slower, but no data loss).
- **Consistency:** Sessions in Redis are eventually consistent (not in Redis = read from DynamoDB). DynamoDB is source of truth.

**Example Implementation (Python):**

```python
import json, redis, boto3, time
from datetime import datetime, timedelta

redis_client = redis.Redis(host='elasticache-endpoint', port=6379, decode_responses=True)
dynamodb = boto3.resource('dynamodb')
sessions_table = dynamodb.Table('sessions')

def get_session(session_id):
    # L1: Redis cache
    cached_session = redis_client.get(f"session:{session_id}")
    if cached_session:
        print(f"Cache hit: {session_id}")
        return json.loads(cached_session)
    
    # L2: DynamoDB
    print(f"Cache miss: {session_id}")
    try:
        response = sessions_table.get_item(
            Key={'SessionID': session_id},
            ConsistentRead=True  # Strong read
        )
        session = response.get('Item')
        if session:
            # Repopulate cache with 1-hour TTL
            redis_client.setex(
                f"session:{session_id}",
                3600,
                json.dumps(session)
            )
            return session
    except Exception as e:
        print(f"DynamoDB error: {e}")
        # Fallback: return None, handle gracefully
    
    return None

def update_session(session_id, data):
    # Write-through: update both
    
    # L2: DynamoDB first (source of truth)
    try:
        sessions_table.put_item(
            Item={
                'SessionID': session_id,
                'Data': json.dumps(data),
                'ExpirationTime': int((datetime.utcnow() + timedelta(days=7)).timestamp()),
                'LastUpdated': datetime.utcnow().isoformat()
            }
        )
        print(f"Session stored in DynamoDB: {session_id}")
    except Exception as e:
        print(f"DynamoDB write failed: {e}")
        # Don't return; ensure cache is also updated
    
    # L1: Redis (1-hour TTL)
    redis_client.setex(
        f"session:{session_id}",
        3600,
        json.dumps({
            'SessionID': session_id,
            'Data': data,
            'LastUpdated': datetime.utcnow().isoformat()
        })
    )
    print(f"Session cached in Redis: {session_id}")

def delete_session(session_id):
    # Delete from both
    redis_client.delete(f"session:{session_id}")
    sessions_table.delete_item(Key={'SessionID': session_id})
```

**Architecture Diagram:**

```
Mermaid:
graph LR
    App["Application"] -->|GET session:ID| Redis["ElastiCache<br/>Redis"]
    Redis -->|hit| CacheHit["Return session<br/>1ms latency"]
    Redis -->|miss| DynamoDB["DynamoDB<br/>Table"]
    DynamoDB -->|return| FallbackRead["Return session<br/>10ms latency"]
    FallbackRead -->|write back| Redis
    
    App -->|UPDATE session| Redis2["Redis<br/>setex"]
    Redis2 -->|OK| DynamoDB2["DynamoDB<br/>put_item"]
    DynamoDB2 -->|OK| Success["Session persisted"]
```

**Trade-offs and alternatives:**
- **Cache Consistency:** With TTL, cache can be stale. Use explicit cache invalidation (delete Redis key) on user logout/update.
- **Session Affinity:** Sticky sessions to single server can reduce cache misses but complicate scaling.
- **Lambda Sessions:** If Lambda functions are stateless, session ID should be unique per client, stored externally (Redis + DynamoDB).
- **Eventual Consistency (Memcached):** If you don't need PITR, Memcached is cheaper; no persistence. Loss on node failure (but cluster spread across AZs mitigates).

**Common mistakes:**
- Using only Redis without DynamoDB → no durability, sessions lost on cluster failure.
- Writing to DynamoDB but not Redis → defeats cache purpose; every read hits DB.
- Not setting TTL on Redis → accumulates expired sessions; memory fills up.
- Assuming Redis and DynamoDB always in sync → they diverge. Code must handle stale reads.

**Likely follow-ups:**
1. How do you handle session invalidation? → Delete key from Redis + mark as invalid in DynamoDB (faster for subsequent checks).
2. What if ElastiCache node fails? → App falls back to DynamoDB; throughput increases but reads are slower.
3. How do you monitor cache hit rate? → CloudWatch `CacheHits` / (`CacheHits` + `CacheMisses`). Target >80% for efficient caching.

---

## TROUBLESHOOTING SCENARIOS

### Scenario 1: RDS CPU Utilization Spike & Slow Queries

**Symptom:**
- RDS CPU hits 90%+ for 30 minutes.
- Application reports timeouts in database queries.
- CloudWatch `DatabaseConnections` rising.

**Investigation Steps:**

1. **Verify instance class and capacity:**
   ```bash
   aws rds describe-db-instances --db-instance-identifier prod-postgres \
     --query 'DBInstances[0].[DBInstanceClass,AllocatedStorage,Iops]' \
     --region us-east-1
   
   # Output: db.r6i.xlarge, 1000 GB, 5000 IOPS
   # ✓ Confirm this is sufficient for workload
   ```

2. **Check slow query log:**
   ```bash
   aws logs tail /aws/rds/instance/prod-postgres/postgresql \
     --follow --since 30m
   
   # Look for log lines like: "Query took 5000ms" (slow)
   ```

3. **Query Performance Insights for wait events:**
   ```bash
   aws pi get-resource-metrics \
     --period-in-seconds 60 \
     --metric-queries file://pi-query.json \
     --service-type RDS \
     --identifier <DBResourceId> \
     --start-time 2024-01-01T12:00:00Z \
     --end-time 2024-01-01T13:00:00Z \
     --region us-east-1
   
   # JSON query:
   # {
   #   "Metric": "db.load.avg",
   #   "GroupBy": {"Group": "wait_event_type"}
   # }
   ```

4. **Analyze top queries:**
   ```bash
   # Enable query logging
   aws rds modify-db-parameter-group \
     --db-parameter-group-name prod-postgres-params \
     --parameters "ParameterName=log_statement,ParameterValue=all,ApplyMethod=immediate" \
     --region us-east-1
   
   # Check logs for long-running queries (> 1 second)
   aws logs filter-log-events \
     --log-group-name /aws/rds/instance/prod-postgres/postgresql \
     --filter-pattern "duration:" \
     --region us-east-1 | jq '.events[] | select(.message | contains("5000"))' # 5+ second queries
   ```

**Plausible Causes (Decision Tree):**

```
Is CPU high continuously (not spiky)?
├─ YES → Problem: Sustained heavy query load
│   ├─ Missing index? → Check query plans: EXPLAIN ANALYZE
│   ├─ Table scan? → Add index on WHERE clause columns
│   └─ Lock contention? → Check pg_stat_activity for blocking queries
│
├─ NO (spiky) → Problem: Burst queries or connection pool exhaustion
    ├─ Too many connections? → Implement RDS Proxy connection pooling
    ├─ Connection leak? → Restart application, check conn close logic
    └─ Scheduled job? → Defer heavy queries to off-peak window
```

**Root Cause Identification:**

Most likely: **Missing index on frequently queried column.**

```bash
# Check for missing indexes
psql -h prod-postgres.xxxx.rds.amazonaws.com -U adminuser -d appdb << 'SQL'
SELECT schemaname, tablename, indexname
FROM pg_indexes
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY tablename;
SQL
# If index is missing on `user_id` in `orders` table, that's the culprit.

# Create index
psql -h prod-postgres.xxxx.rds.amazonaws.com -U adminuser -d appdb << 'SQL'
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
SQL
# CONCURRENTLY = no lock on table during index creation
```

**Fix:**

1. **Add missing index:**
   ```terraform
   # Terraform
   resource "aws_db_instance" "postgres" {
     # ... existing config ...
     # Note: Can't directly manage indexes via Terraform; use provisioner
     provisioner "local-exec" {
       command = <<-EOT
         psql -h ${aws_db_instance.postgres.address} \
              -U ${var.db_username} \
              -d appdb \
              -c "CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id)"
       EOT
     }
   }
   ```

2. **Implement RDS Proxy for connection pooling:**
   ```terraform
   resource "aws_db_proxy" "postgres_proxy" {
     name                   = "prod-postgres-proxy"
     engine_family          = "POSTGRESQL"
     auth {
       auth_scheme = "SECRETS"
       secret_arn  = aws_secretsmanager_secret.db_password.arn
     }
     role_arn               = aws_iam_role.proxy_role.arn
     db_proxy_endpoints {
       db_proxy_endpoint_identifier = "prod-read-write"
       db_proxy_endpoint_type       = "READ_WRITE"
       vpc_subnet_ids               = [aws_subnet.private_1.id, aws_subnet.private_2.id]
     }
     max_connections = 100  # Reuse 100 connections
     session_timeout = 900  # 15 minutes
   }
   
   resource "aws_db_proxy_target" "postgres_target" {
     db_proxy_name           = aws_db_proxy.postgres_proxy.name
     target_arn              = aws_db_instance.postgres.arn
     db_parameter_group_name = "default.postgres15"
   }
   ```

3. **Scale instance class if needed:**
   ```bash
   aws rds modify-db-instance \
     --db-instance-identifier prod-postgres \
     --db-instance-class db.r6i.2xlarge \
     --apply-immediately \
     --region us-east-1
   
   # Wait ~5–10 minutes for instance to scale
   aws rds describe-db-instances \
     --db-instance-identifier prod-postgres \
     --query 'DBInstances[0].DBInstanceStatus' \
     --region us-east-1
   # Response: "available" (when ready)
   ```

**Validation:**
```bash
# Monitor metrics post-fix
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=prod-postgres \
  --statistics Average \
  --start-time 2024-01-01T13:00:00Z \
  --end-time 2024-01-01T14:00:00Z \
  --period 300 \
  --region us-east-1

# Expected: CPU drops to <30% after index creation
```

**Prevention:**
- Enable Performance Insights by default.
- Review query plans for new schemas (dev environment).
- Auto-vacuum configuration tuned for workload.
- Alert if `DatabaseConnections` > threshold.

---

### Scenario 2: DynamoDB Throttling (ProvisionedThroughputExceededException)

**Symptom:**
- Requests return HTTP 400 with message `ProvisionedThroughputExceededException`.
- Visible in application logs around 14:00 UTC.
- CloudWatch `UserErrors` metric spikes.
- Error rate increases for 5–10 minutes then recovers.

**Investigation Steps:**

1. **Check provisioned capacity and consumed capacity:**
   ```bash
   aws dynamodb describe-table --table-name Orders --region us-east-1 | \
     jq '.Table | {BillingModeSummary, ProvisionedThroughput}'
   
   # Output:
   # {
   #   "BillingModeSummary": {"BillingMode": "PROVISIONED"},
   #   "ProvisionedThroughput": {"ReadCapacityUnits": 100, "WriteCapacityUnits": 50}
   # }
   ```

2. **Check CloudWatch metrics for consumed capacity:**
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/DynamoDB \
     --metric-name ConsumedWriteCapacityUnits \
     --dimensions Name=TableName,Value=Orders \
     --statistics Sum \
     --start-time 2024-01-01T13:30:00Z \
     --end-time 2024-01-01T14:30:00Z \
     --period 60 \
     --region us-east-1 | jq '.Datapoints | sort_by(.Timestamp)[]'
   
   # Expected: Sum per minute should be ≤ provisioned capacity (50 WCU/min = 50 × 60 = 3000 writes/min)
   # If consumed > provisioned, throttling occurred.
   ```

3. **Identify hot partition (uneven load):**
   ```bash
   # No direct "hot partition" metric in CloudWatch
   # Use DynamoDB Streams or application logs to trace which partition key (UserID) causes load
   
   # Enable DynamoDB Streams:
   aws dynamodb update-table \
     --table-name Orders \
     --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
     --region us-east-1
   
   # Query stream for top partition keys (via Lambda or application)
   # If one UserID drives 80% of traffic, that's a hot partition.
   ```

**Plausible Causes:**

```
Did throttling occur at a specific time (scheduled batch)?
├─ YES → Cause: Batch operation or scheduled export
│   ├─ Nightly report job? → Reschedule to off-peak or use on-demand billing
│   ├─ Backup/export? → Stagger queries or use on-demand mode
│   └─ Test load? → Run tests on isolated table
│
├─ NO (random) → Cause: Uneven traffic or hot partition
    ├─ One user hammering? → Implement request rate limiting (client-side)
    ├─ VIP user heavy load? → Use separate table or add RCU/WCU
    └─ Spike in regular traffic → Upgrade provisioned capacity
```

**Root Cause Identification:**

Most likely: **Batch export job running at 14:00 UTC, consuming all write capacity.**

```bash
# Check application logs at 14:00
grep -i "14:00" /var/log/app.log | grep -i "dynamodb\|table" | head -20

# Expected pattern:
# 14:00:05 Starting nightly export Orders table to S3
# 14:00:10 Scanning 5 million rows
# 14:00:15 ERROR: ProvisionedThroughputExceededException
```

**Fix:**

1. **Increase provisioned capacity temporarily or convert to on-demand:**
   ```bash
   # Option A: Increase WCU
   aws dynamodb update-table \
     --table-name Orders \
     --provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=200 \
     --region us-east-1
   
   # Wait for update
   aws dynamodb describe-table --table-name Orders --region us-east-1 | \
     jq '.Table.TableStatus'
   # Response: "UPDATING" then "ACTIVE"
   ```

2. **Switch to on-demand billing (if traffic is spiky):**
   ```terraform
   resource "aws_dynamodb_table" "orders" {
     name         = "Orders"
     billing_mode = "PAY_PER_REQUEST"  # On-demand, no throttling
     hash_key     = "OrderID"
     range_key    = "UserID"
     
     attribute {
       name = "OrderID"
       type = "S"
     }
     
     attribute {
       name = "UserID"
       type = "S"
     }
   }
   ```

3. **Optimize batch job (reschedule and paginate):**
   ```python
   import boto3
   from datetime import datetime, time
   
   dynamodb = boto3.client('dynamodb')
   
   def export_orders_optimized():
       """Export with pagination and off-peak scheduling."""
       now = datetime.now().time()
       
       # Only run between 02:00–04:00 UTC (off-peak)
       if not (time(2, 0) < now < time(4, 0)):
           print("Export only runs 02:00–04:00 UTC")
           return
       
       # Paginate scan with small page size
       paginator = dynamodb.get_paginator('scan')
       page_iterator = paginator.paginate(
           TableName='Orders',
           PaginationConfig={
               'PageSize': 100,  # Small batches
               'MaxItems': 5000000
           }
       )
       
       for page in page_iterator:
           items = page.get('Items', [])
           # Write to S3 in batches
           write_to_s3(items)
   
   def write_to_s3(items):
       import json
       s3 = boto3.client('s3')
       s3.put_object(
           Bucket='analytics-bucket',
           Key=f'orders-export-{datetime.now().isoformat()}.json',
           Body=json.dumps(items)
       )
   ```

**Validation:**
```bash
# Re-run export and verify no throttling
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name UserErrors \
  --dimensions Name=TableName,Value=Orders \
  --statistics Sum \
  --start-time 2024-01-02T02:00:00Z \
  --end-time 2024-01-02T04:00:00Z \
  --period 60 \
  --region us-east-1 | jq '.Datapoints'

# Expected: No datapoints or Sum=0 (no errors)
```

**Prevention:**
- Use CloudWatch alarms on `UserErrors` metric.
- Set auto-scaling on provisioned tables (target utilization 70%).
- Use on-demand for unpredictable workloads.
- Implement exponential backoff + retry in application (SDK handles this by default).

---

## SYSTEM DESIGN EXAMPLES

### Design a Real-Time Leaderboard System on AWS

**Functional Requirements:**
- Display top 100 players globally by score.
- Update player score in real-time (<100ms latency).
- Query player rank (e.g., "What's my rank?") in <50ms.
- ~100k concurrent users, ~1M score updates/second peak.

**Non-Functional Requirements:**
- 99.99% availability.
- Support global regions (NA, EU, APAC).
- Cost-effective at scale.
- Exact consistency (no stale leaderboard).

**Capacity Estimation:**

| Component | Calculation | Value |
|-----------|-------------|-------|
| **Writes/sec** | Peak = 1M updates/sec | 1M writes/sec |
| **Reads/sec** | Per user checks rank 10x/min, 100k users | ~16.7k reads/sec |
| **Leaderboard size** | Top 100 + user's rank + surroundings | ~200 items per query |
| **Data size** | UserID (20B) + Score (8B) + Timestamp (8B) per entry | 36B per entry |
| **Daily storage** | Rank history per user per day | ~100GB daily (1M users × 100B) |

**Architecture:**

```
Mermaid:
graph LR
    GameClients["100k Players"] -->|score update<br/>1M/sec| RateLimiter["Rate Limiter<br/>SQS/Kinesis"]
    RateLimiter -->|batch| Lambda["Lambda<br/>Batch Processor"]
    Lambda -->|write sorted set| Redis["Redis Cluster<br/>Sorted Sets<br/>ZADD user:scores"]
    Lambda -->|async persist| DynamoDB["DynamoDB<br/>Leaderboard history"]
    
    GameClients -->|query rank<br/>16k/sec| CacheL["ElastiCache<br/>GET top 100"]
    CacheL -->|cache hit| Return["Return<br/>50ms"]
    CacheL -->|miss| Redis2["Redis<br/>ZRANK, ZRANGE"]
    Redis2 -->|return rank| CacheL
```

**API Design:**

```
1. UpdateScore(UserID, Score, Timestamp)
   - Atomically update user score in Redis sorted set.
   - Return new rank.
   - Input: {"user_id": "user-123", "score": 5000}
   - Output: {"rank": 42, "new_score": 5000, "timestamp": "2024-01-01T12:00:00Z"}

2. GetLeaderboard(Limit=100)
   - Return top N players.
   - Input: {"limit": 100}
   - Output: [{"rank": 1, "user_id": "user-456", "score": 100000}, ...]

3. GetPlayerRank(UserID)
   - Return player's current rank and score.
   - Input: {"user_id": "user-123"}
   - Output: {"rank": 42, "score": 5000, "top_100": [<top 100 list>]}
```

**Data Flow:**

1. **Write Path (UpdateScore):**
   ```
   Player sends score update
   → SQS queue (buffer burst)
   → Lambda (batch 100 updates, 100ms window)
   → Redis ZADD (atomic, O(log N))
   → DynamoDB PutItem (async, for history)
   → Response to client (rank)
   ```

2. **Read Path (GetPlayerRank):**
   ```
   Player requests rank
   → Check Redis cache (top 100 + 10 around player)
   → If in top 100: return from cache
   → If not: ZRANK on Redis (exact rank)
   → Cache result with 30s TTL
   → Response to client
   ```

**Implementation (Terraform + Python):**

```hcl
# Redis Cluster for leaderboard
resource "aws_elasticache_replication_group" "leaderboard" {
  replication_group_description = "Leaderboard sorted sets"
  engine                         = "redis"
  engine_version                 = "7.0"
  node_type                      = "cache.r6g.xlarge"
  num_cache_clusters             = 3
  automatic_failover_enabled     = true
  
  parameter_group_name = "default.redis7"
  port                 = 6379
  
  tags = {
    Name = "leaderboard-redis"
  }
}

# SQS for write buffering
resource "aws_sqs_queue" "score_updates" {
  name                       = "score-updates"
  visibility_timeout_seconds = 30
  message_retention_seconds  = 3600
  receive_wait_time_seconds  = 10
}

# Lambda to process batches
resource "aws_lambda_function" "batch_processor" {
  filename      = "batch_processor.zip"
  function_name = "leaderboard-batch-processor"
  role          = aws_iam_role.lambda_role.arn
  handler       = "index.handler"
  
  timeout       = 30
  memory_size   = 3008  # For Redis connection pool
  
  vpc_config {
    subnet_ids         = [aws_subnet.private_1.id, aws_subnet.private_2.id]
    security_group_ids = [aws_security_group.lambda.id]
  }
  
  environment {
    variables = {
      REDIS_ENDPOINT = aws_elasticache_replication_group.leaderboard.configuration_endpoint_address
      REDIS_PORT     = "6379"
      DYNAMODB_TABLE = aws_dynamodb_table.leaderboard_history.name
    }
  }
}

# Trigger Lambda from SQS every 100ms or 100 messages
resource "aws_lambda_event_source_mapping" "sqs_trigger" {
  event_source_arn  = aws_sqs_queue.score_updates.arn
  function_name     = aws_lambda_function.batch_processor.function_name
  batch_size        = 100
  maximum_batching_window_in_seconds = 1  # Batch every 1 second max
}

# DynamoDB for history
resource "aws_dynamodb_table" "leaderboard_history" {
  name           = "leaderboard-history"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "UserID"
  range_key      = "Timestamp"
  
  attribute {
    name = "UserID"
    type = "S"
  }
  
  attribute {
    name = "Timestamp"
    type = "N"
  }
  
  ttl {
    attribute_name = "ExpirationTime"
    enabled        = true  # Auto-delete after 30 days
  }
}
```

**Python Implementation:**

```python
import json
import redis
import boto3
import time
from datetime import datetime, timedelta

redis_client = redis.Redis(host='leaderboard.xxxxx.ng.0001.use1.cache.amazonaws.com', port=6379)
dynamodb = boto3.resource('dynamodb')
sqs = boto3.client('sqs')
leaderboard_table = dynamodb.Table('leaderboard-history')
queue_url = 'https://sqs.us-east-1.amazonaws.com/123456/score-updates'

def update_score(user_id, score):
    """
    Atomically update score in Redis sorted set.
    Returns new rank.
    """
    # ZADD: add or update score in sorted set
    # Sorted sets ordered by score (descending)
    redis_client.zadd('leaderboard', {user_id: score})
    
    # Get rank (0-indexed, so add 1)
    rank = redis_client.zrevrank('leaderboard', user_id)  # Reverse rank (highest first)
    
    return {'rank': rank + 1 if rank is not None else None, 'score': score}

def get_leaderboard(limit=100):
    """Get top N players."""
    # ZREVRANGE: get top N in descending score order
    top_players = redis_client.zrevrange('leaderboard', 0, limit - 1, withscores=True)
    
    leaderboard = [
        {
            'rank': idx + 1,
            'user_id': user_id,
            'score': int(score)
        }
        for idx, (user_id, score) in enumerate(top_players)
    ]
    
    return leaderboard

def get_player_rank(user_id):
    """Get player's rank and surrounding context."""
    # Get player's score
    score = redis_client.zscore('leaderboard', user_id)
    if score is None:
        return {'rank': None, 'score': None}
    
    rank = redis_client.zrevrank('leaderboard', user_id)
    
    # Get top 100 for context
    top_100 = get_leaderboard(100)
    
    # Get players around this user (±5)
    context_start = max(0, rank - 5) if rank is not None else 0
    context = redis_client.zrevrange('leaderboard', context_start, context_start + 10, withscores=True)
    
    return {
        'rank': rank + 1 if rank is not None else None,
        'score': int(score),
        'top_100': top_100,
        'context': context
    }

def lambda_handler(event, context):
    """Process batch of score updates from SQS."""
    # event = {'Records': [SQS message 1, SQS message 2, ...]}
    
    updates = []
    for record in event['Records']:
        try:
            body = json.loads(record['Body'])
            user_id = body['user_id']
            score = body['score']
            timestamp = body.get('timestamp', int(time.time()))
            
            # Update Redis
            result = update_score(user_id, score)
            
            # Persist to DynamoDB asynchronously
            updates.append({
                'UserID': user_id,
                'Timestamp': timestamp,
                'Score': score,
                'Rank': result['rank'],
                'ExpirationTime': int((datetime.utcnow() + timedelta(days=30)).timestamp())
            })
            
            # Delete from SQS (after successful processing)
            sqs.delete_message(
                QueueUrl=queue_url,
                ReceiptHandle=record['ReceiptHandle']
            )
        except Exception as e:
            print(f"Error processing record: {e}")
    
    # Batch write to DynamoDB
    if updates:
        with leaderboard_table.batch_writer(
            batch_size=25,
            overwrite_by_pkeys=['UserID', 'Timestamp']
        ) as batch:
            for item in updates:
                batch.put_item(Item=item)
    
    return {
        'statusCode': 200,
        'body': json.dumps({'processed': len(updates)})
    }
```

**Failure Handling:**

1. **Redis Node Failure:**
   - Cluster replicas auto-promote. Client library reconnects transparently.
   - Brief spike in latency (~1–2 sec).

2. **DynamoDB Failure:**
   - Async write; doesn't block score update.
   - Retry with exponential backoff (DynamoDB SDK handles).
   - Score still in Redis; history loss is acceptable (leaderboard still serves).

3. **Lambda Timeout:**
   - Batch partially processed. Unprocessed messages remain in SQS queue.
   - Retry (SQS default behavior).

**Multi-Region Design:**

```
Mermaid:
graph LR
    NAClients["NA Players"] -->|writes| NARateLimiter["NA SQS"]
    EUClients["EU Players"] -->|writes| EURateLimiter["EU SQS"]
    
    NARateLimiter -->|Lambda| NARedis["NA Redis<br/>Regional leader"]
    EURateLimiter -->|Lambda| EURedis["EU Redis<br/>Regional leader"]
    
    NARedis -->|eventual sync| GlobalRedis["Global Redis<br/>Read-only"]
    EURedis -->|eventual sync| GlobalRedis
    
    NAClients -->|read global<br/>rank| GlobalRedis
    EUClients -->|read global<br/>rank| GlobalRedis
```

**Cost Optimization:**

| Service | Monthly Cost | Notes |
|---------|--------------|-------|
| Redis (3-node xlarge) | ~$3,000 | On-demand pricing |
| SQS (1M writes/sec) | ~$500 | Pricing per 1M requests |
| Lambda (1M invocations/min) | ~$2,000 | 30s avg duration, 3GB memory |
| DynamoDB (on-demand) | ~$1,000 | 1M writes/sec sustained |
| **Total** | **~$6,500/month** | Can reduce with reserved capacity |

---

## DOCUMENTATION LINKS

- [AWS RDS Documentation](https://docs.aws.amazon.com/rds/)
- [Aurora User Guide](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/)
- [DynamoDB Developer Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)
- [ElastiCache User Guide](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/)
- [Kinesis Data Streams](https://docs.aws.amazon.com/kinesis/latest/dev/)
- [SNS User Guide](https://docs.aws.amazon.com/sns/)
- [SQS Developer Guide](https://docs.aws.amazon.com/AWSSimpleQueueService/SQSDeveloperGuide/)
- [EventBridge User Guide](https://docs.aws.amazon.com/eventbridge/latest/userguide/)
- [Well-Architected Framework - Reliability](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/)

