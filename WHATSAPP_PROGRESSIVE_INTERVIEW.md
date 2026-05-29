# WhatsApp/Messenger System Design - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Understanding the Problem (5 min)
Phase 2: Requirements & Estimation (5 min)
Phase 3: Core Entities & API Design (5 min)
Phase 4: High-Level Design - Iterative Approach (10 min)
Phase 5: Data Models (5 min)
Phase 6: Deep Dives (15 min)
  - Scaling to billions
  - Real-time communication
  - Offline delivery
  - Media handling
  - Multi-device support
```

---

## PHASE 1: UNDERSTANDING THE PROBLEM (5 minutes)

### What to Say:

"WhatsApp is a messaging service that allows users to send encrypted messages in real-time. Before I start designing, let me clarify what we're building and what's in scope."

### Clarifying Questions:

**Q1: Core Features**
- "Should I focus on 1-to-1 messaging, group chats, or both?"
- "What's the maximum group size we need to support?"

**Expected Answer:** Both, limit groups to 100 participants

**Q2: Message Types**
- "Are we supporting just text, or also media (images, videos)?"
- "Do we need voice/video calls?"

**Expected Answer:** Text + media, calls are out of scope

**Q3: Offline Behavior**
- "Should messages be delivered when users come back online?"
- "How long should we store undelivered messages?"

**Expected Answer:** Yes, up to 30 days

**Q4: Message Status**
- "Do we need delivery receipts (sent, delivered, read)?"
- "Should we show online/last seen status?"

**Expected Answer:** Yes to both

**Q5: Scale**
- "How many users are we targeting?"
- "What's the expected message volume?"

**Expected Answer:** 2B MAU, 1B DAU, billions of messages daily

---

## PHASE 2: REQUIREMENTS & ESTIMATION (5 minutes)

### Functional Requirements:

```
✅ In Scope:
1. Users can start group chats (up to 100 participants)
2. Users can send/receive text messages in real-time
3. Users receive messages sent while offline (up to 30 days)
4. Users can send/receive media (images, videos)
5. Message status indicators (sent, delivered, read)
6. User presence (online/last seen)

❌ Out of Scope:
- Voice/Video calling
- Business interactions
- Registration/profile management
- End-to-end encryption details
```

### Non-Functional Requirements:

```
1. Low latency: < 500ms message delivery
2. High availability: 99.99% uptime
3. Guaranteed delivery: No message loss
4. Scalability: Billions of users, high throughput
5. Minimal server storage: Delete messages after delivery
6. Resilience: Handle component failures gracefully
```

### Back-of-Envelope Estimation:

```
Given:
- 1B Daily Active Users (DAU)
- Average 50 messages/user/day
- Average message size: 200 bytes

Traffic:
- Total messages/day: 1B × 50 = 50B messages/day
- Write QPS (average): 50B / 86,400 ≈ 580K QPS
- Write QPS (peak): 580K × 2.5 ≈ 1.5M QPS

Storage (Messages):
- Daily: 50B × 200 bytes = 10 TB/day
- With 30-day retention: 300 TB

Connections:
- Peak concurrent users: ~200M (20% of DAU)
- Need to handle 200M simultaneous connections

Presence (Hidden Load!):
- Heartbeat every 30 seconds
- Presence QPS: 200M / 30 ≈ 6.7M QPS
- This is 10x message traffic!
```

---

## PHASE 3: CORE ENTITIES & API DESIGN (5 min)

### Core Entities:

```
1. User
   - user_id, phone_number, name, profile_picture

2. Chat
   - chat_id, participants[], name, created_at
   - Note: 1-to-1 is just a chat with 2 participants

3. Message
   - message_id, chat_id, sender_id, content, 
     attachments[], timestamp, status

4. Client/Device
   - client_id, user_id, device_type, last_seen
   - Users can have multiple devices
```

### API Design - WebSocket Commands:

**Why WebSocket?** (We'll explore this in detail later)

```
Client → Server Commands:

1. createChat
   { participants: [], name: "" }
   → { chatId: "" }

2. sendMessage
   { chatId: "", message: "", attachments: [] }
   → "SUCCESS" | "FAILURE"

3. modifyChatParticipants
   { chatId: "", userId: "", operation: "ADD|REMOVE" }
   → "SUCCESS" | "FAILURE"

