# QuantumBilling User Story: Plan Phases

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P1-04 |
| **Sprint** | Sprint 8-9 |
| **Phase** | Catalog |
| **Domain** | Plans — Lifecycle |
| **Priority** | P1 — High |

---

## Title

**Plan Phases** — multi-phase plans that automatically transition (trial → ramp-up → evergreen)

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 — High |

---

## Description

**As a product manager**, I want to **define multi-phase plans that automatically transition** (e.g., 14-day free trial → 3-month ramp-up at 50% discount → evergreen pricing) so that I can **automate the SaaS customer lifecycle without manual intervention.**

### Current State

Trials exist — `SubscriptionInput.TrialEnd`, `subWindows()` splits at trial end, `InTrial` suppresses base fee. But no `plan_phase` enum, no RAMP_UP/EVERGREEN phases, no automatic phase transitions.

### Target State

Phase types: `TRIAL` (free, zero-rated), `RAMP_UP` (discount applied), `EVERGREEN` (full price, infinite duration). System auto-transitions at phase end.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Plan supports multiple ordered phases with configurable duration and pricing override per phase |
| AC-2 | Phase types: `TRIAL` (free, zero-rated), `RAMP_UP` (discount applied), `EVERGREEN` (full price, infinite duration) |
| AC-3 | System automatically transitions to the next phase at the end of the current phase duration |
| AC-4 | Invoice for a period spanning multiple phases prorates correctly by day within each phase |
| AC-5 | Early termination during trial produces no charge; early termination during ramp-up calculates at the ramp-up rate |
| AC-6 | Admin can preview when a subscription will transition to the next phase in the UI |

---

## Code Changes Required

### Prisma

```prisma
enum PlanPhaseType {
  TRIAL
  RAMP_UP
  EVERGREEN
}

model PlanPhase {
  id          String        @id @default(uuid()) @db.Uuid
  planId      String        @map("plan_id") @db.Uuid
  phaseType   PlanPhaseType @map("phase_type")
  duration    Int?          // days, null for EVERGREEN
  sequence    Int           // ordering
  priceOverride Decimal?    @db.Decimal(38, 9) @map("price_override") // null = use plan base
  createdAt   DateTime      @default(now()) @map("created_at")
  
  plan Plan @relation(fields: [planId], references: [id])
  @@index([planId])
  @@map("plan_phases")
  @@schema("catalog")
}
```

### Engine — Phase-aware proration

**File:** `engine/internal/invoice/periods.go`
- Extend `subWindows()` to split at phase boundaries (not just trial end)
- Add `PlanPhase` to `subWindow` struct

**Estimate:** 2-3 sprints
