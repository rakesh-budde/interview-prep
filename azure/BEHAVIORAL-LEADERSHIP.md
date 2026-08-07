# Behavioral & Leadership Interview Guide

> STAR format answers, incident management, technical decision-making for senior roles

**Coverage:** 50+ behavioral scenarios | **Focus:** Leadership principles, conflict resolution

---

## STAR Format Examples

### Q1: Tell me about a time you led a critical incident response

**Situation:**
Production database for our payment service went down at 2 AM on Black Friday, affecting 50,000 concurrent users. Error rate spiked to 100%, revenue loss was $50K/minute.

**Task:**
As the on-call SRE, I needed to restore service immediately, coordinate with 5 teams across time zones, and communicate with executives every 15 minutes.

**Action:**
```
Timeline:
2:00 AM - PagerDuty alert triggered
2:02 AM - Joined incident Slack channel, declared SEV-1
2:05 AM - Identified Azure SQL failover had failed
2:10 AM - Engaged DBA team (escalation)
2:15 AM - Decision: Manual failover vs wait for auto-recovery
2:20 AM - Executed manual failover to secondary region
2:25 AM - Service restored (partial)
2:30 AM - Full recovery confirmed
2:45 AM - Customer communication sent
3:00 AM - Incident downgraded to SEV-3 (monitoring)

Key decisions I made:
1. Escalated immediately (didn't try to fix alone)
2. Made quick decision on manual failover (calculated risk)
3. Delegated communication to product manager
4. Focused technical team on root cause
```

**Result:**
- Service restored in 25 minutes (SLA target: 30 min)
- Revenue loss: ~$1.25M (vs potential $5M if delayed)
- Postmortem identified auto-failover bug
- Fixed permanently, no recurrence in 18 months
- Promoted to Senior SRE based on incident leadership

---

### Q2: Describe a technical decision you made that had significant impact

**Situation:**
Our AKS cluster was experiencing frequent pod evictions due to memory pressure. Engineering teams complained about instability affecting their releases.

**Task:**
Design and implement a solution that improves cluster stability while not over-provisioning (cost concern from finance).

**Action:**
```
Analysis (Week 1):
- Collected metrics: 40% of pods had memory requests = limits
- This caused OOM kills when memory spiked
- No headroom for garbage collection

Options considered:
A) Increase node size (expensive, $50K/year)
B) Add more nodes (moderate cost, complexity)
C) Implement Vertical Pod Autoscaler (VPA) + right-sizing
D) Set memory requests = 80% of limits (allow headroom)

Decision: Option D (immediate) + Option C (long-term)

Implementation:
- Updated all deployment templates: requests = 80% of limits
- Deployed VPA in recommendation mode
- Created dashboard showing memory efficiency
- Trained teams on resource management
```

**Result:**
- Pod evictions reduced by 85%
- No additional cloud spend ($0 cost)
- Release success rate improved from 87% to 98%
- Pattern adopted by 3 other teams
- Documented in engineering best practices

---

### Q3: Tell me about a conflict with a colleague and how you resolved it

**Situation:**
Senior developer wanted to deploy directly to production without staging testing. I (as DevOps lead) required all deployments go through staging. Conflict escalated in team meeting.

**Task:**
Resolve disagreement while maintaining deployment quality and preserving working relationship.

**Action:**
```
Step 1: Private conversation
- Scheduled 1:1 to understand their perspective
- Learned: Staging was slow (2 hours), blocking releases
- Their goal: Ship features faster (valid concern)

Step 2: Find common ground
- Both wanted: Fast, reliable deployments
- I wanted: Quality gates
- They wanted: Speed

Step 3: Collaborative solution
- Proposed: Parallel staging (20 min instead of 2 hours)
- I invested 2 days automating staging pipeline
- They agreed to always use staging (faster now)

Step 4: Follow through
- Implemented parallel testing
- Staging time: 2 hours → 25 minutes
- Both satisfied with outcome
```

**Result:**
- Staging pipeline 5x faster
- Zero production incidents from skipped testing
- Relationship strengthened (collaborated on 3 more projects)
- Solution adopted company-wide
- Received peer recognition for collaboration

---

### Q4: Describe a time you had to make a decision with incomplete information

