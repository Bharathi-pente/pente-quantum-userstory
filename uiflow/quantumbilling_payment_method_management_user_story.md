# QuantumBilling User Story: Payment Method Management

**QB-STORY-032** · Sprint 2 · Phase: Billing Foundation

---

## Title

**Payment Method Management** — add, manage, and configure payment methods for organizations

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As an ORG_ADMIN, CUSTOMER, or SUPER_ADMIN**, I want to manage payment methods for an organization, so that invoices can be paid and billing can proceed smoothly.

**Core Concept:** Payment methods are attached to an **Organization** (not Customer or End User). They are used to pay invoices generated from subscriptions.

---

## Entity Model

```
Organization
    └── Payment Method 1 (Visa •••• 4242) — DEFAULT
    └── Payment Method 2 (ACH •••• 9876)
```

---

## RBAC Roles

| Role | Can manage payment methods | Scope |
|------|---------------------------|-------|
| **SUPER_ADMIN** | Yes (all orgs) | Platform-wide |
| **ORG_ADMIN** | Yes (own org) | Own org only |
| **CUSTOMER** | Yes (own org) | Read-only on other's methods |
| **END_USER** | No | No access |

---

## Acceptance Criteria

### Add Payment Method

1. ORG_ADMIN or CUSTOMER can add a payment method to their organization.
2. Supported types: Credit Card, ACH (Bank Transfer).
3. **Credit Card fields:** Card Number (via Stripe/billing provider token), Expiry, CVC (not stored), Billing Address.
4. **ACH fields:** Bank Account (via Plaid/tokenization), Routing Number, Account Type (Checking/Savings).
5. Payment method is validated before being saved.
6. Newly added payment methods are NOT set as default automatically.

### Payment Method List

7. ORG_ADMIN sees all payment methods for their organization.
8. Each payment method shows: Type, Last 4 Digits, Expiry (card only), Status (Default/Active), Added Date.
9. Cannot delete a payment method if:
   - It is the ONLY payment method
   - It is the DEFAULT and there are pending invoices

### Set Default Payment Method

10. ORG_ADMIN can set any active payment method as the default.
11. Default payment method is used for auto-pay on invoices.
12. Only ONE payment method can be default at a time.

### Remove Payment Method

13. ORG_ADMIN can remove a non-default payment method.
14. Cannot remove default payment method if there are pending invoices.
15. Removing a payment method does not delete payment history.

### Payment Method Validation

16. Expired cards are flagged with warning status.
17. Failed payment methods (e.g., card declined in past) are flagged.
18. ORG_ADMIN is notified of payment method issues.

### Payment Gateway Integration

19. Actual card/bank details are stored in Stripe (or similar) — never in QuantumBilling directly.
20. QuantumBilling stores only: token reference, last 4, expiry, type, status.
21. Payment processing happens via the payment gateway API.

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/organizations/:orgId/payment-methods` | Add payment method |
| `GET` | `/api/v1/organizations/:orgId/payment-methods` | List payment methods |
| `PUT` | `/api/v1/payment-methods/:pmId` | Update payment method |
| `DELETE` | `/api/v1/payment-methods/:pmId` | Remove payment method |
| `PUT` | `/api/v1/payment-methods/:pmId/default` | Set as default |
| `POST` | `/api/v1/payment-methods/:pmId/validate` | Validate payment method |

---

## Test Cases

### TC-01 — Add credit card

**Given:** ORG_ADMIN for "Acme AI"
**When:** adding a credit card with token from Stripe
**Then:** payment method is saved with type "card" and last 4 "4242"
**And:** status is "active"

### TC-02 — Set default payment method

**Given:** organization has 2 payment methods
**When:** setting the second one as default
**Then:** first one is no longer default
**And:** second one is now marked "default"

### TC-03 — Cannot delete only payment method

**Given:** organization has 1 payment method
**When:** attempting to delete it
**Then:** error: "Cannot delete the only payment method"
**And:** deletion is blocked

---

## Dependencies

- Payment gateway: Stripe (or similar)
- Tokens stored in billing provider, not in QuantumBilling
- Webhooks: `payment_method.added`, `payment_method.removed`, `payment_method.default_changed`
- Audit log: payment method changes logged
