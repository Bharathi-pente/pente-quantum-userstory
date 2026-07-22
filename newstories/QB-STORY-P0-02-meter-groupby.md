# QuantumBilling User Story: Meter groupBy / Filter System [CODE-VERIFIED]

> **Verified against actual code.** The `matchDimension()` and `matchOptionalTokenDimension()` functions in `engine/internal/rating/resolver.go` are currently hardcoded. This story makes them configurable from meter definitions.

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P0-02 |
| **Sprint** | Sprint 3-6 |
| **Phase** | Catalog |
| **Domain** | Metering — Dimension-Based Pricing |
| **Priority** | P0 — Critical |

---

## Title

**Meter groupBy / Filter System** — define one meter with configurable groupBy dimensions and price each dimension combination via filters

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P0 — Critical |

---

## Description

**As a product manager**, I want to define **one meter with groupBy dimensions** (model, token_type, region) and price each combination via filters so that I can manage 100+ model×type combinations without creating 100 separate meters.

### Current Code — Exact Functions

In `engine/internal/rating/resolver.go`, the dimension matching is **hardcoded**:

```go
// CURRENT — exact functions in resolver.go

// Only matches ModelName:
func matchDimension(rowValue, reqValue string) bool {
    return strings.TrimSpace(rowValue) == strings.TrimSpace(reqValue)
}

// Only matches TokenType (empty = wildcard):
func matchOptionalTokenDimension(rowValue, reqValue string) bool {
    if reqValue == "" { return true }
    return matchDimension(rowValue, reqValue)
}

// Used in rate resolution (waterfall step resolution):
// Rate has: ModelName, TokenType fields
// Request has: ModelName, TokenType fields
// Only these TWO dimensions are matched — nothing else.
```

In `engine/internal/invoice/generate.go`, the `rateSegmentUsage()` function groups usage by:
```go
// CURRENT — exact grouping in rateSegmentUsage():
key := row.Model + "\x00" + row.TokenType  // hardcoded model+token_type
buckets[key] = &usageBucket{model: row.Model, tokenType: row.TokenType, ...}
```

The Prisma `Meter` model has NO `group_by` field:
```prisma
model Meter {
    id          String
    orgId       String
    name        String
    eventType   String      @map("event_type")
    aggregation MeterAggregation
    field       String?
    status      MeterStatus @default(DRAFT)
    // ...NO group_by field...
}
```

**The problem:** For 10 AI models × 3 token types → **30 meters** with 30 separate pricing entries.

### Target State

A single meter with `groupBy` dimensions:
```yaml
meter_tokens_total:
  groupBy:
    model: $.model          # JSONPath from event
    token_type: $.type      # JSONPath from event
```

Features reference the meter with filter criteria:
```js
Feature A: meter=meter_tokens_total, filters: {model:"gpt-4", type:"input"}, price: 0.000025
Feature B: meter=meter_tokens_total, filters: {model:"gpt-4", type:"output"}, price: 0.000050
```

> Aligned with ADR-001 (2026-07-01).

---

## Story ID & Metadata

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P0-02 |
| **Sprint** | Sprint 3-6 |
| **Phase** | Catalog |
| **Domain** | Metering — Dimension-Based Pricing |
| **Priority** | P0 — Critical |

---

## Title

**Meter groupBy / Filter System** — define one meter with configurable groupBy dimensions and price each dimension combination via filters

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P0 — Critical |

---

## Description

**As a product manager**, I want to define **one meter with groupBy dimensions** (model, token_type, region) and then **price each dimension combination differently via filters** so that I can manage **100+ model×type combinations without creating 100 separate meters**.

### Current State

The rating resolver in `engine/internal/rating/resolver.go` has **hardcoded** dimension matching:

```go
func matchDimension(rowValue, reqValue string) bool {
    return strings.TrimSpace(rowValue) == strings.TrimSpace(reqValue)
}
func matchOptionalTokenDimension(rowValue, reqValue string) bool {
    if reqValue == "" { return true }
    return matchDimension(rowValue, reqValue)
}
```

These only work on `ModelName` and `TokenType` fields — no configurable `groupBy` or `filters` on arbitrary event properties. The Prisma `Meter` model has no `group_by` field.

**The problem:** For 10 AI models × 3 token types, you need **30 separate meters**. For 50 models × 3 types, **150 meters**. This doesn't scale.

