# QuantumBilling User Story: End User Events

**QB-STORY-027** · Sprint 8 · Phase: Usage Tracking

---

## Title

**End User Events** — individual event log, usage metrics, and API key management for end users

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As an END_USER**, I want to view my own usage metrics, event log, and manage my API keys, so that I can monitor my API consumption, troubleshoot individual API calls, and manage my authentication credentials.

**Core Concept:** An **End User** is an individual team member or API consumer within an Organization. Each End User has their own API keys, makes their own API calls, and has their own usage records. The End User Portal shows only **their own data** — they cannot see other end users' usage or organization-level data.

The End User Portal has three sections:
- **Dashboard** — summary metrics and usage by model
- **My Events** — real-time event log of individual API calls
- **API Keys** — manage authentication keys

---

## RBAC Roles — Data Access Hierarchy

```
SUPER_ADMIN ──────────────────────► Organization
   │                                    │
   │                                    ▼
   │                              Customer
   │                                    │
   │                                    ▼
   │                              End User
```

| Role | Can view End User events | Scope |
|------|------------------------|-------|
| **SUPER_ADMIN** | Yes — all end users across all organizations | Platform-wide |
| **ORG_ADMIN** | Yes — all end users within their organization | Own org only |
| **CUSTOMER** | Yes — all end users within their customer account | Own customer only |
| **END_USER** | Yes — only their own events | Own data only |

**Access Path:**
- **SUPER_ADMIN** → navigates to `/platform/end-users/:endUserId/events` or `/organizations/:orgId/end-users/:endUserId/events`
- **ORG_ADMIN** → navigates to `/organizations/:orgId/end-users/:endUserId/events`
- **CUSTOMER** → navigates to `/my-account/end-users/:endUserId/events` (or via Team Usage)
- **END_USER** → navigates to `/my-usage` or `/my-events`

**Key Rule:** All roles can view End User events, but the scope differs:
- SUPER_ADMIN sees ALL events across ALL organizations
- ORG_ADMIN sees ALL events within their organization
- CUSTOMER sees ALL events within their customer account
- END_USER sees ONLY their own events (strict `end_user_id` match required)

---

## End User Portal Navigation

| Nav Item | Description |
|----------|-------------|
| Dashboard | Summary metrics and usage by model |
| My Events | Real-time event log |
| API Keys | Manage authentication keys |

---

## Acceptance Criteria

### End User Dashboard

1. Dashboard is accessible at `/my-usage` for authenticated END_USER.
2. Header shows "My Usage" with current billing period.
3. **Summary Cards (3-up):**
   - **My Token Usage** — total tokens consumed (input + output + cached)
   - **My Requests** — total number of API calls
   - **Active API Keys** — count of API keys with status "active"
4. **Usage by Model** section — grid of model cards showing:
   - Model name (e.g., GPT-4, Claude 3, Gemini)
   - Token count for that model
   - Percentage bar showing relative usage
5. Dashboard shows ONLY the authenticated end user's data.

### My Events — Event Log

6. **My Events** tab shows a real-time log of the end user's individual API calls.
7. **Live Toggle** — button to enable/disable live updates:
   - "Live" (green) — events update in real-time
   - "Paused" (gray) — no auto-refresh
8. **Stats Bar (4-up):**
   - **Events** — total count of events in the current view
   - **Total Tokens** — sum of input + output + cached tokens
   - **Total Cost** — sum of all event costs
   - **Error Rate** — percentage of events with status "error"
9. **Model Filter** — toggle buttons to filter by model:
   - All, GPT-4, GPT-3.5, Claude 3, Claude 3 Opus, Gemini, etc.
10. **Events Table** with columns:
    - Event ID (monospace, clickable)
    - Timestamp (YYYY-MM-DD HH:mm:ss.SSS)
    - Model (colored badge)
    - Input Tokens
    - Output Tokens
    - Latency (ms) — highlighted if > 2000ms
    - Cost ($)
    - Status (success/error)
11. Clicking an event row expands to show **Event Detail** with full payload:
    - All fields from the event
    - JSON payload display
12. Events are paginated (50 per page) or infinite scroll.
13. Events support sorting by timestamp (newest first default).
14. Events can be filtered by: date range, model, status (success/error).

