# OpenMeter — Complete Internal Reference Guide
### Metering, Product Catalog, Pricing, Entitlements, Subscriptions & Invoicing

> Compiled from official OpenMeter documentation (openmeter.io/docs), organized end‑to‑end for someone learning the platform from scratch.
> Note: OpenMeter Cloud has been rebranded as **Kong Konnect Metering & Billing** (Kong acquired/absorbed OpenMeter). The open-source core and concepts documented here remain the same.

---

## 1. What is OpenMeter?

OpenMeter is a **real-time, event-based usage metering and usage-based billing platform**. It answers two questions for any SaaS/API/AI product:

1. **"How much did this customer use?"** → Metering
2. **"How much do I charge them for it, and how do I invoice them?"** → Product Catalog + Billing

It is built to solve exactly the kind of problem you're solving with QuantumBill: ingest raw usage events (API calls, AI tokens, compute time, seats), aggregate them accurately in real time, attribute them to the right paying customer, enforce limits/quotas, price them according to a plan, and generate invoices.

### The mental model (top to bottom)

```
Event (raw usage, e.g. "5000 tokens used")
   ↓ aggregated by
Meter (defines HOW to aggregate: SUM, COUNT, etc.)
   ↓ tracked per
Subject (who/what generated the usage — a user, device, service)
   ↓ mapped to
Customer (who PAYS — a company/billable entity)
   ↓ governed by
Feature (the sellable capability — "AI Tokens", "SSO")
   ↓ priced/limited by
Entitlement + Rate Card (limits + price for that feature, per customer)
   ↓ bundled into
Plan (a versioned collection of Rate Cards — e.g. "Pro Plan")
   ↓ assigned via
Subscription (a customer's live instance of a Plan)
   ↓ produces
Invoice (the bill, calculated from usage × price, with discounts/taxes)
```

Everything downstream (pricing, invoicing) is *derived* from what happens upstream (events → meters). Get the event/meter design right first — that's the foundation everything else stands on.

---

## 2. Metering Layer — How Usage Is Measured

### 2.1 Events — the raw input

OpenMeter ingests usage as **CloudEvents** (a CNCF-standard event format), in JSON. Every usage event you send looks like this:

```json
{
  "specversion": "1.0",
  "type": "tokens",
  "id": "00001",
  "time": "2024-01-01T00:00:00.001Z",
  "source": "service-0",
  "subject": "customer-1",
  "data": {
    "total_tokens": "123",
    "model": "gpt-4",
    "type": "output"
  }
}
```

| Field | Meaning |
|---|---|
| `specversion` | CloudEvents spec version, currently `1.0` |
| `type` | Event type — used to match the event to a **meter** (`eventType`) |
| `id` | Unique ID of this specific event (used for deduplication) |
| `time` | RFC3339 timestamp; defaults to ingestion time if omitted |
| `source` | Which service/system emitted the event |
| `subject` | **Who generated this usage** — typically your internal customer/user ID. Meter values are aggregated *per subject*. |
| `data` | Arbitrary JSON payload — the actual usage numbers and metadata (e.g. token counts, model name, route, method) |

**Deduplication**: OpenMeter treats `source + id` as the unique key of an event. If you (or your retry logic) send the same event twice, it is processed only once. This makes it safe to retry sends on network failure.

### 2.2 Subjects — "who used it"

A **Subject** is metering-layer concept: whoever/whatever produced the usage — a user ID, a device ID, a service name, a hostname. It's deliberately generic so OpenMeter can be used for users, orgs, devices, or services alike.

- Usually, **one subject = one customer**, but not always.
- A single **Customer** (billing concept) can have **multiple Subjects** mapped to it — e.g. a company customer "ACME Inc." might have subjects `department-1` and `department-2`, and usage from both departments rolls up into ACME's single bill. This is called **Usage Attribution**.
- OpenMeter Cloud auto-creates a subject the first time it sees a new subject key in an ingested event; you can also explicitly upsert/delete subjects via SDK.

### 2.3 Meters — "how to aggregate the raw numbers"

A **Meter** is the aggregation definition applied to incoming events. This is the core configuration piece for anything you want to track and bill for — like your "input tokens" question.

