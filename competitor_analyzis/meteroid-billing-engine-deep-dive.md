# Meteroid — Billing Engine Deep Dive
### How metering, plans, subscriptions, and invoice calculations actually work

> Source: docs.meteroid.com (fetched July 2026). Written for engineering reference — the goal is to understand the calculation model well enough to replicate/compare it against a custom billing engine (e.g. LLM token metering).

---

## 1. What Meteroid actually is

Meteroid is **not a payment processor** — it's a billing/metering *brain* that sits between your product (usage events) and your PSP (Stripe, etc.). It:

1. Ingests raw usage events (Kafka + ClickHouse pipeline, written in Rust)
2. Aggregates them into **Billable Metrics**
3. Applies **Plan pricing** to those metrics + subscription/recurring charges
4. Generates **Invoices**
5. Tracks payment status via your PSP, but doesn't move money itself

The whole system is a layered hierarchy. Understanding the hierarchy is 80% of understanding the calculations:

```
Tenant (env: prod/sandbox)
 └─ Billing Entity / Invoicing Entity (legal entity, currency, tax)
     └─ Product Family / Product Line (grouping)
         └─ Product (a sellable thing + its entitlements)
             └─ Plan (bundles Products + pricing + trial + net terms)
                 └─ Plan Version (draft → active → superseded, immutable once published)
                     └─ Subscription (Customer × Plan Version, the billing unit)
                         └─ Invoice (one per subscription per billing period)
```

Key structural facts that drive every calculation downstream:
- A **Plan** is a template. A **Subscription** is an instantiated, *frozen* copy of a Plan Version's pricing at the moment it was created — so changing a Plan later never retroactively changes existing subscriptions.
- A Customer can hold **multiple concurrent subscriptions** (e.g. across Product Lines) → gets one invoice per subscription, unless invoice consolidation is turned on.
- Billing is calculated **per pricing component**, then summed into a single subscription invoice.

---

## 2. Metering — turning raw events into a billable number

This is the part most relevant to token-based billing.

### 2.1 The event

You send raw, **unaggregated** events via `POST /api/v1/events/ingest` (1–100 events/request). Meteroid does the aggregation — you must NOT pre-aggregate on your side, or double-counting/incorrect totals will occur.

```json
{
  "events": [
    {
      "event_id": "01J...ULID",          // unique, dedup key
      "code": "llm_tokens",               // must match a Billable Metric's Event Code
      "customer_id": "cus_123",           // Meteroid ID or your external alias
      "timestamp": "2026-07-16T10:30:00Z",// RFC3339, defaults to ingest time
      "properties": {                     // arbitrary string k/v — used for filters/segmentation/grouping
        "model": "claude-sonnet-5",
        "token_type": "input",
        "tokens": "184320",
        "project_id": "proj_9"
      }
    }
  ],
  "allow_partial_failures": false,
  "allow_backfilling": false
}
```

Mechanics that matter for correctness:
- **Deduplication key = `(event_id, customer_id)`.** Re-sending the same pair is a no-op for counting purposes; if timestamps differ across duplicates, the *latest* timestamp wins. This means `event_id` should be a stable idempotency key on your side (UUID/ULID per logical event — e.g. per LLM completion call), not per-retry.
- **Timestamp window**: must be within 24h in the past → 1h in the future by default. Set `allow_backfilling: true` to import older historical events (needed for migrations/backfills).
- **Batch validation**: by default one bad event fails the *whole batch*. Set `allow_partial_failures: true` to get partial success + a `failures[]` array with `event_id` + `reason` per rejected event.
- **`properties` values are all strings** — even numeric ones like token counts. Cast on ingestion.

### 2.2 Billable Metrics — the aggregation layer

