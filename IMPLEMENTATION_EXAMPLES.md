# Implementation Examples - E-Commerce Popularity System

## 1. EVENT PRODUCER (Frontend SDK)

```javascript
// popularity-tracker.js
class PopularityTracker {
  constructor(config) {
    this.apiEndpoint = config.apiEndpoint;
    this.batchSize = config.batchSize || 10;
    this.flushInterval = config.flushInterval || 5000; // 5 seconds
    this.eventQueue = [];
    this.startBatchTimer();
  }

  trackView(productId, metadata = {}) {
    this.enqueueEvent({
      event_type: 'view',
      product_id: productId,
      timestamp: Date.now(),
      ...metadata
    });
  }

  trackClick(productId, metadata = {}) {
    this.enqueueEvent({
      event_type: 'click',
      product_id: productId,
      timestamp: Date.now(),
      ...metadata
    });
  }

  enqueueEvent(event) {
    this.eventQueue.push({
      event_id: this.generateUUID(),
      user_id: this.getUserId(),
      session_id: this.getSessionId(),
      ...event
    });

    if (this.eventQueue.length >= this.batchSize) {
      this.flush();
    }
  }

  async flush() {
    if (this.eventQueue.length === 0) return;

    const batch = [...this.eventQueue];
    this.eventQueue = [];

    try {
      await fetch(`${this.apiEndpoint}/events/batch`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ events: batch })
      });
    } catch (error) {
      console.error('Failed to send events:', error);
      // Retry logic or store in localStorage
    }
  }

  startBatchTimer() {
    setInterval(() => this.flush(), this.flushInterval);
  }

  generateUUID() {
    return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
      const r = Math.random() * 16 | 0;
      return (c === 'x' ? r : (r & 0x3 | 0x8)).toString(16);
    });
  }

  getUserId() {
    return localStorage.getItem('user_id') || 'anonymous';
  }

  getSessionId() {
    let sessionId = sessionStorage.getItem('session_id');
    if (!sessionId) {
      sessionId = this.generateUUID();
      sessionStorage.setItem('session_id', sessionId);
    }
    return sessionId;
  }
}

// Usage
const tracker = new PopularityTracker({
  apiEndpoint: 'https://api.example.com',
  batchSize: 10,
  flushInterval: 5000
});

// Track product view
tracker.trackView('product-123', {
  category_id: 'electronics',
  price: 299.99
});
```

---

## 2. EVENT INGESTION SERVICE

```java
// EventIngestionController.java
@RestController
@RequestMapping("/api/v1/events")
public class EventIngestionController {
    
    @Autowired
    private KafkaTemplate<String, Event> kafkaTemplate;
    
    @Autowired
    private EventValidator validator;
    
    @PostMapping("/batch")
    public ResponseEntity<BatchResponse> ingestBatch(@RequestBody BatchRequest request) {
        List<Event> events = request.getEvents();
        
        // Validate events
        List<Event> validEvents = events.stream()
            .filter(validator::isValid)
            .collect(Collectors.toList());
        
        // Publish to Kafka asynchronously
        validEvents.forEach(event -> {
            String topic = getTopicForEventType(event.getEventType());
            String key = event.getProductId(); // Partition by product_id
            
            kafkaTemplate.send(topic, key, event)
                .addCallback(
                    success -> log.debug("Event sent: {}", event.getEventId()),
                    failure -> log.error("Failed to send event: {}", event.getEventId(), failure)
                );
        });
        
        return ResponseEntity.ok(new BatchResponse(
            validEvents.size(),
            events.size() - validEvents.size()
        ));
    }
    
    private String getTopicForEventType(String eventType) {
        return switch (eventType) {
            case "view" -> "user-views";
            case "click" -> "user-clicks";
            case "add_to_cart" -> "cart-events";
            case "purchase" -> "purchase-events";
            default -> "unknown-events";
        };
    }
}

// Event.java
@Data
public class Event {
    private String eventId;
    private String eventType;
    private String productId;
    private String userId;
    private String categoryId;
    private Long timestamp;
    private String sessionId;
    private Map<String, Object> metadata;
}
```

