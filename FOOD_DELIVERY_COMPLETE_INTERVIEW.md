# Food Delivery Platform (Uber Eats/DoorDash) - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: High-Level Architecture (8 min)
Phase 5: API Design (4 min)
Phase 6: Data Models (5 min)
Phase 7: Core Algorithm - Delivery Assignment (6 min)
Phase 8: Deep Dive - Real-Time Tracking (4 min)
Phase 9: Follow-up Questions (5 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"Thank you for the problem. I want to design a food delivery platform like Uber Eats. This involves three actors: customers, restaurants, and delivery drivers. Let me clarify the requirements."

### Questions to Ask:

**Q1: Core Features Scope**
- "Should I focus on all three flows: restaurant management, order placement, and delivery tracking?"
- "Are we building the driver app as well, or just the customer-facing platform?"

**Expected Answer:** All three flows, including driver app

**Q2: Restaurant Management**
- "Can restaurants register themselves, or is there an onboarding process?"
- "Should restaurants be able to update menus in real-time?"
- "Do we need to handle restaurant operating hours and availability?"

**Expected Answer:** Self-registration, real-time menu updates, operating hours needed

**Q3: Search & Discovery**
- "Should customers search by cuisine type, restaurant name, or specific dishes?"
- "Do we need filters (price range, ratings, delivery time)?"
- "Should we show nearby restaurants based on customer location?"

**Expected Answer:** All of the above, location-based search is critical

**Q4: Order Management**
- "Can customers modify orders after placement? What's the cutoff?"
- "Should we support scheduled orders (order now, deliver later)?"
- "What happens if a restaurant rejects an order?"

**Expected Answer:** Modify before restaurant accepts, scheduled orders nice-to-have, handle rejections

**Q5: Delivery Assignment - THE KEY QUESTION**
- "How should we assign drivers? Nearest available, or consider other factors?"
- "Should drivers be able to reject orders?"
- "Can one driver handle multiple orders (batching)?"

**Expected Answer:** Smart assignment (distance + availability), drivers can reject, batching is follow-up

**Q6: Real-Time Tracking**
- "Should customers see driver location in real-time?"
- "What's the acceptable latency for location updates?"
- "Should restaurants also see driver location?"

**Expected Answer:** Yes real-time, < 5 seconds latency, restaurants see it too

**Q7: Scale & Geography**
- "How many cities are we launching in? Single city or multiple?"
- "Expected number of restaurants, customers, drivers?"
- "Peak vs average traffic patterns?"

**Expected Answer:** Multiple cities, 100K restaurants, 10M customers, 500K drivers, lunch/dinner peaks

**Q8: Payments**
- "Should we handle payments, or integrate with existing gateway?"
- "Support for multiple payment methods?"
- "When do we charge: order placement or delivery?"

**Expected Answer:** Integrate with gateway, multiple methods, charge at placement

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### What to Say:

"Based on our discussion, let me summarize the requirements in priority order."

### Functional Requirements (Priority Order):

```
1. Restaurant Management (CORE)
   - Restaurant registration and onboarding
   - Menu management (add/update/delete items)
   - Set operating hours and availability
   - Real-time order notifications
   - Accept/reject orders

2. Search & Discovery (CORE)
   - Search nearby restaurants (geo-based)
   - Search by cuisine, dish name, restaurant name
   - Filter by price, rating, delivery time, dietary preferences
   - Sort by distance, rating, delivery time, popularity

3. Order Management (CORE)
   - Place order with multiple items
   - Modify order (before restaurant accepts)
   - Cancel order (with refund logic)
   - Real-time order status updates
   - Order history

4. Delivery Assignment (CRITICAL)
   - Automatically assign nearest available driver
   - Consider driver rating, distance, current load
   - Handle driver rejection (reassign)
   - Optimize for delivery time

5. Real-Time Tracking (CORE)
   - Track driver location (< 5 sec updates)
   - Estimated delivery time (ETA)
   - Push notifications for status changes
   - In-app chat (customer ↔ driver)

6. Payments & Checkout (CORE)
   - Multiple payment methods (card, wallet, cash)
   - Apply promo codes and discounts
   - Calculate delivery fee dynamically
   - Handle refunds for cancellations

7. Rating & Review System
   - Rate food quality (1-5 stars)
   - Rate delivery experience
   - Write reviews with photos
   - Display aggregate ratings

Out of Scope:
- Restaurant analytics dashboard (separate system)
- Driver earnings/payouts (covered in seller payment system)
- Fraud detection (mention briefly)
- Multi-restaurant orders (complex, follow-up)
```

### Non-Functional Requirements:

```
1. Performance
   - Search latency: < 200ms
   - Order placement: < 1 second
   - Location updates: < 5 seconds
   - API response time: < 100ms (p99)

2. Scalability
   - 10M daily active customers
   - 100K active restaurants
   - 500K active drivers
   - 50M orders/day
   - 10K orders/second (peak)

3. Availability
   - 99.99% uptime for order placement
   - 99.9% for search/discovery
   - Graceful degradation (show cached results)

4. Consistency
   - Strong consistency for orders (ACID)
   - Eventual consistency for restaurant menus
   - Real-time consistency for driver location

5. Reliability
   - No order loss (durable storage)
   - Exactly-once order processing
   - Retry mechanisms for failures
   - Idempotent APIs

6. Low Latency (Geo-distributed)
   - Serve from nearest datacenter
   - CDN for static content
   - Edge caching for restaurant data

7. Security
   - PCI-DSS compliance for payments
   - Encrypted data in transit and at rest
   - Rate limiting (prevent abuse)
   - Authentication & authorization
```

### Why This Matters:
✓ Shows understanding of multi-actor system
✓ Prioritizes critical features
✓ Identifies key technical challenges
✓ Sets clear performance expectations

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me calculate the scale to validate our design decisions."

### Traffic Estimation:

```
Given:
- 10M daily active customers
- 100K active restaurants
- 500K active drivers
- Average 5 orders per customer per month
- Peak hours: 12-2 PM, 7-9 PM (4 hours/day)

Calculations:

1. Orders
   Daily orders: 10M customers × 5 orders/month ÷ 30 days
              = ~1.67M orders/day
   
   Peak orders: 60% of daily orders in 4 peak hours
              = 1M orders in 4 hours
              = 250K orders/hour
              = ~70 orders/second (peak)
   
   Average: 1.67M / 86,400 = ~19 orders/second

2. Search Requests (Read-Heavy)
   Each customer searches 10 times before ordering
   Daily searches: 10M × 10 = 100M searches/day
   
   Average: 100M / 86,400 = ~1,160 searches/second
   Peak: 1,160 × 3 = ~3,500 searches/second

3. Location Updates (Write-Heavy)
   Active drivers during peak: 100K (20% of 500K)
   Update frequency: Every 5 seconds
   
   Location updates: 100K / 5 = 20,000 updates/second
   
   This is the REAL bottleneck!

4. Real-Time Tracking (Read)
   Active orders being tracked: 100K (at any moment)
   Customers checking location: Every 10 seconds
   
   Tracking requests: 100K / 10 = 10,000 requests/second
```

### Storage Estimation:

```
1. Restaurants
   - 100K restaurants
   - Each: 10 KB (name, address, menu, images URLs)
   - Total: 100K × 10KB = 1 GB

2. Menu Items
   - 100K restaurants × 50 items average
   - 5M menu items
   - Each: 2 KB (name, description, price, image)
   - Total: 5M × 2KB = 10 GB

3. Users (Customers)
   - 50M total users (10M DAU)
   - Each: 1 KB (profile, address, payment methods)
   - Total: 50M × 1KB = 50 GB

4. Drivers
   - 500K drivers
   - Each: 2 KB (profile, vehicle, documents)
   - Total: 500K × 2KB = 1 GB

5. Orders
   - 1.67M orders/day
   - Retention: 2 years
   - Total orders: 1.67M × 365 × 2 = 1.22B orders
   - Each: 5 KB (items, prices, addresses, status)
   - Total: 1.22B × 5KB = 6.1 TB

6. Driver Locations (Time-Series)
   - 100K active drivers
   - Update every 5 seconds
   - Retention: 7 days
   - Points per driver: (86,400 / 5) × 7 = 120K points
   - Total points: 100K × 120K = 12B points
   - Each: 50 bytes (lat, lng, timestamp, driver_id)
   - Total: 12B × 50B = 600 GB

7. Reviews & Ratings
   - 50% of orders get reviewed
   - 1.22B orders × 0.5 = 610M reviews
   - Each: 1 KB (rating, text, photos)
   - Total: 610M × 1KB = 610 GB

Total Storage: ~7.5 TB
```

### Bandwidth Estimation:

```
1. Search API (Outgoing)
   - 3,500 searches/sec × 50 restaurants × 2KB
   = 350 MB/sec
   = ~30 TB/day
   
   With CDN caching (80% hit rate):
   = 6 TB/day

2. Location Updates (Incoming)
   - 20,000 updates/sec × 50 bytes
   = 1 MB/sec
   = ~86 GB/day

3. Real-Time Tracking (Outgoing)
   - 10,000 requests/sec × 100 bytes
   = 1 MB/sec
   = ~86 GB/day

Total Bandwidth: ~7 TB/day (mostly search results)
```

### Cost Estimation:

```
1. Compute (EC2)
   - API servers: 100 instances (m5.large)
     Cost: $7,200/month
   
   - Location service: 50 instances (c5.xlarge)
     Cost: $6,000/month
   
   - Background workers: 20 instances
     Cost: $1,440/month

2. Database (RDS PostgreSQL)
   - Primary: db.r5.4xlarge (Multi-AZ)
     Cost: $4,000/month
   
   - Read replicas: 3 × db.r5.2xlarge
     Cost: $6,000/month

3. Cache (ElastiCache Redis)
   - 5 nodes (cache.r5.xlarge)
     Cost: $1,500/month

4. Storage (S3)
   - 7.5 TB × $0.023/GB
     Cost: $172/month

5. CDN (CloudFront)
   - 6 TB/day × 30 days × $0.085/GB
     Cost: $15,300/month

6. Location Database (TimescaleDB/InfluxDB)
   - Managed service
     Cost: $2,000/month

7. Message Queue (Kafka/SQS)
   - Cost: $500/month

Total Infrastructure: ~$44,000/month
```

### Summary Table:

```
┌─────────────────────────┬──────────────────┐
│ Metric                  │ Value            │
├─────────────────────────┼──────────────────┤
│ Customers (DAU)         │ 10M              │
│ Restaurants             │ 100K             │
│ Drivers                 │ 500K             │
│ Orders/day              │ 1.67M            │
│ Peak order QPS          │ 70               │
│ Search QPS (peak)       │ 3,500            │
│ Location updates/sec    │ 20,000           │
│ Storage                 │ 7.5 TB           │
│ Bandwidth               │ 7 TB/day         │
│ Monthly cost            │ $44,000          │
└─────────────────────────┴──────────────────┘

Key Insights:
1. Location updates (20K/sec) >> Orders (70/sec)
2. Search is read-heavy, needs aggressive caching
3. Orders need strong consistency (ACID)
4. Location data is time-series (specialized DB)
```

### Why This Matters:
✓ Identifies real bottleneck (location updates)
✓ Shows cost awareness
✓ Validates technology choices
✓ Demonstrates scale understanding



---

## PHASE 4: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"Let me present the high-level architecture. I'll organize it by the three main actors: customers, restaurants, and drivers."

### Complete Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENT LAYER                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Customer    │  │  Restaurant  │  │   Driver     │         │
│  │  App         │  │  Dashboard   │  │   App        │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CDN (CloudFront)                              │
│  - Static assets, restaurant images                             │
│  - Cache search results (5 min TTL)                             │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              API GATEWAY / LOAD BALANCER                         │
│  - Rate limiting, Authentication, SSL termination               │
│  - Route to appropriate service                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┬──────────────┐
          ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  APPLICATION SERVICES LAYER                      │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  1. RESTAURANT SERVICE                                 │    │
│  │     - Restaurant CRUD                                  │    │
│  │     - Menu management                                  │    │
│  │     - Operating hours                                  │    │
│  │     - Accept/reject orders                             │    │
│  │     Technology: Java/Spring Boot                       │    │
│  │     Instances: 20 (auto-scaling)                       │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  2. SEARCH SERVICE                                     │    │
│  │     - Geo-spatial search (nearby restaurants)          │    │
│  │     - Full-text search (dishes, restaurants)           │    │
│  │     - Filters & sorting                                │    │
│  │     Technology: Elasticsearch                          │    │
│  │     Instances: 30 (read-heavy)                         │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  3. ORDER SERVICE                                      │    │
│  │     - Place order                                      │    │
│  │     - Order state machine                              │    │
│  │     - Modify/cancel order                              │    │
│  │     - Order history                                    │    │
│  │     Technology: Java/Spring Boot                       │    │
│  │     Instances: 50 (critical path)                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  4. DELIVERY SERVICE (THE CORE)                        │    │
│  │     - Driver assignment algorithm                      │    │
│  │     - ETA calculation                                  │    │
│  │     - Route optimization                               │    │
│  │     - Handle driver rejection                          │    │
│  │     Technology: Go (performance critical)              │    │
│  │     Instances: 40                                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  5. LOCATION SERVICE                                   │    │
│  │     - Ingest driver locations (20K/sec)                │    │
│  │     - Real-time tracking                               │    │
│  │     - Geo-queries (find nearby drivers)                │    │
│  │     Technology: Go + Redis Geo                         │    │
│  │     Instances: 50 (high throughput)                    │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  6. PAYMENT SERVICE                                    │    │
│  │     - Process payments                                 │    │
│  │     - Handle refunds                                   │    │
│  │     - Apply promo codes                                │    │
│  │     Technology: Java/Spring Boot                       │    │
│  │     Instances: 30                                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  7. NOTIFICATION SERVICE                               │    │
│  │     - Push notifications (FCM/APNS)                    │    │
│  │     - SMS notifications                                │    │
│  │     - Email notifications                              │    │
│  │     Technology: Node.js                                │    │
│  │     Instances: 20                                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  8. RATING & REVIEW SERVICE                            │    │
│  │     - Submit ratings                                   │    │
│  │     - Aggregate scores                                 │    │
│  │     - Moderation queue                                 │    │
│  │     Technology: Python/Django                          │    │
│  │     Instances: 10                                      │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┬──────────────┐
          ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  PRIMARY DATABASE (PostgreSQL)                         │    │
│  │  - Restaurants, Menus, Users, Drivers                  │    │
│  │  - Orders (ACID transactions)                          │    │
│  │  - Payments                                            │    │
│  │  Configuration: Multi-AZ, 3 read replicas             │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  SEARCH INDEX (Elasticsearch)                          │    │
│  │  - Restaurant search                                   │    │
│  │  - Menu item search                                    │    │
│  │  - Geo-spatial index                                   │    │
│  │  Configuration: 5-node cluster                         │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  CACHE (Redis Cluster)                                 │    │
│  │  - Restaurant data (hot data)                          │    │
│  │  - Menu items                                          │    │
│  │  - User sessions                                       │    │
│  │  - Driver locations (Redis Geo)                        │    │
│  │  Configuration: 5 shards, 2 replicas each             │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  TIME-SERIES DB (TimescaleDB)                          │    │
│  │  - Driver location history                             │    │
│  │  - Order tracking events                               │    │
│  │  - Analytics data                                      │    │
│  │  Retention: 7 days                                     │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  OBJECT STORAGE (S3)                                   │    │
│  │  - Restaurant images                                   │    │
│  │  - Menu item photos                                    │    │
│  │  - Review photos                                       │    │
│  │  - Driver documents                                    │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  MESSAGING & STREAMING LAYER                     │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  MESSAGE QUEUE (Kafka)                                 │    │
│  │  Topics:                                               │    │
│  │  - order.placed                                        │    │
│  │  - order.accepted                                      │    │
│  │  - order.ready                                         │    │
│  │  - order.picked_up                                     │    │
│  │  - order.delivered                                     │    │
│  │  - driver.location_update                              │    │
│  │  Configuration: 3 brokers, replication factor 3        │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  WEBSOCKET SERVER (Socket.io)                          │    │
│  │  - Real-time order updates                             │    │
│  │  - Live location tracking                              │    │
│  │  - In-app chat                                         │    │
│  │  Instances: 30 (stateful, sticky sessions)            │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  EXTERNAL INTEGRATIONS                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Payment     │  │  Maps API    │  │  SMS/Push    │         │
│  │  Gateway     │  │  (Google)    │  │  (Twilio/FCM)│         │
│  │  (Stripe)    │  │              │  │              │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

---

### Data Flow: Complete Order Journey

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: SEARCH & DISCOVERY                                     │
└─────────────────────────────────────────────────────────────────┘

1. Customer opens app
   ↓
2. App sends location: { lat: 37.7749, lng: -122.4194 }
   ↓
3. Search Service queries Elasticsearch:
   - Geo-spatial query (within 5 km radius)
   - Filter: open now, rating > 3.5
   - Sort by: distance, rating
   ↓
4. Returns 50 restaurants (paginated)
   ↓
5. Customer searches "pizza"
   ↓
6. Full-text search on menu items
   ↓
7. Returns restaurants with pizza

Latency: < 200ms

┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2: ORDER PLACEMENT                                        │
└─────────────────────────────────────────────────────────────────┘

1. Customer adds items to cart
   - Stored in Redis (session)
   ↓
2. Customer clicks "Place Order"
   ↓
3. Order Service:
   BEGIN TRANSACTION
     a) Validate cart (items available, prices correct)
     b) Create order record (status = PENDING)
     c) Reserve inventory (if applicable)
     d) Calculate total (items + tax + delivery fee)
   COMMIT
   ↓
