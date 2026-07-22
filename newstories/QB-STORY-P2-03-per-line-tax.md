# QuantumBilling User Story: Per-Line-Item Tax Computation

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P2-03 |
| **Sprint** | Sprint 11 |
| **Phase** | Billing Core |
| **Domain** | Tax — Line-Item Level |
| **Priority** | P2 — Medium |

---

## Title

**Per-Line-Item Tax Computation** — compute tax per line item with different rates per product type

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P2 — Medium |

---

## Description

**As a tax compliance officer**, I want **tax calculated per line item with different rates per product type** instead of a single invoice-level rate so that **QBill complies with jurisdictions requiring line-item tax breakdown.**

### Current State

`engine/internal/invoice/tax.go` has single `TaxableAmount` in `TaxRequest`, single `Rate` in `TaxResult`. Tax applied once in `generate.go`: `taxable := subtotal.Sub(applied)`.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Each line item stores its own `tax_rate`, `tax_amount`, and `tax_exempt` flag |
| AC-2 | Invoice total tax is the sum of all line-item tax amounts (not computed from invoice-level subtotal) |
| AC-3 | Tax-exempt products can have `tax_rate: 0` independently from taxable products on the same invoice |
| AC-4 | Invoice PDF and CSV show per-line tax breakdown in addition to total tax |
| AC-5 | Backward compatible — existing invoices without per-line tax use invoice-level tax computation |

---

## Code Changes Required

### Engine — Tax interface update

```go
// engine/internal/invoice/tax.go — Updated types

type TaxLine struct {
    LineID       string
    ProductType  string
    Amount       decimal.Decimal
    Quantity     decimal.Decimal
    OriginRegion string
    TaxExempt    bool
}

type TaxRequest struct {
    Lines     []TaxLine  // was: single TaxableAmount
    Customer  TaxCustomer
}

type TaxLineResult struct {
    LineID      string
    TaxableAmount decimal.Decimal
    TaxRate     decimal.Decimal
    TaxAmount   decimal.Decimal
    Jurisdiction string
}

type TaxResult struct {
    Lines []TaxLineResult  // was: single Rate + Amount
    Total TaxAmount
}
```

### Engine — Generate pipeline update

```go
// In generate.go, replace:
// taxRequest := TaxRequest{TaxableAmount: taxable, ...}
// With:
var taxLines []TaxLine
for _, line := range lines {
    taxLines = append(taxLines, TaxLine{
        LineID: line.ID, Amount: line.Amount, ProductType: line.ProductType,
    })
}
taxRequest := TaxRequest{Lines: taxLines, ...}
```

---

**Estimate:** 2 sprints
**Depends on:** QB-STORY-P0-01 (discount pipeline — adjustments affect taxable amounts)
