# QuantumBilling User Story: Scheduled Plan Changes

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P3-02 |
| **Sprint** | Backlog |
| **Phase** | Subscriptions |
| **Domain** | Plan Changes — Scheduling |
| **Priority** | P3 — Low |

---

## Title

**Scheduled Plan Changes** — schedule an upgrade/downgrade for a future date

---

## Description

**As a customer**, I want to **schedule a plan change for a future date** (e.g., "upgrade to Pro plan on September 1st") so that I can **align billing changes with my budget cycle.**

### Current State

`changeSubscriptionPlan()` supports `immediate` or `next_period` but not arbitrary future dates.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Customer can schedule a plan change for any future date |
| AC-2 | Scheduled changes are visible in the portal with countdown |
| AC-3 | Customer can cancel a scheduled change before it takes effect |
| AC-4 | At the scheduled date, the change executes automatically with correct proration |

---

**Estimate:** 1 sprint
