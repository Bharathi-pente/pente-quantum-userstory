# QuantumBilling User Story: Discount & Commitment Pipeline [CODE-VERIFIED]

> **This story has been verified against the actual codebase.** The code snippets below match real function signatures, struct definitions, and pipeline positions in `engine/internal/invoice/generate.go` and `types.go`.

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P0-01 |
| **Sprint** | Sprint 3-6 |
| **Phase** | Billing Core |
| **Domain** | Invoice Engine — Adjustments |
| **Priority** | P0 — Critical |

---

## Title

**Discount & Commitment Pipeline** — add 5 adjustment types (usage discount, amount discount, percentage discount, minimum commitment, maximum spend cap) between subtotal and credit application in the invoice engine

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P0 — Critical |

---

## Description

**As a product manager**, I want to configure discounts and commitments on subscriptions so that I can run promotional pricing ("20% off first year"), enterprise contracts ("$5K/month minimum"), and volume caps ("never more than $10K/month") without engineering involvement.

### Current Code — Exact Pipeline

In `engine/internal/invoice/generate.go`, the `Generate()` function runs this pipeline:

```
1. subWindows() — split window at plan changes + trial end
2. For each segment: base fee (P-2) → seats (P-4) → rateSegmentUsage() (usage + allowance P-3)
3. Commit true-up (C-3) or commit progress (C-2)
4. Adjustment lines appended (CURRENT — no pipeline logic):
   for _, adj := range in.Adjustments {
       lines = append(lines, Line{LineType: LineAdjustment, ...})
   }
5. Sum subtotal
6. IF StageFinal: ApplyCreditsFEFO() → tax → total
```

The current `Adjustment` struct in `engine/internal/invoice/types.go`:
```go
type Adjustment struct {
    Description string
    Amount      decimal.Decimal
}
```

**Problem:** This is just a passive line item. No discount type classification, no pipeline ordering, no proportional distribution, no period-based expiry, no min/max enforcement.

### Target Pipeline

```
1. Subtotal (existing)
2. ADJUSTMENTS (NEW):
   a. applyUsageDiscounts()    → reduce quantity before pricing
   b. applyAmountDiscounts()   → fixed $ off, proportionally distributed
   c. applyPercentageDiscounts() → % off each line
   d. applyMinCommitment()     → floor: max(total, min_amount)
   e. applyMaxCap()            → ceiling: min(total, max_amount)
3. Adjusted subtotal (Invoice.AdjustedSubtotal — NEW)
4. IF StageFinal: ApplyCreditsFEFO() → tax → total
```

---

## RBAC Roles

| Role | Can CRUD adjustments | Can view adjustments | Scope |
|------|--------------------|--------------------|-------|
| **SUPER_ADMIN** | Yes (any org) | Yes (any org) | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Yes (own org) | Own org only |
| **CUSTOMER** | No | Yes (own) | Own account only |
| **END_USER** | No | No | No access |

---

## PRECISE Code Changes Required

### Change 1: New types and structs

**File:** `engine/internal/invoice/types.go`

```go
// NEW — Add after existing LineType constants
type AdjustmentType string

const (
    AdjustmentUsageDiscount    AdjustmentType = "USAGE_DISCOUNT"
    AdjustmentAmountDiscount   AdjustmentType = "AMOUNT_DISCOUNT"
    AdjustmentPercentageDisc   AdjustmentType = "PERCENTAGE_DISCOUNT"
    AdjustmentMinCommitment    AdjustmentType = "MINIMUM_COMMITMENT"
    AdjustmentMaxCap           AdjustmentType = "MAXIMUM_CAP"
)
```

**Replace** the existing `Adjustment` struct (currently has only `Description` and `Amount`):
```go
// REPLACEMENT for existing Adjustment struct
type Adjustment struct {
    ID                string
    SubscriptionID    string
    AdjustmentType    AdjustmentType
    Amount            decimal.Decimal   // for AMOUNT_DISCOUNT / MIN / MAX
    PercentageRate    decimal.Decimal   // for PERCENTAGE_DISCOUNT (20.0 = 20%)
    Description       string
    ExpiresAfterPeriods int             // 0 = never expires
    PeriodCount       int               // incremented each billing cycle
}
```

