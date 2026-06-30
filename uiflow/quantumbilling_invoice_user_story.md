# QuantumBilling User Story: Invoice Management

**QB-STORY-023** · Sprint 7 · Phase: Billing Core

---

## Title

**Invoice Management** — generate, view, and manage invoices for organization subscriptions

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As an ORG_ADMIN or SUPER_ADMIN**, I want to generate, view, and manage invoices for organization subscriptions, so that I can bill organizations for their platform usage based on their active subscriptions, track payment status, and handle overdue payments through the dunning process.

**Core Model:** Invoices are generated **per Subscription** and belong to an **Organization**. Each active subscription generates its own invoice at the start of each billing period. An organization may have multiple active subscriptions, each generating separate invoices.

Key concepts:
- **Invoice** is tied to a **Subscription** and an **Organization**
- **One subscription = one recurring invoice per billing period**
- Invoice line items include: usage-based charges (meters) + plan base fee + overage + credits applied
- Invoice statuses: `draft` → `pending` → `paid` / `overdue` → `canceled` / `voided`
- Credit memos can be applied to offset invoice amounts
- Organizations can view and pay their invoices in the customer portal

---

## Relationship Model

```
Organization
  └── Subscription 1 ──► Invoice (monthly billing period)
  │                      ├── Line items from usage
  │                      ├── Plan base fee
  │                      └── Credits applied
  │
  └── Subscription 2 ──► Invoice (monthly billing period)
                         └── Line items from usage
                         └── Plan base fee
```

**Note:** An organization with 2 active subscriptions receives **2 separate invoices** per billing cycle.

---

## RBAC Roles

| Role | Can view invoices | Can create invoices | Can void invoices | Can pay invoices | Scope |
|------|-----------------|--------------------|--------------------|-----------------|-------|
| **SUPER_ADMIN** | Yes (all orgs) | Yes (all orgs) | Yes (all orgs) | N/A | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Yes (own org) | Yes (own org) | Yes (via portal) | Own org only |
| **CUSTOMER** | Yes (own org) | No | No | Yes (pay own invoices) | Own org only |
| **END_USER** | No | No | No | No | No access |

---

## Invoice States

```
┌─────────┐    generate    ┌──────────┐    auto-pay /     ┌────┐
│  draft  │──────────────►│ pending  │◄─── manual pay ───▶│paid│
└─────────┘               └────┬─────┘                   └────┘
                                │                              ▲
                                │ dunning                      │
                                ▼                              │
                           ┌──────────┐                       │
                           │ overdue  │───────────────────────┘
                           └────┬─────┘    (credit/resolve)
                                │
                          void / write-off
                                │
                                ▼
                          ┌──────────┐
                          │ voided   │
                          └──────────┘
```

**State Definitions:**
| State | Description |
|-------|-------------|
| `draft` | Invoice generated but not yet finalized/sent |
| `pending` | Invoice finalized and sent to customer, awaiting payment |
| `paid` | Payment received, invoice closed |
| `overdue` | Payment past due date, dunning process activated |
| `voided` | Invoice canceled/voided (no payment will be collected) |

---

## Acceptance Criteria

### Invoice Generation

1. An invoice is automatically generated when a subscription's billing period begins.
2. Invoice is tied to: `subscription_id`, `org_id`, `billing_period_start`, `billing_period_end`.
3. Invoice includes line items:
   - **Usage charges**: each meter used during the period (e.g., GPT-4 Input Tokens: 156M × $0.000025 = $3,900)
   - **Plan base fee**: fixed amount from the plan (e.g., Enterprise: $499/month)
   - **Overage charges**: usage beyond included units (e.g., +9443.80 for exceeding limits)
   - **Credits applied**: promotional credits or proration credits deducted
4. Invoice number format: `INV-{YYYY}-{MM}-{###}` (e.g., `INV-2026-01-001`).
5. Invoice includes: issue date, due date, subtotal, tax (if applicable), credits, total amount.
6. Organization must have at least one active payment method to generate invoices.
7. If payment method fails on auto-pay, invoice status becomes `overdue`.

### Invoice List View (ORG_ADMIN / SUPER_ADMIN)

