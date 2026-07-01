# QuantumBilling — Workflow Connectivity Analysis

**Date:** 2026-06-30  
**Purpose:** Verify that all user stories connect properly and data flows correctly between entities

---

## Entity Hierarchy (Confirmed)

```
SUPER_ADMIN
    │
    └── Organization (e.g., "Acme AI Corp")
            │
            └── Customer (e.g., "Acme AI - Engineering")
                    │
                    └── End User (e.g., "John Smith")
                            │
                            └── API Keys
                            │
                            └── Usage Events ──► Recorded via Real-time API
```

| Entity | Belongs To | Examples |
|--------|------------|----------|
| **Organization** | Platform | Acme AI Corp, DataFlow Systems |
| **Customer** | Organization | Engineering Dept, Marketing Dept |
| **End User** | Customer | John Smith, api-service@company.com |
| **Plan/Product** | (standalone) | Starter, Pro, Enterprise |
| **Subscription** | Organization → Plan | Links org to a plan |
| **Invoice** | Subscription → Organization | Generated from subscription |
| **Credits** | Organization | Organization-level balance |
| **Feature Grants** | Customer | Beta features, compensations |
| **Payment Methods** | Organization | Visa, ACH |
| **Usage Events** | End User → Customer → Organization | API calls recorded |

---

## RBAC Matrix (Confirmed)

| Role | Access Scope | Can Manage |
|------|--------------|------------|
| **SUPER_ADMIN** | Platform-wide | Everything |
| **ORG_ADMIN** | Own Organization | Org settings, customers, subscriptions, invoices, credits, payment methods |
| **CUSTOMER** | Own Customer Account | View invoices, pay, view credits (read-only), aggregate team usage |
| **END_USER** | Own End User | Own usage, own events, own API keys |

---

## User Stories — Complete List

| Story ID | Name | Entity | Phase |
|----------|------|--------|-------|
| QB-STORY-017 | Organization Overview | Organization | Sprint 3 |
| QB-STORY-020 | Platform Analytics | Platform | Sprint 6 |
| QB-STORY-021 | Reports | Organization | Sprint 6 |
| QB-STORY-022 | Subscription Management | Organization → Plan | Sprint 7 |
| QB-STORY-023 | Invoice Management | Subscription → Organization | Sprint 7 |
| QB-STORY-024 | Team Usage | Customer/Organization | Sprint 8 |
| QB-STORY-025 | Credits Management | Organization | Sprint 7 |
| QB-STORY-026 | Customer Portal | Customer | Sprint 3 |
| QB-STORY-027 | End User Events | End User | Sprint 8 |
| QB-STORY-028 | End User Dashboard | End User | Sprint 3 |
| QB-STORY-029 | Organization Onboarding | Organization | Sprint 1 |
| QB-STORY-030 | Customer Management | Customer | Sprint 1 |
| QB-STORY-031 | End User Management | End User | Sprint 1 |
| QB-STORY-032 | Payment Method Management | Organization | Sprint 2 |
| QB-STORY-033 | Feature Grants | Customer | Sprint 2 |
| QB-STORY-034 | API Key Management | End User | Sprint 1 |

---

## Workflow 1: Platform Onboarding (Complete Flow)

```
1. SUPER_ADMIN creates Organization "Acme AI Corp"
   └── QB-STORY-029: Organization Onboarding
   └── webhook: organization.created

2. SUPER_ADMIN/ORG_ADMIN sets up Payment Methods
   └── QB-STORY-032: Payment Method Management
   └── Visa, ACH added to organization

3. SUPER_ADMIN/ORG_ADMIN creates Customers under Org
   └── QB-STORY-030: Customer Management
   └── Customer 1: "Engineering"
   └── Customer 2: "Marketing"

4. SUPER_ADMIN/ORG_ADMIN assigns Subscription to Organization
   └── QB-STORY-022: Subscription Management
   └── Organization → Plan (Enterprise)
   └── Organization now has platform access

5. ORG_ADMIN creates End Users under Customers
   └── QB-STORY-031: End User Management
   └── End User: John (Engineering)
   └── End User: Jane (Engineering)
   └── End User: Bob (Marketing)

6. End Users create API Keys
   └── QB-STORY-034: API Key Management
   └── sk_prod_****abc123 (John)
   └── sk_dev_****xyz789 (John)

7. End Users make API calls → Usage Events recorded
   └── QB-STORY-027: End User Events
   └── usage_event.created (per API call)

8. Billing Engine aggregates usage
   └── QB-STORY-024: Team Usage
   └── Per end user → Per customer → Per organization

9. Invoice generated from Subscription (billing period end)
   └── QB-STORY-023: Invoice Management
   └── Credits applied automatically
   └── QB-STORY-025: Credits Management

10. Customer pays invoice
    └── QB-STORY-026: Customer Portal
    └── QB-STORY-032: Payment Method Management
```

---

## Workflow 2: End User Makes API Call