**Add** to `Invoice` struct:
```go
type Invoice struct {
    // ...existing fields (ID, OrgID, CustomerID, etc.)...
    Subtotal           decimal.Decimal  // exists — sum of all lines
    AdjustedSubtotal   decimal.Decimal  // NEW — after adjustments, before credits
    CreditsApplied     decimal.Decimal  // exists
    TaxRate            decimal.Decimal  // exists
    TaxAmount          decimal.Decimal  // exists
    Total              decimal.Decimal  // exists
    // ...
}
```

**Add** `PeriodCount` to `Inputs` struct:
```go
type Inputs struct {
    // ...existing fields...
    Adjustments []Adjustment   // exists
    PeriodCount int            // NEW — tracks which billing period we're on
    // ...
}
```

### Change 2: New adjustment pipeline file

**New file:** `engine/internal/invoice/adjustments.go`

```go
package invoice

import (
    "fmt"
    "github.com/shopspring/decimal"
)

const internalScale = 9

// applyAdjustments runs the 5-step adjustment pipeline in ORDER.
// Returns modified lines and the adjusted subtotal.
func applyAdjustments(lines []Line, adjustments []Adjustment, periodCount int, minor int32) ([]Line, decimal.Decimal) {
    active := filterActiveAdjustments(adjustments, periodCount)

    // Step 1: Usage discounts — reduce quantity before pricing
    lines = applyUsageDiscounts(lines, active)

    // Step 2: Amount discounts — fixed $ off, distributed proportionally
    lines = applyAmountDiscounts(lines, active)

    // Step 3: Percentage discounts — % off each line
    for _, adj := range active {
        if adj.AdjustmentType == AdjustmentPercentageDisc {
            rate := adj.PercentageRate.Div(decimal.NewFromInt(100)) // 20 → 0.20
            for i, line := range lines {
                discount := line.Amount.Mul(rate)
                lines[i].Amount = RoundHalfUp(line.Amount.Sub(discount), minor)
            }
        }
    }

    // Step 4: Sum adjusted lines
    adjustedSubtotal := decimal.Zero
    for _, line := range lines {
        adjustedSubtotal = adjustedSubtotal.Add(line.Amount)
    }

    // Step 5: Minimum commitment — floor
    for _, adj := range active {
        if adj.AdjustmentType == AdjustmentMinCommitment {
            minAmount := RoundHalfUp(adj.Amount, minor)
            if adjustedSubtotal.LessThan(minAmount) {
                adjustedSubtotal = minAmount
            }
        }
    }

    // Step 6: Maximum cap — ceiling
    for _, adj := range active {
        if adj.AdjustmentType == AdjustmentMaxCap {
            maxAmount := RoundHalfUp(adj.Amount, minor)
            if adjustedSubtotal.GreaterThan(maxAmount) {
                adjustedSubtotal = maxAmount
            }
        }
    }

    return lines, adjustedSubtotal
}

func filterActiveAdjustments(adjustments []Adjustment, periodCount int) []Adjustment {
    var active []Adjustment
    for _, adj := range adjustments {
        if adj.ExpiresAfterPeriods > 0 && periodCount >= adj.ExpiresAfterPeriods {
            continue
        }
        active = append(active, adj)
    }
    return active
}

// applyUsageDiscounts reduces quantity: effective_qty = max(0, qty - discount_qty)
func applyUsageDiscounts(lines []Line, adjustments []Adjustment) []Line {
    for _, adj := range adjustments {
        if adj.AdjustmentType != AdjustmentUsageDiscount {
            continue
        }
        for i, line := range lines {
            if line.Quantity.GreaterThan(decimal.Zero) && line.LineType == LineUsage {
                newQty := decimal.Max(decimal.Zero, line.Quantity.Sub(adj.Amount))
                if !newQty.Equals(line.Quantity) {
                    lines[i].Quantity = newQty
                    lines[i].Amount = RoundHalfUp(line.UnitPrice.Mul(newQty), 2)
                }
            }
        }
    }
    return lines
}

// applyAmountDiscounts subtracts fixed $ proportionally across lines.
// E.g., $100 off on lines $600 + $400 → line1: -$60, line2: -$40
func applyAmountDiscounts(lines []Line, adjustments []Adjustment) []Line {
    for _, adj := range adjustments {
        if adj.AdjustmentType != AdjustmentAmountDiscount {
            continue
        }
        subtotal := decimal.Zero
        for _, line := range lines {
            subtotal = subtotal.Add(line.Amount)
        }
        if subtotal.IsZero() {
            continue
        }
        remaining := adj.Amount
        for i := range lines {
            if remaining.LessThanOrEqual(decimal.Zero) {
                break
            }
            share := adj.Amount.Mul(lines[i].Amount).DivRound(subtotal, internalScale)
            if share.GreaterThan(remaining) {
                share = remaining
            }
            lines[i].Amount = RoundHalfUp(lines[i].Amount.Sub(share), 2)
            remaining = remaining.Sub(share)
        }
    }
    return lines
}
```

