# Facebook Live Comments System - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: Core Entities & Data Model (4 min)
Phase 5: API Design (3 min)
Phase 6: High-Level Architecture (8 min)
Phase 7: Data Flow - Comment Creation & Distribution (5 min)
Phase 8: Deep Dive - Real-time Comment Broadcasting (6 min)
Phase 9: Deep Dive - Horizontal Scaling & Coordination (5 min)
Phase 10: Deep Dive - Mega-Streams & CDN Strategy (5 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"I'll design a real-time commenting system for Facebook Live videos that enables millions of viewers to post and see comments with near-instant delivery. Let me clarify the scope and requirements."

### Questions to Ask:

**Q1: Comment Functionality**
- "What actions can viewers perform: post comments, reply to comments, react to comments?"
- "Should we support rich media in comments (emojis, images, GIFs)?"
- "Do we need comment moderation or spam filtering?"

**Expected Answer:** Post and view comments only, emojis supported, moderation out of scope

**Q2: Real-time Requirements**
- "What's the acceptable latency for comment delivery?"
- "Should comments appear instantly or is eventual consistency acceptable?"
- "Do we need to guarantee every viewer sees every comment?"

**Expected Answer:** <200ms latency, eventual consistency fine, best-effort delivery

**Q3: Historical Comments**
- "Should viewers see comments posted before they joined?"
- "How far back should comment history go?"
- "Do we need infinite scrolling for older comments?"

**Expected Answer:** Yes, show recent history with infinite scroll, keep all comments for video duration

**Q4: Scale & Concurrency**
- "What's the expected number of concurrent live videos?"
- "How many viewers per video (average and peak)?"
- "What's the expected comment rate per video?"

**Expected Answer:** Millions of concurrent videos, 100-10K viewers average, viral videos can hit millions, 10-1000 comments/second per video

**Q5: Video Lifecycle**
- "How long do live videos typically last?"
- "What happens to comments after the video ends?"
- "Should comments be available on the recorded video?"

**Expected Answer:** 30 min - 2 hours typical, comments persist for replay viewing

**Q6: User Experience**
- "Should users see their own comments immediately (optimistic updates)?"
- "How should we handle network disconnections?"
- "Do we need to show comment velocity indicators (e.g., 'comments flowing fast')?"

**Expected Answer:** Yes optimistic updates, graceful reconnection with catch-up, velocity indicators nice-to-have

**Q7: Geographic Distribution**
- "Is this a global system or region-specific?"
- "Do we need to handle different languages/character sets?"
- "Should we optimize for specific regions?"

**Expected Answer:** Global system, support Unicode, optimize for major regions

**Q8: Integration Points**
- "Do we integrate with existing Facebook infrastructure (user auth, video streaming)?"
- "Should we support notifications for comment replies?"
- "Do we need analytics on comment engagement?"

**Expected Answer:** Integrate with Facebook auth and video service, notifications out of scope, basic analytics needed

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### What to Say:

"Based on our discussion, let me organize the requirements to guide our design decisions."

### Functional Requirements (Priority Order):

```
CORE REQUIREMENTS (Above the line):

1. Comment Creation (CRITICAL)
   - Viewers can post text comments on live videos
   - Support emojis and Unicode characters
   - Comments associated with specific live video
   - User authentication required
   - Optimistic UI updates (show own comment immediately)

2. Real-time Comment Broadcasting (CRITICAL)
   - New comments appear to all viewers in near real-time (<200ms)
   - Comments stream continuously during live video
   - Handle high comment velocity (1000+ comments/second)
   - Graceful degradation under extreme load

3. Historical Comment Viewing (CRITICAL)
   - Show recent comments when joining live video
   - Infinite scroll to load older comments
   - Cursor-based pagination for efficiency
   - Comments ordered chronologically

4. Connection Management (IMPORTANT)
   - Handle network disconnections gracefully
   - Catch up on missed comments after reconnection
   - Maintain user's position in comment stream
   - Automatic reconnection with backoff

5. Comment Persistence (IMPORTANT)
   - Store all comments durably
   - Comments available after live video ends
   - Support replay viewing with comments
   - Retention for video lifetime

BELOW THE LINE (Out of scope):
- Comment replies and threading
- Comment reactions (likes, loves, etc.)
- Comment moderation and filtering
- Rich media attachments (images, videos)
- Comment editing or deletion
- User mentions and tagging
- Comment notifications
- Advanced analytics dashboard
```

### Non-Functional Requirements:

```
PERFORMANCE:
- Comment delivery latency: <200ms (P95) under normal conditions
- Comment creation latency: <100ms (P95)
- Historical comment load: <500ms for initial 50 comments
- Support 1000+ comments/second per popular video
- Handle 10K+ comments/second for mega-viral videos

SCALABILITY:
- Support 1M+ concurrent live videos
- Handle 100M+ concurrent viewers globally
- Scale to 10M+ viewers on single mega-viral video
- Support 1B+ comments per day across platform
- Horizontal scaling of all components

AVAILABILITY & RELIABILITY:
- System availability: 99.9% uptime
- Prioritize availability over consistency
- Eventual consistency acceptable (comments may arrive out of order)
- Graceful degradation under extreme load
- No comment loss (durable storage)

CONSISTENCY:
- Eventual consistency for comment delivery
- Strong consistency for comment creation (no duplicates)
- Best-effort ordering (timestamp-based)
- Idempotent operations

LATENCY TARGETS:
- Real-time delivery: <200ms (P95)
- Catch-up after disconnect: <2 seconds for 100 comments
- Historical load: <500ms for 50 comments
- Comment creation: <100ms (P95)

GEOGRAPHIC DISTRIBUTION:
- Multi-region deployment
- Edge locations for low latency
- CDN integration for static content
- Regional data compliance
```

### Why This Matters:
✓ Establishes <200ms as the "real-time" threshold
✓ Prioritizes availability over consistency (AP system)
✓ Acknowledges need for different strategies at different scales
✓ Sets clear expectations for graceful degradation

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me calculate the scale to understand infrastructure requirements and identify potential bottlenecks."

### Traffic Estimation:

```
Given:
- 1M concurrent live videos
- Average 1,000 viewers per video
- Peak videos: 10K viewers
- Mega-viral videos: 10M viewers
- Average comment rate: 10 comments/second per video
- Peak comment rate: 1,000 comments/second per video
- Mega-viral rate: 10,000 comments/second per video

Calculations:

1. Concurrent Viewers
   Total viewers: 1M videos × 1,000 viewers = 1B concurrent viewers
   
   Distribution:
   - 999,000 videos × 1,000 viewers = 999M viewers (normal)
   - 1,000 videos × 10,000 viewers = 10M viewers (popular)
   - 1 video × 10M viewers = 10M viewers (mega-viral)
   
   Total: ~1.02B concurrent viewers

2. Comment Creation Rate
   Normal videos: 999,000 × 10 comments/sec = 10M comments/sec
   Popular videos: 1,000 × 1,000 comments/sec = 1M comments/sec
   Mega-viral: 1 × 10,000 comments/sec = 10K comments/sec
   
   Total: ~11M comments/second (peak)
   Average: ~5M comments/second

3. Comment Broadcast Load (Critical Metric)
   Each comment must be sent to all viewers of that video
   
   Normal video: 10 comments/sec × 1,000 viewers = 10K messages/sec per video
   Popular video: 1,000 comments/sec × 10K viewers = 10M messages/sec per video
   Mega-viral: 10K comments/sec × 10M viewers = 100B messages/sec per video!
   
   Total broadcast load:
   - Normal: 999,000 videos × 10K = 10B messages/sec
   - Popular: 1,000 videos × 10M = 10B messages/sec
   - Mega-viral: 1 video × 100B = 100B messages/sec
   
   Total: ~120B messages/second (this is the real challenge!)

4. Read:Write Ratio
   Writes (comment creation): 11M/second
   Reads (comment delivery): 120B/second
   Ratio: 1:10,909 (extremely read-heavy!)
   
   This massive read amplification is the core challenge

5. Connection Management
   SSE connections needed: 1B concurrent viewers
   Connections per server: 100K (realistic limit)
   Servers needed: 1B / 100K = 10,000 servers minimum
   
   With redundancy (2x): 20,000 servers
```

### Storage Estimation:

```
1. Comment Data
   Daily comments: 11M comments/sec × 86,400 = 950B comments/day
   Average comment size: 100 bytes (text + metadata)
   Daily storage: 950B × 100 bytes = 95TB/day
   
   With 30-day retention: 95TB × 30 = 2.85PB
   With compression (3x): ~950TB

2. Comment Metadata
   Per comment: 200 bytes (user_id, video_id, timestamp, etc.)
   Daily metadata: 950B × 200 bytes = 190TB/day
   30-day retention: 5.7PB (compressed: ~1.9PB)

3. Active Comment Cache (Hot Data)
   Recent comments per video: 1,000 comments
   Cache per video: 1,000 × 300 bytes = 300KB
   Total cache: 1M videos × 300KB = 300GB
   
   This is the "hot" data for real-time delivery

4. Historical Comment Index
   For pagination and catch-up
   Index size: ~50% of comment data = 475TB

Total Storage: ~3PB (with compression and 30-day retention)
```

### Bandwidth Estimation:

```
1. Comment Creation (Ingress)
   11M comments/sec × 500 bytes (HTTP overhead) = 5.5GB/sec
   Daily: 5.5GB × 86,400 = 475TB/day

2. Comment Broadcasting (Egress) - THE BOTTLENECK
   120B messages/sec × 300 bytes = 36TB/sec
   Daily: 36TB × 86,400 = 3.1EB/day (exabytes!)
   
   This is why we need CDN and smart distribution strategies

3. Historical Comment Loading
   Assume 10% of viewers load history: 100M requests/sec
   100M × 15KB (50 comments) = 1.5TB/sec
   Daily: 1.5TB × 86,400 = 130PB/day

4. Reconnection Catch-up
   Assume 1% disconnect rate: 10M reconnections/sec
   10M × 5KB (catch-up data) = 50GB/sec
   Daily: 50GB × 86,400 = 4.3PB/day

Total Bandwidth: ~3.2EB/day (dominated by real-time broadcasting)
```

### Infrastructure Estimation:

```
1. Real-time Messaging Servers
   Concurrent connections: 1B viewers
   Connections per server: 100K
   Servers needed: 10,000 servers
   
   Instance type: c5.4xlarge (16 vCPU, 32GB RAM)
   Cost: 10,000 × $0.68/hour = $6,800/hour = $5M/month

2. Comment Management Service
   Write QPS: 11M comments/sec
   Requests per instance: 10K/sec
   Instances needed: 1,100 instances
   
   Instance type: c5.2xlarge (8 vCPU, 16GB RAM)
   Cost: 1,100 × $0.34/hour = $374/hour = $275K/month

3. Database (DynamoDB)
   Write capacity: 11M WCU
   Read capacity: 100M RCU (historical queries)
   Storage: 3PB
   
   Write cost: 11M × $0.00065/hour = $7,150/hour = $5.25M/month
   Read cost: 100M × $0.00013/hour = $13,000/hour = $9.5M/month
   Storage cost: 3PB × $0.25/GB = $750K/month
   Total: ~$15.5M/month

4. Cache (Redis/ElastiCache)
   Hot data: 300GB
   Throughput: 120B reads/sec (distributed)
   Cluster size: 100 nodes (r5.4xlarge)
   Cost: 100 × $1.36/hour = $136/hour = $100K/month

5. CDN (CloudFront)
   Data transfer: 3.2EB/month
   Cost: 3.2EB × $0.085/GB = $272M/month
   
   With optimizations (sampling, compression): ~$50M/month

6. Message Queue (Kafka/Redis Pub/Sub)
   Throughput: 11M messages/sec
   Cluster size: 500 nodes
   Cost: 500 × $0.34/hour = $170/hour = $125K/month

Total Monthly Cost: ~$75M/month (dominated by CDN and database)

Note: These are rough estimates. Real costs would be optimized through:
- Regional distribution
- Intelligent caching
- Sampling for mega-streams
- Compression
- Reserved instances
```

### Key Insights from Estimation:

```
1. Read Amplification is Extreme
   - 1:10,909 read:write ratio
   - Each comment generates 1,000-10M deliveries
   - This is why naive approaches fail

2. Mega-Streams are Different
   - Single video can generate 100B messages/sec
   - Requires completely different architecture
   - CDN-based approach becomes necessary

3. Connection Management is Critical
   - 1B concurrent SSE connections
   - 10K+ servers just for connection handling
   - Server coordination is complex

4. Cost is Dominated by Distribution
   - CDN costs dwarf everything else
   - Database writes are expensive at this scale
   - Optimization is essential for profitability
```

---

## PHASE 4: CORE ENTITIES & DATA MODEL (4 minutes)

### What to Say:

"Let me define the core entities and their relationships. I'll start with a high-level overview, then detail the schema for our real-time system."

### Core Entities:

```
1. User
   - Represents a viewer or broadcaster
   - Managed by Facebook's user service (external)
   - We only store user_id references

2. LiveVideo
   - Represents an active live stream
   - Managed by Facebook's video service (external)
   - We track video_id and metadata

3. Comment
   - The core entity we manage
   - Contains message text and metadata
   - Associated with user and video

4. CommentStream
   - Logical grouping of comments for a video
   - Used for real-time distribution
   - Ephemeral (exists only during live stream)
```

### Data Model Design:

```sql
-- Comments Table (DynamoDB)
-- Primary storage for all comments
{
  "comment_id": "uuid",                    // Partition Key
  "video_id": "string",                    // GSI Partition Key
  "user_id": "string",
  "message": "string",                     // Comment text (max 500 chars)
  "created_at": 1715548800,               // Unix timestamp (milliseconds)
  "created_at_video_id": "1715548800#video_123", // GSI Sort Key
  "metadata": {
    "user_name": "John Doe",
    "user_avatar": "https://...",
    "device_type": "mobile",
    "client_version": "1.2.3"
  },
  "status": "ACTIVE | DELETED",
  "moderation_flags": []                   // For future moderation
}

-- Primary Access Pattern: Get comment by ID
-- Key: comment_id

-- GSI: video_id-created_at-index
-- Access Pattern: Get comments for video, ordered by time
-- Partition Key: video_id
-- Sort Key: created_at_video_id
-- Enables efficient pagination and time-range queries

-- Example Query:
{
  "TableName": "comments",
  "IndexName": "video_id-created_at-index",
  "KeyConditionExpression": "video_id = :vid AND created_at_video_id < :cursor",
  "ExpressionAttributeValues": {
    ":vid": "video_123",
    ":cursor": "1715548800#video_123"
  },
  "ScanIndexForward": false,              // Descending order (newest first)
  "Limit": 50
}
```

```sql
-- Active Videos Table (Redis)
-- Tracks currently live videos and their metadata
{
  "video_id": "video_123",
  "broadcaster_id": "user_456",
  "started_at": 1715548800,
  "viewer_count": 15234,
  "comment_count": 8921,
  "comment_rate": 45.2,                   // Comments per second
  "status": "LIVE | ENDED",
  "ttl": 7200                             // 2 hours
}

-- Stored in Redis with TTL
-- Key: video:{video_id}
-- TTL: Video duration + 1 hour
```

```sql
-- Comment Stream Cache (Redis)
-- Hot cache of recent comments for fast delivery
{
  "video_id": "video_123",
  "comments": [
    {
      "comment_id": "comment_789",
      "user_id": "user_456",
      "user_name": "John Doe",
      "message": "Great stream!",
      "created_at": 1715548800
    },
    // ... last 1000 comments
  ]
}

-- Stored as Redis List (LPUSH for new, LTRIM to limit size)
-- Key: stream:{video_id}:comments
-- Max size: 1000 comments
-- TTL: Video duration + 1 hour

-- Operations:
LPUSH stream:video_123:comments "{comment_json}"
LTRIM stream:video_123:comments 0 999
LRANGE stream:video_123:comments 0 49  // Get 50 most recent
```

```sql
-- Server Registry (Redis)
-- Tracks which servers are handling which videos
{
  "server_id": "server_001",
  "region": "us-east-1",
  "connections": 85234,
  "capacity": 100000,
  "videos": ["video_123", "video_456", ...],
  "last_heartbeat": 1715548800
}

-- Stored in Redis with heartbeat updates
-- Key: server:{server_id}
-- TTL: 30 seconds (refreshed by heartbeat)

-- Video to Server Mapping
-- Key: video:{video_id}:servers
-- Value: Set of server_ids
SADD video:video_123:servers server_001 server_002
```

### Key Design Decisions:

```
1. DynamoDB for Comment Storage
   - Handles massive write throughput (11M/sec)
   - Automatic scaling and partitioning
   - GSI for efficient video-based queries
   - Pay-per-use pricing model

2. Redis for Hot Data
   - Sub-millisecond latency for recent comments
   - List data structure perfect for comment streams
   - Pub/Sub for real-time distribution
   - TTL for automatic cleanup

3. Composite Sort Key (created_at#video_id)
   - Enables efficient time-based pagination
   - Prevents collisions (multiple comments same millisecond)
   - Supports cursor-based pagination

4. Denormalized User Data
   - Store user_name and avatar in comment
   - Avoids joins during comment delivery
   - Acceptable staleness (user updates rare)
   - Reduces latency significantly

5. Separate Hot and Cold Storage
   - Redis: Last 1000 comments (hot)
   - DynamoDB: All comments (cold)
   - Optimizes for read patterns
   - Reduces database load
```

---

## PHASE 5: API DESIGN (3 minutes)

### What to Say:

"Let me design the APIs that satisfy our functional requirements, focusing on simplicity and performance."

### API Endpoints:

```
1. CREATE COMMENT
POST /v1/videos/{video_id}/comments
Headers:
  Authorization: Bearer <jwt_token>
  Content-Type: application/json
  X-Request-ID: <uuid>                    // For idempotency

Request:
{
  "message": "Great stream! 🔥",
  "client_timestamp": 1715548800123       // For latency tracking
}

Response: 201 Created
{
  "comment_id": "comment_789",
  "video_id": "video_123",
  "user_id": "user_456",
  "user_name": "John Doe",
  "user_avatar": "https://cdn.facebook.com/avatars/user_456.jpg",
  "message": "Great stream! 🔥",
  "created_at": 1715548800125,
  "server_timestamp": 1715548800125
}

Error Responses:
400 Bad Request - Invalid message (empty, too long, invalid chars)
401 Unauthorized - Invalid or missing token
404 Not Found - Video doesn't exist or is not live
429 Too Many Requests - Rate limit exceeded
500 Internal Server Error - Server error

Rate Limiting:
- 10 comments per minute per user per video
- 100 comments per hour per user globally
- Enforced via Redis token bucket

---

2. GET RECENT COMMENTS (Historical Load)
GET /v1/videos/{video_id}/comments?cursor={cursor}&limit={limit}
Headers:
  Authorization: Bearer <jwt_token>

Query Parameters:
  - cursor: Pagination cursor (comment_id#timestamp) [optional]
  - limit: Number of comments to return (1-100, default: 50)
  - order: asc | desc (default: desc - newest first)

Response: 200 OK
{
  "comments": [
    {
      "comment_id": "comment_789",
      "user_id": "user_456",
      "user_name": "John Doe",
      "user_avatar": "https://...",
      "message": "Great stream! 🔥",
      "created_at": 1715548800125
    },
    // ... more comments
  ],
  "pagination": {
    "next_cursor": "comment_788#1715548799000",
    "has_more": true,
    "total_count": 8921                   // Approximate
  },
  "metadata": {
    "video_status": "LIVE",
    "comment_rate": 45.2                  // Comments per second
  }
}

---

3. ESTABLISH SSE CONNECTION (Real-time Stream)
GET /v1/videos/{video_id}/comments/stream
Headers:
  Authorization: Bearer <jwt_token>
  Last-Event-ID: <comment_id>            // For reconnection

Response: 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

Stream Format:
id: comment_789
event: comment
data: {"comment_id":"comment_789","user_id":"user_456","user_name":"John Doe","message":"Great!","created_at":1715548800125}

id: comment_790
event: comment
data: {"comment_id":"comment_790","user_id":"user_457","user_name":"Jane","message":"Amazing!","created_at":1715548800250}

// Heartbeat every 30 seconds
event: heartbeat
data: {"timestamp":1715548830000}

// Video ended event
event: video_ended
data: {"video_id":"video_123","ended_at":1715550000000}

---

4. CATCH-UP AFTER DISCONNECT
GET /v1/videos/{video_id}/comments/catchup?since={comment_id}&limit={limit}
Headers:
  Authorization: Bearer <jwt_token>

Query Parameters:
  - since: Last comment_id received (required)
  - limit: Max comments to return (1-100, default: 100)

Response: 200 OK
{
  "comments": [
    // Comments posted after 'since' comment
  ],
  "missed_count": 47,                     // Total comments missed
  "returned_count": 47,                   // Comments in this response
  "truncated": false,                     // True if missed > limit
  "latest_comment_id": "comment_836"
}

If truncated=true, client should:
1. Show "You missed X comments" message
2. Offer "Jump to live" button
3. Resume SSE stream from latest_comment_id

---

5. GET VIDEO METADATA
GET /v1/videos/{video_id}/metadata
Headers:
  Authorization: Bearer <jwt_token>

Response: 200 OK
{
  "video_id": "video_123",
  "status": "LIVE | ENDED",
  "started_at": 1715548800000,
  "ended_at": null,
  "viewer_count": 15234,
  "comment_count": 8921,
  "comment_rate": 45.2,                   // Comments per second
  "broadcaster": {
    "user_id": "user_456",
    "user_name": "John Doe",
    "user_avatar": "https://..."
  }
}
```

### API Design Principles:

```
1. RESTful Design
   - Resource-based URLs (/videos/{id}/comments)
   - Standard HTTP methods (GET, POST)
   - Proper status codes (201, 200, 404, 429)

2. Cursor-Based Pagination
   - Stable pagination (no offset issues)
   - Efficient database queries
   - Works with DynamoDB sort keys

3. SSE for Real-time
   - One-way communication (server → client)
   - Built-in reconnection with Last-Event-ID
   - Standard HTTP (no special protocols)
   - Efficient for read-heavy workload

4. Idempotency
   - X-Request-ID header for duplicate detection
   - Prevents duplicate comments on retry
   - 5-minute deduplication window

5. Rate Limiting
   - Per-user, per-video limits
   - Global user limits
   - 429 status with Retry-After header

6. Optimistic Updates
   - Client shows comment immediately
   - Server confirms via SSE
   - Rollback if server rejects
```



---

## PHASE 6: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"I'll design a distributed architecture that separates comment creation from real-time distribution. The key insight is using a pub/sub model with intelligent routing to handle the massive read amplification."

### Complete System Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│   Mobile App   │   Web Browser   │   Smart TV   │   Desktop    │
│  (iOS/Android) │   (React SPA)   │    App       │     App      │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EDGE & CDN LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│  CloudFront CDN  │  Edge Locations  │  DDoS Protection         │
│  (Static Assets, │  (Global PoPs)   │  (WAF, Shield)           │
│   Comment Snapshots) │              │                          │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY LAYER                            │
├─────────────────────────────────────────────────────────────────┤
│  Layer 7 Load Balancer (NGINX/Envoy)                           │
│  - JWT Validation                                               │
│  - Rate Limiting (per user, per video)                         │
│  - Request Routing (write vs read)                             │
│  - SSL Termination                                              │
│  - Consistent Hashing for SSE connections                       │
└─────────────────────────────────────────────────────────────────┘
                    │                               │
                    │ POST /comments                │ GET /stream
                    ▼                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     SERVICE LAYER                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Comment Management Service (Write Path)                    ││
│  │                                                              ││
│  │  Responsibilities:                                           ││
│  │  - Validate comment content                                 ││
│  │  - Check rate limits (Redis)                                ││
│  │  - Persist to DynamoDB                                      ││
│  │  - Update Redis cache                                       ││
│  │  - Publish to Redis Pub/Sub                                 ││
│  │  - Return response to client                                ││
│  │                                                              ││
│  │  Instances: 1,100 (auto-scaled)                             ││
│  │  Instance Type: c5.2xlarge                                  ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Realtime Messaging Service (Read Path)                     ││
│  │                                                              ││
│  │  Responsibilities:                                           ││
│  │  - Maintain SSE connections to viewers                      ││
│  │  - Subscribe to Redis Pub/Sub channels                      ││
│  │  - Broadcast comments to connected clients                  ││
│  │  - Handle reconnections and catch-up                        ││
│  │  - Send heartbeats                                          ││
│  │                                                              ││
│  │  Connection Management:                                      ││
│  │  - Track video_id → [connections] mapping                   ││
│  │  - Register with Dispatcher on startup                      ││
│  │  - Send heartbeats every 10 seconds                         ││
│  │  - Graceful shutdown (drain connections)                    ││
│  │                                                              ││
│  │  Instances: 10,000 (auto-scaled)                            ││
│  │  Instance Type: c5.4xlarge                                  ││
│  │  Connections per instance: 100K                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  Query Service (Historical Comments)                        ││
│  │                                                              ││
│  │  Responsibilities:                                           ││
│  │  - Serve historical comment queries                         ││
│  │  - Implement cursor-based pagination                        ││
│  │  - Cache frequently accessed pages                          ││
│  │  - Handle catch-up requests                                 ││
│  │                                                              ││
│  │  Instances: 500 (auto-scaled)                               ││
│  │  Instance Type: r5.xlarge                                   ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                    │                               │
                    ▼                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     DATA LAYER                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              DynamoDB (Primary Storage)                     ││
│  │                                                              ││
│  │  Comments Table:                                             ││
│  │  - Partition Key: comment_id                                ││
│  │  - GSI: video_id + created_at                               ││
│  │  - Capacity: 11M WCU, 100M RCU                              ││
│  │  - Storage: 3PB (compressed)                                ││
│  │  - Multi-region replication                                 ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Redis Cluster (Hot Data & Pub/Sub)             ││
│  │                                                              ││
│  │  Use Cases:                                                  ││
│  │  1. Comment Stream Cache                                    ││
│  │     - Key: stream:{video_id}:comments                       ││
│  │     - Type: List (last 1000 comments)                       ││
│  │     - TTL: Video duration + 1 hour                          ││
│  │                                                              ││
│  │  2. Pub/Sub Channels                                        ││
│  │     - Channel: comments:video:{hash(video_id) % 1000}       ││
│  │     - 1000 channels for distribution                        ││
│  │     - Fire-and-forget delivery                              ││
│  │                                                              ││
│  │  3. Rate Limiting                                           ││
│  │     - Key: ratelimit:{user_id}:{video_id}                   ││
│  │     - Type: String (counter)                                ││
│  │     - TTL: 60 seconds                                       ││
│  │                                                              ││
│  │  4. Server Registry                                         ││
│  │     - Key: server:{server_id}                               ││
│  │     - Key: video:{video_id}:servers (Set)                   ││
│  │     - TTL: 30 seconds (heartbeat)                           ││
│  │                                                              ││
│  │  Cluster: 100 nodes (r5.4xlarge)                            ││
│  │  Sharding: Hash-based across nodes                          ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                  COORDINATION LAYER                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │         Dispatcher Service (Optional - Advanced)            ││
│  │                                                              ││
│  │  Responsibilities:                                           ││
│  │  - Maintain video_id → server_ids mapping                   ││
│  │  - Route comments to appropriate servers                    ││
│  │  - Handle server failures and rebalancing                   ││
│  │  - Coordinate with Realtime Messaging Servers               ││
│  │                                                              ││
│  │  Data Store: Zookeeper or etcd                              ││
│  │  Instances: 10 (for redundancy)                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                  MONITORING & OBSERVABILITY                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Prometheus   │  │ Grafana      │  │ ELK Stack    │          │
│  │              │  │              │  │              │          │
│  │ - Metrics    │  │ - Dashboards │  │ - Logs       │          │
│  │ - Alerting   │  │ - Viz        │  │ - Search     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  Key Metrics:                                                    │
│  - Comment creation rate (per video, global)                    │
│  - Comment delivery latency (P50, P95, P99)                     │
│  - SSE connection count (per server, global)                    │
│  - Redis pub/sub throughput                                     │
│  - DynamoDB throttling events                                   │
│  - Server CPU/Memory utilization                                │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities:

```
1. API Gateway Layer
   - SSL termination and security
   - JWT validation and user authentication
   - Rate limiting (per-user, per-video)
   - Consistent hashing for SSE routing
   - Request routing (write vs read paths)

2. Comment Management Service (Write Path)
   - Validate comment content and length
   - Check rate limits via Redis
   - Generate comment_id (UUID)
   - Persist to DynamoDB
   - Update Redis cache (LPUSH to stream)
   - Publish to Redis Pub/Sub channel
   - Return response to client (<100ms)

3. Realtime Messaging Service (Read Path)
   - Accept SSE connections from viewers
   - Maintain video_id → connections mapping
   - Subscribe to relevant Redis Pub/Sub channels
   - Broadcast comments to connected viewers
   - Handle reconnections with catch-up
   - Send heartbeats every 30 seconds
   - Graceful shutdown and connection draining

4. Query Service
   - Serve historical comment queries
   - Implement cursor-based pagination
   - Cache frequently accessed pages
   - Handle catch-up after disconnect
   - Query DynamoDB with GSI

5. Redis Cluster
   - Comment stream cache (hot data)
   - Pub/Sub for real-time distribution
   - Rate limiting counters
   - Server registry and coordination
   - Sub-millisecond latency

6. DynamoDB
   - Durable storage for all comments
   - Auto-scaling for write throughput
   - GSI for video-based queries
   - Multi-region replication
```

### Data Flow Summary:

```
Comment Creation Flow:
Client → API Gateway → Comment Management Service
                    → DynamoDB (persist)
                    → Redis Cache (update stream)
                    → Redis Pub/Sub (publish)

Real-time Distribution Flow:
Redis Pub/Sub → Realtime Messaging Servers → SSE → Viewers

Historical Load Flow:
Client → API Gateway → Query Service → Redis Cache (check)
                                    → DynamoDB (if cache miss)
                                    → Client

Reconnection Flow:
Client → API Gateway → Query Service → Redis Cache (recent comments)
                                    → Resume SSE stream
```

---

## PHASE 7: DATA FLOW - COMMENT CREATION & DISTRIBUTION (5 minutes)

### What to Say:

"Let me walk through the complete data flow from comment creation to delivery, highlighting the critical paths and optimizations."

### Flow 1: Comment Creation (Write Path)

```
┌──────────────────────────────────────────────────────────────────┐
│  STEP 1: CLIENT POSTS COMMENT                                    │
└──────────────────────────────────────────────────────────────────┘

User types: "Great stream! 🔥"
Client sends:
  POST /v1/videos/video_123/comments
  Authorization: Bearer eyJhbGc...
  X-Request-ID: req_abc123
  {
    "message": "Great stream! 🔥",
    "client_timestamp": 1715548800000
  }

Optimistic Update:
  - Client immediately shows comment in UI
  - Marked as "pending" (gray checkmark)
  - Will be confirmed when received via SSE

┌──────────────────────────────────────────────────────────────────┐
│  STEP 2: API GATEWAY PROCESSING                                  │
└──────────────────────────────────────────────────────────────────┘

API Gateway:
  1. Validate JWT token
     - Extract user_id: user_456
     - Verify signature and expiration
     
  2. Check rate limit (Redis)
     key = "ratelimit:user_456:video_123"
     count = INCR key
     if count == 1: EXPIRE key 60
     if count > 10: return 429 Too Many Requests
     
  3. Route to Comment Management Service
     - Round-robin load balancing
     - Add X-User-ID header

Latency: ~20ms

┌──────────────────────────────────────────────────────────────────┐
│  STEP 3: COMMENT MANAGEMENT SERVICE                              │
└──────────────────────────────────────────────────────────────────┘

Comment Management Service:
  1. Validate comment
     - Check message length (max 500 chars)
     - Validate UTF-8 encoding
     - Check for empty/whitespace-only
     
  2. Generate comment_id
     comment_id = UUID.v4()  // "comment_789"
     
  3. Enrich with user data (from cache or user service)
     user_name = "John Doe"
     user_avatar = "https://cdn.facebook.com/avatars/user_456.jpg"
     
  4. Create comment object
     comment = {
       comment_id: "comment_789",
       video_id: "video_123",
       user_id: "user_456",
       user_name: "John Doe",
       user_avatar: "https://...",
       message: "Great stream! 🔥",
       created_at: 1715548800125,
       metadata: {...}
     }

Latency: ~10ms

┌──────────────────────────────────────────────────────────────────┐
│  STEP 4: PERSIST TO DYNAMODB                                     │
└──────────────────────────────────────────────────────────────────┘

DynamoDB Write:
  PutItem(
    TableName: "comments",
    Item: {
      comment_id: "comment_789",
      video_id: "video_123",
      created_at_video_id: "1715548800125#video_123",
      ...comment
    },
    ConditionExpression: "attribute_not_exists(comment_id)"
  )

Latency: ~15ms (single-digit millisecond for DynamoDB)

┌──────────────────────────────────────────────────────────────────┐
│  STEP 5: UPDATE REDIS CACHE                                      │
└──────────────────────────────────────────────────────────────────┘

Redis Operations (pipelined):
  1. Add to comment stream
     LPUSH stream:video_123:comments "{comment_json}"
     LTRIM stream:video_123:comments 0 999  // Keep last 1000
     
  2. Update video metadata
     HINCRBY video:video_123 comment_count 1
     HSET video:video_123 last_comment_at 1715548800125
     
  3. Update comment rate (sliding window)
     ZADD video:video_123:comment_times 1715548800125 comment_789
     ZREMRANGEBYSCORE video:video_123:comment_times 0 (now - 60000)
     ZCARD video:video_123:comment_times  // Comments in last minute

Latency: ~5ms (pipelined)

┌──────────────────────────────────────────────────────────────────┐
│  STEP 6: PUBLISH TO REDIS PUB/SUB                                │
└──────────────────────────────────────────────────────────────────┘

Determine channel:
  channel_id = hash(video_123) % 1000  // e.g., 456
  channel = "comments:video:456"

Publish:
  PUBLISH comments:video:456 "{comment_json}"

Fire-and-forget: No acknowledgment needed
Latency: ~2ms

┌──────────────────────────────────────────────────────────────────┐
│  STEP 7: RETURN RESPONSE TO CLIENT                               │
└──────────────────────────────────────────────────────────────────┘

Response:
  201 Created
  {
    comment_id: "comment_789",
    video_id: "video_123",
    user_id: "user_456",
    user_name: "John Doe",
    message: "Great stream! 🔥",
    created_at: 1715548800125,
    server_timestamp: 1715548800125
  }

Client:
  - Updates optimistic comment to "confirmed" (blue checkmark)
  - Calculates latency: server_timestamp - client_timestamp = 125ms

Total Latency: ~70ms (well within 100ms target)
```

### Flow 2: Real-time Comment Distribution (Read Path)

```
┌──────────────────────────────────────────────────────────────────┐
│  STEP 1: VIEWER ESTABLISHES SSE CONNECTION                       │
└──────────────────────────────────────────────────────────────────┘

Viewer joins video:
  GET /v1/videos/video_123/comments/stream
  Authorization: Bearer eyJhbGc...

API Gateway:
  1. Validate JWT
  2. Use consistent hashing to route to Realtime Messaging Server
     server_id = consistent_hash(video_123) % num_servers
     Route to: server_042
     
  3. Establish SSE connection

Realtime Messaging Server (server_042):
  1. Accept SSE connection
     connection_id = UUID.v4()
     
  2. Add to local mapping
     connections[video_123].add(connection_id)
     
  3. Subscribe to Redis Pub/Sub channel (if not already)
     channel_id = hash(video_123) % 1000  // 456
     if not subscribed_to(channel_456):
       SUBSCRIBE comments:video:456
       
  4. Register with Dispatcher (optional)
     SADD video:video_123:servers server_042
     EXPIRE video:video_123:servers 30
     
  5. Send initial heartbeat
     event: heartbeat
     data: {"timestamp": 1715548800000}

Connection established!

┌──────────────────────────────────────────────────────────────────┐
│  STEP 2: COMMENT PUBLISHED TO REDIS PUB/SUB                      │
└──────────────────────────────────────────────────────────────────┘

(From previous flow - Step 6)
Redis Pub/Sub:
  PUBLISH comments:video:456 "{comment_json}"

All servers subscribed to channel 456 receive the message

┌──────────────────────────────────────────────────────────────────┐
│  STEP 3: REALTIME MESSAGING SERVER RECEIVES MESSAGE              │
└──────────────────────────────────────────────────────────────────┘

Server_042 receives message on channel comments:video:456

Parse message:
  comment = JSON.parse(message)
  video_id = comment.video_id  // "video_123"

Check if any viewers connected:
  if video_123 in connections:
    viewer_connections = connections[video_123]
    // e.g., 1,234 viewers connected to this server

┌──────────────────────────────────────────────────────────────────┐
│  STEP 4: BROADCAST TO CONNECTED VIEWERS                          │
└──────────────────────────────────────────────────────────────────┘

For each connection in viewer_connections:
  try:
    send_sse_event(
      connection_id,
      event_id: comment.comment_id,
      event_type: "comment",
      data: comment_json
    )
  except ConnectionClosed:
    // Remove dead connection
    connections[video_123].remove(connection_id)

SSE Message Format:
  id: comment_789
  event: comment
  data: {"comment_id":"comment_789","user_id":"user_456",...}
  
  (blank line to flush)

Latency per viewer: ~5ms (network + serialization)

┌──────────────────────────────────────────────────────────────────┐
│  STEP 5: CLIENT RECEIVES AND DISPLAYS COMMENT                    │
└──────────────────────────────────────────────────────────────────┘

Client's SSE handler:
  onmessage = (event) => {
    if (event.type === "comment") {
      comment = JSON.parse(event.data)
      
      // Check if this is user's own comment
      if (comment.user_id === current_user_id) {
        // Update optimistic comment to confirmed
        updateCommentStatus(comment.comment_id, "confirmed")
      } else {
        // Add new comment to feed
        addCommentToFeed(comment)
      }
      
      // Smooth scroll animation
      animateNewComment(comment)
      
      // Update last_event_id for reconnection
      localStorage.setItem("last_comment_id", comment.comment_id)
    }
  }

Total End-to-End Latency:
  Comment creation: 70ms
  Redis Pub/Sub: 2ms
  Server processing: 5ms
  Network to client: 50ms (varies by location)
  ─────────────────────
  Total: ~127ms ✓ (< 200ms target)
```

### Flow 3: Historical Comment Loading

```
┌──────────────────────────────────────────────────────────────────┐
│  VIEWER JOINS VIDEO - LOAD RECENT COMMENTS                       │
└──────────────────────────────────────────────────────────────────┘

Client requests:
  GET /v1/videos/video_123/comments?limit=50

Query Service:
  1. Check Redis cache first
     comments = LRANGE stream:video_123:comments 0 49
     
  2. If cache hit (common case):
     - Return cached comments
     - Latency: ~10ms
     
  3. If cache miss (rare - video just started):
     - Query DynamoDB with GSI
     - Query(
         IndexName: "video_id-created_at-index",
         KeyConditionExpression: "video_id = :vid",
         ScanIndexForward: false,
         Limit: 50
       )
     - Latency: ~50ms
     - Populate Redis cache for future requests

Response:
  {
    comments: [...50 comments...],
    pagination: {
      next_cursor: "comment_740#1715548750000",
      has_more: true
    }
  }

Client:
  1. Display comments in feed
  2. Establish SSE connection for real-time updates
  3. Enable infinite scroll for older comments
```

### Flow 4: Reconnection and Catch-up

```
┌──────────────────────────────────────────────────────────────────┐
│  VIEWER DISCONNECTS AND RECONNECTS                               │
└──────────────────────────────────────────────────────────────────┘

Scenario:
  - Viewer was watching video
  - Network hiccup (5 seconds)
  - Last comment received: comment_785
  - Missed comments: comment_786, comment_787, comment_788

Client detects disconnect:
  SSE connection closed
  Show "Reconnecting..." indicator

Client reconnects:
  1. Attempt SSE reconnection with Last-Event-ID
     GET /v1/videos/video_123/comments/stream
     Last-Event-ID: comment_785
     
  2. Server checks if can replay from Last-Event-ID
     - Check Redis cache: LRANGE stream:video_123:comments 0 -1
     - Find position of comment_785
     - If found and < 5 minutes old: replay missed comments
     - If not found or too old: use catch-up API

Catch-up API (if needed):
  GET /v1/videos/video_123/comments/catchup?since=comment_785&limit=100

Query Service:
  1. Query Redis cache for recent comments
  2. Filter comments after comment_785
  3. Return missed comments

Response:
  {
    comments: [comment_786, comment_787, comment_788],
    missed_count: 3,
    returned_count: 3,
    truncated: false,
    latest_comment_id: "comment_788"
  }

Client:
  1. Animate missed comments at 2x speed
  2. Resume SSE stream from comment_788
  3. Hide "Reconnecting..." indicator
  4. Show "Caught up!" message briefly

Total reconnection time: ~2 seconds
```

### Critical Path Analysis:

```
Comment Creation → Delivery:
  API Gateway:        20ms
  Comment Service:    10ms
  DynamoDB Write:     15ms
  Redis Update:       5ms
  Redis Pub/Sub:      2ms
  Server Broadcast:   5ms
  Network to Client:  50ms
  ─────────────────────────
  Total:              107ms ✓ (< 200ms target)

Bottlenecks:
  1. Network latency (50ms) - unavoidable, use edge locations
  2. DynamoDB write (15ms) - acceptable, can't optimize much
  3. API Gateway (20ms) - can optimize with caching

Optimizations:
  1. Pipeline Redis operations (5ms → 2ms)
  2. Cache user data to avoid lookups (10ms saved)
  3. Use regional DynamoDB endpoints (15ms → 10ms)
  4. Optimize JSON serialization (5ms → 2ms)
```



---

## PHASE 8: DEEP DIVE - REAL-TIME COMMENT BROADCASTING (6 minutes)

### What to Say:

"Let me address the critical challenge of broadcasting comments to millions of viewers in real-time. We need to choose the right technology and handle the massive read amplification."

### Problem Statement:

```
Challenge: Broadcast comments to all viewers with <200ms latency

Naive Approach: Polling
┌──────────────────────────────────────────────────────────────────┐
│  Client polls every 2 seconds:                                   │
│  GET /v1/videos/video_123/comments?since=comment_785             │
│                                                                  │
│  Problems:                                                       │
│  1. High latency (average 1 second delay)                       │
│  2. Wasted requests (most polls return empty)                   │
│  3. Database load (1B viewers × 0.5 QPS = 500M QPS!)           │
│  4. Not truly "real-time"                                       │
└──────────────────────────────────────────────────────────────────┘

We need a push-based model where server sends updates to clients.
```

### Technology Comparison: WebSockets vs SSE

```
┌─────────────────────────────────────────────────────────────────┐
│  APPROACH 1: WEBSOCKETS                                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Architecture:                                                  │
│  ┌──────────┐  WebSocket  ┌──────────┐  Pub/Sub  ┌──────────┐ │
│  │  Client  │◄───────────►│  Server  │◄─────────►│  Redis   │ │
│  └──────────┘             └──────────┘           └──────────┘ │
│       ▲                         │                              │
│       │ Bidirectional           │ Bidirectional                │
│       └─────────────────────────┘                              │
│                                                                 │
│  How it works:                                                  │
│  1. Client opens WebSocket connection                          │
│  2. Client sends: {"action": "subscribe", "video_id": "123"}   │
│  3. Server subscribes to Redis channel                         │
│  4. Server pushes comments to client                           │
│  5. Client can send comments over same connection              │
│                                                                 │
│  Pros:                                                          │
│  ✓ True bidirectional communication                            │
│  ✓ Low latency (<10ms overhead)                                │
│  ✓ Efficient binary protocol                                   │
│  ✓ Good for balanced read/write workloads                      │
│                                                                 │
│  Cons:                                                          │
│  ✗ Overkill for read-heavy workload (1:10,909 ratio)          │
│  ✗ More complex to implement and debug                         │
│  ✗ Requires special proxy/load balancer support                │
│  ✗ Higher server resource usage (full duplex)                  │
│  ✗ Not standard HTTP (firewall issues)                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  APPROACH 2: SERVER-SENT EVENTS (SSE) ✓ CHOSEN                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Architecture:                                                  │
│  ┌──────────┐  SSE (Read)  ┌──────────┐  Pub/Sub  ┌──────────┐│
│  │  Client  │◄─────────────│  Server  │◄─────────►│  Redis   ││
│  └──────────┘              └──────────┘           └──────────┘│
│       │                         ▲                              │
│       │ HTTP POST (Write)       │                              │
│       └─────────────────────────┘                              │
│                                                                 │
│  How it works:                                                  │
│  1. Client opens SSE connection (GET /stream)                  │
│  2. Server subscribes to Redis channel                         │
│  3. Server pushes comments via SSE                             │
│  4. Client posts comments via separate HTTP POST               │
│                                                                 │
│  SSE Message Format:                                            │
│  id: comment_789                                                │
│  event: comment                                                 │
│  data: {"comment_id":"comment_789","message":"Great!"}         │
│  (blank line)                                                   │
│                                                                 │
│  Pros:                                                          │
│  ✓ Perfect for read-heavy workloads                            │
│  ✓ Standard HTTP (works through proxies)                       │
│  ✓ Built-in reconnection with Last-Event-ID                    │
│  ✓ Simpler than WebSockets                                     │
│  ✓ Lower server resource usage (one-way)                       │
│  ✓ Native browser support (EventSource API)                    │
│  ✓ Text-based (easy to debug)                                  │
│                                                                 │
│  Cons:                                                          │
│  ✗ Browser limit: 6 connections per domain                     │
│  ✗ Text-only (no binary)                                       │
│  ✗ Some proxies buffer responses                               │
│                                                                 │
│  Mitigations:                                                   │
│  - Connection limit: Use subdomains (stream1.fb.com, etc.)     │
│  - Buffering: Set proper headers (X-Accel-Buffering: no)       │
│  - Binary: JSON is fine for comments                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Decision: SSE is the better choice
- Read:Write ratio is 1:10,909 (extremely read-heavy)
- Simpler implementation and debugging
- Standard HTTP works everywhere
- Built-in reconnection support
- Lower resource usage
```

### SSE Implementation Details:

```python
# Realtime Messaging Server - SSE Handler

class SSEConnectionHandler:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.pubsub = self.redis_client.pubsub()
        self.connections = {}  # video_id → [connection_ids]
        self.subscribed_channels = set()
        
    async def handle_sse_connection(self, request):
        """Handle incoming SSE connection"""
        video_id = request.path_params['video_id']
        user_id = request.user.id
        connection_id = str(uuid.uuid4())
        
        # Validate video is live
        video_status = self.redis_client.hget(f"video:{video_id}", "status")
        if video_status != "LIVE":
            return Response("Video not live", status_code=404)
        
        # Set SSE headers
        headers = {
            'Content-Type': 'text/event-stream',
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive',
            'X-Accel-Buffering': 'no'  # Disable nginx buffering
        }
        
        # Create SSE response stream
        async def event_stream():
            try:
                # Register connection
                self.register_connection(video_id, connection_id)
                
                # Subscribe to Redis channel if needed
                channel = self.get_channel_for_video(video_id)
                if channel not in self.subscribed_channels:
                    self.pubsub.subscribe(channel)
                    self.subscribed_channels.add(channel)
                
                # Handle Last-Event-ID for reconnection
                last_event_id = request.headers.get('Last-Event-ID')
                if last_event_id:
                    # Send missed comments
                    missed_comments = await self.get_missed_comments(
                        video_id, last_event_id
                    )
                    for comment in missed_comments:
                        yield self.format_sse_message(
                            event_id=comment['comment_id'],
                            event_type='comment',
                            data=comment
                        )
                
                # Send initial heartbeat
                yield self.format_sse_message(
                    event_type='heartbeat',
                    data={'timestamp': int(time.time() * 1000)}
                )
                
                # Main event loop
                while True:
                    # Check for new messages from Redis Pub/Sub
                    message = self.pubsub.get_message(timeout=30)
                    
                    if message and message['type'] == 'message':
                        # Parse comment
                        comment = json.loads(message['data'])
                        
                        # Only send if for this video
                        if comment['video_id'] == video_id:
                            yield self.format_sse_message(
                                event_id=comment['comment_id'],
                                event_type='comment',
                                data=comment
                            )
                    else:
                        # Send heartbeat every 30 seconds
                        yield self.format_sse_message(
                            event_type='heartbeat',
                            data={'timestamp': int(time.time() * 1000)}
                        )
                    
                    # Check if video ended
                    video_status = self.redis_client.hget(
                        f"video:{video_id}", "status"
                    )
                    if video_status == "ENDED":
                        yield self.format_sse_message(
                            event_type='video_ended',
                            data={'video_id': video_id}
                        )
                        break
                        
            except asyncio.CancelledError:
                # Client disconnected
                pass
            finally:
                # Cleanup
                self.unregister_connection(video_id, connection_id)
        
        return StreamingResponse(event_stream(), headers=headers)
    
    def format_sse_message(self, event_id=None, event_type=None, data=None):
        """Format SSE message according to spec"""
        message = ""
        if event_id:
            message += f"id: {event_id}\n"
        if event_type:
            message += f"event: {event_type}\n"
        if data:
            message += f"data: {json.dumps(data)}\n"
        message += "\n"  # Blank line to flush
        return message
    
    def register_connection(self, video_id, connection_id):
        """Register new connection"""
        if video_id not in self.connections:
            self.connections[video_id] = set()
        self.connections[video_id].add(connection_id)
        
        # Update server registry in Redis
        self.redis_client.sadd(
            f"video:{video_id}:servers",
            self.server_id
        )
        self.redis_client.expire(f"video:{video_id}:servers", 30)
    
    def unregister_connection(self, video_id, connection_id):
        """Unregister connection"""
        if video_id in self.connections:
            self.connections[video_id].discard(connection_id)
            
            # If no more connections for this video, unsubscribe
            if not self.connections[video_id]:
                del self.connections[video_id]
                channel = self.get_channel_for_video(video_id)
                self.pubsub.unsubscribe(channel)
                self.subscribed_channels.discard(channel)
    
    def get_channel_for_video(self, video_id):
        """Get Redis Pub/Sub channel for video"""
        channel_id = hash(video_id) % 1000
        return f"comments:video:{channel_id}"
    
    async def get_missed_comments(self, video_id, last_comment_id):
        """Get comments missed during disconnect"""
        # Get recent comments from Redis cache
        comments_json = self.redis_client.lrange(
            f"stream:{video_id}:comments",
            0, 999
        )
        
        comments = [json.loads(c) for c in comments_json]
        
        # Find position of last_comment_id
        last_index = None
        for i, comment in enumerate(comments):
            if comment['comment_id'] == last_comment_id:
                last_index = i
                break
        
        if last_index is not None:
            # Return comments after last_comment_id
            return comments[:last_index]
        else:
            # Last comment not in cache (too old)
            # Return last 50 comments
            return comments[:50]
```

### Client-Side SSE Implementation:

```javascript
// Client-Side SSE Handler

class LiveCommentsClient {
    constructor(videoId, userId) {
        this.videoId = videoId;
        this.userId = userId;
        this.eventSource = null;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 5;
        this.lastCommentId = localStorage.getItem(`last_comment_${videoId}`);
    }
    
    connect() {
        // Build SSE URL
        let url = `/v1/videos/${this.videoId}/comments/stream`;
        
        // Add Last-Event-ID if reconnecting
        if (this.lastCommentId) {
            // Browser will automatically send Last-Event-ID header
            // if we set it in EventSource constructor
        }
        
        // Create EventSource
        this.eventSource = new EventSource(url);
        
        // Handle connection open
        this.eventSource.addEventListener('open', (e) => {
            console.log('SSE connection established');
            this.reconnectAttempts = 0;
            this.showConnectionStatus('connected');
        });
        
        // Handle comment events
        this.eventSource.addEventListener('comment', (e) => {
            const comment = JSON.parse(e.data);
            this.handleNewComment(comment);
            
            // Save last comment ID for reconnection
            this.lastCommentId = e.lastEventId;
            localStorage.setItem(`last_comment_${this.videoId}`, e.lastEventId);
        });
        
        // Handle heartbeat events
        this.eventSource.addEventListener('heartbeat', (e) => {
            const data = JSON.parse(e.data);
            console.log('Heartbeat received:', data.timestamp);
            this.updateLastHeartbeat(data.timestamp);
        });
        
        // Handle video ended event
        this.eventSource.addEventListener('video_ended', (e) => {
            console.log('Video ended');
            this.showVideoEndedMessage();
            this.disconnect();
        });
        
        // Handle errors
        this.eventSource.addEventListener('error', (e) => {
            console.error('SSE error:', e);
            
            if (this.eventSource.readyState === EventSource.CLOSED) {
                // Connection closed, attempt reconnect
                this.handleDisconnect();
            }
        });
    }
    
    handleNewComment(comment) {
        // Check if this is user's own comment
        if (comment.user_id === this.userId) {
            // Update optimistic comment to confirmed
            this.updateOptimisticComment(comment.comment_id, 'confirmed');
        } else {
            // Add new comment to feed
            this.addCommentToFeed(comment);
        }
        
        // Smooth scroll animation
        this.animateNewComment(comment);
    }
    
    handleDisconnect() {
        this.showConnectionStatus('disconnected');
        
        // Exponential backoff for reconnection
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
            const delay = Math.min(1000 * Math.pow(2, this.reconnectAttempts), 30000);
            console.log(`Reconnecting in ${delay}ms...`);
            
            setTimeout(() => {
                this.reconnectAttempts++;
                this.connect();
            }, delay);
        } else {
            this.showConnectionStatus('failed');
            this.showManualReconnectButton();
        }
    }
    
    postComment(message) {
        // Optimistic update
        const optimisticComment = {
            comment_id: `temp_${Date.now()}`,
            user_id: this.userId,
            user_name: this.currentUserName,
            user_avatar: this.currentUserAvatar,
            message: message,
            created_at: Date.now(),
            status: 'pending'
        };
        
        this.addCommentToFeed(optimisticComment);
        
        // Send to server
        fetch(`/v1/videos/${this.videoId}/comments`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${this.authToken}`,
                'X-Request-ID': optimisticComment.comment_id
            },
            body: JSON.stringify({
                message: message,
                client_timestamp: Date.now()
            })
        })
        .then(response => response.json())
        .then(data => {
            // Server confirmed, will receive via SSE
            // Update optimistic comment ID
            this.updateOptimisticCommentId(
                optimisticComment.comment_id,
                data.comment_id
            );
        })
        .catch(error => {
            // Failed to post
            this.updateOptimisticComment(
                optimisticComment.comment_id,
                'failed'
            );
            this.showError('Failed to post comment');
        });
    }
    
    disconnect() {
        if (this.eventSource) {
            this.eventSource.close();
            this.eventSource = null;
        }
    }
}

