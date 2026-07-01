# QuantumBilling User Story: Subscription Management

**QB-STORY-022** · Sprint 7 · Phase: Billing Core

---

## Title

**Subscription Management** — assign plans to organizations and manage the full subscription lifecycle

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As a SUPER_ADMIN or ORG_ADMIN**, I want to assign plans to organizations and manage their subscription lifecycle, so that after onboarding an organization, I can give them access to the platform by linking them to a plan, and then manage that subscription through its entire lifecycle (billing, upgrades, cancellations).

The Subscription Management feature covers:

- **Subscription Assignment** — after onboarding an organization, assign a plan to activate their account
- **Subscription Lifecycle** — manage status transitions (trial → active → past_due → canceled)
- **Plan Changes** — upgrade or downgrade organizations between plans with proration handling
- **Cancellation** — cancel subscriptions immediately or schedule for end of billing period
- **Subscription Visibility** — view all subscriptions for an organization

**Core Concept:** A **Subscription** connects an **Organization** to a **Plan**. The organization cannot use the platform without an active subscription. The subscription determines what the organization can access, what pricing applies, and how they are billed.

Key behaviors:
- An **Organization** must have at least one **active subscription** to use the platform
- Subscription is assigned **after onboarding** the organization
- One **Organization** can have **multiple active subscriptions** (e.g., base plan + GPU compute add-on)
- Each subscription generates its **own invoice**
- Subscriptions can start **immediately** or be **future-dated**
- Cancellations can be **immediate** or **scheduled** (cancel at period end)
- Plan upgrades/downgrades can take effect **immediately** or at **next billing cycle**

---

## RBAC Roles

| Role | Can view subscriptions | Can create subscriptions | Can cancel subscriptions | Can change plan | Scope |
|------|----------------------|------------------------|-------------------------|-----------------|-------|
| **SUPER_ADMIN** | Yes (all orgs) | Yes (all orgs) | Yes (all orgs) | Yes (all orgs) | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Yes (own org) | Yes (own org) | Yes (own org) | Own org only |
| **CUSTOMER** | No | No | No | No | No access |
| **END_USER** | No | No | No | No | No access |

---

## Subscription States

```
                    ┌──────────────┐
                    │   trialing   │ (optional, if plan supports trial)
                    └──────┬───────┘
                           │ trial ends / conversion
                           ▼
                    ┌──────────────┐
        ┌──────────►│    active    │◄──────────┐
        │           └──────┬───────┘           │
        │                  │                   │
   suspend/            billing              │
   payment            period                │
   failure            end                   │
        │                  │                 │
        │           ┌──────┴───────┐         │
        │           │              │         │
        ▼           ▼              │         ▼
  ┌──────────┐  ┌──────────┐      │   ┌──────────┐
  │ past_due │  │  active  │      │   │ canceled │
  │          │  │ (renewed)│      │   │          │
  └────┬─────┘  └──────────┘      │   └──────────┘
       │                           │        ▲
       │ payment                   │        │
       │ succeeded                 │   cancel requested
       ▼                           │   (immediate or
  ┌──────────┐                     │   end_of_period)
  │  active  │────────────────────┘
  └──────────┘
```

**State Definitions:**
| State | Description |
|-------|-------------|
| `scheduled` | Future-dated subscription, not yet active (start_date > today) |
| `trialing` | Organization is on a trial period, no charges yet |
| `active` | Subscription is live, organization can use the platform, billing occurs |
| `past_due` | Payment failed at billing, subscription at risk (grace period starts) |
| `suspended` | Grace period expired, organization access restricted |
| `canceled` | Subscription has been terminated, organization access revoked |

---

## Acceptance Criteria

### Subscription Assignment (Post-Onboarding)

1. After an organization is created/onboarded, ORG_ADMIN or SUPER_ADMIN can assign a plan by creating a subscription.
2. The subscription links the **organization** (not a customer) to a **plan**.
3. Creating a subscription requires: organization (required), plan (required), billing period (monthly/annual, default: monthly), start date (default: immediate).
4. For immediate subscriptions, status becomes `active` upon creation.
5. For future-dated subscriptions with a start date in the future, status becomes `scheduled` until the start date is reached.
6. Creating a subscription with a trial-enabled plan sets status to `trialing`.
7. MRR is calculated from the plan's base price and billing period.
8. At least one active payment method must be on file for the organization (unless trial).

### Organization Access Control

