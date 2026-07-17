# Lago vs QBill — Detailed Comparison Analysis

**Date:** 2026-07-16
**Purpose:** Comprehensive comparison of Lago's metering & billing calculation model against QBill's implementation, documenting what QBill does well, where it trails, and actionable gaps.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Comparison](#2-architecture-comparison)
3. [Event Schema & Dedup](#3-event-schema--dedup)
4. [Billable Metrics & Aggregation](#4-billable-metrics--aggregation)
5. [Filters & Dimensions — Lago's Killer Feature](#5-filters--dimensions--lagos-killer-feature)
6. [Charge Models (Pricing Formulas)](#6-charge-models-pricing-formulas)
7. [Subscription Lifecycle](#7-subscription-lifecycle)
8. [Invoice Calculation](#8-invoice-calculation)
9. [AI/LLM Token Billing Comparison](#9-aillm-token-billing-comparison)
10. [Identified Gaps & Action Items](#10-identified-gaps--action-items)
11. [Source Cross-Reference](#11-source-cross-reference)

---

## 1. Executive Summary

| Dimension | Lago | QBill | Advantage |
|---|---|---|---|
| **Pricing models** | 8 (Standard, Package, Graduated, Volume, Percentage, Graduated %, Dynamic, Custom) | 7 (FLAT, PER_UNIT, TIERED_GRADUATED, TIERED_VOLUME, PACKAGE, MATRIX, COST_PLUS) | **Lago** (more granular — Percentage, Dynamic, Custom are unique) |
| **Filter/dimension-based pricing** | ✅ First-class filters on metrics — single metric sliced by model×type×region | ❌ No equivalent — requires separate meters per dimension combination | **Lago — unique advantage** |
| **Money precision** | Up to 15 decimal places internally, rounded at invoice time | ✅ BILLING_MATH.md M-1 to M-6 (9dp, round-half-up, no float) | **QBill** — formal spec |
| **Invoice reproducibility** | Not guaranteed | ✅ Pure function with versioned input snapshots | **QBill** |
| **Aggregation types** | 7 (SUM, COUNT, COUNT UNIQUE, LATEST, MAX, WEIGHTED SUM, CUSTOM/SQL) | 7 (SUM, COUNT, AVG, UNIQUE_COUNT, LATEST, MIN, MAX) | Tie (different sets) |
| **Event dedup scope** | `transaction_id` + `external_subscription_id` | `event_id` + `org_id` (Redis SETNX) | QBill — org-scoped is safer |
| **Progressive billing** | ✅ Built-in (threshold invoicing) | ❌ Not implemented | **Lago** |
| **Minimum commitment** | ✅ Charge-level + plan-level | ❌ Not implemented | **Lago** |
| **Re-rating** | ❌ Not built-in | ✅ Full re-rating with credit notes (CR-1, CR-4) | **QBill** |
| **Dynamic pricing (custom amounts)** | ✅ Via `precise_total_amount_cents` | ❌ Not supported | **Lago** |

---

## 2. Architecture Comparison

### Lago Architecture
```
Your App → Event → Dedup → Match to Subscription → Aggregate (Billable Metric) → Rate (Charge) → Invoice
```

- **Two event store options**: Standard (Postgres) for low volume, ClickHouse for high throughput
- **Concurrent pipelines**: ClickHouse store uses replace semantics; Postgres store rejects duplicates
- **Open-core**: Core features open-source, premium features (plan overrides, progressive billing) under license
- **Owns the full pipeline**: ingestion → dedup → aggregation → rating → invoicing in one system

### QBill Architecture
```
LiteLLM → Go Ingest API → Kafka → ClickHouse (source of truth)
                                         ↓
                            Go Billing Worker → Redis (enforcement cache)
                                         ↓
                            invoice.Generate() ← pure function (versioned inputs)
                                         ↓
                            Postgres billing schema (written by engine, read by NestJS BFF)
```

- **Two-tier with one-writer rule**: Engine writes billing tables, NestJS reads/presents
- **Single event store**: ClickHouse `events.usage_events` (immutable, ReplacingMergeTree)
- **Pure function core**: Byte-for-byte reproducibility with versioned input snapshots
- **Full correction loop**: Re-rating + credit notes for historical corrections

### Key Architectural Differences

| Aspect | Lago | QBill | Impact |
|---|---|---|---|
| **Event store** | Dual (Postgres or ClickHouse) | ClickHouse only | Lago offers flexibility; QBill chooses scale |
| **Invoice engine** | Period-end computation | Pure function with snapshots | QBill enables re-rating, simulation, test clocks |
| **Correction mechanism** | Replace event (ClickHouse) or new event (Postgres) | Credit notes + re-rating | QBill preserves full audit trail |
| **Data ownership** | Lago-owned (SaaS or self-host) | Self-hosted, one-writer rule | QBill clearer ownership boundaries |
| **Filter-based pricing** | ✅ Native (filters on metrics) | ❌ Not supported | **Lago's biggest architectural advantage** |

---

## 3. Event Schema & Dedup

### Lago Event Shape
```json
{
  "transaction_id": "txn_20240314_cust8832_inference_00142",
  "external_subscription_id": "sub_8832",
  "code": "ai_inference",
  "timestamp": 1710421740,
  "properties": {
    "model": "gpt-4",
    "tokens_in": 820,
    "tokens_out": 1500,
    "region": "us-east-1"
  }
}
```

**Key fields:**
- `transaction_id`: Required, dedup key (first-write-wins for Postgres, replace for ClickHouse)
- `external_subscription_id`: Required — ties event to a specific subscription
- `code`: Required — maps to billable metric
- `precise_total_amount_cents`: Optional escape hatch for dynamic/custom pricing

**Dedup:**
- **Postgres store**: `(transaction_id, external_subscription_id)` — duplicate rejected with 422
- **ClickHouse store**: `(transaction_id, timestamp)` — duplicate replaces earlier event
- Cross-delivery dedup: REST + Kafka same ID → only first counts

**Edge cases:**
- Late-arriving events → placed in historical period by `timestamp`
- Events before subscription start or after termination → stored but not billed
- Backfills: validate timestamps carefully

### QBill Event Shape
```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "org_id": "org_abc123",
  "customer_id": "cust_def456",
  "end_user_id": "eu_789ghi",
  "timestamp_ms": 1721152200000,
  "source_mode": "virtual_key",
  "key_id": "key_xyz",
  "event_type": "ai.usage",
  "properties": {
    "model": "gpt-4",
    "total_tokens": 1500,
    "input_tokens": 820,
    "output_tokens": 680,
    "cost": "0.000041"
  }
}
```

**Key fields:**
- `event_id`: Required, client-generated UUID/ULID
- `org_id`: Derived from KeyContext (never from payload — DEC-002 anti-spoofing)
- `customer_id`: From KeyContext; can be empty for BYOK (DEC-007)
- `source_mode`: `direct_ingest`, `virtual_key`, or `byok`

**Dedup:**
- Redis `SETNX idem:{org_id}:{event_id}` with 24h TTL
- First-write-wins — duplicate returns `409 DUPLICATE_EVENT`
- Bloom filter for batch dedup (D-03)

### Comparison

| Aspect | Lago | QBill | Verdict |
|---|---|---|---|
| **Dedup key** | `transaction_id + external_subscription_id` | `event_id + org_id` (Redis) | QBill more secure — org scoping prevents cross-tenant collisions |
| **Dedup behavior** | Postgres: reject 422; ClickHouse: replace | First-write-wins via SETNX (409) | Lago's ClickHouse replace is useful for corrections; QBill's SETNX is safer against double-counting |
| **Subscription binding** | ✅ Required (`external_subscription_id`) | ⚠️ Optional (customer_id from key context) | Lago ensures events always have a billable target; QBill allows unbilled BYOK events (DEC-007) |
| **Escape hatch** | ✅ `precise_total_amount_cents` for custom amounts | ❌ Not supported | **Lago unique** — lets callers skip aggregation entirely |
| **Anti-spoofing** | ❌ Payload `external_subscription_id` trusted | ✅ KeyContext overwrites payload (DEC-002) | **QBill significantly more secure** |
| **Properties type** | Any JSON type (string, int, float, uuid) | String values (coerced to numeric) | Similar |
| **Rate limits** | 500 req/sec events | Not documented | Comparable |

### ✅ QBill Advantages

1. **Anti-spoofing**: DEC-002 ensures KeyContext is the only authority. Lago trusts the payload's `external_subscription_id` — a bug/exploit could misattribute events.
2. **Org-scoped dedup**: QBill's dedup key includes `org_id`, preventing cross-tenant event_id collisions. Lago's `transaction_id` is globally unique by convention only.
3. **BYOK support**: QBill explicitly supports customer-less events (DEC-007). Lago requires `external_subscription_id` on every event.

### ❌ QBill Gaps

1. **No `precise_total_amount_cents` escape hatch**: Lago lets callers bypass aggregation and supply the exact amount. QBill has no equivalent.
2. **No subscription-scoped event matching**: Lago ties events to subscriptions natively. QBill ties events to customers (via key context), with subscription matching happening at invoice time.

---

## 4. Billable Metrics & Aggregation

### Lago Billable Metrics

| Property | Description |
|---|---|
| `code` | Stable identifier referenced by events (immutable post-creation) |
| `aggregation_type` | SUM, COUNT, COUNT UNIQUE, LATEST, MAX, WEIGHTED SUM, CUSTOM/SQL |
| `field_name` | Which `properties.<field>` to aggregate |
| `recurring` | Whether value persists across periods |
| `filters` | Dimensions for slicing (key + values) — **Lago's signature feature** |

### QBill Meters

| Property | Description |
|---|---|
| `name` | Human-readable name |
| `event_type` | Event type to match (not meter_type per ERD conflict resolution) |
| `aggregation` | SUM, COUNT, AVG, UNIQUE_COUNT, LATEST, MIN, MAX |
| `field` | Which property field to aggregate |
| `status` | DRAFT, ACTIVE, INACTIVE |

### Aggregation Type Comparison

| Type | Lago | QBill | Notes |
|---|---|---|---|
| SUM | ✅ `sum_agg` | ✅ | Tie |
| COUNT | ✅ `count_agg` | ✅ | Tie |
| COUNT UNIQUE | ✅ `unique_count_agg` | ✅ `UNIQUE_COUNT` | Tie |
| LATEST | ✅ `latest_agg` | ✅ | Tie |
| MAX | ✅ `max_agg` | ✅ | Tie |
| MIN | ❌ | ✅ | QBill unique |
| AVG | ❌ | ✅ | QBill unique |
| WEIGHTED SUM | ✅ | ❌ | Lago unique — time-proportional |
| CUSTOM/SQL | ✅ | ❌ | Lago unique — arbitrary expressions |
| RECURRING | ✅ (field on metric) | ❌ | Lago unique — persistent metric across periods |

### ✅ QBill Advantages

1. **AVG aggregation**: QBill supports average; Lago does not natively
2. **MIN aggregation**: QBill supports minimum; Lago does not

### ❌ QBill Gaps

1. **No CUSTOM/SQL aggregation**: Lago supports arbitrary SQL expressions (e.g., `properties.tokens_in + properties.tokens_out`). QBill would need separate meters and sum at invoice time.
2. **No WEIGHTED SUM**: Lago supports time-proportional sum for capacity billing. QBill is missing this.
3. **No recurring metrics**: Lago's `recurring: true` lets a metric persist across periods (e.g., seat count). QBill treats all meters as period-resetting.
4. **No filters on meters**: **THIS IS THE BIGGEST LAGO-SPECIFIC GAP** — see next section.

---

## 5. Filters & Dimensions — Lago's Killer Feature

### What Lago Does

Lago's **filters** are the most architecturally significant feature not present in QBill. They let you define **one** billable metric and slice it into independently-priced segments using event properties.

**Example — One metric for all token usage:**
```json
{
  "code": "ai_tokens",
  "aggregation_type": "sum_agg",
  "field_name": "tokens",
  "filters": [
    { "key": "model", "values": ["gpt-4", "gpt-3.5", "claude"] },
    { "key": "type",  "values": ["input", "output"] }
  ]
}
```

Then on a plan, you create **one charge** with per-filter-combination pricing:
```json
"filters": [
  { "values": { "model": ["gpt-4"], "type": ["input"] },  "properties": { "amount": "0.03", "package_size": 1000000 } },
  { "values": { "model": ["gpt-4"], "type": ["output"] }, "properties": { "amount": "0.06", "package_size": 1000000 } },
]
```

**Key benefits:**
1. **No meter explosion**: Adding a new model doesn't require a new meter — just add a filter value
2. **Future-proof**: Send model/type in properties today, price them months later
3. **Decoupled metering from rating**: Metering team doesn't need to know pricing team's plans
4. **Retroactive**: New filter values apply to not-yet-invoiced events

### How QBill Handles This

QBill requires **separate meters** for each dimension combination:
- Meter `gpt4_input_tokens` (event_type = `ai.usage`, field = `input_tokens`)
- Meter `gpt4_output_tokens` (event_type = `ai.usage`, field = `output_tokens`)
- Meter `claude_input_tokens` ...
- ... each new model or token type = new meter

The **MATRIX pricing model** solves the pricing side (per-model × per-token-type rate lookup), but the metering side still requires N meters for N dimension combinations.

### Comparison

| Aspect | Lago Filters | QBill Meters | Impact |
|---|---|---|---|
| **New model =** | Add filter value to existing metric | Create new meter | QBill: more operational overhead |
| **New pricing dimension** | Add filter key to existing metric | Create new meter(s) | QBill: potentially breaking change |
| **Meter count for 10 models × 3 token types** | 1 metric, 30 filter combinations | 30 separate meters | QBill: 30x more meter management |
| **Retroactive pricing** | ✅ Yes (new values apply to past events) | ⚠️ Requires new meter — past events may not reprocess | Lago wins |
| **Event schema coupling** | Loose — send extra properties, price later | Tight — must know all dimensions at meter creation | Lago wins |
| **Pricing resolution** | Per-filter-combination on a single charge | Per-meter via rating waterfall | QBill has waterfall flexibility |

### ✅ QBill Mitigating Factors

1. **MATRIX pricing model**: QBill's MATRIX model handles dimension-based pricing at the rate level, which partially covers Lago's filter use case. The gap is in **metering** (aggregation), not pricing.
2. **Rating waterfall**: QBill's 4-step waterfall (contract → rate card → plan → unrated) is more flexible than Lago's single-path pricing, for enterprise negotiated rates.

### ❌ QBill Gap

**G-LG-01: No meter-level filter/dimension system** — **HIGH PRIORITY GAP**

This is Lago's most significant architectural advantage over QBill. Without filters:
- Each model × token_type combination needs a separate meter
- Adding a new model requires creating a new meter (operational overhead)
- Cannot retroactively price new dimensions against historical events
- Tight coupling between event schema and meter definitions

**Potential solution**: Add a `groupBy` field to QBill meters (similar to OpenMeter's approach), where `groupBy` dimensions are used at invoice time to slice usage into independently-priced buckets. The engine's `rateSegmentUsage()` already groups by `(model, token_type)` — the gap is that this grouping is hardcoded rather than configurable on the meter.

---

## 6. Charge Models (Pricing Formulas)

### Lago's 8 Charge Models

| Model | Formula | QBill Equivalent |
|---|---|---|
| **Standard** | `quantity × unit_price` | PER_UNIT |
| **Package** | `ceil(usage / package_size) × amount` | PACKAGE |
| **Graduated** | `Σ(tier_units × tier_rate)` per bracket | TIERED_GRADUATED |
| **Volume** | `total × rate(tier_containing_total)` | TIERED_VOLUME |
| **Percentage** | `amount × rate + fixed_fee` | ❌ **Not supported** |
| **Graduated Percentage** | Tiered take-rate on monetary amount | ❌ **Not supported** |
| **Dynamic** | Custom amount via `precise_total_amount_cents` | ❌ **Not supported** |
| **Custom** | SQL-like expression for fee computation | ❌ **Not supported** |

### QBill's 7 Pricing Models

| Model | Formula | Lago Equivalent |
|---|---|---|
| FLAT | Fixed flat amount | ✅ Covers Lago's subscription base fee |
| PER_UNIT | `quantity × unit_price` | Standard |
| TIERED_GRADUATED | `Σ(tier_units × tier_rate)` | Graduated |
| TIERED_VOLUME | `total × rate(tier_containing_total)` | Volume |
| PACKAGE | `ceil(qty / packageSize) × packagePrice` | Package |
| MATRIX | Rate lookup by `(model, tokenType)` | ⚠️ Partially covered by filters on Standard/Package charges |
| COST_PLUS | `providerCost × (1 + markupPct)` | ❌ Not in Lago |

### Shared Models (Direct Equivalents)

| Model | Lago Example | QBill Example | Both Calculate As |
|---|---|---|---|
| **Standard/Per-Unit** | $0.05 × 1,000 = $50 | $0.05 × 1,000 = $50 | Same |
| **Package** | ceil(201/100) × $5 = $10 | ceil(201/100) × $5 = $10 | Same |
| **Graduated/Tiered** | (100×$1)+(100×$0.50)+(50×$0.10) = $155 | Same | Same |
| **Volume** | 65,000 × $0.0006 = $39 | 65,000 × $0.0006 = $39 | Same |

### Models Unique to Lago

| Model | Description | Use Case | Severity of Gap |
|---|---|---|---|
| **Percentage** | `amount × rate + fixed_fee_per_event` | Payment processing fees, marketplace take-rates | **Medium** — can be simulated with PER_UNIT on a monetary-value meter |
| **Graduated %** | Tiered percentage (marketplace GMV) | Revenue share with tiered rates | **Low** — niche use case |
| **Dynamic** | Custom amount per event via `precise_total_amount_cents` | Fully custom negotiated pricing | **Medium** — flexibility escape hatch |
| **Custom** | SQL-like expression | Complex formulas | **Low** — edge case |

### Models Unique to QBill

| Model | Description | Use Case |
|---|---|---|
| **MATRIX** | Rate lookup by `(model, tokenType)` | LLM-native per-model × per-token-type pricing |
| **COST_PLUS** | `providerCost × (1 + markupPct)` | BYOK/gateway markup pricing |

### ✅ QBill Advantages

1. **MATRIX pricing**: Native dimension-based pricing for LLM use cases. Lago achieves this via filters + charge model, which is more flexible but requires more configuration.
2. **COST_PLUS**: Native markup pricing for BYOK/gateway. Lago has no equivalent.
3. **Min/max clamping**: QBill's pricing models support `MinimumAmount`/`MaximumAmount` per model. Lago has minimum commitment at plan level only.

### ❌ QBill Gaps

1. **No Percentage charge model**: Cannot handle payment-processing-style fees (1.2% + $0.10/event)
2. **No Dynamic pricing escape hatch**: Cannot accept pre-computed custom amounts per event
3. **No Custom/SQL expression pricing**: Cannot express complex fee formulas

---

## 7. Subscription Lifecycle

### Lago Subscriptions

| Aspect | Lago Behavior |
|---|---|
| **Billing cycle** | Calendar (default, prorated first period) or anniversary |
| **Multiple subscriptions** | ✅ Customer can hold multiple simultaneous subscriptions |
| **Plan overrides** | ✅ Premium: override price, trial, taxes, charge properties |
| **Auto-renew** | ✅ No `ending_at` → indefinite renewal |
| **End date** | With `ending_at` → terminates, no renewal |
| **Plan changes** | Terminate old subscription, create new one |
| **Invoice consolidation** | ✅ Across overlapping subscription cycles |

### QBill Subscriptions (from ERD and catalog.service.ts)

| Aspect | QBill Behavior |
|---|---|
| **Billing cycle** | Anniversary only (T-3: start-date anchored with end-of-month clamping) |
| **Multiple subscriptions** | ✅ Customer can hold multiple subscriptions |
| **Plan overrides** | ✅ Via contract_rates (negotiated per-meter rates) |
| **Auto-renew** | ✅ `auto_renew` on contracts |
| **End date** | Contract end_date or subscription end_date |
| **Plan changes** | Versioned (plan_versions), prorated sub-windows (P-1 through P-5) |
| **Invoice consolidation** | ✅ CR-8 billing groups (planned, D-17 not dispatched yet) |

### Comparison

| Aspect | Lago | QBill | Verdict |
|---|---|---|---|
| **Calendar billing** | ✅ Default option | ❌ Anniversary only | **Lago** — offers flexibility |
| **Plan overrides** | ✅ Premium feature | ✅ Via contract_rates | Tie |
| **Invoice consolidation** | ✅ Automatic for overlapping subscriptions | ✅ Billing groups (CR-8, not yet built) | Lago wins (ready now vs planned) |
| **Proration** | Day-based, first period only | Day-based, all plan changes (P-1 through P-5) | **QBill** — more comprehensive |
| **Trial support** | ✅ Days-based, applies to base fee only | ✅ Trial_days on plans, TR-1 rules | Tie |
| **Cancel behavior** | Immediate or end-of-period | Immediate or end-of-period (P-5) | Tie |
| **Webhook lifecycle** | ✅ Multiple webhook events | ✅ webhooks module | Tie |

### ✅ QBill Advantages

1. **Comprehensive proration**: QBill prorates base fee (P-2), included allowance (P-3), and seats (P-4) for ALL plan changes, not just the first period
2. **Versioned plan changes**: QBill tracks plan versions and rates each sub-window against the correct version
3. **Anniversary anchoring with end-of-month clamping**: T-3 handles Feb 28, leap years, and months with fewer days — enterprise-grade

### ❌ QBill Gaps

1. **No calendar billing option**: Lago offers both calendar (1st to end-of-month) and anniversary billing. QBill only supports anniversary billing.
2. **Billing groups not yet built**: CR-8 is in D-17 (not dispatched). Lago already supports invoice consolidation.

---

## 8. Invoice Calculation

### Lago Invoice Pipeline

Lago's invoicing is simpler than FlexPrice/Orb — it doesn't have an explicit multi-step pipeline documented. Key aspects:
- **Invoices per subscription per billing period** (or consolidated across cycles)
- **Fees** = one line item per charge/metric slice (the audit trail back to raw events)
- **Draft → Finalized** lifecycle, with grace period
- **Auto-payment** on finalization via configured payment gateway
- **Minimum commitment**: floor amount across all charges in a period
- **Progressive billing**: mid-cycle threshold invoicing

### QBill Invoice Pipeline (BILLING_MATH.md M-5)

```
Step 1: Subtotal = Σ(line items) — BASE_FEE + USAGE + OVERAGE + COMMIT_TRUE_UP + SEAT + ADJUSTMENT
Step 2: Credits_applied = min(subtotal, available FEFO credits in priority order)
Step 3: Taxable = subtotal − credits_applied
Step 4: Tax = round_half_up(taxable × tax_rate, minor_units)
Step 5: Total = taxable + tax
```

Additional QBill invoice features:
- **Line item types**: BASE_FEE, USAGE, OVERAGE, COMMIT_TRUE_UP, SEAT, ADJUSTMENT
- **FEFO credit ordering**: priority → expires_at → created_at → ID
- **Rate source tracking**: Each line records `rate_source` and `rate_source_id`
- **Input snapshots**: Invoice stores `plan_version_id`, `rate_card_version_id`, `aggregation_watermark`
- **Grace window**: Draft → finalize after configurable `INVOICE_GRACE_HOURS` (default 36h)
- **State machine**: draft → pending → paid/overdue/voided

### Comparison

| Aspect | Lago | QBill | Verdict |
|---|---|---|---|
| **Line item types** | Per-charge (untyped) | ✅ Typed (6 types) | **QBill** — richer semantics |
| **Credit application** | Not explicit | ✅ FEFO (4 priority levels) | **QBill** |
| **Minimum commitment** | ✅ Plan-level floor | ❌ Not implemented | **Lago** |
| **Progressive billing** | ✅ Mid-cycle threshold invoicing | ❌ Not implemented | **Lago** |
| **Grace window** | ✅ Draft → Finalized | ✅ T-5: configurable grace window | Tie |
| **Input snapshots** | ❌ Not stored | ✅ versioned input refs on invoice | **QBill** — enables audit/re-rating |
| **Rate source tracking** | ❌ Per-charge (single source) | ✅ Per-line-item rate provenance | **QBill** |
| **Invoice state machine** | draft → finalized | draft → pending → paid/overdue/voided | QBill more granular |
| **Auto-collection** | ✅ On finalization | ✅ CR-6: on finalization | Tie |
| **Webhook events** | ✅ `invoice.created` | ✅ webhook module | Tie |

### ✅ QBill Advantages

1. **Input snapshots**: QBill stores the versioned inputs used to generate each invoice — enabling re-rating (CR-1), simulation (CR-9), and full audit
2. **Rate source tracking**: Every line item records where its rate came from (contract, rate card, or plan)
3. **Typed line items**: 6 distinct line types with different behavior (e.g., COMMIT_TRUE_UP only on final contract invoice)
4. **FEFO credit ordering**: Deterministic credit consumption with 4 priority levels

### ❌ QBill Gaps

1. **No minimum commitment**: Lago supports charge-level and plan-level spending minimums. QBill has no `max(subtotal, minimum)` step.
2. **No progressive billing**: Lago invoices mid-cycle when usage crosses a threshold. QBill only invoices at period boundaries.
3. **No coupon/discount pipeline**: (Same gap identified in FlexPrice comparison) — Lago doesn't explicitly document a coupon pipeline either, but its plan-level minimum commitment covers part of this.

---

## 9. AI/LLM Token Billing Comparison

### Lago's OpenAI-Style Template (§7)

Lago provides a **concrete worked example** for per-model, per-token-type pricing:

**One metric** (`ai_tokens`, `SUM` on `tokens`) with **filters** for `model` and `type`:
- Single event shape covers all models and token types
- New model? Add a filter value — no new metric needed

**One charge** with per-filter-combination `package` pricing:
- Each `(model, type)` combination has its own $/1M-tokens rate
- Auto-provisions 4 line items for 2 models × 2 types

### QBill's Approach (Current)

QBill requires:
- **Separate meters** per model × token_type combination
- **MATRIX pricing** to handle per-model × per-token-type rates
- **Rating waterfall** to resolve contract/rate card/plan precedence

### Comparison

| Aspect | Lago | QBill | Verdict |
|---|---|---|---|
| **Meter count for 10 models × 3 token types** | 1 metric, 30 filter combos | 30 separate meters | **Lago** — 30x fewer meters to manage |
| **New model onboarding** | Add filter value | Create new meter, new pricing model | **Lago** — operational simplicity |
| **Per-model rate differentiation** | Per-filter pricing on single charge | MATRIX pricing model | Tie (different mechanisms) |
| **Per-token-type rate differentiation** | Per-filter on same metric | Separate meters per type | **Lago** — fewer meters |
| **Package pricing for tokens** | ✅ `package_size` per filter combo | ✅ PACKAGE model | Tie |
| **Free tier / included allowance** | ✅ Via charge properties | ✅ P-3 prorated included units | Tie |
| **Monitor current usage** | ✅ `GET /customers/{id}/current_usage` | ✅ analytics proxy (Go → ClickHouse) | Tie |

### ✅ QBill Advantages

1. **Rating waterfall**: Contract-specific rates override catalog → enterprise-grade negotiated pricing
2. **COST_PLUS**: Native BYOK markup (Lago would need custom logic)
3. **Invoice reproducibility**: Byte-for-byte audit trail for token billing

### ❌ QBill Gaps

1. **No filter/dimension system**: **HIGHEST PRIORITY LAGO-SPECIFIC GAP**. Without filters, each model×type combination requires a separate meter.
2. **No OpenAI-style template**: Lago provides a copy-paste template for per-model token pricing. QBill has no equivalent quick-start.
3. **No current-usage API documented**: Lago has a dedicated `current_usage` endpoint. QBill's analytics proxy covers this but isn't documented as a billing resource.

---

## 10. Identified Gaps & Action Items

### High-Priority Gaps

| # | Gap | Description | Lago Feature | Impact | Suggested Fix |
|---|---|---|---|---|---|
| **G-LG-01** | **No filter/dimension system on meters** | Cannot slice one meter by model×type×region for independent pricing. Requires N meters for N dimension combinations. | Filters on billable metrics | **Operational overhead**: 10 models × 3 token types = 30 meters instead of 1. Slower to add new models. | Add `groupBy` field to meter definitions (similar to OpenMeter). Engine `rateSegmentUsage()` already groups by `(model, token_type)` — make this configurable on the meter. |
| **G-LG-02** | **No WEIGHTED SUM aggregation** | Cannot do time-proportional capacity billing | WEIGHTED SUM | Missing billing use case (provisioned capacity, reservations) | Add WEIGHTED SUM to engine aggregation functions |
| **G-LG-03** | **No calendar billing option** | Only anniversary billing supported | Calendar billing (default) | Customer preference; some orgs want all customers on the same billing cycle | Add calendar-month billing as an alternative to anniversary billing |
| **G-LG-04** | **No minimum commitment** | Cannot set per-charge or per-plan spending floor | Charge-level + plan-level minimum commitment | Revenue risk: no guaranteed minimum from high-usage customers | Add `minimum_amount` to pricing models (engine already has MinimumAmount/MaximumAmount in resolver) and invoice-level minimum commitment |

### Medium-Priority Gaps

| # | Gap | Lago Feature | Suggested Fix |
|---|---|---|---|
| **G-LG-05** | **No Percentage charge model** | Percentage (1.2% + $0.10) | Add Percentage pricing model for payment/ marketplace use cases |
| **G-LG-06** | **No Dynamic pricing escape hatch** | `precise_total_amount_cents` | Allow events to carry a pre-computed amount that bypasses aggregation |
| **G-LG-07** | **No progressive billing** | Mid-cycle threshold invoicing | Add threshold-based mid-cycle invoice generation |
| **G-LG-08** | **No CUSTOM/SQL aggregation** | Custom SQL expressions | Add support for aggregation expressions (tokens_in + tokens_out) |

### Low-Priority Gaps

| # | Gap | Lago Feature | Suggested Fix |
|---|---|---|---|
| **G-LG-09** | **No recurring metrics** | `recurring: true` on billable metrics | Add persistent-metric support for seat counts, storage snapshots |
| **G-LG-10** | **No per-model token billing template** | OpenAI-style copy-paste template | Create QBill quick-start template for per-model token pricing |

---

## 11. Source Cross-Reference

| QBill Source | Content Referenced |
|---|---|
| `BILLING_MATH.md` M-1 to M-6 | Money precision, rounding, invoice arithmetic |
| `BILLING_MATH.md` P-1 to P-5 | Proration (sub-windows, allowance, seats) |
| `BILLING_MATH.md` T-1 to T-6 | Time rules (UTC, anniversary, grace, half-open periods) |
| `BILLING_MATH.md` §7 | FEFO credit application |
| `ADR-001` §3 | Hybrid billing, rate resolution waterfall |
| `ADR-001` §3.3 | Rating waterfall (contract → rate card → plan → unrated) |
| `ADR-001` §3.4 | Invoice purity invariant |
| `phase_2_billing_worker.md` | Invoice lifecycle, acceptance criteria |
| `ERD.md` §3 | Catalog — meters, pricing models, rate cards |
| `ERD.md` §4 | Billing entities — invoices, credits, wallet |
| `SCAFFOLD.md` §6 | Conventions (money on wire, testing) |
| `DECISIONS.md` DEC-002 | Anti-spoofing (KeyContext authority) |
| `DECISIONS.md` DEC-007 | BYOK customer-less events |
| `engine/internal/invoice/generate.go` | Pure invoice function |
| `engine/internal/rating/resolver.go` | Rating waterfall with all 7 pricing models |
| `engine/internal/invoice/rateSegmentUsage.go` | Usage rating per sub-window |
| `engine/internal/invoice/credits.go` | FEFO credit sorting and application |

---

*End of document. Generated 2026-07-16.*
