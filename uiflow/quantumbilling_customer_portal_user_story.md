# QuantumBilling User Story: Customer Portal

**QB-STORY-026** · Sprint 3 · Phase: UI Feature

---

## Title

**Customer Portal** — self-service dashboard for customers to manage their account, usage, billing, and entitlements

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As a CUSTOMER**, I want to access a self-service portal where I can view my organization's usage, manage invoices, monitor credits, view entitlements, and understand my billing — without needing to contact support.

The Customer Portal is the primary interface for customers (organizations) to self-manage their QuantumBilling account. It provides read-only visibility into usage, billing, and account health, with limited self-service capabilities.

**Note:** In this context, **CUSTOMER** refers to the organization-level account (e.g., "TechCorp AI"), not an individual end user. The CUSTOMER role is scoped to the organization they belong to.

Key sections:
- **Overview** — summary dashboard with key metrics and usage chart
- **Invoices** — view and pay invoices
- **My Entitlements** — plan features and custom feature grants
- **Usage & Limits** — usage against plan limits
- **Payment Methods** — manage payment methods on file
- **My Credits** — credit balance and transaction history
- **Usage Analytics** — detailed usage charts and breakdowns
- **Embed Widgets** — access embeddable components

---

## RBAC Roles

| Role | Access Level | Scope |
|------|-------------|-------|
| **SUPER_ADMIN** | Can impersonate any customer | Platform-wide |
| **ORG_ADMIN** | Full admin access to own org (see Organization-level user stories) | Own org |
| **CUSTOMER** | Read access to own org's data, limited self-service | Own org only |
| **END_USER** | Different portal (see End User Portal) | Own usage only |

---

## Customer Portal Navigation

| Nav Item | Description | Can CUSTOMER self-serve? |
|----------|-------------|-------------------------|
| Overview | Summary dashboard | Read-only |
| Invoices | View and pay invoices | Yes (pay only) |
| My Entitlements | Plan features + custom grants | Read-only |
| Usage & Limits | Usage vs. plan limits | Read-only |
| Payment Methods | Manage payment methods | Yes (add/remove) |
| Team Usage | Aggregate team usage (not per-user) | Read-only |
| My Credits | Credit balance + ledger | Read-only |
| Usage Analytics | Detailed usage charts | Read-only |
| Embed Widgets | Embeddable components | Read-only |

---

## Acceptance Criteria

### Customer Portal Overview

1. The Customer Portal is accessible at `/my-account` for authenticated CUSTOMER.
2. Header shows the organization name and current billing period.
3. **Summary Cards (4-up):**
   - **Current Usage** — total cost for the current billing period
   - **Credit Balance** — available credits to offset charges
   - **Token Usage** — total tokens consumed with % change vs. previous period
   - **Team Members** — count of end users in the organization
4. **Usage This Month** chart — bar chart showing daily token usage for the current billing period.
5. All data is scoped to the customer's organization only.

### Customer Portal Invoices

6. **Invoices** tab lists all invoices for the customer's organization.
7. Invoice list columns: Invoice #, Amount, Status, Due Date, Actions.
8. Status badges: Paid (green), Pending (amber), Overdue (red).
9. **View** action — opens invoice detail.
10. **Download PDF** action — downloads invoice PDF.
11. **Pay Now** action — available for pending/overdue invoices; opens payment flow.
12. Cannot void, edit, or cancel invoices (read-only for customers).
13. Filters: All, Paid, Pending, Overdue.

### My Entitlements

14. **Entitlements** tab shows features available on the customer's plan.
15. **Plan Features** section — list of features included in the customer's current plan (e.g., API Access, Dashboard Access, SSO, SLA).
16. Each feature shows: feature name, description, and "Included" badge.
17. **Custom Grants** section — features granted outside the standard plan (temporary or permanent).
18. Each grant shows: feature name, reason, expiration date (if applicable), and "Custom Grant" badge.
19. Cannot modify entitlements (read-only).

