# Section 16: AWS Troubleshooting Masterclass

> Part of the [AWS Interview Preparation Roadmap](./README.md). Each scenario follows: **Symptom → Investigation → Commands → Root Cause → Fix → Prevention.** Grouped by domain. Use these to answer "walk me through debugging X" questions crisply.

## How to Answer Any Troubleshooting Question

1. **Restate the symptom** and its blast radius (one user vs global).
2. **Form a hypothesis tree** (network? IAM? capacity? config? dependency?).
3. **Check the cheapest signal first** (recent change/deploy, CloudWatch, events).
4. **Narrow with commands**, confirm root cause, apply the **minimal safe fix**.
5. **Prevent recurrence** (alarm, guardrail, automation, runbook).

> Golden rule interviewers love: **"What changed?"** Most incidents follow a deploy, config, quota, or credential change.

---

## A. IAM / Identity

### A1. `AccessDenied` despite an Allow policy
- **Investigate:** explicit Deny? SCP? permission boundary? resource policy? session policy?
- **Commands:**
```bash
aws sts get-caller-identity
aws iam simulate-principal-policy --policy-source-arn <arn> --action-names s3:GetObject --resource-arns <res>
# CloudTrail: find the event, read errorCode/errorMessage
```
- **Root cause:** usually an SCP ceiling or explicit Deny.
- **Fix:** adjust the offending policy layer; grant least privilege.
- **Prevention:** policy tests in CI, Access Analyzer, standardized boundaries.

### A2. Cross-account `AssumeRole` fails
- **Cause:** trust policy principal/ExternalId mismatch, or missing `sts:AssumeRole` in source.
- **Fix:** align trust policy + ExternalId; add source permission.
- **Prevention:** IaC-managed roles, tested assume flows.

### A3. EKS pod gets `AccessDenied` to AWS API (IRSA)
- **Commands:** `kubectl describe sa <sa>`; check role trust `sub`; `kubectl exec ... aws sts get-caller-identity`.
- **Cause:** SA annotation missing, trust `sub` mismatch, OIDC provider not registered.
- **Fix:** correct annotation/trust; register OIDC provider.
- **Prevention:** module-managed IRSA; validation checks.

### A4. Leaked access key
- **Fix:** deactivate/delete key, rotate, review CloudTrail for misuse, GuardDuty.
- **Prevention:** eliminate long-lived keys (OIDC/roles), secret scanning.

---

## B. Networking

### B1. EC2 in public subnet can't reach internet
- **Commands:**
```bash
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet>
aws ec2 describe-security-groups --group-ids <sg>
aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet>
```
- **Cause:** missing IGW route, no public IP, SG egress, or NACL blocking.
- **Fix:** add `0.0.0.0/0`→IGW, assign public IP/EIP, fix SG/NACL.
- **Prevention:** standardized VPC module, Reachability Analyzer.

### B2. Private subnet instance can't reach internet
- **Cause:** no NAT route / NAT in wrong AZ / NAT down.
- **Fix:** route `0.0.0.0/0`→same-AZ NAT; per-AZ NAT.
- **Prevention:** per-AZ NAT in IaC.

### B3. Cannot reach peered VPC
- **Cause:** missing return route, NACL asymmetry, overlapping CIDR (peering non-transitive).
- **Fix:** add both-side routes; use TGW for transitive.
- **Prevention:** CIDR plan, TGW hub-and-spoke.

### B4. Intermittent high latency / cross-AZ cost
- **Cause:** traffic crossing AZs (NAT/DB in another AZ).
- **Fix:** co-locate, topology-aware routing, per-AZ NAT.

### B5. DNS resolution fails for private endpoint
- **Commands:** `dig <name>`; check private hosted zone association; endpoint private DNS enabled.
- **Fix:** associate PHZ / enable private DNS; Route 53 Resolver rules for hybrid.

### B6. ALB returns 502/504
- **Cause:** 502 = bad target response/closed connection; 504 = target timeout.
- **Commands:** target health, access logs, app logs.
- **Fix:** fix app/keep-alive/timeout mismatch; health-check path; scale targets.

### B7. Reachability Analyzer for "why can't A reach B"
```bash
aws ec2 create-network-insights-path --source <A> --destination <B> --protocol tcp --destination-port 443
aws ec2 start-network-insights-analysis --network-insights-path-id <id>
```

---

## C. EC2 / Compute

### C1. Instance fails status checks
- **Cause:** system (AWS host/network) vs instance (OS/boot) check.
- **Commands:** `aws ec2 get-console-output`; check EBS, cloud-init.
- **Fix:** stop/start (moves host) for system check; fix OS/boot for instance check.

### C2. ASG launches then terminates instances
- **Cause:** failing ELB health checks, grace period too short, bad AMI/user-data.
- **Fix:** extend health check grace, fix bootstrap, verify AMI.

### C3. `InsufficientInstanceCapacity`
- **Fix:** diversify instance types/AZs, capacity reservations, Spot pools.

### C4. Spot mass interruption
- **Fix:** capacity-optimized allocation, diversify, On-Demand base, interruption handler.

### C5. Lambda timeouts / throttling
- **Cause:** downstream slow, cold starts, concurrency limit.
- **Fix:** raise timeout/memory, provisioned concurrency, reserve concurrency, fix downstream.

---

## D. EKS / Kubernetes

