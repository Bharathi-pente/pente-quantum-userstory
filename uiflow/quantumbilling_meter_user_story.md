# QuantumBilling User Story: Meter — define and manage billing meters for usage-based pricing

---

## Story ID & Sprint

**QB-STORY-003** · Sprint 2 · Phase: Feature

---

## Title

**Meter — define and manage billing meters for usage-based pricing**

---

## Badges

<div style="display:flex;gap:8px;flex-wrap:wrap;margin-bottom:.5rem">
  <span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;letter-spacing:.3px;background:#EEEDFE;color:#3C3489">Backend</span>
  <span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;letter-spacing:.3px;background:#E1F5EE;color:#085041">UI</span>
  <span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;letter-spacing:.3px;background:#FAEEDA;color:#633806">Auth / RBAC</span>
  <span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;letter-spacing:.3px;background:#F1EFE8;color:#444441">Priority: P0</span>
</div>

---

## Description

Based on `catalog.meters` — `id, org_id, name, event_type, aggregation, field`. Meter is the core usage-tracking entity in QuantumBilling.

> **As an ORG_ADMIN**, I want to define meters that track usage events in my application (e.g., API calls, storage GB, seats), so that QuantumBilling can ingest consumption data and generate accurate usage-based invoices.

Key capabilities:
- ORG_ADMIN can create meters: `name`, `event_type`, `aggregation` (SUM | COUNT | AVG | GAUGE), `field` (the event property to meter)
- Meters are scoped per `org_id` (not per tenant)
- Meters can be updated (name, description) but not their `event_type` or `aggregation` once events have been recorded
- Meters can be deactivated (`status = INACTIVE`) but not deleted if linked to active price plans
- SUPER_ADMIN can manage meters for any org
- Ingest usage events via `POST /api/v1/meters/:meterId/events` — event includes `value`, `timestamp`, `idempotency_key`
- Idempotency: duplicate events with same `idempotency_key` within 24h are deduplicated
- State machine: DRAFT → ACTIVE → INACTIVE (terminal)

---

## RBAC Roles

| Role | Can create / manage meters | Can ingest events | Can view summary | Scope |
|------|----------------------------|-------------------|------------------|-------|
| **SUPER_ADMIN** | Yes — any org | Yes | Yes | Platform-wide |
| **ORG_ADMIN** | Yes — own org only | Yes (own meter keys) | Yes — own org | Own org only |
| **CUSTOMER** | No | No | No | Read-only list of own org's meters |
| **END_USER** | No | No | No | No access |

---

## Acceptance Criteria

1. ORG_ADMIN can create a meter with `name`, `event_type`, `aggregation` (SUM | COUNT | AVG | GAUGE), and optional `field` (the event JSON path to aggregate).
2. Creating a meter without any recorded events allows full edit of all fields including `event_type` and `aggregation`.
3. Once events have been recorded against a meter, `event_type` and `aggregation` become immutable.
4. A meter with `status = DRAFT` transitions to `status = ACTIVE` automatically upon ingestion of its first usage event.
5. ORG_ADMIN or SUPER_ADMIN can deactivate a meter (`status = INACTIVE`); no new events are accepted for an INACTIVE meter.
6. A meter linked to active `catalog.pricing_models` or `catalog.charges` cannot be deleted; a 409 is returned with error `METER_LINKED_TO_PRICING`.
7. SUPER_ADMIN can perform all meter operations (create, update, deactivate) on behalf of any org.
8. `POST /api/v1/meters/:meterId/events` accepts `{value, timestamp, idempotency_key}`; duplicate events with the same `idempotency_key` within 24 hours are deduplicated (200, not 201).
9. List events endpoint supports pagination (`page`, `limit`) and date range filtering (`from`, `to`).
10. `GET /api/v1/meters/:meterId/summary` returns aggregated usage for a given billing period.

---

## Test Cases

### TC-01 — Happy path: create meter and ingest first event

