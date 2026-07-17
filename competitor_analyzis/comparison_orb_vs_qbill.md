# Orb vs QBill — Detailed Comparison Analysis

**Date:** 2026-07-16
**Purpose:** Comprehensive comparison of Orb's (withorb.com) query-based billing architecture against QBill's implementation. Orb is the most architecturally sophisticated competitor and the one with the closest philosophical alignment to QBill's design.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Comparison — Query-Based vs Pure-Function](#2-architecture-comparison--query-based-vs-pure-function)
3. [Events, Ingestion & Corrections](#3-events-ingestion--corrections)
4. [Billable Metrics & Pricing Models](#4-billable-metrics--pricing-models)
5. [Credit Systems — Orb's Dual Model](#5-credit-systems--orbs-dual-model)
6. [Invoice Calculation — The 8-Step Pipeline](#6-invoice-calculation--the-8-step-pipeline)
7. [Adjustments, Discounts & Commitments](#7-adjustments-discounts--commitments)
8. [Subscription Lifecycle & Proration](#8-subscription-lifecycle--proration)
9. [Customer Hierarchy & Billing Groups](#9-customer-hierarchy--billing-groups)
10. [AI/LLM Token Billing Comparison](#10-aillm-token-billing-comparison)
11. [Orb §13 Concept Map Verification](#11-orb-13-concept-map-verification)
12. [Identified Gaps & Action Items](#12-identified-gaps--action-items)
13. [Source Cross-Reference](#13-source-cross-reference)

---

## 1. Executive Summary

| Dimension | Orb | QBill | Advantage |
|---|---|---|---|
| **Invoice engine philosophy** | Query-based (fresh SQL over immutable events) | Pure function with versioned input snapshots | **Tie** — both achieve determinism via different mechanisms |
| **Parallel pipelines** | ✅ Columnar (query, source of truth) + Stream (real-time, never billed from) | ✅ ClickHouse (source of truth) + Redis (enforcement cache, reconciled nightly) | **Tie** — same pattern |
| **Invoice pipeline steps** | **8 steps** (most comprehensive) | 5 steps | **Orb** — full discount/commitment pipeline |
| **Credit system** | **Dual**: Prepaid Credits (cost-basis tracked) + Customer Balance (post-tax AR) | FEFO credits + Wallet (Redis real-time) | **Orb** — cost-basis tracking for rev-rec |
| **Correction mechanism** | Amend event + Deprecate event + Backfill (recalculates affected invoices) | Re-rating + Credit notes (CR-1, CR-4) | **QBill** — credit notes provide formal correction primitive |
| **Pricing models** | 7 (Unit, Tiered, Bulk, Package, Dimensional, Matrix, License) | 7 (FLAT, PER_UNIT, TIERED_GRADUATED, TIERED_VOLUME, PACKAGE, MATRIX, COST_PLUS) | Tie |
| **Adjustments** | **5 types**: Usage discount, Amount discount, Percentage discount, Minimum, Maximum | ❌ Not implemented | **Orb** |
| **Customer hierarchy** | Parent/Child (2 levels, max 100 children) | 4-level (Org → Customer → End User) | QBill has deeper hierarchy |
| **Revenue recognition** | ✅ Via cost-basis tracking on credits | ✅ ASC 606/IFRS 15 (CR-5) | Tie |
| **Dependency graph invalidation** | ✅ Tracks invoice-to-data dependencies, recalculates only affected fragments | ❌ Re-rating re-runs full invoice function | **Orb** — more efficient for targeted corrections |

---

## 2. Architecture Comparison — Query-Based vs Pure-Function

### Orb's Query-Based Philosophy

Orb's core insight: **most billing systems mutate counters on event arrival, losing the context of how those counters were built.** Orb instead:

1. **Stores events immutably** in a columnar OLAP store
2. **An invoice is a fresh SQL query** run at the moment it's needed
3. **Three things are versioned**: raw events (immutable), metric definitions (versioned), pricing/subscription timelines (versioned)
4. **Pure determinism**: same inputs → same invoice, every time

**Two parallel pipelines:**
| Pipeline | Store | Purpose | Source of truth for invoices? |
|---|---|---|---|
| **Query-based** | Columnar OLAP | Invoice generation, revenue reporting, backfills | **Yes — always** |
| **Stream-based** | Redis/MemoryDB | Usage alerts, threshold invoicing, rate limiting | No — cache/optimization only |

**Dependency graph**: Orb tracks invoice-to-data dependencies. When something changes (late event, pricing update), only **affected fragments** are recalculated — not the entire historical dataset.

### QBill's Pure-Function Philosophy

QBill's approach (ADR-001 §3.4):

1. **Stores events immutably** in ClickHouse (ReplacingMergeTree)
2. **Invoice is a pure function**: `invoice.Generate(Inputs)` where Inputs are versioned snapshots
3. **Same inputs → same invoice byte-for-byte**
4. **Input snapshots on invoice**: `plan_version_id`, `rate_card_version_id`, `aggregation_watermark`

**Two parallel pipelines:**
| Pipeline | Store | Purpose | Source of truth for invoices? |
|---|---|---|---|
| **Cold path** | ClickHouse (query) | Invoice generation, re-rating | **Yes — always** |
| **Hot path** | Redis (stream) | Usage counters, wallet burndown, enforcement (<5ms) | No — reconciled nightly (W-4) |

### Comparison

| Aspect | Orb | QBill | Verdict |
|---|---|---|---|
| **Core mechanism** | Query-based (SQL over columnar store) | Pure function (Go code over versioned inputs) | Different — both achieve determinism |
| **Invoice reproducibility** | ✅ Deterministic (same query = same result) | ✅ **Byte-for-byte** (same inputs = same binary) | **QBill** — stronger guarantee |
| **Input snapshots** | ✅ Metric definitions + pricing timelines versioned | ✅ plan_version_id, rate_card_version_id, aggregation_watermark | Tie |
| **Re-rating efficiency** | Dependency graph → recalculate only affected fragments | Re-run full invoice function for the period | **Orb** — more efficient for targeted corrections |
| **Real-time pipeline isolation** | ✅ Stream pipeline NEVER feeds invoice calculation | ✅ Redis NEVER feeds invoice calculation (W-4) | Tie — both architecturally correct |
| **Hot path latency** | Stream-based (seconds) | ✅ Redis-based (<5ms at c=32, DEC-010) | **QBill** — faster enforcement |

### ✅ QBill Advantages

1. **Byte-for-byte reproducibility**: QBill's pure function generates identical binary output for identical inputs — stronger than Orb's "same query = same result"
2. **Faster hot path**: DEC-010 guarantees <5ms at 32 concurrent vs Orb's seconds-latency stream pipeline
3. **Formal correction primitive**: Re-rating + credit notes provide an auditable correction trail

### ❌ QBill Gaps

**G-OR-01: No dependency graph invalidation** — **MEDIUM PRIORITY**

Orb tracks which invoices depend on which data, recalculating only affected fragments. QBill re-runs the full invoice function for the entire period. At scale, this is computationally expensive.

---

## 3. Events, Ingestion & Corrections

### Orb's Event Model

```json
{
  "event_name": "cluster_compute",
  "timestamp": "2022-02-02T00:00:00Z",
  "external_customer_id": "9fc80ac0-d9ff-11ec-9d64-0242ac120002",
  "properties": {
    "cluster_name": "staging-cluster-1",
    "compute_ms": 912,
    "aws_region": "us-east-1"
  },
  "idempotency_key": "unique-per-event-id"
}
```

**Ingestion:**
- Batch: up to 500 events/request
- Native capacity: 1,000+ events/sec; scales to tens of millions/day
- Hosted rollups for 1M+ events/sec
- Multiple ingestion paths: Segment, Reverse ETL, S3/GCS, Kinesis/Lambda

**Corrections (3 mechanisms):**
1. **Amend event**: Overwrite a single event by `event_id`
2. **Deprecate event**: Soft-delete a single event
3. **Backfill**: 3-step flow (create → accumulate corrected events → close) → async recalculation of all affected invoices

**Ingestion grace period**: Configurable (~12h default), also serves as minimum time before end-of-period invoice issuance.

### QBill's Event Model

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "org_id": "org_abc123",
  "customer_id": "cust_def456",
  "end_user_id": "eu_789ghi",
  "timestamp_ms": 1721152200000,
  "source_mode": "virtual_key",
  "properties": { "model": "gpt-4", "total_tokens": 1500, "cost": "0.000041" }
}
```

**Ingestion:**
- Batch: up to 50,000 events/request with partial accept
- Dedup: Redis SETNX with 24h TTL (first-write-wins)
- Anti-spoofing: KeyContext overwrites payload (DEC-002)

**Corrections:**
- **Re-rating (CR-1)**: Re-run invoice function with corrected inputs, emit credit/debit notes
- **Credit notes (CR-4)**: Full state machine (draft → issued → applied/refunded)
- Original invoices are never mutated

### Comparison

| Aspect | Orb | QBill | Verdict |
|---|---|---|---|
| **Event immutability** | ✅ Immutable (corrections layered on top) | ✅ Immutable in ClickHouse (ReplacingMergeTree) | Tie |
| **Correction granularity** | Per-event (amend/deprecate) + bulk backfill | Period-level (re-rating) | Orb more granular at event level |
| **Correction mechanism** | Async recalculation of affected invoices | Credit notes + re-rating run | Different approaches |
| **Batch size** | 500 events/request | 50,000 events/request | **QBill** — 100x larger |
| **Ingestion paths** | 5+ paths (Segment, S3, Kinesis, etc.) | LiteLLM callback + direct API | **Orb** — more integration options |
| **Grace period** | ✅ Configurable (~12h default) | ✅ INVOICE_GRACE_HOURS (default 36h) | Tie |
| **Dedup** | `idempotency_key` within grace period | `event_id + org_id` via SETNX (24h TTL) | QBill — explicit 409 vs Orb's silent dedup |

### ✅ QBill Advantages

1. **Batch size**: 50,000 vs 500 — 100x larger for high-volume scenarios
2. **Explicit dedup rejection**: 409 DUPLICATE_EVENT vs Orb's silent dedup — more auditable
3. **Formal credit note primitive**: Full correction primitive with state machine vs Orb's "amend event"

### ❌ QBill Gaps

**G-OR-02: Limited ingestion paths** — **LOW PRIORITY**

Orb supports 5+ ingestion paths (Segment, Reverse ETL, S3/GCS, Kinesis/Lambda). QBill only supports LiteLLM callback + direct API.

**G-OR-03: No per-event correction** — **MEDIUM PRIORITY**

Orb supports amend/deprecate individual events. QBill can only correct at period level via re-rating.

---

## 4. Billable Metrics & Pricing Models

### Orb's Pricing Models (7 types)

| Model | Formula | QBill Equivalent |
|---|---|---|
| **Unit** | `qty × unit_amount` | PER_UNIT |
| **Tiered** | `Σ(tier_units × tier_rate)` (graduated) | TIERED_GRADUATED |
| **Bulk** (Volume) | `total × rate(tier)` (single rate) | TIERED_VOLUME |
| **Package** | `ceil(qty / packageSize) × packageAmount` | PACKAGE |
| **Dimensional** | Rate by unlimited dimensions (model×region×type) | MATRIX (but Orb's is unlimited dimensions) |
| **Matrix** (legacy) | ≤2 dimensions with default fallback | MATRIX |
| **License** | Seat/entitlement with per-license credit allocation | SEAT line type (no per-license credit) |
| **Fixed fee** | `unit × fixed_price_quantity` | FLAT |

**Orb's Dimensional pricing** is the most advanced among all competitors — **unlimited dimensions** with default fallback. QBill's MATRIX model supports dimensions but isn't explicitly documented as unlimited.

### Bonus Pricing Models in Competitors vs QBill

| Model | Orb | QBill | FlexPrice | Lago | Meteroid | OpenMeter |
|---|---|---|---|---|---|---|
| Flat fee | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Per-unit | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Tiered (graduated) | ✅ | ✅ | ❌ (ambiguous) | ✅ | ✅ | ✅ |
| Volume/Bulk | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Package | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Matrix/Dimensional | ✅ (unlimited) | ✅ (MATRIX) | ❌ | ✅ (via filters) | ✅ (2D) | ❌ |
| Cost-plus/Dynamic | ❌ | ✅ (COST_PLUS) | ❌ | ❌ | ❌ | ✅ |
| Percentage | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| License/Seat | ✅ | ⚠️ (SEAT line) | ❌ | ❌ | ✅ (Slot) | ❌ |
| Fixed fee | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### ✅ QBill Advantages

1. **COST_PLUS**: Unique to QBill among all competitors
2. **MATRIX pricing**: Competitive with Orb's Dimensional pricing for LLM use cases

### ❌ QBill Gaps

1. **Orb's Dimensional pricing** supports unlimited dimensions with default fallback — QBill's MATRIX is dimension-agnostic but less documented for unlimited dimensions
2. **License pricing with per-credit allocation** — Orb supports per-license credit pools; QBill's SEAT line type doesn't

---

## 5. Credit Systems — Orb's Dual Model

### Orb's Two Separate Systems

| | **Prepaid Credits** (Credit Ledger) | **Customer Balance** |
|---|---|---|
| Purpose | Usage commitment / consumption tracking | AR adjustment ("wallet") |
| Applied | **Before tax**, during invoice calc | **After tax**, at issuance |
| Revenue recognition | Yes — recognized as consumed | No impact |
| Expiration | Supported (per-block) | Never expires |
| Currency | Real or **virtual/custom** pricing units | Customer's billing currency only |

**Prepaid Credit deduction order:**
1. License allocations (per-license credits)
2. Item-scoped blocks (before unscoped)
3. Soonest-expiring blocks first
4. Zero/lower cost-basis before higher (free credits burn before paid)
5. FIFO by creation time

**Cost basis per credit block**: Each block has a `per_unit_cost_basis` — $0 = promotional (no rev-rec impact), non-zero = must be invoiced for proper revenue recognition.

**Allocations** (recurring credit grants): Configured at plan level, auto-injected each period. Zero cost-basis = free inclusion; non-zero = billed in-advance, drives revenue recognition line item.

### QBill's Credit System

**FEFO credits** (BILLING_MATH.md §7):
```
Priority order: compensation(0) → promotional(1) → prepaid(2) → commit(3)
Then: expires_at → created_at → ID
```

**Wallet**: Real-time Redis burndown, separate from FEFO credits, nightly reconciliation (W-4), bounded overdraft (W-5).

### Comparison

| Aspect | Orb | QBill | Verdict |
|---|---|---|---|
| **Systems** | Dual (Prepaid Credits + Customer Balance) | Dual (FEFO Credits + Wallet) | Tie — both have two systems |
| **Cost basis tracking** | ✅ Per-block cost basis (drives rev-rec) | ❌ Not tracked | **Orb** — enables proper rev-rec differentiation |
| **Deduction steps** | 5-step (license → item-scoped → expiry → cost-basis → FIFO) | 4-step (priority → expiry → created → ID) | Orb has cost-basis step QBill lacks |
| **Rollover** | ✅ Configurable per grant (via allocations) | ❌ Non-rollover by default (TR-2) | **Orb** |
| **Virtual currencies** | ✅ Custom pricing units (e.g., "tokens") | ❌ Real currency only | **Orb** |
| **Customer balance (post-tax)** | ✅ After-tax AR adjustment | ❌ Not separate from credits | **Orb** |
| **Revenue recognition integration** | ✅ Via cost-basis | ✅ Via revrec module (CR-5) | Tie |
| **Real-time burndown** | ❌ No (applied at invoice time) | ✅ Redis hot-path (<5ms) | **QBill** |
| **Pending→committed entries** | ✅ During grace period | ✅ Ledger entries at finalization | Tie |

### ✅ QBill Advantages

1. **Real-time wallet burndown**: QBill's Redis hot path provides sub-5ms enforcement — Orb applies credits at invoice time only
2. **Nightly reconciliation**: W-4 drift detection and correction — Orb has no equivalent
3. **Bounded overdraft**: W-5 prevents runaway negative balances

### ❌ QBill Gaps

**G-OR-04: No cost-basis tracking on credits** — **HIGH PRIORITY**

Orb's cost-basis per credit block enables:
- Differentiating promotional credits (cost-basis $0, no rev-rec impact) from purchased credits (cost-basis > $0, drives revenue recognition)
- Tracking which specific credit block covered each invoice line
- Proper ASC 606 revenue recognition for prepaid credit consumption

QBill's FEFO credits use 4 priority levels but don't track cost basis — a promotional credit (priority 1) and a prepaid credit (priority 2) are treated differently by priority but the actual $ value of the credit is not tracked for rev-rec purposes.

**G-OR-05: No customer balance (post-tax AR adjustment)** — **MEDIUM PRIORITY**

Orb's Customer Balance handles refunds, goodwill credits, and rounding carryover **after tax** — meaning they don't affect revenue recognition. QBill has no equivalent separate mechanism.

**G-OR-06: No virtual/custom currencies** — **LOW PRIORITY**

Orb supports custom pricing units (e.g., "tokens" as a virtual currency with conversion rate to USD). QBill uses real currency only.

---

## 6. Invoice Calculation — The 8-Step Pipeline

### Orb's 8-Step Pipeline (Most Comprehensive of All Competitors)

```
Step 1:  Quantity         → evaluate billable metric for the period
Step 2:  Subtotal         → apply pricing function (qty × rate)
Step 3:  Adjustments      → Usage discount → Amount discount → Percent discount → Minimum → Maximum
Step 4:  Prepaid credits  → deduct from credit ledger (in-arrears charges only)
Step 5:  Conversion       → virtual currency → real currency
Step 6:  Partial invoices → subtract already-invoiced amounts (threshold billing)
Step 7:  Tax              → per line item, via TaxJar/Avalara/etc.
Step 8:  Customer balance → applied to final total after tax
```

**Key rules:**
- **Minimums BEFORE prepaid credits**: Prevents customers from using credits to dodge contractual minimums
- **Customer balance AFTER tax**: Customer balance has no revenue impact; prepaid credits are pre-tax
- **Adjustment distribution**: Proportional (by subtotal share) for discounts; even for minimums
- **Minimums prorated**: A $100 minimum on a 15-day subscription = $50
- **Threshold invoicing**: Partial invoice when usage crosses threshold; netted at period end

### QBill's 5-Step Pipeline (BILLING_MATH.md M-5)

```
Step 1: Subtotal = Σ(line items) — 6 line types
Step 2: Credits_applied = min(subtotal, available FEFO credits)
Step 3: Taxable = subtotal − credits_applied
Step 4: Tax = round_half_up(taxable × rate, minor_units)
Step 5: Total = taxable + tax
```

### Side-by-Side Pipeline Comparison

```
Step | Orb (8 steps)                          | QBill (5 steps)
─────┼────────────────────────────────────────┼────────────────────────────
1    | Quantity (evaluate metric)             | Subtotal (sum of line items)
2    | Subtotal (apply pricing)               | FEFO credits applied
3    | Adjustments (4 types in fixed order)   | ❌ NO ADJUSTMENT STEP
4    | Prepaid credits (in-arrears only)      | ❌ (part of step 2)
5    | Virtual currency conversion            | ❌ NOT SUPPORTED
6    | Partial invoice netting                | ❌ NOT SUPPORTED
7    | Tax (per line item)                    | Tax (invoice-level, single rate)
8    | Customer balance (post-tax, post-all)  | ❌ NOT SUPPORTED
```

### Gap Analysis of Orb's Pipeline Steps Not in QBill

| Orb Step | QBill Status | Why It Matters |
|---|---|---|
| **3 — Adjustments** (usage/amount/% discount, min/max) | ❌ **Not implemented** | Cannot offer discounts, minimum commitments, or spending caps |
| **4 — Prepaid credits** (separate from FEFO) | ⚠️ Wallet exists but is real-time only, not invoice-time | Wallet credits and FEFO credits are separate systems |
| **5 — Virtual currency conversion** | ❌ Not supported | Cannot use custom pricing units (e.g., "tokens" as currency) |
| **6 — Partial invoice netting** | ❌ Not supported (no progressive billing) | Can't invoice mid-cycle; no threshold billing |
| **7 — Per-line-item tax** | ❌ Single-rate, invoice-level only | Cannot handle per-product-type tax rates |
| **8 — Customer balance (post-tax)** | ❌ Not supported | No after-tax AR adjustments |

### ✅ QBill Advantages

1. **Simplicity**: 5 steps vs 8 — fewer moving parts, easier to reason about
2. **FEFO determinism**: Credit consumption is deterministic with 4 priority levels
3. **Single rounding point**: Round-half-up per line item (M-3), then exact arithmetic
4. **Grace window**: Configurable draft → finalize (T-5) — Orb has similar but less configurable

### ❌ QBill Gaps

**G-OR-07: No adjustment pipeline** — **CRITICAL PRIORITY**

This is the **single biggest gap across ALL competitor comparisons**. Orb has 5 adjustment types: usage discount, amount discount, percentage discount, minimum, maximum. QBill has NONE of these.

**G-OR-08: No per-line-item tax** — **MEDIUM PRIORITY**

Orb computes tax per line item (via TaxJar/Avalara). QBill computes tax at invoice level with a single rate.

**G-OR-09: No after-tax customer balance** — **LOW PRIORITY**

Orb's customer balance handles refunds and goodwill credits after tax. QBill has no equivalent.

---

## 7. Adjustments, Discounts & Commitments

### Orb's 5 Adjustment Types

| Type | Effect | Calculation | Expiration |
|---|---|---|---|
| **Usage discount** | Reduces quantity before pricing | `effective_qty = max(0, qty - discount)` | ✅ Per N periods |
| **Amount discount** | Subtracts flat $ from line | `line -= discount_amount` | ✅ Per N periods |
| **Percentage discount** | Subtracts % from line | `line *= (1 - pct)` | ✅ Per N periods |
| **Minimum** | Floors line/invoice total | `amount = max(calculated, min)` | ✅ Per N periods |
| **Maximum** | Caps line/invoice total | Orbs auto-applies discount to bring to cap | ✅ Per N periods |

**Distribution rules:**
- Multi-price adjustment: distributed **proportionally** by subtotal share for discounts
- Minimum deltas distributed **evenly** across applicable line items
- Minimums NEVER applied to fixed-fee-only invoices

### QBill's Current State

**No adjustment system at all.** QBill's invoice pipeline processes:
- Base fees (from plan)
- Usage charges (rated per-waterfall)
- FEFO credits
- Tax

There is no discount, commitment, minimum, or maximum step.

### Comparison

| Adjustment Type | Orb | QBill | Gap Severity |
|---|---|---|---|
| Usage discount ("first N free") | ✅ | ❌ | **High** |
| Amount discount ("$100 off") | ✅ | ❌ | **High** |
| Percentage discount ("15% off") | ✅ | ❌ | **High** |
| Minimum commitment ("at least $500/mo") | ✅ | ❌ **Not implemented (engine has MinimumAmount on models but no invoice-level minimum)** | **High** |
| Maximum spend cap | ✅ | ❌ **Not implemented (engine has MaximumAmount on models but no invoice-level cap)** | **High** |
| Discount expiration | ✅ Per N billing periods | ❌ | **Medium** |
| Proportional distribution | ✅ By subtotal share | ❌ | **Medium** |
| Minimum-evergreen rule | ✅ Min not applied to fixed-fee-only | ❌ | **Low** |

### ✅ QBill Advantage

QBill's pricing models support `MinimumAmount`/`MaximumAmount` per model in the engine (`internal/rating/resolver.go`). This provides per-component min/max at the pricing level, but NOT at the invoice level.

### ❌ Critical Gap

**G-OR-10: No adjustment system** — **CRITICAL PRIORITY** (consolidation of G-OR-07)

This is the most significant gap across all 5 competitor comparisons. QBill cannot:
- Offer promotional discounts (percentage off, fixed $ off)
- Support contractual minimum commitments ("at least $X/month")
- Enforce spending caps (maximum per line/per invoice)
- Implement "first N units free" usage discounts
- Handle discount expiration timelines
- Distribute adjustments proportionally across line items

Every single competitor (FlexPrice, Lago, Meteroid, OpenMeter, Orb) has some form of discount/commitment pipeline. QBill has none.

---

## 8. Subscription Lifecycle & Proration

### Orb's Subscription Model

| Aspect | Orb Behavior |
|---|---|
| **Term** | Longest cadence among component prices |
| **Billing period** | Shortest cadence among component prices |
| **Billing cycle alignment** | 3 choices: Calendar (1st), Anniversary (creation date), Custom |
| **Plan changes** | Versions the subscription; produces 2 invoices (close old, open new) |
| **Proration** | Day-based: `refund = unused_days / period_days × original_fee` |
| **Cancellation** | Immediate or end-of-term (longest cadence) |
| **Backdated cancellation** | Allowed if no paid invoices exist in the period |
| **Uncancelling** | Lossy — prefer fresh plan change |

**Term vs billing period example:**
| Component Prices | Term | Billing Period |
|---|---|---|
| 2 monthly usage charges | Monthly | Monthly |
| 2 monthly + annual platform fee | Annual | Monthly |
| Quarterly service + annual fee | Annual | Quarterly |

### QBill's Subscription Model

| Aspect | QBill Behavior |
|---|---|
| **Billing period** | Anniversary-only (T-3: start-date anchored) |
| **Plan changes** | Versioned sub-windows (P-1 through P-5) |
| **Proration** | Day-based: `f = days_in_sub_window / days_in_full_period` |
| **Upgrade timing** | Immediate (P-5) |
| **Downgrade timing** | Next period start (P-5) |
| **Cancellation** | Immediate or end-of-period |
| **Billing groups** | CR-8 (planned, D-17 not dispatched) |

### Comparison

| Aspect | Orb | QBill | Verdict |
|---|---|---|---|
| **Billing cycle options** | 3 (Calendar, Anniversary, Custom) | 1 (Anniversary only) | **Orb** |
| **Term vs billing period** | ✅ Supports mixed cadences on one subscription | ❌ Single cadence per subscription | **Orb** |
| **Plan change invoicing** | 2 invoices (close + open) | Single invoice with sub-windows | Different approaches |
| **Proration** | Day-based | Day-based with allowance/seat proration (P-2, P-3, P-4) | QBill more comprehensive |
| **Cancellation** | Immediate or end-of-term (longest) | Immediate or end-of-period | Tie |
| **Backdated cancellation** | ✅ Allowed (with restrictions) | ❌ Not specified | **Orb** |
| **Subscription versioning** | ✅ Full audit trail | ✅ Plan versions | Tie |

### ✅ QBill Advantages

1. **Comprehensive proration**: Prorates base fee (P-2), included allowance (P-3), AND seats (P-4) — Orb prorates only the base fee
2. **Allowance floor**: `floor(included_units × f)` prevents fractional allowance grants (P-3)
3. **Sub-window rating**: Each sub-window rated against its correct plan version

### ❌ QBill Gaps

**G-OR-11: Anniversary-only billing** — **MEDIUM PRIORITY**

Orb offers 3 billing cycle options. QBill only supports anniversary-anchored billing. Some customers prefer calendar-month billing.

**G-OR-12: No mixed-cadence subscriptions** — **MEDIUM PRIORITY**

Orb supports different cadences within one subscription (monthly usage + annual platform fee). QBill requires separate subscriptions.

---

## 9. Customer Hierarchy & Billing Groups

### Orb's Parent/Child Hierarchy

- **Parent + up to 100 children**. A customer can only belong to one hierarchy.
- Single subscription bills any mix of parent + children via `usage_customer_ids`
- Different prices in the same plan can target different hierarchy subsets
- **Usage aggregates to parent's invoice**
- Parent's credit ledger drawn down for hierarchy-wide usage
- Currency/tax follows parent
- Plan change wipes `usage_customer_ids` configuration

### QBill's 4-Level Hierarchy

```
Organization → Users (with roles: SUPER_ADMIN, ORG_ADMIN, CUSTOMER, END_USER, DEVELOPER)
             → Customers (ACTIVE/SUSPENDED/CHURNED)
                 → End Users (active/suspended/canceled)
```

**Billing groups (CR-8)**: Consolidation at customer or organization level. Line items retain subscription_id attribution. D-17 not yet dispatched.

### Comparison

| Aspect | Orb | QBill | Verdict |
|---|---|---|---|
| **Depth** | 2 levels (Parent → Child) | 4 levels (Org → Customer → End User) | **QBill** — deeper |
| **Max children** | 100 | Unlimited (within org) | **QBill** |
| **Usage aggregation** | ✅ To parent's invoice | ⚠️ Via billing groups (CR-8, not built yet) | **Orb** — ready now vs planned |
| **Per-hierarchy pricing** | ✅ Different prices for different subsets on same plan | ❌ Requires separate rate cards/contracts | **Orb** |
| **Credit pooling** | ✅ Parent's credits cover all children | ⚠️ Not specified | **Orb** |
| **Currency/tax consistency** | ✅ Follows parent | ⚠️ Single currency per customer (X-1) | Tie |
| **Plan change impact** | Wipes usage_customer_ids | ⚠️ Not specified | Orb documented; QBill not |

### ✅ QBill Advantages

1. **Deeper hierarchy**: 4 levels vs Orb's 2 — supports more complex organizational structures
2. **No child limit**: Orb caps at 100 children per parent

### ❌ QBill Gaps

1. **Billing groups not yet built**: CR-8 is dispatched in D-17, not yet implemented
2. **No per-hierarchy pricing**: Orb supports different prices for different customer subsets on the same plan

---

## 10. AI/LLM Token Billing Comparison

### Orb's AI/Agent Token Example (§12)

Orb provides a complete worked example for AI token billing:

**Business model:**
- Free tier: 100K tokens/month, $0 upfront
- Premium tier: $50/mo + 1M tokens/month
- On-demand: $10/100K tokens (1-year expiry)
- Free grants expire end-of-month; purchased tokens expire 1 year

**Event shape** (raw token counts, never pre-computed amounts):
```json
{
  "event_name": "agent_checkpoint",
  "external_customer_id": "customer-18940",
  "timestamp": "2025-03-09T16:09:53Z",
  "properties": {
    "input_tokens_used": 240,
    "output_tokens_used": 4599,
    "reasoning_tokens_used": 3021,
    "model": "claude-3-7-sonnet-20250219",
    "idempotency_key": "task-id-3"
  }
}
```

**Mechanism:**
- Metric: SUM of tokens across all checkpoints for a customer
- Pricing unit: virtual currency `token` (1:1 with metric unit)
- Allocations: zero-cost-basis free inclusions; on-demand purchases have cost-basis
- Prepaid credits cover in-arrears usage; auto-topup at threshold
- Balance alerts webhook-driven → enable/disable agent access

### QBill's Approach

- MATRIX pricing for per-model × per-token-type rates
- COST_PLUS for BYOK markup
- FEFO credits for credit application
- Wallet for real-time prepaid burndown
- Rating waterfall for contract-specific negotiated rates

### Comparison

| Aspect | Orb | QBill | Verdict |
|---|---|---|---|
| **Token granularity** | input/output/reasoning (via properties) | input/output (via MATRIX dimensions) | Orb has reasoning_tokens |
| **Included allowances** | Allocations (zero cost-basis) | P-3 prorated included units | Different |
| **Prepaid tokens** | Virtual currency (cost-basis tracked) | Wallet + FEFO credits | Orb has cost-basis |
| **Auto-topup** | ✅ Threshold-based webhook | ✅ Stripe PaymentIntent-based | Tie |
| **Balance alerts** | ✅ credit_balance_depleted/recovered | ✅ Low balance alerts (W-3) | Tie |
| **Contract rates** | ❌ Subscription-level overrides only | ✅ Full rating waterfall | **QBill** |
| **Re-rating** | ✅ Via amend/deprecate/backfill | ✅ Via credit notes (CR-1, CR-4) | Tie |
| **Real-time enforcement** | ❌ Balance check via API | ✅ Redis wallet (<5ms) | **QBill** |
| **Virtual currency** | ✅ "token" as pricing unit | ❌ Real currency only | **Orb** |

### ✅ QBill Advantages

1. **Rating waterfall**: Contract-specific negotiated rates — Orb requires subscription-level overrides
2. **Real-time enforcement**: QBill's Redis wallet provides sub-5ms enforcement
3. **Re-rating**: Formal credit note primitive for corrections

### ❌ QBill Gaps

1. **No virtual currency**: Orb uses "tokens" as a pricing unit. QBill uses real currency only.
2. **No cost-basis tracking**: Orb tracks per-block cost basis for revenue recognition on prepaid tokens

---

## 11. Orb §13 Concept Map Verification

Orb's §13 provides a direct concept map to QBill. Here's the verification:

| # | Orb Concept | QBill Mapping | Verdict |
|---|---|---|---|
| 1 | Columnar event store, query-based invoicing | Kafka → ClickHouse ingestion path (source of truth) | ✅ **Accurate** |
| 2 | Stream-based alerting pipeline (Redis) | Redis/BullMQ — must stay read-only relative to billing calc | ✅ **Accurate** — QBill's Redis is enforcement cache, reconciled nightly (W-4) |
| 3 | Event (`event_name` + `properties`) | Raw usage event from LiteLLM gateway | ✅ **Accurate** |
| 4 | Billable Metric (filter + aggregate) | Meter definitions in `meter` module | ✅ **Accurate** |
| 5 | Price (unit/tiered/bulk/package/dimensional/license) | Pricing model config in `billing` module | ⚠️ **Minor correction**: Pricing models are in the **catalog** module, not billing. Billing reads them via EngineClient. |
| 6 | Plan + Adjustments | Plan/discount/minimum config | ✅ **Accurate** (though adjustments not yet implemented) |
| 7 | Subscription (term vs. billing period) | Subscription lifecycle in `subscription` module | ✅ **Accurate** |
| 8 | Prepaid Credit Ledger (blocks, cost basis, deduction order) | FEFO credits/prepaid balance design | ⚠️ **Recommendation** — QBill's FEFO uses 4-step deduction (priority→expiry→created→ID) vs Orb's 5-step (license→item-scoped→expiry→cost-basis→FIFO). The doc recommends Orb's deduction order. |
| 9 | Customer Balance (AR wallet) | Separate, simpler wallet | ⚠️ **Recommendation** — The doc suggests keeping this distinct from prepaid credits |
| 10 | Customer hierarchy (parent/child) | 4-level tenant hierarchy | ✅ **Accurate** — QBill's hierarchy is deeper |
| 11 | Invoice calculation pipeline (8 ordered steps) | Reusable as ordering contract | ⚠️ **Recommendation** — Not a factual claim about QBill's current architecture |

**Overall**: §13 concept map is **accurate** with 2 minor notes and 3 recommendations (all clearly labeled).

---

## 12. Identified Gaps & Action Items

### Critical Priority

| # | Gap | Orb Feature | Impact |
|---|---|---|---|
| **G-OR-10** | **No adjustment/discount/commitment system** | 5 adjustment types (usage discount, amount discount, % discount, minimum, maximum) with distribution rules and expiration | **Cannot offer discounts, commitments, or caps** — this is the #1 gap across ALL 5 competitor comparisons |

### High Priority

| # | Gap | Orb Feature | Impact |
|---|---|---|---|
| **G-OR-04** | **No cost-basis tracking on credits** | Per-block cost basis ($0=promo, >$0=purchased) driving revenue recognition | Cannot differentiate promotional vs purchased credits for ASC 606 |
| **G-OR-01** | **No dependency graph invalidation** | Tracks invoice-to-data dependencies; recalculates only affected fragments | Full period re-rating is computationally expensive at scale |
| **G-OR-08** | **No per-line-item tax** | Per-line-item tax via TaxJar/Avalara | Cannot handle per-product-type tax rates |

### Medium Priority

| # | Gap | Orb Feature | Suggested Fix |
|---|---|---|---|
| **G-OR-03** | **No per-event correction** | Amend/deprecate individual events | Add per-event correction capability |
| **G-OR-05** | **No post-tax customer balance** | After-tax AR adjustment for refunds/goodwill | Add post-tax balance mechanism |
| **G-OR-11** | **Anniversary-only billing** | 3 billing cycle options (calendar, anniversary, custom) | Add calendar-month billing |
| **G-OR-12** | **No mixed-cadence subscriptions** | Different cadences within one subscription | Support multi-cadence subscriptions (monthly usage + annual fee) |

### Low Priority

| # | Gap | Suggested Fix |
|---|---|---|
| **G-OR-02** | **Limited ingestion paths** | Add more event source integrations |
| **G-OR-06** | **No virtual currencies** | Add custom pricing unit support |
| **G-OR-09** | **No after-tax customer balance** | Add post-tax balance for refunds |

### Cumulative Gaps Across ALL 5 Competitors

| Gap | Count | FlexPrice | Lago | Meteroid | OpenMeter | Orb | Severity |
|---|---|---|---|---|---|---|---|
| **No discount/commitment pipeline** | **5/5** | ✅ | ❌ | ✅ | ✅ | ✅ | **Critical** |
| **No meter groupBy/filters** | **3/5** | ❌ | ✅ | ✅ | ✅ | ❌ | **High** |
| **No progressive billing** | **2/5** | ❌ | ✅ | ❌ | ✅ | ❌ | **High** |
| **No cost-basis on credits** | **1/5** | ❌ | ❌ | ❌ | ❌ | ✅ | **High** |
| **No plan phases** | **1/5** | ❌ | ❌ | ❌ | ✅ | ❌ | **Medium** |
| **No add-ons** | **1/5** | ❌ | ❌ | ❌ | ✅ | ❌ | **Medium** |

---

## 13. Source Cross-Reference

| QBill Source | Content Referenced |
|---|---|
| `BILLING_MATH.md` M-1 to M-6 | Money precision, rounding, invoice arithmetic |
| `BILLING_MATH.md` P-1 to P-5 | Proration rules (sub-windows, allowance, seats) |
| `BILLING_MATH.md` T-1 to T-6 | Time rules (UTC, anniversary, grace, half-open periods) |
| `BILLING_MATH.md` §7 | FEFO credit application |
| `BILLING_MATH.md` TR-1, TR-2 | Trials and recurring grants |
| `BILLING_MATH.md` W-1 to W-5 | Wallet burn, reconciliation, overdraft |
| `BILLING_MATH.md` C-1 to C-4 | Commit true-up |
| `BILLING_MATH.md` X-1, X-2 | Currency rules |
| `ADR-001` §2 | One-writer rule, usage pipeline |
| `ADR-001` §3 | Hybrid billing, rate resolution waterfall |
| `ADR-001` §3.3 | Rating waterfall (contract → rate card → plan → unrated) |
| `ADR-001` §3.4 | Invoice purity invariant |
| `ADR-001` §4 | Core requirements CR-1 through CR-14 |
| `phase_2_billing_worker.md` | Invoice lifecycle, enforcement API, acceptance criteria |
| `ERD.md` §3 | Catalog — meters, pricing models, rate cards, plans |
| `ERD.md` §4 | Billing entities — invoices, credits, wallet, revenue recognition |
| `DECISIONS.md` DEC-002 | Anti-spoofing (KeyContext authority) |
| `DECISIONS.md` DEC-010 | Enforcement latency contract |
| `engine/internal/invoice/generate.go` | Pure invoice function |
| `engine/internal/rating/resolver.go` | Rating waterfall with all 7 pricing models |
| `engine/internal/invoice/credits.go` | FEFO credit sorting and application |
| `engine/internal/wallet/` | Wallet operations (balance, burndown, topup, ledger, reconcile) |
| `engine/internal/revrec/` | Revenue recognition engine |

---

*End of document. Generated 2026-07-16.*
