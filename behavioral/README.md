# Behavioral Interview Questions - Complete Guide

> **200+ Behavioral Interview Questions for Senior DevOps Engineer, SRE, and Platform Engineer roles at FAANG companies**

---

## 📋 Table of Contents

- [STAR Method](#star-method)
- [Leadership & Influence](#leadership--influence)
- [Technical Leadership](#technical-leadership)
- [Conflict Resolution](#conflict-resolution)
- [Failure & Learning](#failure--learning)
- [Project Management](#project-management)
- [Customer Focus](#customer-focus)
- [Amazon Leadership Principles](#amazon-leadership-principles)
- [Google & Meta Values](#google--meta-values)

---

## STAR Method

### Framework for Behavioral Answers

```
┌─────────────────────────────────────────────────────────────────┐
│                    STAR METHOD                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  S - Situation (10-15% of answer)                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Set the context                                       │    │
│  │  • When, where, who was involved                         │    │
│  │  • Business impact at stake                              │    │
│  │  Example: "At my previous company, we had a critical     │    │
│  │  production service handling $2M daily transactions..."  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  T - Task (10-15% of answer)                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Your specific responsibility                          │    │
│  │  • What was expected of YOU                              │    │
│  │  Example: "I was the senior engineer responsible for     │    │
│  │  leading the reliability improvements..."                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  A - Action (60-70% of answer)                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Specific steps YOU took                               │    │
│  │  • Use "I" not "we"                                      │    │
│  │  • Show technical depth and leadership                   │    │
│  │  Example: "I first analyzed our monitoring gaps,         │    │
│  │  then I proposed and implemented..."                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  R - Result (10-15% of answer)                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Quantifiable impact                                   │    │
│  │  • Business outcome                                      │    │
│  │  • Lessons learned                                       │    │
│  │  Example: "As a result, we reduced MTTR by 60%,          │    │
│  │  saved $500K annually, and I was promoted..."            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  TIPS:                                                          │
│  • Prepare 6-8 detailed stories that can cover multiple topics  │
│  • Keep answers to 2-3 minutes                                  │
│  • Always include metrics and business impact                   │
│  • Practice out loud                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Leadership & Influence

### Q1: Tell me about a time you influenced a team to adopt a new technology or practice.

**Sample STAR Response:**

**Situation:**
"At my previous company, our deployment process was entirely manual, taking 4 hours per release with frequent human errors causing outages. The team was resistant to change due to past failed automation attempts."

**Task:**
"As the senior DevOps engineer, I was responsible for modernizing our deployment process and getting buy-in from both the development and operations teams."

**Action:**
"I started by documenting the current pain points with data - I tracked 3 months of deployments showing 40% had rollback-worthy issues. I then created a proof of concept with GitHub Actions for our lowest-risk service, demonstrating zero-downtime deployments. I organized lunch-and-learn sessions to address fears, paired with skeptical team members to walk through the pipeline, and created comprehensive documentation. When we hit resistance from the ops manager, I involved them early in the design, incorporated their security requirements, and gave them ownership of the approval gates."

**Result:**
"Within 6 months, we had fully automated deployments across all 12 services. Deployment time dropped from 4 hours to 15 minutes. Deployment frequency increased from weekly to multiple times daily. The ops manager who was initially resistant became the biggest advocate and presented our approach at a company tech talk."

---

### Q2: Describe a situation where you had to push back on a stakeholder's technical decision.

**Sample STAR Response:**

**Situation:**
"Our VP of Engineering wanted to migrate our entire infrastructure to a new cloud provider within 3 months to meet a vendor contract deadline, which I believed was technically infeasible and risky."

**Task:**
"I needed to provide an alternative that met business objectives while managing technical risk."

**Action:**
"I first sought to understand the underlying business need - the contract deadline. I then created a detailed analysis showing: 1) A rushed migration had 70% probability of major outage based on industry data, 2) Our SLAs would be at risk during the transition, 3) The team lacked expertise in the new platform. I proposed a phased approach: negotiate a 6-month deadline extension, migrate non-critical services first, run parallel systems during transition. I presented this with risk matrices and cost comparisons, and offered to personally lead the migration team."

**Result:**
"The VP agreed to negotiate the deadline extension, which we got. The phased migration completed in 8 months with zero customer-impacting outages. We actually saved $200K compared to the rushed approach due to better planning. The VP later cited this as an example of good technical leadership."

---

## Failure & Learning

### Q3: Tell me about a significant technical failure you were responsible for.

**Sample STAR Response:**

**Situation:**
"I once pushed a configuration change to production that brought down our entire payment processing system for 45 minutes during peak hours, affecting thousands of transactions."

**Task:**
"I needed to quickly restore service, then ensure we learned from this to prevent recurrence."

**Action:**
"First, I immediately rolled back the change and confirmed service restoration - this took 15 minutes. I then took ownership in our incident channel and led the response team. After the incident, I led the blameless postmortem where I was transparent about my mistake: I had skipped the staging environment 'just this once' because it was a 'simple change.' I identified multiple systemic issues: our deployment process allowed bypassing staging, we lacked proper config validation, and our monitoring didn't catch the issue early enough. I then personally implemented: mandatory staging gates in our CI/CD pipeline, automated config validation, and enhanced monitoring for config-related failures."

**Result:**
"We haven't had a configuration-related outage in 18 months since. More importantly, I became an advocate for deployment safety, trained 3 junior engineers on incident response, and our team's deployment confidence increased significantly. I learned that 'simple changes' often cause the worst outages, and I now apply the same rigor to all changes regardless of perceived risk."

---

### Q4: Describe a time when you had to work with incomplete information.

**Sample STAR Response:**

**Situation:**
"During a critical production incident at 2 AM, our database was experiencing severe performance degradation. Our senior DBA was unreachable, and our monitoring showed conflicting signals."

**Task:**
"I needed to restore service within our 30-minute SLA while operating without full database expertise."

**Action:**
"I gathered what data I could: slow query logs, connection counts, disk I/O metrics. Rather than making dangerous changes with incomplete knowledge, I took a conservative mitigation approach. I scaled up the database instance (more headroom), enabled connection pooling to reduce load, and killed the top 5 long-running queries that appeared non-critical. I documented every action with timestamps. Simultaneously, I escalated to our on-call chain and reached the DBA's backup. When they joined, I handed off with a complete picture of what I'd tried."

**Result:**
"Service was restored within 25 minutes. The root cause (runaway analytics query) was identified and fixed. I received positive feedback for methodical approach under pressure. I later created a runbook for database performance issues that non-DBAs could safely execute, and advocated for cross-training sessions."

---

## Amazon Leadership Principles

### Customer Obsession

**Q5: Tell me about a time you went above and beyond for a customer.**

```
┌─────────────────────────────────────────────────────────────────┐
│              AMAZON LEADERSHIP PRINCIPLES                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  14 Leadership Principles (most tested in interviews):          │
│                                                                  │
│  1. Customer Obsession                                          │
│     "Leaders start with the customer and work backwards"        │
│     → Have story about prioritizing customer needs              │
│                                                                  │
│  2. Ownership                                                   │
│     "Leaders never say 'that's not my job'"                     │
│     → Show end-to-end ownership of projects                     │
│                                                                  │
│  3. Invent and Simplify                                         │
│     "Leaders expect innovation from their teams"                │
│     → Demonstrate creative problem solving                      │
│                                                                  │
│  4. Are Right, A Lot                                            │
│     "Leaders have strong judgment and good instincts"           │
│     → Show data-driven decision making                          │
│                                                                  │
│  5. Learn and Be Curious                                        │
│     "Leaders are never done learning"                           │
│     → Share examples of self-improvement                        │
│                                                                  │
│  6. Hire and Develop the Best                                   │
│     "Leaders raise the performance bar"                         │
│     → Discuss mentoring, hiring decisions                       │
│                                                                  │
│  7. Insist on the Highest Standards                             │
│     "Leaders have relentlessly high standards"                  │
│     → Show quality improvements you drove                       │
│                                                                  │
│  8. Think Big                                                   │
│     "Thinking small is a self-fulfilling prophecy"              │
│     → Describe ambitious projects                               │
│                                                                  │
│  9. Bias for Action                                             │
│     "Speed matters in business"                                 │
│     → Show quick decision making                                │
│                                                                  │
│  10. Frugality                                                  │
│      "Accomplish more with less"                                │
│      → Cost optimization examples                               │
│                                                                  │
│  11. Earn Trust                                                 │
│      "Leaders listen attentively and speak candidly"            │
│      → Building relationships, giving feedback                  │
│                                                                  │
│  12. Dive Deep                                                  │
│      "Leaders operate at all levels, stay connected to details" │
│      → Technical deep-dives, root cause analysis                │
│                                                                  │
│  13. Have Backbone; Disagree and Commit                         │
│      "Leaders challenge decisions respectfully"                 │
│      → Pushing back, then committing                            │
│                                                                  │
│  14. Deliver Results                                            │
│      "Leaders focus on key inputs and deliver with quality"     │
│      → Metrics, outcomes, overcoming obstacles                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Google & Meta Values

### Q6: Describe a complex technical problem you solved.

```
┌─────────────────────────────────────────────────────────────────┐
│              GOOGLE HIRING ATTRIBUTES                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. General Cognitive Ability                                   │
│     • Problem-solving approach                                  │
│     • Learning ability                                          │
│     • How you structure thinking                                │
│                                                                  │
│  2. Leadership                                                  │
│     • Emergent leadership (stepping up)                         │
│     • Enabling others                                           │
│     • Taking ownership                                          │
│                                                                  │
│  3. Role-Related Knowledge                                      │
│     • Technical depth                                           │
│     • Domain expertise                                          │
│     • Practical experience                                      │
│                                                                  │
│  4. Googleyness                                                 │
│     • Ambiguity tolerance                                       │
│     • Collaborative nature                                      │
│     • Intellectual humility                                     │
│                                                                  │
│  COMMON GOOGLE BEHAVIORAL QUESTIONS:                            │
│  • "Tell me about a time you failed"                            │
│  • "Describe your most complex project"                         │
│  • "How do you handle disagreements?"                           │
│  • "Tell me about something you learned recently"               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              META CORE VALUES                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Move Fast                                                   │
│     • Speed over perfection initially                           │
│     • Iterative improvement                                     │
│     • Removing blockers                                         │
│                                                                  │
│  2. Be Bold                                                     │
│     • Taking calculated risks                                   │
│     • Ambitious goals                                           │
│     • Innovation                                                │
│                                                                  │
│  3. Focus on Long-Term Impact                                   │
│     • Building sustainable solutions                            │
│     • Infrastructure investments                                │
│     • Strategic thinking                                        │
│                                                                  │
│  4. Build Social Value                                          │
│     • Team contribution                                         │
│     • Knowledge sharing                                         │
│     • Helping others succeed                                    │
│                                                                  │
│  5. Be Open                                                     │
│     • Transparency                                              │
│     • Giving/receiving feedback                                 │
│     • Information sharing                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Technical Leadership Questions

### Q7: How do you balance technical debt with feature development?

**Sample Response:**

"I use a principled approach to balance technical debt:

**Quantify the debt**: I track technical debt with metrics - deployment frequency, incident rate by component, developer velocity. This makes the abstract concrete.

**Allocate dedicated time**: I advocate for 20% of sprint capacity for tech debt, non-negotiable. This prevents debt from compounding.

**Prioritize by impact**: Not all debt is equal. I use a 2x2 matrix of effort vs. impact. A slow build that affects 20 developers daily is higher priority than a rarely-used admin tool.

**Make it visible**: I maintain a tech debt backlog visible to product managers. When they push for more features, I can show the tradeoffs concretely.

**Bundle with features**: When possible, I bundle debt paydown with feature work. Refactoring the authentication system when adding SSO, for example.

Recent example: We had a monolithic deployment that took 45 minutes. Instead of a big-bang rewrite, I proposed incremental microservice extraction, starting with the highest-change components. Over 6 months, deployment time dropped to 5 minutes, and we shipped features faster because deploys were less risky."

---

### Q8: Describe how you've mentored junior engineers.

**Sample Response:**

"I believe effective mentoring is about creating independence, not dependence.

**Structured onboarding**: I create 30-60-90 day plans with clear milestones. First PR in week 1, first production deploy in week 2, first on-call shift (shadowed) in month 2.

**Progressive challenges**: I assign tasks slightly above their current level with guardrails. Junior engineer handles an incident? I'm shadowing, not solving for them.

**Regular 1:1s**: Beyond status updates, I focus on career growth. What do you want to learn? Where do you see yourself in 2 years?

**Document everything**: I encourage juniors to update runbooks as they learn. Teaching solidifies knowledge and improves our documentation.

**Celebrate publicly**: When a junior engineer ships something significant, I highlight it in team channels and to leadership.

Concrete example: I mentored a junior SRE who was intimidated by our complex Kubernetes setup. I started with simple pod debugging, then service meshes, then disaster recovery. After 8 months, they led our cluster upgrade project. They're now a senior engineer at another company, and I'm proud of their growth."

---

## Story Bank Template

```
┌─────────────────────────────────────────────────────────────────┐
│              PREPARE THESE STORIES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Prepare 6-8 detailed stories covering:                         │
│                                                                  │
│  □ Technical achievement (complex project)                      │
│  □ Technical failure (what you learned)                         │
│  □ Leadership without authority (influencing)                   │
│  □ Conflict resolution (disagreement with peer/manager)         │
│  □ Working with difficult stakeholder                           │
│  □ Tight deadline delivery                                      │
│  □ Mentoring/developing others                                  │
│  □ Innovation/process improvement                               │
│                                                                  │
│  EACH STORY SHOULD HAVE:                                        │
│  • Specific numbers and metrics                                 │
│  • Clear YOUR contributions (not "we")                          │
│  • Business impact                                              │
│  • What you learned                                             │
│  • 2-3 minute delivery                                          │
│                                                                  │
│  STORY MAPPING:                                                 │
│  Your stories can answer multiple question types.               │
│  Map each story to 2-3 common question themes.                  │
│                                                                  │
│  Example:                                                       │
│  "Database Migration Story" →                                   │
│    - Technical challenge                                        │
│    - Stakeholder management                                     │
│    - Risk management                                            │
│    - Tight deadline                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📚 Resources

- [Amazon Leadership Principles](https://www.amazon.jobs/en/principles)
- [Google Interview Prep](https://careers.google.com/how-we-hire/)
- [STAR Method Guide](https://www.themuse.com/advice/star-interview-method)

---

**[← Back to Main README](../README.md)** | **[Next: Learning Roadmap →](../roadmap/README.md)**
