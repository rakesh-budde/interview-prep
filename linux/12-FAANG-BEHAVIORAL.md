# Section 12: FAANG Behavioral & Leadership (Linux/SRE Context)

Technical mastery of the kernel and its subsystems is necessary but not sufficient for a senior
Linux/SRE/Platform Engineer role at a FAANG/MANGA-scale organization — every one of these interview
loops also evaluates how you handle ambiguity, incident pressure, cross-team conflict, and the
long-term health of the systems and people you work with. This section covers the behavioral
interview format itself and provides a bank of model STAR answers grounded specifically in Linux/SRE
scenarios, so your technical depth from earlier sections and your behavioral narrative reinforce each
other rather than feeling like two disconnected interview tracks.

## Subtopic Index
- [STAR Method for Linux/SRE Incidents](#star-method-for-linuxsre-incidents)
- [On-call War Stories and Postmortems](#on-call-war-stories-and-postmortems)
- [Blameless Postmortem Culture](#blameless-postmortem-culture)
- [Leading a Major Outage Response](#leading-a-major-outage-response)
- [Mentoring Engineers on Linux Internals](#mentoring-engineers-on-linux-internals)
- [Balancing Reliability vs Feature Velocity](#balancing-reliability-vs-feature-velocity)

---

## STAR Method for Linux/SRE Incidents

STAR (Situation, Task, Action, Result) is the standard structure for answering behavioral questions,
and its value for a Linux/SRE-focused interview specifically is that it forces you to demonstrate not
just *that* you fixed a problem, but the reasoning process and judgment you applied — precisely what a
senior/staff-level interviewer is actually trying to assess, since junior engineers can often describe
the same technical fix without demonstrating the same judgment about prioritization, communication,
and trade-offs under pressure. Situation should be concrete and specific (a real production system, a
real observed symptom, a real timeframe) rather than a vague hypothetical — "our checkout service was
returning elevated 5xx errors starting at 2:14am" is a far stronger opener than "we had a performance
issue once." Task should clarify what specifically you, personally, were responsible for and what
constraints you were operating under (were you the incident commander, a contributing responder, the
person who ultimately made the fix decision, operating under an SLA with real customer impact
accumulating). Action is where your technical depth from earlier sections in this guide should surface
naturally and specifically — not "I looked at the logs" but "I ran `ss -tan state time-wait | wc -l`
and confirmed we'd exhausted the ephemeral port range for outbound connections, which correlated
exactly with the connection-timeout errors we were seeing" — concrete tool names, concrete
observations, and the actual reasoning chain from symptom to root cause. Result should quantify impact
wherever genuinely possible (reduced MTTR from X to Y, eliminated a recurring class of incident,
prevented a specific projected cost) and, critically, should include what you or the team changed
afterward as a direct result of the incident (a new monitor, a runbook update, an automated
remediation) — an answer that ends at "and then it was fixed" without any forward-looking systemic
improvement misses the single most senior-level-differentiating part of the story, since junior
engineers fix incidents but senior/staff engineers are expected to demonstrably prevent classes of
future incidents from the lessons of past ones.

## On-call War Stories and Postmortems

Interviewers ask for "a time you were on call and something went wrong" specifically to probe how you
behave under genuine operational pressure, not merely whether you can recite a kernel internals fact
correctly in a calm setting — the strongest answers demonstrate a calm, structured triage process even
when recalling a stressful moment. A strong on-call narrative typically follows a recognizable
structure: how you were alerted and what your very first diagnostic steps were (ideally demonstrating
the systematic USE-method-style discipline from Section 8 rather than random guessing under pressure);
how you decided what to communicate and to whom, and when (a senior engineer proactively communicates
status/impact to stakeholders on a predictable cadence during a long incident, rather than going silent
while heads-down debugging, since stakeholder anxiety and business impact both compound when
communication is absent); how you made a judgment call under incomplete information (production
incidents rarely present with perfectly clean, unambiguous symptoms, and a mature answer acknowledges
genuine uncertainty in the moment rather than претending the root cause was obvious from the start);
and how the incident concluded, including the immediate mitigation (which is often different from,
and faster than, the true root-cause fix — restarting a service or failing over to a healthy replica
to restore customer impact immediately, while the deeper root-cause investigation continues afterward
without that time pressure) versus the eventual permanent fix. A postmortem (or "post-incident review")
is the structured written artifact produced after significant incidents, and being able to describe
your organization's postmortem process specifically — what sections it includes (timeline, impact,
root cause, contributing factors, action items with owners and due dates), who reviews it, and how
action items are tracked to actual completion rather than being written down and forgotten — is itself
a meaningful signal of operational maturity interviewers are listening for, since a postmortem that
doesn't produce tracked, completed action items is a purely ceremonial exercise providing no real
organizational learning.

## Blameless Postmortem Culture

Blameless postmortem culture is the explicit organizational commitment to investigating incidents by
asking "what in our systems, processes, and tooling allowed this to happen" rather than "who made the
mistake" — a distinction with real, practical consequences for incident response quality that
interviewers specifically probe for because it reveals how you think about systemic reliability versus
individual fault. The core insight blameless culture is built on is that individual human error is
rarely, on its own, a sufficient or actionable root cause: if an engineer fat-fingered a destructive
command, the more useful and more preventable underlying questions are why the tooling allowed that
destructive command to be run without a confirmation step or a dry-run preview, why insufficiently
restrictive access controls (Section 6) permitted it in the first place, and why no automated
safeguard caught it before real impact occurred — a culture that instead simply blames and possibly
disciplines the individual engineer both fails to fix any of those systemic gaps (guaranteeing a
similar incident recurs, caused by a different individual next time) and, more insidiously, actively
discourages future incident responders and postmortem contributors from being fully honest about what
actually happened, since fear of blame incentivizes minimizing, omitting, or reframing details in a
way that specifically undermines the accuracy the whole exercise depends on. When asked to describe
your experience with blameless postmortems, the strongest answers describe a concrete instance where
you (or someone else) made a genuine mistake during an incident, and instead of that being treated as
a personal failing, the retrospective focused on and produced a systemic fix (better tooling
guardrails, an improved runbook, an added automated check) that would have prevented the mistake
regardless of which specific individual was involved — demonstrating you both understand the
principle and have lived it in practice, not merely learned the phrase.

## Leading a Major Outage Response

Leading (not merely participating in) a major outage response is a distinctly different skill from
strong individual technical troubleshooting, and senior/staff-level interviews specifically probe for
it because the skills genuinely diverge — the strongest individual debugger is not automatically the
strongest incident commander, since incident command is fundamentally a coordination and
decision-prioritization role, not primarily a hands-on-keyboard technical one. A strong incident
commander explicitly separates the roles of "who is actively debugging/fixing" from "who is
coordinating, tracking status, and communicating externally," resisting the natural but counter-
productive urge to personally dive into every technical rabbit hole themselves once multiple responders
are engaged, since doing so removes the one person whose job is maintaining overall situational
awareness and making prioritization calls across parallel workstreams. Concretely narrating this
requires describing how you established a clear timeline and communication cadence (a running incident
channel with regular status updates, a single source of truth for current understanding rather than
information scattered across side conversations), how you made explicit trade-off decisions under
time pressure (mitigating customer impact immediately via a faster, imperfect fix like failing over to
a degraded-but-functional mode, versus continuing to pursue a fully correct root-cause fix, and being
transparent about which trade-off you deliberately chose and why), and how you knew when to escalate
or bring in additional expertise rather than letting sunk-cost thinking keep the existing team
struggling alone past the point where fresh expertise would clearly help. A genuinely strong answer
also acknowledges a moment of uncertainty or a decision that, in retrospect, could have gone better —
interviewers are specifically skeptical of narratives where every single decision during a major
incident was obviously and immediately correct, since that pattern more often signals a rehearsed,
sanitized story than a real, honestly-recalled experience.

## Mentoring Engineers on Linux Internals

Being asked to describe how you've mentored others specifically on deep technical material (rather
than general career mentorship) tests whether you can translate the kind of expert-level Linux
internals knowledge covered throughout this guide into something a less experienced engineer can
actually absorb and apply — a genuinely distinct skill from possessing the knowledge yourself. Strong
answers describe a specific, concrete teaching moment (not "I mentor junior engineers regularly" in
the abstract) — walking a junior engineer through a real production incident's root cause using it as
a live teaching opportunity (explaining *why* `D`-state processes inflate load average while the CPU
sits idle, in the context of an actual incident you were jointly debugging, rather than as an abstract
lecture disconnected from a real, motivating problem) tends to land far more memorably and
persuasively than a description of formal, scheduled training sessions alone. Equally important is
describing how you calibrated the depth and pace of explanation to the mentee's actual current level
of understanding, rather than delivering the same maximally-detailed internals explanation regardless
of audience — a genuinely skilled mentor recognizes when a simplified, analogy-based first-pass
explanation ("think of `vruntime` like a fairness ledger tracking how much CPU time each task has
already gotten") is the right starting point before layering in the full red-black-tree/scheduling-
class mechanistic detail, versus when a mentee is ready for and benefits from the complete depth
immediately. Interviewers are also listening for evidence that mentoring was a two-way, sustained
relationship rather than a single one-off explanation — following up afterward to confirm the concept
actually stuck (perhaps by having the mentee independently diagnose a similar issue later, or explain
the concept back in their own words), and being genuinely open about your own gaps or moments you
didn't know an answer and worked through it together with the mentee, which models the kind of
intellectual honesty and continuous learning that makes for a genuinely strong long-term technical
mentor rather than someone merely performing expertise.

## Balancing Reliability vs Feature Velocity

Nearly every senior/staff Linux/SRE/Platform interview loop includes some version of "tell me about a
time you had to push back on a deadline for reliability reasons" or "how do you balance reliability
work against feature delivery pressure," because this tension is one of the most persistent, genuinely
difficult aspects of the role, and how you navigate it reveals both your technical judgment and your
organizational/communication skill simultaneously. A weak answer describes reliability and feature
velocity as a zero-sum conflict resolved purely through personal advocacy or authority ("I told them we
couldn't ship until it was fixed"); a strong answer instead describes translating a reliability concern
into terms the broader organization (including non-infrastructure stakeholders) can genuinely weigh
against feature delivery value — quantifying the actual risk (a specific failure mode's probability
and blast radius, ideally backed by concrete data: "at our current growth rate we project hitting this
conntrack ceiling within six weeks, at which point new customer signups would start failing outright")
rather than an unquantified, purely qualitative appeal to caution, and proposing a genuinely
incremental path (a smaller, faster mitigation that meaningfully reduces risk without requiring the
full, larger reliability investment to be completed before any feature work can proceed at all) rather
than presenting reliability work and feature work as strictly sequential, mutually exclusive
alternatives. The concept of an error budget (a data-driven, pre-agreed-upon acceptable rate of
unreliability, explicitly negotiated in advance between reliability and product stakeholders,
providing a standing, objective mechanism for deciding "do we have room to take on more risk for a
feature launch, or has our error budget for this period already been consumed by recent incidents")
is worth being able to discuss concretely as the structural, non-adversarial mechanism many
mature organizations use specifically to make this trade-off explicit and data-driven rather than an
ad-hoc argument repeated fresh every single time it arises — describing genuine, lived experience
either using or helping establish something like an error budget is a strong, senior-level signal.

---

### 15 Behavioral Questions with Model STAR Answers

**1. Tell me about a production incident you diagnosed and resolved that required deep Linux
internals knowledge.**
- *Situation*: A payment-processing service began intermittently timing out under moderate load,
  with CPU and memory utilization on affected hosts looking unremarkable in standard dashboards.
- *Task*: As the on-call engineer, I needed to identify the root cause before the next peak-traffic
  window, since the intermittent nature meant it could become far more severe under higher load.
- *Action*: I checked load average first and found it elevated despite low CPU utilization —
  immediately suggesting `D`-state processes rather than a CPU bottleneck. `ps -eo stat` confirmed a
  cluster of uninterruptible-sleep processes, and `iostat -x` showed a shared network storage backend
  with `await` times spiking into the hundreds of milliseconds. I traced this to a specific storage
  volume shared with an unrelated batch job that had recently increased its I/O footprint, creating
  contention neither team had visibility into.
- *Result*: I worked with the batch job's owning team to move it to a separate volume, immediately
  resolving the timeouts. I also added D-state-process-count and storage-`await` monitoring to our
  standard dashboard, since load average alone had been misleadingly reassuring throughout the
  incident, and documented this diagnostic pattern in our on-call runbook.

**2. Describe a time you had to make a difficult trade-off decision during an active incident.**
- *Situation*: During a major outage, our primary database's replication lag had grown severe enough
  that failing over to the replica risked measurable data loss, but continuing on the primary meant
  ongoing customer-facing errors.
- *Task*: As incident commander, I had to decide between accepting a bounded, quantifiable amount of
  data loss to restore service quickly, or continuing to investigate a same-primary fix with unknown
  time-to-resolution.
- *Action*: I quickly quantified the actual data-loss window (based on the measured replication lag)
  and communicated both options with their concrete trade-offs to the relevant stakeholders (including
  a data/compliance representative, since data loss had implications beyond pure engineering) rather
  than making the call unilaterally and silently, given the genuine business-impact dimension of the
  choice.
- *Result*: We collectively chose to fail over, accepting a documented, bounded data-loss window,
  restoring service within minutes rather than continuing an open-ended investigation. Afterward, we
  identified and fixed the specific replication-lag root cause and added an automated alert firing well
  before lag reaches a failover-risk threshold, so future incidents have more decision time available
  before this same trade-off becomes urgent.

**3. Tell me about a time you disagreed with a decision made during an incident and how you handled
it.**
- *Situation*: During a live incident, another senior engineer proposed immediately restarting a
  cluster of application servers as the fix, but I suspected (based on recent log patterns) the actual
  issue was a poison-pill message in a shared queue that would simply recur after any restart.
- *Task*: I needed to raise this disagreement constructively without stalling the ongoing mitigation
  effort or undermining the incident commander's authority to make the final call.
- *Action*: I voiced my specific, evidence-based concern directly in the incident channel (rather than
  a side conversation), proposed a quick, low-cost way to test my hypothesis in parallel (inspecting
  the queue for the suspected poison message) while the restart proceeded as the immediate mitigation
  regardless, so we weren't blocking on resolving the disagreement before taking any action at all.
- *Result*: The restart provided temporary relief as expected, and my parallel investigation confirmed
  the poison-pill message shortly afterward, letting us apply the actual fix (purging the specific
  message) before the issue could recur. The incident commander explicitly credited the parallel-
  investigation approach afterward as a pattern worth repeating for future incidents with competing
  hypotheses.

**4. Describe a time you contributed to a blameless postmortem after making a mistake yourself.**
- *Situation*: I ran a cleanup script against what I believed was a decommissioned host, but it was
  actually still serving low-volume production traffic due to a stale inventory record.
- *Task*: I needed to both restore the affected service quickly and be fully transparent about my own
  role in the postmortem that followed, despite the natural instinct to minimize my own mistake.
- *Action*: I immediately flagged the issue myself the moment I noticed the impact, restored the
  service from a recent backup, and in the postmortem explicitly walked through exactly what I did and
  why the stale inventory record made it seem safe, rather than describing the incident vaguely.
- *Result*: The postmortem's action items focused entirely on the systemic gap (the inventory system
  had no automated verification against actual host activity) rather than on my individual action,
  leading to an automated pre-flight check being added to the cleanup tooling that verifies genuine
  inactivity before allowing any destructive operation — a fix that has since prevented at least two
  similar near-misses by other engineers.

**5. Tell me about a time you had to explain a complex Linux/kernel concept to a less experienced
engineer or a non-technical stakeholder.**
- *Situation*: A junior engineer on my team was confused why our monitoring showed high load average
  on a host with plenty of idle CPU, and separately, a product manager needed to understand why we
  couldn't simply "add more CPU" to fix a customer-reported slowness issue.
- *Task*: I needed to give each audience an explanation calibrated to their actual background and
  what decision they needed to make with the information.
- *Action*: For the junior engineer, I walked through the actual mechanism live during our joint
  debugging session — showing `ps -eo stat`, explaining uninterruptible sleep, and connecting it
  directly to the real symptom we were looking at together. For the product manager, I used a
  simpler analogy (comparing it to a warehouse with plenty of workers but a jammed loading dock) to
  convey that the bottleneck was I/O, not raw compute capacity, which was the actual decision-relevant
  fact they needed (that adding CPU wouldn't help, but addressing the storage backend would).
- *Result*: The junior engineer independently diagnosed a similar D-state issue on their own a few
  weeks later, confirming the explanation had genuinely stuck rather than just being nodded along to.
  The product manager correctly redirected the relevant budget conversation toward storage
  infrastructure instead of compute scaling, based on accurately understanding the real bottleneck.

**6. Describe a time you pushed back on a feature deadline for reliability reasons.**
- *Situation*: A new feature launch was scheduled that would significantly increase write volume to a
  database cluster already operating close to its known I/O capacity ceiling, based on our own
  capacity planning data.
- *Task*: I needed to raise this risk clearly enough to actually change the launch plan, without
  simply saying "no" in a way that would be dismissed as generic engineering caution.
- *Action*: I quantified the specific risk using our existing capacity planning data (projected write
  volume against measured I/O headroom) and proposed a concrete, incremental alternative — a phased
  rollout ramping traffic gradually with explicit monitoring gates, rather than either a full delay or
  an unmitigated full-volume launch.
- *Result*: The team adopted the phased rollout; we caught the actual I/O ceiling being approached
  during the second rollout phase (validating the original concern) and had time to provision
  additional capacity before it became customer-impacting, allowing the feature to still launch on
  essentially the original timeline with the risk properly managed rather than ignored.

**7. Tell me about a time a monitoring/alerting gap contributed to an incident being detected late.**
- *Situation*: A slow memory leak in a service went undetected for weeks because our monitoring
  tracked only aggregate memory usage, which grew gradually enough to look like normal, expected growth
  rather than a leak, until an OOM kill finally occurred.
- *Task*: After the incident, I needed to both understand why our existing monitoring missed this and
  propose a genuinely better detection mechanism, not just a reactive alert on the symptom that had
  already occurred.
- *Action*: I analyzed the actual memory growth pattern retrospectively and identified that per-
  process RSS growth rate (not just absolute level) would have flagged the leak weeks earlier, well
  before it became critical, and implemented that as a new proactive alert.
- *Result*: The new growth-rate-based alert has since caught two subsequent memory leaks in other
  services within days of introduction rather than weeks, directly demonstrating the value of the
  changed detection approach with concrete before/after evidence.

**8. Describe a time you had to lead an incident response involving multiple teams.**
- *Situation*: A cascading failure during a traffic spike involved our infrastructure team's load
  balancers, a separate application team's service, and a third team's shared database, with no single
  team having full visibility into the whole chain.
- *Task*: As the incident commander, I needed to coordinate across all three teams' engineers, who
  didn't normally work together directly, toward a shared understanding of the failure chain.
- *Action*: I established a single incident channel as the shared source of truth, explicitly assigned
  each team's lead a specific investigation scope to avoid duplicated effort, and personally maintained
  the overall timeline/status rather than diving into any one team's specific technical investigation
  myself, checking in with each workstream at a regular cadence.
- *Result*: We identified the full cascading chain (load balancer health-check timeout tuning that
  amplified an initially minor database slowdown into full service failure) within 40 minutes, faster
  than any single team likely would have achieved investigating in isolation, and the postmortem
  produced coordinated action items across all three teams.

**9. Tell me about a time you automated a manual, error-prone process.**
- *Situation*: Our team manually applied kernel security patches host-by-host following a checklist,
  and the process was both slow and occasionally inconsistently followed under time pressure.
- *Task*: I wanted to reduce both the time cost and the human-error risk of this recurring operational
  burden.
- *Action*: I built an idempotent automation script (following the production-grade scripting
  discipline from Section 10 — `set -euo pipefail`, proper trap-based cleanup, a dry-run mode) that
  encoded the checklist's logic directly, including the canary/staged-rollout discipline from Section
  11 rather than applying patches fleet-wide at once.
- *Result*: Patch rollout time dropped substantially, and the automation's built-in staged rollout
  caught a regression during a canary wave on one later rollout that a rushed, purely manual process
  under time pressure likely would have missed until it had already spread further.

**10. Describe a time you had to say no to a request that would have compromised system security or
reliability.**
- *Situation*: A team requested a broad, unscoped `CAP_SYS_ADMIN` capability grant for a container to
  quickly resolve a permission error they were blocked on.
- *Task*: I needed to decline the overly broad request without simply blocking their progress
  outright, since they had a genuine, time-sensitive need.
- *Action*: I investigated with them what specific narrow capability their actual operation required,
  found it was a much narrower, specific capability rather than the broad grant they'd defaulted to
  requesting, and helped them apply that instead.
- *Result*: They were unblocked within the same day with a properly minimal capability grant, and this
  became the standard example I've since used when mentoring other teams on why "just grant
  `CAP_SYS_ADMIN`" is almost never actually the right fix for a permission error, directly informed by
  the capability model covered in Section 6.

**11. Tell me about a time you had to recover from a mistake that caused a production incident.**
- *Situation*: I introduced a change to a shared configuration management template that inadvertently
  disabled a security-relevant firewall rule across a subset of hosts.
- *Task*: I needed to both remediate the immediate exposure and take ownership of the mistake
  transparently.
- *Action*: I identified the affected hosts immediately upon discovering the issue, reverted the
  change, and proactively opened an incident and notified the security team rather than quietly
  reverting and hoping the brief exposure window went unnoticed.
- *Result*: No exploitation of the exposure window was found upon investigation, and the postmortem
  led to adding an automated policy check specifically validating firewall rule presence as part of
  the configuration management pipeline's own testing, preventing this specific class of regression
  from reaching production again regardless of which engineer might introduce a similar future change.

**12. Describe a time you improved a runbook or process based on lessons from an incident.**
- *Situation*: During an incident, our existing runbook for "service unresponsive" investigation
  didn't include checking for conntrack table exhaustion, a cause we ultimately found only through ad-
  hoc investigation that took longer than it should have.
- *Task*: I wanted to make sure the next engineer facing a similar symptom wouldn't need to rediscover
  this diagnostic path from scratch.
- *Action*: I updated the runbook with an explicit, ordered diagnostic checklist informed by the USE
  method (Section 8), including the specific `conntrack -L | wc -l` versus `nf_conntrack_max` check
  that had resolved this particular incident, alongside the reasoning for why each check matters.
- *Result*: The updated runbook was used successfully by a different on-call engineer during a later,
  unrelated incident with similar symptoms, who resolved it significantly faster by following the
  updated checklist rather than needing to rediscover the same diagnostic path independently.

**13. Tell me about a time you had to make a decision with incomplete information during an
incident.**
- *Situation*: During an outage, two plausible root causes existed (a recent deployment and an
  unrelated infrastructure change happening around the same time), and we didn't have time to fully
  confirm either before customer impact became severe.
- *Task*: I needed to decide which mitigation to pursue first without certainty about the true root
  cause.
- *Action*: I chose to roll back the recent deployment first, reasoning it was the lower-risk, faster-
  to-reverse action regardless of whether it was the actual root cause, while continuing to
  investigate the infrastructure change in parallel rather than waiting for full certainty before
  acting at all.
- *Result*: The rollback resolved the issue, confirming the deployment as the actual root cause, and I
  explicitly documented in the postmortem that this was an educated bet under uncertainty (lowest-risk,
  fastest-to-reverse action first) rather than implying we had certain knowledge from the start, which
  the team later adopted as an explicit stated principle for similar dual-hypothesis situations.

**14. Describe a time you had to balance urgent operational work against planned project work.**
- *Situation*: I was in the middle of a planned infrastructure migration project when a recurring,
  moderate-severity incident pattern began consuming significant on-call time.
- *Task*: I needed to decide how to allocate my own time between the two, and to make that trade-off
  visible to my manager rather than silently absorbing the conflict myself.
- *Action*: I proposed a short, explicit pause on the migration project specifically to root-cause and
  permanently fix the recurring incident pattern, quantifying the ongoing on-call time cost it was
  creating against the migration project's own timeline cost, and got explicit agreement on the
  trade-off rather than deciding it unilaterally.
- *Result*: The root-cause fix eliminated the recurring incident entirely, and the migration project
  resumed with a clearer runway (no longer competing with recurring on-call interruptions), ultimately
  completing only slightly later than originally planned but without ever needing to interrupt it again.

**15. Tell me about a time you mentored someone and saw them grow as a result.**
- *Situation*: A newer engineer on my team was capable but hesitant to lead incident response,
  consistently deferring to more senior engineers even when they had clearly already identified the
  correct root cause themselves.
- *Task*: I wanted to build their confidence and visibility as an incident responder, not just their
  raw technical skill, which was already solid.
- *Action*: During a lower-severity incident where I was confident they could handle it, I deliberately
  stepped back and let them drive, staying available but not taking over, and afterward gave specific,
  concrete feedback on what they'd done well rather than generic encouragement.
- *Result*: They led several subsequent incidents independently and successfully, and were promoted to
  a senior on-call role within the following review cycle — a change I specifically attribute to
  deliberately creating space for them to practice leading, not just accumulating more individual
  technical knowledge, which they already had.
