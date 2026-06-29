# Story 14 — Security Audit Logging & Retrospective Inspections

> **Phase:** 3 — Key Creation & Control Plane Flow
> **Depends on:** Story 13 (BYOK configuration)
> **Blocks:** Nothing (Completes Phase 3)

---

## Description

As a **platform security operator**, I need a secure audit trail that logs every blocked request (such as invalid keys, exhausted budgets, or rate limit violations) to a database table, and an endpoint to query these logs so I can identify misconfigured clients, abuse, or unauthorized access attempts.

This story implements the security auditing interface:
*   Synchronous SQL write logic inside the Auth/Ingress Middleware to write log records directly to Postgres when a request is blocked.
*   `GET /v1/security-audit-logs?org_id={org_id}`: Retrieves access violation logs for analysis.

---

## Acceptance Criteria

### Access Block Auditing

| # | Criterion |
|---|---|
| 1 | When a request is blocked by the ingestion API due to `invalid_key`, write an audit log to PostgreSQL `security_audit_logs`. Set `org_id="unknown"`, capture `ip_address`, set `violation_type="invalid_key"`, and store the key's prefix (first 8 characters). |
| 2 | When a request is blocked due to `budget_exhausted` (e.g. per-key limit exceeded), write an audit log to PostgreSQL. Set the correct `org_id`, `ip_address`, and `violation_type="budget_exhausted"`, storing details about the limit and accumulated cost. |
| 3 | When a request is blocked due to rate limiting (`rate_limit`), write an audit log to PostgreSQL. Set `org_id`, `ip_address`, and `violation_type="rate_limit"`. |
| 4 | Audit logging must run synchronously before returning the error response to ensure the audit trail is complete even if the client disconnects. |

### Querying Audit Logs

| # | Criterion |
|---|---|
| 5 | `GET /v1/security-audit-logs` retrieves audit logs. |
| 6 | Supports optional query parameters: `org_id` (filters logs by organization) and `violation_type` (filters by type). |
| 7 | If the caller is not an administrator, restrict the logs returned to only those matching the caller's authenticated `org_id`. |
| 8 | Returns `200 OK` with a JSON array of security log records sorted by `created_at` descending. |

---

## Test Cases

### TC-01: Log Security Violation for Invalid Key
**When:** Client queries Ingest API with an invalid key `sk_bad_123`  
**Then:** API returns `401 Unauthorized`. In PostgreSQL, a row is inserted in `security_audit_logs` with `violation_type="invalid_key"`, `key_prefix="sk_bad_1"`, and details containing the validation error.

### TC-02: Query Audit Logs
**Given:** Three audit records exist for `org_acme`  
**When:** `GET /v1/security-audit-logs?org_id=org_acme`  
**Then:** Returns `200 OK` showing the three log entries, exposing timestamps, client IP addresses, and violation types.

---

## Data Tables / Resources Used

| Resource | Operation | Purpose |
|---|---|---|
| `security_audit_logs` (Postgres) | `INSERT`, `SELECT` | Logs and retrieves access violations |
