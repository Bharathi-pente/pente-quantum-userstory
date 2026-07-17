# QBill — Mid-Period Plan Change Explained

**Date:** 2026-07-17
**Question:** What happens if I change to a different plan in the middle of a subscription period?

---

## Table of Contents

1. [Two Plan Change Modes](#1-two-plan-change-modes)
2. [How It Works — The Core Mechanism](#2-how-it-works--the-core-mechanism)
3. [Scenario A: Upgrade Immediate (Default)](#3-scenario-a-upgrade-immediate-default)
4. [Scenario B: Change at Next Period](#4-scenario-b-change-at-next-period)
5. [Worked Example — The Golden Test](#5-worked-example--the-golden-test)
6. [What Happens to Each Component](#6-what-happens-to-each-component)
7. [Code Walkthrough](#7-code-walkthrough)
8. [What You Cannot Do](#8-what-you-cannot-do)
9. [Common Questions](#9-common-questions)

---

## 1. Two Plan Change Modes

QBill supports **two modes** for changing plans mid-period:

```typescript
// API call
POST /subscriptions/:id/change-plan
{
  "plan_id": "plan_pro_annual",
  "mode": "immediate",          // default — takes effect now
  "effective_date": "2026-07-20" // only for immediate mode (optional)
}

// OR
{
  "plan_id": "plan_pro_annual",
  "mode": "next_period"         // takes effect at next period start
}
```

| Mode | When Change Takes Effect | Proration |
|---|---|---|
| **`immediate`** (default) | **Right now** — at 00:00 UTC of today (or specified `effective_date`) | ✅ **Yes** — period split into sub-windows, each billed separately |
| **`next_period`** | At `current_period_end` — next billing cycle | ❌ **No** — current period completes on old plan, new plan starts fresh |

### Key Design Principle (P-5)

```
P-5: Upgrades apply immediately; downgrades apply at next period start
     (default; per-plan override allowed).
     
     Upgrades (more expensive / more features):
       → immediate — customer gets benefits right away
       → prorated charge for remaining days
     
     Downgrades (cheaper / fewer features):
       → next_period — no mid-period allowance clawback
       → current period completes unchanged
```

---

## 2. How It Works — The Core Mechanism

### The Sub-Window Concept

When you change plans mid-period, QBill **splits the billing period** into sub-windows at the plan change boundary:

```
Before Change:                  After Change:
─────────────────────────       ─────────────────────────
Period: Jul 1 → Jul 31          Period: Jul 1 → Jul 31
Plan:   Pro ($99/mo)            Plan:  Pro (first 20 days)
                                 Plan+: Pro+ (last 11 days)
                                 
Invoice:                        Invoice:
  BASE_FEE: $99.00 (full)         BASE_FEE (Pro, Jul 1-20):  $63.87  ← prorated
  USAGE: all at Pro rates         BASE_FEE (Pro+, Jul 21-31): $70.97  ← prorated
                                  USAGE (Jul 1-20): at Pro rates
                                  USAGE (Jul 21-31): at Pro+ rates
                                  OVERAGE (each sub-window separately)
```

### How Sub-Windows Work (from `engine/internal/invoice/periods.go`)

```go
// subWindows splits the invoice window at 00:00 UTC plan-change boundaries
// (P-1) and at trial end, and rates each segment against the plan version
// active during it.
func subWindows(window Window, plans []PlanWindow, trialEnd *time.Time) ([]subWindow, error) {
    // 1. Start with the period window [period_start, period_end)
    // 2. Add boundaries at each plan's EffectiveFrom time
    // 3. Add boundary at trial end (if any)
    // 4. Sort all boundaries
    // 5. Create segments between consecutive boundaries
    // 6. Each segment gets the plan version active during it
}
```

### Proration Formula (P-1, P-2, P-3)

```
For each sub-window:

Base fee:  round_half_up(base_amount × segDays / periodDays, minor_units)
           Example: $99 × 20/31 = $63.87

Allowance: floor(included_units × segDays / periodDays)
           Example: floor(1,000,000 × 20/31) = floor(645,161.29) = 645,161 tokens

Seats:     seat_price × seats × segDays / periodDays → round to minor
```

### Plan Versioning

Each plan change creates a **new plan version** with an `effective_from` timestamp:

```
Plan Versions for this subscription:
┌──────────────────────────────────────────────────────────┐
│ Version 1: "Pro"                                         │
│   EffectiveFrom: Jul 1 00:00:00 UTC                      │
│   EffectiveTo:   Jul 20 00:00:00 UTC                     │  ← closed when plan changed
│   BaseAmount: $99.00                                      │
│                                                          │
│ Version 2: "Pro+"                                        │
│   EffectiveFrom: Jul 20 00:00:00 UTC                      │
│   EffectiveTo:   null (current)                          │  ← open-ended
│   BaseAmount: $199.00                                     │
└──────────────────────────────────────────────────────────┘
```

These versions are stored in `catalog.plan_versions` and used at invoice time to reproduce the correct pricing for each sub-window.

---

## 3. Scenario A: Upgrade Immediate (Default)

### What Happens

```
Timeline: Upgrade from Pro ($99/mo) to Pro+ ($199/mo) on Jul 15
───────────────────────────────────────────────────────────────────────
Jul 1                           Jul 15                          Jul 31
│━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━│
│                                                                   │
│  Sub-Window A (14 days)          Sub-Window B (17 days)           │
│  Plan: Pro ($99/mo)              Plan: Pro+ ($199/mo)             │
│  Allowance: 1M tokens            Allowance: 3M tokens             │
│                                                                   │
│  ▲                                                                │
│  │ Plan change request (Jul 15, mode=immediate)                   │
│  │                                                                 │
│  └──► Subscription.planId → Pro+                                  │
│      ► Pro plan version closed (EffectiveTo = Jul 15)             │
│      ► Pro+ plan version created (EffectiveFrom = Jul 15)         │
│      ► No immediate invoice — change visible at period end        │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
                                                                   │
                                                                   ▼
                                                    At period end (Jul 31):
                                                    Invoice generated with
                                                    TWO sub-windows:
                                                    
                                                    BASE_FEE (Pro, Jul 1-15):     $45.00
                                                    BASE_FEE (Pro+, Jul 16-31):  $102.00
                                                    USAGE (Pro window)            at Pro rates
                                                    USAGE (Pro+ window)           at Pro+ rates
```

### Step-by-Step

| Step | What Happens | Who Does It |
|---|---|---|
| **1** | Customer/Admin calls `POST /subscriptions/:id/change-plan` with `plan_id: "pro_plus"`, `mode: "immediate"` | BFF (NestJS) |
| **2** | Validate: plan exists in same org, currency matches, subscription is active | BFF |
| **3** | Close current plan version: set `effectiveTo = today` | BFF → Postgres |
| **4** | Create new plan version: set `effectiveFrom = today`, link to new plan | BFF → Postgres |
| **5** | Update subscription: `planId = pro_plus` | BFF → Postgres |
| **6** | Publish event: `subscription.updated` | BFF → Redis Pub/Sub |
| **7** | NO immediate invoice — everything waits for period end | — |
| **8** | At period end, engine billing worker scans subscription | Engine |
| **9** | Engine reads plan versions: finds TWO covering this period | Engine |
| **10** | Engine calls `subWindows()` to split period at plan change boundary | Engine → `periods.go` |
| **11** | For each sub-window: compute prorated base fee, allowance, rate usage | Engine → `generate.go` |
| **12** | Combine all sub-window lines into one invoice | Engine |
| **13** | Draft invoice created with multiple line items per sub-window | Engine → Postgres |

### Important: No Mid-Period Invoice

QBill does **NOT** generate an invoice at the time of plan change. Instead:
- The plan change is recorded (new plan version created)
- Everything is resolved at the **next period end**
- The single invoice covers the full period with split sub-windows

This is different from some competitors (e.g., Orb generates two invoices — one closing the old plan, one opening the new plan).

---

## 4. Scenario B: Change at Next Period

### What Happens

```
Timeline: Schedule change from Pro ($99/mo) to Basic ($49/mo) at next period
───────────────────────────────────────────────────────────────────────
Jul 1                                                  Jul 31         Aug 1
│━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━│━━━━━━━━━━━━━│
│                                                      │              │
│  Current Period (Jul 1-31)                          │ Next Period  │
│  Plan: Pro ($99/mo) — FULL, no proration             │ Plan: Basic  │
│  Full allowance: 1M tokens                           │ ($49/mo)     │
│                                                      │              │
│  ▲ Plan change request (Jul 15, mode=next_period)    │              │
│  │ cancelAtPeriodEnd NOT set (this is NOT cancel)    │              │
│  │ Instead: scheduled plan change at period end      │              │
│  │ Current period continues FULL on Pro              │              │
│  └─► Pro continues until Jul 31                     │              │
│      ► Basic starts Aug 1                            │              │
│                                                      │              │
└──────────────────────────────────────────────────────┴──────────────┘
                                                       │
                                                       ▼
                                              Jul 31 period end:
                                              Invoice: FULL Pro ($99 + usage)
                                              No proration.
                                              Aug 1: New period on Basic.
```

### When to Use Each Mode

| Situation | Use Mode | Why |
|---|---|---|
| Customer wants **more features/tokens** right now | `immediate` | They get the upgrade benefits immediately; pay prorated difference |
| Customer wants to **downgrade** to save money | `next_period` | No mid-period allowance clawback; current period completes as-is |
| Customer requests **plan change mid-cycle** (default) | `immediate` | Default mode — works for most cases |
| Enterprise **contract renewal** with new pricing | `next_period` | Clean break at contract boundary |

---

## 5. Worked Example — The Golden Test

This is the **exact test** that runs in CI (`TestGoldenBillingMathSection9WorkedExample`):

### The Scenario (from BILLING_MATH.md §9)

```
Plan A: $99.00/mo base, 1,000,000 included tokens, overage $0.000025/token
Plan B: $199.00/mo base, 2,000,000 included tokens, overage $0.000025/token
Anchor: Jan 31
Period: Jan 31 → Feb 28 (28 days)
Upgrade: Feb 14 (14 days into the 28-day period)
```

### Sub-Window Calculation

```
Sub-Window A: Jan 31 → Feb 14 (14 days)
  Plan: Plan A ($99/mo)
  Factor: 14/28 = 0.5
  Base fee:  round(99.00 × 0.5, 2)  = $49.50
  Allowance: floor(1,000,000 × 0.5) = 500,000 tokens
  Usage:     620,000 tokens
  Overage:   max(0, 620,000 - 500,000) = 120,000 tokens
  Overage $: round(120,000 × 0.000025, 2) = $3.00

Sub-Window B: Feb 14 → Feb 28 (14 days)
  Plan: Plan B ($199/mo)
  Factor: 14/28 = 0.5
  Base fee:  round(199.00 × 0.5, 2)  = $99.50
  Allowance: floor(2,000,000 × 0.5)  = 1,000,000 tokens
  Usage:     800,000 tokens
  Overage:   max(0, 800,000 - 1,000,000) = 0 tokens
  Overage $: $0.00
```

### The Invoice

```
INVOICE (Jan 31 → Feb 28)
┌─────────────────────────────────────────────────┐
│ LINE ITEMS                                      │
│                                                  │
│ BASE_FEE  (Plan A, Jan 31 - Feb 14)     $49.50  │
│ BASE_FEE  (Plan B, Feb 14 - Feb 28)     $99.50  │
│ USAGE     (Plan A window, 620K tokens)   $15.50  │
│ OVERAGE   (Plan A, 120K tokens)           $3.00  │
│ USAGE     (Plan B window, 800K tokens)   $20.00  │
│ OVERAGE   (Plan B)                        $0.00  │
│ ─────────────────────────────────────────────── │
│ SUBTOTAL                                $187.50  │
│                                                  │
│ CREDITS APPLIED (FEFO):                         │
│   Promotional $10 (expires Mar 1)       -$10.00  │
│   Prepaid $200                           -$142.00 │
│   (Prepaid remaining: $58.00)                    │
│                                                  │
│ TAXABLE ($187.50 - $152.00)              $35.50  │
│ TAX (0%)                                  $0.00  │
│ ─────────────────────────────────────────────── │
│ TOTAL                                    $35.50  │
└─────────────────────────────────────────────────┘
```

### The Test Assertions

```go
// From golden_test.go — exact values that must match
expected := invoice.Invoice{
    Subtotal:       decimal.NewFromFloat(187.50),
    CreditsApplied: decimal.NewFromFloat(152.00),
    TaxAmount:      decimal.Zero,
    Total:          decimal.NewFromFloat(35.50),
}
```

---

## 6. What Happens to Each Component

### Base Fee (P-2)

| Before Change | After Change |
|---|---|
| Full month: $99.00 | **Sub-window A**: $99 × 14/28 = **$49.50** |
| | **Sub-window B**: $199 × 14/28 = **$99.50** |

Each sub-window gets its own `BASE_FEE` line on the invoice, labeled with the plan name and date range.

### Included Allowance (P-3)

| Before Change | After Change |
|---|---|
| Full allowance: 1,000,000 tokens | **Sub-window A**: floor(1,000,000 × 14/28) = **500,000** |
| | **Sub-window B**: floor(2,000,000 × 14/28) = **1,000,000** |

Usage events are attributed to sub-windows by `timestamp_ms`. Overage is measured per sub-window against that window's prorated allowance.

### Usage Rating

```
Sub-window A usage → rated against Plan A's pricing model
Sub-window B usage → rated against Plan B's pricing model

Events are bucketed by timestamp_ms:
  Jan 31 15:00 UTC → Sub-window A (before upgrade)
  Feb 05 09:00 UTC → Sub-window A (before upgrade)
  Feb 13 23:00 UTC → Sub-window A (before upgrade)
  Feb 20 12:00 UTC → Sub-window B (after upgrade)
```

### Seats (P-4)

| Before Change | After Change |
|---|---|
| 5 seats included in Plan A | Seat count for sub-window A = 5 |
| 10 seats included in Plan B | Seat count for sub-window B = 10 |
| 8 active seats | Sub-window A: 8 - 5 = 3 billable seats, prorated |
| | Sub-window B: 8 - 10 = 0 billable seats (within included) |

### Rate Cards & Pricing

Each sub-window uses the rate cards and pricing models attached to the plan version active during that window. If contract rates exist, they override both (rating waterfall step 1).

### Credits (FEFO)

Credits are applied to the **total invoice** (across all sub-windows), not per sub-window. The FEFO ordering (priority → expires_at → created_at) works the same way regardless of plan changes.

### Wallet

Wallet burndown continues in real-time regardless of plan changes. The rate used for real-time burndown may change immediately (if the new plan has different rates in the rating cache).

### Stripe

No Stripe interaction at plan change time. Stripe is charged when the final invoice is generated at period end.

---

## 7. Code Walkthrough

### Step 1: BFF Receives Plan Change Request

```typescript
// control-plane/src/catalog/catalog.service.ts
async changeSubscriptionPlan(subscriptionId, body, actor) {
  const sub = await this.requireSubscriptionWritable(subscriptionId, actor);
  const nextPlan = await this.requirePlanInOrg(body.plan_id, sub.orgId);

  // Validate mode
  const mode = body.mode ?? 'immediate';  // default: immediate

  // Currency check: plan change cannot switch currency
  if (sub.plan.currency !== nextPlan.currency) {
    throw new Error('cancel and re-subscribe to bill in a new currency');
  }

  // Determine effective date
  const effective = (mode === 'next_period')
    ? sub.currentPeriodEnd
    : (body.effective_date ?? today);

  // Database transaction
  await this.prisma.$transaction(async (tx) => {
    // 1. Close the current plan version
    await tx.planVersion.updateMany({
      where: { planId: sub.planId, effectiveTo: null },
      data: { effectiveTo: effective }
    });

    // 2. Create a new plan version for the new plan
    await this.createPlanVersion(tx, nextPlan.id, 
      'subscription_plan_change', effective, sub.id, sub.currentPeriodEnd);

    // 3. If immediate, update the subscription's plan reference
    if (mode === 'immediate') {
      await tx.subscription.update({
        where: { id: sub.id },
        data: { planId: nextPlan.id }
      });
    }
    // If next_period: subscription stays on old plan until period end
    // Engine picks up the new plan version at period end
  });
}
```

### Step 2: Engine Picks Up at Period End

```go
// engine/internal/invoice/periods.go
// At invoice time, the engine reads ALL plan versions for this subscription
// that overlap with the invoice window:

func subWindows(window Window, plans []PlanWindow, trialEnd *time.Time) ([]subWindow, error) {
    // plans = [
    //   { PlanID: "pro", BaseAmount: 99, EffectiveFrom: Jan 31, EffectiveTo: Feb 14 },
    //   { PlanID: "pro_plus", BaseAmount: 199, EffectiveFrom: Feb 14, EffectiveTo: nil }
    // ]
    
    boundaries = [Jan 31, Feb 14, Feb 28]  // start, plan change, end
    segments = [
        { Window: [Jan 31, Feb 14), Plan: Pro },
        { Window: [Feb 14, Feb 28), Plan: Pro+ }
    ]
    return segments, nil
}
```

### Step 3: Invoice Generation for Each Sub-Window

```go
// engine/internal/invoice/generate.go
for i, seg := range segments {
    segDays := calendarDays(seg.Window.Start, seg.Window.End)  // 14 days each

    // Prorated base fee (P-2)
    prorated := seg.Plan.BaseAmount × segDays / periodDays
    // Sub-window A: 99 × 14/28 = 49.50
    // Sub-window B: 199 × 14/28 = 99.50

    // Prorated allowance (P-3)
    allowance = floor(seg.Plan.IncludedUnits × segDays / periodDays)
    // Sub-window A: floor(1,000,000 × 14/28) = 500,000
    // Sub-window B: floor(2,000,000 × 14/28) = 1,000,000

    // Rate usage for this segment using its plan's pricing model
    segLines, segExceptions = rateSegmentUsage(in, seg, usageForSegment(usage, segments, i), segDays, periodDays, minor)
}
```

---

## 8. What You Cannot Do

| Operation | Why It's Blocked | Error Message |
|---|---|---|
| **Change to a plan in a different currency** | X-1: one currency per customer | `"cancel and re-subscribe to bill in a new currency"` |
| **Change plan on a cancelled subscription** | Subscription is not writable | `"SUBSCRIPTION_ALREADY_CANCELED"` |
| **Change to a plan in a different org** | Cross-org access not allowed | `"PLAN_NOT_FOUND"` |
| **Change with invalid mode** | Must be `immediate` or `next_period` | `"mode must be immediate or next_period"` |

---

## 9. Common Questions

| Question | Answer |
|---|---|
| **Is there an immediate invoice when I change plans?** | ❌ **No.** No invoice is generated at plan change time. Everything is resolved at period end. |
| **What if I change plans multiple times in one period?** | ✅ **Supported.** Each plan change creates a new sub-window boundary. All sub-windows are resolved in a single invoice at period end. |
| **How is usage attributed to each sub-window?** | By `timestamp_ms`. Events with timestamps before the plan change go to sub-window A; events after go to sub-window B. |
| **Can I set a future effective date?** | ✅ Yes. `effective_date` parameter in immediate mode lets you schedule the change for a specific future date. |
| **What happens to my wallet balance?** | ✅ **Unchanged.** Wallet continues burning in real-time regardless of plan changes. Rate may change if new plan has different rates. |
| **What if I upgrade then downgrade in the same period?** | Three sub-windows: old plan → new plan (upgrade) → older plan (downgrade). Each window rated against its plan version. |
| **Can I change plan and cancel in the same period?** | ✅ Yes. Plan change creates sub-windows. Cancellation (immediate) triggers final invoice with all sub-windows resolved. |
| **Does the customer get notified?** | If webhooks are configured, `subscription.updated` event is published. The BFF can trigger email notifications. |

---

## Summary

| Scenario | Behavior |
|---|---|
| **Upgrade mid-period (immediate)** | Period split into sub-windows. Each sub-window prorated and rated against its plan version. Single invoice at period end. |
| **Downgrade at next period** | Current period completes on old plan (no proration). New plan starts at period boundary. |
| **Multiple plan changes** | Each change creates a sub-window boundary. All resolved in one invoice. |
| **Currency change** | ❌ **Not allowed.** Must cancel and re-subscribe. |
| **Immediate invoice at change time** | ❌ **Not generated.** Everything waits for period end. |

---

*End of document. Generated 2026-07-17.*