```yaml
meters:
  - slug: api_requests_total       # unique ID, used in API calls
    description: API Requests
    eventType: request             # filters events by CloudEvent 'type'
    valueProperty: $.duration_seconds   # JSONPath into event.data
    aggregation: SUM
    groupBy:
      method: $.method
      route: $.route
```

Key properties:

| Property | Notes |
|---|---|
| `slug` | Unique meter ID (immutable once created) |
| `eventType` | Which CloudEvent `type` this meter listens to (immutable) |
| `valueProperty` | JSONPath to the numeric value inside `data` (immutable). Not needed for `COUNT`. |
| `aggregation` | One of `SUM`, `COUNT`, `UNIQUE_COUNT`, `LATEST`, `MIN`, `MAX`, `AVG` (immutable) |
| `groupBy` | Map of JSONPaths to break usage down by extra dimensions, e.g. `model`, `method` (mutable, can be updated later) |
| **Time window** | **Not part of the meter definition.** It's a parameter you pass at *query* time (`windowSize`). Omit it to get one aggregate value for the whole period. |

Only **basic scalar JSONPaths** are supported (`$.property`, `$.nested.property`) — no filters, wildcards, or script expressions.

#### Aggregation types

| Type | What it does | Typical use |
|---|---|---|
| `SUM` | Adds up values in the window | Total tokens used, total API calls, total minutes billed |
| `COUNT` | Counts number of events (no `valueProperty` needed) | Number of transactions/requests |
| `UNIQUE_COUNT` | Counts distinct values of `valueProperty` | Unique active users/sessions in a period |
| `LATEST` | Takes the last reported value in the window | Current disk size, seat count, resource size you report periodically |
| `MIN` | Minimum value in window | Minimum available storage |
| `MAX` | Maximum value in window | Peak concurrent jobs, peak load |
| `AVG` | Average value in window | Average request duration/payload size |

#### How OpenMeter parses `valueProperty` and `groupBy`

`valueProperty` is parsed as a **number** (strings holding numbers are accepted, e.g. `"123.45"` → `123.45`; sending floats as strings avoids precision loss). `groupBy` values are parsed as **strings** — arrays/objects returned by JSONPath are dropped to an empty string.

### 2.4 Worked example: event processing (why this matters for token metering)

Meter definition:
```yaml
meters:
  - slug: api_requests_total
    eventType: request
    valueProperty: $.duration_seconds
    aggregation: SUM
    groupBy:
      method: $.method
      route: $.route
```

Event 1 (`duration_seconds: "10"`) and Event 2 (`duration_seconds: "20"`), same subject, method, route, and same 1-minute time window → OpenMeter **sums** them into a single bucket:

```
windowstart = 2024-01-01T00:00
windowend   = 2024-01-01T00:01
subject     = customer-1
duration    = 30     (10 + 20)
method      = GET
route       = /hello
```

Every unique combination of `(time window, subject, groupBy values)` gets its own running aggregate.

### 2.5 Concrete example: tracking Input Tokens vs Output Tokens for an LLM product

This directly answers "how does input token metering work." You typically send **one event type** (e.g. `tokens`) with a `data.type` field distinguishing input vs output, plus the model name:

```json
{
  "specversion": "1.0",
  "type": "tokens",
  "id": "evt-001",
  "time": "2026-07-16T10:00:00Z",
  "source": "llm-gateway",
  "subject": "customer-1",
  "data": {
    "total_tokens": "1500",
    "model": "gpt-4",
    "type": "input"
  }
}
```

**One underlying meter** (`tokens_total`) aggregates all token events with `SUM`, grouped by `model` and `type`:

```yaml
meters:
  - slug: tokens_total
    eventType: tokens
    valueProperty: $.total_tokens
    aggregation: SUM
    groupBy:
      model: $.model
      type: $.type      # "input" or "output"
```

Then, at the **Feature** level (Product Catalog, see §3), you create **two separate Features** on top of the *same meter*, each filtered to a subset using `meterGroupByFilter`:

```js
// Feature: GPT-4 Input Tokens
await openmeter.features.create({
  key: 'gpt_4_input_tokens',
  name: 'GPT-4 Input Tokens',
  meterSlug: 'tokens_total',
  meterGroupByFilters: { model: 'gpt-4', type: 'input' },
});

// Feature: GPT-4 Output Tokens
await openmeter.features.create({
  key: 'gpt_4_output_tokens',
  name: 'GPT-4 Output Tokens',
  meterSlug: 'tokens_total',
  meterGroupByFilters: { model: 'gpt-4', type: 'output' },
});
```

This lets you **price input tokens and output tokens differently** (which is how every real LLM billing system works, e.g. $X per 1M input tokens vs $Y per 1M output tokens) while only maintaining one underlying meter/event pipeline. The `meterGroupByFilter` keys must match keys already defined in the meter's `groupBy`.

---

## 3. Product Catalog — Defining What You Sell

The Product Catalog is your **pricing and packaging layer**: Plans, Add-ons, Features, Rate Cards.

### 3.1 Feature — the sellable/governable unit

A **Feature** is literally a row on your pricing page: "GPT-4 Input Tokens," "API Requests," "SAML SSO." Features are:
- **Metered** (linked to a meter, e.g. tokens, requests) — usage is tracked and can be limited/billed
- **Non-metered** (e.g. a toggle like SSO) — access-only, no usage number

```js
const feature = await openmeter.features.create({
  key: 'gpt_4_tokens',
  name: 'GPT-4 Tokens',
  meterSlug: 'tokens_total',
  meterGroupByFilters: { model: 'gpt-4' },
});
```

Features can be **archived** (soft-deprecated) — existing entitlements referencing them keep working, but no new entitlements can be created against an archived feature. An archived feature's `key` can be reused by a new feature.

### 3.2 Rate Card — price + access rules for one feature, inside a plan

A **Rate Card** = **Feature + Price + Entitlement (limit)**, as a line inside a Plan.

- Rate cards **without a feature** → flat fee only, no entitlement (no access control), key/name must be typed manually (e.g. a one-time setup fee).
- Rate cards **with a feature** → can be recurring, one-time flat, or usage-based; key/name are pre-filled from the feature; can have an entitlement (usage limit) if the feature has a meter.
- `billingCadence` on the rate card controls how often it's billed (e.g. `P1M` = monthly). Flat-fee rate cards can omit cadence → charged once per subscription phase.
- **Free items**: (a) omit price entirely, (b) set price to $0 explicitly, or (c) apply a 100% discount. Only (a) allows subscribing with *no* payment method on file; (b) and (c) still require one.
- Rate cards can carry **percentage discounts**, **usage discounts**, and **min/max spend commitments** (see §5.6).

### 3.3 Plan — versioned bundle of Rate Cards

A **Plan** (e.g. "Pro Plan") is a collection of Rate Cards, in one currency.

Example — Pro Plan:

| Feature | Price | Entitlement |
|---|---|---|
| AI Tokens | $99/month | 1,000,000/month |
| Storage | $0/month | 10 GB/month |
| SAML SSO | $0/month | true |

**Plan Versioning**: A plan can have exactly one *Published* and one *Draft* version at a time. Editing a published plan creates a new draft; publishing it makes it live. Existing subscriptions stay bound to whichever version they started on, unless explicitly migrated. Status lifecycle: `Draft → Published → Archived → Deleted`.

**Plan Phases**: A plan can have multiple time-ordered phases, each with its own rate cards — used for trials, reverse trials, ramp-ups. Example (reverse trial):
1. Phase 1 (Trial): 100,000 tokens, premium features unlocked
2. Phase 2 (Freemium): drops to 1,000 tokens

**Currency**: One currency per plan. Multi-currency support = separate plans per currency.

### 3.4 Add-Ons — optional extensions

An **Add-on** is a standalone, versioned catalog item that a customer can attach on top of a base plan — extra storage, an overage pack, SSO unlock — without touching the base plan.

- Composed of one or more rate cards (own pricing, entitlements, billing cadence).
- **Single-instance** (max 1 per subscription) or **Multi-instance** (multiple allowed).
- Add-on/plan compatibility is locked once the plan version is published.
- Add-on billing cadence **must match** the plan's billing cadence (for billing alignment).

---

## 4. Pricing Models — How the Money Is Calculated

This is the core "how is the bill calculated" section. All of these are configured as the `Price` on a Rate Card.

### 4.1 Flat Fees