### Change 3: Modify Generate() pipeline

**File:** `engine/internal/invoice/generate.go`

**Replace** the current adjustment loop (around line 175, the `for _, adj := range in.Adjustments` block) and subtotal calculation:

```go
// REMOVE these lines (~174-190):
// for _, adj := range in.Adjustments {
//     lines = append(lines, Line{
//         LineType: LineAdjustment,
//         Description: adj.Description,
//         ...
//     })
// }
// subtotal := decimal.Zero
// for _, line := range lines { subtotal = subtotal.Add(line.Amount) }

// REPLACE WITH:
// Step 2: Apply adjustment pipeline (NEW)
lines, subtotal = applyAdjustments(lines, in.Adjustments, in.PeriodCount, minor)

// Store adjusted subtotal on invoice
inv.AdjustedSubtotal = subtotal
```

**Modify** the tax calculation (around line 210):
```go
// CHANGE FROM:
// taxable := subtotal.Sub(applied)
// CHANGE TO:
taxable := inv.AdjustedSubtotal.Sub(applied)
```

### Change 4: Prisma Migration

```prisma
enum AdjustmentType {
  USAGE_DISCOUNT
  AMOUNT_DISCOUNT
  PERCENTAGE_DISCOUNT
  MINIMUM_COMMITMENT
  MAXIMUM_CAP
}

model Adjustment {
  id              String          @id @default(uuid()) @db.Uuid
  subscriptionId  String          @map("subscription_id") @db.Uuid
  orgId           String          @map("org_id") @db.Uuid
  adjustmentType  AdjustmentType  @map("adjustment_type")
  amount          Decimal?        @map("amount") @db.Decimal(38, 9)
  percentageRate  Decimal?        @map("percentage_rate") @db.Decimal(12, 4)
  description     String
  expiresAfterPeriods Int?        @map("expires_after_periods")
  periodCount     Int?            @map("period_count")
  createdAt       DateTime        @default(now()) @map("created_at")
  updatedAt       DateTime        @updatedAt @map("updated_at")
  subscription    Subscription    @relation(fields: [subscriptionId], references: [id])
  organization    Organization    @relation(fields: [orgId], references: [id])
  @@index([subscriptionId])
  @@index([orgId])
  @@map("adjustments")
  @@schema("billing")
}
```

### Change 5: BFF Module

**New:** `control-plane/src/billing/adjustments/adjustments.controller.ts`
```typescript
@Controller('subscriptions/:subscriptionId/adjustments')
export class AdjustmentsController {
    constructor(private readonly adjustmentsService: AdjustmentsService) {}

    @Post()
    @Roles('ORG_ADMIN', 'SUPER_ADMIN')
    async create(@Param('subscriptionId') subId: string, @Body() dto: CreateAdjustmentDto) {
        return this.adjustmentsService.create(subId, dto);
    }

    @Get()
    @Roles('ORG_ADMIN', 'SUPER_ADMIN', 'CUSTOMER')
    async list(@Param('subscriptionId') subId: string) {
        return this.adjustmentsService.list(subId);
    }

    @Delete(':id')
    @Roles('ORG_ADMIN', 'SUPER_ADMIN')
    async remove(@Param('id') id: string) {
        return this.adjustmentsService.remove(id);
    }
}
```

