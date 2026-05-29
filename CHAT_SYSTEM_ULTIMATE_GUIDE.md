# Chat Application System Design - Ultimate Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: Core Entities & API Design (4 min)
Phase 5: High-Level Design - Progressive Approach (10 min)
Phase 6: Data Models (5 min)
Phase 7: Deep Dives (13 min)
  - Real-time communication (WebSockets)
  - Scaling to billions (Redis Pub/Sub)
  - Offline delivery & reliability
  - Media handling
  - Multi-device support
  - Presence & online status
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"I'm designing a chat application like WhatsApp or Facebook Messenger. Before diving in, let me clarify the scope and requirements."

### Questions to Ask:

**Q1: Core Features**
- "Should I focus on 1-to-1 messaging, group chats, or both?"
- "What's the maximum group size?"

**Expected Answer:** Both, max 100-256 participants

**Q2: Message Types**
- "Text only, or also media (images, videos)?"
- "Voice/video calls in scope?"

**Expected Answer:** Text + media, calls out of scope

**Q3: Real-time & Offline**
- "How real-time should delivery be? Sub-second?"
- "Should offline users receive messages when they reconnect?"
- "How long should we store undelivered messages?"

**Expected Answer:** < 500ms delivery, offline support for 30 days

**Q4: Message Status & Presence**
- "Do we need delivery receipts (sent, delivered, read)?"
- "Typing indicators?"
- "Online/last seen status?"

**Expected Answer:** Yes to all

**Q5: Scale**
- "How many users? Daily active users?"
- "Average messages per user per day?"
- "Geographic distribution - global or single region?"

**Expected Answer:** 2B MAU, 1B DAU, 50 messages/user/day, global

**Q6: Multi-device**
- "Can users be logged in on multiple devices simultaneously?"
- "Should messages sync across devices?"

**Expected Answer:** Yes (3 devices max)

**Q7: Consistency vs Availability**
- "For message ordering, strict or eventual consistency?"
- "What's more important: availability or consistency?"

**Expected Answer:** Strong consistency for message order within chat, eventual for presence

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### Functional Requirements:

```
✅ In Scope:
1. 1-to-1 Messaging
   - Send/receive text messages in real-time
   - Message history retrieval

2. Group Messaging  
   - Create groups (max 100 participants)
   - Add/remove members (admin only)
   - Group message history

3. Offline Support
   - Store messages when recipient offline
   - Deliver when recipient comes online (30 days)
   - Push notifications

4. Message Status
   - Sent (to server)
   - Delivered (to recipient device)
   - Read (by recipient)

5. User Presence
   - Online/Offline status
   - Last seen timestamp
   - Typing indicators

6. Media Support
   - Send/receive images, videos
   - Thumbnails for quick preview

7. Multi-device
   - Up to 3 devices per user
   - Message sync across devices

❌ Out of Scope:
- Voice/Video calls
- Stories/Status updates
- E2E encryption details
- Payment features
```

### Non-Functional Requirements:

```
1. Performance
   - Message delivery: < 500ms (when online)
   - Message history load: < 500ms
   - Support 1B concurrent connections

2. Scalability
   - 2B Monthly Active Users (MAU)
   - 1B Daily Active Users (DAU)
   - 50B messages/day
   - 580K average QPS, 1.5M peak QPS

3. Availability
   - 99.99% uptime
   - No single point of failure
   - Graceful degradation

4. Consistency
   - Strong: Message ordering within conversation
   - Eventual: Presence status
   - At-least-once: Message delivery

5. Reliability
   - No message loss (durability)
   - Idempotent delivery
   - Retry mechanisms

6. Security
   - Authentication (phone number)
   - TLS encryption in transit
   - Rate limiting (anti-spam)
```

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me calculate the scale to understand our constraints and validate design decisions."

### Traffic Estimation:

```
Given:
- 1B Daily Active Users (DAU)
- 50 messages per user per day
- Average message: 200 bytes (text + metadata)

1. Message Traffic (QPS)
   Total messages/day: 1B × 50 = 50B messages/day
   
   Average Write QPS: 50B / 86,400 ≈ 580,000 QPS
   Peak Write QPS: 580K × 2.5 ≈ 1.5 Million QPS
   
   Read QPS (history): ~2x writes ≈ 1.2M QPS

2. Presence Traffic (THE HIDDEN BOTTLENECK!)
   Peak concurrent users: 200M (20% of DAU)
   Heartbeat frequency: Every 30 seconds
   
   Presence QPS: 200M / 30 ≈ 6.7 Million QPS
   
   KEY INSIGHT: Presence traffic is 10x message traffic!

3. Connection Management
   Concurrent connections: 200M WebSocket connections
   
   If each server handles 50K connections:
   Required servers: 200M / 50K = 4,000 chat servers
   
   (WhatsApp famously served 1-2M per server with Erlang)

4. Group Message Amplification
   Average group size: 50 members
   Group messages: 10% of total = 5B/day
   
   Actual deliveries: 5B × 50 = 250B deliveries/day
   Additional QPS: 250B / 86,400 ≈ 2.9M QPS
   
   Total Peak QPS: 1.5M + 2.9M = 4.4M QPS
```

### Storage Estimation:

```
1. Message Storage
   Daily: 50B messages × 200 bytes = 10 TB/day
   Monthly: 10 TB × 30 = 300 TB/month
   Yearly: 10 TB × 365 = 3.65 PB/year
   
   With 5-year retention: 18.25 PB
   
   Note: Can use TTL to auto-delete after 30 days

2. User Data
   Users: 2B
   Per user: 1 KB (profile, settings)
   Total: 2B × 1KB = 2 TB

3. Group Metadata
   Groups: 100M (estimate)
   Per group: 5 KB (members, settings)
   Total: 100M × 5KB = 500 GB

4. Presence Data (In-Memory Cache)
   Active users: 200M
   Per user: 100 bytes (status, server_id, timestamp)
   Total: 200M × 100B = 20 GB (Redis)

5. Session Data (In-Memory)
   Active users: 200M
   Per user: 200 bytes (user_id, client_id, server_id)
   Total: 200M × 200B = 40 GB (Redis)

Total Storage: ~20 PB (5 years)
```

### Bandwidth Estimation:

```
1. Incoming (Messages)
   Peak: 4.4M QPS × 200 bytes = 880 MB/sec ≈ 7 Gbps

2. Outgoing (Deliveries + Presence)
   Messages: 4.4M QPS × 200 bytes = 880 MB/sec
   Presence: 6.7M QPS × 50 bytes = 335 MB/sec
   Total: 1.2 GB/sec ≈ 10 Gbps

3. Media (If included)
   10% of messages are media
   Average media: 500 KB
   Daily: 5B × 500KB = 2.5 PB/day
   Bandwidth: 2.5 PB / 86,400 ≈ 29 GB/sec ≈ 232 Gbps
   
   Media dominates bandwidth!
```

### Summary Table:

```
┌─────────────────────────┬──────────────────┐
│ Metric                  │ Value            │
├─────────────────────────┼──────────────────┤
│ DAU                     │ 1B               │
│ Messages/day            │ 50B              │
│ Message QPS (peak)      │ 4.4M             │
│ Presence QPS            │ 6.7M (10x msgs!) │
│ Concurrent connections  │ 200M             │
│ Chat servers needed     │ 4,000            │
│ Storage (5 years)       │ 20 PB            │
│ Bandwidth (no media)    │ 10 Gbps          │
│ Bandwidth (with media)  │ 232 Gbps         │
└─────────────────────────┴──────────────────┘

Key Insights:
1. Presence is the real bottleneck (6.7M QPS)
2. Connection management is critical (200M connections)
3. Media dominates bandwidth (23x more than text)
4. Need specialized architecture for scale
```

---

## PHASE 4: CORE ENTITIES & API DESIGN (4 minutes)

### Core Entities:

