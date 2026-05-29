# Uber Ride-Sharing Platform - Complete HLD Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: High-Level Architecture (8 min)
Phase 5: API Design (3 min)
Phase 6: Data Models & Core Entities (4 min)
Phase 7: Core Flows - Fare/Request/Match/Accept (7 min)
Phase 8: Deep Dive - Location Services & Geospatial Queries (5 min)
Phase 9: Deep Dive - Matching Algorithm & Concurrency (5 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"I'll design a ride-sharing platform like Uber that connects riders with drivers for on-demand transportation. Let me clarify the scope and requirements."

### Questions to Ask:

**Q1: Core Use Cases**
- "Are we focusing on ride-hailing only, or do we need food delivery, freight, etc.?"
- "Should we support ride categories (UberX, UberXL, UberBlack)?"

**Expected Answer:** Focus on core ride-hailing, single ride type initially

**Q2: User Types**
- "Do we need separate apps for riders and drivers?"
- "Should we support admin/operations dashboards?"

**Expected Answer:** Separate rider and driver apps, basic admin features

**Q3: Geographic Scope**
- "Is this a single city, multi-city, or global platform?"
- "Do we need to handle different currencies and regulations?"

**Expected Answer:** Multi-city platform, focus on core functionality first

**Q4: Real-time Requirements**
- "How real-time should location tracking be?"
- "What's acceptable latency for ride matching?"

**Expected Answer:** Sub-minute matching, 5-second location updates

**Q5: Scale & Traffic**
- "How many concurrent riders and drivers do we expect?"
- "What's the peak traffic scenario (events, rush hour)?"

**Expected Answer:** 1M concurrent users, 100K active drivers per city

**Q6: Payment & Pricing**
- "Do we handle payments in-app or integrate with processors?"
- "Should we support surge pricing?"

**Expected Answer:** Integrate with payment processors, basic pricing model

**Q7: Trip Features**
- "Do we need trip sharing, scheduling, or just on-demand rides?"
- "Should we support multi-stop trips?"

**Expected Answer:** On-demand rides only, single pickup/dropoff

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### Functional Requirements (Priority Order):

```
CORE REQUIREMENTS (Above the line):

1. Fare Estimation (CRITICAL)
   - Input pickup and destination locations
   - Calculate estimated fare and ETA
   - Display route preview
   - Store fare estimates for booking

2. Ride Request (CRITICAL)
   - Request ride based on fare estimate
   - Real-time driver matching
   - Driver assignment and notification
   - Ride state management

3. Driver Matching (CRITICAL)
   - Find nearby available drivers
   - Rank drivers by proximity, rating, acceptance rate
   - Send ride requests to optimal driver
   - Handle driver acceptance/rejection

4. Location Services (CRITICAL)
   - Real-time driver location updates
   - Efficient proximity searches
   - Route tracking during trip
   - ETA calculations

IMPORTANT REQUIREMENTS:

5. Trip Management
   - Trip start/end tracking
   - Route navigation
   - Real-time trip updates
   - Trip completion and payment

6. User Management
   - Rider and driver registration/authentication
   - Profile management
   - Trip history

BELOW THE LINE (Out of scope):
- Ride ratings and reviews
- Scheduled rides
- Multiple ride categories (UberX, UberXL)
- Ride sharing (multiple passengers)
- Driver earnings dashboard
- Surge pricing algorithms
- Multi-stop trips
```

### Non-Functional Requirements:

```
PERFORMANCE:
- Matching latency: < 30 seconds (P95)
- Location update frequency: Every 5 seconds
- Fare calculation: < 2 seconds
- System throughput: 100K ride requests/hour per city

SCALABILITY:
- Support 1M concurrent users globally
- Handle 100K active drivers per major city
- Linear scaling with geographic expansion
- Auto-scaling during peak hours

CONSISTENCY & AVAILABILITY:
- Strong consistency for ride matching (no double assignment)
- Eventual consistency for location updates
- 99.9% uptime for core services
- Graceful degradation during failures

REAL-TIME REQUIREMENTS:
- Driver location updates: 5-second intervals
- Ride status updates: Real-time push notifications
- ETA updates: Every 30 seconds during trip
- Driver-rider communication: Sub-second messaging

GEOSPATIAL PERFORMANCE:
- Proximity search: < 100ms for nearby drivers
- Support for 10M+ location updates per minute
- Efficient geospatial indexing and queries
- Handle geographic boundaries and regions
```

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### Scale Assumptions:

```
GLOBAL SCALE:
- Active cities: 100 major cities
- Concurrent users: 1M globally (10K per city average)
- Active drivers: 100K per major city (10M globally)
- Daily rides: 10M globally (100K per city)
- Peak multiplier: 3x during rush hours

TRAFFIC PATTERNS:
- Location updates: 10M drivers × 12 updates/min = 120M/min = 2M/sec
- Ride requests: 100K/hour per city = 28 requests/sec per city
- Fare estimates: 5x ride requests = 140 estimates/sec per city
- Read:Write ratio: 80:20 (location reads vs updates)
```

### Storage Estimation:

```
USER DATA:
- Riders: 100M users × 2 KB = 200 GB
- Drivers: 10M drivers × 5 KB = 50 GB
- Vehicles: 10M vehicles × 3 KB = 30 GB

TRIP DATA:
- Daily trips: 10M × 365 days = 3.65B trips/year
- Trip record: 2 KB per trip
- Annual trip data: 3.65B × 2 KB = 7.3 TB/year
- 5-year retention: 36.5 TB

LOCATION DATA:
- Location updates: 2M/sec × 100 bytes = 200 MB/sec
- Daily location data: 200 MB/sec × 86400 sec = 17.3 TB/day
- 7-day retention: 121 TB (mostly in Redis)

TOTAL STORAGE:
- Persistent data: ~37 TB (with replication: ~111 TB)
- Cache/temporary data: ~121 TB (Redis clusters)
- Total: ~232 TB across all regions
```

### Throughput Analysis:

```
LOCATION SERVICE:
- Write QPS: 2M location updates/sec globally
- Read QPS: 8M proximity searches/sec (4x multiplier)
- Total: 10M QPS for location operations

RIDE SERVICE:
- Fare estimates: 14K/sec globally (100 cities × 140/sec)
- Ride requests: 2.8K/sec globally (100 cities × 28/sec)
- Trip updates: 5.6K/sec (2x ride requests)
- Total: 22.4K QPS for ride operations

USER SERVICE:
- Authentication: 10K/sec (login/session validation)
- Profile updates: 1K/sec
- Trip history: 5K/sec
- Total: 16K QPS for user operations

DATABASE SIZING:
- Location Service: Redis cluster (10M QPS)
  - 50 Redis instances (200K QPS each)
- Ride Service: PostgreSQL cluster (22.4K QPS)
  - 5 primary + 15 read replicas
- User Service: PostgreSQL cluster (16K QPS)
  - 3 primary + 9 read replicas
```

### Network & Infrastructure:

```
BANDWIDTH REQUIREMENTS:
- Location updates: 2M/sec × 100 bytes = 200 MB/sec
- API responses: 10M QPS × 1 KB avg = 10 GB/sec
- Total bandwidth: ~10.2 GB/sec globally

COMPUTE REQUIREMENTS:
- Application servers: 200 instances (c5.2xlarge)
- Database instances: 72 instances (r5.xlarge)
- Cache instances: 50 instances (r5.2xlarge)
- Load balancers: 20 instances per region

COST ESTIMATION (AWS):
- Compute: $80K/month
- Storage: $15K/month
- Network: $25K/month
- Total: ~$120K/month (~$1.4M/year)
```

---

## PHASE 4: HIGH-LEVEL ARCHITECTURE (8 minutes)

### Complete System Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│    Rider App     │    Driver App    │   Admin Dashboard        │
│   (iOS/Android)  │   (iOS/Android)  │     (Web Portal)         │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    EDGE & LOAD BALANCING                        │
├─────────────────────────────────────────────────────────────────┤
│  CloudFront CDN  │  Application LB  │  Geographic Routing      │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                        API GATEWAY                              │
├─────────────────────────────────────────────────────────────────┤
│  Authentication  │  Rate Limiting   │  Request Routing         │
│  Circuit Breaker │  Monitoring      │  Protocol Translation   │
└─────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES LAYER                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  LOCATION   │  │    RIDE     │  │   MATCHING  │             │
│  │  SERVICE    │  │  SERVICE    │  │  SERVICE    │             │
│  │             │  │             │  │             │             │
│  │ • Track     │  │ • Fare Est  │  │ • Find      │             │
│  │ • Update    │  │ • Request   │  │ • Rank      │             │
│  │ • Query     │  │ • Manage    │  │ • Assign    │             │
│  │ • Proximity │  │ • Payment   │  │ • Notify    │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    USER     │  │  PAYMENT    │  │NOTIFICATION │             │
│  │  SERVICE    │  │  SERVICE    │  │  SERVICE    │             │
│  │             │  │             │  │             │             │
│  │ • Auth      │  │ • Process   │  │ • Push      │             │
│  │ • Profile   │  │ • Billing   │  │ • SMS       │             │
│  │ • History   │  │ • Payout    │  │ • Email     │             │
│  │ • Rating    │  │ • Refund    │  │ • WebSocket │             │
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
│  │    REDIS    │  │ POSTGRESQL  │  │   KAFKA     │             │
│  │  CLUSTER    │  │  CLUSTER    │  │ EVENT BUS   │             │
│  │             │  │             │  │             │             │
│  │ • Location  │  │ • Users     │  │ • Events    │             │
│  │ • Cache     │  │ • Trips     │  │ • Logs      │             │
│  │ • Session   │  │ • Payments  │  │ • Analytics │             │
│  │ • Locks     │  │ • Vehicles  │  │ • Audit     │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │    S3       │  │   STRIPE    │  │   MAPS API  │             │
│  │  STORAGE    │  │  PAYMENTS   │  │  (Google)   │             │
│  │             │  │             │  │             │             │
│  │ • Backups   │  │ • Process   │  │ • Routes    │             │
│  │ • Logs      │  │ • Webhooks  │  │ • Distance  │             │
│  │ • Analytics │  │ • Compliance│  │ • ETA       │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘

                        ┌─────────────────────────────────────────┐
                        │         MONITORING & OBSERVABILITY     │
                        ├─────────────────────────────────────────┤
                        │ Prometheus │ Grafana │ ELK │ Jaeger    │
                        └─────────────────────────────────────────┘
```

### Key Architectural Decisions:

**1. Microservices Architecture**
- Service isolation for independent scaling
- Technology diversity per service needs
- Fault tolerance and resilience

**2. Geospatial Data Strategy**
- Redis with geospatial commands for real-time location
- Efficient proximity searches with GEORADIUS
- Location data partitioned by geographic regions

**3. Event-Driven Architecture**
- Kafka for async communication between services
- Event sourcing for trip state changes
- Real-time updates via WebSocket connections

**4. Caching Strategy**
- Redis for location data and session management
- Application-level caching for user profiles
- CDN for static assets and API responses

**5. Database Per Service**
- PostgreSQL for transactional data (users, trips, payments)
- Redis for high-frequency location updates
- Separate read replicas for analytics queries

---

## PHASE 5: API DESIGN (3 minutes)

### RESTful API Endpoints:

```java
// Fare & Ride Management APIs
POST   /api/v1/fares/estimate                      // Get fare estimate
POST   /api/v1/rides                               // Request ride
GET    /api/v1/rides/{rideId}                      // Get ride details
PATCH  /api/v1/rides/{rideId}/status               // Update ride status
DELETE /api/v1/rides/{rideId}                      // Cancel ride

// Location APIs
POST   /api/v1/drivers/location                    // Update driver location
GET    /api/v1/drivers/nearby                      // Find nearby drivers
GET    /api/v1/rides/{rideId}/location             // Get trip location

// Driver APIs
PATCH  /api/v1/rides/{rideId}/accept               // Accept ride request
PATCH  /api/v1/rides/{rideId}/decline              // Decline ride request
PATCH  /api/v1/drivers/availability                // Update availability

// User Management APIs
POST   /api/v1/auth/login                          // User login
POST   /api/v1/auth/register                       // User registration
GET    /api/v1/users/{userId}/profile              // Get user profile
GET    /api/v1/users/{userId}/trips                // Get trip history

// Payment APIs
POST   /api/v1/payments/process                    // Process payment
GET    /api/v1/payments/{paymentId}/status         // Payment status
POST   /api/v1/payments/webhooks/stripe            // Payment webhooks
```

### API Request/Response Examples:

```java
// Fare Estimate API
POST /api/v1/fares/estimate
{
  "pickupLocation": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "address": "123 Market St, San Francisco, CA"
  },
  "destination": {
    "latitude": 37.7849,
    "longitude": -122.4094,
    "address": "456 Mission St, San Francisco, CA"
  },
  "rideType": "STANDARD"
}

