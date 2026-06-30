# QuantumBilling User Story: Feature Grants

**QB-STORY-033** · Sprint 2 · Phase: Entitlements

---

## Title

**Feature Grants** — grant custom features and entitlements to customers outside the standard plan

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As an ORG_ADMIN or SUPER_ADMIN**, I want to grant custom features and entitlements to customers outside the standard plan, so that I can offer beta features, compensatory access, or special arrangements.

**Core Concept:** Feature Grants override or extend what a customer's standard plan includes. They are tied to a **Customer** and can have expiration dates.

---

## Entity Model

```
Organization
    └── Customer (e.g., "Acme AI")
            ├── Plan: Enterprise (standard features)
            └── Feature Grants:
                    ├── Batch API (beta) — expires 2025-03-01
                    └── Priority Support (compensation) — expires 2025-12-31
```

---

## RBAC Roles

| Role | Can manage feature grants | Scope |
|------|--------------------------|-------|
| **SUPER_ADMIN** | Yes (all customers) | Platform-wide |
| **ORG_ADMIN** | Yes (own customers) | Own org only |
| **CUSTOMER** | No | Read-only (see entitlements) |
| **END_USER** | No | No access |

---

## Acceptance Criteria

### Grant Feature

1. ORG_ADMIN or SUPER_ADMIN can grant a custom feature to a customer.
2. Required fields: Feature ID/Name, Customer, Reason.
3. Optional fields: Expiration Date, Scope (global or per-end-user).
4. Feature grant is created with status `active`.
5. Feature grant can be revoked at any time.

### Feature Grant List

6. ORG_ADMIN sees all feature grants for their customers.
7. List shows: Customer, Feature Name, Granted Date, Expiration, Reason, Status.
8. Filters: Customer, Feature, Status (Active/Expired/Revoked).

### Feature Grant Expiration

9. Feature grants with expiration dates auto-expire.
10. Cron job checks daily for expired grants.
11. Expired grants have status `expired`.
12. Expired grants no longer provide access.

### Revoke Feature Grant

13. ORG_ADMIN can revoke a feature grant.
14. Revoked grants have status `revoked`.
15. Revocation is immediate — access is cut off right away.

### Customer Views Entitlements

16. CUSTOMER sees their standard plan features PLUS custom grants.
17. Customer portal shows:
    - Plan Features (standard)
    - Custom Grants (with expiration if applicable)
18. Cannot modify grants (read-only).

### Feature Grant Scopes

19. **Global Grant:** All end users under the customer get the feature.
20. **Per-End-User Grant:** Only specified end users get the feature.

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/customers/:customerId/entitlements` | Grant feature |
| `GET` | `/api/v1/customers/:customerId/entitlements` | List grants |
| `PUT` | `/api/v1/entitlements/:grantId` | Update grant |
| `POST` | `/api/v1/entitlements/:grantId/revoke` | Revoke grant |
| `GET` | `/api/v1/entitlements/:grantId` | Get grant details |
| `GET` | `/api/v1/customers/:customerId/entitlements/active` | Active grants only |

---

## Test Cases

### TC-01 — Grant beta feature

**Given:** customer "Acme AI" is on Enterprise plan
**When:** granting "Batch API" beta feature with expiration 2025-03-01
**Then:** grant is created with status "active"
**And:** customer sees Batch API in their entitlements
**And:** grant expires on 2025-03-01 automatically

### TC-02 — Revoke grant immediately

**Given:** customer has "Batch API" grant
**When:** SUPER_ADMIN revokes the grant
**Then:** grant status becomes "revoked"
**And:** access is cut off immediately

### TC-03 — Customer sees custom grants

**Given:** customer has custom grants
**When:** viewing "My Entitlements" in portal
**Then:** standard plan features are shown
**And:** custom grants are shown separately with expiration dates

---

## Dependencies

- Cron job: `grant-expiration-checker` (daily)
- Webhooks: `entitlement.granted`, `entitlement.revoked`, `entitlement.expired`
- Audit log: entitlement changes logged