- **One-time fee**: charged once (e.g. setup fee).
- **Recurring fee**: charged every interval (monthly/annual), optionally paired with a usage entitlement/limit (e.g. $199/month flat fee **and** 1,000,000 tokens/month included).

### 4.2 Per-Unit (Pay-As-You-Go)

```
Charge = usage_units × unit_price
```
Example: $0.01/token × 10,000 tokens = **$100**.

### 4.3 Tiered Pricing

Two variants, same tier table:

| First Unit | Last Unit | Unit Price | Flat Price |
|---|---|---|---|
| 0 | 1000 | $0.3 | $0 |
| 1001 | 5000 | $0.2 | $0 |
| 5001 | ∞ | $0.1 | $0 |

**Graduated Pricing** — each unit is charged at *its own tier's* rate (like income tax brackets):
```
6,000 units → (1000×$0.3) + (4000×$0.2) + (1000×$0.1)
            = $300 + $800 + $100 = $1,200
```

**Volume Pricing** — the *entire* quantity is charged at the rate of the **highest tier reached**:
```
6,000 units → 6,000 × $0.1 (rate of the tier that 6,000 falls into) = $600
```

**Flat prices inside tiers** (used for overage-style billing): a tier can carry both a `unitPrice` and a `flatPrice`.

```
Tier: 0–1000 → unitPrice $0, flatPrice $500
Tier: 1001–∞ → unitPrice $0.1, flatPrice $0

2,000 units → (1000×$0 + $500) + (1000×$0.1 + $0) = $500 + $100 = $600
```
⚠️ **Gotcha**: A flat price defined on the **first tier** (which starts at 0) is charged **even at zero usage**, because tiers start counting from 0. To charge $0 when usage is 0, split tier 1 into a "first unit" micro-tier carrying the flat fee and a "0 usage" tier with no flat fee:

```
Tier 1: units 0–1     → unitPrice $500, flatPrice $0
Tier 2: units 2–1000  → unitPrice $0,   flatPrice $0
Tier 3: units 1001–∞  → unitPrice $0.1, flatPrice $0

0 units    → $0
2,000 units → (1×$500) + (999×$0) + (1000×$0.1) = $500 + $0 + $100 = $600
```

### 4.4 Overage Fees

Modeled using a **usage discount**: the discount is deducted from measured usage *before* the per-unit price is applied.

```
Effective billable usage = metered_usage − usage_discount
Charge = effective_billable_usage × unit_price
```
Example: usage = 1,000, usage discount = 900, unit price = $0.10:
```
(1000 − 900) × $0.1 = 100 × $0.1 = $10
```
If a 10% percentage discount is *also* applied (percentage discounts always apply **after** usage discounts):
```
(1000 − 900) × $0.1 × (1 − 0.10) = 100 × $0.1 × 0.9 = $9
```

### 4.5 Package Pricing

Usage sold in fixed-size bundles rather than per unit.

```
packages_needed = ceil(total_usage / quantityPerPackage)
Charge = packages_needed × price_per_package
```

Example — package size 20 units @ $10/package:

| Usage | Packages | Charge |
|---|---|---|
| 0 | 0 | $0 |
| 20 | 1 | $10 |
| 20.1 | 2 (rounds up) | $20 |
| 98 | 5 | $50 |

Zero usage → zero charge by default (combine with a minimum spend commitment if you want to always charge for at least 1 package).

### 4.6 Dynamic Pricing (cost-plus / pass-through)

Used when the "usage" the meter tracks is **already a cost amount** (in currency), not a unit count — e.g. variable-cost SMS/telephony, or pass-through infra costs with markup.

```
Final Price = meter_value (a cost/amount) × markup_rate
```

| Markup Rate | $100 base | Result |
|---|---|---|
| 0.0 | ×0.0 | $0 |
| 0.5 | ×0.5 | $50 (rate reduces price) |
| 1.0 | ×1.0 | $100 (pass-through, no markup) |
| 1.5 | ×1.5 | $150 |
| 2.0 | ×2.0 | $200 |

Constraints: no currency conversion happens — meter cost and customer currency must match; default markup rate is `1`.

---

## 5. Entitlements & Grants — Access Control and Balances

