# QuantumBilling — Improvement Plan

**Date:** 2026-07-08  
**Repository:** `C:\projects\pente_projects\quantumbilling`  
**Spec:** `C:\projects\pente_projects\erd_documentation\docs_user_story\userStory`

---

## Priority Summary

| Priority | Count | Description |
|---|---|---|
| 🔴 P0 — Critical | 2 | User-org assignment broken; seed data only |
| 🟠 P1 — High | 5 | Missing catalog UI pages (APIs exist) |
| 🟡 P2 — Medium | 8 | Backend stubs, Go engine not running |
| ⚪ P3 — Low | 2 | Polish / onboarding wizard |

---

## 🔴 P0 — Critical (Blocks Real Usage)

### 1. No Dynamic User-Org Assignment

**Problem:** `DEV_USER_MAP` in `web/src/app/api/auth/[...nextauth]/route.ts` hardcodes all 4 test users to seed org UUID `00000000-0000-4000-8000-000000000001`. Creating a new organization via UI works, but no user can log into it — their `org_id` is stuck at the hardcoded value.

**Root Cause:** Keycloak users have no `org_id` attribute, and there's no database table mapping usernames to orgs.

**Fix:**
- [ ] Add `identity.user_org_assignments` table (Prisma model: `username` → `org_id`, `customer_id`, `end_user_id`)
- [ ] Create `AssignmentService` + `AssignmentController` in NestJS
- [ ] `POST /api/v1/user-assignments` — SUPER_ADMIN assigns user to org
- [ ] `GET /api/v1/user-assignments` — list all assignments
- [ ] `DELETE /api/v1/user-assignments/:username` — remove assignment
- [ ] Update NextAuth `jwt` callback to query `GET /api/v1/user-assignments?username=...` as primary source
- [ ] Keep `DEV_USER_MAP` as fallback only (when API unavailable)
- [ ] Create `/assignments` UI page for SUPER_ADMIN

**Files to create:**
- `control-plane/src/identity/assignment.service.ts`
- `control-plane/src/identity/assignment.controller.ts`
- `web/src/app/assignments/page.tsx`

**Files to modify:**
- `prisma/schema.prisma` — add `UserOrgAssignment` model
- `control-plane/src/identity/identity.module.ts` — register controller + service
- `web/src/app/api/auth/[...nextauth]/route.ts` — query assignments API
- `web/src/components/Sidebar.tsx` — add Assignments nav item

### 2. Seed Data Is the Only Data

**Problem:** Without running `scripts/seed-dev.sql`, no orgs/customers/end-users/API keys exist. Creating data via UI works but can't be used for login.

**Fix:** Same as #1 — once user-org assignment is dynamic, seed data becomes optional bootstrap only.

---

## 🟠 P1 — High (Missing UI, APIs Exist)

All controllers are implemented in `control-plane/src/catalog/`. Only need Next.js pages.

### 3. Meters Page — `/meters`

**API:** `catalog/meter/meter.controller.ts` — `POST/GET/PATCH/DELETE /meters`  
**Purpose:** Define billing meters (event_type, aggregation method, unit_label)  
**Missing:** UI page with table + create/edit form

### 4. Products Page — `/products`

**API:** `catalog/product/product.controller.ts` — `POST/GET/PATCH /products`  
**Purpose:** Product catalog (name, description, SKU) for grouping plans  
**Missing:** UI page with table + create form

### 5. Pricing & Plans Page — `/pricing`

**API:** `catalog/plan/plan.controller.ts` — `POST/GET /plans`, `POST /pricing-models`, `GET /plans/:id/preview`  
**Purpose:** Pricing models (7 types) + plans (billing_period, base_amount, trial_days)  
**Missing:** UI page with pricing model CRUD + plan configuration

### 6. Contracts Page — `/contracts`

**API:** `catalog/contract/contract.controller.ts` — `GET/POST /contracts`  
**Purpose:** Enterprise contracts (commit_amount, rate card linking, DRAFT→ACTIVE→EXPIRED)  
**Missing:** UI page with contract lifecycle management

### 7. Subscriptions Page — `/subscriptions`

**API:** `catalog/contract/contract.controller.ts` — `GET/POST /subscriptions`, `DELETE /subscriptions/:id`  
**Purpose:** Customer→Plan assignment, trial→active→cancel lifecycle, T-3 anniversary clamping  
**Missing:** UI page with subscription table + create/cancel actions

---

## 🟡 P2 — Medium (Backend Gaps)

### 8. Reports API — Un-stub

**Problem:** `ops/ops.controller.ts` returns `note: 'scheduled_reports table pending Prisma migration (D-18)'`

**Fix:**
- [ ] Run Prisma migration for `reporting.scheduled_reports` table
- [ ] Implement real CRUD in `OpsService`
- [ ] Wire warehouse export (CR-13) for S3/Snowflake/BigQuery Parquet

### 9. Alerts API — Un-stub

**Problem:** Same as reports — stub placeholder responses.

**Fix:**
- [ ] Run Prisma migration for alert tables
- [ ] Implement real alert rule CRUD (threshold/anomaly/budget)
- [ ] Add notification channel integration (email/Slack/webhook/PagerDuty)

### 10. Credits & Wallet — NestJS Read API

**Problem:** No dedicated NestJS controller. Go billing worker owns `billing.credit_ledger` and `billing.wallet_transactions`. UI reads these via BFF proxy but no explicit API.