Response:
{
  "fareId": "fare_123456",
  "estimatedFare": {
    "amount": 12.50,
    "currency": "USD",
    "breakdown": {
      "baseFare": 2.50,
      "distanceFare": 8.00,
      "timeFare": 2.00
    }
  },
  "estimatedDuration": 900,  // seconds
  "estimatedDistance": 3.2,  // miles
  "route": {
    "polyline": "encoded_polyline_string",
    "waypoints": [...]
  },
  "expiresAt": "2024-01-15T14:05:00Z"
}

// Request Ride API
POST /api/v1/rides
{
  "fareId": "fare_123456",
  "paymentMethodId": "pm_card_visa",
  "notes": "Please call when you arrive"
}

Response:
{
  "rideId": "ride_789012",
  "status": "REQUESTED",
  "fare": {
    "amount": 12.50,
    "currency": "USD"
  },
  "pickupLocation": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "address": "123 Market St, San Francisco, CA"
  },
  "destination": {
    "latitude": 37.7849,
    "longitude": -122.4094,
    "address": "456 Mission St, San Francisco, CA"
  },
  "estimatedPickupTime": "2024-01-15T14:08:00Z",
  "createdAt": "2024-01-15T14:00:00Z"
}

// Update Driver Location API
POST /api/v1/drivers/location
{
  "latitude": 37.7849,
  "longitude": -122.4094,
  "heading": 45.0,
  "speed": 25.5,
  "accuracy": 5.0,
  "timestamp": "2024-01-15T14:00:00Z"
}

Response:
{
  "success": true,
  "message": "Location updated successfully"
}
```

---

## PHASE 6: DATA MODELS & CORE ENTITIES (4 minutes)

### Database Schema Design:

```java
// User Entity (Riders and Drivers)
@Entity
@Table(name = "users")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "user_type")
public abstract class User {
    @Id
    private String userId;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(nullable = false)
    private String firstName;
    
    @Column(nullable = false)
    private String lastName;
    
    @Column(nullable = false)
    private String phoneNumber;
    
    @Enumerated(EnumType.STRING)
    private UserStatus status;
    
    private LocalDateTime createdAt;
    private LocalDateTime lastActiveAt;
}

@Entity
@DiscriminatorValue("RIDER")
public class Rider extends User {
    @Column(precision = 3, scale = 2)
    private BigDecimal rating;
    
    private Integer totalTrips;
    private String preferredPaymentMethod;
    
    @OneToMany(mappedBy = "rider", cascade = CascadeType.ALL)
    private List<Trip> trips;
}

@Entity
@DiscriminatorValue("DRIVER")
public class Driver extends User {
    @Column(nullable = false)
    private String licenseNumber;
    
    @Column(precision = 3, scale = 2)
    private BigDecimal rating;
    
    private Integer totalTrips;
    private Integer acceptanceRate;
    
    @Enumerated(EnumType.STRING)
    private DriverStatus driverStatus;
    
    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "vehicle_id")
    private Vehicle vehicle;
    
    @OneToMany(mappedBy = "driver", cascade = CascadeType.ALL)
    private List<Trip> trips;
}

// Vehicle Entity
@Entity
@Table(name = "vehicles")
public class Vehicle {
    @Id
    private String vehicleId;
    
    @Column(nullable = false)
    private String make;
    
    @Column(nullable = false)
    private String model;
    
    @Column(nullable = false)
    private Integer year;
    
    @Column(nullable = false)
    private String color;
    
    @Column(nullable = false, unique = true)
    private String licensePlate;
    
    @Column(nullable = false)
    private String vin;
    
    @Enumerated(EnumType.STRING)
    private VehicleType vehicleType;
    
    private Integer capacity;
}

// Trip Entity
@Entity
@Table(name = "trips")
public class Trip {
    @Id
    private String tripId;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "rider_id", nullable = false)
    private Rider rider;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "driver_id")
    private Driver driver;
    
    @Enumerated(EnumType.STRING)
    private TripStatus status;
    
    // Pickup location
    @Column(nullable = false)
    private Double pickupLatitude;
    
    @Column(nullable = false)
    private Double pickupLongitude;
    
    @Column(nullable = false)
    private String pickupAddress;
    
    // Destination
    @Column(nullable = false)
    private Double destinationLatitude;
    
    @Column(nullable = false)
    private Double destinationLongitude;
    
    @Column(nullable = false)
    private String destinationAddress;
    
    // Fare information
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal estimatedFare;
    
    @Column(precision = 10, scale = 2)
    private BigDecimal actualFare;
    
    @Column(nullable = false)
    private String currency;
    
    // Trip timing
    private LocalDateTime requestedAt;
    private LocalDateTime acceptedAt;
    private LocalDateTime pickedUpAt;
    private LocalDateTime completedAt;
    private LocalDateTime cancelledAt;
    
    // Trip metrics
    private Double estimatedDistance;
    private Double actualDistance;
    private Integer estimatedDuration;
    private Integer actualDuration;
    
    // Payment
    private String paymentMethodId;
    private String paymentTransactionId;
    
    @Enumerated(EnumType.STRING)
    private PaymentStatus paymentStatus;
    
    // Route information
    @Column(length = 10000)
    private String routePolyline;
    
    private String notes;
    
    @Version
    private Long version;
}