### Target State

A single meter with `groupBy` dimensions feeds multiple features/prices via filter criteria:

```yaml
meter_tokens_total:
  event_type: ai.usage
  field: total_tokens
  aggregation: SUM
  groupBy:
    model: $.model
    token_type: $.type
```

Features reference the meter with filters:
```js
Feature 'gpt4_input_tokens':
  meter: meter_tokens_total
  filters: { model: "gpt-4", token_type: "input" }
  price: 0.000025

Feature 'gpt4_output_tokens':
  meter: meter_tokens_total
  filters: { model: "gpt-4", token_type: "output" }
  price: 0.000050
```

---

## RBAC Roles

| Role | Can manage meters/groupBy | Can view meters | Scope |
|------|--------------------------|-----------------|-------|
| **SUPER_ADMIN** | Yes (any org) | Yes (any org) | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Yes (own org) | Own org only |
| **CUSTOMER** | No | Read-only (own plan's meters) | Own account only |
| **END_USER** | No | No | No access |

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Admin can create a meter with `groupBy` JSON config specifying dimension keys and JSONPath expressions (e.g., `{ "model": "$.model", "token_type": "$.type" }`) |
| AC-2 | Admin can create a feature that references a meter with filter criteria (e.g., `{ "model": "gpt-4", "token_type": "input" }`) and assign a price to that filter combination |
| AC-3 | When an event arrives, the engine resolves the correct feature by matching event property values against all filter combinations for that meter using the configured JSONPath expressions |
| AC-4 | Unmatched events (no filter combination matches) are rated as `unrated` and logged with the specific unmatched dimension values — never billed at implicit zero |
| AC-5 | The system supports at least 5 groupBy dimensions and 100 filter combinations per meter without performance degradation (<1ms per event overhead) |
| AC-6 | Migration script converts existing hardcoded model/token_type meters to use the new groupBy system |
| AC-7 | API returns clear error messages when a filter combination conflicts with an existing one on the same meter |

---

## Test Cases

### TC-01 — Create meter with groupBy and price with filters

**Given:** Authenticated ORG_ADMIN
**When:** POST `/api/v1/meters` `{ "name": "Total Tokens", "event_type": "ai.usage", "aggregation": "SUM", "field": "total_tokens", "groupBy": { "model": "$.model", "token_type": "$.type" } }`
**Then:** 201 returned; meter created with groupBy config
**When:** POST `/api/v1/meters/:meterId/features` `{ "name": "GPT-4 Input", "filters": { "model": "gpt-4", "token_type": "input" }, "price": 0.000025 }`
**Then:** 201 returned; feature created with filter mapping

### TC-02 — Event resolved to correct filter combination

**Given:** Meter has filter combos for gpt-4/input ($0.03/1K) and gpt-4/output ($0.06/1K)
**When:** Event arrives with `{ "model": "gpt-4", "type": "input", "total_tokens": 1000 }`
**Then:** Engine matches `gpt-4/input` filter; charge = 1000 × ($0.03/1000) = $0.03

### TC-03 — Unmatched event flagged as unrated

**Given:** Meter has filter combos only for gpt-4 and claude-3
**When:** Event arrives with `{ "model": "gemini-pro", "type": "input", "total_tokens": 1000 }`
**Then:** Engine flags event as `unrated`; logged with `unmatched_dimensions: { model: "gemini-pro", token_type: "input" }`

### TC-04 — Conflicting filter combination rejected

**Given:** Meter has filter `{ "model": "gpt-4", "token_type": "input" }` already configured
**When:** POST same filter combination
**Then:** 409 CONFLICT returned: "Filter combination { model: gpt-4, token_type: input } already exists on this meter"

### TC-05 — 100 filter combinations on one meter

**Given:** Meter has 100 filter combinations across 5 dimensions
**When:** 1000 events are ingested with various dimension values
**Then:** All events resolve correctly; average resolution time <1ms per event

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `POST` | `/api/v1/meters` | Create meter with optional groupBy | JWT · OrgAdminGuard |
| `GET` | `/api/v1/meters` | List meters (groupBy included in response) | JWT · OrgAdminGuard |
| `PATCH` | `/api/v1/meters/:meterId` | Update meter (groupBy immutable after events recorded) | JWT · OrgAdminGuard |
| `POST` | `/api/v1/meters/:meterId/features` | Create feature with filter criteria and price | JWT · OrgAdminGuard |
| `GET` | `/api/v1/meters/:meterId/features` | List features with filter mappings | JWT · OrgAdminGuard |
| `DELETE` | `/api/v1/meters/:meterId/features/:featureId` | Remove feature | JWT · OrgAdminGuard |

---

## Data Tables Used (Prisma)

| Table | Schema | Current | Change |
|-------|--------|---------|--------|
| `meters` | `catalog` | No `group_by` field | ADD `groupBy Json? @map("group_by")` |
| `features` | `catalog` (NEW) | Doesn't exist | CREATE with `meterId, name, filters Json, price Decimal(38,9)` |

### Prisma Migration

```prisma
// Add groupBy to Meter
model Meter {
  // ...existing fields...
  groupBy     Json?   @map("group_by")
  // ...existing relations...
}

// New Feature model
model Feature {
  id        String   @id @default(uuid()) @db.Uuid
  meterId   String   @map("meter_id") @db.Uuid
  name      String
  filters   Json     // e.g., {"model": "gpt-4", "token_type": "input"}
  price     Decimal  @db.Decimal(38, 9)
  unit      String?  // e.g., "per 1K tokens"
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  meter Meter @relation(fields: [meterId], references: [id])

  @@unique([meterId, filters])  // prevent conflicting filters
  @@index([meterId])
  @@map("features")
  @@schema("catalog")
}
```

---

## Code Changes Required

### 1. Engine — Replace hardcoded matchDimension with configurable filter resolution

**File:** `engine/internal/rating/resolver.go`

**Current (hardcoded):**
```go
func matchDimension(rowValue, reqValue string) bool {
    return strings.TrimSpace(rowValue) == strings.TrimSpace(reqValue)
}
```

**Target (configurable from meter definition):**
```go
type FilterCriteria map[string]string  // e.g., {"model": "gpt-4", "token_type": "input"}

// ResolveFeature finds the matching feature for an event's dimension values
func ResolveFeature(features []Feature, eventProperties map[string]string, groupByConfig map[string]string) (*Feature, error) {
    // groupByConfig: {"model": "$.model", "token_type": "$.type"}
    // Extract event dimension values using JSONPath from groupByConfig
    
    for _, feature := range features {
        match := true
        for dimension, jsonPath := range groupByConfig {
            eventValue := eventProperties[jsonPath]
            expectedValue := feature.Filters[dimension]
            
            if eventValue != expectedValue {
                match = false
                break
            }
        }
        if match {
            return &feature, nil
        }
    }
    return nil, fmt.Errorf("no matching feature for dimensions: %v", eventProperties)
}
```

### 2. Engine — Extend rateSegmentUsage for configurable grouping

**File:** `engine/internal/invoice/generate.go`

Current `rateSegmentUsage()` groups by hardcoded model+token_type. Extend to read `groupBy` from meter definition and group dynamically.

### 3. BFF — GroupBy CRUD + Filter Validation

**File:** `control-plane/src/catalog/catalog.service.ts`
- Add `createFeature()`, `listFeatures()`, `deleteFeature()` methods
- Validate filter combinations don't conflict (unique constraint)
- Include groupBy in meter response DTOs

### 4. Web UI — GroupBy Configuration

**New:** `web/app/meters/[id]/groupby-config.tsx`
- Dynamic JSONPath builder for groupBy dimensions
- Feature/filter table with inline price editing
- Validation: no duplicate filter combinations

### 5. Migration Script

**New:** `scripts/migrate-meters-groupby.ts`
- For each meter that has hardcoded model/token_type dimension usage, create equivalent groupBy config
- Create features for each existing dimension combination

---

## Performance Impact

| Scenario | Before | After |
|----------|--------|-------|
| 10 models × 3 types | 30 meters | **1 meter** |
| 50 models × 3 types | 150 meters | **1 meter** |
| Filter resolution per event | N/A (hardcoded) | <1ms with 100 filter combinations |
| Unmatched events | Silent unclear behavior | Explicit `unrated` exception |

---

## Estimate

**3-4 sprints** (design 1, engine resolver changes 1, Prisma + BFF 1, Web UI 1)

**Dependencies:** None — adds new fields to meter, existing meters work unchanged
