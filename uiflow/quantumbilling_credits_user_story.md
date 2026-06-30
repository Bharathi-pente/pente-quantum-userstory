# QuantumBilling User Story: Credits Management

**QB-STORY-025** · Sprint 7 · Phase: Billing Core

---

## Title

**Credits Management** — grant, track, and consume organization credits across billing periods

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As an ORG_ADMIN or SUPER_ADMIN**, I want to manage credits for organizations, so that I can grant promotional offers, apply compensation for service issues, track prepaid credit balances, and have credits automatically applied to invoices to offset usage costs.

**As a CUSTOMER**, I want to view my organization's credit balance and transaction history, so that I can understand how credits are being used and what my remaining balance is.

**Core Concept:** Credits are a monetary balance granted to an **Organization**. When usage occurs or invoices are generated, credits are automatically consumed in priority order and offset the amount owed. Credits can expire and can be restricted to specific AI models or general usage.

Key concepts:
- **Credit Grant** = adding a credit balance to an organization
- **Credit Consumption** = automatic application of credits to invoices/usage
- **Priority Order** = lower number = consumed first (Compensation=0 is highest)
- **FEFO** = First Expiring, First Out — credits expiring sooner are consumed first
- **Applicable Scope** = credits can apply to "all" usage or specific models only

---

## Credit Types

| Type | Description | Example | Priority |
|------|-------------|---------|----------|
| **compensation** | Credits granted for service issues or SLA violations | "SLA violation - Nov 2024 incident" | 0 (highest) |
| **promotional** | Free credits from campaigns or marketing offers | "$500 free trial credit" | 1 |
| **prepaid** | Purchased credit packages | "$50,000 prepaid credit package" | 2 |
| **commit** | Allocated from contract commitment | "$100,000 commitment allocation" | 3 |

---

## How Credits Are Consumed

When an invoice is generated or usage is billed:

```
1. Calculate total amount owed
2. Fetch all active credits for the org (status = 'active')
3. Sort credits by:
   a. Priority (ascending — lower number first)
   b. Expiration date (ascending — sooner expiration first) [if FEFO enabled]
4. Apply credits in order until:
   - All credits exhausted, OR
   - Total amount owed is fully offset
5. Record credit ledger entries for each credit used
6. Update remaining balance on each credit
```

**Example:**
- Org has Compensation ($2,500, priority 0) + Promotional ($500, priority 1)
- Invoice = $1,000
- Compensation credits ($2,500) are applied first → $1,000 offset
- Compensation remaining = $1,500, Promotional unchanged

---

## RBAC Roles

| Role | Can view credits | Can grant credits | Can revoke credits | Can edit credits | Scope |
|------|-----------------|------------------|-------------------|-----------------|-------|
| **SUPER_ADMIN** | Yes (all orgs) | Yes (all orgs) | Yes (all orgs) | Yes (all orgs) | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Yes (own org) | Yes (own org) | Yes (own org) | Own org only |
| **CUSTOMER** | Yes (own org, balance + ledger only) | No | No | No | Own org only |
| **END_USER** | No | No | No | No | No access |

---

## Acceptance Criteria

### Credits Dashboard (ORG_ADMIN / SUPER_ADMIN)

1. Credits are accessible at `/organizations/:orgId/credits` for ORG_ADMIN.
2. Header shows "Credits" with subtitle "Manage promotional, prepaid, and compensation credits".
3. **Summary Cards (4-up):**
   - **Total Active Credits** — sum of `remainingAmount` for all active credits
   - **Promotional** — sum of active promotional credits
   - **Prepaid** — sum of active prepaid credits
   - **Expiring Soon** — sum of active credits expiring within 30 days

### Credit List Table

4. A table titled "All Credits" lists all credits with columns:
   - Customer/Organization
   - Type (badge with color)
   - Original Amount
   - Remaining Amount
   - Usage Progress Bar (used/original)
   - Expires
   - Priority (number in circle)
   - Applies To (e.g., "All", "GPT-4 only")
   - Actions (Edit, Revoke)
5. Tab filters: Active Credits, Exhausted, Expired, Priority Rules
6. Credits can be filtered by type, status, and expiration date range.

### Grant Credits