Entitlements answer: **"Does this customer currently have access, and how much do they have left?"**

### 5.1 Three entitlement types

| Type | Purpose | Example |
|---|---|---|
| **Metered** | Usage limit + real-time balance tracking | 1,000,000 tokens/month |
| **Static** | Arbitrary per-customer JSON config, no usage tracking | `{ enabledModels: ["gpt-3","gpt-4"] }` |
| **Boolean** | Simple on/off feature flag | SAML SSO: `true` |

Rule: a customer can only have **one active entitlement per feature key** at a time. To change it (e.g. plan upgrade), delete the old entitlement and create a new one.

```js
const { hasAccess, balance, usage, overage } =
  await openmeter.customers.entitlements.value('customer-1', 'gpt_4_tokens');
```

### 5.2 Metered entitlement mechanics

- Requires a `usagePeriod` (interval: day/week/month/etc.) — this should usually match the billing cycle.
- At the start of each period, a **reset** occurs: a new tracking marker begins, and grants roll over according to their rollover rules (below).
- `isSoftLimit`: whether usage is hard-blocked at the limit or just tracked/warned.
- `preserveOverageAtReset`: if true, overage from the previous period is deducted from the new period's starting balance (so customers can't escape overage by waiting for reset).
- **Precision limit**: usage history is pre-aggregated in **1-minute chunks** — resets/grants can't have sub-minute precision, and you can only reset an entitlement **once per minute**.
- You can query **historical balance** (`burnDownHistory`, `windowedHistory`) for audits/support.

### 5.3 Grants — the actual "credits" behind a metered entitlement

A **Grant** is a single record of "X usage granted to this customer, active from T1 to T2." A metered entitlement's balance = sum of all its active, non-expired grants, burnt down by usage.

```js
await openmeter.customers.entitlements.createGrant('customer-1', 'gpt_4_tokens', {
  amount: 100,
  priority: 1,
  effectiveAt: new Date().toISOString(),
  expiration: { duration: 'MONTH', count: 1 },
});
```

**Burn-down order** (which grant gets consumed first):
1. Higher **priority** first (lower number = higher priority; range 0–255).
2. Same priority → grant **closest to expiration** burns first.
3. Same priority + expiration → **oldest created** burns first.

**Rollover** — what happens to a grant's leftover balance at reset:
```
Balance After Reset = MIN(maxRolloverAmount, MAX(Balance Before Reset, minRolloverAmount))
```
- Set `min = max = N` → balance is **topped up to exactly N** every reset (a recurring quota).
- Set `max = N`, no floor → balance **can roll over up to N**, unused portion carries forward (bounded).

**Recurrence** — top up a grant's balance at a fixed interval, independent of the entitlement's reset/usage period (e.g. +300 tokens every single day, on top of a monthly 5,000 quota).

**Example — layered grants** (use the cheap/included pool before the paid top-up pool):
- Grant 1: 10,000 tokens/month, rolls over, **priority 5** (higher priority → consumed first)
- Grant 2: 100,000 tokens/year (paid add-on), **priority 10** (consumed only after Grant 1 is exhausted)

### 5.4 Boolean & Static entitlements
Boolean: `{ type: 'boolean', featureKey: 'saml_sso' }` → `hasAccess: true/false`, no balance.
Static: `{ type: 'static', featureKey: 'gpt_models', config: JSON.stringify({ enabledModels: [...] }) }` → returns the config verbatim for your app to enforce.

---

## 6. Customers & Usage Attribution