// Usage
const commentsClient = new LiveCommentsClient('video_123', 'user_456');
commentsClient.connect();
```

### Key Optimizations:

```
1. Connection Pooling
   - Reuse SSE connections across page navigation
   - Maintain connection even when video minimized
   - Close connection only when user leaves site

2. Heartbeat Management
   - Server sends heartbeat every 30 seconds
   - Client detects missed heartbeats
   - Automatic reconnection if no heartbeat for 60 seconds

3. Buffering Prevention
   - Set X-Accel-Buffering: no header
   - Configure nginx/proxy to disable buffering
   - Use chunked transfer encoding

4. Browser Connection Limits
   - Use subdomains: stream1.fb.com, stream2.fb.com
   - Rotate connections across subdomains
   - Limit to 1 SSE connection per video

5. Mobile Optimization
   - Detect app backgrounding
   - Close SSE connection when backgrounded
   - Reconnect with catch-up when foregrounded
   - Saves battery and data
```



---

## PHASE 9: DEEP DIVE - HORIZONTAL SCALING & COORDINATION (5 minutes)

### What to Say:

"Let me address how we scale to handle millions of concurrent videos and billions of viewers. The key challenge is coordinating comment distribution across thousands of servers."

### The Coordination Problem:

```
Problem: Viewers of same video connected to different servers

