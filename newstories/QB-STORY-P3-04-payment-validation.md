# QuantumBilling User Story: Payment Method Auto-Validation

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P3-04 |
| **Sprint** | Backlog |
| **Phase** | Billing Foundation |
| **Domain** | Payments — Validation |
| **Priority** | P3 — Low |

---

## Title

**Payment Method Auto-Validation** — validate payment method immediately when added, not at first charge

---

## Description

**As a customer**, I want **my payment method validated immediately when I add it** so that **I know it's correct before my invoice is due.**

### Current State

No immediate validation on add. Payment methods are only validated at first charge attempt.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | When a card is added, a $0 or $1 auth hold is placed and immediately released via Stripe |
| AC-2 | Card is marked `verified` if auth succeeds, `failed` if declined |
| AC-3 | ACH/bank accounts use micro-deposit verification |
| AC-4 | Expired cards are flagged with warning status immediately |

---

**Estimate:** 1 sprint
