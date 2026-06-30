# QuantumBilling User Story: API Key Management

**QB-STORY-034** · Sprint 1 · Phase: Foundation

---

## Title

**API Key Management** — create, manage, and secure API keys for end users

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As an ORG_ADMIN, CUSTOMER, or END_USER**, I want to manage API keys for end users, so that applications can authenticate to the QuantumBilling API and have their usage tracked.

**Core Concept:** An **API Key** is a secret token that identifies an End User when making API calls. API keys are scoped to an **End User** and are used for:
- Authentication — identify who is making the request
- Authorization — verify the end user has access
- Usage Tracking — attribute API calls to a specific end user
- Cost Attribution — calculate costs per end user

---

## Entity Model

```
End User (john@company.com)
    └── API Key 1: "Production Key" — sk_prod_****abc123 — status: active
    └── API Key 2: "Development Key" — sk_dev_****xyz789 — status: active
```

---

## RBAC Roles

| Role | Can manage own keys | Can manage other's keys | Scope |
|------|-------------------|------------------------|-------|
| **SUPER_ADMIN** | N/A | Yes (all keys) | Platform-wide |
| **ORG_ADMIN** | N/A | Yes (all keys in org) | Own org |
| **CUSTOMER** | N/A | Yes (all keys in customer) | Own customer |
| **END_USER** | Yes (own keys only) | No | Own keys only |

---

## Acceptance Criteria

### API Key Creation

1. END_USER can create API keys for themselves.
2. ORG_ADMIN / CUSTOMER can create API keys on behalf of end users.
3. Required fields: Key Name (e.g., "Production API Key").
4. Optional fields: Expiration Date.
5. On creation:
   - Key is generated: format `sk_{env}_{prefix}_{random}` (e.g., `sk_prod_7Kx9Ab2C`)
   - Full key is shown ONCE in a modal/dialog
   - User must copy the key immediately — it cannot be retrieved later
   - Key hash is stored (never the plaintext)

### API Key Structure

6. API key format: `sk_{environment}_{8-char-prefix}_{32-char-random}`
   - `sk` — QuantumBilling key prefix
   - `environment` — `prod`, `dev`, or `test`
   - `8-char-prefix` — used for display (e.g., `sk_prod_****7Kx9A`)
   - `32-char-random` — the secret portion

7. The `8-char-prefix` is stored with the key for identification.
8. The full key is NEVER stored — only its hash.

### API Key List

9. END_USER sees their own API keys.
10. ORG_ADMIN sees all API keys for all end users in their scope.
11. API Key list shows:
    - Key Name
    - Masked Key (e.g., `sk_prod_****7Kx9A`)
    - Environment (Prod/Dev/Test)
    - Status (Active, Expiring Soon, Expired, Revoked)
    - Last Used (timestamp or "Never")
    - Created Date
    - Expires (date or "Never")

### API Key Status

12. **Active:** Key can be used for API calls.
13. **Expiring Soon:** Key expires within 30 days (warning state).
14. **Expired:** Key has passed its expiration date.
15. **Revoked:** Key was manually deleted/revoked.

### Revoke API Key

16. END_USER can revoke their own keys.
17. ORG_ADMIN / CUSTOMER can revoke keys in their scope.
18. Revocation is immediate — the key cannot be used after revocation.
19. A revoked key cannot be un-revoked — must create a new key.

### API Key Expiration

20. Keys with expiration dates auto-expire.
21. Expired keys cannot be used for API calls.
22. Expiration can be extended by creating a new key (cannot extend existing).

### Use API Key

23. End user's application includes key in request:
    ```
    Authorization: Bearer sk_prod_7Kx9Ab2C...
    ```
24. API validates key:
    - Key exists in database
    - Key hash matches
    - Key is not expired or revoked
    - End user is active
25. If valid: request proceeds, usage is tracked
26. If invalid: 401 Unauthorized returned

### Rate Limiting

27. API keys are subject to rate limits (per plan configuration).
28. Rate limit headers returned on each response:
    - `X-RateLimit-Limit`
    - `X-RateLimit-Remaining`
    - `X-RateLimit-Reset`

---

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/v1/end-users/:endUserId/api-keys` | Create API key |
| `GET` | `/api/v1/end-users/:endUserId/api-keys` | List API keys |
| `GET` | `/api/v1/api-keys/:keyId` | Get API key details |
| `DELETE` | `/api/v1/api-keys/:keyId` | Revoke API key |
| `POST` | `/api/v1/api-keys/:keyId/rotate` | Rotate API key |

---

## API Key Lifecycle

```
┌─────────┐   create    ┌─────────┐   expire    ┌──────────┐
│ (none)  │──────────►│  active │───────────►│ expired  │
└─────────┘           └────┬────┘             └──────────┘
                           │
                           │ revoke
                           ▼
                      ┌──────────┐
                      │ revoked  │
                      └──────────┘
```

---

## Test Cases

### TC-01 — End user creates API key

**Given:** END_USER "John" is authenticated
**When:** creating an API key named "Production Key"
**Then:** key is generated and shown ONCE
**And:** John copies and saves the key
**And:** the key appears in John's key list as "Active"

### TC-02 — API key used for authentication

**Given:** END_USER has API key `sk_prod_7Kx9Ab2C...`
**When:** making an API call with the key
**Then:** usage is recorded for the end user
**And:** response includes rate limit headers

### TC-03 — Invalid API key rejected

**Given:** a request is made with an invalid/revoked key
**When:** API validates the key
**Then:** 401 Unauthorized is returned
**And:** no usage is recorded

### TC-04 — Expired key rejected

**Given:** an API key has passed its expiration date
**When:** making an API call with the expired key
**Then:** 401 Unauthorized is returned
**And:** response: "API key has expired"

### TC-05 — Revoke API key

**Given:** END_USER has an API key they no longer need
**When:** clicking "Revoke" on the key
**Then:** key status changes to "revoked"
**And:** the key cannot be used for API calls
**And:** usage stops being tracked for that key

### TC-06 — ORG_ADMIN views all keys in org

**Given:** ORG_ADMIN for "Acme AI"
**When:** viewing API keys list
**Then:** all keys for all end users under Acme AI are shown
**And:** can filter by end user

---

## Dependencies

- Key hashing: bcrypt or sha256
- Key generation: cryptographically secure random
- Webhooks: `api_key.created`, `api_key.revoked`
- Audit log: key creation and revocation logged
- Rate limiting: per key or per end user
