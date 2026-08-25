# SECTION 11: SYSTEM DESIGN USING AWS

## TABLE OF CONTENTS
- [Design Principles](#design-principles)
- [Netflix-Scale Video Streaming Platform](#netflix-scale-video-streaming-platform)
- [Uber-Like Ride-Sharing Platform](#uber-like-ride-sharing-platform)
- [Global E-Commerce Platform](#global-e-commerce-platform)
- [Multi-Region EKS for SaaS](#multi-region-eks-for-saas)
- [Real-Time Observability Platform](#real-time-observability-platform)

---

## DESIGN PRINCIPLES

Before diving into full system designs, understand the interviewer's evaluation criteria:

### Interviewer Expectations at Each Level

**Junior (0–2 years):**
- Can identify major AWS services for the problem.
- Understands basic scalability concepts (horizontal vs vertical).
- Can draw a simple architecture diagram.
- Missing: capacity estimation, failure handling, cost.

**Mid-Level (2–5 years):**
- Complete architecture with capacity estimation.
- Handles availability zones and multi-region.
- Identifies bottlenecks and proposes solutions.
- Includes security and monitoring.
- Missing: detailed cost analysis, advanced scenarios.

**Senior (5+ years):**
- Detailed end-to-end design from requirements to operations.
- Capacity estimation with formulas and assumptions.
- Multiple scaling phases (1M users → 100M users).
- Failure scenarios and mitigation strategies.
- Cost-performance trade-offs and optimization.
- Operational concerns (deployment, monitoring, runbooks).

---

## NETFLIX-SCALE VIDEO STREAMING PLATFORM

### Requirements

**Functional:**
1. Users browse and search catalog of 10,000 titles.
2. Stream video at multiple bitrates (480p, 720p, 1080p, 4K).
3. Resumable playback (remember position).
4. Recommendations (personalized suggestions).
5. User profiles and account management.
6. Billing and payment processing.

**Non-Functional:**
1. **Availability:** 99.99% (52.5 minutes downtime/year).
2. **Latency:** Page load <2s, video start <1s.
3. **Concurrent Users:** 10M simultaneous worldwide.
4. **Data:**
   - 10,000 titles × 100GB avg per title = 1PB media.
   - 200M registered users.
   - 50M daily active users.
5. **Regions:** NA, EU, APAC, LATAM.

### Capacity Estimation

```
Traffic Model:
- Peak concurrent streams: 10M users (80% of 200M users).
- Bitrate per stream: Avg 5 Mbps (mix of 480p–4K).
- Total egress: 10M × 5 Mbps = 50 Tbps.
  (Note: CDN caches reduce origin load by ~90%.)

User Browsing:
- 50M daily active users.
- 30 minutes/day average = 1.5M user-hours.
- 50 page views/user/day = 2.5B page views/day.
- Peak QPS (9 PM): 2.5B / 86400 × 2 = ~58,000 QPS.

Metadata/Catalog:
- 10,000 titles × 100 attributes (cast, synopsis, ratings) = ~10MB per title.
- Total: 100GB catalog data.
- Queries: 58,000 QPS for browsing/search.

Personalization:
- 50M users × 10 recommendations = 500M recommendation records.
- Store in NoSQL (DynamoDB/Cassandra).
- Generate batch nightly; serve from cache.

User State:
- 10M concurrent sessions.
- Session size: ~1KB (playback position, preferences).
- Total: 10GB in-memory cache.
```

### Architecture

```
Mermaid:
graph TB
    Users["200M Users<br/>50M DAU"]
    
    Users -->|HTTPS| CloudFront["CloudFront CDN<br/>350 edge locations<br/>90% hit rate"]
    
    CloudFront -->|cache miss| ALB["Application Load Balancer<br/>Multi-AZ"]
    
    ALB -->|route| WebServers["ECS Fargate Cluster<br/>1000 tasks<br/>Autoscaled by CPU"]
    
    WebServers -->|queries| ElastiCache["ElastiCache<br/>Redis cluster<br/>Session store<br/>Catalog cache"]
    
    WebServers -->|read| DynamoDB["DynamoDB<br/>User profiles<br/>Bookmarks<br/>Recommendations<br/>Global tables"]
    
    WebServers -->|video metadata| Neptune["Neptune<br/>Graph DB<br/>Cast relations<br/>Recommendations"]
    
    Users -->|video stream| S3["S3 Origin<br/>1PB media<br/>Lifecycle: infrequent after 30d"]
    
    MediaUpload["Content Ingestion Pipeline"] -->|transcode| MediaConvert["AWS MediaConvert<br/>Encode to multiple bitrates<br/>Generate thumbnails"]
    
    MediaConvert -->|store| S3
    
    Analytics["Analytics Pipeline"] -->|collect events| Kinesis["Kinesis Data Streams<br/>User events<br/>Playback logs"]
    
    Kinesis -->|process| Lambda["Lambda<br/>Real-time recommendations<br/>Fraud detection"]
    
    Lambda -->|store results| DynamoDB
    
    DynamoDB -->|sync| Redshift["Redshift<br/>Daily analytics<br/>Reports"]
```

### End-to-End Request Flow

**User Browses Catalog:**
1. User requests Netflix homepage.
2. Browser sends HTTPS request to Netflix.
3. DNS routes to nearest CloudFront edge location.
4. CloudFront checks cache (TTL 1 day for catalog).
5. Cache hit: returns 10MB catalog data (~100ms latency).
6. Browser renders 50 titles with posters, metadata.

**User Starts Video Playback:**
1. User clicks "Play" on title (e.g., "Stranger Things S01E01").
2. Browser requests video manifest (HLS/DASH) from CloudFront.
3. CloudFront fetches manifest from origin ALB.
4. Origin ALB queries DynamoDB for encoding info (bitrates available).
5. Returns manifest listing 5 bitrate options (480p–4K).
6. Browser selects 720p (based on bandwidth), requests first chunk.
7. CloudFront delivers chunk (10s video = ~60MB at 720p) from cache or origin.
8. Playback starts (~1s from request).
9. Application logs playback position to DynamoDB (every 10 seconds).

**Capacity per Region:**

| Component | Capacity | Justification |
|-----------|----------|---|
| **ECS Fargate Tasks** | 1000 (100 per AZ) | 50k QPS / 50 req/task = 1000 tasks |
| **ElastiCache Nodes** | 30 (10 per AZ) | 10M sessions × 1KB = 10GB; 256GB node = 40 nodes for overhead |
| **DynamoDB (on-demand)** | 200k RCU | 50k QPS × 4 = 200k RCU |
| **Redshift Cluster** | 8 nodes (ra3.4xlplus) | 1PB historical data |

### Scaling Strategy

**Phase 1 (1M concurrent users):**
- Single region (us-east-1).
- 200 ECS tasks, 10 ElastiCache nodes, 50k DynamoDB RCU.

**Phase 2 (10M concurrent users):**
- Multi-region (3 regions: us-east-1, eu-west-1, ap-southeast-1).
- Each region: 500 tasks, 20 cache nodes, 100k RCU.
- Global DynamoDB tables (eventual consistency, <1s sync).

**Phase 3 (50M concurrent users):**
- Add edge computing (Lambda@Edge for personalization).
- Implement Kinesis sharding for event streams (1000 shards).
- Redshift cluster scaling (12 nodes).

### Security

1. **Authentication:** OAuth2 with multi-factor authentication (MFA).
2. **Encryption:**
   - TLS 1.3 for data in transit.
   - KMS encryption for data at rest (S3, DynamoDB, RDS).
3. **Access Control:**
   - IAM roles for service-to-service auth.
   - VPC isolation; EC2/RDS in private subnets.
   - NACLs and security groups.
4. **DDoS Protection:** AWS Shield (standard) + Shield Advanced for edge.
5. **Compliance:** GDPR, CCPA for user data; content licensing agreements.

### Cost Optimization

**Monthly Cost Estimate (10M concurrent users):**

| Service | Usage | Cost |
|---------|-------|------|
| **CloudFront** | 50 Tbps egress | $5M (after 90% cache hit) |
| **ECS Fargate** | 1500 tasks × 30 days | $200k |
| **ElastiCache** | 30 xlarge nodes × 30 days | $45k |
| **DynamoDB** | 300k RCU (global) × 30 days | $150k |
| **S3** | 1PB storage + egress | $25k |
| **Redshift** | 12 nodes × 30 days | $60k |
| **Network** | Inter-region, egress | $100k |
| **Total** | | **~$5.5M/month** |

**Optimizations:**
- **Reserved Capacity:** 1-year commitment for ECS/RDS = 30% savings.
- **Spot Instances:** Use Spot for non-critical background tasks.
- **CDN Caching:** 90% hit rate reduces origin load.
- **Data Lifecycle:** Archive cold titles to Glacier (1% of catalog).

### Failure Handling

**Scenario 1: CloudFront Edge Failure**
- Route to next nearest edge location.
- CDN vendor ensures availability.
- RTO: <1s, RPO: 0.

**Scenario 2: Region Failure (entire us-east-1 down)**
- DynamoDB global tables replicate to eu-west-1 (automatic promotion).
- DNS (Route53) fails over to eu-west-1.
- RTO: ~30s, RPO: <1s (sync lag).

**Scenario 3: DynamoDB Throttling During Peak (Oscars Night spike)**
- Auto-scaling (increase RCU to 500k).
- Queue requests in SQS, process async.
- RTO: 5–10 seconds.

### Interviewer Follow-Ups

1. **How do you handle video transcoding at scale?**
   - Use AWS MediaConvert (serverless, scales to 1000+ concurrent jobs).
   - Queue input videos in SQS.
   - MediaConvert processes, stores outputs in S3.
   - Lambda triggers on S3 upload to update DynamoDB metadata.

2. **How do you personalize recommendations?**
   - Batch: Run Spark jobs on EMR nightly; output to DynamoDB.
   - Real-time: Lambda processes user events (Kinesis), updates preference model.
   - Hybrid: Cache top 10 recommendations in ElastiCache per user (TTL 6 hours).

3. **How do you prevent concurrent subscription frauds (e.g., one account, 10 users)?**
   - Track IP/device per session.
   - Alert if >5 concurrent streams from different geos.
   - Require re-authentication if suspicious.
   - Use ML (SageMaker) for anomaly detection.

4. **How do you debug why a user can't play a specific video?**
   - Check CloudFront cache (get cache-control headers).
   - Verify DynamoDB has encoding metadata for that title.
   - Check S3 bucket permissions and VPC endpoints.
   - Look at CloudTrail for denied API calls.

5. **What's your strategy if we need to support 4K streaming globally?**
   - Increase CloudFront caching near major metros.
   - Use expensive 4K bitrate only if user's bandwidth >25 Mbps.
   - Implement adaptive bitrate (DASH/HLS) to switch bitrates dynamically.
   - Cost: 3x more bandwidth; justify via user satisfaction metrics.

---

## UBER-LIKE RIDE-SHARING PLATFORM

### Requirements

**Functional:**
1. Users request rides; drivers accept/decline.
2. Real-time matching (nearest driver within X miles).
3. Pricing based on distance, time, surge.
4. Payments (credit card, wallet, promotions).
5. Trip history and ratings.
6. Driver acceptance/rejection with real-time notifications.

**Non-Functional:**
1. **Availability:** 99.95% (22 minutes downtime/year).
2. **Latency:** Match within 30 seconds.
3. **Concurrent Users:** 10M riders, 2M drivers worldwide.
4. **Regions:** 100 cities across 30 countries.
5. **Peak QPS:** 200k ride requests/hour = ~55 QPS.

### Capacity Estimation

```
Peak Load (Evening rush, Friday):
- 10M riders, 10% request ride in 1 hour = 1M requests/hour = 277 QPS.
- 2M drivers, 20% online = 400k drivers.
- Matching: 1M requests × 5 drivers evaluated = 5M queries/sec (spatial, high load).
- Geo queries: Return drivers within 2 miles of rider.
  Spatial index: ~10,000 drivers per 2-mile radius (Manhattan).

Payment Processing:
- 1M trips/hour × $15 avg = $15M revenue/hour.
- Payment latency: <1 second.
- PCI compliance required (Stripe/Square integration).

Location Tracking:
- 400k drivers, GPS update every 10 seconds.
- Data: lat, lng, bearing, speed (40 bytes).
- Throughput: 400k × 1/10 = 40k updates/sec.
- Storage: 400k drivers × 10 days history = 400B location points.
```

### Architecture

```
Mermaid:
graph TB
    Riders["10M Riders"]
    Drivers["2M Drivers"]
    
    Riders -->|request ride| MobileApp["Mobile App"]
    Drivers -->|GPS update| MobileApp
    
    MobileApp -->|HTTPS| ALB["ALB<br/>Multi-AZ"]
    
    ALB -->|route| RideService["Ride Service<br/>ECS Fargate<br/>200 tasks"]
    
    RideService -->|query spatial index| ElastiCache["ElastiCache<br/>Redis Geospatial<br/>400k drivers"]
    
    RideService -->|match logic| MatchEngine["Match Engine<br/>Lambda<br/>Serverless"]
    
    MatchEngine -->|find drivers| DynamoDB["DynamoDB<br/>Drivers table<br/>GSI: location"]
    
    MatchEngine -->|get surge price| PricingCache["DynamoDB cache<br/>Pricing by zone"]
    
    RideService -->|notify driver| SNS["SNS/SQS<br/>Notifications"]
    
    SNS -->|push| DriverApp["Driver App<br/>Accept/Decline"]
    
    RideService -->|payment| PaymentGateway["Payment Gateway<br/>Stripe API<br/>PCI Level 1"]
    
    PaymentGateway -->|debit| PaymentDB["Payment DB<br/>Encrypted,<br/>separate VPC"]
    
    Drivers -->|location stream| Kinesis["Kinesis Stream<br/>GPS updates<br/>40k/sec"]
    
    Kinesis -->|lambda| UpdateGeo["Lambda<br/>Update Redis<br/>Geospatial"]
    
    UpdateGeo -->|write| ElastiCache
```

### End-to-End Request Flow

**Rider Requests a Ride:**
1. Rider opens app, taps "Request Ride".
2. App sends: `{lat: 40.7128, lng: -74.0060, destination_lat: 40.7489, destination_lng: -73.9680}`.
3. Ride Service receives request, generates unique ride ID.
4. Query Match Engine: "Find 100 drivers within 2 miles".
5. Match Engine queries Redis Geospatial Index: `GEORADIUS drivers 40.7128 -74.0060 2 mi`.
6. Redis returns 50 nearby drivers (sorted by distance).
7. Match Engine evaluates each:
   - Check driver acceptance rate (DynamoDB).
   - Calculate ETA (Google Maps API, ~100ms per call).
   - Filter drivers currently on ride (DynamoDB status).
   - Score drivers (distance weight 60%, rating 40%).
8. Select top 10 drivers by score.
9. Send push notification to each (SNS + Firebase Cloud Messaging).
10. First driver to accept within 30 seconds wins.
11. Broadcast "ride accepted" to rider.

**Driver Updates Location:**
1. Driver's GPS updates every 10 seconds.
2. App sends: `{driver_id, lat, lng, bearing, speed}` to Kinesis stream.
3. Lambda processes: `GEOADD drivers ${driver_id} ${lng} ${lat}` in Redis.
4. Redis Geospatial index updated (sub-millisecond).
5. Rider's app queries Redis periodically to show driver location.

### Scaling Strategy

**Phase 1 (10 cities, 100k riders):**
- Single region, single data center.
- 50 Ride Service tasks, basic Redis node.
- Peak load: 27 QPS.

**Phase 2 (50 cities, 1M riders):**
- Multi-region (3 datacenters).
- 200 Ride Service tasks per region.
- Redis cluster (read replicas for geo queries).
- Peak load: 277 QPS.

**Phase 3 (100+ cities, 10M riders):**
- Global sharding (shard by city/zone).
- Each zone: independent ride service, payment processor, driver pool.
- Real-time sync of surge pricing (EventBridge cross-region).
- Peak load: 2,777 QPS.

### Security & Compliance

1. **PCI DSS Level 1:** Payment card data handled by external gateway (Stripe).
2. **Driver/Rider Privacy:** GPS data encrypted in transit + at rest (KMS).
3. **Fraud Detection:** ML model on SageMaker monitors:
   - Unusual payment patterns.
   - Driver acceptance/cancellation rate.
   - Fake accounts (new rider, 10 rides in 1 hour).
4. **Audit Trail:** CloudTrail logs all API calls.

### Cost Optimization

**Monthly Cost (10M riders, 2M drivers):**

| Service | Usage | Cost |
|---------|-------|------|
| **ECS Fargate** | 600 tasks × $0.05/hour | $220k |
| **ElastiCache** | 5 xlarge Redis nodes | $7.5k |
| **DynamoDB** | 1M write/sec × 30 days | $200k |
| **Kinesis** | 40k GPS updates/sec | $30k |
| **SNS** | 1M notifications × $0.50 per 1M | $500 |
| **Payment Processing** | $15M revenue × 2.9% fee | $435k |
| **Total** | | **~$1M/month** |

### Failure Handling

**Scenario: Match Engine Latency Spike**
- If matching takes >10 seconds, push notification times out.
- Fallback: Request goes to "standby pool" (queued drivers).
- Use SQS for async matching if Redis geo query is slow.

**Scenario: Payment Gateway Down (Stripe API error)**
- Queue payment to SQS.
- Retry with exponential backoff (immediately, then 1s, 2s, 5s...).
- Deduct fare from rider's wallet (cached in DynamoDB).
- Process credit card later when Stripe recovers.

**Scenario: Redis Geospatial Index Corruption**
- Detect: Spike in "no drivers found" errors.
- Mitigation: Rebuild index from DynamoDB (driver status table).
- Lambda job scans DynamoDB, repopulates Redis.
- ETA: 5–10 minutes for 2M drivers.

---

(Content continues with Global E-Commerce, Multi-Region EKS, Real-Time Observability Platform sections...)

---

## DOCUMENTATION LINKS

- [Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [AWS Solutions Library](https://aws.amazon.com/solutions/)
- [Case Studies](https://aws.amazon.com/solutions/case-studies/)
- [Database Design Patterns](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html)