### Event Detail (Expanded Row)

15. Clicking an event expands to show:
    - Event ID
    - Timestamp
    - Model
    - Input Tokens
    - Output Tokens
    - Cached Tokens
    - Latency (ms)
    - Cost ($)
    - Status
    - Endpoint (e.g., `/v1/chat/completions`)
    - Request/Response payload (JSON)
16. Event detail is read-only.

### API Keys Management

17. **API Keys** tab lists all API keys for the authenticated end user.
18. API Key list shows:
    - Key Name — user-defined name
    - Key Value — masked (e.g., `sk_prod_****7Kx9`)
    - Last Used — timestamp or "Never"
    - Status — Active (green) or Expiring (amber)
    - Created At
19. **Create Key** button — opens modal to create a new API key.
20. **Create API Key Modal:**
    - Key Name input (required)
    - Expiration date (optional — for testing keys)
    - On submit: generates a new API key (shown once, never stored in plaintext)
21. **Copy** button on each key to copy the full key value to clipboard.
22. **Delete** button to revoke an API key (with confirmation).
23. **Status indicators:**
    - `active` — key is valid and can be used
    - `expiring` — key expires within 30 days
    - `expired` — key has passed expiration date
    - `revoked` — key was manually deleted

### Real-Time Updates

24. When "Live" is enabled, new events appear at the top of the table automatically.
25. WebSocket connection (`/ws/events`) pushes new events in real-time.
26. Stats bar updates in real-time when new events arrive.
27. If WebSocket connection fails, fall back to polling (every 10 seconds).

---

## Test Cases

### TC-01 — END_USER views own dashboard

**Given:** authenticated END_USER "John Smith"
**When:** navigating to `/my-usage`
**Then:** dashboard shows only John's metrics: tokens, requests, API keys
**And:** no other end users' data is visible

---

### TC-02 — View usage by model

**Given:** END_USER is on the Dashboard
**Then:** a grid shows usage broken down by AI model
**And:** each model shows token count and percentage bar

---

### TC-03 — View event log

**Given:** END_USER is on the My Events tab
**Then:** events table shows all of the end user's API calls
**And:** columns: Event ID, Timestamp, Model, Input, Output, Latency, Cost, Status

---

### TC-04 — Live toggle updates events

**Given:** END_USER has "Live" enabled
**When:** a new API call is made by this end user
**Then:** the new event appears at the top of the table within seconds
**And:** stats bar updates

---

### TC-05 — Pause live updates

**Given:** END_USER has "Live" enabled
**When:** clicking "Paused" to toggle live mode off
**Then:** no new events appear automatically
**And:** clicking "Live" resumes real-time updates

---

### TC-06 — Filter events by model

**Given:** END_USER has events from multiple models
**When:** selecting "GPT-4" from the model filter
**Then:** table shows only GPT-4 events
**And:** stats bar recalculates for filtered events only

---

### TC-07 — Expand event detail

**Given:** END_USER clicks on an event row
**Then:** the row expands to show full event details
**And:** JSON payload is displayed

---

### TC-08 — Pagination

**Given:** END_USER has more than 50 events
**Then:** pagination controls appear (1, 2, 3... or "Load More")
**And:** clicking "Next" loads the next page

---

### TC-09 — View API keys

**Given:** END_USER is on the API Keys tab
**Then:** all API keys for this end user are listed
**And:** each shows: name, masked key, last used, status

---

### TC-10 — Create new API key

**Given:** END_USER clicks "Create Key"
**When:** entering a key name and submitting
**Then:** a new API key is generated
**And:** the full key value is shown once (never shown again)

---

### TC-11 — Copy API key

**Given:** END_USER clicks "Copy" on an API key
**Then:** the full key value is copied to clipboard
**And:** a "Copied!" confirmation appears

---

### TC-12 — Delete API key

**Given:** END_USER clicks "Delete" on an API key
**When:** confirming the deletion
**Then:** the key is revoked immediately
**And:** the key can no longer be used for API calls

---

### TC-13 — END_USER cannot see other users' events

