# QuantumBilling User Story: Self-Serve Subscription Management

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P1-06 |
| **Sprint** | Sprint 8-11 |
| **Phase** | Customer Portal |
| **Domain** | Subscriptions — Customer Self-Service |
| **Priority** | P1 — High |

---

## Title

**Self-Serve Subscription Management** — upgrade, downgrade, or cancel subscriptions from a customer portal

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 — High |

---

## Description

**As a customer**, I want to **upgrade, downgrade, or cancel my subscription from a self-serve portal** with **immediate effective dates and correct mid-cycle proration** so that I can **manage my plan without contacting support.**

### Current State

`changeSubscriptionPlan()`, `cancelSubscription()`, `reactivateSubscription()` in `catalog.service.ts` are all `@Roles('ORG_ADMIN', 'SUPER_ADMIN')`. No CUSTOMER role endpoints. No self-serve portal UI exists beyond dashboards.

### Target State

Customer portal with:
- Current plan details + available upgrade/downgrade paths
- Upgrade: immediate effect
- Downgrade/cancel: immediate or end-of-period (customer choice)
- Correct mid-cycle proration (P-1 through P-5)
- Credit note for unused portion on downgrade

---

## RBAC Roles

| Role | Can upgrade/downgrade | Can cancel | Can view plans | Scope |
|------|----------------------|------------|----------------|-------|
| **SUPER_ADMIN** | Yes (any org) | Yes (any org) | Yes (all) | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Yes (own org) | Yes (own) | Own org only |
| **CUSTOMER** | ✅ Yes (own) | ✅ Yes (own) | Yes (available) | Own account only |
| **END_USER** | No | No | No | No access |

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Customer portal shows current plan details, available upgrade/downgrade paths, and pricing comparison |
| AC-2 | Upgrade takes effect immediately; downgrade and cancellation can be immediate or end-of-period (customer choice) |
| AC-3 | Mid-cycle plan change correctly prorates: unused portion of old plan credited, new plan charged from change date (P-1 through P-5) |
| AC-4 | Plan change generates a credit note for the unused portion (if downgrade) and a new invoice line for the upgrade charge |
| AC-5 | Usage-based charges continue uninterrupted across plan change — no usage data loss |
| AC-6 | Admin can configure which plan changes are allowed (upgrade/downgrade/crossgrade) and effective timing |

---

## Code Changes Required

### 1. BFF — Add CUSTOMER role to subscription endpoints

**File:** `control-plane/src/catalog/catalog.controller.ts`
```typescript
@Post(':subscriptionId/change-plan')
@Roles('ORG_ADMIN', 'SUPER_ADMIN', 'CUSTOMER')  // ADD CUSTOMER
async changePlan(@Param('subscriptionId') id: string, @Body() dto: ChangePlanDto) { ... }
```

### 2. BFF — Scope validation for CUSTOMER

Ensure CUSTOMER can only change their own subscription:
```typescript
if (actor.role === 'CUSTOMER' && subscription.customerId !== actor.customerId) {
    throw new ForbiddenException('Cannot modify another customer\'s subscription');
}
```

### 3. Web UI — Plan Management Page

**New:** `web/app/portal/plan/`
- Current plan card with details
- Available plans comparison table
- Upgrade/downgrade flow with price comparison
- Cancel flow with effective date choice
- Confirmation dialogs with proration details

### 4. Engine — Proration Correctness

Verify P-1 through P-5 rules are correctly applied for mid-cycle changes initiated by customers:
- P-2: Base fee prorated `calendarDays(changeDate, periodEnd) / calendarDays(periodStart, periodEnd)`
- P-3: Allowance prorated similarly
- P-4: Seats prorated

---

**Estimate:** 3-4 sprints (BFF changes 1, Web UI 2, Engine verification + integration 1)
**Dependencies:** QB-STORY-P0-01 (discount pipeline must exist for credit note generation), QB-STORY-P1-04 (plan phases)