```
1. User
   - user_id (UUID)
   - phone_number (unique)
   - name
   - profile_picture_url
   - created_at

2. Chat
   - chat_id (UUID)
   - participants[] (user_ids)
   - name (optional, for groups)
   - created_at
   
   Note: 1-to-1 is just a chat with 2 participants

3. Message
   - message_id (TIMEUUID - time-based)
   - chat_id
   - sender_id
   - content (encrypted)
   - attachments[] (URLs)
   - timestamp
   - status (sent/delivered/read)

4. Client/Device
   - client_id (UUID)
   - user_id
   - device_type (iOS/Android/Web)
   - last_seen
   
   Users can have multiple devices (max 3)
```

### API Design - WebSocket Commands:

**Why WebSocket?** We'll explore alternatives in Deep Dive

```
═══════════════════════════════════════════════════
CLIENT → SERVER COMMANDS
═══════════════════════════════════════════════════

1. createChat
   Request:
   {
     "type": "createChat",
     "participants": ["user-123", "user-456"],
     "name": "Family Group"  // optional
   }
   
   Response:
   {
     "type": "chatCreated",
     "chatId": "chat-789"
   }

2. sendMessage
   Request:
   {
     "type": "sendMessage",
     "tempId": "client-uuid-1",  // for idempotency
     "chatId": "chat-789",
     "content": "Hello!",
     "attachments": ["https://s3.../image.jpg"]
   }
   
   Response:
   {
     "type": "messageAck",
     "tempId": "client-uuid-1",
     "messageId": "msg-server-uuid-99",
     "status": "sent"
   }

3. modifyChatParticipants
   Request:
   {
     "type": "modifyChat",
     "chatId": "chat-789",
     "userId": "user-999",
     "operation": "ADD" | "REMOVE"
   }
   
   Response:
   {
     "type": "success" | "failure"
   }

4. getAttachmentUploadUrl
   Request:
   {
     "type": "getUploadUrl",
     "fileType": "image/jpeg",
     "fileSize": 524288
   }
   
   Response:
   {
     "type": "uploadUrl",
     "url": "https://s3.../presigned-url",
     "expiresIn": 3600
   }

═══════════════════════════════════════════════════
SERVER → CLIENT COMMANDS
═══════════════════════════════════════════════════

1. chatUpdate
   {
     "type": "chatUpdate",
     "chatId": "chat-789",
     "participants": ["user-123", "user-456", "user-999"]
   }
   
   Client Response: { "type": "ack", "messageId": "..." }

2. newMessage
   {
     "type": "newMessage",
     "messageId": "msg-99",
     "chatId": "chat-789",
     "senderId": "user-123",
     "content": "Hello!",
     "attachments": [],
     "timestamp": 1735689600000
   }
   
   Client Response: { "type": "ack", "messageId": "msg-99" }

3. statusUpdate
   {
     "type": "statusUpdate",
     "messageId": "msg-99",
     "status": "delivered" | "read",
     "userId": "user-456"
   }

4. typingIndicator
   {
     "type": "typing",
     "chatId": "chat-789",
     "userId": "user-123",
     "isTyping": true
   }

5. presenceUpdate
   {
     "type": "presence",
     "userId": "user-123",
     "status": "online" | "offline",
     "lastSeen": 1735689500
   }

═══════════════════════════════════════════════════
HEARTBEAT (Keep-Alive)
═══════════════════════════════════════════════════

Client → Server (every 30 seconds):
{
  "type": "ping",
  "timestamp": 1735689600000
}

Server → Client:
{
  "type": "pong",
  "timestamp": 1735689600050
}
```

---

## PHASE 5: HIGH-LEVEL DESIGN - PROGRESSIVE APPROACH (10 min)

### What to Say:

"I'll build this incrementally, starting simple and addressing scalability issues as they arise."

### Iteration 1: Single Server (Naive)

**Goal:** Basic messaging functionality

```
┌──────────┐
│  Client  │
└────┬─────┘
     │ WebSocket
     ▼
┌─────────────────────┐
│   Chat Server       │
│                     │
│  In-Memory Map:     │
│  user_id → socket   │
└────┬────────────────┘
     │
     ▼
┌─────────────────────┐
│     DynamoDB        │
│  - Chats            │
│  - ChatParticipants │
└─────────────────────┘
```

