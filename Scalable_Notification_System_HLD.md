# High-Level Design: Scalable Notification System

## 1. PROBLEM STATEMENT & REQUIREMENTS

### Functional Requirements
- **Multi-Channel Support**: Email, SMS, Push notifications, In-app notifications
- **Message Types**: Transactional (OTP, receipts), Promotional (marketing), System alerts
- **Template Management**: Dynamic content with personalization
- **User Preferences**: Opt-in/opt-out, channel preferences, frequency controls
- **Delivery Tracking**: Status updates, delivery confirmations, failure handling
- **Scheduling**: Immediate, delayed, and recurring notifications

### Non-Functional Requirements
- **Scale**: 100M+ notifications per day, 10K+ notifications per second peak
- **Latency**: <100ms API response, <5 seconds delivery for critical messages
- **Availability**: 99.9% uptime, graceful degradation
- **Reliability**: At-least-once delivery guarantee, exactly-once user experience
- **Consistency**: Eventually consistent across channels
- **Security**: PII protection, secure credential management

### Constraints
- **Third-party Dependencies**: External providers (SendGrid, Twilio, FCM)
- **Rate Limits**: Provider-specific throttling (1000 emails/hour, 100 SMS/min)
- **Cost Optimization**: Minimize provider costs, efficient resource utilization
- **Compliance**: GDPR, CAN-SPAM, data retention policies

---

## 2. CAPACITY ESTIMATION

### Traffic Estimates
- **Daily Notifications**: 100M
- **Peak QPS**: 10,000 notifications/second
- **Read:Write Ratio**: 1:10 (more writes than status checks)

### Storage Estimates
- **Notification Record**: ~2KB (metadata, content, status)
- **Daily Storage**: 100M × 2KB = 200GB/day
- **Annual Storage**: 200GB × 365 = 73TB/year
- **Template Storage**: ~10GB (templates, assets)

### Bandwidth Estimates
- **Ingress**: 10K × 2KB = 20MB/s
- **Egress**: 50MB/s (including status updates, webhooks)
- **Provider API Calls**: 10K/s distributed across channels

---

## 3. SYSTEM ARCHITECTURE

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Client Apps   │    │   Load Balancer  │    │  Notification   │
│                 │───▶│                  │───▶│   API Gateway   │
│ Web/Mobile/API  │    │    (AWS ALB)     │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                                         │
                                                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                    NOTIFICATION SERVICE LAYER                   │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ Notification    │ Template        │ User Preference             │
│ API Service     │ Service         │ Service                     │
│                 │                 │                             │
│ • Validation    │ • Template CRUD │ • Preference Management     │
│ • Deduplication │ • Rendering     │ • Opt-in/out Logic         │
│ • Routing       │ • Personalization│ • Channel Selection        │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      MESSAGE QUEUE LAYER                       │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ High Priority   │ Medium Priority │ Low Priority                │
│ Queue           │ Queue           │ Queue                       │
│                 │                 │                             │
│ • Transactional │ • System Alerts │ • Marketing                 │
│ • OTP/Security  │ • Notifications │ • Newsletters               │
│ • Time-sensitive│ • Reminders     │ • Bulk Messages             │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     WORKER SERVICE LAYER                       │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ Email Workers   │ SMS Workers     │ Push Workers                │
│                 │                 │                             │
│ • Rate Limiting │ • Rate Limiting │ • Rate Limiting             │
│ • Retry Logic   │ • Retry Logic   │ • Retry Logic               │
│ • Status Update │ • Status Update │ • Status Update             │
└─────────────────┴─────────────────┴─────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   EXTERNAL PROVIDER LAYER                      │
├─────────────────┬─────────────────┬─────────────────────────────┤
│ Email Providers │ SMS Providers   │ Push Providers              │
│                 │                 │                             │
│ • SendGrid      │ • Twilio        │ • FCM (Android)             │
│ • AWS SES       │ • AWS SNS       │ • APNS (iOS)                │
│ • Mailgun       │ • MessageBird   │ • Web Push                  │
└─────────────────┴─────────────────┴─────────────────────────────┘
```

---

## 4. DATABASE DESIGN

### Primary Database (PostgreSQL)

```sql
-- Notifications table
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id VARCHAR(255) NOT NULL,
    template_id VARCHAR(255),
    channel VARCHAR(50) NOT NULL, -- email, sms, push, in_app
    priority VARCHAR(20) NOT NULL, -- high, medium, low
    status VARCHAR(50) NOT NULL, -- pending, processing, sent, delivered, failed
    subject VARCHAR(500),
    content TEXT,
    metadata JSONB,
    scheduled_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    retry_count INTEGER DEFAULT 0,
    next_retry_at TIMESTAMP,
    idempotency_key VARCHAR(255) UNIQUE
);