// Location Entity (for real-time tracking)
@Entity
@Table(name = "driver_locations")
public class DriverLocation {
    @Id
    private String locationId;
    
    @Column(nullable = false)
    private String driverId;
    
    @Column(nullable = false)
    private Double latitude;
    
    @Column(nullable = false)
    private Double longitude;
    
    private Double heading;
    private Double speed;
    private Double accuracy;
    
    @Column(nullable = false)
    private LocalDateTime timestamp;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    // Index for efficient queries
    @Index(name = "idx_driver_timestamp", columnList = "driverId, timestamp")
    @Index(name = "idx_location_timestamp", columnList = "latitude, longitude, timestamp")
}

// Fare Estimate Entity
@Entity
@Table(name = "fare_estimates")
public class FareEstimate {
    @Id
    private String fareId;
    
    @Column(nullable = false)
    private String riderId;
    
    // Locations
    @Column(nullable = false)
    private Double pickupLatitude;
    
    @Column(nullable = false)
    private Double pickupLongitude;
    
    @Column(nullable = false)
    private Double destinationLatitude;
    
    @Column(nullable = false)
    private Double destinationLongitude;
    
    // Fare breakdown
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal estimatedFare;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal baseFare;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal distanceFare;
    
    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal timeFare;
    
    @Column(nullable = false)
    private String currency;
    
    // Trip estimates
    private Double estimatedDistance;
    private Integer estimatedDuration;
    
    @Column(length = 10000)
    private String routePolyline;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    @Column(nullable = false)
    private LocalDateTime expiresAt;
    
    @Enumerated(EnumType.STRING)
    private FareStatus status;
}
```

### Enums and Constants:

```java
public enum UserStatus {
    ACTIVE, INACTIVE, SUSPENDED, DELETED
}

public enum DriverStatus {
    AVAILABLE, BUSY, OFFLINE, ON_TRIP
}

public enum TripStatus {
    REQUESTED, ACCEPTED, DRIVER_ARRIVED, IN_PROGRESS, COMPLETED, CANCELLED
}

public enum PaymentStatus {
    PENDING, PROCESSING, COMPLETED, FAILED, REFUNDED
}

public enum VehicleType {
    SEDAN, SUV, HATCHBACK, LUXURY, ELECTRIC
}

public enum FareStatus {
    ACTIVE, EXPIRED, USED
}

// Constants
public class UberConstants {
    public static final int DRIVER_SEARCH_RADIUS_KM = 5;
    public static final int MAX_DRIVERS_TO_CONSIDER = 10;
    public static final int RIDE_REQUEST_TIMEOUT_SECONDS = 30;
    public static final int LOCATION_UPDATE_INTERVAL_SECONDS = 5;
    public static final int FARE_ESTIMATE_VALIDITY_MINUTES = 5;
    
    // Pricing constants
    public static final BigDecimal BASE_FARE = new BigDecimal("2.50");
    public static final BigDecimal RATE_PER_MILE = new BigDecimal("1.75");
    public static final BigDecimal RATE_PER_MINUTE = new BigDecimal("0.35");
}
```

### Database Indexes:

```sql
-- Driver locations for proximity searches
CREATE INDEX idx_driver_locations_spatial ON driver_locations 
USING GIST (ST_Point(longitude, latitude));

-- Trip queries
CREATE INDEX idx_trips_rider_status ON trips(rider_id, status);
CREATE INDEX idx_trips_driver_status ON trips(driver_id, status);
CREATE INDEX idx_trips_requested_at ON trips(requested_at);

-- User lookups
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone_number);
CREATE INDEX idx_drivers_status ON users(driver_status) WHERE user_type = 'DRIVER';

-- Fare estimates
CREATE INDEX idx_fare_estimates_rider_created ON fare_estimates(rider_id, created_at);
CREATE INDEX idx_fare_estimates_expires ON fare_estimates(expires_at) WHERE status = 'ACTIVE';
```

---

## PHASE 7: CORE FLOWS - FARE/REQUEST/MATCH/ACCEPT (7 minutes)

### 1. Fare Estimation Flow:

```java
@RestController
@RequestMapping("/api/v1/fares")
public class FareController {
    
    @Autowired
    private FareService fareService;
    
    @PostMapping("/estimate")
    public ResponseEntity<FareEstimateResponse> estimateFare(
            @RequestBody @Valid FareEstimateRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        
        FareEstimateResponse response = fareService.estimateFare(request, user.getUserId());
        return ResponseEntity.ok(response);
    }
}

@Service
public class FareService {
    
    @Autowired
    private FareEstimateRepository fareRepository;
    
    @Autowired
    private MapsService mapsService;
    
    @Autowired
    private PricingService pricingService;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public FareEstimateResponse estimateFare(FareEstimateRequest request, String riderId) {
        // Get route information from Maps API
        RouteInfo routeInfo = mapsService.calculateRoute(
            request.getPickupLocation(),
            request.getDestination()
        );
        
        // Calculate fare using pricing algorithm
        FareBreakdown fareBreakdown = pricingService.calculateFare(
            routeInfo.getDistance(),
            routeInfo.getDuration(),
            request.getRideType()
        );
        
        // Create fare estimate record
        FareEstimate fareEstimate = FareEstimate.builder()
            .fareId(generateFareId())
            .riderId(riderId)
            .pickupLatitude(request.getPickupLocation().getLatitude())
            .pickupLongitude(request.getPickupLocation().getLongitude())
            .destinationLatitude(request.getDestination().getLatitude())
            .destinationLongitude(request.getDestination().getLongitude())
            .estimatedFare(fareBreakdown.getTotalFare())
            .baseFare(fareBreakdown.getBaseFare())
            .distanceFare(fareBreakdown.getDistanceFare())
            .timeFare(fareBreakdown.getTimeFare())
            .currency("USD")
            .estimatedDistance(routeInfo.getDistance())
            .estimatedDuration(routeInfo.getDuration())
            .routePolyline(routeInfo.getPolyline())
            .createdAt(LocalDateTime.now())
            .expiresAt(LocalDateTime.now().plusMinutes(5))
            .status(FareStatus.ACTIVE)
            .build();
        
        fareRepository.save(fareEstimate);
        
        // Cache fare estimate for quick access
        String cacheKey = "fare:" + fareEstimate.getFareId();
        redisTemplate.opsForValue().set(cacheKey, fareEstimate, 
            Duration.ofMinutes(5));
        
        return convertToFareEstimateResponse(fareEstimate, routeInfo);
    }
}

@Service
public class PricingService {
    
    public FareBreakdown calculateFare(double distanceMiles, int durationMinutes, String rideType) {
        BigDecimal baseFare = UberConstants.BASE_FARE;
        BigDecimal distanceFare = UberConstants.RATE_PER_MILE
            .multiply(BigDecimal.valueOf(distanceMiles));
        BigDecimal timeFare = UberConstants.RATE_PER_MINUTE
            .multiply(BigDecimal.valueOf(durationMinutes));
        
        // Apply surge pricing if needed (simplified)
        BigDecimal surgMultiplier = getSurgeMultiplier(rideType);
        
        BigDecimal totalFare = baseFare.add(distanceFare).add(timeFare)
            .multiply(surgMultiplier);
        
        return FareBreakdown.builder()
            .baseFare(baseFare)
            .distanceFare(distanceFare)
            .timeFare(timeFare)
            .surgeMultiplier(surgMultiplier)
            .totalFare(totalFare)
            .build();
    }
    
    private BigDecimal getSurgeMultiplier(String rideType) {
        // Simplified surge pricing logic
        return BigDecimal.ONE; // No surge for now
    }
}
```

### 2. Ride Request Flow:

```java
@RestController
@RequestMapping("/api/v1/rides")
public class RideController {
    
    @Autowired
    private RideService rideService;
    
    @PostMapping
    public ResponseEntity<RideResponse> requestRide(
            @RequestBody @Valid RideRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        
        RideResponse response = rideService.requestRide(request, user.getUserId());
        return ResponseEntity.ok(response);
    }
    
