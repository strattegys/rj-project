# RJ Project - Learning & Feedback Engine
## How the Agent Gets Smarter Over Time

**Date:** 2026-04-16

---

## The Core Idea

Traditional monitoring is **reactive and static**: you set thresholds, you get alerts, you investigate manually. The thresholds don't learn, and the same type of incident surprises you every time.

This agent should be **proactive and adaptive**:
- It learns what matters to THIS client, not just what's technically abnormal
- It spots slow-moving trends that no one is watching
- It gets feedback on its own outputs and adjusts
- Over time, it becomes an institutional knowledge store that knows this infrastructure better than any single engineer

---

## Three Learning Loops

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   LOOP 1: OUTCOME FEEDBACK (explicit, fast)                 │
│   "Was this alert useful? Was my diagnosis right?"          │
│                                                             │
│   LOOP 2: BEHAVIORAL OBSERVATION (implicit, medium)         │
│   "What did the humans actually do after my alert?"         │
│                                                             │
│   LOOP 3: PATTERN DISCOVERY (proactive, slow)               │
│   "What's changing that nobody asked me to watch?"          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Loop 1: Outcome Feedback

The most direct learning signal. After every escalation, the agent asks: **"Did I get this right?"**

### How it works:

```
Agent escalates an incident to Slack
    │
    v
Engineer receives alert with action buttons:
    [✓ Useful]  [✗ Noise]  [~ Partially Right]
    │
    v
If engineer clicks "Useful":
    → Agent logs: this detection pattern = valuable
    → Confidence weight for similar patterns goes UP
    → If engineer also resolves it, agent logs the resolution

If engineer clicks "Noise":
    → Agent asks: "Why was this noise?" (dropdown or free text)
       - Known maintenance window
       - Expected behavior after deploy
       - Threshold too sensitive
       - Wrong team / not my problem
       - Other: ___________
    → Agent logs the suppression reason
    → Adjusts future detection accordingly

If engineer clicks "Partially Right":
    → Agent asks: "What did I get wrong?"
       - Right problem, wrong severity
       - Right symptom, wrong root cause
       - Right alert, wrong team
       - Missed the actual root cause
    → Agent logs the correction
```

### What the agent learns from this:

- **Noise calibration**: "This client doesn't care about pod restarts under 3 in an hour — stop alerting on that"
- **Severity calibration**: "I keep marking database latency as MEDIUM but the team treats it as HIGH — adjust"
- **Root cause accuracy**: "My database diagnosis was wrong — it was actually a DNS issue. Next time, check DNS earlier in the investigation"
- **Team routing**: "Payment alerts go to #payments-oncall, not #platform-oncall"

### Storage:

```yaml
feedback_record:
  incident_id: "INC-2026-04-16-001"
  detection_pattern: "error_rate_spike + new_exception_type"
  agent_severity: "HIGH"
  agent_root_cause: "database connection pool exhaustion"
  agent_confidence: 0.85
  
  human_feedback:
    useful: true
    actual_severity: "HIGH"          # confirmed
    actual_root_cause: "database connection pool exhaustion"  # confirmed
    resolution: "increased pool size from 100 to 200"
    time_to_resolve: "12 minutes"
    feedback_by: "sarah.chen"
    feedback_at: "2026-04-16T14:45:00Z"
    notes: "Good catch — we didn't have an alert for connection pool usage"
```

---

## Loop 2: Behavioral Observation

Not every engineer will click feedback buttons. The agent should also learn by **watching what happens** after it escalates.

### Signals the agent can observe passively:

```
SIGNAL                              WHAT IT MEANS
──────────────────────────────────  ──────────────────────────────────
Alert posted → no response          Probably noise (or team is busy)
  for 2 hours

Alert posted → engineer             Useful alert, engineer investigated
  queries same service in            on their own
  Grafana within 10 minutes

Alert posted → deployment            Alert was real, engineer deployed
  to same service within              a fix
  30 minutes

Alert posted → Jira ticket           Alert was real, but not urgent
  created next business day           enough for immediate action

Alert posted → engineer              Alert was real AND critical —
  pages additional people             needed more hands

Alert posted → engineer              My investigation was helpful,
  copies my suggested fix             they used my recommendation
  into a PR

Alert posted → engineer runs         I missed something — the engineer
  queries I DIDN'T suggest            had to investigate further on
                                      their own

Alert posted → same issue            My suggested fix didn't work
  recurs within 24 hours              or wasn't applied
```