```
1. End User "John" includes API Key in request
   Authorization: Bearer sk_prod_7Kx9Ab2C...

2. Real-time API validates key
   └── QB-STORY-034: API Key Management
   └── Identifies: end_user_id = "john_123"
                   customer_id = "cust_engineering"
                   org_id = "org_acme"

3. Usage Event recorded
   usage_event {
     end_user_id: "john_123",
     customer_id: "cust_engineering",
     org_id: "org_acme",
     model: "gpt-4",
     input_tokens: 1000,
     output_tokens: 500,
     cost: $0.03,
     timestamp: ...
   }
   └── QB-STORY-027: End User Events

4. Event aggregated into:
   - End User "John" → my usage totals
     └── QB-STORY-028: End User Dashboard
   - Customer "Engineering" → team aggregate
     └── QB-STORY-024: Team Usage
   - Organization "Acme AI Corp" → org totals
     └── QB-STORY-017: Organization Overview

5. Invoice generated at billing period end
   └── QB-STORY-023: Invoice Management
   └── Includes John's usage in total
```

---

## Workflow 3: Invoice Payment Flow

```
1. Invoice generated from Subscription
   invoice {
     org_id: "org_acme",
     subscription_id: "sub_enterprise",
     total: $5,000,
     status: "pending"
   }
   └── QB-STORY-023: Invoice Management

2. Credits automatically applied
   └── QB-STORY-025: Credits Management
   └── Priority order: Compensation → Promotional → Prepaid → Commit
   └── Remaining invoice: $4,200

3. CUSTOMER views invoice in portal
   └── QB-STORY-026: Customer Portal
   └── GET /api/v1/customers/:custId/invoices

4. CUSTOMER pays invoice
   └── QB-STORY-026: Customer Portal
   └── POST /api/v1/invoices/:invId/pay

5. Payment processed via default payment method
   └── QB-STORY-032: Payment Method Management
   └── Charged to Visa •••• 4242

6. Invoice status updated
   └── status: "pending" → "paid"
```

---

## Workflow 4: Feature Grant Flow

```
1. ORG_ADMIN grants custom feature to Customer
   └── QB-STORY-033: Feature Grants
   └── Feature: "Batch API" (beta)
   └── Customer: "Engineering"

2. Grant is created
   └── status: "active"
   └── expires: 2025-03-01

3. Customer sees new feature in entitlements
   └── QB-STORY-026: Customer Portal
   └── My Entitlements → shows standard + custom grants

4. End Users in "Engineering" can use Batch API
   └── Feature check passes

5. Grant expires automatically
   └── Cron job: grant-expiration-checker
   └── status: "active" → "expired"

6. End Users can no longer use Batch API
```

---

## Workflow 5: Usage Viewing by Role

```
SUPER_ADMIN views all:
  └── QB-STORY-020: Platform Analytics
  └── QB-STORY-027: End User Events
  └── Can select any Organization → any Customer → any End User

ORG_ADMIN views org:
  └── QB-STORY-017: Organization Overview
  └── QB-STORY-024: Team Usage (all customers in org)
  └── QB-STORY-027: End User Events (all end users in org)

CUSTOMER views own:
  └── QB-STORY-026: Customer Portal
  └── QB-STORY-024: Team Usage (aggregate only — no per-user)
  └── Cannot see per-end-user breakdown

END_USER views own:
  └── QB-STORY-028: End User Dashboard
  └── QB-STORY-027: End User Events (own only)
  └── QB-STORY-034: API Key Management (own only)
```

---

## Key Points to Remember

1. **Customer ≠ Organization** — Customer is a sub-entity under Organization
2. **End User → Customer** — End users belong to Customer, not directly to Organization
3. **CUSTOMER and ORG_ADMIN are mutually exclusive** — One person has one role only
4. **Credits belong to Organization** — Not to Customer or End User
5. **Invoices are tied to Subscription** — Which belongs to Organization
6. **Usage Events flow:** End User → Customer → Organization (aggregated at each level)
7. **Team Usage at Customer level is AGGREGATE only** — No per-user breakdown
8. **API Key identifies End User** — Used for auth and usage attribution

---

## Missing User Stories — FILLED

| Gap | Story ID | Status |
|-----|----------|--------|
| Organization Onboarding | QB-STORY-029 | ✅ Created |
| Customer Management | QB-STORY-030 | ✅ Created |
| End User Management | QB-STORY-031 | ✅ Created |
| Payment Method Management | QB-STORY-032 | ✅ Created |
| Feature Grants | QB-STORY-033 | ✅ Created |
| API Key Management | QB-STORY-034 | ✅ Created |

---

## Stories Still Needed (Future Phases)

| Story | Description |
|-------|-------------|
| Dunning (detailed) | QB-STORY-012 exists, expand for detailed flow |
| Webhook Configuration | Configure webhooks per org/customer |
| Audit Log Viewer | Search and filter audit logs |
| API Rate Limiting | Configure limits per plan/end user |
| Usage Alerts | Configure alerts for usage thresholds |
| Dashboard Customization | Allow custom dashboards per role |

---

*Generated for QuantumBilling — Workflow Connectivity Analysis*