    @GetMapping("/{rideId}")
    public ResponseEntity<RideResponse> getRide(@PathVariable String rideId) {
        RideResponse response = rideService.getRide(rideId);
        return ResponseEntity.ok(response);
    }
}

@Service
@Transactional
public class RideService {
    
    @Autowired
    private TripRepository tripRepository;
    
    @Autowired
    private FareEstimateRepository fareRepository;
    
    @Autowired
    private MatchingService matchingService;
    
    @Autowired
    private NotificationService notificationService;
    
    public RideResponse requestRide(RideRequest request, String riderId) {
        // Validate fare estimate
        FareEstimate fareEstimate = fareRepository.findById(request.getFareId())
            .orElseThrow(() -> new FareEstimateNotFoundException(request.getFareId()));
        
        if (fareEstimate.getStatus() != FareStatus.ACTIVE || 
            fareEstimate.getExpiresAt().isBefore(LocalDateTime.now())) {
            throw new FareEstimateExpiredException("Fare estimate has expired");
        }
        
        if (!fareEstimate.getRiderId().equals(riderId)) {
            throw new UnauthorizedFareAccessException("Fare estimate belongs to different user");
        }
        
        // Create trip record
        Trip trip = Trip.builder()
            .tripId(generateTripId())
            .rider(riderRepository.findById(riderId).orElseThrow())
            .status(TripStatus.REQUESTED)
            .pickupLatitude(fareEstimate.getPickupLatitude())
            .pickupLongitude(fareEstimate.getPickupLongitude())
            .destinationLatitude(fareEstimate.getDestinationLatitude())
            .destinationLongitude(fareEstimate.getDestinationLongitude())
            .estimatedFare(fareEstimate.getEstimatedFare())
            .currency(fareEstimate.getCurrency())
            .estimatedDistance(fareEstimate.getEstimatedDistance())
            .estimatedDuration(fareEstimate.getEstimatedDuration())
            .routePolyline(fareEstimate.getRoutePolyline())
            .paymentMethodId(request.getPaymentMethodId())
            .notes(request.getNotes())
            .requestedAt(LocalDateTime.now())
            .build();
        
        trip = tripRepository.save(trip);
        
        // Mark fare estimate as used
        fareEstimate.setStatus(FareStatus.USED);
        fareRepository.save(fareEstimate);
        
        // Trigger driver matching asynchronously
        matchingService.findAndAssignDriver(trip);
        
        return convertToRideResponse(trip);
    }
}
```

### 3. Driver Matching Flow:

```java
@Service
public class MatchingService {
    
    @Autowired
    private LocationService locationService;
    
    @Autowired
    private DriverService driverService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Async
    public void findAndAssignDriver(Trip trip) {
        try {
            // Find nearby available drivers
            List<NearbyDriver> nearbyDrivers = locationService.findNearbyDrivers(
                trip.getPickupLatitude(),
                trip.getPickupLongitude(),
                UberConstants.DRIVER_SEARCH_RADIUS_KM
            );
            
            if (nearbyDrivers.isEmpty()) {
                handleNoDriversAvailable(trip);
                return;
            }
            
            // Rank drivers by proximity, rating, acceptance rate
            List<RankedDriver> rankedDrivers = rankDrivers(nearbyDrivers, trip);
            
            // Try to assign to drivers in order
            for (RankedDriver rankedDriver : rankedDrivers) {
                if (tryAssignDriver(trip, rankedDriver.getDriver())) {
                    return; // Successfully assigned
                }
            }
            
            // No driver accepted
            handleNoDriverAccepted(trip);
            
        } catch (Exception e) {
            log.error("Error in driver matching for trip {}", trip.getTripId(), e);
            handleMatchingError(trip);
        }
    }
    
    private boolean tryAssignDriver(Trip trip, Driver driver) {
        String lockKey = "driver:lock:" + driver.getDriverId();
        String lockValue = trip.getTripId();
        
        // Try to acquire lock on driver for 30 seconds
        Boolean lockAcquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, lockValue, Duration.ofSeconds(30));
        
        if (!lockAcquired) {
            return false; // Driver is already handling another request
        }
        
        try {
            // Send ride request to driver
            boolean requestSent = notificationService.sendRideRequest(driver, trip);
            
            if (requestSent) {
                // Update trip status
                trip.setDriver(driver);
                trip.setStatus(TripStatus.REQUESTED);
                tripRepository.save(trip);
                
                // Wait for driver response (handled by separate endpoint)
                return true;
            } else {
                // Failed to send notification, release lock
                redisTemplate.delete(lockKey);
                return false;
            }
            
        } catch (Exception e) {
            // Release lock on error
            redisTemplate.delete(lockKey);
            throw e;
        }
    }
    
    private List<RankedDriver> rankDrivers(List<NearbyDriver> nearbyDrivers, Trip trip) {
        return nearbyDrivers.stream()
            .map(nearbyDriver -> {
                Driver driver = nearbyDriver.getDriver();
                double score = calculateDriverScore(driver, nearbyDriver.getDistance(), trip);
                return new RankedDriver(driver, nearbyDriver.getDistance(), score);
            })
            .sorted(Comparator.comparingDouble(RankedDriver::getScore).reversed())
            .limit(UberConstants.MAX_DRIVERS_TO_CONSIDER)
            .collect(Collectors.toList());
    }
    
    private double calculateDriverScore(Driver driver, double distance, Trip trip) {
        // Scoring algorithm considering multiple factors
        double proximityScore = Math.max(0, 100 - (distance * 10)); // Closer is better
        double ratingScore = driver.getRating().doubleValue() * 20; // Rating out of 5, scaled to 100
        double acceptanceScore = driver.getAcceptanceRate() * 0.5; // Acceptance rate bonus
        
        return proximityScore + ratingScore + acceptanceScore;
    }
}
```

### 4. Driver Accept/Decline Flow:

```java
@RestController
@RequestMapping("/api/v1/rides")
public class DriverRideController {
    
    @Autowired
    private DriverRideService driverRideService;
    
    @PatchMapping("/{rideId}/accept")
    public ResponseEntity<RideResponse> acceptRide(
            @PathVariable String rideId,
            @AuthenticationPrincipal UserPrincipal driver) {
        
        RideResponse response = driverRideService.acceptRide(rideId, driver.getUserId());
        return ResponseEntity.ok(response);
    }
    
    @PatchMapping("/{rideId}/decline")
    public ResponseEntity<Void> declineRide(
            @PathVariable String rideId,
            @AuthenticationPrincipal UserPrincipal driver) {
        
        driverRideService.declineRide(rideId, driver.getUserId());
        return ResponseEntity.ok().build();
    }
}

@Service
@Transactional
public class DriverRideService {
    
    @Autowired
    private TripRepository tripRepository;
    
    @Autowired
    private DriverRepository driverRepository;
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Autowired
    private NotificationService notificationService;
    
    @Autowired
    private MatchingService matchingService;
    
    public RideResponse acceptRide(String rideId, String driverId) {
        // Find the trip
        Trip trip = tripRepository.findById(rideId)
            .orElseThrow(() -> new TripNotFoundException(rideId));
        
        // Validate driver is assigned to this trip
        if (trip.getDriver() == null || !trip.getDriver().getDriverId().equals(driverId)) {
            throw new UnauthorizedDriverAccessException("Driver not assigned to this trip");
        }
        
        // Check if trip is still in requested state
        if (trip.getStatus() != TripStatus.REQUESTED) {
            throw new InvalidTripStateException("Trip is no longer available for acceptance");
        }
        
        // Verify driver lock
        String lockKey = "driver:lock:" + driverId;
        String lockValue = redisTemplate.opsForValue().get(lockKey);
        
        if (!rideId.equals(lockValue)) {
            throw new DriverLockException("Driver lock mismatch");
        }
        
        // Update trip status
        trip.setStatus(TripStatus.ACCEPTED);
        trip.setAcceptedAt(LocalDateTime.now());
        trip = tripRepository.save(trip);
        
        // Update driver status
        Driver driver = trip.getDriver();
        driver.setDriverStatus(DriverStatus.ON_TRIP);
        driverRepository.save(driver);
        
        // Release driver lock
        redisTemplate.delete(lockKey);
        
        // Notify rider
        notificationService.notifyRiderDriverAccepted(trip);
        
        return convertToRideResponse(trip);
    }
    
