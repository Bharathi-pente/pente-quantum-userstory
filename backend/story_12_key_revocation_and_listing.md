# Story 12 — API Key Revocation & Listing

> **Phase:** 3 — Key Creation & Control Plane Flow
> **Depends on:** Story 11 (key creation)
> **Blocks:** Story 13

---

## Description

As a **tenant administrator**, I need to list all API keys registered for my organization so I can verify their settings, and immediately revoke any key if it is compromised or no longer needed.

This story implements the key retrieval and revocation control plane endpoints:
*   `GET /v1/keys?org_id={org_id}`: Lists keys, returning metadata, prefix, and settings (with raw secrets masked for security).
*   `DELETE /v1/keys/{id}`: Marks the key as revoked in Postgres, and synchronously evicts/deletes the context key from the Redis cache (`apikey:{hash}`) to ensure instant rejection on all data ingestion paths.

---

## Acceptance Criteria

### Listing API Keys

| # | Criterion |
|---|---|
| 1 | `GET /v1/keys` requires `org_id` as a query parameter; returns `400 BAD_REQUEST` if missing. |
| 2 | Queries PostgreSQL `api_keys` for all keys matching the `org_id`. |
| 3 | The response must NOT include the SHA-256 hash or raw key string. It must only expose the key `id`, `name`, `key_prefix`, `source_mode`, `status`, `budget_limit_usd`, and `rate_limit_rpm`. |
| 4 | Returns `200 OK` with a JSON array of keys. |

### Revoking API Keys

| # | Criterion |
|---|---|
| 5 | `DELETE /v1/keys/{id}` accepts the key UUID (`id`) in the path parameter. |
| 6 | Queries PostgreSQL to retrieve the key metadata (specifically the key hash). |
| 7 | If the key does not exist in Postgres, return `404 NOT_FOUND`. |
| 8 | Update the key's status to `revoked` in the PostgreSQL database. |
| 9 | Synchronously **evict the cached key context from Redis**: delete the Redis key `apikey:{keyHash}`. |
| 10 | Return `200 OK` or `204 NO_CONTENT` confirming the successful revocation. |
| 11 | Any subsequent requests attempting to authenticate with the revoked key must be rejected immediately by the Auth Middleware with `401 UNAUTHORIZED`. |

---

## Test Cases

### TC-01: List Keys for Organization
**Given:** Three active keys and one revoked key exist in PostgreSQL for `org_acme`  
**When:** `GET /v1/keys?org_id=org_acme`  
**Then:** Returns `200 OK` with an array of all keys; verify that the raw keys and hashes are not in the response payload.

### TC-02: Revoke Key & Evict Cache
**Given:** An active key `sk_test_123` with hash `abc...` is cached in Redis  
**When:** `DELETE /v1/keys/{key_id}`  
**Then:** PostgreSQL status becomes `revoked`; Redis key `apikey:abc...` is deleted. Next call using `sk_test_123` is blocked with `401`.

---

## Data Tables / Resources Used

| Resource | Operation | Purpose |
|---|---|---|
| `api_keys` (Postgres) | `SELECT`, `UPDATE` | Read keys by org ID, mark status as `revoked` |
| `apikey:{hashed_key}` (Redis) | `DEL` | Deletes the cached key context to block further requests |