9. An organization **without an active subscription** (status != `active` or `trialing`) cannot access the platform.
10. API requests from organizations with no active subscription return `403 Subscription Required`.
11. Organization status indicator shows whether they have an active subscription.
12. Onboarding flow includes a step: "Assign Subscription" if none exists.

### Subscription List View

13. An organization's subscriptions are visible at `/organizations/:orgId/subscriptions`.
14. Each subscription row displays: plan name, status badge, MRR, billing period, next billing date, start date.
15. Status badges use colors: scheduled (purple), trialing (blue), active (green), past_due (amber), suspended (red), canceled (gray).
16. Subscriptions can be filtered by status (All, Active, Scheduled, Trialing, Past Due, Suspended, Canceled).
17. Clicking a subscription row opens the subscription detail view.

### Subscription Detail View

18. Subscription detail shows: organization name, plan name, status, MRR, billing period, start date, next billing date, end date (if applicable).
19. Shows subscription line items: included units (e.g., API Calls: 100,000) and their usage vs. limits.
20. Shows associated contracts (if any).
21. "Change Plan" button allows upgrading or downgrading.
22. "Cancel Subscription" button opens cancellation flow.

### Plan Change (Upgrade/Downgrade)

23. Clicking "Change Plan" opens a modal listing available plans.
24. Current plan is highlighted/disabled.
25. Selecting a new plan shows the price difference and proration estimate.
26. "Apply Immediately" option applies the change right away with proration credit.
27. "At Next Billing" option schedules the change for the next billing period.
28. Proration formula: `(daysRemaining / totalDaysInPeriod) × (newPlanPrice - oldPlanPrice)`.
29. Downgrade cannot occur if current usage exceeds the new plan's included unit limits.

### Cancellation

30. "Cancel Subscription" opens a confirmation modal.
31. Two options: "Cancel Immediately" or "Cancel at End of Period".
32. "Cancel at End of Period" sets `cancel_at_period_end = true`; subscription stays active until period end.
33. "Cancel Immediately" sets status to `canceled` and revokes access immediately.
34. Cancellation reason is required (dropdown: customer_request, payment_failed, plan_too_expensive, migrating_away, other).
35. "Other" shows a free-text field for reason.
36. Canceling triggers webhook `subscription.cancelled`.

### Past Due & Suspension Flow

37. When a payment fails, subscription status changes to `past_due`.
38. A warning banner appears on the subscription detail.
39. After `GRACE_PERIOD_DAYS` (default: 7), status changes to `suspended`.
40. During `past_due`, organization still has access.
41. During `suspended`, organization access is restricted (API blocked, dashboard shows warning).
42. Upon successful payment retry, status returns to `active`.

### Subscription Reactivation

43. A `canceled` subscription cannot be reactivated; create a new subscription instead.
44. A `suspended` subscription can be reactivated by retrying payment and succeeding.
45. Reactivation sets status back to `active`.

### Multiple Subscriptions per Organization

46. An organization can have multiple active subscriptions simultaneously.
47. Each subscription appears as a separate billing entity with its own invoices.
48. The org's total MRR is the sum of all active subscriptions' MRR.
49. Usage is tracked independently per subscription (each subscription has its own usage limits).

---

## Test Cases

### TC-01 — Assign subscription to newly onboarded org

**Given:** organization "NewOrg" was just onboarded and has no subscription
**When:** SUPER_ADMIN creates a subscription with plan "Pro", billing period "Monthly", start date "immediate"
**Then:** subscription is created with status `active`, MRR = $99
**And:** org "NewOrg" now has access to the platform
**And:** next billing date = 30 days from now

---

### TC-02 — Assign future-dated subscription

**Given:** organization "UpcomingCorp" is being onboarded with start date "2025-02-01"
**When:** SUPER_ADMIN creates a subscription with plan "Enterprise", start date = "2025-02-01"
**Then:** subscription is created with status `scheduled`
**And:** org does NOT have access until 2025-02-01
**And:** on 2025-02-01, subscription automatically becomes `active`

---

### TC-03 — Assign trial subscription

**Given:** organization "TrialUser" wants to evaluate the platform
**When:** SUPER_ADMIN creates a subscription with plan "Pro" (trial-enabled), start date "immediate"
**Then:** subscription status is `trialing`, trial_end = 14 days from now
**And:** no charges during trial
**And:** org has full access during trial period

---

### TC-04 — Org without subscription is blocked