    public void declineRide(String rideId, String driverId) {
        // Find the trip
        Trip trip = tripRepository.findById(rideId)
            .orElseThrow(() -> new TripNotFoundException(rideId));
        
        // Validate driver is assigned to this trip
        if (trip.getDriver() == null || !trip.getDriver().getDriverId().equals(driverId)) {
            throw new UnauthorizedDriverAccessException("Driver not assigned to this trip");
        }
        
        // Release driver lock
        String lockKey = "driver:lock:" + driverId;
        redisTemplate.delete(lockKey);
        
        // Update driver acceptance rate
        Driver driver = trip.getDriver();
        updateDriverAcceptanceRate(driver, false);
        
        // Remove driver assignment and continue matching
        trip.setDriver(null);
        trip.setStatus(TripStatus.REQUESTED);
        tripRepository.save(trip);
        
        // Continue matching with next available driver
        matchingService.findAndAssignDriver(trip);
    }
    
    private void updateDriverAcceptanceRate(Driver driver, boolean accepted) {
        // Update acceptance rate calculation
        int totalRequests = driver.getTotalTrips() + 1;
        int acceptedRequests = accepted ? driver.getAcceptanceRate() + 1 : driver.getAcceptanceRate();
        
        driver.setAcceptanceRate((acceptedRequests * 100) / totalRequests);
        driverRepository.save(driver);
    }
}
```

This completes the first part of the comprehensive Uber HLD with detailed Java implementations covering the core flows. The design includes proper error handling, concurrency control, and real-world considerations for a production ride-sharing platform.
---

## PHASE 8: DEEP DIVE - LOCATION SERVICES & GEOSPATIAL QUERIES (5 minutes)

### The Location Update Challenge:

**Problem:** With 10M drivers sending location updates every 5 seconds, we have 2M location updates per second. Traditional databases cannot handle this write load efficiently.

### Approach 1: Direct Database Writes (❌ Poor Solution)

```java
// BAD: Writing every location update directly to PostgreSQL
@Service
public class BasicLocationService {
    
    @Autowired
    private DriverLocationRepository locationRepository;
    
    public void updateDriverLocation(String driverId, LocationUpdate update) {
        // This creates 2M database writes per second!
        DriverLocation location = DriverLocation.builder()
            .driverId(driverId)
            .latitude(update.getLatitude())
            .longitude(update.getLongitude())
            .timestamp(LocalDateTime.now())
            .build();
        
        locationRepository.save(location); // Database will crash!
    }
    
    public List<Driver> findNearbyDrivers(double lat, double lng, double radiusKm) {
        // This requires full table scan - extremely slow!
        return locationRepository.findDriversWithinRadius(lat, lng, radiusKm);
    }
}
```

**Problems:**
- 2M writes/second will overwhelm any SQL database
- Proximity searches require full table scans
- High latency and poor scalability
- Expensive database scaling costs

### Approach 2: Batch Processing (✅ Good Solution)

```java
@Service
public class BatchLocationService {
    
    @Autowired
    private DriverLocationRepository locationRepository;
    
    private final Map<String, LocationUpdate> locationBuffer = new ConcurrentHashMap<>();
    
    public void updateDriverLocation(String driverId, LocationUpdate update) {
        // Buffer location updates in memory
        locationBuffer.put(driverId, update);
    }
    
    @Scheduled(fixedRate = 30000) // Every 30 seconds
    public void flushLocationUpdates() {
        if (locationBuffer.isEmpty()) {
            return;
        }
        
        List<DriverLocation> locations = locationBuffer.entrySet().stream()
            .map(entry -> DriverLocation.builder()
                .driverId(entry.getKey())
                .latitude(entry.getValue().getLatitude())
                .longitude(entry.getValue().getLongitude())
                .timestamp(LocalDateTime.now())
                .build())
            .collect(Collectors.toList());
        
        // Batch insert - much more efficient
        locationRepository.saveAll(locations);
        locationBuffer.clear();
    }
    
    public List<Driver> findNearbyDrivers(double lat, double lng, double radiusKm) {
        // Still requires spatial indexing for efficiency
        return locationRepository.findDriversWithinRadiusUsingPostGIS(lat, lng, radiusKm);
    }
}
```

**Improvements:**
- Reduces database writes by 60x (30-second batches)
- Uses PostGIS for efficient spatial queries
- Better database performance

**Challenges:**
- 30-second delay in location updates
- Still not real-time enough for ride matching
- Complex spatial query optimization needed

### Approach 3: Redis Geospatial (✅ Great Solution)

```java
@Service
public class RedisLocationService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Autowired
    private DriverRepository driverRepository;
    
    private static final String DRIVER_LOCATION_KEY = "driver:locations";
    private static final String DRIVER_STATUS_KEY = "driver:status:";
    
    public void updateDriverLocation(String driverId, LocationUpdate update) {
        // Store location in Redis geospatial set
        redisTemplate.opsForGeo().add(
            DRIVER_LOCATION_KEY,
            new Point(update.getLongitude(), update.getLatitude()),
            driverId
        );
        
        // Set TTL to auto-expire stale locations
        redisTemplate.expire(DRIVER_LOCATION_KEY, Duration.ofMinutes(10));
        
        // Update driver metadata
        DriverLocationMetadata metadata = DriverLocationMetadata.builder()
            .driverId(driverId)
            .heading(update.getHeading())
            .speed(update.getSpeed())
            .accuracy(update.getAccuracy())
            .timestamp(update.getTimestamp())
            .build();
        
        redisTemplate.opsForValue().set(
            DRIVER_STATUS_KEY + driverId,
            metadata,
            Duration.ofMinutes(10)
        );
    }
    
    public List<NearbyDriver> findNearbyDrivers(double lat, double lng, double radiusKm) {
        // Use Redis GEOSEARCH for ultra-fast proximity queries
        GeoResults<RedisGeoCommands.GeoLocation<String>> results = 
            redisTemplate.opsForGeo().search(
                DRIVER_LOCATION_KEY,
                GeoReference.fromCoordinate(lng, lat),
                Distance.of(radiusKm, Metrics.KILOMETERS),
                GeoSearchCommandArgs.newGeoSearchArgs()
                    .includeDistance()
                    .includeCoordinates()
                    .sortAscending()
                    .limit(50)
            );
        
        return results.getContent().stream()
            .map(this::convertToNearbyDriver)
            .filter(this::isDriverAvailable)
            .collect(Collectors.toList());
    }
    
    private NearbyDriver convertToNearbyDriver(GeoResult<RedisGeoCommands.GeoLocation<String>> result) {
        String driverId = result.getContent().getName();
        double distance = result.getDistance().getValue();
        Point coordinates = result.getContent().getPoint();
        
        // Get driver details from cache or database
        Driver driver = getDriverDetails(driverId);
        
        return NearbyDriver.builder()
            .driver(driver)
            .distance(distance)
            .latitude(coordinates.getY())
            .longitude(coordinates.getX())
            .build();
    }
    
    private boolean isDriverAvailable(NearbyDriver nearbyDriver) {
        String statusKey = DRIVER_STATUS_KEY + nearbyDriver.getDriver().getDriverId();
        DriverLocationMetadata metadata = (DriverLocationMetadata) 
            redisTemplate.opsForValue().get(statusKey);
        
        if (metadata == null) {
            return false; // Stale location data
        }
        
        // Check if location is recent (within 30 seconds)
        return metadata.getTimestamp().isAfter(LocalDateTime.now().minusSeconds(30)) &&
               nearbyDriver.getDriver().getDriverStatus() == DriverStatus.AVAILABLE;
    }
    
    @Scheduled(fixedRate = 60000) // Every minute
    public void cleanupStaleLocations() {
        // Remove drivers who haven't updated location in 5 minutes
        LocalDateTime cutoff = LocalDateTime.now().minusMinutes(5);
        
        // This is a simplified cleanup - in production, you'd use Lua scripts
        Set<String> allDrivers = redisTemplate.opsForGeo().members(DRIVER_LOCATION_KEY);
        
        for (String driverId : allDrivers) {
            DriverLocationMetadata metadata = (DriverLocationMetadata) 
                redisTemplate.opsForValue().get(DRIVER_STATUS_KEY + driverId);
            
            if (metadata == null || metadata.getTimestamp().isBefore(cutoff)) {
                redisTemplate.opsForGeo().remove(DRIVER_LOCATION_KEY, driverId);
                redisTemplate.delete(DRIVER_STATUS_KEY + driverId);
            }
        }
    }
}
```

### Advanced Geospatial Optimizations:

```java
@Service
public class OptimizedLocationService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    // Partition by geographic regions for better performance
    private static final String LOCATION_KEY_PREFIX = "drivers:geo:";
    
    public void updateDriverLocation(String driverId, LocationUpdate update) {
        // Determine geographic partition (e.g., by city or grid)
        String geoPartition = determineGeoPartition(update.getLatitude(), update.getLongitude());
        String locationKey = LOCATION_KEY_PREFIX + geoPartition;
        
        // Use Lua script for atomic operations
        String luaScript = 
            "redis.call('GEOADD', KEYS[1], ARGV[1], ARGV[2], ARGV[3]) " +
            "redis.call('EXPIRE', KEYS[1], ARGV[4]) " +
            "redis.call('SETEX', KEYS[2], ARGV[4], ARGV[5])";
        
        redisTemplate.execute(
            RedisScript.of(luaScript),
            Arrays.asList(locationKey, DRIVER_STATUS_KEY + driverId),
            String.valueOf(update.getLongitude()),
            String.valueOf(update.getLatitude()),
            driverId,
            "600", // 10 minutes TTL
            serializeMetadata(update)
        );
    }
    
    public List<NearbyDriver> findNearbyDrivers(double lat, double lng, double radiusKm) {
        // Determine which geographic partitions to search
        List<String> partitionsToSearch = determineSearchPartitions(lat, lng, radiusKm);
        
        List<NearbyDriver> allNearbyDrivers = new ArrayList<>();
        
        // Search in parallel across partitions
        List<CompletableFuture<List<NearbyDriver>>> futures = partitionsToSearch.stream()
            .map(partition -> CompletableFuture.supplyAsync(() -> 
                searchPartition(partition, lat, lng, radiusKm)))
            .collect(Collectors.toList());
        
        // Combine results from all partitions
        for (CompletableFuture<List<NearbyDriver>> future : futures) {
            try {
                allNearbyDrivers.addAll(future.get(100, TimeUnit.MILLISECONDS));
            } catch (Exception e) {
                log.warn("Failed to search partition", e);
            }
        }
        
        // Sort by distance and return top results
        return allNearbyDrivers.stream()
            .sorted(Comparator.comparingDouble(NearbyDriver::getDistance))
            .limit(20)
            .collect(Collectors.toList());
    }
    
    private String determineGeoPartition(double lat, double lng) {
        // Simple grid-based partitioning
        int latGrid = (int) (lat * 100); // ~1km precision
        int lngGrid = (int) (lng * 100);
        return latGrid + ":" + lngGrid;
    }
    
    private List<String> determineSearchPartitions(double lat, double lng, double radiusKm) {
        // Calculate which grid cells to search based on radius
        double gridSize = 0.01; // ~1km
        int gridRadius = (int) Math.ceil(radiusKm / 111.0 / gridSize); // Convert km to grid units
        
        List<String> partitions = new ArrayList<>();
        int centerLatGrid = (int) (lat * 100);
        int centerLngGrid = (int) (lng * 100);
        
        for (int latOffset = -gridRadius; latOffset <= gridRadius; latOffset++) {
            for (int lngOffset = -gridRadius; lngOffset <= gridRadius; lngOffset++) {
                partitions.add((centerLatGrid + latOffset) + ":" + (centerLngGrid + lngOffset));
            }
        }
        
        return partitions;
    }
}
```

### Smart Location Update Strategy:

```java
@Service
public class SmartLocationService {
    
