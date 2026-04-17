# RJ Project - Heartbeat Architecture
## The Core Loop: Poll → Detect → Investigate → Escalate

**Date:** 2026-04-16

---

## Operating Model

This is **not** a passive alert receiver that waits for PagerDuty to push notifications. This is an **active agent** that runs on a heartbeat — it reaches out to the monitoring tools on a regular cadence, pulls their current state, and decides for itself whether something needs attention.

Think of it like a senior SRE who checks the dashboards every few minutes, even when nothing has paged yet.

---

## The Heartbeat Loop

```
    ┌─────────────────────────────────────────────────┐
    │                                                   │
    │   ┌──────────┐                                    │
    │   │ HEARTBEAT│  (every N minutes, configurable)   │
    │   │  TIMER   │                                    │
    │   └────┬─────┘                                    │
    │        │                                          │
    │        v                                          │
    │   ┌──────────┐                                    │
    │   │  POLL    │  Query each connected tool:        │
    │   │          │  - Prometheus: current alerts?      │
    │   │          │  - Grafana: any panels in alarm?    │
    │   │          │  - K8s: pod restarts? OOMKills?     │
    │   │          │  - Logs: error rate spike?          │
    │   │          │  - APM: latency anomalies?          │
    │   └────┬─────┘                                    │
    │        │                                          │
    │        v                                          │
    │   ┌──────────┐                                    │
    │   │ DETECT   │  Compare against:                  │
    │   │          │  - Known thresholds                 │
    │   │          │  - Baseline patterns (rolling avg)  │
    │   │          │  - Previous heartbeat state (delta) │
    │   │          │  - LLM pattern recognition          │
    │   └────┬─────┘                                    │
    │        │                                          │
    │        ├── Nothing abnormal ──> Log & loop back ──┘
    │        │
    │        v  (anomaly detected)
    │   ┌──────────┐
    │   │ TRIAGE   │  Tier 1 LLM (fast, small model):
    │   │          │  - Is this real or noise?
    │   │          │  - Severity: low / medium / high / critical
    │   │          │  - Blast radius: isolated vs. spreading
    │   │          │  - Has this happened before? (check history)
    │   └────┬─────┘
    │        │
    │        ├── Noise / known-benign ──> Log & loop back ──┘
    │        │
    │        v  (genuine issue)
    │   ┌──────────────┐
    │   │ INVESTIGATE   │  Tier 2 LLM (deeper, larger model):
    │   │               │  - Pull detailed logs around the event
    │   │               │  - Check recent deployments/changes
    │   │               │  - Walk the dependency graph
    │   │               │  - Correlate with other signals
    │   │               │  - Query RAG for similar past incidents
    │   │               │  - Build root cause hypothesis
    │   │               │  - Generate evidence chain
    │   └──────┬───────┘
    │          │
    │          v
    │   ┌──────────┐
    │   │ ESCALATE │  Based on severity + confidence:
    │   │          │  - Create incident ticket (Jira/ServiceNow)
    │   │          │  - Notify on-call (Slack/Teams/PagerDuty)
    │   │          │  - Include: summary, evidence, suggested fix
    │   │          │  - If high confidence + policy allows:
    │   │          │    trigger auto-remediation
    │   └────┬─────┘
    │        │
    │        v
    │   Continue heartbeat loop ──────────────────────┘
    │
    └─────────────────────────────────────────────────┘
```

---

## Heartbeat Configuration

The polling interval isn't one-size-fits-all. Different data sources have different costs and latencies.

```yaml
# Example heartbeat config
heartbeat:
  default_interval: 60s    # Check most things every minute

  sources:
    prometheus:
      interval: 30s        # Metrics are cheap to query
      endpoint: http://prometheus:9090
      checks:
        - active_alerts     # What's currently firing?
        - targets_down      # Any scrape targets unreachable?

    kubernetes:
      interval: 30s
      checks:
        - pod_restarts      # OOMKills, CrashLoopBackoffs
        - node_conditions   # NotReady, DiskPressure, MemoryPressure
        - failed_jobs       # CronJobs that didn't complete
        - pending_pods      # Pods stuck in Pending (scheduling issues)

    elasticsearch:
      interval: 120s       # Log queries are heavier
      endpoint: http://elasticsearch:9200
      checks:
        - error_rate        # Errors/min vs. rolling baseline
        - new_error_types   # Error messages never seen before
        - log_volume_anomaly # Sudden spike or drop in log volume

    grafana:
      interval: 60s
      endpoint: http://grafana:3000
      checks:
        - alerting_panels   # Any dashboard panels in alert state

    custom_endpoints:       # Health check URLs
      interval: 30s
      targets:
        - name: "API Gateway"
          url: http://api-gateway/health
        - name: "Auth Service"
          url: http://auth-service/health
```

