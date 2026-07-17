# QBill — Upgrade Roadmap

**Date:** 2026-07-16
**Purpose:** Prioritized list of improvements for QBill, with what each upgrade is, why it matters, how it helps, and estimated effort.

---

## Priority Levels

| Level | Meaning | Action Required |
|---|---|---|
| **🔴 P0 — Critical** | Production-blocking. Missing capability that every competitor has. | Must implement before general availability |
| **🟡 P1 — High** | Significant competitive gap or revenue-impacting limitation. | Should implement in next 2-3 sprints |
| **🟢 P2 — Medium** | Important capability that competitors offer; improves customer experience. | Plan for next quarter |
| **⚪ P3 — Low** | Nice-to-have enhancement. Differentiator or operational efficiency. | Add to roadmap backlog |

---

## 🔴 P0 — Critical Upgrades

### 1. Add Discount & Commitment Pipeline

**What to add:** A full discount/commitment system in the invoice calculation pipeline with these adjustment types:

| Type | What It Does | Example |
|---|---|---|
| **Usage discount** | Reduces quantity before pricing | "First 1,000 units free" → `effective_qty = max(0, qty - 1000)` |
| **Amount discount** | Subtracts fixed $ from line total | "$100 off monthly fee" |
| **Percentage discount** | Subtracts % from line total | "15% off for first 3 months" |
| **Minimum commitment** | Floors the invoice total | "At least $500/month" → `max(calculated, 500)` |
| **Maximum spend cap** | Caps the total charged | "Never more than $10,000/month" |

**Pipeline position** (insert between subtotal and credits in M-5):
```
1. Subtotal
2A. Usage discount → Amount discount → % discount → Minimum → Maximum   ← NEW
2B. FEFO credits applied                                                 ← existing
3. Taxable = adjusted_subtotal − credits_applied
4. Tax = round_half_up(taxable × rate, minor_units)
5. Total = taxable + tax
```

**Key rules to implement:**
- Discounts expire after N billing periods (contract terms)
- Minimums apply BEFORE credits (prevents credit-dodging)
- Multi-line discounts distribute proportionally by subtotal share
- Minimums not applied to fixed-fee-only invoices

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| Promotional discount ("20% off first year") | ❌ Impossible | ✅ Create percentage discount with 12-period expiry |
| Enterprise minimum commitment ("at least $5K/mo") | ❌ Impossible | ✅ Set minimum commitment on contract |
| Volume cap ("no more than $10K/mo") | ❌ Impossible | ✅ Set maximum spend cap |
| First-N-free ("100K tokens free") | ⚠️ Via P-3 allowance proration only | ✅ Usage discount on any pricing model |
| Partner discount ("10% off for partners") | ❌ Impossible | ✅ Percentage discount per subscription |

**Competitors that have this:** FlexPrice, Meteroid, OpenMeter, Orb — **all of them**
**Estimated effort:** 3-4 sprints (design + engine changes + BFF changes + tests)

---

### 2. Add Meter groupBy / Filter System

**What to add:** A `groupBy` field on meter definitions that lets a single meter serve multiple pricing dimensions, similar to Lago's filters, Meteroid's segmentation, and OpenMeter's groupBy.

**Current state:** Each model × token_type combination requires a separate meter.
**Target state:** One meter with groupBy dimensions feeds multiple prices via filters.

**How it works:**
```yaml
# Current QBill — 6 meters for 2 models × 3 token types
meter_gpt4_input      # event_type: ai.usage, field: input_tokens
meter_gpt4_output     # event_type: ai.usage, field: output_tokens
meter_gpt4_cached     # event_type: ai.usage, field: cached_tokens
meter_claude_input    # event_type: ai.usage, field: input_tokens
meter_claude_output   # event_type: ai.usage, field: output_tokens
meter_claude_cached   # event_type: ai.usage, field: cached_tokens

# Target — 1 meter with groupBy
meter_tokens_total:
  event_type: ai.usage
  field: total_tokens
  aggregation: SUM
  groupBy:
    model: $.model          # "gpt-4", "claude-3", etc.
    token_type: $.type      # "input", "output", "cached"
```

Then features reference the meter with filters:
```js
Feature 'gpt4_input_tokens'  → meter_tokens_total, filter: { model: 'gpt-4', type: 'input' }
Feature 'gpt4_output_tokens' → meter_tokens_total, filter: { model: 'gpt-4', type: 'output' }
```

**Engine impact:** Extend `rateSegmentUsage()` in `internal/invoice/` to group by configurable dimensions (it already groups by `model, token_type` — make this configurable from the meter definition).

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| Add new model (e.g., "gpt-5") | Create 3 new meters + pricing | Add "gpt-5" to groupBy filter — 1 line change |
| 10 models × 3 token types | 30 meters to manage | 1 meter to manage |
| Retroactively price new dimension | ❌ Requires new meters + backfill | ✅ New filter values apply to past events |
| Customer-specific pricing per model | ❌ Complex rate card per meter | ✅ Filter + contract rate on single meter |

**Competitors that have this:** Lago (filters), Meteroid (segmentation), OpenMeter (groupBy)
**Estimated effort:** 3-4 sprints (catalog changes + engine changes + migration)

---

## 🟡 P1 — High Priority Upgrades

### 3. Add Cost-Basis Tracking to Credits

**What to add:** Track `per_unit_cost_basis` on each credit block to differentiate promotional credits ($0 cost-basis, no revenue recognition impact) from purchased credits (>$0 cost-basis, drives revenue recognition).

**Current state:** QBill's FEFO credits use 4 priority levels (compensation=0, promotional=1, prepaid=2, commit=3) but don't track the actual monetary value of each credit block for revenue recognition purposes.