    public void updateDriverLocationSmart(String driverId, LocationUpdate update) {
        // Get previous location
        DriverLocationMetadata previousLocation = getPreviousLocation(driverId);
        
        if (previousLocation != null) {
            // Calculate distance moved
            double distanceMoved = calculateDistance(
                previousLocation.getLatitude(), previousLocation.getLongitude(),
                update.getLatitude(), update.getLongitude()
            );
            
            // Calculate speed
            double speed = update.getSpeed() != null ? update.getSpeed() : 0.0;
            
            // Adaptive update frequency based on movement
            if (shouldSkipUpdate(distanceMoved, speed, previousLocation.getTimestamp())) {
                return; // Skip this update to reduce load
            }
        }
        
        // Update location
        updateDriverLocation(driverId, update);
    }
    
    private boolean shouldSkipUpdate(double distanceMoved, double speed, LocalDateTime lastUpdate) {
        long secondsSinceLastUpdate = ChronoUnit.SECONDS.between(lastUpdate, LocalDateTime.now());
        
        // Skip update if:
        // 1. Driver hasn't moved much (< 50 meters) AND
        // 2. Speed is low (< 5 mph) AND
        // 3. Last update was recent (< 10 seconds ago)
        return distanceMoved < 0.05 && // 50 meters
               speed < 8.0 && // 5 mph
               secondsSinceLastUpdate < 10;
    }
    
    private double calculateDistance(double lat1, double lng1, double lat2, double lng2) {
        // Haversine formula for distance calculation
        double R = 6371; // Earth's radius in kilometers
        double dLat = Math.toRadians(lat2 - lat1);
        double dLng = Math.toRadians(lng2 - lng1);
        
        double a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
                   Math.cos(Math.toRadians(lat1)) * Math.cos(Math.toRadians(lat2)) *
                   Math.sin(dLng / 2) * Math.sin(dLng / 2);
        
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        return R * c;
    }
}
```

---

## PHASE 9: DEEP DIVE - MATCHING ALGORITHM & CONCURRENCY (5 minutes)

### The Driver Assignment Challenge:

**Problem:** Ensure only one ride is assigned to each driver at a time, and each ride gets exactly one driver, even under high concurrency.

### Approach 1: Application-Level Locking (❌ Poor Solution)

```java
// BAD: In-memory locks don't work across multiple service instances
@Service
public class BasicMatchingService {
    
    private final Set<String> lockedDrivers = ConcurrentHashMap.newKeySet();
    
    public boolean tryAssignDriver(Trip trip, Driver driver) {
        // This only works within a single JVM instance!
        if (lockedDrivers.contains(driver.getDriverId())) {
            return false;
        }
        
        lockedDrivers.add(driver.getDriverId());
        
        try {
            // Send ride request
            notificationService.sendRideRequest(driver, trip);
            return true;
        } finally {
            // Manual cleanup - what if service crashes?
            scheduleUnlock(driver.getDriverId(), 30000);
        }
    }
}
```

**Problems:**
- Locks only work within single service instance
- No coordination between multiple matching service instances
- Manual cleanup is unreliable
- Race conditions between service instances

### Approach 2: Database-Level Locking (✅ Good Solution)

```java
@Service
@Transactional
public class DatabaseLockingMatchingService {
    
    @Autowired
    private DriverRepository driverRepository;
    
    @Autowired
    private TripRepository tripRepository;
    
    public boolean tryAssignDriver(Trip trip, Driver driver) {
        try {
            // Use database transaction with row-level locking
            Driver lockedDriver = driverRepository.findByIdForUpdate(driver.getDriverId());
            
            if (lockedDriver.getDriverStatus() != DriverStatus.AVAILABLE) {
                return false; // Driver is busy
            }
            
            // Update driver status atomically
            lockedDriver.setDriverStatus(DriverStatus.BUSY);
            lockedDriver.setCurrentTripId(trip.getTripId());
            driverRepository.save(lockedDriver);
            
            // Update trip
            trip.setDriver(lockedDriver);
            trip.setStatus(TripStatus.REQUESTED);
            tripRepository.save(trip);
            
            // Send notification
            notificationService.sendRideRequest(lockedDriver, trip);
            
            // Schedule timeout handling
            scheduleDriverTimeout(trip.getTripId(), lockedDriver.getDriverId());
            
            return true;
            
        } catch (Exception e) {
            log.error("Failed to assign driver", e);
            return false;
        }
    }
    
    @Scheduled(fixedRate = 10000) // Every 10 seconds
    public void cleanupExpiredRequests() {
        LocalDateTime cutoff = LocalDateTime.now().minusSeconds(30);
        
        List<Trip> expiredTrips = tripRepository.findByStatusAndRequestedAtBefore(
            TripStatus.REQUESTED, cutoff);
        
        for (Trip trip : expiredTrips) {
            if (trip.getDriver() != null) {
                // Release driver
                Driver driver = trip.getDriver();
                driver.setDriverStatus(DriverStatus.AVAILABLE);
                driver.setCurrentTripId(null);
                driverRepository.save(driver);
                
                // Continue matching with next driver
                continueMatching(trip);
            }
        }
    }
}
```

**Improvements:**
- Database ensures consistency across service instances
- Atomic updates prevent race conditions
- Automatic cleanup of expired requests

**Challenges:**
- Cron job introduces delay in cleanup
- Database locks can cause contention
- Complex timeout management

### Approach 3: Distributed Locking with Redis (✅ Great Solution)

```java
@Service
public class DistributedLockMatchingService {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Autowired
    private TripRepository tripRepository;
    
    @Autowired
    private NotificationService notificationService;
    
