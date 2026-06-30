# Phase 3 — Key Creation & Control Plane Flow

> **Status:** Greenfield Specification | **Scope:** Build the Control Plane CRUD APIs for managing organization keys, tenant registers, encrypted BYOK credentials, and security logs in the unified event billing engine.
>
> This is the **Phase 3 blueprint** — the control plane specification. It defines the key management APIs through which platform operators and organizations generate, rotate, and revoke virtual, direct-ingest, and BYOK credentials. It also covers secure storage of third-party credentials via AES-256-GCM and the creation of the security audit trail.
>
> **Note:** Phase 2 is reserved for the **Billing Worker** (Kafka → Redis real-time token counters + WebSocket balance push). It is defined separately and not included in this specification.

---

## Description

As a **platform administrator or org manager**, I need a secure, developer-friendly control plane that manages the lifecycle of API keys, configures dynamic multi-tenant user groupings (tenants), encrypts third-party AI provider credentials at rest (BYOK), and exposes audit records for requests blocked due to budget or authentication failures.

The key creation flow ensures that whenever keys are created, updated, or revoked, the modifications write-through to Postgres (persistence) and are immediately cached in Redis (hot-path lookup) to keep API gateway enforcement in sync within milliseconds.

### Architecture Flow

```
Admin Client → POST /v1/keys → Database (Insert Postgres) → Write-Through (Set Redis) → Return API Key
                                                                 
Gateway Proxy (LiteLLM) ← Auth Middleware ← Query Redis (Hot Path apikey:{hash})
```

---

## RBAC / Auth Context

While ingestion only checks key validity, the control plane enforces role-based access for modifications:

| Role | Scope | Allowed Actions |
|---|---|---|
| **Platform Operator** | Platform-wide | CRUD all keys, view all logs, provision keys across all orgs |
| **Org Admin** | Single Organization | Create/revoke keys for their org, configure BYOK provider keys, register tenants |
| **Org Developer** | Single Tenant | List keys, read non-sensitive key metadata |

---

## Acceptance Criteria

### API Key Generation
1. `POST /v1/keys` accepts parameters: `org_id`, `tenant_id`, `name`, `source_mode` (direct_ingest/virtual_key/byok), `budget_limit_usd`, and `rate_limit_rpm`.
2. Generates a secure random API key with prefix `sk-live-` (local) or provisions it dynamically from the LiteLLM gateway proxy.
3. Hashes the key using SHA-256 and stores it in Postgres with its prefix, status (`active`), budget limit, and permissions.
4. Performs write-through: caches the full `KeyContext` in Redis under `apikey:{hash}`.

### API Key Revocation & Listing
5. `DELETE /v1/keys/{id}` marks the key status as `revoked` in Postgres.
6. Evicts the key context from Redis immediately to ensure instant lockout.
7. `GET /v1/keys` lists all keys belonging to an organization (with hashed secrets hidden).

### BYOK Configuration
8. `POST /v1/byok/config` accepts provider credentials (`openai`, `anthropic`, `google`) for an organization.
9. Encrypts the raw provider API key using AES-256-GCM at rest using a server-side Master Key (`BYOK_MASTER_KEY`).
10. Saves the encrypted bytes and GCM initialization vector (IV) in the database.

### Security Logging
11. Exceeded budgets, rate limits, or invalid keys are blocked and logged synchronously to the `security_audit_logs` table in Postgres.
12. `GET /v1/security-audit-logs` exposes the audit logs for administrators to inspect access violations.

---

## Phase 3 Completion Checklist

- [ ] Structs `APIKey`, `Tenant`, `BYOKProviderKey`, and `SecurityAuditLog` mapped in PostgreSQL database package
- [ ] AES-256-GCM encryption utility package (`Encrypt`, `Decrypt`) implemented in Go
- [ ] Master key resolution from environment variable `BYOK_MASTER_KEY` at startup
- [ ] `POST /v1/keys` endpoint: generates key → hashes key → inserts to Postgres → write-through to Redis cache
- [ ] `DELETE /v1/keys/{id}` endpoint: Postgres status update (`revoked`) → evict Redis `apikey:{hash}`
- [ ] `GET /v1/keys` endpoint: retrieve active keys by `org_id`, masking raw secrets
- [ ] `POST /v1/byok/config` endpoint: AES encrypts provider key → stores encrypted bytes and IV in `byok_provider_keys`
- [ ] `GET /v1/byok/config` decryption: retrieves and decrypts key in-memory for proxy consumption
- [ ] Security violation log interface (`LogSecurityViolation`) implemented to write audit trails to Postgres
- [ ] Integration tests verifying CRUD lifecycle, key revocation cache eviction, and BYOK credential roundtrip
