# RJ Project - Lightweight AI SRE Agent
## Concept Brief & Architecture Ideas

**Date:** 2026-04-16
**Inspiration:** Datadog Bits AI SRE
**Core Thesis:** Build a lightweight, self-hosted AI SRE agent that monitors applications and infrastructure, consolidates signals, performs intelligent triage and first-step investigation, and escalates to humans — all without sending data to external/cloud LLMs.

---

## What Datadog Bits AI Does (for reference)

- Autonomously investigates every alert the moment it fires
- Correlates millions of signals across infrastructure in parallel
- Identifies root cause in minutes (claims 70% MTTR reduction)
- Suggests code fixes based on findings
- Integrates with Slack, Jira, ServiceNow, GitHub
- Natural language chat interface
- Enterprise RBAC, HIPAA compliance, zero third-party data retention

---

## Our Differentiator: Fully Local / Air-Gapped Intelligence

Datadog requires sending your observability data through their platform. Many orgs (finance, healthcare, gov, defense) can't or won't do that. Our product keeps **all data and all AI inference on-premises or in the customer's private cloud**.

---

## Proposed Architecture

### Layer 1: Data Ingestion & Normalization

The foundation — collect signals from everywhere and normalize them into a common schema.

**Sources to support:**
- **Metrics:** Prometheus, Grafana/Mimir, InfluxDB, CloudWatch, Azure Monitor
- **Logs:** Elasticsearch/OpenSearch, Loki, Splunk (via API), syslog
- **Traces:** OpenTelemetry (OTLP), Jaeger, Zipkin
- **Alerts:** PagerDuty, OpsGenie, Alertmanager, custom webhooks
- **Infra state:** Kubernetes API, cloud provider APIs, Terraform state
- **Incident history:** ServiceNow, Jira, internal runbooks (as RAG corpus)

**How to build it:**
- Lightweight **collector agents** (think: Telegraf/Vector-style) that pull from each source via plugin adapters
- Normalize everything into a **unified event schema** (timestamp, source, severity, entity, payload)
- Store in a local **time-series + event store** (QuestDB, TimescaleDB, or even SQLite for small deployments)
- A **correlation engine** that groups events by time window + affected entity (service, host, pod)

**Tech choices:**
- Rust or Go for the collector (performance, low footprint)
- OpenTelemetry Collector as a starting point — it already has 100+ receivers
- PostgreSQL + TimescaleDB for the event store (battle-tested, self-hosted)

---

### Layer 2: Local LLM Intelligence Engine

This is the heart of the system and the hardest part. The key constraint: **no data leaves the network**.

**Local LLM Options (ranked by practicality):**

| Model | Size | Hardware Needed | Strengths |
|-------|------|-----------------|-----------|
| **Llama 3.1 8B / 70B** | 8-70B | 1 GPU (8B) to 2-4x A100 (70B) | Great general reasoning, Apache 2.0 |
| **Mistral 7B / Mixtral 8x7B** | 7-47B | 1-2 GPUs | Fast, good at structured output |
| **Phi-3 / Phi-4** | 3.8-14B | CPU-capable at small sizes | Surprisingly capable for size |
| **Qwen 2.5 7B-72B** | 7-72B | Similar to Llama | Strong code + reasoning |
| **DeepSeek-R1 distills** | 7-70B | 1-4 GPUs | Excellent chain-of-thought reasoning |

**Recommended approach: Tiered model architecture**

```
Tier 1 (Fast Filter):    Small model (Phi-4 / Llama 8B, quantized)
                          Runs on CPU or single GPU
                          Job: Classify, triage, correlate, discard noise
                          Latency target: <2 seconds

Tier 2 (Deep Analysis):  Larger model (Llama 70B / Mixtral 8x7B)
                          Requires GPU(s)
                          Job: Root cause analysis, remediation suggestions
                          Latency target: <30 seconds
                          Only invoked when Tier 1 flags something interesting

Tier 3 (Optional):       Fine-tuned specialist models
                          Trained on YOUR incident history and runbooks
                          Job: Org-specific pattern matching
```

**Inference infrastructure:**
- **vLLM** or **llama.cpp** for model serving (both self-hosted, both excellent)
- **Ollama** for simpler deployments (wraps llama.cpp with nice API)
- For enterprise: **NVIDIA NIM** or **Intel OpenVINO** for optimized on-prem inference

**RAG (Retrieval-Augmented Generation) for context:**
- Index runbooks, post-mortems, architecture docs, past incident resolutions
- Use a local vector DB: **Chroma**, **Qdrant**, or **pgvector** (PostgreSQL extension)
- Embed with a local embedding model (e.g., `nomic-embed-text`, `bge-large`)
- When the LLM investigates, it pulls relevant historical context automatically

---

### Layer 3: Triage & Investigation Engine

This is the "brain" that orchestrates the workflow.

**Investigation pipeline:**

```
Alert Fires
    |
    v
[1] Signal Consolidation
    - Group related alerts (same service, same time window)
    - Pull recent metrics, logs, traces for affected entities
    - Check: is this a known/recurring pattern?
    |
    v
[2] Automated Triage (Tier 1 LLM)
    - Severity classification (noise / warning / critical)
    - Blast radius estimation (1 pod vs. entire service vs. cross-service)
    - Initial hypothesis generation
    |
    v
[3] Deep Investigation (Tier 2 LLM, if warranted)
    - Correlate across data sources
    - Query RAG for similar past incidents
    - Walk the dependency graph (what changed recently?)
    - Check recent deployments, config changes, scaling events
    - Generate root cause hypothesis with evidence chain
    |
    v
[4] Recommended Actions
    - Suggested remediation steps (from runbooks or LLM reasoning)
    - Auto-generated incident summary
    - Confidence score
    |
    v
[5] Escalation
    - Route to correct team/person based on service ownership
    - Push to Slack/Teams/PagerDuty with full context packet
    - If confidence is high enough + policy allows: auto-remediate
```