-- Indexes
CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_status ON notifications(status);
CREATE INDEX idx_notifications_scheduled_at ON notifications(scheduled_at);
CREATE INDEX idx_notifications_next_retry_at ON notifications(next_retry_at);
CREATE INDEX idx_notifications_created_at ON notifications(created_at);

-- User preferences table
CREATE TABLE user_preferences (
    user_id VARCHAR(255) PRIMARY KEY,
    email_enabled BOOLEAN DEFAULT true,
    sms_enabled BOOLEAN DEFAULT true,
    push_enabled BOOLEAN DEFAULT true,
    marketing_enabled BOOLEAN DEFAULT false,
    frequency_limit INTEGER DEFAULT 10, -- per hour
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    timezone VARCHAR(50) DEFAULT 'UTC',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Templates table
CREATE TABLE templates (
    id VARCHAR(255) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    channel VARCHAR(50) NOT NULL,
    subject_template TEXT,
    body_template TEXT NOT NULL,
    variables JSONB,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Delivery logs table (for analytics)
CREATE TABLE delivery_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    notification_id UUID REFERENCES notifications(id),
    provider VARCHAR(100),
    provider_message_id VARCHAR(255),
    status VARCHAR(50),
    error_message TEXT,
    delivered_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Cache Layer (Redis)

```redis
# Idempotency tracking
SET idempotency:{key} "processed" EX 3600

# Rate limiting
INCR rate_limit:user:{user_id}:hour:{hour} EX 3600
INCR rate_limit:provider:{provider}:minute:{minute} EX 60

# User preferences cache
HSET user_prefs:{user_id} email_enabled true sms_enabled false
EXPIRE user_prefs:{user_id} 3600

# Template cache
SET template:{template_id} "{json_template}" EX 7200

# Dead letter queue tracking
LPUSH dlq:failed_notifications "{notification_json}"
```

---

## 5. API DESIGN

### Core APIs

```yaml
# Send Notification
POST /api/v1/notifications
Content-Type: application/json
{
  "user_id": "user123",
  "template_id": "welcome_email",
  "channel": "email",
  "priority": "high",
  "variables": {
    "user_name": "John Doe",
    "verification_code": "123456"
  },
  "scheduled_at": "2024-01-15T10:00:00Z",
  "idempotency_key": "unique-key-123"
}

Response: 201 Created
{
  "notification_id": "notif_abc123",
  "status": "pending",
  "estimated_delivery": "2024-01-15T10:00:05Z"
}

# Batch Send
POST /api/v1/notifications/batch
{
  "notifications": [
    {
      "user_id": "user123",
      "template_id": "promo_email",
      "channel": "email",
      "priority": "low",
      "variables": {"discount": "20%"}
    }
  ]
}

# Get Notification Status
GET /api/v1/notifications/{notification_id}
Response: 200 OK
{
  "notification_id": "notif_abc123",
  "status": "delivered",
  "channel": "email",
  "sent_at": "2024-01-15T10:00:03Z",
  "delivered_at": "2024-01-15T10:00:07Z",
  "provider": "sendgrid"
}

# User Preferences
GET /api/v1/users/{user_id}/preferences
PUT /api/v1/users/{user_id}/preferences
{
  "email_enabled": true,
  "sms_enabled": false,
  "marketing_enabled": false,
  "quiet_hours": {
    "start": "22:00",
    "end": "08:00",
    "timezone": "America/New_York"
  }
}

# Template Management
POST /api/v1/templates
GET /api/v1/templates/{template_id}
PUT /api/v1/templates/{template_id}
```

---

## 6. CORE COMPONENTS

### 6.1 Notification API Service

```python
# notification_service.py
class NotificationService:
    def __init__(self, queue_manager, user_service, template_service):
        self.queue_manager = queue_manager
        self.user_service = user_service
        self.template_service = template_service
        self.redis = Redis()
    
    async def send_notification(self, request: NotificationRequest):
        # Idempotency check
        if await self._is_duplicate(request.idempotency_key):
            return await self._get_existing_notification(request.idempotency_key)
        
        # Validate request
        await self._validate_request(request)
        
        # Check user preferences
        user_prefs = await self.user_service.get_preferences(request.user_id)
        if not self._should_send(request, user_prefs):
            return self._create_skipped_response(request)
        
        # Create notification record
        notification = await self._create_notification(request)
        
        # Route to appropriate queue
        queue_name = self._get_queue_name(request.priority)
        await self.queue_manager.enqueue(queue_name, notification)
        
        # Mark as processed for idempotency
        await self._mark_processed(request.idempotency_key, notification.id)
        
        return NotificationResponse(
            notification_id=notification.id,
            status="pending",
            estimated_delivery=self._calculate_eta(request.priority)
        )
    
    def _get_queue_name(self, priority: str) -> str:
        return f"notifications_{priority}_priority"
    
    async def _is_duplicate(self, idempotency_key: str) -> bool:
        return await self.redis.exists(f"idempotency:{idempotency_key}")
```

### 6.2 Worker Service

```python
# worker_service.py
class NotificationWorker:
    def __init__(self, channel: str, provider_manager: ProviderManager):
        self.channel = channel
        self.provider_manager = provider_manager
        self.rate_limiter = RateLimiter()
        self.retry_handler = RetryHandler()
    
    async def process_notification(self, notification: Notification):
        try:
            # Rate limiting check
            if not await self.rate_limiter.can_proceed(
                self.channel, notification.user_id
            ):
                await self._requeue_with_delay(notification, delay=60)
                return
            
            # Get user contact info
            contact_info = await self._get_contact_info(
                notification.user_id, self.channel
            )
            
            # Render template
            content = await self._render_template(notification)
            
            # Send via provider
            result = await self.provider_manager.send(
                channel=self.channel,
                recipient=contact_info,
                content=content,
                notification_id=notification.id
            )
            
            # Update status
            await self._update_status(notification.id, "sent", result)
            
        except RetryableError as e:
            await self.retry_handler.handle_retry(notification, e)
        except PermanentError as e:
            await self._move_to_dlq(notification, e)
        except Exception as e:
            logger.error(f"Unexpected error processing {notification.id}: {e}")
            await self.retry_handler.handle_retry(notification, e)
```

### 6.3 Provider Manager

```python
# provider_manager.py
class ProviderManager:
    def __init__(self):
        self.providers = {
            'email': [SendGridProvider(), SESProvider(), MailgunProvider()],
            'sms': [TwilioProvider(), SNSProvider()],
            'push': [FCMProvider(), APNSProvider()]
        }
        self.circuit_breakers = {}
    
    async def send(self, channel: str, recipient: str, content: dict, notification_id: str):
        providers = self.providers[channel]
        
        for provider in providers:
            circuit_breaker = self._get_circuit_breaker(provider.name)
            
            if circuit_breaker.is_open():
                continue
            
            try:
                result = await provider.send(recipient, content, notification_id)
                circuit_breaker.record_success()
                return result
            except ProviderError as e:
                circuit_breaker.record_failure()
                if provider == providers[-1]:  # Last provider
                    raise
                continue
        
        raise AllProvidersFailedError("All providers failed")
    
    def _get_circuit_breaker(self, provider_name: str):
        if provider_name not in self.circuit_breakers:
            self.circuit_breakers[provider_name] = CircuitBreaker(
                failure_threshold=5,
                timeout=60
            )
        return self.circuit_breakers[provider_name]
```

---

## 7. SCALABILITY & PERFORMANCE

### Horizontal Scaling
- **API Services**: Auto-scaling based on CPU/memory usage
- **Worker Services**: Scale based on queue depth
- **Database**: Read replicas, connection pooling
- **Cache**: Redis cluster with sharding

### Performance Optimizations
- **Batch Processing**: Group notifications by user/template
- **Connection Pooling**: Reuse HTTP connections to providers
- **Async Processing**: Non-blocking I/O operations
- **Template Caching**: Cache rendered templates
- **Database Optimization**: Proper indexing, query optimization

### Queue Management
```python
# Priority queue configuration
QUEUE_CONFIG = {
    'high_priority': {
        'workers': 50,
        'batch_size': 1,
        'max_retry': 5
    },
    'medium_priority': {
        'workers': 30,
        'batch_size': 10,
        'max_retry': 3
    },
    'low_priority': {
        'workers': 20,
        'batch_size': 50,
        'max_retry': 2
    }
}
```

---

## 8. RELIABILITY & FAULT TOLERANCE

### Retry Strategy
```python
class ExponentialBackoffRetry:
    def __init__(self, max_retries=5, base_delay=1):
        self.max_retries = max_retries
        self.base_delay = base_delay
    
    def get_delay(self, attempt: int) -> int:
        if attempt > self.max_retries:
            return None  # Move to DLQ
        return min(self.base_delay * (2 ** attempt), 300)  # Max 5 minutes
```

### Circuit Breaker Pattern
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
```

### Dead Letter Queue
- Failed notifications after max retries
- Manual investigation and reprocessing
- Alerting for high DLQ volume

---

## 9. MONITORING & OBSERVABILITY

### Key Metrics
- **Throughput**: Notifications sent per second
- **Latency**: API response time, delivery time
- **Success Rate**: Delivery success percentage
- **Queue Depth**: Pending notifications count
- **Provider Performance**: Success rate per provider
- **Error Rates**: Failed notifications by error type

### Alerting
- Queue depth > threshold
- High error rates
- Provider failures
- DLQ volume spikes
- API latency > SLA

### Logging
```python
# Structured logging
logger.info("notification_sent", extra={
    "notification_id": notification.id,
    "user_id": notification.user_id,
    "channel": notification.channel,
    "provider": provider.name,
    "latency_ms": latency,
    "status": "success"
})
```

---

## 10. SECURITY CONSIDERATIONS

### Data Protection
- **Encryption**: TLS in transit, AES-256 at rest
- **PII Handling**: Minimal data retention, secure deletion
- **Access Control**: RBAC for admin operations
- **Audit Logging**: All notification activities logged

### Provider Security
- **Credential Management**: AWS Secrets Manager
- **API Key Rotation**: Automated rotation
- **Rate Limiting**: Prevent abuse
- **Input Validation**: Sanitize all inputs

---

## 11. DEPLOYMENT ARCHITECTURE

### Infrastructure
```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-api
spec:
  replicas: 10
  selector:
    matchLabels:
      app: notification-api
  template:
    spec:
      containers:
      - name: notification-api
        image: notification-api:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

### Service Mesh
- **Istio**: Traffic management, security policies
- **Load Balancing**: Round-robin with health checks
- **Circuit Breaking**: Automatic failure handling

---

## 12. COST OPTIMIZATION

### Provider Cost Management
- **Smart Routing**: Use cheapest provider first
- **Volume Discounts**: Negotiate better rates
- **Channel Optimization**: Email cheaper than SMS

### Infrastructure Costs
- **Auto-scaling**: Scale down during low traffic
- **Spot Instances**: Use for batch processing
- **Reserved Capacity**: For predictable workloads

---

## 13. FUTURE ENHANCEMENTS

### Advanced Features
- **AI-Powered Optimization**: Send time optimization
- **A/B Testing**: Template performance testing
- **Rich Media**: Support for images, videos
- **Interactive Notifications**: Action buttons, forms

### Analytics & Insights
- **Delivery Analytics**: Open rates, click rates
- **User Behavior**: Engagement patterns
- **Performance Insights**: Provider comparison
- **Predictive Analytics**: Optimal send times

This HLD provides a comprehensive blueprint for building a scalable notification system that can handle millions of notifications while maintaining reliability, performance, and cost-effectiveness.