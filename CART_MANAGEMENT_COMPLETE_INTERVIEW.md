# Cart Management System (Uber-style) - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (3 min)
Phase 4: High-Level Architecture (8 min)
Phase 5: Data Models (6 min)
Phase 6: Component Design (Mobile/Backend) (8 min)
Phase 7: CRUD Operations & Sync Strategy (6 min)
Phase 8: Follow-up: Sub-users & Read-only Access (3 min)
Phase 9: Follow-up: 3rd Party Integrations (3 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"Thank you for the problem. I want to design a Cart Management system similar to Uber. Let me clarify the requirements and understand the scope."

### Context Setting (Interviewer):

"In Uber, the cart works like this:
- Users can add items from multiple merchants (Uber Eats + Uber Rides + Uber Grocery)
- Cart shows active orders, past orders, and scheduled orders
- Users can modify orders before they're confirmed
- Users can cancel orders with refund logic
- Cart syncs across devices (phone, web, tablet)

Our limitations:
- Multiple items from different merchants allowed
- Each merchant has different rules (some allow modifications, some don't)
- Real-time updates needed (order status changes)"

### Questions to Ask:

**Q1: Cart Scope**
- "Is this cart for active shopping (adding items), or for viewing placed orders, or both?"
- "Should I focus on the order management aspect (viewing/modifying placed orders)?"

**Expected Answer:** Focus on order management (viewing, modifying, canceling placed orders)

**Q2: Order Types**
- "You mentioned Delivery, Pickup, and Pickup with Ride. Are there other order types?"
- "Can a single cart have mixed order types?"

**Expected Answer:** Three types mentioned, yes mixed types allowed

**Q3: Multi-Merchant Support**
- "Can users have orders from multiple merchants simultaneously?"
- "Should we show them separately or consolidated?"

**Expected Answer:** Yes, show consolidated view with grouping by merchant

**Q4: Order Modification Rules**
- "What can users modify? Items, quantities, delivery address, time?"
- "Are there time limits for modifications (e.g., can't modify after restaurant accepts)?"

**Expected Answer:** Depends on order status, different rules per merchant

**Q5: Multi-Device Sync**
- "Should cart sync in real-time across devices?"
- "What happens if user modifies on phone while web is open?"

**Expected Answer:** Real-time sync required, last-write-wins with conflict resolution

**Q6: Offline Behavior**
- "Should cart work offline?"
- "What operations are allowed offline?"

**Expected Answer:** Read-only offline, modifications require connectivity

**Q7: Sub-users (Family Accounts)**
- "Can family members (teens) place orders?"
- "Should parents see all family orders or just their own?"

**Expected Answer:** Follow-up question, parents see all, teens see only theirs

**Q8: Scale**
- "How many users? How many orders per user?"
- "Expected concurrent users?"

**Expected Answer:** 50M users, average 10 active orders per user, 5M concurrent

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### What to Say:

"Based on our discussion, let me summarize the requirements."

### Functional Requirements (Priority Order):

```
1. View Orders (CORE)
   - Display all orders (active, past, scheduled)
   - Group by status (In Progress, Completed, Cancelled)
   - Group by merchant
   - Show order details (items, price, status, ETA)
   - Filter by date range, merchant, order type

2. Order Status Tracking (CORE)
   - Real-time status updates
   - Push notifications for status changes
   - Timeline view (placed → confirmed → preparing → delivered)

3. Modify Orders (CRITICAL)
   - Add/remove items (if allowed by merchant)
   - Change quantity
   - Update delivery address
   - Reschedule delivery time
   - Validation based on order status

4. Cancel Orders (CORE)
   - Cancel with refund calculation
   - Cancellation reasons
   - Different rules per merchant
   - Time-based restrictions

5. Multi-Device Sync (CORE)
   - Real-time sync across devices
   - Optimistic updates
   - Conflict resolution

6. Order Types Support
   - Delivery orders
   - Pickup orders
   - Pickup with Ride orders
   - Mixed cart support

7. Sub-user Support (Follow-up)
   - Family account management
   - Parent sees all orders
   - Teen sees only their orders
   - Permission-based access

8. 3rd Party Integration (Follow-up)
   - Partner merchant orders
   - Different data formats
   - Rate limiting
   - Fallback mechanisms

Out of Scope:
- Payment processing (separate system)
- Checkout flow (separate system)
- Product catalog (separate system)
```

### Non-Functional Requirements:

```
1. Performance
   - Cart load time: < 500ms
   - Order details: < 200ms
   - Sync latency: < 2 seconds
   - Support 5M concurrent users

2. Availability
   - 99.9% uptime
   - Graceful degradation (show cached data)
   - Offline read access

3. Consistency
   - Strong consistency for modifications
   - Eventual consistency for status updates
   - Conflict resolution for concurrent edits

4. Scalability
   - 50M users
   - 500M active orders
   - 10K orders/second (peak)

5. Reliability
   - No data loss
   - Idempotent operations
   - Retry mechanisms

6. Testability
   - Modular components
   - Clear abstractions
   - Dependency injection
   - Mock-friendly interfaces

7. Security
   - User can only see their orders
   - Sub-users have restricted access
   - Encrypted data in transit
```

### Why This Matters:
✓ Clarifies cart is for order management, not shopping
✓ Identifies key challenges (multi-device sync, order types)
✓ Sets expectations for testability and modularity

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (3 minutes)

### What to Say:

"Let me quickly estimate the scale to inform design decisions."

### Traffic Estimation:

```
Given:
- 50M total users
- 10M daily active users (DAU)
- Average 10 active orders per user
- Each user checks cart 5 times/day

Calculations:

1. Read Operations (View Cart)
   Daily reads: 10M users × 5 views = 50M reads/day
   Average QPS: 50M / 86,400 = ~580 reads/second
   Peak QPS: 580 × 3 = ~1,740 reads/second

2. Write Operations (Modify/Cancel)
   Assume 20% of views result in modification
   Daily writes: 50M × 0.2 = 10M writes/day
   Average QPS: 10M / 86,400 = ~116 writes/second
   Peak QPS: 116 × 3 = ~350 writes/second

3. Status Updates (Push from Backend)
   Active orders: 50M users × 10 orders = 500M orders
   Status changes: 5 per order (placed → delivered)
   Daily updates: 500M × 5 / 30 days = ~83M updates/day
   Average QPS: 83M / 86,400 = ~960 updates/second

4. Sync Operations (Multi-device)
   Users with multiple devices: 30% = 15M users
   Sync events: 15M × 5 views × 2 devices = 150M syncs/day
   Average QPS: 150M / 86,400 = ~1,740 syncs/second
```

### Storage Estimation:

```
1. Orders
   - 500M active orders
   - Each order: 5 KB (items, status, metadata)
   - Total: 500M × 5KB = 2.5 TB

2. Order History (2 years)
   - 10M orders/day × 365 × 2 = 7.3B orders
   - Each: 5 KB
   - Total: 7.3B × 5KB = 36.5 TB

3. User Cart State (Cache)
   - 50M users
   - Each: 50 KB (10 orders × 5KB)
   - Total: 50M × 50KB = 2.5 TB (in-memory cache)

Total Storage: ~40 TB
```

### Summary Table:

```
┌─────────────────────────┬──────────────────┐
│ Metric                  │ Value            │
├─────────────────────────┼──────────────────┤
│ Total users             │ 50M              │
│ DAU                     │ 10M              │
│ Active orders           │ 500M             │
│ Read QPS (peak)         │ 1,740            │
│ Write QPS (peak)        │ 350              │
│ Status update QPS       │ 960              │
│ Sync QPS                │ 1,740            │
│ Storage                 │ 40 TB            │
└─────────────────────────┴──────────────────┘

Key Insights:
1. Read-heavy workload (5:1 read:write ratio)
2. Real-time sync is significant load (1,740 QPS)
3. Need aggressive caching
4. Multi-device sync is complex
```

### Why This Matters:
✓ Validates need for caching layer
✓ Shows sync is a major concern
✓ Justifies architecture decisions

---

## PHASE 4: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"Let me present the high-level architecture. I'll show both mobile and backend components."

### Complete Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENT LAYER (Mobile/Web)                   │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  PRESENTATION LAYER                                    │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐│    │
│  │  │ CartView     │  │ OrderDetail  │  │ Modification ││    │
│  │  │ Controller   │  │ View         │  │ View         ││    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘│    │
│  └─────────┼──────────────────┼──────────────────┼────────┘    │
│            │                  │                  │              │
│  ┌─────────▼──────────────────▼──────────────────▼────────┐    │
│  │  BUSINESS LOGIC LAYER                                  │    │
│  │  ┌──────────────────────────────────────────────────┐ │    │
│  │  │  CartManager (Coordinator)                       │ │    │
│  │  │  - Orchestrates all cart operations              │ │    │
│  │  │  - Manages state                                 │ │    │
│  │  │  - Handles sync                                  │ │    │
│  │  └────────┬─────────────────────────────────────────┘ │    │
│  │           │                                            │    │
│  │  ┌────────▼────────┐  ┌──────────────┐  ┌──────────┐│    │
│  │  │ OrderRepository │  │ SyncManager  │  │ Validator││    │
│  │  └────────┬────────┘  └──────┬───────┘  └──────────┘│    │
│  └───────────┼────────────────────┼─────────────────────┘    │
│              │                    │                           │
│  ┌───────────▼────────────────────▼─────────────────────┐    │
│  │  DATA LAYER                                           │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐│    │
│  │  │ NetworkLayer │  │ CacheLayer   │  │ LocalDB      ││    │
│  │  │ (API Client) │  │ (In-Memory)  │  │ (SQLite)     ││    │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘│    │
│  └─────────┼──────────────────┼──────────────────┼────────┘    │
└────────────┼──────────────────┼──────────────────┼─────────────┘
             │                  │                  │
             │                  │                  │
             ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    NETWORK LAYER                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ REST APIs    │  │ WebSocket    │  │ GraphQL      │         │
│  │              │  │ (Real-time)  │  │ (Optional)   │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY / LOAD BALANCER                   │
│  - Authentication, Rate limiting, Routing                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┬──────────────┐
          ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  BACKEND SERVICES LAYER                          │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  1. CART SERVICE                                       │    │
│  │     - Get user cart                                    │    │
│  │     - Filter/sort orders                               │    │
│  │     - Aggregate from multiple sources                  │    │
│  │     Technology: Java/Spring Boot                       │    │
│  │     Instances: 20                                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  2. ORDER SERVICE                                      │    │
│  │     - Get order details                                │    │
│  │     - Order history                                    │    │
│  │     - Order status updates                             │    │
│  │     Technology: Java/Spring Boot                       │    │
│  │     Instances: 30                                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  3. MODIFICATION SERVICE                               │    │
│  │     - Validate modification rules                      │    │
│  │     - Apply modifications                              │    │
│  │     - Recalculate pricing                              │    │
│  │     Technology: Java/Spring Boot                       │    │
│  │     Instances: 15                                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  4. CANCELLATION SERVICE                               │    │
│  │     - Validate cancellation rules                      │    │
│  │     - Calculate refund                                 │    │
│  │     - Process cancellation                             │    │
│  │     Technology: Java/Spring Boot                       │    │
│  │     Instances: 10                                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  5. SYNC SERVICE                                       │    │
│  │     - Manage device sync                               │    │
│  │     - Conflict resolution                              │    │
│  │     - Push notifications                               │    │
│  │     Technology: Go (high performance)                  │    │
│  │     Instances: 25                                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  6. NOTIFICATION SERVICE                               │    │
│  │     - WebSocket connections                            │    │
│  │     - Push notifications (FCM/APNS)                    │    │
│  │     - Status update broadcasts                         │    │
│  │     Technology: Node.js                                │    │
│  │     Instances: 30 (stateful)                           │    │
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
│  │  - Orders table                                        │    │
│  │  - Order items                                         │    │
│  │  - Modification history                                │    │
│  │  Configuration: Multi-AZ, 3 read replicas             │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  CACHE (Redis Cluster)                                 │    │
│  │  - User cart cache (hot data)                          │    │
│  │  - Order details cache                                 │    │
│  │  - Session data                                        │    │
│  │  Configuration: 5 shards, 2 replicas each             │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  MESSAGE QUEUE (Kafka)                                 │    │
│  │  Topics:                                               │    │
│  │  - order.status_changed                                │    │
│  │  - order.modified                                      │    │
│  │  - order.cancelled                                     │    │
│  │  - sync.device_update                                  │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

### Data Flow: View Cart

```
1. User opens cart
   ↓
2. CartViewController calls CartManager.getCart()
   ↓
3. CartManager checks CacheLayer
   - If cached (< 5 min old): Return immediately
   - If not cached: Continue
   ↓
4. CartManager calls OrderRepository.fetchOrders()
   ↓
5. OrderRepository calls NetworkLayer.getOrders()
   ↓
6. API Gateway → Cart Service
   ↓
7. Cart Service:
   - Queries Redis cache first
   - If miss: Query PostgreSQL
   - Aggregate orders from multiple merchants
   - Apply filters/sorting
   ↓
8. Response flows back:
   Cart Service → API Gateway → NetworkLayer → OrderRepository
   ↓
9. OrderRepository updates CacheLayer
   ↓
10. CartManager updates UI
    ↓
11. CartViewController displays orders

Latency: < 500ms (cached), < 1s (uncached)
```

---

### Data Flow: Modify Order

```
1. User modifies order (add item)
   ↓
2. ModificationViewController calls CartManager.modifyOrder()
   ↓
3. CartManager calls Validator.validateModification()
   - Check order status (can modify?)
   - Check merchant rules
   - Check item availability
   ↓
4. If valid:
   a) Optimistic update (update UI immediately)
   b) CartManager calls OrderRepository.modifyOrder()
   ↓
5. OrderRepository calls NetworkLayer.modifyOrder()
   ↓
6. API Gateway → Modification Service
   ↓
7. Modification Service:
   BEGIN TRANSACTION
     - Validate again (server-side)
     - Update order in PostgreSQL
     - Recalculate pricing
     - Create modification history record
   COMMIT
   ↓
8. Publish to Kafka: order.modified
   ↓
9. Sync Service consumes event:
   - Identify user's other devices
   - Push update via WebSocket
   ↓
10. Other devices receive update:
    - Update local cache
    - Refresh UI
    ↓
11. Response flows back to original device:
    - Confirm optimistic update
    - Or rollback if failed

Latency: < 200ms (optimistic), < 1s (confirmed)
```