4. Payment Service:
   - Charge customer
   - If success: Update order status = PAYMENT_CONFIRMED
   - If failure: Cancel order, refund
   ↓
5. Publish to Kafka: order.placed
   ↓
6. Notification Service:
   - Send push to restaurant
   - Send confirmation to customer
   ↓
7. Restaurant receives order notification

Latency: < 1 second

┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3: RESTAURANT ACCEPTS ORDER                               │
└─────────────────────────────────────────────────────────────────┘

1. Restaurant clicks "Accept"
   ↓
2. Order Service:
   - Update status = ACCEPTED
   - Set prep_time = 20 minutes
   ↓
3. Publish to Kafka: order.accepted
   ↓
4. Delivery Service (THE CRITICAL PART):
   - Trigger driver assignment algorithm
   - Find best driver (details in Phase 7)
   ↓
5. Assign driver
   - Update order: driver_id = driver-123
   - Update driver: current_order_id = order-456
   ↓
6. Notification Service:
   - Notify customer: "Driver assigned"
   - Notify driver: "New order"
   ↓
7. Driver accepts assignment

Latency: < 5 seconds

┌─────────────────────────────────────────────────────────────────┐
│  PHASE 4: FOOD PREPARATION & PICKUP                              │
└─────────────────────────────────────────────────────────────────┘