7. "Grant Credits" button opens a modal.
8. Grant Credits Modal fields:
   - **Organization** — dropdown to select org (SUPER_ADMIN) or pre-filled (ORG_ADMIN)
   - **Credit Type** — dropdown: Promotional, Prepaid, Compensation, Commit
   - **Amount** — dollar amount to grant
   - **Expiration Date** — date picker (required)
   - **Priority** — number field (default based on type, lower = higher priority)
   - **Applicable To** — dropdown: All Usage, GPT-4 only, Claude 3 only, etc.
   - **Reason/Notes** — optional text field for internal notes
9. On submit, a credit record is created with `status = 'active'`.

### Edit Credit

10. Clicking "Edit" on a credit row opens the Edit Credit modal.
11. Editable fields: Amount (remaining amount), Expiration Date, Priority, Applicable To, Notes.
12. Cannot edit: Original Amount, Type, Organization.
13. Editing does not change `usedAmount`.

### Revoke Credit

14. Clicking "Revoke" on a credit row opens a confirmation modal.
15. Revoked credits have `status = 'revoked'` and are immediately excluded from usage.
16. `remainingAmount` is set to 0.
17. Revoking is permanent and cannot be undone.

### Credit Consumption (Automatic)

18. When an invoice is generated or usage is billed, credits are automatically applied.
19. Credits are applied in priority order (0 = Compensation, 1 = Promotional, 2 = Prepaid, 3 = Commit).
20. Within the same priority, credits expiring sooner are applied first (FEFO).
21. Credits restricted to specific models (e.g., "GPT-4 only") are only applied to that model's usage.
22. Once a credit's `remainingAmount` reaches 0, its status changes to `exhausted`.
23. When a credit's `expiresAt` date passes, its status changes to `expired`.

### Credit Ledger (Transaction History)

24. Each organization has a credit ledger showing all credit transactions.
25. Ledger columns: Date, Type, Description, Amount (+/-), Balance.
26. Transaction types:
    - `grant` — credits added (+)
    - `usage` — credits consumed by usage (-)
    - `adjustment` — manual adjustment (+/-)
    - `expired` — credits expired (-)
    - `revoked` — credits revoked (-)
27. Ledger is sorted by date descending (newest first).
28. Available to ORG_ADMIN, SUPER_ADMIN, and CUSTOMER.

### Expiration Handling

29. Credits with `status = 'active'` and `expiresAt < today` are automatically marked `expired` by a daily cron job.
30. Expired credits have `remainingAmount` set to 0.
31. **FEFO (First Expiring, First Out)** setting — when enabled, credits expiring sooner are consumed before credits expiring later (within the same priority level).

### Expiration Reminders

32. A setting controls whether expiration reminders are sent.
33. Reminder timing options: 30 days, 14 days, 7 days before expiration.
34. Reminders are sent via email to the organization's billing contact.

### Customer Portal View (CUSTOMER)

35. CUSTOMER can view their organization's credits at `/my-account/credits`.
36. Portal shows:
    - Available Credit Balance
    - Used This Month
    - Expiring Soon
    - Recent Transactions (credit ledger)
37. CUSTOMER cannot grant, edit, or revoke credits.
38. CUSTOMER cannot see internal notes or priority configuration.

---

## Test Cases

### TC-01 — Grant promotional credits

**Given:** org "Acme" has no promotional credits
**When:** ORG_ADMIN grants $500 promotional credits with expiration "2025-06-30"
**Then:** a new credit record is created with type="promotional", originalAmount=500, remainingAmount=500, status="active"
**And:** Total Active Credits increases by $500

---

### TC-02 — Grant compensation credits (higher priority)

**Given:** org "Acme" has $500 promotional credits (priority 1)
**When:** SUPER_ADMIN grants $2,500 compensation credits (priority 0)
**Then:** compensation credits have priority 0 (higher than promotional)
**And:** compensation credits will be consumed before promotional credits

---

### TC-03 — Credits automatically consumed on invoice

**Given:** org "Acme" has $500 promotional credits
**When:** an invoice of $300 is generated
**Then:** $300 is deducted from promotional credits
**And:** remaining credits = $200
**And:** ledger entry: type="usage", amount=-300, balance=200

---

### TC-04 — Credits consumed in priority order

**Given:** org "Acme" has:
  - $2,500 compensation credits (priority 0)
  - $500 promotional credits (priority 1)
**When:** an invoice of $3,000 is generated
**Then:** compensation credits are applied first: -$2,500
**And:** promotional credits are applied next: -$500
**And:** invoice total = $0 (fully offset)

