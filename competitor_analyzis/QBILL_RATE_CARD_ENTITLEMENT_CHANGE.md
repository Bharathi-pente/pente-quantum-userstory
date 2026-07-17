# QBill — Changing Rate Cards, Contract Rates & Entitlements Mid-Subscription

**Date:** 2026-07-17
**Question:** How can I change rate cards, contract rates, or entitlements in the middle of a subscription?

---

## Table of Contents

1. [The Rating Waterfall — Quick Refresher](#1-the-rating-waterfall--quick-refresher)
2. [What CAN Be Changed Mid-Subscription](#2-what-can-be-changed-mid-subscription)
3. [Method 1: Change Contract Rates (Highest Priority)](#3-method-1-change-contract-rates-highest-priority)
4. [Method 2: Publish a New Rate Card Version](#4-method-2-publish-a-new-rate-card-version)
5. [Method 3: Change Plan Pricing Model](#5-method-3-change-plan-pricing-model)
6. [Method 4: Change Usage Limits / Entitlements](#6-method-4-change-usage-limits--entitlements)
7. [What Happens to Existing Invoices?](#7-what-happens-to-existing-invoices)
8. [The Rating Cache — How Changes Propagate](#8-the-rating-cache--how-changes-propagate)
9. [Impact Matrix](#9-impact-matrix)
10. [Code Walkthrough](#10-code-walkthrough)
11. [Common Questions](#11-common-questions)

---

## 1. The Rating Waterfall — Quick Refresher

Every usage event is rated through a **4-step waterfall**. The first match wins:

```
Step 1: Contract Rate          ← Highest priority (customer-specific override)
Step 2: Rate Card Version      ← Published rate card pinned to the contract
Step 3: Plan Pricing Model     ← The plan's default pricing
Step 4: Unrated                ← Flagged as exception (never silently zero-billed)
```

```
Example: Rating GPT-4 input tokens for Customer ACME

  Step 1: Does ACME have a contract_rate for (meter=tokens, model=gpt-4, type=input)?
          → YES at $0.000020/token → USE IT (stop here)

  Step 2: (If no contract rate) Does ACME's contract pin a rate card version?
          → YES, version 3 has rate $0.000022/token → USE IT

  Step 3: (If no rate card) Does the plan's charge have a pricing model?
          → YES, $0.000025/token via PER_UNIT → USE IT

  Step 4: (Nothing found) → Flag as rating_exception → never bill at zero
```

**At invoice time**, the engine calls `rating.Resolve(RatingInputs, ResolveRequest)` with a **versioned snapshot** of all rates. The snapshot is frozen at the moment of invoice generation — changes made after invoicing don't affect issued invoices.

---

## 2. What CAN Be Changed Mid-Subscription

| What | Can Change Mid-Subscription? | How |
|---|---|---|
| **Contract Rate** (step 1) | ✅ **YES — immediately** | Add/update/delete contract rates for a customer. New rates apply to future usage immediately. |
| **Rate Card** (step 2) | ✅ **YES — versioned** | Publish a new rate card version. Contracts pinned to the card see new rates (if pinned to latest). |
| **Plan Pricing Model** (step 3) | ⚠️ **Versioned** | Edit plan → creates new plan version. Takes effect from next billing period (not mid-period) unless subscription plan is changed. |
| **Usage Limits / Entitlements** | ✅ **YES — immediately** | Update `customer.usage_limits` or add `limit_overrides`. New limits take effect on next enforcement check. |
| **Wallet Config** | ✅ **YES — immediately** | Update low_balance_threshold, auto_topup_enabled, topup_amount. |
| **Dunning Policy** | ✅ **YES — immediately** | Update retry schedule, escalation rules. |
| **Tax Region / Exemption** | ✅ **YES — immediately** | New tax rates apply to next invoice finalization. |

---

## 3. Method 1: Change Contract Rates (Highest Priority)

### What Are Contract Rates?

Contract rates are **customer-specific rate overrides** — the highest priority in the waterfall. If a customer has a negotiated price, you set it here.

### API: Add a Contract Rate

```typescript
POST /contracts/:contractId/rates
{
  "meter_id": "uuid-of-meter",
  "model_name": "gpt-4",
  "token_type": "input",         // optional: null = all token types
  "rate": "0.000015",            // $0.000015 per token
  "unit_label": "token",
  "effective_date": "2026-07-17", // when this rate starts
  "expires_date": "2027-07-17"    // optional: when it ends
}
```

### What Happens

```
Before:                                   After:
ACME pays $0.000025/token (plan rate)     ACME pays $0.000015/token (negotiated)
                                          
Waterfall at next invoice:                Waterfall at next invoice:
  1. Contract rate? NO                     1. Contract rate? YES → $0.000015
  2. Rate card → $0.000022                 (stops here — no further lookup)
  3. Plan → $0.000025                     
  4. Unrated → never
```

### Effect on Invoicing

| Aspect | Impact |
|---|---|
| **Current period invoices (already issued)** | ❌ **Unaffected** — contract rate change does NOT trigger re-rating |
| **Current period invoice (draft, not yet finalized)** | ✅ **Updated** — draft invoices pick up new rates at finalization |
| **Future invoices** | ✅ **Use new rate** — from effective_date onwards |
| **Real-time wallet burndown** | ✅ **Updated within ~60s** — rating cache refreshes every 60s (W-2) |
| **Real-time enforcement** | ✅ **Updated within ~60s** — same cache refresh cycle |

### API: Update an Existing Contract Rate

```typescript
PATCH /contracts/:contractId/rates/:rateId
{
  "rate": "0.000012",      // change the price
  "expires_date": null     // remove expiration
}
```

### API: Delete a Contract Rate

```typescript
DELETE /contracts/:contractId/rates/:rateId
// Customer falls through to Step 2 (rate card) or Step 3 (plan pricing)
```

### BFF Code

```typescript
// From catalog.service.ts
async addContractRate(contractId, body, actor) {
  const contract = await this.prisma.contract.findUnique({ where: { id: contractId } });
  const rate = await tx.contractRate.create({
    data: {
      contractId,
      meterId: body.meter_id,
      modelName: body.model_name || null,
      rate: this.decimal(body.rate, 'rate'),
      effectiveDate: body.effective_date,
      expiresDate: body.expires_date
    }
  });
  // This publishes a rate change event → engine refreshes cache:
  await this.publishRateChange(orgId, rateCardId, 'contract_rate.added');
  return rate;
}
```

---

## 4. Method 2: Publish a New Rate Card Version

### What Is a Rate Card Version?

Rate cards are **versioned** — every change creates a new version. A contract can be pinned to a specific version (for reproducibility) or track the latest.

### How Rate Card Versioning Works

```
Rate Card "Standard Rates"
├── Version 1 (Jan 1):  gpt-4 input = $0.000025
├── Version 2 (Mar 1):  gpt-4 input = $0.000020  ← NEW rates
├── Version 3 (Jun 1):  gpt-4 input = $0.000018  ← NEW rates
└── Version 4 (today):  gpt-4 input = $0.000015  ← NEW rates (just published)

Contracts can be:
  → Pinned to a SPECIFIC version: always uses Version 2 (reproducible)
  → Tracking LATEST: auto-picks Version 4 when published
```

### API: Add a Rate to a Rate Card

```typescript
POST /rate-cards/:rateCardId/rates
{
  "meter_id": "uuid",
  "model_name": "gpt-4",
  "token_type": "input",
  "rate": "0.000015",
  "unit_label": "token"
}
```

This automatically creates a **new rate card version**:

```typescript
private async createRateCardVersion(tx, rateCardId, orgId, changeType, changeSummary) {
  const lastVersion = await tx.rateCardVersion.findFirst({
    where: { rateCardId }, orderBy: { version: 'desc' }
  });
  const nextVersion = (lastVersion?.version ?? 0) + 1;
  
  // Snapshot ALL current rates into the version
  const rates = await tx.rateCardRate.findMany({ where: { rateCardId } });
  
  await tx.rateCardVersion.create({
    data: {
      rateCardId,
      orgId,
      version: nextVersion,
      changeType,
      snapshotData: rates,       // ← frozen copy of rates at this point
      changeSummary
    }
  });
}
```

### API: Publish a Rate Card (Activate for Contracts)

```typescript
PATCH /rate-cards/:rateCardId
{
  "status": "ACTIVE"    // transitions from DRAFT → ACTIVE
}
```

### What Happens to Contracts

| Contract Pin Setting | Behavior |
|---|---|
| `PinnedRateCardVersionID = "rcv_3"` | **Stays on version 3** — unaffected by new versions. Invoice reproducible. |
| `PinnedRateCardVersionID = null` (tracking latest) | **Picks up new version** — next invoice uses latest published version. |

### At Invoice Time (Engine)

```go
// engine/internal/rating/resolver.go
func resolveRateCard(inputs RatingInputs, contract Contract, req ResolveRequest) (Result, bool) {
    if contract.PinnedRateCardVersionID == "" {
        return Result{}, false  // no pinned version → skip
    }
    for _, version := range inputs.RateCardVersions {
        if version.ID != contract.PinnedRateCardVersionID {
            continue  // not the pinned version → skip
        }
        // Found the pinned version → search its rates
        for _, rate := range version.Rates {
            if rate.MeterID == req.MeterID && /* match dimensions */ {
                return resolved(rate.Rate, ...)
            }
        }
    }
}
```

**Key insight**: The `RatingInputs` snapshot contains the **pinned version** at the time of invoice generation. Even if the rate card has 10 newer versions, this invoice only sees the pinned one.

---

## 5. Method 3: Change Plan Pricing Model

### What Happens When You Edit a Plan's Pricing

Editing a plan's charges or pricing model creates a **new plan version**:

```
Plan "Pro GPT"
├── Version 1 (Jan 1):  PER_UNIT $0.000025/token
├── Version 2 (Jul 1):  PER_UNIT $0.000020/token  ← NEW pricing
└── (future versions...)
```

**Existing subscriptions** stay on the version they started with:
```
Customer ACME subscribed Jan 15 → pinned to Version 1 ($0.000025)
Customer BETA subscribed Jul 5  → gets Version 2 ($0.000020)
```

**To move an existing subscription to the new pricing**, you need a **plan change**:
```typescript
POST /subscriptions/:id/change-plan
{
  "plan_id": "pro_gpt_v2",    // new plan with new pricing
  "mode": "immediate"           // or "next_period"
}
```

This creates sub-windows (see QBILL_PLAN_CHANGE_EXPLAINED.md).

---

## 6. Method 4: Change Usage Limits / Entitlements

### Current QBill: Usage Limits

Usage limits are stored in `customer.usage_limits` — per customer, per meter, per period:

```prisma
model UsageLimit {
  id         String   // UUID
  orgId      String
  productId  String
  meterId    String
  limitType  String   // SOFT, HARD, WARNING
  limitValue Decimal
  period     String   // PER_MONTH, PER_YEAR, LIFETIME
}
```

### API: Update Usage Limit

```typescript
PATCH /organizations/:orgId/usage-limits/:limitId
{
  "limit_value": "2000000",     // new limit: 2M tokens
  "limit_type": "HARD"          // hard block when exceeded
}
```

### API: Add Customer-Specific Override

```typescript
POST /customers/:customerId/limit-overrides
{
  "meter_id": "uuid",
  "new_limit": "5000000",       // override to 5M tokens
  "expires_at": "2027-01-01"    // optional expiration
}
```

Overrides take **precedence** over plan-level limits:
```
Override precedence: 
  Default (plan-level limit) → Customer-specific limit_override (wins)
```

### What Happens on Change

```
Before:                                  After:
ACME: HARD limit = 1M tokens/month       ACME: HARD limit = 2M tokens/month
                                          
Enforcement check at next event:         Enforcement check at next event:
  usage = 950K, estimated = 100K          usage = 950K, estimated = 100K
  950K + 100K = 1.05M ≥ 1M → BLOCKED    950K + 100K = 1.05M ≤ 2M → ALLOWED
```

**Note**: This only affects **future** enforcement checks. Current period counters are NOT reset. If the customer already exceeded the old limit, changing the limit won't retroactively allow blocked requests.

### What's Missing: Proper Entitlements

As identified in the gap analysis, QBill's usage limits are **conflated with meters** — the same meter definition handles both "what we measure for billing" and "what we cap." A proper entitlement system (like Meteroid's or OpenMeter's) would separate these concerns.

---

## 7. What Happens to Existing Invoices?

### Already Issued Invoices

**Nothing changes.** Issued invoices are immutable (finalized → cannot be modified). The rate snapshot used at invoice time is preserved:

```prisma
model Invoice {
  // ...
  rateCardVersionId String?  // ← frozen at invoice time
  planVersionId     String?  // ← frozen at invoice time
  aggregationWatermark DateTime?  // ← frozen at invoice time
}
```

### Draft Invoices (Not Yet Finalized)

**Changes ARE picked up.** During the grace window (INVOICE_GRACE_HOURS, default 36h), draft invoices can be updated:

```
Period ends → draft invoice created (36h grace window)
                                           ↓
Rate card changes during grace window     ← YOU ARE HERE
                                           ↓
Draft invoice picks up new rates at finalization
                                           ↓
Grace expires → invoice finalized → IMMUTABLE
```

### Re-Rating (For Backward Changes)

If you need to change rates that AFFECT already-issued invoices, use **re-rating** (CR-1):

```typescript
POST /rerating-runs
{
  "scope": "invoice",              // or "customer", "org"
  "trigger": "rate_change",        // why this re-rating
  "period_start": "2026-06-01",
  "period_end": "2026-07-01",
  "invoice_id": "inv-xxx"          // if scope=invoice
}
```

Re-rating:
1. Re-runs invoice.Generate() with corrected inputs
2. Diffs against the issued invoice
3. Emits a credit note or debit adjustment
4. **Original invoice is NEVER mutated**

---

## 8. The Rating Cache — How Changes Propagate

### Cache Architecture

```
                            Rating Cache (in-memory, per org)
                            ┌──────────────────────────────────┐
                            │  org_abc: RatingInputs            │
                            │    ├── Contracts []              │
                            │    ├── ContractRates []           │
                            │    ├── RateCardVersions []        │
                            │    └── PlanCharges []             │
                            │                                  │
                            │  Refreshed every 60s (RATING_    │
                            │  CACHE_REFRESH_SECONDS)           │
                            │  Also refreshed on invalidation  │
                            └──────────────────────────────────┘
                                       │
                           ┌───────────┴───────────┐
                           ▼                       ▼
                    ┌────────────────┐    ┌─────────────────┐
                    │ Hot Path:      │    │ Cold Path:       │
                    │ Wallet Burn    │    │ Invoice Generate │
                    │ (real-time)    │    │ (period end)     │
                    │                │    │                  │
                    │ Uses Resolve-  │    │ Loads SNAPSHOT   │
                    │ HotPath() →    │    │ at moment of     │
                    │ cache lookup   │    │ invoice generation│
                    │ or last-known  │    │ → frozen forever  │
                    └────────────────┘    └─────────────────┘
```

### Cache Invalidation Flow

When you change a rate:

```
1. You call PATCH /contract-rates/:id → updates Postgres
2. BFF publishes: publishRateChange(orgId, cardId, 'contract_rate.updated')
3. Engine's InvalidationSource picks up the event
4. Engine calls Cache.Invalidate(orgId) 
5. Cache deletes the org's entry
6. Cache immediately reloads fresh data from Postgres
7. Next ResolveHotPath() call uses fresh data
```

### Cache Staleness (W-2, W-3)

| Scenario | Max Staleness | Behavior |
|---|---|---|
| **Normal refresh** | 60s (RATING_CACHE_REFRESH_SECONDS) | Cache reloads all orgs every 60s |
| **On change (invalidation)** | ~1-2s | Pub/sub event → immediate reload |
| **Cache miss (cold start)** | N/A | Uses last-known rate; flags exception if none exists (W-3) |

### What This Means for Your Changes

| You Change... | Hot Path (Wallet) Reflects In | Cold Path (Invoice) Reflects In |
|---|---|---|
| Contract rate | ~1-60s (cache refresh) | ✅ Next invoice generation |
| Rate card version | ~1-60s (cache refresh) | ✅ Next invoice generation (if contract tracks latest) |
| Plan pricing | N/A (subscription change) | ✅ Next period or after plan change |
| Usage limit | **Immediately** (Postgres read) | N/A (limits are enforcement-only) |

---

## 9. Impact Matrix

### What Changes Apply To

| You Change This... | Current Draft Invoice | Current Active Usage (Wallet) | Next Invoice | Already Issued Invoices |
|---|---|---|---|---|
| **Contract rate** | ✅ Yes (if in grace) | ✅ Yes (~60s) | ✅ Yes | ❌ No → needs re-rating |
| **Rate card version** (tracking latest) | ✅ Yes (if in grace) | ✅ Yes (~60s) | ✅ Yes | ❌ No → needs re-rating |
| **Rate card** (pinned to old version) | ❌ No | ❌ No | ❌ No | ❌ No |
| **Plan pricing** (via plan change) | ✅ Yes (sub-windows) | ✅ Yes (~60s) | ✅ Yes | ❌ No |
| **Usage limit** | N/A | ✅ Immediately | N/A | N/A (limits not on invoices) |
| **Wallet config** | N/A | ✅ Immediately | N/A | N/A |
| **Dunning policy** | N/A | N/A | ✅ Next dunning run | N/A |
| **Tax rate** | ✅ At finalization | N/A | ✅ Yes | ❌ No |

### Retroactive Changes (Via Re-Rating)

| You Want To... | Method | What Happens |
|---|---|---|
| Change a rate on an ALREADY ISSUED invoice | **Re-rating** (CR-1) | New invoice function run → diff → credit/debit note |
| Fix a billing error from last month | **Re-rating** with trigger=correction | Credit note for the difference |
| Apply negotiated rate retroactively | **Re-rating** with trigger=rate_change | Debit or credit depending on direction |

---

## 10. Code Walkthrough

### Flow: Change Contract Rate → Cache Invalidation → Next Invoice

```
Step 1: You call the API
────────
PATCH /contracts/contract_abc/rates/rate_xyz
→ BFF catalog.service.ts updateContractRate()
  → Updates Postgres
  → Creates audit log
  → Calls publishRateChange(orgId, rateCardId, 'contract_rate.updated')

Step 2: Cache invalidation
────────
Engine receives pub/sub event:
  func (c *Cache) Invalidate(ctx, orgID) {
      c.mu.Lock()
      delete(c.entries, orgID)         // remove stale entry
      for key := range c.lastKnown {
          if orgFromKey(key) == orgID {
              delete(c.lastKnown, key)  // clear last-known rates too
          }
      }
      c.mu.Unlock()
      c.refreshAndReport(ctx, orgID)    // immediately reload fresh data
  }

Step 3: Fresh data loaded
────────
  func (c *Cache) Refresh(ctx, orgID) {
      inputs, _ := c.loader.LoadRatingInputs(ctx, orgID)
      // LoadRatingInputs queries Postgres:
      //   SELECT * FROM customer.contracts WHERE org_id = ?
      //   SELECT * FROM billing.contract_rates WHERE ...
      //   SELECT * FROM catalog.rate_card_versions WHERE ...
      //   SELECT * FROM catalog.plan_charges WHERE ...
      c.Put(orgID, inputs, loadedAt)
  }

Step 4: Next ResolveHotPath() call
────────
  func (c *Cache) ResolveHotPath(req ResolveRequest) Result {
      entry := c.entries[req.OrgID]    // ← now has fresh data
      return Resolve(entry.inputs, req) // ← uses new contract rate
  }

Step 5: Next invoice generation
────────
  // At period end, engine calls:
  inputs := loadVersionedInputs(subscription) // ← reads current contract rates
  invoice := Generate(inputs)                 // ← uses new rate
  // Invoice stores rate_source = "contract_rate"
  // and rate_source_id = the contract rate ID
```

---

## 11. Common Questions

| Question | Answer |
|---|---|
| **Can I change a rate and have it apply immediately to wallet burndown?** | ✅ **Yes.** Contract rate changes propagate to the rating cache within ~1-60s. The hot path uses cache. |
| **Can I change a rate and have it affect the current period's invoice?** | ✅ **Yes, if the invoice is still DRAFT** (within grace window). ✅ **Yes, if you trigger re-rating** for already-issued invoices. |
| **Can I pin a customer to an old rate card version while others get the new one?** | ✅ **Yes.** Set `contract.PinnedRateCardVersionID` to the old version. That contract always uses that version. |
| **What if I want different rates for different customers on the same plan?** | ✅ **Contract rates** (step 1) override per customer. Create one contract_rate per customer with their negotiated price. |
| **What if I update a rate but forget to publish it?** | Rate cards have status DRAFT → ACTIVE. Draft rates are NOT used. You must publish (set status=ACTIVE) for the rate to be picked up. |
| **Can I change a usage limit mid-period?** | ✅ **Yes.** Updates take effect immediately for enforcement checks. It does NOT reset existing counters. |
| **Can I change a limit and have it apply retroactively?** | ❌ **No.** Enforcement checks are point-in-time. Previous blocks are not retroactively unblocked. |
| **What happens to the rating cache if the engine restarts?** | Cache is cold. First requests use last-known rates or fall back to rating exceptions (W-3). Cache warms within 60s. |

---

## Summary

| You Want To... | Use Method | Effective |
|---|---|---|
| Negotiate a special price for one customer | **Contract Rate** (step 1) | ✅ Immediate (~1-60s cache) |
| Update published rates for all customers | **Rate Card Version** (step 2) | ✅ Immediate (~1-60s cache) |
| Keep a customer on old pricing for contract term | **Pin contract to old rate card version** | ✅ Stays on version until unpinned |
| Change plan pricing for new customers only | **New plan version** | ✅ New subs get new version |
| Move existing customer to new pricing | **Subscription plan change** | ✅ Immediate or next period |
| Change usage caps mid-month | **Usage limit override** | ✅ Immediate enforcement |
| Fix a billing error from last month | **Re-rating** (CR-1) | ✅ Credit/debit note issued |

---

*End of document. Generated 2026-07-17.*