**Given:** authenticated ORG_ADMIN for org `acme`
**When:** POST `/api/v1/meters` `{name: "API Calls", event_type: "api.call", aggregation: "COUNT", field: null}`
**Then:** 201 returned, meter created with status `DRAFT`
**When:** POST `/api/v1/meters/:meterId/events` `{value: 1500, timestamp: "2026-06-25T10:00:00Z", idempotency_key: "evt-001"}`
**Then:** 201 returned, meter status transitions to `ACTIVE`, event recorded

---

### TC-02 — Duplicate event idempotency

**Given:** an event with `idempotency_key = "evt-001"` was already ingested for this meter within the last 24h
**When:** POST `/api/v1/meters/:meterId/events` `{value: 1500, timestamp: "2026-06-25T10:00:00Z", idempotency_key: "evt-001"}`
**Then:** 200 returned (not 201), no duplicate event created

---

### TC-03 — Reject event for INACTIVE meter

**Given:** meter has status `INACTIVE`
**When:** POST `/api/v1/meters/:meterId/events` `{value: 100, timestamp: "2026-06-25T10:00:00Z", idempotency_key: "evt-002"}`
**Then:** 409 `METER_INACTIVE` returned, no event recorded

---

### TC-04 — Immutable aggregation after events recorded

**Given:** meter has recorded events, aggregation = `COUNT`
**When:** PATCH `/api/v1/meters/:meterId` `{aggregation: "GAUGE"}`
**Then:** 422 `METER_AGGREGATION_IMMUTABLE` returned, no fields changed

---

### TC-05 — Deactivate meter

**Given:** meter `api-calls` is `ACTIVE` and linked to a pricing model
**When:** DELETE `/api/v1/meters/:meterId`
**Then:** 200 returned, meter status set to `INACTIVE`, subsequent event ingest returns 409 `METER_INACTIVE`

---

### TC-06 — RBAC escalation attempt — END_USER cannot create

**Given:** actor role is `END_USER`
**When:** POST `/api/v1/meters`
**Then:** 403 `FORBIDDEN` — guard rejects before service layer

---

### TC-07 — SUPER_ADMIN manages other org's meter

**Given:** SUPER_ADMIN is authenticated; meter belongs to org `acme`
**When:** PATCH `/api/v1/orgs/:orgId/meters/:meterId` `{name: "Updated name"}`
**Then:** 200 returned, meter updated

---

### TC-08 — List events with pagination and date filter

**Given:** 50 events exist for this meter
**When:** GET `/api/v1/meters/:meterId/events?page=2&limit=10&from=2026-06-01T00:00:00Z&to=2026-06-30T23:59:59Z`
**Then:** 200 returned, 10 events (items 11-20), includes `totalCount=50`, `hasNextPage=true`

---

## API Endpoints

### POST `/api/v1/meters`
Create a new meter for the authenticated org.

- **Auth:** JWT · Guard: `OrgAdminGuard`
- **Body:** `{name, event_type, aggregation, field?}`
- **Response:** 201 `{meterId, name, event_type, aggregation, field, status: "DRAFT", createdAt}`

---

### GET `/api/v1/meters`
List all meters for the org (or all orgs for SUPER_ADMIN).

- **Auth:** JWT · Guard: `AuthenticatedGuard`
- **Query:** `?status=ACTIVE&page=1&limit=20`
- **Response:** 200 `{items: [...], totalCount, page, limit, hasNextPage}`

---

### GET `/api/v1/meters/:meterId`
Get full details of a single meter.

- **Auth:** JWT · Guard: `OrgMemberGuard`
- **Response:** 200 `{meterId, name, event_type, aggregation, field, status, lastEventAt, createdAt, updatedAt}`

---

### PATCH `/api/v1/meters/:meterId`
Update meter name and/or field. `event_type` and `aggregation` are immutable once events exist.

- **Auth:** JWT · Guard: `OrgAdminGuard`
- **Body:** `{name?, field?}`
- **Response:** 200 updated meter object
- **Errors:** 422 `METER_AGGREGATION_IMMUTABLE`, 404 `METER_NOT_FOUND`

---

