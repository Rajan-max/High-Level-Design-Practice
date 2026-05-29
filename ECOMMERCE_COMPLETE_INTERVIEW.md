# E-commerce Platform (Amazon/Flipkart) - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: High-Level Architecture (8 min)
Phase 5: API Design (3 min)
Phase 6: Data Models & Storage Strategy (4 min)
Phase 7: Core Data Flows (7 min)
Phase 8: Deep Dive - Inventory Management & Consistency (5 min)
Phase 9: Deep Dive - Search & Caching Strategy (5 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"I'll design a global e-commerce platform like Amazon where millions of users can browse products, manage shopping carts, and place orders. Let me clarify the scope and requirements."

### Questions to Ask:

**Q1: Platform Scope**
- "Are we building a marketplace (multiple sellers) or single-vendor platform?"
- "Do we need seller onboarding and management features?"

**Expected Answer:** Marketplace with multiple sellers, focus on customer journey

**Q2: Product Categories**
- "What types of products - physical goods, digital products, or both?"
- "Do we need to handle different product attributes (clothing sizes, electronics specs)?"

**Expected Answer:** Physical goods with varied attributes, digital products out of scope

**Q3: Geographic Coverage**
- "Is this a global platform or region-specific?"
- "Do we need multi-currency, multi-language support?"

**Expected Answer:** Global platform, multi-currency support needed

**Q4: Core User Journeys**
- "Should we focus on the complete buying journey or specific parts?"
- "Do we need features like reviews, recommendations, wishlists?"

**Expected Answer:** Focus on core buying journey - search, cart, checkout

**Q5: Scale & Traffic**
- "What's the expected scale - DAU, concurrent users, orders per day?"
- "Are there seasonal traffic spikes (Black Friday, holiday sales)?"

**Expected Answer:** 100M DAU, 10x traffic spikes during sales events

**Q6: Inventory Management**
- "How strict should inventory management be?"
- "Can we allow overselling with backorders, or must prevent it?"

**Expected Answer:** Strict inventory management, no overselling allowed

**Q7: Payment & Fulfillment**
- "Do we handle payments directly or integrate with processors?"
- "Should we include shipping/logistics or focus on order management?"

**Expected Answer:** Integrate with payment processors, basic order management

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### Functional Requirements (Priority Order):

```
CORE REQUIREMENTS (Above the line):

1. Product Catalog & Search (CRITICAL)
   - Browse products by category
   - Search by keywords with filters (price, brand, rating)
   - View detailed product pages with images and specs
   - Real-time inventory availability

2. Shopping Cart Management (CRITICAL)
   - Add/remove items from cart
   - Update quantities
   - Persist cart across sessions
   - Calculate totals with taxes and shipping

3. Checkout & Order Processing (CRITICAL)
   - Secure payment processing
   - Inventory reservation during checkout
   - Order confirmation and tracking
   - Prevent overselling (strong consistency)

4. User Account Management (IMPORTANT)
   - User registration and authentication
   - Profile management
   - Order history and tracking
   - Address and payment method storage

BELOW THE LINE (Out of scope):
- Product reviews and ratings
- Recommendation engine
- Seller dashboard and analytics
- Advanced promotions and coupons
- Wishlist and favorites
- Social features and sharing
- Mobile app push notifications
- Customer service chat
```

### Non-Functional Requirements:

```
PERFORMANCE:
- Search latency: < 200ms (P95) for high conversion
- Product page load: < 500ms (P95)
- Checkout completion: < 2 seconds (P95)
- System throughput: 60K QPS peak (5x average)

SCALABILITY:
- Support 100M Daily Active Users (DAU)
- Handle 10x traffic spikes during flash sales
- Process 1M orders per day (12 orders/sec average)
- Support 100M products in catalog

AVAILABILITY & CONSISTENCY:
- Storefront availability: 99.99% (browsing never goes down)
- Checkout availability: 99.9% (favor consistency over availability)
- Catalog consistency: Eventual (prices can lag seconds)
- Inventory consistency: Strong (prevent overselling)

SECURITY & COMPLIANCE:
- PCI DSS compliance for payment processing
- Data encryption at rest and in transit
- Rate limiting to prevent abuse
- GDPR compliance for user data

GEOGRAPHIC DISTRIBUTION:
- Multi-region deployment for global users
- CDN for static assets and images
- Regional data compliance requirements
```

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me calculate the scale to understand our system requirements and inform architectural decisions."

### Traffic Estimation:

```
USER BEHAVIOR:
- DAU: 100 Million users
- Average page views per user: 10 per day
- Conversion rate: 1% (1M users place orders)
- Read:Write ratio: 100:1 (heavy browsing, light purchasing)

QPS CALCULATION:
Total page views: 100M × 10 = 1B requests/day
Average QPS: 1B / 86,400 = ~12K QPS
Peak QPS: 12K × 5 = 60K QPS (during sales events)

Write QPS (Orders): 1M orders/day = ~12 orders/sec average
Peak order QPS: 12 × 100 = 1,200 orders/sec (flash sales)

TRAFFIC BREAKDOWN:
- Product search/browse: 80% = 48K QPS peak
- Product details: 15% = 9K QPS peak  
- Cart operations: 4% = 2.4K QPS peak
- Checkout: 1% = 600 QPS peak
```

### Storage Estimation:

```
PRODUCT CATALOG:
- Total products: 100M items
- Product metadata: 1KB per item = 100GB
- Product images: 5 images × 500KB = 2.5MB per product
- Total image storage: 100M × 2.5MB = 250TB

USER DATA:
- Total users: 500M registered users
- User profile: 2KB per user = 1GB
- User addresses: 3 addresses × 200 bytes = 600 bytes per user = 300GB

ORDER DATA:
- Orders per day: 1M
- Order record size: 2KB
- Daily order data: 1M × 2KB = 2GB/day
- Annual order data: 2GB × 365 = 730GB/year
- 5-year retention: 3.65TB

CART DATA (Redis):
- Active carts: 10M concurrent users
- Cart size: 500 bytes per cart
- Total cart storage: 10M × 500B = 5GB

TOTAL STORAGE REQUIREMENTS:
- Metadata: ~4TB (with replication: 12TB)
- Images/Media: 250TB (CDN + S3)
- Transactional data: ~4TB (with replication: 12TB)
- Cache: 100GB (Redis clusters)
```

### Infrastructure Estimation:

```
COMPUTE REQUIREMENTS:
- Application servers: 200 instances (c5.2xlarge)
- Database instances: 50 instances (r5.xlarge)
- Cache instances: 20 instances (r5.large)
- Search instances: 15 instances (c5.xlarge)

NETWORK BANDWIDTH:
- Peak traffic: 60K QPS × 50KB avg response = 3GB/sec
- Image delivery: 80% via CDN = 2.4GB/sec CDN traffic
- Database traffic: 500MB/sec internal

COST ESTIMATION (AWS):
- Compute: $60K/month
- Storage (RDS): $8K/month
- CDN (CloudFront): $15K/month
- S3 storage: $5K/month
- Data transfer: $12K/month
Total: ~$100K/month (~$1.2M/year)
```

---

## PHASE 4: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"I'll design a microservices architecture that separates read-heavy catalog operations from write-heavy transactional operations. The key insight is using polyglot persistence - different databases optimized for different access patterns."

### Complete System Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│   Web App    │   Mobile App   │   Admin Portal  │  Seller App   │
│  (React.js)  │ (React Native) │   (Dashboard)   │  (Management) │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EDGE & CONTENT DELIVERY                      │
├─────────────────────────────────────────────────────────────────┤
│  CloudFront CDN  │  Application LB  │  WAF & DDoS Protection   │
│  (Images, CSS,   │  (Geographic     │  (Security & Rate        │
│   JS, API Cache) │   Load Balance)  │   Limiting)              │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                        API GATEWAY                              │
├─────────────────────────────────────────────────────────────────┤
│  Authentication  │  Request Routing │  Circuit Breaker         │
│  Rate Limiting   │  Protocol Trans  │  Monitoring & Logging    │
└─────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES LAYER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  PRODUCT    │  │   SEARCH    │  │    CART     │             │
│  │  SERVICE    │  │  SERVICE    │  │  SERVICE    │             │
│  │             │  │             │  │             │             │
│  │ • Catalog   │  │ • Keywords  │  │ • Add/Remove│             │
│  │ • Details   │  │ • Filters   │  │ • Persist   │             │
│  │ • Images    │  │ • Facets    │  │ • Calculate │             │
│  │ • Pricing   │  │ • Suggest   │  │ • Session   │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   ORDER     │  │ INVENTORY   │  │   PAYMENT   │             │
│  │  SERVICE    │  │  SERVICE    │  │  SERVICE    │             │
│  │             │  │             │  │             │             │
│  │ • Create    │  │ • Reserve   │  │ • Process   │             │
│  │ • Track     │  │ • Release   │  │ • Refund    │             │
│  │ • History   │  │ • Update    │  │ • Webhook   │             │
│  │ • Status    │  │ • Check     │  │ • Validate  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    USER     │  │NOTIFICATION │  │  ANALYTICS  │             │
│  │  SERVICE    │  │  SERVICE    │  │  SERVICE    │             │
│  │             │  │             │  │             │             │
│  │ • Auth      │  │ • Email     │  │ • Events    │             │
│  │ • Profile   │  │ • SMS       │  │ • Metrics   │             │
│  │ • Address   │  │ • Push      │  │ • Reports   │             │
│  │ • Prefs     │  │ • Templates │  │ • Insights  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   MONGODB   │  │ POSTGRESQL  │  │    REDIS    │             │
│  │  CLUSTER    │  │  CLUSTER    │  │  CLUSTER    │             │
│  │             │  │             │  │             │             │
│  │ • Products  │  │ • Orders    │  │ • Cart      │             │
│  │ • Catalog   │  │ • Users     │  │ • Session   │             │
│  │ • Reviews   │  │ • Inventory │  │ • Cache     │             │
│  │ • Metadata  │  │ • Payments  │  │ • Locks     │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ELASTICSEARCH│  │   KAFKA     │  │     S3      │             │
│  │  CLUSTER    │  │ EVENT BUS   │  │  STORAGE    │             │
│  │             │  │             │  │             │             │
│  │ • Search    │  │ • Events    │  │ • Images    │             │
│  │ • Index     │  │ • Logs      │  │ • Backups   │             │
│  │ • Analytics │  │ • Audit     │  │ • Archives  │             │
│  │ • Facets    │  │ • Stream    │  │ • Static    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   STRIPE    │  │   TWILIO    │  │ PROMETHEUS  │             │
│  │  PAYMENTS   │  │COMMUNICATION│  │ MONITORING  │             │
│  │             │  │             │  │             │             │
│  │ • Process   │  │ • SMS       │  │ • Metrics   │             │
│  │ • Webhooks  │  │ • Email     │  │ • Alerts    │             │
│  │ • Refunds   │  │ • Templates │  │ • Dashboards│             │
│  │ • Compliance│  │ • Delivery  │  │ • Tracing   │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

### Key Architectural Decisions & Reasoning:

**1. Why Microservices Architecture?**
```
BENEFITS:
✓ Independent scaling: Search service needs 48K QPS, Order service needs 600 QPS
✓ Technology diversity: MongoDB for catalog, PostgreSQL for transactions
✓ Team autonomy: Different teams can own different services
✓ Fault isolation: Search failure doesn't impact checkout

CHALLENGES:
- Distributed system complexity
- Network latency between services
- Data consistency across services
- Operational overhead

DECISION: Benefits outweigh challenges at this scale (100M DAU)
```

**2. Why API Gateway?**
```
BENEFITS:
✓ Single entry point for all clients
✓ Cross-cutting concerns (auth, rate limiting, monitoring)
✓ Protocol translation (REST to gRPC internally)
✓ Circuit breaker pattern for fault tolerance

ALTERNATIVES CONSIDERED:
- Direct service calls: Too complex for clients
- Service mesh: Adds complexity, gateway simpler for external traffic

DECISION: API Gateway for external traffic, service mesh for internal
```

**3. Why CDN (CloudFront)?**
```
BENEFITS:
✓ 80% of bandwidth is images/static assets
✓ Global edge locations reduce latency
✓ Offloads traffic from origin servers
✓ Built-in DDoS protection

COST ANALYSIS:
- Without CDN: 3GB/sec × $0.09/GB = $23K/month data transfer
- With CDN: $15K/month (35% savings + better performance)

DECISION: Essential for global e-commerce platform
```

---

## PHASE 5: API DESIGN (3 minutes)

### What to Say:

"I'll design RESTful APIs that are intuitive for frontend developers while being efficient for our backend services. Security and idempotency are key considerations."

### Core API Endpoints:

```java
// Product Catalog APIs
GET    /api/v1/products/search              // Search products
GET    /api/v1/products/{productId}         // Get product details
GET    /api/v1/products/{productId}/images  // Get product images
GET    /api/v1/categories                   // Get category tree

// Shopping Cart APIs
GET    /api/v1/cart                         // Get current cart
POST   /api/v1/cart/items                   // Add item to cart
PUT    /api/v1/cart/items/{itemId}          // Update item quantity
DELETE /api/v1/cart/items/{itemId}          // Remove item from cart
DELETE /api/v1/cart                         // Clear entire cart

// Order Management APIs
POST   /api/v1/orders                       // Create order (checkout)
GET    /api/v1/orders/{orderId}             // Get order details
GET    /api/v1/orders                       // Get order history
PUT    /api/v1/orders/{orderId}/cancel      // Cancel order

// User Management APIs
POST   /api/v1/auth/login                   // User login
POST   /api/v1/auth/register                // User registration
GET    /api/v1/users/profile                // Get user profile
PUT    /api/v1/users/profile                // Update user profile
GET    /api/v1/users/addresses              // Get saved addresses
POST   /api/v1/users/addresses              // Add new address

// Payment APIs
POST   /api/v1/payments/process             // Process payment
GET    /api/v1/payments/{paymentId}/status  // Get payment status
POST   /api/v1/payments/webhooks/stripe     // Payment webhooks
```

### API Design Principles:

**1. Security First**
```
AUTHENTICATION:
- JWT tokens for user sessions
- API keys for service-to-service communication
- User ID extracted from token, never from request body

AUTHORIZATION:
- Role-based access control (customer, seller, admin)
- Resource-level permissions (user can only access own orders)

DATA VALIDATION:
- Input sanitization to prevent injection attacks
- Rate limiting per user/IP to prevent abuse
- Idempotency keys for critical operations
```

**2. Performance Optimization**
```
CACHING HEADERS:
- Product details: Cache-Control: max-age=300 (5 minutes)
- Product images: Cache-Control: max-age=86400 (24 hours)
- User-specific data: Cache-Control: no-cache

PAGINATION:
- Cursor-based pagination for large result sets
- Default page size: 20 items
- Maximum page size: 100 items

COMPRESSION:
- Gzip compression for all text responses
- Image optimization and WebP format support
```

**3. Error Handling**
```
HTTP STATUS CODES:
- 200: Success
- 400: Bad Request (validation errors)
- 401: Unauthorized (authentication required)
- 403: Forbidden (insufficient permissions)
- 404: Not Found
- 409: Conflict (inventory not available)
- 429: Too Many Requests (rate limited)
- 500: Internal Server Error

ERROR RESPONSE FORMAT:
{
  "error": {
    "code": "INVENTORY_UNAVAILABLE",
    "message": "Product is out of stock",
    "details": {
      "productId": "12345",
      "requestedQuantity": 5,
      "availableQuantity": 0
    }
  }
}
```

### Sample API Request/Response:

```json
// Product Search API
GET /api/v1/products/search?q=laptop&category=electronics&minPrice=500&maxPrice=2000&page=1&size=20

Response:
{
  "products": [
    {
      "productId": "prod_123",
      "title": "MacBook Pro 16-inch",
      "price": {
        "amount": 1999.99,
        "currency": "USD"
      },
      "imageUrl": "https://cdn.example.com/images/prod_123_thumb.jpg",
      "rating": 4.5,
      "reviewCount": 1250,
      "availability": "IN_STOCK",
      "seller": {
        "sellerId": "seller_456",
        "name": "Apple Store",
        "rating": 4.8
      }
    }
  ],
  "pagination": {
    "page": 1,
    "size": 20,
    "totalElements": 1500,
    "totalPages": 75,
    "hasNext": true
  },
  "facets": {
    "brands": [
      {"name": "Apple", "count": 45},
      {"name": "Dell", "count": 32}
    ],
    "priceRanges": [
      {"range": "500-1000", "count": 120},
      {"range": "1000-2000", "count": 80}
    ]
  }
}

// Create Order API
POST /api/v1/orders
Headers: 
  Authorization: Bearer <jwt_token>
  Idempotency-Key: uuid_12345

Body:
{
  "shippingAddressId": "addr_789",
  "paymentMethodId": "pm_card_visa",
  "items": [
    {
      "productId": "prod_123",
      "quantity": 1,
      "price": 1999.99
    }
  ]
}

Response:
{
  "orderId": "order_456789",
  "status": "CONFIRMED",
  "totalAmount": {
    "subtotal": 1999.99,
    "tax": 160.00,
    "shipping": 0.00,
    "total": 2159.99,
    "currency": "USD"
  },
  "estimatedDelivery": "2024-01-20T00:00:00Z",
  "trackingNumber": "1Z999AA1234567890",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

---

## PHASE 6: DATA MODELS & STORAGE STRATEGY (4 minutes)

### What to Say:

"Our storage strategy uses polyglot persistence - choosing the right database for each use case based on access patterns, consistency requirements, and scalability needs."

### Database Selection Reasoning:

**1. MongoDB for Product Catalog**
```
WHY MONGODB?
✓ Flexible schema for diverse product attributes
✓ Horizontal scaling with sharding
✓ Rich query capabilities for complex filters
✓ JSON document model matches API responses
✓ Built-in full-text search capabilities

WHAT DATA?
- Product information (title, description, specs)
- Product categories and hierarchies
- Seller information
- Product reviews and ratings

ACCESS PATTERNS:
- Heavy read workload (100:1 read:write ratio)
- Complex queries with multiple filters
- Frequent updates to inventory counts
- Bulk imports of new products

SCALING STRATEGY:
- Shard by product category or seller ID
- Read replicas in each geographic region
- Indexes on commonly queried fields
```

**2. PostgreSQL for Transactional Data**
```
WHY POSTGRESQL?
✓ ACID compliance for financial transactions
✓ Strong consistency for inventory management
✓ Mature ecosystem and tooling
✓ Complex joins for order/payment data
✓ Excellent performance for OLTP workloads

WHAT DATA?
- User accounts and profiles
- Order records and line items
- Payment transactions
- Inventory quantities
- Shipping addresses

ACCESS PATTERNS:
- Transactional writes requiring consistency
- Complex queries joining multiple tables
- Point lookups by user ID or order ID
- Time-series queries for analytics

SCALING STRATEGY:
- Read replicas for read-heavy queries
- Partition large tables by date or user ID
- Connection pooling for high concurrency
```

**3. Redis for High-Performance Caching**
```
WHY REDIS?
✓ Sub-millisecond latency for hot data
✓ Built-in data structures (sets, sorted sets)
✓ Automatic expiration with TTL
✓ High availability with clustering
✓ Pub/sub for real-time notifications

WHAT DATA?
- Shopping cart contents
- User session data
- Product cache (hot products)
- Search result cache
- Inventory cache (availability flags)

ACCESS PATTERNS:
- Very high read/write frequency
- Temporary data with TTL
- Real-time updates
- Session management

SCALING STRATEGY:
- Redis Cluster for horizontal scaling
- Separate clusters for different use cases
- Master-slave replication for HA
```

**4. Elasticsearch for Search**
```
WHY ELASTICSEARCH?
✓ Full-text search with relevance scoring
✓ Real-time indexing and search
✓ Faceted search and aggregations
✓ Horizontal scaling with sharding
✓ Rich query DSL for complex searches

WHAT DATA?
- Product search index
- Search analytics and logs
- User behavior tracking
- Inventory availability index

ACCESS PATTERNS:
- Complex search queries with filters
- Aggregations for faceted search
- Real-time indexing of product updates
- Analytics queries

SCALING STRATEGY:
- Shard by product category
- Replicas for search performance
- Separate indexes for different data types
```

### Core Data Models:

```sql
-- PostgreSQL Schema

-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    status VARCHAR(20) DEFAULT 'ACTIVE'
);

