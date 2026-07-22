# QuantumBilling User Story: Grant Rollover Configurability

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P1-10 |
| **Sprint** | Sprint 7-8 |
| **Phase** | Wallet |
| **Domain** | Credits — Usage Grants |
| **Priority** | P1 — High |

---

## Title

**Grant Rollover Configurability** — per-grant configurable min/max rollover for recurring usage grants

---

## Description

**As a product manager**, I want to **configure per-grant rollover rules** (`minRolloverAmount`, `maxRolloverAmount`) so that **unused minutes or tokens roll over to the next period up to a configurable cap.**

### Current State

`engine/internal/invoice/credits.go` has FEFO sort/apply with priority→expiry→creation ordering. No `minRolloverAmount`/`maxRolloverAmount`. Credits expire but unspent balance implicitly carries forward — no configurable rollover.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Plan recurring grant configuration supports `min_rollover_amount` and `max_rollover_amount` |
| AC-2 | At period end, unused grant balance rolls over according to: `carry_forward = clamp(remaining, min_rollover, max_rollover)` |
| AC-3 | Default behavior (no config) = no rollover (backward compatible with current TR-2) |
| AC-4 | Rolled-over balance is tracked separately from new period grant for audit purposes |
| AC-5 | UI shows beginning balance, new grant, consumed, rolled-over, and ending balance per period |

---

## Code Changes

```go
// In credits.go — add rollover fields
type RecurringGrantConfig struct {
    Amount           Decimal
    MinRolloverAmount Decimal // default 0 = no rollover below this
    MaxRolloverAmount Decimal // default 0 = no rollover cap
}

// At period boundary:
func calculateRollover(remaining, minRollover, maxRollover Decimal) Decimal {
    if minRollover.IsZero() && maxRollover.IsZero() {
        return DecimalZero // no rollover (default)
    }
    carry := decimal.Max(remaining, minRollover)
    if !maxRollover.IsZero() {
        carry = decimal.Min(carry, maxRollover)
    }
    return carry
}
```

---

**Estimate:** 1-2 sprints
