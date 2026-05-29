# High-Level-Design-Practice

A comprehensive collection of **System Design / High-Level Design (HLD)** interview guides following a structured 10-phase interview framework.

## 📋 Interview Framework (10 Phases)

Each guide follows a consistent structure:
1. Requirements Clarification
2. Functional & Non-Functional Requirements
3. Back-of-Envelope Estimation
4. Core Entities & Data Model
5. API Design
6. High-Level Architecture
7. Data Flows
8. Deep Dive 1 (Consistency / Performance)
9. Deep Dive 2 (Scalability)
10. Deep Dive 3 (Fault Tolerance / Edge Cases)

## 📂 System Design Guides

| System | Key Concepts |
|--------|-------------|
| [Seller Payment](SELLER_PAYMENT_COMPLETE_INTERVIEW.md) | Idempotency, eventual consistency, payment reconciliation |
| [Task Scheduler](TASK_SCHEDULER_COMPLETE_INTERVIEW.md) | DynamoDB + SQS, time-bucket partitioning, write sharding |
| [Facebook Live Comments](FACEBOOK_LIVE_COMMENTS_COMPLETE_INTERVIEW.md) | SSE, Redis Pub/Sub, channel sharding, CDN snapshots |
| [Online Auction](ONLINE_AUCTION_COMPLETE_INTERVIEW.md) | Optimistic Concurrency Control, Kafka durability, SSE |
| [E-Commerce Platform](ECOMMERCE_COMPLETE_INTERVIEW.md) | Inventory management, order processing, catalog |
| [E-Commerce Popularity](ECOMMERCE_POPULARITY_COMPLETE_INTERVIEW.md) | Trending algorithms, real-time ranking |
| [Chat Application](CHAT_APPLICATION_COMPLETE_INTERVIEW.md) | WebSockets, message ordering, presence |
| [Facebook Newsfeed](FACEBOOK_NEWSFEED_COMPLETE_INTERVIEW.md) | Fan-out, ranking, caching strategies |
| [Food Delivery](FOOD_DELIVERY_COMPLETE_INTERVIEW.md) | Location tracking, matching, ETA estimation |
| [Notification System](NOTIFICATION_SYSTEM_COMPLETE_INTERVIEW.md) | Multi-channel delivery, priority queues, deduplication |
| [URL Shortener](URL_SHORTENER_COMPLETE_INTERVIEW.md) | Base62 encoding, read-heavy caching, analytics |
| [Dropbox File Sync](DROPBOX_FILE_SYNC_COMPLETE_INTERVIEW.md) | Chunking, deduplication, conflict resolution |
| [Distributed KV Store](DISTRIBUTED_KV_STORE_COMPLETE_INTERVIEW.md) | Consistent hashing, replication, quorum |
| [Uber](UBER_COMPLETE_HLD.md) | Geospatial indexing, matching, surge pricing |
| [TicketMaster](TICKETMASTER_COMPLETE_HLD.md) | Seat reservation, distributed locking, high concurrency |
| [WhatsApp](WHATSAPP_PROGRESSIVE_INTERVIEW.md) | E2E encryption, message delivery guarantees |
| [Cart Management](CART_MANAGEMENT_COMPLETE_INTERVIEW.md) | Session handling, inventory reservation |
| [Healthcare Recommendation](HEALTHCARE_MEDICINE_RECOMMENDATION_SYSTEM_HLD.md) | ML integration, safety constraints |

## 📚 Concept Deep Dives

| Topic | Coverage |
|-------|----------|
| [Distributed Systems Mastery](DISTRIBUTED_SYSTEMS_MASTERY.md) | Kafka, Redis, Elasticsearch - internals & Java implementations |
| [Isolation Levels & Locking](ISOLATION_LEVELS_AND_LOCKING_EXPLAINED.md) | DB isolation, row locking, MVCC |
| [Chat Deep Dive](CHAT_DEEP_DIVE_FOLLOWUPS.md) | Advanced messaging patterns |

## 🛠️ Tech Stack Covered

- **Databases**: PostgreSQL, DynamoDB, Cassandra, Redis
- **Messaging**: Kafka, SQS, Redis Pub/Sub
- **Real-time**: WebSockets, SSE, Long Polling
- **Search**: Elasticsearch
- **Infrastructure**: Consistent Hashing, Sharding, Replication

## 🎯 How to Use

1. Pick a system design problem
2. Follow the 10-phase structure to practice articulating your solution
3. Use the "What to Say" prompts for interview simulation
4. Review deep dives for follow-up questions