**Target state:** Each credit block has:
- `original_amount` — quantity of credits
- `remaining_amount` — unconsumed balance
- `priority` — 0-3 (existing)
- `cost_basis` — per-unit $ value (NEW)
- `expires_at` — expiration date
- `source_type` — `grant`, `purchase`, `allocation`

**FEFO deduction order updated** (5 steps, matching Orb):
1. License/entitlement-specific allocations
2. Item-scoped blocks before unscoped
3. Soonest-expiring blocks first
4. **Zero/low cost-basis before higher (free credits burn before paid)** — NEW
5. FIFO by creation time

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| Promotional vs purchased credit tracking | ❌ Not differentiated | ✅ Cost-basis $0 vs >$0 — clear rev-rec treatment |
| "Free 100K tokens" promotion | ❌ No rev-rec impact trackable | ✅ Cost-basis $0 — no deferral needed |
| Customer buys $500 prepaid credits | ❌ Cannot track rev-rec deferral | ✅ Cost-basis $0.0005/token — deferral recognized on consumption |
| Audit: which credit block covered which invoice line | ❌ Cannot trace | ✅ Cost-basis + block ID on each ledger entry |

**Competitors that have this:** Orb
**Estimated effort:** 2-3 sprints

---

### 4. Add Progressive Billing (Threshold Invoicing)

**What to add:** Mid-cycle invoice generation when accrued usage crosses a configurable $ threshold.

**Current state:** QBill only invoices at subscription period boundaries.
**Target state:** When usage crosses the threshold mid-period, generate a partial invoice immediately. Net against the final invoice at period end.

**How it works:**
```
Threshold: $500
Day 10: usage hits $520 → partial invoice #1 = $520
Day 15: usage reaches $650 → partial invoice #2 = $650 (supersedes #1)
Day 30: final usage = $800
End-of-period invoice = $800 - $650 (highest partial) = $150 remaining
```

**Key rules:**
- Threshold is per-subscription or per-customer configurable
- Partial invoices are superseded (not additive) — final invoice nets against highest partial
- Credits and adjustments apply at finalization, not on partial invoices
- $0 threshold = invoice per event (real-time billing)

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| High-usage AI customer | Wait 30 days for invoice → cash flow risk | ✅ Invoice at $1K threshold → get paid sooner |
| Prepaid wallet customer (overflow billing) | Wallet burns down; overflow waits for period end | ✅ Invoice overflow immediately |
| Risk management for new customers | No mid-cycle billing → credit risk until period end | ✅ Progressive billing caps exposure |
| Usage monitoring | Only at period end | ✅ Real-time partial invoices show accrual |

**Competitors that have this:** Lago, OpenMeter
**Estimated effort:** 2-3 sprints (engine + BFF + WebSocket notifications)

---

### 5. Add Metered Entitlements Separate from Billing Meters

**What to add:** Decouple "what we measure for billing" (meters/pricing) from "what we cap" (entitlements/limits). Create a separate entitlement entity with its own reset periods, override precedence, and enforcement API.

**Current state:** `customer.usage_limits` and `customer.limit_overrides` reference the same `meter_id` used for billing. The same meter definition handles both measurement and capping — conflated concerns.

**Target state:**

```
Meter → Billing (pricing + invoicing)
      → Entitlement (capping + enforcement) ← NEW separate system
```

**Entitlement types to add:**
| Type | Purpose | Example |
|---|---|---|
| **Metered** | Usage limit with balance tracking | 1,000,000 tokens/month |
| **Boolean** | On/off feature flag | SAML SSO: enabled |
| **Static** | Per-customer JSON config | `{ allowedModels: ["gpt-4", "claude-3"] }` |

**Reset strategies:**
| Strategy | Description |
|---|---|
| **Billing Cycle** | Resets with subscription anniversary (existing) |
| **Calendar** | Resets on absolute calendar boundaries (all customers on 1st) |
| **Fixed Window** | Resets on subscription-relative date (e.g., every 3rd) |
| **Sliding Window** | True rolling window (trailing 1 hour — for rate limiting) |
| **Never** | Lifetime cap |

**Override precedence:** Default (Feature) → Plan → Subscription (most specific wins)

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| Cap tokens at 1M/month but bill for all usage | ❌ Conflated — can't cap without also affecting billing | ✅ Meter tracks billing; separate entitlement tracks cap |
| Rate limiting (sliding window, 100 req/min) | ❌ Not supported | ✅ Sliding window entitlement |
| Feature flag (SSO, API access) | ❌ No boolean entitlement | ✅ Boolean entitlement |
| Per-customer model allowlist | ❌ Not supported | ✅ Static entitlement with JSON config |
| Different reset period for capping vs billing | ❌ Same period (anniversary) | ✅ Entitlement: calendar reset; Billing: anniversary |

**Competitors that have this:** Meteroid (entitlements separate from billing), OpenMeter (3 entitlement types + grants)
**Estimated effort:** 3-4 sprints

---

### 6. Add Grant Rollover Configurability

**What to add:** Configurable rollover rules per grant/credit block — control whether unused credits roll over to the next period, and by how much.

**Current state:** QBill's recurring grants are non-rollover by default (TR-2). No per-grant override.

**Target state:**
```
Balance After Reset = MIN(maxRolloverAmount, MAX(Balance Before Reset, minRolloverAmount))
```

**Configuration options:**
| `minRollover` | `maxRollover` | Behavior |
|---|---|---|
| 0 | 0 | **No rollover** — use it or lose it (current QBill default) |
| N | N | **Topped up to exactly N** each reset (recurring quota) |
| 0 | N | **Bounded rollover** — up to N unused units carry forward |
| M | N | **Floored rollover** — at least M carry forward, up to N |

