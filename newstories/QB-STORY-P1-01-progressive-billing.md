# QuantumBilling User Story: Progressive (Threshold) Billing

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P1-01 |
| **Sprint** | Sprint 7-8 |
| **Phase** | Billing Core |
| **Domain** | Invoice Engine — Mid-Cycle Billing |
| **Priority** | P1 — High |

---

## Title

**Progressive (Threshold) Billing** — mid-cycle invoice generation when usage crosses a configurable dollar threshold

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 — High |

---

## Description

**As a finance manager**, I want to **generate mid-cycle invoices automatically when a customer's accrued usage crosses a configurable dollar threshold** so that I can **reduce cash flow risk and get paid sooner for high-usage customers**.

### Current State

QBill only invoices at subscription period boundaries (anniversary). For a high-usage customer consuming $50K/month, the platform waits 30 days to collect — creating cash flow risk and credit exposure.

### Target State

```
Threshold: $500
Day 10: usage hits $520 → partial invoice generated (status: PARTIAL)
Day 15: usage reaches $650 → previous partial superseded, new partial generated
Day 30: end of period → final invoice nets: $800 - $650 = $150 remaining
```

---

## RBAC Roles

| Role | Can configure threshold | Can view partial invoices | Scope |
|------|------------------------|--------------------------|-------|
| **SUPER_ADMIN** | Yes (any org) | Yes (any org) | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Yes (own org) | Own org only |
| **CUSTOMER** | No | Yes (own) | Own account only |
| **END_USER** | No | No | No access |

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Admin can set a per-subscription or per-customer progressive billing threshold (e.g., $500, $1,000, or $0 for real-time billing) |
| AC-2 | When accrued usage reaches the threshold mid-period, a partial invoice is generated immediately with status `PARTIAL` |
| AC-3 | Subsequent partial invoices supersede prior ones — only the highest partial invoice's net amount is carried forward |
| AC-4 | At period end, the final invoice nets against the highest partial invoice amount |
| AC-5 | Credits and adjustments apply only at finalization, not on partial invoices |
| AC-6 | Threshold is checked on every event ingestion (for $0 threshold = real-time billing) or on a configurable evaluation interval (default: hourly) |

---

## Test Cases

### TC-01 — Threshold crossing triggers partial invoice

**Given:** Subscription `sub_001` has progressive threshold = $500; current period usage = $450
**When:** New event adds $80 in usage (total = $530)
**Then:** Partial invoice generated for $530 with status `PARTIAL`; WebSocket notification sent to frontend

### TC-02 — Final invoice nets against partials

**Given:** Period has partial invoice for $650 (highest); final usage = $800
**When:** Period-end invoice generated
**Then:** Final invoice = $800 - $650 = $150; status `FINAL`; partial invoice marked `SUPERSEDED`

### TC-03 — $0 threshold = real-time billing

**Given:** Subscription has threshold = $0
**When:** Every event is ingested
**Then:** Partial invoice generated for each event's cumulative usage; effectively real-time billing

---

## Code Changes Required

### 1. Engine — Threshold Detection

**File:** `engine/internal/billingworker/invoicerun.go`
- After generating usage aggregates, compare total against `progressive_billing_threshold`
- If threshold crossed, trigger partial invoice generation

### 2. Engine — Partial Invoice State Machine

**New:** Add `PARTIAL` and `SUPERSEDED` to invoice status enum in `types.go`

```
DRAFT ──→ PARTIAL ──→ SUPERSEDED
              │
              └──→ FINAL (at period end, nets against highest partial)
```

### 3. Prisma — Add threshold field

```prisma
model Subscription {
  // ...existing fields...
  progressiveBillingThreshold Decimal?  @map("progressive_billing_threshold") @db.Decimal(38, 9)
}
```

---

## State Machine

```
                    ┌──────────┐
                    │  DRAFT   │
                    └────┬─────┘
                         │ threshold crossed
                         ▼
                    ┌──────────┐
               ┌───▶│  PARTIAL │◀───┐
               │    └────┬─────┘    │
               │         │ new event│
               │         ▼         │
               │    ┌──────────┐    │
               └────│ SUPERSEDED│    │ (previous partial marked superseded)
                    └──────────┘    │
                                    │
                         │ period end
                         ▼
                    ┌──────────┐
                    │  FINAL   │ (nets against highest partial)
                    └──────────┘
```

---

## Estimate

**2-3 sprints** (engine 1, BFF + Prisma 1, Web UI notifications 1)

**Dependencies:** QB-STORY-P0-01 (discount pipeline) — adjustments must apply before progressive thresholds
