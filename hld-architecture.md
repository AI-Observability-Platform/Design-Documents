# HLD — AI-powered observability platform
### Component architecture & data flows

**Document status:** Draft v0.1  
**Author:** TBD  
**Last updated:** April 2026  
**Prerequisite reading:** [hld-requirements.md](./hld-requirements.md)  
**Architecture diagram:** [architecture.drawio](./architecture.drawio)

---

## Table of contents

1. [Architecture overview](#1-architecture-overview)
2. [Tier 1 — Sources](#2-tier-1--sources)
3. [Tier 2 — Collection agents](#3-tier-2--collection-agents)
4. [Tier 3 — Streaming pipeline](#4-tier-3--streaming-pipeline)
5. [Tier 4 — Storage backends](#5-tier-4--storage-backends)
6. [Tier 5 — API and backend services](#6-tier-5--api-and-backend-services)
7. [Tier 6 — Frontend](#7-tier-6--frontend)
8. [AI service](#8-ai-service)
9. [Multi-tenant control plane](#9-multi-tenant-control-plane)
10. [Critical data flows](#10-critical-data-flows)
11. [Cross-cutting concerns](#11-cross-cutting-concerns)
12. [Decisions resolved since requirements doc](#12-decisions-resolved-since-requirements-doc)
13. [Open questions](#13-open-questions)

---

## 1. Architecture overview

The platform is structured as six vertical tiers — sources, collection, streaming, storage, backend, and frontend — with two horizontal layers that cut across all tiers: the AI service and the multi-tenant control plane.

Data flows in one direction through the write path: sources emit signals, agents collect and forward them, Kafka buffers them durably, the stream processor parses and enriches them, and storage backends persist them. The read path is independent: the API gateway queries storage on demand and serves the frontend and AI service.

The architecture follows three consistent principles decided during design:

**Shared infrastructure, logical tenant isolation.** Every tier uses shared infrastructure (one Kafka cluster, one ClickHouse cluster, one VictoriaMetrics instance) with `tenant_id` enforced at the application layer. Physical isolation per tenant was evaluated and rejected — see ADR-001 and ADR-002 for reasoning.

**Operational simplicity over structural elegance.** Where a simpler component satisfies the requirements, it is preferred over a more powerful but operationally complex one. This is why Grafana Tempo (S3-backed) was chosen over Jaeger+Cassandra (ADR-003), and why the alert engine uses a Quartz scheduler rather than a streaming evaluation engine for v1.

**Write path and read path are decoupled.** The ingestion pipeline (agents → Kafka → stream processor → storage) has no dependency on the API layer. A full API outage does not affect data ingestion. This is the primary mechanism for meeting the durability NFR (zero log loss under normal operating conditions).

---

## 2. Tier 1 — Sources

Sources are the instrumented services and infrastructure that emit observability signals. The platform does not control sources — it defines the contracts that sources must satisfy.

### Spring Boot services

The primary signal source. Each service must satisfy three contracts:

- **Logs:** emit structured JSON to stdout, one object per line. Format defined in the log format specification (see `docs/log-format.md`). Every log line must carry `tenant_id`, `trace_id`, `span_id`, and `correlation_id` in the MDC. Stack traces are serialised as a string field — never as multiline stdout output.
- **Metrics:** expose a `/actuator/prometheus` endpoint returning Prometheus text format. The metric exporter scrapes this endpoint on a configurable interval.
- **Traces:** instrument via the OpenTelemetry Java auto-agent (zero code changes). The agent patches the JVM at startup and emits spans to the OTel collector via OTLP/gRPC.

### Kubernetes workloads

Cluster-level signals are collected separately from application signals. `kube-state-metrics` exposes pod restarts, resource requests and limits, deployment rollout status, and job completion state. `node-exporter` exposes node-level CPU, memory, disk, and network metrics. Both expose Prometheus-format endpoints scraped by the metric exporter.

### AWS / GCP infrastructure

Cloud service logs and metrics arrive via adapter services that poll CloudWatch Logs / Cloud Logging APIs and translate them into the platform's internal format before forwarding to Kafka. This is an integration concern, not a core pipeline concern — adapters are thin, stateless, and independently deployable.

### Next.js frontend

The frontend is instrumented with the OpenTelemetry JavaScript SDK for browser trace collection. This enables end-to-end traces that span from browser interaction through Next.js API routes to Spring Boot services — a complete user request trace. Browser spans are emitted to the OTel collector via OTLP/HTTP.

---

## 3. Tier 2 — Collection agents

Collection agents are the ingestion boundary of the platform. They authenticate sources, apply backpressure when quotas are exceeded, buffer locally to absorb downstream slowdowns, and forward to Kafka.

### Log agent

Runs as a Kubernetes DaemonSet — one pod per node, automatically scheduled by K8s. Tails container stdout/stderr logs from `/var/log/containers/`, parses them as JSON (or falls back to a raw message field for non-JSON output), validates that `tenant_id` is present, and forwards batches to the `logs-raw` Kafka topic.

**Local disk buffering is mandatory.** The agent maintains a local write-ahead buffer on node disk before forwarding to Kafka. If Kafka is temporarily unavailable or slow, the agent continues tailing logs and accumulates them locally. On Kafka recovery, it replays the buffer. This is the mechanism that satisfies ING-05 (ingestion continues during downstream degradation) and contributes to the zero-loss durability guarantee.

**Quota enforcement at the agent.** Before forwarding a batch, the agent checks the quota service (Redis) via a lightweight HTTP call. If the tenant is over quota, the agent drops the batch and returns HTTP 429 to the source. The quota check must add less than 10ms to the ingestion path — enforced by the Redis-based quota service design (NFR: quota enforcement latency < 10ms).

Fluent Bit is supported as an alternative agent (ING-07, v2) for tenants that already run it in their infrastructure.

### Metric exporter

A Prometheus-compatible scraper that polls `/actuator/prometheus` endpoints on all registered services and forwards metrics to the `metrics-raw` Kafka topic. It also scrapes `kube-state-metrics` and `node-exporter` endpoints registered at cluster level.

The metric exporter is a stateless deployment — multiple replicas can scrape the same endpoint without coordination issues, as VictoriaMetrics deduplicates identical time-series samples within its ingestion window.

### OTel SDK / collector

The OpenTelemetry Collector runs as a K8s Deployment (not a DaemonSet — it receives pushed spans rather than tailing files). Spring Boot services configured with the OTel Java auto-agent push spans via OTLP/gRPC to the collector's receiver endpoint. The collector applies a head-based sampling policy (10% of traces, configurable per service), adds `tenant_id` as a span attribute from the service's auth context, and forwards to the `traces-raw` Kafka topic via the Kafka exporter.

---

## 4. Tier 3 — Streaming pipeline

The streaming pipeline is the write path's backbone. It decouples ingestion throughput from storage write throughput and provides the durability buffer that absorbs traffic spikes up to 10× sustained load.

### Kafka cluster

Three topics, one per signal type. All decisions governed by ADR-002.

| Topic | Partitions | Replication | Retention | Compression |
|-------|-----------|-------------|-----------|-------------|
| `logs-raw` | 12 | 3 | 72 hours | lz4 |
| `metrics-raw` | 12 | 3 | 72 hours | lz4 |
| `traces-raw` | 12 | 3 | 72 hours | lz4 |

**Partition key:** `hash(tenant_id + service_name) % 12`. This distributes a single tenant's traffic across multiple partitions (preventing hot partitions from a single heavy tenant) while preserving message ordering per service within a tenant (useful for stateful stream processing operations).

**Why 72-hour topic retention?** This is not the signal retention window (that is governed by storage TTLs). This is the Kafka replay window. If the stream processor falls behind or crashes, it can replay up to 72 hours of unprocessed messages from Kafka without needing to re-ingest from sources. This is the recovery mechanism for the "at-most 30s data loss on catastrophic failure" durability NFR.

**Schema Registry** governs the Avro schemas for `metrics-raw` and `traces-raw` topics, enabling schema evolution without breaking consumers. `logs-raw` uses JSON (no schema registry) because log formats evolve frequently and the stream processor handles unknown fields gracefully.

### Stream processor

Consumes from all three topics, processes messages, and writes to the appropriate storage backends. This is the most operationally complex component in the write path.

**Responsibilities per signal type:**

For logs: parse raw JSON, validate all required fields are present, validate `tenant_id` against the auth registry, extract structured fields (`level`, `service`, `trace_id`, etc.), enrich with service metadata (K8s labels, deployment version), normalise timestamps to UTC, and write to ClickHouse in batches of 10,000 rows or 1-second windows (whichever comes first). Also publishes parsed log lines to Redis Pub/Sub for the live-tail feature.

For metrics: deserialise Prometheus text format or Avro, validate `tenant_id`, translate to VictoriaMetrics remote write format, and write in batches. Apply label normalisation (remove high-cardinality labels that would cause metric explosion — see cross-cutting concerns).

For traces: deserialise OTLP protobuf spans, validate `tenant_id` on each span, reconstruct span batches, and forward to the Tempo distributor via OTLP/gRPC.

**Dead-letter routing.** Messages that fail validation (missing `tenant_id`, unparseable format, schema mismatch) are routed to signal-specific dead-letter topics (`logs-dead-letter`, `metrics-dead-letter`, `traces-dead-letter`). Dead-letter topics are monitored and alerted on — a non-zero dead-letter rate is always an operational signal, never an expected condition.

**Technology choice for the stream processor is an open question (OQ-05).** The two candidates are Python-based (Faust or Bytewax) and JVM-based (Apache Flink). The Python path integrates more naturally with the AI service. The Flink path is more operationally mature for exactly-once semantics and stateful windowed operations. This decision is not yet made — see section 13.

---

## 5. Tier 4 — Storage backends

Four storage systems, each chosen for a specific data model and query pattern. No single storage system was used for all signals — this was an explicit design decision driven by the fundamentally different access patterns of logs (full-text search, recent window queries), metrics (time-series aggregations, PromQL), traces (point lookups by ID, span tree reconstruction), and embeddings (approximate nearest-neighbour search).

### ClickHouse — log storage

ClickHouse is the primary storage backend for log events. It is a columnar OLAP database optimised for analytical queries over large volumes of structured data — exactly the access pattern of log search (filter by service + time window, aggregate by level, full-text match on message).

**Schema:** Governed by ADR-001. Shared table with `tenant_id` as the first partition key:

```sql
PARTITION BY (tenant_id, toYYYYMM(timestamp))
ORDER BY (tenant_id, timestamp, service)
TTL timestamp + INTERVAL 30 DAY
```

Partitioning by `tenant_id` first means a query scoped to one tenant only reads that tenant's partitions — partition pruning provides physical data locality without per-tenant tables. The `ORDER BY` key means queries filtering by `(tenant_id, timestamp range, service)` hit the primary index with no full scan.

**Query enforcement:** Every ClickHouse query is constructed through a centralised `QueryBuilder` in the Spring Boot API gateway. The `QueryBuilder` requires a `tenant_id` binding at construction time — it cannot produce a query without one. This is the ADR-001 enforcement mechanism. Integration tests assert that no query type can return rows belonging to a different tenant.

### VictoriaMetrics — metric storage

VictoriaMetrics is a Prometheus-compatible time-series database optimised for high write throughput and efficient long-term storage. It accepts Prometheus remote write and responds to PromQL queries.

It was chosen over Prometheus itself for two reasons: it handles the 100,000 active series target at significantly lower memory footprint than Prometheus, and it supports longer retention windows (90 days) efficiently through its own compressed storage format. Prometheus at 90-day retention at this cardinality would require significantly more disk and memory.

**Tenant isolation in VictoriaMetrics** is enforced via label filtering. Every metric series ingested carries a `tenant_id` label injected by the stream processor. Every PromQL query executed by the API gateway includes a `{tenant_id="..."}` selector injected by the query layer — the same pattern as ClickHouse. VictoriaMetrics's multi-tenancy support (via its enterprise edition or the open-source label-based approach) is evaluated in OQ-05's resolution context.

### Grafana Tempo — trace storage

Grafana Tempo stores distributed traces with S3 as the storage backend. Governed by ADR-003. Traces are written to S3 as compressed Parquet blocks via Tempo's ingester component, indexed by `trace_id` via a bloom filter maintained on local disk.

The Tempo compactor runs as a background deployment and merges small S3 blocks into larger ones over time. An unhealthy compactor does not cause data loss but degrades query performance — it is monitored and alerted on.

**Tenant isolation in Tempo** is enforced via span attribute filtering. Every span carries a `tenant_id` attribute. Tempo queries from the API gateway include a `{.tenant_id="..."}` TraceQL filter — Tempo evaluates this against the bloom filter and S3 blocks.

### Redis — operational state

Redis serves four distinct roles in the platform, all requiring fast in-memory access:

- **Quota tracking:** Per-tenant ingestion counters stored as Redis keys with a TTL. The log agent and metric exporter INCR the counter on every batch; the quota service checks it synchronously. The counter resets on TTL expiry (daily quota window). This satisfies the quota enforcement latency NFR (< 10ms).
- **Live-tail pub/sub:** The stream processor publishes parsed log lines to a Redis Pub/Sub channel keyed by `tenant_id`. The Spring Boot API gateway subscribes to the tenant's channel and fans out to all connected WebSocket clients. This is the mechanism for the live-tail freshness NFR (< 3s end-to-end).
- **Alert state machine:** Active alert states (firing, resolved, suppressed) are stored in Redis with TTLs. The alert engine reads and writes alert state here to implement deduplication (ALT-04) — it will not re-fire an alert that is already in `firing` state in Redis.
- **Query result cache:** Recent log search results and metric panel data are cached with short TTLs (10–30 seconds) to reduce ClickHouse and VictoriaMetrics query load under the 100 concurrent user target.

### pgvector — embedding storage

pgvector is a PostgreSQL extension for approximate nearest-neighbour vector search. It stores log line embeddings generated by the AI log clustering service. When the clustering pipeline embeds a batch of log lines, it writes the embedding vectors to pgvector. At query time, the AI service retrieves semantically similar log lines by vector distance rather than keyword match.

pgvector was chosen over a dedicated vector database (Pinecone, Weaviate) because it runs on standard PostgreSQL — no additional operational dependency. At the scale of log embeddings for this platform, PostgreSQL with pgvector is sufficient. A dedicated vector database would be reconsidered if embedding volume exceeded tens of millions of vectors with sub-100ms query requirements.

---

## 6. Tier 5 — API and backend services

The API and backend tier is the read path entry point and the orchestration layer for alerting and notifications. All components are Spring Boot services deployed on Kubernetes with stateless, horizontally scalable deployments.

### Spring Boot API gateway

The primary interface between the frontend (and AI service) and the storage backends. All client traffic enters the platform here.

**Responsibilities:**

- **Authentication and tenant context:** Validates JWTs on every request. Extracts `tenant_id` and `role` from token claims. Injects tenant context into every downstream query — no request proceeds without a valid tenant context.
- **RBAC enforcement:** Three roles (admin, editor, viewer — TNT-04). Role checks are applied at the handler level, not the storage level. An editor cannot call admin APIs; a viewer cannot modify dashboards or alert rules.
- **Query routing:** Routes log queries to ClickHouse via `QueryBuilder`, metric queries to VictoriaMetrics via PromQL, trace lookups to Tempo via TraceQL. The gateway never accesses storage directly with raw queries — always through the query builder abstractions that enforce tenant isolation.
- **WebSocket server for live tail:** Maintains WebSocket connections for active live-tail sessions. Subscribes to the Redis Pub/Sub channel for the requesting tenant and fans out incoming log lines to all connected clients for that tenant. Connection lifecycle is managed with heartbeats and graceful reconnection.
- **AI service proxy:** Forwards AI copilot requests (NL query, RCA trigger, anomaly explanation) to the Python AI service. Streams token-by-token responses back to the frontend via SSE.
- **Rate limiting:** Per-tenant, per-endpoint rate limits enforced at the gateway. Heavy operations (7-day log searches, full RCA runs) have lower limits than lightweight operations (live tail, single metric panel).

### Alert engine

A Spring Boot service that evaluates alert rules on a Quartz-scheduled 30-second cadence. This is a separate service from the API gateway — it has no user-facing traffic and runs on its own deployment with its own scaling policy.

**Evaluation cycle (every 30 seconds):**
1. Load all active alert rules for all tenants from the rules database (PostgreSQL).
2. For each rule, execute the PromQL expression against VictoriaMetrics with a sliding window covering the last evaluation period.
3. Compare the result against the rule's threshold condition.
4. Check Redis for the current alert state (`firing`, `resolved`, `suppressed`).
5. If the condition is breached and the alert is not already `firing`, set state to `firing` in Redis and enqueue a notification to SQS.
6. If the condition is no longer breached and the alert is currently `firing`, set state to `resolved` in Redis and enqueue a resolution notification.
7. Deduplication: if the alert is already `firing` and the condition is still breached, do nothing — do not re-fire.

**Anomaly-based alerts (v2):** The alert engine will additionally consume anomaly scores published by the AI anomaly detector. If a metric's anomaly score exceeds a tenant-configured threshold, it triggers the same notification flow as a threshold alert. This is an additive change to the evaluation cycle — the core scheduler and notification flow remain unchanged.

### Notification service

Delivers alert payloads to configured destinations (Slack, email, PagerDuty in v2). Consumes from an SQS queue populated by the alert engine, ensuring that notification delivery is decoupled from alert evaluation — a slow Slack API does not slow down alert rule evaluation.

**Delivery guarantees:** At-least-once delivery via SQS. Each notification carries an idempotency key (`alert_id + firing_timestamp`). If the same notification is delivered twice (SQS retry), the destination deduplicates by idempotency key. Delivery state (delivered, failed, retrying) is persisted per notification per destination.

**Retry policy:** Exponential backoff starting at 5 seconds, up to 5 retries, with a maximum backoff of 5 minutes. After 5 failures the notification is moved to a dead-letter queue and the tenant admin is alerted via email (using a fallback SMTP path that bypasses the queue).

---

## 7. Tier 6 — Frontend

The frontend is a Next.js application. All four primary surfaces are v1.

### Log explorer

The primary debugging interface. Serves the backend engineer persona during incident response.

Key engineering challenges: virtual scrolling over potentially millions of matching log rows (the browser cannot hold them all in the DOM), real-time streaming of new lines via WebSocket without disrupting an active scroll position, and debounced field-level filters that retrigger the ClickHouse query on each keystroke change without flooding the API.

The query state (filters, time range, selected fields) is reflected in the URL as query parameters — making searches bookmarkable and shareable (LOG-04).

Clicking a `trace_id` field in any log row navigates directly to the trace viewer with that `trace_id` pre-loaded (LOG-06). This is the primary cross-signal navigation flow.

### Metric dashboards

Serves both the engineering manager persona (pre-built service dashboard, MET-01) and the backend engineer persona (ad-hoc panel builder, MET-02).

Dashboard layout and panel configurations are persisted per tenant in PostgreSQL. The frontend fetches the layout on load and issues individual PromQL queries for each panel in parallel, rendering as results arrive. A dashboard-level time range selector (MET-04) updates all panel queries simultaneously via a shared React context.

Live metric updates on dashboard panels use SSE rather than WebSocket — panels poll the API via SSE streams on a configurable refresh interval (default 30 seconds). This matches the alert evaluation cadence and avoids the complexity of bidirectional WebSocket for a one-way data flow.

### Trace viewer

Renders a distributed trace as a Gantt/waterfall chart (TRA-02). Each span is a horizontal bar positioned by start time and sized by duration. Spans are grouped by service and coloured consistently per service name. Error spans are highlighted in red.

Clicking a span opens an attribute inspector panel showing all span attributes, events, and links. The log correlation panel below the Gantt chart (TRA-04) shows all log lines carrying the same `trace_id` — fetched from ClickHouse on span click rather than on initial trace load (lazy fetch for performance).

### AI copilot chat

A chat interface that streams AI responses token-by-token via SSE. The frontend maintains the full conversation history in React state and sends it with every message — the AI service is stateless between requests.

Responses from the AI service may include structured blocks: a SQL query (rendered as a code block with a "Run query" button that opens the log explorer pre-populated), an anomaly chart (rendered inline), or a structured RCA report (rendered as a collapsible section with citations linking to specific log lines and trace spans). The frontend parses these structured blocks from the streamed response and renders them appropriately.

---

## 8. AI service

The AI service is a Python FastAPI application. It is deployed as a separate service from the Spring Boot API gateway — it has different scaling characteristics (CPU-intensive, GPU-optional), a different language runtime, and different dependency requirements (ML libraries). The API gateway proxies AI requests to it.

The AI service is stateless — all context required for a request (conversation history, tenant_id, query parameters) is passed in the request body.

**Technology choice for the LLM backend is an open question (OQ-04).** The two candidates are API-based (Claude or GPT-4o) and self-hosted (Ollama + Llama 3). This choice affects the NL-to-SQL and RCA components significantly. See section 13.

### NL to SQL (AI-01)

Translates a natural language query into a valid ClickHouse SQL query scoped to the requesting tenant's log schema.

The pipeline: receive the natural language string and tenant context → inject the ClickHouse schema and a set of few-shot examples into the LLM system prompt → call the LLM to generate SQL → validate the generated SQL against an allowlist of safe operations (SELECT only, no JOINs across tenants, must include tenant_id filter) → return the SQL to the API gateway for execution via `QueryBuilder`.

The generated SQL is never executed directly — it passes through the same `QueryBuilder` path as all other queries, which injects the `tenant_id` binding regardless of what the LLM generated. This is the security boundary: LLM output is treated as untrusted user input, not as trusted internal SQL.

### Anomaly detector (AI-02)

Runs statistical time-series forecasting on ingested metrics to surface anomalies that do not breach static thresholds.

The pipeline: on a configurable schedule (every 5 minutes), fetch the last N hours of metric data from VictoriaMetrics for each monitored series → fit a Prophet or ARIMA model to the historical data → compute expected value and confidence interval for the current window → score the actual value against the expected range → publish anomaly scores above a threshold to a Redis channel consumed by the alert engine.

The anomaly detector operates asynchronously and does not affect the query path latency. Its outputs feed the alert engine as a supplementary signal source (ALT-06, v2).

### Root cause analyser (AI-03)

Produces a human-readable RCA report when triggered by an alert or manually by an engineer.

The pipeline: receive the alert ID and time window → fetch the correlated signals from storage (log error rate from ClickHouse, metric values from VictoriaMetrics, trace error spans from Tempo) for the 30-minute window around the alert → construct a structured context document containing the correlated signals → call the LLM with the context and a system prompt asking for a structured RCA → return the report with citation references back to specific log lines and spans.

The RCA is a human-triggered operation (AI-03 is v1; autonomous triggering is AI-05, v2). The report is structured with sections: summary, contributing signals, probable root cause, affected services, recommended actions. Each claim in the report cites a specific log line ID or span ID that the frontend can link to.

### Log clustering (AI-04)

Groups semantically similar log lines to surface novel error patterns that no alert rule covers.

The pipeline: on a configurable batch schedule, fetch the last hour of `ERROR` and `WARN` log lines from ClickHouse for each tenant → embed each log message using a sentence-transformers model → run HDBSCAN clustering on the embedding space → for each cluster, select a representative log line and generate a human-readable cluster label via LLM → store clusters and embeddings in pgvector → surface novel clusters (those not seen in the previous batch) to the frontend as "new pattern detected" notifications.

This feature teaches the engineer about failure modes in their system that they did not know to alert on — it is discovery-oriented rather than response-oriented.

---

## 9. Multi-tenant control plane

The control plane is not a single service — it is a set of cross-cutting concerns implemented across multiple components. It is represented as a conceptual layer in the architecture diagram to make its scope explicit.

### Auth and JWT

Every request to the platform carries a JWT signed by the platform's auth service. The JWT payload contains `tenant_id`, `role`, `user_id`, and `exp`. The API gateway validates the JWT signature and expiry on every request before any other processing. The `tenant_id` claim is the authoritative source of tenant context — it cannot be overridden by request parameters.

Agent authentication uses a separate mechanism: service account tokens issued per tenant, validated by the collection agents before forwarding data to Kafka. The agent token carries `tenant_id` which the stream processor validates against the message header on every consumed message.

### Quota enforcement

Per-tenant daily ingestion quotas are configured at provisioning time and stored in PostgreSQL. The quota service (a thin component in the Spring Boot API gateway) checks the current usage against the limit on every ingestion request. Current usage is maintained in Redis as a per-tenant counter incremented by the stream processor on each batch write. The counter TTL resets at the start of each quota window (daily by default).

The quota check is on the hot path — the log agent calls it synchronously before forwarding to Kafka. This is why Redis was chosen (not PostgreSQL) — the check must add less than 10ms to the ingestion path.

When a tenant exceeds their quota, the quota service returns HTTP 429 to the calling agent. The agent drops the batch locally (not buffered to disk — quota-exceeded batches are intentionally discarded) and logs a quota-exceeded event. The tenant admin receives a quota warning notification via email at 80% and 100% usage.

### Tenant provisioning

Manual for v1 (OQ-06 resolution pending). A tenant is provisioned by calling an admin API that:
1. Creates the tenant record in PostgreSQL with quota limits and retention configuration.
2. Initialises the Redis quota counter.
3. Issues a service account token for agent authentication.
4. Creates ClickHouse TTL policy entries for the tenant's partition.

Self-serve onboarding (OQ-06, v2) will wrap this admin API in a UI flow.

### RBAC

Three roles with additive permissions:

| Permission | viewer | editor | admin |
|-----------|--------|--------|-------|
| Search logs, view dashboards, view traces | ✓ | ✓ | ✓ |
| Create/edit dashboards, create alert rules | — | ✓ | ✓ |
| Manage tenant settings, view quota usage | — | — | ✓ |
| Provision tenants, manage service account tokens | — | — | ✓ |

RBAC is enforced at the API gateway handler level. Storage backends do not implement role checks — they rely entirely on the gateway's tenant context injection.

---

## 10. Critical data flows

These are the five most important data flows in the system. Understanding them end-to-end is the most effective way to reason about the architecture's correctness.

### Flow A — Log ingestion (write path)

```
Spring Boot service
  → stdout (JSON log line)
  → Log agent (tail /var/log/containers/)
  → [quota check: Redis INCR, < 10ms]
  → [local disk buffer]
  → Kafka logs-raw (tenant_id in header, partition by hash(tid+svc))
  → Stream processor (validate, parse, enrich)
  → ClickHouse batch write (10k rows or 1s window)
  → [Redis Pub/Sub publish for live tail]
```

Total write path latency: 1–3 seconds from log emission to ClickHouse persistence. Live tail freshness (< 3s NFR) is satisfied by the Redis Pub/Sub branch which publishes before the ClickHouse write completes.

### Flow B — Live tail (read path, real-time)

```
Engineer opens live tail in log explorer
  → Next.js opens WebSocket to API gateway
  → API gateway subscribes to Redis Pub/Sub channel: logs:{tenant_id}
  → Stream processor publishes parsed lines to channel on each message
  → API gateway receives line, filters by active query params
  → WebSocket push to browser
```

The Redis Pub/Sub channel is the critical component here — it is what decouples the streaming freshness from the ClickHouse write latency. Log lines appear in the live tail before they are queryable in the log explorer search.

### Flow C — Incident debugging (primary use case)

```
Alert fires → engineer receives Slack notification
  → Opens log explorer, filters by service + 15m window around alert time
  → [ClickHouse query with tenant_id + timestamp + service filters]
  → Finds error log line, clicks trace_id
  → Trace viewer loads trace from Tempo by trace_id
  → Trace viewer lazy-fetches correlated logs from ClickHouse (same trace_id)
  → Engineer triggers AI RCA from alert context
  → AI service fetches correlated signals from ClickHouse + VM + Tempo
  → LLM generates structured RCA report
  → Report displayed in copilot chat with citations linking to log lines + spans
```

This flow exercises every read-path component. Performance is dominated by the ClickHouse query latency (NFR: p95 < 1s for 24h window).

### Flow D — Alert evaluation (background path)

```
Quartz scheduler fires every 30 seconds
  → Alert engine loads active rules for all tenants
  → For each rule: PromQL query to VictoriaMetrics
  → Compare result against threshold
  → Check Redis for current alert state
  → If breached and not firing: set Redis state, enqueue SQS notification
  → Notification service dequeues, calls Slack API / SMTP
  → Tenant engineer receives notification
```

End-to-end alert latency: 0–30s from threshold breach to notification (depends on where in the 30s evaluation cycle the breach occurs). Satisfies ALT-02 and the < 30s alert latency NFR.

### Flow E — AI NL log search (AI copilot path)

```
Engineer types natural language query in copilot chat
  → Next.js sends message to API gateway (includes conversation history)
  → API gateway forwards to AI service with tenant context
  → AI service: inject schema + few-shot examples into LLM prompt
  → LLM generates ClickHouse SQL
  → AI service validates SQL (SELECT only, no cross-tenant refs)
  → API gateway executes SQL via QueryBuilder (injects tenant_id binding)
  → ClickHouse executes query, returns results
  → API gateway streams results + SQL back to AI service
  → AI service generates natural language summary of results
  → SSE stream delivers token-by-token to frontend
  → Frontend renders SQL as runnable code block + summary as markdown
```

The `QueryBuilder` is the security boundary: even if the LLM generates SQL without a `tenant_id` filter, the `QueryBuilder` injects it unconditionally.

---

## 11. Cross-cutting concerns

### High cardinality metric labels

The most common operational mistake when deploying Prometheus-style metrics. A label like `user_id` or `request_id` creates a new time-series for every unique value — at 5,000 requests/sec this explodes the active series count from 100,000 to millions within minutes, causing VictoriaMetrics to OOM. The stream processor must apply a label allowlist (configurable per tenant) and strip any label not on the allowlist before writing to VictoriaMetrics. Services must not put unbounded values in metric labels.

### Correlation ID propagation

Every inbound request to any service must carry an `X-Correlation-ID` header. If absent, the receiving service generates one and propagates it to all downstream calls. The correlation ID is injected into the MDC at request ingress (by the `LogContextFilter`) so it appears in every log line emitted during that request. This is distinct from the OTel `trace_id` — the correlation ID is a business-level request identifier that survives sampling (traces may be sampled away; the correlation ID never is).

### Zero-downtime deployments

All services are stateless (Spring Boot API gateway, AI service, notification service, stream processor). State lives in storage (ClickHouse, VictoriaMetrics, Tempo, Redis, PostgreSQL). Kubernetes rolling updates replace pods one at a time — new pods start receiving traffic only after their health check passes, and old pods drain existing connections before termination. This satisfies the zero-downtime deploy NFR without additional coordination.

The alert engine is the one exception to stateless deployment: its Quartz scheduler state is persisted in PostgreSQL (not in-memory) so that scheduler state survives pod restarts. Only one alert engine pod is active at a time (replicas=1) to avoid duplicate alert evaluations. This is a conscious simplification for v1 — a distributed lock (Redis SETNX) would allow multiple replicas in v2.

### Structured log format

Defined separately in `docs/log-format.md`. The stream processor's parser is the authoritative consumer of the format — changes to the format must be validated against the parser before deployment. The key constraint: stack traces must be serialised as a single-line string field, never as multiline stdout output. Multiline output breaks the agent's line-oriented parser and causes log lines to be silently dropped.

---

## 12. Decisions resolved since requirements doc

The following open questions from `hld-requirements.md` section 7 have been resolved. The requirements doc should be updated to close these questions and reference the ADRs.

| OQ ID | Question | Resolution | ADR |
|-------|----------|------------|-----|
| OQ-01 | ClickHouse isolation model | Shared tables with `tenant_id` column, `PARTITION BY (tenant_id, toYYYYMM(timestamp))` | ADR-001 |
| OQ-02 | Kafka topic strategy | Three shared topics, `tenant_id` in message header, `hash(tenant_id + service_name)` partition key | ADR-002 |
| OQ-03 | Trace storage backend | Grafana Tempo with S3 backend | ADR-003 |
| OQ-06 | Tenant provisioning model | Manual via admin API for v1; self-serve UI deferred to v2 | No ADR — documented in section 9 |

---

## 13. Open questions

Two open questions from the requirements doc remain unresolved and affect component design decisions. These must be resolved before detailed component design (LLD) begins for the AI service and stream processor.

### OQ-04 — LLM provider for AI service

**Question:** Self-hosted LLM (Ollama + Llama 3) vs API-based LLM (Claude / GPT-4o)?

**Impact:** Affects the NL-to-SQL pipeline (AI-01), the RCA generator (AI-03), and the log clustering label generator (AI-04). Self-hosted avoids sending tenant log data to a third-party API — this is a significant data privacy consideration for a multi-tenant observability platform where log data may contain sensitive information. API-based is faster to build and produces better results today. A hybrid approach (self-hosted for data-sensitive operations, API-based for general assistance) is worth evaluating.

**Blocking:** AI service LLD, AI service infrastructure sizing (GPU nodes vs CPU-only).

### OQ-05 — Stream processor technology

**Question:** Python-based (Faust or Bytewax) vs JVM-based (Apache Flink)?

**Impact:** Affects the entire write path. Python integrates naturally with the AI service (shared libraries, same language) and reduces the number of language runtimes in the platform. Flink is operationally more mature for exactly-once semantics at scale and has better tooling for windowed aggregation and stateful operations. Bytewax is newer, pure-Python, and simpler to operate than Flink, but has a smaller community and less proven production history.

**Blocking:** Stream processor LLD, K8s deployment configuration for the write path.

---

*Previous section: [Requirements and scale estimates ←](./hld-requirements.md)*  
*Next section: [Low-level design — ingestion pipeline →](./lld-ingestion-pipeline.md)*