---

### TC-05 — FEFO — expiring credits consumed first

**Given:** org "Acme" has two promotional credits:
  - Credit A: $500, expires in 60 days
  - Credit B: $500, expires in 30 days
**And:** FEFO is enabled
**When:** an invoice of $500 is generated
**Then:** Credit B (expiring sooner) is applied first

---

### TC-06 — Model-restricted credits

**Given:** org "Acme" has $500 credits applicable to "GPT-4 only"
**When:** usage costs: GPT-4 = $300, Claude 3 = $200
**Then:** only $300 of GPT-4 usage is offset by credits
**And:** Claude 3 usage is billed normally

---

### TC-07 — Credit exhausted

**Given:** org "Acme" has $500 promotional credits
**When:** $500 of usage is billed
**Then:** credits are fully consumed
**And:** credit status changes to "exhausted"
**And:** remainingAmount = 0

---

### TC-08 — Credit expired

**Given:** a credit for org "Acme" has expiresAt = yesterday
**When:** the daily expiration cron job runs
**Then:** the credit's status changes to "expired"
**And:** remainingAmount = 0

---

### TC-09 — Revoke credits

**Given:** org "Acme" has $500 promotional credits
**When:** SUPER_ADMIN revokes the credits
**Then:** credit status changes to "revoked"
**And:** remainingAmount = 0
**And:** credits are no longer available for use

---

### TC-10 — Edit credit expiration

**Given:** org "Acme" has a credit expiring on "2025-01-01"
**When:** ORG_ADMIN edits the credit to extend expiration to "2025-06-30"
**Then:** the credit's expiresAt is updated
**And:** credit remains active

---

### TC-11 — CUSTOMER views credit balance in portal

**Given:** CUSTOMER is logged into the portal
**When:** navigating to "My Credits"
**Then:** credit balance, used this month, and expiring soon are shown
**And:** recent transactions (ledger) are visible
**And:** Grant/Edit/Revoke buttons are NOT shown

---

### TC-12 — Partial invoice offset by credits

**Given:** org "Acme" has $200 credits
**And:** an invoice of $500 is generated
**When:** credits are applied
**Then:** $200 of credits are applied
**And:** invoice shows: subtotal $500, credits -$200, total $300 owed

---

### TC-13 — View credit ledger

**Given:** org "Acme" has multiple credit transactions
**When:** ORG_ADMIN views the credit ledger
**Then:** all transactions are listed: grants (+), usage (-), adjustments (+/-), expirations (-), revokes (-)
**And:** running balance is shown for each entry

---

### TC-14 — ORG_ADMIN cannot grant credits to another org

**Given:** ORG_ADMIN for org `acme`
**When:** attempting to grant credits to org `othercorp`
**Then:** 403 `FORBIDDEN`

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/v1/organizations/:orgId/credits` | List all credits for an organization | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `GET` | `/api/v1/credits/:creditId` | Get a specific credit | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `POST` | `/api/v1/organizations/:orgId/credits` | Grant credits to an organization | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `PUT` | `/api/v1/credits/:creditId` | Edit a credit | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `POST` | `/api/v1/credits/:creditId/revoke` | Revoke a credit | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `GET` | `/api/v1/organizations/:orgId/credits/ledger` | Get credit ledger (transaction history) | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` or `CustomerGuard` |
| `GET` | `/api/v1/organizations/:orgId/credits/balance` | Get total credit balance | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` or `CustomerGuard` |
| `POST` | `/api/v1/credits/consume` | Manually consume credits (for testing) | JWT · Guard: `SuperAdminGuard` |
| `GET` | `/api/v1/platform/credits` | List all credits across all orgs (SuperAdmin) | JWT · Guard: `SuperAdminGuard` · Query: `?org_id=&type=&status=` |

---

## Data Tables Used

| Table | Schema | Operation | Key Columns |
|-------|--------|-----------|-------------|
| `credits` | `billing` | SELECT · INSERT · UPDATE | `id, org_id, type, original_amount, remaining_amount, used_amount, expires_at, priority, applicable_to, status, notes, created_at` |
| `credit_ledger` | `billing` | SELECT · INSERT | `id, org_id, credit_id, type, amount, balance, description, event_id, created_at` |
| `organizations` | `identity` | SELECT | `id, name, billing_email` |
| `invoices` | `billing` | SELECT · UPDATE | `id, org_id, subscription_id, credits_applied, total, status` |
| `usage_events` | `billing` | SELECT | `id, org_id, meter_id, cost, model` |
| `subscriptions` | `billing` | SELECT | `id, org_id, status` |

---

## Credit Ledger Entry Types

| Type | Direction | Description |
|------|-----------|-------------|
| `grant` | + (credit) | Credits granted to organization |
| `usage` | - (debit) | Credits consumed by usage/invoice |
| `adjustment` | +/- | Manual adjustment by admin |
| `expired` | - (debit) | Credits expired (automatic) |
| `revoked` | - (debit) | Credits revoked by admin |
| `refunded` | + (credit) | Credits refunded (e.g., overpayment) |

---

## State Machine — Credit Lifecycle

```
┌──────────┐    grant    ┌─────────┐   fully used   ┌───────────┐
│ (none)   │──────────►│  active  │──────────────►│ exhausted │
└──────────┘            └────┬─────┘                └───────────┘
                             │
                             │ expiration date passed
                             ▼
                       ┌──────────┐
                       │  expired │
                       └──────────┘