**Given:** END_USER "John" is authenticated
**When:** viewing the event log
**Then:** only John's events are shown
**And:** Jane's events are NOT visible

---

### TC-14 — END_USER cannot access admin portals

**Given:** actor role is `END_USER`
**When:** navigating to `/dashboard`, `/admin`, `/my-account`, or any admin URL
**Then:** 403 `FORBIDDEN`

---

### TC-15 — CUSTOMER can view their org's end user events

**Given:** CUSTOMER for org "TechCorp" is authenticated
**When:** navigating to Team Usage or selecting an end user
**Then:** all end user events for TechCorp are visible
**And:** end users from other organizations are NOT visible

---

### TC-16 — ORG_ADMIN can view end user events within their org

**Given:** ORG_ADMIN for org "TechCorp" is authenticated
**When:** navigating to the end user events section
**Then:** all end user events for TechCorp are visible
**And:** events from other organizations are NOT visible

---

### TC-17 — SUPER_ADMIN can view end user events across all orgs

**Given:** SUPER_ADMIN is authenticated
**When:** selecting any organization and viewing end user events
**Then:** all end user events for the selected org are visible
**And:** can switch between organizations to see their events

---

## API Endpoints

| Method | Path | Description | Auth | Scope |
|--------|------|-------------|------|-------|
| `GET` | `/api/v1/end-users/:endUserId/usage` | Get end user's usage summary | JWT · Guard: `EndUserGuard` | Own end user only |
| `GET` | `/api/v1/end-users/:endUserId/events` | Get event log | JWT · Guard: `EndUserGuard` or `CustomerGuard` or `OrgAdminGuard` or `SuperAdminGuard` | Role-dependent |
| `GET` | `/api/v1/end-users/:endUserId/events/:eventId` | Get single event detail | JWT · Guard: `EndUserGuard` or `OrgAdminGuard` or `SuperAdminGuard` | Role-dependent |
| `GET` | `/api/v1/end-users/:endUserId/api-keys` | List API keys | JWT · Guard: `EndUserGuard` | Own end user only |
| `POST` | `/api/v1/end-users/:endUserId/api-keys` | Create API key | JWT · Guard: `EndUserGuard` | Own end user only |
| `DELETE` | `/api/v1/api-keys/:keyId` | Revoke API key | JWT · Guard: `EndUserGuard` | Own end user only |
| `GET` | `/ws/end-users/:endUserId/events` | WebSocket for real-time events | JWT · Guard: `EndUserGuard` | Own end user only |
| `GET` | `/api/v1/customers/:customerId/end-users/:endUserId/events` | Get events (Customer role) | JWT · Guard: `CustomerGuard` | Own customer |
| `GET` | `/api/v1/organizations/:orgId/end-users/:endUserId/events` | Get events (OrgAdmin role) | JWT · Guard: `OrgAdminGuard` | Own org |
| `GET` | `/api/v1/platform/end-users/:endUserId/events` | Get events (SuperAdmin) | JWT · Guard: `SuperAdminGuard` | Platform-wide |
| `GET` | `/api/v1/customers/:customerId/end-users` | List all end users for customer | JWT · Guard: `CustomerGuard` | Own customer |
| `GET` | `/api/v1/organizations/:orgId/end-users` | List all end users for org | JWT · Guard: `OrgAdminGuard` | Own org |

---

## Data Tables Used

| Table | Schema | Operation | Key Columns |
|-------|--------|-----------|-------------|
| `usage_events` | `billing` | SELECT | `id, end_user_id, org_id, meter_id, input_tokens, output_tokens, cached_tokens, latency_ms, cost, model, status, event_timestamp, endpoint, payload` |
| `end_users` | `customer` | SELECT | `id, org_id, external_user_id, name, email` |
| `api_keys` | `auth` | SELECT · INSERT · DELETE | `id, end_user_id, name, key_hash, key_prefix, status, last_used_at, created_at, expires_at` |
| `organizations` | `identity` | SELECT | `id, name` |

---