8. Invoice list shows all invoices for the organization (or all orgs for SUPER_ADMIN).
9. List columns: Invoice #, Subscription/Plan, Issue Date, Due Date, Amount, Status, Actions.
10. Status filter tabs: All, Draft, Pending, Paid, Overdue.
11. Organization filter (SUPER_ADMIN only): view invoices by organization.
12. Date range filter for invoice search.
13. Each row is clickable → opens invoice detail.

### Invoice Detail View

14. Invoice header: Invoice number, status badge, issue date, due date.
15. Organization name and subscription/plan name displayed.
16. Line items table: Description, Meter/Category, Quantity, Unit Price, Amount.
17. Summary section: Subtotal, Credits (negative), Tax (rate %), Tax Amount, **Total**.
18. Payment information: payment method used (if paid), payment date.
19. Notes field (optional internal notes).
20. Actions: Download PDF, Send Reminder (if overdue), Void (if pending/overdue).

### Invoice PDF

21. Invoice can be downloaded as a PDF.
22. PDF includes: company header, invoice number, org details, line items, totals, payment instructions.
23. PDF follows a professional invoice template.

### Manual Invoice Creation

24. ORG_ADMIN can manually create an invoice (not tied to subscription billing cycle).
25. Manual invoice requires: organization (required), line items (required), due date (required), notes (optional).
26. Manual invoices are marked as `draft` until finalized and sent.

### Payment Recording

27. When payment is received, invoice status changes to `paid`.
28. Payment is recorded with: amount paid, payment method, payment date.
29. Partial payments are supported (invoice remains `pending` until fully paid).
30. Overpayment creates a credit balance on the organization.

### Void Invoice

31. Only `pending` or `overdue` invoices can be voided.
32. Voiding an invoice cancels it — no payment will be collected.
33. Voided invoices cannot be un-voided.
34. Voiding triggers webhook `invoice.voided`.

### Overdue & Dunning

35. When an invoice's due date passes without full payment, status changes to `overdue`.
36. Dunning process automatically sends reminder emails per the dunning schedule.
37. ORG_ADMIN can manually send payment reminders.
38. After max dunning attempts, invoice is written off (voided or escalated).

### Organization Portal View (CUSTOMER)

39. CUSTOMER can view all invoices for their organization in the portal.
40. Portal shows: invoice number, amount, status, due date, PDF download.
41. CUSTOMER can pay pending invoices via the portal (credit card or ACH).
42. CUSTOMER cannot void or edit invoices.

---

## Test Cases

### TC-01 — Auto-generate invoice at billing period start

**Given:** org "Acme" has an active Pro subscription with billing date of Jan 1
**When:** the billing processor runs on Jan 1
**Then:** an invoice is generated for subscription "Pro"
**And:** invoice number = "INV-2026-01-001"
**And:** line items include: usage charges from the period + $99 plan base fee
**And:** status = `pending`

---

### TC-02 — Invoice includes usage line items

**Given:** org "Acme" used 156M GPT-4 input tokens and 78M output tokens in the billing period
**When:** invoice is generated
**Then:** line items include:
  - GPT-4 Input Tokens: 156,000,000 × $0.000025 = $3,900
  - GPT-4 Output Tokens: 78,000,000 × $0.00005 = $3,900
**And:** additional line items for plan base fee and any overage

---

### TC-03 — Credits applied to invoice

**Given:** org "Acme" has $500 in promotional credits
**When:** invoice is generated with total of $5,000
**Then:** credits of $500 are applied
**And:** total = $4,500
**And:** credit balance is reduced by $500

---

### TC-04 — View invoice detail

**Given:** ORG_ADMIN is viewing the invoice list
**When:** clicking on invoice "INV-2026-01-001"
**Then:** invoice detail shows: all line items, subtotal, credits, tax, total, payment status

---

### TC-05 — Download invoice PDF

**Given:** ORG_ADMIN is viewing invoice "INV-2026-01-001"
**When:** clicking "Download PDF"
**Then:** a PDF file is downloaded with the invoice details

---

### TC-06 — Record manual payment