Scenario:
┌──────────────────────────────────────────────────────────────────┐
│  Video: video_123 (10,000 viewers)                               │
│                                                                  │
│  Server_001: 3,000 viewers                                       │
│  Server_002: 2,500 viewers                                       │
│  Server_003: 2,000 viewers                                       │
│  Server_004: 1,500 viewers                                       │
│  Server_005: 1,000 viewers                                       │
└──────────────────────────────────────────────────────────────────┘

Challenge:
- User on Server_001 posts comment
- Comment must reach viewers on ALL servers
- How do servers communicate?
```

### Solution 1: Simple Pub/Sub (Naive Approach)

```
┌─────────────────────────────────────────────────────────────────┐
│  APPROACH: ONE CHANNEL PER VIDEO                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Architecture:                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Server 1 │  │ Server 2 │  │ Server 3 │  │ Server N │      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘      │
│       │             │             │             │              │
│       └─────────────┼─────────────┼─────────────┘              │
│                     ▼             ▼                            │
│              ┌─────────────────────────┐                       │
│              │   Redis Pub/Sub         │                       │
│              │                         │                       │
│              │  Channel: video_123     │                       │
│              │  Channel: video_456     │                       │
│              │  Channel: video_789     │                       │
│              │  ... 1M channels        │                       │
│              └─────────────────────────┘                       │
│                                                                 │
│  How it works:                                                  │
│  1. Each server subscribes to ALL video channels               │
│  2. When comment posted, publish to video's channel            │
│  3. All servers receive message                                │
│  4. Each server broadcasts to its connected viewers            │
│                                                                 │
│  Problems:                                                      │
│  ✗ Each server subscribes to 1M channels                       │
│  ✗ Most messages irrelevant (server has no viewers)            │
│  ✗ Wasted CPU processing irrelevant messages                   │
│  ✗ Memory overhead for 1M subscriptions                        │
│  ✗ Doesn't scale beyond 10K videos                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Solution 2: Channel Sharding with Viewer Co-location

