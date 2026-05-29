# E-Commerce Popularity-Based Browsing System - High Level Design

## INTERVIEW CONTEXT

**Question:** Design a backend system for an e-commerce website that shows a list of merchandise. Focus on browsing (not purchases). Use popularity to select top items. Build an offline batching pipeline.

**Follow-ups:** 
1. Build real-time pipeline for "hot" items
2. Add personalization

**Evaluation Criteria:**
- Workable Solution
- Asks Clarifying Questions
- Efficiency
- Simplicity (senior level)
- Quality of Abstraction (senior level)

---

## 1. REQUIREMENTS CLARIFICATION

**CRITICAL: Start interview by asking these questions!**

### 1.1 Functional Requirements

**Core Features:**
- Display top/popular items to users
- Support category-wise and global popularity rankings
- Handle millions of products (scale: 10M+ products)
- Paginated results (50-100 items per page)
- Batch computation for popularity scores (daily/hourly updates)
- Real-time "hot items" tracking (last 10 minutes)

**Popularity Definition:**
Popularity is a weighted composite score derived from multiple user engagement signals with time decay.

**Out of Scope:**
- Payment processing
- Order fulfillment
- Inventory management
- Returns/refunds
- Seller payouts

### 1.2 Non-Functional Requirements

- **Latency:** < 100ms for API responses (p99)
- **Availability:** 99.99% uptime
- **Consistency:** Eventual consistency acceptable for rankings
- **Accuracy:** Approximate results acceptable (95%+ accuracy)
- **Freshness:** 
  - Batch rankings: 1-hour staleness acceptable
  - Hot items: < 1 minute staleness
- **Scale:**
  - 10M products
  - 100M daily active users
  - 1B events/day
  - 10K requests/second (peak)

---

## 2. POPULARITY SCORE DEFINITION

### 2.1 Composite Score Formula

```
popularity_score = 
    w1 * log(views + 1)
  + w2 * clicks
  + w3 * add_to_cart * 5
  + w4 * purchases * 10
  + w5 * recency_decay
  + w6 * conversion_rate
```

### 2.2 Signal Weights

| Signal | Weight | Rationale |
|--------|--------|-----------|
| Views | 0.1 | Base engagement indicator |
| Clicks | 0.3 | Strong interest signal |
| Add-to-cart | 1.5 | Purchase intent |
| Purchases | 3.0 | Strongest conversion signal |
| Recency decay | 0.5 | Time-based relevance |
| Conversion rate | 1.0 | Quality indicator |

### 2.3 Recency Decay Function

```
recency_decay = e^(-λ * days_since_event)
where λ = 0.1 (decay constant)
```

This ensures recent activity weighs more than old activity.

---

## 3. HIGH-LEVEL ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                             │
│  (Web App, Mobile App, API Consumers)                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API GATEWAY / CDN                           │
│  (Rate Limiting, Auth, Routing, Caching)                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SERVING LAYER (API)                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Popular Items│  │  Hot Items   │  │  Product     │         │
│  │   Service    │  │   Service    │  │  Metadata    │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      CACHE LAYER                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Redis Cluster (Sorted Sets for Rankings)               │  │
│  │  - Key: category:{id}:popular                            │  │
│  │  - Key: global:popular                                   │  │
│  │  - Key: hot:last_10min                                   │  │
│  │  - TTL: 1 hour for batch, 1 min for hot                 │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE LAYER                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  DynamoDB    │  │ ClickHouse   │  │  PostgreSQL  │         │
│  │  (Rankings)  │  │  (Analytics) │  │  (Products)  │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                         ▲
                         │
┌─────────────────────────────────────────────────────────────────┐
│                  DATA PROCESSING LAYER                           │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │           BATCH PIPELINE (Hourly/Daily)                │    │
│  │                                                          │    │
│  │  S3/HDFS → Spark Jobs → Aggregate → Score → Store      │    │
│  │                                                          │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │         REAL-TIME PIPELINE (Streaming)                  │    │
│  │                                                          │    │
│  │  Kafka → Flink → Sliding Window → TopK → Redis         │    │
│  │                                                          │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                         ▲
                         │
