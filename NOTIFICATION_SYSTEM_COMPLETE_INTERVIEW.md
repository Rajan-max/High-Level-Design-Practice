# Scalable Notification System - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: High-Level Architecture (8 min)
Phase 5: API Design (4 min)
Phase 6: Data Models (5 min)
Phase 7: Core Algorithm - Message Processing & Retry Logic (6 min)
Phase 8: Deep Dive - Reliability & Fault Tolerance (4 min)
Phase 9: Follow-up Questions (5 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"Thank you for the problem. I want to design a scalable notification system that can handle multiple channels like email, SMS, and push notifications. Let me clarify the requirements and scope."

### Questions to Ask:

**Q1: Notification Channels**
- "Which channels should we support: email, SMS, push notifications, in-app notifications?"
- "Should we support rich media notifications (images, videos)?"

**Expected Answer:** Email, SMS, push notifications are core. Rich media is nice-to-have.

**Q2: Message Types - THE KEY QUESTION**
- "What types of notifications: transactional (OTP, receipts), promotional (marketing), or system alerts?"
- "Do different types have different priority levels?"

**Expected Answer:** All three types, with transactional being highest priority

**Q3: Delivery Guarantees**
- "What delivery guarantees do we need: at-least-once, exactly-once, or best-effort?"
- "Is it acceptable if users receive duplicate notifications occasionally?"

**Expected Answer:** At-least-once delivery, exactly-once user experience (idempotency)

**Q4: User Preferences**
- "Should users be able to opt-out of certain notification types?"
- "Do we need to respect quiet hours or frequency limits?"

**Expected Answer:** Yes, comprehensive preference management needed

**Q5: Template System**
- "Do we need dynamic templates with personalization?"
- "Should templates support multiple languages?"

**Expected Answer:** Yes dynamic templates, multi-language is follow-up

**Q6: Third-party Providers**
- "Should we integrate with external providers like SendGrid, Twilio, FCM?"
- "Do we need failover between multiple providers?"

**Expected Answer:** Yes external providers, failover is critical for reliability

**Q7: Scale & Performance**
- "What's the expected volume: notifications per day, peak QPS?"
- "What's acceptable latency for delivery?"

**Expected Answer:** 100M notifications/day, 10K peak QPS, <5 seconds for critical messages

**Q8: Analytics & Tracking**
- "Do we need delivery tracking, open rates, click rates?"
- "Should we provide analytics dashboard?"

**Expected Answer:** Basic delivery tracking needed, analytics dashboard is follow-up

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### What to Say:

"Based on our discussion, let me summarize the requirements in priority order."

### Functional Requirements (Priority Order):

```
1. Multi-Channel Delivery (CORE)
   - Email notifications (transactional + promotional)
   - SMS notifications (OTP, alerts)
   - Push notifications (mobile apps)
   - Support for rich content (HTML emails)

2. Message Processing (CORE)
   - Accept notification requests via API
   - Validate message content and recipients
   - Route to appropriate channel
   - Handle message queuing and batching

3. Template Management (CORE)
   - Create and manage message templates
   - Support dynamic content with variables
   - Template versioning and A/B testing
   - Multi-language support (nice-to-have)

4. User Preference Management (IMPORTANT)
   - Opt-in/opt-out for different message types
   - Channel preferences (email vs SMS)
   - Quiet hours and frequency limits
   - Global unsubscribe functionality

5. Delivery Tracking (IMPORTANT)
   - Track message status (sent, delivered, failed)
   - Delivery confirmations via webhooks
   - Retry failed messages with exponential backoff
   - Dead letter queue for permanent failures

6. Provider Management (CRITICAL)
   - Integration with multiple providers per channel
   - Automatic failover on provider failures
   - Rate limiting per provider
   - Cost optimization (cheapest provider first)

7. Scheduling & Batching (IMPORTANT)
   - Schedule messages for future delivery
   - Batch processing for bulk campaigns
   - Time zone aware delivery
   - Recurring notifications

Out of Scope:
- Real-time chat/messaging
- Advanced analytics dashboard
- Machine learning for send-time optimization
- Complex workflow automation
```

### Non-Functional Requirements:

```
1. Performance
   - API response time: < 100ms (p99)
   - Message delivery: < 5 seconds for high priority
   - Throughput: 10K notifications/second (peak)
   - Batch processing: 1M messages/hour

2. Scalability
   - 100M notifications/day
   - 50M active users
   - Horizontal scaling of all components
   - Auto-scaling based on queue depth

3. Reliability
   - 99.9% uptime for API
   - At-least-once delivery guarantee
   - Exactly-once user experience (idempotency)
   - Automatic retry with exponential backoff
   - Circuit breaker for provider failures

4. Consistency
   - Strong consistency for user preferences
   - Eventual consistency for delivery status
   - Idempotent API operations
   - Duplicate detection and prevention

5. Security
   - Encrypted data in transit (TLS)
   - Encrypted sensitive data at rest
   - API authentication and rate limiting
   - PII data protection and retention policies

6. Observability
   - Comprehensive logging and metrics
   - Real-time monitoring and alerting
   - Delivery success/failure rates
   - Provider performance tracking
```

### Why This Matters:
✓ Shows understanding of notification complexity
✓ Prioritizes reliability over features
✓ Identifies key technical challenges
✓ Sets clear performance expectations

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me calculate the scale to validate our design decisions and identify bottlenecks."

### Traffic Estimation:

```
Given:
- 100M notifications/day
- 50M active users
- Peak hours: 9-11 AM, 6-8 PM (4 hours/day)
- Channel distribution: 60% email, 30% push, 10% SMS

Calculations:

1. Average & Peak QPS
   Average: 100M / 86,400 = ~1,160 notifications/second
   
   Peak traffic: 40% of daily volume in 4 peak hours
   Peak: (100M × 0.4) / (4 × 3,600) = 2,778 notifications/second
   
   With safety margin: ~10,000 notifications/second (peak)

2. Channel Breakdown (Daily)
   Email: 100M × 0.6 = 60M emails/day
   Push: 100M × 0.3 = 30M push notifications/day
   SMS: 100M × 0.1 = 10M SMS/day

3. Channel Breakdown (Peak QPS)
   Email: 10K × 0.6 = 6,000 emails/second
   Push: 10K × 0.3 = 3,000 push/second
   SMS: 10K × 0.1 = 1,000 SMS/second

4. Provider Rate Limits (Bottleneck Analysis)
   SendGrid: 10,000 emails/hour = 2.8 emails/second per API key
   Twilio: 1,000 SMS/hour = 0.28 SMS/second per account
   FCM: 1M push/minute = 16,667 push/second
   
   Critical insight: Need 2,143 SendGrid API keys for peak email!
   Critical insight: Need 3,571 Twilio accounts for peak SMS!
```

### Storage Estimation:

```
1. Notification Records
   - 100M notifications/day
   - Retention: 90 days
   - Total records: 100M × 90 = 9B notifications
   - Record size: 2KB (metadata, content, status)
   - Storage: 9B × 2KB = 18TB

2. User Preferences
   - 50M users
   - Record size: 1KB (preferences, settings)
   - Storage: 50M × 1KB = 50GB

3. Templates
   - 10,000 templates
   - Average size: 10KB (HTML, variables)
   - Storage: 10K × 10KB = 100MB

4. Delivery Logs (Time-series)
   - 100M delivery events/day
   - Retention: 30 days
   - Total events: 100M × 30 = 3B events
   - Event size: 500 bytes (status, timestamp, provider)
   - Storage: 3B × 500B = 1.5TB

5. Dead Letter Queue
   - 1% failure rate = 1M failed messages/day
   - Retention: 7 days
   - Storage: 1M × 7 × 2KB = 14GB

Total Storage: ~20TB
```

### Bandwidth Estimation:

```
1. API Ingress (Notification Requests)
   - 10K requests/second × 2KB = 20MB/second
   - Daily: 20MB × 86,400 = 1.7TB/day

2. Provider API Calls (Egress)
   - Email: 6K/sec × 50KB (HTML) = 300MB/second
   - SMS: 1K/sec × 1KB = 1MB/second
   - Push: 3K/sec × 2KB = 6MB/second
   - Total: ~307MB/second = 26TB/day

3. Webhook Callbacks (Ingress)
   - 80% delivery success rate
   - 100M × 0.8 = 80M callbacks/day
   - 80M / 86,400 = 926 callbacks/second
   - 926 × 1KB = 926KB/second = 80GB/day

Total Bandwidth: ~28TB/day (mostly provider API calls)
```

### Cost Estimation:

```
1. Provider Costs (Major expense)
   Email (SendGrid): 60M × $0.0006 = $36,000/month
   SMS (Twilio): 10M × $0.0075 = $75,000/month
   Push (FCM): 30M × $0.00001 = $300/month
   Total: $111,300/month

2. Infrastructure (AWS)
   API Servers: 20 × m5.large = $2,880/month
   Worker Servers: 50 × c5.xlarge = $15,000/month
   Database: RDS PostgreSQL Multi-AZ = $8,000/month
   Cache: ElastiCache Redis = $3,000/month
   Message Queue: Kafka/SQS = $2,000/month
   Storage: S3 + EBS = $1,000/month
   Total: $31,880/month

3. Total Monthly Cost: ~$143,000/month
   Provider costs: 77%
   Infrastructure: 23%
```

### Summary Table:

```
┌─────────────────────────┬──────────────────┐
│ Metric                  │ Value            │
├─────────────────────────┼──────────────────┤
│ Daily notifications     │ 100M             │
│ Peak QPS                │ 10K              │
│ Active users            │ 50M              │
│ Storage required        │ 20TB             │
│ Daily bandwidth         │ 28TB             │
│ Monthly cost            │ $143K            │
│ Provider API keys needed│ 2,143 (SendGrid) │
│ Twilio accounts needed  │ 3,571            │
└─────────────────────────┴──────────────────┘

Key Insights:
1. Provider rate limits are the real bottleneck
2. Need massive provider account management
3. Provider costs dominate (77% of total cost)
4. SMS is most expensive per message
5. Storage is manageable (20TB)
```

### Why This Matters:
✓ Identifies provider rate limits as critical constraint
✓ Shows cost awareness (provider costs dominate)
✓ Validates need for multi-provider architecture
✓ Demonstrates scale understanding

---

## PHASE 4: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"I'll design an event-driven, asynchronous architecture that decouples message acceptance from delivery. The key insight is using priority queues and multiple provider accounts to handle the scale."

### Complete Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENT LAYER                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Web Apps    │  │  Mobile Apps │  │  Backend     │         │
│  │              │  │              │  │  Services    │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              API GATEWAY / LOAD BALANCER                         │
│  - Rate limiting (1000 req/sec per API key)                     │
│  - Authentication & authorization                               │
│  - Request validation                                           │
│  - SSL termination                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  NOTIFICATION API SERVICE                        │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Core Responsibilities:                                │    │
│  │  - Accept notification requests                        │    │
│  │  - Validate payload & recipients                       │    │
│  │  - Check user preferences                              │    │
│  │  - Idempotency handling                                │    │
│  │  - Route to appropriate priority queue                 │    │
│  │  - Return immediate response (< 100ms)                 │    │
│  │                                                        │    │
│  │  Technology: Java/Spring Boot                          │    │
│  │  Instances: 20 (auto-scaling)                          │    │
│  │  Database: PostgreSQL (user prefs, templates)          │    │
│  │  Cache: Redis (hot data, idempotency)                  │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────┬───────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┬───────────────┐
          ▼               ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MESSAGE QUEUE LAYER                           │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  HIGH PRIORITY QUEUE (Kafka Topic)                      │  │
│  │  - Transactional messages (OTP, receipts, alerts)       │  │
│  │  - SLA: < 5 seconds delivery                            │  │
│  │  - Dedicated workers: 30                                │  │
│  │  - Batch size: 1 (immediate processing)                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  MEDIUM PRIORITY QUEUE (Kafka Topic)                    │  │
│  │  - System notifications (password reset, updates)       │  │
│  │  - SLA: < 30 seconds delivery                           │  │
│  │  - Dedicated workers: 20                                │  │
│  │  - Batch size: 10                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  LOW PRIORITY QUEUE (Kafka Topic)                       │  │
│  │  - Marketing campaigns, newsletters                     │  │
│  │  - SLA: < 5 minutes delivery                            │  │
│  │  - Dedicated workers: 10                                │  │
│  │  - Batch size: 100 (bulk processing)                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  RETRY QUEUE (Kafka Topic)                              │  │
│  │  - Failed messages with exponential backoff             │  │
│  │  - Delayed processing based on retry count              │  │
│  │  - Max retries: 5                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Technology: Apache Kafka                                        │
│  Configuration: 3 brokers, replication factor 3                  │
│  Partitioning: By user_id for ordering                           │
└─────────────────────────┬───────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┬───────────────┐
          ▼               ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     WORKER SERVICE LAYER                         │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  EMAIL WORKERS (Channel-Specific)                      │    │
│  │                                                        │    │
│  │  Responsibilities:                                     │    │
│  │  - Poll high/medium/low priority queues               │    │
│  │  - Render email templates with user data              │    │
│  │  - Rate limiting per provider                         │    │
│  │  - Provider failover logic                            │    │
│  │  - Retry with exponential backoff                     │    │
│  │  - Update delivery status                             │    │
│  │                                                        │    │
│  │  Technology: Go (high performance)                     │    │
│  │  Instances: 30 (auto-scaling based on queue depth)    │    │
│  │  Providers: SendGrid, AWS SES, Mailgun                │    │
│  │  Rate Limit: 2,143 API keys × 2.8 req/sec = 6K/sec   │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  SMS WORKERS (Channel-Specific)                        │    │
│  │                                                        │    │
│  │  Responsibilities:                                     │    │
│  │  - Process SMS notifications                          │    │
│  │  - Handle carrier-specific formatting                 │    │
│  │  - Manage Twilio account rotation                     │    │
│  │  - Handle delivery receipts                           │    │
│  │                                                        │    │
│  │  Technology: Go                                        │    │
│  │  Instances: 20                                         │    │
│  │  Providers: Twilio, AWS SNS, MessageBird              │    │
│  │  Rate Limit: 3,571 accounts × 0.28 req/sec = 1K/sec  │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  PUSH WORKERS (Channel-Specific)                       │    │
│  │                                                        │    │
│  │  Responsibilities:                                     │    │
│  │  - Send push notifications                            │    │
│  │  - Handle device token management                     │    │
│  │  - Platform-specific formatting (iOS/Android)        │    │
│  │  - Batch processing for efficiency                    │    │
│  │                                                        │    │
│  │  Technology: Go                                        │    │
│  │  Instances: 15                                         │    │
│  │  Providers: FCM, APNS, Web Push                       │    │
│  │  Rate Limit: No significant limits                    │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                   EXTERNAL PROVIDER LAYER                        │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  EMAIL PROVIDERS                                         │  │
│  │                                                          │  │
│  │  Primary: SendGrid (2,143 API keys)                     │  │
│  │  - Cost: $0.0006 per email                              │  │
│  │  - Rate: 2.8 emails/sec per key                         │  │
│  │  - Reliability: 99.9%                                   │  │
│  │                                                          │  │
│  │  Secondary: AWS SES (500 accounts)                      │  │
│  │  - Cost: $0.0001 per email                              │  │
│  │  - Rate: 14 emails/sec per account                      │  │
│  │  - Reliability: 99.95%                                  │  │
│  │                                                          │  │
│  │  Tertiary: Mailgun (200 accounts)                       │  │
│  │  - Cost: $0.0008 per email                              │  │
│  │  - Rate: 10 emails/sec per account                      │  │
│  │  - Reliability: 99.8%                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  SMS PROVIDERS                                           │  │
│  │                                                          │  │
│  │  Primary: Twilio (3,571 accounts)                       │  │
│  │  - Cost: $0.0075 per SMS                                │  │
│  │  - Rate: 0.28 SMS/sec per account                       │  │
│  │  - Global coverage: 180+ countries                      │  │
│  │                                                          │  │
│  │  Secondary: AWS SNS (1,000 accounts)                    │  │
│  │  - Cost: $0.0065 per SMS                                │  │
│  │  - Rate: 1 SMS/sec per account                          │  │
│  │  - Good for US/EU                                       │  │
│  │                                                          │  │
│  │  Tertiary: MessageBird (500 accounts)                   │  │
│  │  - Cost: $0.0080 per SMS                                │  │
│  │  - Rate: 0.5 SMS/sec per account                        │  │
│  │  - Strong in Europe/Asia                                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  PUSH PROVIDERS                                          │  │
│  │                                                          │  │
│  │  FCM (Firebase Cloud Messaging)                         │  │
│  │  - Android push notifications                           │  │
│  │  - Rate: 1M messages/minute                             │  │
│  │  - Cost: $0.00001 per message                           │  │
│  │                                                          │  │
│  │  APNS (Apple Push Notification Service)                 │  │
│  │  - iOS push notifications                               │  │
│  │  - Rate: No published limits                            │  │
│  │  - Cost: Free                                           │  │
│  │                                                          │  │
│  │  Web Push (Browser notifications)                       │  │
│  │  - Chrome, Firefox, Safari                              │  │
│  │  - Rate: No significant limits                          │  │
│  │  - Cost: Free                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                  │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  PRIMARY DATABASE (PostgreSQL)                         │    │
│  │                                                        │    │
│  │  Tables:                                               │    │
│  │  - notifications (18TB - main records)                │    │
│  │  - users (50GB - user data)                           │    │
│  │  - user_preferences (50GB - opt-in/out settings)      │    │
│  │  - templates (100MB - message templates)              │    │
│  │  - providers (1MB - provider configurations)          │    │
│  │                                                        │    │
│  │  Configuration:                                        │    │
│  │  - Multi-AZ deployment                                │    │
│  │  - 5 read replicas                                    │    │
│  │  - Connection pooling (PgBouncer)                     │    │
│  │  - Partitioning by date (monthly)                     │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  CACHE LAYER (Redis Cluster)                           │    │
│  │                                                        │    │
│  │  Use Cases:                                            │    │
│  │  - User preferences (hot data)                        │    │
│  │  - Template cache                                     │    │
│  │  - Idempotency keys (24 hour TTL)                     │    │
│  │  - Rate limiting counters                             │    │
│  │  - Provider health status                             │    │
│  │                                                        │    │
│  │  Configuration:                                        │    │
│  │  - 6 shards, 2 replicas each                          │    │
│  │  - Total: 12 nodes                                    │    │
│  │  - Memory: 64GB per node                              │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  TIME-SERIES DATABASE (InfluxDB)                       │    │
│  │                                                        │    │
│  │  Metrics:                                              │    │
│  │  - Delivery success/failure rates                     │    │
│  │  - Provider performance metrics                       │    │
│  │  - Queue depth over time                              │    │
│  │  - API response times                                 │    │
│  │                                                        │    │
│  │  Retention: 90 days                                    │    │
│  │  Storage: 1.5TB                                       │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  OBJECT STORAGE (S3)                                   │    │
│  │                                                        │    │
│  │  Contents:                                             │    │
│  │  - Email templates (HTML, images)                     │    │
│  │  - Attachment files                                   │    │
│  │  - Delivery logs (long-term archive)                  │    │
│  │  - Dead letter queue messages                         │    │
│  │                                                        │    │
│  │  Storage: 500GB                                       │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

### Data Flow: Complete Message Journey

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: MESSAGE ACCEPTANCE                                     │
└─────────────────────────────────────────────────────────────────┘

1. Client sends notification request
   POST /api/v1/notifications
   {
     "user_id": "user-123",
     "template_id": "welcome-email",
     "channel": "email",
     "priority": "high",
     "variables": {"name": "John", "code": "123456"}
   }
   ↓
2. API Gateway validates request
   - Check API key and rate limits
   - Validate JSON schema
   ↓
3. Notification API Service processes
   - Check idempotency key (Redis)
   - Validate user exists
   - Check user preferences (Redis cache)
   - Render template with variables
   ↓
4. Route to appropriate queue
   - High priority → high_priority_notifications
   - Medium priority → medium_priority_notifications
   - Low priority → low_priority_notifications
   ↓
5. Return immediate response
   {
     "notification_id": "notif-abc123",
     "status": "queued",
     "estimated_delivery": "2024-01-15T10:00:05Z"
   }

Latency: < 100ms

┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2: MESSAGE PROCESSING                                     │
└─────────────────────────────────────────────────────────────────┘

1. Email Worker polls Kafka queue
   - Consume from high_priority_notifications
   - Batch size: 1 (immediate processing)
   ↓
2. Worker processes message
   - Validate recipient email format
   - Check user preferences again (cache)
   - Select provider (SendGrid primary)
   ↓
3. Rate limiting check
   - Check current rate for selected API key
   - If limit exceeded, select different API key
   - If all keys exhausted, select secondary provider
   ↓
4. Send via provider
   POST https://api.sendgrid.com/v3/mail/send
   {
     "personalizations": [{
       "to": [{"email": "user@example.com"}],
       "subject": "Welcome John!"
     }],
     "content": [{"type": "text/html", "value": "..."}]
   }
   ↓
5. Handle response
   - Success: Update status to "sent"
   - Failure: Add to retry queue with backoff
   ↓
6. Update notification status
   - Database: notifications.status = "sent"
   - Cache: Update delivery metrics

Latency: 1-5 seconds

┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3: DELIVERY CONFIRMATION                                  │
└─────────────────────────────────────────────────────────────────┘

1. Provider sends webhook
   POST /webhooks/sendgrid/delivery
   {
     "event": "delivered",
     "email": "user@example.com",
     "timestamp": 1642248000,
     "sg_message_id": "abc123"
   }
   ↓
2. Webhook handler processes
   - Validate webhook signature
   - Find notification by provider message ID
   - Update status to "delivered"
   ↓
3. Optional: Notify client
   - If client provided callback URL
   - Send delivery confirmation

Total delivery time: 5-30 seconds
```

---

### Why This Architecture Works:

**1. Asynchronous Decoupling**
✅ API accepts requests instantly (< 100ms)
✅ Background workers handle actual delivery
✅ System remains responsive under load

**2. Priority-Based Processing**
✅ Critical messages (OTP) processed first
✅ Marketing campaigns don't block transactional
✅ Different SLAs for different message types

**3. Multi-Provider Resilience**
✅ Automatic failover on provider failures
✅ Rate limit distribution across providers
✅ Cost optimization (cheapest provider first)

**4. Horizontal Scalability**
✅ Stateless workers (easy to scale)
✅ Kafka partitioning for parallel processing
✅ Auto-scaling based on queue depth

**5. Reliability & Fault Tolerance**
✅ Retry with exponential backoff
✅ Dead letter queue for permanent failures
✅ Circuit breaker for provider health

---

## PHASE 5: API DESIGN (4 minutes)

### What to Say:

"Let me define the key API endpoints for sending notifications, managing templates, and tracking delivery status."

### Core APIs:

```yaml
1. Send Single Notification
POST /api/v1/notifications

Request:
{
  "user_id": "user-123",
  "template_id": "welcome-email",
  "channel": "email",
  "priority": "high",
  "variables": {
    "user_name": "John Doe",
    "verification_code": "123456",
    "expiry_time": "10 minutes"
  },
  "scheduled_at": "2024-01-15T10:00:00Z",
  "idempotency_key": "unique-key-123"
}

Response: 201 Created
{
  "notification_id": "notif-abc123",
  "status": "queued",
  "estimated_delivery": "2024-01-15T10:00:05Z",
  "channel": "email",
  "priority": "high"
}

2. Send Bulk Notifications
POST /api/v1/notifications/bulk

Request:
{
  "template_id": "promo-campaign",
  "channel": "email",
  "priority": "low",
  "recipients": [
    {
      "user_id": "user-123",
      "variables": {"discount": "20%"}
    },
    {
      "user_id": "user-456", 
      "variables": {"discount": "15%"}
    }
  ],
  "scheduled_at": "2024-01-15T09:00:00Z"
}

Response: 202 Accepted
{
  "batch_id": "batch-xyz789",
  "total_recipients": 2,
  "estimated_completion": "2024-01-15T09:05:00Z"
}

3. Get Notification Status
GET /api/v1/notifications/{notification_id}

Response: 200 OK
{
  "notification_id": "notif-abc123",
  "status": "delivered",
  "channel": "email",
  "priority": "high",
  "recipient": "user@example.com",
  "sent_at": "2024-01-15T10:00:03Z",
  "delivered_at": "2024-01-15T10:00:07Z",
  "provider": "sendgrid",
  "attempts": 1,
  "error_message": null
}

4. Get Bulk Status
GET /api/v1/notifications/bulk/{batch_id}

Response: 200 OK
{
  "batch_id": "batch-xyz789",
  "status": "completed",
  "total_recipients": 1000,
  "successful": 987,
  "failed": 13,
  "started_at": "2024-01-15T09:00:00Z",
  "completed_at": "2024-01-15T09:04:32Z",
  "failure_details": [
    {
      "user_id": "user-999",
      "error": "Invalid email address"
    }
  ]
}

5. Cancel Scheduled Notification
DELETE /api/v1/notifications/{notification_id}

Response: 200 OK
{
  "notification_id": "notif-abc123",
  "status": "cancelled",
  "cancelled_at": "2024-01-15T09:30:00Z"
}
```

### Template Management APIs:

```yaml
1. Create Template
POST /api/v1/templates

Request:
{
  "name": "Welcome Email",
  "channel": "email",
  "subject": "Welcome {{user_name}}!",
  "body": "<h1>Hello {{user_name}}</h1><p>Your verification code is: {{verification_code}}</p>",
  "variables": [
    {"name": "user_name", "type": "string", "required": true},
    {"name": "verification_code", "type": "string", "required": true}
  ]
}

Response: 201 Created
{
  "template_id": "tmpl-welcome-001",
  "name": "Welcome Email",
  "version": 1,
  "created_at": "2024-01-15T08:00:00Z"
}

2. Get Template
GET /api/v1/templates/{template_id}

Response: 200 OK
{
  "template_id": "tmpl-welcome-001",
  "name": "Welcome Email",
  "channel": "email",
  "subject": "Welcome {{user_name}}!",
  "body": "<h1>Hello {{user_name}}</h1>...",
  "variables": [...],
  "version": 1,
  "is_active": true
}

3. Update Template
PUT /api/v1/templates/{template_id}

Request:
{
  "subject": "Welcome {{user_name}} to our platform!",
  "body": "<h1>Hello {{user_name}}</h1><p>Welcome to our amazing platform!</p>"
}

Response: 200 OK
{
  "template_id": "tmpl-welcome-001",
  "version": 2,
  "updated_at": "2024-01-15T08:30:00Z"
}
```

### User Preference APIs:

```yaml
1. Get User Preferences
GET /api/v1/users/{user_id}/preferences

Response: 200 OK
{
  "user_id": "user-123",
  "email_enabled": true,
  "sms_enabled": false,
  "push_enabled": true,
  "marketing_enabled": false,
  "frequency_limit": 10,
  "quiet_hours": {
    "start": "22:00",
    "end": "08:00",
    "timezone": "America/New_York"
  },
  "channels": {
    "email": {
      "transactional": true,
      "promotional": false,
      "system": true
    },
    "sms": {
      "transactional": true,
      "promotional": false,
      "system": false
    }
  }
}

2. Update User Preferences
PUT /api/v1/users/{user_id}/preferences

Request:
{
  "email_enabled": true,
  "marketing_enabled": false,
  "quiet_hours": {
    "start": "23:00",
    "end": "07:00",
    "timezone": "America/Los_Angeles"
  }
}

Response: 200 OK
{
  "user_id": "user-123",
  "updated_at": "2024-01-15T09:00:00Z"
}

3. Global Unsubscribe
POST /api/v1/users/{user_id}/unsubscribe

Request:
{
  "unsubscribe_token": "secure-token-123",
  "reason": "Too many emails"
}

Response: 200 OK
{
  "user_id": "user-123",
  "unsubscribed_at": "2024-01-15T09:00:00Z",
  "status": "unsubscribed"
}
```

### Webhook APIs:

```yaml
1. Provider Delivery Webhook (SendGrid)
POST /webhooks/sendgrid/events

Request:
[
  {
    "event": "delivered",
    "email": "user@example.com",
    "timestamp": 1642248000,
    "sg_message_id": "abc123",
    "sg_event_id": "event123"
  },
  {
    "event": "bounce",
    "email": "invalid@example.com", 
    "timestamp": 1642248060,
    "reason": "550 Invalid recipient",
    "sg_message_id": "def456"
  }
]

Response: 200 OK

2. Client Delivery Callback
POST {client_callback_url}

Request:
{
  "notification_id": "notif-abc123",
  "status": "delivered",
  "delivered_at": "2024-01-15T10:00:07Z",
  "channel": "email",
  "recipient": "user@example.com"
}
```

---

## PHASE 6: DATA MODELS (5 minutes)

### What to Say:

"I'll design the database schema to support high-volume notifications with efficient querying and proper indexing."

### 1. Notifications Table (Core)

```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Request info
    user_id VARCHAR(255) NOT NULL,
    template_id VARCHAR(255),
    channel VARCHAR(20) NOT NULL, -- email, sms, push
    priority VARCHAR(10) NOT NULL, -- high, medium, low
    
    -- Content
    subject VARCHAR(500),
    body TEXT NOT NULL,
    variables JSONB,
    
    -- Recipient
    recipient VARCHAR(255) NOT NULL, -- email/phone/device_token
    
    -- Status tracking
    status VARCHAR(20) NOT NULL DEFAULT 'queued',
    -- queued, processing, sent, delivered, failed, cancelled
    
    -- Provider info
    provider VARCHAR(50),
    provider_message_id VARCHAR(255),
    
    -- Timing
    created_at TIMESTAMP DEFAULT NOW(),
    scheduled_at TIMESTAMP,
    sent_at TIMESTAMP,
    delivered_at TIMESTAMP,
    
    -- Retry logic
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    next_retry_at TIMESTAMP,
    
    -- Error handling
    error_message TEXT,
    error_code VARCHAR(50),
    
    -- Idempotency
    idempotency_key VARCHAR(255) UNIQUE,
    
    -- Metadata
    metadata JSONB,
    
    CONSTRAINT chk_status CHECK (status IN (
        'queued', 'processing', 'sent', 'delivered', 'failed', 'cancelled'
    )),
    CONSTRAINT chk_channel CHECK (channel IN ('email', 'sms', 'push')),
    CONSTRAINT chk_priority CHECK (priority IN ('high', 'medium', 'low'))
);

