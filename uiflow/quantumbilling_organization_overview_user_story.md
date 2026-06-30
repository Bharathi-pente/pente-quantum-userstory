# QuantumBilling User Story: Organization Overview

**QB-STORY-017** · Sprint 3 · Phase: UI Feature

---

## Title

**Organization Overview** — real-time dashboard showing team usage, token consumption, and billing health

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As an ORG_ADMIN or CUSTOMER**, I want to view a real-time organization overview dashboard that displays aggregated team usage, token consumption broken down by team member, API call volumes, and billing health indicators, so that I can monitor adoption, identify top consumers, and detect anomalies before they become billing issues.

The Organization Overview is the landing-page dashboard within the Customer Portal. It surfaces:

- **Team Usage Summary** — aggregated usage metrics (total tokens, API calls, cost) for the current billing period
- **Token Usage by Team Member (Real-time API)** — per-end-user breakdown of input tokens, output tokens, and cached tokens consumed via the Real-time API
- **Usage Trend** — time-series chart (daily/weekly) showing usage over the current billing period
- **Top Consumers** — ranked list of the top 5 end users by token consumption
- **Billing Health** — credit balance, estimated invoice amount, and any approaching limits

Key capabilities:
- ORG_ADMIN sees full org-wide data; CUSTOMER sees only their own account data
- All metrics refresh in real-time via polling (30s interval) or WebSocket push
- Token breakdown by team member is sourced from `usage_events` joined with `end_users`
- Export capability for the team usage table (CSV)
- Sort by usage (high to low) and filter by team member

---

## RBAC Roles

| Role | Can view org overview | Can view per-user breakdown | Can export | Scope |
|------|----------------------|----------------------------|------------|-------|
| **SUPER_ADMIN** | Yes (any org) | Yes (any org) | Yes | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Yes (own org) | Yes | Own org only |
| **CUSTOMER** | Yes (own account) | No (aggregate only) | No | Own account only |
| **END_USER** | No | No | No | No access |

---

## Acceptance Criteria

### Team Usage Summary

1. The dashboard header displays the current billing period (e.g., "Jun 1 – Jun 30, 2026") and org name.
2. The summary card shows: Total Input Tokens, Total Output Tokens, Total API Calls, and Total Estimated Cost for the period.
3. Credit balance is displayed with a health indicator: green (> 20% of typical spend), amber (5–20%), red (< 5%).
4. Clicking any summary metric navigates to the detailed Usage Analytics page.

### Token Usage by Team Member (Real-time API)

5. A "Team Usage" tab/panel shows a table with columns: End User (name + email), Input Tokens, Output Tokens, Cached Tokens, Total Tokens, % of Total.
6. Table is sorted by Total Tokens descending by default.
7. ORG_ADMIN can click on a row to expand and see per-meter breakdown for that end user.
8. CUSTOMER role sees aggregated team-level data only — per-user breakdown is not visible.
9. A "Sort" dropdown allows sorting by: Usage (High to Low), Usage (Low to High), User Name (A–Z).
10. A "Filter" input allows filtering by end user email or name; filtered results update on apply.
11. An "Export" button generates a CSV of the current table view.

### Usage Trend Chart

12. A line/area chart shows daily token consumption (input + output + cached) over the current billing period.
13. Chart supports hover tooltips showing exact values per day.
14. A period toggle allows switching between "This Period" and "Previous Period" for comparison.

### Top Consumers

15. A "Top 5 Consumers" panel lists the top 5 end users by token volume with their total and % share.
16. Each entry is clickable — clicking navigates to that user's detail usage page.

### Billing Health

17. Credit balance with a visual gauge (0–100% of threshold).
18. Estimated invoice amount for the current period (based on current run-rate).
19. Any approaching usage limits are flagged with a warning badge.

### Real-time Updates

20. Metrics refresh automatically every 30 seconds via polling `GET /api/v1/orgs/:orgId/usage-summary`.
21. WebSocket connection (`/ws/org/:orgId/usage`) pushes live updates when new usage events are ingested — no page refresh required.
22. A "Last updated" timestamp is displayed; clicking "Refresh" manually fetches the latest data.

---

## Test Cases

### TC-01 — Happy path: ORG_ADMIN views organization overview

**Given:** authenticated ORG_ADMIN for org `acme`
**When:** navigating to `/dashboard` (organization overview)
**Then:** 200 returned; dashboard displays billing period, org name, and summary card with Total Input Tokens, Total Output Tokens, Total API Calls, and Estimated Cost
**And:** credit balance health indicator is shown

---

### TC-02 — View per-user token breakdown

**Given:** org `acme` has multiple end users with recorded usage
**When:** clicking on the "Team Usage" tab
**Then:** table shows each end user's Input Tokens, Output Tokens, Cached Tokens, Total Tokens, and % of Total
**And:** rows are sorted by Total Tokens descending

---

### TC-03 — Sort team usage by usage high to low

**Given:** org `acme` has end users with varying usage
**When:** clicking the "Sort" dropdown and selecting "Usage (High to Low)"
**Then:** table is reordered so top consumers appear first