1. Restaurant prepares food
   ↓
2. Restaurant marks "Ready for Pickup"
   - Update status = READY
   ↓
3. Publish to Kafka: order.ready
   ↓
4. Notify driver: "Food is ready"
   ↓
5. Driver arrives at restaurant
   ↓
6. Driver marks "Picked Up"
   - Update status = PICKED_UP
   - Start tracking
   ↓
7. Publish to Kafka: order.picked_up
   ↓
8. Notify customer: "Driver picked up your order"

┌─────────────────────────────────────────────────────────────────┐
│  PHASE 5: DELIVERY & TRACKING                                    │
└─────────────────────────────────────────────────────────────────┘

1. Driver app sends location every 5 seconds
   ↓
2. Location Service:
   - Store in Redis Geo: GEOADD drivers lat lng driver-123
   - Store in TimescaleDB (history)
   - Publish to WebSocket server
   ↓
3. Customer app receives location updates
   - Display on map
   - Update ETA
   ↓
4. Driver arrives at customer location
   ↓
5. Driver marks "Delivered"
   - Update status = DELIVERED
   - Capture delivery photo (optional)
   ↓
6. Publish to Kafka: order.delivered
   ↓
7. Notify customer: "Order delivered"
   ↓
8. Prompt for rating

Latency: Location updates < 5 seconds

