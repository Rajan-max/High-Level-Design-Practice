# Ticketmaster/BookMyShow System - Complete HLD Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: High-Level Architecture (8 min)
Phase 5: API Design (3 min)
Phase 6: Data Models & Core Entities (4 min)
Phase 7: Core Flows - View/Search/Book (7 min)
Phase 8: Deep Dive - Ticket Reservation & Concurrency (5 min)
Phase 9: Deep Dive - Scaling & Performance (5 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"I'll design a ticket booking platform like Ticketmaster that handles high-traffic events. Let me clarify the scope and requirements."

### Questions to Ask:

**Q1: Core Use Cases**
- "Are we focusing on ticket booking, or do we need event management too?"
- "Should we support different event types (concerts, sports, theater)?"

**Expected Answer:** Focus on ticket booking for various event types, event management is secondary

**Q2: User Types**
- "Who are our users - end customers, event organizers, or both?"
- "Do we need admin/organizer dashboards?"

**Expected Answer:** Primary focus on end customers booking tickets, basic organizer features

**Q3: Scale & Traffic Patterns**
- "What's the expected scale - concurrent users, events, tickets?"
- "Are there traffic spikes during popular event releases?"

**Expected Answer:** 10M+ concurrent users during popular events, millions of events globally

**Q4: Geographic Distribution**
- "Is this a global platform or region-specific?"
- "Do we need multi-currency, multi-language support?"

**Expected Answer:** Global platform, focus on core functionality first

**Q5: Payment & Pricing**
- "Do we handle payments directly or integrate with payment processors?"
- "Should we support dynamic pricing based on demand?"

**Expected Answer:** Integrate with payment processors (Stripe), dynamic pricing out of scope

**Q6: Ticket Types & Seating**
- "Do we need to support reserved seating with seat maps?"
- "What about general admission tickets?"

**Expected Answer:** Support both reserved seating and general admission

**Q7: Mobile vs Web**
- "Should we optimize for mobile apps, web, or both?"
- "Any offline capabilities needed?"

**Expected Answer:** Both mobile and web, no offline requirements

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### Functional Requirements (Priority Order):

```
CORE REQUIREMENTS (Above the line):

1. View Events (CRITICAL)
   - Browse events by location, date, category
   - View event details (venue, performers, pricing)
   - Display seat map with availability
   - Show real-time ticket availability

2. Search Events (CRITICAL)
   - Search by keywords, artist, venue, location
   - Filter by date range, price range, category
   - Auto-complete suggestions
   - Geo-based search results

3. Book Tickets (CRITICAL)
   - Select seats from interactive seat map
   - Reserve tickets during checkout (10-min timer)
   - Process payments securely
   - Prevent double booking (strong consistency)
   - Generate booking confirmation

IMPORTANT REQUIREMENTS:

4. User Management
   - User registration/login
   - Profile management
   - Booking history

5. Inventory Management
   - Real-time seat availability updates
   - Automatic ticket release after reservation timeout
   - Support for different ticket types/pricing tiers

BELOW THE LINE (Out of scope):
- Event creation/management by organizers
- Dynamic pricing algorithms
- Resale marketplace
- Mobile app push notifications
- Social features (sharing, reviews)
- Analytics dashboard
- Multi-language support
```

### Non-Functional Requirements:

```
PERFORMANCE:
- Latency: < 500ms for search, < 200ms for booking operations
- Throughput: Handle 10M concurrent users during peak events
- Availability: 99.9% uptime (8.76 hours downtime/year)

SCALABILITY:
- Support 1M+ events globally
- Handle 100M+ tickets in system
- Linear scaling with traffic growth
- Auto-scaling during traffic spikes

CONSISTENCY & RELIABILITY:
- Strong consistency for booking operations (no double booking)
- Eventual consistency acceptable for search/browse
- ACID transactions for payment processing
- Data durability for bookings and payments

SECURITY:
- PCI DSS compliance for payment processing
- Secure user authentication (OAuth 2.0)
- Rate limiting to prevent abuse
- DDoS protection

USABILITY:
- Mobile-responsive design
- Intuitive seat selection interface
- Clear booking flow with progress indicators
- Real-time updates during booking process
```

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### Scale Assumptions:

```
TRAFFIC PATTERNS:
- Peak concurrent users: 10 million (popular event release)
- Average concurrent users: 100,000
- Read:Write ratio: 95:5 (browse heavy, booking light)
- Peak QPS: 500K (10M users, avg 0.05 requests/sec)
- Average QPS: 5K

DATA VOLUME:
- Total events: 1 million active events
- Average tickets per event: 10,000
- Total tickets in system: 10 billion
- Average event data: 10 KB
- Average ticket data: 500 bytes
- User profiles: 100 million users, 2 KB each
```

### Storage Estimation:

```
PRIMARY DATA:
Events: 1M × 10 KB = 10 GB
Tickets: 10B × 500 bytes = 5 TB
Users: 100M × 2 KB = 200 GB
Bookings: 1B bookings × 1 KB = 1 TB
Venues: 100K venues × 50 KB = 5 GB

Total Primary Data: ~6.2 TB
With replication (3x): ~18.6 TB
With indexes and overhead: ~25 TB

CACHE REQUIREMENTS:
Hot events (top 1%): 10K events × 10 KB = 100 MB
Popular searches: 1 GB
Seat maps: 10K events × 1 MB = 10 GB
Total cache: ~12 GB per region
```

### Throughput Analysis:

```
PEAK TRAFFIC BREAKDOWN:
Total QPS: 500K
- Search/Browse: 475K QPS (95%)
- Booking operations: 25K QPS (5%)

PER-SERVICE QPS:
Search Service: 475K QPS
- Can be cached aggressively
- Read replicas can handle load

Booking Service: 25K QPS
- Requires strong consistency
- Database bottleneck
- Need connection pooling

Event Service: 200K QPS
- Event details, seat maps
- Heavy caching candidate

User Service: 50K QPS
- Authentication, profiles
- Session management
```

### Database Sizing:

```
BOOKING DATABASE (Critical Path):
- Write QPS: 25K (bookings)
- Read QPS: 50K (booking status, history)
- Total QPS per DB: 75K

PostgreSQL capacity: ~10K QPS per instance
Required instances: 75K / 10K = 8 instances
With replication: 8 primary + 16 read replicas = 24 instances


SEARCH DATABASE:
- Elasticsearch cluster
- 3 master nodes, 6 data nodes
- Handle 475K search QPS with caching

CACHE LAYER:
- Redis cluster: 12 GB × 3 regions = 36 GB
- 6 Redis instances (6 GB each)
- Handle 400K+ cached requests
```

### Cost Estimation (AWS):

```
COMPUTE:
- Application servers: 50 × c5.2xlarge = $15K/month
- Database instances: 24 × r5.xlarge = $12K/month
- Cache instances: 6 × r5.large = $1K/month

STORAGE:
- RDS storage: 25 TB × $0.115 = $2.9K/month
- S3 backups: 10 TB × $0.023 = $230/month

NETWORK:
- Data transfer: 100 TB × $0.09 = $9K/month
- CloudFront CDN: $2K/month

Total: ~$42K/month (~$500K/year)
```

---

## PHASE 4: HIGH-LEVEL ARCHITECTURE (8 minutes)

### Complete System Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│  Web App (React)  │  Mobile App (React Native)  │  Admin Panel │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                         CDN + LOAD BALANCER                     │
├─────────────────────────────────────────────────────────────────┤
│  CloudFront CDN  │  Application Load Balancer  │  Rate Limiter │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                        API GATEWAY                              │
├─────────────────────────────────────────────────────────────────┤
│  Authentication  │  Request Routing  │  Circuit Breaker        │
└─────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES LAYER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   EVENT     │  │   SEARCH    │  │   BOOKING   │             │
│  │  SERVICE    │  │  SERVICE    │  │  SERVICE    │             │
│  │             │  │             │  │             │             │
│  │ - View      │  │ - Keyword   │  │ - Reserve   │             │
│  │ - Details   │  │ - Filter    │  │ - Confirm   │             │
│  │ - Seat Map  │  │ - Suggest   │  │ - Cancel    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    USER     │  │  PAYMENT    │  │ NOTIFICATION│             │
│  │  SERVICE    │  │  SERVICE    │  │  SERVICE    │             │
│  │             │  │             │  │             │             │
│  │ - Auth      │  │ - Process   │  │ - Email     │             │
│  │ - Profile   │  │ - Refund    │  │ - SMS       │             │
│  │ - History   │  │ - Webhook   │  │ - Push      │             │
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
│  │ POSTGRESQL  │  │ELASTICSEARCH│  │    REDIS    │             │
│  │  CLUSTER    │  │   CLUSTER   │  │   CLUSTER   │             │
│  │             │  │             │  │             │             │
│  │ - Events    │  │ - Search    │  │ - Cache     │             │
│  │ - Tickets   │  │ - Index     │  │ - Session   │             │
│  │ - Bookings  │  │ - Analytics │  │ - Lock      │             │
│  │ - Users     │  │             │  │             │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    S3       │  │   STRIPE    │  │   KAFKA     │             │
│  │  STORAGE    │  │  PAYMENT    │  │ EVENT BUS   │             │
│  │             │  │             │  │             │             │
│  │ - Images    │  │ - Process   │  │ - Events    │             │
│  │ - Backups   │  │ - Webhooks  │  │ - Logs      │             │
│  │ - Logs      │  │             │  │ - Metrics   │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

### Key Architectural Decisions:

**1. Microservices Architecture**
- Independent scaling of services
- Technology diversity (Java, Node.js, Python)
- Fault isolation and resilience

**2. Database Per Service**
- PostgreSQL for transactional data (bookings, users)
- Elasticsearch for search and analytics
- Redis for caching and distributed locks

**3. Event-Driven Architecture**
- Kafka for async communication
- Event sourcing for booking state changes
- CQRS for read/write separation

**4. Caching Strategy**
- CDN for static assets (images, CSS, JS)
- Redis for application cache
- Database query result caching

**5. Consistency Model**
- Strong consistency for booking operations
- Eventual consistency for search/browse
- Distributed transactions where necessary

---

## PHASE 5: API DESIGN (3 minutes)

### RESTful API Endpoints:

```java
// Event Management APIs
GET    /api/v1/events/{eventId}                    // Get event details
GET    /api/v1/events/{eventId}/seats              // Get seat map
GET    /api/v1/events/search                       // Search events
GET    /api/v1/events/categories                   // Get categories
GET    /api/v1/venues/{venueId}                    // Get venue details

// Booking APIs  
POST   /api/v1/bookings/reserve                    // Reserve tickets
POST   /api/v1/bookings/{bookingId}/confirm        // Confirm booking
DELETE /api/v1/bookings/{bookingId}                // Cancel booking
GET    /api/v1/bookings/{bookingId}                // Get booking status
GET    /api/v1/users/{userId}/bookings             // Get user bookings

// User Management APIs
POST   /api/v1/auth/login                          // User login
POST   /api/v1/auth/register                       // User registration
GET    /api/v1/users/{userId}/profile              // Get user profile
PUT    /api/v1/users/{userId}/profile              // Update profile

// Payment APIs
POST   /api/v1/payments/process                    // Process payment
POST   /api/v1/payments/webhooks/stripe            // Payment webhooks
GET    /api/v1/payments/{paymentId}/status         // Payment status
```

### API Request/Response Examples:

```java
// Search Events API
GET /api/v1/events/search?keyword=taylor&city=newyork&date=2024-06-15&page=1&size=20

Response:
{
  "events": [
    {
      "eventId": "evt_123",
      "title": "Taylor Swift - Eras Tour",
      "description": "The most anticipated concert of the year",
      "category": "CONCERT",
      "startDateTime": "2024-06-15T20:00:00Z",
      "venue": {
        "venueId": "venue_456",
        "name": "Madison Square Garden",
        "address": "4 Pennsylvania Plaza, New York, NY 10001",
        "capacity": 20000
      },
      "performers": [
        {
          "performerId": "perf_789",
          "name": "Taylor Swift",
          "type": "ARTIST"
        }
      ],
      "priceRange": {
        "min": 89.99,
        "max": 599.99,
        "currency": "USD"
      },
      "availableTickets": 1250,
      "totalTickets": 20000
    }
  ],
  "pagination": {
    "page": 1,
    "size": 20,
    "totalElements": 156,
    "totalPages": 8
  }
}

// Reserve Tickets API
POST /api/v1/bookings/reserve
{
  "eventId": "evt_123",
  "ticketRequests": [
    {
      "sectionId": "sec_A1",
      "row": "15",
      "seatNumber": "12",
      "ticketType": "PREMIUM",
      "price": 299.99
    },
    {
      "sectionId": "sec_A1", 
      "row": "15",
      "seatNumber": "13",
      "ticketType": "PREMIUM",
      "price": 299.99
    }
  ],
  "userId": "user_456"
}

Response:
{
  "bookingId": "booking_789",
  "status": "RESERVED",
  "reservationExpiresAt": "2024-01-15T14:10:00Z",
  "totalAmount": 599.98,
  "currency": "USD",
  "tickets": [
    {
      "ticketId": "ticket_001",
      "sectionId": "sec_A1",
      "row": "15", 
      "seatNumber": "12",
      "price": 299.99,
      "status": "RESERVED"
    },
    {
      "ticketId": "ticket_002",
      "sectionId": "sec_A1",
      "row": "15",
      "seatNumber": "13", 
      "price": 299.99,
      "status": "RESERVED"
    }
  ]
}
```

---

## PHASE 6: DATA MODELS & CORE ENTITIES (4 minutes)

### Database Schema Design:

```java
// Event Entity
@Entity
@Table(name = "events")
public class Event {
    @Id
    private String eventId;
    
    @Column(nullable = false)
    private String title;
    
    @Column(length = 2000)
    private String description;
    
    @Enumerated(EnumType.STRING)
    private EventCategory category;
    
    @Column(nullable = false)
    private LocalDateTime startDateTime;
    
    private LocalDateTime endDateTime;
    
    @Column(nullable = false)
    private String venueId;
    
    @Enumerated(EnumType.STRING)
    private EventStatus status;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    
    // Getters, setters, constructors
}

// Venue Entity
@Entity
@Table(name = "venues")
public class Venue {
    @Id
    private String venueId;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private String address;
    
    private String city;
    private String state;
    private String country;
    private String zipCode;
    
    @Column(nullable = false)
    private Integer capacity;
    
    @Column(length = 10000)
    private String seatMapLayout; // JSON string
    
    private Double latitude;
    private Double longitude;
}

// Ticket Entity
@Entity
@Table(name = "tickets")
public class Ticket {
    @Id
    private String ticketId;
    
    @Column(nullable = false)
    private String eventId;
    
    @Column(nullable = false)
    private String sectionId;
    
    private String row;
    private String seatNumber;
    
    @Enumerated(EnumType.STRING)
    private TicketType ticketType;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;
    
    @Enumerated(EnumType.STRING)
    private TicketStatus status;
    
    private String bookingId;
    private LocalDateTime reservedAt;
    private LocalDateTime reservationExpiresAt;
    
    @Version
    private Long version; // For optimistic locking
}

// Booking Entity
@Entity
@Table(name = "bookings")
public class Booking {
    @Id
    private String bookingId;
    
    @Column(nullable = false)
    private String userId;
    
    @Column(nullable = false)
    private String eventId;
    
    @Enumerated(EnumType.STRING)
    private BookingStatus status;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal totalAmount;
    
    @Column(nullable = false)
    private String currency;
    
    private String paymentId;
    private String paymentStatus;
    
    private LocalDateTime reservedAt;
    private LocalDateTime confirmedAt;
    private LocalDateTime expiresAt;
    
    @OneToMany(mappedBy = "bookingId", cascade = CascadeType.ALL)
    private List<Ticket> tickets;
}

// User Entity
@Entity
@Table(name = "users")
public class User {
    @Id
    private String userId;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(nullable = false)
    private String firstName;
    
    @Column(nullable = false)
    private String lastName;
    
    private String phoneNumber;
    private LocalDate dateOfBirth;
    
    @Column(nullable = false)
    private String passwordHash;
    
    private LocalDateTime createdAt;
    private LocalDateTime lastLoginAt;
    
    @Enumerated(EnumType.STRING)
    private UserStatus status;
}
```

### Enums and Constants:

```java
public enum EventCategory {
    CONCERT, SPORTS, THEATER, COMEDY, FESTIVAL, CONFERENCE, OTHER
}

public enum EventStatus {
    DRAFT, PUBLISHED, CANCELLED, COMPLETED
}

public enum TicketType {
    GENERAL, PREMIUM, VIP, STUDENT, SENIOR
}

public enum TicketStatus {
    AVAILABLE, RESERVED, SOLD, CANCELLED
}

public enum BookingStatus {
    RESERVED, CONFIRMED, CANCELLED, EXPIRED
}

public enum UserStatus {
    ACTIVE, INACTIVE, SUSPENDED, DELETED
}
```

### Database Indexes:

```sql
-- Events table indexes
CREATE INDEX idx_events_category_datetime ON events(category, start_date_time);
CREATE INDEX idx_events_venue_datetime ON events(venue_id, start_date_time);
CREATE INDEX idx_events_status ON events(status);

-- Tickets table indexes  
CREATE INDEX idx_tickets_event_status ON tickets(event_id, status);
CREATE INDEX idx_tickets_booking ON tickets(booking_id);
CREATE INDEX idx_tickets_reservation_expires ON tickets(reservation_expires_at) 
    WHERE status = 'RESERVED';

-- Bookings table indexes
CREATE INDEX idx_bookings_user_status ON bookings(user_id, status);
CREATE INDEX idx_bookings_event ON bookings(event_id);
CREATE INDEX idx_bookings_expires ON bookings(expires_at) WHERE status = 'RESERVED';

-- Users table indexes
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status ON users(status);
```

---

## PHASE 7: CORE FLOWS - VIEW/SEARCH/BOOK (7 minutes)

### 1. View Event Flow:

```java
@RestController
@RequestMapping("/api/v1/events")
public class EventController {
    
    @Autowired
    private EventService eventService;
    
    @GetMapping("/{eventId}")
    public ResponseEntity<EventDetailsResponse> getEventDetails(
            @PathVariable String eventId) {
        
        EventDetailsResponse response = eventService.getEventDetails(eventId);
        return ResponseEntity.ok(response);
    }
    
    @GetMapping("/{eventId}/seats")
    public ResponseEntity<SeatMapResponse> getSeatMap(
            @PathVariable String eventId) {
        
        SeatMapResponse seatMap = eventService.getSeatMapWithAvailability(eventId);
        return ResponseEntity.ok(seatMap);
    }
}

@Service
public class EventService {
    
    @Autowired
    private EventRepository eventRepository;
    
    @Autowired
    private VenueRepository venueRepository;
    
    @Autowired
    private TicketRepository ticketRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public EventDetailsResponse getEventDetails(String eventId) {
        // Try cache first
        String cacheKey = "event:details:" + eventId;
        EventDetailsResponse cached = (EventDetailsResponse) 
            redisTemplate.opsForValue().get(cacheKey);
        
        if (cached != null) {
            return cached;
        }
        
        // Fetch from database
        Event event = eventRepository.findById(eventId)
            .orElseThrow(() -> new EventNotFoundException(eventId));
        
        Venue venue = venueRepository.findById(event.getVenueId())
            .orElseThrow(() -> new VenueNotFoundException(event.getVenueId()));
        
        // Get available ticket count
        long availableTickets = ticketRepository
            .countByEventIdAndStatus(eventId, TicketStatus.AVAILABLE);
        
        EventDetailsResponse response = EventDetailsResponse.builder()
            .event(event)
            .venue(venue)
            .availableTickets(availableTickets)
            .build();
        
        // Cache for 5 minutes
        redisTemplate.opsForValue().set(cacheKey, response, 
            Duration.ofMinutes(5));
        
        return response;
    }
}
```

### 2. Search Events Flow:

```java
@RestController
@RequestMapping("/api/v1/events")
public class SearchController {
    
    @Autowired
    private SearchService searchService;
    
    @GetMapping("/search")
    public ResponseEntity<SearchResponse> searchEvents(
            @RequestParam(required = false) String keyword,
            @RequestParam(required = false) String city,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate startDate,
            @RequestParam(required = false) @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate endDate,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        SearchRequest request = SearchRequest.builder()
            .keyword(keyword)
            .city(city)
            .category(category)
            .startDate(startDate)
            .endDate(endDate)
            .page(page)
            .size(size)
            .build();
        
        SearchResponse response = searchService.searchEvents(request);
        return ResponseEntity.ok(response);
    }
}

@Service
public class SearchService {
    
    @Autowired
    private ElasticsearchRestTemplate elasticsearchTemplate;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public SearchResponse searchEvents(SearchRequest request) {
        // Generate cache key from search parameters
        String cacheKey = generateCacheKey(request);
        
        // Try cache first
        SearchResponse cached = (SearchResponse) 
            redisTemplate.opsForValue().get(cacheKey);
        
        if (cached != null) {
            return cached;
        }
        
        // Build Elasticsearch query
        BoolQueryBuilder queryBuilder = QueryBuilders.boolQuery();
        
        if (StringUtils.hasText(request.getKeyword())) {
            queryBuilder.must(QueryBuilders.multiMatchQuery(request.getKeyword())
                .field("title", 2.0f)
                .field("description", 1.0f)
                .field("performers.name", 1.5f));
        }
        
        if (StringUtils.hasText(request.getCity())) {
            queryBuilder.filter(QueryBuilders.termQuery("venue.city.keyword", 
                request.getCity()));
        }
        
        if (StringUtils.hasText(request.getCategory())) {
            queryBuilder.filter(QueryBuilders.termQuery("category", 
                request.getCategory()));
        }
        
        if (request.getStartDate() != null || request.getEndDate() != null) {
            RangeQueryBuilder dateRange = QueryBuilders.rangeQuery("startDateTime");
            if (request.getStartDate() != null) {
                dateRange.gte(request.getStartDate().atStartOfDay());
            }
            if (request.getEndDate() != null) {
                dateRange.lte(request.getEndDate().atTime(23, 59, 59));
            }
            queryBuilder.filter(dateRange);
        }
        
        // Execute search
        SearchQuery searchQuery = new NativeSearchQueryBuilder()
            .withQuery(queryBuilder)
            .withPageable(PageRequest.of(request.getPage(), request.getSize()))
            .withSort(SortBuilders.fieldSort("startDateTime").order(SortOrder.ASC))
            .build();
        
        SearchHits<EventDocument> searchHits = 
            elasticsearchTemplate.search(searchQuery, EventDocument.class);
        
        List<EventSummary> events = searchHits.getSearchHits().stream()
            .map(hit -> convertToEventSummary(hit.getContent()))
            .collect(Collectors.toList());
        
        SearchResponse response = SearchResponse.builder()
            .events(events)
            .totalElements(searchHits.getTotalHits())
            .totalPages((int) Math.ceil((double) searchHits.getTotalHits() / request.getSize()))
            .currentPage(request.getPage())
            .build();
        
        // Cache for 2 minutes
        redisTemplate.opsForValue().set(cacheKey, response, 
            Duration.ofMinutes(2));
        
        return response;
    }
}
```

### 3. Ticket Booking Flow:

```java
@RestController
@RequestMapping("/api/v1/bookings")
public class BookingController {
    
    @Autowired
    private BookingService bookingService;
    
    @PostMapping("/reserve")
    public ResponseEntity<BookingResponse> reserveTickets(
            @RequestBody @Valid ReserveTicketsRequest request) {
        
        BookingResponse response = bookingService.reserveTickets(request);
        return ResponseEntity.ok(response);
    }
    
    @PostMapping("/{bookingId}/confirm")
    public ResponseEntity<BookingResponse> confirmBooking(
            @PathVariable String bookingId,
            @RequestBody @Valid ConfirmBookingRequest request) {
        
        BookingResponse response = bookingService.confirmBooking(bookingId, request);
        return ResponseEntity.ok(response);
    }
}

@Service
@Transactional
public class BookingService {
    
    @Autowired
    private TicketRepository ticketRepository;
    
    @Autowired
    private BookingRepository bookingRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private PaymentService paymentService;
    
    public BookingResponse reserveTickets(ReserveTicketsRequest request) {
        // Validate event exists and is active
        validateEventForBooking(request.getEventId());
        
        // Generate booking ID
        String bookingId = generateBookingId();
        
        // Reserve tickets with distributed lock
        List<Ticket> reservedTickets = reserveTicketsWithLock(
            request.getEventId(), 
            request.getTicketRequests(), 
            bookingId
        );
        
        // Create booking record
        Booking booking = createBookingRecord(request, bookingId, reservedTickets);
        
        // Set expiration timer
        scheduleBookingExpiration(bookingId);
        
        return convertToBookingResponse(booking);
    }
    
    private List<Ticket> reserveTicketsWithLock(String eventId, 
            List<TicketRequest> ticketRequests, String bookingId) {
        
        List<Ticket> reservedTickets = new ArrayList<>();
        
        for (TicketRequest ticketRequest : ticketRequests) {
            String lockKey = "ticket:lock:" + eventId + ":" + 
                ticketRequest.getSectionId() + ":" + 
                ticketRequest.getRow() + ":" + 
                ticketRequest.getSeatNumber();
            
            // Try to acquire distributed lock
            Boolean lockAcquired = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, bookingId, Duration.ofMinutes(10));
            
            if (!lockAcquired) {
                // Rollback previously reserved tickets
                rollbackReservation(reservedTickets);
                throw new TicketNotAvailableException("Seat already reserved");
            }
            
            // Find and reserve the ticket
            Ticket ticket = ticketRepository.findByEventIdAndSectionIdAndRowAndSeatNumber(
                eventId, 
                ticketRequest.getSectionId(),
                ticketRequest.getRow(), 
                ticketRequest.getSeatNumber()
            ).orElseThrow(() -> new TicketNotFoundException("Ticket not found"));
            
            if (ticket.getStatus() != TicketStatus.AVAILABLE) {
                rollbackReservation(reservedTickets);
                throw new TicketNotAvailableException("Ticket not available");
            }
            
            // Update ticket status
            ticket.setStatus(TicketStatus.RESERVED);
            ticket.setBookingId(bookingId);
            ticket.setReservedAt(LocalDateTime.now());
            ticket.setReservationExpiresAt(LocalDateTime.now().plusMinutes(10));
            
            ticketRepository.save(ticket);
            reservedTickets.add(ticket);
        }
        
        return reservedTickets;
    }
    
    public BookingResponse confirmBooking(String bookingId, 
            ConfirmBookingRequest request) {
        
        // Find booking
        Booking booking = bookingRepository.findById(bookingId)
            .orElseThrow(() -> new BookingNotFoundException(bookingId));
        
        // Validate booking is still reserved and not expired
        if (booking.getStatus() != BookingStatus.RESERVED) {
            throw new InvalidBookingStateException("Booking is not in reserved state");
        }
        
        if (booking.getExpiresAt().isBefore(LocalDateTime.now())) {
            throw new BookingExpiredException("Booking has expired");
        }
        
        // Process payment
        PaymentResult paymentResult = paymentService.processPayment(
            booking.getTotalAmount(), 
            request.getPaymentDetails()
        );
        
        if (!paymentResult.isSuccessful()) {
            throw new PaymentFailedException("Payment processing failed");
        }
        
        // Update booking and tickets
        booking.setStatus(BookingStatus.CONFIRMED);
        booking.setConfirmedAt(LocalDateTime.now());
        booking.setPaymentId(paymentResult.getPaymentId());
        booking.setPaymentStatus("COMPLETED");
        
        // Update ticket status
        List<Ticket> tickets = ticketRepository.findByBookingId(bookingId);
        tickets.forEach(ticket -> {
            ticket.setStatus(TicketStatus.SOLD);
            // Release distributed lock
            String lockKey = generateTicketLockKey(ticket);
            redisTemplate.delete(lockKey);
        });
        
        ticketRepository.saveAll(tickets);
        bookingRepository.save(booking);
        
        // Send confirmation notification
        sendBookingConfirmation(booking);
        
        return convertToBookingResponse(booking);
    }
}
```

This completes the first part of the comprehensive HLD. The document covers the interview structure, requirements gathering, estimation, architecture, API design, data models, and core flows with detailed Java implementations.

---

## PHASE 8: DEEP DIVE - TICKET RESERVATION & CONCURRENCY (5 minutes)

### The Double Booking Problem:

**Challenge:** Multiple users trying to book the same seat simultaneously can lead to double booking, which creates a terrible user experience and potential legal issues.

**Solutions Comparison:**

### Approach 1: Database Row Locking (❌ Poor Solution)

```java
// BAD: Long-running database locks
@Transactional
public void reserveTicketWithRowLock(String ticketId) {
    // This keeps a database transaction open for 10 minutes!
    Ticket ticket = ticketRepository.findByIdForUpdate(ticketId);
    
    if (ticket.getStatus() != TicketStatus.AVAILABLE) {
        throw new TicketNotAvailableException();
    }
    
    ticket.setStatus(TicketStatus.RESERVED);
    // Transaction stays open until user completes payment...
    // This is terrible for database performance!
}
```

**Problems:**
- Database connections held for minutes
- Deadlock potential increases
- Poor scalability under high load
- Connection pool exhaustion

### Approach 2: Optimistic Locking with Status + Expiration (✅ Good Solution)

```java
@Entity
@Table(name = "tickets")
public class Ticket {
    @Id
    private String ticketId;
    
    @Enumerated(EnumType.STRING)
    private TicketStatus status; // AVAILABLE, RESERVED, SOLD
    
    private LocalDateTime reservationExpiresAt;
    
    @Version
    private Long version; // For optimistic locking
    
    // Check if ticket is actually available (considering expiration)
    public boolean isAvailable() {
        return status == TicketStatus.AVAILABLE || 
               (status == TicketStatus.RESERVED && 
                reservationExpiresAt.isBefore(LocalDateTime.now()));
    }
}

@Service
public class BookingService {
    
    @Transactional
    public void reserveTicket(String ticketId, String bookingId) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException());
        
        // Check availability including expired reservations
        if (!ticket.isAvailable()) {
            throw new TicketNotAvailableException();
        }
        
        // Reserve the ticket
        ticket.setStatus(TicketStatus.RESERVED);
        ticket.setBookingId(bookingId);
        ticket.setReservationExpiresAt(LocalDateTime.now().plusMinutes(10));
        
        try {
            ticketRepository.save(ticket); // Optimistic locking prevents conflicts
        } catch (OptimisticLockingFailureException e) {
            throw new TicketNotAvailableException("Ticket was just booked by another user");
        }
    }
    
    // Background job to clean up expired reservations
    @Scheduled(fixedRate = 30000) // Every 30 seconds
    public void cleanupExpiredReservations() {
        List<Ticket> expiredTickets = ticketRepository
            .findByStatusAndReservationExpiresAtBefore(
                TicketStatus.RESERVED, 
                LocalDateTime.now()
            );
        
        expiredTickets.forEach(ticket -> {
            ticket.setStatus(TicketStatus.AVAILABLE);
            ticket.setBookingId(null);
            ticket.setReservationExpiresAt(null);
        });
        
        ticketRepository.saveAll(expiredTickets);
    }
}
```

### Approach 3: Distributed Locking with Redis (✅ Great Solution)

```java
@Service
public class DistributedLockBookingService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Autowired
    private TicketRepository ticketRepository;
    
    public BookingResponse reserveTickets(ReserveTicketsRequest request) {
        String bookingId = generateBookingId();
        List<String> acquiredLocks = new ArrayList<>();
        List<Ticket> reservedTickets = new ArrayList<>();
        
        try {
            // Acquire locks for all requested tickets
            for (TicketRequest ticketRequest : request.getTicketRequests()) {
                String lockKey = generateTicketLockKey(request.getEventId(), ticketRequest);
                
                // Try to acquire lock with 10-minute TTL
                Boolean lockAcquired = redisTemplate.opsForValue()
                    .setIfAbsent(lockKey, bookingId, Duration.ofMinutes(10));
                
                if (!lockAcquired) {
                    // Release all previously acquired locks
                    releaseLocks(acquiredLocks);
                    throw new TicketNotAvailableException("One or more seats are already reserved");
                }
                
                acquiredLocks.add(lockKey);
            }
            
            // All locks acquired successfully, now reserve tickets in database
            for (TicketRequest ticketRequest : request.getTicketRequests()) {
                Ticket ticket = findAndReserveTicket(ticketRequest, bookingId);
                reservedTickets.add(ticket);
            }
            
            // Create booking record
            Booking booking = createBooking(request, bookingId, reservedTickets);
            
            return convertToBookingResponse(booking);
            
        } catch (Exception e) {
            // Rollback: release locks and unreserve tickets
            releaseLocks(acquiredLocks);
            rollbackTicketReservations(reservedTickets);
            throw e;
        }
    }
    
    private String generateTicketLockKey(String eventId, TicketRequest request) {
        return String.format("ticket:lock:%s:%s:%s:%s", 
            eventId, request.getSectionId(), request.getRow(), request.getSeatNumber());
    }
    
    @Transactional
    private Ticket findAndReserveTicket(TicketRequest request, String bookingId) {
        Ticket ticket = ticketRepository.findByEventIdAndSectionIdAndRowAndSeatNumber(
            request.getEventId(), request.getSectionId(), 
            request.getRow(), request.getSeatNumber()
        ).orElseThrow(() -> new TicketNotFoundException());
        
        if (ticket.getStatus() != TicketStatus.AVAILABLE) {
            throw new TicketNotAvailableException();
        }
        
        ticket.setStatus(TicketStatus.RESERVED);
        ticket.setBookingId(bookingId);
        ticket.setReservedAt(LocalDateTime.now());
        
        return ticketRepository.save(ticket);
    }
    
    public BookingResponse confirmBooking(String bookingId, ConfirmBookingRequest request) {
        Booking booking = bookingRepository.findById(bookingId)
            .orElseThrow(() -> new BookingNotFoundException());
        
        // Process payment
        PaymentResult paymentResult = paymentService.processPayment(
            booking.getTotalAmount(), request.getPaymentDetails());
        
        if (paymentResult.isSuccessful()) {
            // Update booking and tickets
            booking.setStatus(BookingStatus.CONFIRMED);
            booking.setPaymentId(paymentResult.getPaymentId());
            
            List<Ticket> tickets = ticketRepository.findByBookingId(bookingId);
            tickets.forEach(ticket -> ticket.setStatus(TicketStatus.SOLD));
            
            ticketRepository.saveAll(tickets);
            bookingRepository.save(booking);
            
            // Release all distributed locks
            tickets.forEach(ticket -> {
                String lockKey = generateTicketLockKey(ticket);
                redisTemplate.delete(lockKey);
            });
            
            return convertToBookingResponse(booking);
        } else {
            throw new PaymentFailedException("Payment processing failed");
        }
    }
}
```

### Redis Lock Implementation Details:

```java
@Component
public class DistributedLockManager {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    public boolean acquireLock(String lockKey, String lockValue, Duration ttl) {
        return redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, ttl);
    }
    
    public void releaseLock(String lockKey, String lockValue) {
        // Use Lua script to ensure atomic check-and-delete
        String luaScript = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
        
        redisTemplate.execute(
            RedisScript.of(luaScript, Long.class),
            Collections.singletonList(lockKey),
            lockValue
        );
    }
    
    public boolean extendLock(String lockKey, String lockValue, Duration ttl) {
        String luaScript = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('expire', KEYS[1], ARGV[2]) " +
            "else " +
            "    return 0 " +
            "end";
        
        Long result = redisTemplate.execute(
            RedisScript.of(luaScript, Long.class),
            Collections.singletonList(lockKey),
            lockValue, String.valueOf(ttl.getSeconds())
        );
        
        return result != null && result == 1;
    }
}
```

### Why Distributed Locking is Superior:

✅ **Immediate availability**: Locks are released instantly when TTL expires
✅ **High performance**: Redis operations are sub-millisecond
✅ **Fault tolerance**: If service crashes, locks auto-expire
✅ **Scalability**: Redis can handle millions of lock operations
✅ **Atomic operations**: Lua scripts ensure consistency

---

## PHASE 9: DEEP DIVE - SCALING & PERFORMANCE (5 minutes)

### 1. Handling 10M Concurrent Users During Popular Events

**Challenge:** Taylor Swift concert tickets go on sale - 10 million users hit the system simultaneously.

### Virtual Waiting Room Implementation:

```java
@RestController
@RequestMapping("/api/v1/queue")
public class VirtualWaitingRoomController {
    
    @Autowired
    private WaitingRoomService waitingRoomService;
    
    @PostMapping("/join/{eventId}")
    public ResponseEntity<QueueResponse> joinQueue(
            @PathVariable String eventId,
            @RequestParam String userId) {
        
        QueueResponse response = waitingRoomService.joinQueue(eventId, userId);
        return ResponseEntity.ok(response);
    }
    
    @GetMapping("/status/{eventId}/{userId}")
    public ResponseEntity<QueueStatusResponse> getQueueStatus(
            @PathVariable String eventId,
            @PathVariable String userId) {
        
        QueueStatusResponse status = waitingRoomService.getQueueStatus(eventId, userId);
        return ResponseEntity.ok(status);
    }
}

@Service
public class WaitingRoomService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    private static final int BOOKING_CAPACITY_PER_MINUTE = 1000; // Allow 1000 users per minute
    
    public QueueResponse joinQueue(String eventId, String userId) {
        String queueKey = "queue:" + eventId;
        String userKey = "user:queue:" + eventId + ":" + userId;
        
        // Check if user is already in queue
        if (redisTemplate.hasKey(userKey)) {
            return getExistingQueuePosition(eventId, userId);
        }
        
        // Add user to sorted set with timestamp as score
        long timestamp = System.currentTimeMillis();
        redisTemplate.opsForZSet().add(queueKey, userId, timestamp);
        
        // Store user's queue entry time
        redisTemplate.opsForValue().set(userKey, String.valueOf(timestamp), 
            Duration.ofHours(2));
        
        // Calculate position and estimated wait time
        Long position = redisTemplate.opsForZSet().rank(queueKey, userId);
        long estimatedWaitMinutes = (position != null ? position : 0) / BOOKING_CAPACITY_PER_MINUTE;
        
        return QueueResponse.builder()
            .queuePosition(position != null ? position + 1 : 1)
            .estimatedWaitTime(estimatedWaitMinutes)
            .status("WAITING")
            .build();
    }
    
    @Scheduled(fixedRate = 10000) // Every 10 seconds
    public void processQueue() {
        // Process queues for all active events
        Set<String> activeEvents = getActiveEventIds();
        
        for (String eventId : activeEvents) {
            processEventQueue(eventId);
        }
    }
    
    private void processEventQueue(String eventId) {
        String queueKey = "queue:" + eventId;
        String activeKey = "active:" + eventId;
        
        // Get current active users count
        Long activeCount = redisTemplate.opsForSet().size(activeKey);
        if (activeCount == null) activeCount = 0L;
        
        // Calculate how many users we can admit
        int maxActiveUsers = 5000; // Allow 5000 concurrent booking sessions
        long usersToAdmit = Math.max(0, maxActiveUsers - activeCount);
        
        if (usersToAdmit > 0) {
            // Get next users from queue
            Set<String> nextUsers = redisTemplate.opsForZSet()
                .range(queueKey, 0, usersToAdmit - 1);
            
            for (String userId : nextUsers) {
                // Move user from queue to active
                redisTemplate.opsForZSet().remove(queueKey, userId);
                redisTemplate.opsForSet().add(activeKey, userId);
                
                // Set expiration for active session (30 minutes)
                redisTemplate.expire(activeKey, Duration.ofMinutes(30));
                
                // Notify user they can proceed
                notifyUserCanProceed(eventId, userId);
            }
        }
    }
}
```

### 2. Caching Strategy for Scale:

```java
@Service
public class CachedEventService {
    
    @Autowired
    private EventRepository eventRepository;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // Multi-level caching strategy
    @Cacheable(value = "events", key = "#eventId", unless = "#result == null")
    public EventDetailsResponse getEventDetails(String eventId) {
        // L1: Application cache (Caffeine)
        // L2: Redis cache (handled by @Cacheable)
        // L3: Database
        
        Event event = eventRepository.findById(eventId)
            .orElseThrow(() -> new EventNotFoundException(eventId));
        
        return convertToEventDetailsResponse(event);
    }
    
    // Cache seat map with real-time availability
    public SeatMapResponse getSeatMapWithAvailability(String eventId) {
        String cacheKey = "seatmap:" + eventId;
        
        // Try to get from cache first
        SeatMapResponse cached = (SeatMapResponse) 
            redisTemplate.opsForValue().get(cacheKey);
        
        if (cached != null && !isSeatMapStale(cached)) {
            return cached;
        }
        
        // Rebuild seat map with current availability
        SeatMapResponse seatMap = buildSeatMapWithAvailability(eventId);
        
        // Cache for 30 seconds (short TTL for high-demand events)
        redisTemplate.opsForValue().set(cacheKey, seatMap, Duration.ofSeconds(30));
        
        return seatMap;
    }
    
    // Cache warming for popular events
    @EventListener
    public void handleEventPopularityIncrease(EventPopularityEvent event) {
        if (event.getConcurrentUsers() > 10000) {
            // Pre-warm caches for high-traffic event
            warmEventCaches(event.getEventId());
        }
    }
    
    private void warmEventCaches(String eventId) {
        // Warm event details cache
        getEventDetails(eventId);
        
        // Warm seat map cache
        getSeatMapWithAvailability(eventId);
        
        // Pre-compute search results for this event
        preComputeSearchResults(eventId);
    }
}
```

### 3. Database Scaling Strategy:

```java
// Read/Write Splitting Configuration
@Configuration
public class DatabaseConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        RoutingDataSource routingDataSource = new RoutingDataSource();
        
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put("write", writeDataSource());
        dataSourceMap.put("read", readDataSource());
        
        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(writeDataSource());
        
        return routingDataSource;
    }
    
    @Bean
    public DataSource writeDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://write-db:5432/ticketmaster");
        config.setMaximumPoolSize(50);
        config.setConnectionTimeout(30000);
        return new HikariDataSource(config);
    }
    
    @Bean
    public DataSource readDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://read-replica:5432/ticketmaster");
        config.setMaximumPoolSize(100);
        config.setReadOnly(true);
        return new HikariDataSource(config);
    }
}

// Custom annotation for read/write routing
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadOnly {
}

@Aspect
@Component
public class DataSourceRoutingAspect {
    
    @Around("@annotation(readOnly)")
    public Object routeToReadDataSource(ProceedingJoinPoint joinPoint, ReadOnly readOnly) throws Throwable {
        try {
            DatabaseContextHolder.setDatabaseType("read");
            return joinPoint.proceed();
        } finally {
            DatabaseContextHolder.clearDatabaseType();
        }
    }
}

// Usage in service layer
@Service
public class EventService {
    
    @ReadOnly
    public List<Event> searchEvents(SearchCriteria criteria) {
        // This will use read replica
        return eventRepository.findByCriteria(criteria);
    }
    
    public Event createEvent(Event event) {
        // This will use write database
        return eventRepository.save(event);
    }
}
```

### 4. Real-time Updates with Server-Sent Events:

```java
@RestController
@RequestMapping("/api/v1/events")
public class RealTimeEventController {
    
    @Autowired
    private SseEmitterService sseEmitterService;
    
    @GetMapping("/{eventId}/seat-updates")
    public SseEmitter subscribeToSeatUpdates(@PathVariable String eventId) {
        return sseEmitterService.createEmitter(eventId);
    }
}

@Service
public class SseEmitterService {
    
    private final Map<String, Set<SseEmitter>> eventEmitters = new ConcurrentHashMap<>();
    
    public SseEmitter createEmitter(String eventId) {
        SseEmitter emitter = new SseEmitter(300000L); // 5 minutes timeout
        
        eventEmitters.computeIfAbsent(eventId, k -> ConcurrentHashMap.newKeySet())
            .add(emitter);
        
        emitter.onCompletion(() -> removeEmitter(eventId, emitter));
        emitter.onTimeout(() -> removeEmitter(eventId, emitter));
        emitter.onError(e -> removeEmitter(eventId, emitter));
        
        return emitter;
    }
    
    @EventListener
    public void handleTicketStatusChange(TicketStatusChangeEvent event) {
        String eventId = event.getEventId();
        Set<SseEmitter> emitters = eventEmitters.get(eventId);
        
        if (emitters != null && !emitters.isEmpty()) {
            SeatUpdateMessage message = SeatUpdateMessage.builder()
                .sectionId(event.getSectionId())
                .row(event.getRow())
                .seatNumber(event.getSeatNumber())
                .status(event.getNewStatus())
                .timestamp(System.currentTimeMillis())
                .build();
            
            List<SseEmitter> deadEmitters = new ArrayList<>();
            
            for (SseEmitter emitter : emitters) {
                try {
                    emitter.send(SseEmitter.event()
                        .name("seat-update")
                        .data(message));
                } catch (Exception e) {
                    deadEmitters.add(emitter);
                }
            }
            
            // Remove dead emitters
            deadEmitters.forEach(emitter -> removeEmitter(eventId, emitter));
        }
    }
}
```

### 5. Search Performance with Elasticsearch:

```java
@Service
public class ElasticsearchEventService {
    
    @Autowired
    private ElasticsearchRestTemplate elasticsearchTemplate;
    
    public SearchResponse searchEventsWithAggregations(SearchRequest request) {
        // Build complex query with filters and aggregations
        BoolQueryBuilder queryBuilder = QueryBuilders.boolQuery();
        
        // Full-text search with boosting
        if (StringUtils.hasText(request.getKeyword())) {
            queryBuilder.must(
                QueryBuilders.multiMatchQuery(request.getKeyword())
                    .field("title", 3.0f)           // Boost title matches
                    .field("description", 1.0f)
                    .field("performers.name", 2.0f) // Boost performer matches
                    .field("venue.name", 1.5f)
                    .type(MultiMatchQueryBuilder.Type.BEST_FIELDS)
                    .fuzziness(Fuzziness.AUTO)      // Handle typos
            );
        }
        
        // Geo-distance filter for location-based search
        if (request.getLatitude() != null && request.getLongitude() != null) {
            queryBuilder.filter(
                QueryBuilders.geoDistanceQuery("venue.location")
                    .point(request.getLatitude(), request.getLongitude())
                    .distance(request.getRadius(), DistanceUnit.KILOMETERS)
            );
        }
        
        // Date range filter
        if (request.getStartDate() != null || request.getEndDate() != null) {
            RangeQueryBuilder dateRange = QueryBuilders.rangeQuery("startDateTime");
            if (request.getStartDate() != null) {
                dateRange.gte(request.getStartDate());
            }
            if (request.getEndDate() != null) {
                dateRange.lte(request.getEndDate());
            }
            queryBuilder.filter(dateRange);
        }
        
        // Price range filter
        if (request.getMinPrice() != null || request.getMaxPrice() != null) {
            RangeQueryBuilder priceRange = QueryBuilders.rangeQuery("priceRange.min");
            if (request.getMinPrice() != null) {
                priceRange.gte(request.getMinPrice());
            }
            if (request.getMaxPrice() != null) {
                priceRange.lte(request.getMaxPrice());
            }
            queryBuilder.filter(priceRange);
        }
        
        // Build aggregations for faceted search
        AggregationBuilder categoryAgg = AggregationBuilders
            .terms("categories")
            .field("category.keyword")
            .size(20);
        
        AggregationBuilder cityAgg = AggregationBuilders
            .terms("cities")
            .field("venue.city.keyword")
            .size(50);
        
        AggregationBuilder priceRangeAgg = AggregationBuilders
            .range("priceRanges")
            .field("priceRange.min")
            .addRange("budget", 0, 50)
            .addRange("moderate", 50, 150)
            .addRange("premium", 150, 500)
            .addRange("luxury", 500, Double.MAX_VALUE);
        
        // Execute search with aggregations
        SearchQuery searchQuery = new NativeSearchQueryBuilder()
            .withQuery(queryBuilder)
            .withPageable(PageRequest.of(request.getPage(), request.getSize()))
            .withSort(buildSortQuery(request.getSortBy(), request.getSortOrder()))
            .addAggregation(categoryAgg)
            .addAggregation(cityAgg)
            .addAggregation(priceRangeAgg)
            .withHighlightFields(
                new HighlightBuilder.Field("title"),
                new HighlightBuilder.Field("description")
            )
            .build();
        
        SearchHits<EventDocument> searchHits = 
            elasticsearchTemplate.search(searchQuery, EventDocument.class);
        
        // Process results and aggregations
        return buildSearchResponse(searchHits, request);
    }
}
```

---

## FINAL ARCHITECTURE DIAGRAM

```
                                    ┌─────────────────────────────────────┐
                                    │            CLIENT LAYER             │
                                    ├─────────────────────────────────────┤
                                    │  Web App  │  Mobile App  │  Admin   │
                                    └─────────────────────────────────────┘
                                                      │
                                    ┌─────────────────────────────────────┐
                                    │         CDN + LOAD BALANCER         │
                                    ├─────────────────────────────────────┤
                                    │ CloudFront │ ALB │ Rate Limiter     │
                                    └─────────────────────────────────────┘
                                                      │
                                    ┌─────────────────────────────────────┐
                                    │           API GATEWAY               │
                                    ├─────────────────────────────────────┤
                                    │ Auth │ Routing │ Circuit Breaker    │
                                    └─────────────────────────────────────┘
                                                      │
                        ┌─────────────────────────────┼─────────────────────────────┐
                        │                             │                             │
                        ▼                             ▼                             ▼
        ┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
        │     EVENT SERVICE       │   │    SEARCH SERVICE       │   │   BOOKING SERVICE       │
        ├─────────────────────────┤   ├─────────────────────────┤   ├─────────────────────────┤
        │ • View Events           │   │ • Keyword Search        │   │ • Reserve Tickets       │
        │ • Event Details         │   │ • Filters & Facets      │   │ • Confirm Booking       │
        │ • Seat Maps             │   │ • Auto-complete         │   │ • Cancel Booking        │
        │ • Real-time Updates     │   │ • Geo Search            │   │ • Distributed Locking   │
        └─────────────────────────┘   └─────────────────────────┘   └─────────────────────────┘
                        │                             │                             │
                        ▼                             ▼                             ▼
        ┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
        │    USER SERVICE         │   │   PAYMENT SERVICE       │   │  NOTIFICATION SERVICE   │
        ├─────────────────────────┤   ├─────────────────────────┤   ├─────────────────────────┤
        │ • Authentication        │   │ • Payment Processing    │   │ • Email Notifications   │
        │ • User Profiles         │   │ • Stripe Integration    │   │ • SMS Alerts            │
        │ • Booking History       │   │ • Refund Processing     │   │ • Push Notifications    │
        │ • Session Management    │   │ • Webhook Handling      │   │ • Real-time Updates     │
        └─────────────────────────┘   └─────────────────────────┘   └─────────────────────────┘
                                                      │
                        ┌─────────────────────────────┼─────────────────────────────┐
                        │                             │                             │
                        ▼                             ▼                             ▼
        ┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
        │   POSTGRESQL CLUSTER    │   │  ELASTICSEARCH CLUSTER  │   │    REDIS CLUSTER        │
        ├─────────────────────────┤   ├─────────────────────────┤   ├─────────────────────────┤
        │ • Events & Venues       │   │ • Search Index          │   │ • Application Cache     │
        │ • Users & Bookings      │   │ • Analytics Data        │   │ • Session Store         │
        │ • Tickets & Payments    │   │ • Aggregations          │   │ • Distributed Locks     │
        │ • Read Replicas (3x)    │   │ • Auto-complete         │   │ • Queue Management      │
        │ • Write Master (1x)     │   │ • 6 Data Nodes          │   │ • Real-time Data        │
        └─────────────────────────┘   └─────────────────────────┘   └─────────────────────────┘
                                                      │
                        ┌─────────────────────────────┼─────────────────────────────┐
                        │                             │                             │
                        ▼                             ▼                             ▼
        ┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
        │      S3 STORAGE         │   │    STRIPE PAYMENTS      │   │     KAFKA EVENT BUS     │
        ├─────────────────────────┤   ├─────────────────────────┤   ├─────────────────────────┤
        │ • Event Images          │   │ • Payment Processing    │   │ • Event Streaming       │
        │ • Venue Seat Maps       │   │ • Webhook Callbacks     │   │ • Service Communication │
        │ • User Uploads          │   │ • PCI Compliance        │   │ • Audit Logs            │
        │ • Backup Storage        │   │ • Refund Management     │   │ • Analytics Pipeline    │
        └─────────────────────────┘   └─────────────────────────┘   └─────────────────────────┘

                        ┌─────────────────────────────────────────────────────────┐
                        │              MONITORING & OBSERVABILITY                │
                        ├─────────────────────────────────────────────────────────┤
                        │ Prometheus │ Grafana │ ELK Stack │ Jaeger │ AlertManager │
                        └─────────────────────────────────────────────────────────┘
```

## KEY PERFORMANCE METRICS

```
┌─────────────────────────┬──────────────────┬──────────────────┐
│ Metric                  │ Target           │ Peak Load        │
├─────────────────────────┼──────────────────┼──────────────────┤
│ Concurrent Users        │ 100K average     │ 10M peak         │
│ Search Latency (P99)    │ < 500ms          │ < 800ms          │
│ Booking Latency (P99)   │ < 200ms          │ < 500ms          │
│ Throughput              │ 5K QPS average   │ 500K QPS peak    │
│ Availability            │ 99.9%            │ 99.9%            │
│ Database Connections    │ 500 per instance │ 1000 per instance│
│ Cache Hit Ratio         │ > 90%            │ > 85%            │
│ Double Booking Rate     │ 0%               │ 0%               │
└─────────────────────────┴──────────────────┴──────────────────┘
```

This comprehensive HLD covers all aspects of a production-ready Ticketmaster system with detailed Java implementations, scaling strategies, and performance optimizations.