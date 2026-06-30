# QuantumBilling User Story: Organization Onboarding

**QB-STORY-029** · Sprint 1 · Phase: Foundation

---

## Title

**Organization Onboarding** — create organizations and set up initial admin access

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As a SUPER_ADMIN**, I want to create a new organization and set up its initial configuration, so that a new customer can be onboarded onto the QuantumBilling platform.

The Organization Onboarding flow covers:

- **Organization Creation** — create the organization record
- **Initial Setup** — configure billing settings, payment terms, currency
- **ORG_ADMIN Assignment** — assign an initial organization administrator
- **Organization Status** — manage organization active/trial/suspended states

---

## Entity Model

```
Organization (created here)
    └── Created by: SUPER_ADMIN
    └── Has: name, billing_email, billing_address, currency, payment_terms
    └── Status: trial → active → suspended → canceled
```

---

## Acceptance Criteria

### Organization Creation

1. SUPER_ADMIN can create a new organization via UI or API.
2. Required fields: Name, Billing Email.
3. Optional fields: Billing Address, Phone, Website, Currency, Payment Terms.
4. Organization is assigned a unique `organization_id`.
5. Organization status defaults to `trial`.

### Organization Settings

6. Initial configuration includes:
   - Billing email (for invoices and notifications)
   - Currency (default: USD)
   - Payment Terms (default: Net 30)
   - Timezone (for billing calculations)
7. Settings can be edited after creation.

### ORG_ADMIN Assignment

8. During or after organization creation, SUPER_ADMIN assigns at least one ORG_ADMIN.
9. ORG_ADMIN receives an invitation email with setup instructions.
10. ORG_ADMIN must accept invitation and set their password.
11. Organization cannot be used until at least one ORG_ADMIN is activated.

### Organization Status Flow

```
trial (default on creation)
    │
    │ trial_period_end OR subscription_assigned
    ▼
active
    │
    │ payment_failed (past_due → suspended) OR manual_suspend
    ▼
suspended
    │
    │ payment_resumed OR manual_reactivate
    ▼
active

    │
    │ manual_cancel OR max_suspension_reached
    ▼
canceled
```

12. Trial period is configurable (default: 14 days).
13. During trial, organization has limited access.
14. Trial expiration blocks API access until subscription is assigned.

### Organization List (SUPER_ADMIN)

15. SUPER_ADMIN sees all organizations in a list/table.
16. Organization list shows: Name, Status, MRR, Customers Count, Created Date.
17. Filters: Status, Date Range, Search by Name.

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/organizations` | Create organization |
| `GET` | `/api/v1/organizations/:orgId` | Get organization details |
| `PUT` | `/api/v1/organizations/:orgId` | Update organization |
| `POST` | `/api/v1/organizations/:orgId/invite-admin` | Invite ORG_ADMIN |
| `POST` | `/api/v1/organizations/:orgId/suspend` | Suspend organization |
| `POST` | `/api/v1/organizations/:orgId/reactivate` | Reactivate organization |
| `POST` | `/api/v1/organizations/:orgId/cancel` | Cancel organization |
| `GET` | `/api/v1/platform/organizations` | List all organizations (SuperAdmin) |

---

## Test Cases

### TC-01 — Create organization

**Given:** SUPER_ADMIN
**When:** creating a new organization with name "Acme AI" and billing email "billing@acme.ai"
**Then:** organization is created with status "trial" and unique org_id

### TC-02 — Invite ORG_ADMIN

**Given:** organization is created
**When:** SUPER_ADMIN invites an ORG_ADMIN with email "admin@acme.ai"
**Then:** invitation email is sent
**And:** ORG_ADMIN can accept and set password

### TC-03 — Organization cannot be used without ORG_ADMIN

**Given:** organization "Acme AI" is created but no ORG_ADMIN is accepted
**When:** attempting to access the organization
**Then:** access is blocked with message "No administrator configured"

---

## Dependencies

- Requires: SUPER_ADMIN role
- Triggers: Webhook `organization.created`
- Audit log: organization creation logged