### Change 6: Web UI

**New:** `web/app/subscriptions/[id]/adjustments/`
- Table with columns: Type (badge), Amount/Rate, Description, Periods Remaining, Actions
- "Add Adjustment" button → modal with type selector → dynamic form fields
- Type selector determines visible fields:
  - USAGE_DISCOUNT: quantity field
  - AMOUNT_DISCOUNT: dollar amount field
  - PERCENTAGE_DISCOUNT: percentage field (0-100)
  - MINIMUM_COMMITMENT: dollar amount field
  - MAXIMUM_CAP: dollar amount field

---

## Acceptance Criteria

| # | Criterion | How to Verify |
|---|-----------|---------------|
| AC-1 | `POST /subscriptions/:id/adjustments` creates all 5 types | Call API with each type, verify 201 + DB row |
| AC-2 | USAGE_DISCOUNT reduces qty: line with qty=5000, disc=1000 → new qty=4000 | Inject Adjustment{Type:UsageDiscount, Amount:1000} into Generate(), verify Line.Quantity=4000 |
| AC-3 | PERCENTAGE_DISCOUNT: 20% off $100 → $80.00 | Generate() with PercentageRate=20 on $1000 base → assert total=$800 |
| AC-4 | AMOUNT_DISCOUNT: $100 off across $600+$400 → -$60 and -$40 | Generate() with Amount=$100 on 2 lines → verify each line's reduction |
| AC-5 | MINIMUM_COMMITMENT: $400 calculated → $500 floor | Generate() with MinCommit=$500 on $400 subtotal → assert total=$500 |
| AC-6 | MAXIMUM_CAP: $12K calculated → $10K cap | Generate() with MaxCap=$10000 on $12000 subtotal → assert total=$10000 |
| AC-7 | Pipeline order: USAGE → AMOUNT → PERCENTAGE → MIN → MAX | Test with all 5 types active, verify order matches |
| AC-8 | Expiry: expires_after=3 stops at period 4 | Create adj with expires=3, run Generate 4 times with increasing periodCount → assert adj ignored on 4th |
| AC-9 | Min before credits: $500 min + $200 credits → charge $300 | Generate with MinCommit=$500 and Credits available → verify CreditsApplied=$200, Total=$300 |

---

## Test Cases

### TC-01 — Percentage discount on flat fee subscription

**Given:** Subscription has $1,000/mo base fee
**When:** POST adjustment: `{"adjustment_type":"PERCENTAGE_DISCOUNT","percentage_rate":20.0}`
**And:** Invoice generated for full month
**Then:** `invoice.Subtotal = $800.00` (was $1,000); `invoice.AdjustedSubtotal = $800.00`; `invoice.Total = $800.00`

### TC-02 — Minimum commitment floors below-cost invoice

**Given:** Min commitment = $500, calculated total = $400 ($200 base + $200 usage)
**When:** Invoice generated
**Then:** `invoice.Total = $500.00`; `invoice.AdjustedSubtotal` set to $500; A $100 minimum commitment adjustment is implicit

### TC-03 — Maximum cap on high-usage invoice

**Given:** Max cap = $10,000, calculated total = $12,500
**When:** Invoice generated
**Then:** `invoice.Total = $10,000.00` (capped)

### TC-04 — Discount expiry

**Given:** 20% discount with `expires_after_periods=2`
**When:** `periodCount >= 2`
**Then:** `filterActiveAdjustments()` returns empty; no discount applied; full price charged

### TC-05 — All 5 adjustments together

**Given:** Subscription has all 5 adjustment types
**When:** Invoice generated with usage
**Then:** Pipeline executes in order:
1. Usage discount reduces quantity
2. Amount discount distributes proportionally
3. Percentage discount reduces each line
4. Minimum commitment floors total
5. Maximum cap ceilings total

