# Orb (withorb.com) — Complete Calculation & Architecture Reference

> Compiled from official Orb documentation (docs.withorb.com), organized as a from-scratch reference for understanding how usage-based/hybrid billing is calculated end-to-end. Cross-referenced against QuantumBilling where useful.

---

## 1. The Core Philosophy: Query-Based Billing

Before touching any object model, this is the single idea that explains almost every design decision in Orb.

**Most billing systems (stream-mutation model):** an event arrives → a counter is incremented → the counter is what gets billed. Once incremented, the *context* of how that counter got there is lost. Late events, pricing changes, or mistakes require manual patching because there's no way to safely recompute.

**Orb's model (query-based):** every event is stored **immutably** in a columnar OLAP store. An invoice is not "the current value of a counter" — it is the **output of a fresh SQL-like query** run over raw events at the moment the invoice is needed:

> "Given this customer, this billing period, and this subscription/pricing configuration — what should the invoice look like?"

Three things are versioned and immutable, and the invoice is a pure function of them:
- **Raw events** (immutable; corrections are applied as a read-path overlay/amendment, not a mutation)
- **Metric definitions** (versioned; a metric is a query, not a running counter)
- **Pricing/subscription timelines** (versioned; every historical state is reconstructable)

**Consequences that fall out of this for free:**
- Same inputs → same invoice, every time (determinism).
- Late-arriving events don't corrupt anything — just re-run the query.
- You can define a brand-new billable metric today and retroactively apply it to historical events without re-ingesting anything.
- Every dollar on an invoice is traceable back to the specific events that produced it (full audit trail).
- Pricing simulations are just "run the query with a different pricing config" — no separate sandbox needed.

**But real-time still matters** (dashboards, rate limiting, threshold alerts), so Orb runs **two parallel pipelines**:

| Pipeline | Purpose | Latency | Source of truth for invoices? |
|---|---|---|---|
| Query-based (columnar store) | Invoice generation, revenue reporting, backfills | Minutes | **Yes — always** |
| Stream-based (Redis/MemoryDB running totals) | Usage alerts, threshold invoicing, rate limiting | Seconds | No — it's a cache/optimization only |

The stream pipeline updates a fast running total so alerts/webhooks fire in seconds. But billing **never reads from that cache** — invoices always re-derive from raw events. This is the architectural reason Orb can safely support backfills, retroactive pricing changes, and full auditability without sacrificing real-time UX.