---

### TC-04 — Filter team usage by user

**Given:** org `acme` has multiple end users including `john@acme.com`
**When:** entering `john@acme.com` in the Filter field and clicking Apply
**Then:** table displays only `john@acme.com`'s usage row

---

### TC-05 — Export team usage report as CSV

**Given:** org `acme` has team usage data
**When:** clicking the "Export" button and selecting "CSV"
**Then:** a CSV file is downloaded containing the current table view with columns: End User, Email, Input Tokens, Output Tokens, Cached Tokens, Total Tokens, % of Total

---

### TC-06 — CUSTOMER views aggregate team usage only

**Given:** authenticated CUSTOMER for account `acme`
**When:** navigating to the organization overview
**Then:** summary metrics are visible but per-user breakdown is not displayed
**And:** "Export" button is not rendered

---

### TC-07 — Real-time update via polling

**Given:** ORG_ADMIN is viewing the organization overview
**When:** a new usage event is ingested for an end user in the org
**Then:** within 30 seconds, the dashboard metrics update to reflect the new usage without page refresh

---

### TC-08 — WebSocket real-time push

**Given:** ORG_ADMIN has the dashboard open
**When:** a new usage event is ingested
**Then:** a WebSocket message is received on `/ws/org/:orgId/usage` and the UI updates immediately

---

### TC-09 — View usage trend chart

**Given:** org `acme` has usage history for the current billing period
**When:** viewing the Usage Trend section
**Then:** a line/area chart displays daily token consumption for the period
**And:** hovering over a data point shows the exact values for that day

---

### TC-10 — Top 5 consumers panel

**Given:** org `acme` has multiple end users with recorded usage
**When:** viewing the Top 5 Consumers panel
**Then:** the top 5 end users by token volume are listed with their total and percentage share
**And:** clicking any entry navigates to that user's detail usage page

---

### TC-11 — Billing health indicators

**Given:** org `acme` has a credit balance and usage limits configured
**When:** viewing the Billing Health section
**Then:** credit balance gauge is shown with appropriate color coding
**And:** any approaching limits display a warning badge

---

### TC-12 — END_USER denied access