-- Indexes for performance
CREATE INDEX idx_notifications_user_id ON notifications(user_id, created_at DESC);
CREATE INDEX idx_notifications_status ON notifications(status, created_at);
CREATE INDEX idx_notifications_scheduled ON notifications(scheduled_at) 
    WHERE scheduled_at IS NOT NULL;
CREATE INDEX idx_notifications_retry ON notifications(next_retry_at) 
    WHERE next_retry_at IS NOT NULL;
CREATE INDEX idx_notifications_provider_msg ON notifications(provider_message_id);

-- Partitioning by month for large scale
CREATE TABLE notifications_2024_01 PARTITION OF notifications
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

### 2. Users Table

```sql
CREATE TABLE users (
    id VARCHAR(255) PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(20),
    name VARCHAR(255),
    timezone VARCHAR(50) DEFAULT 'UTC',
    language VARCHAR(10) DEFAULT 'en',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
```

### 3. User Preferences Table

```sql
CREATE TABLE user_preferences (
    user_id VARCHAR(255) PRIMARY KEY REFERENCES users(id),
    
    -- Global settings
    email_enabled BOOLEAN DEFAULT true,
    sms_enabled BOOLEAN DEFAULT true,
    push_enabled BOOLEAN DEFAULT true,
    
    -- Marketing preferences
    marketing_enabled BOOLEAN DEFAULT false,
    
    -- Frequency controls
    frequency_limit INTEGER DEFAULT 10, -- per hour
    
    -- Quiet hours
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    timezone VARCHAR(50) DEFAULT 'UTC',
    
    -- Channel-specific preferences
    channel_preferences JSONB DEFAULT '{}',
    -- Example: {"email": {"transactional": true, "promotional": false}}
    
    -- Unsubscribe
    is_unsubscribed BOOLEAN DEFAULT false,
    unsubscribed_at TIMESTAMP,
    unsubscribe_reason TEXT,
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_user_prefs_unsubscribed ON user_preferences(is_unsubscribed);
```