**Flow:**
1. Users connect via WebSocket
2. Server stores: user_id → socket in memory
3. To send message:
   - Look up chat participants from DB
   - Find socket for each participant
   - Send via WebSocket

**Problems:**
❌ Single point of failure
❌ Can't scale beyond ~50K connections
❌ No offline delivery
❌ Messages lost if server crashes

---

### Iteration 2: Add Persistence (Offline Support)

**Goal:** Handle offline users, prevent message loss

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     ▼
┌─────────────────────┐
│   Chat Server       │
└────┬────────────────┘
     │
     ▼
┌──────────────────────────────────┐
│          DynamoDB                 │
│  ┌──────────┐  ┌──────────┐     │
│  │ Messages │  │  Inbox   │     │
│  │ (durable)│  │(per user)│     │
│  └──────────┘  └──────────┘     │
└──────────────────────────────────┘
```

**New Flow:**
1. User A sends message to User B
2. Server writes to:
   - **Messages table** (permanent storage)
   - **Inbox table** for User B (undelivered queue)
3. If User B online:
   - Deliver via WebSocket
   - On ACK: Delete from Inbox
4. If User B offline:
   - Message stays in Inbox
   - Send push notification
   - Deliver when B reconnects

**Why this works:**
✅ Messages persist (no loss)
✅ Offline delivery supported
✅ Durable even if server crashes

**Remaining Problem:**
❌ Still single server (doesn't scale)

---

### Iteration 3: Multiple Servers - The Routing Problem

**Naive: Just add load balancer?**

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

**THE PROBLEM:**
- User A connects to Server 1
- User B connects to Server 2
- When A sends message to B:
  - Server 1 receives it
  - But B's socket is on Server 2!
  - ❌ Can't deliver!

**What to Say:** "We have a routing problem. Let me explore solutions."

---

### Solution Exploration: How to Route Between Servers?

#### Approach 1: Kafka Topics Per User (❌ Fails)

**Idea:** Create Kafka topic for each user

```
Server 1: Publishes to topic "user-B"
Server 2: Subscribes to topic "user-B"
         Delivers to B's socket
```

**Why it fails:**
- Kafka not designed for billions of topics
- Each topic: ~50KB overhead
- 1B users × 50KB = **50TB just for metadata!**
- ❌ Not scalable

---

#### Approach 2: Consistent Hashing (⚠️ Complex)

**Idea:** Assign users to specific servers

```
hash(user_id) % num_servers = assigned_server

User A always → Server 1
User B always → Server 2
```

**How it works:**
- Registry (ZooKeeper/Etcd) tracks assignments
- Route messages to correct server
- Servers connect directly (gRPC/TCP)

**Challenges:**
- N² connections between servers
- Complex rebalancing when adding/removing servers
- Thundering herd on reconnection
- ⚠️ Works but complicated

---

#### Approach 3: Redis Pub/Sub (✅ Best)

**Idea:** Lightweight message broker

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
    │              │
    │ Channels:    │
    │ - user-123   │
    │ - user-456   │
    └──────────────┘
```

**How it works:**
1. User B connects to Server 2
   - Server 2 subscribes to channel "user-B"
2. User A sends message to B
   - Server 1 publishes to channel "user-B"
   - Redis routes to subscribers (Server 2)
   - Server 2 delivers via WebSocket

**Why this is best:**
✅ Lightweight: ~200 bytes per channel
✅ 1B channels = 200GB memory (manageable!)
✅ Simple: No complex routing logic
✅ Fast: Single-digit ms latency
✅ Scalable: Redis handles millions of ops/sec

**Comparison:**
```
┌──────────────┬──────────┬──────────────┐
│ Approach     │ Overhead │ Scalability  │
├──────────────┼──────────┼──────────────┤
│ Kafka        │ 50TB     │ ❌ No        │
│ Consistent   │ Low      │ ⚠️  Complex  │
│ Redis Pub/Sub│ 200GB    │ ✅ Yes       │
└──────────────┴──────────┴──────────────┘
```

**Trade-off:**
- Pub/Sub is "at-most-once" (no delivery guarantee)
- But we have Inbox for durability!
- Pub/Sub = real-time, Inbox = reliability

