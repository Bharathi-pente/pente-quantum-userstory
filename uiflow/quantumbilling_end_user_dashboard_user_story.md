# QuantumBilling User Story: End User Dashboard

**QB-STORY-028** · Sprint 3 · Phase: UI Feature

---

## Title

**End User Dashboard** — personal usage dashboard for individual API consumers

---

## Badges

| Backend | UI | Auth / RBAC | Billing Engine | Priority |
|---------|----|-------------|----------------|----------|
| Backend | UI | Auth / RBAC | Billing Engine | P1 |

---

## Description

**As an END_USER**, I want to access a personal dashboard that shows my usage metrics, token consumption by AI model, and the status of my API keys, so that I can quickly understand how much I've used the API, which models I'm using most, and whether my API keys are active.

The End User Dashboard is the **landing page** for authenticated End Users. It provides a high-level overview of personal API usage and authentication status.

**Note:** The End User Dashboard is different from the "My Events" page — it shows **aggregate metrics** (totals, breakdowns), while "My Events" shows individual API call logs.

---

## RBAC Roles

| Role | Can access End User Dashboard | Scope |
|------|------------------------------|-------|
| **END_USER** | ✅ Yes | Own data only |
| **CUSTOMER** | ❌ No | Different portal (Customer Portal) |
| **ORG_ADMIN** | ❌ No | Different portal (Organization Dashboard) |
| **SUPER_ADMIN** | ❌ No | Different portal (Platform Admin) |

---

## End User Portal Navigation

| Nav Item | Description |
|----------|-------------|
| Dashboard | Landing page — usage overview and API key status |
| My Events | Detailed event log of individual API calls |
| API Keys | Manage authentication keys |

---

## Acceptance Criteria

### Dashboard Landing Page

1. Dashboard is accessible at `/my-usage` or `/dashboard` for authenticated END_USER.
2. Header shows:
   - "My Dashboard" title
   - Current billing period (e.g., "Jun 1 – Jun 30, 2026")
   - Organization name (read-only, for context)
3. All data shown is **strictly for the authenticated end user only**.

### Summary Cards (3-up or 4-up)