A **Customer** is the billing entity — who pays. A customer has:
- **Name**, optional **key** (bring-your-own-ID, e.g. your DB's customer ID or CRM ID)
- One or more **assigned Subjects** — this is what maps raw usage (subject-scoped) to a bill (customer-scoped)
- A **currency** — customers can only subscribe to plans in the same currency
- A **Billing Profile** — controls tax, invoicing, payment provider, collection method/period

```js
const customer = await openmeter.customers.create({
  name: 'ACME, Inc.',
  key: 'my-database-id',
  usageAttribution: { subjectKeys: ['department1', 'department2'] },
});
```
Both `department1` and `department2`'s usage events roll up into ACME's single invoice — this is how you'd model, e.g., multiple API keys/services under one paying org.

---

## 7. Subscriptions — Turning a Plan into a Live, Billable Relationship

A **Subscription** binds a **Customer** to a **Plan** (or a fully custom, ad-hoc configuration for enterprise/negotiated deals) at a point in time.

Plan concept → Subscription equivalent:

| In a Plan (template) | In a Subscription (concrete instance) |
|---|---|
| `PlanPhase` | `SubscriptionPhase` |
| `RateCard` | `SubscriptionItem` |
| `EntitlementTemplate` | Concrete `Entitlement` (gets a real `activeFrom`, keeps the template's `UsagePeriod`) |
| `PriceTemplate` | Concrete `Price` (real timestamps, keeps the template's `BillingCadence`) |

- Subscriptions support both **self-service (PLG)** — pick a published plan — and **sales-led (PLS)** — fully custom pricing/entitlements for one customer.
- **Subscription Alignment**: by default, all items on a subscription are billed **together, on the same cycle** (simpler invoices). You can opt out to let each `SubscriptionItem` bill independently (needed for advanced staggered pricing, but adds complexity).
- Subscriptions can be **edited** (change items/entitlements going forward) or **migrated** to a newer plan version.
- **Add-ons** are attached to a specific subscription, inheriting the compatibility rules set at the plan level (§3.4).

---

## 8. Billing & Invoicing — How Usage Becomes a Bill

### 8.1 The invoice lifecycle
Invoices are created when a subscription starts, and stay in sync with subscription state (item changes, cancellations, etc.). Each Rate Card's `billingCadence` governs how often that specific item generates invoice lines (e.g. a monthly in-advance flat fee → one invoice/month).

Each invoice is an **immutable record** once finalized, containing:
- Detailed usage data (from the meters)
- All applied discounts and their calculated effect
- Any min/max spend commitments applied
- Full customer + billing profile snapshot
- Line-item level pricing breakdown

### 8.2 Gathering Invoices
Before a billing period closes, OpenMeter maintains a **"gathering" invoice** — a live, real-time preview of pending charges for the current cycle. Useful for showing customers "current bill so far" and for threshold/progressive billing.

### 8.3 Progressive Billing
If enabled, OpenMeter **invoices mid-cycle** once accrued usage crosses a configured $ threshold — even before the normal billing cycle ends. Reduces non-payment risk and improves cash flow for high-usage customers. When combined with usage discounts, a single discount pool can be spread and consumed across multiple progressive invoices (see example in §8.4).

### 8.4 Order of Operations for Discounts, Commitments & Tax

This is the **exact calculation pipeline** the invoicing engine runs, in order:

```
1. Usage discount   → subtracted from the metered (raw) quantity
2. Pricing algorithm → the rate card's price model (per-unit/tiered/package/etc.) is applied
3. Percentage discount → % taken off the resulting amount
4. Min / Max spend   → commitment floor/ceiling applied to the final line amount
5. Tax               → calculated last, on top of everything above
```

- **Usage discounts** only apply to usage-based (non-flat) lines; the discount is deducted from `meteredQuantity` before pricing runs. Under Progressive Billing, one usage discount pool can span multiple invoices — it's consumed down to zero across them (e.g. a 500-unit discount split across invoices with quantities 300/250/100 leaves discount balances of 200 → 0 → already-exhausted, with invoice 2 only getting a partial 50-unit discount and invoice 3 getting none).
- **Percentage discounts** apply to the amount *after* pricing and usage discounts.
- **Minimum spend** shows up as its own extra invoice line (category `commitment`) ensuring the customer is billed at least X regardless of usage.
- **Maximum spend** caps the line — any amount above the cap becomes a discount line (reason type `maximumSpend`).
- **Taxes** are computed last, on the post-discount, post-commitment total.

### 8.5 Payments & Tax Integrations
- **Stripe integration**: connect a Stripe account; OpenMeter creates and sends the actual Stripe invoice, and can use Stripe for tax calculation and payment collection.
- **Custom Invoicing app**: lets you plug in any other payment/invoicing provider via a generic API contract.

---

## 9. Putting It All Together — End-to-End Walkthrough

**Scenario**: An AI SaaS product ("Pro Plan" — $99/month base, 1,000,000 AI tokens included, $0.002/1K tokens overage, 10GB storage free, SAML SSO included).

1. **Meter setup**: one meter `tokens_total` (`SUM` aggregation on `data.total_tokens`, `groupBy: {model, type}`).
2. **Feature setup**: `gpt_4_tokens` feature → linked to `tokens_total`, filtered `model=gpt-4`.
3. **Plan setup**: "Pro Plan" with 3 rate cards:
   - AI Tokens: recurring $99/month flat + metered entitlement (1,000,000/month) + overage rate card ($0.002 per 1,000 tokens beyond the included amount, via usage discount = included amount).
   - Storage: $0, entitlement 10GB (static/boolean-style limit).
   - SAML SSO: $0, boolean entitlement `true`.
4. **Customer + Subscription**: `ACME Inc.` is created with subject `acme-service-key`, then **subscribed** to Pro Plan → concrete `SubscriptionItems` and `Entitlements` are instantiated with real dates, and a **Grant** of 1,000,000 tokens is issued for the month (rolling with the usage period).
5. **Runtime usage**: your app sends a `tokens` CloudEvent per LLM call, `subject: "acme-service-key"`. OpenMeter aggregates in real time; your app calls the entitlement `value()` API before/after each call to check `hasAccess`/`balance` and block/allow requests (real-time quota enforcement).
6. **End of month**: 1,200,000 tokens used (200,000 over the grant). The invoicing engine: subtracts the 1,000,000-token usage discount from the 1,200,000 metered quantity → 200,000 billable tokens → applies the $0.002/1,000 overage price → adds the $99 flat fee → applies any % discount ACME has → checks min/max spend → applies tax → finalizes the invoice (pushed to Stripe if connected).

---

## 10. Glossary (Quick Reference)

| Term | One-line definition |
|---|---|
| Event | A single raw usage record (CloudEvent JSON) |
| Subject | The metering-level "who produced this usage" |
| Meter | Aggregation rule (SUM/COUNT/etc.) applied to events over time |
| Customer | The billing-level "who pays" |
| Feature | A sellable/governable capability (row on your pricing page) |
| Rate Card | Feature + Price + Entitlement, as a line inside a Plan |
| Price | The pricing model (flat/per-unit/tiered/package/dynamic) |
| Entitlement | Per-customer access + usage-limit state for a Feature |
| Grant | A single credited amount of usage backing a metered entitlement |
| Plan | Versioned bundle of Rate Cards (e.g. "Pro Plan") |
| Plan Phase | A time-boxed stage within a plan (trial, ramp-up, etc.) |
| Add-On | Optional, separately-priced extension to a plan |
| Subscription | A Customer's live, concrete instance of a Plan |
| Invoice | The generated bill for a billing cycle |
| Progressive Billing | Mid-cycle invoicing once a $ threshold is crossed |

---

## 11. Source Documentation (for deeper reference)

- Overview: https://openmeter.io/docs
- Metering overview: https://openmeter.io/docs/metering/overview
- Usage events / CloudEvents format: https://openmeter.io/docs/metering/events/usage-events
- Creating meters: https://openmeter.io/docs/metering/guides/creating-meters
- Subjects: https://openmeter.io/docs/metering/subjects
- Product Catalog overview: https://openmeter.io/docs/product-catalog/overview
- Feature: https://openmeter.io/docs/product-catalog/feature
- Plan overview: https://openmeter.io/docs/product-catalog/plan/overview
- Rate Card: https://openmeter.io/docs/product-catalog/plan/rate-card
- Pricing Models: https://openmeter.io/docs/product-catalog/pricing-models
- Add-On: https://openmeter.io/docs/product-catalog/add-on
- Entitlements overview: https://openmeter.io/docs/billing/entitlements/overview
- Entitlement types: https://openmeter.io/docs/billing/entitlements/entitlement
- Grant: https://openmeter.io/docs/billing/entitlements/grant
- Customer: https://openmeter.io/docs/billing/customer/overview
- Subscription overview: https://openmeter.io/docs/billing/subscription/overview
- Billing overview: https://openmeter.io/docs/billing/overview
- Invoicing overview: https://openmeter.io/docs/billing/invoicing/overview
- Discounts & Commitments: https://openmeter.io/docs/billing/invoicing/discounts
