# Lago — Metering & Usage-Based Billing: Complete Reference

*A from-scratch walkthrough of how Lago turns raw usage events into invoices — covering billable metrics, events, plans, charges, subscriptions, and a worked LLM token-billing example.*

---

## 1. The mental model

Lago is built around one pipeline. Everything else in the product is a variation on this flow:

```
Your App  →  Event  →  Deduplication  →  Match to Subscription/Plan  →  Aggregate (Billable Metric)  →  Rate (Charge)  →  Invoice
```

Four objects matter, in this order:

| # | Object | Answers the question |
|---|--------|----------------------|
| 1 | **Event** | "What just happened?" (raw usage fact) |
| 2 | **Billable Metric** | "How do I turn a pile of events into one number?" (aggregation) |
| 3 | **Plan / Charge** | "How much does that number cost?" (pricing) |
| 4 | **Subscription** | "Which customer, on which plan, for which period?" (billing context) |

Lago deliberately **decouples metering from rating**: what you measure (the billable metric + its filters) is independent from how you price it (the charge on a plan). You can add a new pricing dimension without touching your event schema, and change a price without changing what you track.

---

## 2. Events — the raw input

An event is a single fact: "this subscription consumed X units of Y, with these attributes, at this time."

### 2.1 Event schema

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

| Field | Required | Purpose |
|---|---|---|
| `transaction_id` | Yes | A unique ID **you** generate. Used for deduplication — if the same ID arrives twice, only the first is billed. Best practice: make it deterministic and traceable, e.g. `inf_20240314_cust42_gpt4_00831`, not a random UUID. |
| `external_subscription_id` | Yes | Ties the event to a specific subscription, which is how Lago knows which plan/pricing to apply. If it doesn't match an active subscription, the event is stored but **not** billed. |
| `code` | Yes | Maps to a **billable metric** you created (e.g. `llm_tokens`, `api_calls`). Treat this as a stable contract — if you rename the metric, you must update the events too. If the code doesn't match any billable metric, the event is stored but not billed. |
| `timestamp` | No | Unix timestamp (seconds, or `.` for millisecond precision) of when usage happened. Used to place the event in the correct **billing period** — not when it arrived. If omitted, reception time is used. |
| `properties` | No | Key–value payload carrying whatever you might price on: token counts, region, instance type, etc. Any JSON type is fine (string, int, float, uuid). **Unused properties are ignored, not rejected** — so it's good practice to send more dimensions than you currently price on. |
| `precise_total_amount_cents` | No | Escape hatch: skip Lago's aggregation entirely and hand it the exact monetary amount yourself (as a string, to avoid float rounding). Used for `dynamic` charges or when your own backend already computed a complex fee (e.g. tiered take-rates). |

### 2.2 Idempotency & deduplication