> **This directly parallels your QuantumBilling design** — you separated a Go/Kafka/ClickHouse ingestion+aggregation path (source of truth, analogous to Orb's columnar store) from NestJS as the control-plane/BFF. The architectural lesson from Orb worth stealing: never let your real-time/alerting path (Redis) become an input to the actual billing calculation — it should only ever read from the same immutable event log ClickHouse holds.

Additionally, Orb tracks a **dependency graph** between invoice fragments and their underlying data, so when something changes (a late event, a pricing update), only the *affected* fragments are invalidated and recomputed — not the entire historical dataset. This is what makes "safe backfills at scale" possible.

---

## 2. Core Object Model

Everything in Orb is scoped to an **Account** (every user gets an isolated **test** and **live** account/environment — separate API keys, no data crossover, test mode can't hit a real payment processor).

```
Account
 └─ Customer ──────────────┐
     ├─ Event stream        │
     ├─ Subscription(s) ────┼─── Plan (template) ─── Price(s) ─── Billable Metric ─── Event filters
     │    └─ Invoice(s) ────┘         └─ Adjustments (discount/min/max)
     ├─ Credit Ledger (prepaid credits, per currency/pricing-unit)
     └─ Customer Balance (AR wallet)
```

| Object | What it is | Key idea |
|---|---|---|
| **Account** | Top-level container (live/test split) | Ingestion grace period, timezone, and branding are configured per account |
| **Customer** | The billing party | Set `external_customer_id` = your own PK so you never store Orb IDs |
| **Event** | Raw, immutable fact you send | Send **raw data**, not a pre-computed billable value (see §3) |
| **Metric** | A *query* over events → an aggregate | Not a counter. Can be redefined retroactively (see §4) |
| **Item** | What you sell (maps to tax/invoicing-provider line items) | Usually created implicitly when you create a Metric |
| **Price** | How much you charge for an Item, driven by a Metric | Has a "price model" (unit/tiered/bulk/etc. — see §5) |
| **Plan** | A named bundle of Prices + Adjustments (your "Pro"/"Enterprise" tier) | Template; can be overridden per subscription |
| **Subscription** | Customer ↔ Plan, with a lifecycle | Generates invoices; the *live* draft invoice always exists (see §7) |
| **Invoice** | The bill | Composed of line items, adjustments, tax, balance (see §9) |

---

## 3. Events & Ingestion

### 3.1 What an event looks like

An event is a **business fact**, not a billing decision. Send the raw thing that happened; decide *how it's billed* later, in the Metric/Price layer. This decoupling is deliberate — it lets you evolve pricing without re-instrumenting your product.

```json
{
  "event_name": "cluster_compute",
  "timestamp": "2022-02-02T00:00:00Z",
  "external_customer_id": "9fc80ac0-d9ff-11ec-9d64-0242ac120002",
  "properties": {
    "cluster_name": "staging-cluster-1",
    "compute_ms": 912,
    "aws_region": "us-east-1",
    "compute_tier": "async_tier_2"
  },
  "idempotency_key": "unique-per-event-id"
}
```

Rules of thumb:
- `properties` is a free-form dict of **primitives only** (string/bool/number). Numbers must fit a signed 64-bit range.
- **Send more properties than you currently need.** Anything not used for pricing today can still be used for invoice grouping, future metrics, or debugging — avoids painful backfills later.
- `idempotency_key` (= `event_id`) gives per-event dedup within the ingestion grace period — critical for safe retries from distributed producers.

### 3.2 Ingestion mechanics

- Batch endpoint, up to **500 events/request**, safe to send concurrently from multiple uncoordinated producers.
- Native capacity: **1,000+ events/sec**; scales to **tens of millions/day** without special provisioning. For higher volumes, Orb offers **hosted rollups** (pre-aggregation before it hits the columnar store) that scale to **1M+ events/sec**.
- Test-mode is throttled (2,000/min, 10 req/sec) so load testing never threatens live-mode capacity.
- Additional ingestion paths: Segment (`track` calls), Reverse ETL (Census/Hightouch), S3/GCS drop, or streaming infra (Kinesis/Cloudwatch → Lambda → Orb).

### 3.3 Corrections: backfills & amendments

Because events are immutable, corrections are **layered on top**, never destructive:
- **Amend event**: overwrite a single event by its `event_id`.
- **Deprecate event**: soft-delete a single event.
- **Backfill**: a 3-step flow (create → let it accumulate corrected/added events → **close**) that makes the correction visible; Orb then asynchronously recalculates every invoice/usage graph affected. Backfills can even be **reverted**.
- The **ingestion grace period** (usually ~12h, configured per account) is the max amount a `timestamp` can lag wall-clock time — and, as a side effect, it's also the *minimum* time Orb waits before issuing an end-of-period invoice, so trailing events have a chance to land.

---

## 4. Billable Metrics — Turning Events into Billable Quantities

A **billable metric** = **event filter** (which events count) + **aggregation** (how they're combined) → a single aggregate number, evaluated **per customer, per billing period**.

```
event_name = 'transaction_processed' AND
capture_status = 'success' AND
(payment_method = 'card' OR payment_method = 'ach')
──► aggregate: SUM(amount_cents)
```

Key rules:
- Metrics measure **usage**, never dollars. If a call costs $3, the metric counts calls — the **Price** object is what multiplies by $3. This separation is what lets the same metric power multiple prices/plans.
- Filter logic is standard boolean AND/OR with parentheses for grouping.
- Metric names are **internal only** — customers never see them; only Price/line-item names appear on invoices.
- For anything the filter+aggregate model can't express, Orb supports **fully custom SQL metrics** (arbitrary aggregation, joins across event properties, computed properties, parameterization).
- Because metrics are just queries, you can invent a new one **today** and it applies to **all historical events retroactively** — no re-ingestion, no backfill needed for the metric itself (only needed if the *events* themselves were wrong).

---

## 5. Pricing Models — How a Quantity Becomes a Dollar Amount

Every `Price` has a `model_type` and a quantity (from a Metric, or a fixed quantity for flat fees). Below are all the standard models with the exact math.

### 5.1 Unit
Linear: `amount = quantity × unit_amount`
```json
{ "model_type": "unit", "unit_config": { "unit_amount": "0.50" } }
```
$0.50 per API call, no matter the volume.

### 5.2 Tiered
Each **unit** is charged at the rate of the tier range it falls into (progressive, like an income tax bracket). `first_unit` exclusive, `last_unit` inclusive.
```json
{ "model_type": "tiered", "tiered_config": { "tiers": [
  { "first_unit": 0,  "last_unit": 10,   "unit_amount": "0.50" },
  { "first_unit": 10, "last_unit": null, "unit_amount": "0.10" }
]}}
```
150,000 calls at tiers (0–10K @ $0.001, 10K–100K @ $0.0008, 100K+ @ $0.0005):
```
10,000 × 0.001  = 10.00
90,000 × 0.0008 = 72.00
50,000 × 0.0005 = 25.00
                  ------
Subtotal       = 107.00
```

### 5.3 Bulk (aka Volume)
Unlike tiered, **the total quantity decides ONE rate applied to ALL units** (cliff pricing, not progressive).
```json
{ "model_type": "bulk", "bulk_config": { "tiers": [
  { "maximum_units": 10,   "unit_amount": "0.50" },
  { "maximum_units": 1000, "unit_amount": "0.40" }
]}}
```
101 units → all 101 priced at $0.40 = $40.40 (not $0.50 for the first 10 + $0.40 after, as tiered would do).

### 5.4 Package
Usage is rounded **up** to a package granularity before pricing.
```json
{ "model_type": "package", "package_config": { "package_amount": "0.80", "package_size": 10 } }
```
4 units billed as if 10 were used; 11 units billed as if 20 were used.

### 5.5 Dimensional pricing
A price matrix keyed by one or more **event properties** (dimensions) — e.g. `region` × `instance_type` × `cloud_provider`. Modern replacement for legacy **Matrix pricing** (which caps out at 2 dimensions and is deprecated). Backed by a `dimensional_price_group` that partitions the billable metric's output by the chosen dimension set; each combination gets its own unit price. Unlimited dimensions supported.

### 5.6 Matrix pricing (legacy, ≤2 dimensions)
```json
{ "model_type": "matrix", "matrix_config": {
  "default_unit_amount": "3.00",
  "dimensions": ["cluster_name", "region"],
  "matrix_values": [{ "dimension_values": ["alpha", "west"], "unit_amount": "2.00" }]
}}
```
Falls back to `default_unit_amount` if the event's dimension values don't match any configured combination. Dimension values must be sent as strings.

### 5.7 License pricing
Models **seat/entitlement-based** billing — a license can represent a seat, a bot, an agent, a workflow, a machine, etc. Each license can carry its **own credit allocation** (for fair-usage enforcement/alerting per-entitlement) while still participating in the subscription's shared credit pool and standard overage billing.

### 5.8 Fixed fees
Not usage-driven at all — `unit` model with an explicit `fixed_price_quantity` (e.g., a flat platform fee of `$2.00 × 3 units = $6.00`, or simply `$500/mo`).

### 5.9 Billing timing (orthogonal to the model above)

| Dimension | Options | Meaning |
|---|---|---|
| **Billing mode** | In-arrears / In-advance | *When in the period* the charge is calculated — end of period (usage) vs. start of period (seats/platform fees) |
| **Cadence** | Monthly / Quarterly / Semi-annual / Annual / One-time / Custom | *Length* of the billing period |

**Critical rule**: prepaid credits can only ever be applied to **in-arrears** charges. In-advance fees are charged immediately and never draw from the credit balance — this is true regardless of whether the in-advance charge is usage-based or a fixed fee.

---

## 6. Plans, Adjustments & Trials

A **Plan** = a bundle of Prices (+ Adjustments), representing your "list pricing" (Free/Pro/Enterprise). A **Subscription** binds a Customer to a Plan and may **override** specific prices/adjustments for that one customer without forking the whole plan.

### 6.1 Adjustments
Applied to one price or a set of prices; must share the same currency, cadence, and billing mode to be combined.

| Type | Effect | Notes |
|---|---|---|
| **Usage discount** | Reduces the **quantity** before pricing | e.g. "first 300 units free" — compatible with any pricing model |
| **Amount discount** | Subtracts a flat $ | e.g. "$100 off" |
| **Percentage discount** | Subtracts a % of subtotal | Can expire after N billing periods |
| **Minimum** | Floors the (line item or invoice) total | Orb auto-adds a "true-up" line for the shortfall |
| **Maximum** | Caps the total | Orb auto-applies a discount to bring it down to the cap |

All can optionally **expire after N billing periods** — ideal for contract terms like "20% off, year one only."

### 6.2 Plan phases
Encode a pre-agreed schedule of changes (e.g., discounted first 6 months, then standard pricing) directly into the plan, instead of manually editing the subscription later. Phase switches always land on a period boundary — one invoice never spans two phases. The final phase is always "evergreen" (runs indefinitely).

### 6.3 Trials
- Configured with a **duration in days** + an optional **maximum usage discount** ($ cap on how much usage is forgiven).
- **Fixed fees are always excluded** during a trial — only usage-based charges are discounted (up to the cap).
- A single subscription can only ever have **one** trial, even across plan changes.
- Maximum usage discount cannot be $0 — use a 100% discount instead if you want "track usage, charge nothing."
- Trials don't support "N days OR N credits, whichever first" natively — model that instead with a separate trial plan + credit-balance alerts.

### 6.4 Billing cycle vs. invoicing cycle
These can differ! A **quarterly** price can still generate **monthly** invoices (invoicing cycle = monthly, billing cycle = quarterly) — Orb prorates the tiered/usage calculation correctly across the sub-periods so a quarterly tiered price billed monthly ($10/$20/$20 across 3 months) sums to the *same total* as if billed all at once at quarter's end ($50), rather than resetting the tier thresholds every month (which would wrongly total only $30).

---

## 7. Subscriptions — Lifecycle Mechanics

### 7.1 Term vs. billing period
- **Term** = renewal cadence = the **longest** cadence among the subscription's component prices.
- **Billing period** = invoicing frequency = the **shortest** cadence among the component prices.

| Component prices | Term | Billing period |
|---|---|---|
| 2 monthly usage charges | Monthly | Monthly |
| 2 monthly usage + annual platform fee | Annual | Monthly |
| Quarterly service + annual fee | Annual | Quarterly |

### 7.2 Status
`upcoming` (start date in future — usage events won't attach to any charge) → `active` → `ended`.

### 7.3 Billing cycle alignment
Three choices: (1) calendar-anchored (bill on the 1st, prorate the first invoice), (2) anchored to subscription creation date (avoids proration but bills on an odd day, e.g. the 14th), or (3) fully custom anchor date.

### 7.4 Cancellation
- **Immediate** or **end of term**. End-of-term uses the **longest** cadence among all prices (so a customer with an annual commitment finishes the full year even if their usage price is monthly).
- **Backdated cancellations** are allowed only if no paid invoices exist between the action date and the effective date. Draft invoices are adjusted in place; issued invoices after the cancel date are voided or credit-noted; already-paid in-advance fees are prorated back onto the customer balance.
- **Uncancelling** is lossy — truncated price intervals aren't automatically restored to "infinity," and anything scheduled after the cancellation point is permanently lost. For complex subscriptions, prefer a fresh plan change over uncancelling.

### 7.5 Plan changes & proration
A plan change **versions** the subscription (same ID, full audit trail of every plan it was ever on) and typically produces **two invoices**: one closing out usage on the old plan, one up-front for the new plan.

**In-advance fee proration example** (bills on the 1st, day-based proration):

| Date | Action | Orb behavior |
|---|---|---|
| 07-01 | Subscribe to Intermediate ($100/mo) | Invoice $100 |
| 07-04 | Change to Advanced ($500/mo) | Invoice usage accrued on Intermediate; credit-note the 07-01 invoice for 28 unused days ($90.32); invoice 28 days of Advanced ($451.61), draw down the new balance to net-charge $361.29 |
| 07-11 | Change to Beginner ($50/mo) | Invoice usage on Advanced; credit 21 unused Advanced days ($338.71) to balance; invoice 21 days of Beginner ($33.87), fully absorbed by balance — $0 charged, $304.84 balance remains |

Proration math is always **day-based**: `refund = unused_days / period_days × original_fee`.

---

## 8. Credit Systems — Prepaid Credits vs. Customer Balance

Orb deliberately keeps these **two separate systems** because they serve different accounting purposes.

| | **Prepaid Credits** (Credit Ledger) | **Customer Balance** |
|---|---|---|
| Purpose | Usage commitment / consumption tracking | AR adjustment ("wallet") |
| Applied | **Before tax**, during invoice calc | **After tax**, at issuance |
| Revenue recognition | Yes — recognized as consumed | No impact |
| Expiration | Supported (per-block) | Never expires |
| Currency | Real or **virtual/custom** pricing units | Customer's billing currency only |
| Typical use | Prepaid usage bundles, trial tokens, enterprise drawdown | Refunds, goodwill credits, rounding carryover |

### 8.1 Prepaid credits (the token/credit-ledger model)
- Structured as an **append-only ledger** of **blocks**, each with an `amount`, `expiry_date`, and optional **cost basis** (per-unit $ value — $0 = promotional, non-zero = must be invoiced for proper revenue recognition).
- **Only applies to in-arrears charges** — this includes in-arrears fixed fees too, not just usage.
- **Deduction order** (deterministic, in this priority order):
  1. License allocations (dedicated per-license credit first)
  2. Blocks scoped to specific items (item-filtered blocks) before unscoped blocks
  3. Soonest-expiring blocks first
  4. Zero/lower cost-basis blocks before higher cost-basis (so free credits burn before paid ones)
  5. FIFO by creation time as the final tiebreaker
- Deductions are recorded **grouped by day/price/invoice/credit-block**, not 1:1 per event — because pricing can be non-linear (bulk/volume tiers depend on total period usage) and adjustments (minimums) aren't fully resolved until period close. If you need **event-level** attribution (per `user_id`, per `api_key`), use the `evaluate prices` endpoint instead of the ledger.
- Entries are **pending** during the reporting grace period, then **committed** (immutable) — this correctly handles a late event that should have deducted from a block *before* that block expired.

### 8.2 Customer balance
- Simple digital wallet, single currency, applied **after tax**, **no revenue impact**.
- Used for: refunds via credit notes, goodwill credits, manual corrections, small-balance carryovers.
- **Never mix the two** — using customer balance as a workaround for usage-commitment tracking will corrupt revenue recognition.

### 8.3 Allocations (recurring credit grants tied to a plan)
- Configured at the **plan** level: an amount + cadence + optional rollover, auto-injected into the customer's ledger each period.
- **Zero cost-basis** allocations = free inclusions (no invoice line item, no revenue impact) — this is the "100K free tokens/month" pattern.
- **Non-zero cost-basis** allocations = billed in-advance, generate a real invoice line item, drive revenue recognition (e.g., "5,000 credits/mo @ $0.25 = $1,250/mo charge").
- **Not prorated** on partial periods — a subscription starting mid-month still gets the *full* allocation.
- **Allocation credits expire immediately on plan change/cancellation**; **purchased credits persist** (only die on their own `expiry_date`). This distinction matters a lot for trial design — grant trial credits via the ledger API (not as a plan allocation) if you want them to survive an upgrade.

---

## 9. Invoice Calculation — The Full Pipeline

This is the answer to "how is the final number actually computed." Every line item passes through these steps **in this exact order**, and only after this per-line-item computation does the invoice sum them up (adjustments spanning multiple prices, then tax, then balance are layered on top).

```
1. Quantity        → evaluate the billable metric for the period
2. Subtotal        → apply the pricing function (quantity × rate, per §5)
3. Adjustments     → in order: Usage discount → Amount discount → Percent discount → Minimum → Maximum
4. Prepaid credits → deduct from credit ledger (in-arrears charges only)
5. Conversion      → virtual currency → real currency, using the price's conversion rate
6. Partial invoices→ subtract amounts already invoiced (threshold billing)
7. Tax             → per line item, via TaxJar/Avalara/etc. (added at issuance, not on drafts)
8. Customer balance→ applied to the final total, after tax
```

### 9.1 Why the *order* matters — the two counter-intuitive rules

**Rule 1 — Minimums apply BEFORE prepaid credits.**
If credits were applied first, customers could always shrink their bill to zero using credits and dodge a contractual minimum. So Orb floors the subtotal to the minimum *first*, and credits only reduce what's left.
```
Usage: $300, Minimum: $400, Prepaid credits: $500
→ Apply minimum: max(300, 400) = 400
→ Apply credits: 400 - 400 = $0 due
→ Remaining credit balance: $100 (NOT $200 — the minimum "cost" $100 of credit)
```

**Rule 2 — Customer balance applies AFTER tax, at the very end; prepaid credits apply BEFORE tax.**
```
Usage charges:          $800
- Prepaid credits:      $500
= Subtotal before tax:  $300
+ Tax (10%):             $30
= Total with tax:       $330
- Customer balance:     $100
= Amount due:           $230
```

### 9.2 Cross-price adjustment distribution
When one adjustment spans multiple prices (e.g., a plan-level 15% discount), it's distributed **proportionally to each line item's share of the subtotal**:
```
Discount = $20, Price A subtotal = $100, Price B subtotal = $25 (total $125)
Price A share = (100/125) × 20 = $16
Price B share = (25/125)  × 20 = $4
```
**Minimum** deltas, in contrast, are distributed **evenly** (not proportionally) across applicable line items.

### 9.3 Full worked example (all components together)
```
Usage price (tiered API calls): 50,000 calls
  10,000 × $0.01 = $100.00
  40,000 × $0.005 = $200.00  → API subtotal = $300.00
Fixed fee (in-arrears platform fee)          = $100.00
Total subtotal                               = $400.00

15% invoice-level discount → -$60.00, distributed:
  API:  (300/400)×60 = $45.00 → API now $255.00
  Fee:  (100/400)×60 = $15.00 → Fee now $85.00
After discount = $340.00

Minimum ($200): $340 > $200 → no change

Prepaid credits ($150 available, in-arrears only, both lines eligible):
  Eligible total = $340 → apply $150 → remaining = $190.00

Tax (8%): $190 × 0.08 = $15.20 → Total with tax = $205.20

Customer balance ($30) → Amount due = $175.20
```

### 9.4 Proration of minimums/maximums
A monthly $100 minimum, applied to a subscription active only 15 of 30 days, becomes a **$50** prorated minimum — commitments always scale with the actual served period, never charge for time not served.

### 9.5 Threshold (partial) invoicing
If usage crosses a $ threshold mid-period, Orb issues a **partial invoice** immediately, then nets it out at period end:
```
Threshold: $500 → Day 10 usage hits $520 → partial invoice #1 = $520
Day 15 usage recalculated to $650        → partial invoice #2 = $650 (supersedes #1)
Day 30 final usage = $800
End-of-period invoice = $800 - $650 (highest partial, not sum of partials) = $150 remaining
```

### 9.6 Invariants worth remembering
- One `Price` → at most one line item per invoice (never duplicated).
- All real-currency line items share one currency; virtual currencies are allowed alongside it if each has a conversion rate to that currency.
- Adjustments combined together must share currency + cadence + billing mode.
- Minimums are **not applied** to an invoice that consists **only** of fixed fees (prevents a minimum wrongly hitting the very first in-advance invoice of a sub with no usage lines yet).

---

## 10. Invoice Lifecycle & States

```
draft ──(grace period elapses)──► pending issue ──► issued ──► paid
  │                                                     │
  │                                                     ├──► synced (external provider, one-way)
  │                                                     └──► void (reverses any balance it consumed)
```

- Every **active subscription always has a live draft invoice** — this is the "upcoming invoice" that continuously reflects accrued usage in real time; it's not just a monthly artifact.
- After the billing period ends, the invoice sits in a **grace period** (matches the ingestion grace period) so late events can still land before the numbers are frozen.
- **$0 invoices auto-mark as paid** on issuance (no email sent).
- **Synced invoices** (Bill.com/Stripe Invoicing/QuickBooks/NetSuite) are a **one-way** push — edits after sync don't reflect back; a synced invoice is always treated as "paid" for credit-note purposes.

---

## 11. Customer Hierarchy (Parent/Child Billing)

For consolidated billing across related entities (e.g., an enterprise customer with multiple business units/sub-accounts):

- One parent, up to **100 children**. A customer can only belong to one hierarchy.
- A single subscription can bill **any mix** of parent + children via `usage_customer_ids` — parent-only, children-only, or both — and different prices in the *same* plan can target different subsets of the hierarchy (e.g., one rate for group A, a duplicated price at a different rate for group B).
- **Usage aggregates to the parent's invoice.** The parent's credit ledger is what gets drawn down for hierarchy-wide usage, even if the usage technically came from a child's events. Plan credit purchases at the parent level accordingly.
- Currency/tax on the consolidated invoice always follows the **parent** even if children are configured differently.
- A **plan change** wipes the `usage_customer_ids` configuration on price intervals (must be reconfigured); a **plan version migration** does not.

---

## 12. Worked Reference: AI-Agent / Token-Based Billing (Orb's own guide)

This is Orb's own worked example for exactly your domain — LLM token metering — and it maps almost 1:1 onto QuantumBilling's problem space.

### 12.1 Business model being modeled
- Free tier: 100K tokens/month, no upfront fee.
- Premium tier: $50/mo upfront + 1M tokens/month.
- On-demand token purchase (either plan): $10 per 100K tokens.
- Free monthly grants expire end of month; purchased tokens expire 1 year after grant.

### 12.2 Event shape
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
    "region": "us-west-2",
    "hardware": "instance-type-2",
    "task_id": "5e0f1823-d525-456c-abad-45c917d2d025",
    "checkpoint": 3
  },
  "idempotency_key": "5e0f1823-d525-456c-abad-45c917d2d025-3"
}
```
Note the design choice explicitly called out: **send raw token counts per checkpoint, not a pre-computed dollar amount.** This is the same "decouple product usage from pricing" principle from §4 — it lets you change the $/token rate later without touching the agent's instrumentation.

### 12.3 Metric & pricing
- Metric = `SUM` of total tokens used across all tasks/checkpoints for a customer, regardless of model/hardware (a coarse metric — you *could* instead build one metric per model if you wanted model-specific rates, the same way input/output/reasoning tokens are often priced differently by real LLM providers).
- Pricing unit: a **custom virtual currency** called `token`. Unit price: **1 token metric-unit = 1 token currency-unit** (a trivial 1:1 conversion — the whole rating step is just "drawn down from the prepaid balance").
- Two Plans (Free/Premium) differ *only* in: recurring fee ($0 vs $50/mo) and monthly token **allocation** (100K vs 1M, both zero-cost-basis so they're free inclusions, not invoiced separately).
- On-demand purchases: a **ledger increment** with `currency=token`, `per_unit_cost_basis=$0.0001` (= $10 / 100,000), `expiry_date` = 1 year out.

### 12.4 Spend control & visibility
- `credit_balance_depleted` / `credit_balance_recovered` alerts, scoped to the `token` currency, created at signup — webhook-driven enable/disable of agent access.
- **Automatic top-ups**: buy another 100K tokens whenever the balance crosses a threshold; capped via a manually-tracked max-top-up counter (Orb doesn't natively cap top-up *count*, only lets you deactivate the top-up once your own logic decides the limit is hit).
- Real-time balance shown to the customer via the **upcoming invoice** endpoint's `line_items[].credits_applied` field (cacheable via the `Orb-Cache-Control: cache` header for a snappy dashboard, refreshed async without the header).
- Historical usage: `list invoices` filtered by `external_customer_id`, distinguishing `invoice_source=subscription` (recurring token drawdown) vs `invoice_source=one_off` (a manual top-up purchase).

### 12.5 Upgrade/downgrade
- **Downgrade** (Premium→Free): `schedule_plan_change` with `change_option=end_of_subscription_term` (already paid for the month — don't claw back). Existing credits are *not* reclaimed; they simply expire on their normal schedule.
- **Upgrade** (Free→Premium): `change_option=immediate` + `billing_cycle_alignment=plan_change_date` so the $50 fee is charged right away on upgrade. Preview the exact charge first via `Orb-Dry-Run: true` + `Include-Changed-Resources: true` before committing the user to the flow.

---

## 13. Orb Concept ↔ QuantumBilling Concept Map

| Orb concept | Rough QuantumBilling equivalent (as currently designed) |
|---|---|
| Columnar event store, query-based invoicing | Kafka → ClickHouse ingestion path (source of truth) |
| Stream-based alerting pipeline (Redis) | Redis/BullMQ — must stay read-only relative to billing calc, per §1 note |
| Event (`event_name` + `properties`) | Raw usage event from LiteLLM gateway (input/output/reasoning tokens per model/provider) |
| Billable Metric (filter + aggregate) | Meter definitions in your `meter` module |
| Price (unit/tiered/bulk/package/dimensional/license) | Pricing model config in `billing` module |
| Plan + Adjustments | Plan/discount/minimum config |
| Subscription (term vs. billing period) | Subscription lifecycle in `subscription` module |
| Prepaid Credit Ledger (blocks, cost basis, deduction order) | Your credits/prepaid balance design — worth explicitly adopting Orb's 5-step deduction order (license → item-scoped → soonest-expiring → lowest-cost-basis → FIFO) rather than inventing a new one |
| Customer Balance (AR wallet) | A separate, simpler wallet — keep it distinct from prepaid credits per §8 |
| Customer hierarchy (parent/child) | Your 4-level tenant hierarchy (Super Admin → Tenant Admin → Customer → End User) — Orb's model only goes 2 levels (parent/child) and caps at 100 children, so your deeper hierarchy is a genuine superset, not a subset, of this pattern |
| Invoice calculation pipeline (8 ordered steps) | Directly reusable as the ordering contract for your own invoice-generation logic |

---

## 14. Quick-Reference: The Things Most Likely to Bite You Later

1. **Minimums beat credits, always.** Never apply credits before checking a minimum — it lets customers dodge commitments.
2. **In-advance charges never touch prepaid credits.** Only in-arrears (usage + in-arrears fixed fees) draw from the ledger.
3. **Customer balance is post-tax; prepaid credits are pre-tax.** Don't conflate them in your own reconciliation logic.
4. **Ledger deductions are grouped (daily/price/block), not per-event** — if you need per-event/per-API-key attribution, that has to be a separate "evaluate" query path, not a ledger read.
5. **Allocations are not prorated**; a mid-cycle-start subscription still gets the full period's grant.
6. **Allocation credits die with the plan; purchased credits don't.** Get this backwards and you'll either double-grant or wrongly claw back credits on a plan change.
7. **Term ≠ billing period.** Term = longest cadence (drives "end of term" cancellation timing); billing period = shortest cadence (drives invoice frequency).
8. **Tiered ≠ Bulk.** Tiered = progressive (each unit priced at its own tier's rate); Bulk = cliff (total volume picks ONE rate for everything). Mixing these up is a classic pricing bug.
9. **Events should carry raw, undecided data.** Never bake a $ amount into an event — keep pricing changeable without touching your instrumentation.
10. **Backfills/amendments are overlays, not mutations.** Design your own event store (ClickHouse) the same way: never rewrite a row in place; append a correction that supersedes it on read.

---

*Sources: docs.withorb.com — Overview, Core Concepts, Query-Based Billing Architecture, Event Ingestion, Construct Metrics, Price Configuration, Configuring Plans, Creating Subscriptions, Subscription Plan Changes, Credit Systems, Configure Prepaid Credits, Invoice Calculations, Invoice Calculation Examples, Structure & Lifecycle, Customer Hierarchy, Pricing an AI Agent (self-serve).*
