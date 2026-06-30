# QuantumBill — Billing Model Documentation

> **Project**: QuantumBill - Usage-Based Billing Platform
> **Version**: 1.0
> **Status**: Draft
> **Last Updated**: 2026-06-30

---

## Table of Contents

1. [Overview](#1-overview)
2. [Entity Hierarchy](#2-entity-hierarchy)
3. [Billing Models](#3-billing-models)
4. [Entity Creation Flows](#4-entity-creation-flows)
5. [Usage Event Lifecycle](#5-usage-event-lifecycle)
6. [Invoice Generation Workflow](#6-invoice-generation-workflow)
7. [Payment & Credits](#7-payment--credits)
8. [Database Schema Mapping](#8-database-schema-mapping)
9. [Appendix: State Machines](#9-appendix-state-machines)

---

## 1. Overview

### Purpose

This document describes the billing model for QuantumBill — a multi-tenant, usage-based billing platform that supports both subscription and pay-as-you-go pricing models.

### Key Billing Concepts

| Term | Definition |
|------|------------|
| **Billing Model** | Defines how customers pay: Subscription, Pay-as-you-go, or Hybrid |
| **Billing Period** | The recurring time window for invoice generation (monthly, quarterly, annual) |
| **Meter** | A definition of what is being measured (e.g., API calls, tokens) |
| **Usage Event** | A single record of consumption for a meter |
| **Rate Card** | A versioned collection of pricing for specific meters |
| **Included Units** | Bundled usage allowance with each subscription |
| **Overage** | Usage beyond included units, billed at per-unit rates |
| **Invoice** | The billing document generated for a billing period |
| **Credits** | Prepaid or promotional balance applied to invoices |
| **FEFO** | First Expiring, First Out — credit consumption priority |

---

## 2. Entity Hierarchy

### Billing Entity Hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│                         BILLING ENTITY HIERARCHY                     │
└─────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────┐
  │  SUPER_ADMIN (Platform Operator)                                │
  │  └── Creates Organizations                                      │
  └─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Organization (identity.organizations)                          │
  │  ├── Has: Payment Methods, Credits, Tax Profiles               │
  │  ├── billing_email, currency, country, timezone                │
  │  └── Status: ACTIVE / SUSPENDED / DELETED                      │
  └─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  Customer (customer.customers)                                  │
  │  ├── Belongs to one Organization                                │
  │  ├── Has: Subscriptions, End Users, Credit Balance             │
  │  ├── Linked to Product (optional)                               │
  │  └── Status: ACTIVE / SUSPENDED / CHURNED                       │
  └─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  End User (customer.end_users)                                  │
  │  ├── Belongs to one Customer                                    │
  │  ├── Has: API Keys, Individual Usage Events                    │
  │  └── Makes API calls that generate billing events               │
  └─────────────────────────────────────────────────────────────────┘
```

### Key Relationships

| Relationship | From | To | Via |
|--------------|------|----|-----|
| Org → Customer | `customer.customers.org_id` | `identity.organizations.id` | FK |
| Customer → End User | `customer.end_users.customer_id` | `customer.customers.id` | FK |
| Subscription → Customer | `customer.subscriptions.customer_id` | `customer.customers.id` | FK |
| Subscription → Product | `customer.subscriptions.product_id` | `catalog.products.id` | FK |
| Invoice → Customer | `billing.invoices.customer_id` | `customer.customers.id` | FK |
| Invoice → Organization | `billing.invoices.org_id` | `identity.organizations.id` | FK |
| Payment → Invoice | `billing.payments.invoice_id` | `billing.invoices.id` | FK |
| Credit → Organization | `billing.credits.org_id` | `identity.organizations.id` | FK |
| Usage Event → End User | `usage.raw_events.end_user_id` | `customer.end_users.id` | FK |

---

## 3. Billing Models

### 3.1 Subscription-Based Billing

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SUBSCRIPTION-BASED BILLING MODEL                   │
└─────────────────────────────────────────────────────────────────────┘

  Product/Plan Configuration:
  ┌─────────────────────────────────────────────────────────────┐
  │  Product: "Pro Plan"                                         │
  │  ├── Base Price: $99/month                                    │
  │  ├── Billing Period: Monthly                                 │
  │  ├── Included Units:                                         │
  │  │   ├── API Calls: 10,000                                   │
  │  │   ├── Input Tokens: 1,000,000                             │
  │  │   └── Output Tokens: 1,000,000                           │
  │  └── Overage Rates:                                          │
  │        ├── API Calls: $0.001/call                            │
  │        └── Tokens: $0.00003/input, $0.00006/output          │
  └─────────────────────────────────────────────────────────────┘

  Billing Cycle:
  1. Billing period begins (e.g., June 1)
  2. Base fee ($99) is charged on invoice
  3. Usage is tracked against included units
  4. If usage > included → calculate overage charges
  5. Apply credits (if any)
  6. Generate invoice for period
```

### 3.2 Pay-as-you-Go Billing

```
┌─────────────────────────────────────────────────────────────────────┐
│                      PAY-AS-YOU-GO BILLING MODEL                     │
└─────────────────────────────────────────────────────────────────────┘

  No base fee, no included units
  All usage is charged at per-unit rates from Rate Card

  Rate Card Configuration:
  ┌─────────────────────────────────────────────────────────────┐
  │  Rate Card: "Standard Usage Rates"                           │
  │  ├── GPT-4 Input Tokens: $0.00003/token                     │
  │  ├── GPT-4 Output Tokens: $0.00006/token                    │
  │  ├── GPT-3.5 Input Tokens: $0.00001/token                   │
  │  ├── GPT-3.5 Output Tokens: $0.00002/token                  │
  │  ├── API Calls: $0.001/call                                  │
  │  └── Storage: $0.10/GB                                       │
  └─────────────────────────────────────────────────────────────┘

  Billing Cycle:
  1. Customer uses service (tokens, API calls, etc.)
  2. Usage events are recorded
  3. At billing period end:
     - Aggregate all usage
     - Multiply by rate card prices
     - Generate invoice
```

### 3.3 Hybrid Billing (Subscription + Overage)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HYBRID BILLING MODEL                          │
└─────────────────────────────────────────────────────────────────────┘

  Combines subscription base with pay-as-you-go overage

  Example:
  ┌─────────────────────────────────────────────────────────────┐
  │  Pro Plan + Overage                                         │
  │  ├── Base Fee: $99/month (includes 10K API calls)           │
  │  ├── Overage: $0.001/call beyond 10K                        │
  │  └── Overage: $0.00003/token beyond included tokens        │
  └─────────────────────────────────────────────────────────────┘

  Usage Example: 15,000 API calls
  ┌─────────────────────────────────────────────────────────────┐
  │  Base Fee:      $99.00                                      │
  │  Overage:       5,000 × $0.001 = $5.00                     │
  │  ─────────────────────────────────                          │
  │  Total:         $104.00                                     │
  └─────────────────────────────────────────────────────────────┘
```

### 3.4 Comparison Table

| Aspect | Subscription | Pay-as-you-go | Hybrid |
|--------|-------------|---------------|--------|
| Base Fee | Yes | No | Yes |
| Included Units | Yes | No | Yes |
| Overage Calculation | Usage - included | N/A | Usage - included |
| Rate Card Application | Only for overage | All usage | Only for overage |
| Invoice Line Items | Base fee + overage + credits | Usage charges + credits | Base fee + overage + credits |
| Proration | Yes | No | Yes |
| Minimum Commitment | Usually annual | None | Annual commit |

---

## 4. Entity Creation Flows

### Flow 1: Super Admin Creates Organization

```
┌─────────────────────────────────────────────────────────────────────┐
│              FLOW 1: SUPER_ADMIN CREATES ORGANIZATION                │
└─────────────────────────────────────────────────────────────────────┘

  Actor: SUPER_ADMIN
  UI Path: /admin/orgs or /platform/organizations
  Portal: Platform Admin Portal

  Steps:
  ┌─────────────────────────────────────────────────────────────────┐
  │  1. Navigate to Organizations section                           │
  │  2. Click "Create Organization" button                         │
  │  3. Fill required form fields:                                  │
  │     ├── Organization Name (required)                            │
  │     ├── Billing Email (required)                                │
  │     ├── Currency (required, default: USD)                      │
  │     ├── Timezone (required)                                     │
  │     ├── Industry (optional)                                     │
  │     └── Country (required)                                     │
  │  4. Click "Create"                                              │
  │  5. System generates org_id (UUID)                              │
  │  6. Organization created with status: ACTIVE                    │
  │  7. Super Admin can now:                                        │
  │     ├── Invite ORG_ADMIN users                                  │
  │     ├── Create/assign products                                 │
  │     └── Configure rate cards                                   │
  └─────────────────────────────────────────────────────────────────┘

  Output: identity.organizations record created

  Database Table: identity.organizations
  ┌─────────────────────────────────────────────────────────────────┐
  │  id              | uuid (PK)                                     │
  │  name            | "Acme Corp"                                  │
  │  slug            | "acme-corp" (unique)                          │
  │  billing_email   | "billing@acme.com"                           │
  │  currency        | "USD"                                        │
  │  timezone        | "America/New_York"                           │
  │  status          | ACTIVE                                       │
  │  created_at      | timestamp                                    │
  │  updated_at      | timestamp                                    │
  └─────────────────────────────────────────────────────────────────┘
```

### Flow 2: Organization Creates Customer

```
┌─────────────────────────────────────────────────────────────────────┐
│               FLOW 2: ORG_ADMIN CREATES CUSTOMER                      │
└─────────────────────────────────────────────────────────────────────┘

  Actor: ORG_ADMIN
  UI Path: /organizations/:orgId/customers or /dashboard/customers
  Portal: Organization Dashboard

  Steps:
  ┌─────────────────────────────────────────────────────────────────┐
  │  1. Navigate to Customers section                              │
  │  2. Click "Add Customer" button                                 │
  │  3. Fill required form fields:                                  │
  │     ├── Customer Name (required)                                │
  │     ├── Customer Email (required)                               │
  │     ├── Product (optional - links to catalog.products)          │
  │     └── Contract (optional - links to rate card)                │
  │  4. Click "Create"                                              │
  │  5. System generates customer_id (UUID)                          │
  │  6. Customer created with defaults:                             │
  │     ├── status: ACTIVE                                          │
  │     ├── credit_balance: 0                                       │
  │     └── health_score: 100                                       │
  │  7. Optional: Auto-provision subscription if product selected   │
  └─────────────────────────────────────────────────────────────────┘

  Output: customer.customers record created

  Database Table: customer.customers
  ┌─────────────────────────────────────────────────────────────────┐
  │  id              | uuid (PK)                                     │
  │  org_id          | uuid (FK → identity.organizations)            │
  │  name            | "Enterprise Client A"                        │
  │  email           | "client@enterprise.com"                       │
  │  product_id      | uuid (FK → catalog.products, nullable)       │
  │  credit_balance  | decimal(19,4) default 0                      │
  │  health_score    | integer (0-100) default 100                  │
  │  status          | ACTIVE                                       │
  │  created_at      | timestamp                                     │
  │  updated_at      | timestamp                                     │
  └─────────────────────────────────────────────────────────────────┘
```

### Flow 3: Customer Creates End User

```
┌─────────────────────────────────────────────────────────────────────┐
│              FLOW 3: CUSTOMER CREATES END USER                       │
└─────────────────────────────────────────────────────────────────────┘

  Actor: ORG_ADMIN or CUSTOMER (depending on configuration)
  UI Path: /organizations/:orgId/customers/:customerId/end-users
  Portal: Organization Dashboard or Customer Portal

  Steps:
  ┌─────────────────────────────────────────────────────────────────┐
  │  1. Navigate to Customer detail page                            │
  │  2. Click "Add End User" or "Create End User"                   │
  │  3. Fill required form fields:                                  │
  │     ├── Name (required)                                         │
  │     ├── Email (required)                                        │
  │     └── External User ID (optional — maps to your system)        │
  │  4. Click "Create"                                               │
  │  5. System generates end_user_id (UUID)                          │
  │  6. End User created with:                                       │
  │     ├── customer_id (linked to parent customer)                  │
  │     ├── org_id (inherited from customer)                        │
  │     └── status: ACTIVE                                          │
  │  7. Optional: Generate API key for end user immediately          │
  └─────────────────────────────────────────────────────────────────┘

  Output: customer.end_users record created

  Database Table: customer.end_users
  ┌─────────────────────────────────────────────────────────────────┐
  │  id                  | uuid (PK)                                │
  │  customer_id         | uuid (FK → customer.customers)           │
  │  org_id              | uuid (FK → identity.organizations)       │
  │  external_user_id    | string (optional, from customer system)  │
  │  name                | "John Doe"                               │
  │  email               | "john@client.com"                        │
  │  status              | ACTIVE                                   │
  │  created_at          | timestamp                                │
  │  updated_at          | timestamp                                │
  └─────────────────────────────────────────────────────────────────┘

  Note: End Users can also be created via:
  - Self-registration (if enabled via feature flag)
  - Bulk import via CSV
```

### Flow 4: End User Usage Triggers Billing

```
┌─────────────────────────────────────────────────────────────────────┐
│              FLOW 4: END USER USAGE → BILLING EVENT                  │
└─────────────────────────────────────────────────────────────────────┘

  This flow shows how End User activity becomes a billable event

  Steps:
  ┌─────────────────────────────────────────────────────────────────┐
  │  1. End User makes API call with virtual API key                 │
  │     ├── Key validates against Redis cache                       │
  │     └── Anti-spoofing: org_id/tenant_id overridden from key     │
  │  2. Real-time validation:                                        │
  │     ├── Organization status = ACTIVE?                           │
  │     ├── Customer status = ACTIVE?                                │
  │     ├── Subscription status = ACTIVE (not paused)?              │
  │     ├── Entitlements/Quota remaining?                            │
  │     └── Rate limiting (budget_limit_usd, rate_limit_rpm)        │
  │  3. Usage event published to Kafka (usage-events topic)          │
  │  4. Analytics Worker consumes and writes to ClickHouse           │
  │  5. Aggregation job syncs to metering.usage_ledger              │
  │  6. At billing period end → Invoice Generation                  │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 5. Usage Event Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                      USAGE EVENT LIFECYCLE                           │
└─────────────────────────────────────────────────────────────────────┘

  [End User]
       │
       │ Makes API call with virtual API key
       ▼
  ┌─────────────────────────────────────────────┐
  │         Real-Time Validation                │
  │  ├── Organization validation (active?)      │
  │  ├── Customer validation (active?)          │
  │  ├── Subscription validation (active?)        │
  │  ├── Entitlement validation (quota left?)    │
  │  └── Budget check (budget_limit_usd)        │
  └─────────────────┬───────────────────────────┘
                    │ (if valid)
                    ▼
  ┌─────────────────────────────────────────────┐
  │           Kafka Event Stream                │
  │  Topic: usage-events (32 partitions)        │
  │  Partition key: org_id                       │
  │  Fields: org_id, customer_id, end_user_id,  │
  │          meter_id, quantity, timestamp,     │
  │          properties (model, service, etc.)  │
  └─────────────────┬───────────────────────────┘
                    │
                    ▼
  ┌─────────────────────────────────────────────┐
  │         ClickHouse - Raw Events             │
  │  Table: events.usage_events                  │
  │  Engine: ReplacingMergeTree(ingested_at)    │
  │  Deduplication view: usage_events_dedup_v   │
  └─────────────────┬───────────────────────────┘
                    │
                    ▼
  ┌─────────────────────────────────────────────┐
  │           Aggregation Job                   │
  │  └── Tokens summed per meter/customer/day   │
  │  └── Synced to metering.usage_ledger        │
  └─────────────────┬───────────────────────────┘
                    │
                    ▼
  ┌─────────────────────────────────────────────┐
  │        Billing & Rating Engine              │
  │  ├── Retrieve aggregated usage              │
  │  ├── Apply rate cards (per meter pricing)    │
  │  ├── Apply contract discounts               │
  │  ├── Apply included units (subscription)    │
  │  ├── Calculate overage                      │
  │  └── Apply credits                          │
  └─────────────────┬───────────────────────────┘
                    │
                    ▼
  ┌─────────────────────────────────────────────┐
  │        Invoice Generation                    │
  │  └── Generate billable charges              │
  └─────────────────────────────────────────────┘
```

### Usage Calculation by Billing Model

```
┌─────────────────────────────────────────────────────────────────────┐
│              USAGE CALCULATION BY BILLING MODEL                       │
└─────────────────────────────────────────────────────────────────────┘

  SUBSCRIPTION MODEL:
  ───────────────────
  1. Get subscription's product → find included_units
  2. For each meter with usage:
     a. Calculate total usage
     b. Subtract included_units[meter_name]
     c. If usage > included: overage = usage - included
     d. charge = overage × overage_rate
  3. Add base_fee from product
  4. Sum all charges

  PAY-AS-YOU-GO MODEL:
  ────────────────────
  1. For each meter with usage:
     a. Get rate from active rate card
     b. charge = usage × rate
  2. No base fee
  3. Sum all charges

  ─────────────────────────────────────────────────────────────────

  Example - Subscription:
  ─────────────────────────────────────────────────────────────────
  Product: Pro Plan ($99/month)
  Included: 10,000 API calls
  Overage Rate: $0.001/call

  Usage: 15,000 API calls
  Overage: 5,000 API calls

  Base Fee:      $99.00
  Overage:       5,000 × $0.001 = $5.00
  Total:         $104.00

  ─────────────────────────────────────────────────────────────────

  Example - Pay-as-you-go:
  ─────────────────────────────────────────────────────────────────
  Rate Card: $0.001 per API call

  Usage: 15,000 API calls

  Total: 15,000 × $0.001 = $15.00
```

---

## 6. Invoice Generation Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      INVOICE GENERATION WORKFLOW                     │
└─────────────────────────────────────────────────────────────────────┘

  Trigger: Cron job (daily at 00:00 UTC)
  ─────────────────────────────────────────

  ┌─────────────────────────────────────────────────────────────────┐
  │  STEP 1: Identify Subscriptions Due                             │
  │  ─────────────────────────────────────                         │
  │  SELECT * FROM customer.subscriptions                           │
  │  WHERE status = 'active'                                        │
  │    AND billing_day_of_month = CURRENT_DAY;                      │
  │                                                                 │
  │  (For PAYG: Generate invoice per customer with usage > 0)     │
  └─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  STEP 2: Create Invoice Record                                  │
  │  ─────────────────────────────────                             │
  │  a. Determine billing period (start_date, end_date)           │
  │  b. Create invoice in draft status                              │
  │  c. Generate invoice number: INV-{YYYY}-{MM}-{###}             │
  │     Example: INV-2026-06-001                                   │
  └─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  STEP 3: Calculate Line Items                                   │
  │  ─────────────────────────────────                             │
  │                                                                 │
  │  For SUBSCRIPTION billing model:                               │
  │  ┌───────────────────────────────────────────────────────────┐  │
  │  │ Line Item Types:                                          │  │
  │  │ ├── Plan Base Fee (e.g., "Pro Plan - Monthly")            │  │
  │  │ ├── Usage Overage (each meter with overage)               │  │
  │  │ ├── Promotional Credits Applied (negative amount)         │  │
  │  │ └── Tax (if applicable)                                   │  │
  │  └───────────────────────────────────────────────────────────┘  │
  │                                                                 │
  │  For PAY-AS-YOU-GO billing model:                              │
  │  ┌───────────────────────────────────────────────────────────┐  │
  │  │ Line Item Types:                                          │  │
  │  │ ├── Usage Charges (each meter × rate)                      │  │
  │  │ ├── Promotional Credits Applied (negative amount)         │  │
  │  │ └── Tax (if applicable)                                   │  │
  │  └───────────────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  STEP 4: Apply Credits                                          │
  │  ───────────────────                                           │
  │  Credits applied in priority order:                            │
  │                                                                 │
  │  1. Compensation (priority 0) - SLA violations                 │
  │  2. Promotional (priority 1) - Marketing offers                │
  │  3. Prepaid (priority 2) - Purchased credit packages           │
  │  4. Commit (priority 3) - Contract commitment allocations      │
  │                                                                 │
  │  Within same priority: FEFO (First Expiring, First Out)        │
  │  Scope restrictions: model-specific credits apply only to       │
  │  that model's usage                                            │
  └─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  STEP 5: Calculate Totals                                      │
  │  ──────────────────────                                         │
  │                                                                 │
  │  Subtotal       = Sum of all line item amounts                 │
  │  Credits        = Sum of credit adjustments (negative)          │
  │  Tax Rate        = Configured tax rate (e.g., 9%)              │
  │  Tax Amount      = (Subtotal - Credits) × Tax Rate             │
  │  Total           = Subtotal - Credits + Tax                    │
  └─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  STEP 6: Finalize Invoice                                       │
  │  ──────────────────────                                         │
  │                                                                 │
  │  Status: draft → pending                                        │
  │  Issue date = today                                             │
  │  Due date = today + payment_terms_days (default: 30)           │
  │                                                                 │
  │  Notification: Invoice email sent to customer                    │
  └─────────────────────────────────────────────────────────────────┘
```

### Invoice State Machine

```
┌─────────────────────────────────────────────────────────────────────┐
│                      INVOICE STATE MACHINE                           │
└─────────────────────────────────────────────────────────────────────┘

  ┌──────────┐   generate    ┌──────────┐   auto-pay /      ┌────┐
  │  draft  │──────────────▶│ pending  │◄─── manual pay ──▶│paid│
  └──────────┘               └────┬─────┘                   └────┘
                                  │                               ▲
                                  │ dunning                       │
                                  ▼                               │
                             ┌──────────┐                         │
                             │ overdue  │─────────────────────────┘
                             └────┬─────┘    (credit/resolve)
                                  │
                            void / write-off
                                  │
                                  ▼
                             ┌──────────┐
                             │ voided   │
                             └──────────┘

  State Descriptions:
  ───────────────────
  • draft      - Invoice being prepared, not yet finalized
  • pending    - Finalized, awaiting payment
  • paid       - Payment received in full
  • overdue    - Payment past due date, dunning triggered
  • voided     - Invoice cancelled or written off
```

---

## 7. Payment & Credits

### 7.1 Payment Recording Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                      PAYMENT RECORDING FLOW                          │
└─────────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────┐
  │  1. Customer initiates payment:                                  │
  │     ├── Manual: Login → Invoices → Pay Now                      │
  │     └── Auto-pay: Payment method on file + scheduled trigger    │
  │                                                                 │
  │  2. Payment gateway processes:                                   │
  │     ├── Stripe / Razorpay / etc.                                 │
  │     └── Payment reference obtained                               │
  │                                                                 │
  │  3. Payment recorded in billing.payments:                        │
  │     ├── status: completed / failed                               │
  │     ├── amount, currency                                        │
  │     └── gateway_reference                                        │
  │                                                                 │
  │  4. If completed:                                               │
  │     ├── Invoice status → paid                                   │
  │     ├── paid_date set                                           │
  │     └── Payment reconciliation logged                            │
  └─────────────────────────────────────────────────────────────────┘
```

### 7.2 Credits System

```
┌─────────────────────────────────────────────────────────────────────┐
│                        CREDITS SYSTEM                                │
└─────────────────────────────────────────────────────────────────────┘

  Credit Types & Priority:
  ┌────────────────────────────────────────────────────────────────┐
  │  Priority  │  Type          │  Description                    │
  │  ──────────┼────────────────┼─────────────────────────────────  │
  │  0 (HIGH)  │  Compensation  │  SLA violations, service issues  │
  │  1         │  Promotional   │  Marketing offers, bonuses       │
  │  2         │  Prepaid       │  Purchased credit packages        │
  │  3 (LOW)   │  Commit        │  Contract commitment allocations │
  └────────────────────────────────────────────────────────────────┘

  Credit Consumption Rules:
  ─────────────────────────
  • Applied automatically during invoice generation
  • FEFO within same priority (First Expiring, First Out)
  • Model-restricted credits apply only to that model's usage
  • Expired credits are automatically invalidated

  Credit Ledger:
  ──────────────
  All credit movements logged in billing.credit_ledger:
  • GRANT → credit granted to customer
  • CONSUME → credit used on invoice
  • REVERSE → credit returned (voided invoice)
  • EXPIRE → credit expired unused
```

### 7.3 Dunning (Overdue Payment Handling)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      DUNNING WORKFLOW                                │
└─────────────────────────────────────────────────────────────────────┘

  Triggered when: invoice status = overdue

  ┌─────────────────────────────────────────────────────────────────┐
  │  Dunning Timeline (configurable per organization):              │
  │                                                                 │
  │  Day 0:  Invoice due date passed → status = overdue            │
  │  Day 1:  Reminder email #1 sent                                 │
  │  Day 7:  Reminder email #2 sent                                 │
  │  Day 14: Final warning email sent                                │
  │  Day 30: Account suspension warning                             │
  │  Day 60: Account suspended / write-off initiated               │
  └─────────────────────────────────────────────────────────────────┘

  Dunning Actions:
  ─────────────────
  • Email reminders (configurable templates)
  • Credit hold (new usage still tracked, not invoiced)
  • Account suspension (access restricted)
  • Write-off (bad debt, mark as unrecoverable)
```

---

## 8. Database Schema Mapping

```
┌─────────────────────────────────────────────────────────────────────┐
│                      DATABASE SCHEMA MAPPING                         │
└─────────────────────────────────────────────────────────────────────┘

  Schema: identity
  ────────────────────────────────────────────────────────────────────
  ┌─────────────────────────────────────────────────────────────────┐
  │  organizations    │ Tenant/org management                        │
  │  users            │ Platform users (org members)                 │
  │  roles            │ RBAC roles                                    │
  │  role_permissions │ Role → permission mappings                   │
  └─────────────────────────────────────────────────────────────────┘

  Schema: customer
  ────────────────────────────────────────────────────────────────────
  ┌─────────────────────────────────────────────────────────────────┐
  │  customers           │ Customer accounts linked to organizations │
  │  end_users           │ End users linked to customers              │
  │  subscriptions       │ Customer subscriptions to products          │
  │  subscription_versions│ Subscription version history              │
  │  contracts           │ Rate card contracts with customers          │
  │  customer_contacts   │ Contact information                       │
  │  entitlement_grants  │ Custom feature grants                      │
  │  usage_limits        │ Per-customer usage limits                  │
  └─────────────────────────────────────────────────────────────────┘

  Schema: catalog
  ────────────────────────────────────────────────────────────────────
  ┌─────────────────────────────────────────────────────────────────┐
  │  products           │ Product definitions                       │
  │  plans              │ Pricing plans within products              │
  │  features           │ Feature flags                             │
  │  meters             │ Usage meters (what is being measured)      │
  │  pricing_models     │ Pricing model definitions                  │
  │  pricing_tiers      │ Tiered pricing configuration               │
  │  rate_cards         │ Versioned rate card collections            │
  │  rate_card_rates    │ Individual rates within rate cards        │
  │  charges            │ Charge definitions                        │
  └─────────────────────────────────────────────────────────────────┘

  Schema: billing
  ────────────────────────────────────────────────────────────────────
  ┌─────────────────────────────────────────────────────────────────┐
  │  invoices           │ Invoice records                           │
  │  invoice_line_items │ Line items per invoice                     │
  │  invoice_batches    │ Batch invoice generation                  │
  │  payments           │ Payment records                           │
  │  payment_methods    │ Stored payment methods                    │
  │  credits            │ Credit balances                           │
  │  credit_ledger      │ Credit transaction history                 │
  │  credit_notes       │ Credit memos                             │
  │  discounts          │ Discount configurations                   │
  │  tax_regions        │ Tax configuration                         │
  │  dunning_policies   │ Overdue payment handling                  │
  │  dunning_steps      │ Dunning timeline steps                    │
  │  dunning_communications│ Dunning email logs                    │
  └─────────────────────────────────────────────────────────────────┘

  Schema: metering
  ────────────────────────────────────────────────────────────────────
  ┌─────────────────────────────────────────────────────────────────┐
  │  usage_snapshots   │ Periodic usage aggregations               │
  │  usage_ledger      │ Running usage totals per meter             │
  └─────────────────────────────────────────────────────────────────┘

  Schema: usage
  ────────────────────────────────────────────────────────────────────
  ┌─────────────────────────────────────────────────────────────────┐
  │  raw_events        │ Raw usage events (ClickHouse)              │
  │  event_ingestion_log│ Ingestion tracking                       │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 9. Appendix: State Machines

### Customer State Machine

```
┌─────────────────────────────────────────────────────────────────────┐
│                      CUSTOMER STATE MACHINE                          │
└─────────────────────────────────────────────────────────────────────┘

        ┌──────────┐
        │  draft   │ (optional initial state)
        └────┬─────┘
             │ activate
             ▼
        ┌──────────┐    suspend     ┌────────────┐
   ────▶│  ACTIVE  │───────────────▶│ SUSPENDED   │
        └────┬─────┘                └──────┬─────┘
             │ churn                         │ resume
             ▼                               │
        ┌──────────┐    delete         ┌────┴────┐
        │ CHURNED  │◀───────────────────│ SUSPENDED│
        └────┬─────┘                   └──────────┘
             │
             │ hard_delete
             ▼
        ┌──────────┐
        │ DELETED  │
        └──────────┘

  State Descriptions:
  ───────────────────
  • draft     - Customer created but not yet active
  • ACTIVE    - Fully operational
  • SUSPENDED  - Temporarily paused (non-payment, etc.)
  • CHURNED   - Customer left (voluntary or involuntary)
  • DELETED   - Hard delete (GDPR, etc.)
```

### Subscription State Machine

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SUBSCRIPTION STATE MACHINE                        │
└─────────────────────────────────────────────────────────────────────┘

        ┌──────────┐
        │  draft   │
        └────┬─────┘
             │ activate
             ▼
        ┌──────────┐    pause      ┌──────────┐
   ────▶│  active  │──────────────▶│  paused  │
        └────┬─────┘               └────┬─────┘
             │                           │
             │ cancel                    │ resume
             ▼                           │
        ┌──────────────┐                 │
        │  cancelled   │◀────────────────┘
        └──────────────┘

  State Descriptions:
  ───────────────────
  • draft    - Subscription being configured
  • active   - Fully operational, usage tracked
  • paused   - Usage tracked but not billed
  • cancelled - Terminated, no further billing
```

### Invoice State Machine

```
┌─────────────────────────────────────────────────────────────────────┐
│                      INVOICE STATE MACHINE                          │
└─────────────────────────────────────────────────────────────────────┘

        ┌──────────┐
        │  draft   │
        └────┬─────┘
             │ finalize
             ▼
        ┌──────────┐    auto-pay /      ┌────┐
        │ pending  │◄──── manual pay ──│paid│
        └────┬─────┘                   └────┘
             │
             │ dunning
             ▼
        ┌──────────┐
        │ overdue  │
        └────┬─────┘
             │
       void / write-off
             │
             ▼
        ┌──────────┐
        │ voided   │
        └──────────┘

  State Descriptions:
  ───────────────────
  • draft   - Being prepared, not sent
  • pending - Sent, awaiting payment
  • paid    - Payment received in full
  • overdue - Past due, dunning active
  • voided  - Cancelled or written off
```

### Credit State Machine

```
┌─────────────────────────────────────────────────────────────────────┐
│                       CREDIT STATE MACHINE                          │
└─────────────────────────────────────────────────────────────────────┘

        ┌──────────┐
        │  active  │
        └────┬─────┘
             │
      ┌──────┴──────┐
      │             │
   consume      expire
      │             │
      ▼             ▼
  ┌──────────┐  ┌──────────┐
  │exhausted │  │ expired   │
  └──────────┘  └──────────┘
      │
      │ reverse (if voided invoice)
      ▼
  ┌──────────┐
  │ reversed  │
  └──────────┘

  State Descriptions:
  ───────────────────
  • active    - Available for use
  • exhausted - Fully consumed on invoices
  • expired   - Past expiration date
  • reversed  - Returned due to invoice void
```

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-06-30 | QuantumBill Team | Initial draft |

---

> **End of Document**
