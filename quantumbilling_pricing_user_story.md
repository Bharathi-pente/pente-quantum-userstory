# QuantumBilling User Story: Pricing — Configure Price Plans and Pricing Models for Meters

---

## Story ID & Metadata

**QB-STORY-004** · Sprint 2 · Phase: Feature

---

## Title

**Pricing** — configure price plans and pricing models for meters

---

## Badges

<span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;letter-spacing:.3px;background:#EEEDFE;color:#3C3489">Backend</span>
<span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;letter-spacing:.3px;background:#E1F5EE;color:#085041">UI</span>
<span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;letter-spacing:.3px;background:#FAEEDA;color:#633806">Auth / RBAC</span>
<span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;letter-spacing:.3px;background:#E6F1FB;color:#0C447C">Billing Engine</span>
<span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;letter-spacing:.3px;background:#F1EFE8;color:#444441">Priority: P0</span>

---

## Description

Based on `catalog.pricing_models`, `catalog.pricing_tiers`, `catalog.plans`, and `catalog.charges`. These are the billing configuration entities.

> **As an ORG_ADMIN**, I want to define pricing models that specify how much to charge for each meter's usage (flat fee + per-unit rate, volume tiers, or graduated tiers), so that QuantumBilling can calculate invoices based on actual consumption.

Key capabilities:
- Pricing models are tied to an org via `org_id` and optionally linked to a specific meter (`catalog.meters`)
- Pricing model types (from `catalog.pricing_models.pricing_type`): `FLAT`, `PER_UNIT`, `TIERED`
- Each `catalog.plans` record has a `billing_period` (MONTHLY | QUARTERLY | YEARLY), `base_amount`, `pay_in_advance`
- `catalog.charges` links a plan to a meter with a specific `charge_model` and `billing_model`
- ORG_ADMIN can create, update, and deactivate pricing models and plans for their org
- SUPER_ADMIN can manage any org's pricing
- State machine: `DRAFT` → `ACTIVE` → `ARCHIVED` (terminal)

---

## RBAC Roles

| Role | Can manage pricing | Scope |
|------|-------------------|-------|
| <span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;background:#FCEBEB;color:#791F1F">SUPER_ADMIN</span> | Yes — any org | Platform-wide |
| <span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;background:#EEEDFE;color:#3C3489">ORG_ADMIN</span> | Yes — own org only | Own org |
| <span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;background:#E1F5EE;color:#085041">CUSTOMER</span> | Read-only — own plan | Own account |
| <span style="display:inline-block;font-size:11px;font-weight:500;padding:2px 8px;border-radius:4px;background:#F1EFE8;color:#444441">END_USER</span> | No access | — |

---

## Acceptance Criteria

1. ORG_ADMIN can create a pricing model with a name, `pricing_type` (`FLAT` | `PER_UNIT` | `TIERED`), optionally link to a meter, and an `effective_from` date.
2. For `FLAT` models: a `base_amount` is set on the linked `catalog.plans` record; charge is the same every billing period.
3. For `PER_UNIT` models: `catalog.charges` records define the per-unit rate; total charge = `unit_price` × quantity consumed from the meter.
4. For `TIERED` models: multiple `catalog.pricing_tiers` rows (`from_qty`, `to_qty`, `price_per_unit`) define volume bands; system applies the correct tier rate based on cumulative consumption.
5. System validates that `TIERED` tier ranges are contiguous (no gaps) and non-overlapping; returns `422 TIER_GAP` or `422 TIER_OVERLAP` if validation fails.
6. When a pricing model is published (status transitions `DRAFT` → `ACTIVE`), the model becomes immutable for existing subscriptions.
7. Updating an `ACTIVE` pricing model does NOT affect existing active subscriptions until their next billing cycle renewal.
8. Archiving a model (`ACTIVE` → `ARCHIVED`) is a soft delete; existing subscriptions are honoured until end of their billing period. Archived models cannot be assigned to new subscriptions.
9. SUPER_ADMIN can perform all CRUD operations on any org's pricing models and plans.
10. All pricing model create/update/archive events are written to `audit_logs` with actor, org_id, model_id, and changed fields.

---

## Test Cases

### TC-01 — Happy path: create and activate a FLAT pricing model

**Given:** authenticated ORG_ADMIN for org `acme`
**When:** POST `/api/v1/pricing-models` `{name: "Basic Monthly", pricing_type: "FLAT", meter_id: null}`
**Then:** `201` returned, pricing model created with status `DRAFT`

**When:** PATCH `/api/v1/pricing-models/:modelId` `{status: "ACTIVE"}`
**Then:** model status transitions to `ACTIVE`; model is now immutable

