# QuantumBilling User Story: Tax Provider Integration

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P2-05 |
| **Sprint** | Sprint 13-15 |
| **Phase** | Billing Core |
| **Domain** | Tax — Provider Integration |
| **Priority** | P2 — Medium |

---

## Title

**Tax Provider Integration (CR-7)** — connect Avalara, Anrok, or Stripe Tax for automated tax calculation

---

## Description

**As a tax compliance officer**, I want to **connect Avalara, Anrok, or Stripe Tax as my tax calculation provider** so that **QBill calculates tax correctly for all jurisdictions where my customers operate.**

### Current State

`TaxProvider` interface exists with `Name()` and `Calculate(TaxRequest)` methods. `InternalTaxProvider` resolves from `billing.tax_regions`. Prisma `TaxProvider` enum: `avalara`, `anrok`, `stripe_tax`, `internal`. But no Avalara/Anrok/Stripe Tax implementations — code comment: "External providers implement this interface in a later unit."

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Pluggable tax provider interface: `TaxProvider { calculate(invoice_lines, customer_address): TaxResult }` |
| AC-2 | Built-in providers: Avalara (AvaTax API), Anrok API, Stripe Tax API |
| AC-3 | Admin can configure which provider to use per organization |
| AC-4 | Tax calculation happens at invoice finalization with provider response cached for 24 hours |
| AC-5 | If provider is unreachable, finalization uses cached rates or falls back to configured default rate with alert |

---

## Code Changes Required

### New implementations for TaxProvider interface

```go
// engine/internal/invoice/tax_avalara.go
type AvalaraTaxProvider struct {
    client *avalara.Client
}
func (p *AvalaraTaxProvider) Calculate(ctx context.Context, req TaxRequest) (*TaxResult, error) {
    // Call AvaTax API with line items and customer address
    // Return per-line tax breakdown
}

// engine/internal/invoice/tax_stripe.go
type StripeTaxProvider struct {
    client *stripe.Client
}
func (p *StripeTaxProvider) Calculate(ctx context.Context, req TaxRequest) (*TaxResult, error) {
    // Call Stripe Tax API
}

// engine/internal/invoice/tax_anrok.go
type AnrokTaxProvider struct {
    client *anrok.Client
}
```

---

**Estimate:** 2-3 sprints
**Depends on:** QB-STORY-P2-03 (per-line-item tax)