### What the agent learns from this:

- **Which alerts get ignored** → reduce severity or suppress
- **Which alerts trigger immediate action** → these are the high-value detections
- **What investigation steps engineers take that the agent didn't** → add those to the investigation playbook
- **Which suggested fixes actually get applied** → weight those remediation patterns higher
- **Recurrence patterns** → the fix didn't stick, flag for deeper investigation next time

### Implementation:

The agent monitors the same tools it already polls (Slack message reactions/threads, Jira ticket creation, deployment events, Grafana query logs) and correlates activity back to its own alerts. No new data sources needed — just smarter observation of existing ones.

---

## Loop 3: Proactive Pattern Discovery

This is the highest-value loop and the hardest to build. The agent doesn't just respond to anomalies — it actively looks for **slow-moving trends and hidden correlations** that humans miss.

### What "proactive" looks like:

#### 3a. Trend Detection (the slow boil)

Things that aren't alarming right now but will be a problem if they continue.

```
EXAMPLE TRENDS TO WATCH:

Capacity:
  "Disk usage on payments-db has grown 2.1% per week for the last
   6 weeks. At this rate, you'll hit 90% in 5 weeks."

  "Peak memory usage on the API servers has crept up from 72% to 84%
   over the past month. No single event caused it — it's gradual."

Performance degradation:
  "P95 latency on /api/search has increased from 180ms to 340ms over
   the past 3 weeks. No alerts have fired because the threshold is
   500ms, but the trend is clear."

  "The nightly ETL job used to complete in 22 minutes. Last week
   it took 38 minutes. It's getting slower every run."

Reliability:
  "The auth service has had 4 brief (<1 min) outages in the past
   2 weeks. Each one self-recovered and didn't trigger an alert,
   but the frequency is increasing."

  "SSL certificate for api.example.com expires in 23 days."

Cost:
  "Outbound data transfer from the CDN has doubled since the last
   frontend deploy. Might be an asset caching issue."

Dependency health:
  "The third-party payment API has been responding 40% slower on
   Tuesdays between 2-4pm for the last month. Might be their
   batch processing window."
```

#### 3b. Correlation Discovery (connecting the dots)

Finding relationships between signals that aren't in any runbook.

```
EXAMPLE CORRELATIONS:

  "Every time the marketing team sends a newsletter (detected via
   spike in /api/unsubscribe traffic), the session service CPU
   spikes to 90%. This has happened 3 out of 3 Tuesdays."

  "Deployments to the inventory service between 2-4pm correlate
   with a 15-minute latency spike in the order service. Deployments
   outside that window don't cause this. Might be related to the
   daily inventory sync job."

  "When node-pool-3 auto-scales past 8 nodes, DNS resolution
   latency increases by 40ms for ~5 minutes. Possible CoreDNS
   cache invalidation issue."
```

#### 3c. Anomaly Profiling (what's "normal" for THIS system)

Building a deep model of what normal looks like, so it can spot deviations that static thresholds miss.

```
LEARNED NORMAL BEHAVIOR:

  "Traffic pattern: 60% of daily requests hit between 9am-5pm EST.
   Weekend traffic is 30% of weekday. Any deviation from this
   pattern is worth investigating."

  "Deployment cadence: this team deploys 2-3 times per day on
   weekdays. A day with zero deploys might mean the CI pipeline
   is broken."

  "Error budget: the payments service normally throws ~12 errors/hour
   (mostly timeouts to the bank API). Above 25/hour is abnormal."

  "Seasonal: Black Friday traffic is 4x normal. Start pre-scaling
   2 days before." (learned from last year's data)
```

### How to implement proactive discovery:

