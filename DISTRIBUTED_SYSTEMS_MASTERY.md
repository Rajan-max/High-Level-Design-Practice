# Distributed Systems Mastery: Kafka, Redis & Elasticsearch

## TABLE OF CONTENTS
1. [Apache Kafka - Event Streaming Platform](#kafka)
2. [Redis - In-Memory Data Store](#redis)
3. [Elasticsearch - Search & Analytics Engine](#elasticsearch)
4. [Integration Patterns & Best Practices](#integration)
5. [Interview Questions & Scenarios](#interview)

---

# APACHE KAFKA - EVENT STREAMING PLATFORM {#kafka}

## What is Kafka?

Kafka is a **distributed event streaming platform** that acts as a high-throughput, fault-tolerant message broker. Think of it as a "commit log" that multiple applications can read from and write to.

### Key Concepts:

```
PRODUCER → TOPIC (Partitions) → CONSUMER
    ↓         ↓                    ↓
  Events   Storage            Processing
```

**Core Components:**
- **Producer**: Publishes events to topics
- **Consumer**: Subscribes to topics and processes events
- **Topic**: Category/feed of events (like a database table)
- **Partition**: Ordered sequence within a topic (for parallelism)
- **Broker**: Kafka server that stores and serves data
- **Cluster**: Group of brokers working together

## Deep Dive Architecture

### 1. Topic & Partition Model

```
Topic: "user-events"
┌─────────────────────────────────────────────────────────┐
│ Partition 0: [msg1] [msg2] [msg3] [msg4] [msg5] ──→     │
│ Partition 1: [msg6] [msg7] [msg8] [msg9] ──→            │
│ Partition 2: [msg10] [msg11] [msg12] ──→                │
└─────────────────────────────────────────────────────────┘
                    ↑
            Messages are immutable and ordered within partition
```

**Why Partitions?**
- **Parallelism**: Multiple consumers can read different partitions
- **Scalability**: Distribute load across multiple brokers
- **Ordering**: Maintains order within each partition

### 2. Replication & Fault Tolerance

```
Topic: "orders" (Replication Factor = 3)

Broker 1 (Leader):   [P0-Leader] [P1-Follower] [P2-Follower]
Broker 2 (Follower): [P0-Follower] [P1-Leader] [P2-Follower]  
Broker 3 (Follower): [P0-Follower] [P1-Follower] [P2-Leader]

If Broker 1 fails → Broker 2 becomes leader for P0
```

**Replication Guarantees:**
- **Leader**: Handles all reads/writes for a partition
- **Followers**: Replicate data from leader
- **ISR (In-Sync Replicas)**: Followers that are caught up
- **Min ISR**: Minimum replicas that must acknowledge writes

### 3. Consumer Groups & Load Balancing

```
Consumer Group: "order-processors"

Topic "orders" (4 partitions):
P0 → Consumer A
P1 → Consumer B  
P2 → Consumer C
P3 → Consumer A

If Consumer B fails:
P0 → Consumer A
P1 → Consumer C (rebalanced)
P2 → Consumer C  
P3 → Consumer A
```

## Kafka Implementation Examples

### 1. Basic Producer (Java)

```java
// Producer Configuration
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("acks", "all"); // Wait for all replicas
props.put("retries", 3);
props.put("batch.size", 16384);
props.put("linger.ms", 5); // Batch messages for 5ms

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// Send message with callback
ProducerRecord<String, String> record = new ProducerRecord<>(
    "user-events", 
    "user123", 
    "{\"action\":\"purchase\",\"amount\":99.99}"
);

producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        System.err.println("Failed to send message: " + exception.getMessage());
    } else {
        System.out.println("Message sent to partition " + metadata.partition() + 
                          " with offset " + metadata.offset());
    }
});
```

### 2. Consumer with Manual Offset Management

```java
// Consumer Configuration
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "order-processors");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("enable.auto.commit", "false"); // Manual offset management
props.put("max.poll.records", 100);

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("user-events"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    
    for (ConsumerRecord<String, String> record : records) {
        try {
            // Process message
            processUserEvent(record.key(), record.value());
            
            // Commit offset after successful processing
            consumer.commitSync(Collections.singletonMap(
                new TopicPartition(record.topic(), record.partition()),
                new OffsetAndMetadata(record.offset() + 1)
            ));
            
        } catch (Exception e) {
            System.err.println("Failed to process message: " + e.getMessage());
            // Don't commit offset on failure - message will be reprocessed
        }
    }
}
```

### 3. Kafka Streams for Real-time Processing

```java
// Stream processing topology
StreamsBuilder builder = new StreamsBuilder();

KStream<String, String> userEvents = builder.stream("user-events");

// Transform and aggregate
KTable<String, Long> userPurchaseCounts = userEvents
    .filter((key, value) -> value.contains("purchase"))
    .groupByKey()
    .count(Materialized.as("user-purchase-counts"));

// Output to new topic
userPurchaseCounts.toStream().to("user-purchase-summary");

// Start streams application
KafkaStreams streams = new KafkaStreams(builder.build(), streamsConfig);
streams.start();
```

## Kafka Use Cases in System Design

### 1. Event-Driven Architecture

```
E-commerce Order Flow:

Order Service → Kafka Topic "orders" → [Payment Service]
                                    → [Inventory Service]  
                                    → [Shipping Service]
                                    → [Analytics Service]
```

### 2. Change Data Capture (CDC)

```
Database Changes → Kafka Connect → Kafka Topic → Search Index Update
                                               → Cache Invalidation
                                               → Analytics Pipeline
```

### 3. Log Aggregation

```
Microservice Logs → Kafka → ELK Stack (Elasticsearch, Logstash, Kibana)
                          → Monitoring & Alerting
```

## Kafka Performance Tuning

### Producer Optimizations:

```java
// High throughput configuration
props.put("batch.size", 65536);        // Larger batches
props.put("linger.ms", 20);            // Wait longer to batch
props.put("compression.type", "lz4");   // Compress messages
props.put("buffer.memory", 67108864);   // 64MB buffer
```

### Consumer Optimizations:

```java
// High throughput configuration  
props.put("fetch.min.bytes", 50000);    // Fetch larger batches
props.put("fetch.max.wait.ms", 500);    // Wait for larger batches
props.put("max.partition.fetch.bytes", 2097152); // 2MB per partition
```

---

# REDIS - IN-MEMORY DATA STORE {#redis}

## What is Redis?

Redis is an **in-memory data structure store** used as database, cache, message broker, and streaming engine. It's single-threaded but extremely fast due to memory-based operations.

### Key Features:
- **Speed**: Sub-millisecond latency
- **Data Structures**: Strings, Lists, Sets, Hashes, Sorted Sets
- **Persistence**: RDB snapshots + AOF logs
- **Replication**: Master-slave with automatic failover
- **Clustering**: Horizontal scaling across nodes

## Redis Data Structures & Use Cases

### 1. Strings (Most Common)

```redis
# Simple key-value
SET user:1001:name "John Doe"
GET user:1001:name

# Atomic counters
INCR page:views:homepage
INCRBY user:1001:score 50

# Expiration
SETEX session:abc123 3600 "user_data"  # Expires in 1 hour
TTL session:abc123  # Check remaining time
```

**Use Cases:**
- Session storage
- Caching API responses  
- Rate limiting counters
- Feature flags

### 2. Hashes (Object Storage)

```redis
# Store user object
HSET user:1001 name "John" email "john@example.com" age 30
HGET user:1001 name
HGETALL user:1001

# Atomic field updates
HINCRBY user:1001 login_count 1
```

**Use Cases:**
- User profiles
- Product catalogs
- Configuration settings

### 3. Lists (Ordered Collections)

```redis
# Queue operations
LPUSH queue:emails "email1@example.com"
RPOP queue:emails

# Recent items
LPUSH user:1001:recent_views "product:123"
LTRIM user:1001:recent_views 0 9  # Keep only 10 items
```

**Use Cases:**
- Message queues
- Activity feeds
- Recent items lists

### 4. Sets (Unique Collections)

```redis
# Add unique items
SADD user:1001:interests "technology" "sports" "music"
SISMEMBER user:1001:interests "technology"  # Check membership

# Set operations
SINTER user:1001:interests user:1002:interests  # Common interests
SUNION user:1001:friends user:1002:friends      # All friends
```

**Use Cases:**
- Tags and categories
- Unique visitors tracking
- Social graph relationships

### 5. Sorted Sets (Ranked Collections)

```redis
# Leaderboard
ZADD leaderboard 1500 "player1" 1200 "player2" 1800 "player3"
ZREVRANGE leaderboard 0 9 WITHSCORES  # Top 10 players

# Time-based data
ZADD user:1001:timeline 1640995200 "event1" 1640995300 "event2"
ZRANGEBYSCORE user:1001:timeline 1640995000 1640996000  # Events in time range
```

**Use Cases:**
- Leaderboards and rankings
- Time-series data
- Priority queues

## Redis Implementation Examples

### 1. Distributed Caching Layer

```java
// Redis connection with Jedis
JedisPool jedisPool = new JedisPool("localhost", 6379);

public class CacheService {
    
    public <T> T get(String key, Class<T> type) {
        try (Jedis jedis = jedisPool.getResource()) {
            String value = jedis.get(key);
            return value != null ? JSON.parseObject(value, type) : null;
        }
    }
    
    public void set(String key, Object value, int ttlSeconds) {
        try (Jedis jedis = jedisPool.getResource()) {
            String json = JSON.toJSONString(value);
            jedis.setex(key, ttlSeconds, json);
        }
    }
    
    // Cache-aside pattern
    public User getUserById(Long userId) {
        String cacheKey = "user:" + userId;
        
        // Try cache first
        User user = get(cacheKey, User.class);
        if (user != null) {
            return user;
        }
        
        // Cache miss - fetch from database
        user = userRepository.findById(userId);
        if (user != null) {
            set(cacheKey, user, 3600); // Cache for 1 hour
        }
        
        return user;
    }
}
```

### 2. Session Management

```java
public class SessionManager {
    
    public void createSession(String sessionId, String userId) {
        try (Jedis jedis = jedisPool.getResource()) {
            String sessionKey = "session:" + sessionId;
            
            // Store session data
            jedis.hset(sessionKey, "userId", userId);
            jedis.hset(sessionKey, "createdAt", String.valueOf(System.currentTimeMillis()));
            jedis.hset(sessionKey, "lastAccessed", String.valueOf(System.currentTimeMillis()));
            
            // Set expiration
            jedis.expire(sessionKey, 1800); // 30 minutes
        }
    }
    
    public boolean isValidSession(String sessionId) {
        try (Jedis jedis = jedisPool.getResource()) {
            String sessionKey = "session:" + sessionId;
            
            if (!jedis.exists(sessionKey)) {
                return false;
            }
            
            // Update last accessed time
            jedis.hset(sessionKey, "lastAccessed", String.valueOf(System.currentTimeMillis()));
            jedis.expire(sessionKey, 1800); // Reset expiration
            
            return true;
        }
    }
}
```

### 3. Rate Limiting with Sliding Window

```java
public class RateLimiter {
    
    public boolean isAllowed(String userId, int maxRequests, int windowSeconds) {
        try (Jedis jedis = jedisPool.getResource()) {
            String key = "rate_limit:" + userId;
            long now = System.currentTimeMillis();
            long windowStart = now - (windowSeconds * 1000);
            
            // Remove old entries
            jedis.zremrangeByScore(key, 0, windowStart);
            
            // Count current requests
            long currentRequests = jedis.zcard(key);
            
            if (currentRequests >= maxRequests) {
                return false;
            }
            
            // Add current request
            jedis.zadd(key, now, String.valueOf(now));
            jedis.expire(key, windowSeconds);
            
            return true;
        }
    }
}
```

### 4. Pub/Sub for Real-time Notifications

```java
// Publisher
public class NotificationPublisher {
    
    public void sendNotification(String channel, String message) {
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.publish(channel, message);
        }
    }
}

// Subscriber
public class NotificationSubscriber extends JedisPubSub {
    
    @Override
    public void onMessage(String channel, String message) {
        System.out.println("Received notification on " + channel + ": " + message);
        
        // Process notification (send email, push notification, etc.)
        processNotification(channel, message);
    }
    
    public void subscribe() {
        try (Jedis jedis = jedisPool.getResource()) {
            jedis.subscribe(this, "user:notifications", "system:alerts");
        }
    }
}
```

## Redis Clustering & High Availability

### 1. Redis Sentinel (High Availability)

```
Master-Slave Setup with Sentinel:

┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Sentinel 1 │    │  Sentinel 2 │    │  Sentinel 3 │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
    │   Master    │───▶│   Slave 1   │    │   Slave 2   │
    │ (Write/Read)│    │ (Read Only) │    │ (Read Only) │
    └─────────────┘    └─────────────┘    └─────────────┘
```

### 2. Redis Cluster (Horizontal Scaling)

```
Redis Cluster (6 nodes):

Node 1: Slots 0-5460     (Master A + Slave A')
Node 2: Slots 5461-10922 (Master B + Slave B')  
Node 3: Slots 10923-16383(Master C + Slave C')

Hash Slot = CRC16(key) % 16384
```

## Redis Performance Optimization

### 1. Memory Optimization

```redis
# Use appropriate data structures
# Instead of: SET user:1001:name "John"
# Use hash:   HSET user:1001 name "John"

# Configure memory policies
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru
```

### 2. Pipeline for Bulk Operations

```java
// Without pipeline (slow)
for (int i = 0; i < 1000; i++) {
    jedis.set("key" + i, "value" + i);
}

// With pipeline (fast)
Pipeline pipeline = jedis.pipelined();
for (int i = 0; i < 1000; i++) {
    pipeline.set("key" + i, "value" + i);
}
pipeline.sync();
```

---

# ELASTICSEARCH - SEARCH & ANALYTICS ENGINE {#elasticsearch}

## What is Elasticsearch?

Elasticsearch is a **distributed search and analytics engine** built on Apache Lucene. It's designed for horizontal scalability, reliability, and real-time search capabilities.

### Key Concepts:

```
Document → Index → Shard → Node → Cluster
    ↓        ↓       ↓       ↓       ↓
  JSON     Table  Partition Server  Database
```

**Core Components:**
- **Document**: JSON object (like a database row)
- **Index**: Collection of documents (like a database table)
- **Shard**: Subset of index data (for distribution)
- **Node**: Single Elasticsearch instance
- **Cluster**: Group of nodes working together

## Elasticsearch Architecture

### 1. Index Structure

```
Index: "products"
┌─────────────────────────────────────────────────────────┐
│ Primary Shard 0: [doc1, doc4, doc7, ...]               │
│ Primary Shard 1: [doc2, doc5, doc8, ...]               │  
│ Primary Shard 2: [doc3, doc6, doc9, ...]               │
└─────────────────────────────────────────────────────────┘

Each shard has replica shards on different nodes for fault tolerance
```

### 2. Search Process

```
Search Request Flow:

Client → Coordinating Node → Query Phase → Fetch Phase → Response
           │                     │            │
           │                     ▼            ▼
           └─────────────→ All Shards → Top Documents
```

## Elasticsearch Implementation Examples

### 1. Index Creation with Mapping

```json
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "product_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "snowball"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "product_analyzer",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "product_analyzer"
      },
      "price": {
        "type": "double"
      },
      "category": {
        "type": "keyword"
      },
      "tags": {
        "type": "keyword"
      },
      "created_at": {
        "type": "date"
      },
      "location": {
        "type": "geo_point"
      }
    }
  }
}
```

### 2. Document Indexing

```java
// Elasticsearch Java client
RestHighLevelClient client = new RestHighLevelClient(
    RestClient.builder(new HttpHost("localhost", 9200, "http"))
);

public class ProductSearchService {
    
    public void indexProduct(Product product) throws IOException {
        IndexRequest request = new IndexRequest("products")
            .id(product.getId().toString())
            .source(XContentType.JSON,
                "name", product.getName(),
                "description", product.getDescription(),
                "price", product.getPrice(),
                "category", product.getCategory(),
                "tags", product.getTags(),
                "created_at", product.getCreatedAt()
            );
            
        IndexResponse response = client.index(request, RequestOptions.DEFAULT);
        System.out.println("Indexed product: " + response.getId());
    }
    
    public void bulkIndexProducts(List<Product> products) throws IOException {
        BulkRequest bulkRequest = new BulkRequest();
        
        for (Product product : products) {
            IndexRequest indexRequest = new IndexRequest("products")
                .id(product.getId().toString())
                .source(convertToJson(product), XContentType.JSON);
            bulkRequest.add(indexRequest);
        }
        
        BulkResponse bulkResponse = client.bulk(bulkRequest, RequestOptions.DEFAULT);
        System.out.println("Bulk indexed " + products.size() + " products");
    }
}
```

### 3. Complex Search Queries

```java
public class SearchService {
    
    public SearchResponse searchProducts(String query, String category, 
                                       Double minPrice, Double maxPrice) throws IOException {
        
        SearchRequest searchRequest = new SearchRequest("products");
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        
        // Build complex query
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        
        // Text search with boosting
        if (query != null && !query.isEmpty()) {
            MultiMatchQueryBuilder multiMatchQuery = QueryBuilders
                .multiMatchQuery(query, "name^2", "description")
                .type(MultiMatchQueryBuilder.Type.BEST_FIELDS)
                .fuzziness(Fuzziness.AUTO);
            boolQuery.must(multiMatchQuery);
        }
        
        // Category filter
        if (category != null) {
            boolQuery.filter(QueryBuilders.termQuery("category", category));
        }
        
        // Price range filter
        if (minPrice != null || maxPrice != null) {
            RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("price");
            if (minPrice != null) rangeQuery.gte(minPrice);
            if (maxPrice != null) rangeQuery.lte(maxPrice);
            boolQuery.filter(rangeQuery);
        }
        
        searchSourceBuilder.query(boolQuery);
        
        // Add aggregations
        searchSourceBuilder.aggregation(
            AggregationBuilders.terms("categories").field("category").size(10)
        );
        searchSourceBuilder.aggregation(
            AggregationBuilders.histogram("price_ranges").field("price").interval(50)
        );
        
        // Sorting and pagination
        searchSourceBuilder.sort("_score", SortOrder.DESC);
        searchSourceBuilder.sort("price", SortOrder.ASC);
        searchSourceBuilder.from(0).size(20);
        
        // Highlighting
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.field("name").field("description");
        searchSourceBuilder.highlighter(highlightBuilder);
        
        searchRequest.source(searchSourceBuilder);
        
        return client.search(searchRequest, RequestOptions.DEFAULT);
    }
}
```

### 4. Real-time Analytics with Aggregations

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "sales_over_time": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "day"
      },
      "aggs": {
        "total_revenue": {
          "sum": {
            "field": "total_amount"
          }
        },
        "avg_order_value": {
          "avg": {
            "field": "total_amount"
          }
        }
      }
    },
    "top_categories": {
      "terms": {
        "field": "category",
        "size": 10
      },
      "aggs": {
        "revenue": {
          "sum": {
            "field": "total_amount"
          }
        }
      }
    },
    "geographic_distribution": {
      "geo_distance": {
        "field": "customer_location",
        "origin": "40.7128,-74.0060",
        "ranges": [
          { "to": 10 },
          { "from": 10, "to": 50 },
          { "from": 50 }
        ]
      }
    }
  }
}
```

## Elasticsearch Performance Optimization

### 1. Index Optimization

```json
PUT /products/_settings
{
  "index": {
    "refresh_interval": "30s",
    "number_of_replicas": 1,
    "translog.flush_threshold_size": "1gb",
    "merge.policy.max_merged_segment": "5gb"
  }
}
```

### 2. Query Optimization

```java
// Use filters instead of queries when possible (cached)
BoolQueryBuilder boolQuery = QueryBuilders.boolQuery()
    .must(QueryBuilders.matchQuery("description", "smartphone"))  // Scored
    .filter(QueryBuilders.termQuery("category", "electronics"))   // Cached
    .filter(QueryBuilders.rangeQuery("price").gte(100).lte(1000)); // Cached
```

### 3. Monitoring and Alerting

```java
public class ElasticsearchMonitor {
    
    public ClusterHealthStatus getClusterHealth() throws IOException {
        ClusterHealthRequest request = new ClusterHealthRequest();
        ClusterHealthResponse response = client.cluster().health(request, RequestOptions.DEFAULT);
        return response.getStatus();
    }
    
    public void checkIndexHealth(String indexName) throws IOException {
        IndicesStatsRequest request = new IndicesStatsRequest();
        request.indices(indexName);
        IndicesStatsResponse response = client.indices().stats(request, RequestOptions.DEFAULT);
        
        // Check metrics
        long docCount = response.getTotal().getDocs().getCount();
        long storeSize = response.getTotal().getStore().getSizeInBytes();
        
        System.out.println("Index: " + indexName);
        System.out.println("Documents: " + docCount);
        System.out.println("Size: " + storeSize + " bytes");
    }
}
```

---

# INTEGRATION PATTERNS & BEST PRACTICES {#integration}

## 1. Event-Driven E-commerce Architecture

```
┌─────────────┐    Kafka     ┌─────────────┐    Redis      ┌─────────────┐
│   Order     │─────────────▶│  Inventory  │──────────────▶│   Cache     │
│  Service    │              │   Service   │               │  Invalidate │
└─────────────┘              └─────────────┘               └─────────────┘
       │                            │
       │ Kafka                      │ Elasticsearch
       ▼                            ▼
┌─────────────┐              ┌─────────────┐
│  Payment    │              │   Search    │
│  Service    │              │   Index     │
└─────────────┘              └─────────────┘
```

### Implementation:

```java
@Service
public class OrderService {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public Order createOrder(OrderRequest request) {
        // 1. Create order
        Order order = new Order(request);
        orderRepository.save(order);
        
        // 2. Publish event to Kafka
        OrderEvent event = new OrderEvent(order.getId(), order.getItems());
        kafkaTemplate.send("order-events", order.getId().toString(), 
                          JSON.toJSONString(event));
        
        // 3. Cache order for quick access
        redisTemplate.opsForValue().set("order:" + order.getId(), order, 
                                       Duration.ofHours(1));
        
        return order;
    }
}

@KafkaListener(topics = "order-events")
public class InventoryService {
    
    public void handleOrderEvent(String orderData) {
        OrderEvent event = JSON.parseObject(orderData, OrderEvent.class);
        
        // Update inventory
        for (OrderItem item : event.getItems()) {
            updateInventory(item.getProductId(), -item.getQuantity());
        }
        
        // Publish inventory update event
        kafkaTemplate.send("inventory-events", event.getOrderId(), 
                          "inventory-updated");
    }
}
```

## 2. CQRS with Elasticsearch

```java
// Command side (writes)
@Service
public class ProductCommandService {
    
    public void createProduct(Product product) {
        // Save to primary database
        productRepository.save(product);
        
        // Publish event for read model update
        ProductCreatedEvent event = new ProductCreatedEvent(product);
        kafkaTemplate.send("product-events", product.getId().toString(), 
                          JSON.toJSONString(event));
    }
}

// Query side (reads)
@KafkaListener(topics = "product-events")
@Service
public class ProductQueryService {
    
    public void handleProductCreated(String eventData) {
        ProductCreatedEvent event = JSON.parseObject(eventData, ProductCreatedEvent.class);
        
        // Index in Elasticsearch for search
        indexProductInElasticsearch(event.getProduct());
        
        // Cache in Redis for quick access
        cacheProductInRedis(event.getProduct());
    }
    
    public List<Product> searchProducts(String query) {
        // Search from Elasticsearch
        return elasticsearchService.search(query);
    }
    
    public Product getProduct(String id) {
        // Try Redis cache first
        Product product = redisService.get("product:" + id, Product.class);
        if (product != null) {
            return product;
        }
        
        // Fallback to Elasticsearch
        return elasticsearchService.getById(id);
    }
}
```

## 3. Distributed Session Management

```java
@Component
public class DistributedSessionManager {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void createSession(String sessionId, String userId) {
        SessionData session = new SessionData(userId, System.currentTimeMillis());
        
        // Store in Redis with TTL
        redisTemplate.opsForValue().set("session:" + sessionId, session, 
                                       Duration.ofMinutes(30));
        
        // Publish session created event
        SessionEvent event = new SessionEvent("CREATED", sessionId, userId);
        kafkaTemplate.send("session-events", sessionId, JSON.toJSONString(event));
    }
    
    public boolean validateSession(String sessionId) {
        SessionData session = (SessionData) redisTemplate.opsForValue()
                                                         .get("session:" + sessionId);
        
        if (session != null) {
            // Extend session TTL
            redisTemplate.expire("session:" + sessionId, Duration.ofMinutes(30));
            return true;
        }
        
        return false;
    }
}
```

---

# INTERVIEW QUESTIONS & SCENARIOS {#interview}

## Kafka Interview Questions

### Q1: "How would you handle message ordering in Kafka?"

**Answer:**
```
Kafka guarantees ordering within a partition, not across partitions.

Solutions:
1. Single Partition: All messages go to one partition (limits scalability)
2. Keyed Messages: Use consistent key to route related messages to same partition
3. Sequence Numbers: Add sequence numbers and handle out-of-order at consumer

Example:
- User events: Use userId as key → all events for user go to same partition
- Order processing: Use orderId as key → all order updates stay ordered
```

### Q2: "How do you prevent message loss in Kafka?"

**Answer:**
```
Producer Side:
- acks=all (wait for all replicas)
- retries > 0
- enable.idempotence=true

Broker Side:  
- replication.factor >= 3
- min.insync.replicas >= 2
- unclean.leader.election.enable=false

Consumer Side:
- enable.auto.commit=false
- Manual offset commit after processing
- Proper error handling and retry logic
```

### Q3: "Design a real-time analytics system using Kafka"

**Answer:**
```
Architecture:
Web App → Kafka → Kafka Streams → Elasticsearch → Dashboard

Implementation:
1. Events: User clicks, page views, purchases
2. Kafka Topics: raw-events, processed-events, alerts
3. Kafka Streams: Windowed aggregations, filtering
4. Output: Real-time metrics, anomaly detection

Code Example:
KStream<String, UserEvent> events = builder.stream("user-events");
KTable<Windowed<String>, Long> clickCounts = events
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count();
```

## Redis Interview Questions

### Q4: "How would you implement a distributed lock using Redis?"

**Answer:**
```java
public class DistributedLock {
    
    public boolean acquireLock(String lockKey, String clientId, int ttlSeconds) {
        try (Jedis jedis = jedisPool.getResource()) {
            String result = jedis.set(lockKey, clientId, "NX", "EX", ttlSeconds);
            return "OK".equals(result);
        }
    }
    
    public boolean releaseLock(String lockKey, String clientId) {
        String script = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
            
        try (Jedis jedis = jedisPool.getResource()) {
            Object result = jedis.eval(script, 1, lockKey, clientId);
            return Long.valueOf(1).equals(result);
        }
    }
}
```

### Q5: "Design a leaderboard system that can handle millions of users"

**Answer:**
```
Use Redis Sorted Sets:

ZADD leaderboard:global score1 user1 score2 user2 ...
ZREVRANGE leaderboard:global 0 99 WITHSCORES  # Top 100

For scale:
1. Sharding: Multiple leaderboards by region/category
2. Caching: Cache top N results
3. Batch Updates: Use pipeline for bulk score updates
4. Time-based: Separate leaderboards for daily/weekly/monthly

Memory optimization:
- Use hash tags for consistent sharding: leaderboard:{region}:global
- Implement score bucketing for very large datasets
```

### Q6: "How do you handle Redis failover and high availability?"

**Answer:**
```
Redis Sentinel Setup:
1. Master-Slave replication
2. Sentinel nodes monitor master health
3. Automatic failover when master fails
4. Client libraries handle failover transparently

Redis Cluster:
1. Horizontal scaling across multiple masters
2. Hash slot distribution (16384 slots)
3. Automatic resharding and rebalancing
4. Each master has replica for fault tolerance

Client-side considerations:
- Connection pooling
- Retry logic with exponential backoff
- Circuit breaker pattern
- Graceful degradation when Redis unavailable
```

## Elasticsearch Interview Questions

### Q7: "How would you design search for an e-commerce platform?"

**Answer:**
```
Index Design:
1. Products index with proper mapping
2. Multi-field analysis (text + keyword)
3. Nested objects for variants/attributes
4. Geo-point for location-based search

Search Features:
1. Full-text search with relevance scoring
2. Faceted search (filters, aggregations)
3. Auto-complete and suggestions
4. Personalized ranking
5. Search analytics and optimization

Performance:
1. Index optimization (refresh interval, replicas)
2. Query optimization (filters vs queries)
3. Caching strategies
4. Monitoring and alerting

Example mapping:
{
  "name": {"type": "text", "analyzer": "standard"},
  "category": {"type": "keyword"},
  "price": {"type": "double"},
  "attributes": {"type": "nested"}
}
```

### Q8: "How do you handle Elasticsearch cluster scaling?"

**Answer:**
```
Horizontal Scaling:
1. Add more nodes to cluster
2. Elasticsearch automatically redistributes shards
3. Increase replica count for read scaling
4. Use index templates for consistent settings

Shard Strategy:
1. Calculate optimal shard count: (Total Data Size) / (Desired Shard Size)
2. Avoid over-sharding (too many small shards)
3. Consider hot-warm-cold architecture for time-series data

Monitoring:
1. Cluster health (green/yellow/red)
2. Node resource utilization
3. Query performance metrics
4. Index size and growth trends

Best Practices:
- Use dedicated master nodes for large clusters
- Separate data and ingest nodes
- Monitor heap usage and GC performance
```

### Q9: "Design a log aggregation system using ELK stack"

**Answer:**
```
Architecture:
Applications → Filebeat → Kafka → Logstash → Elasticsearch → Kibana

Flow:
1. Filebeat: Collect logs from application servers
2. Kafka: Buffer and distribute log events
3. Logstash: Parse, transform, and enrich logs
4. Elasticsearch: Index and store processed logs
5. Kibana: Visualize and analyze logs

Logstash Configuration:
input {
  kafka {
    topics => ["application-logs"]
    bootstrap_servers => "kafka:9092"
  }
}

filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:message}" }
  }
  
  date {
    match => [ "timestamp", "ISO8601" ]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
```

## System Design Scenarios

### Q10: "Design a real-time chat system using these technologies"

**Answer:**
```
Architecture:
WebSocket → API Gateway → Chat Service → Kafka → Redis → Elasticsearch

Components:
1. Kafka: Message delivery and persistence
2. Redis: Online user presence, recent messages cache
3. Elasticsearch: Message search and history

Message Flow:
1. User sends message via WebSocket
2. Chat service publishes to Kafka topic
3. Kafka delivers to all subscribers
4. Cache recent messages in Redis
5. Index messages in Elasticsearch for search

Implementation:
- Kafka topics per chat room for scalability
- Redis pub/sub for real-time notifications
- Elasticsearch for message search across history
- WebSocket connection management with Redis
```

### Q11: "Design a recommendation system with real-time updates"

**Answer:**
```
Architecture:
User Events → Kafka → Stream Processing → ML Model → Redis Cache → API

Real-time Pipeline:
1. Kafka Streams: Process user behavior events
2. Feature Store: Update user/item features
3. Model Serving: Generate recommendations
4. Redis: Cache recommendations with TTL

Batch Pipeline:
1. Elasticsearch: Store user interaction history
2. Spark/Kafka: Batch feature engineering
3. ML Training: Update recommendation models
4. Model Deployment: Deploy new models

Serving:
- Redis: Serve cached recommendations (< 10ms)
- Fallback: Real-time model inference (< 100ms)
- Cold start: Popular items from Elasticsearch
```

This comprehensive guide covers the essential knowledge needed for HLD interviews. Each technology is explained with practical implementations and real-world scenarios that commonly appear in system design discussions.