┌─────────────────────────────────────────────────────────────────┐
│  PHASE 6: POST-DELIVERY                                          │
└─────────────────────────────────────────────────────────────────┘

1. Customer rates food & delivery
   ↓
2. Rating Service:
   - Store rating
   - Update aggregate scores
   - Trigger review moderation (if text)
   ↓
3. Payment Service:
   - Release payment to restaurant
   - Calculate driver earnings
   ↓
4. Order complete
```

---

### Why This Architecture Works:

**1. Separation of Concerns**
✅ Each service has single responsibility
✅ Can scale independently
✅ Easy to maintain and debug

**2. Event-Driven**
✅ Kafka for async communication
✅ Loose coupling between services
✅ Easy to add new consumers

**3. Real-Time Capabilities**
✅ WebSocket for live updates
✅ Redis for fast location queries
✅ TimescaleDB for location history

**4. Scalability**
✅ Stateless services (horizontal scaling)
✅ Read replicas for read-heavy workloads
✅ Caching at multiple layers

**5. Reliability**
✅ ACID transactions for orders
✅ Message queue for durability
✅ Multi-AZ database setup



---

## PHASE 5: API DESIGN (4 minutes)

### What to Say:

"Let me define the key API endpoints for each actor."

### Customer APIs:

```
1. Search Restaurants
GET /api/v1/restaurants/search?lat=37.7749&lng=-122.4194&radius=5000&cuisine=italian&sort=rating

Response: 200 OK
{
  "restaurants": [
    {
      "id": "rest-123",
      "name": "Pizza Palace",
      "cuisine": ["Italian", "Pizza"],
      "rating": 4.5,
      "delivery_time": "25-35 min",
      "delivery_fee": 2.99,
      "distance": 1.2,
      "is_open": true,
      "image_url": "https://cdn.../pizza-palace.jpg"
    }
  ],
  "pagination": {
    "page": 1,
    "total_pages": 10,
    "total_count": 487
  }
}

2. Get Restaurant Details
GET /api/v1/restaurants/{restaurantId}

Response: 200 OK
{
  "id": "rest-123",
  "name": "Pizza Palace",
  "menu": [
    {
      "category": "Pizzas",
      "items": [
        {
          "id": "item-456",
          "name": "Margherita",
          "description": "Classic tomato and mozzarella",
          "price": 12.99,
          "image_url": "...",
          "customizations": [
            {
              "name": "Size",
              "options": ["Small", "Medium", "Large"],
              "required": true
            }
          ]
        }
      ]
    }
  ],
  "operating_hours": {
    "monday": {"open": "11:00", "close": "22:00"},
    ...
  }
}

3. Place Order
POST /api/v1/orders

Request:
{
  "restaurant_id": "rest-123",
  "items": [
    {
      "item_id": "item-456",
      "quantity": 2,
      "customizations": {"size": "Large"},
      "special_instructions": "Extra cheese"
    }
  ],
  "delivery_address": {
    "street": "123 Main St",
    "city": "San Francisco",
    "lat": 37.7749,
    "lng": -122.4194
  },
  "payment_method_id": "pm-789",
  "promo_code": "SAVE10"
}

Response: 201 Created
{
  "order_id": "order-999",
  "status": "PENDING",
  "total": 28.97,
  "estimated_delivery_time": "2024-01-15T13:30:00Z"
}

4. Track Order
GET /api/v1/orders/{orderId}/track

Response: 200 OK
{
  "order_id": "order-999",
  "status": "PICKED_UP",
  "driver": {
    "id": "driver-123",
    "name": "John Doe",
    "phone": "+1234567890",
    "rating": 4.8,
    "vehicle": "Honda Civic - ABC123",
    "current_location": {
      "lat": 37.7750,
      "lng": -122.4195
    }
  },
  "eta": "2024-01-15T13:25:00Z",
  "timeline": [
    {"status": "PLACED", "timestamp": "2024-01-15T12:45:00Z"},
    {"status": "ACCEPTED", "timestamp": "2024-01-15T12:46:00Z"},
    {"status": "READY", "timestamp": "2024-01-15T13:05:00Z"},
    {"status": "PICKED_UP", "timestamp": "2024-01-15T13:10:00Z"}
  ]
}

5. Cancel Order
POST /api/v1/orders/{orderId}/cancel

Request:
{
  "reason": "Changed my mind"
}

Response: 200 OK
{
  "order_id": "order-999",
  "status": "CANCELLED",
  "refund_amount": 28.97,
  "refund_eta": "3-5 business days"
}
```

### Restaurant APIs:

```
1. Get Pending Orders
GET /api/v1/restaurant/orders?status=PENDING

Response: 200 OK
{
  "orders": [
    {
      "order_id": "order-999",
      "customer_name": "Jane Smith",
      "items": [...],
      "total": 28.97,
      "placed_at": "2024-01-15T12:45:00Z",
      "delivery_address": "123 Main St"
    }
  ]
}

2. Accept Order
POST /api/v1/restaurant/orders/{orderId}/accept