    public boolean tryAssignDriver(Trip trip, Driver driver) {
        String driverLockKey = "driver:lock:" + driver.getDriverId();
        String tripLockKey = "trip:lock:" + trip.getTripId();
        String lockValue = trip.getTripId() + ":" + System.currentTimeMillis();
        
        try {
            // Acquire locks with TTL (automatic expiration)
            Boolean driverLocked = redisTemplate.opsForValue()
                .setIfAbsent(driverLockKey, lockValue, Duration.ofSeconds(30));
            
            Boolean tripLocked = redisTemplate.opsForValue()
                .setIfAbsent(tripLockKey, lockValue, Duration.ofSeconds(30));
            
            if (!driverLocked || !tripLocked) {
                // Release any acquired locks
                releaseLocks(driverLockKey, tripLockKey, lockValue);
                return false;
            }
            
            // Both locks acquired successfully
            return processDriverAssignment(trip, driver, driverLockKey, tripLockKey, lockValue);
            
        } catch (Exception e) {
            log.error("Error in driver assignment", e);
            releaseLocks(driverLockKey, tripLockKey, lockValue);
            return false;
        }
    }
    
    private boolean processDriverAssignment(Trip trip, Driver driver, 
            String driverLockKey, String tripLockKey, String lockValue) {
        
        try {
            // Update trip in database
            trip.setDriver(driver);
            trip.setStatus(TripStatus.REQUESTED);
            trip.setRequestedAt(LocalDateTime.now());
            tripRepository.save(trip);
            
            // Send push notification to driver
            boolean notificationSent = notificationService.sendRideRequest(driver, trip);
            
            if (!notificationSent) {
                // Failed to send notification, rollback
                trip.setDriver(null);
                trip.setStatus(TripStatus.SEARCHING);
                tripRepository.save(trip);
                
                releaseLocks(driverLockKey, tripLockKey, lockValue);
                return false;
            }
            
            // Store assignment metadata in Redis
            storeAssignmentMetadata(trip, driver, lockValue);
            
            return true;
            
        } catch (Exception e) {
            log.error("Failed to process driver assignment", e);
            releaseLocks(driverLockKey, tripLockKey, lockValue);
            return false;
        }
    }
    
    private void releaseLocks(String driverLockKey, String tripLockKey, String lockValue) {
        // Use Lua script for atomic lock release
        String luaScript = 
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    redis.call('del', KEYS[1]) " +
            "end " +
            "if redis.call('get', KEYS[2]) == ARGV[1] then " +
            "    redis.call('del', KEYS[2]) " +
            "end";
        
        redisTemplate.execute(
            RedisScript.of(luaScript),
            Arrays.asList(driverLockKey, tripLockKey),
            lockValue
        );
    }
    
    public void handleDriverResponse(String tripId, String driverId, boolean accepted) {
        String driverLockKey = "driver:lock:" + driverId;
        String tripLockKey = "trip:lock:" + tripId;
        
        if (accepted) {
            handleDriverAcceptance(tripId, driverId, driverLockKey, tripLockKey);
        } else {
            handleDriverDecline(tripId, driverId, driverLockKey, tripLockKey);
        }
    }
    
    private void handleDriverAcceptance(String tripId, String driverId, 
            String driverLockKey, String tripLockKey) {
        
        try {
            // Update trip status
            Trip trip = tripRepository.findById(tripId).orElseThrow();
            trip.setStatus(TripStatus.ACCEPTED);
            trip.setAcceptedAt(LocalDateTime.now());
            tripRepository.save(trip);
            
            // Release locks (driver is now committed to trip)
            redisTemplate.delete(driverLockKey);
            redisTemplate.delete(tripLockKey);
            
            // Notify rider
            notificationService.notifyRiderDriverAccepted(trip);
            
        } catch (Exception e) {
            log.error("Error handling driver acceptance", e);
        }
    }
    
    private void handleDriverDecline(String tripId, String driverId, 
            String driverLockKey, String tripLockKey) {
        
        try {
            // Release locks
            redisTemplate.delete(driverLockKey);
            redisTemplate.delete(tripLockKey);
            
            // Update driver stats
            updateDriverDeclineStats(driverId);
            
            // Continue matching with next driver
            Trip trip = tripRepository.findById(tripId).orElseThrow();
            trip.setDriver(null);
            trip.setStatus(TripStatus.SEARCHING);
            tripRepository.save(trip);
            
            // Trigger next matching attempt
            matchingService.continueMatching(trip);
            
        } catch (Exception e) {
            log.error("Error handling driver decline", e);
        }
    }
}
```

### Advanced Matching Algorithm:

```java
@Service
public class AdvancedMatchingService {
    
    @Autowired
    private LocationService locationService;
    
    @Autowired
    private DriverService driverService;
    
    @Autowired
    private MapsService mapsService;
    
    public List<RankedDriver> findOptimalDrivers(Trip trip) {
        // Step 1: Find nearby drivers using geospatial query
        List<NearbyDriver> nearbyDrivers = locationService.findNearbyDrivers(
            trip.getPickupLatitude(),
            trip.getPickupLongitude(),
            UberConstants.DRIVER_SEARCH_RADIUS_KM
        );
        
        // Step 2: Filter available drivers
        List<NearbyDriver> availableDrivers = nearbyDrivers.stream()
            .filter(this::isDriverAvailable)
            .collect(Collectors.toList());
        
        if (availableDrivers.isEmpty()) {
            return Collections.emptyList();
        }
        
        // Step 3: Calculate ETA for each driver
        List<DriverWithETA> driversWithETA = calculateETAs(availableDrivers, trip);
        
        // Step 4: Apply matching algorithm
        return rankDrivers(driversWithETA, trip);
    }
    
    private List<DriverWithETA> calculateETAs(List<NearbyDriver> drivers, Trip trip) {
        // Batch ETA calculation for efficiency
        List<CompletableFuture<DriverWithETA>> futures = drivers.stream()
            .map(nearbyDriver -> CompletableFuture.supplyAsync(() -> {
                try {
                    RouteInfo route = mapsService.calculateRoute(
                        new Location(nearbyDriver.getLatitude(), nearbyDriver.getLongitude()),
                        new Location(trip.getPickupLatitude(), trip.getPickupLongitude())
                    );
                    
                    return DriverWithETA.builder()
                        .driver(nearbyDriver.getDriver())
                        .straightLineDistance(nearbyDriver.getDistance())
                        .routeDistance(route.getDistance())
                        .eta(route.getDuration())
                        .build();
                        
                } catch (Exception e) {
                    // Fallback to straight-line distance
                    double estimatedETA = nearbyDriver.getDistance() / 0.5; // Assume 30 mph average
                    
                    return DriverWithETA.builder()
                        .driver(nearbyDriver.getDriver())
                        .straightLineDistance(nearbyDriver.getDistance())
                        .routeDistance(nearbyDriver.getDistance())
                        .eta((int) (estimatedETA * 60)) // Convert to seconds
                        .build();
                }
            }))
            .collect(Collectors.toList());
        
        // Wait for all ETA calculations (with timeout)
        return futures.stream()
            .map(future -> {
                try {
                    return future.get(2, TimeUnit.SECONDS);
                } catch (Exception e) {
                    return null;
                }
            })
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }
    
    private List<RankedDriver> rankDrivers(List<DriverWithETA> driversWithETA, Trip trip) {
        return driversWithETA.stream()
            .map(driverWithETA -> {
                double score = calculateDriverScore(driverWithETA, trip);
                return RankedDriver.builder()
                    .driver(driverWithETA.getDriver())
                    .eta(driverWithETA.getEta())
                    .distance(driverWithETA.getRouteDistance())
                    .score(score)
                    .build();
            })
            .sorted(Comparator.comparingDouble(RankedDriver::getScore).reversed())
            .limit(5) // Top 5 drivers
            .collect(Collectors.toList());
    }
    
    private double calculateDriverScore(DriverWithETA driverWithETA, Trip trip) {
        Driver driver = driverWithETA.getDriver();
        
        // Multi-factor scoring algorithm
        double etaScore = calculateETAScore(driverWithETA.getEta());
        double ratingScore = calculateRatingScore(driver.getRating());
        double acceptanceScore = calculateAcceptanceScore(driver.getAcceptanceRate());
        double experienceScore = calculateExperienceScore(driver.getTotalTrips());
        double directionScore = calculateDirectionScore(driver, trip);
        
        // Weighted combination
        return (etaScore * 0.4) +           // 40% weight on ETA
               (ratingScore * 0.25) +       // 25% weight on rating
               (acceptanceScore * 0.15) +   // 15% weight on acceptance rate
               (experienceScore * 0.1) +    // 10% weight on experience
               (directionScore * 0.1);      // 10% weight on direction
    }
    
    private double calculateETAScore(int etaSeconds) {
        // Score decreases as ETA increases
        // Perfect score (100) for 0 ETA, decreases to 0 at 30 minutes
        return Math.max(0, 100 - (etaSeconds / 18.0)); // 1800 seconds = 30 minutes
    }
    
