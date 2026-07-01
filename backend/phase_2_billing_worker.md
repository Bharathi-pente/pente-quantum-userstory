# Phase 2 — Billing Worker (Kafka → Redis Counters → Invoice Engine)

> **Status:** Greenfield Specification | **Scope:** Build the billing worker — the hot-path consumer that tracks real-time usage in Redis counters, enforces spend limits at the API gateway, and the cold-path periodic engine that generates invoices, applies credits (FEFO), and manages dunning collections.
>
> This is the **Phase 2 blueprint**. Phase 0 built the ingest pipeline. Phase 1 built the analytics worker (ClickHouse). Phase 2 completes the money loop: **every AI token consumed → tracked in real-time → billed at period-end → collected.**

---

## Description

As a **platform operator running an AI proxy billing service**, I need a billing worker that does two things simultaneously:

**Hot path (real-time):** Consumes usage events from the `usage-events` Kafka topic, increments Redis token counters per org/tenant/user, checks against defined spend limits, and exposes an enforcement endpoint so the API gateway (LiteLLM / Phase 5) can block requests that would exceed budget.

**Cold path (periodic):** Reads aggregated usage from ClickHouse at billing period boundaries, applies rate cards and pricing models from Postgres, generates invoices per subscription with line items, auto-applies credits in FEFO (First Expiring, First Out) order, calculates tax, and manages the dunning collection workflow for unpaid invoices.

### Pipeline Position

```
Phase 0                         Phase 1                           Phase 2 (this)              Phase 4
Ingest API → Kafka ────────→ Analytics Worker → ClickHouse
                              │                                    │
                              └────── usage-events ──────────→ Billing Worker → Redis counters
                                                                   │                    │
                                                                   ▼                    ▼
                                                              Enforcement API    Invoice Engine
                                                              (gateway calls     (periodic: monthly/
                                                               /entitlements/     quarterly/yearly)
                                                               check)
                                                                   │
                                                                   ▼
                                                              Postgres (meters,
                                                              rate cards, credits,
                                                              invoices, dunning)
```

### Two Operational Modes in One Service

| Mode | Trigger | What it does |
|---|---|---|
| **Hot Path** | Every Kafka event | Increment Redis counters, check against limits, push balance updates via Pub/Sub |
| **Cold Path** | Cron (end of billing period) | Generate invoices from ClickHouse usage data, apply credits, trigger dunning |

---

## Acceptance Criteria

### Hot Path — Real-Time Counters

| # | Criterion |
|---|---|
| 1 | Consumer group `billing-v1` consumes from `usage-events` (same topic as analytics worker) |
| 2 | On each event: `INCRBYFLOAT usage:{org_id} <total_tokens>`, `INCRBYFLOAT usage:{org_id}:{tenant_id} <total_tokens>`, `INCRBYFLOAT usage:{org_id}:{user_id} <total_tokens>` |
| 3 | Also track: `INCRBYFLOAT spend:{org_id} <cost>`, `INCRBYFLOAT spend:{org_id}:{tenant_id} <cost>` |
| 4 | Redis counters persist indefinitely (no TTL) — reset on billing period boundaries |
| 5 | On each counter update: publish `INCRBYFLOAT` delta to Redis Pub/Sub channel `updates:{org_id}` for WebSocket push |

### Hot Path — Limit Enforcement

| # | Criterion |
|---|---|
| 6 | Expose `GET /v1/entitlements/check?org_id=X&tenant_id=Y&estimated_tokens=Z` endpoint |
| 7 | Read current usage from Redis: `GET usage:{org_id}:{tenant_id}` |
| 8 | Read applicable limits from Postgres: `usage_limits` table (SOFT / HARD, per customer, per meter, per period) |
| 9 | If current + estimated < SOFT limit: return `200 {"allowed": true}` |
| 10 | If current + estimated ≥ SOFT limit but < HARD limit: return `200 {"allowed": true}` with `X-QB-Usage-Warning: approaching limit` header |
| 11 | If current + estimated ≥ HARD limit: return `429 {"allowed": false, "reason": "usage_limit_exceeded"}` |
| 12 | Customer-specific overrides take precedence over plan-level limits |

### Cold Path — Invoice Generation

