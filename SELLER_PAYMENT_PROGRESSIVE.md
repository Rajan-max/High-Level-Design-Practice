# Seller Payment System - Progressive Design Approach

## Phase 4: Architecture Evolution (Progressive Approach)

### Iteration 1: Single Server (Naive)


**Goal:** Handle order settlements and pay sellers

```
┌──────────────┐
│ Order Service│
│  (Existing)  │
└──────┬───────┘
       │ ORDER_DELIVERED event
       ▼
┌─────────────────────┐
│  Payment Service    │
│  (Single Server)    │
│                     │
│  - Process orders   │
│  - Update balances  │
│  - Call gateway     │
└──────┬──────────────┘
       │
       ▼
┌─────────────────────┐
│   PostgreSQL DB     │
│  - seller_balance   │
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│  Payment Gateway    │
│  - sendCheck()      │
│  - sendWire()       │
└─────────────────────┘
```

**How it works:**
1. Order delivered → Event received
2. Calculate seller earnings
3. Update seller balance in DB
4. Immediately call gateway to pay seller

**Problems:**
❌ Gateway cost: 10M orders × $0.25 = $75M/month!
❌ Single point of failure
❌ No idempotency (duplicate payments possible)
❌ No audit trail
❌ Gateway takes 1 min - blocks for every order
❌ Can't scale beyond one server

**What to Say:** "This works for tiny scale, but we have massive cost and scalability issues. Let me address them."

---

### Iteration 2: Add Batching - THE KEY DECISION

**Problem:** Paying per order costs $75M/month in gateway fees

**Solution:** Batch payments daily

```
┌──────────────┐
│ Order Service│
└──────┬───────┘
       │ ORDER_DELIVERED
       ▼
┌─────────────────────────────────┐
│  Settlement Service             │
│  - Consume events               │
│  - Update seller_balance        │
│  - NO gateway call yet          │
└──────┬──────────────────────────┘
       │
       ▼
┌─────────────────────────────────┐
│   PostgreSQL                    │
│  ┌──────────────────┐          │
│  │ seller_balance   │          │
│  │ - seller_id      │          │
│  │ - pending_balance│          │
│  └──────────────────┘          │
└─────────────────────────────────┘
       │
       │ Daily at 2 AM
       ▼
┌─────────────────────────────────┐
│  Payout Cron Job                │
│  - Query sellers with balance   │
│  - Call gateway for each        │
└──────┬──────────────────────────┘
       │
       ▼
┌─────────────────────────────────┐
│  Payment Gateway                │
└─────────────────────────────────┘
```

**New Flow:**

**Settlement (Real-time):**
```
1. Order delivered
2. Calculate seller earnings
3. UPDATE seller_balance 
   SET pending_balance = pending_balance + amount
4. Done (no gateway call)
```

**Payout (Daily batch):**
```
1. Cron runs at 2 AM
2. SELECT * FROM seller_balance 
   WHERE pending_balance >= 10.00
3. For each seller:
   - Call gateway
   - Set balance to 0
```

**Cost Analysis:**
```
Before: 10M payouts/day × $0.25 = $75M/month
After:  500K payouts/day × $0.25 = $3.75M/month
Savings: $71.25M/month (95% reduction!)
```

**Why this works:**
✅ Massive cost savings
✅ Separates settlement from payout
✅ Acceptable delay (24 hours)