---

### TC-02 — Create a TIERED pricing model with multiple tiers

**Given:** authenticated ORG_ADMIN
**When:** POST `/api/v1/pricing-models` with pricing_type `TIERED` and tiers:
```
[
  { from_qty: 0,    to_qty: 1000, price_per_unit: 0.10 },
  { from_qty: 1001, to_qty: 3000, price_per_unit: 0.08 },
  { from_qty: 3001, to_qty: null, price_per_unit: 0.05 }
]
```
**Then:** `201` returned; model created with all three tiers stored in `catalog.pricing_tiers`

---

### TC-03 — TIERED tier overlap validation

**Given:** tier rows where `from_qty: 500` overlaps with a previous tier ending at `1000`
**When:** POST `/api/v1/pricing-models`
**Then:** `422 TIER_OVERLAP` — no model created

---

### TC-04 — TIERED tier gap validation

**Given:** tier rows with a gap between `to_qty: 1000` and next `from_qty: 1005`
**When:** POST `/api/v1/pricing-models`
**Then:** `422 TIER_GAP` — no model created; error indicates the gap at qty `1001–1004`

---

### TC-05 — Price change on ACTIVE model takes effect at renewal

**Given:** an ACTIVE pricing model assigned to 3 subscriptions
**When:** PATCH `/api/v1/pricing-models/:modelId` with updated tier prices
**Then:** `200` returned; pending update stored; existing subscriptions continue billing at old rate until renewal

---

### TC-06 — Archive an active pricing model

**Given:** an ACTIVE model with 2 active subscriptions
**When:** DELETE `/api/v1/pricing-models/:modelId` (soft archive)
**Then:** `200` returned; model status set to `ARCHIVED`; subscriptions honoured until period end; model no longer assignable

---

### TC-07 — SUPER_ADMIN manages another org's pricing

**Given:** authenticated SUPER_ADMIN
**When:** GET `/api/v1/orgs/:orgId/pricing-models`
**Then:** `200` returned with list of all pricing models for that org

---

### TC-08 — Preview calculation for given usage

**Given:** an ACTIVE TIERED model with tiers:
```
[
  { from_qty: 0,    to_qty: 1000, price_per_unit: 0.10 },
  { from_qty: 1001, to_qty: null, price_per_unit: 0.08 }
]
```
**When:** GET `/api/v1/pricing-models/:modelId/preview?quantity=2500`
**Then:** `200` returned; preview shows:
- Tier 1 (0–1000): 1000 units × $0.10 = $100.00
- Tier 2 (1001–2500): 1500 units × $0.08 = $120.00
- **Total: $220.00**

---

## API Endpoints

### POST `/api/v1/pricing-models` — Create pricing model

- **Method:** `POST`
- **Path:** `/api/v1/pricing-models`
- **Description:** Create a new pricing model in `DRAFT` status. Optionally link to a specific meter.
- **Auth:** JWT · Guard: `OrgAdminGuard`
- **Body:**
  ```json
  {
    "name": "string",
    "pricing_type": "FLAT | PER_UNIT | TIERED",
    "meter_id": "uuid | null",
    "effective_from": "ISO8601",
    "org_id": "uuid"
  }
  ```
- **Response:** `201 Created` with created pricing model object

---

### GET `/api/v1/pricing-models` — List pricing models for org

- **Method:** `GET`
- **Path:** `/api/v1/pricing-models`
- **Description:** List all pricing models for the authenticated user's org. Supports filtering by `status`, `pricing_type`, `meter_id`.
- **Auth:** JWT · Guard: `OrgMemberGuard`
- **Query:** `?status=DRAFT|ACTIVE|ARCHIVED&pricing_type=TIERED&page=1&limit=20`
- **Response:** `200 OK` with paginated pricing model list

---

### GET `/api/v1/pricing-models/:modelId` — Get model details with pricing tiers

- **Method:** `GET`
- **Path:** `/api/v1/pricing-models/:modelId`
- **Description:** Returns full pricing model details including all `catalog.pricing_tiers` rows.
- **Auth:** JWT · Guard: `OrgMemberGuard` (or `SUPER_ADMIN` cross-org)
- **Response:** `200 OK` with pricing model object + nested tiers array

---

### PATCH `/api/v1/pricing-models/:modelId` — Update pricing model

- **Method:** `PATCH`
- **Path:** `/api/v1/pricing-models/:modelId`
- **Description:** Update model fields. For `ACTIVE` models, changes are staged for next billing cycle.
- **Auth:** JWT · Guard: `OrgAdminGuard` | `SuperAdminGuard`
- **Body (partial):** `{ name?, status? }`
- **Response:** `200 OK` with updated model
- **Note:** Publishing a `DRAFT` model: set `status: "ACTIVE"`. This transition is one-way and final.

