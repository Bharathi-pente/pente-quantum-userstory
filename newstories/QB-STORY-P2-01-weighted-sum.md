# QuantumBilling User Story: WEIGHTED SUM Aggregation

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P2-01 |
| **Sprint** | Sprint 11-12 |
| **Phase** | Catalog |
| **Domain** | Metering — Aggregation |
| **Priority** | P2 — Medium |

---

## Title

**WEIGHTED SUM Aggregation** — time-proportional sum for capacity billing

---

## Description

**As a product manager**, I want a **WEIGHTED SUM aggregation that computes time-proportional sums** (e.g., average GB-months of storage) so that I can **bill for capacity-based metrics fairly.**

### Current State

Prisma `MeterAggregation` enum: `SUM`, `COUNT`, `AVG`, `GAUGE`. No `WEIGHTED_SUM` type.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | New aggregation type `WEIGHTED_SUM` available on meter creation |
| AC-2 | Calculation: `sum(event_value × duration_weight)` where `duration_weight = event_duration / period_duration` |
| AC-3 | Works with all 7 pricing models |
| AC-4 | Supports groupBy filters for dimension-based weighted aggregation |

---

## Code Changes Required

### Prisma

```prisma
enum MeterAggregation {
  SUM
  COUNT
  AVG
  GAUGE
  WEIGHTED_SUM  // NEW
}
```

### Engine — Weighted aggregation logic

```go
func weightedSumAggregation(events []UsageEvent, periodDuration time.Duration) decimal.Decimal {
    var total decimal.Decimal
    for _, event := range events {
        weight := decimal.NewFromFloat(event.Duration.Seconds() / periodDuration.Seconds())
        total = total.Add(event.Value.Mul(weight))
    }
    return total
}
```

---

**Estimate:** 1-2 sprints
**Depends on:** QB-STORY-P0-02 (groupBy)