┌─────────────────────────────────────────────────────────────────┐
│                   EVENT INGESTION LAYER                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Kafka Cluster (Partitioned by product_id)              │  │
│  │  Topics: user-views, user-clicks, cart-events,          │  │
│  │          purchase-events                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                         ▲
                         │
┌─────────────────────────────────────────────────────────────────┐
│                    EVENT PRODUCERS                               │
│  (Frontend, Mobile App, Backend Services)                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. DETAILED COMPONENT DESIGN

### 4.1 Event Ingestion Layer

**Event Schema:**
```json
{
  "event_id": "uuid",
  "event_type": "view|click|add_to_cart|purchase",
  "product_id": "string",
  "user_id": "string",
  "category_id": "string",
  "timestamp": "epoch_ms",
  "session_id": "string",
  "metadata": {
    "price": "decimal",
    "device": "mobile|web|app"
  }
}
```

**Kafka Configuration:**
- **Topics:** 4 separate topics (views, clicks, cart, purchases)
- **Partitions:** 100 partitions per topic (partition by product_id)
- **Replication Factor:** 3
- **Retention:** 7 days
- **Compression:** Snappy

**Event Producers:**
- Frontend: JavaScript SDK emits events on user actions
- Mobile App: Native SDK with batching (every 5 seconds)
- Backend: Purchase service publishes to Kafka on order completion

---

### 4.2 Batch Processing Pipeline

**Purpose:** Compute popularity scores for all products periodically

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│  BATCH PIPELINE (Runs every hour)                           │
│                                                               │
│  Step 1: Data Ingestion                                      │
│  ├─ Kafka → S3 (via Kafka Connect)                          │
│  └─ Parquet format, partitioned by date/hour                │
│                                                               │
│  Step 2: Aggregation (Spark Job)                            │
│  ├─ Read last 24 hours of events                            │
│  ├─ Group by product_id                                     │
│  ├─ Count: views, clicks, add_to_cart, purchases           │
│  └─ Apply time decay based on event timestamp               │
│                                                               │
│  Step 3: Score Calculation                                   │
│  ├─ Apply weighted formula                                  │
│  ├─ Normalize scores (0-100 range)                          │
│  └─ Rank products globally and per category                 │
│                                                               │
│  Step 4: Storage                                             │
│  ├─ Write to ClickHouse (for analytics)                     │
│  ├─ Write to DynamoDB (for serving)                         │
│  └─ Update Redis cache (sorted sets)                        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Spark Job Pseudocode:**

```scala
// Read events from S3
val events = spark.read.parquet("s3://events/date=2024-01-*")

// Aggregate by product
val aggregated = events
  .filter($"timestamp" > now() - 24.hours)
  .groupBy("product_id", "category_id")
  .agg(
    count(when($"event_type" === "view", 1)).as("views"),
    count(when($"event_type" === "click", 1)).as("clicks"),
    count(when($"event_type" === "add_to_cart", 1)).as("carts"),
    count(when($"event_type" === "purchase", 1)).as("purchases")
  )

// Calculate popularity score
val scored = aggregated.map { row =>
  val recencyDecay = calculateDecay(row.timestamp)
  val score = 
    0.1 * log(row.views + 1) +
    0.3 * row.clicks +
    1.5 * row.carts +
    3.0 * row.purchases +
    0.5 * recencyDecay
  
  (row.product_id, row.category_id, score)
}

// Rank and store
scored
  .groupBy("category_id")
  .sortBy("score", descending = true)
  .write
  .mode("overwrite")
  .to("dynamodb://popularity_rankings")
```

**Storage Schema (DynamoDB):**

Table: `popularity_rankings`
- Partition Key: `category_id` (or "GLOBAL" for global rankings)
- Sort Key: `score` (numeric, descending)
- Attributes: `product_id`, `rank`, `score`, `updated_at`
- GSI: `product_id` (for reverse lookup)

---

### 4.3 Real-Time Hot Items Pipeline

**Purpose:** Track trending items in the last 10 minutes