Lago guarantees **exactly-once billing** via `transaction_id`:
- Same `transaction_id` + `external_subscription_id` sent twice → only the first counts.
- This holds **across delivery methods** (a REST event and a Kafka event with the same ID won't double-bill).
- This means retries are always safe — timeout on REST? Resend. Kafka consumer restarted? Reprocess.

> Note: on the ClickHouse-based high-throughput event store, uniqueness is `transaction_id` **+** `timestamp` together, and a matching pair **replaces** the earlier event rather than being rejected — which is how you "correct" an event on that pipeline. On the standard Postgres event store, a duplicate `transaction_id` is rejected with `422`; to correct a mistake you send a **new** `transaction_id`.

### 2.3 Edge cases you will hit in production

- **Late-arriving events**: placed in the correct historical billing period by `timestamp`. If that period's invoice is already finalized, the event is ignored for billing (unless it belongs to a `recurring` metric, in which case it rolls into the next cycle).
- **Events before subscription start** or **after termination**: ingested (stored) but ignored during aggregation/billing.
- **Backfills / historical data**: validate your timestamps carefully, since they — not arrival order — decide the billing period.

### 2.4 Delivery methods

| Method | Best for |
|---|---|
| REST API — single event | Getting started; low volume |
| REST API — batch (`/events/batch`) | Up to 100 events per request; higher throughput |
| Kafka / Redpanda | Real-time, high-volume production (needs the ClickHouse event store) |
| Amazon Kinesis | AWS-native streaming |
| Amazon S3 (newline-delimited JSON) | Historical backfills / migrations |

Default REST rate limits: **500 req/sec** for event ingestion, 200 req/sec for current-usage reads, 50 req/sec for everything else (adjustable on request). A `429` means back off — retries are always safe due to dedup.

**Design rule of thumb**: one event per billable action (3 API calls = 3 events, not one event with `count: 3`), except at very high volume where client-side pre-aggregation (e.g. one event per customer per hour) becomes necessary.

---

## 3. Billable metrics — turning events into a number

A **billable metric** defines *how a stream of events for a given `code` gets reduced to a single consumption number* for a billing period. This number is what gets priced by a charge.

### 3.1 Creating one

```json
POST /api/v1/billable_metrics
{
  "billable_metric": {
    "name": "AI Tokens",
    "code": "llm_tokens",
    "description": "Token usage across models",
    "aggregation_type": "sum_agg",
    "field_name": "tokens",
    "recurring": false
  }
}
```

- `code` — what your events must reference.
- `aggregation_type` — the math (see §3.2).
- `field_name` — which property inside `properties` gets aggregated.
- `recurring` — whether the aggregated value **persists** across billing periods (e.g. "active seats") or **resets to zero** each new period (e.g. "API calls this month"). See §3.4.

### 3.2 Aggregation types (the actual math)

| Type | What it computes | SQL-ish equivalent |
|---|---|---|
| **COUNT** | Number of times the event occurred | `COUNT(events.code)` |
| **COUNT UNIQUE** | Number of *distinct* values of a property | `COUNT_DISTINCT(properties.field)` |
| **LATEST** | Most recent value of a property in the period | `LAST_VALUE(...) OVER (...)` |
| **MAX** | Highest value of a property seen in the period | `MAX(properties.field)` |
| **SUM** | Total of a property across all events | `SUM(properties.field)` |
| **WEIGHTED SUM** | Sum of a property, prorated by the time it was "active" within the period | `SUM(value) / seconds_between_events × seconds_in_period` |
| **CUSTOM / SQL expression** | Your own formula, e.g. `properties.tokens_in + properties.tokens_out` | Custom |

**For token billing specifically**: you'd typically use `SUM` with a **custom expression** like `properties.tokens_in + properties.tokens_out` (if you want one combined number), or send `tokens_in` and `tokens_out` as two separately-filtered slices of the same metric (see §3.3) if input and output need different prices — which is the OpenAI-style model (see §7).

### 3.3 Filters & dimensions — slicing one metric many ways

Instead of creating a new billable metric every time you add a pricing dimension, you attach **filters** — a `key` + allowed `values` — to the existing metric. Events carry the matching value in `properties`; Lago routes each event's contribution to the right **charge** on the plan based on that value.

```json
"filters": [
  { "key": "model", "values": ["gpt-4", "gpt-3.5", "claude"] },
  { "key": "type",  "values": ["input", "output"] }
]
```

Filters can combine (e.g. `model` × `type` × `region`) to create a pricing "matrix" from a single metric. This is exactly how you'd model "input tokens vs output tokens, per model" without four separate metrics.

Key properties of filters:
- **Metering and rating are decoupled.** Adding a new filter *value* (e.g. a new model) doesn't require a pricing decision — you can start collecting data on it before you decide what to charge.
- **What's retroactive and what isn't:**
  | Change | Applies to not-yet-invoiced events | Applies to already-invoiced events |
  |---|---|---|
  | New filter value on existing metric | ✅ | ❌ |
  | New filter key on existing metric | ✅ | ❌ |
  | New billable metric code | ❌ (past events used the old code) | ❌ |
  | Price change on an existing charge | ✅ (self-serve) | Requires support |
- **The metric `code` and `aggregation_type` are the only things that can't change retroactively.** Everything else (filters, charge pricing) is flexible. Practical implication: define broad, future-proof metric codes on day one (e.g. `ai_inference`, not `gpt4_us_inference`), and send properties you might price on later even if you don't use them yet — Lago silently ignores properties that don't match a filter.

### 3.4 Recurring vs. metered

- **Metered** (`recurring: false`): value resets to 0 at the start of each new billing period. Typical for API calls, tokens, bandwidth.
- **Recurring** (`recurring: true`): value is carried over and adjusted period to period (e.g. "number of active seats", incremented/decremented via `operation_type: add`/`remove` in event properties for `COUNT UNIQUE` metrics). Typical for seat-based or subscription-count metrics.

---

## 4. Plans — where pricing lives

If billable metrics measure usage, **plans** decide what that usage costs, and how/when the customer is invoiced.

### 4.1 What a plan is made of

1. **Basic info** — `name`, `code`, `description`, plan-level taxes.
2. **Plan model** — billing `interval` (monthly/yearly/weekly), a base subscription `amount_cents`, whether that base fee is billed **in advance** or **in arrears**, and an optional trial period (in days — trial only ever applies to the base fee, never to usage).
3. **Fixed charges (add-ons)** — recurring fixed fees not tied to usage events (e.g. a setup fee, a support tier).
4. **Usage-based charges** — one per billable metric, each with its own charge model, pricing properties, optional spending minimum, and advance/arrears + full/prorated settings.
5. **Minimum commitment** — a floor amount applied across all invoices in a period regardless of actual usage.
6. **Entitlements** — feature access tied to the plan (separate from billing math, but often bundled with it).
7. **Progressive billing** — thresholds that trigger an invoice mid-period once cumulative usage crosses them (protects you from unpaid runaway usage).

A plan only takes effect once it's **assigned to a customer via a subscription** (§6).

### 4.2 Editing / deleting plans

- A plan can be freely edited only while **not yet linked to an active subscription**.
- Once assigned, core structural properties (billing interval, advance/arrears, proration rules) are **locked**; you can still adjust prices, add/remove charges, and tweak charge settings. Some edits will fire on the next invoice.
- To fully change a locked plan: terminate its subscriptions, or create a new plan and migrate customers.
- Deleting a plan **does not fail** if subscriptions are attached — it terminates them and generates the corresponding final invoices/credit notes automatically.

---

## 5. Charge models — the actual pricing formulas

A **charge** attaches a billable metric to a plan with a specific pricing formula (`charge_model`). This is the layer that answers "given N units consumed, how many dollars is that?"

### 5.1 Standard — flat per-unit rate

Every unit costs the same.

> $0.05 per API call × 1,000 calls = **$50**

```json
"properties": { "amount": "0.05" }
```

### 5.2 Package — fixed price per block of units, with free units

You buy in "packs." A `package_size` of 1,000 units costs `amount`, with an optional `free_units` allowance before charging starts.

> $5 per 100 units, first 100 free → 201 units = $0 (free block) + $5 (units 101–200) + $5 (unit 201, rounded up to a full package) = **$10**

```json
"properties": { "amount": "5", "free_units": 100, "package_size": 100 }
```

This is the model used in the OpenAI token-pricing template (§7): each 1,000,000-token package costs a flat `amount`.

### 5.3 Graduated — tiered marginal pricing

Different *marginal* rates per tier, like a progressive tax bracket. Optionally each tier can also carry a flat fee.

> Tier 1: $1/unit for units 1–100, Tier 2: $0.50/unit for units 101–200, Tier 3: $0.10/unit beyond that.
> 250 units = (100 × $1) + (100 × $0.50) + (50 × $0.10) = $100 + $50 + $5 = **$155**

```json
"properties": {
  "graduated_ranges": [
    { "from_value": 0,   "to_value": 100,  "per_unit_amount": "1",    "flat_amount": "0" },
    { "from_value": 101, "to_value": 200,  "per_unit_amount": "0.5",  "flat_amount": "0" },
    { "from_value": 201, "to_value": null, "per_unit_amount": "0.10", "flat_amount": "0" }
  ]
}
```

### 5.4 Volume — single rate applied to the whole quantity, based on which tier the *total* lands in

Unlike graduated (marginal), volume pricing looks at **total consumption** to pick **one** rate, then applies that single rate to *all* units, plus an optional flat fee.

> | Tier | Range | Unit price | Flat fee |
> |---|---|---|---|
> | 1 | 0–10,000 | $0.0010 | $10 |
> | 2 | 10,001–50,000 | $0.0008 | $10 |
> | 3 | 50,001–100,000 | $0.0006 | $10 |
> | 4 | 100,001+ | $0.0004 | $10 |
>
> 65,000 units falls in Tier 3 → 65,000 × $0.0006 + $10 = **$49** (the *entire* 65,000 is charged at the Tier-3 rate, not just the portion above 50,000)

### 5.5 Percentage — rate + fixed fee on a monetary property

Used when the billable metric already aggregates a **monetary amount** (e.g. `SUM(properties.amount)` on payment events), for things like payment processing fees.

Supports: a percentage `rate`, an optional `fixed_amount` per event, and optional free allowances on **number of events** and/or **total amount** before the charge kicks in (and, on premium, per-transaction min/max).

> Rate 1.2%, fixed fee $0.10/event, first 3 events or $500 free (whichever hits first).
> 4th transaction of $50 (after $450 of the $500 free-amount allowance already used) → $0.10 (fixed) + 1.2% × $50 = **$0.70**

### 5.6 Graduated percentage, Dynamic, Custom price

- **Graduated percentage**: combines graduated tiers with percentage-of-amount pricing (tiered take-rates on GMV, common for marketplaces/revenue share).
- **Dynamic**: the price per event is computed by *your* application and sent directly via `precise_total_amount_cents` on the event — Lago skips its own aggregation and just totals what you send. Useful when your pricing logic (negotiated rates, complex tiering) is easier to compute in your own backend than to express in Lago's model.
- **Custom price**: attach a SQL-like custom expression to compute the fee per event/aggregation directly in Lago.

### 5.7 Currency, decimals, and rounding

- All charges on a plan inherit the plan's currency.
- Charge `properties` (unit prices) support **up to 15 decimal places** internally (e.g. `$0.000123456789123` per token, which matters at LLM-token pricing scale).
- The **final invoiced amount** is stored and rounded to `amount_cents` (i.e., normal 2-decimal currency rounding happens only at invoicing time, not during aggregation).

### 5.8 Advance vs. arrears (billing timing)

- **In arrears** (default): the charge is invoiced **at the end of the billing period**, based on actual usage during that period. Best for anything measured cumulatively — API calls, storage, tokens, compute.
- **In advance**: the charge is invoiced **immediately when the usage event is ingested** (or, for the base subscription fee, at the start of the period). Best for discrete, instant actions like adding a seat.
- Controlled per-charge via `pay_in_advance: true|false`.

### 5.9 Full vs. prorated

Charges can be billed for the **full period regardless of when usage started/stopped**, or **prorated** to the actual number of days active within the period — relevant when a subscription starts mid-cycle (see §6.3) or a charge is added/removed mid-cycle.

### 5.10 Spending minimums & minimum commitment

- A **charge-level spending minimum** guarantees a floor for that specific usage-based charge.
- A **plan-level minimum commitment** guarantees a floor across *all* charges combined for the period — if actual usage falls short, the difference is billed as a true-up.

---

## 6. Subscriptions — customer + plan + billing period

A subscription is created by **assigning a plan to a customer**. This is the object every event and every invoice is ultimately anchored to.

```json
POST /api/v1/subscriptions
{
  "subscription": {
    "external_customer_id": "cust_123",
    "plan_code": "startup",
    "external_id": "sub_8832",
    "billing_time": "anniversary"
  }
}
```

Once active, Lago starts (a) generating invoices per the plan model and (b) accepting/matching events tagged with this subscription's `external_subscription_id` (or `external_customer_id`, if you don't need per-subscription granularity).