Request:
{
  "prep_time_minutes": 20
}

Response: 200 OK
{
  "order_id": "order-999",
  "status": "ACCEPTED",
  "ready_by": "2024-01-15T13:05:00Z"
}

3. Reject Order
POST /api/v1/restaurant/orders/{orderId}/reject

Request:
{
  "reason": "Out of ingredients"
}

Response: 200 OK

4. Mark Ready
POST /api/v1/restaurant/orders/{orderId}/ready

Response: 200 OK

5. Update Menu
PUT /api/v1/restaurant/menu/items/{itemId}

Request:
{
  "name": "Margherita Pizza",
  "price": 13.99,
  "available": true
}

Response: 200 OK
```

### Driver APIs:

```
1. Get Available Orders
GET /api/v1/driver/orders/available?lat=37.7749&lng=-122.4194

Response: 200 OK
{
  "orders": [
    {
      "order_id": "order-999",
      "restaurant": {
        "name": "Pizza Palace",
        "address": "456 Oak St",
        "lat": 37.7748,
        "lng": -122.4193
      },
      "delivery_address": {
        "street": "123 Main St",
        "lat": 37.7749,
        "lng": -122.4194
      },
      "distance": 1.2,
      "estimated_earnings": 8.50,
      "pickup_by": "2024-01-15T13:05:00Z"
    }
  ]
}

2. Accept Order
POST /api/v1/driver/orders/{orderId}/accept

Response: 200 OK

3. Update Location
POST /api/v1/driver/location

Request:
{
  "lat": 37.7750,
  "lng": -122.4195,
  "heading": 45,
  "speed": 25
}

Response: 204 No Content

4. Mark Picked Up
POST /api/v1/driver/orders/{orderId}/picked-up

Response: 200 OK

5. Mark Delivered
POST /api/v1/driver/orders/{orderId}/delivered

Request:
{
  "delivery_photo_url": "https://s3.../proof.jpg",
  "notes": "Left at door"
}

Response: 200 OK
```

---

## PHASE 6: DATA MODELS (5 minutes)

### What to Say:

"Let me define the database schemas. I'll use PostgreSQL for transactional data and specialized databases for specific use cases."

### 1. Restaurants Table

```sql
CREATE TABLE restaurants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    cuisine_types VARCHAR(50)[] NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(255),
    
    -- Address
    street VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50) NOT NULL,
    zip_code VARCHAR(10) NOT NULL,
    country VARCHAR(50) NOT NULL DEFAULT 'US',
    
    -- Geo-location
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    
    -- Business info
    is_active BOOLEAN DEFAULT true,
    is_accepting_orders BOOLEAN DEFAULT true,
    average_prep_time_minutes INT DEFAULT 30,
    
    -- Ratings
    rating DECIMAL(3, 2) DEFAULT 0.0,
    total_ratings INT DEFAULT 0,
    
    -- Metadata
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT chk_rating CHECK (rating >= 0 AND rating <= 5)
);

CREATE INDEX idx_restaurants_location ON restaurants USING GIST (
    ll_to_earth(latitude, longitude)
);
CREATE INDEX idx_restaurants_cuisine ON restaurants USING GIN (cuisine_types);
CREATE INDEX idx_restaurants_active ON restaurants (is_active, is_accepting_orders);
```

### 2. Menu Items Table

```sql
CREATE TABLE menu_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    restaurant_id UUID NOT NULL REFERENCES restaurants(id),
    
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(100) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    
    image_url VARCHAR(500),
    is_available BOOLEAN DEFAULT true,
    is_vegetarian BOOLEAN DEFAULT false,
    is_vegan BOOLEAN DEFAULT false,
    
    -- Customizations (JSON)
    customizations JSONB,
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT chk_price_positive CHECK (price > 0)
);

CREATE INDEX idx_menu_restaurant ON menu_items(restaurant_id);
CREATE INDEX idx_menu_category ON menu_items(category);
CREATE INDEX idx_menu_available ON menu_items(is_available);
```

### 3. Users Table

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    phone VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE,
    name VARCHAR(255) NOT NULL,
    
    -- Default address
    default_address_id UUID,
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_phone ON users(phone);
CREATE INDEX idx_users_email ON users(email);
```

### 4. Addresses Table

```sql
CREATE TABLE addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    
    label VARCHAR(50),
    street VARCHAR(255) NOT NULL,
    apartment VARCHAR(50),
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50) NOT NULL,
    zip_code VARCHAR(10) NOT NULL,
    
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    
    delivery_instructions TEXT,
    
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_addresses_user ON addresses(user_id);
```

### 5. Drivers Table

```sql
CREATE TABLE drivers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    name VARCHAR(255) NOT NULL,
    phone VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE,
    
    -- Vehicle info
    vehicle_type VARCHAR(50) NOT NULL,
    vehicle_make VARCHAR(100),
    vehicle_model VARCHAR(100),
    vehicle_plate VARCHAR(20),
    
    -- Status
    is_active BOOLEAN DEFAULT true,
    is_available BOOLEAN DEFAULT false,
    current_order_id UUID,
    
    -- Ratings
    rating DECIMAL(3, 2) DEFAULT 5.0,
    total_ratings INT DEFAULT 0,
    
    -- Earnings
    total_deliveries INT DEFAULT 0,
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT chk_driver_rating CHECK (rating >= 0 AND rating <= 5)
);

CREATE INDEX idx_drivers_available ON drivers(is_available, is_active);
CREATE INDEX idx_drivers_current_order ON drivers(current_order_id);
```

### 6. Orders Table (THE CORE)