### DELETE `/api/v1/meters/:meterId`
Soft-deactivate a meter. Sets `status = INACTIVE`. Meter is not hard-deleted.

- **Auth:** JWT · Guard: `OrgAdminGuard`
- **Response:** 200 `{status: "INACTIVE"}`
- **Errors:** 409 `METER_LINKED_TO_PRICING` (cannot deactivate if linked to active pricing models)

---

### POST `/api/v1/meters/:meterId/events`
Ingest a single usage event for a meter. Uses meter-specific API key auth.

- **Auth:** Meter API Key (header `X-Meter-Api-Key`) · Guard: `MeterApiKeyGuard`
- **Body:** `{value: number, timestamp: ISO8601, idempotency_key: string}`
- **Response:** 201 event created · 200 if deduplicated
- **Errors:** 409 `METER_INACTIVE`, 429 `MAX_EVENTS_PER_METER_PER_DAY_EXCEEDED`

---

### GET `/api/v1/meters/:meterId/events`
List ingested events with pagination and date range filter.

- **Auth:** JWT · Guard: `OrgAdminGuard`
- **Query:** `?page=1&limit=20&from=ISO8601&to=ISO8601`
- **Response:** 200 `{items: [{id, value, timestamp, idempotency_key, createdAt}], totalCount, page, limit, hasNextPage}`

---

### GET `/api/v1/meters/:meterId/summary`
Return aggregated usage totals for a billing period.

- **Auth:** JWT · Guard: `OrgAdminGuard`
- **Query:** `?billing_period_start=ISO8601&billing_period_end=ISO8601`
- **Response:** 200 `{meterId, totalValue, eventCount, billingPeriodStart, billingPeriodEnd}`

---

## Data Tables Used

Based on `catalog.meters` — `id, org_id, name, event_type, aggregation, field`.

| Table | Operation | Key columns |
|-------|-----------|-------------|
| `catalog.meters` | INSERT · SELECT · UPDATE | `id, org_id, name, event_type, aggregation, field, status, last_event_at, created_at, updated_at` |
| `usage_events` | INSERT · SELECT | `id, meter_id, org_id, value, event_timestamp, idempotency_key, created_at` |
| `identity.organizations` | SELECT | `id, name, status` |
| `identity.users` | SELECT | `id, org_id, role_id` |
| `catalog.pricing_models` | SELECT | `id, org_id, meter_id, status` |
| `catalog.charges` | SELECT | `id, meter_id, status` |
| `developer.api_keys` | SELECT | `id, org_id, key_hash, key_prefix` |

---

## State Machine — Meter Lifecycle

```
DRAFT  →  ACTIVE  →  INACTIVE
          (terminal)
```

**Transitions:**

| From | To | Trigger |
|------|----|---------|
| `DRAFT` | `ACTIVE` | First usage event ingested via `POST /events` |
| `ACTIVE` | `INACTIVE` | Admin calls `DELETE /meters/:meterId` (soft deactivate) |

- `INACTIVE` is a terminal state — no events are accepted, no further transitions.
- A meter in `DRAFT` with no events can be fully edited (including aggregation change).

---

## Error Codes

| Code | HTTP | Trigger |
|------|------|---------|
| `METER_NOT_FOUND` | 404 | `meterId` does not exist for this org |
| `METER_AGGREGATION_IMMUTABLE` | 422 | Attempt to change `event_type` or `aggregation` after events exist |
| `METER_INACTIVE` | 409 | Event submitted to a meter with `status = INACTIVE` |
| `METER_LINKED_TO_PRICING` | 409 | Attempt to delete/deactivate a meter linked to active pricing_models or charges |
| `DUPLICATE_EVENT` | 200 | Idempotency key already seen within 24h (deduplicated, not an error) |
| `MAX_EVENTS_PER_METER_PER_DAY_EXCEEDED` | 429 | Daily event quota exceeded |
| `INVALID_AGGREGATION` | 422 | `aggregation` is not one of: `SUM`, `COUNT`, `AVG`, `GAUGE` |
| `FORBIDDEN` | 403 | Actor lacks `ORG_ADMIN` or `SUPER_ADMIN` role for this operation |
| `ORG_NOT_FOUND` | 404 | orgId does not match any org |
| `INVALID_IDEMPOTENCY_KEY` | 422 | `idempotency_key` is missing or exceeds 128 characters |

