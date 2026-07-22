# QuantumBilling User Story: Add-Ons as First-Class Entities

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P1-05 |
| **Sprint** | Sprint 8-9 |
| **Phase** | Catalog |
| **Domain** | Products — Add-Ons |
| **Priority** | P1 — High |

---

## Title

**Add-Ons as First-Class Entities** — catalog of versioned, independently subscribable add-ons attachable to subscriptions

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 — High |

---

## Description

**As a customer**, I want to **browse and subscribe to add-ons from a catalog** (e.g., "Additional Storage 100GB", "Premium Support") and **attach them to my existing subscription** so that I can **expand my service without managing multiple subscriptions.**

### Current State

`ProductType` enum in Prisma has `ADD_ON` value — but no add-on logic exists anywhere in engine or control-plane code. Currently each add-on requires a separate subscription.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Admin can create versioned add-ons in the catalog with name, description, price, billing period, and meter references |
| AC-2 | Customer can subscribe to an add-on and attach it to an existing subscription from the customer portal |
| AC-3 | Add-on charges appear as separate line items on the subscription's invoice with `ADD_ON` line type |
| AC-4 | Add-ons can be independently cancelled without affecting the parent subscription |
| AC-5 | Add-on pricing supports all 7 pricing models (FLAT, PER_UNIT, TIERED_GRADUATED, etc.) |
| AC-6 | Mid-period add-on subscription prorates to the next period boundary |

---

## Code Changes Required

### Prisma — Add-on model

```prisma
model AddOn {
  id             String   @id @default(uuid()) @db.Uuid
  orgId          String   @map("org_id") @db.Uuid
  name           String
  description    String?
  price          Decimal  @db.Decimal(38, 9)
  billingPeriod  BillingPeriod
  meterId        String?  @map("meter_id") @db.Uuid
  pricingModelId String?  @map("pricing_model_id") @db.Uuid
  status         AddOnStatus @default(DRAFT)
  version        Int      @default(1)
  createdAt      DateTime @default(now()) @map("created_at")
  updatedAt      DateTime @updatedAt @map("updated_at")

  organization Organization @relation(fields: [orgId], references: [id])
  @@index([orgId])
  @@map("add_ons")
  @@schema("catalog")
}
```

### Engine — ADD_ON line type

The `LineType` enum already includes `SEAT` and `ADJUSTMENT`. Add `ADD_ON`:

```go
const LineTypeAddOn LineType = "ADD_ON"
```

---

**Estimate:** 2 sprints
