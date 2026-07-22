# QuantumBilling User Story: Credit Note Self-Service

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P3-06 |
| **Sprint** | Backlog |
| **Phase** | Billing Core |
| **Domain** | Corrections — Self-Service |
| **Priority** | P3 — Low |

---

## Title

**Credit Note Self-Service** — request a credit note from the portal with reason selector

---

## Description

**As a customer**, I want to **request a credit note from the portal** with a reason selector so that **I can initiate the correction process without contacting support.**

### Current State

Credit notes exist (CR-4 implemented in engine) but creation is admin-only. No customer-facing request flow.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Customer can request a credit note from the invoice detail page |
| AC-2 | Reason selector: "Service credit", "Billing error", "Cancellation refund", "Other" |
| AC-3 | Request creates a `credit_note_request` with status `PENDING_REVIEW` |
| AC-4 | Admin dashboard shows pending requests for review and approval |
| AC-5 | Approved requests trigger the existing credit note engine (CR-4) |

---

## State Machine

```
REQUESTED (customer submits)
  │
  ├── Admin approves → CREDIT_NOTE_ISSUED (existing CR-4 engine)
  │
  └── Admin rejects → REJECTED (with reason to customer)
```

---

**Estimate:** 1 sprint