### 4. Templates Table

```sql
CREATE TABLE templates (
    id VARCHAR(255) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    channel VARCHAR(20) NOT NULL,
    
    -- Template content
    subject_template TEXT,
    body_template TEXT NOT NULL,
    
    -- Variables definition
    variables JSONB, -- [{"name": "user_name", "type": "string", "required": true}]
    
    -- Versioning
    version INTEGER DEFAULT 1,
    is_active BOOLEAN DEFAULT true,
    
    -- A/B testing
    variant VARCHAR(50) DEFAULT 'default',
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT chk_template_channel CHECK (channel IN ('email', 'sms', 'push'))
);

CREATE INDEX idx_templates_channel ON templates(channel, is_active);
CREATE INDEX idx_templates_name ON templates(name);
```

### 5. Providers Table

```sql
CREATE TABLE providers (
    id VARCHAR(255) PRIMARY KEY,
    name VARCHAR(100) NOT NULL, -- sendgrid, twilio, fcm
    channel VARCHAR(20) NOT NULL,
    
    -- Configuration
    config JSONB NOT NULL, -- API keys, endpoints, etc.
    
    -- Rate limiting
    rate_limit_per_second INTEGER,
    rate_limit_per_hour INTEGER,
    
    -- Health & priority
    is_active BOOLEAN DEFAULT true,
    priority INTEGER DEFAULT 1, -- 1 = highest priority
    health_score DECIMAL(3,2) DEFAULT 1.0, -- 0.0 to 1.0
    
    -- Cost
    cost_per_message DECIMAL(10,6),
    
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_providers_channel ON providers(channel, is_active, priority);
```

