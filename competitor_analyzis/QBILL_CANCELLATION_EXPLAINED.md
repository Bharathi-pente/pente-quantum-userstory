# QBill — Mid-Month Cancellation Explained

**Date:** 2026-07-17
**Question:** What happens if a subscription is cancelled in the middle of the month?

---

## Table of Contents

1. [Two Cancellation Modes](#1-two-cancellation-modes)
2. [Scenario A: Cancel at Period End (Default)](#2-scenario-a-cancel-at-period-end-default)
3. [Scenario B: Cancel Immediately](#3-scenario-b-cancel-immediately)
4. [What Happens to Proration](#4-what-happens-to-proration)
5. [What Happens to Wallet & Credits](#5-what-happens-to-wallet--credits)
6. [What Happens to Contracts & Commit](#6-what-happens-to-contracts--commit)
7. [What Happens to Auto-Collection & Dunning](#7-what-happens-to-auto-collection--dunning)
8. [What Happens to Stripe](#8-what-happens-to-stripe)
9. [What Happens to Revenue Recognition](#9-what-happens-to-revenue-recognition)
10. [Code Walkthrough: How It's Implemented](#10-code-walkthrough-how-its-implemented)
11. [Quick Reference: Cancellation Effects Matrix](#11-quick-reference-cancellation-effects-matrix)

---

## 1. Two Cancellation Modes

QBill supports **two cancellation modes**:

```typescript
// API call
POST /subscriptions/:id/cancel
{
  "mode": "end_of_period" | "immediate",
  "reason": "Customer requested downgrade"   // optional
}
```

| Mode | Effect | When It Takes Effect |
|---|---|---|
| **`end_of_period`** (default) | Subscription continues until current billing period ends, then auto-renew stops | At `current_period_end` |
| **`immediate`** | Subscription ends RIGHT NOW. Prorated final invoice generated at cancellation time | Immediately |

### Subscription Status Flow

```
scheduled ──► trialing ──► active ──► canceled  (immediate)
                                 └──► (stays active until period end)
                                       └──► ended  (end_of_period)

suspended ──► reactivated ──► active
```

---

## 2. Scenario A: Cancel at Period End (Default)

### What Happens

```
Timeline:
───────────────────────────────────────────────────────
Jul 1                              Jul 31            Aug 1
│━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━│━━━━━━━━━━━━━━━━━│
│  Current Period                  │  Next Period     │
│  (active)                        │  (never starts)  │
│                                  │                  │
│  ▲                               │                  │
│  │                               │                  │
│  Cancel request (Jul 15)         │                  │
│  cancelAtPeriodEnd = true        │                  │
│                                  │                  │
│  Subscription stays ACTIVE       │                  │
│  Usage continues to be metered   │                  │
│  Wallet continues to burn        │                  │
│  Redis counters keep updating    │                  │
│                                  │                  │
│                                  │                  │
└──────────────────────────────────┴──────────────────┘
                                  │
                                  ▼
                         Jul 31 (period_end):
                         1. Final invoice generated (full period)
                         2. Auto-collection triggered
                         3. Subscription status → ended
                         4. No renewal → Aug period never opens
                         5. Redis counters reset NOT done (no next period)
```

### What Happens to Each Component

| Component | Effect |
|---|---|
| **Subscription status** | Stays `active` until `current_period_end`, then transitions to `ended` |
| **`cancelAtPeriodEnd` flag** | Set to `true` immediately on cancellation request |
| **Usage metering** | Continues normally — events still tracked, counters updated |
| **Wallet burndown** | Continues — wallet still burns in real-time |
| **Invoice generation** | Normal final invoice at period end — covers FULL period (no proration) |
| **Base fee** | Charged for the FULL period (already paid if pay_in_advance) |
| **Next period** | NEVER OPENS — no renewal, no new invoice |
| **Redis counters** | NOT reset (no new period to reset for) |
| **Auto-collection** | Runs normally on the final invoice |
| **Stripe** | Normal charge for final invoice |
| **Credits** | Unused credits remain available (expire on their own schedule) |
| **Contract commit** | If at contract end → COMMIT_TRUE_UP line on final invoice |
| **Revenue recognition** | Final recognition entries for the period |

### BFF Code (How It's Implemented)

```typescript
// From catalog.service.ts - cancelSubscription()
async cancelSubscription(subscriptionId: string, body, actor) {
  const mode = requiredString(body.mode, 'mode');
  
  if (mode === 'end_of_period') {
    // Just set the flag — subscription stays active
    await tx.subscription.update({
      where: { id: sub.id },
      data: { cancelAtPeriodEnd: true }    // ← ONLY this changes
    });
    // Status stays 'active'
    // Current period continues normally
    // Period end will trigger: invoice → no renewal → status → ended
  }
}
```

---

## 3. Scenario B: Cancel Immediately

### What Happens

```
Timeline:
───────────────────────────────────────────────────────
Jul 1                    Jul 15                     Jul 31
│━━━━━━━━━━━━━━━━━━━━━━━│━━━━━━━━━━━━━━━━━━━━━━━━━━│
│  Active Period         │  Cancelled                │
│                        │                           │
│  ▲                     │                           │
│  │                     │                           │
│  Cancel request        │                           │
│  mode = "immediate"    │                           │
│                        │                           │
│  │                     │                           │
│  ▼                     │                           │
│  IMMEDIATELY:          │                           │
│  ┌─────────────────┐   │                           │
│  │ Status → canceled│   │                           │
│  │ endDate = today  │   │                           │
│  │ Final invoice    │   │                           │
│  │ generated NOW    │   │                           │
│  │ Prorated base    │   │                           │
│  │ fee (15/31 days) │   │                           │
│  │ Usage up to now  │   │                           │
│  │ Auto-collect     │   │                           │
│  └─────────────────┘   │                           │
│                        │                           │
│  After this point:     │                           │
│  ❌ No more usage     │                           │
│  ❌ Events rejected   │                           │
│  ❌ Wallet frozen     │                           │
│  ❌ Redis stopped     │                           │
└────────────────────────┴───────────────────────────┘
```

### What Changes Immediately

| Component | Effect |
|---|---|
| **Subscription status** | Immediately → `canceled` |
| **`endDate`** | Set to today (cancellation date) |
| **`cancelAtPeriodEnd`** | Set to `false` |
| **Final invoice** | Generated IMMEDIATELY with prorated charges |
| **Usage after cancel** | ❌ **NOT accepted** — events for cancelled subscription are rejected |
| **Wallet burndown** | ❌ **STOPS** — wallet frozen |
| **Redis counters** | ❌ **STOP** — no more updates |
| **Auto-collection** | Runs on the final invoice right away |
| **Stripe** | Charge for prorated amount |
| **Credits** | Unused credits remain (expire on their own schedule) |

### BFF Code

```typescript
// From catalog.service.ts - cancelSubscription()
if (mode === 'immediate') {
  await tx.subscription.update({
    where: { id: sub.id },
    data: { 
      status: SubscriptionStatus.canceled,  // ← immediately changed
      endDate: this.todayUtcDate(),          // ← today's date
      cancelAtPeriodEnd: false
    }
  });
  // Engine picks up the cancelled status on next scan
  // → Generates final prorated invoice
}
```

---

## 4. What Happens to Proration

### Proration Rules (BILLING_MATH.md P-1 to P-5)

```
P-1: Day-based, calendar-day granularity
P-2: Base fee prorated: round_half_up(base_amount × days_used / total_days, minor)
P-3: Included units prorated: floor(included_units × days_used / total_days)
P-4: Seats prorated day-based
P-5: Upgrades immediate, downgrades next period, CANCELLATION PRORATES BASE FEE
```

### Worked Example: Cancel Immediate on Day 15 of 31-Day Month

**Plan:** Pro ($99/mo, 1,000,000 included tokens, $0.025/1K overage)
**Used:** 620,000 tokens in 15 days
**Cancelled:** Day 15 (immediate)

**Calculation:**

```
Proration factor: f = 15 / 31 = 0.483870967...

Base fee:   round(99.00 × 0.4839, 2)     = $47.90
Allowance:  floor(1,000,000 × 0.4839)    = 483,870 tokens
Usage:      620,000 tokens
Overage:    max(0, 620,000 - 483,870)    = 136,130 tokens
Overage $:  round(136,130 × 0.000025, 2) = $3.40

Final Invoice:
  BASE_FEE (prorated Jul 1-15)     $47.90
  USAGE (620,000 tokens rated)      $15.50  ← at contract rate
  OVERAGE (136,130 tokens)          $3.40
  ────────────────────────────────────────
  Subtotal                          $66.80
  Tax (8%)                          $5.34
  TOTAL                             $72.14
```

### What About Pay-In-Advance?

If the customer already paid $99 at the start of the month (pay_in_advance):

```
Already paid:       $99.00
Prorated charge:    $47.90
Credit due:         $99.00 - $47.90 = $51.10

What happens to the $51.10 credit?

OPTION A (current QBill behavior):
  → Credit remains as account credit (FEFO, priority = promotional)
  → Applied to future invoices (but no future invoices since cancelled)
  → Customer must request refund via support

OPTION B (what some competitors do):
  → Auto-refund to original payment method
  → Orb: credit to customer balance (post-tax)
  → Stripe: can issue partial refund
```

**Current QBill behavior:** The overpaid amount stays as a credit on the account. It does NOT auto-refund. The customer must contact support for a manual refund (which is processed as a credit note → refund).

---

## 5. What Happens to Wallet & Credits

### Wallet on Cancellation (Immediate)

```
Wallet Balance: $200.00 USD
Cancellation: Immediate on Jul 15

What happens:
┌────────────────────────────────────────────┐
│ Before cancel:                             │
│   Wallet: $200.00                          │
│   Active: yes                              │
│   Real-time burndown: running              │
│                                            │
│ At cancel:                                 │
│   Wallet: $200.00 (unchanged)              │
│   Status: frozen (no more burndown)        │
│   Real-time burndown: STOPPED              │
│                                            │
│ After cancel:                              │
│   Wallet balance stays at $200.00          │
│   No future usage to burn against          │
│   Refund? → Manual process via support     │
└────────────────────────────────────────────┘
```

### Credits on Cancellation

| Credit Type | Behavior on Cancel |
|---|---|
| **Compensation** (priority 0) | Remains — expires on own schedule |
| **Promotional** (priority 1) | Remains — expires on own schedule |
| **Prepaid** (priority 2) | Remains — customer paid for these |
| **Commit** (priority 3) | Evaluated at contract end → COMMIT_TRUE_UP on final invoice |

### Refund Flow (Manual)

```
Customer requests refund
         │
         ▼
Support/admin creates a Credit Note
         │
         ▼
POST /v1/credit-notes
{
  "invoice_id": "inv_final",
  "kind": "credit",
  "amount": "72.14",
  "currency": "USD",
  "reason": "cancellation_refund"
}
         │
         ▼
Credit Note issued → applied → refunded
         │
         ▼
Stripe refund (manual):
  Refund PaymentIntent for $72.14
```

---

## 6. What Happens to Contracts & Commit

### Contract with Commit Amount

If the customer has a contract with a commit amount:

```
Contract: $1,000/mo commit, 12-month term
Customer cancels immediately on month 6

Term spend so far: $3,200 (usage charges only)
Commit amount:     $1,000 × 6 = $6,000
True-up:           max(0, $6,000 - $3,200) = $2,800

The $2,800 true-up appears as a COMMIT_TRUE_UP line
on the final invoice at contract end.
```

**C-3 rule:** Commit true-up is only evaluated on the **final invoice of the contract term**. If cancellation happens mid-term, the true-up is still calculated when the contract end date is reached (unless the contract is also terminated).

---

## 7. What Happens to Auto-Collection & Dunning

### Auto-Collection on Final Invoice

```
Final Invoice Generated (immediate or at period end)
         │
         ▼
Invoice state: draft → pending (grace window skipped for cancellation)
         │
         ▼
Auto-collection triggered:
  Stripe PaymentIntent for final amount
         │
         ├── Success → Invoice: paid
         │             Dunning: cancelled (cancel-on-pay)
         │
         └── Failure → Invoice: overdue
                       Dunning: starts (EMAIL at day 3, SMS at day 7...)
                       Customer: may be suspended
```

### Dunning on Cancellation

From `engine/internal/billingworker/pgcollection.go`:

```go
// cancel-on-pay: when customer pays mid-dunning,
// all pending dunning communications are cancelled
func (s *Store) CancelDunningCommunications(ctx context.Context, invoiceID string) error {
    // Updates status from PENDING → CANCELLED
    // Next dunning cron will skip cancelled items
}
```

---

## 8. What Happens to Stripe

### Immediate Cancellation

```
Step 1: Final invoice generated (prorated $72.14)
Step 2: Stripe PaymentIntent created
        POST /payment_intents
        {
          amount: 7214,          ← 72.14 × 100 (cents)
          currency: "usd",
          customer: "cus_xxx"
        }
Step 3: Payment succeeds → invoice paid

What about the already-paid $99?
→ No auto-refund. Customer must request manually.
→ Support processes a Credit Note + Stripe refund.
```

### End-of-Period Cancellation

```
Step 1: Normal final invoice at period end ($147.42 for full month)
Step 2: Normal Stripe charge for $147.42
Step 3: No renewal → no more charges
Step 4: If already paid in advance → no action needed (period was paid)
```

---

## 9. What Happens to Revenue Recognition

### Immediate Cancellation

```
Before cancel:
  $99 base fee booked as deferred revenue at period start
  Daily ratable recognition: $99 / 31 = $3.19/day
  Recognized so far (15 days): $47.90

At cancel:
  Remaining deferral: $51.10
  ← This stays as deferred (no more recognition since no service)
  ← Or written off if cancellation policy dictates
  
  Final invoice:
  Total: $72.14
  Recognized: $47.90 (base fee already recognized)
  + Usage revenue recognized on consumption
```

---

## 10. Code Walkthrough: How It's Implemented

### Step 1: BFF Receives Cancel Request

```typescript
// control-plane/src/catalog/catalog.service.ts
async cancelSubscription(subscriptionId, body, actor) {
  const sub = await this.requireSubscriptionWritable(subscriptionId, actor);
  
  // Validate status
  if (sub.status === 'canceled' || sub.status === 'ended')
    throw new Error('Already canceled');

  // Validate mode
  const mode = body.mode; // 'immediate' or 'end_of_period'
  if (!['immediate', 'end_of_period'].includes(mode))
    throw new Error('Mode must be immediate or end_of_period');

  // Update database
  const updated = await this.prisma.$transaction(async (tx) => {
    if (mode === 'immediate') {
      // Set status to canceled, endDate to today
      await tx.subscription.update({
        where: { id: sub.id },
        data: {
          status: 'canceled',
          endDate: new Date(),
          cancelAtPeriodEnd: false
        }
      });
    } else {
      // Just set the flag — stays active until period end
      await tx.subscription.update({
        where: { id: sub.id },
        data: { cancelAtPeriodEnd: true }
      });
    }
  });

  // Publish event (engine picks this up)
  await this.publishSubscription(orgId, 'subscription.canceled', sub.id);
}
```

### Step 2: Engine Picks Up the Change

The engine's billing worker scans for subscriptions that need attention:

```go
// engine/internal/billingworker/invoicerun.go (conceptual)
func (w *Worker) scanSubscriptions(ctx context.Context) {
    // Find subscriptions where:
    //   1. status = 'canceled' OR 'ended'
    //   2. final invoice not yet generated
    //   3. current_period_end has passed (or immediate cancel)
    
    for _, sub := range dueSubscriptions {
        if sub.CancelAtPeriodEnd && sub.CurrentPeriodEnd.Before(time.Now()) {
            // End-of-period cancel: generate final invoice for full period
            // Then do NOT create next period
            w.generateFinalInvoice(sub)
        }
        
        if sub.Status == "canceled" && sub.EndDate != nil {
            // Immediate cancel: generate prorated final invoice
            // Window: period_start → endDate (today)
            w.generateProratedFinalInvoice(sub, sub.EndDate)
        }
    }
}
```

### Step 3: Invoice Generation for Immediate Cancel

```go
// engine/internal/invoice/generate.go
// The Generate() function takes a Window parameter.
// For immediate cancellation, the window is:
//   Start: current_period_start
//   End:   cancellation_time (endDate)

// Proration happens naturally:
//   segDays = calendarDays(window.Start, window.End)  ← 15 days
//   periodDays = calendarDays(periodStart, periodEnd)  ← 31 days
//   prorated = baseAmount × 15/31
```

---

## 11. Quick Reference: Cancellation Effects Matrix

| Aspect | Cancel at Period End | Cancel Immediately |
|---|---|---|
| **Status change** | At `current_period_end` → `ended` | Immediately → `canceled` |
| **Usage after cancel** | ✅ Continues until period end | ❌ Stops immediately |
| **Wallet burndown** | ✅ Continues until period end | ❌ Stops immediately |
| **Base fee charged** | Full period (no proration) | **Prorated** (days used / total days) |
| **Included allowance** | Full allowance | **Prorated** `floor(allowance × days_used / total_days)` |
| **Final invoice timing** | At `current_period_end` | **Immediately** |
| **Invoice amount** | Full period total | Prorated total |
| **Auto-collection** | Normal | Runs immediately |
| **Stripe charge** | Normal full amount | Prorated amount |
| **Already-paid advance fee** | No action (period fully used) | **Credit remains** — no auto-refund |
| **Redis counters** | Continue until period end | Stop immediately |
| **Next period** | Never opens | Never opens |
| **Contract commit** | Evaluated at contract end | Evaluated at contract end |
| **Revenue recognition** | Normal period-end recog | Partial period recognition |
| **Dunning** | Normal if unpaid | Normal if unpaid |
| **Customer portal** | Shows "cancels at period end" | Shows "cancelled" |

### Common Questions

| Question | Answer |
|---|---|
| **Does the customer get a refund?** | ❌ **Not automatically.** Overpaid amount stays as account credit. Must request manual refund via support. |
| **Can the customer use the service until period end?** | ✅ With `end_of_period` — yes, full access until `current_period_end`. ❌ With `immediate` — access stops immediately. |
| **Can they reactivate?** | ✅ Only if status is `suspended`. Once `canceled` or `ended` — create a new subscription. |
| **What happens to their data?** | Events in ClickHouse are preserved (immutable). Invoices remain accessible. Customer record stays. |
| **What about pending dunning?** | If customer pays → dunning cancelled (`cancel-on-pay`). If not → dunning continues (SUSPEND, ESCALATE). |
| **What if they cancel and re-subscribe in the same month?** | Creates a new subscription with a new anniversary. Previous period's usage does NOT carry over. |
| **What if they want to change currency?** | Cancel first (end_of_period or immediate), then create new subscription with new currency. BFF shows: `"cancel and re-subscribe to bill in a new currency"` (line 755). |

---

*End of document. Generated 2026-07-17.*
