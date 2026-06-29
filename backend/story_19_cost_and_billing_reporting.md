# Story 19 — Cost & Billing Reporting

> **Phase:** 4 — Aggregation & Analytics Reporting APIs
> **Depends on:** Phase 4 Overview (ClickHouse `usage_events_dedup_v` view must exist from Phase 1)
> **Blocks:** Nothing (Completes Phase 4)

---

## Description

As a **billing manager or finance administrator**, I need clear summaries of financial expenditures associated with API usage across my organization and tenants, bucketed by currency, provider, or date range, so that I can manage budget tracking, generate customer invoices, and prevent cost overruns.

This story implements three financial reporting endpoints:
*   `GET /v1/orgs/{org_id}/costs`: Detailed cost reports for an organization, grouped by breakdowns like `tenant` or `model`.
*   `GET /v1/tenants/{tenant_id}/costs`: Cost reports for a single tenant, optionally filtered by user or model.
*   `GET /v1/analytics/costs/daily`: Daily cost trends for billing charts over a selected date range.

---

## Acceptance Criteria

### Input Parameter Validation

| # | Criterion | Details / Edge Cases |
|---|---|---|
| 1 | Standardize validation of input path parameter variables: `org_id` and `tenant_id` must be non-empty strings. | Invalid inputs return `400 BAD_REQUEST`. |
| 2 | Exposes optional query parameter `breakdown_by` for organization costs. | Supported values: `"tenant"`, `"model"`, `"user"`, `"service"`. Invalid parameters return `400` with code `INVALID_BREAKDOWN_FILTER`. |
| 3 | Exposes optional date parameters: `start_date` and `end_date` (format: `YYYY-MM-DD`). | Malformed formats return `400` with code `INVALID_DATE_FORMAT`. |
| 4 | Date range validation checks: `start_date` must be before or equal to `end_date`. | Invalid ranges return `400` with code `INVALID_DATE_RANGE`. |

### ClickHouse Financial Aggregations

| # | Criterion | Details / Edge Cases |
|---|---|---|
| 5 | All SQL queries executing against ClickHouse must query `events.usage_events_dedup_v` and run under a context execution timeout limit of 5 seconds. | High-performance columnar scan. |
| 6 | Cost values must be computed by summing decimal representation values in ClickHouse: `sum(toDecimal128(cost, 6))` or converting cost string fields to float64/decimal. | Protect billing sums from binary-float precision loss. |
| 7 | Multi-tenant isolation middleware check: Organization billing managers must belong to the organization passed in `org_id`. | Accessing billing records of a different org returns `403 FORBIDDEN` with code `UNAUTHORIZED_ACCESS`. |
| 8 | **Zero-Fill Trends**: The daily cost trend endpoint must fill in any date gaps with zero costs (`"0.000000"`) for dates in the range with no activity, preserving date sorting order. | Prepares clean charting data for rendering. |
| 9 | Default response currency is assumed to be `"USD"`. Field value must be explicitly returned in the JSON payload (e.g. `currency: "USD"`). | Prepares for future multi-currency support. |
| 10 | For empty query results, return `200 OK` with zeroed summaries and empty lists (e.g. `total_cost: "0.000000"`, `breakdowns: []`), never `404` or `null`. | Avoid frontend mapping errors. |

---

## Test Cases

### TC-01: Get Organization Costs with Breakdown by Tenant
* **Given**: ClickHouse contains event logs for tenants `tenant_1` ($50.50) and `tenant_2` ($25.25) under `org_acme`.
* **When**: Calling `GET /v1/orgs/org_acme/costs?breakdown_by=tenant`
* **Then**: Returns `200 OK` with payload:
  ```json
  {
    "org_id": "org_acme",
    "currency": "USD",
    "total_cost": "75.750000",
    "breakdowns": [
      {
        "group_value": "tenant_1",
        "cost": "50.500000",
        "percentage": 66.67
      },
      {
        "group_value": "tenant_2",
        "cost": "25.250000",
        "percentage": 33.33
      }
    ]
  }
  ```

### TC-02: Get Daily Cost Trend - Happy Path
* **Given**: ClickHouse contains event costs for `org_acme` on `2026-06-25` ($10.00) and `2026-06-27` ($5.00).
* **When**: Calling `GET /v1/analytics/costs/daily?org_id=org_acme&start_date=2026-06-25&end_date=2026-06-27`
* **Then**: Returns `200 OK` with exactly 3 cost points (including `2026-06-26` zero-filled):
  ```json
  [
    {
      "date": "2026-06-25",
      "cost": "10.000000",
      "currency": "USD"
    },
    {
      "date": "2026-06-26",
      "cost": "0.000000",
      "currency": "USD"
    },
    {
      "date": "2026-06-27",
      "cost": "5.000000",
      "currency": "USD"
    }
  ]
  ```

### TC-03: Cost Request with Unsupported Breakdown Filter
* **When**: Calling `GET /v1/orgs/org_acme/costs?breakdown_by=ip_address`
* **Then**: Returns `400 BAD_REQUEST` with error code `INVALID_BREAKDOWN_FILTER`.

### TC-04: Tenant Costs - Org Isolation Check
* **Given**: Client is authenticated for `org_acme`.
* **When**: Calling `GET /v1/tenants/tenant_under_org_different/costs`
* **Then**: Returns `403 FORBIDDEN` with error code `UNAUTHORIZED_ACCESS`.

### TC-05: Empty Database Cost State
* **Given**: ClickHouse has no records matching `org_acme`.
* **When**: Calling `GET /v1/orgs/org_acme/costs`
* **Then**: Returns `200 OK` with `"total_cost": "0.000000"` and empty breakdown arrays.

---

## Data Tables / Resources Used

| Resource | Operation | Purpose |
|---|---|---|
| `events.usage_events_dedup_v` (ClickHouse) | `SELECT` (Aggregated) | Aggregates and sums token cost values grouped by filter dimensions |