**Given:** invoice "INV-2026-01-001" for org "Acme" is pending
**When:** ORG_ADMIN records a payment of $4,500 via credit card
**Then:** invoice status changes to `paid`
**And:** payment record is created with: amount, method, date

---

### TC-07 — Partial payment

**Given:** invoice "INV-2026-01-001" has total of $5,000
**When:** customer pays $2,000
**Then:** invoice remains `pending`
**And:** remaining balance = $3,000
**And:** a payment record is created for $2,000

---

### TC-08 — Invoice becomes overdue

**Given:** invoice "INV-2026-01-001" has due date of Jan 31 and is still unpaid
**When:** the due date passes
**Then:** invoice status changes to `overdue`
**And:** dunning process begins (emails sent per schedule)

---

### TC-09 — Void overdue invoice

**Given:** invoice "INV-2026-01-001" is overdue
**When:** SUPER_ADMIN voids the invoice
**Then:** invoice status changes to `voided`
**And:** no further collection attempts are made

---

### TC-10 — Multiple subscriptions = multiple invoices

**Given:** org "Acme" has 2 active subscriptions: Pro ($99) and GPU Add-on ($199)
**When:** billing period begins
**Then:** 2 invoices are generated: one for Pro, one for GPU Add-on

---

### TC-11 — CUSTOMER views own invoices in portal

**Given:** CUSTOMER is logged into the customer portal
**When:** navigating to "Invoices"
**Then:** only invoices for their organization are visible
**And:** they can download PDF but cannot void or edit

---

### TC-12 — Pay invoice in portal

**Given:** CUSTOMER has a pending invoice for $4,500
**When:** clicking "Pay Now" and entering credit card details
**Then:** payment is processed
**And:** invoice status changes to `paid`

---

### TC-13 — SUPER_ADMIN views all org invoices

**Given:** SUPER_ADMIN navigates to the platform invoices view
**When:** viewing `/platform/invoices`
**Then:** all invoices across all organizations are shown
**And:** can filter by organization, status, date range

---

### TC-14 — Manual invoice creation

**Given:** org "Acme" needs a one-time custom charge
**When:** ORG_ADMIN creates a manual invoice with line item "Consulting Services - $1,500"
**Then:** invoice is created in `draft` status
**And:** ORG_ADMIN can finalize and send it

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/v1/organizations/:orgId/invoices` | List all invoices for an organization | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `GET` | `/api/v1/invoices/:invoiceId` | Get invoice detail | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `POST` | `/api/v1/organizations/:orgId/invoices` | Create manual invoice | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `POST` | `/api/v1/invoices/:invoiceId/generate` | Trigger invoice generation for a subscription | JWT · Guard: `SuperAdminGuard` |
| `POST` | `/api/v1/invoices/:invoiceId/void` | Void an invoice | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `POST` | `/api/v1/invoices/:invoiceId/pay` | Record payment for an invoice | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `GET` | `/api/v1/invoices/:invoiceId/pdf` | Download invoice as PDF | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` or `CustomerGuard` |
| `POST` | `/api/v1/invoices/:invoiceId/send` | Send invoice to customer | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `POST` | `/api/v1/invoices/:invoiceId/remind` | Send payment reminder | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `GET` | `/api/v1/platform/invoices` | List all invoices (SuperAdmin) | JWT · Guard: `SuperAdminGuard` · Query: `?org_id=&status=&date_from=&date_to=` |

---

## Data Tables Used

| Table | Schema | Operation | Key Columns |
|-------|--------|-----------|-------------|
| `invoices` | `billing` | SELECT · INSERT · UPDATE | `id, number, org_id, subscription_id, status, issue_date, due_date, paid_date, subtotal, credits, tax_rate, tax_amount, total, currency, payment_method_id` |
| `invoice_line_items` | `billing` | SELECT · INSERT | `id, invoice_id, description, meter_id, quantity, unit_price, amount, model` |
| `subscriptions` | `billing` | SELECT | `id, org_id, plan_id, status, billing_period, mrr` |
| `organizations` | `identity` | SELECT | `id, name, billing_email` |
| `plans` | `catalog` | SELECT | `id, name, base_price` |
| `payments` | `billing` | INSERT · SELECT | `id, invoice_id, amount, method, status, paid_at` |
| `credits` | `billing` | UPDATE | `id, org_id, remaining_amount` |
| `usage_events` | `billing` | SELECT · SUM | `id, subscription_id, meter_id, value, event_timestamp` |