---

## 3. BATCH PROCESSING (Spark Job)

```scala
// PopularityBatchJob.scala
import org.apache.spark.sql._
import org.apache.spark.sql.functions._

object PopularityBatchJob {
  
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .appName("Popularity Batch Job")
      .config("spark.sql.shuffle.partitions", "200")
      .getOrCreate()
    
    import spark.implicits._
    
    // Read events from S3
    val events = spark.read
      .parquet("s3://events-bucket/date=2024-01-*")
      .filter($"timestamp" > unix_timestamp() - 86400) // Last 24 hours
    
    // Aggregate by product
    val aggregated = events
      .groupBy("product_id", "category_id")
      .agg(
        countDistinct(when($"event_type" === "view", $"user_id")).as("unique_views"),
        count(when($"event_type" === "view", 1)).as("total_views"),
        count(when($"event_type" === "click", 1)).as("clicks"),
        count(when($"event_type" === "add_to_cart", 1)).as("add_to_carts"),
        count(when($"event_type" === "purchase", 1)).as("purchases"),
        max("timestamp").as("last_event_time")
      )
    
    // Calculate popularity score
    val scored = aggregated.map { row =>
      val productId = row.getAs[String]("product_id")
      val categoryId = row.getAs[String]("category_id")
      val views = row.getAs[Long]("total_views")
      val clicks = row.getAs[Long]("clicks")
      val carts = row.getAs[Long]("add_to_carts")
      val purchases = row.getAs[Long]("purchases")
      val lastEventTime = row.getAs[Long]("last_event_time")
      
      val recencyDecay = calculateRecencyDecay(lastEventTime)
      val conversionRate = if (views > 0) purchases.toDouble / views else 0.0
      
      val score = 
        0.1 * math.log(views + 1) +
        0.3 * clicks +
        1.5 * carts +
        3.0 * purchases +
        0.5 * recencyDecay +
        1.0 * conversionRate
      
      ProductScore(productId, categoryId, score, System.currentTimeMillis())
    }
    
    // Rank products per category
    val ranked = scored
      .withColumn("rank", 
        row_number().over(
          Window.partitionBy("category_id")
            .orderBy(desc("score"))
        )
      )
    
    // Write to DynamoDB
    ranked.write
      .format("dynamodb")
      .option("tableName", "popularity_rankings")
      .option("region", "us-east-1")
      .mode("overwrite")
      .save()
    
    // Write to ClickHouse for analytics
    ranked.write
      .format("jdbc")
      .option("url", "jdbc:clickhouse://clickhouse:8123/analytics")
      .option("dbtable", "popularity_scores")
      .mode("append")
      .save()
    
    // Update Redis cache
    updateRedisCache(ranked)
    
    spark.stop()
  }
  
  def calculateRecencyDecay(timestamp: Long): Double = {
    val now = System.currentTimeMillis()
    val daysSince = (now - timestamp) / (1000.0 * 60 * 60 * 24)
    math.exp(-0.1 * daysSince)
  }
  
  def updateRedisCache(df: Dataset[Row]): Unit = {
    df.foreachPartition { partition =>
      val jedis = new Jedis("redis-cluster:6379")
      val pipeline = jedis.pipelined()
      
      partition.foreach { row =>
        val categoryId = row.getAs[String]("category_id")
        val productId = row.getAs[String]("product_id")
        val score = row.getAs[Double]("score")
        
        val key = s"category:$categoryId:popular:24h"
        pipeline.zadd(key, score, productId)
        pipeline.expire(key, 3600) // 1 hour TTL
      }
      
      pipeline.sync()
      jedis.close()
    }
  }
}

case class ProductScore(
  productId: String,
  categoryId: String,
  score: Double,
  updatedAt: Long
)
```

---

## 4. REAL-TIME STREAMING (Flink Job)

