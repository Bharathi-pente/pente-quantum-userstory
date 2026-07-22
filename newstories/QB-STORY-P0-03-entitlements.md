# QuantumBilling User Story: Separate Metered Entitlements System [CODE-VERIFIED]

> **Verified against actual code.** Prisma schema has entitlement tables but the Go enforcement engine in `engine/internal/enforcement/` only evaluates usage limits (SOFT/HARD). This story builds the missing engine evaluation logic.

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P0-03 |
| **Sprint** | Sprint 3-6 |
| **Phase** | Enforcement |
| **Domain** | Usage Capping — Feature Flags |
| **Priority** | P0 — Critical |

---

## Title

**Separate Metered Entitlements System** — decouple usage capping from billing measurement with dedicated entitlement types, reset strategies, and enforcement API

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P0 — Critical |

---

## Description

**As a platform engineer**, I want to define entitlements independently from billing meters so that I can cap usage by daily limits, sliding windows, or feature flags without affecting billing measurement.

### Current Code — What Exists

**Prisma schema has the tables** (`control-plane/prisma/schema.prisma`):
```prisma
model EntitlementGrant {
    id              String
    customerId      String              @map("customer_id")
    featureId       String              @map("feature_id")
    status          EntitlementGrantStatus  // GRANTED | EXPIRED | REVOKED
    grantedAt       DateTime            @default(now()) @map("granted_at")
    expiresAt       DateTime?           @map("expires_at")
    // ...relations to features, customers
}

model Feature {
    id          String
    name        String
    featureKey  String   @unique @map("feature_key")
    description String?
}
```

**But engine enforcement** (`engine/internal/enforcement/limits.go`) only evaluates usage limits:
```go
// CURRENT — exact function in limits.go
func (l *Limits) Decide(ctx context.Context, customerID string, meterID string, estimatedTokens int64) (*Decision, error) {
    // Reads plan-level limits from customer.usage_limits table
    // Evaluates SOFT/HARD tiers
    // Returns: ALLOW, WARN, or BLOCK
    // NO entitlement types (Metered/Boolean/Static)
    // NO reset strategies (daily/weekly/sliding)
}
```

**And the enforcement handler** (`engine/internal/enforcement/handler.go`) still uses `StubWalletChecker`:
```go
// CURRENT — exact code in handler.go
// TODO(D-13): replace StubWalletChecker with real RedisWalletChecker
```

### Target State

```
Meter ──→ Billing Feature (pricing + invoicing)
   │
   └──→ Entitlement (capping + enforcement) ← NEW Go package
```

Entitlement types: `METERED` (usage cap), `BOOLEAN` (on/off), `STATIC` (JSON config)
Reset strategies: `BILLING_PERIOD`, `DAILY`, `WEEKLY`, `MONTHLY`, `SLIDING_WINDOW`

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P0-03 |
| **Sprint** | Sprint 3-6 |
| **Phase** | Enforcement |
| **Domain** | Usage Capping — Feature Flags |
| **Priority** | P0 — Critical |

---

## Title

**Separate Metered Entitlements System** — decouple usage capping from billing measurement with dedicated entitlement types, reset strategies, and enforcement API

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P0 — Critical |

---

## Description

**As a platform engineer**, I want to define **entitlements independently from billing meters** with their own reset periods, override semantics, and enforcement logic so that I can **cap usage by daily limits, sliding windows, or feature flags** without affecting billing measurement.

### Current State

The Prisma schema has entitlement tables (`EntitlementGrant`, `EntitlementPolicyVersion`, `catalog.features`, `catalog.plan_features`) — but the **engine enforcement** in `engine/internal/enforcement/` only evaluates usage limits (SOFT/HARD tiers from `customer.usage_limits`). There is no `entitlement` Go package. The enforcement hot path reads the same `meter_id` for both billing and capping — conflated concerns.

### Target State

Decouple "what we measure for billing" from "what we cap for enforcement":

```
Meter ──→ Billing Feature (pricing + invoicing)
   │
   └──→ Entitlement (capping + enforcement) ← NEW
```

**Entitlement types:**
| Type | Purpose | Example |
|------|---------|---------|
| **Metered** | Usage cap with balance tracking | "1M tokens/month, sliding 30-day window" |
| **Boolean** | On/off feature flag | "SAML SSO: enabled" |
| **Static** | Per-customer JSON config | `{ allowedModels: ["gpt-4"] }` |

**Reset strategies:**
| Strategy | Behavior |
|----------|----------|
| `BILLING_PERIOD` | Resets on subscription anniversary |
| `DAILY` | Resets at midnight UTC |
| `WEEKLY` | Resets on Monday 00:00 UTC |
| `MONTHLY` | Resets on 1st of month 00:00 UTC |
| `SLIDING_WINDOW` | Rolling N-day window (Redis sorted sets) |