```
┌─────────────────────────────────────────────────────────────────┐
│  APPROACH: HASH-BASED CHANNEL SHARDING + SMART ROUTING         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Step 1: Shard videos into N channels (e.g., 1000 channels)    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  channel_id = hash(video_id) % 1000                      │  │
│  │                                                          │  │
│  │  video_123 → channel_456                                 │  │
│  │  video_456 → channel_789                                 │  │
│  │  video_789 → channel_123                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Step 2: Use Layer 7 Load Balancer with Consistent Hashing     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Viewer connects to watch video_123                      │  │
│  │  Load balancer: server_id = consistent_hash(video_123)   │  │
│  │  Route to: Server_042                                    │  │
│  │                                                          │  │
│  │  Result: All viewers of video_123 → Server_042          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Step 3: Servers subscribe only to relevant channels           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Server_042 has viewers for:                             │  │
│  │  - video_123 (channel_456)                               │  │
│  │  - video_789 (channel_123)                               │  │
│  │  - video_321 (channel_456)  // Same channel!            │  │
│  │                                                          │  │
│  │  Server_042 subscribes to: [channel_456, channel_123]   │  │
│  │  Only 2 subscriptions instead of 1M!                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Architecture:                                                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Layer 7 Load Balancer (NGINX/Envoy)                    │  │
│  │                                                          │  │
│  │  upstream backend {                                      │  │
│  │    hash $request_uri consistent;                        │  │
│  │    server server_001:8080;                              │  │
│  │    server server_002:8080;                              │  │
│  │    ...                                                   │  │
│  │  }                                                       │  │
│  └─────────────────────────────────────────────────────────┘  │
│                     │                                          │
│                     ▼                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │Server 001│  │Server 002│  │Server 003│  │Server N  │     │
│  │          │  │          │  │          │  │          │     │
│  │Subscribed│  │Subscribed│  │Subscribed│  │Subscribed│     │
│  │channels: │  │channels: │  │channels: │  │channels: │     │
│  │[1,5,9]   │  │[2,6,10]  │  │[3,7,11]  │  │[4,8,12]  │     │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘     │
│       └─────────────┼─────────────┼─────────────┘            │
│                     ▼             ▼                           │
│              ┌─────────────────────────┐                      │
│              │   Redis Pub/Sub         │                      │
│              │                         │                      │
│              │  Channel 0              │                      │
│              │  Channel 1              │                      │
│              │  ...                    │                      │
│              │  Channel 999            │                      │
│              └─────────────────────────┘                      │
│                                                                 │
│  Benefits:                                                      │
│  ✓ Each server subscribes to ~10-50 channels (not 1M)         │
│  ✓ Viewers of same video on same server                       │
│  ✓ Minimal wasted message processing                          │
│  ✓ Scales to millions of videos                               │
│                                                                 │
│  Challenges:                                                    │
│  ⚠️  Load balancer must support consistent hashing              │
│  ⚠️  Server failures require rebalancing                        │
│  ⚠️  Popular videos may overload single server                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Solution 3: Dispatcher Service (Alternative)

```
┌─────────────────────────────────────────────────────────────────┐
│  APPROACH: CENTRALIZED DISPATCHER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Architecture:                                                  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Comment Management Service                              │  │
│  └────────────────┬─────────────────────────────────────────┘  │
│                   │ New comment for video_123                  │
│                   ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Dispatcher Service                                      │  │
│  │                                                          │  │
│  │  1. Receive comment for video_123                       │  │
│  │  2. Query: Which servers have viewers for video_123?    │  │
│  │     → [server_042, server_087, server_123]              │  │
│  │  3. Send comment directly to those servers              │  │
│  └────────────────┬─────────────────────────────────────────┘  │
│                   │                                            │
│       ┌───────────┼───────────┐                                │
│       ▼           ▼           ▼                                │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                          │
│  │Server042│ │Server087│ │Server123│                          │
│  │         │ │         │ │         │                          │
│  │Viewers: │ │Viewers: │ │Viewers: │                          │
│  │3,000    │ │2,500    │ │1,500    │                          │
│  └─────────┘ └─────────┘ └─────────┘                          │
│                                                                 │
│  Server Registry (Redis):                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Key: video:video_123:servers                            │  │
│  │  Value: Set[server_042, server_087, server_123]         │  │
│  │  TTL: 30 seconds (refreshed by heartbeat)               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Heartbeat Protocol:                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Every 10 seconds, each server sends:                   │  │
│  │  {                                                       │  │
│  │    "server_id": "server_042",                           │  │
│  │    "videos": ["video_123", "video_456", ...],           │  │
│  │    "connection_count": 85234,                           │  │
│  │    "capacity": 100000                                   │  │
│  │  }                                                       │  │
│  │                                                          │  │
│  │  Dispatcher updates:                                     │  │
│  │  SADD video:video_123:servers server_042                │  │
│  │  EXPIRE video:video_123:servers 30                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Benefits:                                                      │
│  ✓ No pub/sub subscriptions needed                             │
│  ✓ Precise routing (only to servers with viewers)              │
│  ✓ Easy to add routing logic (load-based, geo-based)           │
│  ✓ Centralized monitoring and control                          │
│                                                                 │
│  Challenges:                                                    │
│  ⚠️  Dispatcher is single point of failure (need redundancy)    │
│  ⚠️  Registry must stay in sync (stale data issues)             │
│  ⚠️  Additional network hop (latency)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Comparison & Decision:

