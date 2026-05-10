# ADR-001 — ClickHouse multi-tenant isolation model

| Field | Value |
|-------|-------|
| **Status** | Accepted |
| **Date** | April 2026 |
| **Deciders** | Engineering |
| **Consulted** | — |

---

## Context

The platform is a multi-tenant observability system storing logs, metrics metadata, and event data in ClickHouse. Each tenant's data must be isolated — no query may return data belonging to a different tenant. We also need per-tenant storage quota enforcement and the ability to apply schema changes as the platform evolves.

Two structural approaches exist for isolating tenant data in ClickHouse: shared tables with a `tenant_id` discriminator column, or separate databases per tenant.

At launch the platform targets 10 active tenants, designed to scale to 50 without architectural changes.

---

## Decision

**We will use shared tables with a `tenant_id` column and partition by `(tenant_id, toYYYYMM(timestamp))`.**

All tenants' log events will live in a single `logs` table. Tenant isolation is enforced logically at the query layer — every ClickHouse query executed by the platform is constructed through a centralised `QueryBuilder` that injects `AND tenant_id = :tenantId` before execution. No query path may bypass this builder.

The core log table schema:

```sql
CREATE TABLE logs (
    tenant_id     String,
    timestamp     DateTime64(3),
    service       String,
    level         LowCardinality(String),
    message       String,
    trace_id      String,
    host          String,
    attributes    Map(String, String)
)
ENGINE = MergeTree()
PARTITION BY (tenant_id, toYYYYMM(timestamp))
ORDER BY (tenant_id, timestamp, service)
TTL timestamp + INTERVAL 30 DAY;
```

The same shared-table pattern applies to metrics metadata and trace index tables.

---

## Consequences

### Positive

- **Single schema to maintain.** Schema migrations (`ALTER TABLE`, TTL changes, index additions) run once and apply to all tenants immediately. With per-tenant databases this would require orchestrated migrations across N databases.
- **Operational simplicity.** One ClickHouse database, one set of tables, one monitoring target. No per-tenant provisioning step when onboarding a new tenant.
- **Physical isolation via partitioning.** Partitioning by `tenant_id` first means ClickHouse stores each tenant's data in separate parts on disk. A query scoped to tenant A only reads tenant A's partitions — partition pruning provides data locality without per-tenant databases.
- **Cross-tenant analytics ready.** If we ever need platform-level analytics (e.g. aggregate ingestion volume across all tenants for billing), a single table makes this trivial. Per-tenant databases would require distributed queries.
- **How production platforms operate.** Datadog, Grafana Cloud, and similar multi-tenant observability platforms use this model at massive scale.

### Negative

- **Isolation is app-enforced, not DB-enforced.** A bug in the query layer could theoretically leak cross-tenant data. This risk is mitigated by the `QueryBuilder` constraint and mandatory test coverage (see below), but it is not zero. Per-tenant databases would make cross-tenant leakage structurally impossible.
- **Noisy tenant risk.** A single tenant running an expensive full-scan query can consume ClickHouse I/O and slow queries for other tenants. Mitigated by per-tenant query timeout limits and query complexity caps enforced at the API layer.
- **Quota enforcement is indirect.** ClickHouse does not natively report per-tenant row counts cheaply. We maintain a `tenant_quota_usage` metadata table (updated by the stream processor on each batch write) and check it via Redis cache on the ingestion path.

---

## Enforcement rules

These rules are non-negotiable and must be reviewed in every PR that touches the query layer:

1. **All ClickHouse queries must be constructed through `QueryBuilder`.** No raw query strings assembled outside this class.
2. **`QueryBuilder` must reject construction of any query missing a `tenant_id` binding.** This is enforced at compile time via a required constructor parameter, not a runtime check.
3. **Integration tests must assert tenant isolation.** For every query type (log search, aggregation, live tail), a test inserts data for two tenants and asserts that querying as tenant A returns zero rows belonging to tenant B.
4. **Partition key must never be changed** without a full data migration plan. Changing `PARTITION BY` in ClickHouse requires rewriting all existing data.

---

## Alternatives considered

### Option B — Per-tenant databases (`tenant_acme.logs`, `tenant_globex.logs`)

**Rejected.** Physical isolation is stronger — a missing `WHERE` clause cannot leak data across databases. However at 10–50 tenants the operational cost is prohibitive: every schema migration requires N coordinated `ALTER TABLE` executions, new tenant onboarding requires database provisioning, and monitoring requires aggregating across N databases. The isolation benefit does not outweigh this cost at our scale. If the platform ever reaches hundreds of tenants with contractual data residency requirements (e.g. GDPR region isolation), per-tenant databases should be revisited.

---

## Review trigger

Revisit this decision if:
- Tenant count exceeds 100 and noisy-tenant complaints become a support burden
- A contractual data residency or compliance requirement demands physical isolation
- ClickHouse introduces native row-level security that can enforce tenant isolation at the engine level
