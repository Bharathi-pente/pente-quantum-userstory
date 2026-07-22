# QuantumBilling User Story: Multi-Currency Support

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P2-02 |
| **Sprint** | Sprint 11-12 |
| **Phase** | Billing Core |
| **Domain** | Currency — Multi-Currency |
| **Priority** | P2 — Medium |

---

## Title

**Multi-Currency Support** — allow subscriptions in different currencies (USD, EUR, JPY) with display conversion

---

## Description

**As a global customer**, I want to **have subscriptions in different currencies** (USD, EUR, JPY) so that **my billing matches my local operating currency.**

### Current State

X-1 guardrails exist: currency validation in `generate.go` (line 37), consolidation guard in `grouping.go` (line 74), payment guard in `collection/manual.go`. `MinorUnits()` supports USD (2) and zero-decimal (JPY, KRW). But no FX support, no multi-currency display conversion, no per-line-item currency.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | System supports all ISO 4217 currencies with configurable minor unit precision |
| AC-2 | Customer can have subscriptions in multiple currencies (relaxing X-1 one-currency-per-customer rule) |
| AC-3 | Each subscription/plan has a fixed currency — no FX conversion in engine (X-2 preserved) |
| AC-4 | Invoice shows the currency symbol and ISO code on every monetary field |
| AC-5 | Billing groups enforce currency uniformity (all subscriptions in a group must share the same currency) |

---

## Code Changes Required

### Engine — Relax X-1 to per-subscription

```go
// generate.go — Change from one-currency-per-customer to one-currency-per-invoice
// Remove the cross-plan currency check (line 37-44)
// Keep per-invoice currency consistency
```

### BFF — Currency assignment on subscription create

```typescript
@Post()
async create(@Body() dto: CreateSubscriptionDto) {
    // Inherit currency from plan
    // Allow override if plan supports multiple currencies
}
```

---

**Estimate:** 2 sprints
