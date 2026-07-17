# Meteroid vs QBill — Detailed Comparison Analysis

**Date:** 2026-07-16
**Purpose:** Comprehensive comparison of Meteroid's billing engine against QBill's implementation, documenting what QBill does well, where it trails, and actionable gaps. Meteroid is written in Rust with a Kafka + ClickHouse pipeline.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Comparison](#2-architecture-comparison)
3. [Event Ingestion & Dedup](#3-event-ingestion--dedup)
4. [Billable Metrics, Segmentation & Grouping](#4-billable-metrics-segmentation--grouping)
5. [Pricing Models](#5-pricing-models)
6. [Entitlements & Quota Enforcement](#6-entitlements--quota-enforcement)
7. [Subscription Lifecycle & Proration](#7-subscription-lifecycle--proration)
8. [Invoice Calculation](#8-invoice-calculation)
9. [AI/LLM Token Billing](#9-aillm-token-billing)
10. [Identified Gaps & Action Items](#10-identified-gaps--action-items)
11. [Source Cross-Reference](#11-source-cross-reference)

---

## 1. Executive Summary

| Dimension | Meteroid | QBill | Advantage |
|---|---|---|---|
| **Pricing models** | 6 (Subscription, Slot, Capacity, Per-Unit, Tiered, Volume, Package, Matrix, Floor price) | 7 (FLAT, PER_UNIT, TIERED_GRADUATED, TIERED_VOLUME, PACKAGE, MATRIX, COST_PLUS) | **QBill** — COST_PLUS unique; Meteroid has Slot and Floor price |
| **Segmentation** | ✅ First-class (none/single/2D independent/2D dependent) | ✅ MATRIX pricing model for dimension-based rates | Tie (different mechanisms) |
| **Grouping key** | ✅ First-class grouping on metrics (with tiering implications) | ⚠️ Not supported — grouping would need separate meters | **Meteroid** |
| **Entitlements** | ✅ Separate concept from billing (Boolean, Metered with reset strategies) | ⚠️ `customer.usage_limits` + `customer.limit_overrides` — conflated with billing | **Meteroid** — cleaner separation |
| **Invoice preview** | ✅ "Upcoming Invoice" real-time recalculation | ❌ Not documented | **Meteroid** |
| **Floor price (minimum)** | ✅ At plan or product level (pre-coupon) | ❌ Not implemented | **Meteroid** |
| **Consolidated invoicing** | ✅ Optional across subscriptions | ✅ Billing groups (CR-8, D-17 not dispatched) | Tie (both planned) |
| **Currency conversion** | ✅ Daily FX rate at finalization | ❌ No FX (X-2: display-only conversion) | **Meteroid** |
| **Re-rating** | ❌ Not built-in | ✅ Full re-rating with credit notes (CR-1, CR-4) | **QBill** |
| **Wallet** | ❌ Not mentioned | ✅ Full wallet with Redis hot-path (CR-2) | **QBill** |

---

## 2. Architecture Comparison

### Meteroid Architecture
```
Tenant
 └─ Billing Entity (legal entity, currency, tax)
     └─ Product Family / Product Line
         └─ Product (sellable thing + entitlements)
             └─ Plan (bundles Products + pricing + trial)
                 └─ Plan Version (draft → active → superseded, immutable once published)
                     └─ Subscription (Customer × Plan Version)
                         └─ Invoice (one per subscription per billing period)
```

**Key characteristics:**
- **Rust-based** with Kafka + ClickHouse pipeline
- **Plan Versioning**: Subscriptions are frozen copies of Plan Versions at creation (plan changes never retroactively affect existing subscriptions)
- **Layered entity hierarchy**: Tenant → Billing Entity → Product Family → Product → Plan → Plan Version → Subscription → Invoice
- **NOT a payment processor** — delegates to PSP (Stripe, etc.)
- **Grouping key**: First-class concept on billable metrics — usage bucketed per group, shown as separate invoice line items

### QBill Architecture
```
LiteLLM → Go Ingest API → Kafka → ClickHouse (source of truth)
                                         ↓
                            Go Billing Worker → Redis (enforcement cache)
                                         ↓
                            invoice.Generate() ← pure function (versioned inputs)
                                         ↓
                            Postgres billing schema (NestJS BFF reads/presents)
```

**Key characteristics:**
- **Go-based** with Kafka + ClickHouse pipeline
- **Pure function core**: Byte-for-byte reproducibility with versioned input snapshots
- **One-writer rule**: Engine writes billing tables, NestJS reads/presents
- **Dual pricing paths**: Product-led (plans → charges → pricing models) + Sales-led (contracts → rate cards → contract rates)
- **Full correction loop**: Re-rating + credit notes

### Key Architectural Differences

| Aspect | Meteroid | QBill | Impact |
|---|---|---|---|
| **Language** | Rust | Go | Different ecosystems; Go is more accessible for the team |
| **Entity hierarchy** | 6+ layers (Tenant → Billing Entity → Product Family → Product → Plan → Version → Subscription) | 4 layers (Org → Customer → End User → Subscription) | Meteroid's additional layers (Billing Entity, Product Family) support multi-entity enterprises |
| **Pipeline** | Rust + Kafka + ClickHouse | Go + Kafka + ClickHouse | Architecturally similar |
| **Invoice engine** | Period-end computation from event store | Pure function with versioned snapshots | QBill's approach enables re-rating, simulation, test clocks |
| **Correction mechanism** | Credit Notes (for finalized invoices) | Credit Notes + Re-rating (CR-1, CR-4) | QBill has a full correction loop |
| **Wallet** | ❌ Not built-in | ✅ Full wallet with real-time burndown | **QBill** |
| **Revenue recognition** | ❌ Not mentioned | ✅ ASC 606/IFRS 15 (CR-5) | **QBill** |

---

## 3. Event Ingestion & Dedup

### Meteroid

**Event Shape:**
```json
{
  "events": [{
    "event_id": "01J...ULID",
    "code": "llm_tokens",
    "customer_id": "cus_123",
    "timestamp": "2026-07-16T10:30:00Z",
    "properties": {
      "model": "claude-sonnet-5",
      "token_type": "input",
      "tokens": "184320",
      "project_id": "proj_9"
    }
  }],
  "allow_partial_failures": false,
  "allow_backfilling": false
}
```

**Dedup:**
- Key: `(event_id, customer_id)` — re-sending same pair is a no-op for counting
- If timestamps differ across duplicates, the **latest** timestamp wins
- Timestamp window: 24h past → 1h future (configurable via `allow_backfilling`)
- Batch: up to 100 events/request
- By default one bad event fails the whole batch (`allow_partial_failures` for partial success)
- `properties` values are all strings — cast on ingestion

### QBill

**Event Shape:**
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
    "cost": "0.000041"
  }
}
```

**Dedup:**
- Key: `event_id + org_id` via Redis SETNX with 24h TTL
- First-write-wins — duplicate returns `409 DUPLICATE_EVENT`
- Bloom filter for batch dedup
- Batch: up to 50,000 events/request with partial accept on per-index errors
- Anti-spoofing: KeyContext overwrites payload values (DEC-002)

### Comparison

| Aspect | Meteroid | QBill | Verdict |
|---|---|---|---|
| **Dedup key** | `(event_id, customer_id)` | `(event_id, org_id)` via Redis SETNX | QBill's org-scoped dedup prevents cross-tenant collisions |
| **Duplicate behavior** | Latest timestamp wins (silent overwrite) | First-write-wins (explicit 409) | **QBill** — no silent data loss |
| **Batch size** | 100 events/request | 50,000 events/request | **QBill** — 500x larger batches |
| **Partial accepts** | ✅ Via `allow_partial_failures` | ✅ Per-index errors | Tie |
| **Backfill support** | ✅ `allow_backfilling: true` | ✅ Via re-rating (CR-1) | Different approaches |
| **Anti-spoofing** | ❌ Payload `customer_id` trusted | ✅ KeyContext overwrites payload (DEC-002) | **QBill** |
| **Timestamp window** | 24h past → 1h future (default) | Via `timestamp_ms` (no explicit window) | Meteroid has explicit guardrails |

### ✅ QBill Advantages

1. **Anti-spoofing**: DEC-002 ensures payload values never override authenticated context
2. **SETNX first-write-wins**: Safer than Meteroid's latest-timestamp-wins for billing accuracy
3. **Batch size**: 50,000 vs 100 — 500x larger for high-volume scenarios

### ❌ QBill Gaps

1. **No explicit timestamp window**: Meteroid enforces 24h past / 1h future guardrails. QBill has no documented timestamp validation window.
2. **No backfill flag**: Meteroid's `allow_backfilling` explicitly controls historical data ingestion. QBill's re-rating handles this at invoice time, not at ingest time.

---

## 4. Billable Metrics, Segmentation & Grouping

### Meteroid's Segmentation (Unique Feature)

Meteroid has a **formal segmentation system** on billable metrics — this is distinct from both Lago's filters and QBill's MATRIX pricing.

| Type | Behavior | Example |
|---|---|---|
| **None** | Single price for whole metric | All tokens at one rate |
| **Single dimension** | Price varies by one property | rate differs by `model` |
| **2D independent** | Every combination of 2 dimensions needs its own price | Every `model × region` |
| **2D dependent** | Restrict to valid combinations | `model → [allowed regions]` (avoids combinatorial blow-up) |

### Meteroid's Grouping Key (Critical Design Decision)

Meteroid supports a **grouping key** on metrics (e.g., `project_id`, `tenant_id`, `cluster_id`):

- Usage is bucketed and shown as separate invoice line items per group
- **⚠️ Groups are evaluated independently for tiered pricing**
- Example: grouped by `project_id` with tiered pricing → each project resets to tier 1, NOT the customer's total across projects
- This is explicitly documented as a **design decision, not a bug**

### QBill's Approach

QBill handles dimension-based pricing through:
- **MATRIX pricing model**: Rate lookup by `(model, tokenType)` dimensions
- **Rating waterfall**: Contract_rate → rate_card → pricing_model
- **No formal grouping key**: Each meter is independent; there's no mechanism to group related meters under a shared allowance or tier threshold

### Comparison

| Aspect | Meteroid | QBill | Verdict |
|---|---|---|---|
| **Segmentation** | ✅ Formal system (none/single/2D independent/2D dependent) | ⚠️ MATRIX model handles rate lookup, but no formal segmentation at metric level | **Meteroid** — cleaner separation of concerns |
| **Grouping key** | ✅ First-class on metrics (with documented tiering implications) | ❌ Not supported — would need separate meters per group | **Meteroid** — multi-tenant/project billing use case |
| **Tier behavior with groups** | Explicitly documented: per-group, not per-total | Not applicable (no grouping) | Meteroid is transparent about the design decision |
| **Conversion factor** | ✅ Raw value divided before pricing (bytes → KB) | ⚠️ Handled at pricing model level (PACKAGE size, PER_UNIT rate) | Similar, different mechanisms |
| **Mutability** | Code + aggregation immutable; segmentation values mutable | Meter: event_type + aggregation immutable (Status: DRAFT/ACTIVE/INACTIVE) | Similar |

### ✅ QBill Advantages

1. **MATRIX pricing**: More flexible than Meteroid's 2D segmentation — supports arbitrary dimensions beyond 2
2. **Rating waterfall**: Contract-specific rates override catalog rates — enterprise-grade

### ❌ QBill Gaps

**G-MT-01: No grouping key support** — **HIGH PRIORITY**

Meteroid's grouping key is critical for multi-tenant/per-project billing. Without it:
- Cannot show per-project costs as separate invoice line items
- Tiered/volume discounts can't be evaluated per-group vs per-total
- Each project/tenant requires a separate subscription

**G-MT-02: No explicit segmentation on metrics** — **MEDIUM PRIORITY**

Meteroid's segmentation system cleanly separates "how we group usage" (metric level) from "how we price it" (product/plan level). QBill conflates this in the MATRIX pricing model.

**G-MT-03: Grouping-tiering interaction not documented** — **MEDIUM PRIORITY**

Meteroid explicitly documents that grouping breaks tiering (each group starts at tier 1). QBill has no grouping at all, but if/when it's added, this interaction must be explicitly decided and documented.

---

## 5. Pricing Models

### Meteroid's 8 Pricing Models

| Model | Formula | Type | QBill Equivalent |
|---|---|---|---|
| **Subscription rate** | `charge = flat_rate` | Fixed recurring | FLAT |
| **Slot-based** | `charge = rate_per_slot × slots` | Per-seat | ⚠️ SEAT line type, no formal slot pricing |
| **Capacity commitment** | `tier_price + max(0, usage - included) × overage_rate` | Committed + overage | ⚠️ Can be modeled with COMMIT_TRUE_UP + PER_UNIT |
| **Per unit** | `charge = usage × unit_price` | Usage-based | PER_UNIT |
| **Tiered** (graduated) | `Σ(tier_units × tier_rate)` | Usage-based | TIERED_GRADUATED |
| **Volume** | `total × rate(tier_containing_total)` | Usage-based | TIERED_VOLUME |
| **Package** | `ceil(usage / block_size) × price_per_block` | Usage-based | PACKAGE |
| **Matrix** | `Σ(dimension_usage × dimension_rate)` | Dimension-based | MATRIX |
| **Floor price** (minimum) | `max(calculated, floor)` | Add-on | ❌ Not implemented |
| **One-time charge** | `quantity × unit_price` | One-off | ❌ Not a formal model |
| **Recurring charge** | Independent cadence from plan | Recurring add-on | ⚠️ Via separate subscription |

### Shared Models (Direct Equivalents)

| Model | Meteroid | QBill | Match |
|---|---|---|---|
| Per unit | ✅ `usage × unit_price` | ✅ PER_UNIT | ✅ Exact |
| Tiered (graduated) | ✅ `Σ(tier_units × tier_rate)` | ✅ TIERED_GRADUATED | ✅ Exact |
| Volume | ✅ `total × rate(tier)` | ✅ TIERED_VOLUME | ✅ Exact |
| Package | ✅ `ceil(usage / size) × price` | ✅ PACKAGE | ✅ Exact |
| Matrix | ✅ `Σ(dim_usage × dim_rate)` | ✅ MATRIX | ✅ Exact |
| Flat fee | ✅ `flat_rate` | ✅ FLAT | ✅ Exact |

### Models Unique to Meteroid

| Model | Description | Use Case | Severity |
|---|---|---|---|
| **Slot-based** | `rate_per_slot × slots` with mid-period proration | Seat/licenses — Meteroid has dedicated slot pricing with add/remove proration rules | **Medium** — QBill handles seats via SEAT line type but without formal slot pricing model |
| **Capacity commitment** | `tier_price + overage` | Committed usage with overage — bundled pricing with included units | **Low** — QBill achieves this via overage lines + included allowance (P-3) |
| **Floor price** | `max(calculated, floor)` | Minimum billed amount | **Medium** — No minimum commitment in QBill |
| **One-time charge** | Non-recurring line item | Setup fees, onboarding | **Low** — Can be modeled as ADJUSTMENT line |
| **Recurring charge** | Independent cadence | Support package add-on | **Low** — Can be separate subscription |

### Models Unique to QBill

| Model | Description | Why QBill Has It |
|---|---|---|
| **COST_PLUS** | `providerCost × (1 + markupPct)` | BYOK/gateway markup pricing — unique to QBill's AI proxy use case |

### ✅ QBill Advantages

1. **COST_PLUS**: Native BYOK markup — Meteroid has no equivalent
2. **MATRIX clarity**: QBill's MATRIX is dimension-agnostic; Meteroid's is limited to 2D
3. **Pricing model set**: 7 vs 8 — comparable coverage with different specializations

### ❌ QBill Gaps

1. **No slot-based pricing model**: Meteroid has dedicated seat/slot pricing with specific mid-period add/remove rules
2. **No floor price/minimum commitment**: Meteroid supports minimum at plan and product level
3. **No one-time charge model**: Setup fees require workarounds

---

## 6. Entitlements & Quota Enforcement

### Meteroid's Entitlement System

Meteroid has a **separate, full-featured entitlement system** decoupled from billing:

| Aspect | Meteroid |
|---|---|
| **Purpose** | "how much can this customer *do*" — separate from "how much do they *owe*" |
| **Enforcement** | Meteroid exposes state via API; YOUR app enforces |
| **Boolean** | On/off flag tied to subscription active state |
| **Metered** | Tied to billable metric, with limit + reset period — a quota independent from billing |
| **Reset strategies** | 5 options: Never, Billing Cycle, Calendar, Fixed Window, Sliding Window |
| **Override precedence** | Default (Feature) → Plan → Subscription (most specific wins) |

**Key insight**: You can meter-and-bill tokens via a Billable Metric AND simultaneously cap usage via a Metered Entitlement — they read from the same event stream but serve different purposes.

### QBill's Approach

QBill has:
- `customer.usage_limits` — per-customer, per-meter, per-period limit configuration (SOFT/HARD/WARNING)
- `customer.limit_overrides` — per-customer overrides for negotiated limits
- Redis counters for real-time enforcement (DEC-010: <5ms at c=32)
- Enforcement API: `GET /v1/entitlements/check`

**Issue**: QBill's usage limits are tied to **meters**, which are also used for billing. The same meter definition handles both "what we measure for billing" and "what we cap." This conflates two distinct concerns.

### Comparison

| Aspect | Meteroid | QBill | Verdict |
|---|---|---|---|
| **Separation from billing** | ✅ Full — entitlements are a separate concept, even sharing the same event stream | ❌ Conflated — usage_limits reference meters which are also billing entities | **Meteroid** |
| **Entitlement types** | Boolean, Metered | SOFT/HARD/WARNING limits | Different taxonomy |
| **Reset strategies** | 5 options (Never, Billing Cycle, Calendar, Fixed Window, Sliding Window) | Per-period (matching subscription cycle) | **Meteroid** — more flexible |
| **Sliding window** | ✅ Supported for rate limiting | ❌ Not supported — only per-period evaluation | **Meteroid** |
| **Override precedence** | ✅ 3-level (Feature → Plan → Subscription) | ✅ 2-level (plan → customer.limit_overrides) | Meteroid more granular |
| **Enforcement latency** | API-based (calls entitlement API) | ✅ **<5ms at 32 concurrent** (DEC-010) | **QBill** — real-time Redis path |
| **Enforcement ownership** | YOUR app enforces (Meteroid only exposes state) | ✅ QBill engine enforces via entitlements API + Redis | QBill — built-in enforcement |

### ✅ QBill Advantages

1. **Real-time enforcement latency**: DEC-010 contract: P99 < 5ms at 32 concurrent callers
2. **Built-in enforcement**: QBill's engine enforces limits, not just exposing state
3. **Redis counters**: Sub-5ms hot path integrated with the billing pipeline

### ❌ QBill Gaps

**G-MT-04: Billing and entitlement concerns are conflated** — **MEDIUM PRIORITY**

The same meter is used for "what we charge for" and "what we cap." This means:
- Adding a metered entitlement requires creating a billing meter (or vice versa)
- Cannot cap usage on a metric without also billing for it
- Cannot independently configure reset periods for entitlements vs billing

**G-MT-05: No sliding window rate limiting** — **LOW PRIORITY**

Meteroid supports true rolling window evaluation (trailing 1 hour). QBill only supports per-period evaluation.

**G-MT-06: Limited reset strategies** — **LOW PRIORITY**

Meteroid offers 5 reset strategies. QBill's limits reset with the subscription billing cycle only.

---

## 7. Subscription Lifecycle & Proration

### Meteroid Subscription States

```
Pending Activation → (Pending Charge →) → Trial Active → Trial Expired → Active → Suspended/Cancelled/Completed/Superseded
```

| State | Description |
|---|---|
| **Active** | Set at `max(activation_date, payment_confirmation_date)` |
| **Suspended** | Billing retries continue; entitlements NOT auto-revoked |
| **Completed** | Reached at defined end date; customer rolls to Free plan if exists |
| **Superseded** | Replaced by upgrade/downgrade/migration; stops billing |

### Meteroid Proration Rules

| Event | Rule |
|---|---|
| Cancel — already invoiced | No auto-refund; manual Credit Note for unused time |
| Cancel — future period | Auto-generates prorated invoice for used days |
| Seat added mid-period | Prorated charge, immediate dedicated invoice |
| Seat removed mid-period | No refund this period; lower count on next invoice |
| Plan upgrade (immediate) | Prorated: credit(old, remaining) + charge(new, remaining) |
| Plan downgrade (immediate) | If less owed: net credit → customer balance → auto-consumed by future invoices |
| Plan change (end-of-period) | No proration — takes effect at next cycle |

### QBill Proration Rules (BILLING_MATH.md P-1 through P-5)

**Formula:**
```
f = days_in_sub_window / days_in_full_period
```

| Event | Rule |
|---|---|
| Base fee proration (P-2) | `round_half_up(base_amount × f, minor)` — per sub-window |
| Included allowance (P-3) | `floor(included_units × f)` — per sub-window |
| Seat proration (P-4) | `seatPrice × seats × segDays / periodDays` — per sub-window |
| Upgrade timing (P-5) | Immediate |
| Downgrade timing (P-5) | Next period start (default) |
| Cancel — immediate | Prorates base fee, generates final invoice |
| Cancel — end-of-period | `cancel_at_period_end=true`, runs to period end |

### Comparison

| Aspect | Meteroid | QBill | Verdict |
|---|---|---|---|
| **Proration granularity** | Day-based | Day-based, calendar days (P-1) | Tie |
| **Included allowance proration** | Not specified | ✅ `floor(included_units × f)` (P-3) | **QBill** |
| **Seat proration** | ✅ Slot-based proration (add→charge, remove→no refund) | ✅ Per sub-window (P-4) | Tie (different mechanisms) |
| **Upgrade handling** | Prorated credit + charge | Immediate (P-5) with sub-window rating | QBill simpler |
| **Downgrade handling** | Net credit → balance (non-refunded) | Next period start (P-5) | Different: Meteroid applies immediately; QBill defers |
| **Mid-period cancel** | Prorated invoice for used days | Prorated base fee + final invoice | Similar |
| **Immediate dedicated invoice** | ✅ Seats trigger immediate invoice | ❌ Not specified | **Meteroid** — better for per-seat billing |
| **No auto-refund on cancel** | ✅ Manual Credit Note | ✅ Not specified | Meteroid is explicit |

### ✅ QBill Advantages

1. **Included allowance proration**: P-3 explicitly handles prorated allowances with floor()
2. **Versioned sub-windows**: Each sub-window rated against its correct plan version
3. **Upgrade/downgrade timing**: Clear rules (P-5) for when changes take effect

### ❌ QBill Gaps

1. **No immediate seat-add invoice**: Meteroid generates a dedicated invoice when a seat is added mid-period. QBill's behavior is not documented.
2. **Downgrade credit treatment**: Meteroid explicitly states downgrades create a non-refunded account credit. QBill defers downgrades to next period but doesn't specify credit behavior for immediate cancellations.
3. **Suspension behavior**: Meteroid explicitly states billing retries continue during suspension. QBill's suspension behavior is less documented.

---

## 8. Invoice Calculation

### Meteroid Invoice Pipeline

- **One invoice per subscription per billing period** (optionally consolidated)
- **Line items map 1:1** to pricing components
- **Sub-lines** (tier breakdowns, per-group usage) are system-generated
- **Draft → Finalized → (Voided | Uncollectible)** with parallel payment status (Paid/Partially Paid/Unpaid/Errored)
- **Grace period**: End of billing period → end of grace → auto-advance to Finalized
- **Once Finalized, immutable** — corrections via Credit Note
- **Coupons**: Fixed-amount or percentage at subscription level
- **Floor price**: `max(calculated, floor)` evaluated before coupons

### QBill Invoice Pipeline (BILLING_MATH.md M-5)

```
Step 1: Subtotal = Σ(line items) — 6 line types
Step 2: Credits_applied = min(subtotal, available FEFO credits)
Step 3: Taxable = subtotal − credits_applied
Step 4: Tax = round_half_up(taxable × rate, minor_units)
Step 5: Total = taxable + tax
```

**State machine**: draft → pending (after grace) → paid/overdue/voided
**Additional features**: Input snapshots, rate source tracking, FEFO credit ordering, 6 typed line items

### Comparison

| Aspect | Meteroid | QBill | Verdict |
|---|---|---|---|
| **Line item types** | Per pricing component (untyped) | ✅ 6 typed types (BASE_FEE, USAGE, OVERAGE, COMMIT_TRUE_UP, SEAT, ADJUSTMENT) | **QBill** |
| **Credit application** | Not detailed | ✅ FEFO (4 priority levels) | **QBill** |
| **Floor price** | ✅ Plan/product level minimum | ❌ Not implemented | **Meteroid** |
| **Coupons** | ✅ Subscription-level fixed/percentage | ❌ Not implemented | **Meteroid** |
| **Input snapshots** | ❌ Not stored | ✅ Versioned refs on invoice | **QBill** |
| **Rate source tracking** | ❌ Not tracked | ✅ Per-line-item provenance | **QBill** |
| **Grace window** | ✅ Configurable | ✅ T-5 (INVOICE_GRACE_HOURS) | Tie |
| **Invoice preview** | ✅ "Upcoming Invoice" real-time | ❌ Not documented | **Meteroid** |
| **Immutability** | ✅ Once Finalized | ✅ After grace (draft mutable, pending+ immutable) | Tie |
| **Sub-lines** | ✅ Tier breakdowns, per-group | ✅ Per sub-window breakdowns | Tie |

### ✅ QBill Advantages

1. **Input snapshots**: Versioned inputs enable re-rating (CR-1), simulation (CR-9), and full audit
2. **Rate source tracking**: Every line item records its rate provenance — Meteroid doesn't track this
3. **Typed line items**: 6 distinct types with different billing behavior
4. **FEFO credit ordering**: Deterministic 4-level credit consumption

### ❌ QBill Gaps

1. **No floor price/minimum commitment**: Meteroid supports plan/product-level minimums
2. **No coupons**: Meteroid supports subscription-level coupons
3. **No invoice preview**: Meteroid has a real-time "Upcoming Invoice" view. QBill only generates at period boundaries.

---

## 9. AI/LLM Token Billing

### Meteroid's Approach (from §9 worked example)

Meteroid provides a concrete LLM token-billing example:

- **One billable metric** (`llm_tokens`) with Sum aggregation on `tokens` property
- **Segmentation** by `token_type` (input/output) — single dimension
- **Grouping key** = `project_id` — per-project cost breakdown
- **Tiered pricing** on output segment, per-unit on input segment
- **Explicit warning**: Grouping breaks tiering — each project resets to tier 1

### QBill's Approach

- **Separate meters** per model × token_type (no filter/segment system)
- **MATRIX pricing** for per-model × per-token-type rates
- **Rating waterfall** for negotiated rates
- **Sub-window proration** for plan changes

### Comparison

| Aspect | Meteroid | QBill | Verdict |
|---|---|---|---|
| **Per-model pricing** | Via segmentation (add model value to segment) | Via MATRIX model (dimension-based rate lookup) | Tie (different mechanisms) |
| **Per-token-type pricing** | Via segmentation (input/output as dimension) | Via separate meters per type | **Meteroid** — cleaner |
| **Per-project breakdown** | ✅ Grouping key on metric | ❌ Would need separate meters per project | **Meteroid** |
| **Tiering × grouping interaction** | ✅ Explicitly documented (per-group, not per-total) | ❌ Not applicable (no grouping) | Meteroid transparent |
| **Conversion factor** | ✅ (e.g., bytes → KB) | ⚠️ Via unit price or PACKAGE size | Similar |
| **Mid-cycle changes** | Prorated credit + charge | ✅ Sub-window proration (P-1 through P-5) | **QBill** |
| **Real-time enforcement** | ❌ Not built-in | ✅ Redis counters + wallet (<5ms) | **QBill** |

### ✅ QBill Advantages

1. **Real-time enforcement**: Redis-based <5ms hot path — Meteroid has no equivalent
2. **Rating waterfall**: Contract-specific rates for enterprise negotiated pricing
3. **COST_PLUS pricing**: Native BYOK markup — unique to QBill
4. **Invoice reproducibility**: Byte-for-byte audit trail

### ❌ QBill Gaps

**G-MT-07: No per-project/per-group cost breakdown** — **HIGH PRIORITY**

Meteroid's grouping key enables per-project invoice line items — critical for multi-tenant AI platforms where each customer project has its own cost center.

**G-MT-08: Segmentation vs separate meters** — **MEDIUM PRIORITY**

Meteroid segments a single metric by `token_type` (input/output). QBill requires separate meters. Same gap as Lago's filters.

---

## 10. Identified Gaps & Action Items

### High-Priority Gaps

| # | Gap | Meteroid Feature | Impact | Suggested Fix |
|---|---|---|---|---|
| **G-MT-01** | **No grouping key on metrics** | First-class grouping key with per-group invoice lines and tiering implications | Cannot provide per-project cost breakdowns — critical for multi-tenant AI platforms | Add `groupBy` field to meter definitions. Engine `rateSegmentUsage()` already groups by `(model, token_type)` — extend to support arbitrary groupBy keys |
| **G-MT-04** | **Billing and entitlement conflated** | Separate entitlement system from billing metrics | Usage limits reference billing meters; cannot cap independently of billing | Separate `customer.usage_limits` from meter definitions; add dedicated entitlement entity with independent reset periods |
| **G-MT-09** | **No floor price / minimum commitment** | Plan/product-level minimum `max(calculated, floor)` | No guaranteed minimum revenue — customers can fall below contractual commitments | Add `minimum_amount` to pricing models (engine already has MinimumAmount in resolver); add plan-level minimum commitment |

### Medium-Priority Gaps

| # | Gap | Meteroid Feature | Suggested Fix |
|---|---|---|---|
| **G-MT-02** | **No explicit segmentation on metrics** | Segmentation (none/single/2D independent/2D dependent) | Add segmentation configuration to meter definitions |
| **G-MT-03** | **Grouping-tiering interaction not documented** | Explicitly documented | When grouping is added, document whether tiering applies per-group or per-total |
| **G-MT-07** | **No per-project cost breakdown** | Grouping key → per-group invoice sub-lines | Extend invoice line items to support sub-lines with group attribution |
| **G-MT-10** | **No invoice preview** | "Upcoming Invoice" real-time recalculation | Add invoice preview endpoint that recalculates from ingested-to-date events |

### Low-Priority Gaps

| # | Gap | Suggested Fix |
|---|---|---|
| **G-MT-05** | **No sliding window rate limiting** | Add sliding window support to enforcement engine |
| **G-MT-06** | **Limited entitlement reset strategies** | Add Calendar, Fixed Window, Sliding Window reset options |
| **G-MT-11** | **No explicit suspension behavior documentation** | Document billing retries during suspension |
| **G-MT-12** | **No explicit timestamp validation window** | Add 24h-past / 1h-future guardrails at ingest |

### Meteroid's Own Recommendations (§10) — Verified

Meteroid §10 highlights 4 contrast points for QBill. Here's the verification:

| # | Meteroid Suggestion | QBill Status | Action Needed |
|---|---|---|---|
| 1 | "Send raw events, aggregate at period-end" — matches QBill's approach | ✅ Already aligned — QBill's Go/Kafka/ClickHouse path follows this model | None |
| 2 | "Grouping-key-breaks-tiering is a design decision" | ❌ QBill has no grouping, so this decision hasn't been made | Document decision when grouping is added |
| 3 | "Metered Entitlement should be separate from Billable Metric" | ⚠️ Partially addressed — engine enforcement is separate, but catalog meter definitions conflate measurement and capping | **G-MT-04**: Separate entitlement from billing meters |
| 4 | "Plan immutability via versioning" | ✅ Already implemented via `catalog.plan_versions` and `catalog.rate_card_versions` | None |

---

## 11. Source Cross-Reference

| QBill Source | Content Referenced |
|---|---|
| `BILLING_MATH.md` M-1 to M-6 | Money precision, rounding, invoice arithmetic |
| `BILLING_MATH.md` P-1 to P-5 | Proration rules (sub-windows, allowance, seats) |
| `BILLING_MATH.md` T-1 to T-6 | Time rules (UTC, anniversary, grace, half-open periods) |
| `BILLING_MATH.md` §7 | FEFO credit application |
| `BILLING_MATH.md` W-1 to W-5 | Wallet burn, reconciliation, overdraft |
| `ADR-001` §3 | Hybrid billing, rate resolution waterfall |
| `ADR-001` §3.3 | Rating waterfall (contract → rate card → plan → unrated) |
| `ADR-001` §3.4 | Invoice purity invariant |
| `phase_2_billing_worker.md` | Invoice lifecycle, enforcement API, acceptance criteria TC-01 to TC-17 |
| `ERD.md` §2 | Customer domain — usage_limits, limit_overrides, usage_summary |
| `ERD.md` §3 | Catalog — meters, pricing models, rate cards |
| `ERD.md` §4 | Billing entities — invoices, credits, wallet |
| `DECISIONS.md` DEC-002 | Anti-spoofing (KeyContext authority) |
| `DECISIONS.md` DEC-010 | Enforcement latency contract (P99 < 5ms @ c=32) |
| `DECISIONS.md` DEC-011 | SOFT limit defaults to 80% of HARD |
| `DECISIONS.md` DEC-012 | Org-level counters are lifetime running totals |
| `engine/internal/invoice/generate.go` | Pure invoice function |
| `engine/internal/rating/resolver.go` | Rating waterfall with all 7 pricing models |
| `engine/internal/enforcement/` | Usage limit enforcement |
| `engine/internal/wallet/` | Wallet operations |

---

*End of document. Generated 2026-07-16.*