## Event Schema

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique event identifier (e.g., `evt_7x9Kp2mN`) |
| `timestamp` | datetime | When the event occurred (ISO 8601) |
| `model` | string | AI model used (e.g., `gpt-4`, `claude-3-opus`) |
| `type` | string | API type (e.g., `chat.completion`, `embedding`) |
| `input_tokens` | integer | Number of input tokens consumed |
| `output_tokens` | integer | Number of output tokens generated |
| `cached_tokens` | integer | Number of cached tokens (prompt caching) |
| `latency_ms` | integer | Request latency in milliseconds |
| `cost` | decimal | Cost in USD |
| `status` | string | `success` or `error` |
| `endpoint` | string | API endpoint called (e.g., `/v1/chat/completions`) |
| `error_message` | string | Error description if status = `error` |

---

## API Key States

```
┌─────────┐   create   ┌─────────┐   expire   ┌──────────┐
│ (none)  │──────────►│  active │───────────►│ expired  │
└─────────┘           └────┬────┘            └──────────┘
                           │
                           │ revoke
                           ▼
                      ┌──────────┐
                      │ revoked  │
                      └──────────┘
```

---

## Error Codes

| Code | HTTP | Trigger |
|------|------|---------|
| `END_USER_NOT_FOUND` | 404 | `endUserId` does not exist |
| `EVENT_NOT_FOUND` | 404 | `eventId` does not exist |
| `API_KEY_NOT_FOUND` | 404 | `keyId` does not exist |
| `API_KEY_EXPIRED` | 401 | Using an expired API key |
| `API_KEY_REVOKED` | 401 | Using a revoked API key |
| `INVALID_API_KEY` | 401 | API key does not match any record |
| `FORBIDDEN` | 403 | `actor.end_user_id !== endUserId` (END_USER trying to access another user's data) |
| `FORBIDDEN` | 403 | `actor.customer_id !== params.customerId` (CUSTOMER accessing another customer's data) |
| `FORBIDDEN` | 403 | `actor.org_id !== params.orgId` (ORG_ADMIN accessing another org's data) |
| `UNAUTHORIZED` | 401 | No valid JWT token or API key |

---

## Environment Config Keys

| Key | Description |
|-----|-------------|
| `EVENT_RETENTION_DAYS` | Days to keep raw event data (default: `90`) |
| `EVENTS_PAGE_SIZE` | Number of events per page (default: `50`) |
| `LIVE_UPDATE_INTERVAL_MS` | Polling interval when WebSocket unavailable (default: `10000`) |
| `WEBSOCKET_ENABLED` | Enable WebSocket for real-time events (default: `true`) |
| `API_KEY_PREFIX_LENGTH` | Length of key prefix shown in UI (default: `8`) |
| `API_KEY_HASH_ALGO` | Hashing algorithm for storing keys (default: `sha256`) |
| `LATENCY_WARNING_THRESHOLD_MS` | Highlight latency above this threshold (default: `2000`) |
| `DATABASE_URL` | PostgreSQL connection string (Prisma) |
| `KEYCLOAK_URL` | Keycloak server base URL |
| `KEYCLOAK_REALM` | `quantumbilling` |

---

## UI Story

### End User Portal Layout

**Sidebar Navigation (left):**
```
┌─────────────────────────┐
│ John Smith              │  ← End User name + avatar
│ (End User Portal)        │
├─────────────────────────┤
│ 📊 Dashboard            │  ← Active
│ 📋 My Events            │
│ 🔑 API Keys            │
├─────────────────────────┤
│ 🚪 Logout               │
└─────────────────────────┘
```

**Header:**
- End User name
- "My Usage" / "My Events" / "My API Keys"
- Organization name (read-only, for context)

### Dashboard Page

**Summary Cards (3-up):**
| Metric | Value | Icon | Color |
|--------|-------|------|-------|
| My Token Usage | 45.2M | zap | amber |
| My Requests | 234K | activity | cyan |
| Active API Keys | 3 | key | purple |

**Usage by Model (4-up grid):**
```
┌──────────────────┐ ┌──────────────────┐
│ GPT-4             │ │ Claude 3 Opus     │
│ 28.4M tokens      │ │ 12.1M tokens      │
│ ████████░░░ 62%  │ │ ███░░░░░░░░ 27%  │
└──────────────────┘ └──────────────────┘
```

### My Events Page

**Header:**
- "My Event Log" title
- Live/Paused toggle button

**Stats Bar (4-up):**
| Metric | Value | Color |
|--------|-------|-------|
| Events | 12,847 | amber |
| Total Tokens | 156.8M | cyan |
| Total Cost | $1,247.50 | green |
| Error Rate | 0.3% | green (or red if >5%) |

**Model Filter:**
```
[All] [GPT-4] [GPT-3.5] [Claude 3] [Claude 3 Opus] [Gemini]
```

**Events Table:**
| Event ID | Timestamp | Model | Input | Output | Latency | Cost | Status |
|----------|-----------|-------|-------|--------|---------|------|--------|
| evt_7x9Kp | 2026-06-30 14:32:45.892 | GPT-4 | 1,247 | 856 | 1,243ms | $0.089 | success |
| evt_8y2Lq | 2026-06-30 14:32:41.234 | Claude 3 | 2,891 | 1,432 | 2,156ms | $0.185 | success |

**Expanded Event Detail:**
```
┌─ Event: evt_7x9Kp2mN ──────────────────────────────────────┐
│ Timestamp: 2026-06-30 14:32:45.892                          │
│ Model: GPT-4                                                 │
│ Input Tokens: 1,247                                          │
│ Output Tokens: 856                                            │
│ Cached Tokens: 0                                              │
│ Latency: 1,243ms                                              │
│ Cost: $0.089                                                  │
│ Status: success                                               │
│ Endpoint: /v1/chat/completions                                 │
│                                                               │
│ Payload:                                                      │
│ {                                                             │
│   "input_tokens": 1247,                                       │
│   "output_tokens": 856,                                       │
│   "cached_tokens": 0,                                         │
│   "latency_ms": 1243,                                         │
│   "status": "success"                                         │
│ }                                                             │
└───────────────────────────────────────────────────────────────┘
```

### API Keys Page

**Header:**
- "My API Keys" title
- "+ Create Key" button

**API Key Cards:**
```
┌─────────────────────────────────────────────────────────────┐
│ 🔑 Production Key                                           │
│ sk_prod_****7Kx9                    [Copy] [Delete]        │
│ Last used: 2 min ago                                        │
│ Created: 2024-10-15                         Status: Active │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ 🔑 Development Key                                          │
│ sk_dev_****3Nm2                     [Copy] [Delete]        │
│ Last used: 1 hour ago                                       │
│ Created: 2024-11-20                        Status: Expiring │
└─────────────────────────────────────────────────────────────┘
```

**Create Key Modal:**
- Key Name: input
- Expiration: date picker (optional)
- [Cancel] [Create Key]

---

## Webhooks — Not Applicable

End User Events is a read-only portal. No webhooks are triggered by viewing events.

---

## Dependencies & Notes for Agent

- **Authentication:** End Users authenticate via JWT (from Keycloak) or API Key. The JWT/API key contains `end_user_id`.
- **Data Isolation:** ALL queries MUST filter by `end_user_id = actor.end_user_id`. This is enforced at the guard layer.
- **Event Recording:** Events are recorded by the Real-time API when end users make API calls. Each event includes `end_user_id`.
- **Real-Time Updates:** Use WebSocket (`/ws/end-users/:endUserId/events`) for live updates. Fall back to polling if WebSocket fails.
- **API Key Security:** API keys are stored as hashes (never plaintext). The full key is shown only once at creation.
- **Latency Highlighting:** If `latency_ms > LATENCY_WARNING_THRESHOLD_MS`, display in amber/red to indicate slowness.
- **Pagination:** Use cursor-based or offset pagination for events. Default 50 per page.
- **End User Context:** The End User Portal shows data for ONE end user. It does NOT show org-level or other end users' data.
- **No Admin Access:** End Users cannot access any admin portals. Redirect to End User Portal or show 403.
- **Audit Logging:** Log API key creation and revocation for security auditing.

---

## Future Enhancements (Out of Scope for v1)

- Export events as CSV/JSON
- Event detail drill-down (view full request/response)
- Anomaly detection alerts (unusual spike in errors or latency)
- Cost预算 alerts per end user
- Batch event download
- Compare usage across time periods
- Team leaderboard (if ORG_ADMIN shares results)