**Architecture:**

```
Kafka → Flink Streaming Job → Redis TopK
         (Sliding Window)
```

**Flink Job Design:**

```java
// Flink streaming job
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<Event> events = env
  .addSource(new FlinkKafkaConsumer<>("user-events", schema, props));

// Sliding window: 10 min window, 1 min slide
DataStream<ProductCount> hotItems = events
  .keyBy(event -> event.getProductId())
  .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.minutes(1)))
  .aggregate(new CountAggregator())
  .filter(count -> count.getCount() > THRESHOLD);

// Write to Redis using TopK data structure
hotItems.addSink(new RedisSink<>(redisConfig));
```

**Redis TopK Configuration:**
- Data Structure: `TOPK.RESERVE hot_items 1000 50 7 0.925`
- Key: `hot:last_10min`
- TTL: 15 minutes
- Update Frequency: Every 1 minute

**Why TopK?**
- Space-efficient: O(k) memory instead of O(n)
- Approximate but accurate enough (95%+)
- Fast updates: O(log k) per operation

---

### 4.4 Serving Layer (API Design)

**API Endpoints:**

#### 4.4.1 Get Popular Items

```
GET /api/v1/products/popular

Query Parameters:
- category_id (optional): Filter by category
- page: Page number (default: 1)
- limit: Items per page (default: 50, max: 100)
- time_range: 24h|7d|30d (default: 24h)

Response:
{
  "data": [
    {
      "product_id": "p123",
      "name": "Product Name",
      "price": 29.99,
      "image_url": "https://cdn.../image.jpg",
      "popularity_score": 87.5,
      "rank": 1
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 1000,
    "has_next": true
  },
  "metadata": {
    "updated_at": "2024-01-15T10:00:00Z",
    "cache_hit": true
  }
}
```

#### 4.4.2 Get Hot Items

```
GET /api/v1/products/hot

Query Parameters:
- limit: Number of items (default: 20, max: 100)

Response:
{
  "data": [
    {
      "product_id": "p456",
      "name": "Trending Product",
      "event_count": 15420,
      "trend_score": 95.2
    }
  ],
  "metadata": {
    "window": "last_10_minutes",
    "updated_at": "2024-01-15T10:05:00Z"
  }
}
```

**Service Implementation (Pseudocode):**

```java
@Service
public class PopularItemsService {
    
    @Autowired
    private RedisTemplate<String, String> redis;
    
    @Autowired
    private DynamoDBClient dynamodb;
    
    @Autowired
    private ProductService productService;
    
    public PopularItemsResponse getPopularItems(String categoryId, int page, int limit) {
        String cacheKey = buildCacheKey(categoryId, page, limit);
        
        // Try cache first
        List<String> productIds = redis.opsForZSet()
            .reverseRange(cacheKey, (page-1)*limit, page*limit-1);
        
        if (productIds != null && !productIds.isEmpty()) {
            return buildResponse(productIds, true);
        }
        
        // Cache miss - fetch from DynamoDB
        QueryRequest query = QueryRequest.builder()
            .tableName("popularity_rankings")
            .keyConditionExpression("category_id = :cat")
            .expressionAttributeValues(Map.of(":cat", categoryId))
            .limit(limit)
            .build();
        
        QueryResponse response = dynamodb.query(query);
        productIds = response.items().stream()
            .map(item -> item.get("product_id").s())
            .collect(Collectors.toList());
        
        // Update cache
        updateCache(cacheKey, productIds);
        
        return buildResponse(productIds, false);
    }
    
    private void updateCache(String key, List<String> productIds) {
        ZSetOperations<String, String> zset = redis.opsForZSet();
        for (int i = 0; i < productIds.size(); i++) {
            zset.add(key, productIds.get(i), productIds.size() - i);
        }
        redis.expire(key, 1, TimeUnit.HOURS);
    }
}
```

---

### 4.5 Cache Layer Design

**Redis Cluster Configuration:**
- **Topology:** 6 nodes (3 masters, 3 replicas)
- **Memory:** 64GB per node
- **Eviction Policy:** allkeys-lru