---

## RBAC Roles

| Role | Can manage entitlements | Can override entitlements | Can view entitlements | Scope |
|------|------------------------|--------------------------|----------------------|-------|
| **SUPER_ADMIN** | Yes (any org) | Yes (any org) | Yes (any org) | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Yes (own org) | Yes (own org) | Own org only |
| **CUSTOMER** | No | No | Yes (own entitlements) | Own account only |
| **END_USER** | No | No | No | No access |

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Admin can create three entitlement types: **Metered** (usage cap with balance, e.g., "1M tokens/month"), **Boolean** (on/off, e.g., "SAML SSO: enabled"), **Static** (JSON config, e.g., `{ "allowedModels": ["gpt-4"] }`) |
| AC-2 | Metered entitlements support 5 reset strategies: `BILLING_PERIOD`, `DAILY`, `WEEKLY`, `MONTHLY`, `SLIDING_WINDOW` |
| AC-3 | Enforcement API checks entitlements independently from billing — exceeded Metered entitlement returns `BLOCKED` with descriptive message while billing continues counting |
| AC-4 | Boolean entitlements return `GRANTED` or `DENIED` based on plan/override configuration; admin can override per customer |
| AC-5 | Static entitlements return the configured JSON blob to the caller for client-side enforcement |
| AC-6 | Existing `customer.usage_limits` and `customer.limit_overrides` tables are migrated to the new entitlement schema with zero data loss |
| AC-7 | Enforcement P99 latency remains <5ms at 32 concurrent callers (DEC-010 compliant) |
| AC-8 | API supports batch entitlement checks: `POST /api/v1/entitlements/check` accepts multiple feature keys and returns status for each |

---

## Test Cases

### TC-01 — Create Metered entitlement with daily reset

**Given:** Authenticated ORG_ADMIN
**When:** POST `/api/v1/entitlements` `{ "name": "Daily Token Cap", "type": "METERED", "meter_id": "meter_001", "limit_value": 1000000, "reset_strategy": "DAILY" }`
**Then:** 201 returned; entitlement created; enforcement API checks limit against Redis counter

### TC-02 — Boolean entitlement blocks feature access

**Given:** Customer `cust_001` has Boolean entitlement `saml_sso` = DENIED
**When:** GET `/api/v1/entitlements/check?customer_id=cust_001&feature=saml_sso`
**Then:** 200 returned `{ "has_access": false, "reason": "Feature not included in current plan" }`

### TC-03 — Metered entitlement blocks at limit

**Given:** Customer `cust_001` has HARD limit of 1M tokens/month, current usage = 1,000,050
**When:** GET `/api/v1/entitlements/check?customer_id=cust_001&feature=api_access`
**Then:** 200 returned `{ "has_access": false, "limit_type": "HARD", "limit_value": 1000000, "current_usage": 1000050, "reason": "Monthly token limit exceeded" }`

### TC-04 — Sliding window enforcement

**Given:** Customer has sliding 30-day limit of 500K tokens; 500K used in last 29 days
**When:** New event arrives at enforcement check
**Then:** Redis ZREMRANGEBYSCORE removes events older than 30 days; ZCARD returns current window count; if ≥500K, enforcement returns BLOCKED

### TC-05 — Static entitlement returns config

**Given:** Customer `cust_001` has Static entitlement `allowed_models` with config `{ "models": ["gpt-4", "claude-3"] }`
**When:** GET `/api/v1/entitlements/check?customer_id=cust_001&feature=allowed_models`
**Then:** 200 returned `{ "has_access": true, "config": { "models": ["gpt-4", "claude-3"] } }`

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `POST` | `/api/v1/entitlements` | Create entitlement definition | JWT · OrgAdminGuard |
| `GET` | `/api/v1/entitlements` | List entitlements | JWT · OrgAdminGuard |
| `PATCH` | `/api/v1/entitlements/:id` | Update entitlement config | JWT · OrgAdminGuard |
| `DELETE` | `/api/v1/entitlements/:id` | Soft-delete entitlement | JWT · OrgAdminGuard |
| `POST` | `/api/v1/entitlements/check` | Batch entitlement check (accepts array of feature keys) | JWT · GatewayServiceGuard |
| `POST` | `/api/v1/customers/:customerId/entitlements/override` | Override entitlement for customer | JWT · OrgAdminGuard |

---

## Data Tables Used (Prisma — Already Exists)

