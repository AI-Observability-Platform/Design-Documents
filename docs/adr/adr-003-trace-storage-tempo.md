# ADR-003 — Trace storage backend: Grafana Tempo over Jaeger

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | April 2026 |
| **Deciders** | Engineering |
| **Consulted** | — |
| **Related** | ADR-001 (ClickHouse isolation model), ADR-002 (Kafka topic strategy) |

---

## Context

The platform ingests distributed traces in OpenTelemetry format (OTLP) and must store them durably, support lookup by `trace_id`, and render full trace trees in the UI. Traces are retained for 7 days (see requirements document — large payloads, 7 days covers all practical incident lookback windows).

Two production-grade options were evaluated: **Jaeger** (the dominant CNCF tracing backend) and **Grafana Tempo** (a newer, object-storage-native tracing backend). Both support OTLP ingestion and trace lookup by ID. Both have Kubernetes operators.

The platform already runs on AWS/GCP and provisions S3/GCS for other purposes (agent buffering, AI model artifacts). ClickHouse and VictoriaMetrics are already in the stack for logs and metrics respectively — adding a third heavy stateful system solely for traces is a meaningful operational cost.

---

## Decision

**We will use Grafana Tempo with S3 as the storage backend for distributed traces.**

Tempo receives trace spans via OTLP (gRPC and HTTP), writes them to S3 as compressed Parquet blocks, maintains a local bloom filter index for fast `trace_id` lookup, and serves trace queries via its own gRPC query API. The Tempo compactor runs as a separate stateless deployment to merge and deduplicate S3 blocks over time.

The platform's trace viewer (Next.js) queries traces via the Spring Boot API gateway, which calls Tempo's query API. Grafana is also deployed as an internal ops dashboard, with Tempo configured as a datasource — giving the engineering team a native trace exploration UI without building one from scratch for internal use.

Tenant isolation for traces follows the same model as ADR-001 and ADR-002: `tenant_id` is carried through the OTLP pipeline as a span attribute and enforced at the Tempo query layer via tag filtering. Every trace query the API executes includes a `tenant_id` filter.

---

## Architecture

```
OTLP agent  →  Tempo distributor  →  Tempo ingester  →  S3 (trace blocks)
                                                              ↑
                                                      Tempo compactor
                                                      (merges blocks)

Query path:
Next.js UI  →  Spring Boot API  →  Tempo query frontend  →  S3
```

**Tempo components deployed on Kubernetes:**

| Component | Role | Deployment type |
|-----------|------|-----------------|
| Distributor | Receives OTLP, routes to ingesters | Stateless, HPA |
| Ingester | Buffers spans in memory, flushes to S3 | StatefulSet (small) |
| Query frontend | Serves trace lookup queries | Stateless |
| Compactor | Merges and deduplicates S3 blocks | Single replica, CronJob-style |

---

## Consequences

### Positive

- **Dramatically lower operational footprint.** Tempo's only stateful dependency is S3 — a managed service with 99.999999999% durability that requires zero operational effort. Jaeger with Cassandra would add a complex, tuning-intensive stateful cluster to the stack. Jaeger with Elasticsearch would do the same. Tempo eliminates both.
- **S3 is already provisioned.** The platform uses S3 for log agent local buffering and AI model artifacts. Adding a Tempo bucket is a configuration change, not a new infrastructure dependency.
- **Cost profile is predictable and low.** 500 traces/sec after sampling × average 5 spans/trace × 1KB/span = ~2.5 MB/sec raw. Compressed Parquet at ~5:1 = ~500 KB/sec written to S3. At 7-day retention, storage cost is negligible.
- **Grafana integration.** Adopting Tempo naturally brings Grafana into the stack as an internal ops layer — Grafana has first-class datasource plugins for Tempo, VictoriaMetrics (Prometheus-compatible), and Loki. The engineering team gets a polished ops view of the platform itself with minimal additional work.
- **TraceQL is expressive.** Tempo's query language supports filtering by duration, service, span attributes, and status — sufficient for all v1 trace search requirements.
- **Consistent isolation philosophy.** Tenant isolation via `tenant_id` span attribute and query-layer filtering is consistent with ADR-001 (ClickHouse) and ADR-002 (Kafka). One mental model across the entire pipeline.

