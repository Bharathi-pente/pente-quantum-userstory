# QuantumBilling User Story: Rate Cards — QB-STORY-013

---

## Story ID & Metadata

**QB-STORY-013** · Sprint 2 · Phase: Feature

---

## Title

Rate Cards — define and manage pricing rate cards that map meters to prices

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | Priority: P0 |

---

## Description

As an **ORG_ADMIN**, I want to create and manage rate cards — named collections of meter-to-price mappings — so that contracts and subscriptions can reference a single rate card instead of hardcoding individual meter prices, and I can version-control pricing changes over time.

### Key capabilities

- **Rate card**: a named, versioned pricing template scoped to an org
- **rate_card_rates**: individual rows linking `meter_id` → `rate` (price per unit) + `model_name` (FLAT | PER_UNIT | TIERED) + `unit_label`
- Multiple meters can be in one rate card (e.g., a "AI Platform Base" rate card covering: GPT-4 calls, embedding calls, storage GB, compute hours)
- **effective_date**: the date from which this rate card's rates apply to new usage
- Rate cards can be **ACTIVE** (current) or **ARCHIVED** (superseded)
- **rate_card_versions**: every time a rate card is updated, a new version is created with `snapshot_data` (full copy of rates at that point) and `change_summary`
- **Contracts** (`customer.contracts`) link to a `rate_card_id` — the contract inherits all rates from the rate card
- **billing.contract_rates**: contract-specific rates that override the rate card's rates for a specific contract; these take precedence over `rate_card_rates`
- **Rate card versioning**: updating a rate card creates a new `billing.rate_card_versions` entry; existing contracts continue using the rate card's rates as of their contract date (historical billing preserved)
- **SUPER_ADMIN** can manage rate cards for any org

---

## RBAC Roles

