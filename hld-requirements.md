# HLD — AI-powered observability platform
### Requirements & scale estimates

**Document status:** Draft v0.1  
**Author:** Rohan Deoli  
**Last updated:** April 2026  
**Reviewers:** TBD

---

## Table of contents

1. [Problem statement](#1-problem-statement)
2. [Goals and non-goals](#2-goals-and-non-goals)
3. [User personas](#3-user-personas)
4. [Functional requirements](#4-functional-requirements)
5. [Scale assumptions](#5-scale-assumptions)
6. [Non-functional requirements](#6-non-functional-requirements)
7. [Open questions](#7-open-questions)

---

## 1. Problem statement

Modern engineering teams operate distributed systems composed of many services, each emitting logs, metrics, and traces continuously. Debugging a production incident requires correlating all three signal types simultaneously — a task that today requires hopping between multiple tools (log search, metric dashboards, trace viewers) with no unified view and no intelligent assistance.

This platform aims to solve that by providing a unified observability layer — ingesting logs, metrics, and traces from instrumented services, storing and querying them efficiently, and layering an AI copilot on top that can answer natural language questions, detect anomalies, correlate signals, and assist with root cause analysis.

Think: a self-hosted, extensible Datadog with an AI-native interface.

---

## 2. Goals and non-goals

### Goals

- Ingest logs, metrics, and traces from any instrumented service via standard protocols (OpenTelemetry, Prometheus, structured JSON logs)
- Store all three signal types durably with configurable per-tenant retention
- Provide a real-time log explorer, metric dashboards, and distributed trace viewer
- Support multi-tenancy with per-tenant data isolation and storage quotas
- Alert on metric thresholds and anomalies with sub-60-second detection latency
- Expose an AI copilot capable of natural language log search, anomaly explanation, and root cause analysis
- Design for autonomous incident response (AI-triggered paging, Slack summaries) as a first-class v2 capability

### Non-goals (v1)

- Mobile application
- Compliance or audit logging (SOC2, HIPAA)
- Synthetic monitoring or uptime checks
- Log-based billing or cost attribution
- Self-serve tenant onboarding UI (manual onboarding for v1)
- Autonomous AI paging and remediation (v2)
- SLA reporting and error budget dashboards (v2)

---

## 3. User personas

### Persona 1 — Engineering manager (v1, dashboard consumer)
**Context:** Wants a weekly reliability view across services — error rates, p99 latency trends, deployment frequency. Does not query logs directly. Needs a configurable dashboard they can share.  
**Primary need:** Pre-built metric dashboards with weekly/monthly time ranges, clear health indicators per service.

### Persona 2 — Backend engineer (v1, primary user)
**Context:** On-call engineer responding to a production alert. Needs to search logs by service and time window, correlate error spikes with a specific deployment or upstream service, and find the root trace for a failing request.  
**Primary need:** Fast log search, trace lookup by request ID, metric charts scoped to a specific service and time window, and AI-assisted RCA.

### Persona 3 — SRE (v2, alert owner)
**Context:** Sets up and tunes alert rules, manages on-call rotation, reviews weekly reliability trends. Needs the alert engine to be expressive enough for composite conditions (e.g. error rate > 5% AND p99 > 2s for 5 minutes).  
**Primary need:** Alert rule editor, alert history, suppression rules, Slack/PagerDuty integration.

---

## 4. Functional requirements

Requirements are tagged: **[v1]** ships at launch, **[v2]** planned follow-on.

### 4.1 Data ingestion

| ID | Requirement | Priority |
|----|-------------|----------|
| ING-01 | Ingest structured log events from services via a log agent (DaemonSet on K8s) | v1 |
| ING-02 | Ingest Prometheus-format metrics via scraping and push gateway | v1 |
| ING-03 | Ingest distributed traces in OpenTelemetry format via OTLP endpoint | v1 |
| ING-04 | All ingested events must carry a `tenant_id` extracted from the agent auth token | v1 |
| ING-05 | Ingestion must continue accepting data during downstream storage degradation, buffering in Kafka | v1 |
| ING-06 | Reject ingestion requests when tenant storage quota is exceeded, returning HTTP 429 | v1 |
| ING-07 | Support log ingestion via Fluent Bit as an alternative to the native agent | v2 |

### 4.2 Log explorer

| ID | Requirement | Priority |
|----|-------------|----------|
| LOG-01 | Full-text search across log messages with field-level filters (service, level, host, trace_id) | v1 |
| LOG-02 | Live tail — stream log lines in real time with sub-3s freshness | v1 |
| LOG-03 | Time range selector with presets (last 15m, 1h, 6h, 24h, 7d) and custom range | v1 |
| LOG-04 | Save and share named queries | v1 |
| LOG-05 | Log line context expansion — show N lines before/after a selected line | v1 |
| LOG-06 | Click a `trace_id` in a log line to jump directly to the corresponding trace | v1 |
| LOG-07 | Natural language query — translate English to ClickHouse SQL via AI copilot | v1 |

### 4.3 Metric dashboards

| ID | Requirement | Priority |
|----|-------------|----------|
| MET-01 | Pre-built service dashboard: request rate, error rate, p50/p95/p99 latency, JVM metrics | v1 |
| MET-02 | Custom dashboard builder with drag-and-drop panels | v1 |
| MET-03 | Time-series panels with zoom, pan, and overlay (compare two time windows) | v1 |
| MET-04 | Dashboard-level time range selector applied to all panels | v1 |
| MET-05 | Variable-driven dashboards (select service, environment from dropdown) | v2 |

### 4.4 Distributed trace viewer

| ID | Requirement | Priority |
|----|-------------|----------|
| TRA-01 | Look up a trace by `trace_id` | v1 |
| TRA-02 | Render trace as a Gantt/waterfall chart with spans, services, and durations | v1 |
| TRA-03 | Highlight error spans and show span attributes on click | v1 |
| TRA-04 | Show logs correlated to a trace (same `trace_id`) in-line below the trace | v1 |
| TRA-05 | Service dependency graph derived from trace data | v2 |

### 4.5 Alerting

| ID | Requirement | Priority |
|----|-------------|----------|
| ALT-01 | Define threshold alerts on any PromQL-compatible metric expression | v1 |
| ALT-02 | Alert evaluation runs on a 30-second schedule | v1 |
| ALT-03 | Deliver alert notifications to Slack and email | v1 |
| ALT-04 | Alert deduplication — do not re-fire while alert is already active | v1 |
| ALT-05 | Alert history and status page per tenant | v1 |
| ALT-06 | Anomaly-based alerts triggered by ML detector, not just static thresholds | v2 |
| ALT-07 | PagerDuty integration | v2 |
| ALT-08 | Alert suppression rules and maintenance windows | v2 |

### 4.6 AI copilot

| ID | Requirement | Priority |
|----|-------------|----------|
| AI-01 | Natural language to ClickHouse SQL for log search | v1 |
| AI-02 | Anomaly detection on time-series metrics using statistical forecasting | v1 |
| AI-03 | Human-triggered RCA report: correlate logs, metrics, traces around an alert time window and produce a written summary | v1 |
| AI-04 | Log clustering — surface novel error patterns not covered by alert rules | v1 |
| AI-05 | Autonomous incident responder: AI-triggered alert → page on-call → post Slack summary | v2 |
| AI-06 | Auto-remediation suggestions (restart pod, scale deployment) with human approval gate | v2 |

### 4.7 Multi-tenancy

| ID | Requirement | Priority |
|----|-------------|----------|
| TNT-01 | Each tenant's data is logically isolated — no query can cross tenant boundaries | v1 |
| TNT-02 | Per-tenant storage quotas enforced at ingestion and queried via an admin API | v1 |
| TNT-03 | JWT-based auth with `tenant_id` and `role` claims; all API calls scoped to tenant | v1 |
| TNT-04 | Roles: `admin` (full access), `editor` (create dashboards/alerts), `viewer` (read-only) | v1 |

---

## 5. Scale assumptions

These numbers reflect a mid-sized engineering organisation with 10 active tenants, designed to scale to 50 tenants without architectural changes.

| Dimension | Assumed value | Reasoning |
|-----------|--------------|-----------|
| Tenants | 10 active, design for 50 | Real multi-tenant isolation problems without over-engineering |
| Instrumented services | 20 per tenant avg, 200 total | Realistic for a mid-sized eng org |
| Log ingestion rate | 5,000 lines/sec sustained peak | ~25 services/tenant × ~10 lines/sec avg across 10 tenants |
| Log ingestion spike | 50,000 lines/sec (10× spike) | Absorbed by Kafka; does not reach storage directly |
| Avg log line size | 200 bytes (after parsing) | Structured JSON with standard fields |
| Metrics time-series | 100,000 active series total | 200 services × ~500 series each |
| Trace volume | 500 traces/sec after sampling | ~5,000 req/sec total, 10% head-based sampling |
| Log retention | 30 days | Beyond 30 days: marginal debugging value, high storage cost |
| Metric retention | 90 days | Required for weekly/monthly trend dashboards |
| Trace retention | 7 days | Large payloads; 7 days covers almost all incident lookback |
| Raw log storage (30d) | ~2 TB compressed | 5k lines/sec × 200 bytes × 86,400s × 30d ÷ 10× compression |
| Concurrent platform users | 100 | Dashboards + log searches across all tenants simultaneously |

### Back-of-envelope: why Kafka is required

At 5,000 lines/sec sustained with 10× spike capacity (50,000 lines/sec), direct writes to ClickHouse would be overwhelmed during traffic spikes. Kafka acts as a durable buffer and decouples ingestion throughput from storage write throughput. At 50,000 lines/sec × 200 bytes = 10 MB/sec — well within a 3-broker Kafka cluster's capacity.

---

## 6. Non-functional requirements

| Requirement | Target | Notes |
|-------------|--------|-------|
| Availability | 99.99% design target | Multi-AZ, no single points of failure, automated failover. v1 operational target is 99.9% as the system matures. |
| Log search latency | p95 < 1s (24h window), p95 < 3s (7d window) | Requires ClickHouse partition pruning on `(tenant_id, timestamp)` |
| Metric query latency | p95 < 500ms for dashboard panels | VictoriaMetrics pre-aggregation for long time ranges |
| Live tail freshness | < 3s end-to-end | Agent → Kafka → stream processor → Redis → WebSocket → browser |
| Alert latency | < 30s from threshold breach to notification delivery | Scheduled evaluation engine in v1; streaming evaluation in v2 |
| Log durability | Zero loss under normal operating conditions | Agent buffers to local disk; Kafka replication factor = 3 |
| Ingestion durability | At-most 30s data loss on catastrophic ingestion failure | Kafka consumer offset commit after successful write to ClickHouse |
| Tenant data isolation | No query can return data across tenant boundaries | `tenant_id` enforced at query layer, not just application layer |
| Quota enforcement latency | Quota check adds < 10ms to ingestion path | Redis incr/decr with TTL |
| Deployment | Zero-downtime deploys | K8s rolling updates; stateless services only |

---

## 7. Open questions

These are unresolved decisions that must be answered before detailed component design begins.

| ID | Question | Impact | Owner |
|----|----------|--------|-------|
| OQ-01 | ClickHouse isolation model: shared tables with `tenant_id` column vs per-tenant databases? Shared tables are operationally simpler; per-tenant DBs give stronger isolation and easier quota enforcement. | Storage schema, query layer design | — |
| OQ-02 | Kafka topic strategy: one shared topic per signal type (`logs-raw`, `metrics-raw`, `traces-raw`) vs per-tenant topics? Shared is simpler; per-tenant gives backpressure isolation for noisy tenants. | Kafka broker sizing, consumer group design | — |
| OQ-03 | Trace storage backend: Jaeger (battle-tested, complex ops) vs Grafana Tempo (simpler, uses S3 as backend)? Tempo reduces infra footprint significantly. | Infra complexity, query latency | — |
| OQ-04 | AI copilot LLM: self-hosted (Ollama + Llama 3) vs API-based (Claude / GPT-4o)? Self-hosted avoids sending log data to third parties; API-based is faster to build. | Data privacy, AI service cost, latency | — |
| OQ-05 | Stream processor: Python-based (Faust/Bytewax, easier to integrate with AI service) vs JVM-based (Apache Flink, more operationally mature)? | Ingestion pipeline complexity, language consistency | — |
| OQ-06 | What is the admin model for tenant provisioning? Manual via API for v1, or a self-serve onboarding flow? | Auth service design, onboarding UX | — |

---

*Next section: [Component overview and architecture diagram →](./hld-architecture.md)*