**Given:** organization "NoSubOrg" has a subscription with status `canceled`
**When:** an API request is made by "NoSubOrg"
**Then:** response is `403 Subscription Required`
**And:** dashboard shows a banner: "Your subscription is not active. Please contact support."

---

### TC-05 — View subscriptions for organization

**Given:** org "Acme" has 2 active subscriptions: Pro and GPU Add-on
**When:** ORG_ADMIN navigates to the organization's subscriptions view
**Then:** both subscriptions are listed with plan, status, MRR, billing period, next billing date

---

### TC-06 — Upgrade plan immediately

**Given:** org "Acme" has Pro subscription ($99/month)
**When:** SUPER_ADMIN clicks "Change Plan", selects "Enterprise" ($499/month), chooses "Apply Immediately"
**Then:** plan changes to Enterprise immediately
**And:** proration credit is calculated and stored
**And:** MRR updates to $499

---

### TC-07 — Schedule downgrade for next billing

**Given:** org "Acme" has Enterprise subscription, billing date in 15 days
**When:** SUPER_ADMIN downgrades to Pro with "At Next Billing"
**Then:** plan remains Enterprise until billing date
**And:** `effective_next_period = true` is set
**And:** on billing date, plan becomes Pro automatically

---

### TC-08 — Cancel at end of period

**Given:** org "Leaving" has active subscription with billing date in 10 days
**When:** SUPER_ADMIN cancels with "Cancel at End of Period"
**Then:** status remains `active`
**And:** `cancel_at_period_end = true`
**And:** subscription auto-cancels on billing date
**And:** no more invoices after cancellation

---

### TC-09 — Cancel immediately

**Given:** org "LeavingNow" requests immediate cancellation
**When:** SUPER_ADMIN cancels with "Cancel Immediately" and reason "customer_request"
**Then:** status changes to `canceled` immediately
**And:** org access is revoked immediately
**And:** webhook `subscription.canceled` is fired

---

### TC-10 — Past due → suspended flow

**Given:** org "LatePayer" has active subscription, payment fails
**When:** billing processor runs after payment failure
**Then:** status changes to `past_due`
**And:** warning banner appears
**And:** after 7 days of non-payment, status changes to `suspended`
**And:** org access is blocked

---

### TC-11 — Reactivate suspended subscription

**Given:** org "LatePayer" has suspended subscription
**When:** SUPER_ADMIN retries payment and it succeeds
**Then:** status changes back to `active`
**And:** org access is restored

---

### TC-12 — Multiple subscriptions per org

**Given:** org "EnterprisePlus" wants base plan + GPU add-on
**When:** creating two subscriptions: Pro ($99) + GPU Compute Add-on ($199)
**Then:** both appear in org's subscription list
**And:** each generates separate invoices
**And:** combined MRR = $298

---

### TC-13 — Downgrade blocked by usage

**Given:** org "HeavyUser" on Enterprise uses 50M tokens (exceeds Pro limit of 1M)
**When:** attempting to downgrade to Pro
**Then:** error: "Cannot downgrade: usage exceeds new plan limits"
**And:** downgrade is blocked

---

### TC-14 — ORG_ADMIN cannot manage another org's subscription