```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Relationships
    customer_id UUID NOT NULL REFERENCES users(id),
    restaurant_id UUID NOT NULL REFERENCES restaurants(id),
    driver_id UUID REFERENCES drivers(id),
    
    -- Status
    status VARCHAR(20) NOT NULL,
    -- PENDING, PAYMENT_CONFIRMED, ACCEPTED, PREPARING, READY, 
    -- ASSIGNED, PICKED_UP, DELIVERED, CANCELLED
    
    -- Addresses
    delivery_address_id UUID NOT NULL REFERENCES addresses(id),
    delivery_lat DECIMAL(10, 8) NOT NULL,
    delivery_lng DECIMAL(11, 8) NOT NULL,
    
    -- Items (denormalized for performance)
    items JSONB NOT NULL,
    
    -- Pricing
    subtotal DECIMAL(10, 2) NOT NULL,
    tax DECIMAL(10, 2) NOT NULL,
    delivery_fee DECIMAL(10, 2) NOT NULL,
    discount DECIMAL(10, 2) DEFAULT 0,
    total DECIMAL(10, 2) NOT NULL,
    
    -- Timing
    placed_at TIMESTAMP DEFAULT NOW(),
    accepted_at TIMESTAMP,
    ready_at TIMESTAMP,
    picked_up_at TIMESTAMP,
    delivered_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    
    estimated_prep_time_minutes INT,
    estimated_delivery_time TIMESTAMP,
    
    -- Payment
    payment_method_id VARCHAR(100),
    payment_status VARCHAR(20) DEFAULT 'PENDING',
    transaction_id VARCHAR(100),
    
    -- Special instructions
    special_instructions TEXT,
    cancellation_reason TEXT,
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT chk_order_status CHECK (status IN (
        'PENDING', 'PAYMENT_CONFIRMED', 'ACCEPTED', 'PREPARING',
        'READY', 'ASSIGNED', 'PICKED_UP', 'DELIVERED', 'CANCELLED'
    ))
);

CREATE INDEX idx_orders_customer ON orders(customer_id, created_at DESC);
CREATE INDEX idx_orders_restaurant ON orders(restaurant_id, status);
CREATE INDEX idx_orders_driver ON orders(driver_id, status);
CREATE INDEX idx_orders_status ON orders(status, created_at DESC);
CREATE INDEX idx_orders_placed_at ON orders(placed_at DESC);
```

### 7. Order Items Table (Normalized)

```sql
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    menu_item_id UUID NOT NULL REFERENCES menu_items(id),
    
    quantity INT NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    customizations JSONB,
    special_instructions TEXT,
    
    subtotal DECIMAL(10, 2) NOT NULL,
    
    CONSTRAINT chk_quantity_positive CHECK (quantity > 0)
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
```

### 8. Ratings & Reviews Table

```sql
CREATE TABLE ratings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id),
    user_id UUID NOT NULL REFERENCES users(id),
    
    -- Separate ratings
    food_rating INT NOT NULL,
    delivery_rating INT NOT NULL,
    
    -- Reviews
    food_review TEXT,
    delivery_review TEXT,
    
    -- Photos
    photo_urls VARCHAR(500)[],
    
    created_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT chk_food_rating CHECK (food_rating >= 1 AND food_rating <= 5),
    CONSTRAINT chk_delivery_rating CHECK (delivery_rating >= 1 AND delivery_rating <= 5),
    CONSTRAINT unique_order_rating UNIQUE (order_id)
);

CREATE INDEX idx_ratings_restaurant ON ratings(order_id);
CREATE INDEX idx_ratings_user ON ratings(user_id);
```

### 9. Driver Locations (TimescaleDB)

```sql
-- TimescaleDB hypertable for time-series data
CREATE TABLE driver_locations (
    driver_id UUID NOT NULL,
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    heading INT,
    speed DECIMAL(5, 2),
    timestamp TIMESTAMPTZ NOT NULL,
    
    PRIMARY KEY (driver_id, timestamp)
);

-- Convert to hypertable
SELECT create_hypertable('driver_locations', 'timestamp');

-- Retention policy (7 days)
SELECT add_retention_policy('driver_locations', INTERVAL '7 days');

CREATE INDEX idx_driver_locations_driver ON driver_locations(driver_id, timestamp DESC);
```

### 10. Redis Data Structures

```redis
# Current driver locations (Redis Geo)
GEOADD drivers:locations {longitude} {latitude} {driver_id}

# Example:
GEOADD drivers:locations -122.4194 37.7749 driver-123

# Find nearby drivers (within 5km)
GEORADIUS drivers:locations -122.4194 37.7749 5 km WITHDIST

# Driver availability
SET driver:{driver_id}:available true EX 300

# Active orders cache
HSET order:{order_id} status PICKED_UP driver_id driver-123

# Restaurant cache
HSET restaurant:{restaurant_id} name "Pizza Palace" rating 4.5
```

### 11. Elasticsearch Index

```json
{
  "mappings": {
    "properties": {
      "restaurant_id": {"type": "keyword"},
      "name": {"type": "text", "analyzer": "standard"},
      "cuisine_types": {"type": "keyword"},
      "location": {"type": "geo_point"},
      "rating": {"type": "float"},
      "is_open": {"type": "boolean"},
      "delivery_time": {"type": "integer"},
      "menu_items": {
        "type": "nested",
        "properties": {
          "name": {"type": "text"},
          "description": {"type": "text"},
          "price": {"type": "float"}
        }
      }
    }
  }
}
```

---

### Why This Data Model Works:

**1. Normalized for Consistency**
✅ Restaurants, users, drivers in separate tables
✅ Foreign keys enforce referential integrity
✅ ACID transactions for orders

**2. Denormalized for Performance**
✅ Order items stored as JSONB (avoid joins)
✅ Delivery address copied to orders (immutable)
✅ Restaurant data cached in Redis

**3. Specialized Databases**
✅ TimescaleDB for location history (time-series)
✅ Redis Geo for real-time location queries
✅ Elasticsearch for full-text search

**4. Indexing Strategy**
✅ Geo-spatial indexes for location queries
✅ Composite indexes for common queries
✅ GIN indexes for array/JSONB columns


---

## PHASE 7: CORE ALGORITHM - DELIVERY ASSIGNMENT (6 minutes)

### What to Say:

"The delivery assignment algorithm is the heart of the system. Let me explain how we match orders with drivers."

### Problem Statement

When restaurant accepts order, we need to:
1. Find available drivers nearby
2. Score each driver based on multiple factors
3. Assign best driver
4. Handle rejection and reassignment

### Naive Approach: Nearest Driver (❌ SUBOPTIMAL)

```python
def assign_driver_naive(order):
    # Find nearest available driver
    drivers = find_nearby_drivers(
        lat=order.restaurant_lat,
        lng=order.restaurant_lng,
        radius=5000  # 5km
    )
    
    if not drivers:
        return None
    
    # Sort by distance
    drivers.sort(key=lambda d: d.distance)
    
    # Assign first driver
    return drivers[0]
```

**Why it fails:**
- Ignores driver rating
- Ignores driver's current location vs delivery location
- Ignores driver's acceptance rate
- No optimization for delivery time

---

### Smart Assignment Algorithm (✅ PRODUCTION-READY)