| # | Criterion |
|---|---|
| 13 | Cron job triggers at billing period boundaries (monthly/quarterly/yearly per subscription) |
| 14 | Query ClickHouse for aggregated usage: `SELECT org_id, tenant_id, user_id, sum(total_tokens), sum(cost) FROM usage_events_dedup_v WHERE timestamp_ms BETWEEN ? AND ? GROUP BY ...` |
| 15 | Read active rate cards from Postgres: `rate_cards` → `rate_card_versions` → `meter_id → price per unit` |
| 16 | Generate one invoice per subscription with status `draft`, invoice number `INV-YYYY-MM-NNN` |
| 17 | Line items: usage charges (meter_reading × rate), plan base fee, overage charges |
| 18 | Auto-apply credits in FEFO order: priority (compensation→promo→prepaid→commit), then expiration date ascending |
| 19 | Calculate tax from `tax_rates` table; total = subtotal − credits + tax |

### Invoice State Machine

| # | Criterion |
|---|---|
| 20 | `draft` → `pending` when finalized and sent to customer |
| 21 | `pending` → `paid` when full payment recorded |
| 22 | `pending` → `overdue` when due date passes without payment |
| 23 | `overdue` → `paid` when payment received after dunning |
| 24 | `overdue` → `voided` when written off or cancelled |
| 25 | Partial payment keeps invoice at current state (stays `pending` or `overdue`) |

### Dunning Workflow

| # | Criterion |
|---|---|
| 26 | Configurable per-org dunning policy with retry schedule |
| 27 | Default schedule: EMAIL (day 3) → SMS (day 7) → SUSPEND service (day 14) → ESCALATE to admin (day 30) |
| 28 | When customer pays mid-dunning: all pending dunning actions cancelled |
| 29 | Dunning actions are logged to `dunning_logs` with timestamp, action, status |
| 30 | SUSPEND: set customer status to `SUSPENDED` — all API keys blocked, gateway returns 429 |

### Credit System

| # | Criterion |
|---|---|
| 31 | Credit types: `compensation` (priority 0), `promotional` (1), `prepaid` (2), `commit` (3) |
| 32 | Credits have: `initial_amount`, `remaining_amount`, `priority`, `expires_at` |
| 33 | FEFO consumption: sort by priority (ascending) then expires_at (ascending) |
| 34 | Every credit consumption writes to `credit_ledger` with invoice_id, amount, transaction_type |
| 35 | Credits can be granted via `POST /v1/credits/grant` — sets initial_amount = remaining_amount |

### Cross-Cutting

| # | Criterion |
|---|---|
| 36 | `GET /health` returns 200; `GET /ready` checks Kafka + Redis + Postgres + ClickHouse |
| 37 | Structured JSON logging: `event_id`, `org_id`, `action` (counter_update / invoice_generated / dunning_triggered), `amount`, `latency_ms` |
| 38 | OpenTelemetry spans: `kafka_consume` → `redis_counter_update` → `limit_check` |
| 39 | Graceful shutdown: drain Kafka consumer, flush Redis pipeline, close Postgres pool |

---

## Test Cases

### TC-01: Real-time counter increment
**Given:** Event with org=acme, tenant=site1, user=bob, total_tokens=1000, cost=0.05
**When:** Billing worker consumes event from Kafka
**Then:** Redis keys `usage:acme`, `usage:acme:site1`, `usage:acme:bob`, `spend:acme`, `spend:acme:site1` all incremented correctly

### TC-02: SOFT limit — warning returned
**Given:** HARD limit=1000 tokens, current usage=800, request estimates 150 tokens
**When:** Gateway calls `GET /v1/entitlements/check`
**Then:** Returns 200 with `X-QB-Usage-Warning` header; request allowed

### TC-03: HARD limit — blocked
**Given:** HARD limit=1000 tokens, current usage=950, request estimates 100 tokens
**When:** Gateway calls `GET /v1/entitlements/check`
**Then:** Returns 429 with `{"allowed": false, "reason": "usage_limit_exceeded"}`

### TC-04: Invoice generation — end of month
**Given:** Customer with subscription billing monthly, 1000 GPT-4 calls totaling 150M tokens at $0.000025/token
**When:** Cron triggers invoice generation for billing period
**Then:** Invoice created: draft state, line items show $3,750 usage charge + plan base fee; credits auto-applied

### TC-05: FEFO credit consumption
**Given:** $100 promotional credit (expires Jan 30), $50 prepaid credit (expires Feb 15); invoice = $80
**When:** Credits auto-applied
**Then:** $80 consumed from promotional (priority 1, expires sooner) first; $0 from prepaid; promotional remaining = $20

### TC-06: Invoice state transitions
**Given:** Invoice in `pending` state, due date today
**When:** Full payment recorded
**Then:** Invoice transitions to `paid`; dunning actions cancelled

### TC-07: Dunning — EMAIL step
**Given:** Invoice overdue 3 days, dunning policy has EMAIL at day 3
**When:** Dunning cron runs
**Then:** Dunning action EMAIL logged; notification dispatched; next action scheduled for day 7 (SMS)

