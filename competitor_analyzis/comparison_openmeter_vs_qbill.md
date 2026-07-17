# OpenMeter vs QBill — Detailed Comparison Analysis

**Date:** 2026-07-16
**Purpose:** Comprehensive comparison of OpenMeter's (now Kong Konnect Metering & Billing) metering, product catalog, and billing model against QBill's implementation.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Comparison](#2-architecture-comparison)
3. [Event Model & Metering](#3-event-model--metering)
4. [Product Catalog & Features](#4-product-catalog--features)
5. [Rate Cards & Pricing Models](#5-rate-cards--pricing-models)
6. [Entitlements & Grants — OpenMeter's Signature Feature](#6-entitlements--grants--openmeters-signature-feature)
7. [Subscriptions & Plan Phases](#7-subscriptions--plan-phases)
8. [Invoice Calculation & Progressive Billing](#8-invoice-calculation--progressive-billing)
9. [AI/LLM Token Billing](#9-aillm-token-billing)
10. [Identified Gaps & Action Items](#10-identified-gaps--action-items)
11. [Source Cross-Reference](#11-source-cross-reference)

---

## 1. Executive Summary

| Dimension | OpenMeter | QBill | Advantage |
|---|---|---|---|
| **Event format** | CloudEvents (CNCF standard) | Custom JSON schema | **OpenMeter** — standards-compliant |
| **Meter groupBy** | ✅ First-class (mutable JSONPath dimensions) | ❌ Not supported — requires separate meters | **OpenMeter** |
| **Features (sellable units)** | ✅ First-class entity separate from meters | ❌ No equivalent — products/plans directly reference meters | **OpenMeter** |
| **Entitlements** | ✅ 3 types (Metered, Boolean, Static) with grant-based balance | ⚠️ Usage limits tied to meters (conflated) | **OpenMeter** |
| **Grant system** | ✅ Full grant lifecycle with priority, rollover, recurrence | ✅ FEFO credits (4 priority levels) | Tie (different approaches) |
| **Plan phases** | ✅ Multi-phase plans (trial, ramp-up, reverse trial) | ❌ Not supported — single phase per plan | **OpenMeter** |
| **Add-ons** | ✅ First-class versioned add-on entities | ❌ Not supported — separate subscription needed | **OpenMeter** |
| **Progressive billing** | ✅ Mid-cycle threshold invoicing | ❌ Not implemented | **OpenMeter** |
| **Discount/commitment pipeline** | Usage discount → pricing → % discount → min/max → tax | Subtotal → FEFO credits → tax | **OpenMeter** — full pipeline |
| **Dynamic pricing** | ✅ Cost-plus/pass-through markup | ✅ COST_PLUS model | Tie |
| **Re-rating** | ❌ Not built-in | ✅ Full re-rating with credit notes (CR-1) | **QBill** |
| **Revenue recognition** | ❌ Not mentioned | ✅ ASC 606/IFRS 15 (CR-5) | **QBill** |
| **Test clocks** | ❌ Not mentioned | ✅ CR-12 | **QBill** |

---

## 2. Architecture Comparison

### OpenMeter Architecture
```
Event (CloudEvent) → Meter (aggregation) → Subject (who used it)
                                               ↓
                                          Customer (who pays)
                                               ↓
                          Feature (sellable unit) ← Rate Card (price + limit) → Plan (bundle)
                                               ↓
                                          Subscription (live instance)
                                               ↓
                                          Invoice (generated bill)
```

**Key characteristics:**
- **CloudEvents standard**: CNCF-standard event format (`specversion`, `type`, `id`, `source`, `subject`, `data`)
- **Subject-based metering**: Usage attributed to Subjects; Subjects mapped to Customers via Usage Attribution
- **Feature abstraction**: Sellable units are Features (metered or non-metered), decoupled from raw meters
- **Grants**: First-class credit/debit system with priority, rollover, and recurrence
- **Plan Phases**: Multi-phase plans for trials, ramp-ups, reverse trials
- **Add-ons**: Versioned, independently-priced extensions to base plans
- **Acquired by Kong**: Now rebranded as Kong Konnect Metering & Billing

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
- **Dual pricing paths**: Product-led + Sales-led with 4-step rating waterfall
- **Full correction loop**: Re-rating + credit notes
- **Wallet**: Real-time Redis-based prepaid burndown

### Key Architectural Differences

| Aspect | OpenMeter | QBill | Impact |
|---|---|---|---|
| **Event standard** | CloudEvents (CNCF) | Custom JSON | OpenMeter has broader ecosystem compatibility |
| **Identity model** | Subject → Customer (N:1 via Usage Attribution) | API Key → KeyContext (org → customer → end_user) | Different: OpenMeter is more flexible; QBill is more structured |
| **Sellable unit** | Feature (decoupled from meter) | Product/Plan references meters directly | OpenMeter's Feature abstraction enables cleaner reuse |
| **Entitlement mechanism** | Grants (priority, rollover, recurrence) | FEFO credits + wallet | Different: OpenMeter's grants are more granular |
| **Plan structure** | Multi-phase + add-ons | Single phase, no add-ons | OpenMeter supports trial-to-paid and ramp-up scenarios natively |
| **Invoice pipeline** | discount → pricing → % discount → min/max → tax | subtotal → FEFO credits → tax | OpenMeter has a richer discount/commitment pipeline |
| **Correction mechanism** | ❌ Not built-in | ✅ Re-rating + credit notes | **QBill** |

---

## 3. Event Model & Metering

### OpenMeter's CloudEvents Format

```json
{
  "specversion": "1.0",
  "type": "tokens",
  "id": "evt-001",
  "time": "2024-01-01T00:00:00.001Z",
  "source": "llm-gateway",
  "subject": "customer-1",
  "data": {
    "total_tokens": "1500",
    "model": "gpt-4",
    "type": "input"
  }
}
```

**Key fields:**
- `id`: Unique event ID (dedup key with `source`)
- `subject`: Who generated the usage (generic — user, device, service, org)
- `data`: Arbitrary payload with usage numbers and metadata

**Dedup**: `source + id` = unique key. Same event sent twice → processed once.

### OpenMeter's Meter groupBy (Key Differentiator)

```yaml
meters:
  - slug: tokens_total
    eventType: tokens
    valueProperty: $.total_tokens
    aggregation: SUM
    groupBy:
      model: $.model
      type: $.type
```

**Key properties:**
- `groupBy` is **mutable** — can be updated after creation without reprocessing
- Uses JSONPath for value extraction from event data
- Time window is a **query-time parameter**, not part of meter definition
- Every unique combination of `(time window, subject, groupBy values)` gets its own aggregate

**groupBy + Features**: A single meter with groupBy can feed multiple Features via `meterGroupByFilters`:
```js
Feature A: meterSlug='tokens_total', filter: { model: 'gpt-4', type: 'input' }
Feature B: meterSlug='tokens_total', filter: { model: 'gpt-4', type: 'output' }
```

### QBill's Meter Model

| Property | Description |
|---|---|
| `name` | Human-readable name |
| `event_type` | Event type to match |
| `aggregation` | SUM, COUNT, AVG, UNIQUE_COUNT, LATEST, MIN, MAX |
| `field` | Which property field to aggregate |
| `status` | DRAFT, ACTIVE, INACTIVE |

**No groupBy support** in QBill meters — each dimension combination requires a separate meter.

### Comparison

| Aspect | OpenMeter | QBill | Verdict |
|---|---|---|---|
| **Event standard** | CloudEvents (CNCF standard) | Custom JSON schema | **OpenMeter** — interoperable |
| **Dedup key** | `source + id` | `event_id + org_id` (SETNX) | Both valid |
| **Subject flexibility** | Generic (any entity type) | Fixed hierarchy (org → customer → end_user) | OpenMeter more flexible |
| **Usage attribution** | Subject → Customer (N:1) | API Key → KeyContext (strictly scoped) | Different approaches |
| **Meter groupBy** | ✅ Mutable JSONPath dimensions | ❌ Not supported | **OpenMeter** — single meter serves multiple pricing dimensions |
| **groupBy mutability** | ✅ Can add dimensions later | N/A | OpenMeter — retroactive dimension support |
| **Aggregation types** | SUM, COUNT, UNIQUE_COUNT, LATEST, MIN, MAX, AVG | SUM, COUNT, UNIQUE_COUNT, LATEST, MIN, MAX, AVG | Tie (identical set) |
| **Time window** | Query-time parameter | Subscription-period (anniversary) | Different: OpenMeter more flexible for ad-hoc queries |

### ✅ QBill Advantages

1. **Structured identity**: QBill's 4-level hierarchy (org → customer → end_user) provides clear billing ownership
2. **Anti-spoofing**: DEC-002 ensures KeyContext is the only authority for identity — no subject-mapping vulnerabilities
3. **Event dedup via SETNX**: First-write-wins is safer than OpenMeter's process-once dedup for billing accuracy

### ❌ QBill Gaps

**G-OM-01: No meter groupBy** — **HIGH PRIORITY** (same gap as Lago filters)

Without groupBy:
- Each model × token_type combination requires a separate meter
- Adding a new model requires a new meter definition
- Cannot use a single event stream for multiple pricing dimensions
- OpenMeter's approach: one meter, N features via `meterGroupByFilters`

**G-OM-02: No CloudEvents standard** — **LOW PRIORITY**

QBill uses a custom JSON event schema. Adopting CloudEvents would improve ecosystem compatibility.

---

## 4. Product Catalog & Features

### OpenMeter's Feature Abstraction

OpenMeter has a **Feature** entity as the sellable/governable unit — this is a critical abstraction layer that QBill lacks:

| Feature Type | Description | Example |
|---|---|---|
| **Metered** | Linked to a meter via `meterSlug` + `meterGroupByFilters` | "GPT-4 Input Tokens" |
| **Non-metered** | No usage tracking, toggle-only | "SAML SSO" |

**Feature decouples "what we track" from "what we sell":**
- One meter (`tokens_total`) → multiple Features (`gpt4_input_tokens`, `gpt4_output_tokens`, `claude_input_tokens`)
- Features can be **archived** (soft-deprecated) — existing entitlements keep working, no new ones created
- Feature key can be **reused** after archiving

### QBill's Catalog Model

QBill hierarchy:
```
Organization
  └── Products → Plans → Charges → PricingModels → PricingTiers
  └── Features (standalone, for entitlement)
  └── Meters (standalone, for measurement)
  └── RateCards → RateCardRates
  └── Contracts → ContractRates
  └── Subscriptions
```

**No Feature-mapping layer**: Products and Plans reference meters directly via Charges. There's no intermediate "Feature" that decouples measurement (meters) from commercialization (plans).

### Comparison

| Aspect | OpenMeter | QBill | Verdict |
|---|---|---|---|
| **Feature as abstraction** | ✅ First-class entity | ❌ No equivalent | **OpenMeter** — cleaner separation of concerns |
| **Meter-to-price binding** | Via Feature (meterSlug + groupBy filter) | Via Charge (plan → meter) | OpenMeter allows one meter to feed multiple prices |
| **Non-metered features** | ✅ Boolean and Static types | ⚠️ Features exist (in ERD) but are for entitlements, not pricing | OpenMeter more comprehensive |
| **Feature archival** | ✅ Soft-deprecation with key reuse | ❌ Not specified | OpenMeter |
| **Rate Card bundling** | ✅ Feature + Price + Entitlement in one rate card | ✅ Plan → Charges → PricingModels + separate usage_limits | Similar (different decomposition) |
| **Product bundling** | Features → Plans | Products → Plans → Charges | Different decomposition |

### ✅ QBill Advantages

1. **Dual pricing paths**: QBill supports both product-led (plans → charges) and sales-led (contracts → rate cards → contract rates) — OpenMeter has only plan-based pricing
2. **Rating waterfall**: 4-step resolution with contract-specific overrides — enterprise-grade

### ❌ QBill Gaps

**G-OM-03: No Feature abstraction layer** — **MEDIUM PRIORITY**

QBill lacks a Feature entity that decouples what you measure (meters) from what you sell (plans/prices). This means:
- Adding a new Feature requires touching both meter definitions and plan configurations
- Cannot archive/deprecate a Feature independently of its meter
- No clean way to offer the same usage metric under different names/descriptions across plans

---

## 5. Rate Cards & Pricing Models

### OpenMeter's Rate Card Structure

A **Rate Card** = Feature + Price + Entitlement (limit), as a line inside a Plan.

| Rate Card Component | Description |
|---|---|
| **Feature** | The sellable unit (metered or non-metered) |
| **Price** | Pricing model (Flat, Per-Unit, Tiered, Package, Dynamic) |
| **Entitlement** | Usage limit (e.g., 1,000,000 tokens/month) |
| **Billing cadence** | How often billed (e.g., `P1M` = monthly) |
| **Discounts** | Percentage discounts, usage discounts |
| **Min/max spend** | Commitment floor/ceiling |

**Free items**: 3 options — (a) omit price, (b) $0 price, (c) 100% discount. Only (a) allows subscribing without payment method.

### OpenMeter's Pricing Models (5 types)

| Model | Formula | QBill Equivalent |
|---|---|---|
| **Flat fee** (one-time or recurring) | Fixed amount | FLAT |
| **Per-unit** | `usage × unit_price` | PER_UNIT |
| **Tiered** (Graduated + Volume) | Graduated or Volume with optional flat prices per tier | TIERED_GRADUATED + TIERED_VOLUME |
| **Package** | `ceil(usage / packageSize) × packagePrice` | PACKAGE |
| **Dynamic** (cost-plus) | `meter_value × markup_rate` | COST_PLUS |

**Tier table with flat prices**: OpenMeter supports flat prices per tier — a critical feature for overage/included-allowance billing:
```
Tier 1 (0–1000): unitPrice $0, flatPrice $500
Tier 2 (1001–∞): unitPrice $0.1, flatPrice $0
```
This combines "first N units included" with "then pay per unit" in a single tier definition.

### QBill's Pricing Models (7 types)

| Model | Formula | OpenMeter Equivalent |
|---|---|---|
| FLAT | Fixed amount | Flat fee |
| PER_UNIT | `quantity × unitPrice` | Per-unit |
| TIERED_GRADUATED | `Σ(tier_units × tier_rate)` | Tiered (Graduated) |
| TIERED_VOLUME | `total × rate(tier)` | Tiered (Volume) |
| PACKAGE | `ceil(qty / packageSize) × packagePrice` | Package |
| MATRIX | Rate lookup by `(model, tokenType)` | ❌ Not in OpenMeter |
| COST_PLUS | `providerCost × (1 + markupPct)` | Dynamic (cost-plus) |

### Comparison

| Pricing Feature | OpenMeter | QBill | Verdict |
|---|---|---|---|
| Flat fee (recurring) | ✅ | ✅ | Tie |
| Per-unit | ✅ | ✅ | Tie |
| Tiered (graduated) | ✅ | ✅ TIERED_GRADUATED | Tie |
| Tiered (volume) | ✅ | ✅ TIERED_VOLUME | Tie |
| Package | ✅ | ✅ PACKAGE | Tie |
| Dynamic/cost-plus | ✅ | ✅ COST_PLUS | Tie |
| **Matrix (dimension-based)** | ❌ | ✅ **MATRIX** | **QBill** |
| **Flat prices in tiers** | ✅ **Flat price per tier** | ❌ Not supported | **OpenMeter** |
| **Overage via tier flat price** | ✅ Native (tier flat price) | ⚠️ Via P-3 allowance proration + separate overage line | Different approach |
| **Included allowance** | Via usage discount | Via P-3 prorated included units | Different mechanisms |
| **Percentage discount** | ✅ Applied after pricing | ❌ Not supported | **OpenMeter** |
| **Usage discount** | ✅ Deducted from quantity before pricing | ❌ Not supported | **OpenMeter** |
| **Min/max spend** | ✅ Per-line commitment | ❌ Not supported (only per-model min/max in resolver) | **OpenMeter** |

### ✅ QBill Advantages

1. **MATRIX pricing**: Native dimension-based pricing (model × token_type) — OpenMeter would need separate rate cards per dimension combination
2. **Pricing model count**: 7 vs 5 — QBill has greater variety

### ❌ QBill Gaps

**G-OM-04: No flat prices in tiers** — **MEDIUM PRIORITY**

OpenMeter's tier table with `flatPrice` enables clean "first N units included at flat fee, then per-unit" pricing. QBill requires:
1. Define included units as plan allowance (P-3)
2. Compute overage separately
3. This works but is more complex for simple use cases

**G-OM-05: No discount pipeline in pricing models** — **HIGH PRIORITY** (same as FlexPrice gap)

OpenMeter supports usage discounts (quantity reduction) and percentage discounts (amount reduction). QBill has neither at the pricing model level.

---

## 6. Entitlements & Grants — OpenMeter's Signature Feature

### OpenMeter's Grant System

OpenMeter has the most sophisticated credit/entitlement system among all competitors analyzed:

**Three entitlement types:**
| Type | Purpose |
|---|---|
| **Metered** | Usage limit + real-time balance tracking via grants |
| **Boolean** | Simple on/off feature flag |
| **Static** | Arbitrary per-customer JSON config |

**Grant lifecycle:**
```js
await openmeter.customers.entitlements.createGrant('customer-1', 'gpt_4_tokens', {
  amount: 100,
  priority: 1,
  effectiveAt: '2026-01-01T00:00:00Z',
  expiration: { duration: 'MONTH', count: 1 },
});
```

**Burn-down order:**
1. Priority (lower number = higher priority, range 0–255)
2. Closest to expiration
3. Oldest created

**Rollover configurability:**
```
Balance After Reset = MIN(maxRolloverAmount, MAX(Balance Before Reset, minRolloverAmount))
```
- `min = max = N` → topped up to exactly N each reset (recurring quota)
- `max = N`, no floor → bounded rollover
- `min = 0, max = 0` → no rollover (use it or lose it)

**Recurrence**: Independent top-up interval decoupled from entitlement reset (e.g., +300 tokens daily on top of monthly 5,000 quota)

**Key features:**
- `isSoftLimit`: hard-block or just warn
- `preserveOverageAtReset`: overage carries forward
- Historical balance queries (`burnDownHistory`, `windowedHistory`)
- Layered grants: high-priority grant burns first (e.g., free credits before paid)

### QBill's Credit System

**FEFO credits** (BILLING_MATH.md §7):
```
Sort: priority (ascending: compensation=0, promotional=1, prepaid=2, commit=3)
      → expires_at (ascending) → created_at (ascending) → ID
```

**Wallet** (real-time Redis burndown):
- Separate from FEFO credits
- Nightly reconciliation (W-4)
- Bounded overdraft (W-5)

**Limits**: `customer.usage_limits` (SOFT/HARD) + `customer.limit_overrides` — tied to meters

### Comparison

| Aspect | OpenMeter Grants | QBill FEFO Credits | Verdict |
|---|---|---|---|
| **Priority range** | 0–255 (any integer) | 4 fixed levels (0/1/2/3) | **OpenMeter** — more granular priority |
| **Rollover configurability** | ✅ Min/max per grant | ❌ Non-rollover by default (TR-2) | **OpenMeter** — configurable rollover |
| **Recurrence** | ✅ Independent top-up interval | ⚠️ Recurring grants at period start (TR-2) | OpenMeter — decoupled from billing cycle |
| **Expiration** | ✅ Per-grant expiration | ✅ Per-credit expires_at | Tie |
| **Soft/hard limits** | ✅ isSoftLimit flag | ✅ SOFT/HARD/WARNING | Tie |
| **Overage preservation** | ✅ `preserveOverageAtReset` | ❌ Not specified | **OpenMeter** |
| **Balance history** | ✅ `burnDownHistory`, `windowedHistory` | ✅ Credit ledger entries | Tie |
| **Layered grants** | ✅ Multiple grants per feature with different priorities | ✅ FEFO with 4 levels | Tie |
| **Entitlement types** | ✅ 3 types (Metered, Boolean, Static) | ⚠️ Usage limits only (no Boolean/Static) | **OpenMeter** |
| **Real-time balance** | ✅ Via entitlements API | ✅ Redis wallet (<5ms) + enforcement API | QBill faster |
| **Entitlement scope** | Per-feature, per-customer | Per-meter, per-customer (conflated) | **OpenMeter** — cleaner |
| **Static config** | ✅ JSON config per customer | ❌ Not supported | **OpenMeter** |

### ✅ QBill Advantages

1. **Real-time enforcement speed**: Redis-based <5ms hot path (DEC-010) vs OpenMeter's API-based entitlement checking
2. **Wallet**: Real-time prepaid burndown with nightly reconciliation — OpenMeter doesn't have a wallet
3. **Credit ledger**: Full immutable audit trail for all credit movements
4. **Bounded overdraft**: W-5 prevents runaway negative balances

### ❌ QBill Gaps

**G-OM-06: Fixed 4-level priority vs 0–255 range** — **LOW PRIORITY**

QBill's 4 fixed levels (compensation/promotional/prepaid/commit) are sufficient for most use cases. OpenMeter's 0–255 range offers more granularity for complex credit stacking.

**G-OM-07: No rollover configurability** — **MEDIUM PRIORITY**

OpenMeter's grants support configurable rollover (`minRolloverAmount`, `maxRolloverAmount`) per grant. QBill's recurring grants are non-rollover by default with no override.

**G-OM-08: No Boolean/Static entitlements** — **MEDIUM PRIORITY**

QBill only supports usage-based limits. OpenMeter's Boolean (on/off flags like SSO) and Static (JSON config like allowed models) entitlements are not present.

**G-OM-09: No overage preservation** — **LOW PRIORITY**

OpenMeter's `preserveOverageAtReset` prevents customers from escaping overage by waiting for period reset. QBill doesn't have this.

---

## 7. Subscriptions & Plan Phases

### OpenMeter's Plan Phases (Unique Feature)

OpenMeter supports **multi-phase plans** — a plan can have multiple time-ordered phases, each with its own rate cards:

| Phase Type | Description | Example |
|---|---|---|
| **Trial phase** | Free access for N days | 14-day trial with 100,000 tokens |
| **Ramp-up phase** | Graduated pricing over time | Month 1: 50% discount, Months 2–3: 25% |
| **Reverse trial** | Full access for N days, then reduced | Premium features for 14 days, then freemium |
| **Evergreen phase** | Runs indefinitely (final phase) | Standard pricing |

**Phase transitions** always land on period boundaries — no invoice spans two phases.

### OpenMeter's Add-Ons (Unique Feature)

**Add-ons** are standalone, versioned catalog items attachable to subscriptions:
- Composed of rate cards with own pricing and entitlements
- **Single-instance** (max 1 per subscription) or **Multi-instance** (multiple allowed)
- Add-on billing cadence must match the plan's cadence
- Compatibility locked at plan version publication

### QBill's Subscription Model

- Single phase per plan (no trial/ramp-up/reverse trial phases)
- No add-on entity — requires separate subscription for additional services
- Plan versioning: `plan_versions` with effective_from/effective_to
- Anniversary-anchored billing (T-3)
- Contract-based commit amounts with true-up at term end (C-1 through C-4)

### Comparison

| Aspect | OpenMeter | QBill | Verdict |
|---|---|---|---|
| **Plan phases** | ✅ Multi-phase (trial, ramp-up, reverse, evergreen) | ❌ Single phase | **OpenMeter** |
| **Add-ons** | ✅ Versioned, single/multi-instance | ❌ Separate subscription needed | **OpenMeter** |
| **Trial handling** | Via plan phases (trial phase) | ✅ trial_days on plan (TR-1) | Different approaches |
| **Plan versioning** | ✅ Draft → Published → Archived → Deleted | ✅ plan_versions with snapshots | Tie |
| **Subscription alignment** | ✅ All items billed together by default (opt-out staggered) | ✅ One subscription, one billing period | Tie |
| **Self-service + sales-led** | ✅ Both supported | ✅ Dual pricing paths (product-led + sales-led) | Tie |
| **Subscription editing** | ✅ Change items/entitlements going forward | ✅ Via contract_rates + plan changes | Tie |

### ✅ QBill Advantages

1. **Contract commit amounts**: QBill's commit true-up (C-1 through C-4) with term-end evaluation is enterprise-grade — OpenMeter has no equivalent
2. **Anniversary anchoring with end-of-month clamping**: T-3 handles Feb 28, leap years gracefully

### ❌ QBill Gaps

**G-OM-10: No plan phases** — **MEDIUM PRIORITY**

OpenMeter's multi-phase plans enable:
- Free trials that automatically convert to paid (without manual subscription change)
- Ramp-up discounts for new customers
- Reverse trials (full features for N days, then restriction)
QBill requires separate subscription changes for these scenarios.

**G-OM-11: No add-on system** — **MEDIUM PRIORITY**

QBill requires a separate subscription for add-on services. OpenMeter's add-on system provides:
- Versioned add-on catalog items
- Instance control (single vs multi)
- Compatibility locking
- Simplified customer management (one subscription with add-ons vs multiple subscriptions)

---

## 8. Invoice Calculation & Progressive Billing

### OpenMeter's Order of Operations

OpenMeter's invoice calculation pipeline (most detailed of all competitors):

```
1. Usage discount           → subtracted from metered quantity
2. Pricing algorithm         → rate card's price model applied
3. Percentage discount       → % taken off the resulting amount
4. Min / Max spend           → commitment floor/ceiling applied
5. Tax                       → calculated last, on post-discount total
```

**Usage discounts** only apply to usage-based lines. Under **Progressive Billing**, one usage discount pool can span multiple invoices — consumed down to zero across them.

**Progressive billing**: If enabled, OpenMeter invoices mid-cycle once accrued usage crosses a configured $ threshold. Reduces non-payment risk and improves cash flow.

**Gathering invoice**: OpenMeter maintains a live "gathering" invoice — a real-time preview of pending charges for the current cycle.

### QBill's Invoice Pipeline (BILLING_MATH.md M-5)

```
1. Subtotal = Σ(line items)
2. Credits_applied = min(subtotal, available FEFO credits)
3. Taxable = subtotal − credits_applied
4. Tax = round_half_up(taxable × rate, minor_units)
5. Total = taxable + tax
```

**Grace window**: Draft → finalize after `INVOICE_GRACE_HOURS` (default 36h) (T-5)
**State machine**: draft → pending → paid/overdue/voided
**Input snapshots**: `plan_version_id`, `rate_card_version_id`, `aggregation_watermark`

### Comparison

| Aspect | OpenMeter | QBill | Verdict |
|---|---|---|---|
| **Pipeline steps** | 5 (discount → pricing → % discount → min/max → tax) | 5 (subtotal → credits → taxable → tax → total) | Different pipelines |
| **Usage discounts** | ✅ Subtract from quantity before pricing | ❌ Not supported | **OpenMeter** |
| **Percentage discounts** | ✅ After pricing | ❌ Not supported | **OpenMeter** |
| **Min/max spend** | ✅ Per-line commitment | ❌ Not supported | **OpenMeter** |
| **Progressive billing** | ✅ Mid-cycle threshold invoicing | ❌ Not implemented | **OpenMeter** |
| **Gathering invoice** | ✅ Live real-time preview | ❌ Not documented | **OpenMeter** |
| **FEFO credits** | ❌ Via grants (different mechanism) | ✅ 4-level priority ordering | **QBill** |
| **Input snapshots** | ❌ Not stored | ✅ Versioned refs on invoice | **QBill** |
| **Rate source tracking** | ❌ Not tracked | ✅ Per-line-item rate provenance | **QBill** |
| **Tax** | Last step, after all discounts | After credits | Different ordering |

### ✅ QBill Advantages

1. **Input snapshots**: Versioned inputs enable re-rating (CR-1) and simulation (CR-9)
2. **Rate source tracking**: Every line item records its rate provenance
3. **FEFO credit ordering**: Deterministic 4-level credit consumption
4. **Grace window**: Configurable draft → finalize window (T-5)

### ❌ QBill Gaps

**G-OM-12: No progressive billing** — **HIGH PRIORITY** (same gap across multiple competitors)

OpenMeter's progressive billing invoices mid-cycle when usage crosses a threshold. QBill only invoices at period boundaries, creating:
- Cash flow risk: high-usage customers are not billed until period end
- Credit risk: can't throttle mid-cycle based on accumulated charges

**G-OM-13: No discount pipeline** — **HIGH PRIORITY** (same gap across all competitors)

OpenMeter has a full discount pipeline (usage discount → pricing → % discount → min/max). QBill has none.

**G-OM-14: No gathering/invoice preview** — **MEDIUM PRIORITY**

OpenMeter maintains a live gathering invoice. QBill only generates at period boundaries.

---

## 9. AI/LLM Token Billing

### OpenMeter's Approach

**One meter** with `groupBy` for model and type:
```yaml
meters:
  - slug: tokens_total
    eventType: tokens
    valueProperty: $.total_tokens
    aggregation: SUM
    groupBy:
      model: $.model
      type: $.type
```

**Multiple Features** via `meterGroupByFilters`:
```js
Feature: 'gpt4_input_tokens' → filter: { model: 'gpt-4', type: 'input' }
Feature: 'gpt4_output_tokens' → filter: { model: 'gpt-4', type: 'output' }
```

**Grants** for included allowances:
```js
Grant: 1,000,000 tokens/month, priority 1, recurring
Grant: purchased tokens, priority 2, expires in 1 year
```

**Entitlement check**: `hasAccess, balance, usage, overage` in real-time via API

### QBill's Approach

- **Separate meters** per model × token_type
- **MATRIX pricing** for per-model × per-token-type rates
- **Rating waterfall** for contract-specific negotiated rates
- **FEFO credits** for credit application at invoice time
- **Wallet** for real-time prepaid burndown

### Comparison

| Aspect | OpenMeter | QBill | Verdict |
|---|---|---|---|
| **Meter count for N models × M types** | 1 meter | N × M separate meters | **OpenMeter** — single event pipeline |
| **Per-model pricing** | Via Features (meterGroupByFilter) | Via MATRIX model or separate meters | OpenMeter cleaner for metering |
| **Included allowances** | Grants (priority, rollover, recurrence) | P-3 prorated included units + FEFO credits | Different mechanisms |
| **Real-time balance** | ✅ Entitlements API | ✅ Redis wallet (<5ms) | QBill faster |
| **Grant rollover** | ✅ Configurable | ❌ Non-rollover by default | **OpenMeter** |
| **Contract-specific rates** | ❌ Via subscription overrides (limited) | ✅ Full rating waterfall | **QBill** |
| **BYOK markup** | ✅ Dynamic pricing (cost-plus) | ✅ COST_PLUS model | Tie |
| **Re-rating** | ❌ Not built-in | ✅ Full re-rating loop | **QBill** |
| **Enforcement** | ✅ Blocking via entitlement balance | ✅ Redis counters + wallet (<5ms) | Tie |

### ✅ QBill Advantages

1. **Rating waterfall**: Contract-specific negotiated rates — OpenMeter requires subscription-level overrides
2. **MATRIX pricing**: Native dimension-based pricing for LLM use cases
3. **Re-rating**: Full correction loop with credit notes — OpenMeter has no equivalent
4. **Wallet**: Real-time prepaid burndown with nightly reconciliation

### ❌ QBill Gaps

1. **No groupBy on meters**: Same gap (G-OM-01) — requires N×M meters instead of 1
2. **No grant rollover configurability**: Same gap (G-OM-07)
3. **No plan phases for trials**: Same gap (G-OM-10)

---

## 10. Identified Gaps & Action Items

### High-Priority Gaps

| # | Gap | OpenMeter Feature | Impact | Suggested Fix |
|---|---|---|---|---|
| **G-OM-01** | **No meter groupBy** | Mutable JSONPath groupBy dimensions on meters | Requires separate meters for each model×type combination → operational overhead, no retroactive pricing | Add `groupBy` field to meter definitions in catalog module; extend engine `rateSegmentUsage()` to group by configurable dimensions |
| **G-OM-05** | **No discount pipeline** | Usage discount + % discount + min/max spend in invoice pipeline | No promotional discounts, no volume commitments, no spending caps | Add discount/commitment step to `invoice.Generate()` pipeline (same gap across all competitors) |
| **G-OM-12** | **No progressive billing** | Mid-cycle threshold invoicing | Cash flow risk for high-usage customers | Add threshold-based mid-cycle invoice generation |
| **G-OM-13** | **No discount pipeline** | Full 5-step discount/commitment pipeline | (Duplicate of G-OM-05) | — |

### Medium-Priority Gaps

| # | Gap | OpenMeter Feature | Suggested Fix |
|---|---|---|---|
| **G-OM-03** | **No Feature abstraction layer** | Feature entity decoupled from meters | Add Feature entity that links meters (via groupBy filters) to plans (via rate cards) |
| **G-OM-07** | **No grant rollover configurability** | Configurable min/max rollover per grant | Add rollover configuration to recurring grants |
| **G-OM-08** | **No Boolean/Static entitlements** | Boolean and Static entitlement types | Add on/off feature flags and per-customer JSON config |
| **G-OM-10** | **No plan phases** | Multi-phase plans (trial, ramp-up, reverse trial) | Add time-ordered phases to plan versions |
| **G-OM-11** | **No add-on system** | Versioned, instance-controlled add-on catalog | Add add-on entity to catalog module |
| **G-OM-14** | **No gathering/invoice preview** | Live real-time invoice preview | Add "upcoming invoice" preview endpoint |

### Low-Priority Gaps

| # | Gap | Suggested Fix |
|---|---|---|
| **G-OM-02** | **No CloudEvents standard** | Consider adopting CloudEvents for broader ecosystem compatibility |
| **G-OM-04** | **No flat prices in tiers** | Add `flatPrice` to tier definitions in pricing models |
| **G-OM-06** | **Fixed 4-level priority** | Consider expanding to broader priority range (0–255) |
| **G-OM-09** | **No overage preservation** | Add `preserveOverageAtReset` option to recurring grants |

---

## 11. Source Cross-Reference

| QBill Source | Content Referenced |
|---|---|
| `BILLING_MATH.md` M-1 to M-6 | Money precision, rounding, invoice arithmetic |
| `BILLING_MATH.md` P-1 to P-5 | Proration rules (sub-windows, allowance, seats) |
| `BILLING_MATH.md` T-1 to T-6 | Time rules (UTC, anniversary, grace, half-open periods) |
| `BILLING_MATH.md` §7 | FEFO credit application |
| `BILLING_MATH.md` TR-1, TR-2 | Trials and recurring grants |
| `BILLING_MATH.md` W-1 to W-5 | Wallet burn, reconciliation, overdraft |
| `BILLING_MATH.md` C-1 to C-4 | Commit true-up |
| `ADR-001` §3 | Hybrid billing, rate resolution waterfall |
| `ADR-001` §3.3 | Rating waterfall (contract → rate card → plan → unrated) |
| `ADR-001` §3.4 | Invoice purity invariant |
| `phase_2_billing_worker.md` | Invoice lifecycle, enforcement API, acceptance criteria |
| `ERD.md` §2 | Customer domain — usage_limits, limit_overrides, usage_summary |
| `ERD.md` §3 | Catalog — meters, pricing models, rate cards, plans, features |
| `ERD.md` §4 | Billing entities — invoices, credits, wallet |
| `DECISIONS.md` DEC-002 | Anti-spoofing (KeyContext authority) |
| `DECISIONS.md` DEC-010 | Enforcement latency contract |
| `engine/internal/invoice/generate.go` | Pure invoice function |
| `engine/internal/rating/resolver.go` | Rating waterfall with all 7 pricing models |
| `engine/internal/invoice/credits.go` | FEFO credit sorting and application |
| `engine/internal/enforcement/` | Usage limit enforcement |
| `engine/internal/wallet/` | Wallet operations |

---

*End of document. Generated 2026-07-16.*
