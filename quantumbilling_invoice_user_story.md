# QuantumBilling User Story: Invoice

## QB-STORY-008 — Sprint 2 — Phase: Feature

---

## Invoice — generate, view, and manage invoices for customers

**Badges**

| Domain | Tags |
|---|---|
| Backend | UI | Auth / RBAC | Billing Engine | Priority: P0 |

---

## Description

**As a ORG_ADMIN, I want to generate invoices based on my customers' subscriptions and usage, apply credits, track payment status, and send invoice documents to customers, so that the billing cycle runs smoothly and customers receive clear, accurate bills.**

Invoices are generated from: subscription base fees + usage from `catalog.meters` + prorated adjustments + discounts.

Key capabilities:
- **Invoice batches** (`billing.invoice_batches`): batch invoice generation runs — processes all ACTIVE subscriptions for a billing period
- **Line items** (`billing.invoice_line_items`): each invoice has line items with description, amount, and linked `meter_id`
- **Invoice status** (from `invoice_status` enum): `draft` → `pending` → `issued` → `past_due` / `paid` → `voided` / `credited`
- **Invoice amount**: total before credits; `credits_applied` shows credits used; net due = `amount` - `credits_applied`
- **Credits** from `billing.credits` can be applied manually or auto-applied to invoices
- **Credit notes** (`billing.credit_notes`) are created when credits exceed invoice amount or for adjustments
- **Invoice documents** (`billing.invoice_documents`): PDF storage path per invoice
- **Tax calculation**: `billing.tax_calculation_audit` tracks per-region tax applied per invoice
- **Invoice reminders**: `billing.invoice_reminder_schedules` sends automated reminders (email) at configurable day offsets
- **Org branding** (`billing.branding_configs`) applied to invoice PDF: logo, colors, footer, T&C
- **VOID**: cancels an invoice (must be before payment); creates a reversal credit note
- **SUPER_ADMIN** can manage any org's invoices

---

## RBAC Roles

| Role | Can Generate | Can Issue / Void | Can Apply Credit | Can View | Scope |
|---|---|---|---|---|---|
| `SUPER_ADMIN` | Yes | Yes | Yes | All orgs | Platform-wide |
| `ORG_ADMIN` | Yes | Yes | Yes | Own org | Own org only |
| `CUSTOMER` | No | No | No | Own invoices | Own account only |
| `END_USER` | No | No | No | No access | No access |

---

## Acceptance Criteria

1. ORG_ADMIN can trigger a batch invoice run for a billing period via `POST /api/v1/invoices/batch`; the system generates one invoice per active contract, creating line items from `billing.invoice_line_items` (base fees + usage from `catalog.meters` + prorated adjustments).
2. Each generated invoice is created with status `DRAFT`, a unique `invoice_number`, amount = sum of line items, and `credits_applied = 0`.
3. `billing.invoice_batches` record is created with `status = RUNNING`, then updated to `COMPLETED` or `FAILED` when the batch finishes; includes `billing_period_start` and `billing_period_end`.
4. `billing.invoice_status_history` records every status transition with `changed_at`, `changed_by`, `old_status`, and `new_status`.
5. ORG_ADMIN can list invoices with pagination (default 20/page) filtered by `status` and/or `customer_id` via `GET /api/v1/invoices`.
6. `POST /api/v1/invoices/:invoiceId/issue` transitions DRAFT or PENDING invoice to ISSUED; creates an invoice document PDF using `billing.branding_configs` (logo, primary_color, footer_text, terms_conditions); stores path in `billing.invoice_documents`.
7. `POST /api/v1/invoices/:invoiceId/void` cancels DRAFT or ISSUED invoices; creates a reversal credit note in `billing.credit_notes`; transitions status to VOID. Cannot void PAID, CREDITED, or already VOID invoices.
8. `POST /api/v1/invoices/:invoiceId/apply-credit` applies available credit from `billing.credits` to the invoice; updates `credits_applied` on the invoice; creates or updates `billing.credit_notes` if credit exceeds invoice net due.
9. `billing.tax_calculation_audit` stores per-invoice tax breakdown: `tax_region_id`, `taxable_amount`, `tax_rate`, `tax_amount`, `tax_type`; retrievable via `GET /api/v1/invoices/:invoiceId/tax-breakdown`.
10. `billing.invoice_reminder_schedules` controls automated reminder emails; `POST /api/v1/invoices/:invoiceId/reminders` triggers a manual reminder; reminders only sent for ISSUED invoices that are overdue.