### TC-08: Dunning — SUSPEND step
**Given:** Invoice overdue 14 days
**When:** Dunning cron runs
**Then:** Customer status set to SUSPENDED; all API keys blocked; gateway returns 429 for all requests

### TC-09: Mid-dunning payment
**Given:** Invoice in overdue state, EMAIL action executed, SMS scheduled for day 7
**When:** Customer pays on day 5
**Then:** Invoice transitions to paid; SMS and all future dunning actions cancelled

### TC-10: Credit grant
**Given:** No existing credits for customer
**When:** `POST /v1/credits/grant {type: "promotional", amount: 500, expires_at: "..."}`
**Then:** Credit record created with remaining_amount=500; credit_ledger entry for grant

---

## API Endpoints (Exposed by Billing Worker)

| Method | Path | Auth | Purpose |
|---|---|---|---|
| `GET` | `/v1/entitlements/check` | `X-API-Key` (internal service) | Real-time usage limit enforcement |
| `POST` | `/v1/credits/grant` | `X-API-Key` (admin) | Grant credits to a customer |
| `GET` | `/v1/organizations/:orgId/credits` | `X-API-Key` (admin) | List active credits for an org |
| `POST` | `/v1/invoices/generate` | Internal cron trigger | Generate invoice for billing period (also auto-triggered) |
| `GET` | `/v1/invoices/:id` | `X-API-Key` (admin/customer) | Get invoice details |
| `GET` | `/v1/invoices/:id/pdf` | `X-API-Key` (admin/customer) | Download invoice PDF |
| `POST` | `/v1/payments` | `X-API-Key` (admin) | Record a payment |
| `PATCH` | `/v1/payments/:id/reconciliation` | `X-API-Key` (admin) | Update payment reconciliation status |
| `GET` | `/health` | Public | Liveness probe |
| `GET` | `/ready` | Public | Readiness probe (all deps) |

---

## Data Tables Used

### PostgreSQL (New Tables for Phase 2)

| Table | Key Columns | Purpose |
|---|---|---|
| `meters` | `id`, `org_id`, `name`, `event_type`, `aggregation` (SUM/COUNT/AVG), `field`, `status` (DRAFT/ACTIVE/INACTIVE) | Define what to measure |
| `pricing_models` | `id`, `org_id`, `name`, `model_type` (FLAT/PER_UNIT/TIERED), `config` (JSON) | Define how to charge |
| `rate_cards` | `id`, `org_id`, `name`, `status` | Groups of meter→price mappings |
| `rate_card_versions` | `id`, `rate_card_id`, `version`, `prices` (JSON: meter_id→rate), `effective_at`, `is_current` | Versioned pricing |
| `usage_limits` | `id`, `org_id`, `customer_id`, `meter_id`, `limit_type` (SOFT/HARD), `limit_value`, `period` (PER_MONTH/PER_YEAR/LIFETIME) | Spend/usage caps |
| `customer_contracts` | `id`, `org_id`, `customer_id`, `rate_card_version_id`, `commit_amount`, `status`, `starts_at`, `ends_at`, `auto_renew` | Contract with commit |
| `customer_entitlements` | `id`, `org_id`, `customer_id`, `feature_key`, `status`, `expires_at` | Feature grants |
| `customer_credits` | `id`, `org_id`, `customer_id`, `type` (compensation/promo/prepaid/commit), `initial_amount`, `remaining_amount`, `priority`, `expires_at` | Credit balances |
| `credit_ledger` | `id`, `org_id`, `credit_id`, `invoice_id`, `amount`, `transaction_type`, `created_at` | Credit consumption audit |
| `billing_invoices` | `id`, `org_id`, `customer_id`, `subscription_id`, `invoice_number`, `status`, `period_start`, `period_end`, `subtotal`, `credits_applied`, `tax`, `total`, `due_date`, `issued_at`, `paid_at` | Invoice records |
| `billing_invoice_lines` | `id`, `org_id`, `invoice_id`, `meter_id`, `description`, `quantity`, `unit_price`, `amount`, `line_type` | Invoice line items |
| `billing_payments` | `id`, `org_id`, `invoice_id`, `amount`, `currency`, `payment_method`, `status`, `paid_at`, `reconciliation_status` | Payment records |
| `dunning_policies` | `id`, `org_id`, `name`, `retry_schedule` (JSON: day→action mapping) | Collection policies |
| `dunning_logs` | `id`, `org_id`, `invoice_id`, `action` (EMAIL/SMS/SUSPEND/ESCALATE), `status`, `scheduled_at`, `executed_at` | Dunning audit trail |
| `tax_rates` | `id`, `org_id`, `name`, `rate`, `jurisdiction`, `effective_at`, `expires_at` | Tax configuration |