---

**Estimate:** 3-4 sprints
**Dependencies:** None


> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P0-01 |
| **Sprint** | Sprint 3-6 |
| **Phase** | Billing Core |
| **Domain** | Invoice Engine — Adjustments |
| **Priority** | P0 — Critical |

---

## Title

**Discount & Commitment Pipeline** — percentage discounts, fixed amount discounts, minimum commitments, and maximum spend caps on subscriptions

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P0 — Critical |

---

## Description

**As a product manager**, I want to configure **percentage discounts, fixed amount discounts, minimum commitments, and maximum spend caps** on subscriptions so that I can run **promotional campaigns, enterprise contracts, and volume-based pricing** without engineering involvement.

### Current State

The current invoice pipeline in `engine/internal/invoice/generate.go` follows M-5:
```
subtotal → FEFO credits → taxable → tax → total
```

There is **no adjustment step** between subtotal and credits. The `Adjustment` struct in `types.go` is minimal:
```go
type Adjustment struct {
    Description string
    Amount      decimal.Decimal
}
```

No discount types, no pipeline ordering, no coupon system.

### Target State

Add a full adjustment pipeline between subtotal and credits:

```
1. Subtotal (sum of all line items)
2. ADJUSTMENTS (NEW):
   a. Usage discounts (reduce quantity pre-pricing)
   b. Amount discounts (fixed $ off)
   c. Percentage discounts (% off line total)
   d. Minimum commitment (floor)
   e. Maximum spend cap (ceiling)
3. Adjusted subtotal = subtotal after adjustments
4. FEFO credits applied
5. Taxable = adjusted subtotal − credits_applied
6. Tax = round_half_up(taxable × rate, minor_units)
7. Total = taxable + tax
```

---

## RBAC Roles

| Role | Can create adjustments | Can update adjustments | Can view adjustments | Scope |
|------|----------------------|----------------------|---------------------|-------|
| **SUPER_ADMIN** | Yes (any org) | Yes (any org) | Yes (any org) | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Yes (own org) | Yes (own org) | Own org only |
| **CUSTOMER** | No | No | Yes (own adjustments) | Own account only |
| **END_USER** | No | No | No | No access |

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Admin can create, read, update, and delete 5 adjustment types: `USAGE_DISCOUNT`, `AMOUNT_DISCOUNT`, `PERCENTAGE_DISCOUNT`, `MINIMUM_COMMITMENT`, `MAXIMUM_CAP` on a subscription or contract via API and UI |
| AC-2 | Usage discounts reduce quantity before pricing: `effective_qty = max(0, qty - discount_qty)` |
| AC-3 | Percentage discounts correctly reduce line item totals: if a $100 line has a 20% discount, the adjusted amount is $80.00 |
| AC-4 | Minimum commitment floors the invoice total: if calculated total is $400 and minimum is $500, invoice total is $500.00 |
| AC-5 | Maximum spend cap ceilings the invoice total: if calculated total is $12,000 and cap is $10,000, invoice total is $10,000.00 |
| AC-6 | Discounts with N-period expiry stop applying after the specified number of billing periods |
| AC-7 | Minimums apply BEFORE credits — a $500 minimum with $200 in credits results in $300 charged (not $0) |
| AC-8 | Multi-line percentage discounts distribute proportionally by subtotal share across line items |
| AC-9 | Pipeline order is preserved: usage discount → amount discount → percentage discount → minimum → maximum → credits → tax → total |

---

## Test Cases

### TC-01 — Create and apply a percentage discount

**Given:** Authenticated ORG_ADMIN, subscription `sub_001` with $1,000 subtotal
**When:** POST `/api/v1/subscriptions/sub_001/adjustments` `{ "adjustment_type": "PERCENTAGE_DISCOUNT", "percentage_rate": 20.0, "description": "20% launch promo" }`
**Then:** 201 returned; adjustment created with id
**When:** Invoice is generated for the period
**Then:** Invoice total shows $800.00 (20% off $1,000); adjustment line item shows "$200.00 discount (20% launch promo)"