---

## Test Cases

### TC-01 — Happy path: batch invoice generation

**Given:** ORG_ADMIN for org `acme`, billing period 2026-06-01 to 2026-06-30, 3 active contracts with usage data in `catalog.meters`

**When:** `POST /api/v1/invoices/batch` with `{ "billing_period_start": "2026-06-01", "billing_period_end": "2026-06-30" }`

**Then:**
- `billing.invoice_batches` row created with status `RUNNING`
- 3 invoices created in `billing.invoices` with status `DRAFT`
- Each invoice has line items in `billing.invoice_line_items` sourced from meters
- Batch status updated to `COMPLETED`
- 201 returned with `{ batch_id, invoice_ids: [...] }`

---

### TC-02 — Happy path: issue an invoice

**Given:** DRAFT invoice `inv_001` exists for customer `cust_abc`, org `acme`

**When:** `POST /api/v1/invoices/inv_001/issue`

**Then:**
- Invoice status transitions DRAFT → ISSUED
- `billing.invoice_documents` row created with `document_type = INVOICE_PDF`, `file_name = "inv_001.pdf"`, storage path populated
- `billing.invoice_status_history` row inserted
- 200 returned with `{ invoice_id, invoice_number, status: "ISSUED", document_url }`

---

### TC-03 — Happy path: apply credit to invoice

**Given:** ISSUED invoice `inv_001` with `amount = 500.00`, `credits_applied = 0.00`; customer `cust_abc` has `credit_balance = 150.00` in `billing.credits`

**When:** `POST /api/v1/invoices/inv_001/apply-credit` with `{ "credit_amount": 150.00 }`

**Then:**
- Invoice `credits_applied` updated to `150.00`, net due = `350.00`
- `billing.credits` balance reduced by `150.00`
- `billing.credit_notes` row created if credit exceeds net due
- `billing.invoice_status_history` row inserted
- 200 returned

---

### TC-04 — Happy path: void an invoice

**Given:** ISSUED invoice `inv_001` (not yet paid), org `acme`

**When:** `POST /api/v1/invoices/inv_001/void`

**Then:**
- Invoice status transitions ISSUED → VOID
- Reversal credit note created in `billing.credit_notes` for full invoice amount
- `billing.invoice_status_history` row inserted
- 200 returned with `{ invoice_id, status: "VOID", credit_note_id }`

---

### TC-05 — Negative: void a paid invoice

**Given:** Invoice `inv_001` has status `PAID`

**When:** `POST /api/v1/invoices/inv_001/void`

**Then:**
- 409 `INVOICE_NOT_VOIDABLE` — paid invoices cannot be voided
- No state change

---

### TC-06 — Negative: issue an already issued invoice

**Given:** Invoice `inv_001` has status `ISSUED`

**When:** `POST /api/v1/invoices/inv_001/issue`

**Then:**
- 409 `INVOICE_ALREADY_ISSUED`
- No state change

---

### TC-07 — Negative: apply credit exceeding available balance

**Given:** Customer `cust_abc` has `credit_balance = 50.00`; invoice `inv_001` has net due `200.00`

**When:** `POST /api/v1/invoices/inv_001/apply-credit` with `{ "credit_amount": 100.00 }`

**Then:**
- 422 `INSUFFICIENT_CREDIT_BALANCE`
- Invoice `credits_applied` unchanged

---

### TC-08 — Negative: customer cannot access another customer's invoice

