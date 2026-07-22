# QuantumBilling User Story: Recurring / Persisting Metrics

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P1-09 |
| **Sprint** | Sprint 7-8 |
| **Phase** | Catalog |
| **Domain** | Metering — Cumulative Metrics |
| **Priority** | P1 — High |

---

## Title

**Recurring / Persisting Metrics** — meters that accumulate usage across billing periods instead of resetting

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 — High |

---

## Description

**As a product manager**, I want to **mark meters as `recurring: true`** so that **usage accumulates across billing periods** (e.g., storage GB-months, seat-months) instead of resetting at period boundaries.

### Current State

No `recurring` flag on meters in Prisma schema or engine code. All meters are period-resetting — usage resets to zero at each billing period boundary.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Meter supports `recurring: true` flag; recurring meters do not reset at period boundaries |
| AC-2 | Usage recorded in period N carries forward as starting balance for period N+1 |
| AC-3 | Invoice for recurring meter shows: opening balance, period usage, closing balance, and charge (can be on delta or absolute) |
| AC-4 | Non-recurring meters (default) continue to reset per period (backward compatible) |
| AC-5 | Enforcement checks for recurring meters consider cumulative balance across periods |

---

## Code Changes Required

### Prisma

```prisma
model Meter {
  // ...existing fields...
  recurring Boolean @default(false)
}
```

### Engine — Invoice generation

```go
// In generate.go, for recurring meters:
// Load opening balance from previous period
// Add current period usage
// Charge on delta or absolute (configurable)
recurring := meter.Recurring
if recurring {
    openingBalance := loadOpeningBalance(meterID, customerID, window.Start)
    totalUsage := openingBalance.Add(periodUsage)
    // charge = rate × totalUsage (absolute) OR rate × periodUsage (delta)
} else {
    // existing behavior: reset each period
}
```

### Counter reset

Modify counter reset logic to skip `recurring: true` meters.

---

**Estimate:** 1-2 sprints