---

## Environment Config Keys

| Key | Description |
|-----|-------------|
| `USAGE_EVENT_RETENTION_DAYS` | Number of days to retain raw usage events before archival (default: 365) |
| `MAX_EVENTS_PER_METER_PER_DAY` | Per-meter daily event ingest limit (default: 1,000,000) |
| `METER_API_KEY_PREFIX` | Prefix for meter API key hashes (e.g., `qb_mk_`) |
| `IDEMPOTENCY_KEY_TTL_HOURS` | TTL for idempotency key deduplication window (default: 24) |
| `DATABASE_URL` | PostgreSQL connection string (Prisma) |
| `KEYCLOAK_URL` | Keycloak server base URL |
| `KEYCLOAK_REALM` | quantumbilling |
| `KEYCLOAK_CLIENT_ID` / `KEYCLOAK_CLIENT_SECRET` | Backend confidential client credentials |

---

## UI Story

### Meters list page
Accessible from **Settings › Meters**. Displays a card per meter showing: name, event_type, aggregation badge, status badge, last event timestamp. Actions: "Edit", "Deactivate". ORG_ADMIN can click "New meter" to open the create modal.

### Create / Edit meter modal
Fields:
- **Name** (text input, required)
- **Event type** (text input) — e.g., `api.call`, `storage.gb`, `seat.occupied`
- **Aggregation** (select: COUNT | SUM | AVG | GAUGE) — locked to edit after events recorded
- **Field** (text input, optional) — JSON path within the event payload to extract the value

CTA: "Create meter" / "Save changes". On success: toast "Meter created", modal closes, list refreshes.

### Meter detail page
- Header: meter name, status badge, event_type, aggregation badge, created date
- **Usage chart**: line chart showing aggregated value per day over the last 30 days
- **Recent events table**: last 20 events with columns: timestamp, value, idempotency key
- **API integration panel**: displays the meter-specific API key (masked, with "Reveal" toggle) and curl example

### Deactivate meter — confirmation dialog
Warning message: "Deactivating this meter means QuantumBilling will no longer accept new events. Existing data will be retained for billing. This action cannot be undone." On 409 `METER_LINKED_TO_PRICING`: dialog shows inline error.

---

## Dependencies & Notes for Agent

- **Schema alignment:** The ERD shows `catalog.meters` with columns `id, org_id, name, event_type, aggregation, field`. Note that `event_type` (not `meter_type`) and `aggregation` (not `unit`) are the correct column names.
- **Slug/naming:** Unlike the reference HTML which used `meter_slug`, this story uses `meterId` (UUID) as the primary identifier. A `meter_slug` may be added as a convenience field but is not in the ERD.
- **Idempotency deduplication:** On `POST /events`, check `usage_events.idempotency_key` for a matching key within the last 24h window. If found, return 200 without inserting.
- **State transition guard:** Only transition DRAFT → ACTIVE if current status is DRAFT. Use a DB transaction to atomically insert the event and update meter status.
- **Meter API key auth:** Each org can have `developer.api_keys` entries scoped to specific meters. The `MeterApiKeyGuard` validates the `X-Meter-Api-Key` header.
- **Prisma model:** `Meter` with enum `MeterStatus { DRAFT ACTIVE INACTIVE }` and enum `MeterAggregation { SUM COUNT AVG GAUGE }`.
- **Price plan linkage check:** Before deactivating, query `catalog.pricing_models` and `catalog.charges` for any record linking this meter with `status = ACTIVE`. If found, return 409 `METER_LINKED_TO_PRICING`.
- **SUPER_ADMIN scope:** Controller methods accept `:orgId` to scope meters to the correct organization.
- **Audit logging:** All create/update/deactivate operations on meters must be written to `audit_logs`.