**Additional OpenMeter-style features:**
- `isSoftLimit`: hard-block or just warn when exceeded
- `preserveOverageAtReset`: overage carries forward (can't escape by waiting for reset)
- **Recurrence**: independent top-up interval decoupled from reset period (e.g., +300 tokens daily on top of monthly 5,000 quota)

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| "Use it or lose it" monthly tokens | ✅ Non-rollover (default) | ✅ Same — no change needed |
| "Up to 500K unused tokens roll over" | ❌ Impossible | ✅ Set maxRollover=500000 |
| Monthly subscription with daily top-up | ❌ Not supported | ✅ Recurrence: +300/day independent of monthly reset |
| Overage forgiveness | ❌ Overage resets at period end | ✅ preserveOverageAtReset=true |

**Competitors that have this:** OpenMeter (grants), Orb (allocations)
**Estimated effort:** 1-2 sprints

---

## 🟢 P2 — Medium Priority Upgrades

### 7. Add Plan Phases (Multi-Stage Plans)

**What to add:** Support for time-ordered phases within a single plan — trial phase → ramp-up phase → evergreen phase.

**Current state:** QBill plans have a single phase with optional `trial_days`. No multi-phase support.

**Target state:**
```
Plan "Pro":
  Phase 1 (Trial, days 1-14): $0/mo, 500K tokens included
  Phase 2 (Ramp-up, days 15-45): $49/mo, 1M tokens included  ← reduced price
  Phase 3 (Evergreen, day 46+): $99/mo, 1M tokens included   ← standard price
```

**Phase properties:**
- Each phase has its own rate cards (prices + entitlements)
- Phase transitions land on period boundaries (no invoice spans two phases)
- Final phase is always "evergreen" (runs indefinitely)
- Reverse trial: full features in phase 1, reduced in phase 2

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| Free trial that auto-converts to paid | ⚠️ Manual subscription change at trial end | ✅ Built into plan — no action needed |
| "First 3 months at 50% off" | ❌ Requires coupon/discount system | ✅ Ramp-up phase with reduced price |
| Reverse trial (full features 14 days, then freemium) | ❌ Impossible | ✅ Phase 1: premium; Phase 2: standard |
| Contract ramp (year 1: $5K/mo, year 2: $10K/mo) | ❌ Requires separate contracts | ✅ Multi-year phase plan |

**Competitors that have this:** OpenMeter (plan phases), Orb (plan phases)
**Estimated effort:** 2-3 sprints

---

### 8. Add Add-On System

**What to add:** Versioned, independently-priced add-on catalog items attachable to subscriptions.

**Current state:** Add-ons require a separate subscription. No concept of instance control, compatibility locking, or versioned add-on catalog.

**Target state:**
```
Add-on "Extra Storage 100GB":
  - Rate card: $10/mo, 100GB storage
  - Instance: Multi-instance (customer can add multiple)
  - Compatibility: "Pro Plan" and "Enterprise Plan" only
  - Versioned: Draft → Published → Archived
  
Subscription "ACME Inc. — Pro Plan":
  - Base plan: Pro ($99/mo)
  - Add-ons:
    - Extra Storage 100GB × 2 = $20/mo
    - SAML SSO add-on = $0/mo
```

**Add-on properties:**
| Property | Options |
|---|---|
| Instance control | Single-instance (max 1) or Multi-instance (unlimited) |
| Compatibility | List of compatible plan versions |
| Billing cadence | Must match base plan cadence |
| Versioning | Draft → Published → Archived (like plans) |

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| Customer wants extra storage | Create separate subscription | ✅ Add add-on to existing subscription |
| Overage package ("10K extra tokens") | ❌ No mechanism | ✅ Multi-instance add-on |
| Plan compatibility control | ❌ Any plan can add anything | ✅ Compatibility locked at plan publication |
| Catalog management | ❌ No add-on catalog | ✅ Versioned add-on catalog |

**Competitors that have this:** OpenMeter (add-ons)
**Estimated effort:** 2-3 sprints

---

### 9. Add Calendar-Month Billing Option

**What to add:** Support for calendar-month billing periods (1st to end-of-month) in addition to anniversary-based periods.

**Current state:** Anniversary-only billing (T-3). Subscription start date anchors the billing period.

**Target state:** Two billing cycle options:
| Option | Behavior | Best For |
|---|---|---|
| **Anniversary** (existing) | Period anchored to subscription start date | Per-subscription billing integrity |
| **Calendar** (new) | Period aligns to calendar month (1st → end) | Org-wide billing consistency |

**Calendar billing rules:**
- First period prorated if subscription starts mid-month
- All customers on same billing cycle (easier for org-wide reporting)
- Proration matches QBill's existing day-based approach (P-1)

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| Org wants all customers billed on the 1st | ❌ Each customer has different anniversary date | ✅ Calendar billing — all on 1st |
| Simple reporting ("September revenue") | ❌ Need to aggregate across staggered anniversaries | ✅ Calendar-aligned periods |
| Easy-to-understand invoices for customers | ⚠️ "Period: Jul 15 - Aug 15" | ✅ "Period: July 1 - July 31" |

**Competitors that have this:** Lago (calendar as default), Orb (3 options), Meteroid (both options)
**Estimated effort:** 1 sprint

---

### 10. Add Invoice Preview (Upcoming Invoice) Endpoint

**What to add:** A real-time "upcoming invoice" endpoint that shows accrued charges for the current period before the invoice is generated.

**Current state:** QBill generates invoices only at period boundaries. No preview of in-progress charges.

**Target state:**
```http
GET /api/v1/organizations/:orgId/customers/:customerId/upcoming-invoice
```
Response shows:
- Accrued line items (from ClickHouse usage this period)
- Estimated FEFO credits that will apply
- Estimated tax
- Wallet balance
- Current period start/end dates

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| Customer asks "how much do I owe so far?" | ❌ No way to show | ✅ Real-time preview |
| Dashboard "current charges this month" | ❌ Not available | ✅ Accrued charges display |
| Prepaid wallet burn-down visibility | ✅ Wallet balance only (not invoice impact) | ✅ Wallet + projected invoice |
| Budget tracking | ❌ No mid-period visibility | ✅ Preview enables budget alerts |

**Competitors that have this:** Orb (live draft invoice always exists), Meteroid (upcoming invoice view)
**Estimated effort:** 1-2 sprints

---

### 11. Add Per-Line-Item Tax Calculation

**What to add:** Support for different tax rates per line item (product type) instead of a single invoice-level rate.

**Current state:** QBill computes tax at invoice level with a single rate (M-5).
**Target state:** Each line item can have its own tax rate, summed at invoice level.

**Pipeline change:**
```
# Current (M-5):
Taxable = subtotal − credits_applied
Tax = round_half_up(taxable × single_rate, minor_units)

# New:
Tax = Σ(round_half_up(line_i.amount × line_i.tax_rate, minor_units)) for all lines
```

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| SaaS product (0% tax) + services (8% tax) on same invoice | ❌ Single rate misapplies to both | ✅ Per-line-item rates |
| Multiple jurisdictions on one invoice | ❌ Not supported | ✅ Per-line jurisdiction resolution |
| Tax-exempt + taxable items combined | ❌ Single rate applies to all | ✅ Exempt lines at 0%, taxable at normal rate |

**Competitors that have this:** Orb (per-line-item via TaxJar/Avalara), FlexPrice (per-line-item rates)
**Estimated effort:** 1-2 sprints

---

### 12. Add Virtual Currency / Custom Pricing Units

**What to add:** Support for virtual currencies (e.g., "tokens", "credits", "API calls") as pricing units with configurable conversion rates to real currency.

**Current state:** QBill uses real currency only (USD, EUR, etc.). No virtual/custom pricing units.

**Target state:**
```json
{
  "pricing_unit": {
    "name": "Token",
    "symbol": "TOK",
    "conversion_rate": 0.000001,  // 1 TOK = $0.000001
    "type": "virtual"
  }
}
```

**How it helps:**
| Use Case | Current QBill | With This Upgrade |
|---|---|---|
| "10,000 tokens = $0.01" pricing model | ⚠️ Must use micro-USD (hard to read) | ✅ "10,000 TOK = $0.01" with 1:0.000001 conversion |
| Prepaid credit packs ("Buy 1M credits for $100") | ❌ Only real-currency wallet topup | ✅ Credit pack as virtual currency with conversion |
| Multi-currency pricing (USD price, EUR invoice) | ❌ Not supported (X-2) | ✅ Virtual → real conversion at invoice time |
| Customer-facing usage dashboards | ⚠️ Raw token counts | ✅ Priced in virtual currency with $ equivalent |

**Competitors that have this:** Orb (custom pricing units), FlexPrice (Pricing Units)
**Estimated effort:** 2-3 sprints

---

## ⚪ P3 — Low Priority / Roadmap Upgrades

### 13. Add Plan-Based Discount Expiration

**What to add:** Discounts that expire after N billing periods (e.g., "20% off for first 12 months").

**Depends on:** #1 (Discount & Commitment Pipeline)

**How it helps:** Automates contract term discounts without manual intervention.

---

### 14. Add Sliding Window Rate Limiting

**What to add:** True rolling-window rate limits (trailing 1 hour, continuously evaluated) in addition to per-period limits.

**Depends on:** #5 (Metered Entitlements Separate from Billing Meters)

**How it helps:** API rate limiting, burst protection, fair usage enforcement.

---

### 15. Add Standardized AI Event Shape

**What to add:** A documented, purpose-built `ai.usage` event schema for LLM billing, similar to FlexPrice's and Orb's templates.

**How it helps:** Simplifies customer onboarding, ensures consistent event structure across all LLM providers.

---

### 16. Add Pre-Built AI Metering Templates

**What to add:** Auto-provisioned metering templates for common AI scenarios (cost tracking, team budget, resale markup).

**Depends on:** #2 (Meter groupBy/Filter System)

**How it helps:** One-click setup for common AI billing patterns — reduces onboarding friction.

---

### 17. Add Dependency Graph Invoice Invalidation

**What to add:** Track which invoices depend on which data (events, rates, plans). When something changes, recalculate only affected invoices instead of full-period re-runs.

**How it helps:** Efficient targeted re-rating at scale. Orb has this as a key differentiator.

---

### 18. Add Multiple Ingestion Paths

**What to add:** Support for Segment, S3/GCS drop, Kinesis, Reverse ETL (Census/Hightouch) as event sources.

**How it helps:** Broader integration ecosystem, easier customer onboarding.

---

### 19. Add Post-Tax Customer Balance

**What to add:** A simple after-tax balance for refunds, goodwill credits, and rounding carryover (separate from prepaid credits).

**Depends on:** #3 (Cost-Basis Tracking)

**How it helps:** Cleaner handling of non-revenue-impacting adjustments.

---

### 20. Add CloudEvents Standard Support

**What to add:** Optionally accept CloudEvents-format events alongside the existing custom JSON schema.

**How it helps:** CNCF-standard interoperability, broader ecosystem compatibility.

---

## Summary: Upgrade Priority Matrix

| # | Upgrade | Priority | Competitors That Have It | Effort | Dependencies |
|---|---|---|---|---|---|
| 1 | Discount & Commitment Pipeline | **🔴 P0 Critical** | FlexPrice, Meteroid, OpenMeter, Orb | 3-4 sprints | None |
| 2 | Meter groupBy / Filter System | **🔴 P0 Critical** | Lago, Meteroid, OpenMeter | 3-4 sprints | None |
| 3 | Cost-Basis Tracking on Credits | **🟡 P1 High** | Orb | 2-3 sprints | None |
| 4 | Progressive Billing | **🟡 P1 High** | Lago, OpenMeter | 2-3 sprints | #1 (pipeline) |
| 5 | Separate Metered Entitlements | **🟡 P1 High** | Meteroid, OpenMeter | 3-4 sprints | None |
| 6 | Grant Rollover Configurability | **🟡 P1 High** | OpenMeter | 1-2 sprints | #3 (cost-basis) |
| 7 | Plan Phases | **🟢 P2 Medium** | OpenMeter, Orb | 2-3 sprints | None |
| 8 | Add-On System | **🟢 P2 Medium** | OpenMeter | 2-3 sprints | None |
| 9 | Calendar-Month Billing | **🟢 P2 Medium** | Lago, Orb, Meteroid | 1 sprint | None |
| 10 | Invoice Preview Endpoint | **🟢 P2 Medium** | Orb, Meteroid | 1-2 sprints | None |
| 11 | Per-Line-Item Tax | **🟢 P2 Medium** | Orb, FlexPrice | 1-2 sprints | None |
| 12 | Virtual Currency / Custom Pricing Units | **🟢 P2 Medium** | Orb, FlexPrice | 2-3 sprints | #3 (cost-basis) |
| 13-20 | Lower priority items | **⚪ P3 Low** | Various | 1-3 sprints each | Various |

---

## Quick Reference: "If we want to..."

| Business Goal | Upgrade Needed | Priority |
|---|---|---|
| Offer promotional discounts | #1 Discount Pipeline | 🔴 P0 |
| Support enterprise minimum commitments | #1 Discount Pipeline | 🔴 P0 |
| Simplify meter management (10 models × 3 types = 1 meter, not 30) | #2 Meter groupBy | 🔴 P0 |
| Support per-model AI token pricing easily | #2 Meter groupBy | 🔴 P0 |
| Track promotional vs purchased credits for accounting | #3 Cost-Basis Tracking | 🟡 P1 |
| Invoice high-usage customers mid-month | #4 Progressive Billing | 🟡 P1 |
| Cap API usage without affecting billing | #5 Separate Entitlements | 🟡 P1 |
| Allow unused tokens to roll over | #6 Grant Rollover | 🟡 P1 |
| Auto-convert free trial to paid plan | #7 Plan Phases | 🟢 P2 |
| Sell add-ons without separate subscriptions | #8 Add-On System | 🟢 P2 |
| Bill all customers on the 1st of the month | #9 Calendar Billing | 🟢 P2 |
| Show customers their current bill so far | #10 Invoice Preview | 🟢 P2 |
| Handle different tax rates per product type | #11 Per-Line-Item Tax | 🟢 P2 |
| Price in "tokens" instead of micro-dollars | #12 Virtual Currency | 🟢 P2 |
| Automate first-year discount expiration | #13 Discount Expiration | ⚪ P3 |
| Implement API rate limiting (sliding window) | #14 Sliding Window Limits | ⚪ P3 |
| Standardize AI event ingestion | #15 AI Event Shape | ⚪ P3 |
| One-click AI metering setup | #16 AI Metering Templates | ⚪ P3 |
| Efficient targeted re-rating at scale | #17 Dependency Graph | ⚪ P3 |
| Ingest events from Segment, S3, Kinesis | #18 Multiple Ingestion Paths | ⚪ P3 |
| Handle refunds without affecting revenue | #19 Post-Tax Balance | ⚪ P3 |
| Support CloudEvents standard | #20 CloudEvents Support | ⚪ P3 |

---

## Beyond Features — Operational, Security & Process Improvements

The 20 upgrades above are **product features** (what customers see and competitors match). Below are the **non-feature improvements** — operational excellence, security hardening, testing, documentation, and process improvements that QBill also needs.

---

### 🟡 P1 — High Priority (Non-Feature)

#### N-1. Golden Test for BILLING_MATH.md §9 in CI

**What:** The BILLING_MATH.md §9 worked example must be implemented as a golden test in the invoice engine suite, enforced by CI.

**Current state:** SCAFFOLD §6 requires this but it hasn't been verified as present or enforced.

**What needs to happen:**
1. Create a test that runs the §9 example through `invoice.Generate()`
2. Assert the exact output values: $49.50, $99.50, $3.00, $152.00 subtotal, $152.00 credits_applied, $0.00 total
3. Wire into CI pipeline so any code change that produces different output fails immediately

**How it helps:**
- Catches calculation regressions instantly
- Ensures every developer change respects BILLING_MATH.md
- Provides a single source of truth for "is the invoice engine correct?"

**Estimated effort:** 1-2 days

---

#### N-2. Redis Failover Strategy

**What:** Document and implement a Redis failover strategy. Redis is on the enforcement hot path (<5ms). If Redis goes down, enforcement stops working.

**Current state:** No documented Redis failover strategy. Redis Stack is deployed as a single instance in compose.

**What needs to happen:**
1. Choose a Redis HA topology (Sentinel, Cluster, or managed Redis with automatic failover)
2. Document the failover behavior for each Redis key type:
   - `apikey:*` — cache miss → fall back to Postgres (graceful degradation)
   - `usage:{org}:{customer}` — counter loss → reconciled at next invoice run from ClickHouse
   - `wallet:{customer_id}` — **critical** — balance could be stale; enforce read-only mode
   - `idem:{org}:{event_id}` — TTL-based, loss means potential duplicate events at ingest
3. Implement connection retry with exponential backoff for all Redis operations
4. Add health check integration: if Redis is down, enforcement returns 503 (fail closed)

**How it helps:**
- Prevents billing data corruption during Redis outages
- Ensures wallet balance integrity across failover
- Meets enterprise SLA requirements

**Estimated effort:** 2-3 sprints

---

#### N-3. Edge Case Test Suite

**What:** Create a systematic edge case test suite for the billing engine, driven by a formal edge case catalog.

**Current state:** edge case catalog doesn't exist (G-DOC-04). Tests exist for golden path scenarios but edge cases are untested systematically.

**Edge cases to cover:**

| Category | Edge Cases |
|---|---|
| **Zero values** | Zero-usage invoice, zero-rate plan, zero-amount credit, zero-tax region, zero-balance wallet |
| **Negative values** | Negative credit remaining, negative adjustment, negative tax (exemption refund) |
| **Boundary conditions** | Midnight transitions (23:59:59.999 → 00:00:00.000), leap year Feb 29, DST transitions, month-end clamping (Jan 31 → Feb 28) |
| **Concurrency** | Simultaneous plan change + invoice generation, concurrent wallet burndown + topup, overlapping re-rating runs |
| **Partial states** | Partial payment on invoice, partially consumed credits, partially filled rate card (some models priced, others unrated) |
| **Race conditions** | Late event arriving during grace window finalization, event ingested after period close, credit expiring mid-invoice-generation |
| **Idempotency** | Duplicate event after network retry, duplicate re-rating run, duplicate credit grant |
| **Cross-currency** | Customer currency mismatch with plan, billing group currency mismatch, FX rate rounding edge cases |
| **Time travel** | Test clock advancement beyond period end, backdated cancellation, retroactive rate change |
| **Overflow** | Max decimal precision (38,9) overflow, max wallet overdraft boundary, max invoice line items |

**What needs to happen:**
1. Create the edge case catalog document (G-DOC-04)
2. For each edge case, determine expected behavior (some may need product decisions)
3. Implement automated tests for each edge case
4. Wire into CI as part of the full cumulative suite

**How it helps:**
- Prevents production incidents from untested boundary conditions
- Documents expected behavior for future developers
- Reduces support burden from edge-case billing questions

**Estimated effort:** 3-4 sprints (ongoing — add edge cases as discovered)

---

#### N-4. Documentation Reconciliation Sprint

**What:** A dedicated sprint to reconcile all 98 MD files with the current implementation. 30 files (31%) are outdated with pre-ADR vocabulary; 14 files (15%) reference deleted tables or processes.

**Current state:** ADR-001 §6 specifies which docs need changes, but most backend stories still use `tenant_id`/`user_id` and most uiflow stories reference the deleted `billing.usage_events` table.

**What needs to happen (in priority order):**

| Priority | Files | Change |
|---|---|---|
| **Critical** | `quantumbilling_invoice_user_story.md` | Remove Postgres invoice generator; convert to read/present/pay flow |
| **Critical** | `quantumbilling_meter_user_story.md` | Remove `billing.usage_events` references; rewrite as facade → Go ingest API |
| **Critical** | `workflow_connectivity_analysis.md` | Add event engine to topology |
| **High** | `story_1` through `story_10` | Rename `tenant_id`→`customer_id`, `user_id`→`end_user_id` |
| **High** | 8 uiflow data-source stories | Rewrite from Postgres `billing.usage_events` to Go phase-4 API calls |
| **High** | `quantumbilling_credits_user_story.md` | Add wallet, burndown display, auto top-up (CR-2) |
| **High** | `quantumbilling_pricing_user_story.md` | Add CR-3 models (MATRIX, COST_PLUS) and CR-9 simulation |
| **Medium** | `story_25` through `story_33` | Verify against engine implementation |

**How it helps:**
- New developers can trust documentation as source of truth
- Eliminates the "docs vs code" confusion that slows development
- Prevents implementation errors from following outdated specs
- Reduces onboarding time for new team members

**Estimated effort:** 2-3 sprints for critical + high items; ongoing for medium/low

---

### 🟢 P2 — Medium Priority (Non-Feature)

#### N-5. End-to-End Load Test for Billing Worker

**What:** Create a load test for the billing worker's cold path (invoice generation at scale), not just the ingest hot path.

**Current state:** Load tests exist for ingest (50k events/sec targets). No documented load test for the billing worker generating invoices for thousands of subscriptions simultaneously.

**What needs to happen:**
1. Create a load test harness that simulates N subscriptions with M meters each
2. Generate P periods of historical usage data
3. Measure: invoice generation time, Postgres write throughput, memory usage
4. Establish baselines in `.perf-baselines.json`
5. Run in CI as a perf gate (TEST_PLAN G3)

**Target metrics:**
- Invoice generation: P95 < 5s for 10,000 subscriptions with 50+ line items each
- Memory: < 500MB peak during batch generation
- Postgres write: < 10s for batch insert of 10,000 invoices + 50,000 line items

**How it helps:**
- Prevents performance regressions as the billing engine grows
- Ensures month-end invoice runs complete within SLAs
- Identifies bottlenecks before they hit production

**Estimated effort:** 2-3 sprints

---

#### N-6. KMS / Secrets Management Integration

**What:** Replace the `BYOK_MASTER_KEY` env var with a proper KMS provider (AWS KMS, HashiCorp Vault, or Azure Key Vault) for envelope encryption.

**Current state:** ADR-001 §7 identifies KMS as a deferred decision. Production uses `BYOK_MASTER_KEY` env var with SHA-256 — inadequate for production security.

**What needs to happen:**
1. Choose a KMS provider
2. Implement envelope encryption: KMS encrypts a DEK (Data Encryption Key), DEK encrypts BYOK provider keys
3. Replace `BYOK_MASTER_KEY` env var with KMS key reference
4. Update `story_13` implementation to use KMS
5. Add key rotation support
6. Document disaster recovery procedure (what happens if KMS is unavailable)

**How it helps:**
- Production-grade security for customer BYOK encryption keys
- Meets compliance requirements (SOC 2, ISO 27001)
- Enables key rotation without data re-encryption

**Estimated effort:** 2-3 sprints

---

#### N-7. Public API Documentation Portal

**What:** Generate and publish HTML API documentation from the OpenAPI specs (bff-core.yaml, event-engine.yaml, analytics.yaml), similar to how competitors host docs.getlago.com, docs.withorb.com, etc.

**Current state:** OpenAPI YAML files exist but no generated HTML reference is published.

**What needs to happen:**
1. Choose a documentation generator (Redoc, Swagger UI, Stoplight)
2. Set up a build pipeline that generates HTML from OpenAPI specs
3. Deploy to a documentation subdomain (e.g., docs.quantumbilling.dev)
4. Include interactive API playground (try-it-yourself)
5. Add getting-started guides and code samples

**How it helps:**
- Enables self-service API exploration for customers
- Reduces support burden from API usage questions
- Professional appearance for customer evaluation
- Serves as the single source of truth for API contracts

**Estimated effort:** 1-2 sprints (initial), ongoing maintenance

---

#### N-8. Audit Log for All Billing Mutations

**What:** Ensure every billing mutation (invoice state change, credit consumption, wallet movement, payment recording) has a complete, immutable audit trail with actor, timestamp, before/after values.

**Current state:** `platform.audit_logs` exists but may not cover all billing mutations. `billing.invoice_status_history` covers invoice state changes. Gaps may exist for other billing entities.

**What needs to happen:**
1. Audit every billing mutation path in the engine
2. Ensure each mutation writes to an audit/history table with: `actor_id`, `action`, `resource_type`, `resource_id`, `old_values`, `new_values`, `timestamp`
3. For financial entities, ensure audit is in the same database transaction as the mutation
4. Add a centralized audit query API

**How it helps:**
- SOC 2 / SOX compliance for financial operations
- Customer support can trace exactly what happened and when
- Forensic analysis capability for billing disputes
- Debugging aid for production issues

**Estimated effort:** 1-2 sprints

---

#### N-9. Developer Environment Improvements

**What:** Improve the local development experience with better tooling, faster feedback loops, and clearer documentation.

**Current issues:**
- `docker compose up -d` brings up 6+ services (postgres, redis, kafka, clickhouse, keycloak, kafka-ui) — slow startup
- No hot-reload for Go engine services
- Test data seeding requires multiple manual steps
- No mock/stub for external dependencies (Stripe, LiteLLM)

**What needs to happen:**
| Improvement | Description |
|---|---|
| **Docker Compose profiles** | Improve profile separation so developers only run services relevant to their work |
| **Hot reload for Go** | Add `air` or `watcher` for auto-rebuild on file changes |
| **One-command seed** | Combine `prisma migrate deploy` + `clickhouse-migrate.sh` + `seed-dev.sql` + `warm-redis.sh` into one command |
| **Stripe mock** | Use `stripe-mock` or similar for local payment testing without live Stripe keys |
| **LiteLLM stub** | Provide a LiteLLM mock that returns predictable responses for callback testing |
| **Test clock UI** | CLI or web UI for managing test clocks (create, advance, inspect) |
| **OpenAPI contract testing** | Auto-generated contract tests from OpenAPI specs that run in CI |

**How it helps:**
- Faster onboarding for new developers (from days to hours)
- Faster feedback loop during development (from minutes to seconds)
- Reduced "works on my machine" issues
- More reliable integration tests

**Estimated effort:** 2-3 sprints for the complete set

---

### ⚪ P3 — Low Priority (Non-Feature)

#### N-10. Flink vs Go Aggregator Decision

**What:** Make and implement the decision about stream aggregation technology (Flink vs custom Go aggregator) for 1/5-minute window aggregations.

**Current state:** Deferred per ADR-001 §7. The decision is needed before Phase 1 build at production scale.

**How it helps:** Determines the architecture for real-time aggregation capabilities.

**Estimated effort:** 1 sprint (decision + PoC)

---

#### N-11. SMS Provider Selection (Dunning)

**What:** Select and integrate an SMS provider (Twilio suggested) for dunning workflow notifications.

**Current state:** Deferred. Dunning has EMAIL action implemented but SMS action requires a provider.

**How it helps:** Enables SMS-based dunning notifications — improves payment collection rates.

**Estimated effort:** 1 sprint (integration)

---

#### N-12. Object Storage Selection (Reports & Invoices)

**What:** Select and integrate an S3-compatible object storage for invoice PDFs, reports, and warehouse export (CR-13).

**Current state:** Deferred. No object storage backend selected.

**How it helps:** Enables invoice PDF storage, report generation, and warehouse-native data export.

**Estimated effort:** 1-2 sprints

---

#### N-13. Documentation → Implementation Drift Detection

**What:** Implement automated checks that detect when documentation and implementation diverge (e.g., OpenAPI spec vs actual API behavior, BILLING_MATH.md formulas vs engine calculations).

**Current state:** No drift detection process. Documentation may fall out of sync with code.

**Possible approaches:**
- OpenAPI contract tests: validate API responses against OpenAPI schemas in CI
- Golden tests: BILLING_MATH.md worked examples as automated tests
- API surface diff: compare documented endpoints against actual route registrations
- Schema diff: compare ERD.md against Prisma schema

**How it helps:**
- Keeps documentation reliable as the codebase evolves
- Catches undocumented API changes before they ship
- Reduces the need for large documentation reconciliation efforts

**Estimated effort:** 1-2 sprints (setup), ongoing maintenance

---

#### N-14. Performance Baselines for All Critical Paths

**What:** Establish and enforce performance baselines for ALL critical code paths (not just enforcement).

**Current state:** Only enforcement has a documented latency contract (DEC-010: P99 < 5ms @ c=32). Invoice generation, analytics queries, and wallet operations lack baselines.

**Critical paths needing baselines:**

| Path | Suggested Baseline | Why It Matters |
|---|---|---|
| Invoice generation (per subscription) | P95 < 2s for 50 line items | Month-end batch processing SLA |
| Analytics query (org summary) | P95 < 500ms for 1M events | Dashboard responsiveness |
| Wallet burndown (single event) | P99 < 10ms | Real-time enforcement SLA |
| Re-rating (per period) | P95 < 5s for 30-day period | Correction processing SLA |
| Credit note generation | P95 < 1s | Customer-facing credit issuance |
| BFF invoice list | P95 < 200ms for 100 invoices | UI responsiveness |

**How it helps:**
- Prevents silent performance regressions
- Provides clear SLAs for operations
- Enables capacity planning

**Estimated effort:** 2-3 sprints for initial baselines; automated in CI

---

## Complete Upgrade Inventory

| ID | Upgrade | Type | Priority | Effort |
|---|---|---|---|---|
| 1 | Discount & Commitment Pipeline | Feature | 🔴 P0 Critical | 3-4 sprints |
| 2 | Meter groupBy / Filter System | Feature | 🔴 P0 Critical | 3-4 sprints |
| 3 | Cost-Basis Tracking on Credits | Feature | 🟡 P1 High | 2-3 sprints |
| 4 | Progressive Billing | Feature | 🟡 P1 High | 2-3 sprints |
| 5 | Separate Metered Entitlements | Feature | 🟡 P1 High | 3-4 sprints |
| 6 | Grant Rollover Configurability | Feature | 🟡 P1 High | 1-2 sprints |
| **N-1** | **Golden Test for BILLING_MATH.md** | **Testing** | **🟡 P1 High** | **1-2 days** |
| **N-2** | **Redis Failover Strategy** | **Operations** | **🟡 P1 High** | **2-3 sprints** |
| **N-3** | **Edge Case Test Suite** | **Testing** | **🟡 P1 High** | **3-4 sprints** |
| **N-4** | **Documentation Reconciliation Sprint** | **Process** | **🟡 P1 High** | **2-3 sprints** |
| 7 | Plan Phases | Feature | 🟢 P2 Medium | 2-3 sprints |
| 8 | Add-On System | Feature | 🟢 P2 Medium | 2-3 sprints |
| 9 | Calendar-Month Billing | Feature | 🟢 P2 Medium | 1 sprint |
| 10 | Invoice Preview Endpoint | Feature | 🟢 P2 Medium | 1-2 sprints |
| 11 | Per-Line-Item Tax | Feature | 🟢 P2 Medium | 1-2 sprints |
| 12 | Virtual Currency / Custom Pricing Units | Feature | 🟢 P2 Medium | 2-3 sprints |
| **N-5** | **E2E Load Test for Billing Worker** | **Testing** | **🟢 P2 Medium** | **2-3 sprints** |
| **N-6** | **KMS / Secrets Management** | **Security** | **🟢 P2 Medium** | **2-3 sprints** |
| **N-7** | **Public API Documentation Portal** | **Developer Experience** | **🟢 P2 Medium** | **1-2 sprints** |
| **N-8** | **Audit Log for All Billing Mutations** | **Compliance** | **🟢 P2 Medium** | **1-2 sprints** |
| **N-9** | **Developer Environment Improvements** | **Developer Experience** | **🟢 P2 Medium** | **2-3 sprints** |
| 13-20 | Feature backlog items | Feature | ⚪ P3 Low | 1-3 sprints each |
| **N-10** | **Flink vs Go Aggregator Decision** | **Architecture** | **⚪ P3 Low** | **1 sprint** |
| **N-11** | **SMS Provider Selection** | **Infrastructure** | **⚪ P3 Low** | **1 sprint** |
| **N-12** | **Object Storage Selection** | **Infrastructure** | **⚪ P3 Low** | **1-2 sprints** |
| **N-13** | **Doc → Implementation Drift Detection** | **Process** | **⚪ P3 Low** | **1-2 sprints** |
| **N-14** | **Performance Baselines for All Paths** | **Operations** | **⚪ P3 Low** | **2-3 sprints** |

---

## If You Could Only Do 5 Things

If you have limited time, here are the **5 highest-impact items** across both feature and non-feature categories:

| Rank | ID | Item | Why |
|---|---|---|---|
| **1** | **1** | Discount & Commitment Pipeline | Every competitor has it. Without it, you can't offer promotions, commitments, or caps. Production-blocker. |
| **2** | **2** | Meter groupBy / Filter System | Every major competitor has it. Without it, meter management is 30× harder. Production-blocker. |
| **3** | **N-3** | Edge Case Test Suite | Catches bugs before they reach production. Month-end billing errors are expensive and erode trust. |
| **4** | **5** | Separate Metered Entitlements | Decouples capping from billing — enables rate limiting, feature flags, and per-customer controls. |
| **5** | **N-4** | Documentation Reconciliation | 30 of 98 MD files are outdated. New developers waste weeks navigating stale docs. Fix this once. |

---

*End of document. Generated 2026-07-16.*