### 6. Delivery Logs Table (Time-series)

```sql
CREATE TABLE delivery_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notification_id UUID NOT NULL,
    
    -- Event details
    event_type VARCHAR(20) NOT NULL, -- sent, delivered, bounced, clicked
    timestamp TIMESTAMP NOT NULL,
    
    -- Provider info
    provider VARCHAR(50),
    provider_message_id VARCHAR(255),
    
    -- Additional data
    metadata JSONB,
    
    created_at TIMESTAMP DEFAULT NOW()
);

-- Time-series partitioning
CREATE INDEX idx_delivery_logs_notification ON delivery_logs(notification_id);
CREATE INDEX idx_delivery_logs_timestamp ON delivery_logs(timestamp DESC);
CREATE INDEX idx_delivery_logs_event ON delivery_logs(event_type, timestamp DESC);
```

### 7. Batch Jobs Table

```sql
CREATE TABLE batch_jobs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Job details
    template_id VARCHAR(255) NOT NULL,
    channel VARCHAR(20) NOT NULL,
    priority VARCHAR(10) NOT NULL,
    
    -- Recipients
    total_recipients INTEGER NOT NULL,
    processed_count INTEGER DEFAULT 0,
    successful_count INTEGER DEFAULT 0,
    failed_count INTEGER DEFAULT 0,
    
    -- Status
    status VARCHAR(20) DEFAULT 'queued',
    -- queued, processing, completed, failed, cancelled
    
    -- Timing
    created_at TIMESTAMP DEFAULT NOW(),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    
    -- Configuration
    scheduled_at TIMESTAMP,
    variables JSONB, -- Common variables for all recipients
    
    CONSTRAINT chk_batch_status CHECK (status IN (
        'queued', 'processing', 'completed', 'failed', 'cancelled'
    ))
);

CREATE INDEX idx_batch_jobs_status ON batch_jobs(status, created_at);
```