**Given:** ORG_ADMIN for org `acme`
**When:** attempting to view/modify subscription for org `othercorp`
**Then:** 403 `FORBIDDEN`

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/v1/organizations/:orgId/subscriptions` | List all subscriptions for an organization | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `GET` | `/api/v1/subscriptions/:subscriptionId` | Get a specific subscription | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `POST` | `/api/v1/organizations/:orgId/subscriptions` | Create and assign subscription to org | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `PUT` | `/api/v1/subscriptions/:subscriptionId` | Update subscription (billing period, etc.) | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `POST` | `/api/v1/subscriptions/:subscriptionId/cancel` | Cancel subscription | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` · Body: `{ "mode": "immediate" \| "end_of_period", "reason": "..." }` |
| `POST` | `/api/v1/subscriptions/:subscriptionId/reactivate` | Reactivate suspended subscription | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `POST` | `/api/v1/subscriptions/:subscriptionId/change-plan` | Change plan (upgrade/downgrade) | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` · Body: `{ "planId": "...", "mode": "immediate" \| "next_period" }` |
| `GET` | `/api/v1/platform/subscriptions` | List all subscriptions (SuperAdmin) | JWT · Guard: `SuperAdminGuard` · Query: `?org_id=&status=` |

---

## Data Tables Used

| Table | Schema | Operation | Key Columns |
|-------|--------|-----------|-------------|
| `subscriptions` | `billing` | SELECT · INSERT · UPDATE | `id, org_id, plan_id, status, billing_period, mrr, start_date, end_date, cancel_at_period_end, trial_end, created_at` |
| `organizations` | `identity` | SELECT | `id, name, status` |
| `plans` | `catalog` | SELECT | `id, name, base_price, billing_period, status` |
| `invoices` | `billing` | SELECT | `id, subscription_id, status, total_amount` |
| `contracts` | `customer` | SELECT | `id, org_id, subscription_id` |
| `payment_methods` | `billing` | SELECT | `id, org_id, status` |

---

## State Machine — Subscription Lifecycle

### State Transitions

| From State | To State | Trigger |
|-----------|----------|---------|
| `scheduled` | `active` | Start date reached (cron job) |
| `scheduled` | `canceled` | Cancel before start date |
| `trialing` | `active` | Trial ends, payment succeeds |
| `trialing` | `canceled` | Trial ends, no conversion |
| `active` | `past_due` | Payment fails at billing |
| `active` | `active` | Billing period renews (new invoice created) |
| `active` | `canceled` | Cancel immediately |
| `active` | `canceled` | Cancel at end of period (flag set, period ends) |
| `past_due` | `active` | Payment retried and succeeds |
| `past_due` | `suspended` | Grace period expires |
| `suspended` | `active` | Payment retried and succeeds |
| `suspended` | `canceled` | Max suspension days reached |

### Cron Jobs Required

1. **`subscription-activator`** — runs every hour: activates `scheduled` subscriptions where `start_date <= now()`
2. **`subscription-canceler`** — runs daily: cancels subscriptions where `cancel_at_period_end = true` AND `end_date <= now()`
3. **`billing-processor`** — runs daily: attempts to charge active subscriptions, sets `past_due` on failure, then `suspended` after grace period
4. **`trial-converter`** — runs daily: converts expired trials to canceled if no payment method or trial not converted

---

## Error Codes

| Code | HTTP | Trigger |
|------|------|---------|
| `SUBSCRIPTION_NOT_FOUND` | 404 | `subscriptionId` does not exist |
| `ORGANIZATION_NOT_FOUND` | 404 | `orgId` does not exist |
| `PLAN_NOT_FOUND` | 404 | `planId` does not exist |
| `SUBSCRIPTION_ALREADY_CANCELED` | 409 | Canceling an already canceled subscription |
| `SUBSCRIPTION_NOT_SUSPENDED` | 409 | Reactivate called on non-suspended subscription |
| `DOWNGRADE_BLOCKED_BY_USAGE` | 422 | Current usage exceeds new plan's included units |
| `NO_PAYMENT_METHOD` | 422 | Creating non-trial subscription with no payment method on file |
| `SUBSCRIPTION_ALREADY_EXISTS` | 409 | Only one active subscription allowed per org (if not multi-subscription) |
| `INVALID_BILLING_PERIOD` | 422 | `billing_period` must be `monthly` or `annual` |
| `FUTURE_START_DATE_REQUIRED` | 422 | Scheduled subscription requires future `start_date` |
| `FORBIDDEN` | 403 | Actor not authorized for this organization's subscriptions |
| `UNAUTHORIZED` | 401 | No valid JWT token |

---

## Environment Config Keys

| Key | Description |
|-----|-------------|
| `DEFAULT_BILLING_PERIOD` | Default billing period for new subscriptions: `monthly` or `annual` (default: `monthly`) |
| `TRIAL_PERIOD_DAYS` | Default trial period in days (default: `14`) |
| `GRACE_PERIOD_DAYS` | Days after `past_due` before `suspended` (default: `7`) |
| `MAX_SUSPENSION_DAYS` | Max days suspended before auto-cancel (default: `30`) |
| `PRORATION_HANDLING` | Proration on plan change: `credit` or `immediate_invoice` (default: `credit`) |
| `ALLOW_MULTIPLE_SUBSCRIPTIONS` | Allow multiple active subscriptions per org (default: `true`) |
| `REQUIRE_PAYMENT_METHOD_FOR_TRIAL` | Require payment method for trial subscriptions (default: `true`) |
| `ACCESS_WITHOUT_SUBSCRIPTION` | Allow API access without subscription (for testing) (default: `false`) |
| `DATABASE_URL` | PostgreSQL connection string (Prisma) |
| `KEYCLOAK_URL` | Keycloak server base URL |
| `KEYCLOAK_REALM` | `quantumbilling` |

---

## UI Story

### Organization Subscription List

Accessible at `/organizations/:orgId/subscriptions` for ORG_ADMIN and SUPER_ADMIN.

**Header:**
- Organization name + "Subscriptions" title
- "Create Subscription" button

**Metric Cards (3-up):**
| Metric | Calculation |
|--------|-------------|
| Active Subscriptions | Count where `status IN ('active', 'trialing')` |
| Total MRR | Sum of MRR from all active subscriptions |
| Next Billing | Earliest upcoming billing date among active subscriptions |

### Subscription Table

| Column | Content |
|--------|---------|
| Plan | Plan name with icon and badge (if trial) |
| Status | Colored badge |
| Billing Period | Monthly / Annual |
| MRR | Formatted currency |
| Start Date | Date |
| Next Billing | Date (blank if canceled) |
| Actions | Change Plan, Cancel (buttons) |

### Create Subscription Flow

**Step 1 — Select Plan:**
- Grid of available plans (cards showing plan name, price, key features)
- Plans show as selectable cards
- Unavailable plans (incompatible, deprecated) are disabled

**Step 2 — Configure:**
- Billing Period: Monthly / Annual toggle
- Start Date: "Immediate" (default) or date picker for future
- Trial: Enable/disable (if plan supports trials), duration selector (7/14/30 days)

**Step 3 — Review & Confirm:**
- Summary: Org name, Plan, MRR, Billing Period, Start Date
- Payment method shown (must have one unless trial)
- "Create Subscription" button

### Change Plan Modal

**Current Plan:** Displayed at top (read-only, highlighted)

**Available Plans:** Grid of other plans

**New Plan Selected:**
- Price difference shown: "+$400/month" or "-$50/month"
- Proration estimate: "~$133 credit on next invoice"

**Apply Options:**
- Radio: "Apply Immediately (with proration)"
- Radio: "Apply at Next Billing Period"

### Cancel Subscription Modal

**Warning:** "Are you sure? This will revoke organization access."

**Reason:** Dropdown (required)
- Customer requested
- Payment failed
- Plan too expensive
- Migrating away
- Other (text field)

**Timing:**
- Radio: "Cancel Immediately" — access revoked now
- Radio: "Cancel at End of Period" — access continues until billing date

---

## Webhooks

| Event | Trigger |
|-------|---------|
| `subscription.created` | New subscription created for org |
| `subscription.activated` | Scheduled/trial subscription becomes active |
| `subscription.updated` | Plan changed or billing period updated |
| `subscription.trialing` | Organization enters trial period |
| `subscription.past_due` | Payment failed, subscription past due |
| `subscription.suspended` | Grace period expired, access restricted |
| `subscription.reactivated` | Suspended subscription reactivated |
| `subscription.canceled` | Subscription canceled (immediate or period end) |

---

## Dependencies & Notes for Agent

- **Subscription = Organization Activation:** The platform must enforce that only organizations with `active` or `trialing` subscriptions can make API calls. Middleware should check subscription status.
- **Proration:** On mid-period plan changes, calculate unused days on current plan as credit. Formula: `(daysRemaining / daysInPeriod) × (newPlanPrice - oldPlanPrice)`.
- **Scheduled Subscriptions:** Cron job activates subscriptions on `start_date`. Until then, org cannot access platform.
- **Cancel at Period End:** Set `cancel_at_period_end = true`. Cron job cancels on `end_date`.
- **Past Due → Suspended:** Payment failure → `past_due`. After `GRACE_PERIOD_DAYS` → `suspended`. Block API access in suspended state.
- **Multi-Subscription:** If `ALLOW_MULTIPLE_SUBSCRIPTIONS = false`, prevent a second active subscription if one exists.
- **Downgrade Guard:** Before downgrade, check current period usage vs. new plan's included units. Block if exceeded.
- **Onboarding Integration:** After org is created, onboarding wizard should check if subscription exists. If not, prompt to create one before org can use platform.
- **MRR Calculation:** Organization MRR = sum of all active subscriptions' MRR. Used in platform analytics.
- **Audit Logging:** Log all subscription state changes with actor, timestamp, old state, new state, and reason.

---

## Future Enhancements (Out of Scope for v1)

- Self-serve subscription management for ORG_ADMIN in portal
- Subscription cloning / templates
- Bulk subscription operations
- Subscription pause (pause without canceling)
- A/B testing subscription offers
- Commitment / contracted subscriptions with auto-renewal