---

### DELETE `/api/v1/pricing-models/:modelId` — Archive pricing model (soft delete)

- **Method:** `DELETE`
- **Path:** `/api/v1/pricing-models/:modelId`
- **Description:** Soft-delete: sets `status = ARCHIVED`. Existing subscriptions honoured. Archived models cannot accept new assignments.
- **Auth:** JWT · Guard: `OrgAdminGuard` | `SuperAdminGuard`
- **Response:** `200 OK`

---

### POST `/api/v1/plans` — Create a plan linked to a pricing model

- **Method:** `POST`
- **Path:** `/api/v1/plans`
- **Description:** Create a `catalog.plans` record linked to a `catalog.pricing_model` and optionally a `catalog.product`.
- **Auth:** JWT · Guard: `OrgAdminGuard`
- **Body:** `{name, slug, billing_period, base_amount, currency, pay_in_advance, pricing_model_id?, product_id?}`
- **Response:** `201 Created`

---

### GET `/api/v1/plans/:planId/preview` — Preview calculation

- **Method:** `GET`
- **Path:** `/api/v1/plans/:planId/preview`
- **Description:** Given a usage quantity, return the expected charge based on the linked pricing model's tiers.
- **Auth:** JWT · Guard: `OrgMemberGuard`
- **Query:** `?quantity=number`
- **Response:** `200 OK` with tier breakdown and total

---

## Data Tables Used

Based on `catalog.pricing_models`, `catalog.pricing_tiers`, `catalog.plans`, `catalog.charges`.

| Table | Operation | Key Columns |
|-------|-----------|-------------|
| `catalog.pricing_models` | INSERT · SELECT · UPDATE | `id, org_id, meter_id, name, pricing_type, effective_from, status` |
| `catalog.pricing_tiers` | INSERT · SELECT · UPDATE | `id, pricing_model_id, from_qty, to_qty, price_per_unit, sort_order` |
| `catalog.plans` | INSERT · SELECT · UPDATE | `id, product_id, name, slug, billing_period, trial_days, base_amount, currency, pay_in_advance, is_active` |
| `catalog.charges` | INSERT · SELECT | `id, plan_id, meter_id, name, charge_type, charge_model, billing_model, pay_in_advance, is_active` |
| `catalog.rate_cards` | SELECT | `id, org_id, name, effective_date, status` |
| `catalog.rate_card_rates` | SELECT | `id, rate_card_id, meter_id, model_name, rate, unit_label` |
| `identity.organizations` | SELECT | `id, currency` |
| `audit_logs` | INSERT | `id, actor_id, action, target_model_id, org_id, metadata, created_at` |

---

## State Machine — Pricing Model Lifecycle

```
DRAFT ──────── publish ────────► ACTIVE
                                     │
                                     │ archive
                                     ▼
                               ARCHIVED (terminal)
```

| State | Description |
|-------|-------------|
| `DRAFT` | Model is being configured; not yet visible to subscriptions. Can be edited freely. |
| `ACTIVE` | Model is published and assignable to subscriptions. Price changes are staged for next billing cycle. |
| `ARCHIVED` | Model is deactivated (soft delete). Existing subscriptions honoured until billing period end. Cannot be assigned to new subscriptions. Terminal. |

**Transitions:**
- `DRAFT` → `ACTIVE` — triggered by `PATCH /pricing-models/:id` with `{ status: "ACTIVE" }`. One-way, cannot be reverted.
- `ACTIVE` → `ARCHIVED` — triggered by `DELETE /pricing-models/:id`. Terminal; cannot be undone.

---

## Error Codes

| Code | HTTP | Trigger |
|------|------|---------|
| `PRICING_MODEL_NOT_FOUND` | 404 | `modelId` does not exist or belongs to another org |
| `PLAN_NOT_ASSIGNABLE` | 409 | Attempting to assign an `ARCHIVED` model to a new subscription |
| `TIER_GAP` | 422 | TIERED model has non-contiguous tier ranges |
| `TIER_OVERLAP` | 422 | TIERED model has overlapping tier ranges |
| `TIER_MISSING` | 422 | TIERED model submitted with zero tiers |
| `INVALID_PRICING_TYPE` | 422 | `pricing_type` is not one of `FLAT`, `PER_UNIT`, `TIERED` |
| `SUPER_ADMIN_REQUIRED` | 403 | Actor is ORG_ADMIN attempting cross-org plan management |
| `FORBIDDEN` | 403 | Actor role is `CUSTOMER`, `END_USER`, or unauthenticated |
| `METER_NOT_FOUND` | 404 | `meter_id` references a meter that does not exist or belongs to another org |
| `DUPLICATE_DEFAULT_PLAN` | 409 | Another plan is already set as default for the same meter/org combination |

