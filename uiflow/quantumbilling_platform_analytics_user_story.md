# QuantumBilling User Story: Platform Analytics

**QB-STORY-020** · Sprint 6 · Phase: Platform Intelligence

---

## Title

**Platform Analytics** — platform-wide metrics and performance insights

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Platform Intelligence | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As a SUPER_ADMIN**, I want to view a platform-wide analytics dashboard that aggregates metrics across all organizations, shows revenue trends, tracks platform health, and surfaces top-performing organizations, so that I can monitor the overall health of the QuantumBilling platform, identify growth opportunities, and detect platform-wide issues before they impact customers.

The Platform Analytics dashboard is accessible exclusively to SUPER_ADMIN roles and surfaces:

- **Platform Health Metrics** — aggregated key performance indicators (MRR, events, active orgs, API uptime)
- **Revenue Trend Analysis** — time-series visualization of platform MRR growth over time
- **Revenue Distribution** — breakdown of platform revenue by plan type (Enterprise, Pro, Starter)
- **Top Organizations** — ranked list of the highest-revenue organizations on the platform

Key capabilities:
- SUPER_ADMIN only access — requires `SUPER_ADMIN` role; no other role can access
- Real-time metric refresh via polling (60s interval)
- All metrics are platform-wide aggregates across all organizations
- MRR trend chart supports monthly granularity with cohort breakdown (new, expansion, contraction, churn)
- Revenue by plan displays as a donut chart with legend
- Top organizations table shows MRR, event volume, and growth percentage

---

## RBAC Roles

| Role | Can view platform analytics | Scope |
|------|----------------------------|-------|
| **SUPER_ADMIN** | Yes (all orgs) | Platform-wide |
| **ORG_ADMIN** | No | No access |
| **CUSTOMER** | No | No access |
| **END_USER** | No | No access |

---

## Acceptance Criteria

### Platform Health Metrics

1. The dashboard header displays "Platform Analytics" with subtitle "Platform-wide metrics and performance insights".
2. Four metric cards are displayed in a row: **Platform MRR**, **Total Events (30d)**, **Active Orgs**, and **API Uptime**.
3. Each metric card shows: title, formatted value, change indicator (where applicable), icon, and accent color.
4. Platform MRR card displays the total Monthly Recurring Revenue with percentage change vs. previous period.
5. Total Events (30d) card displays the aggregate event count ingested in the last 30 days with percentage change.
6. Active Orgs card displays the count of organizations with status `active` and numeric change indicator.
7. API Uptime card displays the platform API availability percentage for the last 30 days.
8. Clicking a metric card navigates to the detailed view for that metric (if available).

### Revenue Trend Chart