### 6.1 Billing cycles

- **Calendar billing** (default): billing periods align to calendar months (1st–end of month), with the *first* period prorated on a daily basis if the subscription starts mid-month.
  > $50/month plan started Aug 10 → 22 remaining days of 31 → 22 × $50 / 31 = **$35.48** for that first partial period.
- **Anniversary billing**: the period is anchored to the subscription's start date instead (e.g. always the 10th of each month), so there's no proration needed for the first period.

### 6.2 Start date behavior

| Scenario | Base fee: in advance | Base fee: in arrears |
|---|---|---|
| Start date in the **past** | Treated as already paid for current period; next invoice = usage for current period + fee for next period | Included together with usage charges in the next invoice |
| Start date in the **future** (subscription is "pending" until then) | When it activates, an invoice is generated immediately for the base fee; usage charges land on the following invoice | No invoice until the period ends; base fee + usage billed together then |

### 6.3 End date, renewal, multiple plans

- No `ending_at` → subscription **auto-renews** indefinitely.
- With `ending_at` set → terminates on that date, no renewal (webhook reminders fire 45 and 15 days before).
- A customer can hold **multiple subscriptions** simultaneously (e.g. one per workspace/project) — each needs its own `external_subscription_id` on events, and Lago consolidates all of a customer's invoices where cycles overlap.
- (Premium) **Plan overrides**: a subscription can override specific plan values (price, trial, taxes, charge properties) without creating a whole new plan — useful for negotiated enterprise deals.

