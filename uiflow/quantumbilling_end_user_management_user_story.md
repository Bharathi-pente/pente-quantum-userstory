# QuantumBilling User Story: End User Management

**QB-STORY-031** · Sprint 1 · Phase: Foundation

---

## Title

**End User Management** — create and manage end users within a customer account

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As an ORG_ADMIN, CUSTOMER, or SUPER_ADMIN**, I want to create and manage end users within a customer account, so that team members can access the API and have their usage tracked individually.

**Core Concept:** An **End User** is an individual API consumer (developer, service, or application) who belongs to a **Customer**. End Users make API calls and their usage is tracked per-user for:
- Cost attribution
- Usage monitoring
- Team analytics

---

## Entity Model

```
Organization
    └── Customer
            └── End User 1 (e.g., "john@company.com")
            │       └── API Keys (multiple)
            │       └── Usage Events
            │
            └── End User 2 (e.g., "api-service@company.com")
                    └── API Keys
                    └── Usage Events
```

---

## RBAC Roles

| Role | Can manage end users | Scope |
|------|---------------------|-------|
| **SUPER_ADMIN** | Yes (all end users) | Platform-wide |
| **ORG_ADMIN** | Yes (all customers in org) | Own org only |
| **CUSTOMER** | Yes (own customer) | Own customer only |
| **END_USER** | No (can only manage own API keys) | No access |

---

## Acceptance Criteria

### End User Creation

1. ORG_ADMIN or CUSTOMER can create an end user under their scope.
2. Required fields: Name, Email.
3. Optional fields: External ID (for mapping to external systems), Metadata.
4. End user is assigned a unique `end_user_id`.
5. End user receives an invitation email (optional — can be API-only).
6. End user status defaults to `active`.

### End User List

7. ORG_ADMIN sees all end users under their organization (all customers).
8. CUSTOMER sees all end users under their customer account.
9. End user list shows: Name, Email, Status, API Keys Count, Total Usage, Created Date.
10. Filters: Status, Search by Name/Email, Customer (ORG_ADMIN only).

### End User Detail

11. Clicking an end user shows:
    - Profile information
    - API Keys
    - Usage Summary (tokens, requests, cost)
    - Recent Events
    - Activity Log

### API Key Management (for End User)

12. End users can create their own API keys (self-service).
13. ORG_ADMIN / CUSTOMER can create API keys on behalf of end users.
14. API key fields: Name, Expiration (optional).
15. API key is shown ONCE at creation (full value, never stored in plaintext).
16. API keys can be revoked at any time.

### End User Status

17. End user can be: active, suspended, canceled.
18. Suspending an end user:
    - Revokes all their API keys
    - Blocks their API access
19. Deleting an end user:
    - All API keys are revoked
    - Usage history is preserved
    - Cannot be undone

### Invitation Flow

20. Optional: Send invitation email to end user.
21. Invitation includes: Name, Email, Organization, Setup Link.
22. End user accepts invitation and sets password (if using password auth).
23. ORG_ADMIN can also create API-only end users (no invitation sent).

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/customers/:customerId/end-users` | Create end user |
| `GET` | `/api/v1/end-users/:endUserId` | Get end user |
| `PUT` | `/api/v1/end-users/:endUserId` | Update end user |
| `POST` | `/api/v1/end-users/:endUserId/suspend` | Suspend end user |
| `POST` | `/api/v1/end-users/:endUserId/reactivate` | Reactivate end user |
| `DELETE` | `/api/v1/end-users/:endUserId` | Delete end user |
| `POST` | `/api/v1/end-users/:endUserId/invite` | Send invitation |
| `GET` | `/api/v1/customers/:customerId/end-users` | List end users for customer |
| `GET` | `/api/v1/organizations/:orgId/end-users` | List end users for org (ORG_ADMIN) |
| `POST` | `/api/v1/end-users/:endUserId/api-keys` | Create API key for end user |
| `GET` | `/api/v1/end-users/:endUserId/api-keys` | List API keys |
| `DELETE` | `/api/v1/api-keys/:keyId` | Revoke API key |

---

## API Key Lifecycle

```
┌─────────┐   create    ┌─────────┐   revoke   ┌──────────┐
│ (none)  │──────────►│  active │──────────►│ revoked │
└─────────┘           └────┬────┘           └──────────┘
                           │
                           │ expires
                           ▼
                      ┌──────────┐
                      │ expired  │
                      └──────────┘
```

---

## Test Cases

### TC-01 — Create end user

**Given:** CUSTOMER for "Acme AI - Engineering"
**When:** creating an end user "John Smith" with email "john@acme.ai"
**Then:** end user is created with status "active"
**And:** can create API keys immediately

### TC-02 — Create API key for end user

**Given:** end user "John Smith" exists
**When:** creating an API key "Production Key"
**Then:** API key is generated
**And:** full key value is shown ONCE
**And:** key is stored as hash only

### TC-03 — Suspend end user

**Given:** end user "John Smith" is active
**When:** ORG_ADMIN suspends the end user
**Then:** all API keys are revoked
**And:** API calls return 403 Forbidden
**And:** existing usage data is preserved

### TC-04 — End user creates own API key

**Given:** end user "John Smith" is authenticated
**When:** creating an API key via self-service
**Then:** key is created and shown once
**And:** John can use the key immediately

---

## Dependencies

- Requires: ORG_ADMIN, CUSTOMER, or SUPER_ADMIN
- Webhooks: `end_user.created`, `end_user.suspended`, `end_user.deleted`, `api_key.created`, `api_key.revoked`
- Audit log: end user management logged