4. **My Token Usage** — total tokens consumed (input + output + cached)
   - Displayed as human-readable: e.g., "45.2M" or "1.2B"
   - Icon: zap
   - Color: amber (#F59E0B)

5. **My Requests** — total number of API calls made
   - Displayed with abbreviation: e.g., "234K" or "1.5M"
   - Icon: activity
   - Color: cyan (#00D9FF)

6. **Active API Keys** — count of API keys with status "active" or "expiring"
   - Icon: key
   - Color: purple (#A855F7)
   - Clicking navigates to API Keys page

7. **My Cost** (optional) — estimated cost of usage
   - Calculated from tokens × applicable rates
   - Icon: dollar
   - Color: green (#22C55E)

### Usage by Model Section

8. **Usage by Model** section displays a grid of model cards.
9. Each model card shows:
   - **Model Name** (e.g., "GPT-4", "Claude 3 Opus", "Gemini Pro")
   - **Token Count** for that model (formatted: e.g., "28.4M")
   - **Percentage Bar** showing relative usage compared to other models
   - **Percentage Label** (e.g., "62%")
10. Model cards are ordered by usage (highest first).
11. Models with 0 usage are hidden.
12. Clicking a model card navigates to "My Events" filtered by that model.

### Organization Context

13. A small "context bar" or badge shows the organization name (e.g., "TechCorp AI").
14. This is **read-only** — end users cannot change organizations.

### Quick Actions

15. **Quick Action: Create New API Key**
    - "Create Key" button or shortcut on dashboard
    - Navigates to API Keys page with creation modal

16. **Quick Action: View My Events**
    - "View Event Log" link
    - Navigates to My Events page

### Recent Activity (Optional)

17. **Recent Activity** section shows the last 3-5 events as a mini-preview.
18. Each preview row shows:
    - Timestamp
    - Model (icon/badge)
    - Token count
    - Cost
19. "View All" link navigates to full My Events page.

---

## Test Cases

### TC-01 — END_USER logs into dashboard

**Given:** authenticated END_USER "John Smith"
**When:** navigating to `/my-usage`
**Then:** dashboard shows John's metrics only
**And:** no other end users' data is visible

---

### TC-02 — View token usage summary

**Given:** END_USER is on the dashboard
**Then:** "My Token Usage" card shows total tokens
**And:** "My Requests" card shows total API calls
**And:** both are scoped to the current billing period

---

### TC-03 — View usage by model

**Given:** END_USER has used multiple AI models
**Then:** "Usage by Model" grid shows each model
**And:** each model shows token count and percentage bar
**And:** models are sorted by highest usage first

---

### TC-04 — Click model card navigates to filtered events

**Given:** END_USER clicks on "GPT-4" model card
**Then:** navigates to "My Events" page
**And:** events are pre-filtered to GPT-4 only

---

### TC-05 — View active API keys count

**Given:** END_USER has 3 active API keys
**Then:** "Active API Keys" card shows "3"
**And:** clicking the card navigates to API Keys page

---

### TC-06 — END_USER cannot see other users' data

**Given:** END_USER "John" is authenticated
**When:** viewing the dashboard
**Then:** only John's metrics are shown
**And:** no other end users' data is visible

---

### TC-07 — Organization context is shown

**Given:** END_USER belongs to "TechCorp AI"
**Then:** dashboard shows "TechCorp AI" as the organization context
**And:** end user cannot change this

---

### TC-08 — Quick action to create API key

**Given:** END_USER is on the dashboard
**When:** clicking "Create Key" or similar quick action
**Then:** navigates to API Keys page
**And:** create key modal opens

---

### TC-09 — Recent activity preview

**Given:** END_USER has recent API calls
**Then:** "Recent Activity" section shows last 3-5 events
**And:** "View All" links to full My Events page

---

## API Endpoints

| Method | Path | Description | Auth |
|--------|------|-------------|------|
| `GET` | `/api/v1/end-users/:endUserId/usage` | Get end user's usage summary (tokens, requests, cost) | JWT · Guard: `EndUserGuard` |
| `GET` | `/api/v1/end-users/:endUserId/usage/by-model` | Get usage breakdown by AI model | JWT · Guard: `EndUserGuard` |
| `GET` | `/api/v1/end-users/:endUserId/api-keys` | List API keys with status | JWT · Guard: `EndUserGuard` |
| `GET` | `/api/v1/end-users/:endUserId/events/recent` | Get last N events for preview | JWT · Guard: `EndUserGuard` |

---

## Data Tables Used

| Table | Schema | Operation | Key Columns |
|-------|--------|-----------|-------------|
| `usage_events` | `billing` | SELECT · SUM | `end_user_id, input_tokens, output_tokens, cached_tokens, cost, model, event_timestamp` |
| `end_users` | `customer` | SELECT | `id, org_id, name, email` |
| `api_keys` | `auth` | SELECT | `end_user_id, status, created_at` |
| `organizations` | `identity` | SELECT | `id, name` |

---

## Usage Calculation

```
My Token Usage = SUM(input_tokens) + SUM(output_tokens) + SUM(cached_tokens)
My Requests = COUNT(*)
My Cost = SUM(input_tokens × input_rate) + SUM(output_tokens × output_rate)
```

---

## UI Story

### End User Portal Layout

**Sidebar Navigation (left):**
```
┌─────────────────────────┐
│ John Smith              │  ← End User name + avatar
│ TechCorp AI             │  ← Organization (read-only)
├─────────────────────────┤
│ 📊 Dashboard            │  ← Active (landing)
│ 📋 My Events           │
│ 🔑 API Keys            │
├─────────────────────────┤
│ 🚪 Logout               │
└─────────────────────────┘
```

**Header:**
- End User name
- Current billing period
- Organization name badge

### Dashboard Page

**Summary Cards (4-up):**
| Metric | Value | Icon | Color |
|--------|-------|------|-------|
| My Token Usage | 45.2M | zap | amber |
| My Requests | 234K | activity | cyan |
| Active API Keys | 3 | key | purple |
| My Cost | $1,247.50 | dollar | green |

**Usage by Model (4-up grid):**
```
┌────────────────────────┐ ┌────────────────────────┐
│ GPT-4                   │ │ Claude 3 Opus           │
│ 28.4M tokens           │ │ 12.1M tokens           │
│ ████████████░░░░ 62%  │ │ ██████░░░░░░░░░ 27%   │
│ Click to view →        │ │ Click to view →        │
└────────────────────────┘ └────────────────────────┘

┌────────────────────────┐ ┌────────────────────────┐
│ Gemini Pro              │ │ GPT-3.5 Turbo          │
│ 3.2M tokens            │ │ 1.5M tokens            │
│ ███░░░░░░░░░░░░ 7%    │ │ █░░░░░░░░░░░░░░ 3%   │
│ Click to view →        │ │ Click to view →        │
└────────────────────────┘ └────────────────────────┘
```

**Quick Actions:**
- [+ Create API Key] — button
- [View Event Log →] — link

**Recent Activity:**
```
┌────────────────────────────────────────────────────────┐
│ Recent Activity                          [View All →]  │
├────────────────────────────────────────────────────────┤
│ GPT-4 · 1,247 tokens · $0.089 · 2 min ago          │
│ Claude 3 · 2,891 tokens · $0.185 · 5 min ago        │
│ GPT-4 · 892 tokens · $0.016 · 12 min ago            │
└────────────────────────────────────────────────────────┘
```

---

## Dependencies & Notes for Agent

- **Data Isolation:** ALL queries MUST filter by `end_user_id = actor.end_user_id`. This is strictly enforced.
- **Organization Context:** End Users belong to one Organization. Display the org name for context but do not allow changes.
- **Usage Period:** Metrics are for the current billing period by default (or last 30 days for new users without subscription).
- **Real-Time Updates:** API key status should refresh periodically. Use polling or WebSocket for live status updates.
- **Model Colors:** Use consistent model colors across the dashboard (e.g., GPT-4 = cyan, Claude = purple, Gemini = amber).
- **No Cross-User Access:** Even if an End User guesses another user's ID, the guard layer blocks access.
- **Billing Period:** Display the billing period so users understand the time window for metrics.
- **Large Number Formatting:** Format tokens as "45.2M" or "1.2B" — do not show raw integers.
- **Cost Estimate:** If real-time cost calculation is expensive, cache or calculate periodically rather than on every page load.

---

## Future Enhancements (Out of Scope for v1)

- Cost budget alerts (notify when approaching budget)
- Usage comparison vs. last period
- Model recommendation (suggest optimal model for use case)
- Export dashboard as PDF
- Dark mode / theme customization
- Mobile-responsive dashboard
- Usage goals and achievements
- Team comparison (how I rank vs. team average)