**Given:** actor role is `END_USER`
**When:** navigating to `/dashboard` or any Organization Overview endpoint
**Then:** 403 `FORBIDDEN` — guard rejects before service layer

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/v1/orgs/:orgId/usage-summary` | Aggregated usage summary for the billing period (tokens, calls, cost) | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `GET` | `/api/v1/orgs/:orgId/usage-summary/realtime` | Per-end-user token breakdown for Team Usage panel | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` · Query: `?period_start=&period_end=&sort=&user_filter=` |
| `GET` | `/api/v1/orgs/:orgId/usage-summary/trend` | Daily usage trend data for chart | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` · Query: `?period_start=&period_end=` |
| `GET` | `/api/v1/orgs/:orgId/usage-summary/top-consumers` | Top 5 end users by token volume | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |
| `GET` | `/api/v1/customers/:customerId/usage-summary` | Aggregated summary for CUSTOMER role (own account only) | JWT · Guard: `CustomerGuard` |
| `GET` | `/ws/org/:orgId/usage` | WebSocket endpoint for real-time usage push | JWT · Guard: `OrgAdminGuard` or `SuperAdminGuard` |

---

## Data Tables Used

| Table | Schema | Operation | Key Columns |
|-------|--------|-----------|-------------|
| `usage_events` | `billing` | SELECT · SUM | `id, meter_id, end_user_id, value, event_timestamp` |
| `end_users` | `customer` | SELECT | `id, customer_id, external_user_id, name, email, status` |
| `meters` | `catalog` | SELECT | `id, name, event_type, aggregation` |
| `customers` | `customer` | SELECT | `id, org_id, credit_balance, health_score` |
| `organizations` | `identity` | SELECT | `id, name, billing_email, currency` |
| `pricing_models` | `catalog` | SELECT | `id, org_id, meter_id, pricing_type` |
| `pricing_tiers` | `catalog` | SELECT | `id, pricing_model_id, price_per_unit` |
| `subscriptions` | `customer` | SELECT | `id, org_id, status, current_period_end` |

---

## State Machine — Not Applicable

This story is a read-only dashboard. No state transitions are managed by this feature.

---

## Error Codes

| Code | HTTP | Trigger |
|------|------|---------|
| `ORG_NOT_FOUND` | 404 | `orgId` does not exist in `identity.organizations` |
| `CUSTOMER_NOT_FOUND` | 404 | `customerId` does not exist in `customer.customers` |
| `FORBIDDEN` | 403 | Actor is `END_USER` or unauthenticated |
| `PERIOD_INVALID` | 422 | `period_start` or `period_end` is invalid or `period_start > period_end` |
| `WEBSOCKET_AUTH_FAILED` | 401 | WebSocket connection attempted without valid JWT |

---

## Environment Config Keys

| Key | Description |
|-----|-------------|
| `USAGE_SUMMARY_REFRESH_INTERVAL_SEC` | Polling interval for dashboard refresh in seconds (default: `30`) |
| `WEBSOCKET_ENABLED` | Enable WebSocket real-time push (default: `true`) |
| `CREDIT_BALANCE_WARNING_THRESHOLD_PCT` | Percentage of typical spend that triggers amber health indicator (default: `20`) |
| `CREDIT_BALANCE_CRITICAL_THRESHOLD_PCT` | Percentage that triggers red health indicator (default: `5`) |
| `TOP_CONSUMERS_COUNT` | Number of top consumers to display (default: `5`) |
| `USAGE_EXPORT_MAX_ROWS` | Maximum rows per CSV export (default: `10000`) |
| `DATABASE_URL` | PostgreSQL connection string (Prisma) |
| `KEYCLOAK_URL` | Keycloak server base URL |
| `KEYCLOAK_REALM` | `quantumbilling` |

---

## UI Story

### Organization Overview Dashboard — Landing Page

Accessible at `/dashboard` for ORG_ADMIN and SUPER_ADMIN; `/my-account/overview` for CUSTOMER.

**Header Section:**
- Org name (or "My Account" for CUSTOMER)
- Current billing period label (e.g., "Jun 1 – Jun 30, 2026")
- "Last updated: {timestamp}" with manual refresh icon button

**Summary Card (4-up grid):**
- **Total Input Tokens** — aggregated from `usage_events` where `meter.event_type = 'llm.inference'` and `field = 'input_tokens'`; displayed as human-readable (e.g., "98.4M tokens")
- **Total Output Tokens** — same pattern for output_tokens
- **Total API Calls** — count of events where `meter.aggregation = COUNT`
- **Estimated Cost** — calculated from token counts × applicable rate card prices
- Each card is clickable → navigates to Usage Analytics for that metric

**Credit Balance Health Gauge:**
- Visual gauge (semicircle or linear bar) showing credit balance vs. threshold
- Color: green (> 20%), amber (5–20%), red (< 5%)
- Displays current balance and threshold value

### Team Usage Panel (Tab: "Team Usage")

**Table — Token Usage by Team Member:**
| Column | Description |
|--------|-------------|
| End User | Name + email of the end user (from `customer.end_users`) |
| Input Tokens | Sum of input token events for this user |
| Output Tokens | Sum of output token events for this user |
| Cached Tokens | Sum of cached token events for this user |
| Total Tokens | Sum of all three token types |
| % of Total | This user's total as percentage of org total |

**Controls above table:**
- **Sort** dropdown: "Usage (High to Low)" · "Usage (Low to High)" · "User Name (A–Z)"
- **Filter** input: text field for user email/name + "Apply" button
- **Export** button: opens format dropdown (CSV) → downloads file

**Expandable row:** Clicking a row expands to show per-meter breakdown for that end user.

### Usage Trend Chart

- Line or area chart with three series: Input Tokens, Output Tokens, Cached Tokens
- X-axis: days of the billing period
- Y-axis: token count (auto-scaled)
- Hover tooltip: shows all three values for that day
- Period toggle: "This Period" | "Previous Period"

### Top 5 Consumers Panel

- Ranked list: rank number, user name, total tokens, % share
- Progress bar visual showing relative share
- Clicking an entry → navigates to `/end-users/:userId/usage`

### Billing Health Panel

- **Credit Balance** gauge (see above)
- **Estimated Invoice** amount for current period (run-rate × days elapsed / total days)
- **Limit Warnings** — any `customer_limits` approaching threshold show a warning badge

---

## Dependencies & Notes for Agent

- **Real-time polling:** Frontend polls `GET /api/v1/orgs/:orgId/usage-summary` every `USAGE_SUMMARY_REFRESH_INTERVAL_SEC` seconds (default 30). Show "Live" indicator when polling is active.
- **WebSocket:** If `WEBSOCKET_ENABLED=true`, establish WS connection to `/ws/org/:orgId/usage`. On message, update dashboard state without full refresh. Fall back to polling if WS connection fails.
- **Token unit display:** Format large numbers as "98.4M tokens", "1.2B tokens" — do not show raw integers.
- **Prisma queries:** The per-user token breakdown requires joining `usage_events` → `meters` → `end_users`. Use `GROUP BY end_user_id` with `SUM` aggregation. Add index on `(meter_id, event_timestamp)` for performance.
- **CUSTOMER role restriction:** The per-user breakdown endpoint must check `actor.role !== CUSTOMER` before returning `end_user_id`-level data. For CUSTOMER, return only the aggregate totals.
- **Rate calculation:** Estimated cost is computed by looking up the active `pricing_model` or `rate_card_rates` for each meter. Fall back to a default rate if no pricing is configured.
- **Audit logging:** This is a read-only feature — no audit logs are written for dashboard views.
- **SUPER_ADMIN access:** SUPER_ADMIN can switch org context via `X-ORG-ID` header or `:orgId` path param to view any org's overview.