**Remaining Problems:**
❌ Still single server (can't scale)
❌ No duplicate prevention
❌ No audit log
❌ What if cron fails midway?
❌ What if gateway is down?

---

### Iteration 3: Add Idempotency & Audit Log

**Problem:** Duplicate settlements and no traceability

**Solution:** Add idempotency keys and audit log

```
┌─────────────────────────────────────────┐
│   PostgreSQL                            │
│  ┌──────────────────┐                  │
│  │ seller_balance   │                  │
│  │ - version (NEW)  │ ← Optimistic lock│
│  └──────────────────┘                  │
│                                         │
│  ┌──────────────────┐                  │
│  │ idempotency_keys │ ← Prevent dupes  │
│  │ - key (PK)       │                  │
│  │ - reference_id   │                  │
│  └──────────────────┘                  │
│                                         │
│  ┌──────────────────┐                  │
│  │ audit_log        │ ← Traceability   │
│  │ - event_type     │                  │
│  │ - seller_id      │                  │
│  │ - amount         │                  │
│  └──────────────────┘                  │
└─────────────────────────────────────────┘
```

**Enhanced Settlement Flow:**
```
1. Receive ORDER_DELIVERED event

2. Check idempotency:
   SELECT * FROM idempotency_keys 
   WHERE key = 'settlement:order-123'
   
   If EXISTS: Return (already processed)

3. BEGIN TRANSACTION
   
   a) Update balance (optimistic lock):
      UPDATE seller_balance
      SET pending_balance = pending_balance + 150,
          version = version + 1
      WHERE seller_id = 'seller-456'
      AND version = 5;
      
      If rows_affected = 0:
        → Concurrent update detected
        → Retry
   
   b) Insert audit log:
      INSERT INTO audit_log (
        event_type = 'SETTLEMENT',
        seller_id = 'seller-456',
        reference_id = 'order-123',
        amount = 150
      )
   
   c) Insert idempotency key:
      INSERT INTO idempotency_keys (
        key = 'settlement:order-123',
        reference_id = 'order-123'
      )
   
   COMMIT
```

**Why this works:**
✅ Idempotency key prevents duplicate settlements
✅ Optimistic locking prevents concurrent updates
✅ Audit log provides traceability
✅ ACID transaction ensures consistency

**Remaining Problems:**
❌ Still single server
❌ Payout job has no fault tolerance
❌ No retry logic for gateway failures

---

### Iteration 4: Scale Settlement Service

**Problem:** Single settlement server can't handle 350 settlements/sec peak

**Solution:** Horizontal scaling with Kafka

```
┌──────────────┐
│ Order Service│
└──────┬───────┘
       │
       ▼
┌─────────────────────┐
│   Kafka Topic       │
│ ORDER_DELIVERED     │
│ (Partitioned by     │
│  seller_id)         │
└──────┬──────────────┘
       │
       ├──────────┬──────────┬──────────┐
       ▼          ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│Settlement│ │Settlement│ │Settlement│
│Service 1 │ │Service 2 │ │Service N │
│          │ │          │ │          │
│Consumer  │ │Consumer  │ │Consumer  │
│Group     │ │Group     │ │Group     │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │            │            │
     └────────────┼────────────┘
                  ▼
         ┌─────────────────┐
         │   PostgreSQL    │
         │  (Multi-AZ)     │
         └─────────────────┘
```

**How it works:**
1. Order Service publishes to Kafka
2. Kafka partitions by seller_id
3. Multiple settlement services consume in parallel
4. Each service processes independently
5. Idempotency prevents duplicates

**Why this works:**
✅ Horizontal scaling (add more consumers)
✅ Kafka handles load distribution
✅ Partition by seller_id ensures ordering per seller
✅ Consumer group provides fault tolerance
✅ Can handle 350+ settlements/sec

**Remaining Problems:**
❌ Payout job still single-threaded
❌ No gateway failure handling

---

### Iteration 5: Distributed Payout Processing

**Problem:** Single cron job takes too long (500K payouts sequentially)

**Solution:** Distributed worker pool

```
┌─────────────────────────────────────────┐
│  Payout Scheduler (2 AM daily)          │
│  - Query eligible sellers               │
│  - Create payout records                │
│  - Publish to queue                     │
└──────┬──────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│   SQS Queue                             │
│   - payout-queue                        │
│   - 500K messages                       │
└──────┬──────────────────────────────────┘
       │
       ├──────────┬──────────┬──────────┐
       ▼          ▼          ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Payout   │ │ Payout   │ │ Payout   │
│ Worker 1 │ │ Worker 2 │ │ Worker N │
│          │ │          │ │          │
│ 25 payouts│ │ 25 payouts│ │ 25 payouts│
│ per sec  │ │ per sec  │ │ per sec  │
└────┬─────┘ └────┬─────┘ └────┬─────┘
     │            │            │
     └────────────┼────────────┘
                  ▼
         ┌─────────────────┐
         │ Payment Gateway │
         └─────────────────┘
```

**Scheduler Logic:**
```
1. Run at 2 AM daily

2. Query eligible sellers:
   SELECT seller_id, pending_balance
   FROM seller_balance
   WHERE pending_balance >= 10.00
   FOR UPDATE SKIP LOCKED;  ← Prevent concurrent processing

3. Process in batches of 1000:
   FOR EACH batch:
     BEGIN TRANSACTION
       - Create payout_records (status=PENDING)
       - Reserve balance (set to 0)
       - Publish to SQS
     COMMIT

4. Workers pick up from SQS
```

**Worker Logic:**
```
1. Poll SQS for payout message

2. Fetch seller payment preference

3. Update status to IN_PROGRESS

4. Call gateway:
   IF payment_method = 'CHECK':
     txnId = gateway.sendCheck(details, amount)
   ELSE:
     txnId = gateway.sendWire(details, amount)

5. Handle response:
   SUCCESS: Update status, save txn_id
   ERROR: Update status to FAILED, schedule retry

6. Delete SQS message
```

**Why this works:**
✅ Parallel processing (20 workers)
✅ 500K payouts in 2 hours (vs 347 days sequential)
✅ SQS provides durability and retry
✅ Workers can scale independently
✅ FOR UPDATE SKIP LOCKED prevents duplicates

**Remaining Problems:**
❌ Gateway failures not handled gracefully
❌ No circuit breaker

---

### Iteration 6: Add Resilience - Circuit Breaker & Retry

**Problem:** Gateway downtime causes all payouts to fail

**Solution:** Circuit breaker + exponential backoff

```
┌─────────────────────────────────────────┐
│  Payout Worker                          │
│                                         │
│  ┌────────────────────────────────┐   │
│  │  Circuit Breaker               │   │
│  │  - Track failure rate          │   │
│  │  - Open after 5 failures       │   │
│  │  - Half-open after 5 min       │   │
│  └────────────────────────────────┘   │
│                                         │
│  ┌────────────────────────────────┐   │
│  │  Retry Logic                   │   │
│  │  - Attempt 1: Immediate        │   │
│  │  - Attempt 2: 1 min later      │   │
│  │  - Attempt 3: 5 min later      │   │
│  │  - Max: 3 attempts             │   │
│  └────────────────────────────────┘   │
└──────┬──────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│  Payment Gateway                        │
└─────────────────────────────────────────┘
```

**Circuit Breaker States:**
```
CLOSED (Normal):
- All requests go through
- Track failure rate
- If 5 consecutive failures → OPEN

OPEN (Gateway down):
- Reject all requests immediately
- Don't call gateway
- Mark payouts as DEFERRED
- After 5 minutes → HALF_OPEN

HALF_OPEN (Testing):
- Allow 1 test request
- If success → CLOSED
- If failure → OPEN (wait longer)
```

**Retry Strategy:**
```
Table: payout_records
- attempt_count (track retries)
- next_retry_at (schedule next attempt)
- max_retries = 3

Worker logic:
1. Attempt payout
2. If FAILED:
   - Increment attempt_count
   - Calculate next_retry_at:
     * Attempt 1: now + 1 min
     * Attempt 2: now + 5 min
     * Attempt 3: now + 15 min
   - If attempt_count >= max_retries:
     → Move to manual intervention queue
     → Alert ops team
```

**Why this works:**
✅ Circuit breaker prevents cascading failures
✅ Exponential backoff reduces load during issues
✅ Manual queue for persistent failures
✅ System degrades gracefully

---

### Final Architecture (All Iterations Combined)

```
┌─────────────────────────────────────────────────────────────┐
│                    EXISTING SERVICES                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │  Seller  │  │ Product  │  │  Order   │                 │
│  │ Service  │  │ Service  │  │ Service  │                 │
│  └──────────┘  └──────────┘  └────┬─────┘                 │
└────────────────────────────────────┼─────────────────────────┘
                                     │ ORDER_DELIVERED
                                     ▼
                            ┌─────────────────┐
                            │  Kafka Topic    │
                            │ (Partitioned)   │
                            └────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
        ┌──────────┐       ┌──────────┐      ┌──────────┐
        │Settlement│       │Settlement│      │Settlement│
        │Service 1 │       │Service 2 │      │Service N │
        │          │       │          │      │          │
        │ 35 TPS   │       │ 35 TPS   │      │ 35 TPS   │
        └────┬─────┘       └────┬─────┘      └────┬─────┘
             │                  │                  │
             └──────────────────┼──────────────────┘
                                ▼
                    ┌────────────────────────┐
                    │   PostgreSQL (Multi-AZ)│
                    │                        │
                    │  - seller_balance      │
                    │  - idempotency_keys    │
                    │  - audit_log           │
                    │  - payout_records      │
                    └────────┬───────────────┘
                             │
                             │ Daily 2 AM
                             ▼
                    ┌────────────────────────┐
                    │  Payout Scheduler      │
                    │  - Query eligible      │
                    │  - Create records      │
                    │  - Publish to SQS      │
                    └────────┬───────────────┘
                             │
                             ▼
                    ┌────────────────────────┐
                    │   SQS Queue            │
                    │   500K messages        │
                    └────────┬───────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │ Payout   │   │ Payout   │   │ Payout   │
        │ Worker 1 │   │ Worker 2 │   │ Worker N │
        │          │   │          │   │          │
        │ Circuit  │   │ Circuit  │   │ Circuit  │
        │ Breaker  │   │ Breaker  │   │ Breaker  │
        │ + Retry  │   │ + Retry  │   │ + Retry  │
        └────┬─────┘   └────┬─────┘   └────┬─────┘
             │              │              │
             └──────────────┼──────────────┘
                            ▼
                   ┌─────────────────┐
                   │ Payment Gateway │
                   │  - sendCheck()  │
                   │  - sendWire()   │
                   └─────────────────┘
```

**Complete Flow Summary:**

**Settlement (Real-time, 350 TPS peak):**
```
1. Order delivered → Kafka
2. Settlement service consumes
3. Check idempotency
4. BEGIN TRANSACTION
   - Update balance (optimistic lock)
   - Insert audit log
   - Insert idempotency key
   COMMIT
5. Done in < 5 seconds
```

**Payout (Daily batch, 2 hours):**
```
1. Scheduler runs at 2 AM
2. Query 500K eligible sellers
3. Create payout records + publish to SQS
4. 20 workers process in parallel
5. Each worker:
   - Fetch payment preference
   - Call gateway (with circuit breaker)
   - Handle response
   - Retry on failure (exponential backoff)
6. Complete in ~2 hours
```

---

## Comparison: Evolution Summary

```
┌──────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│ Iteration    │ Cost/Month  │ Scalability │ Reliability │ Complexity  │
├──────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 1. Single    │ $75M        │ 1 server    │ Low         │ Low         │
│    Server    │             │             │             │             │
├──────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 2. Batching  │ $3.75M      │ 1 server    │ Low         │ Low         │
│              │ (95% ↓)     │             │             │             │
├──────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 3. Idempotency│ $3.75M     │ 1 server    │ Medium      │ Medium      │
│    + Audit   │             │             │             │             │
├──────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 4. Scale     │ $3.75M      │ N servers   │ Medium      │ Medium      │
│    Settlement│             │             │             │             │
├──────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 5. Distributed│ $3.75M     │ N servers   │ High        │ High        │
│    Payout    │             │ M workers   │             │             │
├──────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ 6. Resilience│ $3.75M      │ N servers   │ Very High   │ High        │
│    (Final)   │             │ M workers   │             │             │
└──────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

**Key Insights from Progressive Approach:**

1. **Batching is THE key decision** - Saves $71.25M/month (95%)
2. **Idempotency is non-negotiable** - Financial correctness requires it
3. **Horizontal scaling requires message queue** - Kafka for settlement, SQS for payout
4. **Resilience requires circuit breaker** - Gateway failures are inevitable
5. **Audit log is mandatory** - Every transaction must be traceable

---

## Why This Progressive Approach Works

**Interview Benefits:**
✅ Shows iterative thinking (not just final solution)
✅ Identifies problems before solving them
✅ Demonstrates cost awareness (batching decision)
✅ Explains trade-offs at each step
✅ Builds complexity gradually

**What to Say:**
- "Let me start simple and address issues as they arise"
- "This works at small scale, but we have a cost problem"
- "Batching saves $71M/month - that's the key decision"
- "Now we need to scale horizontally"
- "Finally, we add resilience for production readiness"

This approach demonstrates Top 20% thinking! 🎯
