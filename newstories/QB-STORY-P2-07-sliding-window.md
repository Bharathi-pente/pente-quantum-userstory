# QuantumBilling User Story: Sliding Window Rate Limiting

> Aligned with ADR-001 (2026-07-01).

---

| Field | Value |
|-------|-------|
| **Story ID** | QB-STORY-P2-07 |
| **Sprint** | Sprint 11 |
| **Phase** | Enforcement |
| **Domain** | Rate Limiting — Sliding Window |
| **Priority** | P2 — Medium |

---

## Title

**Sliding Window Rate Limiting** — enforce usage caps using sliding-window time windows instead of fixed period resets

---

## Description

**As a platform engineer**, I want **entitlements to support sliding-window reset** (e.g., "1M tokens in the last 30 days") so that I can **enforce usage caps that don't align to fixed calendar boundaries.**

### Current State

`engine/internal/enforcement/limits.go` has per-period only limit evaluation. No Redis sorted sets for rolling window.

---

## Acceptance Criteria

| # | Criterion |
|---|-----------|
| AC-1 | Metered entitlements support `SLIDING_WINDOW` reset strategy with configurable window duration (1-90 days) |
| AC-2 | Sliding window uses Redis sorted sets: `ZREMRANGEBYSCORE` removes expired entries, `ZCARD` counts current window |
| AC-3 | Enforcement P99 latency <5ms at 32 concurrent callers with sliding window enabled |
| AC-4 | Window slides continuously — an event at 3:00 PM on day N expires at 3:00 PM on day (N + window_duration) |

---

## Code Changes Required

```go
// engine/internal/enforcement/entitlements/sliding.go

type SlidingWindow struct {
    redisClient *redis.Client
    prefix      string  // e.g., "sliding:{customer_id}:{meter_id}"
    duration    time.Duration
}

func (sw *SlidingWindow) Check(ctx context.Context, customerID, meterID string) (int64, error) {
    now := time.Now()
    cutoff := now.Add(-sw.duration)
    key := fmt.Sprintf("sliding:%s:%s", customerID, meterID)
    
    pipe := sw.redisClient.TxPipeline()
    pipe.ZRemRangeByScore(ctx, key, "0", strconv.FormatInt(cutoff.UnixMilli(), 10))
    count := pipe.ZCard(ctx, key)
    _, err := pipe.Exec(ctx)
    if err != nil {
        return 0, err
    }
    return count.Val(), nil
}
```

---

**Estimate:** 1 sprint
**Depends on:** QB-STORY-P0-03 (separate entitlements)