**Key design principle:** Every LLM conclusion must cite its evidence (specific log lines, metrics, traces). No black-box "trust me" answers. This is critical for SRE adoption.

---

### Layer 4: Interface & Escalation

**Chat interface:**
- Slack/Teams bot (most natural for SRE teams)
- Simple web UI for investigation timeline view
- Engineers can ask follow-up questions in natural language
- All queries processed locally

**Escalation integrations:**
- PagerDuty / OpsGenie (trigger or update incidents)
- Jira / ServiceNow (create tickets with investigation context)
- Slack / Teams (post investigation summaries)
- Webhooks (for anything custom)

---

## Security Architecture

Since security is the primary differentiator:

```
+--------------------------------------------------+
|              Customer Network / VPC               |
|                                                   |
|  +-----------+  +-----------+  +---------------+  |
|  | Collector |  | Collector |  | Collector     |  |
|  | (k8s)     |  | (cloud)   |  | (on-prem)     |  |
|  +-----+-----+  +-----+-----+  +-------+-------+  |
|        |              |                |            |
|        v              v                v            |
|  +-------------------------------------------+     |
|  |         Event Store (TimescaleDB)          |     |
|  +-------------------------------------------+     |
|        |                                            |
|        v                                            |
|  +-------------------------------------------+     |
|  |    Investigation Engine (Orchestrator)     |     |
|  +-------------------------------------------+     |
|        |              |                             |
|        v              v                             |
|  +-----------+  +-----------+  +---------------+   |
|  | Tier 1 LLM|  | Tier 2 LLM|  | Vector DB     |  |
|  | (vLLM)    |  | (vLLM)    |  | (RAG/Qdrant)  |  |
|  +-----------+  +-----------+  +---------------+   |
|        |                                            |
|        v                                            |
|  +-------------------------------------------+     |
|  |      Escalation / Notification Layer       |     |
|  |  (Slack, PagerDuty, Jira, ServiceNow)      |     |
|  +-------------------------------------------+     |
|                                                     |
|  NOTHING LEAVES THIS BOUNDARY (except escalation    |
|  notifications, which contain only summaries)        |
+-----------------------------------------------------+
```

**Security controls:**
- All LLM inference runs inside the customer's network
- No telemetry phone-home
- Escalation messages can be configured to include only summaries (no raw logs/metrics)
- Encrypt at rest (event store) and in transit (mTLS between components)
- RBAC on the investigation UI and chat interface
- Audit log of all LLM queries and actions

---

## MVP Scope (What to Build First)

**Phase 1 — Prove the concept (4-8 weeks):**
1. Single data source: Prometheus + Alertmanager
2. Single LLM: Llama 3.1 8B via Ollama
3. Simple alert → triage → Slack notification pipeline
4. No RAG yet, just direct prompt-based analysis
5. Docker Compose deployment

**Phase 2 — Make it useful (8-12 weeks):**
1. Add log ingestion (Elasticsearch or Loki)
2. Add RAG with runbook indexing
3. Tiered model architecture (fast triage + deep analysis)
4. Correlation engine (group related alerts)
5. Web UI for investigation timelines
6. Helm chart for Kubernetes deployment

**Phase 3 — Make it enterprise (12-20 weeks):**
1. Full OpenTelemetry support (metrics, logs, traces)
2. Multiple escalation integrations
3. Fine-tuning pipeline for org-specific models
4. RBAC, audit logging, SSO
5. Auto-remediation actions (with approval workflows)

---

## Competitive Positioning

| Feature | Datadog Bits AI | Our Product |
|---------|----------------|-------------|
| Data residency | Datadog cloud | Fully on-premises |
| LLM provider | Cloud LLM (opaque) | Local LLM (transparent) |
| Cost model | Per-host/per-GB pricing | One-time + infrastructure |
| Observability platform required | Datadog only | Works with existing stack |
| Setup complexity | Low (if already on DD) | Medium (self-hosted) |
| Air-gapped support | No | Yes |
| Customization | Limited | Full (fine-tune models, custom pipelines) |

**Target customers:**
- Orgs that can't use cloud AI (finance, healthcare, gov, defense)
- Orgs already invested in open-source observability (Prometheus, Grafana, ELK)
- Orgs frustrated with Datadog pricing who want a "good enough" AI layer
- MSPs/MSSPs who want to offer AI-powered monitoring to their clients

---

## Open Questions

1. **Naming** — What do we call this thing?
2. **Business model** — Open core? Fully commercial? SaaS for non-sensitive workloads?
3. **Hardware requirements** — What's the minimum viable GPU? Can Tier 1 run on CPU only?
4. **Fine-tuning strategy** — How much customer-specific training is needed vs. good prompting?
5. **Auto-remediation** — How aggressive do we want to be? (restart pods, rollback deploys, scale up)
6. **Multi-tenancy** — Does an MSP need to run separate instances per client?