    private double calculateRatingScore(BigDecimal rating) {
        // Convert 5-star rating to 0-100 scale
        return rating.doubleValue() * 20;
    }
    
    private double calculateAcceptanceScore(Integer acceptanceRate) {
        // Acceptance rate is already 0-100
        return acceptanceRate != null ? acceptanceRate : 50; // Default to 50 if unknown
    }
    
    private double calculateExperienceScore(Integer totalTrips) {
        // Logarithmic scale for experience
        if (totalTrips == null || totalTrips == 0) {
            return 0;
        }
        return Math.min(100, Math.log10(totalTrips) * 25); // Max score at 10,000 trips
    }
    
    private double calculateDirectionScore(Driver driver, Trip trip) {
        // Bonus if driver is heading towards pickup location
        // This would require current driver heading and route analysis
        // Simplified implementation returns neutral score
        return 50;
    }
}
```

### Queue-Based Matching for High Load:

```java
@Service
public class QueueBasedMatchingService {
    
    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void requestRide(Trip trip) {
        // Add ride request to queue instead of immediate processing
        RideRequestEvent event = RideRequestEvent.builder()
            .tripId(trip.getTripId())
            .riderId(trip.getRider().getUserId())
            .pickupLocation(new Location(trip.getPickupLatitude(), trip.getPickupLongitude()))
            .destination(new Location(trip.getDestinationLatitude(), trip.getDestinationLongitude()))
            .requestedAt(trip.getRequestedAt())
            .priority(calculatePriority(trip))
            .build();
        
        // Partition by geographic region for better load distribution
        String partitionKey = determinePartition(trip.getPickupLatitude(), trip.getPickupLongitude());
        
        kafkaTemplate.send("ride-requests", partitionKey, event);
    }
    
    @KafkaListener(topics = "ride-requests", concurrency = "10")
    public void processRideRequest(RideRequestEvent event) {
        try {
            // Process ride request with distributed locking
            Trip trip = tripRepository.findById(event.getTripId()).orElseThrow();
            
            if (trip.getStatus() != TripStatus.SEARCHING) {
                return; // Trip already processed or cancelled
            }
            
            // Find and assign driver
            List<RankedDriver> rankedDrivers = findOptimalDrivers(trip);
            
            if (rankedDrivers.isEmpty()) {
                handleNoDriversAvailable(trip);
                return;
            }
            
            // Try to assign drivers in order
            for (RankedDriver rankedDriver : rankedDrivers) {
                if (tryAssignDriver(trip, rankedDriver.getDriver())) {
                    return; // Successfully assigned
                }
            }
            
            // No driver accepted, retry after delay
            scheduleRetry(event);
            
        } catch (Exception e) {
            log.error("Error processing ride request", e);
            scheduleRetry(event);
        }
    }
    
    private void scheduleRetry(RideRequestEvent event) {
        // Exponential backoff for retries
        int retryCount = event.getRetryCount() + 1;
        long delayMs = Math.min(30000, 1000 * (1L << retryCount)); // Max 30 seconds
        
        if (retryCount <= 5) {
            event.setRetryCount(retryCount);
            
            // Schedule delayed retry
            CompletableFuture.delayedExecutor(delayMs, TimeUnit.MILLISECONDS)
                .execute(() -> kafkaTemplate.send("ride-requests", event));
        } else {
            // Max retries exceeded
            handleMatchingFailure(event.getTripId());
        }
    }
}
```

---

## FINAL ARCHITECTURE DIAGRAM WITH OPTIMIZATIONS

```
                                    ┌─────────────────────────────────────┐
                                    │            CLIENT LAYER             │
                                    ├─────────────────────────────────────┤
                                    │  Rider App  │  Driver App │ Admin   │
                                    └─────────────────────────────────────┘
                                                      │
                                    ┌─────────────────────────────────────┐
                                    │         EDGE & CDN LAYER            │
                                    ├─────────────────────────────────────┤
                                    │ CloudFront │ Route53 │ WAF │ ALB    │
                                    └─────────────────────────────────────┘
                                                      │
                                    ┌─────────────────────────────────────┐
                                    │           API GATEWAY               │
                                    ├─────────────────────────────────────┤
                                    │ Auth │ Rate Limit │ Circuit Breaker │
                                    └─────────────────────────────────────┘
                                                      │
                        ┌─────────────────────────────┼─────────────────────────────┐
                        │                             │                             │
                        ▼                             ▼                             ▼
        ┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
        │   LOCATION SERVICE      │   │    RIDE SERVICE         │   │  MATCHING SERVICE       │
        ├─────────────────────────┤   ├─────────────────────────┤   ├─────────────────────────┤
        │ • Redis Geospatial      │   │ • Fare Calculation      │   │ • Driver Ranking        │
        │ • Smart Updates         │   │ • Trip Management       │   │ • Distributed Locks     │
        │ • Proximity Search      │   │ • Payment Integration   │   │ • Queue Processing      │
        │ • Geographic Sharding   │   │ • Route Optimization    │   │ • Retry Logic           │
        └─────────────────────────┘   └─────────────────────────┘   └─────────────────────────┘
                        │                             │                             │
                        ▼                             ▼                             ▼
        ┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
        │    USER SERVICE         │   │  NOTIFICATION SERVICE   │   │   PAYMENT SERVICE       │
        ├─────────────────────────┤   ├─────────────────────────┤   ├─────────────────────────┤
        │ • Authentication        │   │ • Push Notifications    │   │ • Stripe Integration    │
        │ • Profile Management    │   │ • WebSocket Connections │   │ • Billing & Payouts     │
        │ • Trip History          │   │ • SMS & Email           │   │ • Fraud Detection       │
        │ • Rating System         │   │ • Real-time Updates     │   │ • Compliance            │
        └─────────────────────────┘   └─────────────────────────┘   └─────────────────────────┘
                                                      │
                        ┌─────────────────────────────┼─────────────────────────────┐
                        │                             │                             │
                        ▼                             ▼                             ▼
        ┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
        │    REDIS CLUSTER        │   │  POSTGRESQL CLUSTER     │   │     KAFKA CLUSTER       │
        ├─────────────────────────┤   ├─────────────────────────┤   ├─────────────────────────┤
        │ • Location Data (2M/s)  │   │ • User Data             │   │ • Ride Requests         │
        │ • Distributed Locks     │   │ • Trip Records          │   │ • Location Updates      │
        │ • Session Management    │   │ • Payment Data          │   │ • Event Streaming       │
        │ • Geospatial Queries    │   │ • Analytics             │   │ • Dead Letter Queue     │
        │ • 50 Instances          │   │ • Read Replicas         │   │ • Geographic Partitions │
        └─────────────────────────┘   └─────────────────────────┘   └─────────────────────────┘
                                                      │
                        ┌─────────────────────────────┼─────────────────────────────┐
                        │                             │                             │
                        ▼                             ▼                             ▼
        ┌─────────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────┐
        │      S3 STORAGE         │   │    EXTERNAL APIS        │   │    MONITORING           │
        ├─────────────────────────┤   ├─────────────────────────┤   ├─────────────────────────┤
        │ • Trip Data Archive     │   │ • Google Maps API       │   │ • Prometheus Metrics    │
        │ • Analytics Data        │   │ • Stripe Payments       │   │ • Grafana Dashboards    │
        │ • Backup Storage        │   │ • Twilio SMS            │   │ • ELK Stack Logging     │
        │ • Driver Documents      │   │ • Firebase Push         │   │ • Jaeger Tracing        │
        └─────────────────────────┘   └─────────────────────────┘   └─────────────────────────┘
```

## KEY PERFORMANCE METRICS

```
┌─────────────────────────┬──────────────────┬──────────────────┐
│ Metric                  │ Target           │ Peak Load        │
├─────────────────────────┼──────────────────┼──────────────────┤
│ Location Updates/sec    │ 2M globally      │ 5M during events │
│ Ride Matching Time     │ < 30 seconds     │ < 60 seconds     │
│ Proximity Search       │ < 100ms          │ < 200ms          │
│ Driver Assignment      │ < 5 seconds      │ < 10 seconds     │
│ System Availability    │ 99.9%            │ 99.9%            │
│ Concurrent Users       │ 1M globally      │ 3M during events │
│ Database QPS           │ 100K reads       │ 300K reads       │
│ Redis QPS              │ 10M operations   │ 25M operations   │
│ Double Assignment Rate │ 0%               │ 0%               │
└─────────────────────────┴──────────────────┴──────────────────┘
```

This comprehensive Uber HLD covers all aspects of a production-ready ride-sharing platform with advanced geospatial optimizations, distributed concurrency control, and intelligent matching algorithms that can handle millions of concurrent users and location updates.