A **Billable Metric** = Event Code + Aggregation Type + (optional) Conversion Factor + (optional) Segmentation + (optional) Grouping + (optional) Filters. Meteroid re-aggregates from raw events **at the end of each billing period** — this is computed on demand from the event store, not by incrementally mutating a counter (which is why sending pre-aggregated numbers breaks things: they'd get summed again).

**Aggregation types** (applied to the numeric value carried in the event, typically via a designated property):
| Type | Behavior | Example use |
|---|---|---|
| Count | # of matching events | API calls, requests |
| Count Distinct | # of unique values of a property | unique active users |
| Sum | Σ of a numeric property | **total tokens consumed**, GB transferred |
| Mean | average of a numeric property | avg response time |
| Min / Max | extremes | peak concurrent usage |
| Latest | most recent value | current storage snapshot |

For LLM token billing, you'd almost always use **Sum** on a `tokens` property, with a metric per token type (`llm_input_tokens`, `llm_output_tokens`) or a single metric segmented by `token_type`.

**Conversion Factor**: raw value is **divided** by this before pricing. E.g. metric reports usage in bytes, factor = 1024 → billed in KB. For tokens, you might set a factor if you want to bill per 1,000 or 1,000,000 tokens instead of per single token (equivalent to what OpenAI/Anthropic pricing pages show as "$/1M tokens") — though this can also be handled purely in the price-per-unit instead.

**Segmentation** (defines independent pricing axes, not just labels):
- *None* — single price for the whole metric.
- *Single dimension* — e.g. price varies by `model` (`claude-sonnet-5`, `claude-opus-4-8`, ...).
- *Two-dimensional independent* — every combination of two dimensions needs its own price (e.g. every model × every region).
- *Two-dimensional dependent* — restrict to valid combinations only (e.g. `model → [allowed regions]`), avoiding the combinatorial price table blow-up.

This maps well onto multi-provider/multi-model token billing: segment by `model` (or `provider` + `model` as dependent dimensions), then use **Matrix pricing** (§3) to assign a $/unit rate per cell.

**Grouping key** (e.g. `project_id`, `tenant_id`, `cluster_id`): usage is bucketed and shown as separate invoice line items per group — critical for multi-tenant / per-project cost breakdown. ⚠️ **Groups are evaluated independently** — if you use tiered pricing, tier thresholds apply *within* each group, not on the summed total across groups. This matters a lot: if you group by `project_id` and use tiered pricing expecting volume discounts across a customer's total usage, you won't get them — each project resets to tier 1.

**Filters**: restrict which events count toward the metric at all, based on `properties` matching (e.g. only count events where `token_type == "output"` in metric A, `token_type == "input"` in metric B — an alternative to segmentation when you want fully separate metrics rather than one segmented metric).

**Metric mutability**: Event Code and Aggregation Type are **immutable post-creation** — to change either, archive and recreate. Segmentation values can be edited in place (add a new model/provider) without breaking existing Plan configs.

---

## 3. Pricing models — the calculation formulas

Plans are made of **Products**, and each Product has exactly one pricing model. A Plan = 1..N Products, each contributing one or more invoice line items. Prices are configured **before tax**.

### 3.1 Subscription rate (flat fee)
Fixed charge per billing period (monthly/quarterly/annual), billed **in advance** (start of period), independent of usage. `charge = flat_rate`.

### 3.2 Slot-based (seats/licenses)
Billed in advance. `charge = rate_per_slot × number_of_slots`.
- **Add a slot mid-period** → pro-rated charge for that slot for the remainder of the period, using the *average* per-slot rate (`total_rate / total_slots`) if the addition pushes past the pre-defined slot count. Triggers an immediate dedicated invoice.
- **Remove a slot mid-period** → no refund for the current period (customer already paid for the full period); the reduced count only takes effect on the *next* invoice.

### 3.3 Capacity commitment (committed-then-overage)
Customer pre-buys a package (a fixed unit allowance + tier price); if actual usage exceeds it, overage is billed per-unit. Billed **in arrears** (end of period), because actual usage must be known first.

```
if usage <= package.included:
    charge = package.tier_price
else:
    charge = package.tier_price + (usage - package.included) × package.per_unit_overage
```

Worked example (from docs):
| Package | Included | Tier Price | Overage/unit |
|---|---|---|---|
| Package 1 | 100 | $10 | $0.10 |
| Package 2 | 200 | $18 | $0.05 |

150 units consumed → Package 1: `$10 + 50×$0.10 = $15`. Package 2: `$18` flat (within included).

### 3.4 Usage-based (pay-as-you-go) — 5 sub-models, billed in arrears

**Per unit**: `charge = usage × unit_price`. (5 units × $100 = $500)

**Tiered** (graduated — different rates for different *slices* of usage, like income tax brackets):
```
charge = Σ over each tier: (units_falling_in_that_tier × tier_rate)
```
Example: 18 units consumed across tiers [0–10]@$300/unit, [11–15]@$200/unit, [16+]@$100/unit →
`10×$300 + 5×$200 + 3×$100 = $4,300`.

**Volume**: unlike tiered, the **entire quantity** is billed at the rate of the tier the *total* falls into (no slicing).
```
charge = total_usage × rate(tier containing total_usage)
```
Example: 18 units all at the $100/unit tier (because 18 falls in that bracket) → `18 × $100 = $1,800`. Volume pricing is more "cliff-y" than tiered — crossing a threshold can retroactively lower (or, at the boundary, effectively make marginal usage cheap) the whole bill.

**Package** (block pricing): billed per **block/bundle** of N units, rounded up.
```
charge = ceil(usage / block_size) × price_per_block
```
Example: 25 units/$5 block; 40 units used → `ceil(40/25) = 2` blocks → $10.

**Matrix**: price depends on 1–2 **segmentation dimensions** from the Billable Metric (e.g. model × region). Each valid dimension-combination is a matrix cell with its own $/unit rate.
```
charge = Σ over each (dimension, usage) pair: usage_for_dimension × rate[dimension]
```
This is the natural fit for **per-model, per-token-type LLM pricing** — e.g. a matrix of `model × token_type` → $/1K-tokens, mirroring how OpenAI/Anthropic publish differentiated input vs. output token prices per model.

### 3.5 One-time charge
Non-recurring line item: `charge = quantity × price_per_unit`. Appears once, not tied to a cadence (setup fees, onboarding).

### 3.6 Recurring charge
Like subscription rate but decoupled from the main plan cadence; has its own **cadence** and **billing type** (paid upfront vs. postpaid) — e.g. a monthly support package add-on billed alongside the plan.

### 3.7 Floor price (minimum billed amount)
```
invoiced_amount = max(calculated_amount, floor_price)
```
Can be set at Plan level (applies to sum of all Products) or Product level (applies to the sum of just the flagged Products). Evaluated **before** coupons/discounts — so a bill that clears the floor pre-discount but would fall under it post-discount still only gets the discount applied normally (floor is not re-checked after discount).

---

## 4. Currency & timing rules that affect the final number

- **Plan currency** is fixed at account creation (based on incorporation country) and can't change. End customers are billed in the **Invoicing Entity's** currency, converted at the **daily FX rate at invoice finalization** if it differs from the Plan currency.
- **Billing period timing** — this is the single most important "when does the number get calculated" rule:
  - **In advance** (start of period): Subscription rate, Slot-based.
  - **In arrears** (end of period, always monthly): Capacity commitment, Usage-based.
  - Practically: the Month M invoice = **Month M+1's flat/seat fees (prepaid)** + **Month M's actual usage (postpaid, now known)**. This is why an invoice can contain both a forward-looking flat charge and a backward-looking usage charge in the same document.

---

## 5. Subscriptions — the billing unit's lifecycle

A **Subscription = Customer + Plan Version + start/end dates + billing cycle + settings**. It's a frozen instantiation: editing the Plan later never touches existing subscriptions (they stay pinned to their Plan Version).

### 5.1 Billing cycle choices
- **1st of the month**: first invoice always charges the *full* month even for a mid-month signup (no proration on the first invoice under this mode).
- **Anniversary date**: billing recurs on the subscription's start-date day-of-month.

### 5.2 Status machine
`Pending Activation → (Pending Charge |) → Trial Active → Trial Expired → Active → Suspended/Cancelled/Completed/Superseded`. Notable calculation-relevant states:
- **Active** is set at `max(activation_date, payment_confirmation_date)`.
- **Suspended**: billing retries continue, but Meteroid does **not** auto-revoke entitlements — access control is your app's job via the Entitlements API.
- **Completed**: reached at the defined end date; by default the customer rolls to the Free plan if one exists.
- **Superseded**: replaced by upgrade/downgrade or migration; stops billing.

### 5.3 Activation conditions
- **On start date** — past date → immediately Active; future date → Pending until reached.
- **On Checkout** — Pending until customer completes payment.
- **Manual** — stays Pending until an Account Manager explicitly activates it.

### 5.4 Trials
Configured at Plan level, apply only at subscription **start**. A trial can:
- Suppress subscription charges only, or subscription + usage charges (trial affects both).
- Grant access to *another* plan's features during the trial (either still charged per the original plan, or fully free).
- Interact with billing-cycle-alignment: e.g. 14-day trial + anniversary billing → first invoice on day 15, recurring every 15th thereafter. Same trial + "1st of month" billing → a pro-rated stub invoice on day 15 covering the partial remainder of the month, then normal cycles from the 1st.

### 5.5 Proration events
| Event | Rule |
|---|---|
| Cancel — already invoiced for the period | No auto-refund; Account Manager must manually issue a Credit Note for unused time. |
| Cancel — not yet invoiced (future period) | Meteroid auto-generates a prorated invoice for just the used days. |
| Seat added mid-period | Prorated charge, immediate dedicated invoice. |
| Seat removed mid-period | No refund this period; lower count applies next invoice. |
| Plan upgrade/downgrade, immediate | If more owed: prorated invoice = credit(old plan, remaining days) + charge(new plan, remaining days). If less owed: net credit applied to the customer's balance, auto-consumed by future invoices. |
| Plan upgrade/downgrade, end-of-period | No proration — takes effect cleanly at next cycle. |

---

## 6. Entitlements — connecting billing to feature/quota gating

Entitlements are Meteroid's answer to "how much can this customer actually *do*", separate from "how much do they *owe*." **Meteroid does not enforce access itself** — it just exposes entitlement state via API; your app enforces it.

- **Boolean entitlement**: on/off flag tied to subscription being active.
- **Metered entitlement**: tied to a Billable Metric, with a **limit** + **reset period**. This is a *quota*, independent from billing — you can meter-and-bill tokens via a Billable Metric AND simultaneously cap usage via a Metered Entitlement on the same metric.

**Reset strategies** (important — pick based on what "period" should mean):
1. **Never** — lifetime cap.
2. **Billing Cycle** — resets with the subscription's invoice cycle.
3. **Calendar** — resets on absolute calendar boundaries (all customers reset together, e.g. every 1st of month) — decoupled from each subscription's own start date.
4. **Fixed Window** — resets on a period relative to *that subscription's* start date (e.g. every 3rd of the month if it started June 3rd) — decoupled from billing cycle, but still per-subscription anchored.
5. **Sliding Window** — true rolling window (e.g. usage over the trailing 1 hour, continuously evaluated) — this is the one to use for **rate limiting**, as opposed to billing-period quotas.

**Override precedence**: `Default (Feature config) → Plan override → Subscription override` (most specific wins) — lets you grant a specific enterprise customer a bespoke token quota without creating a whole new Plan.

---

## 7. Usage query APIs (what you'd poll/display)

| Endpoint | Purpose |
|---|---|
| `GET /api/v1/usage/subscription/{id}` | Aggregated usage for one subscription's usage-based components, optionally filtered to one `metric_id`, defaults to current billing period if no dates given. Returns `total_value` plus `grouped_usage[]` (per grouping-key breakdown, e.g. per `project_id`). |
| `GET /api/v1/usage/customer/{id}` | Same, aggregated across a customer (possibly multiple subscriptions). |
| `GET /api/v1/usage/summary` | Tenant-wide aggregated usage across all customers. |

All return decimal strings (not floats) — treat as arbitrary-precision, don't parse into a native float in your own reconciliation code or you'll get rounding drift versus Meteroid's invoice totals.

**Invoice preview**: subscriptions expose an "Upcoming Invoice" view that recalculates in real time from ingested-to-date events — useful for a live cost dashboard (e.g. "here's what you owe so far this month for token usage").

---

## 8. Invoices — where all the above resolves into a bill

- One invoice **per subscription per billing period** (optionally consolidated across subscriptions sharing billing properties).
- **Draft → Finalized → (Voided | Uncollectible)**, with **Paid/Partially Paid/Unpaid/Errored** payment status tracked in parallel.
- Draft window = end of billing period → end of configured Grace Period. Auto-advances to Finalized unless "Auto-Advance" is disabled (then a human must finalize).
- Once Finalized, immutable (accounting document) — corrections go through a **Credit Note**, not an edit.
- Line items map 1:1 to pricing components; sub-lines (tier breakdowns, per-group usage) are system-generated and not manually editable.
- Coupons: fixed-amount or percentage, applied at **subscription** level (not customer level — to discount a customer's entire relationship you must apply the coupon to each of their subscriptions).

---

## 9. Putting it together — worked LLM token-billing example

Say you're metering `claude-sonnet-5` input/output tokens per customer, grouped by `project_id`, with a tiered discount on total monthly output tokens.

**Setup:**
- Billable Metric `llm_tokens`: Event Code `llm_tokens`, Aggregation = Sum (on a `tokens` property), Segmentation = single dimension on `token_type` (`input`,`output`), Grouping key = `project_id`.
- Product `Sonnet-5 Usage`: Usage-based, metric = `llm_tokens`, pricing = **Tiered** on the `output` segment (e.g. 0–1M tokens @ $3/1M, 1M–10M @ $2.50/1M...), **Per-unit** on the `input` segment (flat $1/1M tokens — no volume discount).
- Plan `Pro`: Subscription rate $49/mo (in advance) + the above usage Product (in arrears).

**Runtime:** your app sends one event per completion call:
```json
{"event_id":"uuid-per-call","code":"llm_tokens","customer_id":"cus_123",
 "properties":{"token_type":"output","tokens":"842","project_id":"proj_9"}}
```

**At period end:** Meteroid sums all `output` events → total per project (since grouped) → ⚠️ tiered thresholds apply **per project_id group independently**, not to the customer's grand total across projects. If you actually want the discount to apply across the customer's combined usage, don't group by `project_id` for pricing purposes — instead, put `project_id` only in `properties` for invoice *display/attribution* without configuring it as the metric's grouping key, or use it only as a filter/segment where independence-per-bucket is the intended behavior.

**Invoice for Month M** (arrives ~Month M+1, in advance/arrears combined): `$49 (Month M+1 subscription, prepaid)` + `tiered charge on Month M's output tokens` + `per-unit charge on Month M's input tokens`, each as separate line items, with per-project sub-lines under the usage line.

---

## 10. Contrast points worth flagging for your own platform (QuantumBilling)

- Meteroid's **"send raw events, we aggregate at period-end"** model matches your Go/Kafka/ClickHouse ingestion path conceptually — the key discipline is the same: never pre-aggregate before ingestion, and dedupe on `(event_id, customer_id)` equivalent.
- The **grouping-key-breaks-tiering** behavior is a real design decision, not a bug — worth deciding explicitly whether your tiered/volume pricing should be evaluated per-tenant-total or per-sub-group (e.g. per API key, per project) before you copy the pattern.
- Meteroid's **Metered Entitlement** (quota/rate-limit) is modeled as a *separate concept* from the Billable Metric used for pricing, even though both read from the same event stream — useful precedent if QuantumBilling's `meter` module currently conflates "what we charge for" with "what we cap."
- Plan **immutability via versioning** (existing subscriptions pinned to the version they started on) is the standard safe pattern for your `subscription` module to avoid retroactively changing a live customer's price when a plan is edited.