**Given:** Customer `cust_abc` is authenticated; invoice `inv_001` belongs to customer `cust_xyz`

**When:** `GET /api/v1/invoices/inv_001`

**Then:**
- 403 `FORBIDDEN`
- Invoice not returned

---

### TC-09 — Negative: batch with no active contracts

**Given:** Org `acme` has zero ACTIVE contracts for billing period 2026-07-01 to 2026-07-31

**When:** `POST /api/v1/invoices/batch` with `{ "billing_period_start": "2026-07-01", "billing_period_end": "2026-07-31" }`

**Then:**
- `billing.invoice_batches` row created with status `COMPLETED`, `invoice_count = 0`
- 200 returned with `{ batch_id, invoice_ids: [] }`

---

### TC-10 — Negative: issue invoice without branding config

**Given:** Org `acme` has no row in `billing.branding_configs`; DRAFT invoice `inv_001` exists

**When:** `POST /api/v1/invoices/inv_001/issue`

**Then:**
- 422 `BRANDING_CONFIG_MISSING` — invoice document cannot be generated without org branding config

---

## API Endpoints

| Method | Path | Description | Auth |
|---|---|---|---|
| `POST` | `/api/v1/invoices/batch` | Run invoice batch for a billing period — generates DRAFT invoices for all ACTIVE contracts | JWT · Guard: `OrgAdminGuard` |
| `GET` | `/api/v1/invoices` | List invoices for org — paginated, filterable by `status` and `customer_id` | JWT · Guard: `OrgAdminGuard` · Query: `?status=ISSUED&customer_id=cust_abc&page=1&limit=20` |
| `GET` | `/api/v1/invoices/:invoiceId` | Get invoice with line items — returns invoice + `billing.invoice_line_items` array | JWT · Guard: `OrgAdminGuard` or `CustomerOwnerGuard` |
| `POST` | `/api/v1/invoices/:invoiceId/issue` | Transition DRAFT/PENDING → ISSUED; generates invoice PDF document | JWT · Guard: `OrgAdminGuard` |
| `POST` | `/api/v1/invoices/:invoiceId/void` | Void DRAFT or ISSUED invoice (before payment); creates reversal credit note | JWT · Guard: `OrgAdminGuard` |
| `POST` | `/api/v1/invoices/:invoiceId/apply-credit` | Apply credit from `billing.credits` to this invoice | JWT · Guard: `OrgAdminGuard` |
| `GET` | `/api/v1/invoices/:invoiceId/document` | Get/download invoice PDF from `billing.invoice_documents` | JWT · Guard: `OrgAdminGuard` or `CustomerOwnerGuard` |
| `GET` | `/api/v1/invoices/:invoiceId/tax-breakdown` | Get tax calculation audit trail from `billing.tax_calculation_audit` | JWT · Guard: `OrgAdminGuard` |
| `POST` | `/api/v1/invoices/:invoiceId/reminders` | Trigger manual invoice reminder email | JWT · Guard: `OrgAdminGuard` |

---

## Data Tables Used