### Negative

- **Younger ecosystem than Jaeger.** Jaeger has a larger community, more third-party integrations, and more documented operational runbooks. Tempo is production-proven (used by Grafana Cloud at massive scale) but has a smaller community body of knowledge.
- **Query latency is slightly higher for cold traces.** Traces stored in S3 require a bloom filter lookup followed by an S3 object fetch. For a `trace_id` lookup this is typically 100–300ms. Jaeger with a hot Cassandra cluster can serve the same query in 20–50ms. For post-incident investigation this difference is irrelevant; for latency-sensitive trace search it is worth monitoring.
- **Ingester is stateful.** Spans buffered in the ingester's memory are not yet in S3 — a crash before flush loses the most recent ~2 minutes of traces. Mitigated by Kafka-backed ingestion (traces pass through `traces-raw` topic first) and ingester replication (2 replicas in production).
- **Compactor must run continuously.** Without the compactor, S3 accumulates many small block files and query performance degrades over time. The compactor is a simple stateless deployment but it must be monitored and kept healthy.

---

## Enforcement rules

1. **All OTLP spans must include `tenant_id` as a span attribute** set by the OTel SDK instrumentation in each service. Spans missing `tenant_id` are rejected at the Tempo distributor via an attribute filter pipeline.
2. **Every Tempo query executed by the Spring Boot API must include a `tenant_id` tag filter.** This is enforced by the same `QueryBuilder` pattern established in ADR-001 — a shared trace query builder that injects the tenant filter before execution.
3. **The Tempo compactor deployment must be monitored.** Alert on compactor lag exceeding 1 hour. An unhealthy compactor does not cause data loss but degrades query performance over days.
4. **Ingester replication must be set to 2 in production.** `replication_factor: 2` in Tempo's ingester config ensures span data survives a single ingester pod failure before the flush window.

---

## Alternatives considered

### Jaeger with Cassandra backend

**Rejected.** Jaeger is mature and battle-tested. However, running Cassandra as its storage backend adds a complex, operationally demanding stateful cluster to the stack — one that requires careful JVM heap tuning, compaction strategy configuration, and dedicated monitoring. At a platform already running ClickHouse, VictoriaMetrics, Kafka, and Redis, adding Cassandra solely to support trace storage is unjustifiable operational complexity. Cassandra expertise is also a specialised skill; Tempo + S3 requires no such specialisation.

### Jaeger with Elasticsearch backend

**Rejected.** Elasticsearch is more operationally familiar than Cassandra but has its own complexity: index management, shard sizing, mapping explosions on high-cardinality span attributes, and significant memory requirements. The same objection applies — adding Elasticsearch to the stack for one use case (trace storage) when S3-backed Tempo satisfies the same requirements is not justified.

### OpenTelemetry Collector → ClickHouse (traces in ClickHouse)

**Considered but deferred.** Storing traces in ClickHouse alongside logs is architecturally appealing — one storage system, one query interface, native correlation between logs and traces. The ClickHouse community has published trace schemas and the `otel-collector` ClickHouse exporter supports this pattern. This is worth revisiting in v2 if Tempo's operational complexity or query capability becomes limiting. For v1, Tempo is the lower-risk path.

---

## Review trigger

Revisit this decision if:
- Tempo's query latency exceeds acceptable thresholds under production load and Jaeger with a hot index is required
- The team evaluates consolidating trace storage into ClickHouse (the deferred alternative above) as part of a stack simplification effort
- Grafana Tempo introduces breaking changes to the S3 block format that require a full data migration
