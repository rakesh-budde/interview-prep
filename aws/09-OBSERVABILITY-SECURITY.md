# OBSERVABILITY & SECURITY — Deep Dive Interview Preparation

> **Scope:** Sections 11–12 of 20 | Beginner → Expert | FAANG-level depth  
> **Coverage:** CloudWatch, CloudTrail, X-Ray, Prometheus, Grafana, SRE practices, GuardDuty, KMS, WAF, 40+ Q&A

---

## Table of Contents

**Section 11: Observability & SRE**
1. [Observability Pillars](#1-observability-pillars)
2. [Amazon CloudWatch Deep Dive](#2-amazon-cloudwatch-deep-dive)
3. [AWS CloudTrail](#3-aws-cloudtrail)
4. [AWS X-Ray](#4-aws-x-ray)
5. [Amazon Managed Prometheus & Grafana](#5-amazon-managed-prometheus--grafana)
6. [OpenTelemetry on AWS](#6-opentelemetry-on-aws)
7. [SRE Fundamentals: SLI, SLO, SLA, Error Budgets](#7-sre-fundamentals)
8. [Incident Management & RCA](#8-incident-management--rca)

**Section 12: AWS Security**
9. [AWS GuardDuty](#9-aws-guardduty)
10. [Amazon Inspector](#10-amazon-inspector)
11. [AWS Security Hub](#11-aws-security-hub)
12. [AWS KMS Deep Dive](#12-aws-kms-deep-dive)
13. [Secrets Manager vs. Parameter Store](#13-secrets-manager-vs-parameter-store)
14. [AWS WAF & Shield](#14-aws-waf--shield)
15. [Certificate Manager (ACM)](#15-certificate-manager-acm)
16. [Encryption at Rest & in Transit](#16-encryption-at-rest--in-transit)
17. [Interview Questions & Answers](#17-interview-questions--answers)
18. [Documentation Links](#18-documentation-links)

---

## 1. Observability Pillars

**Three pillars of observability:**

| Pillar | What it answers | AWS service | Open source |
|---|---|---|---|
| **Metrics** | How is the system performing? | CloudWatch Metrics | Prometheus |
| **Logs** | What happened? | CloudWatch Logs | Loki, Elasticsearch |
| **Traces** | How did a request flow through services? | X-Ray | Jaeger, Zipkin |

**Events** (sometimes called the "fourth pillar"): Changes in system state (deployments, config changes, auto-scaling events). CloudTrail, CloudWatch Events, EventBridge.

**The difference between monitoring and observability:**
- **Monitoring:** Watching known metrics for known failure modes. "Alert when CPU > 90%."
- **Observability:** Understanding system behavior from its outputs. "Why is this request slow for this specific user from this IP?" — answerable without pre-defining the question.

**RED Method (for services):**
- **Rate:** Requests per second.
- **Errors:** Error rate (% of requests failing).
- **Duration:** Latency (p50, p95, p99).

**USE Method (for resources):**
- **Utilization:** % time resource is busy.
- **Saturation:** How much work is queued/waiting.
- **Errors:** Count of errors.

---

## 2. Amazon CloudWatch Deep Dive

### Metrics

**Namespaces:** Metrics are organized in namespaces. AWS services use `AWS/EC2`, `AWS/RDS`, `AWS/Lambda`, etc. Custom metrics use your own namespace.

**Resolution:**
- Standard: 1-minute data points (free).
- High-resolution: 1-second data points (custom metrics, costs extra).
- Data stored: 15 months (standard resolution). Data retention:
  - < 60 s resolution: 3 hours
  - 1 min resolution: 15 days
  - 5 min resolution: 63 days
  - 1 hour resolution: 15 months (455 days)

**Custom Metrics:**
```bash
# Publish a custom metric (application-level business KPI)
aws cloudwatch put-metric-data \
  --namespace "MyApp/Payments" \
  --metric-name "PaymentSuccessRate" \
  --value 99.5 \
  --unit Percent \
  --dimensions Service=CheckoutAPI,Environment=Production \
  --timestamp $(date -u +%Y-%m-%dT%H:%M:%SZ)
```

**Metric Math:**
```bash
# Create a compound metric: error rate from request counts
# In CloudWatch Metrics console or API:
METRIC_MATH: errors/requests * 100
# Allows alerting on computed values without emitting them separately
```

**CloudWatch Agent (for OS-level metrics):**
```json
{
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "mem": {"measurement": ["mem_used_percent"]},
      "disk": {"measurement": ["disk_used_percent"], "resources": ["/", "/data"]},
      "netstat": {"measurement": ["tcp_established", "tcp_time_wait"]}
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {"file_path": "/var/log/nginx/access.log", "log_group_name": "/nginx/access"}
        ]
      }
    }
  }
}
```

### Alarms

**Alarm states:** OK, ALARM, INSUFFICIENT_DATA.

**Alarm actions:**
- SNS notification → email, SMS, PagerDuty/OpsGenie webhook, Lambda.
- EC2 actions: stop, terminate, reboot, recover.
- Auto Scaling: trigger scale-out/scale-in policy.
- Systems Manager: run automation document.

```hcl
resource "aws_cloudwatch_metric_alarm" "api_error_rate" {
  alarm_name          = "api-5xx-error-rate-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  datapoints_to_alarm = 2  # Must breach in 2 of 2 periods

  metric_query {
    id          = "error_rate"
    expression  = "errors/total * 100"
    label       = "Error Rate %"
    return_data = true
  }

  metric_query {
    id = "errors"
    metric {
      metric_name = "HTTPCode_Target_5XX_Count"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "Sum"
      dimensions  = { LoadBalancer = "app/my-alb/abc123" }
    }
  }

  metric_query {
    id = "total"
    metric {
      metric_name = "RequestCount"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "Sum"
      dimensions  = { LoadBalancer = "app/my-alb/abc123" }
    }
  }

  threshold           = 1  # Alert if error rate > 1%
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]
  treat_missing_data  = "notBreaching"
}
```

### CloudWatch Logs Insights

```bash
# Find slow Lambda executions
aws logs start-query \
  --log-group-name "/aws/lambda/my-function" \
  --query-string '
    filter @type = "REPORT"
    | parse @message "Duration: * ms" as duration
    | stats max(duration) as maxDuration,
            avg(duration) as avgDuration,
            percentile(duration, 99) as p99Duration
      by bin(5m)
    | sort @timestamp desc'

# Find errors by IP in ALB logs
aws logs start-query \
  --log-group-name "/alb/access-logs" \
  --query-string '
    fields @timestamp, client_ip, target_status_code, request_url
    | filter target_status_code >= 500
    | stats count(*) as errorCount by client_ip
    | sort errorCount desc
    | limit 20'

# Correlate with trace IDs
aws logs start-query \
  --query-string '
    fields @timestamp, @message, traceId
    | filter @message like /ERROR/
    | limit 100'
```

**Contributor Insights:** Automatically identifies the top contributors to unusual patterns (top IP addresses causing 5xx errors, most active API callers, top error codes).

---

## 3. AWS CloudTrail

**CloudTrail** records all AWS API calls (who, what, when, from where). Every `RunInstances`, `PutBucketPolicy`, `AssumeRole`, `CreateUser` call generates a CloudTrail event.

### Organization Trail (Multi-Account)

```bash
# Create organization trail from management account
aws cloudtrail create-trail \
  --name org-trail \
  --s3-bucket-name central-cloudtrail-logs \
  --is-organization-trail \
  --is-multi-region-trail \
  --include-global-service-events \
  --enable-log-file-validation  # Detect log tampering

aws cloudtrail start-logging --name org-trail
```

### Event Types

- **Management events:** Control plane API calls (EC2, IAM, S3 bucket management). Always logged.
- **Data events:** Data plane API calls (S3 object access, Lambda invocations, DynamoDB row reads). Disabled by default — **charged extra at $0.10/100K events**.
- **Insights events:** Detect unusual activity patterns (unusual API call rates, unusual IAM error rates).

### Security Investigation with CloudTrail

```bash
# Find who deleted an S3 bucket
aws logs filter-log-events \
  --log-group-name "cloudtrail-logs" \
  --filter-pattern '{ $.eventName = "DeleteBucket" && $.requestParameters.bucketName = "my-critical-bucket" }' \
  --query 'events[*].message'

# Find all IAM changes in last 24 hours
aws logs start-query \
  --log-group-name "aws-cloudtrail-logs" \
  --query-string '
    fields @timestamp, userIdentity.arn, eventName, requestParameters
    | filter eventSource = "iam.amazonaws.com"
    | sort @timestamp desc
    | limit 50'

# Detect root account usage (critical alert)
aws logs start-query \
  --query-string '
    fields @timestamp, eventName, sourceIPAddress, userAgent
    | filter userIdentity.type = "Root"
    | sort @timestamp desc'
```

---

## 4. AWS X-Ray

**X-Ray** provides distributed tracing — tracking a request as it flows through multiple services, measuring latency at each hop.

**Key concepts:**
- **Trace:** A single request's journey through the system.
- **Segment:** One service's portion of the trace (e.g., the API service's processing time).
- **Subsegment:** A component within a segment (e.g., a DynamoDB call within the API service).
- **Sampling:** X-Ray samples a percentage of requests to avoid overhead (default: 1 req/sec + 5%).

**X-Ray SDK integration:**
```python
# Python with X-Ray
from aws_xray_sdk.core import xray_recorder, patch_all
from aws_xray_sdk.ext.flask import XRayMiddleware

app = Flask(__name__)
XRayMiddleware(app, xray_recorder)
patch_all()  # Auto-instrument: boto3, requests, SQLAlchemy, etc.

@app.route('/api/users/<user_id>')
def get_user(user_id):
    with xray_recorder.in_subsegment('get-user-from-db') as subsegment:
        subsegment.put_annotation('user_id', user_id)  # Searchable
        subsegment.put_metadata('query_params', request.args)  # Not searchable
        user = dynamodb.get_item(Key={'user_id': user_id})
    return jsonify(user)
```

**Trace ID propagation:** X-Ray injects `X-Amzn-Trace-Id` header in all requests. Downstream services read this header and continue the trace. For services without X-Ray SDK, manually forward the header.

**Service Map:** Visual graph of all services and their dependencies, with latency and error rates on each connection. Invaluable for finding which service is causing cascading failures.

---

## 5. Amazon Managed Prometheus & Grafana

**Amazon Managed Service for Prometheus (AMP):** A fully managed Prometheus-compatible service. No Prometheus server to manage. Ingest from Kubernetes (via ADOT Collector or Prometheus remote_write), query with PromQL.

```yaml
# Configure Prometheus remote_write to AMP
remoteWrite:
  - url: "https://aps-workspaces.us-east-1.amazonaws.com/workspaces/ws-xxx/api/v1/remote_write"
    sigv4:
      region: us-east-1
      role_arn: arn:aws:iam::123:role/AMPIngestRole
    queue_config:
      max_samples_per_send: 1000
      max_shards: 200
      capacity: 2500
```

**Amazon Managed Grafana (AMG):** Managed Grafana with native AWS data source integration (CloudWatch, AMP, X-Ray, OpenSearch, Athena, IoT SiteWise). SSO via IAM Identity Center.

**Key dashboards to have in production:**
1. **Service Health:** Request rate, error rate, p95/p99 latency per service.
2. **Infrastructure:** Node CPU, memory, disk usage; network throughput; EBS IOPS.
3. **Kubernetes:** Pod restarts, pending pods, node utilization, HPA scaling events.
4. **Cost:** AWS Cost Explorer linked data, resource utilization efficiency.
5. **SLO Tracking:** SLO compliance rate, error budget remaining, burn rate.

---

## 6. OpenTelemetry on AWS

**OpenTelemetry (OTel)** is a vendor-neutral observability framework that standardizes how you collect metrics, logs, and traces. AWS supports OTel via the **AWS Distro for OpenTelemetry (ADOT)**.

**ADOT Collector on EKS:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: adot-collector
  namespace: aws-otel-eks
spec:
  template:
    spec:
      containers:
      - name: adot-collector
        image: public.ecr.aws/aws-observability/aws-otel-collector:latest
        env:
        - name: K8S_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - mountPath: /conf
          name: otel-config
      volumes:
      - name: otel-config
        configMap:
          name: adot-config
```

**OTel pipeline: app → ADOT Collector → X-Ray + AMP + CloudWatch:**
```yaml
# ADOT Collector config (otel-config ConfigMap)
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch:
    timeout: 1s

exporters:
  awsxray:
    region: us-east-1
  prometheusremotewrite:
    endpoint: https://aps-workspaces.us-east-1.amazonaws.com/workspaces/ws-xxx/api/v1/remote_write
    auth:
      authenticator: sigv4auth

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [awsxray]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheusremotewrite]
```

---

## 7. SRE Fundamentals

### SLI, SLO, SLA

**SLI (Service Level Indicator):** A quantitative measure of service behavior. The metric itself.

```
Availability SLI: (successful requests) / (total requests) × 100%
Latency SLI: % of requests completing in < 200ms
Error rate SLI: HTTP 5xx responses / total HTTP responses
```

**SLO (Service Level Objective):** A target for an SLI. Your internal commitment about reliability.

```
Availability SLO: 99.9% monthly availability (allows 43.8 min downtime/month)
Latency SLO: 95% of requests < 200ms
Error rate SLO: < 0.1% HTTP 5xx responses
```

**SLA (Service Level Agreement):** A legal contract with customers. Usually more lenient than internal SLO. If you miss the SLA, you pay service credits. Your SLO should be stricter than your SLA so you catch issues before violating the SLA.

```
SLA: 99.5% availability (allows 3.6 hr downtime/month)
Internal SLO: 99.9% (catch degradation before SLA breach)
Internal SLI target: Alert when availability < 99.95% (even before SLO breach)
```

**Error Budget:**

Error budget = time/requests your service is allowed to fail while still meeting the SLO.

```
SLO: 99.9% availability over 30 days
Total minutes in month: 43,200
Error budget: (1 - 0.999) × 43,200 = 43.2 minutes of allowed downtime

If you've spent 30 minutes of downtime already:
Error budget remaining: 13.2 minutes
Error budget consumed: 69.4%

Policy: If error budget < 5% remaining:
- Freeze new feature releases
- Prioritize reliability work
- Daily incident review
```

**Error Budget Burn Rate:**

```
Burn rate = (error budget consumed rate) / (acceptable burn rate)
If burn rate > 1: consuming error budget faster than it replenishes → SLO at risk

Alert thresholds:
- Burn rate > 14.4 for 1 hour = consuming 1 month budget in 2 hours → page immediately
- Burn rate > 6 for 6 hours = consuming 1 month budget in 5 days → alert team
- Burn rate > 3 for 3 days = consuming 1 month budget in 10 days → ticket
```

**Prometheus SLO burn rate alert:**
```yaml
groups:
- name: slo-burn-rate
  rules:
  - alert: APIHighBurnRate
    expr: |
      sum(rate(http_requests_total{status=~"5.."}[1h])) /
      sum(rate(http_requests_total[1h])) > (14.4 * 0.001)
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "High error burn rate - SLO at risk"
      description: "Error budget burn rate is {{ $value | humanizePercentage }}"
```

---

## 8. Incident Management & RCA

### Incident Lifecycle

```
1. Detect: Automated alert fires (CloudWatch alarm → SNS → PagerDuty)
2. Acknowledge: On-call engineer claims the incident
3. Triage: Assess impact and severity (how many users, what's broken)
4. Coordinate: Incident commander, communications lead, technical responders
5. Investigate: Check metrics, logs, traces, recent deployments
6. Mitigate: Roll back, redirect traffic, add capacity — STOP the bleeding first
7. Resolve: Permanent fix deployed, monitors cleared
8. RCA: Root cause analysis, action items, blameless postmortem
```

**Severity levels:**
| Sev | Impact | Response time | Examples |
|---|---|---|---|
| SEV-1 | Complete service outage | Immediately (24×7) | Payment system down, data loss |
| SEV-2 | Significant degradation | Within 30 min | 50% error rate, all checkout failing |
| SEV-3 | Partial feature impact | Next business day | One region elevated errors |
| SEV-4 | Minor, no user impact | Sprint planning | Internal tool slow |

### Blameless Postmortem

Key principles:
1. **Blameless:** Focus on systems and processes, not individuals. Anyone would have made the same decisions with the same information.
2. **Learning-focused:** Goal is to prevent recurrence, not to punish.
3. **Action items:** Every postmortem produces concrete, assigned, time-bounded action items.

**Postmortem template:**

```markdown
## Incident Summary
**Date:** 2024-01-15  **Duration:** 47 minutes  **Severity:** SEV-2
**Impact:** 34% of checkout requests failed. ~15,000 users affected.
**Responders:** Alice (IC), Bob (Backend), Carol (Infra)

## Timeline
- 14:23 UTC: Alert fired (HTTP 5xx > 5%)
- 14:26 UTC: Alice acknowledged, started investigation
- 14:31 UTC: Bob identified DynamoDB throttling in X-Ray traces
- 14:38 UTC: Root cause confirmed: new feature query missing index
- 14:52 UTC: Feature flag disabled, error rate returned to < 0.1%
- 15:10 UTC: Permanent fix deployed with proper index

## Root Cause
New feature deployed at 13:45 UTC introduced a DynamoDB scan query (full table scan) triggered on every checkout event. Under normal load, this caused DynamoDB provisioned capacity to be consumed, throttling legitimate checkout queries.

## Contributing Factors
1. Load testing was performed on an isolated environment with a smaller dataset — scan didn't appear slow at scale
2. DynamoDB read capacity alarm threshold was set to 80% (not 60%) — alert fired too late
3. No query review step in deployment checklist for DynamoDB operations

## Action Items
| Action | Owner | Due |
|---|---|---|
| Add DynamoDB query scan detection to code review checklist | Platform Team | 2024-01-22 |
| Lower DynamoDB capacity alarms to 60% | Alice | 2024-01-16 |
| Add load test with production-scale dataset to release checklist | Bob | 2024-02-01 |
| Implement DynamoDB query advisor integration in CI | Carol | 2024-02-15 |
```

---

# SECTION 12: AWS SECURITY

## 9. AWS GuardDuty

**GuardDuty** is a threat detection service that continuously monitors for malicious activity and unauthorized behavior in your AWS account.

**Data sources GuardDuty analyzes:**
- **CloudTrail management events:** Detects suspicious API call patterns (unusual regions, unusual services, credential abuse).
- **VPC Flow Logs:** Detects network-level threats (port scanning, C2 communication, cryptocurrency mining traffic).
- **DNS logs:** Detects DNS-based exfiltration, DGA (Domain Generation Algorithm) malware.
- **S3 data events:** Detects unusual S3 access patterns (mass download, public ACL changes).
- **EKS audit logs:** Detects suspicious Kubernetes API activity (privilege escalation, DaemonSet creation).
- **RDS login events:** Detects brute-force and unusual login patterns.

**Enabling organization-wide GuardDuty:**
```bash
# Designate a GuardDuty admin account (security tooling account)
aws guardduty enable-organization-admin-account \
  --admin-account-id SECURITY-ACCOUNT-ID

# Auto-enroll new accounts
aws guardduty update-organization-configuration \
  --detector-id DETECTOR-ID \
  --auto-enable-organization-members ALL
```

**Responding to GuardDuty findings via EventBridge:**
```python
# Lambda triggered by GuardDuty high-severity finding
def handle_guardduty_finding(event, context):
    detail = event['detail']
    finding_type = detail['type']
    severity = detail['severity']
    account_id = detail['accountId']
    
    if severity >= 7.0:  # High severity
        if 'UnauthorizedAccess:IAMUser' in finding_type:
            # Immediately revoke the compromised credential
            iam = boto3.client('iam')
            access_key_id = detail['resource']['accessKeyDetails']['accessKeyId']
            iam.update_access_key(
                AccessKeyId=access_key_id,
                Status='Inactive'
            )
            # Notify security team
            sns.publish(
                TopicArn=SECURITY_ALERT_TOPIC,
                Subject=f"[SEV-1] Compromised IAM key in {account_id}",
                Message=json.dumps(detail, indent=2)
            )
```

---

## 10. Amazon Inspector

**Amazon Inspector** continuously scans:
- EC2 instances for OS vulnerabilities (CVEs in packages).
- Lambda functions for application dependency vulnerabilities.
- ECR container images for package vulnerabilities and secrets.

Inspector uses SSM Agent for EC2 scanning (no agent to install separately). It auto-discovers new EC2 instances and containers.

**Inspector + ECR integration:**
- Scan on push: Each new image pushed to ECR is automatically scanned.
- Findings appear in both Inspector and ECR consoles.
- Severity: Critical, High, Medium, Low, Informational.

**Act on findings:**
```bash
# Get all critical ECR findings
aws inspector2 list-findings \
  --filter-criteria '{
    "severity": [{"comparison": "EQUALS", "value": "CRITICAL"}],
    "resourceType": [{"comparison": "EQUALS", "value": "AWS_ECR_CONTAINER_IMAGE"}]
  }' \
  --query 'findings[*].{Resource:resources[0].id,Vuln:packageVulnerabilityDetails.vulnerabilityId,Score:inspectorScore}'
```

---

## 11. AWS Security Hub

**Security Hub** aggregates security findings from GuardDuty, Inspector, Macie, Config, IAM Access Analyzer, Firewall Manager, and third-party security tools into a single pane.

**Security standards:**
- AWS Foundational Security Best Practices (FSBP): AWS-curated controls mapped to AWS services.
- CIS AWS Foundations Benchmark: Center for Internet Security controls.
- PCI DSS: Payment Card Industry controls.
- NIST SP 800-53: Federal security controls.

**Automated remediation:**
```json
// EventBridge rule + Lambda: auto-remediate S3 public ACL finding
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "ProductName": ["Security Hub"],
      "GeneratorId": ["aws-foundational-security-best-practices/v/1.0.0/S3.2"]
    }
  }
}
```

---

## 12. AWS KMS Deep Dive

### Beginner Foundation

**AWS KMS (Key Management Service)** is a managed service for creating and controlling cryptographic keys. It integrates with almost every AWS service for encryption at rest.

**Key types:**
- **AWS-managed keys:** Created and managed by AWS on your behalf (e.g., `aws/s3`, `aws/rds`). Free. Cannot be used across accounts. Cannot be rotated on demand.
- **Customer-managed keys (CMK):** You create, control rotation, set key policies. $1/month + $0.03/10K API calls.
- **Data keys:** AES-256 keys generated by KMS for envelope encryption — not stored in KMS.

### Intermediate Mechanics

**Envelope Encryption:**

Direct KMS encryption of data would be too slow (KMS API latency, rate limits). Envelope encryption solves this:

```
1. KMS generates a Data Key (AES-256): plaintext key + ciphertext of the key
2. Your application encrypts data with the plaintext data key (fast, local operation)
3. Your application stores the ciphertext data key alongside the encrypted data
4. Plaintext data key is immediately discarded from memory

To decrypt:
1. Call KMS.Decrypt(ciphertext_data_key) → KMS decrypts and returns plaintext data key
2. Use plaintext data key to decrypt your data locally
3. Discard plaintext data key from memory
```

```python
import boto3
from cryptography.fernet import Fernet
import base64

kms = boto3.client('kms')
KEY_ID = 'arn:aws:kms:us-east-1:123:key/key-id'

# Encrypt
def encrypt_data(plaintext: bytes) -> dict:
    # Get data key from KMS
    response = kms.generate_data_key(KeyId=KEY_ID, KeySpec='AES_256')
    plaintext_key = response['Plaintext']
    encrypted_key = response['CiphertextBlob']
    
    # Encrypt data locally
    f = Fernet(base64.urlsafe_b64encode(plaintext_key[:32]))
    encrypted_data = f.encrypt(plaintext)
    
    # Delete plaintext key from memory
    del plaintext_key
    
    return {'encrypted_data': encrypted_data, 'encrypted_key': encrypted_key}

# Decrypt
def decrypt_data(encrypted_data: bytes, encrypted_key: bytes) -> bytes:
    response = kms.decrypt(CiphertextBlob=encrypted_key)
    plaintext_key = response['Plaintext']
    
    f = Fernet(base64.urlsafe_b64encode(plaintext_key[:32]))
    return f.decrypt(encrypted_data)
```

**KMS Key Policy:**
```json
{
  "Statement": [
    {
      "Sid": "EnableRootAccount",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "AllowAppEncryption",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::123456789012:role/AppRole"},
      "Action": ["kms:GenerateDataKey", "kms:Decrypt"],
      "Resource": "*"
    },
    {
      "Sid": "DenyDeleteWithoutMFA",
      "Effect": "Deny",
      "Principal": "*",
      "Action": ["kms:ScheduleKeyDeletion", "kms:DeleteAlias"],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}
      }
    }
  ]
}
```

**Key rotation:** CMKs can be configured for automatic annual rotation. The key material rotates but the key ID remains the same — no application changes needed. Old key material is retained for decrypting data encrypted before rotation.

### Advanced Engineering

**KMS cross-account:** Grant another account permission to use your KMS key:
1. Update the key policy to allow the other account's principal.
2. The other account's IAM policy must also allow `kms:Decrypt`.

```json
// Key policy in Account A
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::ACCOUNT-B:role/DataReaderRole"},
  "Action": ["kms:Decrypt", "kms:DescribeKey"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:CallerAccount": "ACCOUNT-B"
    }
  }
}
```

**KMS API rate limits:** `GenerateDataKey`: 10,000 req/s (across region). At high throughput (100K S3 PutObjects/s), you'll hit KMS limits without S3 Bucket Keys. Bucket Keys create a short-lived data key in S3 that encrypts multiple objects — reduces KMS calls by 99%.

---

## 13. Secrets Manager vs. Parameter Store

| Dimension | Secrets Manager | Parameter Store |
|---|---|---|
| Cost | $0.40/secret/month + $0.05/10K API calls | Free (Standard) / $0.05/parameter/month (Advanced) |
| Automatic rotation | Yes (built-in Lambda rotation templates) | No (custom Lambda required) |
| Secret size | 64 KB | 4 KB (Standard) / 8 KB (Advanced) |
| Cross-account | Yes | Limited |
| Versioning | Yes (via staging labels: AWSCURRENT, AWSPREVIOUS) | Yes (parameter history) |
| RDS integration | Built-in rotation support | Manual |
| Best for | Database credentials, API keys needing rotation | Config values, feature flags, non-sensitive config |

**Secrets Manager rotation example:**
```hcl
resource "aws_secretsmanager_secret_rotation" "db_password" {
  secret_id           = aws_secretsmanager_secret.db.id
  rotation_lambda_arn = aws_lambda_function.rotation.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

AWS provides Lambda rotation functions for RDS (MySQL, PostgreSQL), Redshift, DocumentDB, and other services. The rotation function:
1. Creates a new password and sets it in the database.
2. Tests the new credential.
3. Marks the new secret version as AWSCURRENT.

---

## 14. AWS WAF & Shield

*(Full coverage in Section 3 — Networking. Key interview points for security section:)*

**WAF v2 vs. v1:** WAF v2 (WAFv2) is the current version with a unified API, more flexible rule syntax, web ACL capacity units (WCUs), and JSON-based rules. WAF Classic (v1) is deprecated.

**WAF managed rules cost:** Managed rule groups charge $1/million WCUs consumed. AWS Managed Rules Common Rule Set (CRS) = 700 WCUs. At 1M requests/day → ~$0.70/month for the rule group.

**Shield Advanced features for FAANG-level answer:**
- Automatic application layer DDoS mitigation (L7) — analyzes traffic patterns and auto-creates WAF rules.
- DRT (DDoS Response Team) access — AWS engineers actively assist during attacks.
- Cost protection — credits for scaling costs incurred due to DDoS.
- Integration with AWS Firewall Manager for centralized WAF + Shield management.

---

## 15. Certificate Manager (ACM)

**ACM** provisions, manages, and deploys SSL/TLS certificates for AWS services (ALB, CloudFront, API Gateway, App Runner).

**Key properties:**
- Free for certificates used with AWS services (no certificate cost).
- Automatic renewal (managed by AWS, renewals happen ~60 days before expiry).
- Cannot export private keys for certificates issued by ACM (security by design).
- Can import third-party certificates (enables using your own CA).

**ACM DNS validation (recommended for automation):**
```bash
# Request certificate
CERT_ARN=$(aws acm request-certificate \
  --domain-name api.example.com \
  --subject-alternative-names "*.api.example.com" \
  --validation-method DNS \
  --query CertificateArn --output text)

# Get CNAME record to add to DNS
aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
# Returns: {Name: _xxx.api.example.com, Type: CNAME, Value: _yyy.acm-validations.aws.}

# Add CNAME to Route 53 (validates domain ownership, enables auto-renewal)
```

---

## 16. Encryption at Rest & in Transit

### Encryption at Rest

**AWS encryption hierarchy:**
```
Customer Managed Key (CMK) in KMS
  → Data Key (AES-256, generated per operation)
    → Encrypts: S3 objects, EBS volumes, RDS data, DynamoDB items, Secrets
```

**Default encryption by service:**
| Service | Default encryption | Type |
|---|---|---|
| S3 | SSE-S3 (enabled by default since Jan 2023) | AES-256 with S3-managed keys |
| EBS | Not by default, account-level default can be set | AES-256 with KMS |
| RDS | Optional (must enable at creation) | KMS |
| DynamoDB | Always encrypted | AWS-owned key (free) or KMS CMK |
| Lambda env vars | AWS-managed KMS | KMS |
| Secrets Manager | Always encrypted with KMS | CMK or AWS-managed |

### Encryption in Transit

**TLS everywhere:**
- ALB, CloudFront, API Gateway: Enforce HTTPS. Redirect HTTP to HTTPS.
- RDS: `require_ssl = 1` parameter. Use SSL certificate from AWS RDS trust store.
- EKS: API server → kubelet communication uses TLS (certificate rotated automatically).
- Internal service-to-service: Use mutual TLS (mTLS) via service mesh (Istio, App Mesh) or certificate-based authentication.

**Minimum TLS version:**
```json
// CloudFront security policy
"MinimumProtocolVersion": "TLSv1.2_2021"  // Supports TLS 1.2 and 1.3 only

// ALB HTTPS listener
// SslPolicy: ELBSecurityPolicy-TLS13-1-2-2021-06
// Supports TLS 1.2 and 1.3, excludes all TLS 1.0/1.1
```

---

## 17. Interview Questions & Answers

---

### Question 1: Define SLO, SLI, and SLA. How does error budget factor into release decisions?

**What the interviewer is testing:** SRE concepts mastery and ability to apply them to real engineering decisions.

**Strong answer:**

**SLI (Service Level Indicator):** The measured metric that reflects user experience. Must be measurable and meaningful. Examples:
- Availability: % of successful HTTP responses (non-5xx)
- Latency: % of requests completing in < 200ms
- Error rate: % of requests resulting in error

**SLO (Service Level Objective):** The target value for an SLI. Your engineering team's commitment to reliability. The number that drives reliability work. Example: "99.9% of requests complete in < 200ms."

**SLA (Service Level Agreement):** A legal contract with customers. Less strict than SLO (your SLO is your buffer). SLA breach triggers financial penalties (service credits).

**Practical relationship:** SLO > SLA > actual measured SLI. If SLA is 99.5% uptime, set SLO to 99.9% — you have a buffer before SLA breach. Alert on SLI degradation before the SLO is violated.

**Error budget application to releases:**

```
SLO: 99.9% availability (30-day window)
Error budget per month: 43.2 minutes
Current month status: Used 35 minutes (81% consumed) with 15 days remaining

Decision framework:
- Error budget > 50% remaining: green light on feature releases
- Error budget 20-50%: proceed with caution, enhanced monitoring
- Error budget < 20%: freeze new features, prioritize reliability work
- Error budget fully consumed: all hands on reliability work, no releases
```

**This transforms reliability from a vague goal into an engineering resource.** Teams negotiate feature velocity against reliability investment using measurable data rather than opinion.

**Error budget drives concrete behaviors:**
- When budget is healthy: teams can deploy frequently with acceptable risk.
- When budget is low: automatic triggers enforce a "reliability tax" — features must wait.
- Prevents the tragedy of the commons where every team "just this once" deploys risky features.

**Likely follow-ups:**
1. *How do you set appropriate SLOs?* — Start with user expectations (what's acceptable?), analyze historical performance, set the SLO slightly above current performance to have room to improve. Don't set SLOs too high initially — overly strict SLOs consume error budget constantly and paralyze releases.
2. *What is burn rate and when do you use it for alerting?* — Burn rate measures how fast you're consuming error budget relative to the rate you need to sustain the SLO. At burn rate 1 = consuming budget exactly as fast as it replenishes. At burn rate 14.4, you'd exhaust the monthly budget in 2 hours. Page on high burn rates immediately; alert (not page) on sustained moderate burn rates.

---

### Question 2: How does envelope encryption work in AWS KMS? Why is it used instead of direct KMS encryption?

**What the interviewer is testing:** Cryptography fundamentals, AWS security architecture.

**Strong answer:**

**Direct KMS encryption problem:**

If you had AWS encrypt your data directly using KMS, you'd call `KMS.Encrypt(plaintext)` which returns the ciphertext. Problems:
1. **KMS API payload limit:** KMS can encrypt at most 4 KB. Your database row might be 1 MB.
2. **Latency:** KMS API call is 1–10 ms. Encrypting millions of records would be extremely slow.
3. **KMS API rate limits:** KMS limits calls per second. High-throughput applications would throttle.

**Envelope encryption solution:**

```
Step 1: Call KMS.GenerateDataKey(CMK)
→ Returns: plaintext_data_key (32 bytes AES-256) + encrypted_data_key (opaque blob)

Step 2: Encrypt your data locally using plaintext_data_key
→ Your application uses AES-256-GCM encryption locally (milliseconds, no API call)
→ Unlimited data size

Step 3: Store [encrypted_data + encrypted_data_key] together
→ The encrypted_data_key cannot be decrypted without access to the CMK in KMS
→ Plaintext key is immediately wiped from memory

To decrypt:
Step 1: Call KMS.Decrypt(encrypted_data_key)
→ KMS checks caller has permission to use the CMK
→ Returns: plaintext_data_key (only for authorized callers)

Step 2: Decrypt data locally using plaintext_data_key
→ Fast, local operation
→ Wipe plaintext_data_key from memory immediately
```

**Security properties:**
- Data is protected by AES-256 (NIST-approved, computationally infeasible to brute-force).
- The data key itself is protected by the KMS CMK (hardware security module).
- Audit trail: every `Decrypt` call is logged in CloudTrail with the caller's identity.
- Access control: removing `kms:Decrypt` permission from an IAM role immediately prevents decryption, even without re-encrypting all data.

**Services that use envelope encryption transparently:** S3 SSE-KMS, RDS encryption, EBS encryption, DynamoDB CMK encryption — all use envelope encryption internally. You call `PutObject` to S3 and it handles the GenerateDataKey → Encrypt → store cycle for you.

**Likely follow-ups:**
1. *What is the difference between GenerateDataKey and GenerateDataKeyWithoutPlaintext?* — `GenerateDataKeyWithoutPlaintext` returns only the encrypted data key (no plaintext key). Used when you need to pre-generate keys for future use without immediately encrypting. The plaintext key is derived later by calling `Decrypt`. Useful for storing encrypted keys that will be used offline.
2. *How does KMS key deletion work and what are the safeguards?* — KMS requires a waiting period before deletion (7–30 days, configurable). During this period, any attempt to use the key fails with an error. AWS sends email notifications. You can cancel deletion during the waiting period. After deletion, data encrypted with that key is permanently unrecoverable. This is why `prevent_destroy = true` in Terraform is critical for KMS keys.

---

## 18. Documentation Links

| Topic | Official Link |
|---|---|
| CloudWatch User Guide | https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html |
| CloudWatch Logs Insights | https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html |
| AWS X-Ray | https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html |
| Amazon Managed Prometheus | https://docs.aws.amazon.com/prometheus/ |
| Amazon Managed Grafana | https://docs.aws.amazon.com/grafana/latest/userguide/what-is-Amazon-Managed-Service-Grafana.html |
| AWS Distro for OpenTelemetry | https://aws-otel.github.io/ |
| AWS CloudTrail | https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html |
| SRE Workbook (Google) | https://sre.google/workbook/table-of-contents/ |
| AWS GuardDuty | https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html |
| Amazon Inspector | https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html |
| AWS Security Hub | https://docs.aws.amazon.com/securityhub/latest/userguide/what-is-securityhub.html |
| AWS KMS | https://docs.aws.amazon.com/kms/latest/developerguide/overview.html |
| Secrets Manager | https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html |
| Systems Manager Parameter Store | https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html |
| AWS Certificate Manager | https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html |
| AWS Security Best Practices | https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html |