9. An area chart titled "Platform Revenue Trend" displays monthly MRR data.
10. Chart uses `sampleMrrHistory` data with fields: month, mrr, newMrr, expansionMrr, contractionMrr, churnMrr.
11. X-axis shows month labels; Y-axis is formatted as currency (`$Xk`).
12. The chart area has a gradient fill with the platform accent color (#00D9FF).
13. Hovering over a data point shows a tooltip with all MRR components for that month.
14. A period toggle allows switching between "This Period" and "Previous Period" for comparison.

### Revenue by Plan

15. A donut/pie chart titled "Revenue by Plan" displays revenue distribution across plan types.
16. Chart uses `sampleRevenueByPlan` data with fields: name, value, customers, color.
17. Three segments: Enterprise (purple #A855F7), Pro (cyan #00D9FF), Starter (green #22C55E).
18. Legend displays plan name, revenue amount, customer count, and percentage of total.
19. Hovering a segment highlights it and shows a tooltip with name and value.

### Top Organizations by Revenue

20. A table titled "Top Organizations by Revenue" lists the top organizations by MRR.
21. Table columns: Organization, MRR, Events (30d), Growth.
22. Default view shows top 3 organizations: Acme AI Corp, Neural Networks Inc, DeepMind Labs.
23. MRR column is formatted as currency with monospace font.
24. Growth column displays a green percentage with "+" prefix.
25. Each row is clickable — clicking navigates to that organization's detail view (future feature).
26. A "View All" link at the bottom navigates to the full Organizations list.

### Real-time Updates

27. Metrics refresh automatically every 60 seconds via polling `GET /api/v1/platform/analytics/summary`.
28. A "Last updated" timestamp is displayed in the header; clicking "Refresh" manually fetches the latest data.
29. During refresh, a loading spinner or skeleton state is shown to indicate data is being fetched.

### Super Admin Guard

30. Attempting to access the Platform Analytics endpoint with a non-SUPER_ADMIN actor returns `403 FORBIDDEN`.
31. Attempting to access without authentication returns `401 UNAUTHORIZED`.

---

## Test Cases

### TC-01 — Happy path: SUPER_ADMIN views platform analytics

**Given:** authenticated SUPER_ADMIN
**When:** navigating to `/platform/analytics`
**Then:** 200 returned; dashboard displays the four metric cards (Platform MRR, Total Events, Active Orgs, API Uptime)
**And:** Revenue Trend area chart is rendered with 12 months of data
**And:** Revenue by Plan donut chart is rendered with three segments

---

### TC-02 — Platform MRR metric card displays correct value and change

**Given:** platform has aggregate MRR of `$2,450,000` with 18.3% growth
**When:** viewing the Platform Analytics dashboard
**Then:** the Platform MRR card shows "$2.45M" with a "+18.3%" change indicator in green

---

### TC-03 — Revenue trend chart renders with correct data

**Given:** platform has 12 months of MRR history
**When:** viewing the Revenue Trend section
**Then:** an area chart displays with month labels on X-axis and MRR values on Y-axis formatted as `$Xk`
**And:** hovering over "Dec" shows tooltip with mrr: 418000, newMrr: 38000, expansionMrr: 10000, contractionMrr: -4000, churnMrr: -6000

---

### TC-04 — Revenue by plan shows correct distribution

**Given:** platform has Enterprise ($245k), Pro ($128k), Starter ($45k) revenue
**When:** viewing the Revenue by Plan chart
**Then:** donut chart shows three segments with correct colors and proportions
**And:** legend displays customer counts: Enterprise 47, Pro 156, Starter 234

---

### TC-05 — Top organizations table displays correctly

**Given:** platform has multiple organizations with revenue
**When:** viewing the Top Organizations table
**Then:** table displays Organization name, MRR (formatted), Events (30d), and Growth %
**And:** top 3 orgs are shown: Acme AI Corp ($45,000, 2.1B events, +15.7%), Neural Networks Inc ($38,500, 1.8B events, +22.3%), DeepMind Labs ($12,800, 890M events, +8.9%)

---

### TC-06 — ORG_ADMIN denied access to platform analytics

**Given:** actor role is `ORG_ADMIN`
**When:** navigating to `/platform/analytics` or calling `GET /api/v1/platform/analytics/summary`
**Then:** 403 `FORBIDDEN` — guard rejects before service layer

---

### TC-07 — CUSTOMER denied access to platform analytics

**Given:** actor role is `CUSTOMER`
**When:** navigating to `/platform/analytics`
**Then:** 403 `FORBIDDEN`

---

### TC-08 — Real-time metric refresh via polling

**Given:** SUPER_ADMIN is viewing the platform analytics dashboard
**When:** 60 seconds have elapsed since last fetch
**Then:** a new request is made to `GET /api/v1/platform/analytics/summary`
**And:** dashboard metrics update without full page refresh

---

### TC-09 — Manual refresh updates metrics

**Given:** SUPER_ADMIN is viewing the platform analytics dashboard
**When:** clicking the "Refresh" button
**Then:** a new request is made to `GET /api/v1/platform/analytics/summary`
**And:** "Last updated" timestamp is refreshed to current time

---

### TC-10 — Unauthenticated request denied

**Given:** no JWT token is provided
**When:** calling `GET /api/v1/platform/analytics/summary`
**Then:** 401 `UNAUTHORIZED`

---

### TC-11 — ORG_ADMIN navigation to platform analytics blocked

**Given:** actor role is `ORG_ADMIN`
**When:** clicking on "Platform Analytics" in the sidebar navigation
**Then:** the Platform Analytics nav item is either hidden or disabled
**And:** if navigated via direct URL, 403 `FORBIDDEN` is returned

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/v1/platform/analytics/summary` | Aggregated platform metrics (MRR, events, orgs, uptime) | JWT · Guard: `SuperAdminGuard` |
| `GET` | `/api/v1/platform/analytics/mrr-trend` | Monthly MRR history with cohort breakdown | JWT · Guard: `SuperAdminGuard` · Query: `?period_start=&period_end=` |
| `GET` | `/api/v1/platform/analytics/revenue-by-plan` | Revenue distribution by plan type | JWT · Guard: `SuperAdminGuard` |
| `GET` | `/api/v1/platform/analytics/top-organizations` | Top organizations by revenue | JWT · Guard: `SuperAdminGuard` · Query: `?limit=5` |

---

## Data Tables Used

| Table | Schema | Operation | Key Columns |
|-------|--------|-----------|-------------|
| `organizations` | `identity` | SELECT · COUNT · SUM | `id, name, status, mrr, created_at` |
| `subscriptions` | `customer` | SELECT · SUM | `id, org_id, status, mrr, current_period_end` |
| `customers` | `customer` | SELECT · COUNT | `id, org_id, status, plan_type` |
| `usage_events` | `billing` | SELECT · COUNT | `id, org_id, event_timestamp, meter_id` |
| `meters` | `catalog` | SELECT | `id, event_type, aggregation` |
| `pricing_models` | `catalog` | SELECT · SUM | `id, org_id, meter_id, pricing_type` |
| `products` | `catalog` | SELECT | `id, name, plan_type, base_price` |

---

## State Machine — Not Applicable

This story is a read-only analytics dashboard. No state transitions are managed by this feature.

---

## Error Codes

| Code | HTTP | Trigger |
|------|------|---------|
| `UNAUTHORIZED` | 401 | No valid JWT token provided |
| `FORBIDDEN` | 403 | Actor is not `SUPER_ADMIN` |
| `PERIOD_INVALID` | 422 | `period_start` or `period_end` is invalid or `period_start > period_end` |
| `ANALYTICS_UNAVAILABLE` | 503 | Platform analytics service is temporarily unavailable |

---

## Environment Config Keys

| Key | Description |
|-----|-------------|
| `PLATFORM_ANALYTICS_REFRESH_INTERVAL_SEC` | Polling interval for dashboard refresh in seconds (default: `60`) |
| `TOP_ORGANIZATIONS_DEFAULT_LIMIT` | Default number of top organizations to display (default: `5`) |
| `PLATFORM_ANALYTICS_RETENTION_DAYS` | Number of days of historical data to retain for trend analysis (default: `365`) |
| `API_UPTIME_WINDOW_HOURS` | Time window for calculating API uptime percentage (default: `720` hours / 30 days) |
| `DATABASE_URL` | PostgreSQL connection string (Prisma) |
| `KEYCLOAK_URL` | Keycloak server base URL |
| `KEYCLOAK_REALM` | `quantumbilling` |

---

## UI Story

### Platform Analytics Dashboard — SuperAdmin Only

Accessible at `/platform/analytics` for SUPER_ADMIN only.

**Header Section:**
- Title: "Platform Analytics"
- Subtitle: "Platform-wide metrics and performance insights"
- "Last updated: {timestamp}" with manual refresh icon button

### Metric Cards (4-up grid)

| Metric | Value | Change | Icon | Color |
|--------|-------|--------|------|-------|
| **Platform MRR** | $2.45M | +18.3% | dollar | #22C55E (green) |
| **Total Events (30d)** | 45.2B | +24.1% | activity | #A855F7 (purple) |
| **Active Orgs** | 847 | +12 | building | #00D9FF (cyan) |
| **API Uptime** | 99.98% | — | server | #22C55E (green) |

**Card Design:**
- Rounded card with subtle border (`rgba(255,255,255,0.08)`)
- Icon on left in a colored circle background
- Title in small caps (11px, #6B7280)
- Value in large bold text (24px)
- Change indicator in green/red with arrow icon

### Revenue Trend Chart

- Card with header: trendingUp icon + "Platform Revenue Trend"
- Area chart (height: 240px)
- Data: 12 months (Jan–Dec) from `sampleMrrHistory`
- Gradient fill: cyan (#00D9FF) at 40% opacity fading to transparent
- X-axis: month labels
- Y-axis: `$Xk` format (e.g., $200k, $300k, $400k)
- Tooltip: shows all MRR breakdown fields on hover
- Period toggle: "This Period" | "Previous Period"

### Revenue by Plan

- Card with header: pieChart icon + "Revenue by Plan"
- Donut chart (inner radius: 50, outer radius: 75)
- Three segments with colors from `sampleRevenueByPlan`:
  - Enterprise: #A855F7 (purple) — $245k, 47 customers
  - Pro: #00D9FF (cyan) — $128k, 156 customers
  - Starter: #22C55E (green) — $45k, 234 customers
- Legend below or beside chart with plan name, revenue, customer count

### Top Organizations Table

- Card with header: building icon + "Top Organizations by Revenue"
- Table with columns: Organization, MRR, Events (30d), Growth
- Sample data rows:
  | Organization | MRR | Events (30d) | Growth |
  |-------------|-----|--------------|--------|
  | Acme AI Corp | $45,000 | 2.1B | +15.7% |
  | Neural Networks Inc | $38,500 | 1.8B | +22.3% |
  | DeepMind Labs | $12,800 | 890M | +8.9% |
- MRR column: monospace font, bold
- Growth column: green text with "+" prefix
- "View All" link at bottom right of table card

---

## Dependencies & Notes for Agent

- **SUPER_ADMIN Guard:** All endpoints MUST check `actor.role === SUPER_ADMIN` before processing. No fallback for ORG_ADMIN or CUSTOMER.
- **Polling interval:** Frontend polls `GET /api/v1/platform/analytics/summary` every `PLATFORM_ANALYTICS_REFRESH_INTERVAL_SEC` seconds (default 60).
- **Prisma aggregation queries:** Platform-wide metrics require aggregation across all orgs. Use `SUM`, `COUNT`, `AVG` with `GROUP BY` as needed. Ensure indexes on `(org_id, status)` and `(event_timestamp)`.
- **MRR calculation:** Platform MRR is the sum of all active subscriptions' `mrr` values. Exclude `status = 'churned'` or `status = 'trial'`.
- **Event count:** Total Events (30d) counts all records in `usage_events` where `event_timestamp > now() - interval '30 days'`.
- **API Uptime:** Calculated as `(total_requests - failed_requests) / total_requests * 100` over the last 30 days. Source from API request logs or a dedicated uptime tracking table.
- **Top Organizations query:** `SELECT org_id, SUM(mrr) as total_mrr FROM subscriptions WHERE status = 'active' GROUP BY org_id ORDER BY total_mrr DESC LIMIT X`.
- **No WebSocket for Platform Analytics:** Unlike Organization Overview, platform analytics does not require WebSocket real-time push. Standard polling is sufficient given the aggregated nature of the data.
- **Caching:** Consider caching platform analytics results in Redis with a 60-second TTL to reduce database load, since this is a heavy aggregation query.
- **Audit logging:** This is a read-only feature — no audit logs are written for dashboard views.
- **Rate limiting:** Apply standard API rate limiting to all platform analytics endpoints (e.g., 60 requests/minute per SUPER_ADMIN).

---

## Future Enhancements (Out of Scope for v1)

- Drill-down from platform to organization (click top org → org detail)
- Export platform analytics report as PDF
- Custom date range picker for trend analysis
- Anomaly detection and alerting for platform metrics
- Real-time event streaming visualization
- Cohort analysis for new vs. churned organizations