| Table | Schema | Operation | Key Columns |
|---|---|---|---|
| `invoices` | `billing` | INSERT · SELECT · UPDATE | `id, customer_id, contract_id, invoice_number, amount, credits_applied, currency, status` |
| `invoice_line_items` | `billing` | INSERT · SELECT | `id, invoice_id, meter_id, description, amount, model_name` |
| `invoice_batches` | `billing` | INSERT · SELECT · UPDATE | `id, org_id, billing_period_start, billing_period_end, run_started_at, run_completed_at, status` |
| `invoice_documents` | `billing` | INSERT · SELECT | `id, invoice_id, org_id, document_type, file_name, storage_path, created_by` |
| `invoice_status_history` | `billing` | INSERT | `id, invoice_id, changed_at, changed_by, old_status, new_status` |
| `invoice_reminder_schedules` | `billing` | SELECT · UPDATE | `id, invoice_id, org_id, reminder_number, trigger_days, trigger_type, status` |
| `payments` | `billing` | SELECT | `id, customer_id, invoice_id, amount, currency, status` |
| `credit_notes` | `billing` | INSERT · SELECT | `id, customer_id, invoice_id, amount, credit_note_number, applied_date` |
| `tax_calculation_audit` | `billing` | INSERT · SELECT | `id, invoice_id, org_id, tax_region_id, taxable_amount, tax_rate, tax_amount, tax_type` |
| `branding_configs` | `billing` | SELECT | `id, org_id, logo_url, company_name, primary_color, footer_text, terms_conditions` |
| `customers` | `customer` | SELECT | `id, org_id, name, email, credit_balance` |
| `contracts` | `customer` | SELECT | `id, customer_id, status, billing_period_start, billing_period_end` |
| `credits` | `billing` | SELECT · UPDATE | `id, customer_id, amount, remaining_balance, status` |
| `meters` | `catalog` | SELECT | `id, name, model_name, unit, rate` |
| `tax_regions` | `billing` | SELECT | `id, org_id, country_code, state_code, name, rate` |

---

## State Machine — Invoice Lifecycle

**Note:** `invoice_status` enum values in postgres: `draft`, `issued`, `viewed`, `partially_paid`, `past_due`, `cancelled`, `voided`, `pending`, `overdue`, `credited`

```
DRAFT
  └─── issue() ───→ PENDING
                     └─── issue() ───→ ISSUED
                                         ├─── payment received ───→ PAID
                                         ├─── past due ─────────→ PAST_DUE ───→ PAID
                                         ├─── void() ────────────→ VOIDED
                                         └─── credit note ───────→ CREDITED

DRAFT ──── void() ───→ VOIDED
ISSUED ─── void() ────→ VOIDED
PAST_DUE ─── void() ────→ (rejected — cannot void past_due invoice)

Note: VOIDED and CREDITED are terminal states.
      PAID invoices cannot be voided; a reversal credit note must be issued instead.
      PAST_DUE triggers dunning process via invoice_reminder_schedules.
```

| From | To | Trigger |
|---|---|---|
| `DRAFT` | `PENDING` | `issue()` called |
| `PENDING` | `ISSUED` | `issue()` called (confirmation step) |
| `ISSUED` | `PAST_DUE` | Payment not received by due date (batch job) |
| `ISSUED` | `PAID` | Payment recorded in `billing.payments` |
| `PAST_DUE` | `PAID` | Payment recorded |
| `DRAFT` | `VOIDED` | `void()` called by ORG_ADMIN |
| `ISSUED` | `VOIDED` | `void()` called (before payment) |
| `ISSUED` | `CREDITED` | Credit note applied exceeds or equals invoice amount |
| `PAST_DUE` | `VOIDED` | Not allowed — must pay first |

---

## Error Codes

