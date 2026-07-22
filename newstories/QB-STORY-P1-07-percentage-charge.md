# QuantumBilling User Story: Percentage Charge Model

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P1-07 |
| **Sprint** | Sprint 7 |
| **Phase** | Pricing |
| **Domain** | Pricing Models |
| **Priority** | P1 — High |

---

## Title

**Percentage Charge Model** — pricing model where charge = (amount × rate) + fixed_fee for revenue share and marketplace pricing

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 — High |

---

## Description

**As a product manager**, I want a **PERCENTAGE pricing model where charge = (amount × rate) + fixed_fee** so that I can **price payment processing fees, revenue share arrangements, and marketplace commissions.**

### Current State

`PricingType` enum has 7 values: `FLAT`, `PER_UNIT`, `TIERED_GRADUATED`, `TIERED_VOLUME`, `PACKAGE`, `MATRIX`, `COST_PLUS`. No `PERCENTAGE` type. `evaluatePricingModel()` switch has no percentage case. Lago has this.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Admin can select `PERCENTAGE` pricing model on any rate card entry or plan price |
| AC-2 | Configuration includes `percentage_rate` (e.g., 2.9%), `fixed_fee` (e.g., $0.30), optional `cap_amount` and `min_amount` |
| AC-3 | Charge calculation: `max(min_amount, min(amount × percentage_rate + fixed_fee, cap_amount))` |
| AC-4 | Works with both product-led (plan) and sales-led (rate card) pricing paths |
| AC-5 | Percentage rate uses 4 decimal precision (e.g., 2.9000%) |

---

## Code Changes Required

### 1. Engine — Add PERCENTAGE pricing type

```go
// engine/internal/rating/types.go
const PricingPercentage PricingType = "PERCENTAGE"

// engine/internal/rating/resolver.go
func evaluatePercentage(rate Rate, quantity decimal.Decimal) decimal.Decimal {
    percentageRate := rate.Config.PercentageRate      // e.g., 0.029 (2.9%)
    fixedFee := rate.Config.FixedFee                  // e.g., 0.30
    charge := quantity.Mul(percentageRate).Add(fixedFee)
    if rate.Config.MinAmount != nil {
        charge = decimal.Max(charge, *rate.Config.MinAmount)
    }
    if rate.Config.MaxAmount != nil {
        charge = decimal.Min(charge, *rate.Config.MaxAmount)
    }
    return charge
}
```

### 2. Prisma — Add percentage fields

```prisma
model PricingModel {
  // ...existing fields...
  percentageRate Decimal? @map("percentage_rate") @db.Decimal(12, 4)
  fixedFee       Decimal? @map("fixed_fee") @db.Decimal(38, 9)
}
```

---

**Estimate:** 1 sprint