┌──────────┐    grant    ┌─────────┐    revoked    ┌─────────┐
│ (none)   │──────────►│  active  │──────────────►│ revoked │
└──────────┘            └─────────┘                └─────────┘
```

### Cron Jobs

1. **`credit-expiration-checker`** — runs daily: marks credits as `expired` where `expires_at < today` and `status = active`
2. **`credit-reminder-sender`** — runs daily: sends expiration reminders based on configured lead time (30/14/7 days)

---

## Error Codes

| Code | HTTP | Trigger |
|------|------|---------|
| `CREDIT_NOT_FOUND` | 404 | `creditId` does not exist |
| `ORGANIZATION_NOT_FOUND` | 404 | `orgId` does not exist |
| `CREDIT_ALREADY_REVOKED` | 409 | Attempting to revoke an already revoked credit |
| `CREDIT_ALREADY_EXHAUSTED` | 409 | Attempting to use an exhausted credit |
| `CREDIT_ALREADY_EXPIRED` | 409 | Attempting to use an expired credit |
| `INVALID_AMOUNT` | 422 | Amount must be greater than 0 |
| `INVALID_EXPIRATION` | 422 | Expiration date must be in the future |
| `INVALID_PRIORITY` | 422 | Priority must be between 0 and 3 |
| `FORBIDDEN` | 403 | Actor not authorized for this organization's credits |
| `UNAUTHORIZED` | 401 | No valid JWT token |

---

## Environment Config Keys

| Key | Description |
|-----|-------------|
| `DEFAULT_CREDIT_PRIORITY_{TYPE}` | Default priority per credit type (compensation=0, promotional=1, prepaid=2, commit=3) |
| `FEFO_ENABLED` | Enable First Expiring, First Out consumption (default: `true`) |
| `EXPIRATION_REMINDER_ENABLED` | Send expiration reminder emails (default: `true`) |
| `EXPIRATION_REMINDER_DAYS` | Days before expiration to send reminder (default: `14`) |
| `MAX_CREDIT_AMOUNT` | Maximum credits that can be granted at once (default: `1000000`) |
| `CREDIT_CONSUMPTION_BATCH_SIZE` | Number of credits to process per consumption run (default: `100`) |
| `CREDIT_LEDGER_RETENTION_DAYS` | Days to retain ledger entries (default: `2555` / 7 years) |
| `DATABASE_URL` | PostgreSQL connection string (Prisma) |
| `KEYCLOAK_URL` | Keycloak server base URL |
| `KEYCLOAK_REALM` | `quantumbilling` |

---

## UI Story

### Credits Dashboard (ORG_ADMIN / SUPER_ADMIN)

Accessible at `/organizations/:orgId/credits`.

**Header:**
- "Credits" title
- "Grant Credits" button

**Metric Cards (4-up):**
| Metric | Calculation | Icon | Color |
|--------|-------------|------|-------|
| Total Active Credits | SUM(remainingAmount) where status='active' | creditCard | green |
| Promotional | SUM(remainingAmount) where type='promotional' AND status='active' | gift | purple |
| Prepaid | SUM(remainingAmount) where type='prepaid' AND status='active' | wallet | cyan |
| Expiring Soon | SUM(remainingAmount) where expiresAt < 30 days AND status='active' | clock | amber |

**Tab Filters:** Active Credits | Exhausted | Expired | Priority Rules

**Credits Table:**
| Column | Content |
|--------|---------|
| Organization | Org name |
| Type | Colored badge |
| Original | Original amount |
| Remaining | Current balance (green if > 0) |
| Usage | Progress bar |
| Expires | Expiration date (amber if < 30 days) |
| Priority | Number in circle |
| Applies To | "All" or specific model |
| Actions | Edit, Revoke |

### Priority Rules Tab

**Priority Order Display:**
- Priority 0: Compensation Credits — "Applied first to offset service issues"
- Priority 1: Promotional Credits — "Free credits from campaigns"
- Priority 2: Prepaid Credits — "Purchased credit packages"
- Priority 3: Commit Credits — "Contract commitment allocations"

**Expiration Policy Settings:**
- FEFO toggle: "Consume expiring credits first"
- Reminder toggle: "Send expiration reminders"
- Reminder timing dropdown: 30/14/7 days

### Grant Credits Modal

**Fields:**
| Field | Type | Required | Notes |
|-------|------|----------|-------|
| Organization | Dropdown | Yes | Pre-filled for ORG_ADMIN |
| Credit Type | Dropdown | Yes | Promotional/Prepaid/Compensation/Commit |
| Amount | Number | Yes | Dollar amount |
| Expiration Date | Date Picker | Yes | Must be future date |
| Priority | Number | No | Default by type |
| Applicable To | Dropdown | Yes | All / specific model |
| Notes | Text | No | Internal notes |

### Customer Portal Credits View (CUSTOMER)

Accessible at `/my-account/credits`.

**Summary Cards:**
| Metric | Value |
|--------|-------|
| Available | Current credit balance |
| Used This Month | Credits consumed this month |
| Expiring Soon | Credits expiring within 30 days |

**Recent Transactions Table:**
| Date | Type | Description | Amount | Balance |
|------|------|-------------|--------|---------|
| Dec 25 | usage | Daily usage - Dec 25 | -$1,250 | $20,600 |
| Dec 24 | usage | Daily usage - Dec 24 | -$1,180 | $21,850 |
| Dec 22 | adjustment | Promotional credit applied | +$500 | $24,370 |

---

## Webhooks

| Event | Trigger |
|-------|---------|
| `credit.granted` | Credits granted to an organization |
| `credit.consumed` | Credits consumed by usage/invoice |
| `credit.expired` | Credits automatically expired |
| `credit.revoked` | Credits manually revoked |
| `credit.reminder` | Expiration reminder sent |

---

## Dependencies & Notes for Agent

- **Credit Consumption Logic:** When an invoice is generated, credits are consumed automatically before payment is attempted. The consumption engine must respect priority order and FEFO.
- **Applicable Scope:** When a credit is restricted to a specific model (e.g., "GPT-4 only"), it can only offset usage costs for that model. Calculate: for each credit, if `applicable_to` matches the model's usage, apply credit.
- **Partial Consumption:** If a credit has $500 remaining and $300 needs to be consumed, apply $300, update `remainingAmount` to $200, and mark as `exhausted` only when `remainingAmount` reaches 0.
- **Expiration Cron:** Run daily to check `expires_at < now()` for active credits and mark as `expired`.
- **Reminder Cron:** Run daily to check for credits expiring within `EXPIRATION_REMINDER_DAYS` and send email notifications.
- **Credit Ledger:** Every credit transaction (grant, usage, adjustment, expiration, revoke) must create a ledger entry for audit and reconciliation.
- **Org-Level Credits:** Credits belong to the Organization, not the Subscription. Multiple subscriptions under the same org share the same credit pool.
- **Invoice Integration:** When generating an invoice, first calculate the credit offset, then bill only the remaining amount.
- **RBAC:** ORG_ADMIN can only manage credits for their own org. SUPER_ADMIN can manage all.
- **Audit Logging:** Log all credit operations (grant, edit, revoke, consume) with actor, timestamp, credit ID, and amount.

---

## Future Enhancements (Out of Scope for v1)

- Credit transfers between organizations
- Credit packages (pre-set credit bundles with discounts)
- Credit refunds
- Self-serve credit purchase (buy credits directly)
- Credit usage alerts (notify when balance drops below threshold)
- Credit sharing within organization (team-based budgets)
- Credit rollover (extend expiration on active credits)
- Bulk credit operations (grant to multiple orgs at once)
