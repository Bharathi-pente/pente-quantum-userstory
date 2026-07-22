# QuantumBilling User Story: Slot-Based Pricing (Seats)

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P2-09 |
| **Sprint** | Sprint 12-13 |
| **Phase** | Pricing |
| **Domain** | Pricing Models — Seats |
| **Priority** | P2 — Medium |

---

## Title

**Slot-Based Pricing (Seats)** — per-seat pricing with mid-period add/remove proration

---

## Description

**As a product manager**, I want a **dedicated slot/seat pricing model** where customers pay per-seat-per-month with **mid-period add/remove proration** so that I can **bill for team plans, user licenses, and device entitlements.**

### Current State

`SEAT` line type exists (`LineType = "SEAT"` in `types.go`). `Generate()` processes seat lines with proration. But no dedicated slot/seat pricing model — seats are a line type, not a standalone pricing model with add/remove rules.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Plan supports `SEAT` pricing model with per-seat price and min/max seat counts |
| AC-2 | Customer can add or remove seats mid-period with daily proration |
| AC-3 | Adding a seat mid-period: charge = `seat_count × per_seat_price × (remaining_days / period_days)` |
| AC-4 | Removing a seat mid-period: credit note for unused portion, seat removed immediately |
| AC-5 | Minimum seat count enforced at subscription level (admin-configurable) |

---

**Estimate:** 2 sprints