```python
class DeliveryAssignmentService:
    
    def assign_driver(self, order):
        """
        Multi-factor scoring algorithm
        """
        # Step 1: Find candidate drivers
        candidates = self.find_candidate_drivers(order)
        
        if not candidates:
            return self.handle_no_drivers(order)
        
        # Step 2: Score each driver
        scored_drivers = []
        for driver in candidates:
            score = self.calculate_driver_score(driver, order)
            scored_drivers.append((driver, score))
        
        # Step 3: Sort by score (highest first)
        scored_drivers.sort(key=lambda x: x[1], reverse=True)
        
        # Step 4: Try to assign (handle rejections)
        for driver, score in scored_drivers:
            if self.try_assign(driver, order):
                return driver
        
        # Step 5: No driver accepted
        return self.handle_no_acceptance(order)
    
    def find_candidate_drivers(self, order):
        """
        Find available drivers within radius
        """
        # Query Redis Geo
        drivers = redis.georadius(
            'drivers:locations',
            order.restaurant_lng,
            order.restaurant_lat,
            5000,  # 5km radius
            unit='m',
            withdist=True
        )
        
        # Filter available drivers
        available = []
        for driver_id, distance in drivers:
            driver = self.get_driver(driver_id)
            
            if (driver.is_available and 
                driver.is_active and 
                not driver.current_order_id):
                
                driver.distance_to_restaurant = distance
                available.append(driver)
        
        return available
    
    def calculate_driver_score(self, driver, order):
        """
        Multi-factor scoring:
        - Distance to restaurant (40%)
        - Driver rating (30%)
        - Acceptance rate (20%)
        - Total delivery route efficiency (10%)
        """
        # Factor 1: Distance to restaurant (closer is better)
        # Normalize: 0-5km → score 100-0
        distance_score = max(0, 100 - (driver.distance_to_restaurant / 50))
        
        # Factor 2: Driver rating (higher is better)
        # Normalize: 0-5 stars → score 0-100
        rating_score = (driver.rating / 5.0) * 100
        
        # Factor 3: Acceptance rate (higher is better)
        acceptance_rate = driver.accepted_orders / max(driver.offered_orders, 1)
        acceptance_score = acceptance_rate * 100
        
        # Factor 4: Route efficiency
        # Calculate total distance: driver → restaurant → customer
        total_distance = (
            driver.distance_to_restaurant +
            self.calculate_distance(
                order.restaurant_lat, order.restaurant_lng,
                order.delivery_lat, order.delivery_lng
            )
        )
        # Normalize: 0-10km → score 100-0
        route_score = max(0, 100 - (total_distance / 100))
        
        # Weighted sum
        final_score = (
            distance_score * 0.40 +
            rating_score * 0.30 +
            acceptance_score * 0.20 +
            route_score * 0.10
        )
        
        return final_score
    
    def try_assign(self, driver, order):
        """
        Attempt to assign driver (with distributed lock)
        """
        # Distributed lock to prevent double assignment
        lock_key = f'driver:{driver.id}:lock'
        lock = redis.set(lock_key, order.id, nx=True, ex=30)
        
        if not lock:
            return False  # Driver already assigned
        
        try:
            # Update database
            with transaction():
                # Update order
                order.driver_id = driver.id
                order.status = 'ASSIGNED'
                order.save()
                
                # Update driver
                driver.current_order_id = order.id
                driver.is_available = False
                driver.offered_orders += 1
                driver.save()
            
            # Send notification to driver
            self.notify_driver(driver, order)
            
            # Wait for acceptance (30 seconds timeout)
            accepted = self.wait_for_acceptance(driver, order, timeout=30)
            
            if accepted:
                driver.accepted_orders += 1
                driver.save()
                return True
            else:
                # Driver rejected or timeout
                self.rollback_assignment(driver, order)
                return False
                
        finally:
            redis.delete(lock_key)
    
    def handle_no_drivers(self, order):
        """
        No drivers available - expand search
        """
        # Expand radius to 10km
        candidates = self.find_candidate_drivers_expanded(order, radius=10000)
        
        if candidates:
            return self.assign_driver(order)
        
        # Still no drivers - notify restaurant
        self.notify_restaurant_no_drivers(order)
        
        # Add to retry queue
        self.add_to_retry_queue(order, delay=60)  # Retry in 1 minute
        
        return None
    
    def handle_no_acceptance(self, order):
        """
        All drivers rejected - retry with incentive
        """
        # Increase delivery fee
        order.delivery_fee += 2.00
        order.save()
        
        # Retry assignment
        self.add_to_retry_queue(order, delay=30)
        
        return None
```

---

### Assignment Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  DELIVERY ASSIGNMENT FLOW                                        │
└─────────────────────────────────────────────────────────────────┘

Restaurant accepts order
  ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Find Candidate Drivers                                   │
│                                                                   │
│ Redis GEORADIUS query:                                           │
│ - Center: Restaurant location                                    │
│ - Radius: 5km                                                    │
│ - Filter: is_available = true                                    │
│                                                                   │
│ Result: 15 drivers found                                         │
└─────────────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Score Each Driver                                        │
│                                                                   │
│ Driver A: distance=1.2km, rating=4.8, acceptance=95%            │
│   Score = 85*0.4 + 96*0.3 + 95*0.2 + 90*0.1 = 91.8             │
│                                                                   │
│ Driver B: distance=0.5km, rating=4.2, acceptance=80%            │
│   Score = 95*0.4 + 84*0.3 + 80*0.2 + 85*0.1 = 89.7             │
│                                                                   │
│ Driver C: distance=2.0km, rating=4.9, acceptance=98%            │
│   Score = 75*0.4 + 98*0.3 + 98*0.2 + 80*0.1 = 87.4             │
│                                                                   │
│ Sorted: A (91.8), B (89.7), C (87.4), ...                       │
└─────────────────────────────────────────────────────────────────┘
  ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Try Assign to Driver A                                   │
│                                                                   │
│ 1. Acquire distributed lock                                      │
│ 2. Update database (order.driver_id = A)                        │
│ 3. Send push notification to Driver A                           │
│ 4. Wait for acceptance (30 sec timeout)                         │
│                                                                   │
│ Outcome: Driver A accepts ✓                                      │
└─────────────────────────────────────────────────────────────────┘
  ↓
Assignment complete
  ↓
Notify customer: "Driver assigned"