| Code | HTTP | Trigger |
|---|---|---|
| `INVOICE_NOT_FOUND` | 404 | `invoiceId` does not exist in `billing.invoices` |
| `INVOICE_ALREADY_ISSUED` | 409 | `issue()` called on an invoice already in `ISSUED` status |
| `INVOICE_NOT_VOIDABLE` | 409 | `void()` called on `PAID`, `CREDITED`, or `VOIDED` invoice |
| `INVOICE_PAST_DUE_NOT_VOIDABLE` | 409 | `void()` called on `PAST_DUE` invoice — must be resolved via payment or credit |
| `INVOICE_BATCH_NOT_FOUND` | 404 | `batchId` does not exist in `billing.invoice_batches` |
| `CUSTOMER_NOT_FOUND` | 404 | `customer_id` in request does not exist in `customer.customers` |
| `INSUFFICIENT_CREDIT_BALANCE` | 422 | `apply-credit` amount exceeds customer's available credit in `billing.credits` |
| `CREDIT_NOT_APPLICABLE` | 422 | Credit belongs to a different customer than the invoice |
| `BRANDING_CONFIG_MISSING` | 422 | `issue()` called but org has no row in `billing.branding_configs` |
| `INVOICE_DOCUMENT_NOT_FOUND` | 404 | Invoice PDF not found in `billing.invoice_documents` |
| `TAX_BREAKDOWN_NOT_FOUND` | 404 | No rows in `billing.tax_calculation_audit` for this invoice |
| `REMINDER_NOT_ISSUED` | 422 | Reminder trigger attempted on a non-ISSUED or already-paid invoice |
| `BATCH_ALREADY_RUNNING` | 409 | Another batch is currently `RUNNING` for the same org and billing period |
| `CONTRACT_INACTIVE` | 422 | Contract referenced in invoice line item is not ACTIVE |
| `ORG_MISMATCH` | 403 | Invoice belongs to a different org than the authenticated org |
| `CUSTOMER_ACCESS_DENIED` | 403 | Authenticated CUSTOMER attempting to access another customer's invoice |
| `END_USER_NO_ACCESS` | 403 | END_USER role has no invoice access |
| `INVALID_STATUS_TRANSITION` | 409 | Status transition not permitted by the state machine |
| `CURRENCY_MISMATCH` | 422 | Credit currency does not match invoice currency |
| `BILLING_PERIOD_MISMATCH` | 422 | Invoice's billing period does not match the batch request |

---

## Environment Config Keys

| Key | Description |
|---|---|
| `INVOICE_BATCH_LOCK_TIMEOUT_SEC` | Max duration for batch job lock to prevent duplicate runs (default: 3600) |
| `INVOICE_DEFAULT_CURRENCY` | Default currency code for invoices when not specified (default: `USD`) |
| `INVOICE_NUMBER_PREFIX` | Prefix for invoice numbers, e.g. `INV-` (default: `INV-`) |
| `INVOICE_NUMBER_SEQUENCE_PAD` | Zero-padding width for invoice number sequence (default: `6`) |
| `INVOICE_DUE_DAYS` | Default payment due in days from ISSUED date (default: `30`) |
| `INVOICE_PDF_STORAGE_PATH` | Base path for invoice PDF storage (e.g. `/invoices/{org_id}/{year}/{month}/`) |
| `INVOICE_EMAIL_FROM` | Sender address for invoice emails — e.g. `billing@quantumbilling.io` |
| `INVOICE_EMAIL_SUBJECT_TEMPLATE` | Email subject template, e.g. `Invoice {invoice_number} from {company_name}` |
| `INVOICE_OVERDUE_CHECK_CRON` | Cron expression for overdue invoice check batch (default: `0 0 * * *`) |
| `AUTO_APPLY_CREDITS` | Whether credits are auto-applied on invoice generation (default: `true`) |
| `MAX_CREDIT_AUTO_APPLY_PCT` | Max percentage of invoice amount that can be auto-applied as credit (default: `100`) |
| `BRANDING_FALLBACK_LOGO_URL` | Fallback logo URL when org has no `billing.branding_configs` logo |
| `BRANDING_FALLBACK_COMPANY_NAME` | Fallback company name when org has no branding config |
| `SMTP_HOST` / `SMTP_PORT` | Email transport host and port |
| `SMTP_USER` / `SMTP_PASS` | SMTP credentials (or SES access key) |
| `KEYCLOAK_URL` | Keycloak server base URL |
| `KEYCLOAK_REALM` | `quantumbilling` |
| `KEYCLOAK_CLIENT_ID` / `KEYCLOAK_CLIENT_SECRET` | Backend confidential client credentials |
| `DATABASE_URL` | PostgreSQL connection string (Prisma) |
| `TAX_CALCULATION_PROVIDER` | Tax provider name (e.g. `stripe_tax`, `avalara`) |
| `TAX_DEFAULT_REGION` | Default tax region when customer has no `tax_region_id` |
| `CREDIT_NOTE_NUMBER_PREFIX` | Prefix for credit note numbers, e.g. `CN-` (default: `CN-`) |
| `DUNNING_REMINDER_DAY_OFFSETS` | Comma-separated day offsets for dunning reminders (e.g. `1,7,14,30`) |

