# Seller Payment System - HLD Flow Diagram

## Flow 1: Settlement Flow (Order Completion → Balance Update)

```mermaid
sequenceDiagram
    participant Order as Order Service
    participant Kafka as Kafka
    participant Settlement as Settlement Service
    participant Idempotency as Idempotency Table
    participant Balance as Seller Balance Table
    participant Audit as Audit Log
    
    Order->>Kafka: Publish ORDER_COMPLETED event<br/>(order_id, seller_id, amount)
    Kafka->>Settlement: Consume event
    
    Settlement->>Idempotency: Check if order_id processed
    alt Already Processed
        Idempotency-->>Settlement: Duplicate detected
        Settlement-->>Kafka: Skip (idempotent)
    else New Order
        Idempotency-->>Settlement: Not found
        
        Settlement->>Settlement: Calculate seller earnings<br/>(amount - commission)
        
        Settlement->>Balance: UPDATE with optimistic lock<br/>SET pending_balance += earnings<br/>WHERE seller_id = ? AND version = ?
        
        alt Lock Success
            Balance-->>Settlement: Updated (version++)
            Settlement->>Audit: INSERT settlement record<br/>(event_type=SETTLEMENT)
            Settlement->>Idempotency: INSERT order_id
            Settlement-->>Kafka: ACK
        else Lock Failure
            Balance-->>Settlement: Version mismatch
            Settlement->>Settlement: Retry with backoff
        end
    end
```

## Flow 2: Payout Initiation Flow (Daily Scheduler)

```mermaid
sequenceDiagram
    participant Scheduler as Payout Scheduler<br/>(2 AM UTC)
    participant Balance as Seller Balance Table
    participant Payout as Payout Records Table
    participant Queue as Payout Queue
    
    Scheduler->>Scheduler: Trigger daily job
    
    Scheduler->>Balance: SELECT seller_id, pending_balance<br/>WHERE pending_balance >= 10<br/>AND last_payout_date < today
    Balance-->>Scheduler: Return eligible sellers<br/>(~500K sellers)
    
    loop For each seller batch (1000 sellers)
        Scheduler->>Balance: BEGIN TRANSACTION
        Scheduler->>Balance: UPDATE seller_balance<br/>SET pending_balance = 0,<br/>last_payout_date = today<br/>WHERE seller_id = ? AND version = ?
        
        alt Lock Success
            Balance-->>Scheduler: Balance reserved
            Scheduler->>Payout: INSERT payout record<br/>(payout_id, seller_id, amount,<br/>status=PENDING)
            Scheduler->>Balance: COMMIT
            Scheduler->>Queue: Enqueue payout_id
        else Lock Failure
            Balance-->>Scheduler: Conflict
            Scheduler->>Balance: ROLLBACK
            Scheduler->>Scheduler: Log error & continue
        end
    end
    
    Scheduler-->>Scheduler: Job complete (2 hours)
```

## Flow 3: Payout Processing Flow (Worker Pool)

```mermaid
sequenceDiagram
    participant Queue as Payout Queue
    participant Worker as Payout Worker
    participant Seller as Seller Service
    participant Payout as Payout Records Table
    participant Gateway as Payment Gateway
    participant Audit as Audit Log
    
    Queue->>Worker: Dequeue payout_id
    
    Worker->>Payout: SELECT * FROM payout_records<br/>WHERE payout_id = ?
    Payout-->>Worker: Return payout details
    
    Worker->>Seller: GET /sellers/{seller_id}
    Seller-->>Worker: Return payment preference<br/>(check/wire + details)
    
    Worker->>Payout: UPDATE status = PROCESSING
    
    Worker->>Gateway: POST /sendCheck or /sendWire<br/>(amount, payment_details)
    
    alt Gateway Success
        Gateway-->>Worker: 200 OK<br/>{transaction_id: "TXN123"}
        Worker->>Payout: UPDATE status = COMPLETED,<br/>gateway_txn_id = "TXN123"
        Worker->>Audit: INSERT payout success event
        
    else Gateway Failure (4xx/5xx)
        Gateway-->>Worker: Error response
        Worker->>Payout: UPDATE status = FAILED,<br/>attempt_count++,<br/>error_message
        
        alt Retry Eligible (attempt < 3)
            Worker->>Queue: Re-enqueue with delay<br/>(exponential backoff)
        else Max Retries
            Worker->>Audit: INSERT payout failed event
            Worker->>Worker: Alert operations team
        end
    end
```

## Flow 4: Gateway Webhook Callback Flow