| Role | Can create | Can update | Can archive | Can assign to contract | Scope |
|------|------------|------------|-------------|------------------------|-------|
| `SUPER_ADMIN` | Yes (any org) | Yes (any org) | Yes (any org) | Yes (any org) | Platform-wide |
| `ORG_ADMIN` | Yes (own org) | Yes (own org) | Yes (own org) | Yes (own org) | Own org only |
| `CUSTOMER` | No | No | No | No | Read-only (view own contract's rate card) |
| `END_USER` | No | No | No | No | No access |

---

## Acceptance Criteria

1. ORG_ADMIN can create a rate card with `name`, `effective_date`, and initial status `DRAFT`. The rate card is scoped to their `org_id`.
2. ORG_ADMIN can add one or more rate lines to a rate card via `POST /api/v1/rate-cards/:rateCardId/rates`, each linking a `meter_id` to a `rate`, `model_name` (FLAT | PER_UNIT | TIERED), and `unit_label`.
3. ORG_ADMIN can transition a rate card from `DRAFT` → `ACTIVE`. Once `ACTIVE`, the `effective_date` governs when its rates apply to new usage.
4. ORG_ADMIN can update an `ACTIVE` rate card (add/update/remove rates). Each update creates a new entry in `billing.rate_card_versions` with `snapshot_data` (JSONB of full rate card state) and a `change_summary`. The rate card itself stays `ACTIVE`.
5. ORG_ADMIN can archive an `ACTIVE` rate card (→ `ARCHIVED`). A rate card in `ARCHIVED` status cannot be assigned to new contracts but remains valid for existing contracts that reference it.
6. `billing.contract_rates` entries for a contract take precedence over `rate_card_rates` for the same `meter_id` and date range. The billing engine must check `contract_rates` first.
7. SUPER_ADMIN can perform all CRUD operations on rate cards for any org (identified by `org_id` in path or body).
8. CUSTOMER can read the rate card linked to their contract (via `GET /api/v1/rate-cards/:rateCardId`) but cannot modify it.
9. `GET /api/v1/rate-cards/:rateCardId/preview` accepts a usage map (`{meter_id: quantity}`) and returns a calculated cost breakdown based on the rate card's current active rates.
10. All rate card create/update/archive operations are written to `audit_logs` with actor, target `rate_card_id`, `org_id`, and a JSON `metadata` blob.

---

## Test Cases

### TC-01 — Happy path: create rate card and add rates

**Given:** authenticated `ORG_ADMIN` for org `acme` (`org_id` = UUID)  
**When:** `POST /api/v1/rate-cards` `{ "name": "AI Platform Base", "effective_date": "2026-07-01", "status": "DRAFT", "org_id": "<acme_uuid>" }`  
**Then:** `201` returned, `catalog.rate_cards` row created with `status = DRAFT`  
**When:** `POST /api/v1/rate-cards/<rateCardId>/rates` with body:
```json
[
  { "meter_id": "<gpt4_meter_uuid>", "model_name": "PER_UNIT", "rate": 0.03, "unit_label": "per 1K tokens" },
  { "meter_id": "<storage_meter_uuid>", "model_name": "PER_UNIT", "rate": 0.10, "unit_label": "per GB-month" }
]
```
**Then:** `201` returned, two rows inserted into `catalog.rate_card_rates`  
**✓** Rate card shows 2 rates on `GET /api/v1/rate-cards/<rateCardId>`

---

### TC-02 — Activate a DRAFT rate card

**Given:** DRAFT rate card exists with at least one rate  
**When:** `PATCH /api/v1/rate-cards/<rateCardId>` `{ "status": "ACTIVE" }`  
**Then:** `200` returned, `catalog.rate_cards.status = ACTIVE`  
**✓** Subsequent `GET /api/v1/rate-cards` lists it as `ACTIVE`

---

### TC-03 — Update an ACTIVE rate card (versioning)

**Given:** ACTIVE rate card with 2 rates  
**When:** `PATCH /api/v1/rate-cards/<rateCardId>/rates/<rateId>` `{ "rate": 0.035 }`  
**Then:** `200` returned, rate updated in `catalog.rate_card_rates`  
**And:** a new row inserted into `billing.rate_card_versions` with `version = N+1`, `change_type = "UPDATE"`, `snapshot_data` containing full copy of all rates, and `change_summary = "Updated rate for meter <id>"`  
**✓** `GET /api/v1/rate-cards/<rateCardId>/versions` returns at least 2 versions

---

### TC-04 — Assign rate card to a contract

**Given:** ACTIVE rate card `RC-001` and a `DRAFT` contract `CT-001` for customer `ACME Corp`  
**When:** `POST /api/v1/rate-cards/<rateCardId>/assign` `{ "contract_id": "<ct001_uuid>" }`  
**Then:** `200` returned, `customer.contracts.rate_card_id` set to `RC-001`  
**✓** Contract now inherits all rates from `RC-001`; `billing.contract_rates` rows are still consulted for overrides

---

### TC-05 — Archive an ACTIVE rate card

**Given:** ACTIVE rate card with 2 active contracts referencing it  
**When:** `PATCH /api/v1/rate-cards/<rateCardId>` `{ "status": "ARCHIVED" }`  
**Then:** `200` returned, `catalog.rate_cards.status = ARCHIVED`  
**✓** Existing contracts remain valid (historical billing preserved)  
**✗** `POST /api/v1/rate-cards/<rateCardId>/assign` to a new contract returns `409 RATE_CARD_ARCHIVED`

---

### TC-06 — RBAC escalation attempt — END_USER tries to create

**Given:** actor role is `END_USER`  
**When:** `POST /api/v1/rate-cards` with body  
**Then:** `403 FORBIDDEN` — guard rejects before service layer  
**✗** No record created in `catalog.rate_cards`

---

### TC-07 — CUSTOMER reads their contract's rate card

**Given:** customer `bob@acme.com` has a contract linking to rate card `RC-001`  
**When:** `GET /api/v1/rate-cards/<rateCardId>` as `CUSTOMER` role  
**Then:** `200` returned with full rate card + rates  
**✗** `PATCH /api/v1/rate-cards/<rateCardId>` returns `403 FORBIDDEN`

---

### TC-08 — Preview usage cost

**Given:** ACTIVE rate card `RC-001` with:
- meter `gpt4` → `rate: 0.03`, `model_name: PER_UNIT`, `unit_label: "per 1K tokens"`
- meter `storage` → `rate: 0.10`, `model_name: PER_UNIT`, `unit_label: "per GB-month"`
**When:** `POST /api/v1/rate-cards/<rateCardId>/preview` body:
```json
{ "usage": { "<gpt4_uuid>": 50000, "<storage_uuid>": 100 } }
```
**Then:** `200` returned:
```json
{
  "total": 1600.00,
  "breakdown": [
    { "meter_id": "<gpt4_uuid>", "meter_name": "GPT-4", "quantity": 50000, "rate": 0.03, "unit_label": "per 1K tokens", "cost": 1500.00 },
    { "meter_id": "<storage_uuid>", "meter_name": "Storage", "quantity": 100, "rate": 0.10, "unit_label": "per GB-month", "cost": 10.00 }
  ]
}
```

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `POST` | `/api/v1/rate-cards` | Create a new rate card (DRAFT) | JWT · Guard: `OrgAdminGuard` · Body: `{org_id, name, effective_date}` |
| `GET` | `/api/v1/rate-cards` | List rate cards for org (filterable by `?status=ACTIVE\|DRAFT\|ARCHIVED`) | JWT · Guard: `OrgMemberGuard` · Query: `?org_id=&status=&page=1&limit=20` |
| `GET` | `/api/v1/rate-cards/:rateCardId` | Get rate card with all rates | JWT · Guard: `OrgMemberGuard` or `CustomerGuard` (own contract only) |
| `PATCH` | `/api/v1/rate-cards/:rateCardId` | Update rate card metadata or status (creates new version) | JWT · Guard: `OrgAdminGuard` |
| `POST` | `/api/v1/rate-cards/:rateCardId/rates` | Add one or more rates to the card | JWT · Guard: `OrgAdminGuard` · Body: `[{meter_id, model_name, rate, unit_label}]` |
| `PATCH` | `/api/v1/rate-cards/:rateCardId/rates/:rateId` | Update a specific rate | JWT · Guard: `OrgAdminGuard` · Body: `{rate, model_name, unit_label}` |
| `DELETE` | `/api/v1/rate-cards/:rateCardId/rates/:rateId` | Remove a rate from the card | JWT · Guard: `OrgAdminGuard` |
| `GET` | `/api/v1/rate-cards/:rateCardId/versions` | List all versions of this rate card | JWT · Guard: `OrgAdminGuard` |
| `GET` | `/api/v1/rate-cards/:rateCardId/versions/:versionId` | Get a specific version snapshot | JWT · Guard: `OrgAdminGuard` |
| `POST` | `/api/v1/rate-cards/:rateCardId/preview` | Preview cost for a given usage map | JWT · Guard: `OrgMemberGuard` · Body: `{usage: {meter_id: quantity}}` |
| `POST` | `/api/v1/rate-cards/:rateCardId/assign` | Assign this rate card to a contract | JWT · Guard: `OrgAdminGuard` · Body: `{contract_id}` |

---

## Data Tables Used

| Table | Operation | Key Columns |
|-------|-----------|-------------|
| `catalog.rate_cards` | INSERT · SELECT · UPDATE | `id, org_id, name, effective_date, status` |
| `catalog.rate_card_rates` | INSERT · SELECT · UPDATE · DELETE | `id, rate_card_id, meter_id, model_name, rate, unit_label` |
| `billing.rate_card_versions` | INSERT · SELECT | `id, rate_card_id, org_id, version, change_type, snapshot_data (jsonb), change_summary, created_at` |
| `catalog.meters` | SELECT | `id, org_id, name, event_type, aggregation` |
| `identity.organizations` | SELECT | `id, name` |
| `customer.contracts` | SELECT · UPDATE | `id, customer_id, rate_card_id, name, status` |
| `billing.contract_rates` | INSERT · SELECT · DELETE | `id, contract_id, meter_id, model_name, effective_date, expires_date, rate, unit_label` |
| `audit_logs` | INSERT | `id, actor_id, action, target_id (rate_card_id), org_id, metadata (jsonb), created_at` |

---

## State Machine

### Rate Card Lifecycle

```
DRAFT → ACTIVE → ARCHIVED
```

| State | Description | Allowed Transitions |
|-------|-------------|---------------------|
| `DRAFT` | Rate card is being built; rates can be added/removed | → `ACTIVE` (via `PATCH` with `status: ACTIVE`) |
| `ACTIVE` | Rate card is live; can be assigned to contracts and receive usage | → `ARCHIVED` (via `PATCH` with `status: ARCHIVED`); updates create new version rows |
| `ARCHIVED` | Rate card is superseded; read-only, not assignable to new contracts | Terminal — no outgoing transitions |

### Versioning Rule

- Each `PATCH` to rates (add/update/delete) on an `ACTIVE` card creates a new `billing.rate_card_versions` row with incremented `version` and full `snapshot_data`
- Historical contracts continue using rates as of their `effective_date` from `rate_card_rates` — not the latest version
- `ARCHIVED` is terminal; archived cards cannot be re-activated

---

## Error Codes

| Code | HTTP | Trigger |
|------|------|---------|
| `RATE_CARD_NOT_FOUND` | 404 | `rateCardId` does not exist in `catalog.rate_cards` |
| `RATE_NOT_FOUND` | 404 | `rateId` does not exist in `catalog.rate_card_rates` for this rate card |
| `VERSION_NOT_FOUND` | 404 | `versionId` does not exist in `billing.rate_card_versions` |
| `CONTRACT_NOT_FOUND` | 404 | `contract_id` does not exist in `customer.contracts` |
| `METER_NOT_FOUND` | 404 | `meter_id` does not exist in `catalog.meters` |
| `RATE_CARD_DUPLICATE_NAME` | 409 | A rate card with the same `name` + `org_id` already exists |
| `RATE_CARD_ARCHIVED` | 409 | Attempted to assign an `ARCHIVED` rate card to a new contract |
| `INVALID_STATUS_TRANSITION` | 422 | e.g., `ARCHIVED` → `DRAFT`, or `DRAFT` → `ARCHIVED` directly |
| `RATE_CARD_INACTIVE` | 422 | Attempted to assign a `DRAFT` rate card to a contract |
| `CONTRACT_ALREADY_HAS_RATE_CARD` | 409 | Contract already has a `rate_card_id` set |
| `ORphaned_RATE` | 409 | Attempted to delete a rate that is referenced by an active `billing.contract_rates` override |
| `INSUFFICIENT_ROLE` | 403 | Actor role cannot perform this operation on this org's resource |
| `FORBIDDEN` | 403 | `END_USER` or `CUSTOMER` (for write operations) |
| `INVALID_DATE` | 422 | `effective_date` is in the past for a new rate card |
| `INVALID_MODEL_NAME` | 422 | `model_name` is not one of `FLAT | PER_UNIT | TIERED` |

---

## Environment Config Keys

| Key | Description |
|-----|-------------|
| `RATE_CARD_MAX_RATES_PER_CARD` | Maximum number of rates allowed per rate card (default: 100) |
| `RATE_CARD_NAME_MAX_LENGTH` | Maximum character length for rate card name (default: 128) |
| `AUDIT_LOG_ENABLED` | Boolean — write all rate card mutations to `audit_logs` (default: true) |
| `DATABASE_URL` | PostgreSQL connection string (Prisma) |
| `KEYCLOAK_URL` | Keycloak server base URL |
| `KEYCLOAK_REALM` | `quantumbilling` |
| `KEYCLOAK_CLIENT_ID / KEYCLOAK_CLIENT_SECRET` | Backend confidential client credentials |
| `BILLING_ENGINE_URL` | Internal URL of the billing engine service for preview calculations |
| `RATE_CARD_VERSION_RETENTION_DAYS` | Days to retain old version snapshots before archival (default: 365) |

---

## UI Story

### Rate Cards List Page

Accessible from **Billing › Rate Cards**. Shows all rate cards for the current org in a table: Name, Status badge (DRAFT / ACTIVE / ARCHIVED), Effective Date, # of Rates, Last Updated. Filter tabs: All / Active / Draft / Archived. "Create Rate Card" button (top right) — visible only to ORG_ADMIN and SUPER_ADMIN.

### Create / Edit Rate Card Modal

**Create flow** — "New Rate Card" modal with fields:
- Name (text input, required, max 128 chars)
- Effective Date (date picker, required, cannot be in the past)
- CTA: "Create" → `POST /api/v1/rate-cards`, then immediately opens the Edit Rate Card view

**Edit flow** — Rate card detail page `/rate-cards/:rateCardId`:
- Header: name, status badge, effective date, "Activate" / "Archive" action button (based on current status)
- Rates table: columns = Meter Name, Model, Rate, Unit Label, Actions (edit / delete)
- "Add Rate" button → inline form row: Meter (searchable dropdown from `catalog.meters`), Model (FLAT / PER_UNIT / TIERED select), Rate (decimal), Unit Label (text)
- On save: `POST /api/v1/rate-cards/:rateCardId/rates` → rates table refreshes
- Each rate edit/delete triggers `PATCH/DELETE /api/v1/rate-cards/:rateCardId/rates/:rateId`

### Version History Panel

"Version History" tab on the rate card detail page. Lists all `billing.rate_card_versions` entries: version number, change type (CREATED / UPDATE / ARCHIVED), change summary, created at, created by. Clicking a version opens a read-only snapshot showing the full rates array at that point in time.

### Assign to Contract

Accessible from **Contracts › [Contract Name] › Rate Card** section. Shows current linked rate card (if any). "Change Rate Card" opens a searchable modal listing all ACTIVE rate cards for the org. On select: `POST /api/v1/rate-cards/:rateCardId/assign`. Success toast: "Rate card 'AI Platform Base' linked to contract 'ACME Q2'."

### Cost Preview

Accessible from the rate card detail page — "Preview Cost" panel. User enters quantities per meter (meter name shown, unit label shown). "Calculate" button → `POST /api/v1/rate-cards/:rateCardId/preview`. Results shown as a cost breakdown table: Meter, Quantity, Rate, Unit, Total Cost. Total shown prominently.

---

## Dependencies & Notes for Agent

- **Prisma models required:**
  - `RateCard` — map to `catalog.rate_cards`; enum `RateCardStatus { DRAFT ACTIVE ARCHIVED }`
  - `RateCardRate` — map to `catalog.rate_card_rates`; enum `RateModelName { FLAT PER_UNIT TIERED }`
  - `RateCardVersion` — map to `billing.rate_card_versions`; `snapshot_data` is `Json` type
  - `Contract` — map to `customer.contracts`; add `rateCardId` optional relation
  - `ContractRate` — map to `billing.contract_rates`; takes precedence over `RateCardRate` in billing engine

- **Versioning logic:** Every mutating operation on `catalog.rate_card_rates` (INSERT / UPDATE / DELETE) for an `ACTIVE` rate card must be wrapped in a transaction that also inserts a new `billing.rate_card_versions` row. Use `snapshot_data = Prisma.JsonValue` with a serialized copy of all current `RateCardRate` rows.

- **Billing engine precedence:** When calculating contract cost, the billing engine must check `billing.contract_rates` first (by `contract_id` + `meter_id` + date range), falling back to `catalog.rate_card_rates` via the contract's linked `rate_card_id`. Use `effective_date` and `expires_date` on `contract_rates` for range filtering.

- **State machine guard:** `RateCardStateMachine` service class with `transition(rateCardId, targetStatus)` method. Valid transitions: DRAFT→ACTIVE, ACTIVE→ARCHIVED. Throw `INVALID_STATUS_TRANSITION` for invalid transitions.

- **RBAC:** `OrgAdminGuard` checks `actor.org_id === resource.org_id` OR `actor.role === SUPER_ADMIN`. `CustomerGuard` only allows GET on the rate card linked to their own contract.

- **Audit logging:** All write operations on rate cards and rates must emit an `audit_log` entry. Action names: `RATE_CARD_CREATED`, `RATE_CARD_UPDATED`, `RATE_CARD_ACTIVATED`, `RATE_CARD_ARCHIVED`, `RATE_ADDED`, `RATE_UPDATED`, `RATE_DELETED`, `RATE_CARD_ASSIGNED`. Metadata blob includes `{rateCardId, orgId, changes: {...}}`.

- **Preview endpoint:** `POST /api/v1/rate-cards/:rateCardId/preview` — no contract involved, just reads `catalog.rate_card_rates` directly and applies `model_name` logic:
  - `FLAT`: cost = `rate` (flat fee, ignore quantity)
  - `PER_UNIT`: cost = `rate` × `quantity`
  - `TIERED` (future): implement tiered pricing logic as defined in the billing spec

- **SUPER_ADMIN cross-org:** When `actor.role === SUPER_ADMIN`, `org_id` from request body or path takes precedence. No ownership check against `actor.org_id`.

---

*Generated from QB-STORY-013 · QuantumBilling · Rate Cards user story*