Server → Client Commands:

1. chatUpdate
   { chatId: "", participants: [] }
   → "RECEIVED"

2. newMessage
   { chatId: "", userId: "", message: "", attachments: [] }
   → "RECEIVED"

3. statusUpdate
   { messageId: "", status: "delivered|read" }
   → "RECEIVED"
```

---

## PHASE 4: HIGH-LEVEL DESIGN - ITERATIVE APPROACH (10 min)

### What to Say:

"Let me build this system incrementally, starting simple and addressing issues as they arise. I'll start with a single-server solution, then scale it out."

### Iteration 1: Single Server (Naive Approach)

**Goal:** Enable users to create chats and send messages

```
┌──────────┐
│  Client  │
└────┬─────┘
     │ WebSocket
     ▼
┌─────────────────┐
│  Chat Server    │
│  - In-memory    │
│    user→socket  │
│    map          │
└────┬────────────┘
     │
     ▼
┌─────────────────┐
│   DynamoDB      │
│  - Chats        │
│  - Participants │
└─────────────────┘
```

**How it works:**
1. Users connect via WebSocket
2. Server stores user_id → socket mapping in memory
3. To send message:
   - Look up chat participants from DB
   - Find socket for each participant
   - Send message via WebSocket

**Problems:**
❌ Single point of failure
❌ Can't scale beyond one server's connection limit (~50K)
❌ No offline message delivery
❌ Messages lost if server crashes

**What to Say:** "This works for a small scale, but we have obvious problems. Let me address them one by one."

---

### Iteration 2: Add Message Persistence

**Goal:** Handle offline users and prevent message loss

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     ▼
┌─────────────────┐
│  Chat Server    │
└────┬────────────┘
     │
     ▼
┌─────────────────────────────────┐
│         DynamoDB                 │
│  ┌──────────┐  ┌──────────┐    │
│  │ Messages │  │  Inbox   │    │
│  │          │  │ (per user)│    │
│  └──────────┘  └──────────┘    │
└─────────────────────────────────┘
```

**New Flow:**
1. User A sends message to User B
2. Server writes to:
   - Messages table (durable storage)
   - Inbox table for User B (undelivered queue)
3. If User B is online:
   - Deliver via WebSocket
   - On ACK: Delete from Inbox
4. If User B is offline:
   - Message stays in Inbox
   - Delivered when B connects

**Why this works:**
✅ Messages persist even if server crashes
✅ Offline delivery supported
✅ No message loss