---

## State Management: The Key to Smart Detection

The heartbeat isn't just "check and forget." It maintains state between polls so it can detect **trends**, not just point-in-time snapshots.

### What the agent tracks between heartbeats:

```
┌─────────────────────────────────────────────────────┐
│                   STATE STORE                        │
│                                                      │
│  Current Snapshot          Previous Snapshots         │
│  ─────────────────         ──────────────────         │
│  - Active alerts           - Last N snapshots         │
│  - Error rates             - Rolling baselines        │
│  - Pod health              - Trend direction          │
│  - Service latencies       - Time since last change   │
│                                                      │
│  Active Investigations     Detection Rules            │
│  ─────────────────────     ────────────────           │
│  - Issue ID                - Static thresholds        │
│  - First detected at       - Baseline deviations      │
│  - Last checked at         - Pattern rules (LLM)      │
│  - Current status          - Correlation rules        │
│  - Evidence collected                                 │
│  - Escalation status                                 │
│                                                      │
│  Cooldowns                                           │
│  ─────────                                           │
│  - Don't re-alert on X                               │
│    for next 30 minutes                               │
│  - Issue Y already                                   │
│    escalated, track only                             │
└─────────────────────────────────────────────────────┘
```

### Detection modes:

1. **Threshold-based** (simple, fast, no LLM needed)
   - CPU > 90% for 5 minutes
   - Error rate > 50/min
   - Pod restart count > 3 in 10 minutes

2. **Baseline deviation** (statistical, no LLM needed)
   - Error rate is 3x the rolling 1-hour average
   - Latency p99 jumped 200% since last heartbeat
   - Log volume dropped to zero (service might be dead)

3. **Pattern-based** (this is where the LLM adds value)
   - "Error rate is normal but the error *messages* changed — new exception type"
   - "Three unrelated services all started throwing timeout errors at the same time — upstream dependency?"
   - "This looks like the same pattern as the outage on March 3rd"
   - "Disk usage has been growing linearly — at this rate, it fills up in 6 hours"

---

## The Investigation Phase (Deep Dive)

When the heartbeat detects something real, the investigation phase kicks in. This is where the bigger LLM earns its keep.

### Investigation playbook (what the LLM does):

```
INVESTIGATION STEPS FOR: "API latency spike detected"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Step 1: SCOPE THE BLAST RADIUS
  → Query: Which endpoints are affected?
  → Query: Is it all traffic or specific paths?
  → Query: When exactly did it start? (narrow the time window)

Step 2: CHECK RECENT CHANGES
  → Query: Any deployments in the last 2 hours?
  → Query: Any config changes? (K8s configmaps, feature flags)
  → Query: Any infrastructure changes? (scaling events, node additions)

Step 3: CHECK DEPENDENCIES
  → Query: Are downstream services healthy?
  → Query: Database latency / connection pool usage?
  → Query: External API response times?
  → Query: Message queue depth / consumer lag?

Step 4: CHECK RESOURCES
  → Query: CPU / memory / disk on affected pods/hosts?
  → Query: Network errors or packet loss?
  → Query: Connection limits / file descriptor usage?

Step 5: EXAMINE LOGS
  → Query: Error logs from affected service in the time window
  → Query: Any new error patterns?
  → Query: Any stack traces?

Step 6: CORRELATE
  → Are any other services showing symptoms?
  → Does the timeline match a known event?
  → Does this match any historical incident pattern? (RAG lookup)

Step 7: SYNTHESIZE
  → Build root cause hypothesis
  → Assign confidence level
  → Generate evidence chain (every claim linked to a specific data point)
  → Suggest remediation steps
```

### Investigation output (what gets escalated):

