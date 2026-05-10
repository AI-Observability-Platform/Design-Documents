# ADR-002 — Kafka topic strategy for multi-tenant ingestion

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | April 2026 |
| **Deciders** | Engineering |
| **Consulted** | — |
| **Related** | ADR-001 (ClickHouse isolation model) |

---

## Context

The platform ingests three signal types — logs, metrics, and traces — from instrumented services across multiple tenants. Kafka sits between the collection agents and the stream processor, acting as a durable buffer that decouples ingestion throughput from storage write throughput.

At peak load the platform handles 5,000 log lines/sec sustained with a 10× spike capacity (50,000 lines/sec). There are 10 active tenants at launch, designed to scale to 50.

Two structural approaches exist for organising Kafka topics in a multi-tenant system: shared topics (one per signal type, all tenants write into the same topic) or per-tenant topics (a dedicated set of topics per tenant).

This decision is made in the context of ADR-001, which established shared ClickHouse tables with logical `tenant_id` isolation as the storage model. Consistency of isolation model across the pipeline is an explicit goal.

---

## Decision

**We will use three shared topics — one per signal type — with `tenant_id` carried in both the message payload and as a Kafka message header.**

```
logs-raw        # all tenants, all log events
metrics-raw     # all tenants, all metric data points
traces-raw      # all tenants, all trace spans
```

Each topic will be created with **12 partitions** and **replication factor 3**.

Messages will be produced with a partition key of `hash(tenant_id + service_name) % num_partitions`. This spreads a single tenant's traffic across multiple partitions, preventing hot partitions when one tenant has significantly higher volume than others.

The stream processor (consumer) reads `tenant_id` from the message header on every message, validates it against the originating agent's auth token, and passes it to the ClickHouse writer — which enforces the `tenant_id` column on every insert per ADR-001.

---

## Consequences

### Positive

- **Operational simplicity.** Three topics to create, monitor, and manage regardless of tenant count. Adding a new tenant requires zero Kafka provisioning — the agent authenticates, starts producing, and messages flow into the existing topics immediately.
- **Consistent mental model with ADR-001.** The entire pipeline — agent → Kafka → stream processor → ClickHouse — follows the same pattern: shared infrastructure, logical tenant separation enforced at the application layer. One model to understand end to end.
- **Kafka is built for this.** At 50,000 lines/sec × 200 bytes = 10 MB/sec peak, shared topics are well within a 3-broker Kafka cluster's throughput ceiling (typically 100–500 MB/sec). Partitioning for per-tenant isolation would be premature optimisation at this scale.
- **Partition count can grow.** Kafka allows increasing partition count without downtime. Starting at 12 gives sufficient parallelism for 200 services across 10 tenants and leaves room to scale.
- **Quota enforcement at the edge removes the main risk of shared topics.** Noisy tenants are throttled at the ingestion API (HTTP 429) before messages reach Kafka — see ING-06 in the requirements document. A quota-throttled tenant cannot flood the shared topic.

### Negative

- **Tenant isolation is application-enforced, not structurally enforced.** A bug in the stream processor that drops the `tenant_id` header could misattribute messages. Mitigated by mandatory header validation before any processing step and integration tests asserting correct tenant attribution.
- **Consumer lag is shared.** If one tenant produces a large burst that slips through quota enforcement (e.g. during a quota service outage), it increases consumer lag for all tenants on that topic. Per-tenant topics would give complete lag isolation. Accepted at current scale; revisit at 50+ tenants.
- **No per-tenant retention policy at the Kafka level.** All messages in a topic share the same retention window (72 hours — enough for replay on processor failure). Per-tenant Kafka retention requires per-tenant topics. Not a current requirement.

---

## Topic and partition configuration

```properties
# logs-raw
num.partitions=12
replication.factor=3
retention.ms=259200000        # 72 hours
max.message.bytes=1048576     # 1 MB max per message
compression.type=lz4

# metrics-raw — same config
# traces-raw — same config
```

**Partition key:** `hash(tenant_id + service_name) % 12`
This ensures:
- A single tenant's traffic is spread across multiple partitions (no hot partition)
- Messages from the same service within a tenant land on the same partition (preserves ordering per service, useful for stream processor stateful operations)

---

## Enforcement rules

1. **Every produced message must include `tenant_id` as a Kafka header** in addition to the payload field. The header is the authoritative source for the stream processor — payload fields are not trusted alone.
2. **The stream processor must validate `tenant_id` from the header against the agent's JWT on every message.** Messages with missing or mismatched `tenant_id` are sent to a dead-letter topic (`logs-dead-letter`, `metrics-dead-letter`, `traces-dead-letter`) and never written to storage.
3. **Dead-letter topics must be monitored and alerted on.** A non-zero dead-letter rate indicates an agent misconfiguration or a potential security issue, not a normal operational condition.
4. **Partition count changes require a migration plan.** Increasing partitions changes the partition key mapping for existing producers — coordinate with agent deployments.

---

## Alternatives considered

### Per-tenant topics (`tenant-acme.logs-raw`, `tenant-globex.logs-raw`)

**Rejected.** Structural isolation is stronger — consumer lag is fully isolated per tenant and per-topic retention is trivially configurable. However the operational cost at 10–50 tenants is significant: 3 topics × 50 tenants = 150 topics to create, monitor, and manage. New tenant onboarding requires Kafka provisioning as a prerequisite. Schema Registry subject management becomes N times more complex. The isolation benefit does not outweigh this cost at current scale, especially given quota enforcement at the ingestion API already prevents the primary failure mode (noisy tenant flooding the topic).

---

## Review trigger

Revisit this decision if:
- Tenant count exceeds 50 and shared consumer lag becomes a measurable support issue
- A tenant requires a contractual guarantee of ingestion isolation (e.g. their messages must not share infrastructure with other tenants)
- Per-tenant retention policies become a product requirement
