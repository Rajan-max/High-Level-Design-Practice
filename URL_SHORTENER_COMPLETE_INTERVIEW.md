# URL Shortener (Bit.ly/TinyURL) - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: High-Level Architecture (8 min)
Phase 5: API Design (3 min)
Phase 6: Data Models (4 min)
Phase 7: Core Algorithm - Short Code Generation (8 min)
Phase 8: Deep Dive - Scaling Reads (5 min)
Phase 9: Deep Dive - Distributed ID Generation (4 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"Thank you for the problem. I want to design a URL shortener like Bit.ly. Let me clarify the requirements and scope."

### Questions to Ask:

**Q1: Core Functionality**
- "Should users be able to create short URLs without authentication?"
- "Do we need user accounts and URL management?"

**Expected Answer:** Start with anonymous creation, accounts are follow-up

**Q2: Custom Aliases**
- "Can users specify custom aliases (e.g., bit.ly/my-custom-link)?"
- "What happens if the alias is already taken?"

**Expected Answer:** Yes, return error if taken

**Q3: Expiration**
- "Should short URLs expire after a certain time?"
- "Can users set custom expiration dates?"

**Expected Answer:** Optional expiration, default never expires

**Q4: Analytics - THE KEY QUESTION**
- "Do we need to track click analytics (count, location, device)?"
- "Should we show real-time analytics or batch processed?"

**Expected Answer:** Track clicks, real-time nice-to-have but not critical

**Q5: Scale**
- "How many URLs do we expect to shorten per day?"
- "What's the expected click-through rate?"
- "Read-to-write ratio?"

**Expected Answer:** 100M new URLs/day, 1000:1 read:write ratio

**Q6: URL Validation**
- "Should we validate that the long URL is accessible?"
- "Do we need to check for malicious URLs?"

**Expected Answer:** Basic validation, malicious detection is follow-up

**Q7: Redirect Type**
- "Should we use 301 (permanent) or 302 (temporary) redirects?"
- "Does the business need analytics on every click?"

**Expected Answer:** 302 for analytics, discuss trade-offs

**Q8: Short Code Length**
- "Any preference for short code length?"
- "Character set restrictions?"

**Expected Answer:** 6-7 characters, alphanumeric (a-z, A-Z, 0-9)

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### What to Say:

"Based on our discussion, let me summarize the requirements."

### Functional Requirements (Priority Order):

```
1. Shorten URL (CORE)
   - Accept long URL
   - Generate unique short code
   - Return shortened URL
   - Support custom aliases (optional)
   - Set expiration date (optional)

2. Redirect to Original URL (CORE)
   - Accept short code
   - Look up original URL
   - Redirect user (302)
   - Handle expired URLs

3. Analytics (IMPORTANT)
   - Track click count
   - Track timestamp
   - Track user agent (device type)
   - Track geographic location (IP-based)

4. URL Management (Nice-to-have)
   - List user's URLs
   - Delete short URL
   - Update expiration

Out of Scope:
- User authentication (mention briefly)
- QR code generation
- Link preview
- Spam/malicious URL detection
- Custom domains
```

### Non-Functional Requirements:

```
1. Performance
   - Redirect latency: < 100ms (p99)
   - URL creation: < 500ms
   - Support 10K redirects/second (peak)

2. Scalability
   - 1B total shortened URLs
   - 100M new URLs/day
   - 100B redirects/day (1000:1 read:write)

3. Availability
   - 99.99% uptime
   - Graceful degradation
   - No single point of failure

4. Uniqueness
   - Each short code maps to exactly one long URL
   - No collisions
   - Deterministic generation preferred

5. Durability
   - No data loss
   - URLs persist indefinitely (unless expired)

6. Simplicity
   - Easy to remember short codes
   - Short as possible (6-7 characters)
```

### Why This Matters:
✓ Read-heavy workload (1000:1 ratio)
✓ Uniqueness is critical
✓ Low latency is key for user experience
✓ Scale is massive (billions of URLs)

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me calculate the scale to inform our design decisions."

### Traffic Estimation:

```
Given:
- 100M new URLs created per day
- 1000:1 read-to-write ratio
- Peak traffic: 3x average

Calculations:

1. Write Operations (URL Creation)
   Daily writes: 100M URLs/day
   Average write QPS: 100M / 86,400 = ~1,160 writes/second
   Peak write QPS: 1,160 × 3 = ~3,500 writes/second

2. Read Operations (Redirects)
   Daily reads: 100M × 1000 = 100B redirects/day
   Average read QPS: 100B / 86,400 = ~1.16M reads/second
   Peak read QPS: 1.16M × 3 = ~3.5M reads/second
   
   This is MASSIVE read traffic!

3. Short Code Space
   Characters: a-z, A-Z, 0-9 = 62 characters (Base62)
   Length: 7 characters
   
   Total combinations: 62^7 = 3.5 trillion
   
   At 100M URLs/day:
   Years to exhaust: 3.5T / (100M × 365) = ~95 years
   
   6 characters: 62^6 = 56 billion (1.5 years)
   7 characters: 62^7 = 3.5 trillion (95 years) ✓
   8 characters: 62^8 = 218 trillion (5,972 years)
```

### Storage Estimation:

```
1. URL Mappings
   Total URLs: 1B (after 10 days)
   Each record: 500 bytes
   - short_code: 8 bytes
   - long_url: 200 bytes
   - user_id: 8 bytes
   - created_at: 8 bytes
   - expires_at: 8 bytes
   - click_count: 8 bytes
   - metadata: 260 bytes
   
   Total: 1B × 500 bytes = 500 GB

2. Analytics (Click Events)
   Daily clicks: 100B
   Retention: 30 days
   Total events: 100B × 30 = 3 trillion events
   
   Each event: 100 bytes
   - short_code: 8 bytes
   - timestamp: 8 bytes
   - ip_address: 16 bytes
   - user_agent: 50 bytes
   - referrer: 18 bytes
   
   Total: 3T × 100 bytes = 300 TB
   
   With compression: ~100 TB

Total Storage: ~100 TB (mostly analytics)
```

### Bandwidth Estimation:

```
1. Incoming (URL Creation)
   3,500 writes/sec × 200 bytes = 700 KB/sec = ~60 GB/day

2. Outgoing (Redirects)
   3.5M reads/sec × 500 bytes = 1.75 GB/sec = ~150 TB/day
   
   With caching (90% hit rate):
   150 TB × 0.1 = 15 TB/day
```

### Summary Table:

```
┌─────────────────────────┬──────────────────┐
│ Metric                  │ Value            │
├─────────────────────────┼──────────────────┤
│ New URLs/day            │ 100M             │
│ Total URLs (target)     │ 1B               │
│ Write QPS (peak)        │ 3,500            │
│ Read QPS (peak)         │ 3.5M             │
│ Read:Write ratio        │ 1000:1           │
│ Short code length       │ 7 chars          │
│ Short code space        │ 3.5 trillion     │
│ Storage                 │ 100 TB           │
│ Bandwidth (with cache)  │ 15 TB/day        │
└─────────────────────────┴──────────────────┘

Key Insights:
1. Reads dominate (1000:1 ratio) - caching critical
2. 7 characters sufficient for 95 years
3. Analytics storage >> URL storage
4. Need distributed system for 3.5M read QPS
```

### Why This Matters:
✓ Validates 7-character short codes
✓ Shows caching is mandatory
✓ Identifies analytics as storage bottleneck
✓ Demonstrates scale understanding

---

## PHASE 4: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"Let me present the high-level architecture. I'll separate the write path (URL creation) from the read path (redirects)."

### Complete Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENT LAYER                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Browser    │  │  Mobile App  │  │  API Client  │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CDN (CloudFront)                              │
│  - Cache redirect responses (for popular URLs)                  │
│  - Reduce load on origin servers                                │
│  - TTL: 1 hour                                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              LOAD BALANCER (Layer 7)                             │
│  - Route /shorten → Write Service                               │
│  - Route /{short_code} → Read Service                           │
│  - Rate limiting (1000 req/min per IP)                          │
│  - SSL termination                                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  APPLICATION SERVICES LAYER                      │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  WRITE SERVICE (URL Creation)                          │    │
│  │  - Validate long URL                                   │    │
│  │  - Generate short code                                 │    │
│  │  - Check for duplicates                                │    │
│  │  - Store in database                                   │    │
│  │  - Invalidate cache                                    │    │
│  │  Technology: Java/Spring Boot                          │    │
│  │  Instances: 10 (auto-scaling)                          │    │
│  │  Throughput: 350 writes/sec per instance               │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  READ SERVICE (Redirects)                              │    │
│  │  - Look up short code                                  │    │
│  │  - Check cache first                                   │    │
│  │  - Return 302 redirect                                 │    │
│  │  - Async: Log analytics event                          │    │
│  │  Technology: Go (high performance)                     │    │
│  │  Instances: 100 (auto-scaling)                         │    │
│  │  Throughput: 35K reads/sec per instance                │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  ANALYTICS SERVICE                                     │    │
│  │  - Consume click events from Kafka                     │    │
│  │  - Aggregate statistics                                │    │
│  │  - Store in time-series DB                             │    │
│  │  Technology: Python/Spark                              │    │
│  │  Instances: 20                                         │    │
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
│  │  - url_mappings table                                  │    │
│  │  - Primary key: short_code                             │    │
│  │  - Index on: long_url (for deduplication)             │    │
│  │  Configuration: Multi-AZ, 3 read replicas             │    │
│  │  Size: 500 GB                                          │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  CACHE (Redis Cluster)                                 │    │
│  │  - Key: short_code                                     │    │
│  │  - Value: long_url                                     │    │
│  │  - TTL: 1 hour                                         │    │
│  │  - Eviction: LRU (Least Recently Used)                │    │
│  │  Configuration: 10 shards, 2 replicas each            │    │
│  │  Size: 50 GB (hot data)                                │    │
│  │  Hit rate: 90%+                                        │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  ID GENERATOR (Redis)                                  │    │
│  │  - Atomic counter                                      │    │
│  │  - Range allocation (1M at a time)                     │    │
│  │  - Distributed across write services                   │    │
│  │  Configuration: Redis Cluster with persistence         │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  ANALYTICS DB (ClickHouse/TimescaleDB)                │    │
│  │  - Click events (time-series)                          │    │
│  │  - Partitioned by date                                 │    │
│  │  - Retention: 30 days                                  │    │
│  │  Size: 100 TB                                          │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  MESSAGE QUEUE (Kafka)                                 │    │
│  │  Topics:                                               │    │
│  │  - click_events (high throughput)                      │    │
│  │  - url_created                                         │    │
│  │  Configuration: 10 partitions, replication factor 3    │    │
│  └────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

### Data Flow: Create Short URL

```
1. User submits long URL
   POST /shorten
   { "long_url": "https://www.example.com/very/long/url" }
   ↓
2. Load Balancer → Write Service
   ↓
3. Write Service validates URL:
   - Check format (regex)
   - Check length (< 2048 chars)
   - Optional: Check if URL is accessible
   ↓
4. Check for existing short code (deduplication):
   SELECT short_code FROM url_mappings 
   WHERE long_url = ?
   
   If exists: Return existing short_code
   If not: Continue
   ↓
5. Generate unique ID:
   - Get next ID from Redis counter
   - ID = INCR global_counter
   ↓
6. Convert ID to Base62:
   - ID: 125 → Base62: "21"
   - ID: 1,000,000,000 → Base62: "15FTGg"
   ↓
7. Store in database:
   INSERT INTO url_mappings (
     short_code, long_url, created_at, expires_at
   ) VALUES (?, ?, NOW(), ?)
   ↓
8. Publish event to Kafka:
   Topic: url_created
   { short_code, long_url, timestamp }
   ↓
9. Return response:
   {
     "short_url": "https://short.ly/21",
     "long_url": "https://www.example.com/very/long/url"
   }

Latency: < 500ms
```

---

### Data Flow: Redirect (Read Path)

```
1. User clicks short URL
   GET /21
   ↓
2. CDN check:
   - If cached: Return 302 redirect immediately
   - If not: Forward to Load Balancer
   ↓
3. Load Balancer → Read Service
   ↓
4. Read Service checks Redis cache:
   GET short_code:21
   
   Cache HIT (90% of requests):
   - Retrieve long_url from cache
   - Skip to step 7
   
   Cache MISS (10% of requests):
   - Continue to step 5
   ↓
5. Query database:
   SELECT long_url, expires_at 
   FROM url_mappings 
   WHERE short_code = '21'
   ↓
6. Store in cache:
   SET short_code:21 long_url EX 3600
   ↓
7. Check expiration:
   IF expires_at < NOW():
     Return 404 Not Found
   ↓
8. Async: Log analytics event to Kafka:
   Topic: click_events
   {
     short_code: "21",
     timestamp: 1705334400,
     ip: "192.168.1.1",
     user_agent: "Mozilla/5.0...",
     referrer: "https://twitter.com"
   }
   ↓
9. Return 302 redirect:
   HTTP/1.1 302 Found
   Location: https://www.example.com/very/long/url
   Cache-Control: public, max-age=3600

Latency: 
- Cache hit: < 10ms
- Cache miss: < 100ms
```

---

### Why This Architecture Works:

**1. Separation of Read/Write**
✅ Write service optimized for ID generation
✅ Read service optimized for low latency
✅ Can scale independently

**2. Caching Strategy**
✅ Redis for hot data (90% hit rate)
✅ CDN for popular URLs
✅ Reduces database load by 90%

**3. Async Analytics**
✅ Doesn't block redirect
✅ Kafka handles high throughput
✅ Can process offline

**4. High Availability**
✅ Multi-AZ database
✅ Redis cluster with replication
✅ Stateless services (easy to scale)


---

## PHASE 5: API DESIGN (3 minutes)

### What to Say:

"Let me define the key API endpoints."

### API Endpoints:

```
1. Shorten URL
POST /api/v1/shorten

Request:
{
  "long_url": "https://www.example.com/very/long/url",
  "custom_alias": "my-link",  // Optional
  "expires_at": "2025-12-31T23:59:59Z"  // Optional
}

Response: 201 Created
{
  "short_url": "https://short.ly/my-link",
  "short_code": "my-link",
  "long_url": "https://www.example.com/very/long/url",
  "created_at": "2024-01-15T10:30:00Z",
  "expires_at": "2025-12-31T23:59:59Z"
}

Errors:
- 400 Bad Request: Invalid URL format
- 409 Conflict: Custom alias already taken
- 429 Too Many Requests: Rate limit exceeded

2. Redirect
GET /{short_code}

Response: 302 Found
Location: https://www.example.com/very/long/url
Cache-Control: public, max-age=3600

Errors:
- 404 Not Found: Short code doesn't exist or expired

3. Get URL Info (Optional)
GET /api/v1/urls/{short_code}

Response: 200 OK
{
  "short_code": "abc123",
  "long_url": "https://www.example.com/very/long/url",
  "created_at": "2024-01-15T10:30:00Z",
  "expires_at": null,
  "click_count": 1523,
  "last_clicked_at": "2024-01-15T15:45:00Z"
}

4. Get Analytics (Optional)
GET /api/v1/urls/{short_code}/analytics?start_date=2024-01-01&end_date=2024-01-31

Response: 200 OK
{
  "short_code": "abc123",
  "total_clicks": 1523,
  "clicks_by_date": [
    {"date": "2024-01-15", "count": 45},
    {"date": "2024-01-16", "count": 67}
  ],
  "clicks_by_country": [
    {"country": "US", "count": 800},
    {"country": "UK", "count": 300}
  ],
  "clicks_by_device": [
    {"device": "mobile", "count": 900},
    {"device": "desktop", "count": 623}
  ]
}
```

---

## PHASE 6: DATA MODELS (4 minutes)

### What to Say:

"Let me define the database schemas."

### PostgreSQL Schema:

```sql
-- URL Mappings Table
CREATE TABLE url_mappings (
    short_code VARCHAR(10) PRIMARY KEY,
    long_url TEXT NOT NULL,
    user_id UUID,
    
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    
    click_count BIGINT DEFAULT 0,
    last_clicked_at TIMESTAMP,
    
    CONSTRAINT chk_short_code_length CHECK (LENGTH(short_code) >= 6),
    CONSTRAINT chk_long_url_length CHECK (LENGTH(long_url) <= 2048)
);

-- Indexes
CREATE INDEX idx_long_url ON url_mappings(long_url);  -- For deduplication
CREATE INDEX idx_user_id ON url_mappings(user_id);    -- For user's URLs
CREATE INDEX idx_expires_at ON url_mappings(expires_at) WHERE expires_at IS NOT NULL;

-- Users Table (Optional)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### ClickHouse Schema (Analytics):

```sql
-- Click Events Table (Time-Series)
CREATE TABLE click_events (
    short_code String,
    timestamp DateTime,
    ip_address String,
    country String,
    city String,
    user_agent String,
    device_type String,
    referrer String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (short_code, timestamp);

-- Materialized View for Aggregations
CREATE MATERIALIZED VIEW click_stats_daily
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (short_code, date)
AS SELECT
    short_code,
    toDate(timestamp) as date,
    count() as click_count,
    uniq(ip_address) as unique_visitors
FROM click_events
GROUP BY short_code, date;
```

### Redis Data Structures:

```redis
# Short code to long URL mapping
SET short_code:abc123 "https://www.example.com/very/long/url" EX 3600

# Global counter for ID generation
INCR global_counter
→ Returns: 1000000001

# Range allocation (per write service instance)
SET write_service:1:range_start 1000000000
SET write_service:1:range_end 1001000000
SET write_service:1:current 1000000000
```

---

## PHASE 7: CORE ALGORITHM - SHORT CODE GENERATION (8 minutes)

### What to Say:

"The heart of the system is generating unique short codes. Let me compare different approaches."

### Approach 1: Random String Generation (❌ FAILS AT SCALE)

```python
import random
import string

def generate_short_code():
    chars = string.ascii_letters + string.digits  # a-z, A-Z, 0-9
    return ''.join(random.choice(chars) for _ in range(7))

# Usage
short_code = generate_short_code()  # e.g., "x7A9mNp"

# Check if exists in database
while db.exists(short_code):
    short_code = generate_short_code()  # Retry

db.insert(short_code, long_url)
```

**Why it fails:**
- Collision probability increases with database size
- At 80% capacity: 8 out of 10 attempts collide
- Requires multiple database queries per insert
- Database becomes bottleneck
- Latency increases exponentially

**Collision probability:**
```
With 62^7 = 3.5T possible codes
After 1B URLs (28% full):
  Collision probability = 1B / 3.5T = 0.028% (1 in 3,500)
  
After 2.8B URLs (80% full):
  Collision probability = 2.8B / 3.5T = 80% (4 in 5)
```

---

### Approach 2: Hash Function (⚠️ BETTER BUT STILL ISSUES)

```python
import hashlib

def generate_short_code(long_url):
    # Hash the URL
    hash_object = hashlib.sha256(long_url.encode())
    hash_hex = hash_object.hexdigest()
    
    # Convert to Base62
    hash_int = int(hash_hex, 16)
    short_code = base62_encode(hash_int)[:7]
    
    return short_code

def base62_encode(num):
    chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    if num == 0:
        return chars[0]
    
    result = []
    while num:
        result.append(chars[num % 62])
        num //= 62
    
    return ''.join(reversed(result))
```

**Pros:**
✅ Deterministic (same URL → same code)
✅ No database lookup needed for generation
✅ Natural deduplication

**Cons:**
❌ Collisions still possible (truncated hash)
❌ Predictable (security concern)
❌ Need collision handling
❌ Longer codes for uniqueness

**Collision handling:**
```python
def generate_with_collision_handling(long_url):
    attempt = 0
    max_attempts = 5
    
    while attempt < max_attempts:
        # Add salt to reduce collisions
        salted_url = f"{long_url}:{attempt}"
        short_code = generate_short_code(salted_url)
        
        if not db.exists(short_code):
            return short_code
        
        attempt += 1
    
    raise Exception("Failed to generate unique code")
```

---

### Approach 3: Counter + Base62 Encoding (✅ BEST)

```python
class ShortCodeGenerator:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    
    def generate(self):
        # Get next ID from Redis (atomic)
        id = self.redis.incr('global_counter')
        
        # Convert to Base62
        short_code = self.base62_encode(id)
        
        return short_code
    
    def base62_encode(self, num):
        if num == 0:
            return self.chars[0]
        
        result = []
        while num:
            result.append(self.chars[num % 62])
            num //= 62
        
        return ''.join(reversed(result))
    
    def base62_decode(self, code):
        """Reverse operation (useful for lookups)"""
        num = 0
        for char in code:
            num = num * 62 + self.chars.index(char)
        return num

# Usage
generator = ShortCodeGenerator(redis_client)
short_code = generator.generate()  # "21" for ID=125
```

**Why this is best:**
✅ **Zero collisions** (each ID is unique)
✅ **Fast** (no database lookups)
✅ **Scalable** (distributed counter)
✅ **Predictable length** (grows slowly)
✅ **Reversible** (can decode back to ID)

**Length analysis:**
```
ID Range          Base62 Code    Length
1 - 61            0 - z          1 char
62 - 3,843        10 - zz        2 chars
3,844 - 238,327   100 - zzz      3 chars
...
56B - 3.5T        100000 - zzzzzz  6 chars
3.5T - 218T       1000000 - zzzzzzz 7 chars

At 100M URLs/day:
- Day 1: 100M → 6 chars
- Year 1: 36.5B → 6 chars
- Year 2: 73B → 7 chars
```

---

### Optimization: Range Allocation

**Problem:** All write services hitting single Redis counter = bottleneck

**Solution:** Allocate ranges to each service

```python
class RangeAllocator:
    def __init__(self, redis_client, service_id, range_size=1_000_000):
        self.redis = redis_client
        self.service_id = service_id
        self.range_size = range_size
        
        self.range_start = None
        self.range_end = None
        self.current = None
    
    def allocate_range(self):
        """Get a range of IDs from Redis"""
        # Atomic operation
        start = self.redis.incrby('global_counter', self.range_size)
        
        self.range_start = start - self.range_size + 1
        self.range_end = start
        self.current = self.range_start
        
        print(f"Service {self.service_id} allocated range: {self.range_start} - {self.range_end}")
    
    def get_next_id(self):
        # Check if we need a new range
        if self.current is None or self.current > self.range_end:
            self.allocate_range()
        
        id = self.current
        self.current += 1
        return id

# Usage across multiple write services
service_1 = RangeAllocator(redis, service_id=1)
service_2 = RangeAllocator(redis, service_id=2)

# Service 1 gets IDs: 1 - 1,000,000
# Service 2 gets IDs: 1,000,001 - 2,000,000
# No coordination needed between services!
```

**Benefits:**
✅ Reduces Redis load (1 request per 1M IDs)
✅ Services work independently
✅ No network latency per ID
✅ Can handle service crashes (waste some IDs, but acceptable)

**Trade-off:**
- Wasted IDs if service crashes (acceptable)
- IDs not strictly sequential (acceptable)

---

## PHASE 8: DEEP DIVE - SCALING READS (5 minutes)

### What to Say:

"With 3.5M read QPS, we need aggressive caching and optimization."

### Layer 1: Database Indexing

```sql
-- Primary key on short_code (automatic B-tree index)
CREATE TABLE url_mappings (
    short_code VARCHAR(10) PRIMARY KEY,
    long_url TEXT NOT NULL,
    ...
);

-- Query performance
SELECT long_url FROM url_mappings WHERE short_code = 'abc123';
-- O(log n) lookup with B-tree index
-- ~10ms on SSD
```

**Why not enough:**
- 3.5M QPS × 10ms = 35,000 concurrent queries
- Database can't handle this load
- Need caching

---

### Layer 2: Redis Cache

```python
class URLService:
    def __init__(self, redis, db):
        self.redis = redis
        self.db = db
    
    def get_long_url(self, short_code):
        # Check cache first
        cache_key = f"short_code:{short_code}"
        long_url = self.redis.get(cache_key)
        
        if long_url:
            # Cache HIT (90% of requests)
            return long_url
        
        # Cache MISS (10% of requests)
        long_url = self.db.query(
            "SELECT long_url FROM url_mappings WHERE short_code = ?",
            short_code
        )
        
        if long_url:
            # Store in cache (1 hour TTL)
            self.redis.setex(cache_key, 3600, long_url)
        
        return long_url
```

**Performance:**
```
Without cache:
- 3.5M QPS → Database
- Database fails

With cache (90% hit rate):
- 3.15M QPS → Redis (fast)
- 350K QPS → Database (manageable)

Redis performance:
- Memory access: ~100 nanoseconds
- Can handle millions of QPS per instance
- 10 Redis shards = 10M+ QPS capacity
```

---

### Layer 3: CDN Caching

```
Client Request
  ↓
CDN Edge (Cloudflare/CloudFront)
  - Cache popular URLs
  - TTL: 1 hour
  - Reduces origin load by 50%
  ↓
Load Balancer
  ↓
Read Service
  ↓
Redis Cache
  ↓
Database
```

**CDN Configuration:**
```
Cache-Control: public, max-age=3600
Vary: User-Agent

Popular URLs (top 1%):
- Served from CDN edge
- Never hit origin servers
- < 10ms latency globally
```

---

### Caching Strategy Comparison:

```
┌──────────────────┬─────────────┬─────────────┬─────────────┐
│ Layer            │ Hit Rate    │ Latency     │ Capacity    │
├──────────────────┼─────────────┼─────────────┼─────────────┤
│ CDN              │ 50%         │ 10ms        │ Unlimited   │
│ Redis            │ 90%         │ 1ms         │ 50 GB       │
│ Database         │ 100%        │ 10ms        │ 500 GB      │
└──────────────────┴─────────────┴─────────────┴─────────────┘

Request flow:
- 50% served by CDN (1.75M QPS)
- 45% served by Redis (1.575M QPS)
- 5% hit database (175K QPS)

Database load reduced by 95%!
```

---

## PHASE 9: DEEP DIVE - DISTRIBUTED ID GENERATION (4 minutes)

### What to Say:

"For high availability, we need distributed ID generation without a single point of failure."

### Approach 1: Single Redis Counter (⚠️ SPOF)

```
All Write Services → Single Redis → INCR counter

Problem: If Redis goes down, can't create new URLs
```

---

### Approach 2: Redis Cluster with Replication (✅ BETTER)

```
┌─────────────────────────────────────────────────────────────────┐
│  Redis Cluster                                                   │
│                                                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐   │
│  │ Master 1     │────▶│ Replica 1A   │     │ Replica 1B   │   │
│  │ (Primary)    │     │              │     │              │   │
│  └──────────────┘     └──────────────┘     └──────────────┘   │
│                                                                   │
│  Automatic Failover:                                             │
│  - If Master fails, Replica promoted                            │
│  - Downtime: < 1 second                                         │
│  - Counter persisted to disk (AOF)                              │
└─────────────────────────────────────────────────────────────────┘
```

**Configuration:**
```redis
# Enable persistence
appendonly yes
appendfsync everysec

# Replication
replicaof master-ip 6379

# Sentinel for automatic failover
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 10000
```

---

### Approach 3: Snowflake-style IDs (✅ NO COORDINATION)

```python
class SnowflakeIDGenerator:
    """
    64-bit ID structure:
    - 41 bits: Timestamp (milliseconds since epoch)
    - 10 bits: Machine ID (1024 machines)
    - 12 bits: Sequence (4096 IDs per millisecond per machine)
    """
    
    def __init__(self, machine_id):
        self.machine_id = machine_id  # 0-1023
        self.sequence = 0
        self.last_timestamp = -1
        
        # Epoch: 2024-01-01 00:00:00
        self.epoch = 1704067200000
    
    def generate(self):
        timestamp = self.current_timestamp()
        
        # Same millisecond
        if timestamp == self.last_timestamp:
            self.sequence = (self.sequence + 1) & 0xFFF  # 12 bits
            
            # Sequence overflow, wait for next millisecond
            if self.sequence == 0:
                timestamp = self.wait_next_millis(timestamp)
        else:
            self.sequence = 0
        
        self.last_timestamp = timestamp
        
        # Construct 64-bit ID
        id = ((timestamp - self.epoch) << 22) | (self.machine_id << 12) | self.sequence
        
        return id
    
    def current_timestamp(self):
        return int(time.time() * 1000)
    
    def wait_next_millis(self, last_timestamp):
        timestamp = self.current_timestamp()
        while timestamp <= last_timestamp:
            timestamp = self.current_timestamp()
        return timestamp

# Usage
generator = SnowflakeIDGenerator(machine_id=1)
id = generator.generate()  # e.g., 1234567890123456789
short_code = base62_encode(id)  # Convert to Base62
```

**Benefits:**
✅ No coordination between machines
✅ No single point of failure
✅ Time-ordered IDs
✅ 4096 IDs per millisecond per machine
✅ Supports 1024 machines

**Capacity:**
```
Per machine: 4096 IDs/ms = 4M IDs/second
1024 machines: 4B IDs/second

Our requirement: 3,500 IDs/second
Headroom: 1,000,000x
```

---

### Comparison:

```
┌──────────────────┬─────────────┬─────────────┬─────────────┐
│ Approach         │ Coordination│ SPOF        │ Complexity  │
├──────────────────┼─────────────┼─────────────┼─────────────┤
│ Single Redis     │ Required    │ Yes         │ Low         │
│ Redis Cluster    │ Required    │ No          │ Medium      │
│ Snowflake        │ None        │ No          │ Medium      │
│ Range Allocation │ Minimal     │ No          │ Low         │
└──────────────────┴─────────────┴─────────────┴─────────────┘

Recommendation: Range Allocation (best balance)
- Simple to implement
- No SPOF
- Minimal coordination
- Acceptable ID waste
```



---

## FOLLOW-UP QUESTIONS

### Follow-up 1: 301 vs 302 Redirects

**Question:** "Should we use 301 or 302 redirects?"

**Answer:**

```
301 Permanent: Browser caches, no analytics
302 Temporary: Every click tracked, full analytics

Recommendation: 302 Redirect
- Analytics is core business value
- Server load manageable with caching
```

---

### Follow-up 2: Custom Aliases

**Question:** "How do we handle custom aliases?"

**Answer:**

```python
def create_short_url(long_url, custom_alias=None):
    if custom_alias:
        if not is_valid_alias(custom_alias):
            raise ValueError("Invalid alias")
        if db.exists(custom_alias):
            raise ConflictError("Alias taken")
        short_code = custom_alias
    else:
        id = get_next_id()
        short_code = base62_encode(id)
    
    db.insert(short_code, long_url)
    return short_code
```

---

## SUMMARY: TOP 20% PERFORMANCE

### What Makes This Outstanding:

1. **Identified core challenge** (uniqueness without collisions)
2. **Compared approaches** (random, hash, counter) with trade-offs
3. **Optimal solution** (counter + Base62 with range allocation)
4. **Aggressive caching** (CDN + Redis + DB = 95% reduction)
5. **Massive read scale** (3.5M QPS handled)
6. **Distributed ID generation** (no single point of failure)
7. **Complete analytics** (Kafka + Spark + ClickHouse)

---

## KEY TAKEAWAYS

**The Core Insight:**
"URL shortening is about distributed ID generation at scale with 1000:1 read-to-write ratio."

**The Winning Strategy:**
1. Counter + Base62 for zero collisions
2. Range allocation for distributed scaling
3. 3-layer caching for 95% load reduction
4. 302 redirects for analytics
5. Async analytics via Kafka

This demonstrates senior-level system design thinking! 🚀
