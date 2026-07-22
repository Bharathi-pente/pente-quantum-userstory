# QuantumBilling User Story: Partner / Reseller Billing

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P2-10 |
| **Sprint** | Sprint 14-17 |
| **Phase** | Platform |
| **Domain** | Partnerships — Reseller |
| **Priority** | P2 — Medium |

---

## Title

**Partner / Reseller Billing** — white-label billing, reseller markup, and sub-merchant-of-record support for channel partners

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P2 — Medium |

---

## Description

**As a partner manager**, I want to **support white-label billing, reseller markup, and sub-merchant-of-record flows** so that **channel partners can resell QBill-powered services under their own brand.**

### Current State

No partner/reseller entity, markup logic, or white-label code exists anywhere.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Partner/reseller entity with its own branding, pricing markup rules, and customer roster |
| AC-2 | Reseller markup can be percentage (add 15%) or fixed (add $0.005/unit) applied on top of base price |
| AC-3 | Partner customers see partner's branding on invoices and portal |
| AC-4 | Revenue split: QBill platform owner gets base price, partner gets markup |
| AC-5 | Partner has its own dashboard showing customer usage, revenue, and commission |

---

## Estimate

**3-4 sprints**
**Depends on:** QB-STORY-P0-01 (discount pipeline — markup uses similar pipeline position)