---

## Invoice Line Item Types

| Type | Source | Example |
|------|--------|---------|
| **Usage-based** | Meter readings from `usage_events` | GPT-4 Input Tokens: 156M × $0.000025 = $3,900 |
| **Plan base fee** | Plan's `base_price` | Enterprise Plan: $499/month |
| **Overage** | Usage beyond plan limits | Overage Usage: $9,443.80 |
| **Proration credit** | Credit from mid-period plan change | Proration Credit: -$133.00 |
| **Promotional credit** | Applied credits from `credits` table | Credits Applied: -$500.00 |
| **Custom charge** | Manual invoice line item | Consulting Services: $1,500 |

---

## State Machine — Invoice Lifecycle

### State Transitions

| From State | To State | Trigger |
|-----------|----------|---------|
| (none) | `draft` | Manual invoice created |
| `draft` | `pending` | Invoice finalized/sent |
| `draft` | `voided` | Manual void before send |
| `pending` | `paid` | Payment received (full) |
| `pending` | `overdue` | Due date passed without full payment |
| `pending` | `voided` | Manual void |
| `overdue` | `paid` | Payment received (full) |
| `overdue` | `voided` | Write-off after dunning fails |

### Cron Jobs

1. **`invoice-generator`** — runs daily: generates invoices for subscriptions where billing date = today
2. **`overdue-checker`** — runs daily: marks invoices as `overdue` if `due_date < today` and `status = pending`
3. **`dunning-processor`** — runs per dunning schedule: sends reminder emails, escalates after max attempts

---

## Error Codes

| Code | HTTP | Trigger |
|------|------|---------|
| `INVOICE_NOT_FOUND` | 404 | `invoiceId` does not exist |
| `ORGANIZATION_NOT_FOUND` | 404 | `orgId` does not exist |
| `SUBSCRIPTION_NOT_FOUND` | 404 | `subscriptionId` does not exist |
| `INVOICE_ALREADY_PAID` | 409 | Attempting to void a paid invoice |
| `INVOICE_ALREADY_VOIDED` | 409 | Attempting to void an already voided invoice |
| `INVALID_LINE_ITEMS` | 422 | No line items provided for manual invoice |
| `PAYMENT_FAILED` | 500 | Payment processing failed |
| `FORBIDDEN` | 403 | Actor not authorized for this organization's invoices |
| `UNAUTHORIZED` | 401 | No valid JWT token |

---

## Environment Config Keys

| Key | Description |
|-----|-------------|
| `INVOICE_NUMBER_PREFIX` | Prefix for invoice numbers (default: `INV`) |
| `DEFAULT_PAYMENT_TERMS_DAYS` | Default due date offset in days (default: `30`) |
| `AUTO_GENERATE_INVOICES` | Enable automatic invoice generation at billing period start (default: `true`) |
| `AUTO_CHARGE_PAYMENT_METHOD` | Automatically charge payment method on due date (default: `true`) |
| `DEFAULT_TAX_RATE` | Default tax rate applied to invoices (default: `0`) |
| `OVERDUE_CHECK_HOUR` | Hour of day to run overdue check (default: `0`) |
| `DUNNING_SCHEDULE_ID` | ID of the dunning schedule to apply to overdue invoices |
| `INVOICE_STORAGE_PATH` | Path for storing invoice PDFs (local or S3) |
| `INVOICE_EMAIL_FROM` | From email address for invoice emails |
| `DATABASE_URL` | PostgreSQL connection string (Prisma) |
| `KEYCLOAK_URL` | Keycloak server base URL |
| `KEYCLOAK_REALM` | `quantumbilling` |

---

## UI Story

### Invoice List (ORG_ADMIN / SUPER_ADMIN)

Accessible at `/organizations/:orgId/invoices` for ORG_ADMIN; `/platform/invoices` for SUPER_ADMIN.

**Header:**
- "Invoices" title
- "Create Manual Invoice" button (optional)
- "Export" button