### 8. Redis Data Structures

```redis
# User preferences cache (1 hour TTL)
HSET user_prefs:user-123 email_enabled true sms_enabled false marketing_enabled false
EXPIRE user_prefs:user-123 3600

# Template cache (2 hours TTL)
SET template:welcome-email '{"subject": "Welcome {{name}}", "body": "..."}'
EXPIRE template:welcome-email 7200

# Idempotency keys (24 hours TTL)
SET idempotency:unique-key-123 "notif-abc123" EX 86400

# Rate limiting (per provider API key)
INCR rate_limit:sendgrid:api-key-1:hour:2024011510 EX 3600
INCR rate_limit:sendgrid:api-key-1:second:1642248000 EX 1

# Provider health tracking
HSET provider_health:sendgrid success_count 9850 failure_count 150 last_updated 1642248000

# Dead letter queue
LPUSH dlq:email '{"notification_id": "notif-failed", "error": "Invalid email", "attempts": 5}'
```

### 9. Kafka Topics Structure

```yaml
# Topic: high_priority_notifications
Partitions: 10
Replication Factor: 3
Retention: 7 days
Message Format:
{
  "notification_id": "notif-abc123",
  "user_id": "user-123",
  "channel": "email",
  "priority": "high",
  "template_id": "welcome-email",
  "recipient": "user@example.com",
  "variables": {"name": "John"},
  "scheduled_at": "2024-01-15T10:00:00Z"
}

# Topic: retry_notifications
Partitions: 5
Replication Factor: 3
Retention: 30 days
Message Format:
{
  "notification_id": "notif-abc123",
  "retry_count": 2,
  "next_retry_at": "2024-01-15T10:05:00Z",
  "error_message": "Rate limit exceeded"
}

# Topic: delivery_events
Partitions: 20
Replication Factor: 3
Retention: 90 days
Message Format:
{
  "notification_id": "notif-abc123",
  "event_type": "delivered",
  "timestamp": "2024-01-15T10:00:07Z",
  "provider": "sendgrid",
  "metadata": {"open_count": 1}
}
```

---

### Why This Data Model Works:

**1. Scalability**
✅ Partitioned notifications table by month
✅ Proper indexing for common queries
✅ Separate time-series table for logs

**2. Performance**
✅ Redis caching for hot data
✅ JSONB for flexible metadata
✅ Optimized indexes for status queries

**3. Reliability**
✅ Idempotency key constraints
✅ Retry tracking with timestamps
✅ Comprehensive error logging

**4. Flexibility**
✅ JSONB for dynamic variables
✅ Template versioning support
✅ Extensible provider configuration

---

## PHASE 7: CORE ALGORITHM - MESSAGE PROCESSING & RETRY LOGIC (6 minutes)

### What to Say:

"The core algorithm handles message processing with intelligent retry logic and provider failover. Let me explain the multi-layered approach."

### Message Processing Flow