### Redis (New Keys)

| Key Pattern | Type | Value | Purpose |
|---|---|---|---|
| `usage:{org_id}` | String (float) | Cumulative token count | Org-level usage |
| `usage:{org_id}:{tenant_id}` | String (float) | Cumulative token count | Tenant-level usage |
| `usage:{org_id}:{user_id}` | String (float) | Cumulative token count | User-level usage |
| `spend:{org_id}` | String (float) | Cumulative spend in USD | Org-level spend |
| `spend:{org_id}:{tenant_id}` | String (float) | Cumulative spend in USD | Tenant-level spend |
| `updates:{org_id}` | Pub/Sub channel | Delta message (JSON) | Real-time balance push |

### Existing Resources Reused

| Resource | Operation | From Phase |
|---|---|---|
| Kafka `usage-events` | Consume (group: `billing-v1`) | Phase 0 |
| ClickHouse `usage_events_dedup_v` | SELECT (aggregation queries) | Phase 1 |
| PostgreSQL `organizations`, `tenants`, `users` | SELECT | Phase 0 Story 5 |
| Redis `apikey:{key_value}` | (not used — billing worker uses org_id, not API keys) | Phase 0 Story 2 |

---

## Environment Config Keys

| Key | Description | Default |
|---|---|---|
| `KAFKA_BROKERS` | Kafka bootstrap servers | `localhost:9092` |
| `KAFKA_TOPIC` | Kafka topic to consume | `usage-events` |
| `KAFKA_GROUP_ID` | Consumer group ID | `billing-v1` |
| `REDIS_ADDR` | Redis host:port | `localhost:6379` |
| `DATABASE_URL` | PostgreSQL connection string | (required) |
| `CLICKHOUSE_ADDR` | ClickHouse host:port | `localhost:9000` |
| `CLICKHOUSE_DATABASE` | ClickHouse database | `events` |
| `PORT` | HTTP listen port | `8031` |
| `INVOICE_CRON_SCHEDULE` | Cron expression for invoice generation | `0 0 1 * *` (midnight, 1st of month) |
| `DUNNING_CRON_SCHEDULE` | Cron expression for dunning checks | `0 */6 * * *` (every 6 hours) |
| `LOG_LEVEL` | Log level | `info` |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | OTLP collector | — |
| `OTEL_SERVICE_NAME` | Trace service name | `billing-worker` |
| `SHUTDOWN_TIMEOUT` | Graceful shutdown max wait | `30s` |

---

## Dependencies & Notes for Agent

### How This Phase Connects to Previous Phases

| Previous Phase | What Phase 2 Uses |
|---|---|
| **Phase 0 Story 1** | `UsageEvent` struct — billing worker deserializes same JSON from Kafka |
| **Phase 0 Story 5** | PostgreSQL `organizations`, `tenants`, `users` tables — validated at ingest, queried by billing |
| **Phase 0 Story 7** | Kafka topic `usage-events` — billing worker consumes with separate group `billing-v1` |
| **Phase 1** | ClickHouse `usage_events_dedup_v` — billing worker queries for invoice generation |
| **Phase 5** | LiteLLM gateway calls `GET /v1/entitlements/check` before forwarding requests |

### Key Design Decisions

- **Dual consumer groups:** `analytics-v1` (Phase 1) and `billing-v1` (Phase 2) both consume `usage-events`. Kafka's pub/sub model means both get every message independently — no coordination needed.
- **Redis counters are NOT the source of truth for billing.** They provide real-time enforcement. ClickHouse is the auditable source of truth for invoice generation. A nightly reconciliation job compares Redis against ClickHouse and flags discrepancies.
- **Invoice generation reads from ClickHouse, not Redis.** ClickHouse stores every event with dedup, making it the authoritative source for per-period aggregation. Redis counters are approximate (no event-level storage).
- **FEFO credit consumption is deterministic.** Given the same set of credits and invoice amount, the consumption order is always the same. This makes invoice regeneration idempotent.
- **Dunning is a state machine, not a workflow engine.** The dunning cron reads overdue invoices + dunning policies, determines the next action based on days overdue, and executes it. No external workflow engine needed.
- **Entitlement check is a hot-path endpoint.** It must respond in < 5ms to avoid adding latency to the API gateway. Redis counters ensure this. No Postgres or ClickHouse queries on the hot path.

### Package Layout (Building from Scratch)