---

## UI Story

### Invoice List Page (ORG_ADMIN)

Accessible from Billing › Invoices. Displays a paginated table of invoices for the org with columns: Invoice #, Customer, Billing Period, Amount, Credits Applied, Net Due, Status, Issue Date. Filters: Status dropdown (All / DRAFT / PENDING / ISSUED / OVERDUE / PAID / VOID / CREDITED), Customer search, Date range picker. Row actions: View (eye icon), Download PDF, Issue (if DRAFT), Void (if DRAFT or ISSUED), Apply Credit.

Bulk action: Select multiple invoices → Batch Issue or Batch Void. Pagination: 20/page with total count.

### Invoice Detail Page (ORG_ADMIN)

Full invoice view with:
- **Header**: Invoice number, status badge, customer name + email, billing period, issue date, due date
- **Line Items table**: Description, Meter/Model, Amount — sourced from `billing.invoice_line_items`
- **Tax Breakdown panel**: Tax region, taxable amount, rate, tax amount, tax type — from `billing.tax_calculation_audit`
- **Credits Applied section**: Credit note #, amount applied, date
- **Net Due banner**: Amount − Credits Applied = Net Due
- **Status History timeline**: Every status change with timestamp and actor
- **Actions panel**: Issue (DRAFT/PENDING), Void (DRAFT/ISSUED), Apply Credit, Send Reminder, Download PDF

### Invoice PDF

Generated on `issue()` using `billing.branding_configs`:
- Org logo (`logo_url`) at top left
- Org company name and address
- Primary color (`primary_color`) as header accent
- Invoice number, date, due date
- Bill-to customer info
- Line items table
- Subtotal, tax line items, total
- Credits applied line
- Footer text + Terms & Conditions (`footer_text`, `terms_conditions`)

### Customer Invoice View (CUSTOMER)

Customers access their own invoices via their billing portal. Read-only view. Can download PDF. Cannot issue, void, or apply credits. If customer has no invoices, empty state: "No invoices yet."

### Invoice Reminder Email

Triggered manually via `POST /api/v1/invoices/:invoiceId/reminders` or automatically via `billing.invoice_reminder_schedules`. Email contains: invoice PDF attachment, outstanding balance, due date, payment link (if applicable).

---

## AI Recommendations

The following steps should be followed to implement this Invoice user story:

### Step 1: Database Schema Setup
- Verify Prisma enums align with postgres `invoice_status` enum: `{ DRAFT PENDING ISSUED PAST_DUE PAID VOIDED CREDITED }`
- Verify `InvoiceBatchStatus { PENDING RUNNING COMPLETED FAILED CANCELLED }`
- Verify `ReminderStatus { PENDING SENT FAILED }`
- Verify `PaymentStatus { PENDING PROCESSING SUCCEEDED FAILED REFUNDED CANCELLED }`
- Run Prisma migrations to sync schema with database

### Step 2: RBAC Guards Implementation
- Implement `OrgAdminGuard`: allows `ORG_ADMIN` and `SUPER_ADMIN` roles
- Implement `CustomerOwnerGuard`: allows `CUSTOMER` whose `customer.customers.id` matches invoice's `customer_id`
- Implement `END_USER` denial at guard level — always reject
- Test all guards with each role type before proceeding

### Step 3: Invoice Batch Generation (`POST /api/v1/invoices/batch`)
- Wrap batch operation in a DB transaction
- Use row-level lock on `billing.invoice_batches` with `FOR UPDATE` to prevent duplicate concurrent runs
- Check for idempotency: if a COMPLETED batch exists for same org + billing period, return existing `batch_id` with 200
- Query all ACTIVE contracts for the billing period
- Generate line items from subscription base fees + usage from `catalog.meters` + prorated adjustments + discounts
- Create invoice records with status `DRAFT`, unique `invoice_number`, amount = sum of line items, `credits_applied = 0`
- Create `billing.invoice_batches` record with status `RUNNING`, then update to `COMPLETED` or `FAILED`
- Return `{ batch_id, invoice_ids: [...] }` with 201 on success