```python
class NotificationProcessor:
    def __init__(self):
        self.provider_manager = ProviderManager()
        self.retry_handler = RetryHandler()
        self.rate_limiter = RateLimiter()
        self.circuit_breaker = CircuitBreaker()
    
    async def process_notification(self, notification):
        """
        Main processing logic with comprehensive error handling
        """
        try:
            # Step 1: Validate and prepare
            if not await self._validate_notification(notification):
                await self._mark_failed(notification, "Validation failed")
                return
            
            # Step 2: Check user preferences
            if not await self._check_user_preferences(notification):
                await self._mark_skipped(notification, "User opted out")
                return
            
            # Step 3: Rate limiting check
            if not await self._check_rate_limits(notification):
                await self._requeue_with_delay(notification, delay=60)
                return
            
            # Step 4: Select provider and send
            result = await self._send_with_failover(notification)
            
            if result.success:
                await self._mark_sent(notification, result)
            else:
                await self._handle_failure(notification, result.error)
                
        except Exception as e:
            logger.error(f"Unexpected error processing {notification.id}: {e}")
            await self._handle_failure(notification, str(e))
    
    async def _send_with_failover(self, notification):
        """
        Try multiple providers with circuit breaker protection
        """
        providers = self.provider_manager.get_providers(notification.channel)
        
        for provider in providers:
            # Check circuit breaker
            if self.circuit_breaker.is_open(provider.name):
                continue
            
            try:
                # Check provider-specific rate limits
                if not await self.rate_limiter.can_send(provider, notification):
                    continue
                
                # Attempt to send
                result = await provider.send(notification)
                
                # Record success
                self.circuit_breaker.record_success(provider.name)
                return result
                
            except ProviderError as e:
                # Record failure
                self.circuit_breaker.record_failure(provider.name)
                logger.warning(f"Provider {provider.name} failed: {e}")
                continue
        
        # All providers failed
        return SendResult(success=False, error="All providers failed")
```

### Intelligent Retry Logic

```python
class RetryHandler:
    def __init__(self):
        self.max_retries = 5
        self.base_delay = 1  # seconds
        self.max_delay = 300  # 5 minutes
    
    async def handle_failure(self, notification, error):
        """
        Determine if failure is retryable and calculate next retry time
        """
        # Classify error type
        error_type = self._classify_error(error)
        
        if error_type == ErrorType.PERMANENT:
            await self._move_to_dlq(notification, error)
            return
        
        if notification.retry_count >= self.max_retries:
            await self._move_to_dlq(notification, "Max retries exceeded")
            return
        
        # Calculate exponential backoff with jitter
        delay = self._calculate_backoff_delay(
            notification.retry_count, 
            error_type
        )
        
        # Update notification for retry
        notification.retry_count += 1
        notification.next_retry_at = datetime.now() + timedelta(seconds=delay)
        notification.error_message = str(error)
        
        await self._schedule_retry(notification, delay)
    
    def _classify_error(self, error):
        """
        Classify errors as permanent, temporary, or rate-limited
        """
        error_str = str(error).lower()
        
        # Permanent errors (don't retry)
        permanent_indicators = [
            'invalid email', 'invalid phone', 'unsubscribed',
            'blocked', 'spam', 'invalid recipient'
        ]
        
        if any(indicator in error_str for indicator in permanent_indicators):
            return ErrorType.PERMANENT
        
        # Rate limit errors (retry with longer delay)
        rate_limit_indicators = [
            'rate limit', 'quota exceeded', 'too many requests'
        ]
        
        if any(indicator in error_str for indicator in rate_limit_indicators):
            return ErrorType.RATE_LIMITED
        
        # Default to temporary (network issues, server errors)
        return ErrorType.TEMPORARY
    
    def _calculate_backoff_delay(self, retry_count, error_type):
        """
        Exponential backoff with jitter and error-type specific delays
        """
        if error_type == ErrorType.RATE_LIMITED:
            # Longer delays for rate limiting
            base_delay = 60  # 1 minute
        else:
            base_delay = self.base_delay
        
        # Exponential backoff: delay = base * (2 ^ retry_count)
        delay = base_delay * (2 ** retry_count)
        
        # Add jitter (±25% randomness)
        jitter = delay * 0.25 * (random.random() - 0.5)
        delay += jitter
        
        # Cap at maximum delay
        return min(delay, self.max_delay)
```

### Provider Management & Circuit Breaker

```python
class ProviderManager:
    def __init__(self):
        self.providers = {}
        self.load_providers()
    
    def get_providers(self, channel):
        """
        Get providers for channel, sorted by priority and health
        """
        channel_providers = [
            p for p in self.providers[channel] 
            if p.is_active
        ]
        
        # Sort by priority (1 = highest) and health score
        return sorted(
            channel_providers,
            key=lambda p: (p.priority, -p.health_score)
        )
    
    async def update_provider_health(self, provider_name, success):
        """
        Update provider health score based on recent performance
        """
        provider = self.providers_by_name[provider_name]
        
        # Exponential moving average
        if success:
            provider.health_score = 0.9 * provider.health_score + 0.1 * 1.0
        else:
            provider.health_score = 0.9 * provider.health_score + 0.1 * 0.0
        
        # Deactivate if health too low
        if provider.health_score < 0.3:
            provider.is_active = False
            await self._alert_provider_unhealthy(provider_name)

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_counts = {}
        self.last_failure_times = {}
        self.states = {}  # CLOSED, OPEN, HALF_OPEN
    
    def is_open(self, provider_name):
        """
        Check if circuit breaker is open for provider
        """
        state = self.states.get(provider_name, 'CLOSED')
        
        if state == 'CLOSED':
            return False
        
        if state == 'OPEN':
            # Check if timeout period has passed
            last_failure = self.last_failure_times.get(provider_name, 0)
            if time.time() - last_failure > self.timeout:
                self.states[provider_name] = 'HALF_OPEN'
                return False
            return True
        
        if state == 'HALF_OPEN':
            return False
        
        return False
    
    def record_success(self, provider_name):
        """
        Record successful request
        """
        self.failure_counts[provider_name] = 0
        self.states[provider_name] = 'CLOSED'
    
    def record_failure(self, provider_name):
        """
        Record failed request
        """
        self.failure_counts[provider_name] = (
            self.failure_counts.get(provider_name, 0) + 1
        )
        self.last_failure_times[provider_name] = time.time()
        
        if self.failure_counts[provider_name] >= self.failure_threshold:
            self.states[provider_name] = 'OPEN'
```

### Rate Limiting Implementation

```python
class RateLimiter:
    def __init__(self):
        self.redis = Redis()
    
    async def can_send(self, provider, notification):
        """
        Check if we can send through this provider without hitting rate limits
        """
        # Check per-second limit
        if provider.rate_limit_per_second:
            current_second = int(time.time())
            key = f"rate_limit:{provider.name}:second:{current_second}"
            
            current_count = await self.redis.get(key) or 0
            if int(current_count) >= provider.rate_limit_per_second:
                return False
        
        # Check per-hour limit
        if provider.rate_limit_per_hour:
            current_hour = int(time.time() // 3600)
            key = f"rate_limit:{provider.name}:hour:{current_hour}"
            
            current_count = await self.redis.get(key) or 0
            if int(current_count) >= provider.rate_limit_per_hour:
                return False
        
        return True
    
    async def record_send(self, provider):
        """
        Record that we sent a message through this provider
        """
        current_second = int(time.time())
        current_hour = int(time.time() // 3600)
        
        # Increment counters
        await self.redis.incr(
            f"rate_limit:{provider.name}:second:{current_second}"
        )
        await self.redis.expire(
            f"rate_limit:{provider.name}:second:{current_second}", 
            1
        )
        
        await self.redis.incr(
            f"rate_limit:{provider.name}:hour:{current_hour}"
        )
        await self.redis.expire(
            f"rate_limit:{provider.name}:hour:{current_hour}", 
            3600
        )
```

### Dead Letter Queue Management