```
┌──────────────────────────────────────────────────────────┐
│                 WEEKLY ANALYSIS JOB                        │
│          (runs alongside the heartbeat, slower cadence)    │
│                                                            │
│  1. TREND SCAN                                             │
│     Pull 30-day metrics for all monitored services         │
│     Feed to LLM: "identify any concerning trends"          │
│     Compare this week's summary to last week's             │
│                                                            │
│  2. CORRELATION SCAN                                       │
│     Look at all incidents from the past 2 weeks            │
│     Feed to LLM: "are any of these related?"               │
│     Check for recurring time-of-day or day-of-week         │
│     patterns                                               │
│                                                            │
│  3. BASELINE UPDATE                                        │
│     Recalculate rolling baselines for all key metrics      │
│     Flag any baselines that shifted significantly          │
│     Update "normal" profiles                               │
│                                                            │
│  4. PROACTIVE REPORT                                       │
│     Generate a "State of Infrastructure" summary           │
│     Highlight: trends, predictions, risks, improvements    │
│     Deliver to team lead via Slack/email                   │
│                                                            │
│  DAILY: trend detection + baseline updates                 │
│  WEEKLY: full correlation scan + proactive report          │
│  MONTHLY: model accuracy review + learning summary         │
│                                                            │
└──────────────────────────────────────────────────────────┘
```

---

## The Feedback-Driven Improvement Cycle

All three loops feed into a continuous improvement cycle:

```
                 ┌──────────────┐
                 │  RAW SIGNALS │
                 │  (metrics,   │
                 │   logs, etc) │
                 └──────┬───────┘
                        │
                        v
              ┌───────────────────┐
              │ DETECTION ENGINE  │◄──── Learning updates:
              │                   │      - adjusted thresholds
              │ - Thresholds      │      - new patterns
              │ - Baselines       │      - suppression rules
              │ - LLM patterns    │      - correlation rules
              └────────┬──────────┘
                       │
                       v
              ┌───────────────────┐
              │ INVESTIGATION     │◄──── Learning updates:
              │                   │      - better playbooks
              │ - Root cause      │      - new investigation steps
              │ - Evidence chain  │        (learned from human behavior)
              │ - Remediation     │      - improved root cause patterns
              └────────┬──────────┘
                       │
                       v
              ┌───────────────────┐
              │ ESCALATION        │◄──── Learning updates:
              │                   │      - severity calibration
              │ - Route to team   │      - team routing corrections
              │ - Severity level  │      - escalation timing
              │ - Alert format    │
              └────────┬──────────┘
                       │
                       v
              ┌───────────────────┐
              │ FEEDBACK CAPTURE  │
              │                   │
              │ Loop 1: explicit  │
              │ Loop 2: observed  │
              │ Loop 3: proactive │
              └────────┬──────────┘
                       │
                       v
              ┌───────────────────┐
              │ LEARNING ENGINE   │
              │                   │
              │ - Update weights  │
              │ - Refine rules    │
              │ - Retrain/fine-   │
              │   tune models     │
              │ - Update RAG      │
              │   knowledge base  │
              └───────────────────┘
                       │
                       └──────── feeds back to Detection ──►
```

---

## Knowledge Base: The Agent's Growing Brain

Over time, the agent accumulates institutional knowledge that lives in the RAG store:

### What goes into the knowledge base:

```
INCIDENT MEMORY
  Every investigation the agent runs gets stored:
  - What was detected
  - What the root cause was (confirmed by human feedback)
  - What fixed it
  - How long it took to resolve
  - What the agent got right and wrong

  → Enables: "This looks like INC-2026-03-15 — last time,
     the fix was to restart the cache cluster"

SERVICE PROFILES  
  Learned behavior models per service:
  - Normal error rate range
  - Normal latency range
  - Traffic patterns (daily, weekly, seasonal)
  - Common failure modes
  - Dependencies (what breaks when this service breaks)
  - Owner team + escalation preferences

  → Enables: "This error rate would be alarming for most services,
     but payments-service normally runs at 15 errors/hour due to
     bank API timeouts. This is within normal range."

TEAM PREFERENCES
  Per-team learned preferences:
  - Alert sensitivity (some teams want everything, some want critical only)
  - Preferred escalation channel
  - Working hours / on-call schedule awareness
  - Which investigation details they find most useful
  - Response time patterns (fast responders vs. "check it tomorrow" teams)

  → Enables: "The data team doesn't respond to Slack alerts before 10am.
     For overnight issues, create a Jira ticket instead."

INFRASTRUCTURE PATTERNS
  Cross-service learned correlations:
  - Deployment → incident correlation map
  - Time-of-day patterns
  - Dependency cascade paths
  - Seasonal patterns
  - Known maintenance windows

  → Enables: "Newsletter send detected. Pre-emptively scaling the
     session service based on the pattern observed over the past month."

RUNBOOK EVOLUTION
  Static runbooks imported at setup, but enriched over time:
  - "The runbook says to restart the cache, but the last 3 times
     that didn't work — the real fix was clearing the session store"
  - Agent annotates runbooks with actual resolution data

  → Enables: runbooks that stay current without manual updating
```