```
┌─────────────────────────────────────────────────────────────────┐
│  SOLUTION COMPARISON                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Metric              │ Simple    │ Sharding  │ Dispatcher      │
│                      │ Pub/Sub   │ + Routing │ Service         │
│  ────────────────────┼───────────┼───────────┼─────────────    │
│  Scalability         │ Poor      │ Excellent │ Good            │
│  Latency             │ Low       │ Low       │ Medium          │
│  Complexity          │ Low       │ Medium    │ High            │
│  Operational Overhead│ Low       │ Medium    │ High            │
│  Wasted Messages     │ High      │ Low       │ None            │
│  Failure Handling    │ Simple    │ Medium    │ Complex         │
│  ────────────────────┴───────────┴───────────┴─────────────    │
│                                                                 │
│  DECISION: Sharding + Routing (Solution 2) ✓                   │
│                                                                 │
│  Reasons:                                                       │
│  1. Best scalability (handles millions of videos)              │
│  2. Low latency (no additional hops)                           │
│  3. Reasonable complexity                                       │
│  4. Proven at scale (used by major platforms)                  │
│  5. No single point of failure                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation Details:

```python
# Realtime Messaging Server - Dynamic Subscription Management

class RealtimeMessagingServer:
    def __init__(self, server_id):
        self.server_id = server_id
        self.redis_client = redis.Redis()
        self.pubsub = self.redis_client.pubsub()
        self.connections = {}  # video_id → set(connection_ids)
        self.subscribed_channels = set()
        self.channel_to_videos = {}  # channel_id → set(video_ids)
        
    def register_viewer(self, video_id, connection_id):
        """Register new viewer connection"""
        # Add to connections map
        if video_id not in self.connections:
            self.connections[video_id] = set()
        self.connections[video_id].add(connection_id)
        
        # Subscribe to channel if needed
        channel_id = self.get_channel_for_video(video_id)
        if channel_id not in self.subscribed_channels:
            self.subscribe_to_channel(channel_id)
        
        # Track video → channel mapping
        if channel_id not in self.channel_to_videos:
            self.channel_to_videos[channel_id] = set()
        self.channel_to_videos[channel_id].add(video_id)
        
        # Register with dispatcher (optional)
        self.redis_client.sadd(
            f"video:{video_id}:servers",
            self.server_id
        )
        self.redis_client.expire(f"video:{video_id}:servers", 30)
        
        print(f"Registered viewer for {video_id}, channel {channel_id}")
        print(f"Total subscribed channels: {len(self.subscribed_channels)}")
    
    def unregister_viewer(self, video_id, connection_id):
        """Unregister viewer connection"""
        if video_id in self.connections:
            self.connections[video_id].discard(connection_id)
            
            # If no more viewers for this video
            if not self.connections[video_id]:
                del self.connections[video_id]
                
                # Check if we can unsubscribe from channel
                channel_id = self.get_channel_for_video(video_id)
                self.channel_to_videos[channel_id].discard(video_id)
                
                # If no more videos on this channel, unsubscribe
                if not self.channel_to_videos[channel_id]:
                    self.unsubscribe_from_channel(channel_id)
                    del self.channel_to_videos[channel_id]
    
    def subscribe_to_channel(self, channel_id):
        """Subscribe to Redis Pub/Sub channel"""
        channel_name = f"comments:video:{channel_id}"
        self.pubsub.subscribe(channel_name)
        self.subscribed_channels.add(channel_id)
        print(f"Subscribed to channel {channel_id}")
    
    def unsubscribe_from_channel(self, channel_id):
        """Unsubscribe from Redis Pub/Sub channel"""
        channel_name = f"comments:video:{channel_id}"
        self.pubsub.unsubscribe(channel_name)
        self.subscribed_channels.discard(channel_id)
        print(f"Unsubscribed from channel {channel_id}")
    
    def get_channel_for_video(self, video_id):
        """Get channel ID for video using consistent hashing"""
        return hash(video_id) % 1000
    
    async def message_loop(self):
        """Main loop to process Redis Pub/Sub messages"""
        while True:
            message = self.pubsub.get_message(timeout=1)
            
            if message and message['type'] == 'message':
                # Parse comment
                comment = json.loads(message['data'])
                video_id = comment['video_id']
                
                # Check if we have viewers for this video
                if video_id in self.connections:
                    # Broadcast to all viewers
                    await self.broadcast_comment(video_id, comment)
    
    async def broadcast_comment(self, video_id, comment):
        """Broadcast comment to all viewers of video"""
        if video_id not in self.connections:
            return
        
        connections = self.connections[video_id]
        print(f"Broadcasting comment to {len(connections)} viewers")
        
        # Format SSE message
        sse_message = self.format_sse_message(
            event_id=comment['comment_id'],
            event_type='comment',
            data=comment
        )
        
        # Send to all connections
        dead_connections = []
        for connection_id in connections:
            try:
                await self.send_to_connection(connection_id, sse_message)
            except ConnectionClosed:
                dead_connections.append(connection_id)
        
        # Clean up dead connections
        for connection_id in dead_connections:
            self.unregister_viewer(video_id, connection_id)
    
    def send_heartbeat(self):
        """Send heartbeat to dispatcher"""
        heartbeat = {
            "server_id": self.server_id,
            "timestamp": int(time.time() * 1000),
            "connection_count": sum(len(conns) for conns in self.connections.values()),
            "video_count": len(self.connections),
            "subscribed_channels": len(self.subscribed_channels),
            "videos": list(self.connections.keys())
        }
        
        # Update server registry
        self.redis_client.setex(
            f"server:{self.server_id}",
            30,
            json.dumps(heartbeat)
        )
        
        # Update video → server mappings
        for video_id in self.connections.keys():
            self.redis_client.sadd(f"video:{video_id}:servers", self.server_id)
            self.redis_client.expire(f"video:{video_id}:servers", 30)