**Fix:**
- [ ] `GET /api/v1/credits` — list credits with FEFO ordering
- [ ] `POST /api/v1/credits` — grant credit (forwards to Go worker or writes directly)
- [ ] `GET /api/v1/wallet/:customerId` — wallet balance + transaction history

### 11. Go Analytics Engine Not Running

**Problem:** 5 pages depend on Go phase-4 analytics APIs and show empty/zero data:
- Organization Overview (`/dashboard`)
- End User Dashboard (`/my-usage`)
- End User Events (`/my-usage/events`)
- Platform Analytics (`/platform/analytics`)
- Team Usage (`/dashboard/team-usage`)

**Fix:**
- [ ] Install Go toolchain (https://go.dev/dl/)
- [ ] `cd engine && go mod tidy`
- [ ] Run `analytics-worker` (Kafka → ClickHouse ingestion)
- [ ] Run `analytics-api` (phase-4 HTTP API)
- [ ] Verify BFF `bff-proxy.controller.ts` routes correctly to Go APIs

### 12. Rate Limit Policy CRUD

**Problem:** `rate-limits/page.tsx` calls `GET/POST/DELETE /api/v1/rate-limits` but no NestJS controller exists. Enforcement runs on Go engine Redis hot path.

**Fix:**
- [ ] Create `RateLimitController` with `GET/POST/DELETE /rate-limits`
- [ ] Back by Postgres `developer.rate_limit_policies` table
- [ ] Sidebar: add under Management section

### 13. Recommendations — Wire to Real API

**Problem:** `recommendations/page.tsx` uses hardcoded `MOCK_RECS` array.

**Fix:**
- [ ] `GET /api/v1/recommendations` endpoint
- [ ] Back by `analytics.ai_recommendations` table (Go worker populates)

### 14. Payments UI

**Problem:** `payment/payment.controller.ts` exists but no `/payments` page.

**Fix:**
- [ ] Create `/payments` page with payment history table + manual recording form
- [ ] Show status badges (pending/succeeded/failed), amounts, linked invoices
- [ ] Sidebar: add under Management section

### 15. Audit & Compliance UI

**Problem:** `ops/ops.controller.ts` has `GET /organizations/:orgId/audit-logs` but no dedicated UI page.

**Fix:**
- [ ] Create `/audit` page with searchable audit log table
- [ ] Filters: resource_type, date range, action, actor
- [ ] Sidebar: add under Management section (SUPER_ADMIN only)

---

## ⚪ P3 — Low (Polish)

### 16. Onboarding Wizard

**Problem:** Current flow requires navigating 7+ separate pages to fully onboard (org → customer → end-user → meter → product → plan/pricing → subscription). No guidance.

**Fix:**
- [ ] Create `/onboarding` page with step progress indicator
- [ ] Chain: Create Customer → Create End User → Create Meter → Create Plan → Create Subscription
- [ ] Show completion checkmarks
- [ ] Sidebar: add under Management section

### 17. AI Chatbot Verification

**Problem:** Chatbot widget is a shared component (`ChatWidget.tsx`). Needs verification it loads correctly and responds with role-aware context.

**Fix:**
- [ ] Verify `ChatWidget` renders in bottom-right corner
- [ ] Test: SUPER_ADMIN asks "show me platform revenue" → gets platform data
- [ ] Test: END_USER asks "how many tokens did I use" → gets personal data

---

## Complete Checklist

### P0 — Critical
- [ ] `UserOrgAssignment` Prisma model
- [ ] `assignment.service.ts` + `assignment.controller.ts`
- [ ] Update `identity.module.ts`
- [ ] Update NextAuth `route.ts` to query assignments API
- [ ] `web/src/app/assignments/page.tsx`
- [ ] Add "Assignments" to Sidebar (SUPER_ADMIN)

### P1 — Catalog Pages
- [ ] `web/src/app/meters/page.tsx`
- [ ] `web/src/app/products/page.tsx`
- [ ] `web/src/app/pricing/page.tsx`
- [ ] `web/src/app/contracts/page.tsx`
- [ ] `web/src/app/subscriptions/page.tsx`
- [ ] Add all 5 to Sidebar

### P2 — Backend
- [ ] Un-stub reports API (Prisma migration + service)
- [ ] Un-stub alerts API (Prisma migration + service)
- [ ] Credits/wallet read API controller
- [ ] Rate limits CRUD controller
- [ ] Recommendations API endpoint
- [ ] Go analytics engine running
- [ ] `web/src/app/payments/page.tsx`
- [ ] `web/src/app/audit/page.tsx`

### P3 — Polish
- [ ] `web/src/app/onboarding/page.tsx`
- [ ] Verify AI chatbot widget

---

## File Count

| Category | To Create | To Modify |
|---|---|---|
| Prisma | 0 | 1 |
| NestJS (control-plane) | 2 | 1 |
| Next.js (web) | 14 | 2 |
| **Total** | **16** | **4** |

---

## Current vs Target Role-to-Page Matrix

| Role | Currently Sees | Missing |
|---|---|---|
| **SUPER_ADMIN** | 20 pages | Assignments, Meters, Products, Pricing, Contracts, Subscriptions, Payments, Audit |
| **ORG_ADMIN** | 17 pages | Meters, Products, Pricing, Contracts, Subscriptions, Payments |
| **CUSTOMER** | 7 pages | Subscriptions (view only) |
| **END_USER** | 2 pages | — |
| **DEVELOPER** | 1 page | — |