| Table | Schema | Key Columns |
|-------|--------|-------------|
| `entitlement_catalog` | `entitlement` | `id, org_id, name, type (METERED\|BOOLEAN\|STATIC), meter_id?, limit_value?, reset_strategy?, config?` |
| `customer_entitlements` | `entitlement` | `id, customer_id, entitlement_id, status (GRANTED\|DENIED), override_value?, expires_at?` |
| `entitlement_grants` | `entitlement` | `id, customer_id, entitlement_id, balance, window_start, window_end` |
| `features` | `catalog` | Already exists — feature definitions |
| `plan_features` | `catalog` | Already exists — plan-to-feature associations |

---

## Code Changes Required

### 1. Engine — New `entitlement` enforcement package

**New directory:** `engine/internal/enforcement/entitlements/`

```go
// entitlement.go — Core entitlement types and evaluation

type EntitlementType string
const (
    EntMetered EntitlementType = "METERED"
    EntBoolean EntitlementType = "BOOLEAN"
    EntStatic  EntitlementType = "STATIC"
)

type ResetStrategy string
const (
    ResetBillingPeriod ResetStrategy = "BILLING_PERIOD"
    ResetDaily         ResetStrategy = "DAILY"
    ResetWeekly        ResetStrategy = "WEEKLY"
    ResetMonthly       ResetStrategy = "MONTHLY"
    ResetSlidingWindow ResetStrategy = "SLIDING_WINDOW"
)

type Entitlement struct {
    ID            string
    OrgID         string
    Name          string
    Type          EntitlementType
    MeterID       string  // nil for BOOLEAN/STATIC
    LimitValue    Decimal // for METERED
    ResetStrategy ResetStrategy
    Config        map[string]interface{} // for STATIC
}

type CheckResult struct {
    HasAccess    bool
    LimitType    string  // "SOFT" | "HARD"
    LimitValue   Decimal
    CurrentUsage Decimal
    Config       map[string]interface{}
    Reason       string
}
```

### 2. Engine — Entitlement Evaluator

**New file:** `engine/internal/enforcement/entitlements/evaluator.go`

```go
type EntitlementEvaluator struct {
    store EntitlementStore
    clock clock.Clock
}

func (e *EntitlementEvaluator) Check(ctx context.Context, customerID string, featureKey string) (*CheckResult, error) {
    // 1. Load entitlement definition from cache/Postgres
    // 2. Check if customer has override
    // 3. Evaluate based on type:
    //    - METERED: Check Redis counter against limit_value with reset strategy
    //    - BOOLEAN: Return GRANTED/DENIED from plan or override
    //    - STATIC: Return config blob
    // 4. Return CheckResult
}

func (e *EntitlementEvaluator) BatchCheck(ctx context.Context, customerID string, featureKeys []string) (map[string]*CheckResult, error) {
    // Batch version — reduces Redis round-trips
}
```

### 3. Engine — Sliding Window Support

**New file:** `engine/internal/enforcement/entitlements/sliding.go`

```go
func (e *EntitlementEvaluator) checkSlidingWindow(ctx context.Context, customerID, meterID string, windowDays int) (Decimal, error) {
    // Use Redis sorted sets:
    // ZREMRANGEBYSCORE wallet:{id}:{dimension} 0 {now - window}
    // ZCARD wallet:{id}:{dimension}
    // Compare count against limit
    //
    // O(log n) for insert, O(log n) for range deletion, O(1) for count
}
```

### 4. Engine — Wire into Existing Enforcement Handler

**File:** `engine/internal/enforcement/handler.go`
- Replace `StubWalletChecker` with real `EntitlementEvaluator`
- Add entitlement check alongside existing usage limit check
- Batch entitlement endpoint

### 5. BFF — Entitlements Module

**New directory:** `control-plane/src/entitlements/`
- `entitlements.controller.ts` — CRUD + check endpoints
- `entitlements.service.ts` — business logic with Prisma
- `entitlements.module.ts` — NestJS module

### 6. Web UI — Entitlement Management

**New directory:** `web/app/entitlements/`
- Entitlement definition list/create/edit forms
- Customer entitlement override interface
- Feature flag management per plan

### 7. Migration

**Script:** Migrate existing `customer.usage_limits` and `customer.limit_overrides` to new entitlement schema

---

## Performance Targets

| Operation | Target | Method |
|-----------|--------|--------|
| Metered entitlement check | <5ms P99 | Redis pipeline — single round-trip |
| Boolean entitlement check | <2ms P99 | In-memory cache from Postgres |
| Static entitlement check | <2ms P99 | In-memory cache from Postgres |
| Sliding window check | <5ms P99 | Redis sorted set (ZREMRANGEBYSCORE + ZCARD) |
| Batch check (10 features) | <10ms P99 | Redis pipeline — multi-key |

---

## Estimate

**3-4 sprints** (design 1, engine enforcement 1, Prisma + BFF 1, Web UI 1)

**Dependencies:** Recommended: QB-STORY-P0-02 (groupBy) — entitlements can reference meters with or without filters
