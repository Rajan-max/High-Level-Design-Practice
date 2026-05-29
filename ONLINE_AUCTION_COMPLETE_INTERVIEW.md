# Online Auction System - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: Core Entities & Data Model (4 min)
Phase 5: API Design (3 min)
Phase 6: High-Level Architecture (8 min)
Phase 7: Data Flow - Auction Creation & Bidding (5 min)
Phase 8: Deep Dive - Strong Consistency for Bids (6 min)
Phase 9: Deep Dive - Fault Tolerance & Durability (4 min)
Phase 10: Deep Dive - Real-time Updates & Scaling (5 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"I'll design an online auction system similar to eBay where users can list items for sale and others can bid on them. The highest bidder wins when the auction ends. Let me clarify the scope and requirements."

### Questions to Ask:

**Q1: Auction Types**
- "What types of auctions: English (ascending bids), Dutch (descending price), sealed bid?"
- "Do we support reserve prices (minimum price seller will accept)?"
- "Can auctions have 'Buy It Now' options?"

**Expected Answer:** English auctions only, reserve prices supported, Buy It Now out of scope

**Q2: Bidding Rules**
- "What are the bidding rules: minimum bid increment, proxy bidding, automatic bidding?"
- "Can users retract bids or only place new higher bids?"
- "Do we need bid validation (e.g., user has sufficient funds)?"

**Expected Answer:** Fixed increment bidding, no retractions, basic validation only

**Q3: Auction Duration**
- "What's the typical auction duration: hours, days, weeks?"
- "Do auctions have fixed end times or dynamic (extend if bid in last minutes)?"
- "Can sellers end auctions early?"

**Expected Answer:** 1-7 days typical, fixed end times, no early termination

**Q4: Item Categories**
- "What types of items: physical goods, digital goods, services?"
- "Do we need category-specific attributes (e.g., car auctions need VIN)?"
- "How do we handle item images and descriptions?"

**Expected Answer:** Physical goods only, generic attributes, images stored separately

**Q5: Payment & Fulfillment**
- "Do we handle payment processing or integrate with external providers?"
- "What happens after auction ends: automatic payment, manual coordination?"
- "Do we handle shipping/logistics?"

**Expected Answer:** Integrate with payment providers, manual coordination, shipping out of scope

**Q6: Scale & Concurrency**
- "What's the expected scale: concurrent auctions, users, bids per second?"
- "What's the expected bid rate for popular auctions?"
- "Are there peak times (e.g., auction endings)?"

**Expected Answer:** 10M concurrent auctions, 100M users, 15K bids/sec peak, surge at auction endings

**Q7: Consistency Requirements**
- "How critical is bid consistency: can we show stale data briefly?"
- "What happens if two users bid same amount simultaneously?"
- "Do we need audit trail for all bids?"

**Expected Answer:** Strong consistency critical, first bid wins on tie, full audit trail required

**Q8: Real-time Updates**
- "Should users see new bids in real-time or is polling acceptable?"
- "What's acceptable latency for bid updates?"
- "Do we need notifications (email, push) for outbid events?"

**Expected Answer:** Real-time updates needed, <2 second latency, notifications nice-to-have

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### What to Say:

"Based on our discussion, let me organize the requirements to guide our design decisions."

### Functional Requirements (Priority Order):

```
CORE REQUIREMENTS (Above the line):

1. Auction Creation (CRITICAL)
   - Sellers can create auctions with item details
   - Set starting price and reserve price
   - Set auction duration (start and end time)
   - Upload item images and description
   - Specify shipping and payment terms

2. Bid Placement (CRITICAL)
   - Users can place bids on active auctions
   - Bids must be higher than current highest bid
   - Validate bid amount (minimum increment)
   - Reject invalid bids immediately
   - Record all bids for audit trail

3. Auction Viewing (CRITICAL)
   - View auction details and item information
   - See current highest bid in real-time
   - View bid history (all bids or just own)
   - See time remaining until auction ends
   - View seller information and ratings

4. Auction Ending (CRITICAL)
   - Automatically end auctions at specified time
   - Determine winner (highest bidder)
   - Notify winner and seller
   - Handle reserve price not met scenario
   - Lock auction (no more bids accepted)

5. User Management (IMPORTANT)
   - User registration and authentication
   - User profiles with bidding history
   - Watchlist for tracking auctions
   - Saved searches and preferences

BELOW THE LINE (Out of scope):
- Advanced search and filtering
- Category browsing and recommendations
- Proxy/automatic bidding
- Buy It Now functionality
- Seller ratings and reviews
- Dispute resolution
- Payment processing (integrate only)
- Shipping and logistics
```

### Non-Functional Requirements:

```
PERFORMANCE:
- Bid placement latency: <100ms (P95)
- Bid validation: <50ms
- Real-time update latency: <2 seconds
- Auction page load: <500ms (P95)
- System throughput: 15K bids/second (peak)

SCALABILITY:
- Support 10M concurrent auctions
- Handle 100M registered users
- Process 1B bids per day
- Support 100K concurrent viewers per popular auction
- Horizontal scaling of all components

CONSISTENCY & RELIABILITY:
- Strong consistency for bid placement (CRITICAL)
- No lost bids (durable storage)
- Accurate bid ordering (first bid wins on tie)
- Atomic auction state transitions
- Complete audit trail for all bids

AVAILABILITY:
- System availability: 99.9% uptime
- Graceful degradation under load
- No auction data loss
- Automatic failover for critical components
- Read availability during write failures

DURABILITY:
- All bids persisted immediately
- Auction state recoverable after failures
- Message queue for bid processing
- Database replication and backups
- Point-in-time recovery capability

FAIRNESS:
- First-come-first-served for equal bids
- No bid manipulation or front-running
- Transparent bid history
- Consistent auction end times
- No preferential treatment
```

### Why This Matters:
✓ Strong consistency is non-negotiable for bids
✓ Durability prevents lost bids (trust issue)
✓ Real-time updates critical for user experience
✓ Fairness ensures platform credibility

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me calculate the scale to understand infrastructure requirements and identify potential bottlenecks."

### Traffic Estimation:

```
Given:
- 10M concurrent auctions
- 100M registered users
- Average auction duration: 7 days
- Average bids per auction: 100
- Peak traffic: 10x average (auction endings)

Calculations:

1. Auction Creation Rate
   Auctions per day: 10M / 7 = 1.43M auctions/day
   Creation QPS: 1.43M / 86,400 = 16.5 auctions/sec
   Peak: 16.5 × 10 = 165 auctions/sec

2. Bid Placement Rate
   Total bids: 10M auctions × 100 bids = 1B bids per week
   Daily bids: 1B / 7 = 143M bids/day
   Average QPS: 143M / 86,400 = 1,655 bids/sec
   
   Peak QPS (auction endings): 1,655 × 10 = 16,550 bids/sec
   
   This is our critical metric!

3. Auction Viewing (Read Traffic)
   Assume 10% of users actively browsing: 10M users
   Page views per user per day: 20
   Total page views: 10M × 20 = 200M/day
   Average QPS: 200M / 86,400 = 2,315 views/sec
   Peak: 2,315 × 5 = 11,575 views/sec

4. Real-time Updates (WebSocket/SSE Connections)
   Active viewers: 10M concurrent
   Popular auction: 100K viewers
   Connections per server: 10K (realistic)
   Servers needed: 10M / 10K = 1,000 servers

5. Read:Write Ratio
   Reads (views): 11,575/sec
   Writes (bids): 16,550/sec
   Ratio: 1:1.4 (write-heavy during peaks!)
   
   This is unusual - most systems are read-heavy
```

### Storage Estimation:

```
1. Auction Data
   Total auctions: 10M concurrent
   Auction record: 2KB (item details, prices, dates)
   Storage: 10M × 2KB = 20GB
   
   Annual auctions: 10M × 52 = 520M
   Annual storage: 520M × 2KB = 1TB/year

2. Item Data
   Items per auction: 1
   Item record: 5KB (description, attributes)
   Storage: 10M × 5KB = 50GB
   Annual: 520M × 5KB = 2.5TB/year

3. Item Images
   Images per item: 5
   Image size: 500KB (compressed)
   Storage per auction: 5 × 500KB = 2.5MB
   Total: 10M × 2.5MB = 25TB
   Annual: 520M × 2.5MB = 1.3PB/year

4. Bid Data (Critical)
   Total bids: 1B per week
   Bid record: 500 bytes (user, amount, timestamp)
   Weekly storage: 1B × 500B = 500GB/week
   Annual: 500GB × 52 = 26TB/year
   
   With 5-year retention: 130TB

5. User Data
   Total users: 100M
   User record: 2KB (profile, preferences)
   Storage: 100M × 2KB = 200GB

Total Storage Requirements:
- Hot data (active auctions): ~100GB
- Warm data (recent auctions): ~5TB
- Cold data (historical): ~130TB
- Images (CDN): ~1.3PB/year
- Total: ~135TB + 1.3PB images
```

### Database Load Estimation:

```
1. Write Load (Peak)
   Bid writes: 16,550/sec
   Auction updates: 16,550/sec (update max bid)
   Total writes: 33,100/sec
   
   This is VERY high for a single database!

2. Read Load (Peak)
   Auction views: 11,575/sec
   Bid history queries: 2,000/sec
   User queries: 1,000/sec
   Total reads: 14,575/sec

3. Database Capacity Analysis
   Single PostgreSQL instance:
   - Max writes: ~10K/sec (with optimization)
   - Max reads: ~50K/sec (with caching)
   
   Conclusion: Need database sharding for writes!
```

### Message Queue Estimation:

```
1. Bid Queue (Kafka)
   Message rate: 16,550 bids/sec (peak)
   Message size: 1KB (bid data + metadata)
   Throughput: 16,550 × 1KB = 16.5MB/sec
   
   Daily volume: 143M × 1KB = 143GB/day
   Retention: 7 days
   Storage: 143GB × 7 = 1TB

2. Notification Queue
   Outbid notifications: 16,550/sec (each bid outbids previous)
   Auction ending notifications: 165/sec
   Total: ~17K messages/sec
   Storage: Similar to bid queue

Kafka Capacity:
- Single broker: 100MB/sec
- Our load: 16.5MB/sec
- Conclusion: Single broker sufficient, but need replication
```

### Infrastructure Estimation:

```
1. Application Servers
   Bid Service: 50 instances (c5.2xlarge)
   Auction Service: 30 instances (c5.xlarge)
   API Gateway: 20 instances (c5.xlarge)
   Total: 100 instances

2. Database Servers
   PostgreSQL shards: 10 instances (r5.4xlarge)
   Read replicas: 20 instances (r5.2xlarge)
   Total: 30 instances

3. Cache Servers (Redis)
   Hot data: 100GB
   Cluster: 10 nodes (r5.xlarge)

4. Message Queue (Kafka)
   Brokers: 5 nodes (m5.xlarge)
   Zookeeper: 3 nodes (t3.medium)

5. Real-time Update Servers
   WebSocket/SSE: 1,000 instances (c5.large)

Total Monthly Cost (AWS):
- Compute: $150K
- Database: $80K
- Cache: $15K
- Message Queue: $10K
- Storage (S3): $30K
- Data Transfer: $50K
- CDN (CloudFront): $100K
Total: ~$435K/month
```

### Key Insights from Estimation:

```
1. Write-Heavy Workload
   - 33K writes/sec at peak
   - Requires database sharding
   - Message queue critical for durability

2. Real-time Connections
   - 10M concurrent connections
   - 1,000 servers just for WebSocket/SSE
   - Coordination challenge

3. Bid Consistency Critical
   - 16,550 bids/sec must be ordered correctly
   - Race conditions likely
   - Need strong consistency mechanism

4. Storage Growth
   - 130TB bid data over 5 years
   - 1.3PB images annually
   - Need archival strategy
```

---

## PHASE 4: CORE ENTITIES & DATA MODEL (4 minutes)

### What to Say:

"Let me define the core entities and their relationships. I'll start with a high-level overview, then detail the schema for our auction system."

### Core Entities:

```
1. User
   - Represents a buyer or seller
   - Attributes: user_id, email, name, rating

2. Auction
   - Represents an auction for an item
   - Attributes: auction_id, seller_id, start_time, end_time, status
   - Contains current highest bid information

3. Item
   - Represents the item being auctioned
   - Attributes: item_id, title, description, category, images
   - Can be reused across multiple auctions

4. Bid
   - Represents a bid on an auction
   - Attributes: bid_id, auction_id, user_id, amount, timestamp
   - Immutable once created (audit trail)
```

### Data Model Design:

```sql
-- Users Table (PostgreSQL)
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    rating DECIMAL(3,2) DEFAULT 0.00,
    total_bids INTEGER DEFAULT 0,
    total_wins INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'ACTIVE',
    
    INDEX idx_email (email),
    INDEX idx_status (status)
);

-- Items Table (PostgreSQL)
CREATE TABLE items (
    item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    title VARCHAR(500) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    condition VARCHAR(50),
    images JSONB,  -- Array of image URLs
    attributes JSONB,  -- Flexible attributes
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_category (category),
    INDEX idx_created_at (created_at)
);

-- Auctions Table (PostgreSQL) - CRITICAL TABLE
CREATE TABLE auctions (
    auction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    item_id UUID NOT NULL REFERENCES items(item_id),
    seller_id UUID NOT NULL REFERENCES users(user_id),
    
    -- Pricing
    starting_price DECIMAL(10,2) NOT NULL,
    reserve_price DECIMAL(10,2),
    current_max_bid DECIMAL(10,2) DEFAULT 0.00,
    current_winner_id UUID REFERENCES users(user_id),
    bid_increment DECIMAL(10,2) DEFAULT 1.00,
    
    -- Timing
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    
    -- Status
    status VARCHAR(20) NOT NULL DEFAULT 'SCHEDULED',
    -- SCHEDULED, ACTIVE, ENDED, CANCELLED
    
    -- Metadata
    total_bids INTEGER DEFAULT 0,
    total_views INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    -- Version for optimistic locking
    version INTEGER DEFAULT 0,
    
    INDEX idx_seller (seller_id),
    INDEX idx_status (status),
    INDEX idx_end_time (end_time),
    INDEX idx_status_end_time (status, end_time),
    
    CONSTRAINT chk_prices CHECK (
        starting_price > 0 AND
        (reserve_price IS NULL OR reserve_price >= starting_price) AND
        current_max_bid >= 0
    ),
    CONSTRAINT chk_times CHECK (end_time > start_time)
);

-- Bids Table (PostgreSQL) - CRITICAL TABLE
CREATE TABLE bids (
    bid_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    auction_id UUID NOT NULL REFERENCES auctions(auction_id),
    user_id UUID NOT NULL REFERENCES users(user_id),
    
    -- Bid details
    amount DECIMAL(10,2) NOT NULL,
    bid_time TIMESTAMP DEFAULT NOW(),
    
    -- Status
    status VARCHAR(20) NOT NULL,
    -- ACCEPTED, REJECTED, OUTBID
    
    -- Metadata
    previous_max_bid DECIMAL(10,2),
    ip_address INET,
    user_agent TEXT,
    
    INDEX idx_auction_time (auction_id, bid_time DESC),
    INDEX idx_user (user_id),
    INDEX idx_auction_status (auction_id, status),
    
    CONSTRAINT chk_amount CHECK (amount > 0)
);

-- Partition bids table by auction_id for better performance
-- (PostgreSQL 10+)
CREATE TABLE bids_partitioned (
    LIKE bids INCLUDING ALL
) PARTITION BY HASH (auction_id);

-- Create 10 partitions
CREATE TABLE bids_p0 PARTITION OF bids_partitioned
    FOR VALUES WITH (MODULUS 10, REMAINDER 0);
-- ... repeat for p1-p9
```

### Redis Cache Schema:

```redis
# Auction Cache (Hot Data)
Key: auction:{auction_id}
Type: Hash
Fields:
  current_max_bid: "150.00"
  current_winner_id: "user_123"
  total_bids: "45"
  status: "ACTIVE"
  end_time: "1715548800"
  version: "45"  # For optimistic locking
TTL: Auction duration + 1 hour

# Recent Bids Cache
Key: auction:{auction_id}:recent_bids
Type: List
Value: JSON array of last 100 bids
TTL: Auction duration + 1 hour

# User Active Bids
Key: user:{user_id}:active_bids
Type: Sorted Set
Score: bid_time (timestamp)
Member: auction_id
TTL: 30 days

# Auction Ending Soon (for cron job)
Key: auctions:ending_soon
Type: Sorted Set
Score: end_time (timestamp)
Member: auction_id
TTL: None (managed by cron)
```

### Key Design Decisions:

```
1. Separate Auction and Item Tables
   - Items can be reused (relist unsold items)
   - Cleaner data model
   - Easier to add item-specific features

2. current_max_bid in Auctions Table
   - Enables optimistic concurrency control
   - Avoids scanning bids table for max
   - Critical for performance

3. version Field for Optimistic Locking
   - Prevents race conditions
   - Lightweight compared to pessimistic locks
   - Enables horizontal scaling

4. Immutable Bids Table
   - Complete audit trail
   - No updates, only inserts
   - Easier to reason about consistency

5. Partitioned Bids Table
   - Distributes write load
   - Improves query performance
   - Enables parallel processing

6. Redis for Hot Data
   - Sub-millisecond reads
   - Reduces database load
   - Enables real-time updates
```

---

## PHASE 5: API DESIGN (3 minutes)

### What to Say:

"Let me design the REST APIs that satisfy our functional requirements, focusing on simplicity and performance."

### API Endpoints:

```
1. CREATE AUCTION
POST /v1/auctions
Headers:
  Authorization: Bearer <jwt_token>
  Content-Type: application/json

Request:
{
  "item": {
    "title": "Vintage Camera",
    "description": "Rare 1960s film camera in excellent condition",
    "category": "Electronics",
    "condition": "Used - Excellent",
    "images": ["https://cdn.example.com/img1.jpg", ...],
    "attributes": {
      "brand": "Leica",
      "model": "M3",
      "year": "1965"
    }
  },
  "starting_price": 500.00,
  "reserve_price": 1000.00,
  "bid_increment": 10.00,
  "duration_days": 7
}

Response: 201 Created
{
  "auction_id": "auction_123",
  "item_id": "item_456",
  "seller_id": "user_789",
  "starting_price": 500.00,
  "reserve_price": 1000.00,
  "current_max_bid": 0.00,
  "start_time": "2024-05-12T10:00:00Z",
  "end_time": "2024-05-19T10:00:00Z",
  "status": "SCHEDULED",
  "created_at": "2024-05-12T09:45:00Z"
}

---

2. PLACE BID
POST /v1/auctions/{auction_id}/bids
Headers:
  Authorization: Bearer <jwt_token>
  Content-Type: application/json
  X-Idempotency-Key: <uuid>  // Prevent duplicate bids

Request:
{
  "amount": 550.00
}

Response: 201 Created (Bid Accepted)
{
  "bid_id": "bid_789",
  "auction_id": "auction_123",
  "user_id": "user_456",
  "amount": 550.00,
  "bid_time": "2024-05-12T10:15:30Z",
  "status": "ACCEPTED",
  "previous_max_bid": 500.00,
  "is_winning": true
}

Response: 409 Conflict (Bid Rejected)
{
  "error": "BID_TOO_LOW",
  "message": "Bid must be at least $560.00",
  "current_max_bid": 550.00,
  "minimum_bid": 560.00
}

---

3. GET AUCTION DETAILS
GET /v1/auctions/{auction_id}
Headers:
  Authorization: Bearer <jwt_token>

Response: 200 OK
{
  "auction_id": "auction_123",
  "item": {
    "item_id": "item_456",
    "title": "Vintage Camera",
    "description": "...",
    "images": [...],
    "attributes": {...}
  },
  "seller": {
    "user_id": "user_789",
    "name": "John Doe",
    "rating": 4.8
  },
  "starting_price": 500.00,
  "reserve_price": null,  // Hidden from bidders
  "current_max_bid": 550.00,
  "current_winner_id": "user_456",  // Only visible to winner
  "bid_increment": 10.00,
  "start_time": "2024-05-12T10:00:00Z",
  "end_time": "2024-05-19T10:00:00Z",
  "time_remaining": 604800,  // seconds
  "status": "ACTIVE",
  "total_bids": 5,
  "total_views": 1234
}

---

4. GET BID HISTORY
GET /v1/auctions/{auction_id}/bids?limit={limit}&cursor={cursor}
Headers:
  Authorization: Bearer <jwt_token>

Query Parameters:
  - limit: 1-100 (default: 20)
  - cursor: Pagination cursor (optional)
  - status: ACCEPTED | REJECTED | OUTBID (optional)

Response: 200 OK
{
  "bids": [
    {
      "bid_id": "bid_789",
      "user_id": "user_456",  // Anonymized for privacy
      "user_name": "User ***456",
      "amount": 550.00,
      "bid_time": "2024-05-12T10:15:30Z",
      "status": "ACCEPTED"
    },
    // ... more bids
  ],
  "pagination": {
    "next_cursor": "bid_788#1715548530",
    "has_more": true
  }
}

---

5. GET USER'S ACTIVE BIDS
GET /v1/users/me/bids?status={status}&limit={limit}
Headers:
  Authorization: Bearer <jwt_token>

Query Parameters:
  - status: WINNING | OUTBID | ALL (default: ALL)
  - limit: 1-100 (default: 20)

Response: 200 OK
{
  "bids": [
    {
      "auction_id": "auction_123",
      "item_title": "Vintage Camera",
      "my_bid": 550.00,
      "current_max_bid": 550.00,
      "is_winning": true,
      "end_time": "2024-05-19T10:00:00Z",
      "time_remaining": 604800
    },
    // ... more bids
  ]
}

---

6. WATCH/UNWATCH AUCTION
POST /v1/auctions/{auction_id}/watch
DELETE /v1/auctions/{auction_id}/watch
Headers:
  Authorization: Bearer <jwt_token>

Response: 200 OK
{
  "auction_id": "auction_123",
  "is_watching": true
}

---

7. ESTABLISH REAL-TIME CONNECTION (SSE)
GET /v1/auctions/{auction_id}/stream
Headers:
  Authorization: Bearer <jwt_token>
  Last-Event-ID: <bid_id>  // For reconnection

Response: 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

Stream Format:
id: bid_789
event: new_bid
data: {"auction_id":"auction_123","amount":550.00,"bid_time":1715548530}

id: bid_790
event: new_bid
data: {"auction_id":"auction_123","amount":560.00,"bid_time":1715548545}

event: auction_ending_soon
data: {"auction_id":"auction_123","time_remaining":300}

event: auction_ended
data: {"auction_id":"auction_123","winner_id":"user_456","final_price":560.00}
```

### API Design Principles:

```
1. RESTful Design
   - Resource-based URLs
   - Standard HTTP methods
   - Proper status codes

2. Idempotency
   - X-Idempotency-Key for bid placement
   - Prevents duplicate bids on retry
   - 5-minute deduplication window

3. Optimistic Response
   - Return bid status immediately
   - Don't wait for all processing
   - Use SSE for updates

4. Privacy
   - Anonymize other bidders
   - Hide reserve price from bidders
   - Only show winner to winner

5. Real-time Updates
   - SSE for live bid updates
   - Reconnection support
   - Graceful degradation
```



---

## PHASE 6: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"I'll design a distributed architecture that separates auction management from bid processing. The key insight is using a message queue for durability and optimistic concurrency control for consistency."

### Complete System Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│   Web App    │   Mobile App   │   Admin Portal  │  Seller App   │
│  (React.js)  │ (React Native) │   (Dashboard)   │  (Management) │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EDGE & CDN LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│  CloudFront CDN  │  Application LB  │  WAF & DDoS Protection   │
│  (Item Images,   │  (Geographic     │  (Security & Rate        │
│   Static Assets) │   Load Balance)  │   Limiting)              │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                        API GATEWAY                              │
├─────────────────────────────────────────────────────────────────┤
│  Authentication  │  Request Routing │  Circuit Breaker         │
│  Rate Limiting   │  Protocol Trans  │  Monitoring & Logging    │
└─────────────────────────────────────────────────────────────────┘
                    │                               │
                    │ POST /bids                    │ GET /auctions
                    ▼                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     SERVICE LAYER                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Auction Service (Read Path)                                ││
│  │                                                              ││
│  │  Responsibilities:                                           ││
│  │  - Create new auctions                                       ││
│  │  - Retrieve auction details                                  ││
│  │  - List auctions (search, filter)                           ││
│  │  - Update auction metadata                                   ││
│  │  - Serve from cache when possible                           ││
│  │                                                              ││
│  │  Instances: 30 (auto-scaled)                                ││
│  │  Instance Type: c5.xlarge                                   ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Bid Producer Service (Write Path - Fast)                   ││
│  │                                                              ││
│  │  Responsibilities:                                           ││
│  │  - Accept bid requests                                       ││
│  │  - Basic validation (amount, format)                        ││
│  │  - Write to Kafka immediately                               ││
│  │  - Return acknowledgment to client                          ││
│  │  - Idempotency check (Redis)                                ││
│  │                                                              ││
│  │  Goal: Minimize latency, ensure durability                  ││
│  │  Latency: <50ms                                             ││
│  │                                                              ││
│  │  Instances: 20 (auto-scaled)                                ││
│  │  Instance Type: c5.large                                    ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Bid Consumer Service (Write Path - Consistent)             ││
│  │                                                              ││
│  │  Responsibilities:                                           ││
│  │  - Consume bids from Kafka                                   ││
│  │  - Validate bid against current max                         ││
│  │  - Use optimistic concurrency control                       ││
│  │  - Update auction max bid atomically                        ││
│  │  - Write bid to database                                    ││
│  │  - Publish to real-time update system                       ││
│  │  - Handle retries on conflicts                              ││
│  │                                                              ││
│  │  Goal: Strong consistency, no lost bids                     ││
│  │                                                              ││
│  │  Instances: 50 (auto-scaled)                                ││
│  │  Instance Type: c5.2xlarge                                  ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Real-time Update Service (SSE/WebSocket)                   ││
│  │                                                              ││
│  │  Responsibilities:                                           ││
│  │  - Maintain SSE connections to clients                      ││
│  │  - Subscribe to Redis Pub/Sub                               ││
│  │  - Broadcast bid updates to viewers                         ││
│  │  - Handle reconnections                                      ││
│  │  - Send heartbeats                                          ││
│  │                                                              ││
│  │  Instances: 1,000 (auto-scaled)                             ││
│  │  Instance Type: c5.large                                    ││
│  │  Connections per instance: 10K                              ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Auction Ending Service (Cron Job)                          ││
│  │                                                              ││
│  │  Responsibilities:                                           ││
│  │  - Query auctions ending soon (Redis sorted set)            ││
│  │  - End auctions at specified time                           ││
│  │  - Determine winner                                         ││
│  │  - Send notifications                                        ││
│  │  - Update auction status                                    ││
│  │                                                              ││
│  │  Runs: Every 10 seconds                                     ││
│  │  Instances: 3 (with leader election)                        ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                    │                               │
                    ▼                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     MESSAGE QUEUE LAYER                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Kafka Cluster                                   ││
│  │                                                              ││
│  │  Topic: bids                                                 ││
│  │  Partitions: 100 (partitioned by auction_id)                ││
│  │  Replication: 3                                              ││
│  │  Retention: 7 days                                           ││
│  │                                                              ││
│  │  Guarantees:                                                 ││
│  │  - Ordering within partition (per auction)                  ││
│  │  - At-least-once delivery                                   ││
│  │  - Durable storage                                           ││
│  │                                                              ││
│  │  Throughput: 16,550 messages/sec (peak)                     ││
│  │  Brokers: 5 nodes (m5.xlarge)                               ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Redis Pub/Sub                                   ││
│  │                                                              ││
│  │  Channels: auction:{auction_id}:updates                      ││
│  │  Purpose: Real-time bid updates to SSE servers              ││
│  │  Fire-and-forget delivery                                   ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                     DATA LAYER                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              PostgreSQL (Sharded)                            ││
│  │                                                              ││
│  │  Shard Strategy: Hash(auction_id) % 10                      ││
│  │                                                              ││
│  │  Shard 0: Auctions, Items, Bids (auction_id % 10 == 0)     ││
│  │  Shard 1: Auctions, Items, Bids (auction_id % 10 == 1)     ││
│  │  ...                                                         ││
│  │  Shard 9: Auctions, Items, Bids (auction_id % 10 == 9)     ││
│  │                                                              ││
│  │  Each shard:                                                 ││
│  │  - Primary: r5.4xlarge (write)                              ││
│  │  - Replicas: 2x r5.2xlarge (read)                           ││
│  │                                                              ││
│  │  Total: 10 primaries + 20 replicas = 30 instances           ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Redis Cluster (Cache & Coordination)            ││
│  │                                                              ││
│  │  Use Cases:                                                  ││
│  │  1. Auction Cache (hot data)                                ││
│  │     - Key: auction:{auction_id}                             ││
│  │     - Type: Hash                                            ││
│  │     - TTL: Auction duration + 1 hour                        ││
│  │                                                              ││
│  │  2. Recent Bids Cache                                       ││
│  │     - Key: auction:{auction_id}:recent_bids                 ││
│  │     - Type: List (last 100 bids)                            ││
│  │                                                              ││
│  │  3. Idempotency Check                                       ││
│  │     - Key: bid:idempotency:{key}                            ││
│  │     - TTL: 5 minutes                                        ││
│  │                                                              ││
│  │  4. Auction Ending Soon                                     ││
│  │     - Key: auctions:ending_soon                             ││
│  │     - Type: Sorted Set (score = end_time)                   ││
│  │                                                              ││
│  │  5. Distributed Locks                                       ││
│  │     - For auction ending coordination                       ││
│  │                                                              ││
│  │  Cluster: 10 nodes (r5.xlarge)                              ││
│  │  Sharding: Hash-based                                       ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities:

```
1. API Gateway
   - JWT validation and user authentication
   - Rate limiting (per user, per endpoint)
   - Request routing to appropriate service
   - Circuit breaker for downstream failures

2. Auction Service (Read-Heavy)
   - CRUD operations for auctions
   - Serve from Redis cache when possible
   - Fallback to database on cache miss
   - Handle search and filtering

3. Bid Producer (Write-Fast)
   - Accept bid requests
   - Minimal validation (format, basic rules)
   - Write to Kafka immediately (durability)
   - Return acknowledgment quickly (<50ms)
   - Check idempotency key in Redis

4. Bid Consumer (Write-Consistent)
   - Consume from Kafka (ordered per auction)
   - Validate bid against current max
   - Optimistic concurrency control
   - Update auction atomically
   - Write bid to database
   - Publish to Redis Pub/Sub for real-time updates

5. Real-time Update Service
   - Maintain SSE connections
   - Subscribe to Redis Pub/Sub
   - Broadcast updates to connected clients
   - Handle reconnections with catch-up

6. Auction Ending Service
   - Query Redis for auctions ending soon
   - End auctions at specified time
   - Determine winner
   - Send notifications
   - Use distributed lock for coordination
```

### Data Flow Summary:

```
Auction Creation Flow:
Client → API Gateway → Auction Service → PostgreSQL
                                      → Redis Cache

Bid Placement Flow (Fast Path):
Client → API Gateway → Bid Producer → Kafka → Acknowledgment

Bid Processing Flow (Consistent Path):
Kafka → Bid Consumer → PostgreSQL (optimistic lock)
                    → Redis Cache (update)
                    → Redis Pub/Sub (publish)

Real-time Update Flow:
Redis Pub/Sub → Real-time Update Service → SSE → Clients

Auction Ending Flow:
Cron → Auction Ending Service → Redis (query)
                              → PostgreSQL (update)
                              → Notification Service
```

---

## PHASE 7: DATA FLOW - AUCTION CREATION & BIDDING (5 minutes)

### What to Say:

"Let me walk through the complete data flow from auction creation to bid placement, highlighting the critical paths for consistency and durability."

### Flow 1: Create Auction

```
┌──────────────────────────────────────────────────────────────────┐
│  STEP 1: CLIENT CREATES AUCTION                                  │
└──────────────────────────────────────────────────────────────────┘

User submits auction:
  POST /v1/auctions
  Authorization: Bearer eyJhbGc...
  {
    "item": {...},
    "starting_price": 500.00,
    "duration_days": 7
  }

┌──────────────────────────────────────────────────────────────────┐
│  STEP 2: API GATEWAY PROCESSING                                  │
└──────────────────────────────────────────────────────────────────┘

API Gateway:
  1. Validate JWT token
     - Extract user_id: user_789
     - Verify seller permissions
     
  2. Check rate limit
     key = "ratelimit:user_789:create_auction"
     count = INCR key
     if count > 10: return 429 Too Many Requests
     
  3. Route to Auction Service

Latency: ~20ms

┌──────────────────────────────────────────────────────────────────┐
│  STEP 3: AUCTION SERVICE PROCESSING                              │
└──────────────────────────────────────────────────────────────────┘

Auction Service:
  1. Validate auction data
     - Check starting_price > 0
     - Validate duration (1-30 days)
     - Validate item details
     
  2. Generate IDs
     auction_id = UUID.v4()
     item_id = UUID.v4()
     
  3. Calculate times
     start_time = now()
     end_time = now() + duration_days
     
  4. Determine shard
     shard_id = hash(auction_id) % 10

Latency: ~10ms

┌──────────────────────────────────────────────────────────────────┐
│  STEP 4: PERSIST TO DATABASE                                     │
└──────────────────────────────────────────────────────────────────┘

Transaction (PostgreSQL Shard):
  BEGIN;
  
  -- Insert item
  INSERT INTO items (item_id, title, description, ...)
  VALUES (...);
  
  -- Insert auction
  INSERT INTO auctions (
    auction_id, item_id, seller_id,
    starting_price, current_max_bid,
    start_time, end_time, status, version
  ) VALUES (
    auction_id, item_id, user_789,
    500.00, 0.00,
    start_time, end_time, 'ACTIVE', 0
  );
  
  COMMIT;

Latency: ~30ms

┌──────────────────────────────────────────────────────────────────┐
│  STEP 5: UPDATE REDIS CACHE                                      │
└──────────────────────────────────────────────────────────────────┘

Redis Operations (pipelined):
  1. Cache auction data
     HMSET auction:auction_123
       current_max_bid 0.00
       current_winner_id ""
       status "ACTIVE"
       end_time 1715548800
       version 0
     EXPIRE auction:auction_123 604800  // 7 days
     
  2. Add to ending soon index
     ZADD auctions:ending_soon 1715548800 auction_123
     
  3. Initialize recent bids list
     DEL auction:auction_123:recent_bids
     EXPIRE auction:auction_123:recent_bids 604800

Latency: ~5ms

Total Latency: ~65ms
```

### Flow 2: Place Bid (Fast Path - Durability)

```
┌──────────────────────────────────────────────────────────────────┐
│  STEP 1: CLIENT PLACES BID                                       │
└──────────────────────────────────────────────────────────────────┘

User bids:
  POST /v1/auctions/auction_123/bids
  Authorization: Bearer eyJhbGc...
  X-Idempotency-Key: req_abc123
  {
    "amount": 550.00
  }

┌──────────────────────────────────────────────────────────────────┐
│  STEP 2: API GATEWAY PROCESSING                                  │
└──────────────────────────────────────────────────────────────────┘

API Gateway:
  1. Validate JWT
  2. Check rate limit (10 bids/min per user)
  3. Route to Bid Producer Service

Latency: ~15ms

┌──────────────────────────────────────────────────────────────────┐
│  STEP 3: BID PRODUCER SERVICE (FAST!)                            │
└──────────────────────────────────────────────────────────────────┘

Bid Producer:
  1. Check idempotency key
     key = "bid:idempotency:req_abc123"
     if EXISTS key:
       return cached response
     
  2. Basic validation
     - amount > 0
     - auction exists (Redis check)
     - auction is ACTIVE
     
  3. Create bid message
     bid_message = {
       bid_id: UUID.v4(),
       auction_id: "auction_123",
       user_id: "user_456",
       amount: 550.00,
       timestamp: now(),
       idempotency_key: "req_abc123"
     }
     
  4. Write to Kafka (CRITICAL - Durability)
     partition = hash(auction_123) % 100
     kafka.produce(
       topic="bids",
       partition=partition,
       key=auction_123,
       value=bid_message
     )
     
     Wait for acknowledgment from Kafka
     
  5. Cache idempotency response
     SETEX bid:idempotency:req_abc123 300 "PENDING"
     
  6. Return to client
     Response: 202 Accepted
     {
       bid_id: "bid_789",
       status: "PENDING",
       message: "Bid received and being processed"
     }

Latency: ~35ms (dominated by Kafka write)

Key Point: Bid is now DURABLE in Kafka!
Even if everything crashes, bid will be processed.
```

### Flow 3: Process Bid (Consistent Path - Strong Consistency)

```
┌──────────────────────────────────────────────────────────────────┐
│  STEP 1: BID CONSUMER RECEIVES MESSAGE                           │
└──────────────────────────────────────────────────────────────────┘

Bid Consumer (consuming from Kafka):
  message = kafka.consume(topic="bids", partition=42)
  bid = parse(message.value)
  
  auction_id = bid.auction_id  // "auction_123"
  amount = bid.amount  // 550.00

┌──────────────────────────────────────────────────────────────────┐
│  STEP 2: OPTIMISTIC CONCURRENCY CONTROL                          │
└──────────────────────────────────────────────────────────────────┘

Bid Consumer:
  1. Read current auction state (Redis)
     auction_data = HGETALL auction:auction_123
     current_max_bid = 500.00
     current_version = 0
     
  2. Validate bid
     minimum_bid = current_max_bid + bid_increment
     if amount < minimum_bid:
       reject_bid(bid, "BID_TOO_LOW")
       return
     
  3. Attempt atomic update (PostgreSQL)
     shard_id = hash(auction_123) % 10
     
     BEGIN;
     
     -- Optimistic lock: Update only if version matches
     UPDATE auctions
     SET 
       current_max_bid = 550.00,
       current_winner_id = 'user_456',
       total_bids = total_bids + 1,
       version = version + 1,
       updated_at = NOW()
     WHERE 
       auction_id = 'auction_123'
       AND version = 0  // Optimistic lock condition
       AND status = 'ACTIVE';
     
     -- Check if update succeeded
     IF (ROW_COUNT = 0) THEN
       ROLLBACK;
       // Version mismatch - concurrent bid won
       // Retry from step 1
     ELSE
       -- Insert bid record
       INSERT INTO bids (
         bid_id, auction_id, user_id,
         amount, bid_time, status,
         previous_max_bid
       ) VALUES (
         'bid_789', 'auction_123', 'user_456',
         550.00, NOW(), 'ACCEPTED',
         500.00
       );
       
       COMMIT;
     END IF;

Latency: ~40ms (with potential retries)

┌──────────────────────────────────────────────────────────────────┐
│  STEP 3: UPDATE REDIS CACHE                                      │
└──────────────────────────────────────────────────────────────────┘

Bid Consumer (after successful DB update):
  Redis pipeline:
    1. Update auction cache
       HMSET auction:auction_123
         current_max_bid 550.00
         current_winner_id user_456
         total_bids 1
         version 1
       
    2. Add to recent bids
       LPUSH auction:auction_123:recent_bids "{bid_json}"
       LTRIM auction:auction_123:recent_bids 0 99
       
    3. Update idempotency cache
       SETEX bid:idempotency:req_abc123 300 "ACCEPTED"

Latency: ~5ms

┌──────────────────────────────────────────────────────────────────┐
│  STEP 4: PUBLISH REAL-TIME UPDATE                                │
└──────────────────────────────────────────────────────────────────┘

Bid Consumer:
  // Publish to Redis Pub/Sub for real-time updates
  PUBLISH auction:auction_123:updates "{
    event: 'new_bid',
    auction_id: 'auction_123',
    amount: 550.00,
    bid_time: 1715548800,
    total_bids: 1
  }"

Real-time Update Servers (subscribed to channel):
  - Receive message
  - Broadcast to all SSE connections watching auction_123
  - Clients update UI in real-time

Latency: ~10ms (pub/sub + SSE)

Total Processing Time: ~55ms
Total End-to-End (from client): ~90ms ✓
```

### Flow 4: Retry on Conflict

```
┌──────────────────────────────────────────────────────────────────┐
│  SCENARIO: TWO BIDS ARRIVE SIMULTANEOUSLY                        │
└──────────────────────────────────────────────────────────────────┘

Timeline:
T+0ms:  User A bids $550 (current max: $500, version: 0)
T+5ms:  User B bids $560 (current max: $500, version: 0)

Both read version=0 from cache

T+10ms: User A's update executes
        UPDATE ... WHERE version = 0
        SUCCESS! version → 1, max_bid → $550

T+15ms: User B's update executes
        UPDATE ... WHERE version = 0
        FAILURE! (version is now 1, not 0)
        
User B's consumer:
  1. Detect conflict (ROW_COUNT = 0)
  2. Rollback transaction
  3. Re-read current state
     current_max_bid = $550
     current_version = 1
  4. Re-validate bid
     $560 > $550 + increment ✓
  5. Retry update
     UPDATE ... WHERE version = 1
     SUCCESS! version → 2, max_bid → $560

Result:
- User A: Bid accepted at $550, then immediately outbid
- User B: Bid accepted at $560, currently winning
- Both bids recorded in database
- Correct final state achieved
- No lost updates!

Retry latency: +50ms (acceptable)
```

### Critical Path Analysis:

```
Bid Placement → Acknowledgment (Fast Path):
  API Gateway:        15ms
  Bid Producer:       10ms
  Kafka Write:        10ms
  Response:           5ms
  ─────────────────────────
  Total:              40ms ✓ (< 100ms target)

Bid Processing → Real-time Update (Consistent Path):
  Kafka → Consumer:   5ms
  Read Cache:         2ms
  DB Update (OCC):    40ms
  Update Cache:       5ms
  Pub/Sub:            3ms
  SSE Broadcast:      5ms
  ─────────────────────────
  Total:              60ms ✓ (< 2s target)

End-to-End (Client → Real-time Update):
  Fast Path:          40ms
  Consistent Path:    60ms
  ─────────────────────────
  Total:              100ms ✓ (Excellent!)
```



---

## PHASE 8: DEEP DIVE - STRONG CONSISTENCY FOR BIDS (6 minutes)

### The Consistency Problem:

```
Race Condition Example:
Current max bid: $100

T+0ms:  User A reads max=$100, bids $150
T+5ms:  User B reads max=$100, bids $120
T+10ms: User A's bid accepted ($150)
T+15ms: User B's bid accepted ($120) ❌ WRONG!

Result: Both think they're winning, but $120 < $150
This violates auction integrity!
```

### Solution Comparison:

```
┌─────────────────────────────────────────────────────────────────┐
│  APPROACH 1: ROW LOCKING (❌ DON'T USE)                          │
├─────────────────────────────────────────────────────────────────┤
│  BEGIN;                                                          │
│  SELECT * FROM bids WHERE auction_id = X FOR UPDATE;            │
│  -- Locks ALL bid rows for auction                              │
│  SELECT MAX(amount) FROM bids WHERE auction_id = X;             │
│  INSERT INTO bids ...;                                           │
│  COMMIT;                                                         │
│                                                                  │
│  Problems:                                                       │
│  ✗ Locks many rows (100+ bids per auction)                      │
│  ✗ Doesn't prevent concurrent inserts                           │
│  ✗ Poor performance (serializes all bids)                       │
│  ✗ Doesn't scale                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  APPROACH 2: REDIS CACHE (⚠️  COMPLEX)                           │
├─────────────────────────────────────────────────────────────────┤
│  -- Lua script for atomic compare-and-set                       │
│  local current = redis.call('GET', 'auction:X:max')             │
│  if tonumber(new_bid) > tonumber(current) then                  │
│    redis.call('SET', 'auction:X:max', new_bid)                  │
│    return true                                                   │
│  end                                                             │
│  return false                                                    │
│                                                                  │
│  Problems:                                                       │
│  ⚠️  Cache-DB consistency issues                                 │
│  ⚠️  Need to sync Redis → DB                                     │
│  ⚠️  What if Redis fails?                                        │
│  ⚠️  Complex error handling                                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  APPROACH 3: OPTIMISTIC CONCURRENCY CONTROL (✓ BEST)            │
├─────────────────────────────────────────────────────────────────┤
│  -- Store max_bid and version in auctions table                 │
│  UPDATE auctions                                                 │
│  SET                                                             │
│    current_max_bid = 150.00,                                     │
│    version = version + 1                                         │
│  WHERE                                                           │
│    auction_id = 'X'                                              │
│    AND version = 5  -- Optimistic lock                          │
│    AND current_max_bid < 150.00;                                 │
│                                                                  │
│  IF (ROW_COUNT = 0) THEN                                         │
│    -- Conflict detected, retry                                  │
│  END IF;                                                         │
│                                                                  │
│  Benefits:                                                       │
│  ✓ Locks single row (auction)                                   │
│  ✓ Short lock duration                                          │
│  ✓ Scales horizontally                                          │
│  ✓ Database is source of truth                                  │
│  ✓ Automatic conflict detection                                 │
│  ✓ Retry on conflict                                            │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Details:

```python
def process_bid(bid):
    max_retries = 3
    for attempt in range(max_retries):
        # Read current state
        auction = get_auction_from_cache(bid.auction_id)
        
        # Validate bid
        if bid.amount <= auction.current_max_bid + auction.bid_increment:
            reject_bid(bid, "BID_TOO_LOW")
            return
        
        # Attempt atomic update
        success = update_auction_optimistic(
            auction_id=bid.auction_id,
            new_max_bid=bid.amount,
            new_winner=bid.user_id,
            expected_version=auction.version
        )
        
        if success:
            # Update succeeded
            insert_bid(bid, status="ACCEPTED")
            update_cache(auction_id, bid.amount, auction.version + 1)
            publish_update(auction_id, bid)
            return
        else:
            # Conflict - another bid won
            if attempt < max_retries - 1:
                time.sleep(0.01 * (2 ** attempt))  # Exponential backoff
                continue
            else:
                reject_bid(bid, "CONFLICT_MAX_RETRIES")
                return

def update_auction_optimistic(auction_id, new_max_bid, new_winner, expected_version):
    result = db.execute("""
        UPDATE auctions
        SET 
            current_max_bid = %s,
            current_winner_id = %s,
            version = version + 1,
            total_bids = total_bids + 1
        WHERE 
            auction_id = %s
            AND version = %s
            AND status = 'ACTIVE'
    """, (new_max_bid, new_winner, auction_id, expected_version))
    
    return result.rowcount > 0
```

---

## PHASE 9: DEEP DIVE - FAULT TOLERANCE & DURABILITY (4 minutes)

### The Durability Problem:

```
Without Message Queue:
Client → Bid Service → Database
              ↓ (crashes here)
           BID LOST! ❌

With Message Queue:
Client → Bid Service → Kafka → Bid Consumer → Database
              ↓                      ↓
         Acknowledged          Processes later
         BID SAFE! ✓           Even after crash
```

### Kafka Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│  KAFKA TOPIC: bids                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Partition 0: [auction_001, auction_011, auction_021, ...]     │
│  Partition 1: [auction_002, auction_012, auction_022, ...]     │
│  Partition 2: [auction_003, auction_013, auction_023, ...]     │
│  ...                                                            │
│  Partition 99: [auction_100, auction_200, auction_300, ...]    │
│                                                                 │
│  Partitioning: hash(auction_id) % 100                           │
│  Replication: 3 (across different brokers)                      │
│  Retention: 7 days                                              │
│                                                                 │
│  Guarantees:                                                    │
│  ✓ Ordering within partition (per auction)                     │
│  ✓ At-least-once delivery                                      │
│  ✓ Durable storage (replicated)                                │
│  ✓ Survives broker failures                                    │
└─────────────────────────────────────────────────────────────────┘

Consumer Groups:
┌──────────────────────────────────────────────────────────────┐
│  Consumer Group: bid-processors                              │
│                                                              │
│  Consumer 1: Partitions 0-19                                 │
│  Consumer 2: Partitions 20-39                                │
│  Consumer 3: Partitions 40-59                                │
│  Consumer 4: Partitions 60-79                                │
│  Consumer 5: Partitions 80-99                                │
│                                                              │
│  If Consumer 2 fails:                                        │
│  - Partitions 20-39 reassigned to other consumers            │
│  - No message loss                                           │
│  - Processing continues                                      │
└──────────────────────────────────────────────────────────────┘
```

### Failure Scenarios:

```
SCENARIO 1: Bid Producer Crashes
Before Kafka write: Bid lost, client gets error, can retry
After Kafka write: Bid safe, will be processed

SCENARIO 2: Kafka Broker Fails
Replication ensures no data loss
Automatic failover to replica
Processing continues

SCENARIO 3: Bid Consumer Crashes
Mid-processing: Message not committed, will be reprocessed
After DB write: Idempotency prevents duplicate

SCENARIO 4: Database Fails
Kafka retains messages
Consumers retry with exponential backoff
Processing resumes when DB recovers
```

---

## PHASE 10: DEEP DIVE - REAL-TIME UPDATES & SCALING (5 minutes)

### Real-time Update Technologies:

```
┌─────────────────────────────────────────────────────────────────┐
│  POLLING (❌ DON'T USE)                                          │
├─────────────────────────────────────────────────────────────────┤
│  Client polls every 2 seconds:                                  │
│  GET /auctions/X/max-bid                                         │
│                                                                  │
│  Problems:                                                       │
│  ✗ High latency (average 1 second)                              │
│  ✗ Wasted requests (95% return no change)                       │
│  ✗ Database load (11K QPS for 10M viewers)                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  LONG POLLING (⚠️  BETTER BUT NOT IDEAL)                         │
├─────────────────────────────────────────────────────────────────┤
│  Client opens connection, server holds until update              │
│                                                                  │
│  Benefits:                                                       │
│  ✓ Lower latency than polling                                   │
│  ✓ Fewer wasted requests                                        │
│                                                                  │
│  Problems:                                                       │
│  ⚠️  Still requires many connections                             │
│  ⚠️  Thundering herd on updates                                  │
│  ⚠️  Complex server coordination                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  SERVER-SENT EVENTS (✓ BEST CHOICE)                             │
├─────────────────────────────────────────────────────────────────┤
│  Persistent one-way connection from server to client            │
│                                                                  │
│  Benefits:                                                       │
│  ✓ True real-time (<2 second latency)                          │
│  ✓ Efficient (one connection per viewer)                        │
│  ✓ Built-in reconnection                                        │
│  ✓ Standard HTTP (works through proxies)                        │
│  ✓ Perfect for read-heavy (auctions are 90% viewing)           │
│                                                                  │
│  Architecture:                                                   │
│  Bid Consumer → Redis Pub/Sub → SSE Servers → Clients          │
└─────────────────────────────────────────────────────────────────┘
```

### Scaling Real-time Updates:

```
Problem: 10M concurrent viewers across 10M auctions

Solution: Redis Pub/Sub + Server Coordination

┌──────────────────────────────────────────────────────────────┐
│  When bid processed:                                         │
│  1. Bid Consumer publishes to Redis                          │
│     PUBLISH auction:auction_123:updates "{bid_data}"         │
│                                                              │
│  2. All SSE servers subscribed to that auction receive it    │
│                                                              │
│  3. SSE servers broadcast to their connected clients         │
└──────────────────────────────────────────────────────────────┘

Optimization: Channel Sharding
- Instead of 10M channels (one per auction)
- Use 1000 channels: hash(auction_id) % 1000
- SSE servers subscribe only to channels they need
- Reduces subscription overhead

Load Balancing:
- Use consistent hashing: hash(auction_id) → server
- Viewers of same auction go to same server
- Reduces cross-server coordination
```

### Database Sharding:

```
Shard Strategy: Hash(auction_id) % 10

Shard 0: Auctions 0, 10, 20, 30, ...
Shard 1: Auctions 1, 11, 21, 31, ...
...
Shard 9: Auctions 9, 19, 29, 39, ...

Benefits:
✓ Distributes write load (3,300 writes/sec per shard)
✓ No cross-shard queries needed
✓ Easy to add more shards
✓ Each shard can have read replicas

Routing:
def get_shard(auction_id):
    return hash(auction_id) % 10

All operations for an auction go to same shard
```

---

## FOLLOW-UP QUESTIONS

### Q1: "How would you implement dynamic auction end times?"

**Answer:**
```
Requirement: Extend auction by 5 minutes if bid in last 5 minutes

Implementation:
1. On each bid, check time_remaining
2. If time_remaining < 300 seconds:
   UPDATE auctions
   SET end_time = end_time + INTERVAL '5 minutes'
   WHERE auction_id = X

3. Publish update to SSE clients
4. Auction Ending Service handles new end_time

Alternative: Use Redis sorted set
ZADD auctions:ending_soon new_end_time auction_id
```

### Q2: "How would you handle payment processing?"

**Answer:**
```
After auction ends:
1. Auction Ending Service determines winner
2. Create payment intent (Stripe/PayPal)
3. Send email to winner with payment link
4. Winner has 24 hours to pay
5. If payment fails:
   - Offer to next highest bidder
   - Repeat until payment succeeds or no bidders left
6. Update auction status to PAID
7. Notify seller to ship item
```

### Q3: "How would you implement search and filtering?"

**Answer:**
```
Use Elasticsearch:
1. Index auctions on creation
2. Update on bid (current_max_bid)
3. Support queries:
   - Full-text search on title/description
   - Filter by category, price range, end time
   - Sort by price, end time, popularity

Query:
GET /auctions/_search
{
  "query": {
    "bool": {
      "must": [{"match": {"title": "camera"}}],
      "filter": [
        {"range": {"current_max_bid": {"lte": 1000}}},
        {"term": {"status": "ACTIVE"}}
      ]
    }
  },
  "sort": [{"end_time": "asc"}]
}
```

---

## KEY TAKEAWAYS

### What Makes This Design Strong:

```
1. Strong Consistency (Optimistic Concurrency Control)
   ✓ Prevents race conditions
   ✓ Scales horizontally
   ✓ Minimal lock contention
   ✓ Automatic conflict detection

2. Durability (Kafka Message Queue)
   ✓ No lost bids
   ✓ Survives crashes
   ✓ At-least-once delivery
   ✓ Ordered processing per auction

3. Real-time Updates (SSE + Redis Pub/Sub)
   ✓ <2 second latency
   ✓ Efficient for read-heavy workload
   ✓ Scales to millions of viewers
   ✓ Built-in reconnection

4. Horizontal Scalability
   ✓ Database sharding by auction_id
   ✓ Kafka partitioning
   ✓ Stateless services
   ✓ Redis clustering
```

### Common Mistakes to Avoid:

```
❌ Using row locking on bids table (doesn't scale)
❌ Not using message queue (bids can be lost)
❌ Polling for real-time updates (high latency, wasteful)
❌ Not handling concurrent bids (race conditions)
❌ Storing max bid only (no audit trail)
❌ Not sharding database (write bottleneck)
❌ Synchronous processing (slow, not fault-tolerant)
```

### Interview Presentation Strategy:

```
1. Start Simple (5 min)
   - Basic CRUD for auctions
   - Simple bid placement
   - Explain the consistency problem

2. Add Durability (5 min)
   - Introduce Kafka
   - Explain fast path vs consistent path
   - Show how bids are never lost

3. Add Consistency (10 min)
   - Compare locking approaches
   - Explain optimistic concurrency control
   - Walk through conflict resolution

4. Add Real-time (5 min)
   - Compare polling vs SSE
   - Show Redis Pub/Sub architecture
   - Explain scaling strategy

5. Deep Dives (15 min)
   - Database sharding
   - Failure scenarios
   - Performance optimization
```

---

## CONCLUSION

This Online Auction System demonstrates a production-ready architecture that:

**Ensures Strong Consistency:**
- Optimistic concurrency control prevents race conditions
- Version-based locking scales horizontally
- Automatic conflict detection and retry
- Database is single source of truth

**Guarantees Durability:**
- Kafka ensures no bid loss
- At-least-once delivery semantics
- Survives service crashes
- Ordered processing per auction

**Provides Real-time Experience:**
- SSE for efficient push updates
- <2 second latency for bid updates
- Scales to millions of concurrent viewers
- Graceful reconnection handling

**Scales Horizontally:**
- Database sharding by auction_id (10 shards)
- Kafka partitioning (100 partitions)
- Stateless services (auto-scaling)
- Redis clustering for cache

**Key Architectural Insights:**
1. **Separate fast path (durability) from consistent path (processing)**
2. **Optimistic concurrency control beats pessimistic locking at scale**
3. **Message queue is critical for durability and decoupling**
4. **SSE perfect for read-heavy real-time updates**
5. **Shard by auction_id for natural data locality**

The design handles 10M concurrent auctions, 16K bids/second peak, and 10M concurrent viewers while maintaining strong consistency, durability, and real-time updates with <100ms latency.