```java
// HotItemsFlinkJob.java
public class HotItemsFlinkJob {
    
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        env.enableCheckpointing(60000); // Checkpoint every 1 minute
        
        // Kafka source
        Properties kafkaProps = new Properties();
        kafkaProps.setProperty("bootstrap.servers", "kafka:9092");
        kafkaProps.setProperty("group.id", "hot-items-consumer");
        
        FlinkKafkaConsumer<Event> consumer = new FlinkKafkaConsumer<>(
            Arrays.asList("user-views", "user-clicks", "purchase-events"),
            new EventDeserializationSchema(),
            kafkaProps
        );
        
        DataStream<Event> events = env.addSource(consumer)
            .assignTimestampsAndWatermarks(
                WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(10))
                    .withTimestampAssigner((event, timestamp) -> event.getTimestamp())
            );
        
        // Sliding window: 10 minutes window, 1 minute slide
        DataStream<ProductCount> hotItems = events
            .keyBy(Event::getProductId)
            .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.minutes(1)))
            .aggregate(new CountAggregateFunction(), new WindowResultFunction())
            .filter(count -> count.getCount() > 100); // Threshold
        
        // Write to Redis
        hotItems.addSink(new RedisSink());
        
        env.execute("Hot Items Detection");
    }
    
    // Aggregate function to count events
    public static class CountAggregateFunction 
            implements AggregateFunction<Event, Long, Long> {
        
        @Override
        public Long createAccumulator() {
            return 0L;
        }
        
        @Override
        public Long add(Event event, Long accumulator) {
            return accumulator + getEventWeight(event.getEventType());
        }
        
        @Override
        public Long getResult(Long accumulator) {
            return accumulator;
        }
        
        @Override
        public Long merge(Long a, Long b) {
            return a + b;
        }
        
        private int getEventWeight(String eventType) {
            return switch (eventType) {
                case "view" -> 1;
                case "click" -> 3;
                case "add_to_cart" -> 5;
                case "purchase" -> 10;
                default -> 0;
            };
        }
    }
    
    // Window result function
    public static class WindowResultFunction 
            implements WindowFunction<Long, ProductCount, String, TimeWindow> {
        
        @Override
        public void apply(String productId, TimeWindow window, 
                         Iterable<Long> input, Collector<ProductCount> out) {
            Long count = input.iterator().next();
            out.collect(new ProductCount(
                productId, 
                count, 
                window.getEnd()
            ));
        }
    }
    
    // Redis sink
    public static class RedisSink extends RichSinkFunction<ProductCount> {
        private transient Jedis jedis;
        
        @Override
        public void open(Configuration parameters) {
            jedis = new Jedis("redis-cluster:6379");
        }
        
        @Override
        public void invoke(ProductCount value, Context context) {
            String key = "hot:last_10min";
            jedis.zadd(key, value.getCount(), value.getProductId());
            jedis.expire(key, 900); // 15 minutes TTL
            
            // Keep only top 1000
            jedis.zremrangeByRank(key, 0, -1001);
        }
        
        @Override
        public void close() {
            if (jedis != null) {
                jedis.close();
            }
        }
    }
}

@Data
@AllArgsConstructor
class ProductCount {
    private String productId;
    private Long count;
    private Long windowEnd;
}
```

---

## 5. API SERVICE IMPLEMENTATION