```python
class DeadLetterQueueManager:
    def __init__(self):
        self.redis = Redis()
        self.s3_client = boto3.client('s3')
    
    async def add_to_dlq(self, notification, error_reason):
        """
        Add failed notification to dead letter queue
        """
        dlq_entry = {
            'notification_id': notification.id,
            'user_id': notification.user_id,
            'channel': notification.channel,
            'recipient': notification.recipient,
            'error_reason': error_reason,
            'retry_count': notification.retry_count,
            'failed_at': datetime.now().isoformat(),
            'original_payload': notification.to_dict()
        }
        
        # Add to Redis for immediate processing
        await self.redis.lpush(
            f'dlq:{notification.channel}',
            json.dumps(dlq_entry)
        )
        
        # Archive to S3 for long-term storage
        await self._archive_to_s3(dlq_entry)
        
        # Update notification status
        notification.status = 'failed'
        notification.error_message = error_reason
        await notification.save()
        
        # Alert if DLQ size is growing
        dlq_size = await self.redis.llen(f'dlq:{notification.channel}')
        if dlq_size > 1000:
            await self._alert_high_dlq_volume(notification.channel, dlq_size)
    
    async def process_dlq_manually(self, channel, limit=100):
        """
        Manual processing of DLQ items (for investigation/reprocessing)
        """
        dlq_items = []
        for _ in range(limit):
            item = await self.redis.rpop(f'dlq:{channel}')
            if not item:
                break
            dlq_items.append(json.loads(item))
        
        return dlq_items
```

### Performance Metrics

```
┌──────────────────────────┬─────────────┬─────────────┐
│ Metric                   │ Target      │ Actual      │
├──────────────────────────┼─────────────┼─────────────┤
│ Processing latency       │ < 1 sec     │ 0.3 sec     │
│ Success rate (first try) │ > 95%       │ 97%         │
│ Retry success rate       │ > 80%       │ 85%         │
│ DLQ rate                 │ < 1%        │ 0.3%        │
│ Provider failover time   │ < 5 sec     │ 2 sec       │
│ Circuit breaker recovery │ < 60 sec    │ 45 sec      │
└──────────────────────────┴─────────────┴─────────────┘
```

---

## PHASE 8: DEEP DIVE - RELIABILITY & FAULT TOLERANCE (4 minutes)

### What to Say:

"Reliability is critical for a notification system. Let me explain our multi-layered approach to handle failures gracefully."

### Fault Tolerance Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  RELIABILITY LAYERS                                              │
└─────────────────────────────────────────────────────────────────┘

Layer 1: API Level
├── Rate limiting (prevent overload)
├── Input validation (reject bad requests)
├── Idempotency (prevent duplicates)
└── Circuit breaker (fail fast)

Layer 2: Queue Level  
├── Message durability (Kafka replication)
├── Dead letter queues (permanent failures)
├── Priority queues (critical messages first)
└── Backpressure handling (queue depth monitoring)