-- Addresses table
CREATE TABLE addresses (
    address_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    type VARCHAR(20), -- 'SHIPPING', 'BILLING'
    street_address VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(100),
    postal_code VARCHAR(20),
    country VARCHAR(50),
    is_default BOOLEAN DEFAULT FALSE
);

-- Orders table
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    user_id UUID REFERENCES users(user_id),
    status VARCHAR(20), -- 'PENDING', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED'
    subtotal DECIMAL(10,2),
    tax_amount DECIMAL(10,2),
    shipping_amount DECIMAL(10,2),
    total_amount DECIMAL(10,2),
    currency VARCHAR(3),
    shipping_address_id UUID REFERENCES addresses(address_id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Order items table
CREATE TABLE order_items (
    order_item_id UUID PRIMARY KEY,
    order_id UUID REFERENCES orders(order_id),
    product_id VARCHAR(50),
    quantity INTEGER,
    unit_price DECIMAL(10,2),
    total_price DECIMAL(10,2)
);

-- Inventory table
CREATE TABLE inventory (
    product_id VARCHAR(50) PRIMARY KEY,
    quantity INTEGER NOT NULL DEFAULT 0,
    reserved_quantity INTEGER NOT NULL DEFAULT 0,
    updated_at TIMESTAMP DEFAULT NOW(),
    CONSTRAINT positive_quantity CHECK (quantity >= 0),
    CONSTRAINT positive_reserved CHECK (reserved_quantity >= 0)
);

-- Payments table
CREATE TABLE payments (
    payment_id UUID PRIMARY KEY,
    order_id UUID REFERENCES orders(order_id),
    amount DECIMAL(10,2),
    currency VARCHAR(3),
    payment_method VARCHAR(50),
    payment_provider VARCHAR(50),
    provider_transaction_id VARCHAR(255),
    status VARCHAR(20), -- 'PENDING', 'COMPLETED', 'FAILED', 'REFUNDED'
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

```javascript
// MongoDB Schema (Product Catalog)

// Products collection
{
  "_id": "prod_123456",
  "title": "MacBook Pro 16-inch",
  "description": "Powerful laptop for professionals",
  "category": {
    "primary": "Electronics",
    "secondary": "Computers",
    "tertiary": "Laptops"
  },
  "brand": "Apple",
  "price": {
    "amount": 1999.99,
    "currency": "USD",
    "compareAtPrice": 2199.99
  },
  "attributes": {
    "screenSize": "16 inches",
    "processor": "M2 Pro",
    "memory": "16GB",
    "storage": "512GB SSD",
    "color": "Space Gray"
  },
  "images": [
    {
      "url": "https://cdn.example.com/prod_123456_1.jpg",
      "alt": "MacBook Pro front view",
      "order": 1
    }
  ],
  "seller": {
    "sellerId": "seller_apple",
    "name": "Apple Store",
    "rating": 4.8
  },
  "seo": {
    "slug": "macbook-pro-16-inch-m2-pro",
    "metaTitle": "MacBook Pro 16-inch with M2 Pro",
    "metaDescription": "Professional laptop with M2 Pro chip"
  },
  "inventory": {
    "trackInventory": true,
    "quantity": 50,
    "lowStockThreshold": 5
  },
  "status": "ACTIVE",
  "createdAt": ISODate("2024-01-01T00:00:00Z"),
  "updatedAt": ISODate("2024-01-15T10:30:00Z")
}

// Categories collection
{
  "_id": "cat_electronics",
  "name": "Electronics",
  "slug": "electronics",
  "parentId": null,
  "level": 0,
  "children": ["cat_computers", "cat_phones", "cat_accessories"],
  "image": "https://cdn.example.com/categories/electronics.jpg",
  "seo": {
    "metaTitle": "Electronics - Best Deals Online",
    "metaDescription": "Shop electronics with great prices"
  }
}
```

```json
// Redis Data Structures

// Shopping Cart (Hash)
Key: "cart:user_123"
Value: {
  "prod_123": {"quantity": 2, "addedAt": "2024-01-15T10:00:00Z"},
  "prod_456": {"quantity": 1, "addedAt": "2024-01-15T10:05:00Z"}
}
TTL: 7 days

// Product Cache (Hash)
Key: "product:prod_123"
Value: {
  "title": "MacBook Pro 16-inch",
  "price": 1999.99,
  "availability": "IN_STOCK",
  "imageUrl": "https://cdn.example.com/prod_123_thumb.jpg"
}
TTL: 5 minutes

// User Session (Hash)
Key: "session:sess_abc123"
Value: {
  "userId": "user_123",
  "email": "user@example.com",
  "loginAt": "2024-01-15T09:00:00Z",
  "lastActivity": "2024-01-15T10:30:00Z"
}
TTL: 24 hours

// Inventory Cache (String)
Key: "inventory:prod_123"
Value: "IN_STOCK"  // or "OUT_OF_STOCK", "LOW_STOCK"
TTL: 1 minute
```

### Data Partitioning Strategy:

**1. Geographic Partitioning**
```
STRATEGY:
- Partition user data by geographic region
- Keep orders close to users for compliance
- Replicate product catalog globally

BENEFITS:
- Reduced latency for users
- Regulatory compliance (GDPR, data residency)
- Disaster recovery isolation
```

**2. Functional Partitioning**
```
STRATEGY:
- Separate read-heavy (catalog) from write-heavy (orders) data
- Different databases optimized for different workloads
- Independent scaling of different data types

BENEFITS:
- Optimized performance per use case
- Independent scaling and maintenance
- Technology diversity
```

**3. Time-based Partitioning**
```
STRATEGY:
- Partition order data by date (monthly tables)
- Archive old data to cheaper storage
- Keep recent data in fast storage

BENEFITS:
- Improved query performance
- Cost optimization
- Simplified data lifecycle management
```

---

## PHASE 7: CORE DATA FLOWS (7 minutes)

### What to Say:

"Let me walk through the critical data flows, explaining how data moves through our system and the architectural decisions that ensure performance and consistency."

### Flow 1: Product Search & Browse

```
DATA FLOW:
User → CDN → API Gateway → Search Service → Elasticsearch → Product Service → MongoDB

DETAILED STEPS:
1. User types "laptop" in search box
2. Frontend sends GET /api/v1/products/search?q=laptop
3. CDN checks for cached search results (cache miss for personalized results)
4. API Gateway authenticates request and routes to Search Service
5. Search Service queries Elasticsearch with full-text search
6. Elasticsearch returns product IDs ranked by relevance
7. Search Service calls Product Service for detailed product info
8. Product Service checks Redis cache for product details
   - Cache hit: Return cached data
   - Cache miss: Query MongoDB, cache result, return data
9. Search Service aggregates results and returns to client
10. CDN caches non-personalized parts of response

WHY THIS FLOW?
- Elasticsearch optimized for complex search queries
- Redis cache reduces MongoDB load for hot products
- CDN reduces latency for static search results
- Separation allows independent scaling of search vs catalog

PERFORMANCE OPTIMIZATIONS:
- Search result caching for popular queries (5-minute TTL)
- Product detail caching (30-minute TTL)
- Async index updates to Elasticsearch
- Connection pooling between services

FAILURE HANDLING:
- Elasticsearch down: Fallback to MongoDB with basic search
- MongoDB down: Serve stale data from Redis cache
- Redis down: Direct MongoDB queries (degraded performance)
```

### Flow 2: Shopping Cart Management

```
DATA FLOW:
User → API Gateway → Cart Service → Redis → (Optional) User Service

DETAILED STEPS:
1. User clicks "Add to Cart" on product page
2. Frontend sends POST /api/v1/cart/items with productId and quantity
3. API Gateway extracts userId from JWT token
4. Cart Service validates product exists and is available
5. Cart Service updates Redis hash for user's cart
   - Key: "cart:user_123"
   - Add/update product with quantity and timestamp
6. Cart Service calculates cart totals (subtotal, tax, shipping)
7. Response includes updated cart summary
8. For logged-in users: Optionally sync to persistent storage

WHY THIS FLOW?
- Redis provides sub-millisecond cart operations
- In-memory storage perfect for temporary cart data
- TTL automatically cleans up abandoned carts
- No database writes needed for cart operations

CART PERSISTENCE STRATEGY:
- Anonymous users: Redis only (7-day TTL)
- Logged-in users: Redis primary, periodic sync to PostgreSQL
- Cross-device sync: Load from PostgreSQL on login

CONSISTENCY CONSIDERATIONS:
- Cart operations are eventually consistent
- Inventory validation happens at checkout, not cart add
- Price validation happens at checkout with current prices
```

### Flow 3: Checkout & Order Processing (The Critical Path)

```
DATA FLOW:
User → API Gateway → Order Service → Inventory Service → Payment Service → Notification Service

DETAILED STEPS:
1. User clicks "Place Order" on checkout page
2. Frontend sends POST /api/v1/orders with idempotency key
3. API Gateway validates authentication and routes to Order Service
4. Order Service begins distributed transaction:

   STEP 4A: Order Creation
   - Create order record in PostgreSQL with status "PENDING"
   - Generate unique order ID
   - Validate shipping address and payment method

   STEP 4B: Inventory Reservation (Critical Section)
   - Order Service calls Inventory Service for each cart item
   - Inventory Service executes atomic SQL:
     UPDATE inventory 
     SET reserved_quantity = reserved_quantity + ?
     WHERE product_id = ? AND (quantity - reserved_quantity) >= ?
   - If any item fails reservation, rollback entire order

   STEP 4C: Payment Processing
   - Order Service calls Payment Service with order total
   - Payment Service calls Stripe API to charge customer
   - Store payment transaction record with provider response

   STEP 4D: Order Confirmation
   - If payment succeeds: Update order status to "CONFIRMED"
   - Update inventory: quantity = quantity - reserved_quantity
   - Clear user's cart from Redis
   - Publish order event to Kafka for downstream processing

   STEP 4E: Failure Handling
   - If payment fails: Release inventory reservations
   - Update order status to "FAILED"
   - Return error to user with specific failure reason

5. Async processing (via Kafka events):
   - Send order confirmation email
   - Update search index with new inventory levels
   - Trigger fulfillment process
   - Update analytics and reporting

WHY THIS FLOW?
- Strong consistency prevents overselling
- Idempotency prevents duplicate orders
- Atomic operations ensure data integrity
- Async processing improves response time

CONCURRENCY HANDLING:
- Database row-level locking for inventory updates
- Optimistic locking with version numbers
- Retry logic with exponential backoff
- Circuit breaker for external payment API

FAILURE SCENARIOS:
- Inventory unavailable: Return 409 Conflict immediately
- Payment failure: Rollback inventory, return 402 Payment Required
- Service timeout: Use saga pattern for distributed rollback
- Database failure: Queue order for retry when service recovers
```

### Flow 4: Real-time Inventory Updates

```
DATA FLOW:
Inventory Service → Kafka → Search Service → Elasticsearch
                         → Product Service → Redis Cache
                         → Notification Service → WebSocket

DETAILED STEPS:
1. Inventory quantity changes (order, restock, adjustment)
2. Inventory Service publishes event to Kafka:
   {
     "eventType": "INVENTORY_UPDATED",
     "productId": "prod_123",
     "oldQuantity": 10,
     "newQuantity": 5,
     "timestamp": "2024-01-15T10:30:00Z"
   }

3. Multiple consumers process the event:

   SEARCH SERVICE:
   - Updates Elasticsearch index with new availability status
   - Refreshes search results for affected categories
   - Updates facet counts for availability filters

   PRODUCT SERVICE:
   - Invalidates Redis cache for affected product
   - Updates availability flags in cache
   - Refreshes product detail pages

   NOTIFICATION SERVICE:
   - Sends low-stock alerts to sellers
   - Notifies users with items in wishlist
   - Updates real-time dashboards via WebSocket

WHY THIS FLOW?
- Event-driven architecture ensures all systems stay synchronized
- Kafka provides durability and replay capability
- Async processing doesn't slow down order processing
- Multiple consumers can process same event independently

CONSISTENCY TRADE-OFFS:
- Search results may show stale inventory (eventual consistency)
- Product pages validate inventory at checkout (strong consistency)
- Acceptable delay: 1-2 seconds for inventory updates to propagate
```

### Data Consistency Strategy:

**1. Strong Consistency (ACID)**
```
USE CASES:
- Order processing and payment
- Inventory reservations and updates
- User account and financial data

IMPLEMENTATION:
- PostgreSQL transactions with proper isolation levels
- Row-level locking for inventory updates
- Two-phase commit for distributed transactions
```

**2. Eventual Consistency**
```
USE CASES:
- Product catalog updates
- Search index synchronization
- Cache invalidation
- Analytics and reporting

IMPLEMENTATION:
- Event-driven updates via Kafka
- Async processing with retry logic
- Conflict resolution strategies
- Monitoring for consistency lag
```

**3. Session Consistency**
```
USE CASES:
- Shopping cart operations
- User preferences and settings
- Browsing history and recommendations

IMPLEMENTATION:
- Sticky sessions for cart operations
- Read-your-own-writes guarantee
- Session-scoped caching
```

---

## PHASE 8: DEEP DIVE - INVENTORY MANAGEMENT & CONSISTENCY (5 minutes)

### What to Say:

"Inventory management is the most critical part of our system. We must prevent overselling while maintaining high performance during flash sales. This requires careful design of our concurrency control and consistency mechanisms."

### The Inventory Challenge:

**Problem Statement:**
- Prevent overselling (selling more items than available)
- Handle high concurrency during flash sales (10K+ orders/sec)
- Maintain performance while ensuring data consistency
- Support inventory reservations during checkout process

### Inventory Architecture:

**Database Design:**
```sql
-- Inventory table with optimistic locking
CREATE TABLE inventory (
    product_id VARCHAR(50) PRIMARY KEY,
    quantity INTEGER NOT NULL DEFAULT 0,
    reserved_quantity INTEGER NOT NULL DEFAULT 0,
    version INTEGER NOT NULL DEFAULT 1,
    updated_at TIMESTAMP DEFAULT NOW(),
    
    CONSTRAINT positive_quantity CHECK (quantity >= 0),
    CONSTRAINT positive_reserved CHECK (reserved_quantity >= 0),
    CONSTRAINT available_quantity CHECK (quantity >= reserved_quantity)
);

-- Index for fast lookups
CREATE INDEX idx_inventory_product_id ON inventory(product_id);
CREATE INDEX idx_inventory_low_stock ON inventory(quantity) WHERE quantity < 10;
```

### Concurrency Control Strategies:

**Approach 1: Pessimistic Locking (❌ Poor for High Concurrency)**
```sql
-- BAD: Locks row for entire transaction duration
BEGIN;
SELECT quantity FROM inventory WHERE product_id = 'prod_123' FOR UPDATE;
-- User fills out payment form (30+ seconds)
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 'prod_123';
COMMIT;

PROBLEMS:
- Long-running locks reduce concurrency
- Deadlock potential with multiple products
- Poor user experience during checkout delays
- Database connection exhaustion
```

**Approach 2: Optimistic Locking with Reservations (✅ Good Solution)**
```sql
-- GOOD: Short transactions with reservation system
-- Step 1: Reserve inventory
UPDATE inventory 
SET reserved_quantity = reserved_quantity + ?, 
    version = version + 1
WHERE product_id = ? 
  AND version = ?
  AND (quantity - reserved_quantity) >= ?;

-- Step 2: Complete order (after payment)
UPDATE inventory 
SET quantity = quantity - ?,
    reserved_quantity = reserved_quantity - ?,
    version = version + 1
WHERE product_id = ?;

-- Step 3: Release reservation (if payment fails)
UPDATE inventory 
SET reserved_quantity = reserved_quantity - ?,
    version = version + 1
WHERE product_id = ?;

BENEFITS:
- Short transaction duration
- No long-running locks
- Handles payment processing delays
- Automatic cleanup of expired reservations
```

**Approach 3: Redis-based Flash Sale Optimization (✅ Great for Hot Items)**
```
PROBLEM: Single hot product creates database bottleneck
SOLUTION: Use Redis for high-frequency inventory operations

REDIS SETUP:
- Load available inventory into Redis: SET inventory:prod_123 100
- Use Lua script for atomic decrement operations
- Async sync to PostgreSQL for persistence

LUA SCRIPT:
local key = KEYS[1]
local quantity = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or 0)

if current >= quantity then
    redis.call('DECRBY', key, quantity)
    return current - quantity
else
    return -1  -- Insufficient inventory
end

BENEFITS:
- Sub-millisecond inventory checks
- Handles 100K+ operations per second
- Reduces database load during flash sales
- Eventual consistency with database
```

### Inventory Reservation Flow:

```
RESERVATION PROCESS:
1. User initiates checkout
2. Order Service calls Inventory Service to reserve items
3. Inventory Service updates reserved_quantity atomically
4. Reservation expires after 10 minutes if not confirmed
5. Payment processing happens with reserved inventory
6. On success: Convert reservation to actual sale
7. On failure: Release reservation back to available pool

RESERVATION CLEANUP:
- Background job runs every minute
- Finds reservations older than 10 minutes
- Releases expired reservations automatically
- Prevents inventory from being locked indefinitely

MONITORING:
- Track reservation success/failure rates
- Monitor reservation expiration rates
- Alert on inventory discrepancies
- Dashboard for real-time inventory levels
```

### Flash Sale Architecture:

```
FLASH SALE CHALLENGES:
- 50K users trying to buy 100 items simultaneously
- Database hot-spot on single inventory row
- Need sub-second response times
- Prevent overselling under extreme load

SOLUTION: Multi-tier Inventory System

TIER 1: Redis (Hot Path)
- Pre-load flash sale inventory into Redis
- Use atomic Lua scripts for inventory decrements
- Handle 100K+ requests per second
- Immediate response to users

TIER 2: Database (Persistent)
- Async sync from Redis to PostgreSQL
- Maintain authoritative inventory record
- Handle complex inventory operations
- Support reporting and analytics

TIER 3: Queue System (Overflow)
- Queue excess requests during traffic spikes
- Process queued orders when system capacity available
- Prevent system overload
- Fair ordering for customers

IMPLEMENTATION:
```

```java
// Flash Sale Inventory Service
@Service
public class FlashSaleInventoryService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    private static final String INVENTORY_SCRIPT = 
        "local key = KEYS[1] " +
        "local quantity = tonumber(ARGV[1]) " +
        "local current = tonumber(redis.call('GET', key) or 0) " +
        "if current >= quantity then " +
        "    redis.call('DECRBY', key, quantity) " +
        "    return current - quantity " +
        "else " +
        "    return -1 " +
        "end";
    
    public boolean reserveInventory(String productId, int quantity) {
        String key = "flash_inventory:" + productId;
        
        Long result = redisTemplate.execute(
            RedisScript.of(INVENTORY_SCRIPT, Long.class),
            Collections.singletonList(key),
            String.valueOf(quantity)
        );
        
        if (result != null && result >= 0) {
            // Async sync to database
            asyncSyncToDatabase(productId, quantity);
            return true;
        }
        
        return false; // Insufficient inventory
    }
    
    @Async
    private void asyncSyncToDatabase(String productId, int quantity) {
        // Update PostgreSQL with inventory change
        inventoryRepository.decrementQuantity(productId, quantity);
    }
}
```

### Consistency Monitoring:

```
CONSISTENCY CHECKS:
1. Periodic reconciliation between Redis and PostgreSQL
2. Alert on inventory discrepancies > threshold
3. Automatic correction for small discrepancies
4. Manual review for large discrepancies

METRICS TO TRACK:
- Inventory reservation success rate
- Average reservation duration
- Expired reservation percentage
- Database vs Redis inventory drift
- Flash sale conversion rates

ALERTING:
- Inventory goes negative (critical bug)
- High reservation expiration rate (UX issue)
- Large Redis/DB discrepancy (consistency issue)
- Flash sale inventory exhausted (business alert)
```

---

## PHASE 9: DEEP DIVE - SEARCH & CACHING STRATEGY (5 minutes)

### What to Say:

"Search is critical for discovery and conversion. With 48K search QPS at peak, we need a sophisticated caching strategy and search architecture that can handle complex queries while maintaining sub-200ms response times."

### Search Architecture:

**Multi-layer Search System:**
```
LAYER 1: CDN Cache (Global Edge)
- Cache popular search results
- Geographic distribution
- 90%+ cache hit rate for common searches
- TTL: 5 minutes for non-personalized results

LAYER 2: Application Cache (Redis)
- Cache search results and facets
- User-specific search history
- Auto-complete suggestions
- TTL: 2 minutes for search results

LAYER 3: Elasticsearch (Search Engine)
- Full-text search with relevance scoring
- Faceted search and aggregations
- Real-time indexing
- Horizontal scaling with sharding

LAYER 4: MongoDB (Source of Truth)
- Complete product catalog
- Authoritative product data
- Complex product relationships
- Backup for search failures
```

### Elasticsearch Index Strategy:

**Index Design:**
```json
// Product Search Index
{
  "mappings": {
    "properties": {
      "productId": {"type": "keyword"},
      "title": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": {"type": "keyword"},
          "suggest": {"type": "completion"}
        }
      },
      "description": {"type": "text", "analyzer": "standard"},
      "category": {
        "type": "nested",
        "properties": {
          "primary": {"type": "keyword"},
          "secondary": {"type": "keyword"},
          "tertiary": {"type": "keyword"}
        }
      },
      "brand": {"type": "keyword"},
      "price": {"type": "double"},
      "rating": {"type": "double"},
      "reviewCount": {"type": "integer"},
      "availability": {"type": "keyword"},
      "attributes": {
        "type": "nested",
        "dynamic": true
      },
      "seller": {
        "properties": {
          "sellerId": {"type": "keyword"},
          "name": {"type": "keyword"},
          "rating": {"type": "double"}
        }
      },
      "createdAt": {"type": "date"},
      "updatedAt": {"type": "date"}
    }
  }
}
```

**Indexing Strategy:**
```
REAL-TIME INDEXING:
- Change Data Capture (CDC) from MongoDB
- Kafka streams for index updates
- Bulk indexing for better performance
- Separate indexes for different product types

SHARDING STRATEGY:
- Shard by product category for query locality
- 6 primary shards, 2 replicas each
- Hot/warm architecture for old vs new products
- Cross-cluster replication for disaster recovery

PERFORMANCE OPTIMIZATION:
- Custom analyzers for product-specific terms
- Synonym filters for brand names and categories
- Boost factors for popular products
- Query result caching at Elasticsearch level
```

### Advanced Search Features:

**1. Faceted Search Implementation:**
```json
// Search Query with Facets
{
  "query": {
    "bool": {
      "must": [
        {"multi_match": {
          "query": "laptop",
          "fields": ["title^3", "description", "brand^2"]
        }}
      ],
      "filter": [
        {"range": {"price": {"gte": 500, "lte": 2000}}},
        {"term": {"category.primary": "Electronics"}},
        {"term": {"availability": "IN_STOCK"}}
      ]
    }
  },
  "aggs": {
    "brands": {
      "terms": {"field": "brand", "size": 20}
    },
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          {"to": 500},
          {"from": 500, "to": 1000},
          {"from": 1000, "to": 2000},
          {"from": 2000}
        ]
      }
    },
    "categories": {
      "nested": {"path": "category"},
      "aggs": {
        "secondary": {
          "terms": {"field": "category.secondary"}
        }
      }
    }
  },
  "sort": [
    {"_score": {"order": "desc"}},
    {"rating": {"order": "desc"}},
    {"reviewCount": {"order": "desc"}}
  ]
}
```

**2. Auto-complete and Suggestions:**
```
IMPLEMENTATION:
- Elasticsearch completion suggester
- Prefix matching with fuzzy tolerance
- Popular search terms weighted higher
- Real-time updates from search analytics

CACHING STRATEGY:
- Cache suggestions in Redis with 1-hour TTL
- Pre-compute popular suggestions
- User-specific suggestion history
- A/B testing for suggestion algorithms
```

### Caching Strategy Deep Dive:

**1. Multi-level Caching Architecture:**
```
L1 CACHE: Browser Cache
- Static assets (images, CSS, JS)
- API responses with appropriate headers
- Cache-Control: max-age=300 for product data

L2 CACHE: CDN (CloudFront)
- Product images and static content
- Popular search results (non-personalized)
- API responses for anonymous users
- Geographic distribution for global users

L3 CACHE: Application Cache (Redis)
- Product details and pricing
- Search results and facets
- User sessions and preferences
- Shopping cart contents

L4 CACHE: Database Query Cache
- MongoDB query result cache
- PostgreSQL shared buffer cache
- Elasticsearch query cache
- Connection pooling
```

**2. Cache Invalidation Strategy:**
```
CACHE INVALIDATION PATTERNS:

Time-based (TTL):
- Product details: 5 minutes
- Search results: 2 minutes
- User sessions: 24 hours
- Shopping carts: 7 days

Event-based:
- Product updates → Invalidate product cache
- Inventory changes → Invalidate search cache
- Price changes → Invalidate category cache
- User actions → Invalidate user-specific cache

Write-through:
- Critical data written to cache and database
- Ensures cache consistency
- Higher write latency but consistent reads

Write-behind:
- Non-critical data written to cache first
- Async write to database
- Better write performance
- Risk of data loss
```

**3. Cache Performance Optimization:**
```java
// Redis Cache Implementation
@Service
public class ProductCacheService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Cacheable(value = "products", key = "#productId", unless = "#result == null")
    public Product getProduct(String productId) {
        // Cache miss - fetch from database
        return productRepository.findById(productId);
    }
    
    @CacheEvict(value = "products", key = "#productId")
    public void invalidateProduct(String productId) {
        // Explicit cache invalidation
    }
    
    @Cacheable(value = "search", key = "#query.hashCode()", unless = "#result.isEmpty()")
    public SearchResult searchProducts(SearchQuery query) {
        // Cache search results for popular queries
        return elasticsearchService.search(query);
    }
    
    // Batch cache warming for popular products
    @Scheduled(fixedRate = 300000) // Every 5 minutes
    public void warmCache() {
        List<String> popularProducts = analyticsService.getPopularProducts();
        popularProducts.forEach(this::getProduct);
    }
}
```

### Search Performance Monitoring:

**Key Metrics:**
```
SEARCH PERFORMANCE:
- Average search response time (target: <200ms)
- Search result relevance score
- Zero-result search percentage
- Search-to-purchase conversion rate

CACHE PERFORMANCE:
- Cache hit ratio by layer (target: >90%)
- Cache miss penalty (time to fetch from source)
- Cache invalidation frequency
- Memory usage and eviction rates

ELASTICSEARCH METRICS:
- Query latency percentiles (P50, P95, P99)
- Index size and growth rate
- Shard allocation and balance
- Cluster health and node performance
```

**Performance Optimization Techniques:**
```
QUERY OPTIMIZATION:
- Use filters instead of queries when possible
- Limit result size and use pagination
- Optimize aggregations with sampling
- Use query profiling to identify bottlenecks

INDEX OPTIMIZATION:
- Regular index optimization and merging
- Hot/warm/cold architecture for data lifecycle
- Custom routing for query locality
- Index templates for consistent mapping

CACHING OPTIMIZATION:
- Cache warming for popular content
- Intelligent cache eviction policies
- Compression for large cached objects
- Monitoring and alerting for cache performance
```

---

## FINAL ARCHITECTURE SUMMARY

### Technology Stack Justification:

```
┌─────────────────────────┬──────────────────┬──────────────────┐
│ Component               │ Technology       │ Justification    │
├─────────────────────────┼──────────────────┼──────────────────┤
│ Product Catalog         │ MongoDB          │ Flexible schema  │
│ Transactional Data      │ PostgreSQL       │ ACID compliance  │
│ Search Engine           │ Elasticsearch    │ Full-text search │
│ Caching Layer           │ Redis Cluster    │ Sub-ms latency   │
│ Message Queue           │ Apache Kafka     │ Event streaming  │
│ Load Balancer           │ AWS ALB          │ Auto-scaling     │
│ CDN                     │ CloudFront       │ Global delivery  │
│ Object Storage          │ Amazon S3        │ Image storage    │
│ Payment Processing      │ Stripe           │ PCI compliance   │
│ Monitoring              │ Prometheus       │ Metrics & alerts │
└─────────────────────────┴──────────────────┴──────────────────┘
```

### Scalability Characteristics:

```
HORIZONTAL SCALING:
- Microservices: Independent scaling per service
- Database sharding: By user ID, product category, geography
- Cache clustering: Redis Cluster with automatic sharding
- Search scaling: Elasticsearch cluster with replica shards

PERFORMANCE TARGETS:
- 99.99% availability for storefront (52 minutes downtime/year)
- 99.9% availability for checkout (8.76 hours downtime/year)
- <200ms search response time (P95)
- <500ms product page load time (P95)
- 60K QPS peak capacity with auto-scaling
- 1M orders per day processing capacity

COST OPTIMIZATION:
- CDN reduces bandwidth costs by 35%
- Caching reduces database load by 90%
- Auto-scaling reduces compute costs during low traffic
- Reserved instances for baseline capacity
```

This comprehensive e-commerce platform architecture provides a production-ready foundation that can handle Amazon-scale traffic with proper consideration for performance, consistency, scalability, and cost optimization.