### Step 4: Invoice Status State Machine
- Implement state machine validation for all status transitions
- All transitions must be validated against the state machine table before UPDATE
- Invalid transitions return `409 INVALID_STATUS_TRANSITION`
- Record every status transition in `billing.invoice_status_history` with `changed_at`, `changed_by`, `old_status`, `new_status`
- Write all financial operations (issue, void, apply-credit) to `audit.security_audit_logs` with actor_id, action, and metadata

### Step 5: Issue Invoice (`POST /api/v1/invoices/:invoiceId/issue`)
- Validate current status is `DRAFT` or `PENDING` — return `409 INVOICE_ALREADY_ISSUED` if `ISSUED`
- Check org has a row in `billing.branding_configs` — return `422 BRANDING_CONFIG_MISSING` if missing
- Calculate tax via external tax provider (Avalara/Stripe Tax) and store in `billing.tax_calculation_audit`
- If tax provider unavailable, fail with `502 TAX_PROVIDER_UNAVAILABLE`
- Generate invoice PDF using `puppeteer` or `pdfkit` with branding (logo, primary_color, footer_text, terms_conditions)
- Store PDF in object storage (S3/MinIO) and record path in `billing.invoice_documents`
- Transition status DRAFT → PENDING → ISSUED
- Return `{ invoice_id, invoice_number, status: "ISSUED", document_url }`

### Step 6: Void Invoice (`POST /api/v1/invoices/:invoiceId/void`)
- Check `billing.payments` for any `PAYMENT_CAPTURED` rows linked to this invoice — if exists, reject with `409 INVOICE_NOT_VOIDABLE`
- Cannot void `PAID`, `CREDITED`, or already `VOIDED` invoices
- Cannot void `PAST_DUE` invoices — return `409 INVOICE_PAST_DUE_NOT_VOIDABLE`
- Create reversal credit note in `billing.credit_notes` for full invoice amount
- Transition status to `VOIDED`
- Record in `billing.invoice_status_history` and `audit.security_audit_logs`
- Return `{ invoice_id, status: "VOID", credit_note_id }`

### Step 7: Apply Credit (`POST /api/v1/invoices/:invoiceId/apply-credit`)
- Validate `billing.credits.customer_id` matches `billing.invoices.customer_id` — return `422 CREDIT_NOT_APPLICABLE` if mismatch
- Validate credit currency matches invoice currency — return `422 CURRENCY_MISMATCH` if mismatch
- Check `billing.credits.remaining_balance` >= requested credit_amount — return `422 INSUFFICIENT_CREDIT_BALANCE` if insufficient
- Update `credits_applied` on invoice; reduce `billing.credits.remaining_balance`
- If credit exceeds invoice net due, create or update `billing.credit_notes`
- Record in `billing.invoice_status_history` and `audit.security_audit_logs`

### Step 8: Invoice Number Generation
- Use per-org sequence or ULID prefixed with `INVOICE_NUMBER_PREFIX`
- Format: `{PREFIX}-{ORG_CODE}-{SEQUENCE}` (e.g., `INV-ACM-000001`)
- Must be unique per org — enforce with DB constraint

### Step 9: Overdue Detection & Dunning
- Implement cron job reading all `ISSUED` invoices where `due_date < NOW()` and no `PAYMENT_CAPTURED` in `billing.payments`
- Transition overdue invoices to `OVERDUE` status
- Create reminder schedule entries in `billing.invoice_reminder_schedules`
- Use `DUNNING_REMINDER_DAY_OFFSETS` config for day offsets (e.g., `1,7,14,30`)
- Reminders only sent for `ISSUED` invoices that are overdue

