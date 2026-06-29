# Story 11 — API Key Generation & Write-Through Caching

> **Phase:** 3 — Key Creation & Control Plane Flow
> **Depends on:** Phase 0 Story 1 (domain types), Story 2 (Redis auth setup)
> **Blocks:** Story 12 (revocation)

---

## Description

As a **tenant administrator**, I need to create new API keys for my organization, configuring custom metadata (budgets, rate limits, allowlists) and storing them securely, so that my applications can authenticate against the event ingestion endpoints immediately after key creation.

This story implements the `POST /v1/keys` endpoint. The service generates a random API key (or registers it via the upstream LiteLLM Gateway), hashes the key using SHA-256 for secure storage in PostgreSQL, and caches the raw key context contextually in Redis (`apikey:{hash}`) to update the gateway authorization path in real-time.

---

## Acceptance Criteria

### HTTP Request Validation

| # | Criterion |
|---|---|
| 1 | `POST /v1/keys` accepts a JSON request body with: `org_id` (required), `name` (required), `tenant_id` (optional), `source_mode` (optional, default: `direct_ingest`), `budget_limit_usd` (optional), `rate_limit_rpm` (optional), `allowed_models` (optional). |
| 2 | Reject requests missing `org_id` or `name` with `400 BAD_REQUEST`. |
| 3 | Reject invalid `source_mode` values (must be one of `direct_ingest`, `virtual_key`, `byok`). |

### Key Provisioning & Hashing

| # | Criterion |
|---|---|
| 4 | If `source_mode` is `virtual_key` or `byok`, attempt to provision the key in the LiteLLM Gateway proxy by calling its key generation endpoint (using the `LITELLM_PROXY_URL` and `LITELLM_MASTER_KEY` variables). |
| 5 | Fall back to local secure random key generation (`sk-live-` prefix + 48 hex characters) if the LiteLLM sync is skipped or fails. |
| 6 | Hash the generated key using SHA-256 to produce a unique, secure hex digest for database mapping. |
| 7 | Store the key prefix (first 11 characters, e.g. `sk-live-abc`) to assist debugging and administrative display. |

### Postgres & Cache Write-Through

| # | Criterion |
|---|---|
| 8 | Insert a row into the PostgreSQL `api_keys` table with: generated UUID, hashed key, prefix, org/tenant IDs, mode, active status, cost budget limit, RPM limit, allowed models, and metadata context. |
| 9 | Perform a **write-through update to Redis**: store the JSON-serialized `KeyContext` under the key `apikey:{hashedKey}`. |
| 10 | The Redis cache record must be stored permanently (no TTL) to prevent authentication failure on ingestion paths. |
| 11 | Return `201 CREATED` along with the raw unhashed key (only shown once to the creator) and metadata context. |

---

## Test Cases

### TC-01: Happy Path - Create Direct Ingest Key
**When:** `POST /v1/keys` with `org_id="org_acme"`, `name="Acme SDK Key"`, and `source_mode="direct_ingest"`  
**Then:** Returns `201 CREATED`, showing the plain key (starting with `sk-live-`); PostgreSQL contains a hashed key record; Redis contains the corresponding key context at `apikey:{hash}`.

### TC-02: Create Mode B Virtual Key with Budget Limit
**When:** `POST /v1/keys` with `org_id="org_acme"`, `source_mode="virtual_key"`, and `budget_limit_usd=100.00`  
**Then:** In Redis, key context is seeded with `source_mode="virtual_key"` and `budget_limit_usd=100.0` to activate budget-checking middleware immediately.

### TC-03: Invalid Payload Validation
**When:** `POST /v1/keys` with empty `org_id`  
**Then:** Returns `400 BAD_REQUEST` and does not persist any keys.

---

## Data Tables / Resources Used

| Resource | Operation | Purpose |
|---|---|---|
| `api_keys` (Postgres) | `INSERT` | Securely stores API key metadata and SHA-256 hash |
| `apikey:{hashed_key}` (Redis) | `SET` | Caches the JSON KeyContext for sub-millisecond gateway lookups |

---

## Environment Config Keys

| Key | Description | Default |
|---|---|---|
| `LITELLM_PROXY_URL` | Endpoint of the LiteLLM Gateway | `http://localhost:4000` |
| `LITELLM_MASTER_KEY` | Secret credentials to manage LiteLLM keys | `sk-1234` |