```mermaid
sequenceDiagram
    participant Gateway as Payment Gateway
    participant Webhook as Webhook Endpoint
    participant Payout as Payout Records Table
    participant Audit as Audit Log
    participant Notification as Notification Service
    
    Note over Gateway: Payment processed<br/>(~1 minute later)
    
    Gateway->>Webhook: POST /webhooks/payment-success<br/>{transaction_id, status}
    
    Webhook->>Webhook: Verify signature
    
    Webhook->>Payout: SELECT * FROM payout_records<br/>WHERE gateway_txn_id = ?
    Payout-->>Webhook: Return payout record
    
    alt Success Webhook
        Webhook->>Payout: UPDATE status = SETTLED,<br/>settled_at = now()
        Webhook->>Audit: INSERT settlement confirmation
        Webhook->>Notification: Send seller notification<br/>(email/SMS)
        Webhook-->>Gateway: 200 OK
        
    else Failure Webhook
        Webhook->>Payout: UPDATE status = GATEWAY_FAILED,<br/>error_message
        Webhook->>Audit: INSERT gateway failure event
        Webhook->>Notification: Alert operations team
        Webhook-->>Gateway: 200 OK
    end
```

## Flow 5: Reconciliation Flow (Daily 3 AM)

```mermaid
sequenceDiagram
    participant Recon as Reconciliation Service
    participant Gateway as Payment Gateway
    participant Payout as Payout Records Table
    participant Audit as Audit Log
    participant Alert as Alert System
    
    Recon->>Recon: Trigger daily job (3 AM)
    
    Recon->>Gateway: GET /reports/daily<br/>(date = yesterday)
    Gateway-->>Recon: Return transaction report<br/>(CSV/JSON)
    
    Recon->>Payout: SELECT * FROM payout_records<br/>WHERE created_at = yesterday
    Payout-->>Recon: Return internal records
    
    Recon->>Recon: Match records by transaction_id
    
    loop For each record
        alt Match Found
            Recon->>Recon: Verify amount matches
            alt Amount Mismatch
                Recon->>Alert: Send discrepancy alert
                Recon->>Audit: Log mismatch
            end
        else Missing in Gateway
            Recon->>Alert: Send missing transaction alert
            Recon->>Audit: Log missing transaction
        else Extra in Gateway
            Recon->>Alert: Send unknown transaction alert
            Recon->>Audit: Log unknown transaction
        end
    end
    
    Recon->>Recon: Generate reconciliation report
    Recon->>Audit: INSERT reconciliation summary
```

## Flow 6: Balance Query Flow (API)

```mermaid
sequenceDiagram
    participant Client as Client/UI
    participant API as Payment Status API
    participant Redis as Redis Cache
    participant Balance as Seller Balance Table
    participant Payout as Payout Records Table
    
    Client->>API: GET /sellers/{seller_id}/balance
    
    API->>Redis: GET cache key<br/>("balance:{seller_id}")
    
    alt Cache Hit
        Redis-->>API: Return cached balance
        API-->>Client: 200 OK {balance, last_updated}
        
    else Cache Miss
        Redis-->>API: NULL
        API->>Balance: SELECT * FROM seller_balance<br/>WHERE seller_id = ?
        Balance-->>API: Return balance data
        API->>Redis: SET cache (TTL=60s)
        API-->>Client: 200 OK {balance, last_updated}
    end
    
    Note over Client,Payout: Payout History Query
    
    Client->>API: GET /sellers/{seller_id}/payouts
    API->>Payout: SELECT * FROM payout_records<br/>WHERE seller_id = ?<br/>ORDER BY created_at DESC<br/>LIMIT 50
    Payout-->>API: Return payout history
    API-->>Client: 200 OK [{payout_id, amount, status}]
```

## Component Interaction Summary

```mermaid
graph LR
    A[Order Service] -->|Event| B[Kafka]
    B -->|Consume| C[Settlement Service]
    C -->|Write| D[(Database)]
    
    E[Scheduler] -->|Query| D
    E -->|Trigger| F[Payout Workers]
    F -->|API Call| G[Payment Gateway]
    G -->|Webhook| F
    
    H[Reconciliation] -->|Compare| D
    H -->|Fetch Report| G
    
    I[Status API] -->|Read| D
    I -->|Cache| J[(Redis)]
    
    K[Monitoring] -.->|Observe| C
    K -.->|Observe| F
    K -.->|Observe| E
    
    style A fill:#e1f5ff
    style C fill:#fff4e1
    style E fill:#ffe1f5
    style F fill:#f5ffe1
    style G fill:#ffe1e1
    style H fill:#e1ffe1
```

## Key Flow Characteristics

| Flow | Frequency | Throughput | Latency |
|------|-----------|------------|---------|
| Settlement | Real-time | 350/sec | <100ms |
| Payout Initiation | Daily 2 AM | 500K in 2h | N/A |
| Payout Processing | Continuous | 25/sec | ~1 min |
| Reconciliation | Daily 3 AM | 500K records | ~30 min |
| Balance Query | On-demand | Low | <50ms |
