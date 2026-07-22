# QuantumBilling User Story: Sandbox / Test Environment

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P3-01 |
| **Sprint** | Backlog |
| **Phase** | Platform |
| **Domain** | Developer Experience |
| **Priority** | P3 — Low |

---

## Title

**Sandbox / Test Environment** — isolated environment where test events don't affect real billing

---

## Description

**As a developer**, I want a **sandbox environment where test events don't affect real billing** so that I can **integrate and test without risk.**

### Current State

Test clocks exist (D-09) for time manipulation but no fully isolated sandbox environment. Developers currently integrate against production or staging with real data risk.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Sandbox orgs are marked with `is_sandbox = true` and never feed production billing |
| AC-2 | Sandbox events are stored in separate ClickHouse tables or filtered by org flag |
| AC-3 | Sandbox invoices are generated but marked `is_sandbox` and never sent to collection |
| AC-4 | Sandbox has reset capability — wipe all data and start fresh |
| AC-5 | Test clocks work in sandbox to fast-forward time for invoice testing |

---

## Estimate

**2 sprints**