**Cache Keys Structure:**

```
# Popular items (sorted sets)
category:{category_id}:popular:24h → ZSET (score = popularity_score)
global:popular:24h → ZSET
category:{category_id}:popular:7d → ZSET

# Hot items (TopK)
hot:last_10min → TOPK data structure

# Product metadata (hash)
product:{product_id}:meta → HASH

# Cache metadata
cache:updated_at:{key} → STRING (timestamp)
```

**Cache Update Strategy:**
- **Write-through:** Batch job updates cache after writing to DynamoDB
- **TTL:** 1 hour for popular items, 1 minute for hot items
- **Invalidation:** On-demand invalidation via admin API

**Cache Warming:**
- Pre-populate top 1000 items per category on deployment
- Background job refreshes cache every 30 minutes

---

## 5. DATA FLOW DIAGRAMS

### 5.1 Write Path (Event Ingestion)

```
User Action (View/Click/Purchase)
    ↓
Frontend/Mobile SDK
    ↓
Event Batching (5 sec buffer)
    ↓
API Gateway
    ↓
Event Ingestion Service
    ↓
Kafka Producer (async)
    ↓
Kafka Cluster (partitioned by product_id)
    ↓
├─→ Kafka Connect → S3 (for batch processing)
└─→ Flink Consumer (for real-time processing)
```

### 5.2 Read Path (Get Popular Items)

```
Client Request
    ↓
API Gateway (rate limiting, auth)
    ↓
Load Balancer
    ↓
Popular Items Service
    ↓
Check Redis Cache
    ├─→ Cache Hit → Return cached results (< 10ms)
    └─→ Cache Miss
            ↓
        Query DynamoDB
            ↓
        Enrich with Product Metadata (PostgreSQL)
            ↓
        Update Redis Cache
            ↓
        Return results (< 100ms)
```

---

## 6. SCALABILITY & PERFORMANCE

### 6.1 Scaling Strategy

**Horizontal Scaling:**
- API Services: Auto-scale based on CPU (target: 70%)
- Kafka: Add partitions and brokers
- Redis: Add shards to cluster
- Spark: Increase executor count

**Vertical Scaling:**
- DynamoDB: Increase provisioned throughput
- ClickHouse: Add more nodes to cluster

### 6.2 Performance Optimizations

**1. Pagination Optimization:**
- Use cursor-based pagination for large result sets
- Cache first 5 pages per category

**2. Query Optimization:**
- DynamoDB: Use GSI for efficient queries
- Redis: Use pipelining for batch operations

**3. Network Optimization:**
- CDN for static assets
- gRPC for internal service communication
- Connection pooling

**4. Computation Optimization:**
- Pre-compute rankings (no runtime computation)
- Use approximate algorithms (TopK, Count-Min Sketch)
- Incremental updates instead of full recomputation

### 6.3 Capacity Planning

**Storage Estimates:**
- Events: 1B events/day × 500 bytes = 500GB/day
- S3 retention (7 days): 3.5TB
- DynamoDB: 10M products × 1KB = 10GB
- Redis: Top 1000 items × 1000 categories × 1KB = 1GB

**Compute Estimates:**
- Spark job: 500GB data, 100 executors, 30 min runtime
- Flink job: 10K events/sec, 10 task managers

---

## 7. RELIABILITY & FAULT TOLERANCE

### 7.1 Failure Scenarios & Handling

| Component | Failure | Impact | Mitigation |
|-----------|---------|--------|------------|
| Kafka | Broker down | Event loss | Replication factor 3, ISR |
| Redis | Cache miss | Slower reads | Fallback to DynamoDB |
| DynamoDB | Throttling | Failed writes | Exponential backoff, retry |
| Spark Job | Job failure | Stale rankings | Retry logic, alerting |
| Flink Job | Checkpoint failure | Data loss | Savepoints every 5 min |
| API Service | Instance down | Request failure | Load balancer health checks |

### 7.2 Data Consistency

**Event Processing:**
- **At-least-once delivery:** Kafka guarantees
- **Idempotent processing:** Use event_id for deduplication
- **Exactly-once semantics:** Flink checkpointing