---

## Environment Config Keys

| Key | Description |
|-----|-------------|
| `DEFAULT_CURRENCY` | Default currency for new orgs (e.g., `USD`, `EUR`, `GBP`) |
| `TAX_RATE_DEFAULT` | Default tax rate applied to invoices if no org-specific rate is set |
| `BILLING_CYCLE_START_DAY` | Day of month when billing cycles begin (1–28; default: `1`) |
| `DATABASE_URL` | PostgreSQL connection string (Prisma) |
| `KEYCLOAK_URL` | Keycloak server base URL |
| `KEYCLOAK_REALM` | `quantumbilling` |
| `KEYCLOAK_CLIENT_ID` / `KEYCLOAK_CLIENT_SECRET` | Backend confidential client credentials |
| `SMTP_HOST` / `SMTP_PORT` | Email transport host and port |
| `SMTP_USER` / `SMTP_PASS` | SMTP credentials |
| `AUDIT_LOG_ENABLED` | Boolean; enable/disable audit logging (default: `true`) |

---

## UI Story

### Price Plans / Pricing Models Page

Accessible from **Settings › Pricing**. Displays a table of all pricing models with columns:
- **Model Name** — clickable to expand details
- **Pricing Type** — badge: `FLAT` (purple), `PER_UNIT` (teal), `TIERED` (amber)
- **Status** — badge: `DRAFT` (gray), `ACTIVE` (green), `ARCHIVED` (red)
- **Linked Meter** — name of the meter or "N/A" for flat subscription
- **Actions** — Edit (DRAFT only), Archive (ACTIVE), Assign

Filter bar: filter by status, pricing type, or linked meter.

---

### Create Plan / Pricing Model Wizard

**Step 1 — Basics**
- Model name (text input, required)
- Pricing type selector: `FLAT` | `PER_UNIT` | `TIERED`
- Link to meter (optional search/select dropdown)

**Step 2 — Pricing Details**
- **FLAT:** Base amount input (currency-formatted number)
- **PER_UNIT:** Unit price input + currency select
- **TIERED:** Add tier rows: `From qty` (number), `To qty` (number or "unlimited"), `Price per unit`. Live validation warns on gaps or overlaps.

**Step 3 — Plan Configuration (catalog.plans)**
- Plan name and slug
- Billing period: MONTHLY / QUARTERLY / YEARLY
- Pay in advance? (checkbox)
- Trial days (optional)

**Step 4 — Review & Publish**
- Summary card showing all configured values
- Option to save as `DRAFT` or publish immediately as `ACTIVE`

---

### Preview Calculator Widget

Embedded on the pricing model detail page:
- **Usage slider**: range 0 to configurable max
- **Real-time display:** itemized breakdown by tier + total in org's currency

---

## Dependencies & Notes for Agent

- **Schema alignment:** Uses `catalog.pricing_models` (not `price_plans`), `catalog.pricing_tiers` (not `pricing_tiers` as a standalone concept), `catalog.plans` (subscription plans), `catalog.charges` (plan-meter linkage). The ERD has `catalog.rate_cards` and `catalog.rate_card_rates` as alternative pricing structures — confirm which path (pricing_model vs rate_card) is the primary billing engine.
- **Prisma models:** `PricingModel` with enum `PricingType { FLAT PER_UNIT TIERED }` and `PricingModelStatus { DRAFT ACTIVE ARCHIVED }`; `PricingTier` linked to `PricingModel` via `pricing_model_id`; `Plan` linked to `PricingModel` optionally.
- **TIERED calculation:** Iterate tiers in `sort_order` ascending; for each tier, `billableUnits = min(quantityRemaining, tier.toQty - tier.fromQty)`. Stop when `quantityRemaining = 0`.
- **Billing engine integration:** When a subscription's billing cycle renews, the billing service must query the current active `PricingModel` to determine the rate to apply.
- **Currency handling:** All monetary values stored as `numeric` in `catalog.plans.base_amount` and `catalog.pricing_tiers.price_per_unit`. Use org's `identity.organizations.currency` setting.
- **Audit logging:** All CRUD operations must emit an `audit_log` entry with `action`, `actorId`, `modelId`, `orgId`, and `changeSnapshot` (JSON of before/after state).
- **Concurrency:** Use optimistic locking (`version` column) on `PricingModel` to prevent race conditions.