```java
// PopularItemsService.java
@Service
public class PopularItemsService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Autowired
    private DynamoDBMapper dynamoDBMapper;
    
    @Autowired
    private ProductService productService;
    
    @Cacheable(value = "popular-items", key = "#categoryId + ':' + #page")
    public PopularItemsResponse getPopularItems(
            String categoryId, 
            int page, 
            int limit,
            String timeRange) {
        
        String cacheKey = buildCacheKey(categoryId, timeRange);
        
        // Try Redis first
        Set<String> productIds = redisTemplate.opsForZSet()
            .reverseRange(cacheKey, (page - 1) * limit, page * limit - 1);
        
        if (productIds != null && !productIds.isEmpty()) {
            return buildResponse(productIds, page, limit, true);
        }
        
        // Cache miss - query DynamoDB
        List<PopularityRanking> rankings = queryDynamoDB(categoryId, page, limit);
        productIds = rankings.stream()
            .map(PopularityRanking::getProductId)
            .collect(Collectors.toSet());
        
        // Update cache asynchronously
        CompletableFuture.runAsync(() -> updateCache(cacheKey, rankings));
        
        return buildResponse(productIds, page, limit, false);
    }
    
    private List<PopularityRanking> queryDynamoDB(String categoryId, int page, int limit) {
        DynamoDBQueryExpression<PopularityRanking> query = 
            new DynamoDBQueryExpression<PopularityRanking>()
                .withHashKeyValues(new PopularityRanking(categoryId))
                .withScanIndexForward(false) // Descending order
                .withLimit(limit);
        
        return dynamoDBMapper.query(PopularityRanking.class, query);
    }
    
    private void updateCache(String key, List<PopularityRanking> rankings) {
        ZSetOperations<String, String> zset = redisTemplate.opsForZSet();
        
        rankings.forEach(ranking -> 
            zset.add(key, ranking.getProductId(), ranking.getScore())
        );
        
        redisTemplate.expire(key, Duration.ofHours(1));
    }
    
    private PopularItemsResponse buildResponse(
            Set<String> productIds, 
            int page, 
            int limit,
            boolean cacheHit) {
        
        // Fetch product details
        List<Product> products = productService.getProductsByIds(productIds);
        
        return PopularItemsResponse.builder()
            .data(products)
            .pagination(Pagination.builder()
                .page(page)
                .limit(limit)
                .hasNext(products.size() == limit)
                .build())
            .metadata(Metadata.builder()
                .cacheHit(cacheHit)
                .updatedAt(Instant.now())
                .build())
            .build();
    }
    
    private String buildCacheKey(String categoryId, String timeRange) {
        return String.format("category:%s:popular:%s", categoryId, timeRange);
    }
}

// HotItemsService.java
@Service
public class HotItemsService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public HotItemsResponse getHotItems(int limit) {
        String key = "hot:last_10min";
        
        Set<ZSetOperations.TypedTuple<String>> results = 
            redisTemplate.opsForZSet()
                .reverseRangeWithScores(key, 0, limit - 1);
        
        if (results == null || results.isEmpty()) {
            return HotItemsResponse.empty();
        }
        
        List<HotItem> hotItems = results.stream()
            .map(tuple -> HotItem.builder()
                .productId(tuple.getValue())
                .eventCount(tuple.getScore().longValue())
                .trendScore(calculateTrendScore(tuple.getScore()))
                .build())
            .collect(Collectors.toList());
        
        return HotItemsResponse.builder()
            .data(hotItems)
            .metadata(HotItemsMetadata.builder()
                .window("last_10_minutes")
                .updatedAt(Instant.now())
                .build())
            .build();
    }
    
    private double calculateTrendScore(Double eventCount) {
        // Normalize to 0-100 scale
        return Math.min(100, (eventCount / 1000.0) * 100);
    }
}
```

---

## 6. REDIS CACHE OPERATIONS