### Usage & Limits

20. **Usage & Limits** tab shows current usage against plan limits.
21. Each meter (e.g., Input Tokens, Output Tokens, API Calls) shows:
    - Current usage amount
    - Plan limit (or "Unlimited")
    - Percentage used
    - Status badge: OK (green), Warning (amber, 75-90%), Critical (red, >90%)
22. Progress bar visualization for each meter.
23. Usage is shown for the current billing period.
24. Cannot modify limits (read-only; contact sales to change plan).

### Payment Methods

25. **Payment Methods** tab lists all payment methods on file.
26. Payment method shows: type (Credit Card, ACH), last 4 digits, expiry (card), status (default/active).
27. **Add Payment Method** — opens form to add new credit card or ACH account.
28. **Remove** action — removes a payment method (cannot remove if it's the only one or the default).
29. **Set as Default** action — sets a payment method as the default for future invoices.
30. Cannot edit existing payment methods (only add/remove/set default).

### Team Usage (Aggregate)

31. **Team Usage** tab shows aggregate team usage for the customer's organization.
32. Shows total tokens, total requests, total cost for the billing period.
33. **Does NOT show per-user breakdown** — this is visible only to ORG_ADMIN.
34. Chart showing daily usage trend.
35. Can export aggregate team usage as CSV.

### My Credits

36. **Credits** tab shows credit balance and transaction history.
37. Summary cards: Available Balance, Used This Month, Expiring Soon.
38. **Recent Transactions** table shows:
    - Date, Type (grant/usage/adjustment), Description, Amount (+/-), Running Balance
39. Cannot grant, edit, or revoke credits (read-only for customers).

### Usage Analytics

40. **Usage Analytics** tab shows detailed usage breakdowns.
41. Charts:
    - Token usage by model (GPT-4, Claude 3, Gemini, etc.)
    - Daily usage trend
    - Cost breakdown by model
42. Filters: Date range, Model.
43. Can export usage data as CSV.

### Embed Widgets

44. **Embed Widgets** tab provides embeddable components for the customer's website.
45. Shows available widgets: Usage Widget, Invoice Widget, Balance Widget.
46. Each widget shows: description, preview, and code snippet to embed.
47. Widgets require authentication (handled by QuantumBilling embed SDK).

### SUPER_ADMIN Impersonation

48. SUPER_ADMIN can switch to "Customer View" to impersonate any customer.
49. When impersonating, the SUPER_ADMIN sees the exact Customer Portal UI.
50. A banner indicates "Viewing as {customer_name}" with an "Exit" button to return to admin view.
51. Impersonation is logged in the audit trail.

---

## Test Cases

### TC-01 — CUSTOMER logs into customer portal

**Given:** CUSTOMER is authenticated
**When:** navigating to `/my-account`
**Then:** the Customer Portal Overview is displayed
**And:** organization name and billing period are shown in the header

---

### TC-02 — View overview dashboard

**Given:** CUSTOMER is on the Overview tab
**Then:** summary cards show: Current Usage, Credit Balance, Token Usage (with change %), Team Members
**And:** a bar chart shows daily usage for the billing period

---

### TC-03 — View invoices

**Given:** CUSTOMER navigates to Invoices tab
**Then:** all invoices for their organization are listed
**And:** each shows: Invoice #, Amount, Status, Due Date
**And:** filters for All, Paid, Pending, Overdue are available

---

### TC-04 — Pay an invoice

**Given:** CUSTOMER has a pending invoice for $1,200
**When:** clicking "Pay Now" on the invoice
**Then:** a payment form opens with saved payment methods
**When:** selecting a payment method and confirming
**Then:** payment is processed
**And:** invoice status changes to "Paid"

---

### TC-05 — Download invoice PDF

**Given:** CUSTOMER is viewing an invoice
**When:** clicking "Download PDF"
**Then:** the invoice PDF is downloaded

---

### TC-06 — View entitlements

**Given:** CUSTOMER is on the My Entitlements tab
**Then:** all plan features are listed with descriptions
**And:** any custom grants are shown separately with expiration dates

---

### TC-07 — View usage vs. limits

**Given:** CUSTOMER is on the Usage & Limits tab
**Then:** each meter shows current usage, plan limit, and percentage used
**And:** status badges indicate OK/Warning/Critical based on usage

---

### TC-08 — Add a payment method

**Given:** CUSTOMER is on the Payment Methods tab
**When:** clicking "Add Payment Method"
**Then:** a form appears to enter credit card details or ACH information
**When:** submitting valid details
**Then:** the payment method is added and appears in the list

---

### TC-09 — Remove a payment method

**Given:** CUSTOMER has 2+ payment methods on file
**When:** clicking "Remove" on a non-default payment method
**Then:** the payment method is removed from the account

---

### TC-10 — Set default payment method

**Given:** CUSTOMER has multiple payment methods
**When:** clicking "Set as Default" on a payment method
**Then:** that method becomes the default for future invoices

---

### TC-11 — View team usage (aggregate only)

**Given:** CUSTOMER is on the Team Usage tab
**Then:** aggregate team usage is displayed (total tokens, requests, cost)
**And:** per-user breakdown is NOT shown

---

### TC-12 — Export team usage

**Given:** CUSTOMER is on the Team Usage tab
**When:** clicking "Export"
**Then:** a CSV file is downloaded with aggregate team usage data

---

### TC-13 — View credit balance and ledger

**Given:** CUSTOMER is on the Credits tab
**Then:** available balance, used this month, and expiring soon are shown
**And:** recent transactions table shows all credit ledger entries

---

### TC-14 — View usage analytics

**Given:** CUSTOMER is on the Usage Analytics tab
**Then:** charts show token usage by model and daily trend
**And:** filters for date range and model are available

---

### TC-15 — View and copy embed widget code

**Given:** CUSTOMER is on the Embed Widgets tab
**Then:** available widgets are listed with descriptions and previews
**When:** clicking "Copy" on a widget
**Then:** the embed code is copied to clipboard

---

### TC-16 — SUPER_ADMIN impersonates customer

**Given:** SUPER_ADMIN is in the admin dashboard
**When:** selecting "View as Customer" and choosing an organization
**Then:** the SUPER_ADMIN sees the Customer Portal as that customer would
**And:** a banner shows "Viewing as TechCorp AI" with an Exit button

---

### TC-17 — END_USER cannot access customer portal

**Given:** actor role is `END_USER`
**When:** navigating to `/my-account` or any customer portal page
**Then:** 403 `FORBIDDEN`
**And:** END_USER is redirected to their own end user portal

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/v1/customers/:customerId/portal/overview` | Overview data for customer portal | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/customers/:customerId/invoices` | List invoices for customer | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/invoices/:invoiceId` | Get invoice detail | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/invoices/:invoiceId/pdf` | Download invoice PDF | JWT · Guard: `CustomerGuard` |
| `POST` | `/api/v1/invoices/:invoiceId/pay` | Pay an invoice | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/customers/:customerId/entitlements` | Get plan features and grants | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/customers/:customerId/usage-limits` | Get usage vs. limits | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/customers/:customerId/payment-methods` | List payment methods | JWT · Guard: `CustomerGuard` |
| `POST` | `/api/v1/customers/:customerId/payment-methods` | Add payment method | JWT · Guard: `CustomerGuard` |
| `DELETE` | `/api/v1/payment-methods/:pmId` | Remove payment method | JWT · Guard: `CustomerGuard` |
| `PUT` | `/api/v1/payment-methods/:pmId/default` | Set default payment method | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/customers/:customerId/team-usage` | Aggregate team usage (no per-user) | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/customers/:customerId/credits` | Get credit balance | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/customers/:customerId/credits/ledger` | Get credit ledger | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/customers/:customerId/usage-analytics` | Detailed usage analytics | JWT · Guard: `CustomerGuard` |
| `GET` | `/api/v1/customers/:customerId/widgets` | List available embed widgets | JWT · Guard: `CustomerGuard` |

---

## Data Tables Used

| Table | Schema | Operation | Key Columns |
|-------|--------|-----------|-------------|
| `customers` | `customer` | SELECT | `id, org_id, name, status, plan` |
| `organizations` | `identity` | SELECT | `id, name` |
| `invoices` | `billing` | SELECT | `id, customer_id, status, total, due_date` |
| `invoice_line_items` | `billing` | SELECT | `id, invoice_id, description, amount` |
| `subscriptions` | `billing` | SELECT | `id, customer_id, plan_id, status` |
| `plans` | `catalog` | SELECT | `id, name, features` |
| `plan_features` | `catalog` | SELECT | `id, plan_id, feature_id` |
| `features` | `catalog` | SELECT | `id, name, description` |
| `entitlement_grants` | `customer` | SELECT | `id, customer_id, feature_id, expires_at` |
| `usage_events` | `billing` | SELECT · SUM | `id, customer_id, meter_id, value, cost, event_timestamp` |
| `meters` | `catalog` | SELECT | `id, name, aggregation` |
| `payment_methods` | `billing` | SELECT · INSERT · DELETE | `id, customer_id, type, last_four, is_default, status` |
| `credits` | `billing` | SELECT | `id, customer_id, remaining_amount, status` |
| `credit_ledger` | `billing` | SELECT | `id, customer_id, type, amount, balance` |

---

## UI Story

### Customer Portal Layout

**Sidebar Navigation (left):**
```
┌─────────────────────────┐
│ TechCorp AI             │  ← Organization name + logo
│ (Customer Portal)        │
├─────────────────────────┤
│ 📊 Overview             │  ← Active
│ 🧾 Invoices             │
│ 🛡️ My Entitlements      │
│ 📈 Usage & Limits       │
│ 💳 Payment Methods      │
│ 👥 Team Usage           │
│ 🎁 My Credits           │
│ 📉 Usage Analytics      │
│ 📦 Embed Widgets        │
├─────────────────────────┤
│ ⚙️ Settings             │
│ 🚪 Logout               │
└─────────────────────────┘
```

**Header:**
- Organization name
- Current billing period
- User avatar / name
- Notification bell

### Overview Page

**Summary Cards (4-up):**
| Metric | Value | Change | Icon |
|--------|-------|--------|------|
| Current Usage | $48,500 | — | activity |
| Credit Balance | $20,600 | — | creditCard |
| Token Usage | 234M | +18.2% | zap |
| Team Members | 145 | — | users |

**Usage This Month:**
- Bar chart showing daily token/cost usage
- X-axis: dates in billing period
- Y-axis: token count

### Invoices Page

**Invoice Table:**
| Invoice # | Amount | Status | Due Date | Actions |
|-----------|--------|--------|----------|---------|
| INV-2026-01-001 | $48,500 | Paid | Jan 31 | View, Download |
| INV-2025-12-001 | $52,340 | Paid | Dec 31 | View, Download |

**Invoice Detail:**
- Invoice number, status badge
- Issue date, due date
- Line items table
- Summary: Subtotal, Credits, Tax, Total
- Pay Now button (if pending/overdue)

### My Entitlements Page

**Plan Features:**
```
┌─────────────────────────────────────────────────────┐
│ ✅ API Access — Full API functionality               │
│ ✅ Dashboard Access — Web dashboard                 │
│ ✅ Priority Support — 24/7 priority support         │
│ ✅ SSO/SAML — Enterprise single sign-on             │
│ ✅ SLA — 99.9% uptime SLA                          │
└─────────────────────────────────────────────────────┘
```

**Custom Grants:**
```
┌─────────────────────────────────────────────────────┐
│ 🎁 Batch API — Beta program — Expires: 2025-03-01  │
└─────────────────────────────────────────────────────┘
```

### Usage & Limits Page

**Usage Cards (2-up grid):**
```
┌─────────────────────────────┐ ┌─────────────────────────────┐
│ Input Tokens                │ │ Output Tokens               │
│ 156,000,000 / 500,000,000  │ │ 78,000,000 / 250,000,000   │
│ ████████░░░░░░░░░ 31%      │ │ ███░░░░░░░░░░░░░░ 31%     │
│ Status: OK                 │ │ Status: OK                 │
└─────────────────────────────┘ └─────────────────────────────┘
```

### Payment Methods Page

**Payment Method Card:**
```
┌─────────────────────────────────────────────────────┐
│ 💳 Visa •••• 4242                                  │
│ Expires: 12/2027                    [Default]      │
│                                       [Remove]     │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ + Add Payment Method                                │
└─────────────────────────────────────────────────────┘
```

### Team Usage Page

**Summary:**
| Total Tokens | Total Requests | Total Cost |
|--------------|----------------|------------|
| 2.34B | 12.5M | $48,500 |

**Chart:** Daily usage trend (aggregate only — no per-user breakdown)

### My Credits Page

**Summary:**
| Available | Used This Month | Expiring Soon |
|-----------|----------------|---------------|
| $20,600 | $3,770 | $0 |

**Ledger:**
| Date | Type | Description | Amount | Balance |
|------|------|-------------|--------|---------|
| Dec 25 | usage | Daily usage - Dec 25 | -$1,250 | $20,600 |

### Usage Analytics Page

**Charts:**
- Token usage by model (donut chart)
- Daily usage trend (area chart)
- Cost by model (bar chart)

### Embed Widgets Page

**Widget Cards:**
```
┌─────────────────────────────────────────────────────┐
│ Usage Widget                                        │
│ Embed a real-time usage meter on your dashboard    │
│ [Preview]  [Copy Code]                             │
└─────────────────────────────────────────────────────┘
```

---

## Dependencies & Notes for Agent

- **Authentication:** Customers authenticate via the same JWT system. The JWT contains `customer_id` (org-level) which scopes all portal queries.
- **RBAC Enforcement:** Customer portal endpoints use `CustomerGuard` which verifies `actor.customer_id === params.customer_id`.
- **No Per-User Data for CUSTOMER:** Team Usage in the customer portal is aggregate only. Per-end-user breakdown is restricted to ORG_ADMIN.
- **Impersonation:** SUPER_ADMIN can impersonate a customer by passing `X-Impersonate: customer_id` header or using an admin switch feature. Impersonation is logged.
- **Payment Processing:** Invoice payment integrates with the payment gateway (Stripe). Never store raw card details — use payment gateway tokens.
- **Credits Read-Only for Customers:** Customers cannot grant or revoke credits. This is managed by ORG_ADMIN or SUPER_ADMIN.
- **Entitlements:** Features are determined by the customer's active plan(s). Custom grants override plan defaults temporarily.
- **Usage Aggregation:** Usage in the customer portal is aggregated daily or per billing period. Real-time event data is not shown directly.
- **Embed Widgets:** Widgets use the QuantumBilling Embed SDK which handles authentication via JWT. Widgets are read-only displays.
- **Audit Logging:** All customer self-service actions (pay invoice, add payment method, etc.) are logged in the audit trail.

---

## Future Enhancements (Out of Scope for v1)

- Self-serve plan upgrade/downgrade
- Self-serve team member management
- Self-serve API key management
- In-app chat with support
- Invoice dispute/requets adjustment
- Download all invoices as ZIP
- Budget alerts configuration
- Custom report generation
- Multi-org switching (for customers with multiple orgs)
