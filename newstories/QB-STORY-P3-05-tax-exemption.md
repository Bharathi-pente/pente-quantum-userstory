# QuantumBilling User Story: Tax Exemption Certificate Management

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P3-05 |
| **Sprint** | Backlog |
| **Phase** | Tax |
| **Domain** | Compliance — Tax Exemption |
| **Priority** | P3 — Low |

---

## Title

**Tax Exemption Certificate Management** — upload tax exemption certificates and apply to invoices

---

## Description

**As a customer**, I want to **upload my tax exemption certificate** so that **I'm not charged sales tax when exempt.**

### Current State

No certificate upload or management functionality exists.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Customer can upload tax exemption certificate (PDF/image) from the portal |
| AC-2 | Admin can review, approve, or reject certificates |
| AC-3 | Approved certificates are applied to all future invoices — tax_rate = 0 for exempt products |
| AC-4 | Certificate expiration is tracked; customer notified before expiry |

---

**Estimate:** 1-2 sprints