**Remaining Problems:**
❌ Still single server (can't scale)
❌ What if we need multiple servers?

---

### Iteration 3: Multiple Servers - The Routing Problem

**Goal:** Scale beyond single server

**Naive Approach: Just add load balancer?**

```
┌──────────┐  ┌──────────┐
│ Client A │  │ Client B │
└────┬─────┘  └────┬─────┘
     │             │
     └──────┬──────┘
            ▼
    ┌──────────────┐
    │Load Balancer │
    └──────┬───────┘
           │
     ┌─────┴─────┐
     ▼           ▼
┌─────────┐ ┌─────────┐
│Server 1 │ │Server 2 │
└─────────┘ └─────────┘
```

**Problem:**
- User A connects to Server 1
- User B connects to Server 2
- When A sends message to B:
  - Server 1 receives it
  - But B's socket is on Server 2!
  - ❌ Can't deliver!

**What to Say:** "We have a routing problem. Server 1 doesn't know B is on Server 2. Let me explore solutions."

---

### Approach 1: Kafka Topics Per User (❌ Doesn't Work)

**Idea:** Create Kafka topic for each user

```
Server 1: Publishes to "user-B-topic"
Server 2: Subscribes to "user-B-topic"
         Delivers to B's socket
```

**Why it fails:**
- Kafka isn't designed for billions of topics
- Each topic has ~50KB overhead
- 1B users × 50KB = 50TB just for topic metadata!
- ❌ Not scalable

---

### Approach 2: Consistent Hashing (⚠️ Complex)

**Idea:** Assign users to specific servers based on user_id hash

```
hash(user_id) % num_servers = assigned_server

User A always connects to Server 1
User B always connects to Server 2
```

**How it works:**
- Keep registry (ZooKeeper/Etcd) of server assignments
- When message arrives, route to correct server
- Servers connect directly to each other

**Challenges:**
- Each server needs connections to all other servers (N²)
- Adding/removing servers requires careful rebalancing
- Complex failure handling
- ⚠️ Works but complicated

---

### Approach 3: Redis Pub/Sub (✅ Best Solution)

**Idea:** Use lightweight message broker between servers

```
┌──────────┐  ┌──────────┐
│ Client A │  │ Client B │
└────┬─────┘  └────┬─────┘
     │             │
     ▼             ▼
┌─────────┐   ┌─────────┐
│Server 1 │   │Server 2 │
└────┬────┘   └────┬────┘
     │             │
     └──────┬──────┘
            ▼
    ┌──────────────┐
    │ Redis Pub/Sub│
    │ Channels per │
    │    user_id   │
    └──────────────┘
```

**How it works:**
1. When User B connects to Server 2:
   - Server 2 subscribes to channel "user-B"
2. When User A sends message to B:
   - Server 1 publishes to channel "user-B"
   - Redis routes to all subscribers (Server 2)
   - Server 2 delivers via WebSocket

**Why this is best:**
✅ Lightweight: ~200 bytes per channel (vs 50KB for Kafka)
✅ 1B channels = 200GB memory (manageable!)
✅ Simple: No complex routing logic
✅ Fast: Single-digit millisecond latency
✅ At-most-once delivery (acceptable with Inbox fallback)

**Trade-off:**
- Pub/Sub doesn't guarantee delivery
- But we already have Inbox for durability!
- Pub/Sub is for real-time, Inbox is for reliability

---

### Final Architecture (Iteration 3 Complete):

```
┌─────────────────────────────────────────────────────────────┐
│                      CLIENTS                                 │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │Load Balancer │
                  │   (Layer 4)  │
                  └──────┬───────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    ┌─────────┐    ┌─────────┐    ┌─────────┐
    │Server 1 │    │Server 2 │    │Server N │
    │         │    │         │    │         │
    │ 50K     │    │ 50K     │    │ 50K     │
    │ conns   │    │ conns   │    │ conns   │
    └────┬────┘    └────┬────┘    └────┬────┘
         │              │              │
         └──────────────┼──────────────┘
                        ▼
                ┌──────────────┐
                │ Redis Pub/Sub│
                │              │
                │ Channels:    │
                │ - user-123   │
                │ - user-456   │
                │ - ...        │
                └──────────────┘
                        │
                        ▼
                ┌──────────────┐
                │   DynamoDB   │
                │              │
                │ - Messages   │
                │ - Inbox      │
                │ - Chats      │
                └──────────────┘
```

**Complete Message Flow:**

```
1. User A (on Server 1) sends message to User B

2. Server 1:
   a) Writes to Messages table (durability)
   b) Writes to Inbox table for User B
   c) Returns SUCCESS to User A
   d) Publishes to Redis channel "user-B"

3. Server 2 (subscribed to "user-B"):
   a) Receives message from Redis
   b) Delivers via WebSocket to User B
   
4. User B:
   a) Receives message
   b) Sends ACK to Server 2
   
5. Server 2:
   a) Deletes from Inbox table
   b) Publishes status update to "user-A"
   
6. Server 1:
   a) Receives status update
   b) Shows "delivered" to User A
```

**Why This Works:**
✅ Scales horizontally (add more servers)
✅ Real-time delivery via Pub/Sub
✅ Reliable delivery via Inbox
✅ No message loss (written to DB first)
✅ Handles offline users
✅ Simple routing (Redis handles it)

---

## PHASE 5: DATA MODELS (5 minutes)

### What to Say:

"Now let me define the data models. I'll use DynamoDB for scalability and Cassandra-style partitioning for messages."

### 1. Users Table (DynamoDB)

```
Table: users
Primary Key: user_id

Attributes:
- user_id (UUID)
- phone_number (unique)
- name
- profile_picture_url
- created_at

GSI: phone_number (for lookup during registration)
```

### 2. Chats Table (DynamoDB)

```
Table: chats
Primary Key: chat_id

Attributes:
- chat_id (UUID)
- name (optional, for group chats)
- created_at
- updated_at
```

### 3. Chat Participants Table (DynamoDB)

```
Table: chat_participants
Primary Key: chat_id (Partition Key) + user_id (Sort Key)

Attributes:
- chat_id
- user_id
- role (admin/member)
- joined_at

GSI: user_id (Partition Key) + chat_id (Sort Key)
Purpose: Query all chats for a user
```

**Access Patterns:**
- Get all participants for chat: Query by chat_id
- Get all chats for user: Query GSI by user_id

### 4. Messages Table (DynamoDB)

```
Table: messages
Primary Key: chat_id (Partition Key) + message_id (Sort Key)

Attributes:
- chat_id
- message_id (TIMEUUID - time-based)
- sender_id
- content (encrypted)
- attachments[] (URLs)
- created_at

TTL: 30 days (auto-delete old messages)
```

**Why this design:**
- Partition by chat_id: All messages for conversation together
- Sort by message_id (time-based): Chronological order
- Fast queries: "Get last N messages for chat"

### 5. Inbox Table (DynamoDB)

```
Table: inbox
Primary Key: user_id (Partition Key) + message_id (Sort Key)

Attributes:
- user_id
- message_id
- chat_id
- created_at

TTL: 30 days
```

**Purpose:** Track undelivered messages per user

**Flow:**
- Message sent → Write to Inbox
- Message delivered + ACK → Delete from Inbox
- User reconnects → Query Inbox, deliver all

### 6. Clients Table (DynamoDB)

```
Table: clients
Primary Key: user_id (Partition Key) + client_id (Sort Key)

Attributes:
- user_id
- client_id (device identifier)
- device_type (iOS/Android/Web)
- last_seen
- created_at

Limit: 3 clients per user
```

**Purpose:** Support multiple devices per user

---

## PHASE 6: DEEP DIVES (15 minutes)

### Deep Dive 1: Real-Time Communication - Why WebSockets?

**What to Say:** "Let me explain why we need WebSockets by exploring alternatives."

#### Option 1: HTTP Polling (❌ Terrible)

```
Client repeatedly asks: "Any new messages?"

while (true) {
  response = GET /messages/new
  if (response.hasMessages) {
    display(response.messages)
  }
  sleep(1 second)
}
```

**Problems:**
- 200M users × 1 request/second = 200M QPS
- 99% of requests return empty (wasteful!)
- High latency (up to 1 second delay)
- Battery drain on mobile
- ❌ Not acceptable

#### Option 2: Long Polling (⚠️ Better but not great)

```
Client: GET /messages/new
Server: Holds connection open until message arrives
        (or timeout after 30 seconds)
Server: Returns message
Client: Immediately reconnects
```

**Problems:**
- Still many connections (reconnect overhead)
- Timeout management complex
- Only server→client (need separate path for client→server)
- ⚠️ Works but inefficient

#### Option 3: Server-Sent Events (SSE) (⚠️ One-way only)

```
Client: Opens SSE connection
Server: Pushes events as they arrive
```

**Problems:**
- Only server→client (one-way)
- Need separate HTTP POST for client→server
- ⚠️ Works but not ideal for chat

#### Option 4: WebSockets (✅ Perfect for Chat)

```
Client: Opens WebSocket connection
Both: Can send messages anytime (bidirectional)
```

**Why WebSockets win:**
✅ Bidirectional: Both sides send/receive
✅ Persistent: Single connection, no reconnect overhead
✅ Low latency: <10ms per message
✅ Efficient: No HTTP headers on every message
✅ Real-time: Messages pushed immediately

**WebSocket Lifecycle:**

```
1. Handshake (HTTP Upgrade):
   Client: GET /connect HTTP/1.1
           Upgrade: websocket
   Server: HTTP/1.1 101 Switching Protocols

2. Persistent Connection:
   - Full-duplex communication
   - Low overhead frames
   - Both sides send anytime

3. Heartbeat (Keep-Alive):
   Every 30 seconds:
   Client → Server: PING
   Server → Client: PONG
   
   If no PONG: Connection dead, reconnect

4. Close:
   Either side: CLOSE frame
```

---

### Deep Dive 2: Handling Offline Users

**Scenario:** User B is offline when A sends message

**Solution: Inbox + Push Notifications**

```
1. User A sends message
   ↓
2. Server writes to:
   - Messages table (permanent)
   - Inbox table for User B (temporary)
   ↓
3. Server tries Pub/Sub delivery
   - No subscriber (B is offline)
   - Message not delivered
   ↓
4. Server sends Push Notification
   - FCM (Android) or APNS (iOS)
   - "You have a new message from A"
   ↓
5. User B opens app
   ↓
6. Server queries Inbox for User B
   SELECT * FROM inbox 
   WHERE user_id = 'user-B'
   ORDER BY created_at
   ↓
7. Server delivers all pending messages
   ↓
8. User B sends ACK for each
   ↓
9. Server deletes from Inbox
```

**Why this works:**
✅ No message loss (written to DB)
✅ User notified via push
✅ All messages delivered on reconnect
✅ Inbox cleaned up after delivery

---

### Deep Dive 3: Group Messages & Fan-Out

**Challenge:** Sending to 100 participants is expensive

**Naive Approach:**
```
For each of 100 participants:
  - Write to their Inbox
  - Publish to their Pub/Sub channel
  - Send push notification if offline

Result: 100 DB writes, 100 Pub/Sub publishes
```

**Optimization 1: Batch by Server**

```
1. Look up all 100 participants
2. Group by which server they're connected to:
   - Server 1: 30 users
   - Server 2: 45 users
   - Offline: 25 users
3. Send one message per server (not per user)
4. Each server fans out locally
```

**Optimization 2: Single Message Storage**

```
Instead of:
  - 100 copies in Inbox table

Do:
  - 1 copy in Messages table
  - 100 pointers in Inbox table (just message_id)
```

**Result:**
- 1 message write (not 100)
- 100 small Inbox entries (just IDs)
- Significant storage savings

---

### Deep Dive 4: Media Files (Images/Videos)

**What to Say:** "Media files are 1000x larger than text. We need a different approach."

#### Approach 1: Store in Database (❌ Bad)

```
User → Server → DynamoDB (store blob)
```

**Problems:**
- DynamoDB not optimized for large blobs
- Expensive ($0.25 per GB)
- Slow retrieval
- Wastes server bandwidth
- ❌ Don't do this

#### Approach 2: Server Proxies to S3 (⚠️ Better)

```
User → Server → S3
Recipient → Server → S3 → Recipient
```

**Problems:**
- Server still handles all bytes
- Wastes server bandwidth
- Server becomes bottleneck
- ⚠️ Works but inefficient

#### Approach 3: Pre-Signed URLs (✅ Best)

```
Upload:
1. Client: "I want to upload image"
2. Server: Generates pre-signed S3 URL
3. Client: Uploads directly to S3
4. Client: Sends message with S3 URL

Download:
1. Server: Sends message with S3 URL
2. Client: Downloads directly from S3
```

**Why this is best:**
✅ Server doesn't handle media bytes
✅ Direct client↔S3 transfer (fast)
✅ S3 handles scale automatically
✅ Can add CloudFront CDN easily
✅ Secure (pre-signed URLs expire)

**Additional Optimizations:**
- Generate thumbnails (Lambda on upload)
- Progressive loading (low-res → high-res)
- Compression (reduce size by 70%)
- TTL on S3 (delete after 30 days)

---

### Deep Dive 5: Multi-Device Support

**Challenge:** User has phone + laptop + tablet

**Problem with Single Inbox:**
```
- Phone receives message
- Phone sends ACK
- Inbox entry deleted
- Laptop never gets message!
❌ Broken
```

**Solution: Client-Specific Inbox**

```
Old Schema:
inbox: user_id + message_id

New Schema:
inbox: user_id + client_id + message_id
```

**Flow:**
```
1. User has 3 devices:
   - Phone (client-1)
   - Laptop (client-2)
   - Tablet (client-3)

2. Message arrives:
   - Write to inbox for client-1
   - Write to inbox for client-2
   - Write to inbox for client-3

3. Phone receives + ACKs:
   - Delete from inbox for client-1
   - Keep for client-2 and client-3

4. Laptop comes online:
   - Query inbox for client-2
   - Deliver pending messages
```

**Pub/Sub doesn't change:**
- Still subscribe to channel "user-B"
- All connected devices receive via Pub/Sub
- Inbox is backup for offline devices

---

### Deep Dive 6: Handling Connection Failures

**Problem:** WebSocket silently dies (poor network)

#### Detection Strategy 1: Wait for TCP Timeout (❌ Too Slow)

```
TCP keepalive: 2+ minutes
User thinks they're connected but aren't!
❌ Bad UX
```

#### Detection Strategy 2: Application-Level Heartbeat (✅ Best)

```
Every 10-30 seconds:
Client → Server: PING
Server → Client: PONG (within 5 seconds)

If no PONG:
- Client: Assumes connection dead, reconnects
- Server: Closes socket after 3 missed pings
```

**Recovery Flow:**
```
1. Connection dies
2. Detected within 15 seconds (heartbeat + timeout)
3. Client reconnects with exponential backoff
4. Client syncs from Inbox:
   "Give me messages since message_id X"
5. Server delivers missed messages
6. Client back in sync
```

**Why this works:**
✅ Fast detection (15 seconds vs 2 minutes)
✅ Automatic recovery
✅ No message loss (Inbox backup)
✅ Good UX (user sees reconnecting state)

---

### Deep Dive 7: Message Ordering

**Problem:** Messages arrive out of order

**Why it happens:**
- Network delays vary
- Different servers process at different speeds
- Pub/Sub doesn't guarantee order

**Solution: Timestamp + Client-Side Sorting**

```
1. Server timestamps each message on receipt
   - All servers sync time via NTP
   - Timestamp: 1735689600.123

2. Client receives messages
   - May arrive out of order
   - Client sorts by timestamp before displaying

3. Occasional "pop-in"
   - Message arrives late
   - Appears "above" newer message
   - Users find this acceptable!
```

**Why we don't guarantee strict ordering:**
- Would require delays (wait for late messages)
- Complex coordination between servers
- Users prefer speed over perfect order
- WhatsApp/Messenger work this way

---

### Deep Dive 8: Last Seen / Online Status

**Challenge:** Tracking online status for 200M users

#### Approach 1: Update DB on Every Heartbeat (❌ Terrible)

```
Every 30 seconds × 200M users = 6.7M writes/second
Just for presence!
❌ Expensive and wasteful
```

#### Approach 2: Update Only on Connect/Disconnect (✅ Better)

```
Table: last_seen
- user_id
- last_disconnect_timestamp

On disconnect:
  UPDATE last_seen 
  SET timestamp = NOW()
  WHERE user_id = ?

To check if online:
  1. Query last_seen table
  2. Publish to Pub/Sub channel "user-X"
  3. If user online, they respond "ONLINE"
  4. If no response, show last_seen timestamp
```

**Why this works:**
✅ Minimal writes (only on disconnect)
✅ Online users respond in real-time
✅ Offline users show last seen
✅ Scalable

---

## SUMMARY: EVALUATION CRITERIA

```
┌──────────────────────────┬─────────────────────────┬──────────────┐
│ Evaluation Element       │ Evidence in Solution    │ Rating       │
├──────────────────────────┼─────────────────────────┼──────────────┤
│ Workable Solution        │ Complete, handles scale │ Outstanding  │
│ Covers Edge Cases        │ Offline, multi-device,  │ Outstanding  │
│                          │ failures, ordering      │              │
│ Clarifying Questions     │ Asked 8+ questions      │ Outstanding  │
│ Efficiency               │ WebSockets, Pub/Sub,    │ Outstanding  │
│                          │ pre-signed URLs         │              │
│ Simplicity               │ Iterative approach,     │ Outstanding  │
│                          │ clear progression       │              │
│ Quality of Abstraction   │ Layered design, clear   │ Outstanding  │
│                          │ separation of concerns  │              │
└──────────────────────────┴─────────────────────────┴──────────────┘

Overall Rating: Top 20%
```

## KEY INTERVIEW TALKING POINTS

### Opening:
"I'll build this incrementally, starting simple and addressing scalability issues as they arise."

### Critical Insights:
1. "HTTP polling would generate 200M QPS with mostly empty responses - WebSockets are essential"
2. "The routing problem is key - we need Redis Pub/Sub to connect servers"
3. "Pub/Sub is for real-time, Inbox is for reliability - we need both"
4. "Media files bypass our servers entirely using pre-signed URLs"
5. "Multi-device requires client-specific Inbox, not user-specific"

### Trade-offs:
- "Pub/Sub doesn't guarantee delivery, but Inbox provides durability"
- "We accept out-of-order messages for lower latency"
- "Heartbeats add overhead but detect failures in seconds vs minutes"

This approach shows deep thinking by exploring alternatives before arriving at solutions! 🎯