---

## Proactive Reports: Showing Value Beyond Firefighting

A weekly "State of Infrastructure" digest that demonstrates the agent is watching things humans aren't.

### Example weekly report:

```markdown
# Weekly Infrastructure Intelligence — Apr 14-20, 2026

## 🔴 Action Required (2 items)

1. **payments-db disk usage will hit 90% in ~4 weeks**
   Growth rate: 2.3%/week (consistent for 8 weeks)
   Recommendation: Archive transactions older than 90 days
   or provision additional storage

2. **Auth service intermittent failures increasing**
   4 self-healing outages this week (up from 2 last week)
   Each lasted <60 seconds, no customer impact yet
   Pattern: all occurred during GC pauses — likely needs
   heap tuning

## 🟡 Worth Watching (3 items)

3. **API search latency trending up**
   P95: 180ms → 260ms over 3 weeks (threshold: 500ms)
   Correlates with growing product catalog — may need
   index optimization

4. **Third-party SMS API degraded on Thursdays 3-5pm**
   Observed 3 consecutive weeks
   No impact yet but notification delivery slows by ~40%

5. **Deploy-to-incident correlation for order-service**
   3 of last 5 deploys to order-service caused brief
   latency spikes. Suggest adding canary deployment.

## 🟢 Improvements Noticed

6. **Cart service stability improved**
   Zero restart events this week (was 6/week last month)
   Likely related to memory limit increase on Apr 10

7. **Alert noise reduced 34% vs. last week**
   Learning adjustments suppressed 12 false positives
   (all confirmed as noise via feedback)

## 📊 Agent Accuracy This Week
- Alerts sent: 8
- Confirmed useful: 6 (75%)
- Confirmed noise: 1 (12.5%)
- No feedback yet: 1
- Root cause accuracy: 5/6 correct (83%)
- Avg time to detection: 2.4 minutes
- Avg investigation time: 45 seconds
```

---

## How Learning is Implemented Technically

### Short-term learning (immediate, no retraining):

- **Prompt context injection**: Before every LLM call, inject relevant history
  ```
  "Previous incidents for this service: [list]"
  "This team prefers severity calibration: [adjustments]"
  "Known false positive patterns: [list]"
  ```
- **RAG retrieval**: Pull similar past incidents and their resolutions
- **Rule updates**: Adjust thresholds and suppression rules in config
- **Weighting**: Confidence scores adjusted based on feedback history

### Medium-term learning (weekly, lightweight):

- **Baseline recalculation**: Update rolling averages and "normal" profiles
- **Pattern extraction**: LLM reviews the week's incidents and extracts reusable patterns → stored in RAG
- **Correlation updates**: Refresh the dependency/correlation map

### Long-term learning (monthly, heavier):

- **Model fine-tuning** (optional): Take the accumulated feedback data and fine-tune the Tier 1 triage model on THIS client's data
  - Input: raw signals that the agent saw
  - Output: the human-confirmed correct severity + root cause
  - This makes the small/fast model increasingly accurate for this specific environment
- **Playbook refinement**: Merge successful investigation patterns into the standard playbook
- **Accuracy trend reporting**: Is the agent getting better over time? Track precision, recall, root cause accuracy month-over-month

---

## The Value Proposition This Creates

```
Month 1:    "It's like a junior SRE that catches the obvious stuff"
            Alert accuracy ~60%, needs a lot of feedback

Month 3:    "It knows our system pretty well"
            Alert accuracy ~80%, catching things we missed,
            weekly reports are actually useful

Month 6:    "It's like having a senior SRE who never sleeps"
            Alert accuracy ~90%, proactive trend detection
            is preventing incidents before they happen,
            institutional knowledge survives team turnover

Month 12:   "We can't imagine operating without it"
            Deep correlation knowledge, seasonal awareness,
            the agent catches issues 30 minutes before
            customers notice, new engineers use it to
            learn the system
```

This trajectory is the real moat — the longer a client uses it, the harder it is to rip out, because all that learned context is unique to their environment.
