# E-Commerce Popularity System - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: High-Level Architecture (8 min)
Phase 5: API Design (3 min)
Phase 6: Data Models (5 min)
Phase 7: Core Algorithm - Popularity Definition (8 min)
Phase 8: Deep Dive - Batch Pipeline (5 min)
Phase 9: Follow-up: Real-Time Hot Items (3 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"Thank you for the problem. Before I start designing, let me ask some clarifying questions to ensure I understand the requirements correctly."

### Questions to Ask:

**Q1: Scope Clarification**
- "You mentioned browsing, not purchases. So we're focusing on displaying popular items, not the checkout flow, correct?"
- "Should we support search, or just browsing by category?"

**Expected Answer:** Just browsing/discovery, no search needed

**Q2: Scale & Traffic**
- "How many products do we have in the catalog? Thousands, millions?"
- "How many daily active users are we expecting?"
- "What's the expected read vs write ratio?"

**Expected Answer:** 10M products, 100M DAU, read-heavy (99:1)

**Q3: Define "Popularity"**
- "What makes an item 'popular'? Is it based on views, clicks, purchases, or a combination?"
- "Should recent activity be weighted more than historical data?"
- "Do we need global popularity or category-specific rankings?"

**Expected Answer:** Combination of signals, recent activity matters, both global and category-wise

**Q4: Freshness Requirements**
- "How fresh does the popularity data need to be? Real-time, near real-time, or can it be stale by hours?"
- "Is it acceptable if rankings update every hour?"

**Expected Answer:** Hourly updates acceptable for batch, real-time is follow-up

**Q5: User Experience**
- "How many items should we show? Top 10, top 100, paginated?"
- "What's the acceptable latency for API responses?"

**Expected Answer:** Paginated (50 per page), < 100ms latency

**Q6: Geographic Distribution**
- "Is this a global system or single region?"
- "Do we need to handle multiple languages/currencies?"

**Expected Answer:** Start with single region, can discuss multi-region later

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### What to Say:

"Based on our discussion, let me summarize the requirements."

### Functional Requirements:

```
1. Display popular items to users
   - Global popularity rankings
   - Category-wise popularity rankings
   - Paginated results (50 items per page)

2. Calculate popularity based on multiple signals
   - Product views
   - Product clicks
   - Add-to-cart events
   - Purchase events
   - Time decay (recent > old)

3. Batch processing pipeline
   - Aggregate events periodically (hourly)
   - Calculate popularity scores
   - Update rankings

4. API endpoints
   - GET /products/popular?category={id}&page={n}
   - Return product details with popularity scores

Out of Scope (explicitly state):
- Search functionality
- Checkout/payment flow
- Inventory management
- User authentication (assume handled elsewhere)
- Personalization (follow-up question)
```

### Non-Functional Requirements:

```
1. Performance
   - API latency: < 100ms (p99)
   - Support 10K requests/second (peak)
   - Batch job completion: < 30 minutes

2. Scalability
   - Handle 10M products
   - Process 1B events/day
   - Support 100M daily active users

3. Availability
   - 99.9% uptime for read path
   - Graceful degradation if batch job fails

4. Consistency
   - Eventual consistency acceptable
   - Rankings can be stale by 1 hour
   - Approximate rankings acceptable (95%+ accuracy)

5. Cost Efficiency
   - Optimize for read-heavy workload
   - Batch processing during off-peak hours
```

### Why This Matters:
✓ Shows structured thinking
✓ Clarifies scope explicitly
✓ Sets expectations with interviewer

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me do some quick calculations to understand the scale we're dealing with."

### Traffic Estimation:

```
Given:
- 100M daily active users (DAU)
- Each user browses 20 products/day
- Peak traffic: 3x average

Calculations:

1. Daily Requests (Read)
   = 100M users × 20 products
   = 2B product views/day
   = 2B / 86,400 seconds
   = ~23,000 requests/second (average)
   = ~70,000 requests/second (peak)

2. Daily Events (Write)
   Views: 2B/day
   Clicks (20% of views): 400M/day
   Add-to-cart (5% of views): 100M/day
   Purchases (1% of views): 20M/day
   
   Total events: ~2.5B/day
   = ~29,000 events/second (average)
```

### Storage Estimation:

```
1. Product Catalog
   - 10M products
   - Each product: ~2KB (name, description, price, images URLs)
   - Total: 10M × 2KB = 20GB
   - With indexes: ~30GB

2. Events (Raw)
   - 2.5B events/day
   - Each event: ~500 bytes
   - Daily: 2.5B × 500B = 1.25TB/day
   - Retention (7 days): ~9TB

3. Popularity Rankings
   - 10M products × 1KB (product_id, score, rank, metadata)
   - Total: 10GB
   - Per category (100 categories): 10GB
   - Total: ~20GB

4. Cache (Redis)
   - Top 1000 products per category
   - 100 categories × 1000 products × 100 bytes
   - Total: ~10MB (very small!)

Total Storage: ~9TB (mostly raw events)
```

### Bandwidth Estimation:

```
1. Incoming (Events)
   - 29,000 events/sec × 500 bytes
   = 14.5 MB/sec
   = ~1.2 TB/day

2. Outgoing (API Responses)
   - 23,000 requests/sec × 50 products × 2KB
   = 2.3 GB/sec
   = ~200 TB/day

Note: Most responses served from CDN/cache
Actual bandwidth: ~20 TB/day (10% cache miss)
```

### Compute Estimation:

```
Batch Processing (Spark):
- Process 1.25TB of data (daily events)
- Aggregate 10M products
- Calculate scores
- Estimated time: 20-30 minutes
- Cluster size: 50 nodes (m5.xlarge)
- Cost: ~$50/day

API Servers:
- 70K peak RPS
- Each server: 1K RPS
- Required: 70 servers
- With redundancy: 100 servers
- Cost: ~$7,200/month
```

### Summary Table:

```
┌─────────────────────┬──────────────────┐
│ Metric              │ Value            │
├─────────────────────┼──────────────────┤
│ Products            │ 10M              │
│ DAU                 │ 100M             │
│ Events/day          │ 2.5B             │
│ Peak RPS            │ 70K              │
│ Storage             │ ~9TB             │
│ Bandwidth           │ ~20TB/day        │
│ API Servers         │ 100              │
│ Batch Cluster       │ 50 nodes         │
└─────────────────────┴──────────────────┘
```

### Why This Matters:
✓ Shows quantitative thinking
✓ Validates design decisions
✓ Identifies bottlenecks early

---

## PHASE 4: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"Now let me present the high-level architecture. I'll separate the read path (serving) from the write path (data processing)."

### Complete Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Web App    │  │  Mobile App  │  │  3rd Party   │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CDN (CloudFront)                              │
│  - Static assets (HTML, JS, CSS, Images)                        │
│  - Cache API responses (5 min TTL)                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              API GATEWAY / LOAD BALANCER                         │
│  - Rate limiting (1000 req/min per user)                        │
│  - Authentication (JWT validation)                               │
│  - Request routing                                               │
│  - SSL termination                                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                             │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Popular Items Service (Stateless)                     │    │
│  │  - GET /products/popular                               │    │
│  │  - GET /products/hot                                   │    │
│  │  - Auto-scaling (50-100 instances)                     │    │
│  └────────────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Product Catalog Service                               │    │
│  │  - GET /products/{id}                                  │    │
│  │  - Product details, images, prices                     │    │
│  └────────────────────────────────────────────────────────┘    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      CACHE LAYER                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Redis Cluster (6 nodes: 3 master, 3 replica)           │  │
│  │  - Sorted sets for rankings                             │  │
│  │  - Key: "category:{id}:popular"                         │  │
│  │  - TTL: 1 hour                                          │  │
│  │  - Memory: 64GB per node                                │  │
│  │  - Eviction: allkeys-lru                                │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STORAGE LAYER                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  PostgreSQL  │  │  DynamoDB    │  │ ClickHouse   │         │
│  │  (Products)  │  │  (Rankings)  │  │  (Analytics) │         │
│  │  - Master    │  │  - On-demand │  │  - OLAP      │         │
│  │  - 2 Replicas│  │  - Auto-scale│  │  - Queries   │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                         ▲
                         │ Batch updates (hourly)
                         │
┌─────────────────────────────────────────────────────────────────┐
│                  DATA PROCESSING LAYER                           │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  BATCH PIPELINE (Cron: hourly)                         │    │
│  │                                                          │    │
│  │  S3 (Raw Events)                                        │    │
│  │    ↓                                                     │    │
│  │  Spark Cluster (50 nodes)                              │    │
│  │    ├─ Read events (last 24h)                           │    │
│  │    ├─ Aggregate by product_id                          │    │
│  │    ├─ Calculate popularity scores                      │    │
│  │    └─ Rank products                                    │    │
│  │    ↓                                                     │    │
│  │  Write to DynamoDB + Redis                             │    │
│  │                                                          │    │
│  │  Duration: ~30 minutes                                  │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  REAL-TIME PIPELINE (Streaming) - Follow-up            │    │
│  │                                                          │    │
│  │  Kafka → Flink → Sliding Window → Redis                │    │
│  │  (For "hot items" in last 10 minutes)                  │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                         ▲
                         │
┌─────────────────────────────────────────────────────────────────┐
│                   EVENT INGESTION LAYER                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Kafka Cluster (MSK)                                     │  │
│  │  - Topics: user-views, user-clicks, cart-events,        │  │
│  │            purchase-events                               │  │
│  │  - Partitions: 100 per topic                            │  │
│  │  - Replication: 3                                       │  │
│  │  - Retention: 7 days                                    │  │
│  │  - Throughput: 30K events/sec                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Kafka Connect                                           │  │
│  │  - Streams events to S3 (Parquet format)                │  │
│  │  - Partitioned by date/hour                             │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                         ▲
                         │
┌─────────────────────────────────────────────────────────────────┐
│                    EVENT PRODUCERS                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Frontend    │  │  Mobile SDK  │  │  Backend     │         │
│  │  (JS SDK)    │  │  (Native)    │  │  Services    │         │
│  │  - Views     │  │  - Views     │  │  - Purchases │         │
│  │  - Clicks    │  │  - Clicks    │  │  - Cart      │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### Explain Data Flow:

**READ PATH (Serving Popular Items):**
```
1. User requests popular items
   ↓
2. CDN (cache hit: 10ms) → Return
   ↓ (cache miss)
3. API Gateway → Load Balancer
   ↓
4. Popular Items Service
   ↓
5. Redis (cache hit: 20ms) → Return
   ↓ (cache miss)
6. DynamoDB (50ms) → Return
   ↓
7. Update Redis cache
   ↓
8. Return to user

Total latency:
- CDN hit: 10ms (80% of requests)
- Redis hit: 30ms (15% of requests)
- DynamoDB hit: 70ms (5% of requests)
- p99: < 100ms ✓
```

**WRITE PATH (Event Processing):**
```
1. User interacts with product
   ↓
2. Frontend/Backend emits event
   ↓
3. Kafka (buffered, async)
   ↓
4. Kafka Connect → S3 (continuous)
   ↓
5. Spark Job (hourly)
   ├─ Read events from S3
   ├─ Aggregate counts
   ├─ Calculate scores
   └─ Rank products
   ↓
6. Write to DynamoDB + Redis
   ↓
7. Rankings updated (available for reads)
```

### Why This Matters:
✓ Separates read and write paths
✓ Shows scalability at each layer
✓ Considers caching strategy
✓ Explains data flow clearly

---

## PHASE 5: API DESIGN (3 minutes)

### What to Say:

"Let me define the key API endpoints we'll expose."

### API Endpoints:

**1. Get Popular Items**
```
GET /api/v1/products/popular

Query Parameters:
- category_id (optional): string - Filter by category
- page (optional): integer - Page number (default: 1)
- limit (optional): integer - Items per page (default: 50, max: 100)
- time_range (optional): enum[24h, 7d, 30d] - Time window (default: 24h)

Response: 200 OK
{
  "data": [
    {
      "product_id": "prod-123",
      "name": "iPhone 15 Pro",
      "description": "Latest iPhone with A17 chip",
      "category_id": "electronics",
      "price": 999.99,
      "currency": "USD",
      "image_url": "https://cdn.example.com/images/prod-123.jpg",
      "popularity_score": 95.5,
      "rank": 1,
      "metrics": {
        "views": 150000,
        "purchases": 5000
      }
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total_items": 1000,
    "total_pages": 20,
    "has_next": true,
    "has_prev": false
  },
  "metadata": {
    "updated_at": "2024-01-15T10:00:00Z",
    "cache_hit": true,
    "response_time_ms": 25
  }
}

Error Responses:
- 400 Bad Request: Invalid parameters
- 429 Too Many Requests: Rate limit exceeded
- 500 Internal Server Error: System error
```

**2. Get Product Details**
```
GET /api/v1/products/{product_id}

Response: 200 OK
{
  "product_id": "prod-123",
  "name": "iPhone 15 Pro",
  "description": "...",
  "category_id": "electronics",
  "price": 999.99,
  "images": [
    "https://cdn.example.com/images/prod-123-1.jpg",
    "https://cdn.example.com/images/prod-123-2.jpg"
  ],
  "seller": {
    "seller_id": "seller-456",
    "name": "Apple Store"
  },
  "created_at": "2023-09-15T00:00:00Z"
}
```

**3. Get Hot Items (Follow-up)**
```
GET /api/v1/products/hot

Query Parameters:
- limit (optional): integer - Number of items (default: 20, max: 100)

Response: 200 OK
{
  "data": [
    {
      "product_id": "prod-789",
      "name": "Trending Product",
      "event_count": 15420,
      "trend_score": 95.2,
      "window": "last_10_minutes"
    }
  ],
  "metadata": {
    "window_start": "2024-01-15T10:00:00Z",
    "window_end": "2024-01-15T10:10:00Z",
    "updated_at": "2024-01-15T10:10:00Z"
  }
}
```

### Why This Matters:
✓ Clear contract for clients
✓ Considers pagination
✓ Includes metadata for debugging
✓ Defines error cases

---

## PHASE 6: DATA MODELS (5 minutes)

### What to Say:

"Let me define the schemas for our key data stores."

### 1. Product Catalog (PostgreSQL)

```sql
CREATE TABLE products (
    product_id VARCHAR(50) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category_id VARCHAR(50) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    seller_id VARCHAR(50) NOT NULL,
    image_urls TEXT[], -- Array of CDN URLs
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    is_active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_products_category ON products(category_id) WHERE is_active = TRUE;
CREATE INDEX idx_products_seller ON products(seller_id);
CREATE INDEX idx_products_created ON products(created_at DESC);

-- Partitioning by category for better performance
CREATE TABLE products_electronics PARTITION OF products
    FOR VALUES IN ('electronics');
```

**Explain:**
- "Images stored in S3, served via CloudFront CDN"
- "image_urls contains CDN URLs, not actual images"
- "Partitioned by category for faster queries"
- "is_active for soft deletes"

### 2. Events (Kafka Schema)

```json
{
  "schema": {
    "type": "record",
    "name": "ProductEvent",
    "fields": [
      {"name": "event_id", "type": "string"},
      {"name": "event_type", "type": {"type": "enum", "symbols": ["view", "click", "add_to_cart", "purchase"]}},
      {"name": "product_id", "type": "string"},
      {"name": "user_id", "type": "string"},
      {"name": "category_id", "type": "string"},
      {"name": "timestamp", "type": "long"},
      {"name": "session_id", "type": "string"},
      {"name": "device", "type": {"type": "enum", "symbols": ["web", "mobile", "tablet"]}},
      {"name": "price", "type": ["null", "double"]},
      {"name": "metadata", "type": ["null", "string"]}
    ]
  }
}
```

**Example Event:**
```json
{
  "event_id": "evt-uuid-123",
  "event_type": "view",
  "product_id": "prod-456",
  "user_id": "user-789",
  "category_id": "electronics",
  "timestamp": 1705334400000,
  "session_id": "session-abc",
  "device": "mobile",
  "price": 999.99,
  "metadata": "{\"source\":\"homepage\",\"position\":3}"
}
```

**Kafka Topics:**
```
user-views:
- Partitions: 100 (by product_id hash)
- Replication: 3
- Retention: 7 days
- Compression: Snappy

user-clicks:
- Same configuration

cart-events:
- Same configuration

purchase-events:
- Same configuration
- Higher retention: 30 days (for analytics)
```

### 3. Popularity Rankings (DynamoDB)

```
Table: popularity_rankings

Primary Key:
- Partition Key: category_id (String)
- Sort Key: score (Number, descending)

Attributes:
- product_id (String)
- score (Number)
- rank (Number)
- metrics (Map)
  - views (Number)
  - clicks (Number)
  - add_to_cart (Number)
  - purchases (Number)
- updated_at (String, ISO 8601)

Global Secondary Index (GSI):
- GSI Name: product_id-index
- Partition Key: product_id
- Purpose: Reverse lookup (get rank for specific product)

Capacity:
- On-demand billing
- Auto-scaling enabled
- Estimated: 1000 RCU, 100 WCU

Example Item:
{
  "category_id": "electronics",
  "score": 95.5,
  "product_id": "prod-123",
  "rank": 1,
  "metrics": {
    "views": 150000,
    "clicks": 30000,
    "add_to_cart": 7500,
    "purchases": 5000
  },
  "updated_at": "2024-01-15T10:00:00Z"
}
```

### 4. Redis Cache Structure

```
Data Structure: Sorted Set (ZSET)

Keys:
- "category:{category_id}:popular:24h"
- "category:{category_id}:popular:7d"
- "global:popular:24h"
- "hot:last_10min" (for real-time)

Example:
Key: "category:electronics:popular:24h"
Type: ZSET
Members: product_ids
Scores: popularity_scores

Commands:
# Add product with score
ZADD category:electronics:popular:24h 95.5 "prod-123"

# Get top 50 products
ZREVRANGE category:electronics:popular:24h 0 49 WITHSCORES

# Get rank of specific product
ZREVRANK category:electronics:popular:24h "prod-123"

# Get score of specific product
ZSCORE category:electronics:popular:24h "prod-123"

# Set TTL
EXPIRE category:electronics:popular:24h 3600

Memory Estimation:
- 100 categories
- Top 1000 products per category
- Each entry: ~100 bytes (product_id + score + overhead)
- Total: 100 × 1000 × 100 bytes = 10MB
```

### 5. Aggregated Events (S3)

```
Path Structure:
s3://events-bucket/
  ├─ user-views/
  │   ├─ date=2024-01-15/
  │   │   ├─ hour=00/
  │   │   │   ├─ part-00001.parquet
  │   │   │   └─ part-00002.parquet
  │   │   ├─ hour=01/
  │   │   └─ ...
  │   └─ date=2024-01-16/
  ├─ user-clicks/
  ├─ cart-events/
  └─ purchase-events/

Format: Parquet (columnar)
Compression: Snappy
Partitioning: By date and hour
Retention: 7 days (lifecycle policy)

Schema:
- event_id: string
- event_type: string
- product_id: string
- user_id: string
- category_id: string
- timestamp: long
- session_id: string
- device: string
- price: double
- metadata: string
```

### Why This Matters:
✓ Shows understanding of different storage types
✓ Considers access patterns
✓ Explains partitioning strategies
✓ Includes indexing for performance


## PHASE 7: CORE ALGORITHM - POPULARITY DEFINITION (8 minutes)

### What to Say:

"Now let's tackle the most important part: defining what 'popularity' means and how we calculate it."

### Step 1: Identify Signals

**Explain the thinking process:**

"Popularity is subjective, so we need to make it concrete. I'll consider multiple engagement signals with different weights:"

```
Signal Hierarchy (strongest to weakest):

1. PURCHASE (Weight: 10x)
   - Strongest signal - actual revenue
   - User completed transaction
   - Tracked by: Backend purchase service

2. ADD_TO_CART (Weight: 5x)
   - Strong purchase intent
   - User seriously considering
   - Tracked by: Backend cart service

3. CLICK (Weight: 1x)
   - User interested enough to view details
   - Tracked by: Frontend click handler

4. VIEW (Weight: 0.1x, logarithmic)
   - Basic engagement
   - Can be inflated (bots, accidental views)
   - Use log scale to prevent gaming
   - Tracked by: Frontend impression tracker

5. RECENCY (Weight: 0.5x)
   - Time decay factor
   - Recent activity > old activity
   - Prevents stale products from dominating
```

### Step 2: Mathematical Formula

**Present the formula:**

```
popularity_score = 
    w1 × log(views + 1)
  + w2 × clicks
  + w3 × add_to_cart
  + w4 × purchases
  + w5 × recency_decay
  + w6 × conversion_rate

Where:
w1 = 0.1  (views weight)
w2 = 0.3  (clicks weight)
w3 = 1.5  (add-to-cart weight)
w4 = 3.0  (purchases weight)
w5 = 0.5  (recency weight)
w6 = 1.0  (conversion weight)

recency_decay = e^(-λ × days_since_last_event)
where λ = 0.1 (decay constant)

conversion_rate = purchases / views (if views > 0)
```

**Explain each component:**

```
1. log(views + 1):
   - Logarithmic scale prevents view-bombing
   - 100 views → log(101) = 4.6
   - 10,000 views → log(10,001) = 9.2
   - Only 2x increase for 100x more views

2. Linear for clicks, cart, purchases:
   - These are harder to game
   - Direct correlation with popularity

3. Recency decay:
   - Today: e^0 = 1.0 (100% weight)
   - 7 days ago: e^(-0.7) = 0.50 (50% weight)
   - 14 days ago: e^(-1.4) = 0.25 (25% weight)
   - 30 days ago: e^(-3.0) = 0.05 (5% weight)

4. Conversion rate:
   - Quality indicator
   - High views but low purchases = not truly popular
   - Balances pure volume metrics
```

### Step 3: Example Calculation

**Walk through a concrete example:**

```
Product A (iPhone):
- Views: 100,000
- Clicks: 20,000
- Add-to-cart: 5,000
- Purchases: 2,000
- Last event: Today
- Conversion rate: 2,000/100,000 = 0.02

Calculation:
= 0.1 × log(100,001)
+ 0.3 × 20,000
+ 1.5 × 5,000
+ 3.0 × 2,000
+ 0.5 × 1.0
+ 1.0 × 0.02

= 0.1 × 11.5
+ 6,000
+ 7,500
+ 6,000
+ 0.5
+ 0.02

= 1.15 + 6,000 + 7,500 + 6,000 + 0.5 + 0.02
= 19,501.67

Normalized (0-100 scale):
= (19,501.67 / max_score) × 100
= 95.5

Product B (Less popular item):
- Views: 1,000
- Clicks: 100
- Add-to-cart: 10
- Purchases: 2
- Last event: 7 days ago
- Conversion rate: 2/1,000 = 0.002

Calculation:
= 0.1 × log(1,001)
+ 0.3 × 100
+ 1.5 × 10
+ 3.0 × 2
+ 0.5 × 0.50
+ 1.0 × 0.002

= 0.1 × 6.9
+ 30
+ 15
+ 6
+ 0.25
+ 0.002

= 51.94

Normalized: 25.2
```

### Step 4: Input Data Sources

**Draw the data flow:**

```
┌─────────────────────────────────────────────────────────┐
│                   EVENT SOURCES                          │
└─────────────────────────────────────────────────────────┘

Frontend (JavaScript SDK):
┌──────────────────────────────────────────┐
│ User Action          → Event Emitted     │
├──────────────────────────────────────────┤
│ Product visible      → VIEW              │
│ Product clicked      → CLICK             │
│ Image loaded         → (no event)        │
│ Scroll past product  → (no event)        │
└──────────────────────────────────────────┘
        ↓
    Batched (every 5 seconds)
        ↓
    POST /api/v1/events/batch
        ↓
    Kafka topic: "user-views", "user-clicks"

Backend Services:
┌──────────────────────────────────────────┐
│ User Action          → Event Emitted     │
├──────────────────────────────────────────┤
│ Add to cart          → ADD_TO_CART       │
│ Remove from cart     → (no event)        │
│ Purchase completed   → PURCHASE          │
└──────────────────────────────────────────┘
        ↓
    Synchronous (immediate)
        ↓
    Kafka topic: "cart-events", "purchase-events"

All Events:
        ↓
    Kafka (central event bus)
        ↓
    ├─→ Kafka Connect → S3 (for batch processing)
    └─→ Flink (for real-time processing)
```

### Step 5: Why This Approach?

**Explain the rationale:**

```
Advantages:
✓ Multi-dimensional: Captures different engagement types
✓ Weighted: Prioritizes revenue-generating actions
✓ Time-aware: Recent activity matters more
✓ Gaming-resistant: Log scale for views, conversion rate check
✓ Tunable: Weights can be adjusted based on business goals
✓ Explainable: Clear formula, easy to debug

Trade-offs:
✗ Complexity: More complex than simple view count
✗ Tuning required: Weights need experimentation
✗ Cold start: New products have low scores

Alternatives considered:
1. Simple view count
   - Too easy to game
   - Doesn't reflect quality

2. Purchase count only
   - Ignores browsing behavior
   - Biased toward cheap items

3. Machine learning model
   - More accurate but less explainable
   - Higher complexity and cost
   - Can be added later for personalization
```

### Why This Matters:
✓ Shows analytical thinking
✓ Turns fuzzy requirement into concrete algorithm
✓ Explains trade-offs
✓ Considers gaming/abuse

---

## PHASE 8: DEEP DIVE - BATCH PIPELINE (5 minutes)

### What to Say:

"Let me explain the batch processing pipeline that runs hourly to compute popularity scores."

### Pipeline Architecture:

```
┌─────────────────────────────────────────────────────────────┐
│  BATCH PIPELINE (Scheduled: Every hour at :00)              │
│                                                               │
│  Trigger: Cron (AWS EventBridge / Airflow)                  │
│  Duration: ~30 minutes                                       │
│  Cluster: 50 Spark nodes (m5.xlarge)                        │
│                                                               │
│  ┌────────────────────────────────────────────────────┐    │
│  │  STEP 1: Data Ingestion (5 min)                    │    │
│  │                                                      │    │
│  │  Source: S3 (Parquet files)                        │    │
│  │  Path: s3://events/*/date=2024-01-15/hour=*/      │    │
│  │  Window: Last 24 hours                             │    │
│  │  Size: ~1.25 TB                                    │    │
│  │                                                      │    │
│  │  val events = spark.read                           │    │
│  │    .parquet("s3://events/*/date=*/hour=*/")       │    │
│  │    .filter($"timestamp" > now() - 24.hours)       │    │
│  └────────────────────────────────────────────────────┘    │
│                         ↓                                    │
│  ┌────────────────────────────────────────────────────┐    │
│  │  STEP 2: Aggregation (10 min)                      │    │
│  │                                                      │    │
│  │  Group by: product_id, category_id                 │    │
│  │  Aggregate: Count events by type                   │    │
│  │                                                      │    │
│  │  val aggregated = events                           │    │
│  │    .groupBy("product_id", "category_id")          │    │
│  │    .agg(                                           │    │
│  │      count(when($"event_type" === "view", 1))     │    │
│  │        .as("views"),                               │    │
│  │      count(when($"event_type" === "click", 1))    │    │
│  │        .as("clicks"),                              │    │
│  │      count(when($"event_type" === "add_to_cart")) │    │
│  │        .as("add_to_carts"),                        │    │
│  │      count(when($"event_type" === "purchase", 1)) │    │
│  │        .as("purchases"),                           │    │
│  │      max("timestamp").as("last_event_time")       │    │
│  │    )                                               │    │
│  └────────────────────────────────────────────────────┘    │
│                         ↓                                    │
│  ┌────────────────────────────────────────────────────┐    │
│  │  STEP 3: Score Calculation (10 min)                │    │
│  │                                                      │    │
│  │  Apply formula to each product                     │    │
│  │                                                      │    │
│  │  val scored = aggregated.map { row =>             │    │
│  │    val views = row.getAs[Long]("views")           │    │
│  │    val clicks = row.getAs[Long]("clicks")         │    │
│  │    val carts = row.getAs[Long]("add_to_carts")    │    │
│  │    val purchases = row.getAs[Long]("purchases")   │    │
│  │    val lastEvent = row.getAs[Long]("last_event")  │    │
│  │                                                      │    │
│  │    val recencyDecay = calculateDecay(lastEvent)    │    │
│  │    val conversionRate = purchases / views.toDouble │    │
│  │                                                      │    │
│  │    val score =                                     │    │
│  │      0.1 * log(views + 1) +                       │    │
│  │      0.3 * clicks +                               │    │
│  │      1.5 * carts +                                │    │
│  │      3.0 * purchases +                            │    │
│  │      0.5 * recencyDecay +                         │    │
│  │      1.0 * conversionRate                         │    │
│  │                                                      │    │
│  │    (row.product_id, row.category_id, score)       │    │
│  │  }                                                 │    │
│  └────────────────────────────────────────────────────┘    │
│                         ↓                                    │
│  ┌────────────────────────────────────────────────────┐    │
│  │  STEP 4: Ranking (3 min)                           │    │
│  │                                                      │    │
│  │  Rank within each category                         │    │
│  │                                                      │    │
│  │  val ranked = scored                               │    │
│  │    .withColumn("rank",                             │    │
│  │      row_number().over(                            │    │
│  │        Window.partitionBy("category_id")           │    │
│  │          .orderBy(desc("score"))                   │    │
│  │      )                                             │    │
│  │    )                                               │    │
│  │    .filter($"rank" <= 1000)  // Keep top 1000     │    │
│  └────────────────────────────────────────────────────┘    │
│                         ↓                                    │
│  ┌────────────────────────────────────────────────────┐    │
│  │  STEP 5: Storage (2 min)                           │    │
│  │                                                      │    │
│  │  Write to multiple destinations:                   │    │
│  │                                                      │    │
│  │  1. DynamoDB (for serving)                         │    │
│  │     ranked.write                                   │    │
│  │       .format("dynamodb")                          │    │
│  │       .option("tableName", "popularity_rankings")  │    │
│  │       .mode("overwrite")                           │    │
│  │       .save()                                      │    │
│  │                                                      │    │
│  │  2. Redis (for caching)                            │    │
│  │     ranked.foreachPartition { partition =>         │    │
│  │       val redis = new Jedis("redis-cluster")      │    │
│  │       partition.foreach { row =>                   │    │
│  │         val key = s"category:${row.category}:pop" │    │
│  │         redis.zadd(key, row.score, row.product)   │    │
│  │         redis.expire(key, 3600)                    │    │
│  │       }                                            │    │
│  │     }                                              │    │
│  │                                                      │    │
│  │  3. ClickHouse (for analytics)                     │    │
│  │     ranked.write                                   │    │
│  │       .format("jdbc")                              │    │
│  │       .option("url", "jdbc:clickhouse://...")     │    │
│  │       .mode("append")                              │    │
│  │       .save()                                      │    │
│  └────────────────────────────────────────────────────┘    │
│                                                               │
│  Total Duration: ~30 minutes                                 │
│  Success Rate: 99.5% (with retries)                          │
└─────────────────────────────────────────────────────────────┘
```

### Optimization Techniques:

```
1. Partitioning:
   - Events partitioned by date/hour in S3
   - Only read last 24 hours (not entire dataset)
   - Reduces I/O by 90%

2. Columnar Format (Parquet):
   - Only read needed columns (product_id, event_type, timestamp)
   - Compression: 10x smaller than JSON
   - Faster reads: 5x improvement

3. Spark Optimizations:
   - Broadcast small tables (categories, products)
   - Repartition by product_id for even distribution
   - Cache intermediate results
   - Dynamic partition pruning

4. Incremental Processing:
   - Only process new events since last run
   - Merge with previous rankings
   - Reduces computation by 80%

5. Parallel Writes:
   - Write to DynamoDB, Redis, ClickHouse in parallel
   - Use connection pooling
   - Batch writes (100 items per batch)
```

### Failure Handling:

```
Scenario 1: Spark Job Fails
- Retry automatically (3 attempts)
- Alert on-call engineer
- Serve stale data from cache (acceptable)
- Manual trigger available

Scenario 2: Partial Failure (some partitions fail)
- Checkpoint progress
- Resume from last successful partition
- Idempotent writes (safe to retry)

Scenario 3: Downstream Write Fails
- Write to DLQ (Dead Letter Queue)
- Retry with exponential backoff
- Alert if DLQ size > threshold

Monitoring:
- Job duration (alert if > 45 min)
- Records processed (alert if < expected)
- Error rate (alert if > 1%)
- Data freshness (alert if > 2 hours old)
```

### Why This Matters:
✓ Shows big data processing knowledge
✓ Explains optimizations
✓ Considers failure scenarios
✓ Demonstrates production thinking

---

## PHASE 9: FOLLOW-UP - REAL-TIME HOT ITEMS (3 minutes)

### What to Say:

"For the follow-up on real-time 'hot' items, we need a streaming pipeline to capture what's trending in the last 10 minutes."

### Key Differences:

```
┌──────────────────┬─────────────────┬─────────────────┐
│ Aspect           │ Batch Pipeline  │ Streaming       │
├──────────────────┼─────────────────┼─────────────────┤
│ Window           │ 24 hours        │ 10 minutes      │
│ Update Frequency │ Hourly          │ Every minute    │
│ Technology       │ Spark Batch     │ Flink Streaming │
│ Latency          │ 1 hour          │ 1 minute        │
│ Accuracy         │ Exact (100%)    │ Approximate(95%)│
│ Use Case         │ "Top sellers"   │ "Trending now"  │
│ Cost             │ Lower           │ Higher          │
└──────────────────┴─────────────────┴─────────────────┘
```

### Streaming Architecture:

```
Kafka (real-time events)
  ↓
Flink Streaming Job
  ├─ Sliding Window: 10 minutes
  ├─ Slide Interval: 1 minute
  ├─ Keyed by: product_id
  └─ Aggregate: Count events
  ↓
TopK Algorithm (keep top 100)
  ↓
Redis (key: "hot:last_10min")
  ├─ Data structure: Sorted Set
  ├─ TTL: 15 minutes
  └─ Updated: Every minute
  ↓
API: GET /products/hot
```

### Flink Implementation (Pseudo-code):

```java
StreamExecutionEnvironment env = 
    StreamExecutionEnvironment.getExecutionEnvironment();

// Kafka source
DataStream<Event> events = env
    .addSource(new FlinkKafkaConsumer<>("user-events", ...))
    .assignTimestampsAndWatermarks(...);

// Sliding window: 10 min window, 1 min slide
DataStream<ProductCount> hotItems = events
    .keyBy(event -> event.getProductId())
    .window(SlidingEventTimeWindows.of(
        Time.minutes(10), 
        Time.minutes(1)
    ))
    .aggregate(new CountAggregator())
    .windowAll(TumblingProcessingTimeWindows.of(Time.minutes(1)))
    .process(new TopKFunction(100));

// Write to Redis
hotItems.addSink(new RedisSink());
```

### Why Approximate Algorithms?

```
Problem with Exact Counting:
- 10M products × 10 bytes = 100MB per window
- Need to track all products
- Memory intensive
- Slow updates

Solution: TopK Data Structure
- Only track top K items (e.g., K=100)
- Space: O(K) instead of O(N)
- Accuracy: 95%+ (good enough for "hot items")
- Fast: O(log K) per update

Redis TopK:
TOPK.RESERVE hot:last_10min 100 50 7 0.925
TOPK.ADD hot:last_10min prod-123
TOPK.LIST hot:last_10min WITHCOUNT
```

### Why This Matters:
✓ Shows streaming knowledge
✓ Explains batch vs streaming trade-offs
✓ Demonstrates approximate algorithms
✓ Considers memory constraints

---

## SUMMARY: EVALUATION CRITERIA MAPPING

### How This Solution Scores:

```
┌──────────────────────────┬─────────────────────────┬──────────────┐
│ Evaluation Element       │ Evidence in Solution    │ Rating       │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Workable Solution        │ Complete end-to-end     │ Outstanding  │
│                          │ All components defined  │              │
│                          │ Production-ready        │              │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Clarifying Questions     │ Asked 6+ key questions  │ Outstanding  │
│                          │ Defined fuzzy concepts  │              │
│                          │ Negotiated scope        │              │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Efficiency               │ Multi-layer caching     │ Outstanding  │
│                          │ Batch processing        │              │
│                          │ Approximate algorithms  │              │
│                          │ Partitioning strategy   │              │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Simplicity               │ Clear separation        │ Outstanding  │
│                          │ Straightforward flow    │              │
│                          │ Not over-engineered     │              │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Quality of Abstraction   │ Event-driven arch       │ Outstanding  │
│                          │ Microservices pattern   │              │
│                          │ Layered design          │              │
│                          │ Technology-agnostic     │              │
└──────────────────────────┴─────────────────────────┴──────────────┘

Overall Rating: Top 20% (Above the Bar)
```

### What Makes This Top 20%:

1. **Started with clarifying questions** (not assumptions)
2. **Defined functional & non-functional requirements explicitly**
3. **Did back-of-envelope calculations** (showed quantitative thinking)
4. **Turned "popularity" from fuzzy to concrete** (mathematical formula)
5. **Explained trade-offs** (batch vs streaming, exact vs approximate)
6. **Showed scalability** (horizontal scaling at every layer)
7. **Addressed follow-ups comprehensively** (real-time streaming)
8. **Demonstrated depth** (Spark optimizations, Flink windows, TopK)
9. **Considered failure scenarios** (retry logic, monitoring)
10. **Used concrete numbers** (not hand-wavy estimates)

---

## INTERVIEW TIPS

### Time Management:

```
0-5 min:   Clarifying questions
5-8 min:   Requirements + Estimation
8-16 min:  High-level architecture
16-19 min: API design
19-24 min: Data models
24-32 min: Popularity algorithm
32-37 min: Batch pipeline deep dive
37-40 min: Real-time follow-up
40-45 min: Q&A, trade-offs discussion
```

### What to Emphasize:

1. **Ask questions first** - Don't assume
2. **Draw diagrams** - Visual > verbal
3. **Use numbers** - "10M products, 100M users"
4. **Explain WHY** - Every decision has rationale
5. **Mention trade-offs** - Shows senior thinking
6. **Start simple, then extend** - Don't over-engineer upfront

### Common Pitfalls to Avoid:

❌ Jumping to implementation without clarifying
❌ Not defining "popularity" concretely
❌ Single server solution (not scalable)
❌ Ignoring caching (poor performance)
❌ Not considering failure scenarios
❌ Over-engineering (blockchain for rankings)
❌ Vague answers ("we'll use a database")

### Strong Signals to Send:

✓ "Let me clarify the requirements first"
✓ "Here are the trade-offs between X and Y"
✓ "This scales to N because..."
✓ "We can start simple and extend later"
✓ "The key challenge here is..."
✓ "Based on the numbers, we need..."

---

## FINAL CHECKLIST

Before ending the interview, ensure you've covered:

- [ ] Asked clarifying questions
- [ ] Defined functional requirements
- [ ] Defined non-functional requirements
- [ ] Did back-of-envelope estimation
- [ ] Drew high-level architecture
- [ ] Designed APIs
- [ ] Defined data models
- [ ] Explained popularity algorithm
- [ ] Described batch pipeline
- [ ] Addressed real-time follow-up
- [ ] Discussed trade-offs
- [ ] Mentioned failure handling
- [ ] Talked about monitoring
- [ ] Considered scalability

This comprehensive approach demonstrates **Top 20%** performance! 🎯
