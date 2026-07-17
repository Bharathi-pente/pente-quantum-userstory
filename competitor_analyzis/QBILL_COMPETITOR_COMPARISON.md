# QBill vs Competitors — Complete Comparison

**Date:** 2026-07-17
**Comparison Against:** FlexPrice · Lago · Meteroid · OpenMeter · Orb
**Goal:** For each competitor, what QBill does well (✅ GOOD) and what QBill is missing or does worse (❌ NOT GOOD)

---

## Table of Contents

1. [QBill vs FlexPrice](#1-qbill-vs-flexprice)
2. [QBill vs Lago](#2-qbill-vs-lago)
3. [QBill vs Meteroid](#3-qbill-vs-meteroid)
4. [QBill vs OpenMeter](#4-qbill-vs-openmeter)
5. [QBill vs Orb](#5-qbill-vs-orb)
6. [Consolidated Gap Summary](#6-consolidated-gap-summary)
7. [Final Verdict](#7-final-verdict)

---

## 1. QBill vs FlexPrice

### ✅ What QBill Does Well (GOOD)

| # | Area | QBill | FlexPrice | Why QBill Wins |
|---|---|---|---|---|
| 1 | **Pricing models** | **7 models** — FLAT, PER_UNIT, TIERED_GRADUATED, TIERED_VOLUME, PACKAGE, MATRIX, COST_PLUS | 3 models — Flat, Package, Volume Tiered | QBill has explicit GRADUATED vs VOLUME separation; MATRIX for LLM model×token pricing; COST_PLUS for BYOK markup |
| 2 | **Tiered pricing clarity** | Explicitly separates TIERED_GRADUATED (marginal) from TIERED_VOLUME (single-rate) | Ambiguous — "tiered" examples use graduated in some places, volume in others | QBill prevents billing surprises |
| 3 | **Money precision** | BILLING_MATH.md M-1 to M-6: 9dp internal, round-half-up, no float anywhere | No formal specification | QBill has audit-grade precision rules |
| 4 | **Invoice reproducibility** | **Byte-for-byte** pure function with versioned input snapshots | Deterministic but not byte-reproducible | QBill enables re-rating, simulation, test clocks |
| 5 | **Rating waterfall** | **Dual path**: product-led (plan → pricing_model) + sales-led (contract → rate_card) with 4-step waterfall | Single path (plan → price → feature) | QBill supports enterprise negotiated pricing |
| 6 | **Event dedup** | **First-write-wins** via Redis SETNX (explicit 409 on duplicate) | Latest-value-wins (silent overwrite) | QBill is audit-safe — no silent data loss |
| 7 | **Re-rating** | ✅ Full re-rating with credit notes (CR-1, CR-4) | ❌ Not mentioned | QBill has formal correction primitive |
| 8 | **Revenue recognition** | ✅ ASC 606/IFRS 15 (CR-5) | ❌ Not mentioned | QBill is enterprise-grade |
| 9 | **Batch ingest size** | **50,000 events/request** with partial accept | 1,000 events/request | QBill handles 50x larger batches |
| 10 | **Anti-spoofing** | ✅ DEC-002: KeyContext is the only authority — payload values never trusted | ❌ Payload `external_customer_id` is trusted | QBill is security-hardened |

### ❌ What QBill Needs to Improve (NOT GOOD)

| # | Area | FlexPrice | QBill | Gap Severity |
|---|---|---|---|---|
| 1 | **Discount/coupon pipeline** | ✅ **Full 3-step**: line-item coupons → subscription coupons (stackable) → wallet credits | ❌ **Not implemented** — no discount or coupon step exists | **🔴 CRITICAL** — every competitor has this |
| 2 | **Aggregation model variety** | 8 types including **SUM×multiplier** and **WEIGHTED SUM** | 7 types — missing WEIGHTED SUM for time-proportional billing | 🟡 High |
| 3 | **AI onboarding templates** | 3 pre-built templates (cost_tracking, team_budget, resale_markup) | ❌ Manual meter setup required | 🟡 High |
| 4 | **Standardized AI event shape** | Purpose-built `ai.usage` schema with input/output/cached/reasoning tokens | ❌ No standardized AI event shape documented | 🟢 Medium |
| 5 | **Multiple ingestion patterns** | Push (edge) + Managed pull (SaaS connectors) + Aggregate (CSV) | Single pattern: LiteLLM callback / direct API | 🟢 Medium |
| 6 | **Token re-pricing catalog** | Daily-refreshed model pricing repository | COST_PLUS model exists but no catalog automation | 🟢 Medium |

### Pipeline Comparison

```
FlexPrice Pipeline (7 steps):      QBill Pipeline (5 steps):
─────────────────────────────      ─────────────────────────
1. Subtotal                        1. Subtotal
2. Line-item coupon discounts      2. FEFO credits applied
3. Subscription coupon discounts   ❌ NO COUPON/DISCOUNT STEP
4. Prepaid wallet credits          (wallet is real-time, not invoice-time)
5. Taxable = subtotal − coupons    3. Taxable = subtotal − credits
6. Tax (multi-rate)                4. Tax (single rate)
7. Total = subtotal − coupons      5. Total = taxable + tax
         − wallet + tax
```

---

## 2. QBill vs Lago

### ✅ What QBill Does Well (GOOD)

| # | Area | QBill | Lago | Why QBill Wins |
|---|---|---|---|---|
| 1 | **Invoice reproducibility** | ✅ Byte-for-byte pure function with versioned input snapshots | ❌ Not guaranteed | QBill enables re-rating, simulation, audit |
| 2 | **Anti-spoofing** | ✅ DEC-002: KeyContext overwrites payload | ❌ Payload `external_subscription_id` trusted | QBill prevents event misattribution |
| 3 | **Event dedup scope** | ✅ `event_id + org_id` — org-scoped, prevents cross-tenant collisions | `transaction_id + external_subscription_id` — globally unique by convention only | QBill more secure |
| 4 | **COST_PLUS pricing** | ✅ Native markup for BYOK/gateway | ❌ Not supported | QBill unique advantage |
| 5 | **MATRIX pricing** | ✅ Dimension-based rate lookup for LLM use cases | ⚠️ Via filters + charge models (more config needed) | QBill simpler for LLM pricing |
| 6 | **Proration comprehensiveness** | ✅ Prorates base fee (P-2), allowance (P-3), AND seats (P-4) | Prorates first period only | QBill handles all plan changes |
| 7 | **Input snapshots on invoices** | ✅ Stores plan_version_id, rate_card_version_id, aggregation_watermark | ❌ Not stored | QBill enables audit trail |
| 8 | **Rate source tracking** | ✅ Every line records rate_source and rate_source_id | ❌ Not tracked | QBill has full rate provenance |
| 9 | **Typed line items** | ✅ 6 types (BASE_FEE, USAGE, OVERAGE, COMMIT_TRUE_UP, SEAT, ADJUSTMENT) | Per-charge (untyped) | QBill has richer semantics |

### ❌ What QBill Needs to Improve (NOT GOOD)

| # | Area | Lago | QBill | Gap Severity |
|---|---|---|---|---|
| 1 | **Filters/dimensions on metrics** | ✅ **First-class filters**: single metric sliced by model×type×region → independent pricing per combination | ❌ **Not supported** — requires N separate meters for N dimension combinations | **🔴 CRITICAL** — 30 meters vs 1 for 10 models × 3 types |
| 2 | **Dynamic pricing escape hatch** | ✅ `precise_total_amount_cents` — skip aggregation, supply exact amount | ❌ Not supported | 🟡 High |
| 3 | **Percentage charge model** | ✅ `amount × rate + fixed_fee` for payment processing | ❌ Not supported | 🟡 High |
| 4 | **WEIGHTED SUM aggregation** | ✅ Time-proportional sum for capacity billing | ❌ Not supported | 🟡 High |
| 5 | **Calendar billing option** | ✅ Calendar (1st to end-of-month) as default | ❌ Anniversary-only (T-3) | 🟡 High |
| 6 | **Minimum commitment** | ✅ Charge-level + plan-level spending minimums | ❌ Not implemented | 🟡 High |
| 7 | **Progressive billing** | ✅ Mid-cycle threshold invoicing | ❌ Not implemented | 🟡 High |
| 8 | **CUSTOM/SQL aggregation** | ✅ Custom SQL expressions (e.g., `tokens_in + tokens_out`) | ❌ Not supported | 🟢 Medium |
| 9 | **Recurring metrics** | ✅ `recurring: true` — persists across periods (seats, storage) | ❌ All meters period-resetting | 🟢 Medium |

### Lago's Signature Feature (QBill's Biggest Gap vs Lago)

**Filters on Billable Metrics** — Lago lets you define ONE metric with filters for model, type, region:

```json
"filters": [
  { "key": "model", "values": ["gpt-4", "gpt-3.5", "claude"] },
  { "key": "type",  "values": ["input", "output"] }
]
```

Then on a single charge, price each filter combination independently. QBill requires **separate meters** for each model×type combination. This is Lago's single biggest architectural advantage.

---

## 3. QBill vs Meteroid

### ✅ What QBill Does Well (GOOD)

| # | Area | QBill | Meteroid | Why QBill Wins |
|---|---|---|---|---|
| 1 | **Real-time enforcement** | ✅ **<5ms** at 32 concurrent callers (DEC-010) | ❌ Not built-in — period-end aggregation only | QBill has sub-5ms hot path |
| 2 | **Wallet** | ✅ Full wallet with Redis hot-path burndown, nightly reconciliation (W-4), bounded overdraft (W-5) | ❌ Not mentioned | QBill has real-time prepaid capability |
| 3 | **Revenue recognition** | ✅ ASC 606/IFRS 15 (CR-5) | ❌ Not mentioned | QBill enterprise-grade |
| 4 | **Re-rating** | ✅ Full correction loop with credit notes | ❌ Not built-in | QBill can correct historical periods |
| 5 | **Invoice reproducibility** | ✅ Byte-for-byte pure function | ❌ Period-end computation | QBill enables audit and simulation |
| 6 | **Input snapshots** | ✅ Versioned refs on every invoice | ❌ Not stored | QBill full audit trail |
| 7 | **Rate source tracking** | ✅ Per-line-item rate provenance | ❌ Not tracked | QBill rate waterfall transparency |
| 8 | **Anti-spoofing** | ✅ KeyContext authority (DEC-002) | ❌ Payload `customer_id` trusted | QBill security-hardened |
| 9 | **Batch size** | **50,000 events/request** | 100 events/request | QBill 500x larger batches |

### ❌ What QBill Needs to Improve (NOT GOOD)

| # | Area | Meteroid | QBill | Gap Severity |
|---|---|---|---|---|
| 1 | **Grouping key on metrics** | ✅ **First-class grouping** (e.g., by `project_id`) with per-group invoice lines and documented tiering implications | ❌ **Not supported** — each group requires separate subscription | **🔴 CRITICAL** for multi-tenant billing |
| 2 | **Segmentation system** | ✅ Formal segmentation (none/single/2D independent/2D dependent) | ❌ No explicit segmentation — relies on MATRIX at pricing level | 🟡 High |
| 3 | **Entitlements separate from billing** | ✅ **Fully separate**: Metered, Boolean entitlements with 5 reset strategies, independent from billing metrics | ❌ **Conflated**: `usage_limits` reference same `meter_id` used for billing | 🟡 High |
| 4 | **Floor price / minimum** | ✅ Plan/product-level minimum `max(calculated, floor)` | ❌ Not implemented (engine has per-model min but no invoice-level min) | 🟡 High |
| 5 | **Invoice preview** | ✅ "Upcoming Invoice" real-time recalculation | ❌ Not documented | 🟡 High |
| 6 | **Sliding window rate limiting** | ✅ 5 reset strategies including sliding window | ❌ Per-period only | 🟢 Medium |
| 7 | **Coupons** | ✅ Subscription-level fixed/percentage coupons | ❌ Not implemented | 🟢 Medium |
| 8 | **Slot-based pricing** | ✅ Dedicated seat/slot model with mid-period add/remove rules | ❌ No formal slot pricing (SEAT line type exists but less structured) | 🟢 Medium |

### Meteroid's Signature Feature (QBill's Biggest Gap vs Meteroid)

**Grouping Key + Segmentation** — Meteroid lets you define a single metric, segment it by `token_type` (input/output), and group results by `project_id`. Each group gets its own invoice sub-lines and independent tier evaluation. QBill requires separate meters and subscriptions for the same result.

---

## 4. QBill vs OpenMeter

### ✅ What QBill Does Well (GOOD)

| # | Area | QBill | OpenMeter | Why QBill Wins |
|---|---|---|---|---|
| 1 | **Rating waterfall** | ✅ **Dual path**: contract_rate → rate_card → pricing_model → unrated | ❌ Single path (plan → rate card) | QBill supports enterprise negotiated rates |
| 2 | **Contract commit amounts** | ✅ Commit true-up at term end (C-1 through C-4) | ❌ Not built-in | QBill has enterprise contract management |
| 3 | **Re-rating** | ✅ Full correction loop with credit notes | ❌ Not built-in | QBill can correct historical periods |
| 4 | **Wallet** | ✅ Real-time Redis wallet with nightly reconciliation | ❌ Grant-based only (no wallet) | QBill has real-time prepaid |
| 5 | **Real-time enforcement speed** | ✅ **<5ms** Redis hot path | ⚠️ API-based entitlement checking | QBill faster for enforcement |
| 6 | **Invoice reproducibility** | ✅ Byte-for-byte with versioned inputs | ❌ Not guaranteed | QBill enables audit |
| 7 | **FEFO credit determinism** | ✅ 4-level priority (compensation → promotional → prepaid → commit) | ⚠️ Grant-based (priority 0-255) | Both capable, QBill simpler |
| 8 | **COST_PLUS pricing** | ✅ Native BYOK markup | ✅ Dynamic pricing (cost-plus) | Tie — both support it |

### ❌ What QBill Needs to Improve (NOT GOOD)

| # | Area | OpenMeter | QBill | Gap Severity |
|---|---|---|---|---|
| 1 | **Meter groupBy** | ✅ **Mutable JSONPath groupBy** — single meter serves multiple features via `meterGroupByFilters` | ❌ **Not supported** — requires separate meters | **🔴 CRITICAL** — 30 meters vs 1 |
| 2 | **Discount/commitment pipeline** | ✅ Full 5-step: usage discount → pricing → % discount → min/max → tax | ❌ **Not implemented** | **🔴 CRITICAL** — every competitor has this |
| 3 | **Grant rollover configurability** | ✅ Configurable min/max rollover per grant | ❌ Non-rollover by default (TR-2), no override | 🟡 High |
| 4 | **Plan phases** | ✅ Multi-phase plans (trial → ramp-up → evergreen) | ❌ Single phase per plan | 🟡 High |
| 5 | **Add-ons** | ✅ Versioned, instance-controlled add-on catalog | ❌ Separate subscription required | 🟡 High |
| 6 | **Entitlement types** | ✅ 3 types: Metered (with grants), Boolean (on/off), Static (JSON config) | ❌ Usage limits only | 🟡 High |
| 7 | **Progressive billing** | ✅ Mid-cycle threshold invoicing | ❌ Not implemented | 🟡 High |
| 8 | **Feature abstraction layer** | ✅ Features decouple meters from plans (sellable unit) | ❌ Plans reference meters directly | 🟡 High |
| 9 | **Flat prices in tiers** | ✅ `flatPrice` per tier for overage-style pricing | ❌ Not supported — requires separate allowance + overage | 🟢 Medium |
| 10 | **Gathering invoice (preview)** | ✅ Live real-time "gathering" invoice | ❌ Not documented | 🟢 Medium |

### OpenMeter's Signature Features (QBill's Biggest Gaps vs OpenMeter)

**1. Meter groupBy + Features**: One meter (`tokens_total`) → multiple Features via `meterGroupByFilters` → independent pricing per filter combination.

**2. Grant system**: Configurable rollover (`minRolloverAmount`, `maxRolloverAmount`), recurrence independent of billing cycle, layered grants with 0-255 priority.

**3. Plan Phases + Add-ons**: Multi-phase plans (trial → ramp-up → evergreen) and versioned add-on catalog with instance control.

---

## 5. QBill vs Orb

### ✅ What QBill Does Well (GOOD)

| # | Area | QBill | Orb | Why QBill Wins |
|---|---|---|---|---|
| 1 | **Invoice reproducibility** | ✅ **Byte-for-byte** pure function with versioned snapshots | ✅ Query-based (same query → same result) | QBill has stronger binary guarantee |
| 2 | **Hot path latency** | ✅ **<5ms** Redis enforcement (DEC-010) | ⚠️ Seconds-latency stream pipeline | QBill faster for real-time enforcement |
| 3 | **Formal correction primitive** | ✅ Re-rating + credit notes with full state machine | ⚠️ Amend/deprecate/backfill (less formal) | QBill has auditable correction trail |
| 4 | **Customer hierarchy depth** | ✅ 4 levels (Org → Customer → End User), unlimited children | 2 levels (Parent → Child), max 100 children | QBill supports deeper org structures |
| 5 | **Batch ingest size** | **50,000 events/request** | 500 events/request | QBill 100x larger batches |
| 6 | **Dedup explicitness** | ✅ First-write-wins via SETNX (explicit 409) | Silent dedup within grace period | QBill more auditable |
| 7 | **Anti-spoofing** | ✅ DEC-002: KeyContext is the only authority | ❌ Payload `external_customer_id` trusted | QBill security-hardened |
| 8 | **Comprehensive proration** | ✅ Prorates base fee (P-2), allowance (P-3), AND seats (P-4) | Day-based base fee only | QBill more complete |

### ❌ What QBill Needs to Improve (NOT GOOD)

| # | Area | Orb | QBill | Gap Severity |
|---|---|---|---|---|
| 1 | **Adjustment/discount pipeline** | ✅ **5 types**: usage discount, amount discount, % discount, minimum, maximum — with proportional distribution and expiration | ❌ **Not implemented** | **🔴 CRITICAL** — every competitor has this |
| 2 | **Cost-basis tracking on credits** | ✅ **Per-block cost basis** ($0=promotional, >$0=purchased) driving revenue recognition | ❌ Not tracked — FEFO uses priority only | 🟡 High |
| 3 | **Dependency graph invalidation** | ✅ Tracks invoice-to-data dependencies → recalculates only affected fragments | ❌ Full period re-run required | 🟡 High |
| 4 | **Invoice pipeline steps** | 8 steps (most comprehensive) | 5 steps | Orb has richer pipeline |
| 5 | **Post-tax customer balance** | ✅ After-tax AR adjustment for refunds/goodwill (no rev-rec impact) | ❌ Not supported | 🟡 High |
| 6 | **Virtual/custom currencies** | ✅ Custom pricing units (e.g., "tokens" as currency) | ❌ Real currency only | 🟢 Medium |
| 7 | **Per-line-item tax** | ✅ Per-line-item via TaxJar/Avalara | ❌ Single rate, invoice-level | 🟢 Medium |
| 8 | **Calendar billing** | ✅ 3 options: Calendar, Anniversary, Custom | ❌ Anniversary only | 🟢 Medium |
| 9 | **Mixed-cadence subscriptions** | ✅ Different cadences in one subscription (monthly usage + annual fee) | ❌ Single cadence per subscription | 🟢 Medium |
| 10 | **Multiple ingestion paths** | 5+ paths: Segment, S3, Kinesis, Reverse ETL, API | LiteLLM + direct API only | 🟢 Medium |

### Orb's Signature Feature (QBill's Biggest Gap vs Orb)

**The 8-Step Invoice Pipeline** — Orb has the most comprehensive invoice pipeline of any competitor:

```
1. Quantity → 2. Subtotal → 3. Adjustments (5 types) → 4. Prepaid credits
→ 5. Currency conversion → 6. Partial invoice netting → 7. Tax → 8. Customer balance
```

QBill has only: `1. Subtotal → 2. Credits → 3. Taxable → 4. Tax → 5. Total`

Orb also has **cost-basis tracking on credits** — each credit block tracks its per-unit $ value for proper revenue recognition differentiation between promotional and purchased credits.

---

## 6. Consolidated Gap Summary

### 🔴 Critical Gaps (Present in Every Competitor, Missing in QBill)

| # | Gap | FlexPrice | Lago | Meteroid | OpenMeter | Orb | QBill |
|---|---|---|---|---|---|---|---|
| **1** | **Discount/commitment pipeline** | ✅ | ❌* | ✅ | ✅ | ✅ | ❌ |
| **2** | **Meter groupBy / filter system** | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ |

\* Lago doesn't have an explicit discount pipeline but has minimum commitment and charge-level spending minimums

### 🟡 High Priority Gaps (Present in 2+ Competitors)

| # | Gap | Competitors That Have It | QBill |
|---|---|---|---|
| **3** | **Entitlements separate from billing** | Meteroid, OpenMeter | ❌ Conflated |
| **4** | **Progressive billing** | Lago, OpenMeter | ❌ Not implemented |
| **5** | **Cost-basis on credits** | Orb | ❌ Not tracked |
| **6** | **Calendar-month billing** | Lago, Orb, Meteroid | ❌ Anniversary only |
| **7** | **Invoice preview** | Orb, Meteroid | ❌ Not documented |
| **8** | **WEIGHTED SUM aggregation** | FlexPrice, Lago | ❌ Not supported |
| **9** | **Plan phases** | OpenMeter, Orb | ❌ Single phase |
| **10** | **Add-on system** | OpenMeter | ❌ Not supported |

### 🟢 Medium Priority Gaps

| # | Gap | Competitors | QBill |
|---|---|---|---|
| **11** | Grant rollover configurability | OpenMeter | ❌ Non-rollover only |
| **12** | Sliding window rate limiting | Meteroid, OpenMeter | ❌ Per-period only |
| **13** | Dynamic pricing escape hatch | Lago (precise_total_amount_cents) | ❌ Not supported |
| **14** | Virtual/custom currencies | Orb, FlexPrice | ❌ Real currency only |
| **15** | AI onboarding templates | FlexPrice | ❌ Manual setup |
| **16** | Multiple ingestion paths | FlexPrice, Orb, Lago | ❌ LiteLLM only |

---

## 7. Final Verdict

### What QBill Does BEST (Competitive Advantages)

| Capability | Uniqueness | Competitors That Have It |
|---|---|---|
| **Pure-function byte-reproducible invoices** | QBill is the ONLY competitor with this | None |
| **Revenue recognition (ASC 606/IFRS 15)** | QBill is the ONLY competitor with this | None |
| **Real-time enforcement <5ms** | QBill is fastest (DEC-010 verified) | Orb (seconds), OpenMeter (API-based) |
| **Dual pricing paths (product-led + sales-led)** | QBill is the ONLY competitor with this | None — all have single path |
| **Test clocks for deterministic testing** | QBill is the ONLY competitor with this | None |
| **Re-rating with credit notes** | QBill has the most formal correction loop | Orb (backfill), Lago (manual) |
| **COST_PLUS pricing (BYOK markup)** | QBill + OpenMeter only | OpenMeter has Dynamic pricing |
| **Decimal math discipline** | QBill has the most rigorous spec (M-1 to M-6) | Most competitors less strict |

### What QBill needs to fix MOST URGENTLY

| Priority | What | Why |
|---|---|---|
| **#1 🔴** | **Discount & Commitment Pipeline** | Every single competitor has it. Without it you can't offer promotions, minimum commitments, or spending caps. |
| **#2 🔴** | **Meter groupBy / Filter System** | Lago, Meteroid, OpenMeter all have it. 10 models × 3 types = 30 meters instead of 1. |
| **#3 🟡** | **Separate Entitlements from Billing** | Meteroid and OpenMeter have clean separation. QBill conflates capping with billing. |
| **#4 🟡** | **Progressive Billing** | Lago and OpenMeter can invoice mid-cycle. QBill waits for period end. |
| **#5 🟡** | **Cost-Basis Tracking on Credits** | Orb tracks per-block cost basis for proper revenue recognition. QBill doesn't. |

### Final Scorecard

| Competitor | QBill Advantages | QBill Gaps | Overall |
|---|---|---|---|
| **vs FlexPrice** | 10 wins | 6 gaps | **QBill leads** — stronger architecture, pricing models, and audit capabilities |
| **vs Lago** | 9 wins | 9 gaps | **Close match** — QBill has better core engine; Lago has better meter management |
| **vs Meteroid** | 9 wins | 8 gaps | **QBill leads** — stronger real-time, wallet, and correction capabilities |
| **vs OpenMeter** | 8 wins | 10 gaps | **Tight race** — QBill has better core; OpenMeter has better entitlements, phases, add-ons |
| **vs Orb** | 8 wins | 10 gaps | **Tight race** — QBill has better latency; Orb has the best invoice pipeline |

---

*End of document. Generated 2026-07-17.*