### TC-02 — Minimum commitment floors the invoice

**Given:** Subscription `sub_001` has minimum commitment of $500.00; calculated total = $400.00
**When:** Invoice is generated
**Then:** Invoice total = $500.00 (floored); invoice shows $100.00 as "Minimum Commitment Adjustment"

### TC-03 — Maximum spend cap applied

**Given:** Subscription `sub_001` has maximum cap of $10,000.00; calculated total = $12,000.00
**When:** Invoice is generated
**Then:** Invoice total = $10,000.00 (capped); invoice shows $2,000.00 as "Spend Cap Adjustment (savings)"

### TC-04 — Discount expires after N periods

**Given:** Subscription has a 20% discount with `expires_after_periods: 3`, `period_count: 3`
**When:** Period 4 invoice is generated
**Then:** Discount no longer applies; subtotal = full amount; adjustment pipeline skipped for expired discounts

### TC-05 — Multi-line proportional distribution

**Given:** Subscription has 2 lines: Line A = $600, Line B = $400; 10% percentage discount applied
**When:** Invoice is generated
**Then:** Line A adjusted = $540.00; Line B adjusted = $360.00; total = $900.00

### TC-06 — Minimum + credits interaction

**Given:** Subscription has minimum commitment of $500; calculated total = $400; customer has $200 in credits
**When:** Invoice is generated
**Then:** Minimum floors to $500; credits applied: $500 - $200 = $300 charged

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `POST` | `/api/v1/subscriptions/:subscriptionId/adjustments` | Create adjustment | JWT · OrgAdminGuard |
| `GET` | `/api/v1/subscriptions/:subscriptionId/adjustments` | List adjustments | JWT · OrgAdminGuard |
| `PATCH` | `/api/v1/adjustments/:adjustmentId` | Update adjustment | JWT · OrgAdminGuard |
| `DELETE` | `/api/v1/adjustments/:adjustmentId` | Remove adjustment | JWT · OrgAdminGuard |
| `GET` | `/api/v1/adjustments/:adjustmentId` | Get adjustment detail | JWT · OrgAdminGuard |

---

## Data Tables Used (Prisma)

| Table | Schema | Operation | Key Columns |
|-------|--------|-----------|-------------|
| `adjustments` | `billing` (NEW) | INSERT, SELECT, UPDATE, DELETE | `id, subscription_id, org_id, adjustment_type (enum), amount DECIMAL(38,9), percentage_rate DECIMAL(12,4), description, expires_after_periods INT, period_count INT, applies_to (ALL\|LINE_ITEMS), created_at, updated_at` |
| `invoices` | `billing` | SELECT, UPDATE | `id, subscription_id, subtotal, adjusted_subtotal, credits_applied, total` |
| `invoice_line_items` | `billing` | SELECT | `id, invoice_id, line_type, amount, description, adjustment_id (nullable FK)` |

---

## Code Changes Required

### 1. Prisma Schema — New `billing.adjustments` table

```prisma
model Adjustment {
  id                  String          @id @default(uuid()) @db.Uuid
  subscriptionId      String          @map("subscription_id") @db.Uuid
  orgId               String          @map("org_id") @db.Uuid
  adjustmentType      AdjustmentType  @map("adjustment_type")
  amount              Decimal?        @db.Decimal(38, 9)  // for AMOUNT/MIN/MAX
  percentageRate      Decimal?        @map("percentage_rate") @db.Decimal(12, 4)  // for PERCENTAGE
  description         String
  expiresAfterPeriods Int?            @map("expires_after_periods")
  periodCount         Int?            @map("period_count")
  createdAt           DateTime        @default(now()) @map("created_at")
  updatedAt           DateTime        @updatedAt @map("updated_at")

  subscription Subscription @relation(fields: [subscriptionId], references: [id])
  organization Organization  @relation(fields: [orgId], references: [id])

  @@index([subscriptionId])
  @@index([orgId])
  @@map("adjustments")
  @@schema("billing")
}

enum AdjustmentType {
  USAGE_DISCOUNT
  AMOUNT_DISCOUNT
  PERCENTAGE_DISCOUNT
  MINIMUM_COMMITMENT
  MAXIMUM_CAP
}
```

