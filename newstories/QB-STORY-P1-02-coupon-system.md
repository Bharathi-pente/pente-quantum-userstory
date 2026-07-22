# QuantumBilling User Story: Customer-Facing Coupon System

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P1-02 |
| **Sprint** | Sprint 11-12 |
| **Phase** | Marketing |
| **Domain** | Discounts — Promotions |
| **Priority** | P1 — High |

---

## Title

**Customer-Facing Coupon System** — stackable coupon codes with expiration dates and usage limits

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 — High |

---

## Description

**As a marketing manager**, I want to **create stackable coupon codes** (percentage off, fixed amount, first-N-free) with **expiration dates and usage limits** so that I can **run promotional campaigns and track redemption rates**.

### Current State

No coupon system exists anywhere in the codebase. Zero Prisma models, zero Go code, zero TypeScript endpoints.

### Target State

- Coupon codes with discount types: PERCENTAGE, FIXED_AMOUNT, USAGE_DISCOUNT
- Stackable with configurable max stack count (default: 3)
- Expiration dates, max redemptions, applicable plan filters
- Redemption tracking dashboard

---

## RBAC Roles

| Role | Can create coupons | Can apply coupons | Can view redemptions | Scope |
|------|-------------------|-------------------|---------------------|-------|
| **SUPER_ADMIN** | Yes (any org) | N/A | Yes (all orgs) | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | N/A | Yes (own org) | Own org only |
| **CUSTOMER** | No | Yes (at checkout) | Yes (own applied) | Own account only |
| **END_USER** | No | No | No | No access |

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Admin can create coupons with: `code`, `discount_type` (PERCENTAGE | FIXED_AMOUNT | USAGE_DISCOUNT), `value`, `max_redemptions`, `expires_at`, `applicable_plan_ids` |
| AC-2 | Customer can apply a coupon code to their subscription at checkout in the portal |
| AC-3 | Multiple coupons can be stacked on a single subscription (configurable max stack count) |
| AC-4 | Coupon discounts appear as line items on the invoice with the coupon code and description |
| AC-5 | Redemption tracking: admin dashboard shows coupons issued, redeemed, expired, remaining |

---

## Test Cases

### TC-01 — Create and apply percentage coupon

**Given:** Admin creates coupon `LAUNCH20` with 20% off, max 100 redemptions
**When:** Customer applies `LAUNCH20` at checkout
**Then:** Invoice shows 20% discount applied with code `LAUNCH20`

### TC-02 — Coupon expiration

**Given:** Coupon `EXPIRED50` expired 5 days ago
**When:** Customer tries to apply `EXPIRED50`
**Then:** 422 returned: "Coupon EXPIRED50 has expired"

---

## Code Changes Required

### Prisma — New tables

```prisma
model Coupon {
  id              String       @id @default(uuid()) @db.Uuid
  orgId           String       @map("org_id") @db.Uuid
  code            String       @unique
  discountType    DiscountType @map("discount_type")
  value           Decimal      @db.Decimal(38, 9)
  maxRedemptions  Int?         @map("max_redemptions")
  expiresAt       DateTime?    @map("expires_at")
  applicablePlanIds String[]?  @map("applicable_plan_ids")
  maxStackCount   Int          @default(3) @map("max_stack_count")
  createdAt       DateTime     @default(now()) @map("created_at")
  @@schema("marketing")
}

model CouponRedemption {
  id          String   @id @default(uuid()) @db.Uuid
  couponId    String   @map("coupon_id") @db.Uuid
  subscriptionId String @map("subscription_id") @db.Uuid
  redeemedAt  DateTime @default(now()) @map("redeemed_at")
  @@unique([couponId, subscriptionId])
  @@schema("marketing")
}
```

**Estimate:** 2 sprints
**Depends on:** QB-STORY-P0-01 (discount pipeline)