```

### Load Balancer Configuration (NGINX):

```nginx
# NGINX Configuration for Consistent Hashing

upstream realtime_messaging_servers {
    # Use consistent hashing based on request URI
    hash $request_uri consistent;
    
    # Server pool
    server server_001:8080 max_fails=3 fail_timeout=30s;
    server server_002:8080 max_fails=3 fail_timeout=30s;
    server server_003:8080 max_fails=3 fail_timeout=30s;
    # ... 10,000 servers
}

server {
    listen 443 ssl http2;
    server_name stream.facebook.com;
    
    # SSE specific settings
    location /v1/videos/*/comments/stream {
        proxy_pass http://realtime_messaging_servers;
        
        # SSE headers
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        chunked_transfer_encoding on;
        
        # Disable buffering for SSE
        proxy_buffering off;
        proxy_cache off;
        
        # Timeouts
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### Handling Server Failures:

```
Scenario: Server_042 crashes

Before:
  video_123 → [server_042, server_087, server_123]
  server_042: 3,000 viewers

After crash:
  1. Load balancer detects failure (health check)
  2. New viewers routed to server_087 or server_123
  3. Existing viewers' connections drop
  4. Clients auto-reconnect (SSE built-in)
  5. Load balancer routes to different server
  6. Viewers catch up on missed comments

Recovery time: ~5 seconds
Comment loss: None (catch-up mechanism)
```



---

## PHASE 10: DEEP DIVE - MEGA-STREAMS & CDN STRATEGY (5 minutes)

### What to Say:

"Let me address the extreme case: a mega-viral video with 10 million viewers and 10,000 comments per second. At this scale, our normal architecture breaks down and we need a fundamentally different approach."

### The Mega-Stream Problem:

```
Scenario: World Cup Final Live Stream
- 10M concurrent viewers
- 10K comments/second
- Average comment: 4 words

Math:
- 10K comments/sec × 4 words = 40K words/second
- War and Peace: ~587K words
- Time to read War and Peace at this rate: 14.7 seconds!

UI Reality:
- Display last 20 comments on screen
- Each comment visible for: 20 / 10,000 = 0.002 seconds = 2ms
- Human reaction time: 200ms
- Humans can't read comments flowing this fast!

Infrastructure Reality:
- 10K comments/sec × 10M viewers = 100B messages/second
- Even with our optimized architecture, this is unsustainable
- Cost would be astronomical

Key Insight: At mega-scale, requirements change
- Users don't need to see EVERY comment
- They need to feel the "vibe" of the crowd
- Sampling and approximation become acceptable
```

### Solution 1: Comment Sampling

```
┌─────────────────────────────────────────────────────────────────┐
│  APPROACH: ADAPTIVE SAMPLING                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sampling Strategy:                                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  def calculate_sampling_rate(comments_per_second):       │  │
│  │      if comments_per_second < 100:                       │  │
│  │          return 1.0  # 100% - show all                   │  │
│  │      elif comments_per_second < 500:                     │  │
│  │          return 0.5  # 50% sampling                      │  │
│  │      elif comments_per_second < 1000:                    │  │
│  │          return 0.2  # 20% sampling                      │  │
│  │      elif comments_per_second < 5000:                    │  │
│  │          return 0.05 # 5% sampling                       │  │
│  │      else:                                                │  │
│  │          return 0.01 # 1% sampling                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Implementation:                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Comment Management Service:                             │  │
│  │                                                          │  │
│  │  1. Receive comment                                      │  │
│  │  2. Persist to database (always)                        │  │
│  │  3. Check comment rate for video                        │  │
│  │  4. Calculate sampling rate                             │  │
│  │  5. Random sampling decision:                           │  │
│  │     if random() < sampling_rate:                        │  │
│  │         publish_to_redis_pubsub(comment)                │  │
│  │     else:                                                │  │
│  │         skip_broadcast()                                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Smart Sampling (Priority-based):                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Prioritize certain comments:                            │  │
│  │  - Comments from verified users: 5x weight              │  │
│  │  - Comments from followed users: 3x weight              │  │
│  │  - Comments with reactions: 2x weight                   │  │
│  │  - Regular comments: 1x weight                          │  │
│  │                                                          │  │
│  │  def should_broadcast(comment, sampling_rate):           │  │
│  │      weight = calculate_weight(comment)                  │  │
│  │      adjusted_rate = min(1.0, sampling_rate * weight)   │  │
│  │      return random() < adjusted_rate                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Benefits:                                                      │
│  ✓ Reduces broadcast load by 99% for mega-streams              │
│  ✓ Maintains "vibe" of crowd participation                     │
│  ✓ Users still see steady stream of comments                   │
│  ✓ Important comments more likely to be seen                   │
│                                                                 │
│  Challenges:                                                    │
│  ⚠️  Still maintaining 10M SSE connections                      │
│  ⚠️  Still significant infrastructure cost                      │
│  ⚠️  Sampling rate calculation needs tuning                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Solution 2: CDN-Based Snapshot Delivery (Optimal)

```
┌─────────────────────────────────────────────────────────────────┐
│  APPROACH: CDN SNAPSHOT POLLING                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Key Insight: For mega-streams, switch from push to pull       │
│                                                                 │
│  Architecture:                                                  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Comment Aggregator Service                              │  │
│  │                                                          │  │
│  │  1. Maintain ring buffer of last 200 comments           │  │
│  │  2. Every 1 second, create snapshot:                    │  │
│  │     {                                                    │  │
│  │       "video_id": "video_123",                          │  │
│  │       "timestamp": 1715548800000,                       │  │
│  │       "comments": [                                      │  │
│  │         {comment_1},                                     │  │
│  │         {comment_2},                                     │  │
│  │         ...                                              │  │
│  │         {comment_100}  // Last 100 comments             │  │
│  │       ],                                                 │  │
│  │       "comment_rate": 9847,  // Comments per second     │  │
│  │       "viewer_count": 10234567                          │  │
│  │     }                                                    │  │
│  │  3. Write snapshot to Redis                             │  │
│  │  4. Publish to CDN origin                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                     │                                          │
│                     ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  CDN (CloudFront)                                        │  │
│  │                                                          │  │
│  │  URL: /snapshots/video_123/latest.json                  │  │
│  │  Cache-Control: max-age=1, stale-while-revalidate=2     │  │
│  │  Edge locations: 200+ globally                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                     │                                          │
│                     ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Client (10M viewers)                                    │  │
│  │                                                          │  │
│  │  setInterval(() => {                                     │  │
│  │    fetch('/snapshots/video_123/latest.json')            │  │
│  │      .then(snapshot => {                                 │  │
│  │        // Animate new comments smoothly                  │  │
│  │        animateComments(snapshot.comments);               │  │
│  │      });                                                 │  │
│  │  }, 1000);  // Poll every 1 second                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Client-Side Animation:                                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  function animateComments(newComments) {                 │  │
│  │    // Don't dump all comments at once                    │  │
│  │    // Spread them over the 1-second interval            │  │
│  │                                                          │  │
│  │    const interval = 1000 / newComments.length;          │  │
│  │    // e.g., 100 comments → 10ms per comment             │  │
│  │                                                          │  │
│  │    newComments.forEach((comment, index) => {            │  │
│  │      setTimeout(() => {                                  │  │
│  │        addCommentToFeed(comment);                       │  │
│  │      }, interval * index);                              │  │
│  │    });                                                   │  │
│  │  }                                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Threshold-Based Switching:                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  if (viewer_count > 100K || comment_rate > 500):        │  │
│  │      use_cdn_snapshot_mode()                             │  │
│  │  else:                                                   │  │
│  │      use_sse_realtime_mode()                             │  │
│  │                                                          │  │
│  │  Client handles transition seamlessly:                  │  │
│  │  - Close SSE connection                                 │  │
│  │  - Start polling CDN                                    │  │
│  │  - Smooth animation maintains UX                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Benefits:                                                      │
│  ✓ Leverages existing CDN infrastructure                       │
│  ✓ No SSE connections to maintain                              │
│  ✓ Scales to unlimited viewers (CDN's job)                     │
│  ✓ Dramatically lower cost                                     │
│  ✓ Better global latency (edge caching)                        │
│                                                                 │
│  Trade-offs:                                                    │
│  ⚠️  Latency: 1-2 seconds instead of <200ms                     │
│  ⚠️  But acceptable for mega-streams (users can't read anyway)  │
│  ⚠️  "Read your own write" needs special handling               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation: Comment Aggregator Service

```python
# Comment Aggregator Service for Mega-Streams

class CommentAggregator:
    def __init__(self):
        self.redis_client = redis.Redis()
        self.s3_client = boto3.client('s3')
        self.cloudfront_client = boto3.client('cloudfront')
        self.ring_buffers = {}  # video_id → deque(maxlen=200)
        
    async def run(self):
        """Main loop - create snapshots every second"""
        while True:
            # Get list of mega-streams
            mega_streams = self.get_mega_streams()
            
            for video_id in mega_streams:
                await self.create_snapshot(video_id)
            
            await asyncio.sleep(1)
    
    def get_mega_streams(self):
        """Get videos that qualify as mega-streams"""
        mega_streams = []
        
        # Query Redis for videos with high viewer count or comment rate
        for key in self.redis_client.scan_iter("video:*"):
            video_data = self.redis_client.hgetall(key)
            viewer_count = int(video_data.get('viewer_count', 0))
            comment_rate = float(video_data.get('comment_rate', 0))
            
            if viewer_count > 100000 or comment_rate > 500:
                video_id = key.decode().split(':')[1]
                mega_streams.append(video_id)
        
        return mega_streams
    
    async def create_snapshot(self, video_id):
        """Create and publish comment snapshot"""
        # Get recent comments from ring buffer
        if video_id not in self.ring_buffers:
            self.ring_buffers[video_id] = deque(maxlen=200)
        
        ring_buffer = self.ring_buffers[video_id]
        
        # Get latest comments from Redis
        comments_json = self.redis_client.lrange(
            f"stream:{video_id}:comments",
            0, 99  # Last 100 comments
        )
        
        comments = [json.loads(c) for c in comments_json]
        
        # Update ring buffer
        for comment in comments:
            if comment not in ring_buffer:
                ring_buffer.append(comment)
        
        # Get video metadata
        video_data = self.redis_client.hgetall(f"video:{video_id}")
        
        # Create snapshot
        snapshot = {
            "video_id": video_id,
            "timestamp": int(time.time() * 1000),
            "comments": list(ring_buffer)[-100:],  # Last 100
            "comment_rate": float(video_data.get('comment_rate', 0)),
            "viewer_count": int(video_data.get('viewer_count', 0)),
            "version": int(time.time())
        }
        
        # Write to Redis (for immediate access)
        self.redis_client.setex(
            f"snapshot:{video_id}:latest",
            5,  # 5 second TTL
            json.dumps(snapshot)
        )
        
        # Write to S3 (CDN origin)
        await self.publish_to_s3(video_id, snapshot)
        
        # Invalidate CloudFront cache
        await self.invalidate_cdn(video_id)
    
    async def publish_to_s3(self, video_id, snapshot):
        """Publish snapshot to S3"""
        key = f"snapshots/{video_id}/latest.json"
        
        self.s3_client.put_object(
            Bucket='fb-live-comments',
            Key=key,
            Body=json.dumps(snapshot),
            ContentType='application/json',
            CacheControl='max-age=1, stale-while-revalidate=2',
            Metadata={
                'video_id': video_id,
                'timestamp': str(snapshot['timestamp'])
            }
        )
    
    async def invalidate_cdn(self, video_id):
        """Invalidate CloudFront cache for snapshot"""
        # Note: In production, use versioned URLs instead of invalidation
        # e.g., /snapshots/video_123/v1715548800.json
        pass
    
    def add_comment_to_buffer(self, video_id, comment):
        """Add new comment to ring buffer (called by Comment Management Service)"""
        if video_id not in self.ring_buffers:
            self.ring_buffers[video_id] = deque(maxlen=200)
        
        self.ring_buffers[video_id].append(comment)
```

### Client-Side Implementation:

```javascript
// Client-Side Mega-Stream Handler

class MegaStreamClient {
    constructor(videoId) {
        this.videoId = videoId;
        this.mode = 'sse';  // 'sse' or 'cdn'
        this.sseClient = null;
        this.pollInterval = null;
        this.lastSnapshot = null;
    }
    
    async connect() {
        // Check if video is mega-stream
        const metadata = await this.getVideoMetadata();
        
        if (metadata.viewer_count > 100000 || metadata.comment_rate > 500) {
            this.switchToCDNMode();
        } else {
            this.switchToSSEMode();
        }
        
        // Monitor and switch modes dynamically
        setInterval(() => this.checkAndSwitchMode(), 30000);  // Every 30s
    }
    
    switchToSSEMode() {
        if (this.mode === 'sse') return;
        
        console.log('Switching to SSE mode');
        this.mode = 'sse';
        
        // Stop CDN polling
        if (this.pollInterval) {
            clearInterval(this.pollInterval);
            this.pollInterval = null;
        }
        
        // Start SSE connection
        this.sseClient = new LiveCommentsClient(this.videoId, this.userId);
        this.sseClient.connect();
    }
    
    switchToCDNMode() {
        if (this.mode === 'cdn') return;
        
        console.log('Switching to CDN mode');
        this.mode = 'cdn';
        
        // Close SSE connection
        if (this.sseClient) {
            this.sseClient.disconnect();
            this.sseClient = null;
        }
        
        // Start CDN polling
        this.startCDNPolling();
    }
    
    startCDNPolling() {
        // Initial fetch
        this.fetchSnapshot();
        
        // Poll every second
        this.pollInterval = setInterval(() => {
            this.fetchSnapshot();
        }, 1000);
    }
    
    async fetchSnapshot() {
        try {
            const response = await fetch(
                `/snapshots/${this.videoId}/latest.json`,
                {
                    cache: 'no-cache',  // Always get fresh data
                    headers: {
                        'Authorization': `Bearer ${this.authToken}`
                    }
                }
            );
            
            const snapshot = await response.json();
            
            // Check if this is a new snapshot
            if (!this.lastSnapshot || 
                snapshot.timestamp > this.lastSnapshot.timestamp) {
                
                this.processSnapshot(snapshot);
                this.lastSnapshot = snapshot;
            }
            
        } catch (error) {
            console.error('Failed to fetch snapshot:', error);
        }
    }
    
    processSnapshot(snapshot) {
        // Get new comments (not in previous snapshot)
        const newComments = this.getNewComments(snapshot.comments);
        
        if (newComments.length === 0) return;
        
        // Animate comments smoothly over 1 second
        const interval = 1000 / newComments.length;
        
        newComments.forEach((comment, index) => {
            setTimeout(() => {
                this.addCommentToFeed(comment);
            }, interval * index);
        });
        
        // Update stats
        this.updateStats({
            commentRate: snapshot.comment_rate,
            viewerCount: snapshot.viewer_count
        });
    }
    
    getNewComments(snapshotComments) {
        if (!this.lastSnapshot) {
            return snapshotComments;
        }
        
        const lastCommentIds = new Set(
            this.lastSnapshot.comments.map(c => c.comment_id)
        );
        
        return snapshotComments.filter(
            c => !lastCommentIds.has(c.comment_id)
        );
    }
    
    async checkAndSwitchMode() {
        const metadata = await this.getVideoMetadata();
        
        // Hysteresis to prevent flapping
        if (metadata.viewer_count > 150000 || metadata.comment_rate > 600) {
            this.switchToCDNMode();
        } else if (metadata.viewer_count < 80000 && metadata.comment_rate < 400) {
            this.switchToSSEMode();
        }
    }
    
    postComment(message) {
        // Always use HTTP POST for writing
        const optimisticComment = {
            comment_id: `temp_${Date.now()}`,
            user_id: this.userId,
            user_name: this.currentUserName,
            message: message,
            created_at: Date.now(),
            status: 'pending'
        };
        
        // Show immediately (optimistic update)
        this.addCommentToFeed(optimisticComment);
        
        // Send to server
        fetch(`/v1/videos/${this.videoId}/comments`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${this.authToken}`
            },
            body: JSON.stringify({ message })
        })
        .then(response => response.json())
        .then(data => {
            // Update optimistic comment
            this.updateOptimisticComment(
                optimisticComment.comment_id,
                data.comment_id,
                'confirmed'
            );
        })
        .catch(error => {
            this.updateOptimisticComment(
                optimisticComment.comment_id,
                null,
                'failed'
            );
        });
    }
}
```

### Cost Comparison:

```
Scenario: 10M viewers, 10K comments/second

SSE Approach:
- 10M SSE connections
- 10K servers (1K connections each)
- c5.4xlarge: $0.68/hour × 10K = $6,800/hour
- Monthly: $5M
- Bandwidth: 100B messages/sec × 300 bytes = 30TB/sec
- Data transfer: 30TB × 86,400 × $0.09/GB = $233M/day
- Total: ~$7B/month (unsustainable!)

CDN Approach:
- 0 SSE connections
- 10M HTTP requests/second to CDN
- CloudFront: $0.085/GB
- Snapshot size: 50KB
- Data transfer: 10M × 50KB × 86,400 = 43.2PB/day
- Cost: 43.2PB × $0.085/GB = $3.7M/day = $110M/month
- Plus origin: $100K/month
- Total: ~$110M/month (90% savings!)

With sampling (1%):
- SSE approach: $70M/month (still expensive)
- CDN approach: $110M/month (same, but scales better)

Decision: CDN approach for mega-streams (>100K viewers)
```

### Hybrid Strategy (Production):

```
┌─────────────────────────────────────────────────────────────────┐
│  PRODUCTION STRATEGY: TIERED APPROACH                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Tier 1: Normal Videos (<10K viewers)                           │
│  - Use SSE with full real-time delivery                         │
│  - <200ms latency                                               │
│  - Show all comments                                            │
│  - Cost: $5/hour per video                                      │
│                                                                 │
│  Tier 2: Popular Videos (10K-100K viewers)                      │
│  - Use SSE with 10-50% sampling                                 │
│  - <200ms latency                                               │
│  - Smart sampling (prioritize important comments)               │
│  - Cost: $50/hour per video                                     │
│                                                                 │
│  Tier 3: Mega-Streams (>100K viewers)                           │
│  - Use CDN snapshot polling                                     │
│  - 1-2 second latency                                           │
│  - Show sampled comments                                        │
│  - Cost: $1,000/hour per video                                  │
│                                                                 │
│  Automatic tier switching based on:                             │
│  - Viewer count                                                 │
│  - Comment rate                                                 │
│  - Cost projections                                             │
│  - Hysteresis to prevent flapping                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```



---

## FOLLOW-UP QUESTIONS & ADVANCED TOPICS

### Q1: "How would you handle comment moderation and spam filtering?"

**Answer:**
```
Multi-Layer Approach:

Layer 1: Client-Side Validation
- Character limit (500 chars)
- Rate limiting UI (disable button for 2 seconds)
- Profanity filter (basic)

Layer 2: API Gateway
- Rate limiting (10 comments/min per user)
- Token bucket algorithm in Redis
- Block known spam patterns

Layer 3: Async Moderation Pipeline
┌──────────────────────────────────────────────────────────────┐
│  Comment Created → Kafka Topic → Moderation Service          │
│                                                              │
│  Moderation Service:                                         │
│  1. ML model for spam detection (99% accuracy)              │
│  2. Profanity filter                                         │
│  3. Hate speech detection                                    │
│  4. Link spam detection                                      │
│                                                              │
│  If flagged:                                                 │
│  - High confidence (>95%): Auto-hide comment                │
│  - Medium confidence (70-95%): Queue for human review       │
│  - Low confidence (<70%): Allow but monitor                 │
└──────────────────────────────────────────────────────────────┘

Layer 4: User Reporting
- Users can report comments
- Threshold-based auto-hiding (10 reports → hide)
- Human moderator review queue

Implementation:
- Moderation happens async (doesn't block comment creation)
- Comments visible immediately, hidden retroactively if needed
- Broadcaster can enable "slow mode" (30s between comments)
```

### Q2: "How would you implement comment reactions (likes, hearts)?"

**Answer:**
```
Data Model Extension:
{
  "comment_id": "comment_789",
  "reactions": {
    "like": 1234,
    "love": 567,
    "laugh": 89
  },
  "user_reactions": {
    "user_456": "like",
    "user_789": "love"
  }
}

Real-time Updates:
1. User clicks reaction → HTTP POST
2. Update DynamoDB (increment counter)
3. Update Redis cache
4. Publish reaction event to Pub/Sub
5. Broadcast to viewers via SSE

Optimization:
- Batch reaction updates (every 5 seconds)
- Show optimistic update immediately
- Eventual consistency acceptable
- Use Redis sorted set for top reactions

SSE Event:
event: reaction
data: {"comment_id":"comment_789","reaction":"like","count":1235}
```

### Q3: "How would you support comment replies and threading?"

**Answer:**
```
Data Model:
{
  "comment_id": "comment_789",
  "parent_comment_id": null,  // Top-level comment
  "reply_count": 5,
  "replies": ["comment_790", "comment_791", ...]
}

UI Pattern:
- Show top-level comments in main feed
- "View 5 replies" button expands thread
- Replies loaded on-demand (not in real-time stream)
- Max depth: 2 levels (comment → reply, no nested replies)

API:
GET /v1/comments/{comment_id}/replies?cursor={cursor}&limit=20

Real-time:
- Only broadcast top-level comments via SSE
- Replies loaded via HTTP when user expands thread
- Reduces SSE bandwidth significantly

Trade-off:
- Replies not real-time (acceptable for UX)
- Simpler architecture
- Lower bandwidth costs
```

### Q4: "How would you handle different time zones and timestamps?"

**Answer:**
```
Timestamp Strategy:

Storage:
- Always store UTC timestamps (milliseconds since epoch)
- created_at: 1715548800125 (UTC)

Display:
- Client converts to local time zone
- JavaScript: new Date(timestamp).toLocaleString()
- Show relative time: "2 seconds ago", "5 minutes ago"

Sorting:
- Always sort by UTC timestamp
- Ensures consistent ordering globally

Edge Case: Clock Skew
- Client sends client_timestamp
- Server uses server_timestamp for ordering
- Track latency: server_timestamp - client_timestamp
- Alert if latency > 5 seconds (clock skew issue)

Implementation:
{
  "comment_id": "comment_789",
  "created_at": 1715548800125,  // Server UTC
  "client_timestamp": 1715548800000,  // Client time
  "latency_ms": 125
}
```

### Q5: "How would you implement comment search and filtering?"

**Answer:**
```
Search Architecture:

Real-time Comments (Last 1 hour):
- Store in Redis with full-text search (RediSearch module)
- Key: search:{video_id}
- Index: message, user_name
- Query: FT.SEARCH search:video_123 "@message:great"

Historical Comments (>1 hour):
- Index in Elasticsearch
- Async indexing via Kafka
- Full-text search with filters

API:
GET /v1/videos/{video_id}/comments/search?q={query}&filter={filter}

Filters:
- By user: filter=user_id:user_456
- By time range: filter=created_at:[1715548800000 TO 1715552400000]
- By verified users: filter=verified:true

Implementation:
┌──────────────────────────────────────────────────────────────┐
│  Comment Created → Kafka → Elasticsearch Indexer             │
│                                                              │
│  Elasticsearch Index:                                        │
│  {                                                           │
│    "mappings": {                                             │
│      "properties": {                                         │
│        "message": {"type": "text", "analyzer": "standard"}, │
│        "user_name": {"type": "keyword"},                    │
│        "created_at": {"type": "date"}                       │
│      }                                                       │
│    }                                                         │
│  }                                                           │
└──────────────────────────────────────────────────────────────┘
```

### Q6: "How would you handle multi-language support?"

**Answer:**
```
Translation Strategy:

Option 1: Client-Side Translation
- Client detects user's language
- Translate comments using browser API or client library
- Pros: No server load, instant
- Cons: Quality varies, costs on client

Option 2: Server-Side Translation (On-Demand)
- Store comments in original language
- Translate on request: GET /comments/{id}?translate=es
- Cache translations in Redis
- Use Google Translate API or AWS Translate
- Pros: Better quality, cached
- Cons: API costs, latency

Option 3: Pre-Translation (Popular Languages)
- Translate to top 10 languages async
- Store translations in database
- Serve pre-translated version
- Pros: Fast, no API calls
- Cons: Storage cost, not all languages

Recommendation: Hybrid
- Pre-translate to top 5 languages (EN, ES, FR, DE, PT)
- On-demand translation for others
- Cache all translations in Redis (24 hour TTL)

Data Model:
{
  "comment_id": "comment_789",
  "message": "Great stream!",
  "language": "en",
  "translations": {
    "es": "¡Gran transmisión!",
    "fr": "Super diffusion!",
    "de": "Tolle Übertragung!"
  }
}
```

---

## KEY TAKEAWAYS FOR INTERVIEW

### What Makes This Design Strong:

```
1. Technology Choice (SSE over WebSockets)
   ✓ Perfect for read-heavy workload (1:10,909 ratio)
   ✓ Simpler implementation and debugging
   ✓ Built-in reconnection support
   ✓ Standard HTTP (works everywhere)

2. Scalability Strategy
   ✓ Channel sharding (1000 channels)
   ✓ Consistent hashing for viewer co-location
   ✓ Dynamic subscription management
   ✓ Horizontal scaling of all components

3. Tiered Approach for Different Scales
   ✓ Normal videos: Full SSE real-time
   ✓ Popular videos: SSE with sampling
   ✓ Mega-streams: CDN snapshot polling
   ✓ Automatic tier switching

4. Cost Optimization
   ✓ CDN for mega-streams (90% cost savings)
   ✓ Redis caching reduces database load
   ✓ Sampling reduces bandwidth
   ✓ Smart routing minimizes wasted messages

5. User Experience
   ✓ <200ms latency for normal videos
   ✓ Optimistic updates (instant feedback)
   ✓ Graceful reconnection with catch-up
   ✓ Smooth animations even in CDN mode
```

### Common Mistakes to Avoid:

```
❌ Using WebSockets for read-heavy workload
❌ Not considering mega-stream scenarios
❌ Subscribing to all channels (doesn't scale)
❌ Polling database for real-time updates
❌ Not handling reconnections gracefully
❌ Ignoring cost implications at scale
❌ Forgetting about "read your own write" consistency
❌ Not using CDN for static/cacheable content
```

### How to Present This Design:

```
1. Start Simple (5 min)
   - Basic architecture: Client → Server → Database
   - Explain why polling doesn't work
   - Introduce SSE for push-based updates

2. Add Complexity Gradually (10 min)
   - Horizontal scaling with pub/sub
   - Channel sharding and consistent hashing
   - Redis caching for hot data
   - Explain coordination problem and solution

3. Deep Dive on Request (15 min)
   - Detailed data flows
   - SSE vs WebSocket comparison
   - Mega-stream CDN strategy
   - Cost analysis and optimizations

4. Show Trade-offs (5 min)
   - Latency vs cost
   - Consistency vs availability
   - Complexity vs scalability
   - Different strategies for different scales
```

### Key Metrics to Discuss:

```
Performance Metrics:
- Comment creation latency: <100ms (P95)
- Comment delivery latency: <200ms (P95)
- SSE connection establishment: <500ms
- Historical load time: <500ms for 50 comments

Scale Metrics:
- Concurrent viewers: 1B
- Concurrent videos: 1M
- Comments per second: 11M (global)
- Messages per second: 120B (broadcast)

Cost Metrics:
- Normal video: $5/hour
- Popular video: $50/hour
- Mega-stream (SSE): $7B/month (unsustainable)
- Mega-stream (CDN): $110M/month (90% savings)

Reliability Metrics:
- System availability: 99.9%
- Comment loss rate: <0.01%
- Reconnection success rate: >99%
- Cache hit rate: >95%
```

---

## COMPLETE CODE IMPLEMENTATION

### Comment Management Service (Python/FastAPI)

```python
from fastapi import FastAPI, HTTPException, Header, Depends
from pydantic import BaseModel
import boto3
import redis
import json
import time
import uuid
from typing import Optional

app = FastAPI()

# Clients
dynamodb = boto3.resource('dynamodb')
comments_table = dynamodb.Table('comments')
redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

class CommentCreate(BaseModel):
    message: str
    client_timestamp: int

class Comment(BaseModel):
    comment_id: str
    video_id: str
    user_id: str
    user_name: str
    user_avatar: str
    message: str
    created_at: int
    server_timestamp: int

@app.post("/v1/videos/{video_id}/comments", response_model=Comment)
async def create_comment(
    video_id: str,
    comment: CommentCreate,
    authorization: str = Header(...),
    x_request_id: Optional[str] = Header(None)
):
    # Extract user from JWT (simplified)
    user_id = extract_user_from_token(authorization)
    
    # Check rate limit
    rate_limit_key = f"ratelimit:{user_id}:{video_id}"
    request_count = redis_client.incr(rate_limit_key)
    if request_count == 1:
        redis_client.expire(rate_limit_key, 60)
    if request_count > 10:
        raise HTTPException(status_code=429, detail="Rate limit exceeded")
    
    # Validate message
    if not comment.message or len(comment.message) > 500:
        raise HTTPException(status_code=400, detail="Invalid message")
    
    # Check for duplicate (idempotency)
    if x_request_id:
        dedup_key = f"dedup:{x_request_id}"
        if redis_client.exists(dedup_key):
            # Return cached response
            cached = redis_client.get(dedup_key)
            return json.loads(cached)
    
    # Generate comment ID
    comment_id = str(uuid.uuid4())
    server_timestamp = int(time.time() * 1000)
    
    # Get user data (from cache or user service)
    user_data = get_user_data(user_id)
    
    # Create comment object
    comment_obj = {
        "comment_id": comment_id,
        "video_id": video_id,
        "user_id": user_id,
        "user_name": user_data['name'],
        "user_avatar": user_data['avatar'],
        "message": comment.message,
        "created_at": server_timestamp,
        "created_at_video_id": f"{server_timestamp}#{video_id}",
        "status": "ACTIVE"
    }
    
    # Persist to DynamoDB
    comments_table.put_item(Item=comment_obj)
    
    # Update Redis cache
    redis_pipeline = redis_client.pipeline()
    
    # Add to comment stream
    redis_pipeline.lpush(
        f"stream:{video_id}:comments",
        json.dumps(comment_obj)
    )
    redis_pipeline.ltrim(f"stream:{video_id}:comments", 0, 999)
    
    # Update video metadata
    redis_pipeline.hincrby(f"video:{video_id}", "comment_count", 1)
    redis_pipeline.hset(f"video:{video_id}", "last_comment_at", server_timestamp)
    
    # Update comment rate (sliding window)
    redis_pipeline.zadd(
        f"video:{video_id}:comment_times",
        {comment_id: server_timestamp}
    )
    redis_pipeline.zremrangebyscore(
        f"video:{video_id}:comment_times",
        0,
        server_timestamp - 60000
    )
    
    redis_pipeline.execute()
    
    # Calculate comment rate
    comment_rate = redis_client.zcard(f"video:{video_id}:comment_times")
    
    # Determine if should broadcast (sampling for high-rate videos)
    should_broadcast = should_broadcast_comment(comment_rate)
    
    if should_broadcast:
        # Publish to Redis Pub/Sub
        channel_id = hash(video_id) % 1000
        channel = f"comments:video:{channel_id}"
        redis_client.publish(channel, json.dumps(comment_obj))
    
    # Cache response for idempotency
    if x_request_id:
        redis_client.setex(
            f"dedup:{x_request_id}",
            300,  # 5 minutes
            json.dumps(comment_obj)
        )
    
    # Return response
    return Comment(
        **comment_obj,
        server_timestamp=server_timestamp
    )

def should_broadcast_comment(comment_rate: int) -> bool:
    """Determine if comment should be broadcast based on rate"""
    if comment_rate < 100:
        return True  # 100% - show all
    elif comment_rate < 500:
        return random.random() < 0.5  # 50% sampling
    elif comment_rate < 1000:
        return random.random() < 0.2  # 20% sampling
    elif comment_rate < 5000:
        return random.random() < 0.05  # 5% sampling
    else:
        return random.random() < 0.01  # 1% sampling

@app.get("/v1/videos/{video_id}/comments")
async def get_comments(
    video_id: str,
    cursor: Optional[str] = None,
    limit: int = 50,
    authorization: str = Header(...)
):
    # Try Redis cache first
    if not cursor:
        # Get recent comments from cache
        comments_json = redis_client.lrange(
            f"stream:{video_id}:comments",
            0,
            limit - 1
        )
        
        if comments_json:
            comments = [json.loads(c) for c in comments_json]
            
            next_cursor = None
            if len(comments) == limit:
                last_comment = comments[-1]
                next_cursor = f"{last_comment['created_at']}#{last_comment['comment_id']}"
            
            return {
                "comments": comments,
                "pagination": {
                    "next_cursor": next_cursor,
                    "has_more": len(comments) == limit
                }
            }
    
    # Cache miss or pagination - query DynamoDB
    query_params = {
        "IndexName": "video_id-created_at-index",
        "KeyConditionExpression": "video_id = :vid",
        "ExpressionAttributeValues": {
            ":vid": video_id
        },
        "ScanIndexForward": False,
        "Limit": limit
    }
    
    if cursor:
        query_params["ExclusiveStartKey"] = {
            "video_id": video_id,
            "created_at_video_id": cursor
        }
    
    response = comments_table.query(**query_params)
    comments = response.get('Items', [])
    
    next_cursor = None
    if 'LastEvaluatedKey' in response:
        last_key = response['LastEvaluatedKey']
        next_cursor = last_key['created_at_video_id']
    
    return {
        "comments": comments,
        "pagination": {
            "next_cursor": next_cursor,
            "has_more": 'LastEvaluatedKey' in response
        }
    }
```

---

## CONCLUSION

This Facebook Live Comments system demonstrates a production-ready architecture that:

**Handles Massive Scale:**
- 1B concurrent viewers across 1M live videos
- 11M comments/second creation rate
- 120B messages/second broadcast load
- Scales horizontally across all components

**Optimizes for Different Scenarios:**
- Normal videos (<10K viewers): Full SSE real-time with <200ms latency
- Popular videos (10K-100K viewers): SSE with smart sampling
- Mega-streams (>100K viewers): CDN snapshot polling with 90% cost savings

**Provides Excellent User Experience:**
- Sub-200ms latency for real-time comments
- Optimistic updates for instant feedback
- Graceful reconnection with automatic catch-up
- Smooth animations even in high-velocity streams

**Maintains Reliability:**
- 99.9% system availability
- At-least-once delivery guarantee
- Automatic failover and recovery
- No comment loss under normal conditions

**Key Architectural Insights:**
1. **SSE over WebSockets** for read-heavy workloads (1:10,909 ratio)
2. **Channel sharding + consistent hashing** for efficient server coordination
3. **Tiered approach** adapting strategy based on scale
4. **CDN for mega-streams** leveraging existing infrastructure
5. **Redis for hot data** reducing database load by 95%

The design showcases staff-level thinking by recognizing that requirements fundamentally change at extreme scale, and having the judgment to apply different solutions at different tiers rather than forcing a one-size-fits-all approach.