**Ranking Consistency:**
- **Eventual consistency:** Rankings may be stale by 1 hour
- **Acceptable:** Users don't expect real-time accuracy
- **Conflict resolution:** Last-write-wins (timestamp-based)

### 7.3 Monitoring & Alerting

**Key Metrics:**
- Event ingestion rate (events/sec)
- Kafka lag (consumer lag per partition)
- API latency (p50, p95, p99)
- Cache hit rate (target: > 95%)
- Batch job duration (target: < 30 min)
- Error rate (target: < 0.1%)

**Alerting Rules:**
- Kafka lag > 1M messages
- API p99 latency > 500ms
- Cache hit rate < 90%
- Batch job failure
- Error rate > 1%

**Tools:**
- Prometheus + Grafana (metrics)
- ELK Stack (logs)
- PagerDuty (alerting)
- Jaeger (distributed tracing)

---

## 8. SECURITY CONSIDERATIONS

### 8.1 Authentication & Authorization

- API Gateway: OAuth 2.0 / JWT tokens
- Service-to-service: mTLS
- Admin APIs: Role-based access control (RBAC)

### 8.2 Data Privacy

- PII masking in logs
- Encryption at rest (S3, DynamoDB)
- Encryption in transit (TLS 1.3)
- GDPR compliance: User data deletion API

### 8.3 Rate Limiting

- Per-user: 100 requests/minute
- Per-IP: 1000 requests/minute
- Burst allowance: 2x normal rate

---

## 9. TRADE-OFFS & DESIGN DECISIONS

### 9.1 Batch vs Streaming

**Decision:** Hybrid approach (batch + streaming)

| Aspect | Batch | Streaming |
|--------|-------|-----------|
| Accuracy | High (100%) | Approximate (95%) |
| Latency | 1 hour | 1 minute |
| Cost | Lower | Higher |
| Complexity | Lower | Higher |
| Use Case | Popular items | Hot items |

**Rationale:**
- Batch for accuracy and cost-efficiency
- Streaming for real-time trending items
- Users accept 1-hour staleness for popular items

### 9.2 Redis vs DynamoDB for Serving

**Decision:** Redis as primary, DynamoDB as fallback

| Aspect | Redis | DynamoDB |
|--------|-------|----------|
| Latency | < 10ms | 10-50ms |
| Cost | Higher (memory) | Lower (pay-per-request) |
| Durability | Lower | Higher |
| Scalability | Vertical + sharding | Horizontal (auto-scale) |

**Rationale:**
- Redis for ultra-low latency reads
- DynamoDB for durability and fallback
- Cache hit rate > 95% makes Redis cost-effective

### 9.3 Exact vs Approximate Counting

**Decision:** Approximate for hot items

**Rationale:**
- Exact counting requires DB write per event (not scalable)
- TopK/Count-Min Sketch: 95%+ accuracy, O(1) space
- Users don't notice 5% error in trending items

### 9.4 Global vs Category Rankings

**Decision:** Support both

**Implementation:**
- Separate cache keys for global and per-category
- Batch job computes both in single pass
- API parameter to switch between views

---

## 10. FUTURE ENHANCEMENTS

### 10.1 Personalization

- User-specific popularity (based on browsing history)
- Collaborative filtering (users like you also viewed)
- A/B testing framework for ranking algorithms

### 10.2 Advanced Features

- Geographic popularity (trending in your region)
- Time-of-day patterns (morning vs evening trends)
- Seasonal adjustments (holidays, events)
- Price-sensitivity scoring

### 10.3 Machine Learning

- ML-based popularity prediction
- Anomaly detection (sudden spikes)
- Fraud detection (fake clicks/purchases)
- Dynamic weight optimization

---

## 11. DEPLOYMENT STRATEGY

### 11.1 Infrastructure

**Cloud Provider:** AWS

**Services:**
- EKS (Kubernetes) for API services
- MSK (Managed Kafka)
- ElastiCache (Redis)
- DynamoDB
- EMR (Spark)
- Kinesis Data Analytics (Flink)
- S3 for data lake
- CloudFront (CDN)

