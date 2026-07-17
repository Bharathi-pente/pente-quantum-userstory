# FlexPrice — Metering & Billing Calculation Guide

Source: [docs.flexprice.io](https://docs.flexprice.io/docs/welcome-to-flexprice) (fetched Jul 2026)

## 0. What FlexPrice actually is

Open-source metering + billing + feature-management platform. You send **usage events**, it aggregates them into billable **quantities**, prices them against a **Plan**, ties that to a **Subscription**, and produces an **Invoice**. Everything downstream (proration, coupons, tax, wallets) is deterministic math on top of that aggregated quantity — that's the part you actually need to internalize.

```
Event → Feature (aggregation) → Price (billing model) → Plan → Subscription → Invoice
```

---

## 1. Events — the raw input

Every unit of usage (an API call, a request, an input token) is a JSON event posted to FlexPrice.

`POST https://api.cloud.flexprice.io/v1/events` (single) or `/v1/events/bulk` (batch, up to 1000/request)
Auth: header `x-api-key: <key>`

```json
{
  "event_name": "model.usage",           // must match the Feature's Event Name exactly
  "external_customer_id": "cust_123",    // your own user/customer id
  "properties": {                        // only needed for Sum/Max/Latest/Unique-Count
    "credits": 2,
    "model": "gpt-4",
    "region": "us-east-1"
  },
  "event_id": "evt_abc123",              // optional, use it — enables de-dup
  "timestamp": "2025-08-22T07:05:49.441Z",// ISO-8601 UTC, optional (defaults to server time)
  "source": "api"                        // optional, free text
}
```

- `properties.*` values are sent as **strings**; FlexPrice coerces to numeric during aggregation.
- Response is **202 Accepted** — ingestion is async. You get back an `event_id`; the actual aggregation happens in the pipeline, not synchronously.
- Rate limits: 1000 req/min single events, 100 req/min bulk (≤1000 events/request).
- `event_id` is your idempotency key — a duplicate `event_id` is deduplicated, **keeping the latest value**, not summed twice (see §3 worked example).

---

## 2. Features — what you're metering, and how

A **Feature** is the definition layer between raw events and money. Three feature types exist: **Metered** (usage-driven, what this doc is about), **Boolean** (on/off gate), **Static** (a config value, e.g. "max seats: 10"). For usage billing you create a **Metered Feature**:

| Setting | Meaning |
|---|---|
| Event Name | Must match `event_name` in the incoming events |
| Aggregation Function | How events roll up into one number (§3) |
| Aggregation Field | Which `properties.<field>` gets aggregated (not needed for COUNT) |
| Usage Reset | `Periodic` (resets every billing cycle) or `Cumulative` (keeps growing) |
| Unit Name | Display label, e.g. `credits`, `tokens`, `GB` |

---

## 3. Aggregation types — the actual math

This is the core calculation layer: **how a pile of events becomes one billable number** for a billing period.

| Type | Formula | Notes |
|---|---|---|
| **COUNT** | `COUNT(DISTINCT events.id)` | Only one needing no `properties` field — just counts matching events. |
| **SUM** | `SUM(properties.field)` | Straight total. |
| **AVERAGE** | `AVG(properties.field)` | Mean across matching events. |
| **COUNT UNIQUE** | `COUNT(DISTINCT properties.field)` | e.g. unique active users. |
| **LATEST** | `argMax(properties.field, timestamp)` | Most recent value wins — for "current state" metrics like seat count. |
| **SUM WITH MULTIPLIER** | `SUM(properties.field) × multiplier` | Sum **first**, multiply **once** at the end — not per-event. Multiplier is immutable once the feature is created. |
| **MAX** | `MAX(properties.field)` | Peak value, supports bucketing (e.g. daily max). |
| **WEIGHTED SUM** | Time-proportional sum, prorated by event duration | For capacity/reservation billing (e.g. "GB provisioned for N hours"). |

**Deduplication rule (applies to all types keyed by `event_id`):** if you send the same `event_id` twice with different property values, FlexPrice keeps the **latest** value for that ID, not a running total. Example from FlexPrice's own docs (SUM WITH MULTIPLIER, multiplier = 0.001):

```
evt_001 (first send):  credits = 1000
evt_002:                credits = 2500
evt_003:                credits = 1500
evt_001 (re-sent):      credits = 800   ← treated as an UPDATE, not a new event

Dedup result: evt_001 → 800 (latest), evt_002 → 2500, evt_003 → 1500
Sum = 800 + 2500 + 1500 = 4800
Charge = 4800 × 0.001 = $4.80
```

This matters a lot for token billing: **always send a stable, unique `event_id` per API request** (e.g. the LLM provider's `request_id`). If you don't, and you retry a failed send, you can double count. If you reuse an ID by mistake, you silently overwrite instead of adding.

---

## 4. Token / AI usage — the exact mechanism for "input tokens"

FlexPrice has a **standardized `ai.usage` event shape** purpose-built for LLM billing. This is what you send per completion/request:

```json
{
  "event_name": "ai.usage",
  "external_customer_id": "team_platform",
  "timestamp": "2026-06-14T10:30:00Z",
  "source": "litellm",
  "properties": {
    "provider": "anthropic",
    "model": "claude-opus-4-8",
    "operation": "chat",
    "input_tokens": "1840",
    "output_tokens": "320",
    "cached_tokens": "1024",
    "reasoning_tokens": "0",
    "reported_cost": "0.041",
    "request_id": "req_abc123",     // ← use this as event_id for dedup
    "raw_user": "u_91",
    "raw_team": "team_platform",
    "agent_id": "agent_support_bot",
    "fidelity": "per_request"
  }
}
```

### 4.1 How the token counts turn into a charge

Two mutually-exclusive cost strategies, chosen automatically per event:

1. **Trust the source cost (default)** — if the event already carries `reported_cost` (LiteLLM, OpenRouter, Portkey, Langfuse, Helicone, Cloudflare, Vercel all compute this themselves), FlexPrice stores that number as-is and bills on it directly. This is exact even for negotiated/BYOK rates.
2. **Re-price from FlexPrice's catalog** — if a source gives you **tokens only** (raw Bedrock logs, Databricks token tables) with no `reported_cost`, or you explicitly want a markup/margin or a single normalized rate across providers, FlexPrice prices it from its own **model pricing repository** (refreshed daily) — i.e. it looks up `$/1M input tokens` and `$/1M output tokens` for that `provider` + `model` and multiplies. You can override this catalog per provider/model/customer (committed pricing) and layer a markup factor on top when reselling.

So mechanically, when re-pricing from catalog:

```
input_cost  = input_tokens   × rate_input(provider, model)
output_cost = output_tokens  × rate_output(provider, model)
cached_cost = cached_tokens  × rate_cached(provider, model)     // usually discounted vs input
total_cost  = input_cost + output_cost + cached_cost + reasoning_cost
charged     = total_cost × (1 + markup_pct)                     // only if reselling
```

`input_tokens`, `output_tokens`, `cached_tokens`, `reasoning_tokens` are each their own aggregatable property — meaning you can create **separate Metered Features** for each (e.g. "Input Tokens", "Output Tokens") and price them differently, which is exactly how OpenAI/Anthropic/etc. price themselves (input ≠ output ≠ cached rate).

`unit`/`quantity` generic fields are used instead of the token fields for non-token sources (characters, seconds, requests, credits) — same event shape, different property.

### 4.2 Auto-provisioned metering templates

You don't have to hand-build meters per model. Enabling an AI connector + choosing a template auto-creates the whole metering graph:

| Template | Provisions | Use case |
|---|---|---|
| `cost_tracking` | Meters for input/output/cached/reasoning tokens + request count, features per dimension, catalog pricing at raw cost | Internal showback / "what is this costing us" |
| `team_budget` | Everything above **+ a wallet + recurring monthly credit grant + info/warning/critical alerts** wired to gateway enforcement | Team/agent budget caps |
| `resale_markup` | Catalog re-pricing with configurable margin + margin analytics | Billing your own customers for AI features you resell |

Model/provider breakdowns come from `filters`/`group_by` on the `provider` and `model` properties — you don't need a meter per model, a handful of generic token meters covers everything.

### 4.3 Getting events in

Three ingestion patterns, same canonical event on the other end:

| Type | Mechanism | Latency | Best for |
|---|---|---|---|
| **Push (edge)** | Source POSTs to a FlexPrice Collector running in your infra | Real-time | LiteLLM webhooks, anything that must stay in-network |
| **Managed pull** | FlexPrice polls the source via API/SQL/object storage on a schedule | Seconds–hours | Langfuse, Databricks, Bedrock — no deployment needed, you just paste read-only creds |
| **Aggregate** | FlexPrice ingests periodic dashboard/CSV/email exports | Daily | Sources with no per-request API (Salesforce Agentforce, SAP Joule) — marked `fidelity: aggregate`, treat per-request attribution as approximate |

For managed pull, a per-source **watermark** + `request_id` dedup guarantees no double-counting across overlapping polling windows.

If you're wiring up LiteLLM specifically: register `FlexpriceLogger` as a LiteLLM callback (SDK) or enable it in the proxy's `config.yaml` (no code) — every `completion()` call is metered automatically, with the team/user identity pulled from LiteLLM's `metadata`.

### 4.4 Identity resolution / hierarchy

- One field (e.g. `raw_team`) resolves to the FlexPrice **customer** being billed.
- Other fields (`raw_user`, `agent_id`) become **child entities** under that customer via Customer Hierarchy — so one wallet/budget can cover a whole team while you still see usage broken out per user and per agent.

### 4.5 Enforcement loop

On an `info`/`warning`/`critical` alert (wallet or usage threshold crossed), FlexPrice can call back into the source gateway:
- **LiteLLM**: zero out `max_budget` on a key/team (soft) or `key/block` (hard), via your LiteLLM master key.
- **OpenRouter**: set a per-key credit limit or disable the key via its Provisioning API.
- **Portkey**: alert-only; enforcement is configured in the Portkey dashboard.

---

## 5. Plans, Prices & Billing Models — turning quantity into money

A **Plan** bundles recurring charges + usage-based charges + entitlements (feature access/limits) + trial rules, in a given currency. A **Price** attached to a Plan is where the actual rate/model lives. Three billing models:

### 5.1 Flat Fee — linear, per unit
```
charge = usage_quantity × unit_price
```
1,000 API calls × $0.05 = $50. Constant regardless of volume.

### 5.2 Package — banded flat price
Usage falls into a **range**, entire range costs one fixed price regardless of exact count within it.
```
1–1,000 msgs      → $50
1,001–5,000 msgs  → $200
5,001–10,000 msgs → $350
```
4,500 messages → falls in the 1,001–5,000 band → charged flat **$200** (even if actual usage is 1,500).

### 5.3 Volume Tiered — single rate applied to *entire* usage, chosen by the highest tier reached
```
Tier 1: 0–10,000        @ $0.0010
Tier 2: 10,001–50,000   @ $0.0008
Tier 3: 50,001–100,000  @ $0.0006
Tier 4: 100,001+        @ $0.0004
```
65,000 calls → total usage lands in Tier 3 → **entire 65,000** billed at $0.0006 → `65,000 × 0.0006 = $39`. (Note: this is *volume* tiering — one rate for all units based on where the total lands — not graduated/marginal tiering where each tier's units are billed at that tier's rate separately. If you need graduated tiering, model it explicitly; FlexPrice's named "tiered" examples in the connecting-to-billing guide use the graduated interpretation — first N at rate A, next N at rate B — so confirm which behavior your dashboard config produces before relying on it for a real pricing page.)

### 5.4 Free tier / included allowance
A Plan can define a free quantity before billing starts:
```
Total Usage - Free Tier = Billable Usage   (floored at 0)
Amount = Billable Usage × unit_price (or applicable model above)
```
Example: 10 free credits/month, customer uses 15 → 5 billable → charged for 5.

### 5.5 Multiple features on one Plan
Each metered feature on a plan is priced and totaled independently, then summed for the invoice:
```
Plan: Pro
  Model Usage: $1.00/credit   × 50 credits  = $50.00
  API Calls:   $0.01/call     × 1000 calls  = $10.00
  Storage:     $0.10/GB       × 5 GB        = $0.50
  ---------------------------------------------------
  Total                                     = $60.50
```

### 5.6 Advance vs Arrear (timing, not amount)
Independent of billing *model* — this is *when* you charge:
- **Advance**: pay before usage (prepaid credits, minimum commit, subscription paid on the 1st). Guarantees cash flow, eliminates unpaid-invoice risk.
- **Arrear**: pay after usage (AWS-style end-of-month invoice based on actual consumption). Standard for unpredictable pay-as-you-go usage.
Recurring (flat subscription) charges and usage charges can each independently be advance or arrear — e.g. subscription fee in advance + usage overage in arrears is the most common SaaS hybrid.

---

## 6. Subscriptions — connecting a Customer to a Plan

Create a Customer → attach a Subscription to a Plan → set start date (and optional end date, default "Forever"). A customer can hold **multiple independent subscriptions simultaneously** (e.g. Base Plan + an Addon subscription), each with its own billing period and charges.

**Per-subscription customization** without touching the shared Plan:
- **Override Line Items** — change amount/quantity/billing model/tier structure for just that one subscription (enterprise discount, custom rate) without affecting other customers on the same plan.
- **Seat-based pricing** — set quantity at creation, adjust anytime (optionally future-dated).

**Billing period math**: usage aggregation (§3) runs against the **current billing period window** of the subscription. When `Usage Reset = Periodic`, the aggregated quantity zeroes out at the start of each new period; `Cumulative` keeps the running total across periods (used for things like total storage, not monthly consumption).

---

## 7. Proration — mid-cycle plan changes

Time-based, linear proration on any plan upgrade/downgrade:

```
Credit (unused time on old plan) = (days_remaining / total_days_in_period) × old_plan_price
Charge (new plan, remaining time) = (days_remaining / total_days_in_period) × new_plan_price
Net amount due = Charge − Credit
```

Worked example — $100/mo → $200/mo plan, changed on day 15 of a 30-day month:
```
days_remaining = 15
Credit  = (15/30) × $100 = $50
Charge  = (15/30) × $200 = $100
Net due = $100 − $50 = $50
```
A downgrade produces a **negative** net amount (i.e. a credit applied to the account, not a refund by default) — see the quarterly downgrade example in FlexPrice's docs where net comes out to -$75. Works identically for monthly/quarterly/yearly periods (day-count based, leap years included).

---

## 8. Wallets & Credits — prepaid balances

A **Wallet** is a per-currency prepaid balance tied to a customer (one active wallet per currency). Used to fund advance/prepaid billing models (§5.6) and AI team budgets (§4.2).

- **Credit-to-currency ratio**: configurable via **Pricing Units** (custom currency, e.g. "1 credit = $0.01"); default free-credit wallets are 1:1.
- **Top-up**: manual or automatic (**Auto Top-Up** — recharges when balance drops below a threshold).
- **Consumption order**: wallet credits are applied to invoices **after** all coupon discounts, **before** tax is computed on the discounted (not credit-reduced) amount — see §9, step 4/5.
- **Alerts**: low-balance thresholds (`info`/`warning`/`critical`) fire webhooks and can trigger enforcement at the AI gateway (§4.5).
- **Manual Debit**: admin-side deduction for refunds/corrections outside the normal usage-billing flow.

---

## 9. Invoice Calculation — the exact, deterministic order

This is the sequence FlexPrice runs on **every** invoice, and it's the part worth memorizing if you're building something similar:

```
1. Subtotal = Σ(all line items) = usage charges (§3-5) + flat/recurring fees

2. Line-item coupon discounts
   Reduce each targeted line item individually.

3. Subscription (invoice)-level coupon discounts
   Applied sequentially to the running subtotal — each coupon sees
   what's left after the previous one (NOT the original subtotal).
   Two 10% coupons chained = 19% effective discount, not 20%:
     $100 → −10% → $90 → −10% (of $90) → $81

4. Prepaid wallet credits
   Deducted AFTER all coupon discounts.

5. Taxable amount = MAX(subtotal − all_coupon_discounts, 0)
   Floored at zero — coupons can never push the taxable base negative,
   and excess discount is NOT carried to the next invoice.
   IMPORTANT: wallet credits do NOT reduce the taxable base — tax is
   computed on (subtotal − coupons), not (subtotal − coupons − wallet).

6. Tax = Σ(each applicable tax rate × taxable_amount)
   Multiple tax rates apply independently to the SAME taxable amount —
   they do not compound with each other.
     State 6% + Federal 2% on $100 = $6 + $2 = $8 total (not $8.12)

7. Invoice total = subtotal − total_coupon_discounts − wallet_credits_applied + tax
```

Full worked example:
```
Usage charge (API calls)              $80.00
Flat monthly fee                     +$20.00
Subtotal                             =$100.00

Line-item coupon (25% off API calls) −$20.00  → $80.00
Subscription coupon ($10 off)        −$10.00  → $70.00
Prepaid wallet credit                −$5.00   → $65.00 owed

Taxable amount (subtotal − coupons only, NOT − wallet) = $70.00
Tax @ 8.25%                           +$5.78
Invoice total                        =$70.78
```

---

## 10. End-to-end walkthrough: billing "input tokens" like an LLM provider

Tying §1–9 together for the exact scenario in your question:

1. **Ingest**: your app (or LiteLLM callback) posts an `ai.usage` event per completion, with `input_tokens`, `output_tokens`, `cached_tokens`, and `request_id` as `event_id`.
2. **Feature**: create a Metered Feature `"Input Tokens"` → Event Name `ai.usage`, Aggregation = **SUM**, Aggregation Field = `input_tokens`, Usage Reset = Periodic. Repeat for `"Output Tokens"` (often at a higher rate) and optionally `"Cached Tokens"` (usually discounted).
3. **Pricing**: attach each feature to a Plan as a **Flat Fee** price (e.g. $0.003 / 1K input tokens) — or use **SUM WITH MULTIPLIER** directly on the feature if you want the rate baked into the aggregation rather than the plan price. Optionally set a free tier (e.g. first 100K tokens/month free).
4. **Subscription**: customer subscribes to that Plan; billing period = monthly.
5. **Aggregation**: over the period, SUM(input_tokens) across all deduped events = total billable input tokens.
6. **Free tier**: subtract included allowance, floor at 0.
7. **Charge**: `billable_input_tokens × rate + billable_output_tokens × rate + ...` → line items on the invoice.
8. **Wallet (optional)**: if the customer prepaid credits (advance billing), those deduct from the amount owed after coupons, before tax.
9. **Invoice**: subtotal → coupons → wallet → tax (per §9) → final amount due.
10. **If they upgrade mid-cycle** (e.g. bigger token-rate plan or annual commit), proration (§7) applies for the remaining days in the period.

---

## 11. Reference: webhook events (for reacting to state changes)

Non-exhaustive but the ones relevant to billing/usage state:

| Event | Fires when |
|---|---|
| `subscription.created` / `.activated` / `.cancelled` / `.paused` / `.resumed` | Subscription lifecycle changes |
| `subscription.renewal.due` | Cron-driven, upcoming renewal |
| `invoice.update.finalized` / `.voided` / `.payment` | Invoice lifecycle |
| `invoice.payment.overdue` | Past due date |
| `wallet.credit_balance.dropped` / `.recovered` | Wallet threshold crossed |
| `wallet.ongoing_balance.dropped` / `.recovered` | Ongoing (non-credit) balance threshold |
| `wallet.transaction.created` | Top-up or debit recorded |
| `feature.wallet_balance.alert` | Feature-scoped wallet alert threshold crossed |
| `payment.success` / `.failed` / `.pending` | Payment attempt outcome |

---

## 12. Quick-reference cheat sheet

| Question | Answer |
|---|---|
| What's the smallest unit of input? | One JSON event (`event_name`, `external_customer_id`, `properties`, optional `event_id`/`timestamp`) |
| How is usage measured? | An **Aggregation function** (SUM, COUNT, AVG, MAX, LATEST, COUNT UNIQUE, SUM×multiplier, WEIGHTED SUM) run on event properties over the current billing period |
| How does an amount get attached? | Aggregated quantity × the **Price**'s billing model (Flat Fee / Package / Volume Tiered), minus any free tier |
| How is a Plan different from a Price? | Plan = bundle (recurring fee + usage charges + entitlements + trial); Price = one specific charge/rate inside it |
| How does a Subscription relate to a Plan? | Subscription = a specific customer's live instance of a Plan, with its own billing period, overrides, and quantity |
| What happens on plan change mid-cycle? | Time-based proration: credit unused days on old plan, charge remaining days on new plan |
| How is tax computed? | On `subtotal − coupon discounts` (NOT minus wallet credits), each applicable rate applied independently, non-compounding |
| Where do prepaid credits fit? | Deducted from amount owed after coupons, before tax is calculated (tax ignores wallet deduction) |
| How is AI/token cost attached specifically? | Either trust the source's `reported_cost` verbatim, or re-price `input_tokens`/`output_tokens`/`cached_tokens`/`reasoning_tokens` against FlexPrice's daily-refreshed model pricing catalog (with optional markup) |
