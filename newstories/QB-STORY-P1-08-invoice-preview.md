# QuantumBilling User Story: Invoice Preview (Upcoming Invoice)

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P1-08 |
| **Sprint** | Sprint 9-10 |
| **Phase** | Billing Core |
| **Domain** | Invoices — Customer Visibility |
| **Priority** | P1 — High |

---

## Title

**Invoice Preview (Upcoming Invoice)** — real-time preview of next invoice based on current-period usage

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 — High |

---

## Description

**As a customer**, I want to **see a real-time preview of my upcoming invoice** based on current-period usage so that I can **forecast my costs and budget accurately without waiting for period end.**

### Current State

Rate card preview exists (`POST /rate-cards/:rateCardId/preview`). Draft invoices exist (`StageDraft`). But no dedicated subscription-level preview endpoint for customers.

### Target State

API endpoint `GET /api/v1/subscriptions/:id/preview_invoice` returns estimated invoice. Uses same engine pure-function as real invoice generation — no separate calculation path.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Customer portal shows "Upcoming Invoice" section with real-time calculated total |
| AC-2 | Preview includes: base fees (prorated to date), usage charges (YTD), credits (expected), estimated tax |
| AC-3 | Preview auto-refreshes on page load and is labeled "Estimated — subject to change" |
| AC-4 | API endpoint `GET /api/v1/subscriptions/:id/preview_invoice` returns preview without persisting |
| AC-5 | Preview uses the same engine pure-function as real invoice generation — same logic, no separate calculation path |

---

## Code Changes Required

### Engine — Preview mode

**File:** `engine/internal/invoice/generate.go`
- Add `PreviewMode` parameter to `Inputs`
- In preview mode: run `StageDraft` only (no credits, no tax, no persistence)
- Return estimated totals

```go
type Inputs struct {
    // ...existing fields...
    Mode InvoiceMode  // STAGE_DRAFT | STAGE_FINAL | PREVIEW
}

func Generate(in Inputs) (Invoice, []Exception, error) {
    // ...same logic up to subtotal...
    if in.Mode == PREVIEW {
        return Invoice{Subtotal: subtotal, Lines: lines}, exceptions, nil
    }
    // ...continue with credits, tax...
}
```

### BFF — Preview endpoint

```typescript
@Get(':subscriptionId/preview')
@Roles('ORG_ADMIN', 'SUPER_ADMIN', 'CUSTOMER')
async previewInvoice(@Param('subscriptionId') id: string) {
    // Build Inputs from current subscription state + current usage
    // Call engine Generate with Mode=Preview
    // Return estimated invoice (not persisted)
}
```

---

**Estimate:** 2 sprints
