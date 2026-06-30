# Phase 4 — Aggregation & Analytics Reporting APIs

> **Status:** Greenfield Specification | **Scope:** Build the read-path analytical aggregation and reporting APIs that query ClickHouse to deliver organization, tenant, user, model, time-series, and cost analytics.
>
> This is the **Phase 4 blueprint** — the reporting API specification. It defines the GET analytical routes querying ClickHouse (`events.usage_events`). Because ClickHouse is a columnar database designed for high-performance scans rather than individual updates, these APIs must utilize batch-aggregation queries, read from the deduplicated view, and be resilient under heavy analytical query loads.

---

## Description

As a **platform tenant, manager, or billing admin**, I need a comprehensive set of reporting APIs to fetch aggregate usage statistics (total events, token counts, cost estimations) grouped by organization, tenant, model, provider, or individual user, over custom time-series buckets (hourly, daily, weekly, monthly).

These reporting APIs represent the **read path** of the billing platform. To prevent performance degradation, they query the `events.usage_events_dedup_v` view rather than the raw events table, avoiding duplicate event counts. They use process-local caching where possible and run queries under strict execution limits.

### Architecture Flow

```
Dashboard/Client → GET /v1/orgs/{org_id}/summary → Ingest API Service → ClickHouse (Select & Aggregate) → JSON response
```

---

## RBAC / Auth Context

The aggregation APIs enforce multi-tenant isolation to ensure organizations can only view their own usage data:

| Actor | Allowed Actions | Restriction |
|---|---|---|
| **Platform Administrator** | Read all metrics globally | No filters enforced. |
| **Org Manager** | Read summaries for their org and its tenants | Queries must explicitly filter ClickHouse by the authenticated `org_id`. |
| **Tenant User** | Read summaries for their specific `tenant_id` | Queries must filter ClickHouse by both `org_id` and `tenant_id`. |

---

## Acceptance Criteria

### ClickHouse Dedup Constraint
1. All analytical queries MUST read from the view `events.usage_events_dedup_v` (which uses `argMax(column, ingested_at)` grouping) to prevent reporting inflated token counts from duplicate ingest requests.

### Concurrency and Timeout Restrictions
2. Every database query must have a `context.WithTimeout` limit of 5 seconds to ensure slow-running aggregations do not lock connection slots.
3. Use a thread-safe connection pooling structure for the ClickHouse Go client, allowing up to 10 parallel queries.

### Parameter Validation
4. Standardize pagination: query parameters `limit` (max 1000) and `offset` must be verified. Negative values return `400 BAD_REQUEST`.
5. Standardize date parameters: `start_date` and `end_date` must conform to `YYYY-MM-DD` ISO-8601 strings. Malformed strings return `400 BAD_REQUEST` with details.

### Empty Data Resilience
6. If no events match the queried parameters, return `200 OK` with zeroed summaries and empty collections (e.g. `"total_events": 0`, `"models": []`), never `404 NOT_FOUND` or null results.

---

## Phase 4 Completion Checklist

- [ ] All aggregation handler routes registered under the `/v1` prefix in [`handler.go`](file:///c:/Users/Administrateur/Downloads/pente.ai%202/pente_quantum_be/event-engine-core/internal/api/handler.go)
- [ ] Multi-tenant isolation middleware verified for read paths (blocking cross-org metric queries)
- [ ] Database query layer using Go ClickHouse native client (port 9000) implemented in [`clickhouse_aggregations.go`](file:///c:/Users/Administrateur/Downloads/pente.ai%202/pente_quantum_be/event-engine-core/internal/db/clickhouse_aggregations.go)
- [ ] All queries target `events.usage_events_dedup_v` (dedup-safe view)
- [ ] Parameter validation helpers implemented for dates, limits, and offsets
- [ ] Structured JSON models for aggregation responses mapped in [`aggregations.go`](file:///c:/Users/Administrateur/Downloads/pente.ai%202/pente_quantum_be/event-engine-core/internal/models/aggregations.go)
- [ ] OpenTelemetry child tracing spans wrapped around ClickHouse query executions
- [ ] Automated integration tests covering empty database states and range validations
