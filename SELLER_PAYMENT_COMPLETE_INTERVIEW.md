# Seller-Side Payment System - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: High-Level Architecture (8 min)
Phase 5: API Design (3 min)
Phase 6: Data Models (5 min)
Phase 7: Core Design Decision - Payment Frequency (5 min)
Phase 8: Deep Dive - Settlement & Payout Flow (8 min)
Phase 9: Preventing Double Payouts (5 min)
Phase 10: Follow-up Questions (3 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"Thank you for the problem. This is a seller-side payment system, so we're focusing on paying out sellers, not collecting from buyers. Let me ask some clarifying questions to understand the requirements better."

### Questions to Ask:

**Q1: Existing System Context**
- "I see we have SellerService, ProductService, and OrderService. Can you confirm the schemas?"
- "SellerService has paymentDetails[check/wire] - so sellers can choose their payment method?"

**Expected Answer:** Yes, sellers choose check or wire transfer

**Q2: When to Pay Sellers**
- "When should sellers be paid? After order is placed, delivered, or after return window?"
- "Is there a holding period for refunds/chargebacks?"

**Expected Answer:** After order is delivered (assume no return window for simplicity)

**Q3: Payment Frequency - THE KEY QUESTION**
- "Should we pay sellers after every order, or can we batch payments daily/weekly?"
- "What's more important: immediate payment or minimizing gateway fees?"

**Expected Answer:** This is for you to propose (hint: batching saves fees)

**Q4: Scale & Volume**
- "How many sellers do we have?"
- "How many orders per day?"
- "What's the average order value?"

**Expected Answer:** 1M sellers, 10M orders/day, $50 average

**Q5: Payment Gateway**
- "The gateway takes ~1 minute to process. Is this synchronous or async?"
- "What's the fixed fee per transfer?"
- "Do we get callbacks/webhooks for payment status?"

**Expected Answer:** Async with callbacks, $0.25 per transfer

**Q6: Requirements Priority**
- "You mentioned audit log is #1 priority. Should every transaction be traceable?"
- "For 'no duplicate payments' - how critical is this? Financial correctness?"

**Expected Answer:** Audit log is mandatory, no duplicates is critical (money!)

**Q7: Seller Experience**
- "Sellers will call about payment status. What information should we provide?"
- "Should we show pending vs available balance?"

**Expected Answer:** Show status, balance, and action required if failed

**Q8: Failure Handling**
- "What if gateway is down? Should we retry or queue?"
- "What if a payout fails after multiple retries?"

**Expected Answer:** Retry with backoff, manual intervention queue

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### What to Say:

"Based on our discussion, let me summarize the requirements in order of priority."

### Functional Requirements (Priority Order):

```
1. Audit Log (HIGHEST PRIORITY)
   - Every transaction must be logged
   - Immutable, append-only
   - Traceable: order → settlement → payout
   - Include: seller_id, amount, timestamp, status

2. Payment Method Support
   - Check payments (via sendCheck API)
   - Wire transfers (via sendWire API)
   - Fetch preference from SellerService

3. Payment Status API
   - Sellers can query their balance
   - Sellers can view payout history
   - Show payout status with actionable messages
   - Example: "Update bank details" if failed

4. Minimize Gateway Fees
   - Gateway charges $0.25 per transfer
   - Batch payments to reduce transaction count
   - Daily batch preferred over per-order

5. No Dropped Payments
   - Retry failed payouts (with exponential backoff)
   - Manual intervention queue for persistent failures
   - Alert operations team

6. No Duplicate Payments (CRITICAL)
   - Idempotency at every level
   - Optimistic locking for balance updates
   - State machine for payout status
   - Database constraints

Out of Scope:
- Buyer-side payments (already exists)
- Fraud detection
- Tax calculation
- Currency conversion
- Seller onboarding/KYC
```

### Non-Functional Requirements:

```
1. Consistency
   - STRONG consistency for money operations
   - ACID transactions required
   - No eventual consistency acceptable

2. Correctness
   - Exactly-once payout semantics
   - No double payouts (financial correctness)
   - Balance must always be accurate

3. Performance
   - Settlement: < 5 seconds per order
   - Balance query: < 100ms
   - Payout initiation: < 10 seconds
   - Batch job: Complete in 2 hours

4. Scalability
   - 1M sellers
   - 10M orders/day
   - 500K payouts/day (if daily batch)
   - Handle 3x growth

5. Availability
   - 99.9% uptime for read operations
   - Graceful degradation if gateway down
   - Async processing for resilience

6. Auditability
   - Every transaction traceable
   - Reconciliation with gateway
   - Compliance ready (SOC 2, PCI-DSS)
```

### Why This Matters:
✓ Shows understanding of priorities
✓ Clarifies money = strong consistency
✓ Sets expectations explicitly

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me do some quick calculations to understand the scale and validate our design decisions."

### Traffic Estimation:

```
Given:
- 1M sellers
- 10M orders/day
- Average order value: $50
- Gateway fee: $0.25 per transfer
- Gateway processing time: ~1 minute

Calculations:

1. Settlement Operations (Write)
   = 10M orders/day
   = 10M / 86,400 seconds
   = ~116 settlements/second (average)
   = ~350 settlements/second (peak, 3x)

2. Balance Queries (Read)
   = Assume 10% of sellers check daily
   = 100K queries/day
   = ~1.2 queries/second (negligible)

3. Payout Operations
   
   Option A: Per-Order Payouts
   = 10M payouts/day
   = ~116 payouts/second
   = Gateway cost: 10M × $0.25 = $2.5M/day = $75M/month ❌
   
   Option B: Daily Batch Payouts
   = Assume 500K unique sellers receive orders daily
   = 500K payouts/day
   = ~6 payouts/second (during batch window)
   = Gateway cost: 500K × $0.25 = $125K/day = $3.75M/month ✓
   
   Savings: $75M - $3.75M = $71.25M/month (95% reduction!)

4. Batch Processing Time
   = 500K payouts
   = Gateway: 1 minute per payout
   = If sequential: 500K minutes = 347 days ❌
   = If parallel (100 workers): 5,000 minutes = 83 hours ❌
   = If parallel (1000 workers): 500 minutes = 8.3 hours ❌
   = Solution: Process in batches, gateway handles concurrency
   = Realistic: 2 hours with proper parallelization ✓
```

### Storage Estimation:

```
1. Seller Balance Table
   - 1M sellers
   - Each row: ~200 bytes
   - Total: 1M × 200B = 200MB

2. Payout Records
   - 500K payouts/day
   - Retention: 7 years (compliance)
   - Total records: 500K × 365 × 7 = 1.28B records
   - Each record: ~500 bytes
   - Total: 1.28B × 500B = 640GB

3. Audit Log
   - Events: settlements (10M/day) + payouts (500K/day)
   - Total: 10.5M events/day
   - Retention: 7 years
   - Total records: 10.5M × 365 × 7 = 27B records
   - Each record: ~300 bytes
   - Total: 27B × 300B = 8.1TB

4. Idempotency Keys
   - 10M settlements/day + 500K payouts/day
   - Retention: 7 days (rolling)
   - Total: 10.5M × 7 = 73.5M keys
   - Each key: ~100 bytes
   - Total: 73.5M × 100B = 7.35GB

Total Storage: ~9TB (mostly audit logs)
```

### Cost Estimation:

```
1. Gateway Fees (Daily Batch)
   - 500K payouts/day × $0.25
   = $125,000/day
   = $3.75M/month
   = $45M/year

2. Database (PostgreSQL RDS)
   - db.r5.2xlarge (Multi-AZ)
   - Storage: 10TB
   - Cost: ~$2,000/month

3. Application Servers
   - Settlement service: 10 instances (m5.large)
   - Payout service: 20 instances (m5.large)
   - Cost: ~$2,000/month

4. Monitoring & Logging
   - CloudWatch, DataDog
   - Cost: ~$500/month

Total Infrastructure: ~$4,500/month
Total with Gateway Fees: ~$3.75M/month

Key Insight: Gateway fees dominate (99.9% of cost)
→ Batching is critical for cost savings!
```

### Summary Table:

```
┌─────────────────────────┬──────────────────┐
│ Metric                  │ Value            │
├─────────────────────────┼──────────────────┤
│ Sellers                 │ 1M               │
│ Orders/day              │ 10M              │
│ Settlements/sec (peak)  │ 350              │
│ Payouts/day (batch)     │ 500K             │
│ Storage                 │ ~9TB             │
│ Gateway cost (batch)    │ $3.75M/month     │
│ Gateway cost (per-order)│ $75M/month       │
│ Savings with batch      │ $71.25M/month    │
│ Infrastructure cost     │ $4,500/month     │
└─────────────────────────┴──────────────────┘
```

### Why This Matters:
✓ Shows quantitative thinking
✓ Validates design decision (batching)
✓ Demonstrates cost awareness
✓ Identifies key constraint (gateway fees)

---

## PHASE 4: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"Let me present the high-level architecture. I'll separate the settlement flow (order → balance) from the payout flow (balance → bank)."

### Complete Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXISTING SERVICES                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Seller     │  │   Product    │  │    Order     │         │
│  │   Service    │  │   Service    │  │   Service    │         │
│  │              │  │              │  │              │         │
│  │ - seller_id  │  │ - product_id │  │ - order_id   │         │
│  │ - name       │  │ - seller_id  │  │ - buyer_id   │         │
│  │ - payment    │  │ - sellerPrice│  │ - products[] │         │
│  │   Details    │  │ - buyerPrice │  │ - timestamp  │         │
│  │   [check/    │  │              │  │              │         │
│  │    wire]     │  │              │  │              │         │
│  └──────────────┘  └──────────────┘  └──────┬───────┘         │
└─────────────────────────────────────────────┼─────────────────┘
                                              │
                                              │ Order Delivered Event
                                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              NEW: SELLER PAYMENT SYSTEM                          │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  1. SETTLEMENT SERVICE (Event Consumer)                │    │
│  │     - Consumes ORDER_COMPLETED events from Kafka       │    │
│  │     - Calculates seller earnings                       │    │
│  │     - Updates seller balance (with optimistic lock)    │    │
│  │     - Writes audit log                                 │    │
│  │     - Ensures idempotency                              │    │
│  │                                                          │    │
│  │     Technology: Java/Spring Boot                       │    │
│  │     Instances: 10 (auto-scaling)                       │    │
│  │     Throughput: 350 settlements/sec (peak)             │    │
│  └────────────────────────────────────────────────────────┘    │
│                         ↓                                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  2. DATABASE LAYER (PostgreSQL)                        │    │
│  │                                                          │    │
│  │  ┌──────────────────────────────────────────────┐     │    │
│  │  │  seller_balance                              │     │    │
│  │  │  - seller_id (PK)                            │     │    │
│  │  │  - pending_balance                           │     │    │
│  │  │  - last_payout_date                          │     │    │
│  │  │  - version (optimistic locking)              │     │    │
│  │  └──────────────────────────────────────────────┘     │    │
│  │                                                          │    │
│  │  ┌──────────────────────────────────────────────┐     │    │
│  │  │  payout_records                              │     │    │
│  │  │  - payout_id (PK)                            │     │    │
│  │  │  - seller_id                                 │     │    │
│  │  │  - amount, status, gateway_txn_id           │     │    │
│  │  │  - attempt_count, error_message              │     │    │
│  │  └──────────────────────────────────────────────┘     │    │
│  │                                                          │    │
│  │  ┌──────────────────────────────────────────────┐     │    │
│  │  │  audit_log (append-only)                     │     │    │
│  │  │  - event_id (PK)                             │     │    │
│  │  │  - event_type, seller_id, amount             │     │    │
│  │  │  - reference_id, timestamp                   │     │    │
│  │  └──────────────────────────────────────────────┘     │    │
│  │                                                          │    │
│  │  ┌──────────────────────────────────────────────┐     │    │
│  │  │  idempotency_keys                            │     │    │
│  │  │  - idempotency_key (PK)                      │     │    │
│  │  │  - reference_id, created_at                  │     │    │
│  │  └──────────────────────────────────────────────┘     │    │
│  │                                                          │    │
│  │  Configuration: db.r5.2xlarge (Multi-AZ)               │    │
│  │  Replication: 1 master + 2 read replicas               │    │
│  └────────────────────────────────────────────────────────┘    │
│                         ↓                                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  3. PAYOUT SCHEDULER (Cron Job)                        │    │
│  │     - Runs daily at 2 AM UTC                           │    │
│  │     - Queries sellers with balance ≥ $10               │    │
│  │     - Creates payout records                           │    │
│  │     - Reserves balance (optimistic lock)               │    │
│  │     - Triggers payout processor (async)                │    │
│  │                                                          │    │
│  │     Technology: AWS EventBridge + Lambda               │    │
│  │     Duration: ~2 hours for 500K payouts                │    │
│  └────────────────────────────────────────────────────────┘    │
│                         ↓                                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  4. PAYOUT PROCESSOR (Worker Pool)                     │    │
│  │     - Fetches seller payment preference                │    │
│  │     - Calls gateway API (check/wire)                   │    │
│  │     - Handles async callbacks                          │    │
│  │     - Updates payout status                            │    │
│  │     - Retry logic with exponential backoff             │    │
│  │     - Circuit breaker for gateway failures             │    │
│  │                                                          │    │
│  │     Technology: Java/Spring Boot                       │    │
│  │     Instances: 20 workers                              │    │
│  │     Throughput: 25 payouts/sec (500K in 2 hours)       │    │
│  └────────────────────────────────────────────────────────┘    │
│                         ↓                                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  5. PAYMENT STATUS API (REST)                          │    │
│  │     - GET /sellers/{id}/balance                        │    │
│  │     - GET /sellers/{id}/payouts                        │    │
│  │     - GET /payouts/{id}/status                         │    │
│  │                                                          │    │
│  │     Technology: Java/Spring Boot                       │    │
│  │     Instances: 5 (low traffic)                         │    │
│  │     Cache: Redis (balance queries)                     │    │
│  └────────────────────────────────────────────────────────┘    │
│                         ↓                                        │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  6. RECONCILIATION SERVICE (Daily)                     │    │
│  │     - Runs at 3 AM (after payouts)                     │    │
│  │     - Fetches gateway report                           │    │
│  │     - Matches with internal records                    │    │
│  │     - Alerts on discrepancies                          │    │
│  │     - Generates reconciliation report                  │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│           THIRD PARTY PAYMENT GATEWAY                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  API:                                                     │  │
│  │  - sendCheck(checkDetails, amount) → transactionId       │  │
│  │  - sendWire(wireDetails, amount) → transactionId         │  │
│  │                                                            │  │
│  │  Webhooks:                                                │  │
│  │  - POST /webhooks/payment-success                        │  │
│  │  - POST /webhooks/payment-failed                         │  │
│  │                                                            │  │
│  │  Characteristics:                                         │  │
│  │  - Processing time: ~1 minute                            │  │
│  │  - Fee: $0.25 per transfer                               │  │
│  │  - Rate limit: 100 requests/second                       │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  SUPPORTING INFRASTRUCTURE                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │    Kafka     │  │    Redis     │  │  Monitoring  │         │
│  │  (Events)    │  │   (Cache)    │  │  (DataDog)   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### Explain Data Flow:

**SETTLEMENT FLOW (Order → Balance):**
```
1. Order Service: Order delivered
   ↓
2. Publish ORDER_COMPLETED event to Kafka
   ↓
3. Settlement Service consumes event
   ↓
4. Check idempotency (already processed?)
   ├─ YES → Skip
   └─ NO → Continue
   ↓
5. Fetch products from ProductService
   ↓
6. Calculate seller earnings (sum of sellerPrice)
   ↓
7. BEGIN TRANSACTION
   ├─ Update seller_balance (optimistic lock)
   ├─ Insert audit_log
   └─ Insert idempotency_key
   COMMIT
   ↓
8. Return success

Latency: < 5 seconds
Throughput: 350 settlements/sec (peak)
```

**PAYOUT FLOW (Balance → Bank):**
```
1. Cron triggers at 2 AM daily
   ↓
2. Payout Scheduler queries eligible sellers
   SELECT * FROM seller_balance 
   WHERE pending_balance >= 10.00
   ↓
3. For each seller (in batches of 1000):
   BEGIN TRANSACTION
   ├─ Create payout record (status=PENDING)
   ├─ Reserve balance (set to 0, optimistic lock)
   └─ Insert audit_log
   COMMIT
   ↓
4. Trigger Payout Processor (async, 20 workers)
   ↓
5. For each payout:
   ├─ Fetch seller payment preference
   ├─ Update status to IN_PROGRESS
   ├─ Call gateway API (sendCheck/sendWire)
   ├─ Handle response:
   │  ├─ SUCCESS → Update status, save txn_id
   │  └─ ERROR → Update status to FAILED, schedule retry
   └─ Write audit_log
   ↓
6. Gateway processes (async, ~1 minute)
   ↓
7. Gateway sends webhook (success/failure)
   ↓
8. Update payout status to SUCCESS/FAILED
   ↓
9. Reconciliation job (3 AM)
   ├─ Fetch gateway report
   ├─ Match with internal records
   └─ Alert on discrepancies

Duration: ~2 hours for 500K payouts
Success Rate: 99%+ (with retries)
```

### Why This Matters:
✓ Separates concerns (settlement vs payout)
✓ Shows async processing
✓ Demonstrates scalability
✓ Considers failure handling

---

## PHASE 5: API DESIGN (3 minutes)

### What to Say:

"Let me define the key API endpoints for seller payment status queries."

### API Endpoints:

**1. Get Seller Balance**
```
GET /api/v1/sellers/{sellerId}/balance

Response: 200 OK
{
  "seller_id": "seller-456",
  "pending_balance": 250.00,
  "currency": "USD",
  "last_payout_date": "2024-01-14T02:00:00Z",
  "next_payout_date": "2024-01-15T02:00:00Z",
  "payout_threshold": 10.00,
  "eligible_for_payout": true
}

Error Responses:
- 404 Not Found: Seller not found
- 500 Internal Server Error
```

**2. Get Payout History**
```
GET /api/v1/sellers/{sellerId}/payouts?page=1&limit=20

Response: 200 OK
{
  "payouts": [
    {
      "payout_id": "payout-789",
      "amount": 500.00,
      "currency": "USD",
      "payment_method": "WIRE",
      "status": "SUCCESS",
      "gateway_transaction_id": "txn-abc123",
      "created_at": "2024-01-14T02:00:00Z",
      "completed_at": "2024-01-14T02:01:30Z"
    },
    {
      "payout_id": "payout-790",
      "amount": 150.00,
      "currency": "USD",
      "payment_method": "CHECK",
      "status": "IN_PROGRESS",
      "created_at": "2024-01-15T02:00:00Z",
      "estimated_completion": "2024-01-15T02:02:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total_count": 100,
    "has_next": true
  }
}
```

**3. Get Specific Payout Status**
```
GET /api/v1/payouts/{payoutId}/status

Response: 200 OK
{
  "payout_id": "payout-790",
  "seller_id": "seller-456",
  "amount": 150.00,
  "status": "FAILED",
  "payment_method": "WIRE",
  "error_message": "Invalid bank account details",
  "attempt_count": 2,
  "max_retries": 3,
  "next_retry_at": "2024-01-15T02:10:00Z",
  "action_required": "UPDATE_BANK_DETAILS",
  "user_message": "Your payout failed because of invalid bank account details. Please update your bank information in seller settings and we will retry automatically.",
  "created_at": "2024-01-15T02:00:00Z",
  "updated_at": "2024-01-15T02:05:00Z"
}

Status Values:
- PENDING: Scheduled, not yet processed
- IN_PROGRESS: Being processed by gateway
- SUCCESS: Completed successfully
- FAILED: Failed, will retry
- CANCELLED: Cancelled by system/admin
```

**4. Admin: Manual Payout Trigger**
```
POST /api/v1/admin/payouts/trigger

Request:
{
  "seller_id": "seller-456",
  "amount": 150.00,
  "reason": "Manual payout requested by seller"
}

Response: 202 Accepted
{
  "payout_id": "payout-999",
  "status": "PENDING",
  "message": "Payout scheduled for processing"
}

Authorization: Admin role required
```

### Why This Matters:
✓ Clear contract for sellers
✓ Actionable error messages
✓ Supports troubleshooting
✓ Admin capabilities


## PHASE 6: DATA MODELS (5 minutes)

### What to Say:

"Let me define the database schemas. Since this is a financial system, we need strong consistency, so I'm using PostgreSQL with ACID guarantees."

### 1. Seller Balance Table

```sql
CREATE TABLE seller_balance (
    seller_id VARCHAR(50) PRIMARY KEY,
    pending_balance DECIMAL(19, 4) NOT NULL DEFAULT 0,
    last_payout_date TIMESTAMP,
    version INT NOT NULL DEFAULT 0,  -- Optimistic locking
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT chk_balance_non_negative CHECK (pending_balance >= 0)
);

CREATE INDEX idx_balance_pending ON seller_balance(pending_balance) 
WHERE pending_balance >= 10.00;

CREATE INDEX idx_balance_updated ON seller_balance(updated_at DESC);
```

**Explain:**
- "pending_balance: Money available for payout"
- "version: For optimistic locking (prevents concurrent updates)"
- "Index on pending_balance for efficient payout scheduling"
- "Constraint ensures balance never goes negative"

### 2. Payout Records Table

```sql
CREATE TABLE payout_records (
    payout_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seller_id VARCHAR(50) NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    payment_method VARCHAR(20) NOT NULL,  -- CHECK, WIRE
    status VARCHAR(20) NOT NULL,
    gateway_transaction_id VARCHAR(100),
    attempt_count INT DEFAULT 0,
    max_retries INT DEFAULT 3,
    error_message TEXT,
    idempotency_key VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    
    CONSTRAINT chk_amount_positive CHECK (amount > 0),
    CONSTRAINT chk_status_valid CHECK (status IN (
        'PENDING', 'IN_PROGRESS', 'SUCCESS', 'FAILED', 'CANCELLED'
    ))
);

CREATE INDEX idx_payout_seller ON payout_records(seller_id, created_at DESC);
CREATE INDEX idx_payout_status ON payout_records(status, created_at);
CREATE INDEX idx_payout_retry ON payout_records(status, attempt_count) 
WHERE status = 'FAILED' AND attempt_count < max_retries;
```

**Explain:**
- "idempotency_key: Prevents duplicate payouts for same seller/day"
- "attempt_count: Track retry attempts"
- "Index on (status, attempt_count) for efficient retry queries"

### 3. Audit Log Table

```sql
CREATE TABLE audit_log (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type VARCHAR(20) NOT NULL,
    seller_id VARCHAR(50) NOT NULL,
    reference_id VARCHAR(50),
    amount DECIMAL(19, 4) NOT NULL,
    balance_before DECIMAL(19, 4),
    balance_after DECIMAL(19, 4),
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT chk_event_type_valid CHECK (event_type IN (
        'SETTLEMENT', 'PAYOUT', 'REFUND', 'ADJUSTMENT'
    ))
);

CREATE INDEX idx_audit_seller ON audit_log(seller_id, created_at DESC);
CREATE INDEX idx_audit_reference ON audit_log(reference_id);
CREATE INDEX idx_audit_type_time ON audit_log(event_type, created_at DESC);

-- Partition by month for better performance
CREATE TABLE audit_log_2024_01 PARTITION OF audit_log
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

**Explain:**
- "Append-only: Never update or delete"
- "Partitioned by month for query performance"
- "JSONB metadata for flexible additional data"
- "Every transaction creates audit entry"

### 4. Idempotency Keys Table

```sql
CREATE TABLE idempotency_keys (
    idempotency_key VARCHAR(255) PRIMARY KEY,
    reference_id VARCHAR(50) NOT NULL,
    operation_type VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_idempotency_created ON idempotency_keys(created_at);

-- Auto-cleanup old keys (7 days retention)
CREATE OR REPLACE FUNCTION cleanup_old_idempotency_keys()
RETURNS void AS $$
BEGIN
    DELETE FROM idempotency_keys 
    WHERE created_at < NOW() - INTERVAL '7 days';
END;
$$ LANGUAGE plpgsql;
```

**Explain:**
- "Prevents duplicate processing of same order/payout"
- "Key format: 'settlement:{order_id}' or 'payout:{seller_id}:{date}'"
- "7-day retention (old keys auto-deleted)"

### Why This Matters:
✓ Shows understanding of financial data modeling
✓ Includes constraints for data integrity
✓ Considers indexing for performance
✓ Explains partitioning strategy

---

## PHASE 7: CORE DESIGN DECISION - PAYMENT FREQUENCY (5 minutes)

### What to Say:

"The most critical design decision is payment frequency. This directly impacts gateway fees, which dominate our costs."

### Options Analysis:

```
┌──────────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│ Frequency        │ Gateway Fees│ Complexity  │ Seller      │ Recommended │
│                  │             │             │ Satisfaction│             │
├──────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Per Order        │ $75M/month  │ Low         │ High        │ ❌          │
│ Daily Batch      │ $3.75M/month│ Medium      │ Medium      │ ✅          │
│ Weekly Batch     │ $0.5M/month │ Medium      │ Low         │ ❌          │
│ Threshold-based  │ Variable    │ High        │ Variable    │ Maybe       │
└──────────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

### Recommended: Daily Batch with Threshold

**Formula:**
```
Pay seller IF:
  - pending_balance >= $10 (threshold)
  - AND last_payout_date < today
```

**Cost Analysis:**
```
Per-Order Payments:
- 10M orders/day
- 10M transactions × $0.25 = $2.5M/day
- $75M/month
- Pros: Instant payment to sellers
- Cons: Extremely expensive

Daily Batch Payments:
- Assume 50% of sellers get orders daily
- 500K unique sellers/day
- 500K transactions × $0.25 = $125K/day
- $3.75M/month
- Pros: 95% cost savings, acceptable delay
- Cons: Sellers wait up to 24 hours

Savings: $71.25M/month (95% reduction!)
```

### Implementation Strategy:

```
Daily Batch Job (2 AM UTC):

1. Query eligible sellers:
   SELECT seller_id, pending_balance
   FROM seller_balance
   WHERE pending_balance >= 10.00
   AND (last_payout_date IS NULL 
        OR last_payout_date < CURRENT_DATE)
   ORDER BY seller_id;

2. Process in batches of 1000:
   FOR EACH batch:
     - Create payout records
     - Reserve balance (set to 0)
     - Trigger async processing

3. Parallel processing (20 workers):
   - Each worker processes payouts
   - Calls gateway API
   - Handles responses
   - Updates status

4. Duration: ~2 hours for 500K payouts
```

### Why This Approach?

```
Advantages:
✓ Massive cost savings (95%)
✓ Predictable schedule (sellers know when to expect payment)
✓ Easier reconciliation (batch reports)
✓ Reduces gateway load (spread over 2 hours vs all day)
✓ Acceptable delay (24 hours max)

Trade-offs:
✗ Not instant (sellers wait)
✗ More complex than per-order
✗ Requires batch job infrastructure

Business Justification:
- $71M/month savings funds entire engineering team
- 24-hour delay is industry standard
- Can offer "instant payout" as premium feature later
```

### Why This Matters:
✓ Shows cost awareness
✓ Demonstrates business thinking
✓ Explains trade-offs clearly
✓ Provides quantitative justification

---

## PHASE 8: DEEP DIVE - SETTLEMENT & PAYOUT FLOW (8 minutes)

### What to Say:

"Let me walk through the detailed flows for settlement and payout, including all edge cases."

### Settlement Flow (Order → Balance):

```
┌─────────────────────────────────────────────────────────────┐
│  SETTLEMENT FLOW (Per Order)                                │
│                                                               │
│  Trigger: ORDER_COMPLETED event from Order Service          │
│  Latency: < 5 seconds                                        │
│  Throughput: 350 settlements/sec (peak)                      │
│                                                               │
│  Step 1: Event Consumption                                   │
│  ├─ Kafka consumer receives ORDER_COMPLETED                 │
│  └─ Event: {order_id, seller_id, product_ids, timestamp}    │
│                                                               │
│  Step 2: Idempotency Check                                   │
│  ├─ Query: SELECT * FROM idempotency_keys                   │
│  │         WHERE idempotency_key = 'settlement:{order_id}'  │
│  ├─ If EXISTS: Return success (already processed)           │
│  └─ If NOT EXISTS: Continue                                  │
│                                                               │
│  Step 3: Calculate Seller Earnings                           │
│  ├─ Fetch products: GET /products?ids={product_ids}         │
│  ├─ Sum sellerPrice: amount = Σ(product.sellerPrice)        │
│  └─ Example: $100 + $50 = $150                              │
│                                                               │
│  Step 4: Database Transaction (ACID)                         │
│  BEGIN TRANSACTION (Isolation: READ_COMMITTED)               │
│    │                                                          │
│    ├─ 1. Update Balance (Optimistic Lock):                  │
│    │    UPDATE seller_balance                               │
│    │    SET pending_balance = pending_balance + 150.00,     │
│    │        version = version + 1,                          │
│    │        updated_at = NOW()                              │
│    │    WHERE seller_id = 'seller-456'                      │
│    │    AND version = 5;  -- Current version                │
│    │                                                          │
│    │    If rows_affected = 0:                               │
│    │      → OptimisticLockException                         │
│    │      → Retry (up to 3 times)                           │
│    │                                                          │
│    ├─ 2. Insert Audit Log:                                  │
│    │    INSERT INTO audit_log (                             │
│    │      event_type = 'SETTLEMENT',                        │
│    │      seller_id = 'seller-456',                         │
│    │      reference_id = 'order-123',                       │
│    │      amount = 150.00,                                  │
│    │      balance_before = 100.00,                          │
│    │      balance_after = 250.00                            │
│    │    );                                                   │
│    │                                                          │
│    └─ 3. Insert Idempotency Key:                            │
│         INSERT INTO idempotency_keys (                       │
│           idempotency_key = 'settlement:order-123',         │
│           reference_id = 'order-123',                       │
│           operation_type = 'SETTLEMENT'                     │
│         );                                                   │
│  COMMIT                                                       │
│                                                               │
│  Step 5: Return Success                                      │
│  └─ Kafka offset committed                                   │
│                                                               │
│  Error Handling:                                             │
│  - OptimisticLockException: Retry with backoff              │
│  - DatabaseException: Kafka will retry                       │
│  - ProductServiceDown: Retry with circuit breaker           │
└─────────────────────────────────────────────────────────────┘
```

### Payout Flow (Balance → Bank):

```
┌─────────────────────────────────────────────────────────────┐
│  PAYOUT FLOW (Daily Batch)                                  │
│                                                               │
│  Trigger: Cron at 2 AM UTC daily                            │
│  Duration: ~2 hours for 500K payouts                         │
│  Success Rate: 99%+ (with retries)                           │
│                                                               │
│  ┌────────────────────────────────────────────────────┐    │
│  │  PHASE 1: Payout Scheduling (10 minutes)           │    │
│  │                                                      │    │
│  │  1. Query eligible sellers:                         │    │
│  │     SELECT seller_id, pending_balance               │    │
│  │     FROM seller_balance                             │    │
│  │     WHERE pending_balance >= 10.00                  │    │
│  │     ORDER BY seller_id;                             │    │
│  │                                                      │    │
│  │     Result: 500K sellers                            │    │
│  │                                                      │    │
│  │  2. Process in batches of 1000:                     │    │
│  │     FOR EACH batch (500 batches total):            │    │
│  │       BEGIN TRANSACTION                             │    │
│  │         ├─ Create payout record:                    │    │
│  │         │  INSERT INTO payout_records (             │    │
│  │         │    seller_id, amount,                     │    │
│  │         │    payment_method, status,                │    │
│  │         │    idempotency_key                        │    │
│  │         │  ) VALUES (                               │    │
│  │         │    'seller-456', 250.00,                  │    │
│  │         │    'WIRE', 'PENDING',                     │    │
│  │         │    'payout:seller-456:2024-01-15'         │    │
│  │         │  );                                        │    │
│  │         │                                            │    │
│  │         ├─ Reserve balance:                         │    │
│  │         │  UPDATE seller_balance                    │    │
│  │         │  SET pending_balance = 0,                 │    │
│  │         │      last_payout_date = NOW(),            │    │
│  │         │      version = version + 1                │    │
│  │         │  WHERE seller_id = 'seller-456'           │    │
│  │         │  AND version = current_version;           │    │
│  │         │                                            │    │
│  │         └─ Audit log                                │    │
│  │       COMMIT                                         │    │
│  │                                                      │    │
│  │  3. Trigger async processing                        │    │
│  └────────────────────────────────────────────────────┘    │
│                         ↓                                    │
│  ┌────────────────────────────────────────────────────┐    │
│  │  PHASE 2: Payout Processing (110 minutes)          │    │
│  │                                                      │    │
│  │  Worker Pool: 20 workers                            │    │
│  │  Each worker processes: 25K payouts                 │    │
│  │  Rate: ~4 payouts/second per worker                 │    │
│  │                                                      │    │
│  │  For each payout:                                   │    │
│  │                                                      │    │
│  │  1. Fetch seller payment preference:                │    │
│  │     GET /sellers/{seller_id}                        │    │
│  │     → paymentDetails: {type: 'wire', ...}           │    │
│  │                                                      │    │
│  │  2. Update status to IN_PROGRESS:                   │    │
│  │     UPDATE payout_records                           │    │
│  │     SET status = 'IN_PROGRESS',                     │    │
│  │         attempt_count = attempt_count + 1           │    │
│  │     WHERE payout_id = 'payout-789';                 │    │
│  │                                                      │    │
│  │  3. Call Payment Gateway:                           │    │
│  │     IF payment_method = 'CHECK':                    │    │
│  │       txnId = gateway.sendCheck(details, amount)    │    │
│  │     ELSE:                                            │    │
│  │       txnId = gateway.sendWire(details, amount)     │    │
│  │                                                      │    │
│  │     Gateway returns: transactionId or error         │    │
│  │                                                      │    │
│  │  4. Handle Response:                                │    │
│  │     SUCCESS:                                         │    │
│  │       UPDATE payout_records                         │    │
│  │       SET status = 'IN_PROGRESS',                   │    │
│  │           gateway_transaction_id = txnId            │    │
│  │       (Wait for webhook confirmation)               │    │
│  │                                                      │    │
│  │     ERROR:                                           │    │
│  │       UPDATE payout_records                         │    │
│  │       SET status = 'FAILED',                        │    │
│  │           error_message = error                     │    │
│  │       IF attempt_count < max_retries:               │    │
│  │         Schedule retry (exponential backoff)        │    │
│  │       ELSE:                                          │    │
│  │         Move to manual intervention queue           │    │
│  └────────────────────────────────────────────────────┘    │
│                         ↓                                    │
│  ┌────────────────────────────────────────────────────┐    │
│  │  PHASE 3: Webhook Processing (Async)               │    │
│  │                                                      │    │
│  │  Gateway sends webhook:                             │    │
│  │  POST /webhooks/payment-success                     │    │
│  │  {                                                   │    │
│  │    "transaction_id": "txn-abc123",                  │    │
│  │    "status": "success",                             │    │
│  │    "completed_at": "2024-01-15T02:01:30Z"           │    │
│  │  }                                                   │    │
│  │                                                      │    │
│  │  Update payout:                                     │    │
│  │  UPDATE payout_records                              │    │
│  │  SET status = 'SUCCESS',                            │    │
│  │      completed_at = NOW()                           │    │
│  │  WHERE gateway_transaction_id = 'txn-abc123';       │    │
│  │                                                      │    │
│  │  Audit log                                          │    │
│  └────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Why This Matters:
✓ Shows detailed understanding
✓ Includes error handling
✓ Demonstrates transaction management
✓ Explains async processing


## PHASE 9: PREVENTING DOUBLE PAYOUTS (5 minutes)

### What to Say:

"Preventing double payouts is critical since this is a financial system. I'll implement multiple layers of protection."

### Multi-Layer Defense Strategy:

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 1: Idempotency Keys                                  │
│                                                               │
│  Key Format: "payout:{seller_id}:{date}"                    │
│  Example: "payout:seller-456:2024-01-15"                    │
│                                                               │
│  CREATE UNIQUE INDEX idx_payout_idempotency                 │
│  ON payout_records(idempotency_key);                        │
│                                                               │
│  Protection: Database constraint prevents duplicate inserts │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  LAYER 2: Optimistic Locking                                │
│                                                               │
│  UPDATE seller_balance                                       │
│  SET pending_balance = 0,                                    │
│      version = version + 1                                   │
│  WHERE seller_id = 'seller-456'                              │
│  AND version = 5;  -- Must match current version            │
│                                                               │
│  If rows_affected = 0:                                       │
│    → Concurrent update detected                              │
│    → Transaction rolled back                                 │
│    → No double payout                                        │
│                                                               │
│  Protection: Prevents concurrent balance updates            │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  LAYER 3: State Machine Enforcement                         │
│                                                               │
│  Allowed Transitions:                                        │
│  PENDING → IN_PROGRESS → SUCCESS (terminal)                 │
│         ↓                    ↓                               │
│      CANCELLED            FAILED → RETRY                     │
│                                                               │
│  Never retry from SUCCESS state                              │
│                                                               │
│  Protection: Prevents reprocessing completed payouts        │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  LAYER 4: Distributed Lock                                  │
│                                                               │
│  RLock lock = redisson.getLock("payout:lock:" + sellerId);  │
│  try {                                                        │
│    if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {            │
│      // Process payout                                       │
│    }                                                          │
│  } finally {                                                  │
│    lock.unlock();                                            │
│  }                                                            │
│                                                               │
│  Protection: Prevents concurrent payout processing          │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  LAYER 5: Gateway Idempotency                               │
│                                                               │
│  gateway.sendWire(                                           │
│    details,                                                   │
│    amount,                                                    │
│    idempotencyKey: payout_id  // Gateway deduplicates       │
│  );                                                           │
│                                                               │
│  Gateway returns same response for duplicate requests       │
│                                                               │
│  Protection: Gateway-level deduplication                     │
└─────────────────────────────────────────────────────────────┘
```

### Scenario Testing:

**Scenario 1: Scheduler Runs Twice Accidentally**
```
Time T1: Scheduler Run 1
  - Finds seller with $100 balance
  - Creates payout P1
  - Sets balance to $0 (version 5 → 6)

Time T2: Scheduler Run 2 (accidental)
  - Finds seller with $0 balance
  - No payout created (balance < threshold)
  ✓ Protected by balance check
```

**Scenario 2: Concurrent Processing**
```
Thread 1                    Thread 2
├─ Read balance = $100      ├─ Read balance = $100
├─ Create payout P1         ├─ Create payout P2
├─ UPDATE ... version=5     ├─ UPDATE ... version=5
│  ✓ Success (v=6)          │  ✗ Fails (version mismatch)
│                           │  Transaction rolled back
│                           ✓ Protected by optimistic lock
```

**Scenario 3: Retry After Success**
```
Payout P1: status = SUCCESS
  ↓
Retry triggered (bug/manual)
  ↓
State machine check:
  if (payout.status == SUCCESS) {
    return "Already completed";
  }
  ✓ Protected by state machine
```

**Scenario 4: Duplicate Gateway Call**
```
Call 1: gateway.sendWire(idempotencyKey: "payout-123")
  → Returns: txn-abc

Call 2: gateway.sendWire(idempotencyKey: "payout-123")
  → Returns: txn-abc (same transaction)
  ✓ Protected by gateway idempotency
```

### Why This Matters:
✓ Shows defense-in-depth thinking
✓ Demonstrates financial system expertise
✓ Explains each layer's purpose
✓ Tests scenarios explicitly

---

## PHASE 10: FOLLOW-UP QUESTIONS (3 minutes)

### Follow-up 1: Order Cancellations

**Question:** "How would you handle order cancellations?"

**Answer:**

```
Scenario 1: Order cancelled BEFORE settlement
  - Order Service publishes ORDER_CANCELLED event
  - Settlement Service ignores (no settlement created)
  - No action needed

Scenario 2: Order cancelled AFTER settlement, BEFORE payout
  - Deduct from seller balance:
    UPDATE seller_balance
    SET pending_balance = pending_balance - amount
    WHERE seller_id = seller_id;
  
  - Create audit log (type = REFUND)
  - Balance can go negative (deducted from next payout)

Scenario 3: Order cancelled AFTER payout
  - Create negative balance:
    UPDATE seller_balance
    SET pending_balance = pending_balance - amount;
  
  - Next payout will deduct this amount:
    IF pending_balance < 0:
      Skip payout
    ELSE IF pending_balance < threshold:
      Skip payout
    ELSE:
      Payout (pending_balance)
```

### Follow-up 2: Gateway Downtime

**Question:** "How would you deal with gateway downtime?"

**Answer:**

```
Detection:
- Circuit breaker pattern
- After 5 consecutive failures, open circuit
- Stop sending requests for 5 minutes

Response:
1. Mark payouts as DEFERRED
2. Alert operations team
3. Stop processing new payouts
4. Wait for circuit breaker timeout

Recovery:
1. Circuit breaker attempts test request
2. If success: Close circuit, resume processing
3. If failure: Keep circuit open, wait longer

Graceful Degradation:
- Sellers can still query balance (read-only)
- Settlements continue (balance updates)
- Payouts queued for later processing
- No data loss
```

### Follow-up 3: Monitoring & Alerting

**Question:** "What monitoring would you set up?"

**Answer:**

```
Key Metrics:

1. Settlement Metrics:
   - Settlements processed/sec
   - Settlement latency (p50, p95, p99)
   - Settlement error rate
   - Alert: Error rate > 1%

2. Payout Metrics:
   - Payouts initiated/day
   - Payouts succeeded/day
   - Payout failure rate
   - Alert: Failure rate > 5%

3. Balance Metrics:
   - Total pending balance
   - Sellers with negative balance
   - Alert: Negative balance count > 100

4. Gateway Metrics:
   - Gateway latency
   - Gateway error rate
   - Circuit breaker state
   - Alert: Circuit breaker open

5. Job Metrics:
   - Batch job duration
   - Batch job success rate
   - Alert: Job duration > 3 hours
   - Alert: Job failed

Dashboard:
┌─────────────────────────────────────────────┐
│ Settlements Today:        125,432           │
│ Payouts Today:             45,678           │
│ Success Rate:              99.2%            │
│ Failed Payouts:               365           │
│ Pending Balance:      $12.5M                │
│ Gateway Latency:       1.2s (p99)           │
│ Circuit Breaker:       CLOSED ✓             │
└─────────────────────────────────────────────┘
```

### Follow-up 4: Job Failure Scenarios

**Question:** "What if the batch job fails midway?"

**Answer:**

```
Scenario 1: Job fails to start
- Detection: Monitor last run time
- Alert: If last_run > 25 hours
- Recovery: Manual trigger via admin API

Scenario 2: Job fails after processing some records
- Checkpointing: Track last processed seller_id
- Resume: Start from last checkpoint
- Idempotency: Safe to reprocess (idempotency keys)

Scenario 3: Job fails during payout processing
- Atomic transactions: Each payout is atomic
- Retry: Failed payouts marked for retry
- No partial state: Either fully processed or not

Implementation:
CREATE TABLE job_executions (
  execution_id UUID PRIMARY KEY,
  start_time TIMESTAMP,
  end_time TIMESTAMP,
  status VARCHAR(20),
  last_processed_seller_id VARCHAR(50),
  sellers_processed INT,
  sellers_total INT
);

Resume logic:
SELECT last_processed_seller_id 
FROM job_executions
WHERE status = 'FAILED'
ORDER BY start_time DESC
LIMIT 1;

Continue from: seller_id > last_processed_seller_id
```

### Why This Matters:
✓ Shows production readiness
✓ Demonstrates failure handling
✓ Considers operational aspects
✓ Provides concrete solutions

---

## SUMMARY: EVALUATION CRITERIA MAPPING

### How This Solution Scores:

```
┌──────────────────────────┬─────────────────────────┬──────────────┐
│ Evaluation Element       │ Evidence in Solution    │ Rating       │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Workable Solution        │ Complete end-to-end     │ Outstanding  │
│                          │ All requirements met    │              │
│                          │ Production-ready        │              │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Clarifying Questions     │ Asked 8 key questions   │ Outstanding  │
│                          │ Identified key decision │              │
│                          │ (payment frequency)     │              │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Efficiency               │ Batch processing        │ Outstanding  │
│                          │ 95% cost savings        │              │
│                          │ Optimistic locking      │              │
│                          │ Parallel processing     │              │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Simplicity               │ Clear separation        │ Outstanding  │
│                          │ Straightforward flow    │              │
│                          │ Not over-engineered     │              │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Quality of Abstraction   │ Event-driven            │ Outstanding  │
│                          │ Service separation      │              │
│                          │ State machine           │              │
│                          │ Multi-layer protection  │              │
└──────────────────────────┴─────────────────────────┴──────────────┘

Overall Rating: Top 20% (Above the Bar)
```

### What Makes This Top 20%:

1. **Identified the key decision** (payment frequency) with cost analysis
2. **Strong consistency** for money operations (ACID, optimistic locking)
3. **Multi-layer protection** against double payouts (5 layers)
4. **Quantitative thinking** (back-of-envelope calculations)
5. **Production-ready** (monitoring, failure handling, reconciliation)
6. **Addressed all requirements** in priority order
7. **Comprehensive follow-ups** (cancellations, downtime, monitoring, failures)
8. **Clear trade-offs** (batch vs per-order, cost vs latency)

---

## INTERVIEW TIPS

### Time Management:

```
0-5 min:   Clarifying questions
5-8 min:   Requirements + Estimation
8-16 min:  High-level architecture
16-19 min: API design
19-24 min: Data models
24-29 min: Payment frequency decision
29-37 min: Settlement & payout flows
37-42 min: Preventing double payouts
42-45 min: Follow-up questions
```

### What to Emphasize:

1. **Payment frequency is THE key decision** - Mention cost savings
2. **Strong consistency for money** - ACID, optimistic locking
3. **Multi-layer protection** - Defense in depth
4. **Audit log is mandatory** - Every transaction traceable
5. **Idempotency everywhere** - No duplicates

### Common Pitfalls to Avoid:

❌ Not asking about payment frequency
❌ Using eventual consistency for money
❌ Single layer of duplicate protection
❌ Ignoring gateway fees
❌ Not considering failure scenarios
❌ Forgetting audit log requirement

### Strong Signals to Send:

✓ "The key decision is payment frequency - let me analyze costs"
✓ "Money requires strong consistency, not eventual"
✓ "I'll implement multiple layers to prevent double payouts"
✓ "Every transaction must be auditable"
✓ "Here's how we handle gateway downtime"

---

## FINAL CHECKLIST

Before ending the interview, ensure you've covered:

- [ ] Asked clarifying questions (especially payment frequency)
- [ ] Defined functional requirements (in priority order)
- [ ] Defined non-functional requirements (strong consistency!)
- [ ] Did back-of-envelope estimation (cost analysis)
- [ ] Drew high-level architecture
- [ ] Designed APIs (with status messages)
- [ ] Defined data models (with constraints)
- [ ] Explained payment frequency decision
- [ ] Detailed settlement flow
- [ ] Detailed payout flow
- [ ] Explained double payout prevention (5 layers)
- [ ] Addressed order cancellations
- [ ] Discussed gateway downtime
- [ ] Mentioned monitoring & alerting
- [ ] Explained job failure handling

This comprehensive approach demonstrates **Top 20%** performance! 🎯