┌─────────────────────────────────────────────────────────────────┐
│ REJECTION SCENARIO                                               │
└─────────────────────────────────────────────────────────────────┘

Driver A rejects (busy, too far, etc.)
  ↓
Rollback assignment:
  - order.driver_id = NULL
  - driver.current_order_id = NULL
  - driver.is_available = TRUE
  ↓
Try Driver B (next highest score)
  ↓
Driver B accepts ✓
  ↓
Assignment complete
```

---

### Optimization: Predictive Assignment

```python
class PredictiveAssignmentService:
    """
    Assign driver BEFORE restaurant accepts order
    """
    
    def predictive_assign(self, order):
        """
        When order is placed (PENDING status):
        1. Predict restaurant will accept (90% probability)
        2. Pre-assign driver (soft assignment)
        3. If restaurant accepts, confirm assignment
        4. If restaurant rejects, release driver
        """
        # Find best driver
        driver = self.find_best_driver(order)
        
        if driver:
            # Soft assignment (not committed)
            redis.setex(
                f'order:{order.id}:predicted_driver',
                300,  # 5 min TTL
                driver.id
            )
            
            # Notify driver (optional)
            self.notify_driver_potential_order(driver, order)
        
        return driver
    
    def confirm_assignment(self, order):
        """
        Restaurant accepted - confirm the prediction
        """
        predicted_driver_id = redis.get(f'order:{order.id}:predicted_driver')
        
        if predicted_driver_id:
            driver = self.get_driver(predicted_driver_id)
            
            # Check still available
            if driver.is_available:
                return self.try_assign(driver, order)
        
        # Fallback to normal assignment
        return self.assign_driver(order)
```

**Benefits:**
✅ Reduces assignment time by 30-60 seconds
✅ Driver already aware of potential order
✅ Better customer experience (faster delivery)

---

### Handling Edge Cases

**1. No Drivers Available**
```python
def handle_no_drivers(order):
    # Option 1: Expand search radius
    drivers = find_drivers(radius=10000)  # 10km
    
    # Option 2: Increase incentive
    order.delivery_fee += 2.00
    
    # Option 3: Notify customer of delay
    notify_customer("High demand, finding driver...")
    
    # Option 4: Add to retry queue
    retry_queue.add(order, delay=60)
```

**2. All Drivers Reject**
```python
def handle_all_rejections(order):
    # After 3 rounds of rejections:
    
    # Option 1: Cancel order with refund
    if order.rejection_count >= 3:
        cancel_order(order, reason="No drivers available")
        refund_customer(order)
    
    # Option 2: Increase delivery fee significantly
    order.delivery_fee += 5.00
    retry_assignment(order)
```

**3. Driver Accepts Multiple Orders (Race Condition)**
```python
def prevent_double_assignment():
    # Use distributed lock
    lock_key = f'driver:{driver_id}:lock'
    
    # Only one assignment can acquire lock
    if redis.set(lock_key, order_id, nx=True, ex=30):
        # Proceed with assignment
        assign_order(driver, order)
    else:
        # Driver already assigned, try next driver
        return False
```

---

### Performance Metrics

```
┌──────────────────────────┬─────────────┬─────────────┐
│ Metric                   │ Target      │ Actual      │
├──────────────────────────┼─────────────┼─────────────┤
│ Assignment time          │ < 10 sec    │ 5-8 sec     │
│ Driver acceptance rate   │ > 80%       │ 85%         │
│ First driver accepts     │ > 60%       │ 65%         │
│ No driver found rate     │ < 5%        │ 3%          │
│ Algorithm latency        │ < 500ms     │ 200ms       │
└──────────────────────────┴─────────────┴─────────────┘
```


---

## PHASE 8: DEEP DIVE - REAL-TIME TRACKING (4 minutes)

### What to Say:

"Real-time tracking is critical for user experience. Let me explain how we achieve < 5 second latency for location updates."

### Architecture for Real-Time Tracking

```
┌─────────────────────────────────────────────────────────────────┐
│  REAL-TIME TRACKING ARCHITECTURE                                 │
└─────────────────────────────────────────────────────────────────┘

┌──────────┐
│ Driver   │
│ App      │
└────┬─────┘
     │
     │ Every 5 seconds
     │ POST /driver/location
     │ { lat: 37.7749, lng: -122.4194 }
     ▼
┌─────────────────┐
│ Location Service│
│ (Go, high perf) │
└────┬────────────┘
     │
     ├──────────────┬──────────────┬──────────────┐
     ▼              ▼              ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Redis Geo│  │TimescaleDB│ │ WebSocket│  │  Kafka   │
│ (Current)│  │ (History) │  │ (Push)   │  │ (Events) │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
     │              │              │              │
     │              │              ▼              │
     │              │         ┌──────────┐       │
     │              │         │ Customer │       │
     │              │         │ App      │       │
     │              │         └──────────┘       │
     │              │                             │
     │              └─────────────────────────────┘
     │                    (Analytics)
     ▼
┌──────────┐
│ ETA      │
│ Service  │
└──────────┘
```

---

### WebSocket Implementation

```javascript
// Server-side (Node.js + Socket.io)
const io = require('socket.io')(server);

io.on('connection', (socket) => {
  socket.on('track_order', async (orderId) => {
    const order = await getOrder(orderId);
    if (order.customer_id !== socket.user_id) {
      return socket.emit('error', 'Unauthorized');
    }
    
    socket.join(`order:${orderId}`);
    
    if (order.driver_id) {
      const location = await redis.hgetall(`driver:${order.driver_id}:location`);
      socket.emit('location_update', location);
    }
  });
});

// Client-side
socket.emit('track_order', orderId);
socket.on('location_update', (location) => {
  updateDriverMarker(location.lat, location.lng);
});
```

---

### ETA Calculation

```python
class ETAService:
    async def calculate_eta(self, driver_id, order_id):
        order = await self.get_order(order_id)
        driver_location = await self.get_driver_location(driver_id)
        
        if order.status == 'ASSIGNED':
            destination = (order.restaurant_lat, order.restaurant_lng)
        else:
            destination = (order.delivery_lat, order.delivery_lng)
        
        route = await google_maps.directions(
            origin=(driver_location.lat, driver_location.lng),
            destination=destination
        )
        
        eta_seconds = route['duration'] * 1.2  # Add buffer
        
        if order.status == 'ASSIGNED':
            eta_seconds += order.estimated_prep_time_minutes * 60
        
        return time.time() + eta_seconds
```