**Situation:**
During a security incident, we detected potential data exfiltration. Logs were incomplete (only 60% visibility). Had to decide: shut down service (certain business impact) or investigate further (risk of more data loss).

**Task:**
Make rapid decision balancing security risk vs business continuity.

**Action:**
```
Information available:
- Unusual API calls from single IP
- 10GB data transferred (could be legitimate or malicious)
- Logs showed access to customer database
- Customer PII potentially exposed

Information NOT available:
- Whether data was actually exfiltrated (logs incomplete)
- Attacker's entry point
- Scope of access

Decision framework:
1. What's worst case if I do nothing? (Major data breach, regulatory fines)
2. What's worst case if I shut down? (4 hours revenue loss = $200K)
3. Probability analysis: 40% chance of real breach

Decision: Shut down service, investigate
- Communicated to executives immediately
- Engaged security team
- Preserved logs for forensics
```

**Result:**
- Investigation revealed: False positive (legitimate batch job)
- Service restored in 2 hours (not 4)
- Revenue loss: $100K (less than worst case)
- Executives praised "err on side of caution" approach
- Improved logging to prevent future ambiguity
- No regrets: Would make same decision again

---

### Q5: How do you handle pushback on technical recommendations?

**Situation:**
Recommended migrating from VMs to Kubernetes. CTO skeptical: "Too complex, team doesn't know it."

**Task:**
Convince leadership while respecting their concerns.

**Action:**
```
Step 1: Understand objections
- Complexity: Valid (Kubernetes learning curve)
- Team skills: Valid (no K8s experience)
- Hidden: Previous container project failed

Step 2: Address each concern with data
- Complexity: "We'll use managed AKS, not self-hosted"
- Team skills: "I'll lead training program (2 weeks)"
- Previous failure: "That was Docker Swarm, different tech"

Step 3: Propose pilot
- "Let's try non-critical service first"
- "If it fails, we revert (low risk)"
- "If it succeeds, we have proof point"

Step 4: Deliver results
- Pilot succeeded (3 weeks, internal tool)
- Metrics: 40% faster deployments
- Team feedback: "Easier than expected"
```

**Result:**
- CTO approved broader migration
- 18 months later: 80% of services on AKS
- Cost reduced 25% (better resource utilization)
- Deployment frequency: Weekly → Daily
- CTO became advocate for platform modernization

---

## Leadership Principles

### Ownership
"I take ownership of problems even outside my immediate scope."

Example: Noticed billing alerts weren't working. Not my team's responsibility, but I fixed it anyway, preventing $50K overrun.

### Bias for Action
"I prefer making decisions with 70% information rather than waiting for 100%."

Example: During incident, chose manual failover with incomplete data. Restored service in 25 min vs potential 2+ hours waiting for full diagnosis.

### Earn Trust
"I build trust through transparency and delivering on commitments."

Example: When migration took longer than estimated, I communicated early, adjusted timeline, and delivered on revised date. Team trusted my future estimates.

### Dive Deep
"I investigate root causes, not just symptoms."

Example: Pod failures seemed random. Investigated for 3 days, found subtle memory leak in sidecar. Fixed permanently.

### Disagree and Commit
"I voice concerns, but commit fully once decision is made."

Example: Disagreed with vendor choice (preferred open-source). After team decided, I became vendor's champion and made implementation successful.

---

## Incident Management Framework

```
SEV-1 (Critical): Revenue impact, customer-facing outage
├─ Response time: 5 minutes
├─ Communication: Every 15 minutes
├─ Escalation: VP-level within 30 minutes
└─ Postmortem: Required within 48 hours

SEV-2 (High): Degraded service, partial outage
├─ Response time: 15 minutes
├─ Communication: Every 30 minutes
├─ Escalation: Director-level within 1 hour
└─ Postmortem: Required within 1 week

SEV-3 (Medium): Non-critical service impact
├─ Response time: 1 hour
├─ Communication: Daily update
├─ Escalation: Manager-level as needed
└─ Postmortem: Optional

INCIDENT COMMANDER RESPONSIBILITIES:
1. Declare incident severity
2. Assign roles (communications, technical lead)
3. Make rapid decisions
4. Escalate when needed
5. Document timeline
6. Lead postmortem
```

This covers behavioral and leadership interview preparation for senior roles.
