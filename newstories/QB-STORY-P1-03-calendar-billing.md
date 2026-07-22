# QuantumBilling User Story: Calendar Billing Option

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P1-03 |
| **Sprint** | Sprint 7-8 |
| **Phase** | Subscriptions |
| **Domain** | Billing Period — Calendar Alignment |
| **Priority** | P1 — High |

---

## Title

**Calendar Billing Option** — allow subscriptions to bill on calendar-month boundaries (1st to last day of month) instead of anniversary dates

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 — High |

---

## Description

**As a customer**, I want my **subscription to bill on calendar-month boundaries (1st to last day of month)** instead of anniversary dates so that **my invoices align with my accounting calendar and are easier to reconcile.**

### Current State

`engine/internal/invoice/periods.go` implements **anniversary-only** anchoring (T-3). Prisma `BillingPeriod` enum: `MONTHLY | QUARTERLY | YEARLY` — no `billing_period_type` or `CALENDAR`/`ANNIVERSARY` field. All subscriptions anchor to their start date.

### Target State

Add `billing_period_type: ANNIVERSARY | CALENDAR` to plan/subscription. Calendar type auto-aligns to month boundaries. A subscription created on March 15 produces:
- March 15–March 31 (prorated 17 days)
- April 1–April 30 (full month)
- May 1–May 31 (full month)

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Plan/subscription supports `billing_period_type: ANNIVERSARY | CALENDAR` |
| AC-2 | Calendar subscriptions auto-align to month boundaries — March 15 start → Mar 15–Mar 31 (prorated), then Apr 1–Apr 30 |
| AC-3 | Calendar monthly periods handle February correctly (28/29 days) and 31-day months |
| AC-4 | Mid-cycle changes on calendar subscriptions prorate to the next calendar boundary, not anniversary |
| AC-5 | No changes to existing anniversary subscriptions — backward compatible |

---

## Code Changes Required

### 1. Prisma — Add enum field

```prisma
enum BillingPeriodType {
  ANNIVERSARY
  CALENDAR
}

model Plan {
  // ...existing fields...
  billingPeriodType BillingPeriodType @default(ANNIVERSARY) @map("billing_period_type")
}

model Subscription {
  // ...existing fields...
  billingPeriodType BillingPeriodType? @map("billing_period_type") // overrides plan if set
}
```

### 2. Engine — Calendar period calculation

**File:** `engine/internal/invoice/periods.go`

Add `CalendarBoundary()` function:
```go
func CalendarBoundary(start, end time.Time) (Window, error) {
    // First period: start_date → end of month (prorated)
    // Subsequent: full calendar months
    if isFirstPeriod {
        periodEnd = endOfMonth(start)
    } else {
        periodStart = startOfMonth(start)
        periodEnd = endOfMonth(start)
    }
}
```

Modify `Boundary()` and `PeriodWindows()` to accept `billingPeriodType` parameter.

### 3. Engine — Proration for calendar alignment

**File:** `engine/internal/invoice/generate.go`

First calendar period uses `calendarDays(start, endOfMonth)` instead of full month days for proration factors (P-2).

---

**Estimate:** 1-2 sprints
**Dependencies:** None