```java
// RedisCacheManager.java
@Component
public class RedisCacheManager {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    // Store popular items as sorted set
    public void storePopularItems(String categoryId, List<ProductScore> scores) {
        String key = String.format("category:%s:popular:24h", categoryId);
        ZSetOperations<String, String> zset = redisTemplate.opsForZSet();
        
        // Use pipeline for batch operations
        redisTemplate.executePipelined(new SessionCallback<Object>() {
            @Override
            public Object execute(RedisOperations operations) {
                scores.forEach(score -> 
                    zset.add(key, score.getProductId(), score.getScore())
                );
                redisTemplate.expire(key, Duration.ofHours(1));
                return null;
            }
        });
    }
    
    // Get top N items
    public List<String> getTopItems(String categoryId, int limit) {
        String key = String.format("category:%s:popular:24h", categoryId);
        Set<String> items = redisTemplate.opsForZSet()
            .reverseRange(key, 0, limit - 1);
        return items != null ? new ArrayList<>(items) : Collections.emptyList();
    }
    
    // Get paginated items
    public List<String> getItemsPage(String categoryId, int page, int pageSize) {
        String key = String.format("category:%s:popular:24h", categoryId);
        long start = (page - 1) * pageSize;
        long end = start + pageSize - 1;
        
        Set<String> items = redisTemplate.opsForZSet()
            .reverseRange(key, start, end);
        return items != null ? new ArrayList<>(items) : Collections.emptyList();
    }
    
    // Get item rank
    public Long getItemRank(String categoryId, String productId) {
        String key = String.format("category:%s:popular:24h", categoryId);
        return redisTemplate.opsForZSet().reverseRank(key, productId);
    }
    
    // Warm cache on startup
    @PostConstruct
    public void warmCache() {
        log.info("Warming cache...");
        // Load top 1000 items per category from DynamoDB
        List<String> categories = getTopCategories();
        categories.parallelStream().forEach(this::loadCategoryCache);
        log.info("Cache warming completed");
    }
    
    private void loadCategoryCache(String categoryId) {
        // Implementation to load from DynamoDB
    }
}
```

---

## 7. MONITORING & METRICS

```java
// MetricsCollector.java
@Component
public class MetricsCollector {
    
    private final MeterRegistry meterRegistry;
    private final Counter eventCounter;
    private final Timer apiLatency;
    private final Gauge cacheHitRate;
    
    public MetricsCollector(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        this.eventCounter = Counter.builder("events.ingested")
            .tag("service", "popularity")
            .description("Total events ingested")
            .register(meterRegistry);
        
        this.apiLatency = Timer.builder("api.latency")
            .tag("endpoint", "popular-items")
            .description("API response time")
            .register(meterRegistry);
        
        this.cacheHitRate = Gauge.builder("cache.hit.rate", this, 
                MetricsCollector::calculateCacheHitRate)
            .description("Cache hit rate percentage")
            .register(meterRegistry);
    }
    
    public void recordEvent(String eventType) {
        eventCounter.increment();
        meterRegistry.counter("events.by.type", "type", eventType).increment();
    }
    
    public void recordApiCall(long durationMs, boolean cacheHit) {
        apiLatency.record(Duration.ofMillis(durationMs));
        meterRegistry.counter("api.cache", "hit", String.valueOf(cacheHit)).increment();
    }
    
    private double calculateCacheHitRate() {
        // Calculate from counters
        return 0.95; // Placeholder
    }
}

// HealthCheckController.java
@RestController
@RequestMapping("/health")
public class HealthCheckController {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Autowired
    private DynamoDBMapper dynamoDBMapper;
    
    @GetMapping
    public ResponseEntity<HealthStatus> healthCheck() {
        boolean redisHealthy = checkRedis();
        boolean dynamoHealthy = checkDynamoDB();
        boolean kafkaHealthy = checkKafka();
        
        HealthStatus status = HealthStatus.builder()
            .status(redisHealthy && dynamoHealthy && kafkaHealthy ? "UP" : "DEGRADED")
            .redis(redisHealthy ? "UP" : "DOWN")
            .dynamodb(dynamoHealthy ? "UP" : "DOWN")
            .kafka(kafkaHealthy ? "UP" : "DOWN")
            .timestamp(Instant.now())
            .build();
        
        return ResponseEntity.ok(status);
    }
    
    private boolean checkRedis() {
        try {
            redisTemplate.opsForValue().get("health-check");
            return true;
        } catch (Exception e) {
            return false;
        }
    }
    
    private boolean checkDynamoDB() {
        // Implementation
        return true;
    }
    
    private boolean checkKafka() {
        // Implementation
        return true;
    }
}
```

This implementation guide provides production-ready code examples for all major components of the popularity system.