```markdown
## Incident Summary
**Severity:** HIGH
**Detected:** 2026-04-16 14:32 UTC (heartbeat #4,721)
**Affected:** api-gateway (all endpoints, 3 pods)

## What's Happening
API gateway p99 latency spiked from 120ms to 2.4s starting at 14:28 UTC.
All three pods are affected equally.

## Root Cause (confidence: 85%)
The `payments-db` PostgreSQL instance hit its connection pool limit (max: 100)
at 14:27 UTC. This is causing connection queue buildup in the payments service,
which is causing timeout cascading to the API gateway.

## Evidence
1. [Prometheus] api-gateway p99 latency: 120ms → 2,400ms at 14:28 UTC
2. [Prometheus] payments-db active connections: 100/100 since 14:27 UTC
3. [Logs] payments-service: "could not obtain connection within 5000ms" (247 occurrences since 14:27)
4. [K8s] No recent deployments or config changes
5. [Prometheus] payments-db query duration p99 jumped from 8ms to 340ms at 14:25 (likely the trigger)

## Suggested Actions
1. **Immediate:** Increase payments-db connection pool to 200 (config change)
2. **Investigate:** Why did query duration spike? Check for slow queries / missing index
3. **Longer-term:** Add connection pool monitoring alert at 80% utilization

## Similar Past Incidents
- 2026-03-03: Same pattern — DB connection exhaustion caused API cascade (resolved by adding index on `transactions.created_at`)
```

---

## Escalation Decision Matrix

Not everything needs to wake someone up at 3am.

```
                        ┌─────────────┬──────────────┐
                        │ Confidence  │ Confidence   │
                        │ HIGH (>80%) │ LOW (<80%)   │
┌───────────────────────┼─────────────┼──────────────┤
│ Severity: CRITICAL    │ Page on-call│ Page on-call │
│ (service down,        │ + auto-     │ + full       │
│  data loss risk)      │ remediate   │ investigation│
│                       │ if allowed  │ attached     │
├───────────────────────┼─────────────┼──────────────┤
│ Severity: HIGH        │ Slack alert │ Slack alert  │
│ (degraded perf,       │ + suggested │ + request    │
│  partial outage)      │ fix         │ human review │
├───────────────────────┼─────────────┼──────────────┤
│ Severity: MEDIUM      │ Ticket      │ Ticket       │
│ (slow degradation,    │ created,    │ created,     │
│  non-urgent)          │ next-day    │ flagged for  │
│                       │ review      │ monitoring   │
├───────────────────────┼─────────────┼──────────────┤
│ Severity: LOW         │ Log only    │ Log only     │
│ (cosmetic, noise)     │             │              │
└───────────────────────┴─────────────┴──────────────┘
```

---

## Anti-Patterns to Avoid

1. **Alert fatigue** — The whole point is to REDUCE noise, not add another noisy tool. Aggressive deduplication and cooldown timers are essential.

2. **Hallucinated root causes** — LLM says "the database is down" when it's actually a network blip. Every claim must be backed by a specific data point. If the LLM can't cite evidence, it should say "I don't know" rather than guess.

3. **Stale investigations** — If an issue self-resolves between heartbeats, close it cleanly. Don't leave phantom investigations open.

4. **Over-polling** — Hammering Elasticsearch every 10 seconds will create the outage you're trying to detect. Respect rate limits and query costs.

5. **Missing the forest for the trees** — 10 separate alerts that are all symptoms of one root cause should be ONE investigation, not 10. The correlation engine is critical.

---

## Technical Implementation Notes

### Heartbeat scheduler
- Use a simple **cron-style scheduler** (Go's `time.Ticker` or Python's `APScheduler`)
- Each source gets its own goroutine/async task with independent timing
- Results feed into a shared **event bus** (in-memory channel or lightweight queue like NATS)

### State store
- **Redis** for hot state (current snapshot, active investigations, cooldowns)
- **PostgreSQL/TimescaleDB** for historical state (trend data, past investigations)
- State must survive restarts — no in-memory-only state for anything important

### LLM interaction pattern
- Tier 1 calls should be **structured output** (JSON mode) for reliable parsing
- Tier 2 calls can be more free-form but should follow a **chain-of-thought template**
- All LLM calls get logged with input/output for debugging and improvement

### Graceful degradation
- If the LLM is down or overloaded, fall back to threshold-only detection
- If a data source is unreachable, that itself becomes an alert
- If the state store is down, continue polling but skip deduplication (better to double-alert than miss)
