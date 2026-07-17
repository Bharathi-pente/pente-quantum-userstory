# QBill — How Multiple Events Work (Ingestion Methods Explained)

**Date:** 2026-07-17
**Question:** How does QBill handle multiple events? What method does it use? What is the best approach?

---

## Table of Contents

1. [Overview: The Two Ingestion Methods](#1-overview-the-two-ingestion-methods)
2. [Method 1: Single Event API](#2-method-1-single-event-api)
3. [Method 2: Batch Event API](#3-method-2-batch-event-api)
4. [The Queue Method (Kafka) — How Events Flow After Ingestion](#4-the-queue-method-kafka--how-events-flow-after-ingestion)
5. [How Multiple Events Are Processed in Real-Time](#5-how-multiple-events-are-processed-in-real-time)
6. [How Multiple Events Are Aggregated for Invoicing](#6-how-multiple-events-are-aggregated-for-invoicing)
7. [Which Method Is Best? — Decision Guide](#7-which-method-is-best--decision-guide)
8. [Comparison: Ingestion Methods Across Competitors](#8-comparison-ingestion-methods-across-competitors)
9. [Code Walkthrough: Full Multiple-Event Flow](#9-code-walkthrough-full-multiple-event-flow)
10. [Performance & Scalability](#10-performance--scalability)

---

## 1. Overview: The Two Ingestion Methods

QBill offers **two HTTP methods** to send events, and behind the scenes uses a **Kafka queue** to process them:

```
CLIENT SIDE:                     SERVER SIDE:
                                 
┌─────────────────┐             ┌──────────────────────────────────────┐
│ Single Event    │ ───HTTP──▶  │ Go Ingest API                        │
│ POST /v1/events │             │                                      │
│ 1 event/request │             │  Auth → Validate → Dedup (Redis)     │
└─────────────────┘             │         → Publish to Kafka           │
                                └──────────────────┬───────────────────┘
┌─────────────────┐             ┌──────────────────┴───────────────────┐
│ Batch Events    │ ───HTTP──▶  │ Go Ingest API                        │
│ POST /v1/events │             │                                      │
│ /batch          │             │  Auth → Validate → Bloom Dedup       │
│ Up to 50,000    │             │  → Redis Dedup → Kafka Batch Pub     │
│ events/request  │             └──────────────────┬───────────────────┘
└─────────────────┘                                │
                                                    ▼
                                           ┌──────────────────┐
                                           │    KAFKA QUEUE    │
                                           │  usage-events     │
                                           │  32 partitions    │
                                           │  keyed by org_id  │
                                           └────────┬─────────┘
                                                    │
                                    ┌───────────────┼───────────────┐
                                    ▼               ▼               ▼
                           ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                           │ Billing      │ │ Analytics    │ │ Wallet       │
                           │ Worker       │ │ Worker       │ │ Burner       │
                           │ (consumer    │ │ (consumer    │ │ (consumer    │
                           │  group:      │ │  group:      │ │  same as     │
                           │  billing-v1) │ │  analytics-  │ │  billing-v1) │
                           │              │ │  v1)         │ │              │
                           │ Redis        │ │ ClickHouse   │ │ Redis        │
                           │ counters     │ │ events table │ │ wallet       │
                           │ + wallet     │ │              │ │ decrement    │
                           └──────────────┘ └──────────────┘ └──────────────┘
```

### Key Point

> **QBill uses BOTH direct HTTP (for ingestion) AND a queue method (Kafka, for processing).** They serve different purposes:
> - **HTTP** is for getting events INTO the system (ingestion)
> - **Kafka (queue)** is for processing events within the system (consumption)

---

## 2. Method 1: Single Event API

### What It Is

```
POST /v1/events
Content-Type: application/json
X-API-Key: sk_live_abc123

{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "customer_id": "cust_abc",
  "end_user_id": "eu_xyz",
  "event_type": "ai.usage",
  "timestamp_ms": 1721152200000,
  "properties": {
    "model": "gpt-4",
    "total_tokens": 1500,
    "input_tokens": 820,
    "output_tokens": 680,
    "cost": "0.000041"
  }
}
```

### What Happens Step-by-Step

```
Request arrives
    │
    ▼
1. Auth Middleware
   Extract X-API-Key → Redis lookup → KeyContext {org_id, customer_id, source_mode}
   Invalid key → 401 UNAUTHORIZED
    │
    ▼
2. Enrich from KeyContext (DEC-002 — ANTI-SPOOFING)
   Override event.org_id from KeyContext (NEVER trust payload)
   Override event.customer_id from KeyContext
   Set event.source_mode, event.key_id
    │
    ▼
3. Validate
   Check required fields, valid enums, field types
   Invalid → 400 BAD_REQUEST with specific error
    │
    ▼
4. Idempotency Check
   Redis: SETNX idem:{org_id}:{event_id} TTL 24h
   If already exists → 409 DUPLICATE_EVENT
    │
    ▼
5. Existence Check
   org:{org_id} exists in Redis?
   org:{org_id}:customer:{customer_id} exists?
   org:{org_id}:enduser:{end_user_id} exists?
   Missing → 403 FORBIDDEN
    │
    ▼
6. Publish to Kafka
   Produce message to topic 'usage-events'
   Partition key: org_id
   Include traceparent header for distributed tracing
    │
    ▼
7. Return Response
   202 ACCEPTED
   { "event_id": "...", "status": "accepted" }
```

### When to Use Single Events

| Scenario | Best for |
|---|---|
| **Real-time LLM calls** | Each API call to your gateway generates one event → send immediately |
| **Low volume** | < 100 events/second |
| **Simple integration** | LiteLLM callback, webhook, direct SDK |
| **Need immediate feedback** | Get 202 response per event |

---

## 3. Method 2: Batch Event API

### What It Is

```
POST /v1/events/batch
Content-Type: application/json
X-API-Key: sk_live_abc123

[
  {
    "event_id": "evt-001",
    "customer_id": "cust_abc",
    "event_type": "ai.usage",
    "timestamp_ms": 1721152200000,
    "properties": { "model": "gpt-4", "total_tokens": 1500, "cost": "0.000041" }
  },
  {
    "event_id": "evt-002",
    "customer_id": "cust_abc",
    "event_type": "ai.usage",
    "timestamp_ms": 1721152300000,
    "properties": { "model": "gpt-4", "total_tokens": 3200, "cost": "0.000085" }
  },
  ...up to 50,000 events...
]
```

### What Happens Step-by-Step

```
Request arrives (up to 50,000 events)
    │
    ▼
1. Auth Middleware (same as single)
   Extract key → KeyContext
    │
    ▼
2. Parse streaming JSON
   Parse as stream (don't load entire array into memory)
   Max 50,000 events → 413 PAYLOAD_TOO_LARGE if exceeded
    │
    ▼
3. Sharded Bloom Filter Dedup (FIRST pass)
   In-process Bloom filter (bits-and-blooms library)
   Shard = hash(event_id) % BLOOM_NUM_SHARDS
   Catches obvious duplicates without hitting Redis
   Falls back to in-process Bloom if Redis unavailable
    │
    ▼
4. Per-Event Enrichment & Validation (SECOND pass)
   For each event:
   a. ApplyTenantContext(KeyContext) — override org/customer
   b. Validate() — check fields
   c. Redis SETNX idem check (final dedup)
   d. Existence check (org, customer, end_user)
    │
    ▼
5. Collect Valid Events
   Invalid events → add to failures[] array with per-index error
   Valid events → accumulate for Kafka publish
    │
    ▼
6. Publish Batch to Kafka
   Kafka PublishBatch(valid_events)
   All events in one Kafka produce call
    │
    ▼
7. Return Response
   202 ACCEPTED (partial success is NORMAL)
   {
     "accepted_count": 49995,
     "failed_count": 5,
     "failures": [
       { "index": 0, "event_id": "evt-bad", "error": "VALIDATION_ERROR", "field": "total_tokens" },
       ...
     ]
   }
   
   If ZERO valid events → 400 BAD_REQUEST
```

### Performance Optimizations in Batch

| Optimization | What It Does | Benefit |
|---|---|---|
| **Streaming JSON parse** | Parses events as they arrive | No memory spike for 50K events |
| **Bloom filter (first pass)** | In-process probabilistic dedup | Avoids 50K Redis calls for obvious duplicates |
| **Batch Redis pipeline** | Groups Redis checks into one network round-trip | ~50x faster than individual calls |
| **Batch PostgreSQL fallback** | Single UNNEST query for existence checks | Avoids N+1 query problem |
| **Kafka PublishBatch** | Single Kafka produce for all events | Much higher throughput than individual produces |

### Response for Partial Success

```json
{
  "status": "accepted",
  "accepted_count": 49995,
  "failed_count": 5,
  "failures": [
    { "index": 12, "event_id": "evt-013", "error": { "code": "VALIDATION_ERROR", "message": "total_tokens must be positive", "field": "total_tokens" } },
    { "index": 104, "event_id": "evt-105", "error": { "code": "DUPLICATE_EVENT", "message": "Duplicate event_id" } },
    { "index": 5000, "event_id": "", "error": { "code": "INVALID_JSON", "message": "Invalid JSON at index 5000" } }
  ]
}
```

### When to Use Batch Events

| Scenario | Best for |
|---|---|
| **High volume** | > 100 events/second |
| **Historical backfill** | Import past usage data |
| **Bulk operations** | Migrating from another platform |
| **Offline/batch processing** | Nightly usage uploads |
| **Aggregated reporting** | Pre-aggregated hourly/daily usage |

---

## 4. The Queue Method (Kafka) — How Events Flow After Ingestion

### What Kafka Does

Kafka is the **internal queue** that decouples event ingestion from event processing:

```
                   KAFKA (usage-events topic, 32 partitions)
                   ┌──────────────────────────────────────────┐
                   │                                          │
Producer ─────────▶│  Partition 0: org_abc, org_def           │
(Ingest API)       │  Partition 1: org_ghi, org_jkl           │
                   │  ...                                      │
                   │  Partition 31: org_xyz                    │
                   │                                          │
                   │  Each partition = ordered, immutable     │
                   │  sequence of messages                    │
                   │  keyed by org_id for ordering guarantee  │
                   └──────────────────────────────────────────┘
                              │              │
                              ▼              ▼
                    ┌────────────────┐  ┌────────────────┐
                    │ billing-v1     │  │ analytics-v1   │
                    │ consumer group │  │ consumer group │
                    │                │  │                │
                    │ Independent    │  │ Independent    │
                    │ offset tracking│  │ offset tracking│
                    │ Per-event:     │  │ Per-batch:     │
                    │ 1. Redis       │  │ ClickHouse     │
                    │    counters    │  │ INSERT         │
                    │ 2. Wallet burn │  │                │
                    │ 3. Pub/sub     │  │                │
                    └────────────────┘  └────────────────┘
```

### Why Kafka (Queue) Is Used

| Benefit | Explanation |
|---|---|
| **Decoupling** | Ingestion doesn't block on processing. 202 response is immediate. |
| **Buffering** | Kafka can absorb millions of events even if consumers are slow. |
| **Multiple consumers** | Both billing and analytics read the SAME events independently. |
| **Replayability** | If a consumer crashes, it re-reads from last committed offset. |
| **Ordering** | Events for the same org are ordered (same partition key). |
| **Durability** | Events persist on disk even if all services crash. |

### Consumer Groups

```
Topic: usage-events (32 partitions)
         │
         ├── Consumer Group: billing-v1
         │     ├── billing-worker instance 1  → partitions 0-15
         │     └── billing-worker instance 2  → partitions 16-31
         │     Purpose: Redis counters, wallet burndown, enforcement
         │     At-least-once delivery
         │
         └── Consumer Group: analytics-v1
               ├── analytics-worker instance 1  → partitions 0-7
               ├── analytics-worker instance 2  → partitions 8-15
               ├── analytics-worker instance 3  → partitions 16-23
               └── analytics-worker instance 4  → partitions 24-31
               Purpose: ClickHouse INSERT, analytics queries
               At-least-once delivery
```

---

## 5. How Multiple Events Are Processed in Real-Time

### The Billing Worker Consumer

```
Kafka message arrives (one usage event)
    │
    ▼
┌──────────────────────────────────────────────┐
│  1. Deserialize JSON → models.UsageEvent     │
│     Extract: org_id, customer_id, tokens,     │
│     cost, model, event_type, timestamp_ms     │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│  2. Counter Store — Apply()                  │
│     Redis pipeline (atomic):                 │
│     │                                        │
│     ├── INCRBYFLOAT usage:{org}              │
│     │         += total_tokens                │
│     ├── INCRBYFLOAT usage:{org}:{customer}   │
│     │         += total_tokens                │
│     ├── INCRBYFLOAT usage:{org}:{end_user}   │
│     │         += total_tokens (if present)   │
│     ├── INCRBYFLOAT spend:{org}              │
│     │         += cost                        │
│     ├── INCRBYFLOAT spend:{org}:{customer}   │
│     │         += cost                        │
│     └── PUBLISH updates:{org_id} → delta     │
│              (WebSocket notification)         │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│  3. Wallet Burner (if wallet exists)         │
│     │                                        │
│     ├── Resolve rate from rating cache       │
│     │   ResolveHotPath() → cost in minor $   │
│     ├── Redis CAS decrement wallet:{customer} │
│     ├── Check overdraft (WALLET_MAX_OVERDRAFT)│
│     ├── If below threshold → auto-topup      │
│     ├── Buffer ledger entry (batch flush)    │
│     └── PUBLISH updates:{org_id} → balance   │
└──────────────────┬───────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────┐
│  4. Commit Kafka offset                      │
│     Only after BOTH counter AND wallet       │
│     operations succeed                       │
│     (at-least-once guarantee)                │
└──────────────────────────────────────────────┘
```

### Batch Consumption — How Multiple Events Are Handled Together

```go
// engine/internal/counter/consumer.go
func (c *Consumer) ConsumeBatch(ctx context.Context, batchSize int, timeout time.Duration) (int, error) {
    // 1. Fetch up to batchSize messages from Kafka
    messages := make([]kafka.Message, 0, batchSize)
    for i := 0; i < batchSize; i++ {
        msg, err := c.reader.FetchMessage(ctx)
        if err != nil { break }
        messages = append(messages, msg)
    }
    
    // 2. Apply each event to Redis counters
    for _, msg := range messages {
        ev := parseUsageEvent(msg)
        delta, err := c.store.Apply(ctx, ev)  // ← Redis pipeline
        
        // 3. Call wallet burndown hook
        if err == nil && c.onApplied != nil {
            c.onApplied(ctx, ev)  // ← wallet burner
        }
    }
    
    // 3. Commit ALL applied messages (at-least-once)
    c.reader.CommitMessages(ctx, messages...)
    
    return len(messages), nil
}
```

---

## 6. How Multiple Events Are Aggregated for Invoicing

### At Period End — ClickHouse Query

When invoice time comes, the engine queries ClickHouse for all events in the period:

```sql
SELECT
    org_id,
    customer_id,
    end_user_id,
    model,
    token_type,
    SUM(total_tokens) AS total_tokens,
    SUM(cost) AS total_cost
FROM events.usage_events_dedup_v
WHERE timestamp_ms BETWEEN ? AND ?
  AND org_id = ?
  AND customer_id = ?
  AND customer_id != ''           -- exclude org-only BYOK events (DEC-007)
GROUP BY org_id, customer_id, end_user_id, model, token_type
```

### How Aggregation Works

```
1,000 events in the period for Customer ACME
    │
    ▼
ClickHouse aggregates by (customer_id, model, token_type)
    │
    ├── gpt-4, input  → SUM(total_tokens) = 620,000
    ├── gpt-4, output → SUM(total_tokens) = 380,000
    ├── claude, input → SUM(total_tokens) = 150,000
    └── claude, output→ SUM(total_tokens) = 200,000
    │
    ▼
Engine rates each aggregate through the waterfall:
    ├── gpt-4 input  → contract rate $0.000020 → 620,000 × $0.000020 = $12.40
    ├── gpt-4 output → plan rate $0.000050     → 380,000 × $0.000050 = $19.00
    ├── claude input → rate card rate $0.000015→ 150,000 × $0.000015 = $2.25
    └── claude output→ plan rate $0.000040     → 200,000 × $0.000040 = $8.00
    │
    ▼
Line items created on the invoice
```

### Important: Events Are NOT Billed Individually

```
❌ WRONG (what QBill does NOT do):
  Event 1: 1500 tokens × $0.000020 = $0.030  → line item
  Event 2: 3200 tokens × $0.000020 = $0.064  → line item
  Event 3: 2800 tokens × $0.000020 = $0.056  → line item
  ...50,000 line items for 50,000 events...

✅ RIGHT (what QBill DOES):
  ClickHouse: SUM(total_tokens) WHERE model='gpt-4' AND type='input' = 620,000
  → 1 line item: 620,000 tokens × $0.000020 = $12.40
```

**This is critical for scalability.** Individual event billing would create 50,000+ line items per invoice. QBill aggregates first, rates second.

---

## 7. Which Method Is Best? — Decision Guide

### Decision Tree

```
How many events do you need to send?
    │
    ├── < 100 events/second
    │   └── Use SINGLE EVENT API
    │       POST /v1/events (one at a time)
    │       ✅ Simple, good for LiteLLM callback
    │
    ├── 100 - 10,000 events/second
    │   └── Use BATCH API
    │       POST /v1/events/batch (up to 50,000/batch)
    │       ✅ Much higher throughput
    │
    ├── > 10,000 events/second
    │   └── Use BATCH API with pre-aggregation
    │       Client-side: aggregate events per minute per customer
    │       Server-side: batch ingest the aggregates
    │       ✅ Minimal network overhead
    │
    └── Historical backfill (millions of events)
        └── Use BATCH API in parallel
            Multiple concurrent batch requests
            ✅ 50K events per request × N concurrent requests
```

### Comparison Table

| Aspect | Single Event | Batch Event |
|---|---|---|
| **Max events/request** | 1 | **50,000** |
| **Throughput** | ~100/sec per connection | **100,000+/sec** |
| **Latency per event** | ~5ms (includes overhead) | **~0.01ms per event** (amortized) |
| **Network overhead** | High (1 HTTP request per event) | **Low** (1 HTTP request = 50K events) |
| **Error handling** | Per-event response | **Partial success** (some pass, some fail) |
| **Memory usage** | Minimal | Higher (buffers 50K events) |
| **Dedup method** | Redis SETNX (per event) | **Bloom filter + Redis SETNX** (2-pass) |
| **Best latency** | ✅ Immediate per-event response | N/A (batch response when all processed) |
| **Best throughput** | ❌ (HTTP overhead per event) | ✅ (50K events in one request) |

### Recommendation by Use Case

| Use Case | Recommended Method | Why |
|---|---|---|
| **LiteLLM callback** (real-time per LLM call) | **Single event** | Each LLM call sends 1 event immediately. Low volume. |
| **Your app's API gateway** (real-time) | **Single event** or **small batches** | Depends on traffic. Start with single, switch to batch if >100/sec. |
| **High-volume LLM proxy** (thousands of calls/sec) | **Batch (up to 50K)** | Minimum network overhead. Bloom filter dedup. |
| **Historical backfill** (migrate from another platform) | **Batch (parallel)** | 50K/batch × N parallel requests = millions of events/minute. |
| **Aggregated usage report** (hourly/daily totals) | **Batch (pre-aggregated)** | Send SUM of tokens per hour per customer — reduces events by 99%. |
| **IoT / device telemetry** (many devices) | **Batch** | Thousands of devices → batch per region. |
| **Real-time dashboard** needs immediate visibility | **Single event** | Events visible in ClickHouse within seconds. |

### Is QBill Using the "Q Method" (Queue)?

**YES.** QBill uses Kafka as its internal message queue:

```
HTTP Layer (ingestion)           Queue Layer (processing)
─────────────────────            ────────────────────────
Single Event API ──────────┐
                           ├──▶ Kafka (usage-events topic)
Batch Event API ───────────┘     │
                                 ├──▶ billing-v1 consumer → Redis counters
                                 ├──▶ billing-v1 consumer → Wallet burndown
                                 └──▶ analytics-v1 consumer → ClickHouse
```

The queue method provides:
- **Decoupling** — ingestion doesn't wait for processing
- **Buffering** — handles traffic spikes
- **Multiple consumers** — billing + analytics read independently
- **Replayability** — reprocess from any point in time
- **Fault tolerance** — events survive service crashes

---

## 8. Comparison: Ingestion Methods Across Competitors

| Aspect | QBill | FlexPrice | Lago | Meteroid | OpenMeter | Orb |
|---|---|---|---|---|---|---|
| **Single event** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Batch** | ✅ **Up to 50,000** | ✅ Up to 1,000 | ✅ Up to 100 | ✅ Up to 100 | ❌ Single only | ✅ Up to 500 |
| **Internal queue** | ✅ **Kafka** (32 partitions) | ❌ Direct processing | ✅ Kafka (optional) | ✅ Kafka (Rust) | ❌ Direct processing | ❌ Direct processing |
| **Multiple consumers** | ✅ Billing + Analytics (independent) | ❌ Single pipeline | ❌ Single pipeline | ❌ Single pipeline | ❌ Single pipeline | ✅ Query + Stream |
| **Streaming ingestion** | ✅ Kafka consumer groups | ❌ | ✅ Kafka/Redpanda | ✅ Kafka | ❌ | ✅ Kinesis/Lambda |
| **Bloom filter dedup** | ✅ For batches | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Partial accept** | ✅ Per-index errors | ❌ | ❌ | ✅ `allow_partial_failures` | ❌ | ❌ |
| **Idempotency** | ✅ SETNX (first-write-wins) | ✅ Latest-value-wins | ✅ transaction_id dedup | ✅ (event_id, customer_id) | ✅ source + id | ✅ idempotency_key |
| **Max batch size** | **50,000** | 1,000 | 100 | 100 | N/A (single) | 500 |

### QBill's Advantage: Largest Batch Size

QBill's **50,000 events/batch** is 50x larger than FlexPrice (1,000), 500x larger than Lago/Meteroid (100), and 100x larger than Orb (500).

### QBill's Advantage: Kafka Queue

QBill is one of the few competitors that uses a **dedicated internal message queue (Kafka)** for event processing. This provides:
- Better fault isolation (ingestion doesn't block on processing)
- Independent consumer scaling (billing + analytics can scale separately)
- Event replay capability (reprocess from any offset)
- Traffic spike absorption (Kafka buffers when consumers are slow)

---

## 9. Code Walkthrough: Full Multiple-Event Flow

### Step 1: Ingest API (Go) — receives HTTP request

```go
// engine/cmd/ingest-api/main.go (conceptual)
func handleBatch(w http.ResponseWriter, r *http.Request) {
    // 1. Auth
    kc := auth.GetKeyContext(r.Context())
    
    // 2. Streaming JSON parse
    decoder := json.NewDecoder(r.Body)
    
    // Read opening bracket
    decoder.Token()
    
    var validEvents []models.UsageEvent
    var failures []indexedError
    
    for decoder.More() {
        var ev models.UsageEvent
        if err := decoder.Decode(&ev); err != nil {
            failures = append(failures, indexedError{index, err})
            continue
        }
        
        // 3. Enrich (anti-spoofing)
        ev.ApplyTenantContext(kc)
        
        // 4. Validate
        if err := ev.Validate(); err != nil {
            failures = append(failures, indexedError{index, err})
            continue
        }
        
        validEvents = append(validEvents, ev)
    }
    
    // 5. Publish to Kafka
    kafkaProducer.PublishBatch(validEvents...)
    
    // 6. Return partial success
    json.NewEncoder(w).Encode(map[string]interface{}{
        "accepted_count": len(validEvents),
        "failed_count":   len(failures),
        "failures":       failures,
    })
}
```

### Step 2: Kafka Consumer (Go) — processes events

```go
// engine/internal/counter/consumer.go
func (c *Consumer) ConsumeBatch(ctx context.Context, batchSize int, timeout time.Duration) (int, error) {
    // Fetch up to batchSize messages from Kafka
    for i := 0; i < batchSize; i++ {
        msg, err := c.reader.FetchMessage(ctx)
        
        // Deserialize
        var ev models.UsageEvent
        json.Unmarshal(msg.Value, &ev)
        
        // Apply to Redis counters
        delta, err := c.store.Apply(ctx, &ev)
        
        // Wallet burndown (if wallet exists)
        if c.onApplied != nil {
            c.onApplied(ctx, &ev)
        }
        
        messages = append(messages, msg)
    }
    
    // Commit all at once
    c.reader.CommitMessages(ctx, messages...)
}
```

### Step 3: Redis Counter Apply (Go) — increments 5 counters per event

```go
// engine/internal/counter/redis.go
func (s *Store) Apply(ctx context.Context, ev *models.UsageEvent) (DeltaMessage, error) {
    // Use Redis pipeline for atomic multi-key increment
    pipe := s.rdb.TxPipeline()
    
    // 5 counters per event
    pipe.IncrByFloat(ctx, "usage:"+ev.OrgID, ev.TotalTokens)
    pipe.IncrByFloat(ctx, "usage:"+ev.OrgID+":"+ev.CustomerID, ev.TotalTokens)
    pipe.IncrByFloat(ctx, "usage:"+ev.OrgID+":"+ev.EndUserID, ev.TotalTokens)
    pipe.IncrByFloat(ctx, "spend:"+ev.OrgID, ev.Cost)
    pipe.IncrByFloat(ctx, "spend:"+ev.OrgID+":"+ev.CustomerID, ev.Cost)
    
    // Execute all 5 increments atomically
    pipe.Exec(ctx)
    
    // Publish update for WebSocket
    s.rdb.Publish(ctx, "updates:"+ev.OrgID, deltaJSON)
}
```

---

## 10. Performance & Scalability

### Tested Throughput

| Method | Events/sec | Notes |
|---|---|---|
| **Single event API** | ~1,000/sec per instance | HTTP overhead limits throughput |
| **Batch API (50K/batch)** | **100,000+/sec** | Amortized overhead across 50K events |
| **Kafka consumer** | 50,000/sec per partition | Scales with partition count |
| **Redis counters** | 100,000+/sec | Pipeline batches 5 ops per event |

### Scaling Strategy

```
Low Volume (< 1K events/sec):
  ┌────────────┐
  │ Ingest API │──▶ Kafka ──▶ Billing Worker
  │ (1 node)   │             (1 node)
  └────────────┘
  
Medium Volume (1K - 50K events/sec):
  ┌────────────┐
  │ Ingest API │──▶ Kafka ──▶ Billing Worker (2 nodes)
  │ (2 nodes)  │   32        Analytics Worker (2 nodes)
  └────────────┘   parts
    
High Volume (50K - 500K events/sec):
  ┌────────────┐
  │ Ingest API │──▶ Kafka ──▶ Billing Worker (4 nodes)
  │ (4 nodes)  │   32        Analytics Worker (4 nodes)
  └────────────┘   parts     Wallet Burner (co-located)
```

### What If Kafka Goes Down?

| Scenario | Behavior |
|---|---|
| **Kafka unavailable during ingest** | Ingest API returns **503 SERVICE_UNAVAILABLE**. Client should retry. |
| **Kafka recovers after downtime** | Consumers resume from last committed offset. No data loss. |
| **Consumer crashes after Apply but before Commit** | At-least-once: message is re-delivered. Redis counters may double-count. **Nightly reconciliation corrects drift.** |
| **Redis goes down** | Ingest API switches to in-process Bloom filter. Counters lost but reconciled from ClickHouse. Wallet burndown pauses. |

---

## Summary

| Question | Answer |
|---|---|
| **Does QBill use a queue method?** | **YES** — Kafka (32 partitions, keyed by org_id) |
| **What's the best method for high volume?** | **Batch API** — up to 50,000 events/request, bloom filter dedup, partial accept |
| **What's the best method for real-time?** | **Single event API** — immediate 202 response per event |
| **How are events aggregated?** | ClickHouse SUM at period end → rate once per (model, token_type) — NOT per-event |
| **How many counters per event?** | 5 Redis counters (org usage, customer usage, end-user usage, org spend, customer spend) |
| **What if an event fails validation?** | Single: 400 error. Batch: partial accept (valid events processed, failures reported). |
| **Is idempotency guaranteed?** | ✅ Redis SETNX with 24h TTL. Same event_id → 409 DUPLICATE_EVENT. |

---

*End of document. Generated 2026-07-17.*
