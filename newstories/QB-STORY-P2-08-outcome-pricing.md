# QuantumBilling User Story: Outcome-Action Pricing

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P2-08 |
| **Sprint** | Sprint 13-14 |
| **Phase** | Pricing |
| **Domain** | AI — Outcome-Based Pricing |
| **Priority** | P2 — Medium |

---

## Title

**Outcome-Action Pricing (CR-10)** — price based on AI agent outcomes/actions rather than raw token counts

---

## Description

**As a product manager**, I want to **price based on AI agent outcomes/actions** (e.g., "per successful API call", "per document processed") rather than raw token counts so that I can **offer value-based pricing to my AI customers.**

### Current State

No outcome/action pricing logic exists anywhere. **No competitor has this feature either** — it's a differentiation opportunity.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Support outcome-based event schema with `action` and `outcome` fields alongside raw usage |
| AC-2 | Pricing can be configured per outcome type (e.g., "document_classified: $0.01", "image_generated: $0.05") |
| AC-3 | A single event can trigger both outcome pricing AND raw usage pricing (e.g., per-call fee + per-token compute) |
| AC-4 | GroupBy filters work on outcome/action fields |

---

**Estimate:** 2 sprints
**Depends on:** QB-STORY-P0-02 (groupBy)
