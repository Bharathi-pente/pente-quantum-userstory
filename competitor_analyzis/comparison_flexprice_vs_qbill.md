# FlexPrice vs QBill — Detailed Comparison Analysis

**Date:** 2026-07-16
**Purpose:** Comprehensive comparison of FlexPrice's metering & billing calculation model against QBill's implementation, documenting what QBill does well, where it trails, and actionable gaps.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Comparison](#2-architecture-comparison)
3. [Event Ingestion & Dedup](#3-event-ingestion--dedup)
4. [Aggregation Models](#4-aggregation-models)
5. [Pricing Models](#5-pricing-models)
6. [Invoice Calculation Pipeline](#6-invoice-calculation-pipeline)
7. [Proration Methodology](#7-proration-methodology)
8. [Wallet & Credit Systems](#8-wallet--credit-systems)
9. [Tax Calculation](#9-tax-calculation)
10. [AI/LLM Token Billing](#10-aillm-token-billing)
11. [Identified Gaps & Action Items](#11-identified-gaps--action-items)
12. [Source Cross-Reference](#12-source-cross-reference)

---

## 1. Executive Summary

| Dimension | FlexPrice | QBill | Advantage |
|---|---|---|---|
| **Pricing models** | 3 (Flat, Package, Volume Tiered) | **7** (FLAT, PER_UNIT, TIERED_GRADUATED, TIERED_VOLUME, PACKAGE, MATRIX, COST_PLUS) | **QBill** |
| **Coupon/discount pipeline** | ✅ Full 3-step discount system | ❌ **Not implemented** | **FlexPrice** |
| **Money precision** | No formal specification | ✅ BILLING_MATH.md M-1 to M-6 (9dp, round-half-up, no float) | **QBill** |
| **Invoice reproducibility** | Deterministic but not byte-reproducible | ✅ Pure function (ADR-001 §3.4) with versioned input snapshots | **QBill** |
| **Aggregation types** | 8 types (incl. SUM×multiplier, WEIGHTED SUM) | 7 types (missing WEIGHTED SUM) | **FlexPrice** (minor) |
| **AI onboarding** | 3 pre-built templates | Manual meter setup | **FlexPrice** |
| **Rate resolution** | Single path only | ✅ Dual path (product-led + sales-led) with 4-step waterfall | **QBill** |
| **Re-rating** | ❌ Not mentioned | ✅ Full re-rating with credit notes (CR-1, CR-4) | **QBill** |
| **Revenue recognition** | ❌ Not mentioned | ✅ ASC 606/IFRS 15 (CR-5) | **QBill** |

---

## 2. Architecture Comparison

### FlexPrice Architecture
```
Event → Feature (aggregation) → Price (billing model) → Plan → Subscription → Invoice
```

- **Single pipeline**: events flow through features (aggregation) → prices (rating) → plans (bundling) → subscriptions (customer binding) → invoices
- **Deterministic math**: all downstream calculation is deterministic, but no byte-for-byte reproducibility guarantee
- **SaaS platform**: FlexPrice is a SaaS platform with managed infrastructure (collector agents, managed pull connectors)

### QBill Architecture
```
LiteLLM → Go Ingest API → Kafka → ClickHouse (source of truth)
                                           ↓
                              Go Billing Worker → Redis (enforcement cache)
                                           ↓
                              invoice.Generate() ← pure function
                                           ↓
                              Postgres billing schema
                                           ↓
                              NestJS BFF (read-only presenter)
```

- **Two-tier architecture**: Go engine (sole calculation authority) + NestJS BFF (read/present/forward)
- **Pure function core**: `invoice.Generate(Inputs)` is a pure function — byte-for-byte reproducibility
- **Versioned snapshots**: every invoice stores `plan_version_id`, `rate_card_version_id`, `aggregation_watermark` for audit
- **One-writer rule**: Go engine writes financial artifacts; NestJS writes control-plane config only

### Key Architectural Differences

| Aspect | FlexPrice | QBill | Impact |
|---|---|---|---|
| **Invoice engine** | Deterministic pipeline | Pure function with versioned inputs | QBill's approach enables re-rating (CR-1), simulation (CR-9), and test clocks (CR-12) |
| **Event storage** | FlexPrice-managed (not specified) | ClickHouse `events.usage_events` (immutable, ReplacingMergeTree) | QBill's immutable event log is more auditable |
| **Rate source** | Single: plan → price | Dual: contract_rate → rate_card_version → pricing_model → unrated | QBill supports both self-serve and enterprise negotiated pricing |
| **Data ownership** | FlexPrice-owned | Self-hosted (Postgres, ClickHouse, Redis, Kafka) | QBill gives customers full data control |

---

## 3. Event Ingestion & Dedup

### FlexPrice Approach

**Event Shape:**
```json
{
  "event_name": "model.usage",
  "external_customer_id": "cust_123",
  "properties": { "credits": 2, "model": "gpt-4" },
  "event_id": "evt_abc123",
  "timestamp": "2025-08-22T07:05:49.441Z",
  "source": "api"
}
```

**Dedup: Latest-value-wins**
- Same `event_id` sent twice → **keeps the latest value** (not a running total)
- This means retries can OVERWRITE data if the values differ
- Example: `evt_001` first with `credits=1000`, re-sent with `credits=800` → final value is 800

**Batch:** Up to 1,000 events/request, 100 req/min
**Response:** 202 Accepted (async aggregation)

### QBill Approach

**Event Shape:**
```json
{
  "event_name": "ai.usage",
  "customer_id": "cust_uuid",
  "end_user_id": "end_user_uuid",
  "properties": { "model": "gpt-4", "tokens": 1500 },
  "event_id": "req_uuid",
  "timestamp": "2026-07-16T10:30:00Z",
  "source_mode": "virtual_key"
}
```

**Dedup: First-write-wins (SETNX)**
- Same `event_id` + `org_id` → **first write wins**, duplicate returns `409 DUPLICATE_EVENT`
- Idempotency via Redis `SETNX idem:{org_id}:{event_id}` with 24h TTL
- No silent data overwrite — explicit rejection of duplicates

**Batch:** Up to 50,000 events/request, partial accept on per-index errors
**Response:** 202 Accepted (async to Kafka)

### Comparison

| Aspect | FlexPrice | QBill | Verdict |
|---|---|---|---|
| **Dedup behavior** | Latest-value-wins (silent overwrite) | First-write-wins (explicit 409) | **QBill safer** — no silent data loss |
| **Batch size** | 1,000/request | 50,000/request | **QBill** — 50x larger batches |
| **Partial accepts** | ❌ Not mentioned | ✅ Yes (per-index errors) | **QBill** — graceful degradation |
| **Anti-spoofing** | ❌ Payload `external_customer_id` trusted | ✅ KeyContext overwrites payload (DEC-002) | **QBill** — security-hardened |
| **Idempotency** | Latest timestamp wins | SETNX first-write-wins | **QBill safer** |

### ✅ QBill Advantage

QBill's dedup model is **strictly safer** for billing:
- No risk of silent data loss from retries with different values
- 24h TTL on idempotency keys covers the full grace window
- Anti-spoofing via DEC-002 (KeyContext is the only authority for org/customer attribution)

### ❌ QBill Gap

QBill has no equivalent of FlexPrice's **managed pull ingestion** (SaaS connectors for Langfuse, Databricks, Bedrock). Currently only supports LiteLLM callback / direct API push.

---

## 4. Aggregation Models

### FlexPrice (8 Types)

| Type | Formula | Notes |
|---|---|---|
| COUNT | `COUNT(DISTINCT events.id)` | No field needed |
| SUM | `SUM(properties.field)` | Straight total |
| AVERAGE | `AVG(properties.field)` | Mean across events |
| COUNT UNIQUE | `COUNT(DISTINCT properties.field)` | Unique values |
| LATEST | `argMax(properties.field, timestamp)` | Most recent wins |
| **SUM WITH MULTIPLIER** | `SUM(properties.field) × multiplier` | Sum first, multiply once |
| MAX | `MAX(properties.field)` | Peak value |
| **WEIGHTED SUM** | Time-proportional sum, prorated by duration | Capacity/reservation billing |

### QBill (7 Types, in engine)

| Type | Formula | Notes |
|---|---|---|
| SUM | `SUM(properties.field)` | Straight total |
| COUNT | `COUNT(DISTINCT events.id)` | Event count |
| AVG | `AVG(properties.field)` | Mean |
| UNIQUE_COUNT | `COUNT(DISTINCT properties.field)` | Unique values |
| LATEST | `argMax(properties.field, timestamp)` | Most recent |
| MIN | `MIN(properties.field)` | Minimum value |
| MAX | `MAX(properties.field)` | Peak value |

### Gap Analysis

| Missing in QBill | FlexPrice Equivalent | Use Case | Severity |
|---|---|---|---|
| **SUM WITH MULTIPLIER** | `SUM(field) × multiplier` | Token billing: `SUM(tokens) × 0.001` (per-1K tokens) | **Low** — QBill can achieve same via PER_UNIT pricing with unit price = multiplier |
| **WEIGHTED SUM** | Time-proportional sum | Capacity billing (e.g. "GB-provisioned × hours") | **Medium** — needed for infrastructure/resource-based billing |

### ✅ QBill Has But FlexPrice Doesn't

| QBill Feature | Use Case |
|---|---|
| **MIN** aggregation | Minimum concurrent usage, resource floor tracking |
| **MATRIX pricing** | Per-model × per-token-type pricing (LLM-native) |
| **COST_PLUS pricing** | BYOK/gateway markup pricing |

---

## 5. Pricing Models

### FlexPrice (3 Models)

| Model | Formula | Example |
|---|---|---|
| **Flat Fee** | `charge = quantity × unit_price` | $0.05 per API call |
| **Package** | Fixed price per usage band | $50 for 1–1,000 messages |
| **Volume Tiered** | Single rate for ALL units at the tier the total falls in | 65,000 units × $0.0006 (tier 3 rate) |

**Note:** FlexPrice's "tiered" examples use graduated interpretation in some places and volume in others. The doc itself flags this ambiguity.

### QBill (7 Models, in `internal/rating/resolver.go`)

| Model | Formula | Notes |
|---|---|---|
| **FLAT** | Fixed amount | Identical to FlexPrice |
| **PER_UNIT** | `quantity × pricePerUnit` | Identical to FlexPrice's Flat Fee |
| **TIERED_GRADUATED** | `Σ(units_in_tier × tier_rate)` per bracket | **Different from FlexPrice** — marginal, like tax brackets |
| **TIERED_VOLUME** | `total_qty × rate(tier_containing_total)` | Same as FlexPrice's Volume Tiered |
| **PACKAGE** | `ceil(qty / packageSize) × packagePrice` | Same as FlexPrice's Package |
| **MATRIX** | Rate lookup by `(model, tokenType)` | **QBill unique** — no FlexPrice equivalent |
| **COST_PLUS** | `providerCost × (1 + markupPct)` | **QBill unique** — no FlexPrice equivalent |

### Comparison Matrix

| Pricing Feature | FlexPrice | QBill | Advantage |
|---|---|---|---|
| Flat rate | ✅ | ✅ | Tie |
| Per-unit | ✅ (as Flat Fee) | ✅ (as PER_UNIT) | Tie |
| Package/block | ✅ | ✅ | Tie |
| Volume tiered | ✅ (called "Tiered") | ✅ (TIERED_VOLUME, explicit) | **QBill** — explicit naming avoids ambiguity |
| Graduated tiered | ❌ (ambiguous) | ✅ (TIERED_GRADUATED, explicit) | **QBill** |
| Matrix (dimension-based) | ❌ | ✅ (MATRIX) | **QBill** — LLM-native pricing |
| Cost-plus / markup | ❌ | ✅ (COST_PLUS) | **QBill** — BYOK-native pricing |
| Per-seat | ❌ | ✅ (SEAT line type) | **QBill** |
| Min/max clamping | ❌ | ✅ (MinimumAmount/MaximumAmount) | **QBill** |
| Free tier / included allowance | ✅ | ✅ (BILLING_MATH.md P-3) | Tie |
| Advance vs arrears | ✅ | ✅ (phase_2 criteria) | Tie |

---

## 6. Invoice Calculation Pipeline

### FlexPrice Pipeline (7 Steps)

```
Step 1:  Subtotal = Σ(line items)
Step 2:  Line-item coupon discounts     ← applied per line
Step 3:  Subscription coupon discounts   ← stacked, sequential
Step 4:  Prepaid wallet credits          ← after all coupons
Step 5:  Taxable = subtotal − coupons    ← wallet does NOT reduce tax base
Step 6:  Tax = Σ(rate × taxable)        ← non-compounding, multi-rate
Step 7:  Total = subtotal − coupons − wallet + tax
```

**Key rule:** `MAX(subtotal − coupons, 0)` — coupons can never make taxable base negative.

### QBill Pipeline (5 Steps, per BILLING_MATH.md M-5)

```
Step 1:  Subtotal = Σ(line items)
Step 2:  FEFO credits applied           ← credits_applied = min(subtotal, available_FEFO_credits)
Step 3:  Taxable = subtotal − credits_applied  ← credits DO reduce tax base
Step 4:  Tax = round_half_up(taxable × rate, minor_units)
Step 5:  Total = taxable + tax
```

**Key rule:** Negative totals impossible — credits cap at `subtotal`.

### Side-by-Side Pipeline

```
Step   | FlexPrice                     | QBill
───────┼───────────────────────────────┼──────────────────────────────
1      | Subtotal                      | Subtotal
2      | Line-item coupons             | FEFO credits
3      | Subscription coupons (stacked)| (nothing — credits already applied)
4      | Wallet credits                | (nothing — wallet is real-time Redis, not invoice-time)
5      | Taxable = subtotal − coupons  | Taxable = subtotal − credits
6      | Tax (multi-rate, non-compound)| Tax (single rate, invoice-level)
7      | Total = subtotal − coupons −  | Total = taxable + tax
       |         wallet + tax          |
```

### Critical Differences

| Aspect | FlexPrice | QBill | Impact |
|---|---|---|---|
| **Coupon/discount support** | ✅ Full pipeline (line-item + subscription level) | ❌ **None** | **FlexPrice wins** — QBill has no discount mechanism |
| **Wallet vs tax base** | Wallet does NOT reduce taxable base | Credits DO reduce taxable base | FlexPrice's approach generates higher tax revenue (tax on full subtotal minus coupons) |
| **Tax granularity** | Multi-rate, non-compounding (per line item) | Single rate, invoice-level | FlexPrice handles jurisdictions with multiple concurrent tax rates (state + federal) |
| **Credit application** | After coupons, before tax | After subtotal, before tax | Different ordering — QBill applies credits first, FlexPrice applies coupons first |
| **Negative totals** | Prevented via MAX(0) on taxable | Impossible (credits cap at subtotal) | Both safe, different mechanisms |

### ✅ QBill Advantages in Pipeline

1. **Simplicity**: 5 steps vs FlexPrice's 7 — fewer moving parts
2. **FEFO determinism**: Credit consumption is deterministic (priority → expires_at → created_at → ID)
3. **No rounding accumulation**: Rounding happens once per line item (M-3), then sums are exact

### ❌ QBill Gaps in Pipeline

1. **No coupon/discount system**: **CRITICAL GAP**. FlexPrice has line-item coupons, subscription coupons, and wallet credits all in the pipeline. QBill has none of these.
2. **No multi-rate tax**: QBill computes tax at invoice level with a single tax rate. Some jurisdictions require per-line-item tax with different rates per product type.
3. **Wallet integration**: QBill's wallet burns down in real-time on Redis (hot path), not at invoice time. This means wallet credits and FEFO credits are separate systems that don't interact in the invoice pipeline.

---

## 7. Proration Methodology

### FlexPrice

**Formula:**
```
Credit (old plan) = (days_remaining / total_days) × old_plan_price
Charge (new plan) = (days_remaining / total_days) × new_plan_price
Net = Charge − Credit
```

- Day-based, linear, calendar-day granularity
- Downgrade produces negative net → credit applied to account (not refunded)
- Works for monthly/quarterly/yearly

### QBill (BILLING_MATH.md P-1 through P-5)

**Formula:**
```
f = days_in_sub_window / days_in_full_period
```

- Day-based, calendar-day granularity (P-1)
- Base fee: each sub-window bills `round_half_up(base_amount × f, minor)` as its own BASE_FEE line (P-2)
- Included units: prorated by `floor(included_units × f)` (P-3)
- Seats: prorated like base fees, seat count per sub-window (P-4)
- Upgrades immediate, downgrades at next period start by default (P-5)

### Comparison

| Aspect | FlexPrice | QBill |
|---|---|---|
| **Granularity** | Calendar days | Calendar days |
| **Base fee proration** | Linear day-based | Linear day-based, per sub-window |
| **Included allowance proration** | Not mentioned | ✅ `floor(included_units × f)` — explicit formula |
| **Seat proration** | Not mentioned | ✅ Per sub-window with seat count tracking |
| **Upgrade/downgrade rules** | Not specified | ✅ P-5: upgrades immediate, downgrades next period |
| **Rounding** | Not specified | ✅ round-half-up per sub-window line |

### ✅ QBill Advantage

QBill's proration is **more complete**:
1. Explicit included-unit allowance proration with floor (P-3)
2. Per-sub-window seat proration (P-4)
3. Clear upgrade/downgrade timing rules (P-5)
4. Formal rounding rules (M-4)

### ❌ QBill Gap

No explicit semantics for what happens when a downgrade produces a negative net (credit vs. refund). FlexPrice explicitly documents that downgrades create a non-refunded account credit.

---

## 8. Wallet & Credit Systems

### FlexPrice

- **Wallet**: Per-currency prepaid balance, one active wallet per currency per customer
- **Credit-to-currency ratio**: Configurable via Pricing Units (e.g. "1 credit = $0.01")
- **Top-up**: Manual or auto (recharge when balance drops below threshold)
- **Consumption order**: Wallet credits applied AFTER all coupon discounts, BEFORE tax
- **Alerts**: 3 thresholds (info/warning/critical) fire webhooks, trigger enforcement
- **Manual debit**: Admin-side deduction

### QBill

- **Wallet**: Per-customer prepaid wallet with Redis hot-path burndown (W-1 through W-5)
- **Dual credit system**: FEFO credits (compensation/promotional/prepaid/commit, 4 priority levels) + wallet (real-time burndown)
- **Auto-topup**: Stripe PaymentIntent on threshold crossing (CR-2)
- **Nightly reconciliation**: Redis vs Postgres drift detection at 1% threshold (W-4)
- **Bounded overdraft**: `WALLET_MAX_OVERDRAFT` (default $1.00) hard-stops runaway (W-5)
- **FEFO ordering**: priority (ascending) → expires_at (ascending) → created_at (ascending) → ID

### Comparison

| Aspect | FlexPrice | QBill | Advantage |
|---|---|---|---|
| **Real-time burndown** | ❌ Invoice-time deduction | ✅ Redis hot-path (sub-5ms) | **QBill** |
| **Nightly reconciliation** | ❌ Not mentioned | ✅ W-4: drift detection + adjustment | **QBill** |
| **FEFO priority levels** | ❌ Single queue | ✅ 4 levels (compensation → promotional → prepaid → commit) | **QBill** |
| **Bounded overdraft** | ❌ Not mentioned | ✅ W-5: `WALLET_MAX_OVERDRAFT` | **QBill** |
| **Auto-topup** | ✅ Threshold-based | ✅ Stripe PaymentIntent-based | Tie |
| **Alerts/notifications** | ✅ 3 thresholds (info/warning/critical) | ✅ Low balance alerts | Tie |
| **Credit-to-currency ratio** | ✅ Configurable | ⚠️ 1:1 (not configurable) | **FlexPrice** |
| **Wallet on invoices** | ✅ Applied at invoice time | ⚠️ Real-time only (no invoice-time wallet application) | **FlexPrice** — invoice-level wallet credit visibility |

### ✅ QBill Advantages

1. **Real-time enforcement**: Wallet burns down on Redis hot path, enabling sub-5ms enforcement (DEC-010)
2. **Reconciliation loop**: Nightly drift correction ensures Redis cache matches Postgres ledger (W-4)
3. **Bounded overdraft**: Prevents runaway negative balances (W-5)
4. **FEFO priority**: 4-level credit priority enables complex credit stacking

### ❌ QBill Gaps

1. **Wallet not integrated into invoice pipeline**: QBill wallet burns down in real-time on Redis, not at invoice finalization. FlexPrice applies wallet at invoice time, giving clearer invoice-level credit visibility.
2. **No configurable credit-to-currency ratio**: FlexPrice supports custom Pricing Units (e.g. "1 credit = $0.01"). QBill appears to use 1:1.

---

## 9. Tax Calculation

### FlexPrice

```
Taxable amount = MAX(subtotal − all_coupon_discounts, 0)
Tax = Σ(each applicable tax rate × taxable_amount)
Total = subtotal − total_coupon_discounts − wallet_credits_applied + tax
```

- **Multiple tax rates** apply independently to the SAME taxable amount (non-compounding)
- Wallet credits do NOT reduce the tax base
- Tax is computed on `(subtotal − coupons)`, NOT `(subtotal − coupons − wallet)`

### QBill (BILLING_MATH.md M-5, CR-7)

```
Taxable = subtotal − credits_applied
Tax = round_half_up(taxable × tax_rate, minor_units)
Total = taxable + tax
```

- Single tax rate at invoice level
- Credits DO reduce the tax base
- Pluggable tax provider interface (CR-7): internal `billing.tax_regions` fallback, Avalara/Anrok/Stripe Tax supported

### Comparison

| Aspect | FlexPrice | QBill | Impact |
|---|---|---|---|
| **Tax granularity** | Multi-rate, non-compounding | Single rate, invoice-level | FlexPrice handles multiple concurrent tax rates (e.g. state + federal) |
| **Tax base** | `subtotal − coupons` (not minus wallet) | `subtotal − credits` | Different: FlexPrice protects tax base from wallet deductions; QBill reduces tax base by credits |
| **Tax provider** | Built-in (not specified) | Pluggable (CR-7: Avalara/Anrok/Stripe Tax) | QBill more flexible for enterprise |
| **Rounding** | Not specified | ✅ `round_half_up()` to minor units | QBill more precise |

### ✅ QBill Advantages

1. **Pluggable tax provider**: CR-7 interface supports Avalara, Anrok, Stripe Tax — enterprise-grade
2. **Formal rounding**: `round_half_up` to currency minor units (M-4)
3. **Tax exemption support**: `billing.tax_exemptions` and `billing.tax_calculation_audit` — full audit trail

### ❌ QBill Gaps

1. **Single-rate only**: QBill computes tax at invoice level with a single rate. Some jurisdictions require per-line-item tax at different rates (e.g., SaaS products taxed at 0%, professional services at 8%).
2. **No multi-rate non-compounding**: FlexPrice supports multiple concurrent rates (state 6% + federal 2% = 8% total, not 8.12%). QBill would need a separate line item per rate category.

---

## 10. AI/LLM Token Billing

### FlexPrice Approach

- **Standardized `ai.usage` event shape**: input_tokens, output_tokens, cached_tokens, reasoning_tokens, reported_cost, model, provider, agent_id
- **Two cost strategies**: (1) Trust source `reported_cost` verbatim, or (2) Re-price from daily-refreshed model pricing catalog
- **Auto-provisioned templates**: cost_tracking, team_budget, resale_markup — pre-built metering graphs
- **Markup/margin**: configurable markup percentage for resale scenarios

### QBill Approach

- **LiteLLM callback** → standard event shape (provider, model, tokens, cost as decimal string)
- **Rating waterfall** resolves rate per `(customer, meter, model, token_type)` — contract_rate → rate_card → pricing_model
- **MATRIX pricing model** enables per-model × per-token-type rates
- **COST_PLUS pricing model** enables BYOK markup pricing (CR-11)

### Comparison

| Aspect | FlexPrice | QBill | Advantage |
|---|---|---|---|
| **Standard event shape** | ✅ Purpose-built `ai.usage` | ⚠️ LiteLLM schema (equivalent) | Tie |
| **Token granularity** | input/output/cached/reasoning | input/output (via MATRIX dimensions) | FlexPrice has reasoning_tokens explicitly |
| **Cost strategies** | Trust source OR re-price from catalog | Rating waterfall (contract → rate card → plan) | QBill more flexible for negotiated rates |
| **Markup/margin** | ✅ Configurable markup % | ✅ COST_PLUS pricing model | Tie |
| **Pre-built templates** | ✅ 3 templates | ❌ Manual setup | **FlexPrice** |
| **Catalog refresh** | ✅ Daily refreshed | ⚠️ Manual rate card management | **FlexPrice** |
| **Model × token pricing** | Via separate features | ✅ MATRIX model (native) | **QBill** |

### ✅ QBill Advantages

1. **MATRIX pricing**: Native per-model × per-token-type rate resolution — LLM-native design
2. **Rating waterfall**: Contract-specific negotiated rates override catalog rates — enterprise-grade
3. **COST_PLUS for BYOK**: Native markup pricing for gateway traffic

### ❌ QBill Gaps

1. **No pre-built AI metering templates**: FlexPrice auto-creates the whole metering graph for common AI scenarios. QBill requires hand-building meters per model.
2. **No automated rate catalog refresh**: FlexPrice refreshes model pricing from providers daily. QBill requires manual rate card updates.
3. **No standardized `ai.usage` event shape documented**: QBill uses LiteLLM's schema but has no equivalent of the FlexPrice `ai.usage` specification in its own documentation.
4. **reasoning_tokens not explicitly handled**: FlexPrice explicitly separates reasoning_tokens (common in OpenAI-style chain-of-thought billing). QBill's MATRIX model can support this but it's not called out as a built-in dimension.

---

## 11. Identified Gaps & Action Items

### Critical Gaps

| # | Gap | Description | Impact | Suggested Fix |
|---|---|---|---|---|
| **G-FP-01** | **No coupon/discount pipeline** | QBill invoice pipeline (M-5) has no coupon or discount step. FlexPrice has line-item and subscription-level coupons with stacking. | Cannot offer promotional discounts, contract discounts, or coupon codes. Blocks enterprise sales. | Add coupon/discount step to `invoice.Generate()` pipeline. Create `billing.coupons` and `billing.discounts` models. |

### High-Priority Gaps

| # | Gap | Description | Impact | Suggested Fix |
|---|---|---|---|---|
| **G-FP-02** | **No WEIGHTED SUM aggregation** | QBill supports 7 aggregation types but not WEIGHTED SUM (time-proportional sum). | Cannot do capacity/reservation billing (e.g., GB-provisioned × hours) | Add WEIGHTED SUM to engine aggregation |
| **G-FP-03** | **No multi-rate tax** | QBill computes tax with a single rate at invoice level. | Fails for jurisdictions with multiple concurrent tax rates (state + federal + local) | Add per-line-item tax rate support or multi-rate invoice-level tax |
| **G-FP-04** | **No AI onboarding templates** | No pre-built metering templates for common AI scenarios. | Slower customer onboarding | Create templates: cost_tracking, team_budget, resale_markup |

### Medium-Priority Gaps

| # | Gap | Description | Suggested Fix |
|---|---|---|---|
| **G-FP-05** | **No standardized AI event shape** | QBill lacks a documented `ai.usage`-equivalent event specification | Create QBill AI event shape spec with input/output/cached/reasoning tokens |
| **G-FP-06** | **Downgrade proration not specified** | QBill P-5 says downgrades apply at next period but doesn't specify credit/refund behavior | Document downgrade credit treatment (like FlexPrice: non-refunded account credit) |
| **G-FP-07** | **Wallet not in invoice pipeline** | QBill wallet burns in real-time, not at invoice time | Add optional invoice-time wallet credit application for postpaid customers |
| **G-FP-08** | **No configurable credit ratio** | Wallet credits appear to be 1:1 | Add Pricing Units / credit-to-currency ratio configuration |

### Low-Priority Gaps

| # | Gap | Suggested Fix |
|---|---|---|
| **G-FP-09** | **No managed pull ingestion** | Add SaaS connector framework for Langfuse, Databricks, Bedrock |
| **G-FP-10** | **No automated rate catalog** | Add daily model pricing refresh from provider APIs |

---

## 12. Source Cross-Reference

| QBill Source | Content Referenced |
|---|---|
| `BILLING_MATH.md` M-1 to M-6 | Money precision, rounding, invoice arithmetic |
| `BILLING_MATH.md` P-1 to P-5 | Proration rules |
| `BILLING_MATH.md` W-1 to W-5 | Wallet burn, reconciliation, overdraft |
| `BILLING_MATH.md` C-1 to C-4 | Commit true-up |
| `BILLING_MATH.md` §7 | FEFO credit application |
| `BILLING_MATH.md` §9 | Worked example (golden test) |
| `ADR-001` §2 | One-writer rule, usage pipeline |
| `ADR-001` §3 | Hybrid billing, rate resolution, invoice pipeline |
| `ADR-001` §3.3 | Rating waterfall (4 steps) |
| `ADR-001` §3.4 | Invoice purity invariant |
| `ADR-001` §4 | Core requirements CR-1 through CR-14 |
| `phase_2_billing_worker.md` | Invoice lifecycle, acceptance criteria TC-01 through TC-17 |
| `ERD.md` §3 | Catalog, pricing models, rate cards |
| `ERD.md` §4 | Billing entities (invoices, credits, wallet) |
| `DECISIONS.md` DEC-010 | Enforcement latency contract |
| `SCAFFOLD.md` §6 | Conventions (money on wire, IDs, testing) |
| `engine/internal/invoice/generate.go` | Pure invoice function |
| `engine/internal/rating/resolver.go` | Rating waterfall |
| `engine/internal/wallet/` | Wallet operations |

---

*End of document. Generated 2026-07-16.*