### 2. Engine — New Adjustment Types

**File:** `engine/internal/invoice/types.go`
- Add `AdjustmentType` enum with 5 values
- Enhance `Adjustment` struct with `Type AdjustmentType`, `PercentageRate`, `ExpiresAfterPeriods`, `PeriodCount`

```go
type AdjustmentType string

const (
    AdjustmentUsageDiscount    AdjustmentType = "USAGE_DISCOUNT"
    AdjustmentAmountDiscount   AdjustmentType = "AMOUNT_DISCOUNT"
    AdjustmentPercentageDisc   AdjustmentType = "PERCENTAGE_DISCOUNT"
    AdjustmentMinCommitment    AdjustmentType = "MINIMUM_COMMITMENT"
    AdjustmentMaxCap           AdjustmentType = "MAXIMUM_CAP"
)

type Adjustment struct {
    ID                string
    SubscriptionID    string
    AdjustmentType    AdjustmentType
    Amount            decimal.Decimal
    PercentageRate    decimal.Decimal
    Description       string
    ExpiresAfterPeriods int
    PeriodCount       int
}
```

### 3. Engine — Pipeline Integration

**File:** `engine/internal/invoice/generate.go`
- Add `applyAdjustments()` function called between subtotal calculation and credit application
- Pipeline order: usage discount → amount discount → percentage discount → minimum → maximum

```go
func (e *InvoiceEngine) applyAdjustments(lines []LineItem, adjs []Adjustment, periodCount int) ([]LineItem, Decimal, error) {
    // 1. Filter active adjustments (not expired)
    active := filterActiveAdjustments(adjs, periodCount)
    
    // 2. Apply usage discounts (reduce quantity)
    lines = applyUsageDiscounts(lines, active)
    
    // 3. Apply amount discounts (fixed $, proportionally distributed)
    lines, amountTotal := applyAmountDiscounts(lines, active)
    
    // 4. Apply percentage discounts (% off each line)
    lines, pctTotal := applyPercentageDiscounts(lines, active)
    
    // 5. Sum adjusted lines
    subtotal := sumLines(lines)
    
    // 6. Apply minimum commitment
    subtotal = applyMinCommitment(subtotal, active)
    
    // 7. Apply maximum cap
    subtotal = applyMaxCap(subtotal, active)
    
    return lines, subtotal, nil
}
```

### 4. BFF — Adjustments Module

**New directory:** `control-plane/src/billing/adjustments/`
- `adjustments.controller.ts` — CRUD endpoints
- `adjustments.service.ts` — business logic with Prisma
- `adjustments.module.ts` — NestJS module
- DTOs with validation for each adjustment type

### 5. Web UI — Adjustment Configuration

**New directory:** `web/app/subscriptions/[id]/adjustments/`
- Adjustment list table with type badges, amounts, expiry status
- "Add Adjustment" modal/form with type-specific fields
- Edit/delete actions

---

## State Machine

### Adjustment Lifecycle

```
CREATED (active)
  │
  ├── period_count >= expires_after_periods → EXPIRED (auto)
  │
  ├── Admin deletes → REMOVED
  │
  └── UPDATE → MODIFIED (period_count resets on edit)
```

### Pipeline Ordering

```
1. USAGE_DISCOUNT    → reduce quantity: effective_qty = max(0, qty - discount_qty)
2. AMOUNT_DISCOUNT   → fixed $ off: distribute proportionally across lines
3. PERCENTAGE_DISCOUNT → % off: each line reduced by percentage
4. MINIMUM_COMMITMENT → floor: max(calculated_total, min_amount)
5. MAXIMUM_CAP       → ceiling: min(calculated_total, max_amount)
```

---

## Estimate

**3-4 sprints** (design 1, engine changes 1, BFF + Prisma 1, Web UI 1)

**Dependencies:** None — new pipeline step, does not modify existing steps