### D1. Pod stuck Pending
```bash
kubectl describe pod <p>   # Events
kubectl get nodes; kubectl top nodes
```
- **Cause:** insufficient resources, taints/affinity, no IPs (CNI), unbound PVC.
- **Fix:** scale nodes (Karpenter), fix requests/tolerations, prefix delegation, storage class.

### D2. CrashLoopBackOff
```bash
kubectl logs <p> --previous; kubectl describe pod <p>
```
- **Cause:** app crash, bad config/secret, failing liveness probe.
- **Fix:** fix config/probe; correct dependency.

### D3. OOMKilled
- **Fix:** raise memory limits, fix leak, set requests==limits for guaranteed QoS.

### D4. Node NotReady
```bash
kubectl describe node <n>; journalctl -u kubelet   # on node
```
- **Cause:** kubelet down, disk/PID pressure, CNI failure, lost API connectivity.
- **Fix:** free disk, restart kubelet, fix CNI/networking.

### D5. ImagePullBackOff
- **Cause:** wrong tag, ECR auth (`ecr:GetAuthorizationToken`), missing pull secret, no route to ECR.
- **Fix:** correct tag/permissions, add ECR VPC endpoint.

### D6. DNS failures in cluster
```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl run -it dns --image=busybox --restart=Never -- nslookup kubernetes.default
```
- **Fix:** scale/repair CoreDNS, NodeLocal DNS, fix NetworkPolicy, `ndots`.

### D7. VPC CNI IP exhaustion
- **Metrics:** `awscni_total_ip_addresses` vs `assigned`.
- **Fix:** prefix delegation, larger/more subnets, custom networking.

### D8. Service has no endpoints
- **Cause:** selector mismatch or pods not Ready.
- **Fix:** align labels/selectors; fix readiness probe.

---

## E. Storage / S3 / EBS

### E1. S3 503 SlowDown
- **Cause:** request spikes per prefix.
- **Fix:** spread prefixes, retries with backoff, multipart.

### E2. S3 AccessDenied on encrypted object
- **Cause:** missing `kms:Decrypt` in addition to S3 permission.
- **Fix:** grant KMS key access.

### E3. EBS volume slow / high latency
- **Cause:** IOPS/throughput ceiling or burst-balance exhausted (gp2).
- **Fix:** gp3/io2, raise IOPS/throughput.

### E4. CRR not replicating
- **Cause:** versioning off, IAM role, filter mismatch.
- **Fix:** enable versioning, fix role/rule.

---

## F. Databases

### F1. DynamoDB throttling
- **Cause:** hot partition or under-provisioned.
- **Fix:** on-demand, write sharding, better key; check per-partition metrics.

### F2. RDS connection exhaustion
- **Cause:** too many short-lived connections (Lambda).
- **Fix:** RDS Proxy, connection pooling, raise `max_connections` cautiously.

### F3. RDS/Aurora replica lag
- **Cause:** long transactions, write bursts.
- **Fix:** scale writer, batch writes, read critical from writer.

### F4. Aurora failover confusion
- **Cause:** app used instance endpoint not cluster endpoint.
- **Fix:** use cluster/reader endpoints + retry.

---

## G. CI/CD & Deploys

### G1. Deploy succeeds but app unhealthy
- **Fix:** add ValidateService hook + alarms; verify health-check path.

### G2. Canary auto-rollback loops
- **Cause:** oversensitive alarm or bad "last good".
- **Fix:** tune thresholds, pin known-good.

### G3. GitHub Actions can't assume AWS role
- **Cause:** OIDC `sub`/`aud` mismatch.
- **Fix:** correct trust condition to exact `repo:org/name:ref:...`.

---

## H. Cost / Quotas / Events

### H1. Sudden bill spike
- **Investigate:** Cost Explorer group by service/tag; look for NAT data, cross-AZ, egress, forgotten resources.
- **Fix:** delete waste, add endpoints, right-size, budgets/anomaly detection.

### H2. `LimitExceeded` at scale-out
- **Fix:** Service Quotas increase; pre-raise before launches.

### H3. Region-wide degradation
- **Fix:** AWS Health Dashboard; fail over per DR plan; rely on static stability.

---

## Rapid-Fire Symptom → First Check (table)

| Symptom | First check |
|---------|-------------|
| AccessDenied | CloudTrail errorCode, SCP/boundary |
| Pod Pending | `kubectl describe` Events |
| CrashLoopBackOff | `kubectl logs --previous` |
| 502 from ALB | target health + app logs |
| 504 from ALB | target/timeout mismatch |
| No internet (private) | NAT route + AZ |
| DynamoDB throttle | hot partition metrics |
| RDS too many connections | RDS Proxy/pooling |
| S3 503 | prefix spread + retries |
| Bill spike | Cost Explorer by tag/service |
| Spot churn | diversify + capacity-optimized |
| DNS fail in EKS | CoreDNS pods + NetworkPolicy |

## Documentation Links

- VPC Reachability Analyzer: https://docs.aws.amazon.com/vpc/latest/reachability/
- EKS troubleshooting: https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting.html
- CloudWatch Logs Insights: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html
- AWS Health Dashboard: https://docs.aws.amazon.com/health/latest/ug/

---

> Next: **[Sections 17–18 — FAANG Rounds & Behavioral](./13-FAANG-BEHAVIORAL.md)**.