### Step 10: Invoice List & Detail Endpoints
- `GET /api/v1/invoices`: paginated list (default 20/page), filterable by `status` and `customer_id`
- `GET /api/v1/invoices/:invoiceId`: return invoice with `billing.invoice_line_items` array
- `GET /api/v1/invoices/:invoiceId/tax-breakdown`: return rows from `billing.tax_calculation_audit`
- `GET /api/v1/invoices/:invoiceId/document`: return/download PDF from `billing.invoice_documents`
- All endpoints enforce RBAC: `ORG_ADMIN` sees own org, `CUSTOMER` sees own invoices only

### Step 11: Manual Reminder (`POST /api/v1/invoices/:invoiceId/reminders`)
- Validate invoice is `ISSUED` and overdue — return `422 REMINDER_NOT_ISSUED` otherwise
- Trigger email via SMTP with invoice PDF attachment, outstanding balance, due date, payment link
- Update `billing.invoice_reminder_schedules` status to `SENT`

### Step 12: Testing
- Execute all test cases (TC-01 through TC-10) to verify acceptance criteria
- Test state machine transitions for all valid and invalid paths
- Test RBAC guards with each role type
- Test concurrent batch job prevention with row-level locking

---

## Dependencies & Notes for Agent

- **Invoice PDF generation**: Use a PDF library (e.g. `puppeteer` or `pdfkit`) with org branding from `billing.branding_configs`. Store result in object storage (S3/MinIO) and record path in `billing.invoice_documents`.
- **Batch job**: Wrap `POST /api/v1/invoices/batch` in a DB transaction. Use a row-level lock on `billing.invoice_batches` with `FOR UPDATE` to prevent duplicate concurrent runs for the same org + period.
- **Invoice number generation**: Use a per-org sequence or ULID prefixed with `INVOICE_NUMBER_PREFIX`. Must be unique per org.
- **Credit application**: Validate customer ownership of credit (`billing.credits.customer_id` must match `billing.invoices.customer_id`). Check `billing.credits.remaining_balance` before applying. Use `CURRENCY_MISMATCH` if currencies differ.
- **VOID flow**: Before voiding, check `billing.payments` for any PAYMENT_CAPTURED rows linked to this invoice. If payments exist, reject with `INVOICE_NOT_VOIDABLE`. Create reversal credit note in `billing.credit_notes`.
- **Overdue detection**: Cron job reads all ISSUED invoices where `due_date < NOW()` and no PAYMENT_CAPTURED in `billing.payments`; transitions them to OVERDUE and creates reminder schedule entries.
- **Tax calculation**: Call external tax provider (e.g. Avalara, Stripe Tax) before finalizing invoice. Store result in `billing.tax_calculation_audit`. If tax provider is unavailable, fail the `issue()` call with 502 `TAX_PROVIDER_UNAVAILABLE`.
- **RBAC guards**:
  - `OrgAdminGuard`: allows `ORG_ADMIN` and `SUPER_ADMIN`
  - `CustomerOwnerGuard`: allows `CUSTOMER` whose `customer.customers.id` matches invoice's `customer_id`
  - `END_USER`: always denied at guard level
- **Prisma enums** (aligns with postgres `invoice_status` enum):
  - `InvoiceStatus { DRAFT PENDING ISSUED PAST_DUE PAID VOIDED CREDITED }`
  - `InvoiceBatchStatus { PENDING RUNNING COMPLETED FAILED CANCELLED }`
  - `ReminderStatus { PENDING SENT FAILED }`
  - `PaymentStatus { PENDING PROCESSING SUCCEEDED FAILED REFUNDED CANCELLED }`
- **State machine enforcement**: All status transitions must be validated against the state machine table before UPDATE. Invalid transitions return `INVALID_STATUS_TRANSITION`.
- **Audit logging**: All state transitions must be recorded in `billing.invoice_status_history`. All financial operations (issue, void, apply-credit) must also write to `audit.security_audit_logs` with actor_id, action, and metadata.
- **Idempotency**: `POST /api/v1/invoices/batch` must be idempotent for the same billing period + org combination. If a COMPLETED batch already exists for the same period, return its existing `batch_id` with 200.
