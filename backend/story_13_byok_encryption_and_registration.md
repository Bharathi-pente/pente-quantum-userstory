# Story 13 — BYOK Credential Encryption & Registration

> **Phase:** 3 — Key Creation & Control Plane Flow
> **Depends on:** Story 12 (key revocation/listing infrastructure)
> **Blocks:** Story 14

---

## Description

As a **tenant administrator**, I need to configure my own third-party AI provider credentials (BYOK) so that the LiteLLM Gateway proxy can forward my requests using my own billing details with the AI provider, without exposing my raw keys to the network or storing them unencrypted on the platform.

This story implements the BYOK configuration API (`POST /v1/byok/config`). The service accepts provider credentials, encrypts the raw provider API key using **AES-256-GCM** in-memory using a Master Key derived from the server environment, and stores the resulting ciphertext and initialization vector (IV) in PostgreSQL. When requests route through the proxy gateway, the platform decrypts the credential dynamically in-memory only.

---

## Acceptance Criteria

### Master Key Resolution

| # | Criterion |
|---|---|
| 1 | Load the master key string from the environment variable `BYOK_MASTER_KEY`. |
| 2 | Format the master key to exactly 32 bytes using SHA-256 to serve as a valid AES-256 key. |
| 3 | If `BYOK_MASTER_KEY` is not set, log a warning and fall back to a secure default key string during local testing. |

### BYOK Configuration Endpoint

| # | Criterion |
|---|---|
| 4 | `POST /v1/byok/config` accepts JSON payload: `org_id` (required), `provider` (required, e.g. `openai`, `anthropic`, `google`), and `api_key` (required, the raw provider key string). |
| 5 | Verify that the target organization `org_id` exists in Postgres; return `404 NOT_FOUND` if unknown. |
| 6 | Encrypt the `api_key` using **AES-256-GCM** encryption. This returns `encrypted_key` (ciphertext) and `key_iv` (initialization vector). |
| 7 | Store the encrypted credential in PostgreSQL under table `byok_provider_keys`. |
| 8 | Return `200 OK` or `201 CREATED` indicating successful BYOK key registration (the response must NEVER show the raw secret key or its IV). |

### Decryption Helper

| # | Criterion |
|---|---|
| 9 | Implement a retrieval helper `GetBYOKProviderKey(ctx, org_id, provider) (string, error)` in the PostgreSQL database layer. |
| 10 | Queries `byok_provider_keys` to fetch the `encrypted_key` and `key_iv` bytes. |
| 11 | Decrypts the ciphertext using the AES-256-GCM key and returns the raw provider API key string. |
| 12 | The decrypted key must remain in-memory and be discarded immediately after the gateway proxy request completes. |

---

## Test Cases

### TC-01: Configure and Store BYOK Key
**When:** `POST /v1/byok/config` with `org_id="org_acme"`, `provider="openai"`, `api_key="sk-proj-12345..."`  
**Then:** Returns `201 CREATED`. In PostgreSQL, a row is inserted in `byok_provider_keys` with encrypted bytes and IV; verify that `encrypted_key` is not human-readable.

### TC-02: Retrieve and Decrypt helper
**When:** Invoking `GetBYOKProviderKey(ctx, "org_acme", "openai")`  
**Then:** Successfully decrypts and returns `sk-proj-12345...`. If the Master Key is changed, decryption fails with an error.

---

## Data Tables / Resources Used

| Resource | Operation | Purpose |
|---|---|---|
| `byok_provider_keys` (Postgres) | `INSERT`, `SELECT` | Stores AES-256-GCM encrypted provider credentials and IV bytes |