### 11.2 CI/CD Pipeline

```
Code Commit → GitHub
    ↓
CI (GitHub Actions)
    ├─ Unit Tests
    ├─ Integration Tests
    └─ Build Docker Images
    ↓
Push to ECR
    ↓
CD (ArgoCD)
    ├─ Deploy to Dev
    ├─ Deploy to Staging
    └─ Deploy to Prod (Blue-Green)
```

### 11.3 Rollout Strategy

- **Blue-Green Deployment:** Zero-downtime deployments
- **Canary Releases:** 5% → 25% → 50% → 100%
- **Feature Flags:** Gradual feature rollout
- **Rollback Plan:** Automated rollback on error rate spike

---

## 12. COST ESTIMATION

**Monthly Cost Breakdown (AWS):**

| Component | Specification | Monthly Cost |
|-----------|--------------|--------------|
| EKS (API) | 20 nodes (m5.xlarge) | $3,000 |
| MSK (Kafka) | 6 brokers (kafka.m5.large) | $2,500 |
| ElastiCache | 6 nodes (r6g.xlarge) | $2,000 |
| DynamoDB | 10GB, 1000 RCU/WCU | $500 |
| EMR (Spark) | 10 nodes, 2 hrs/day | $1,000 |
| S3 | 10TB storage, 1TB transfer | $300 |
| CloudFront | 10TB transfer | $850 |
| **Total** | | **~$10,150/month** |

**Cost Optimization:**
- Use spot instances for Spark jobs (70% savings)
- Reserved instances for steady-state workloads
- S3 lifecycle policies (move to Glacier after 30 days)

---

## 13. INTERVIEW TALKING POINTS

### 13.1 Opening Statement

"We're designing a data-heavy ranking and aggregation system for e-commerce browsing, optimized for popularity-based discovery. The focus is on scalability, low latency, and handling millions of products with billions of user interactions."

### 13.2 Key Discussion Points

1. **Popularity Definition:** Weighted composite score with time decay
2. **Hybrid Architecture:** Batch for accuracy, streaming for freshness
3. **Cache-First Strategy:** Redis for sub-100ms latency
4. **Approximate Algorithms:** TopK for space efficiency
5. **Event-Driven Design:** Kafka as central nervous system
6. **Idempotent Processing:** Handle at-least-once delivery
7. **Graceful Degradation:** Fallback to DynamoDB on cache miss

### 13.3 Handling Follow-Up Questions

**Q: How do you handle sudden traffic spikes?**
- Auto-scaling API services
- Kafka buffering (absorbs spikes)
- Redis cluster can handle 1M ops/sec
- Rate limiting at API gateway

**Q: What if Spark job fails?**
- Retry logic with exponential backoff
- Alerting to on-call engineer
- Serve stale data from cache (acceptable)
- Manual trigger option

**Q: How do you prevent fake clicks/purchases?**
- Rate limiting per user/IP
- Anomaly detection (ML-based)
- Bot detection at API gateway
- Fraud score in popularity calculation

**Q: How do you handle new products (cold start)?**
- Boost score for new products (first 24 hours)
- Editorial curation for featured items
- Hybrid recommendation (content + collaborative)

---

## 14. SUMMARY

This design provides:

✅ **Scalability:** Handles 10M products, 100M users, 1B events/day  
✅ **Performance:** < 100ms API latency, 95%+ cache hit rate  
✅ **Reliability:** 99.99% availability, fault-tolerant architecture  
✅ **Flexibility:** Supports batch and real-time processing  
✅ **Cost-Effective:** ~$10K/month for large-scale operation  

**Core Principles:**
1. Pre-compute everything possible (no runtime computation)
2. Cache aggressively (Redis-first strategy)
3. Accept eventual consistency (1-hour staleness OK)
4. Use approximate algorithms (95% accuracy sufficient)
5. Design for failure (graceful degradation)

This architecture is production-ready and demonstrates senior-level system design thinking with clear trade-offs, scalability considerations, and operational excellence.