---

## 7. Worked example: replicating OpenAI-style per-token billing

This mirrors exactly how you'd bill LLM input/output tokens, model-by-model, on Lago — directly relevant to a token-metered product.

**Step 1 — One billable metric for all token usage**, sliced by `model` and `type` (input/output) via filters:

```json
POST /api/v1/billable_metrics
{
  "billable_metric": {
    "name": "AI Tokens",
    "code": "ai_tokens",
    "aggregation_type": "sum_agg",
    "field_name": "tokens",
    "recurring": false,
    "filters": [
      { "key": "model", "values": ["8k", "32k"] },
      { "key": "type",  "values": ["input", "output"] }
    ]
  }
}
```

**Step 2 — One plan, with a `package` charge per model×type combination** (mirrors OpenAI's $/1,000-token pricing, expressed here as $/1,000,000 tokens per package):

```json
POST /api/v1/plans
{
  "plan": {
    "name": "OpenAI-style Pricing",
    "code": "ai_plan",
    "amount_cents": 0,
    "interval": "monthly",
    "pay_in_advance": false,
    "charges": [{
      "billable_metric_id": "<id>",
      "charge_model": "package",
      "filters": [
        { "values": { "model": ["8k"],  "type": ["input"]  }, "properties": { "amount": "0.03", "free_units": 0, "package_size": 1000000 } },
        { "values": { "model": ["8k"],  "type": ["output"] }, "properties": { "amount": "0.06", "free_units": 0, "package_size": 1000000 } },
        { "values": { "model": ["32k"], "type": ["input"]  }, "properties": { "amount": "0.06", "free_units": 0, "package_size": 1000000 } },
        { "values": { "model": ["32k"], "type": ["output"] }, "properties": { "amount": "0.12", "free_units": 0, "package_size": 1000000 } }
      ]
    }]
  }
}
```

**Step 3 — Create customer → create subscription** (standard, as in §6).

**Step 4 — Ingest usage: one event per inference call**, tagging `model` and `type`:

```json
POST /api/v1/events
{
  "event": {
    "transaction_id": "inf_20240314150900_cust42_gpt4_00831",
    "external_subscription_id": "sub_42",
    "code": "ai_tokens",
    "properties": { "total": 5000, "model": "8k", "type": "input" }
  }
}
```

**Step 5 — Monitor**: `GET /customers/{id}/current_usage?external_subscription_id=...` gives real-time, in-period consumption before the invoice is even generated — useful for showing customers a live usage dashboard.

At invoice time, Lago sums all `ai_tokens` events per `model`×`type` slice for the period, applies the matching package rate, and produces one line item per slice on the invoice.

---

## 8. Invoicing — where it all lands

- Lago automatically generates invoices per the plan's billing cadence; each generation fires an `invoice.created` webhook.
- An invoice contains: invoice number, billing period, customer info, **fees** (one line item per charge/metric slice — this is the audit trail back to raw events), and taxes.
- Invoices can be previewed before finalization (even for a not-yet-existing customer, to quote pricing), held as **drafts** for review (grace period), refreshed, voided, or downloaded as PDF.
- A **fee** is the atomic invoice line item — you can trace every dollar on an invoice back to the aggregation of specific events for a specific billable-metric slice.

---

## 9. Practical checklist — building your own metering model

1. **Pick broad, future-proof metric `code`s.** You can't retroactively change a metric's code or aggregation type on already-ingested events — but you *can* add filters, prices, and charges later.
2. **Send more properties than you price on today.** Unmatched properties are silently ignored; there's no cost to including them, and it saves you from re-instrumenting later.
3. **Make `transaction_id` deterministic and traceable** (`{type}_{date}_{customer}_{category}_{seq}`), not a random UUID — this is your deduplication key *and* your audit trail.
4. **Decide aggregation type per metric**: `SUM` for tokens/bytes/hours, `COUNT` for discrete actions, `COUNT UNIQUE` for seats, `MAX`/`LATEST` for point-in-time measures like storage snapshots, `CUSTOM`/SQL expression when you need a formula across multiple properties (e.g. `tokens_in + tokens_out`).
5. **Use filters, not new metrics**, to add pricing dimensions (new model, new region, new tier).
6. **Choose the right charge model** per the shape of your pricing: flat rate → `standard`; buy-in-blocks → `package`; marginal tax-bracket style → `graduated`; single-rate-by-total-volume → `volume`; take-rate on money → `percentage`; fully custom logic computed in your own backend → `dynamic` with `precise_total_amount_cents`.
7. **Decide advance vs. arrears** per charge based on whether the action should be billed instantly (e.g. seat added) or accumulated (e.g. tokens over the month).
8. **Test end-to-end with one event first** (`POST /events` then `GET /events/{transaction_id}`) before wiring up batch/streaming ingestion.
9. **Reconcile**: compare your own event counts against Lago's (`GET /events` filtered by subscription + time range) to catch dropped or misrouted events before they silently under-bill a customer.

---

## 10. Quick glossary

| Term | Meaning |
|---|---|
| **Event** | A single usage fact sent to Lago (`transaction_id`, `code`, `properties`, `timestamp`) |
| **Billable metric** | Defines how events with a given `code` are aggregated into one number |
| **Aggregation type** | The math used (SUM, COUNT, MAX, LATEST, COUNT UNIQUE, WEIGHTED SUM, CUSTOM) |
| **Filter** | A dimension (key + values) that slices one metric into independently-priced segments |
| **Plan** | The pricing + billing-cadence container, made of a base fee, fixed charges, and usage charges |
| **Charge** | A billable metric priced with a specific charge model on a specific plan |
| **Charge model** | The formula: standard, package, graduated, volume, percentage, graduated percentage, dynamic, custom |
| **Subscription** | A customer assigned to a plan for a billing period — the anchor for events and invoices |
| **Fee** | One invoice line item, the result of aggregating + pricing one metric slice for one period |
| **In advance / in arrears** | Billed instantly on event vs. billed at end of period |
| **Prorated** | Charge scaled to actual days active instead of the full period |
| **Recurring metric** | Value persists/accumulates across periods rather than resetting each period |

---

*Sources: Lago official documentation (getlago.com/docs) — Introduction, Ingest usage, Billable metrics & aggregation types, Filters and dimensions, Plans overview, Charge models (Standard/Package/Graduated/Volume/Percentage), Arrears vs Advance, Subscriptions (Assign a plan), Invoicing overview, and the "Clone OpenAI pricing" template.*
