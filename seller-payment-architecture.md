# Seller Payment System - High Level Architecture

```mermaid
graph TB
    subgraph EXISTING["EXISTING SERVICES"]
        SellerSvc["Seller Service<br/>- seller_id<br/>- name<br/>- payment details"]
        ProductSvc["Product Service<br/>- product_id<br/>- seller_id<br/>- sellerPrice<br/>- buyerPrice"]
        OrderSvc["Order Service<br/>- order_id<br/>- buyer_id<br/>- products[]<br/>- timestamp"]
    end

    subgraph PAYMENT["SELLER PAYMENT SYSTEM"]
        subgraph Settlement["1. SETTLEMENT SERVICE"]
            SettlementSvc["Event Consumer<br/>- Consumes ORDER_COMPLETED<br/>- Calculates earnings<br/>- Updates balance<br/>- Audit logging<br/>- Idempotency<br/><br/>Java/Spring Boot<br/>10 instances<br/>350 settlements/sec"]
        end

        subgraph Database["2. DATABASE LAYER (PostgreSQL)"]
            BalanceTable["seller_balance<br/>- seller_id PK<br/>- pending_balance<br/>- last_payout_date<br/>- version"]
            PayoutTable["payout_records<br/>- payout_id PK<br/>- seller_id<br/>- amount, status<br/>- gateway_txn_id<br/>- attempt_count"]
            AuditTable["audit_log<br/>- event_id PK<br/>- event_type<br/>- seller_id, amount<br/>- reference_id"]
            IdempotencyTable["idempotency_keys<br/>- idempotency_key PK<br/>- reference_id<br/>- created_at"]
        end

        subgraph Scheduler["3. PAYOUT SCHEDULER"]
            CronJob["Cron Job<br/>- Runs daily 2 AM UTC<br/>- Queries balance ≥ $10<br/>- Creates payout records<br/>- Reserves balance<br/><br/>AWS EventBridge + Airflow<br/>~2 hours for 500K payouts"]
        end

        subgraph Processor["4. PAYOUT PROCESSOR"]
            WorkerPool["Worker Pool<br/>- Fetches payment preference<br/>- Calls gateway API<br/>- Handles callbacks<br/>- Retry + Circuit breaker<br/><br/>Java/Spring Boot<br/>20 workers<br/>25 payouts/sec"]
        end

        subgraph API["5. PAYMENT STATUS API"]
            RestAPI["REST Endpoints<br/>- GET /sellers/{id}/balance<br/>- GET /sellers/{id}/payouts<br/>- GET /payouts/{id}/status<br/><br/>Java/Spring Boot<br/>5 instances<br/>Redis cache"]
        end

        subgraph Recon["6. RECONCILIATION SERVICE"]
            ReconSvc["Daily Job<br/>- Runs at 3 AM<br/>- Fetches gateway report<br/>- Matches records<br/>- Alerts discrepancies<br/>- Generates report"]
        end
    end

    subgraph Gateway["THIRD PARTY PAYMENT GATEWAY"]
        GatewayAPI["API<br/>- sendCheck()<br/>- sendWire()<br/><br/>Webhooks<br/>- payment-success<br/>- payment-failed<br/><br/>~1 min processing<br/>$0.25 per transfer<br/>100 req/sec limit"]
    end

    subgraph Infrastructure["SUPPORTING INFRASTRUCTURE"]
        Kafka["Kafka<br/>(Events)"]
        Redis["Redis<br/>(Cache)"]
        Monitor["DataDog<br/>(Monitoring)"]
    end

    OrderSvc -->|ORDER_COMPLETED event| Kafka
    Kafka -->|consume| SettlementSvc
    SettlementSvc -->|update| BalanceTable
    SettlementSvc -->|write| AuditTable
    SettlementSvc -->|check| IdempotencyTable
    
    CronJob -->|query| BalanceTable
    CronJob -->|create| PayoutTable
    CronJob -->|trigger| WorkerPool
    
    WorkerPool -->|fetch| SellerSvc
    WorkerPool -->|update| PayoutTable
    WorkerPool -->|call| GatewayAPI
    GatewayAPI -->|webhook| WorkerPool
    
    RestAPI -->|read| BalanceTable
    RestAPI -->|read| PayoutTable
    RestAPI -->|cache| Redis
    
    ReconSvc -->|fetch report| GatewayAPI
    ReconSvc -->|compare| PayoutTable
    
    Monitor -.->|monitor| SettlementSvc
    Monitor -.->|monitor| WorkerPool
    Monitor -.->|monitor| Database

    style EXISTING fill:#e1f5ff
    style PAYMENT fill:#fff4e1
    style Gateway fill:#ffe1e1
    style Infrastructure fill:#e1ffe1
```

## Flow Description

### Settlement Flow (Order → Balance)
1. **Order Service** publishes `ORDER_COMPLETED` event to Kafka
2. **Settlement Service** consumes event and:
   - Validates idempotency
   - Calculates seller earnings
   - Updates `seller_balance` with optimistic locking
   - Writes to `audit_log`

### Payout Flow (Balance → Bank)
1. **Payout Scheduler** (daily 2 AM):
   - Queries sellers with balance ≥ $10
   - Creates records in `payout_records`
   - Reserves balance
2. **Payout Processor**:
   - Fetches seller payment preference
   - Calls payment gateway API
   - Handles async callbacks
   - Updates payout status
3. **Reconciliation Service** (daily 3 AM):
   - Matches gateway reports with internal records
   - Alerts on discrepancies

## Key Characteristics
- **Throughput**: 350 settlements/sec, 25 payouts/sec
- **Scale**: 500K payouts in 2 hours
- **Database**: PostgreSQL Multi-AZ with 2 read replicas
- **Reliability**: Idempotency, optimistic locking, retry logic, circuit breaker