**Metric Cards (4-up):**
| Metric | Calculation |
|--------|-------------|
| Total Outstanding | SUM of `total` where `status IN (pending, overdue)` |
| Paid This Month | SUM of `total` where `status = paid` AND `paid_date` this month |
| Overdue Count | COUNT where `status = overdue` |
| Avg Payment Time | Average days between issue date and paid date |

**Filters:**
- Status tabs: All, Draft, Pending, Paid, Overdue
- Date range picker
- Organization dropdown (SUPER_ADMIN only)

**Invoice Table:**
| Column | Content |
|--------|---------|
| Invoice # | Invoice number (clickable) |
| Subscription/Plan | Plan name or "One-time" |
| Issue Date | Date |
| Due Date | Date |
| Amount | Total with currency |
| Status | Colored badge |
| Actions | View, Download PDF, Send, Void |

### Invoice Detail

**Header Section:**
- Invoice number (large)
- Status badge
- Issue date, Due date
- Org name

**Line Items Table:**
| Description | Meter/Model | Qty | Unit Price | Amount |
|------------|-------------|-----|------------|--------|
| GPT-4 Input Tokens | Input Tokens / GPT-4 | 156,000,000 | $0.000025 | $3,900 |
| Enterprise Plan - Base Fee | — | 1 | $499 | $499 |
| Overage Usage | — | 1 | $9,443.80 | $9,443.80 |

**Summary:**
| | |
|-|---|
| Subtotal | $13,842.80 |
| Credits | -$500.00 |
| Tax (9%) | $1,199.35 |
| **Total** | **$14,542.15** |

**Actions:**
- Download PDF
- Send Reminder (if overdue)
- Void (if pending/overdue)

### Customer Portal Invoice View (CUSTOMER)

**Invoice List:**
- Invoice #, Amount, Status, Due Date
- PDF download button
- "Pay Now" button for pending invoices

**Invoice Detail:**
- Read-only view of invoice
- "Pay Now" button for pending invoices
- No void or edit options

---

## Webhooks

| Event | Trigger |
|-------|---------|
| `invoice.draft` | Draft invoice created |
| `invoice.created` | Invoice finalized (from draft or auto-generated) |
| `invoice.sent` | Invoice sent to customer |
| `invoice.paid` | Full payment received |
| `invoice.overdue` | Invoice became overdue (past due date) |
| `invoice.voided` | Invoice was voided |
| `invoice.reminder_sent` | Payment reminder sent |

---

## Dependencies & Notes for Agent

- **Invoice Generation:** Triggered by subscription billing period. Query `usage_events` for the period, aggregate by meter, multiply by rate from pricing model.
- **Line Item Calculation:** For each meter used: `SUM(value) * rate_per_unit` from the plan's pricing model.
- **Tax Calculation:** Applied to subtotal after credits. Formula: `(subtotal - credits) * tax_rate`.
- **Credit Application:** Automatically apply available credits from `credits` table. Prioritize by expiration date (earliest first).
- **Proration Credits:** When subscription plan changes mid-period, calculate proration and add as a credit line item.
- **Invoice PDF:** Generated using a template (e.g., PDFKit, Puppeteer, or dedicated service). Store the file and serve via `/invoices/:id/pdf`.
- **Multiple Subscriptions:** Each subscription generates its own invoice. If org has 2 subscriptions, they get 2 invoices per billing cycle.
- **Payment Recording:** When payment is received, update invoice status and create a `payments` record.
- **Dunning Integration:** Overdue invoices feed into the dunning process (see Dunning user story QB-STORY-012).
- **Audit Logging:** Log all invoice state changes with actor, timestamp, old state, new state.
- **Access Control:** ORG_ADMIN can only see invoices for their own org. SUPER_ADMIN sees all.

---

## Future Enhancements (Out of Scope for v1)

- Consolidated invoices (one invoice per org, multiple subscriptions combined)
- Custom invoice templates/branding
- Multi-currency support
- Automatic payment retry with exponential backoff
- Invoice approval workflow (internal review before sending)
- Credit notes (refunds or billing corrections)
- Partial refunds
- Payment plan / installments
- E-invoicing (ISO 20022 format for enterprise customers)