```
cmd/billing-worker/
└── main.go                          # Entrypoint: wire Kafka consumer, Redis, Postgres, ClickHouse, HTTP server

internal/
├── counter/
│   └── redis.go                     # Redis counter: INCRBYFLOAT, GET, Pub/Sub publish
├── enforcement/
│   ├── handler.go                   # GET /v1/entitlements/check
│   └── limits.go                    # Limit check logic: SOFT vs HARD, override priority
├── invoice/
│   ├── generator.go                 # Invoice generation: query ClickHouse, apply rate cards, build line items
│   ├── credit_engine.go             # FEFO credit consumption
│   └── tax.go                       # Tax calculation
├── dunning/
│   ├── engine.go                    # Dunning cron: read policies, determine actions, execute
│   └── actions.go                   # EMAIL, SMS, SUSPEND, ESCALATE implementations
├── meter/
│   └── cache.go                     # In-memory meter/rate card cache, periodic refresh from Postgres
├── health/
│   └── handler.go                   # GET /health, GET /ready
└── telemetry/
    ├── tracing.go                   # OpenTelemetry
    └── logging.go                   # slog setup
```

---

## Implementation Stories (Planned)

| Story | Name | Depends On | Summary |
|---|---|---|---|
| **Story 25** | Kafka Consumer & Redis Real-Time Counters | Phase 0 (Kafka topic, UsageEvent model) | Consume usage-events, increment Redis token/spend counters per org/tenant/user, Pub/Sub publish |
| **Story 26** | Entitlement Enforcement API | Story 25 (counters exist) | GET /v1/entitlements/check, SOFT/HARD limit checks, customer override priority, <5ms response |
| **Story 27** | Meter, Rate Card & Pricing Configuration | Phase 0 Story 5 (Postgres tables) | Postgres tables for meters/pricing/rate cards, in-memory cache, periodic refresh |
| **Story 28** | Credit System & FEFO Engine | Postgres credit tables | Credit types, FEFO consumption, credit ledger, grant/consume APIs |
| **Story 29** | Invoice Generation Engine | Stories 27 (rate cards), 28 (credits), Phase 1 (ClickHouse) | Cron-triggered, ClickHouse aggregation, rate card application, line items, tax, invoice state machine |
| **Story 30** | Dunning & Collection Workflow | Story 29 (invoices exist) | Dunning policy, retry schedule, EMAIL/SMS/SUSPEND/ESCALATE, payment recording, reconciliation |
| **Story 31** | Health, Observability & Deployment | Stories 25-30 | Health/readiness endpoints, structured logging, OpenTelemetry, Dockerfile, graceful shutdown |

---

## Phase 2 Completion Checklist

- [ ] Kafka consumer group `billing-v1` reads from `usage-events`
- [ ] `INCRBYFLOAT` counters: `usage:{org}`, `usage:{org}:{tenant}`, `usage:{org}:{user}`, `spend:{org}`, `spend:{org}:{tenant}`
- [ ] `GET /v1/entitlements/check` — SOFT limit returns warning, HARD limit returns 429
- [ ] Customer overrides take precedence over plan-level limits
- [ ] Meters table: DRAFT→ACTIVE on first event, SUM/COUNT/AVG aggregations
- [ ] Rate cards: versioned meter→price mappings, contract-specific overrides
- [ ] Pricing models: FLAT, PER_UNIT, TIERED with JSON config
- [ ] Credit types: compensation(0), promotional(1), prepaid(2), commit(3)
- [ ] FEFO credit consumption: priority asc → expires_at asc
- [ ] Credit ledger records every consumption with invoice_id
- [ ] Invoice generation: cron-triggered, ClickHouse aggregation, rate card application
- [ ] Invoice state machine: draft→pending→paid | overdue→voided
- [ ] Tax calculation from `tax_rates` table
- [ ] Dunning policies: configurable per org, default schedule EMAIL→SMS→SUSPEND→ESCALATE
- [ ] Mid-dunning payment cancels all pending dunning actions
- [ ] Payment recording: full/partial, reconciliation status
- [ ] `GET /health` (200), `GET /ready` (Kafka+Redis+PG+CH checks)
- [ ] Structured JSON logging: `org_id`, `action`, `amount`, `latency_ms`
- [ ] OpenTelemetry spans for counter update and enforcement check
- [ ] Graceful shutdown: drain Kafka, flush Redis, close pools
- [ ] All 10 test cases passing
- [ ] End-to-end: Kafka event → Redis counter increment → enforcement check returns correct status
- [ ] End-to-end: month-end → invoice generated → credits applied (FEFO) → invoice in pending state