Layer 3: Worker Level
├── Graceful shutdown (finish current work)
├── Health checks (auto-restart unhealthy workers)
├── Resource limits (prevent memory leaks)
└── Error isolation (one failure doesn't affect others)

Layer 4: Provider Level
├── Multi-provider failover (automatic switching)
├── Circuit breakers (isolate failing providers)
├── Rate limiting (respect provider limits)
└── Health monitoring (track provider performance)

Layer 5: Data Level
├── Database replication (Multi-AZ)
├── Backup and recovery (point-in-time restore)
├── Data validation (consistency checks)
└── Audit logging (track all changes)
```

### Disaster Recovery Implementation

```python
class DisasterRecoveryManager:
    def __init__(self):
        self.health_checker = HealthChecker()
        self.failover_manager = FailoverManager()
        self.backup_manager = BackupManager()
    
    async def handle_database_failure(self):
        """
        Handle primary database failure
        """
        logger.critical("Primary database failure detected")
        
        # 1. Stop accepting new requests
        await self._enable_maintenance_mode()
        
        # 2. Promote read replica to primary
        new_primary = await self.failover_manager.promote_replica()
        
        # 3. Update connection strings
        await self._update_database_connections(new_primary)
        
        # 4. Resume operations
        await self._disable_maintenance_mode()
        
        # 5. Alert operations team
        await self._alert_database_failover(new_primary)
    
    async def handle_kafka_failure(self):
        """
        Handle message queue failure
        """
        logger.critical("Kafka cluster failure detected")
        
        # 1. Switch to backup queue (Redis)
        await self._switch_to_backup_queue()
        
        # 2. Continue processing with reduced throughput
        await self._reduce_worker_count()
        
        # 3. Monitor Kafka recovery
        asyncio.create_task(self._monitor_kafka_recovery())
    
    async def handle_provider_outage(self, provider_name):
        """
        Handle complete provider outage
        """
        logger.warning(f"Provider {provider_name} outage detected")
        
        # 1. Mark provider as inactive
        await self._deactivate_provider(provider_name)
        
        # 2. Redistribute load to other providers
        await self._rebalance_provider_load()
        
        # 3. Increase retry delays for affected messages
        await self._increase_retry_delays(provider_name)
        
        # 4. Monitor for recovery
        asyncio.create_task(self._monitor_provider_recovery(provider_name))

class HealthChecker:
    async def check_system_health(self):
        """
        Comprehensive health check
        """
        health_status = {
            'database': await self._check_database_health(),
            'kafka': await self._check_kafka_health(),
            'redis': await self._check_redis_health(),
            'providers': await self._check_provider_health(),
            'workers': await self._check_worker_health()
        }
        
        overall_health = all(health_status.values())
        
        if not overall_health:
            await self._trigger_alerts(health_status)
        
        return health_status
    
    async def _check_database_health(self):
        try:
            # Simple query with timeout
            result = await asyncio.wait_for(
                self.db.execute("SELECT 1"),
                timeout=5.0
            )
            return True
        except asyncio.TimeoutError:
            return False
        except Exception:
            return False
    
    async def _check_provider_health(self):
        """
        Check all providers are responding
        """
        provider_health = {}
        
        for provider in self.providers:
            try:
                # Send test message or health check
                response = await provider.health_check()
                provider_health[provider.name] = response.is_healthy
            except Exception:
                provider_health[provider.name] = False
        
        return all(provider_health.values())
```

### Graceful Degradation

```python
class GracefulDegradationManager:
    def __init__(self):
        self.degradation_levels = {
            'NORMAL': 0,
            'LIGHT_LOAD': 1,
            'HEAVY_LOAD': 2,
            'CRITICAL': 3,
            'EMERGENCY': 4
        }
        self.current_level = 'NORMAL'
    
    async def adjust_system_behavior(self, queue_depth, error_rate):
        """
        Adjust system behavior based on current conditions
        """
        new_level = self._calculate_degradation_level(queue_depth, error_rate)
        
        if new_level != self.current_level:
            await self._apply_degradation_level(new_level)
            self.current_level = new_level
    
    def _calculate_degradation_level(self, queue_depth, error_rate):
        """
        Determine appropriate degradation level
        """
        if queue_depth > 1000000 or error_rate > 0.5:
            return 'EMERGENCY'
        elif queue_depth > 500000 or error_rate > 0.3:
            return 'CRITICAL'
        elif queue_depth > 100000 or error_rate > 0.15:
            return 'HEAVY_LOAD'
        elif queue_depth > 50000 or error_rate > 0.05:
            return 'LIGHT_LOAD'
        else:
            return 'NORMAL'
    
    async def _apply_degradation_level(self, level):
        """
        Apply specific degradation measures
        """
        if level == 'LIGHT_LOAD':
            # Reduce batch sizes, increase worker count
            await self._adjust_batch_sizes(0.8)
            await self._scale_workers(1.2)
        
        elif level == 'HEAVY_LOAD':
            # Pause low priority messages
            await self._pause_low_priority_processing()
            await self._scale_workers(1.5)
        
        elif level == 'CRITICAL':
            # Only process high priority messages
            await self._pause_medium_priority_processing()
            await self._enable_emergency_scaling()
        
        elif level == 'EMERGENCY':
            # Reject new requests, focus on clearing backlog
            await self._enable_maintenance_mode()
            await self._maximum_worker_scaling()
            await self._alert_emergency_mode()
```

### Monitoring & Alerting

```python
class MonitoringSystem:
    def __init__(self):
        self.metrics_collector = MetricsCollector()
        self.alert_manager = AlertManager()
    
    async def collect_metrics(self):
        """
        Collect comprehensive system metrics
        """
        metrics = {
            # Throughput metrics
            'notifications_per_second': await self._get_current_qps(),
            'queue_depth': await self._get_total_queue_depth(),
            'processing_latency': await self._get_avg_processing_time(),
            
            # Success metrics
            'success_rate': await self._get_success_rate(),
            'delivery_rate': await self._get_delivery_rate(),
            'retry_rate': await self._get_retry_rate(),
            
            # Provider metrics
            'provider_health': await self._get_provider_health_scores(),
            'provider_latency': await self._get_provider_latencies(),
            'provider_costs': await self._get_provider_costs(),
            
            # System metrics
            'worker_utilization': await self._get_worker_utilization(),
            'database_connections': await self._get_db_connection_count(),
            'memory_usage': await self._get_memory_usage()
        }
        
        await self._check_alert_conditions(metrics)
        return metrics
    
    async def _check_alert_conditions(self, metrics):
        """
        Check if any metrics exceed alert thresholds
        """
        alerts = []
        
        # High queue depth
        if metrics['queue_depth'] > 100000:
            alerts.append({
                'severity': 'WARNING',
                'message': f"High queue depth: {metrics['queue_depth']}"
            })
        
        # Low success rate
        if metrics['success_rate'] < 0.95:
            alerts.append({
                'severity': 'CRITICAL',
                'message': f"Low success rate: {metrics['success_rate']:.2%}"
            })
        
        # High processing latency
        if metrics['processing_latency'] > 5.0:
            alerts.append({
                'severity': 'WARNING',
                'message': f"High latency: {metrics['processing_latency']:.2f}s"
            })
        
        for alert in alerts:
            await self.alert_manager.send_alert(alert)

# Alert configuration
ALERT_RULES = {
    'queue_depth_high': {
        'threshold': 100000,
        'severity': 'WARNING',
        'cooldown': 300  # 5 minutes
    },
    'success_rate_low': {
        'threshold': 0.95,
        'severity': 'CRITICAL',
        'cooldown': 60  # 1 minute
    },
    'provider_down': {
        'threshold': 0.5,  # Health score
        'severity': 'CRITICAL',
        'cooldown': 180  # 3 minutes
    }
}
```

---

## PHASE 9: FOLLOW-UP QUESTIONS (5 minutes)

### What to Say:

"Let me address some common follow-up questions and advanced scenarios."

### Q1: How do you handle message ordering?

```python
# Kafka partitioning by user_id ensures ordering per user
def get_partition_key(notification):
    return notification.user_id

# For global ordering (rare requirement):
class OrderedNotificationProcessor:
    def __init__(self):
        self.sequence_number = 0
        self.lock = asyncio.Lock()
    
    async def process_ordered(self, notification):
        async with self.lock:
            self.sequence_number += 1
            notification.sequence = self.sequence_number
            await self._send_notification(notification)
```

### Q2: How do you prevent duplicate notifications?

```python
class DuplicationPrevention:
    async def check_duplicate(self, notification):
        # Method 1: Idempotency key
        if notification.idempotency_key:
            existing = await redis.get(f"idempotency:{notification.idempotency_key}")
            if existing:
                return existing
        
        # Method 2: Content-based deduplication
        content_hash = hashlib.sha256(
            f"{notification.user_id}:{notification.template_id}:{notification.variables}".encode()
        ).hexdigest()
        
        recent_key = f"recent:{content_hash}"
        if await redis.exists(recent_key):
            return "duplicate"
        
        # Mark as sent for 1 hour
        await redis.setex(recent_key, 3600, notification.id)
        return None
```

### Q3: How do you handle time zone-aware delivery?

```python
class TimeZoneAwareScheduler:
    async def schedule_notification(self, notification, user_timezone):
        # Convert scheduled time to user's timezone
        user_tz = pytz.timezone(user_timezone)
        scheduled_time = notification.scheduled_at.astimezone(user_tz)
        
        # Check quiet hours
        quiet_start = user.quiet_hours_start  # e.g., 22:00
        quiet_end = user.quiet_hours_end      # e.g., 08:00
        
        current_time = scheduled_time.time()
        
        if self._is_in_quiet_hours(current_time, quiet_start, quiet_end):
            # Reschedule to end of quiet hours
            next_day = scheduled_time.date() + timedelta(days=1)
            new_time = datetime.combine(next_day, quiet_end)
            notification.scheduled_at = new_time.astimezone(pytz.UTC)
```

### Q4: How do you implement A/B testing for templates?

```python
class ABTestingManager:
    async def select_template_variant(self, user_id, template_id):
        # Get A/B test configuration
        ab_test = await self.get_ab_test(template_id)
        if not ab_test:
            return template_id
        
        # Consistent assignment based on user_id
        user_hash = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
        bucket = user_hash % 100
        
        if bucket < ab_test.variant_a_percentage:
            return ab_test.variant_a_template_id
        else:
            return ab_test.variant_b_template_id
    
    async def track_ab_test_result(self, user_id, template_id, event):
        # Track opens, clicks, conversions
        await self.metrics.increment(
            f"ab_test.{template_id}.{event}",
            tags={'user_id': user_id}
        )
```

### Q5: How do you handle regulatory compliance (GDPR, CAN-SPAM)?

```python
class ComplianceManager:
    async def check_compliance(self, notification):
        # GDPR: Check consent
        if notification.user_region == 'EU':
            consent = await self.get_user_consent(notification.user_id)
            if not consent.marketing_allowed and notification.type == 'marketing':
                return False
        
        # CAN-SPAM: Include unsubscribe link
        if notification.channel == 'email' and notification.type == 'marketing':
            unsubscribe_link = self.generate_unsubscribe_link(notification.user_id)
            notification.body += f"\n\nUnsubscribe: {unsubscribe_link}"
        
        return True
    
    async def handle_unsubscribe(self, user_id, token):
        # Verify token
        if not self.verify_unsubscribe_token(user_id, token):
            raise InvalidTokenError()
        
        # Update preferences
        await self.update_user_preferences(user_id, {
            'marketing_enabled': False,
            'unsubscribed_at': datetime.now()
        })
        
        # Add to suppression list
        await self.add_to_suppression_list(user_id)
```

### Q6: How do you optimize costs?

```python
class CostOptimizer:
    async def select_cheapest_provider(self, notification):
        providers = self.get_available_providers(notification.channel)
        
        # Sort by cost per message
        providers.sort(key=lambda p: p.cost_per_message)
        
        for provider in providers:
            if await self.can_use_provider(provider, notification):
                return provider
        
        return None
    
    async def batch_optimize(self, notifications):
        # Group by provider and send in batches
        provider_batches = defaultdict(list)
        
        for notification in notifications:
            provider = await self.select_cheapest_provider(notification)
            provider_batches[provider].append(notification)
        
        # Send batches
        for provider, batch in provider_batches.items():
            await provider.send_batch(batch)
```

### Performance Summary:

```
┌──────────────────────────┬─────────────┬─────────────┐
│ Component                │ Throughput  │ Latency     │
├──────────────────────────┼─────────────┼─────────────┤
│ API Gateway              │ 50K req/sec │ < 10ms      │
│ Notification API         │ 10K req/sec │ < 100ms     │
│ Email Workers            │ 6K msg/sec  │ < 2 sec     │
│ SMS Workers              │ 1K msg/sec  │ < 5 sec     │
│ Push Workers             │ 3K msg/sec  │ < 1 sec     │
│ Database (writes)        │ 15K ops/sec │ < 50ms      │
│ Redis (cache)            │ 100K ops/sec│ < 1ms       │
│ Kafka (messages)         │ 50K msg/sec │ < 10ms      │
└──────────────────────────┴─────────────┴─────────────┘

Total System Capacity: 100M notifications/day
Peak Throughput: 10K notifications/second
End-to-end Latency: < 5 seconds (high priority)
Availability: 99.9% uptime
Cost: $143K/month (77% provider costs)
```

This completes the comprehensive notification system design that can handle 100M notifications per day with high reliability, fault tolerance, and cost optimization.