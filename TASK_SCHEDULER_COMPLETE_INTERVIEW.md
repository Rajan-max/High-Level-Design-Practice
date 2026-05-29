# Task Scheduler System - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: Core Entities & Data Model (4 min)
Phase 5: API Design (3 min)
Phase 6: High-Level Architecture (8 min)
Phase 7: Data Flow & Job Execution (5 min)
Phase 8: Deep Dive - Timely Execution (5 min)
Phase 9: Deep Dive - Scalability (4 min)
Phase 10: Deep Dive - Fault Tolerance & Retries (3 min)
```

---

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"I'll design a distributed task scheduler that can execute jobs at specified times or intervals. This is similar to a distributed cron system. Let me clarify the scope and requirements."

### Questions to Ask:

**Q1: Job Types & Scheduling**
- "What types of schedules should we support: one-time, recurring (cron), or both?"
- "Do we need to support complex cron expressions like '0 10 * * MON-FRI'?"
- "Should we support immediate execution or only scheduled jobs?"

**Expected Answer:** Support one-time, recurring (cron), and immediate execution

**Q2: Job Definition**
- "How are tasks defined: HTTP callbacks, code execution, message publishing?"
- "Do we need to support different task types (email, API call, database operation)?"
- "Should tasks accept parameters?"

**Expected Answer:** Support HTTP callbacks and message publishing with parameters

**Q3: Execution Guarantees**
- "What delivery guarantees: at-least-once, at-most-once, or exactly-once?"
- "Is it acceptable if a job executes multiple times?"
- "How should we handle job failures?"

**Expected Answer:** At-least-once execution with retry logic

**Q4: Timing Precision**
- "What's the acceptable execution delay: seconds, minutes?"
- "Do we need sub-second precision or is minute-level acceptable?"

**Expected Answer:** Execute within 2 seconds of scheduled time

**Q5: Job Management**
- "Should users be able to cancel, pause, or modify scheduled jobs?"
- "Do we need job history and execution logs?"
- "Should we support job dependencies (job B runs after job A)?"

**Expected Answer:** Cancel/pause needed, job history needed, dependencies out of scope

**Q6: Scale & Performance**
- "What's the expected scale: jobs per second, total active jobs?"
- "What's the expected job duration: seconds, minutes, hours?"
- "Are there traffic patterns (peak hours, batch processing)?"

**Expected Answer:** 10K jobs/second, jobs typically complete in seconds, uniform traffic

**Q7: Multi-tenancy**
- "Is this a multi-tenant system with different users/organizations?"
- "Do we need isolation between tenants?"
- "Should we support rate limiting per tenant?"

**Expected Answer:** Multi-tenant with basic isolation and rate limiting

**Q8: Monitoring & Observability**
- "Do we need real-time monitoring of job execution?"
- "Should we provide metrics like success rate, latency?"
- "Do we need alerting for failed jobs?"

**Expected Answer:** Basic monitoring and metrics needed, alerting is nice-to-have

---

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### What to Say:

"Based on our discussion, let me organize the requirements in priority order to guide our design."

### Functional Requirements (Priority Order):

```
CORE REQUIREMENTS (Above the line):

1. Job Scheduling (CRITICAL)
   - Schedule jobs for immediate execution
   - Schedule jobs for specific future date/time
   - Schedule recurring jobs with cron expressions
   - Support job parameters and metadata
   - Validate cron expressions and schedules

2. Job Execution (CRITICAL)
   - Execute jobs at scheduled time (within 2s precision)
   - Support HTTP callback tasks
   - Support message publishing tasks
   - Pass parameters to task execution
   - Handle long-running jobs (up to 15 minutes)

3. Job Management (CRITICAL)
   - Create new jobs via API
   - Cancel scheduled jobs
   - Pause/resume recurring jobs
   - Query job status and history
   - Update job parameters

4. Retry & Failure Handling (CRITICAL)
   - Retry failed jobs with exponential backoff
   - Maximum retry attempts (3 retries)
   - Dead letter queue for permanent failures
   - Track failure reasons and error logs

5. Job Monitoring (IMPORTANT)
   - Track job execution status (pending, running, completed, failed)
   - View job execution history
   - Query jobs by user, status, time range
   - Basic metrics (success rate, execution time)

BELOW THE LINE (Out of scope):
- Job dependencies and workflows
- Complex job orchestration (DAGs)
- Job priority levels
- Advanced analytics dashboard
- Job result storage and retrieval
- Multi-step job pipelines
```

### Non-Functional Requirements:

```
PERFORMANCE:
- API response time: < 100ms (P95)
- Job execution precision: Within 2 seconds of scheduled time
- System throughput: 10K jobs/second (peak)
- Job processing latency: < 500ms (from queue to execution)

SCALABILITY:
- Support 10K jobs/second sustained throughput
- Handle 100M active scheduled jobs
- Support 1B job executions per day
- Horizontal scaling of all components

AVAILABILITY & RELIABILITY:
- System availability: 99.9% uptime
- At-least-once execution guarantee
- No job loss (durable storage)
- Automatic failover for worker failures
- Graceful degradation under load

CONSISTENCY:
- Strong consistency for job creation/cancellation
- Eventual consistency for job status updates
- Idempotent job execution (same job doesn't execute twice)
- Prevent duplicate job creation

OBSERVABILITY:
- Comprehensive logging for all operations
- Real-time metrics (jobs/sec, success rate, latency)
- Distributed tracing for job lifecycle
- Alerting for system health issues
```

### Why This Matters:
✓ Clarifies that timing precision is critical
✓ Establishes at-least-once semantics upfront
✓ Sets clear performance expectations
✓ Identifies retry logic as core requirement

---

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me calculate the scale to understand infrastructure requirements and identify potential bottlenecks."

### Traffic Estimation:

```
Given:
- 10K jobs/second (peak throughput)
- 100M active scheduled jobs
- Average job execution time: 5 seconds
- Job distribution: 70% one-time, 30% recurring

Calculations:

1. Daily Job Volume
   Average QPS: 10K jobs/second
   Daily executions: 10K × 86,400 = 864M jobs/day
   
   With 30% recurring: ~300M unique jobs, 864M executions

2. Concurrent Job Executions
   Jobs/second: 10K
   Avg execution time: 5 seconds
   Concurrent executions: 10K × 5 = 50K concurrent jobs
   
   This is critical for worker capacity planning!

3. Job Creation Rate
   If 70% are one-time jobs:
   New jobs created: 864M × 0.7 = 605M/day
   Creation QPS: 605M / 86,400 = 7K jobs/second
   
   Recurring jobs: 300M × 0.3 = 90M recurring jobs
   These create 864M × 0.3 = 259M executions

4. Database Write Load
   Job creation: 7K writes/second
   Status updates: 10K × 3 (pending→running→completed) = 30K writes/second
   Total: ~37K writes/second
   
   This is a significant write load!

5. Database Read Load
   Job scheduling queries: Every 5 minutes for 5-minute window
   Jobs in 5-min window: 10K × 300 = 3M jobs
   Query frequency: 1 query per 5 minutes = 0.003 QPS
   
   Worker job fetches: 10K jobs/second
   Status queries by users: ~1K QPS
   Total: ~11K reads/second
```

### Storage Estimation:

```
1. Jobs Table (Job Definitions)
   Total jobs: 100M active jobs
   Job record size: 1KB (task_id, schedule, parameters, metadata)
   Storage: 100M × 1KB = 100GB
   
   With indexes: 100GB × 3 = 300GB

2. Executions Table (Job Instances)
   Daily executions: 864M
   Retention: 30 days
   Total executions: 864M × 30 = 25.9B records
   
   Execution record size: 500 bytes (job_id, time, status, attempt)
   Storage: 25.9B × 500B = 12.95TB
   
   With indexes: 12.95TB × 2 = 25.9TB

3. Execution Logs
   Daily executions: 864M
   Retention: 7 days
   Total logs: 864M × 7 = 6B log entries
   
   Log entry size: 2KB (error messages, stack traces, timing)
   Storage: 6B × 2KB = 12TB

4. Time-Bucket Index (Hot Data)
   Jobs in next 1 hour: 10K × 3,600 = 36M jobs
   Record size: 200 bytes (minimal metadata)
   Storage: 36M × 200B = 7.2GB
   
   This is the "hot" data that needs fast access!

Total Storage: ~38TB (mostly execution history)
```

### Message Queue Estimation:

```
1. Queue Depth (Steady State)
   Jobs scheduled in 5-minute window: 10K × 300 = 3M jobs
   Message size: 200 bytes (job_id, execution_time, metadata)
   Queue size: 3M × 200B = 600MB
   
   This is manageable for any message queue!

2. Queue Throughput
   Enqueue rate: 10K messages/second (new jobs + retries)
   Dequeue rate: 10K messages/second (workers pulling jobs)
   
   With retries (10% failure rate, 3 retries):
   Retry messages: 10K × 0.1 × 3 = 3K messages/second
   Total throughput: 13K messages/second

3. Dead Letter Queue
   Permanent failures: 1% of jobs
   Daily failures: 864M × 0.01 = 8.64M failed jobs/day
   Retention: 7 days
   Total: 8.64M × 7 = 60.5M messages
   Storage: 60.5M × 200B = 12.1GB
```

### Worker Capacity Estimation:

```
1. Worker Requirements
   Concurrent executions: 50K jobs
   Jobs per worker: 10 concurrent jobs (thread pool)
   Workers needed: 50K / 10 = 5,000 workers
   
   With 20% buffer: 6,000 workers

2. Worker Resource Requirements
   CPU: 2 vCPUs per worker
   Memory: 4GB per worker
   Total: 12,000 vCPUs, 24TB RAM
   
   Using c5.xlarge (4 vCPU, 8GB): 3,000 instances

3. Auto-scaling Considerations
   Baseline: 3,000 instances (50% utilization)
   Peak: 6,000 instances (100% utilization)
   Scale-up trigger: Queue depth > 5M messages
   Scale-down trigger: Queue depth < 1M messages
```

### Cost Estimation (AWS):

```
1. Compute (Workers)
   3,000 × c5.xlarge × $0.17/hour = $510/hour
   Monthly: $510 × 730 = $372,300/month
   
   With spot instances (70% discount):
   Monthly: $372,300 × 0.3 = $111,690/month

2. Database (DynamoDB)
   Write capacity: 37K WCU × $0.00065/hour = $24/hour = $17,520/month
   Read capacity: 11K RCU × $0.00013/hour = $1.43/hour = $1,044/month
   Storage: 38TB × $0.25/GB = $9,728/month
   Total: $28,292/month

3. Message Queue (SQS)
   Requests: 13K/sec × 86,400 = 1.12B requests/day
   Cost: 1.12B × $0.0000004 = $448/day = $13,440/month

4. Data Transfer
   Outbound: 10K jobs/sec × 50KB response × 86,400 = 43.2TB/day
   Cost: 43.2TB × $0.09/GB = $3,888/day = $116,640/month

Total Monthly Cost: ~$270,000/month
(Dominated by compute and data transfer)
```

### Key Insights from Estimation:

```
1. Worker capacity is the primary cost driver
2. Database write load (37K WCU) requires sharding strategy
3. Hot data (7.2GB) can fit entirely in memory for fast access
4. Message queue is not a bottleneck (600MB steady state)
5. Need efficient time-bucket indexing for job scheduling queries
```

---

## PHASE 4: CORE ENTITIES & DATA MODEL (4 min)

### What to Say:

"Let me define the core entities and their relationships. I'll start with a high-level overview, then detail the schema as we discuss the architecture."

### Core Entities:

```
1. User
   - Represents a user/tenant who can schedule jobs
   - Attributes: user_id, email, api_key, rate_limit

2. Task
   - Represents a reusable task definition
   - Attributes: task_id, task_type (HTTP/MESSAGE), endpoint, config
   - Example: "send_email" task, "process_payment" task

3. Job
   - Represents a scheduled job (instance of a task)
   - Attributes: job_id, user_id, task_id, schedule, parameters, status
   - This is the job definition that can be recurring

4. Execution
   - Represents a single execution instance of a job
   - Attributes: execution_id, job_id, scheduled_time, status, attempt
   - For recurring jobs, multiple executions are created

5. Schedule
   - Embedded in Job entity
   - Attributes: type (ONE_TIME/CRON), expression, timezone
```

### Data Model Design:

```sql
-- Jobs Table (Job Definitions)
-- DynamoDB Schema
{
  "job_id": "uuid",                    // Partition Key
  "user_id": "string",                 // GSI Partition Key
  "task_id": "string",
  "schedule": {
    "type": "CRON | ONE_TIME | IMMEDIATE",
    "expression": "0 10 * * *",        // CRON expression or ISO timestamp
    "timezone": "America/New_York"
  },
  "parameters": {
    "key": "value"                     // Task-specific parameters
  },
  "status": "ACTIVE | PAUSED | CANCELLED",
  "created_at": 1715548800,
  "updated_at": 1715548800,
  "next_execution_time": 1715548800,   // For recurring jobs
  "metadata": {
    "name": "Daily Report",
    "description": "Send daily sales report"
  }
}

-- Executions Table (Job Instances)
-- DynamoDB Schema with Time-Bucket Partitioning
{
  "time_bucket": "1715547600#shard_3", // Partition Key (hour + shard)
  "execution_time_job_id": "1715548800-uuid", // Sort Key (timestamp-jobId)
  "execution_id": "uuid",
  "job_id": "uuid",
  "user_id": "string",
  "scheduled_time": 1715548800,
  "actual_start_time": 1715548802,
  "actual_end_time": 1715548807,
  "status": "PENDING | RUNNING | COMPLETED | FAILED | RETRYING",
  "attempt": 0,
  "error_message": "string",
  "result": {
    "status_code": 200,
    "response": "success"
  }
}

-- GSI on Executions for User Queries
{
  "user_id": "string",                 // GSI Partition Key
  "execution_time": 1715548800,        // GSI Sort Key
  "execution_id": "uuid",
  "job_id": "uuid",
  "status": "string"
}

-- Tasks Table (Task Definitions)
{
  "task_id": "string",                 // Partition Key
  "task_type": "HTTP_CALLBACK | MESSAGE_PUBLISH",
  "config": {
    "http": {
      "url": "https://api.example.com/webhook",
      "method": "POST",
      "headers": {},
      "timeout_seconds": 30
    },
    "message": {
      "topic": "job-results",
      "queue": "job-queue"
    }
  },
  "retry_config": {
    "max_attempts": 3,
    "backoff_multiplier": 2,
    "initial_delay_seconds": 5
  },
  "timeout_seconds": 300,
  "created_at": 1715548800
}

-- Users Table
{
  "user_id": "string",                 // Partition Key
  "email": "string",
  "api_key": "string",
  "rate_limit": {
    "jobs_per_minute": 100,
    "concurrent_jobs": 1000
  },
  "created_at": 1715548800,
  "status": "ACTIVE | SUSPENDED"
}
```

### Key Design Decisions:

```
1. Separation of Jobs and Executions
   - Jobs: Persistent definitions (100M records)
   - Executions: Transient instances (25B records)
   - Enables efficient recurring job handling

2. Time-Bucket Partitioning
   - Partition key: time_bucket (hour) + shard_id
   - Enables efficient "jobs due in next N minutes" queries
   - Prevents hot partitions with sharding

3. Composite Sort Key
   - execution_time + job_id ensures uniqueness
   - Enables range queries by time
   - Prevents duplicate executions

4. GSI for User Queries
   - Partition by user_id for tenant isolation
   - Sort by execution_time for chronological listing
   - Supports filtering by status

5. Embedded Schedule Configuration
   - Avoids additional table join
   - Schedule is tightly coupled with job
   - Simplifies queries and updates
```

---

## PHASE 5: API DESIGN (3 minutes)

### What to Say:

"Let me design the REST APIs that satisfy our functional requirements. I'll organize them by resource."

### API Endpoints:

```
1. CREATE JOB
POST /v1/jobs
Headers:
  Authorization: Bearer <api_key>
  Content-Type: application/json

Request:
{
  "task_id": "send_email",
  "schedule": {
    "type": "CRON",
    "expression": "0 10 * * *",
    "timezone": "America/New_York"
  },
  "parameters": {
    "to": "user@example.com",
    "subject": "Daily Report"
  },
  "metadata": {
    "name": "Daily Email Report",
    "description": "Send sales report every day at 10 AM"
  }
}

Response: 201 Created
{
  "job_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "ACTIVE",
  "next_execution_time": "2024-05-13T10:00:00Z",
  "created_at": "2024-05-12T15:30:00Z"
}

---

2. GET JOB STATUS
GET /v1/jobs/{job_id}
Headers:
  Authorization: Bearer <api_key>

Response: 200 OK
{
  "job_id": "123e4567-e89b-12d3-a456-426614174000",
  "user_id": "user_123",
  "task_id": "send_email",
  "schedule": {
    "type": "CRON",
    "expression": "0 10 * * *",
    "timezone": "America/New_York"
  },
  "status": "ACTIVE",
  "next_execution_time": "2024-05-13T10:00:00Z",
  "last_execution": {
    "execution_id": "exec_456",
    "scheduled_time": "2024-05-12T10:00:00Z",
    "status": "COMPLETED",
    "duration_ms": 1250
  },
  "created_at": "2024-05-12T15:30:00Z"
}

---

3. LIST JOBS
GET /v1/jobs?status={status}&limit={limit}&cursor={cursor}
Headers:
  Authorization: Bearer <api_key>

Query Parameters:
  - status: ACTIVE | PAUSED | CANCELLED (optional)
  - limit: 1-100 (default: 20)
  - cursor: pagination token (optional)

Response: 200 OK
{
  "jobs": [
    {
      "job_id": "uuid",
      "task_id": "send_email",
      "status": "ACTIVE",
      "next_execution_time": "2024-05-13T10:00:00Z",
      "created_at": "2024-05-12T15:30:00Z"
    }
  ],
  "next_cursor": "eyJsYXN0X2lkIjoidXVpZCJ9",
  "has_more": true
}

---

4. CANCEL JOB
DELETE /v1/jobs/{job_id}
Headers:
  Authorization: Bearer <api_key>

Response: 200 OK
{
  "job_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "CANCELLED",
  "cancelled_at": "2024-05-12T16:00:00Z"
}

---

5. PAUSE/RESUME JOB
PATCH /v1/jobs/{job_id}
Headers:
  Authorization: Bearer <api_key>
  Content-Type: application/json

Request:
{
  "status": "PAUSED"  // or "ACTIVE"
}

Response: 200 OK
{
  "job_id": "123e4567-e89b-12d3-a456-426614174000",
  "status": "PAUSED",
  "updated_at": "2024-05-12T16:00:00Z"
}

---

6. GET JOB EXECUTIONS
GET /v1/jobs/{job_id}/executions?limit={limit}&cursor={cursor}
Headers:
  Authorization: Bearer <api_key>

Query Parameters:
  - status: PENDING | RUNNING | COMPLETED | FAILED (optional)
  - start_time: ISO timestamp (optional)
  - end_time: ISO timestamp (optional)
  - limit: 1-100 (default: 20)
  - cursor: pagination token (optional)

Response: 200 OK
{
  "executions": [
    {
      "execution_id": "exec_456",
      "job_id": "job_123",
      "scheduled_time": "2024-05-12T10:00:00Z",
      "actual_start_time": "2024-05-12T10:00:01Z",
      "actual_end_time": "2024-05-12T10:00:02Z",
      "status": "COMPLETED",
      "attempt": 1,
      "duration_ms": 1250
    }
  ],
  "next_cursor": "eyJsYXN0X2lkIjoidXVpZCJ9",
  "has_more": true
}

---

7. GET EXECUTION DETAILS
GET /v1/executions/{execution_id}
Headers:
  Authorization: Bearer <api_key>

Response: 200 OK
{
  "execution_id": "exec_456",
  "job_id": "job_123",
  "scheduled_time": "2024-05-12T10:00:00Z",
  "actual_start_time": "2024-05-12T10:00:01Z",
  "actual_end_time": "2024-05-12T10:00:02Z",
  "status": "COMPLETED",
  "attempt": 1,
  "duration_ms": 1250,
  "result": {
    "status_code": 200,
    "response_body": "success"
  },
  "logs": [
    {
      "timestamp": "2024-05-12T10:00:01Z",
      "level": "INFO",
      "message": "Starting job execution"
    }
  ]
}

---

8. RETRY FAILED EXECUTION
POST /v1/executions/{execution_id}/retry
Headers:
  Authorization: Bearer <api_key>

Response: 200 OK
{
  "execution_id": "exec_789",
  "job_id": "job_123",
  "status": "PENDING",
  "scheduled_time": "2024-05-12T16:05:00Z"
}
```

### API Design Principles:

```
1. RESTful Design
   - Resource-based URLs (/jobs, /executions)
   - Standard HTTP methods (GET, POST, PATCH, DELETE)
   - Proper status codes (201, 200, 404, 400)

2. Authentication & Authorization
   - API key in Authorization header
   - User-scoped access (can only see own jobs)
   - Rate limiting per user

3. Pagination
   - Cursor-based pagination for large result sets
   - Configurable page size (limit parameter)
   - has_more flag for client convenience

4. Filtering & Querying
   - Status filtering for jobs and executions
   - Time range filtering for executions
   - Consistent query parameter naming

5. Idempotency
   - POST /jobs with idempotency key (optional)
   - Prevents duplicate job creation
   - Safe retry of API calls
```



---

## PHASE 6: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"I'll design a distributed architecture that separates job scheduling from job execution. The key insight is using a two-layered approach: durable storage for persistence and a message queue for timely execution."

### Complete System Architecture:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
├─────────────────────────────────────────────────────────────────┤
│   Web Dashboard   │   Mobile App   │   API Clients   │   CLI    │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY LAYER                            │
├─────────────────────────────────────────────────────────────────┤
│  Authentication  │  Rate Limiting  │  Request Routing           │
│  API Key Validation │ Load Balancing │ Circuit Breaker         │
└─────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     SERVICE LAYER                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  Job Service    │  │ Execution       │  │  Query Service  │ │
│  │                 │  │ Service         │  │                 │ │
│  │ - Create Job    │  │ - Track Status  │  │ - List Jobs     │ │
│  │ - Update Job    │  │ - Update Logs   │  │ - Get Status    │ │
│  │ - Cancel Job    │  │ - Handle Retry  │  │ - Get History   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                    │               │               │
                    ▼               ▼               ▼
┌──────────────────────────────────────────────────────────────────┐
│                     DATA LAYER                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              DynamoDB / Cassandra                           ││
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     ││
│  │  │ Jobs Table   │  │ Executions   │  │ Tasks Table  │     ││
│  │  │              │  │ Table        │  │              │     ││
│  │  │ - Job Defs   │  │ - Instances  │  │ - Task Defs  │     ││
│  │  │ - Schedules  │  │ - Status     │  │ - Configs    │     ││
│  │  └──────────────┘  └──────────────┘  └──────────────┘     ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Redis Cache                                    ││
│  │  - Active Jobs Index (next 1 hour)                         ││
│  │  - User Rate Limit Counters                                ││
│  │  - Distributed Locks                                       ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                  SCHEDULING LAYER                                │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │         Scheduler Service (Cron - Every 5 minutes)          ││
│  │                                                             ││
│  │  1. Query Executions table for jobs due in next 5 min      ││
│  │  2. For each job, calculate delay until execution time     ││
│  │  3. Publish to SQS with DelaySeconds parameter             ││
│  │  4. Update job status to SCHEDULED                         ││
│  │                                                             ││
│  │  Handles:                                                   ││
│  │  - Time bucket queries (efficient range scans)             ││
│  │  - Recurring job next execution calculation                ││
│  │  - Distributed locking (prevent duplicate scheduling)      ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                   MESSAGE QUEUE LAYER                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Amazon SQS (Standard Queue)                    ││
│  │                                                             ││
│  │  Main Queue:                                                ││
│  │  - Jobs with DelaySeconds (up to 15 min)                   ││
│  │  - Messages become visible at execution time               ││
│  │  - Visibility timeout: 30 seconds                          ││
│  │  - Message retention: 14 days                              ││
│  │                                                             ││
│  │  Dead Letter Queue (DLQ):                                  ││
│  │  - Failed jobs after max retries                           ││
│  │  - Manual inspection and reprocessing                      ││
│  │  - Retention: 14 days                                      ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                    EXECUTION LAYER                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │         Worker Pool (ECS / Kubernetes)                      ││
│  │                                                             ││
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ││
│  │  │ Worker 1 │  │ Worker 2 │  │ Worker 3 │  │ Worker N │  ││
│  │  │          │  │          │  │          │  │          │  ││
│  │  │ Thread   │  │ Thread   │  │ Thread   │  │ Thread   │  ││
│  │  │ Pool(10) │  │ Pool(10) │  │ Pool(10) │  │ Pool(10) │  ││
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  ││
│  │                                                             ││
│  │  Each Worker:                                               ││
│  │  1. Poll SQS for messages (long polling)                   ││
│  │  2. Fetch job details from database                        ││
│  │  3. Execute task (HTTP callback / message publish)         ││
│  │  4. Update execution status                                ││
│  │  5. Delete message from SQS (on success)                   ││
│  │  6. Re-queue with backoff (on failure)                     ││
│  │                                                             ││
│  │  Auto-scaling:                                              ││
│  │  - Scale up: Queue depth > 5M messages                     ││
│  │  - Scale down: Queue depth < 1M messages                   ││
│  │  - Min: 1000 workers, Max: 6000 workers                    ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────┐
│                  MONITORING & OBSERVABILITY                      │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ CloudWatch   │  │ Datadog      │  │ Prometheus   │          │
│  │              │  │              │  │              │          │
│  │ - Metrics    │  │ - APM        │  │ - Metrics    │          │
│  │ - Logs       │  │ - Tracing    │  │ - Alerting   │          │
│  │ - Alarms     │  │ - Dashboards │  │ - Grafana    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities:

```
1. API Gateway Layer
   - Authentication: Validate API keys, JWT tokens
   - Rate Limiting: Per-user limits (100 req/min)
   - Load Balancing: Distribute across service instances
   - Circuit Breaker: Fail fast on downstream issues

2. Job Service
   - Create/Update/Cancel jobs
   - Validate cron expressions
   - Calculate next execution time for recurring jobs
   - Write to Jobs table
   - For immediate jobs (<5 min), publish directly to SQS

3. Execution Service
   - Track execution status updates
   - Store execution logs
   - Handle retry logic
   - Update Executions table
   - Publish metrics

4. Query Service
   - List jobs for user
   - Get job/execution details
   - Query execution history
   - Read from DynamoDB with caching

5. Scheduler Service
   - Runs every 5 minutes (distributed cron)
   - Query time-bucket index for upcoming jobs
   - Calculate SQS delay for each job
   - Publish to SQS with DelaySeconds
   - Use distributed lock to prevent duplicate scheduling

6. Worker Pool
   - Poll SQS continuously (long polling)
   - Execute tasks (HTTP/message)
   - Update execution status
   - Handle retries with exponential backoff
   - Delete message on success
```

### Data Flow Summary:

```
Job Creation Flow:
User → API Gateway → Job Service → DynamoDB (Jobs table)
                                 → SQS (if immediate execution)

Scheduled Execution Flow:
Scheduler (every 5 min) → Query DynamoDB (time bucket)
                        → Publish to SQS (with delay)

Job Execution Flow:
Worker → Poll SQS → Fetch Job Details (DynamoDB)
                  → Execute Task (HTTP/Message)
                  → Update Status (DynamoDB)
                  → Delete Message (SQS)

Failure & Retry Flow:
Worker → Task Fails → Update Status (RETRYING)
                    → Re-publish to SQS (with backoff delay)
                    → Increment attempt counter
                    → If max retries → Move to DLQ
```

### Key Design Decisions:

```
1. Two-Layered Scheduling
   - Database: Durable storage, 5-minute granularity
   - SQS: Timely execution, 2-second precision
   - Decouples persistence from execution

2. Time-Bucket Partitioning
   - Partition key: hour + shard_id
   - Enables efficient range queries
   - Prevents hot partitions

3. SQS DelaySeconds
   - Native support for delayed delivery
   - No custom priority queue needed
   - Automatic visibility management

4. Separation of Concerns
   - Job Service: Job lifecycle management
   - Scheduler Service: Time-based scheduling
   - Worker Pool: Task execution
   - Each can scale independently

5. Idempotency
   - Execution ID prevents duplicate processing
   - SQS message deduplication
   - Database conditional writes
```



---

## PHASE 7: DATA FLOW & JOB EXECUTION (5 minutes)

### What to Say:

"Let me walk through the complete data flow from job creation to execution, highlighting the critical paths and decision points."

### Flow 1: Create One-Time Job (Immediate Execution)

```
┌──────────┐
│  Client  │
└────┬─────┘
     │ POST /v1/jobs
     │ {schedule: {type: "IMMEDIATE"}}
     ▼
┌─────────────────┐
│  API Gateway    │
│                 │
│ 1. Validate API │
│    key          │
│ 2. Rate limit   │
│    check        │
└────┬────────────┘
     │
     ▼
┌─────────────────┐
│  Job Service    │
│                 │
│ 1. Generate     │
│    job_id       │
│ 2. Validate     │
│    parameters   │
│ 3. Write to     │
│    Jobs table   │
└────┬────────────┘
     │
     ├─────────────────────────────┐
     │                             │
     ▼                             ▼
┌─────────────────┐      ┌─────────────────┐
│  DynamoDB       │      │  Amazon SQS     │
│  Jobs Table     │      │                 │
│                 │      │ Publish message │
│ {               │      │ with            │
│   job_id,       │      │ DelaySeconds=0  │
│   status:       │      │                 │
│   "ACTIVE"      │      │ Message:        │
│ }               │      │ {               │
└─────────────────┘      │   execution_id, │
                         │   job_id,       │
                         │   scheduled_time│
                         │ }               │
                         └────┬────────────┘
                              │
                              │ Worker polls
                              ▼
                         ┌─────────────────┐
                         │  Worker Node    │
                         │                 │
                         │ 1. Receive msg  │
                         │ 2. Fetch job    │
                         │    details      │
                         │ 3. Execute task │
                         │ 4. Update status│
                         │ 5. Delete msg   │
                         └────┬────────────┘
                              │
                              ▼
                         ┌─────────────────┐
                         │  DynamoDB       │
                         │  Executions     │
                         │                 │
                         │ {               │
                         │   execution_id, │
                         │   status:       │
                         │   "COMPLETED",  │
                         │   result: {...} │
                         │ }               │
                         └─────────────────┘

Timeline:
T+0ms:    Client sends request
T+50ms:   Job created in database
T+100ms:  Message published to SQS
T+200ms:  Worker receives message
T+500ms:  Task execution starts
T+1500ms: Task completes
T+1600ms: Status updated, message deleted

Total latency: ~1.6 seconds
```

### Flow 2: Create Recurring Job (CRON Schedule)

```
┌──────────┐
│  Client  │
└────┬─────┘
     │ POST /v1/jobs
     │ {schedule: {type: "CRON", expression: "0 10 * * *"}}
     ▼
┌─────────────────┐
│  Job Service    │
│                 │
│ 1. Parse CRON   │
│    expression   │
│ 2. Calculate    │
│    next exec    │
│    time         │
│ 3. Create job   │
│    record       │
└────┬────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│  DynamoDB - Jobs Table              │
│                                     │
│ {                                   │
│   job_id: "job_123",                │
│   schedule: {                       │
│     type: "CRON",                   │
│     expression: "0 10 * * *"        │
│   },                                │
│   status: "ACTIVE",                 │
│   next_execution_time: 1715594400   │ ← Tomorrow 10 AM
│ }                                   │
└─────────────────────────────────────┘
     │
     │ Scheduler service (runs every 5 min)
     ▼
┌─────────────────────────────────────┐
│  Scheduler Service                  │
│                                     │
│ Query: Get jobs where               │
│   next_execution_time BETWEEN       │
│   now AND now + 5 minutes           │
│                                     │
│ For each job:                       │
│ 1. Create execution record          │
│ 2. Calculate SQS delay              │
│ 3. Publish to SQS                   │
│ 4. Update next_execution_time       │
└────┬────────────────────────────────┘
     │
     ├─────────────────────────────────┐
     │                                 │
     ▼                                 ▼
┌─────────────────┐         ┌─────────────────┐
│  DynamoDB       │         │  Amazon SQS     │
│  Executions     │         │                 │
│                 │         │ Message with    │
│ {               │         │ DelaySeconds    │
│   execution_id, │         │ = 290 (4m 50s)  │
│   job_id,       │         │                 │
│   time_bucket,  │         │ Becomes visible │
│   status:       │         │ at 10:00:00 AM  │
│   "PENDING"     │         │                 │
│ }               │         └────┬────────────┘
└─────────────────┘              │
                                 │ At 10:00 AM
                                 ▼
                         ┌─────────────────┐
                         │  Worker Node    │
                         │                 │
                         │ Execute task    │
                         └────┬────────────┘
                              │
                              ▼
                         ┌─────────────────┐
                         │  Update Status  │
                         │  to COMPLETED   │
                         └─────────────────┘

Next Iteration:
Scheduler calculates next execution (tomorrow 10 AM)
Updates job.next_execution_time
Process repeats daily
```

### Flow 3: Job Execution with Retry

```
┌─────────────────┐
│  Worker Node    │
└────┬────────────┘
     │ 1. Poll SQS
     ▼
┌─────────────────┐
│  Amazon SQS     │
│                 │
│ Receive message │
│ Set visibility  │
│ timeout: 30s    │
└────┬────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│  Worker: Execute Task               │
│                                     │
│ 1. Fetch job details from DB        │
│ 2. Update status: RUNNING           │
│ 3. Execute HTTP callback            │
│                                     │
│    try {                            │
│      response = http.post(url)      │
│      if (response.status == 200) {  │
│        // Success path              │
│      } else {                       │
│        // Failure path              │
│      }                              │
│    } catch (Exception e) {          │
│      // Error path                  │
│    }                                │
└────┬────────────────────────────────┘
     │
     ├─────────────────┬───────────────┐
     │                 │               │
     │ SUCCESS         │ FAILURE       │ TIMEOUT
     ▼                 ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Update      │  │ Update      │  │ Visibility  │
│ Status:     │  │ Status:     │  │ timeout     │
│ COMPLETED   │  │ RETRYING    │  │ expires     │
│             │  │             │  │             │
│ Delete SQS  │  │ Increment   │  │ Message     │
│ message     │  │ attempt     │  │ returns to  │
│             │  │ counter     │  │ queue       │
│ Store       │  │             │  │             │
│ result      │  │ Calculate   │  │ Another     │
└─────────────┘  │ backoff     │  │ worker will │
                 │             │  │ retry       │
                 │ Re-publish  │  └─────────────┘
                 │ to SQS with │
                 │ delay       │
                 │             │
                 │ delay =     │
                 │ 5 * (2^att) │
                 │             │
                 │ Attempt 1:  │
                 │ 5s delay    │
                 │ Attempt 2:  │
                 │ 10s delay   │
                 │ Attempt 3:  │
                 │ 20s delay   │
                 └─────┬───────┘
                       │
                       ▼
                 ┌─────────────┐
                 │ If attempt  │
                 │ > max (3)   │
                 │             │
                 │ Move to DLQ │
                 │ Status:     │
                 │ FAILED      │
                 └─────────────┘

Retry Timeline:
T+0s:     First attempt fails
T+5s:     Retry attempt 1 (5s backoff)
T+15s:    Retry attempt 2 (10s backoff)
T+35s:    Retry attempt 3 (20s backoff)
T+35s:    Max retries reached → DLQ
```

### Flow 4: Scheduler Service (Every 5 Minutes)

```
┌─────────────────────────────────────┐
│  Scheduler Service (Cron Job)      │
│  Runs: Every 5 minutes              │
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│  Step 1: Acquire Distributed Lock  │
│                                     │
│  Redis SETNX:                       │
│  SET scheduler:lock:1715594400      │
│      "instance_id"                  │
│      NX EX 300                      │
│                                     │
│  Only one instance proceeds         │
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│  Step 2: Calculate Time Window     │
│                                     │
│  current_time = now()               │
│  window_start = current_time        │
│  window_end = current_time + 5min   │
│                                     │
│  time_buckets = [                   │
│    current_hour,                    │
│    next_hour                        │
│  ]                                  │
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│  Step 3: Query Executions Table    │
│                                     │
│  For each time_bucket:              │
│    For each shard (0-9):            │
│      Query DynamoDB:                │
│        PK = time_bucket#shard_X     │
│        SK BETWEEN window_start      │
│                   AND window_end    │
│        Filter: status = PENDING     │
│                                     │
│  Parallel queries across shards     │
│  Aggregate results                  │
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│  Step 4: Process Each Job          │
│                                     │
│  For each execution:                │
│    1. Calculate delay:              │
│       delay = execution_time - now  │
│                                     │
│    2. Validate delay:               │
│       if delay < 0:                 │
│         delay = 0  (overdue)        │
│       if delay > 300:               │
│         skip (too far in future)    │
│                                     │
│    3. Publish to SQS:               │
│       SendMessage(                  │
│         MessageBody: {              │
│           execution_id,             │
│           job_id,                   │
│           scheduled_time            │
│         },                          │
│         DelaySeconds: delay         │
│       )                             │
│                                     │
│    4. Update status:                │
│       execution.status = SCHEDULED  │
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│  Step 5: Handle Recurring Jobs     │
│                                     │
│  For each completed recurring job:  │
│    1. Fetch job definition          │
│    2. Calculate next execution:     │
│       next_time = cron.next(        │
│         current_time                │
│       )                             │
│    3. Create new execution record:  │
│       INSERT INTO executions        │
│       VALUES (                      │
│         time_bucket,                │
│         execution_time,             │
│         job_id,                     │
│         status: PENDING             │
│       )                             │
│    4. Update job:                   │
│       job.next_execution_time =     │
│         next_time                   │
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│  Step 6: Release Lock & Metrics    │
│                                     │
│  Redis DEL scheduler:lock:...       │
│                                     │
│  Publish metrics:                   │
│  - Jobs scheduled: 3,000,000        │
│  - Query time: 2.5s                 │
│  - Publish time: 5.2s               │
│  - Total time: 8.1s                 │
└─────────────────────────────────────┘

Performance Considerations:
- Query time: ~2-3 seconds (parallel shard queries)
- Publish time: ~5-7 seconds (batch SQS sends)
- Total time: ~8-10 seconds (well within 5-min window)
- Lock prevents duplicate scheduling
- Idempotent: Safe to run multiple times
```

### Critical Path Analysis:

```
1. Job Creation → Execution (Immediate)
   API Gateway:        50ms
   Job Service:        50ms
   Database Write:     20ms
   SQS Publish:        30ms
   Worker Poll:        100ms (avg)
   Task Execution:     1000ms
   Status Update:      20ms
   ─────────────────────────
   Total:              1270ms ✓ (< 2s requirement)

2. Scheduled Job → Execution
   Scheduler Query:    2000ms
   SQS Publish:        30ms
   SQS Delay:          Variable (0-300s)
   Worker Poll:        100ms
   Task Execution:     1000ms
   ─────────────────────────
   Precision:          ±200ms ✓ (< 2s requirement)

3. Failed Job → Retry
   Failure Detection:  Immediate
   Status Update:      20ms
   Backoff Delay:      5-20s (exponential)
   Re-execution:       1000ms
   ─────────────────────────
   Total:              6-21s (acceptable)
```



---

## PHASE 8: DEEP DIVE - TIMELY EXECUTION (5 minutes)

### What to Say:

"Let me address the critical challenge of executing jobs within 2 seconds of their scheduled time. This requires a two-layered architecture that balances durability with precision."

### Problem Statement:

```
Challenge: Execute 10K jobs/second within 2 seconds of scheduled time

Naive Approach (Doesn't Work):
┌─────────────────────────────────────┐
│  Cron Job (Every 2 seconds)        │
│                                     │
│  Query: SELECT * FROM executions    │
│         WHERE scheduled_time        │
│         BETWEEN now AND now+2s      │
│         AND status = PENDING        │
│                                     │
│  Result: 20,000 jobs                │
└─────────────────────────────────────┘

Problems:
1. Query Overhead: 500ms to fetch 20K jobs
2. Network Transfer: 20K × 200 bytes = 4MB payload
3. Processing Time: 200ms to distribute to workers
4. Total: 700ms overhead → Only 1.3s left for execution
5. Database Load: 0.5 queries/second × 20K rows = 10K reads/sec
6. Precision Loss: Jobs could be delayed by up to 2 seconds
```

### Solution: Two-Layered Scheduler Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 1: DURABLE STORAGE                     │
│                    (5-minute granularity)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Scheduler Service (Runs every 5 minutes)                │  │
│  │                                                          │  │
│  │  1. Query DynamoDB for jobs in next 5-minute window     │  │
│  │     - Query time: ~2 seconds                            │  │
│  │     - Jobs fetched: 3M jobs (10K/sec × 300 sec)         │  │
│  │                                                          │  │
│  │  2. For each job, calculate precise delay:              │  │
│  │     delay_seconds = execution_time - current_time       │  │
│  │                                                          │  │
│  │  3. Publish to SQS with DelaySeconds parameter          │  │
│  │     - Batch publish: 10 messages per API call           │  │
│  │     - Total API calls: 300K                             │  │
│  │     - Publish time: ~5 seconds                          │  │
│  │                                                          │  │
│  │  Total time: ~7 seconds (well within 5-minute window)   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Benefits:                                                      │
│  ✓ Durable storage (jobs survive system crashes)               │
│  ✓ Low database load (1 query per 5 minutes)                   │
│  ✓ Efficient batch processing                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 2: TIMELY DELIVERY                     │
│                    (2-second precision)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Amazon SQS with DelaySeconds                            │  │
│  │                                                          │  │
│  │  Message published at T=0 with DelaySeconds=290:        │  │
│  │                                                          │  │
│  │  T=0s:    Message published (invisible)                 │  │
│  │  T=290s:  Message becomes visible                       │  │
│  │  T=290s:  Worker polls and receives message             │  │
│  │  T=290s:  Worker executes task immediately              │  │
│  │                                                          │  │
│  │  Precision: ±100ms (SQS delivery jitter)                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Worker Pool (Continuous Polling)                        │  │
│  │                                                          │  │
│  │  - Long polling: 20-second wait time                    │  │
│  │  - Batch receive: 10 messages per poll                  │  │
│  │  - Parallel execution: 10 threads per worker            │  │
│  │  - No scheduling overhead: Execute immediately          │  │
│  │                                                          │  │
│  │  Execution latency: <100ms from message receipt         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Benefits:                                                      │
│  ✓ High precision (±100ms)                                      │
│  ✓ No polling overhead (long polling)                          │
│  ✓ Automatic load balancing (SQS distributes messages)         │
│  ✓ Horizontal scalability (add more workers)                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Handling Edge Cases:

```
CASE 1: Job Created with Execution Time < 5 Minutes

Problem: Scheduler runs every 5 minutes, might miss the job

Solution: Direct SQS Publishing
┌──────────────────────────────────────┐
│  Job Service                         │
│                                      │
│  if (execution_time - now < 5min) { │
│    // Bypass scheduler               │
│    delay = execution_time - now     │
│    sqs.sendMessage(                 │
│      message,                       │
│      DelaySeconds: delay            │
│    )                                │
│  } else {                           │
│    // Let scheduler handle it       │
│    // Just write to database        │
│  }                                  │
└──────────────────────────────────────┘

Example:
- Current time: 10:00:00
- Job scheduled for: 10:03:00 (3 minutes away)
- Action: Publish directly to SQS with DelaySeconds=180
- Result: Job executes at 10:03:00 ✓

CASE 2: Overdue Jobs (Scheduled Time in Past)

Problem: Job scheduled for 10:00:00, but it's now 10:05:00

Solution: Immediate Execution
┌──────────────────────────────────────┐
│  Scheduler Service                   │
│                                      │
│  for each job in window:             │
│    delay = execution_time - now     │
│    if (delay < 0) {                 │
│      delay = 0  // Execute now      │
│    }                                │
│    sqs.sendMessage(                 │
│      message,                       │
│      DelaySeconds: delay            │
│    )                                │
└──────────────────────────────────────┘

Tracking:
- Record actual_execution_time vs scheduled_time
- Metric: execution_delay = actual - scheduled
- Alert if execution_delay > 10 seconds

CASE 3: Scheduler Service Failure

Problem: Scheduler crashes, jobs not published to SQS

Solution: Multiple Scheduler Instances + Distributed Lock
┌──────────────────────────────────────┐
│  Scheduler Instance 1                │
│                                      │
│  1. Try acquire lock:                │
│     SETNX scheduler:lock:window_id   │
│           instance_1                 │
│           NX EX 300                  │
│                                      │
│  2. If acquired:                     │
│     - Process jobs                   │
│     - Publish to SQS                 │
│     - Release lock                   │
│                                      │
│  3. If not acquired:                 │
│     - Another instance is running    │
│     - Wait for next cycle            │
└──────────────────────────────────────┘

Failover:
- Lock TTL: 5 minutes (same as scheduler cycle)
- If instance crashes, lock expires automatically
- Next instance acquires lock and processes jobs
- Idempotent: Safe to process same window twice

CASE 4: SQS Message Limit (DelaySeconds max = 15 minutes)

Problem: Job scheduled for 1 hour from now

Solution: Staged Scheduling
┌──────────────────────────────────────┐
│  Scheduler Service                   │
│                                      │
│  if (delay > 900) {  // > 15 min    │
│    // Don't publish yet              │
│    // Will be picked up in next      │
│    // scheduler cycle                │
│  } else {                            │
│    // Publish to SQS                 │
│    sqs.sendMessage(                 │
│      message,                       │
│      DelaySeconds: delay            │
│    )                                │
│  }                                  │
└──────────────────────────────────────┘

Timeline:
- Job scheduled for: 11:00:00
- Current time: 10:00:00 (60 min away)
- Scheduler at 10:00: Skip (delay > 15 min)
- Scheduler at 10:05: Skip (delay > 15 min)
- ...
- Scheduler at 10:50: Publish (delay = 10 min) ✓
```

### Performance Analysis:

```
Latency Breakdown (Immediate Job):

Component                    Latency      Cumulative
─────────────────────────────────────────────────────
API Gateway                  20ms         20ms
Job Service (validation)     30ms         50ms
DynamoDB Write               15ms         65ms
SQS Publish                  25ms         90ms
Worker Poll (avg)            50ms         140ms
Worker Processing            10ms         150ms
Task Execution               1000ms       1150ms
Status Update                15ms         1165ms
─────────────────────────────────────────────────────
Total                                     1165ms ✓

Precision: ±100ms (well within 2-second requirement)

Latency Breakdown (Scheduled Job):

Component                    Latency      Cumulative
─────────────────────────────────────────────────────
Scheduler Query              2000ms       2000ms
Scheduler Processing         100ms        2100ms
SQS Batch Publish            5000ms       7100ms
SQS Delay                    Variable     Variable
Message Visible              0ms          0ms
Worker Poll                  50ms         50ms
Worker Processing            10ms         60ms
Task Execution               1000ms       1060ms
Status Update                15ms         1075ms
─────────────────────────────────────────────────────
Total (from scheduled time)              1075ms ✓

Precision: ±200ms (well within 2-second requirement)
```

### Alternative Approaches (Discussed but Not Chosen):

```
APPROACH 1: Redis Sorted Set (Priority Queue)

Implementation:
ZADD job_queue execution_time job_data

Worker Loop:
while True:
  now = time.now()
  jobs = ZRANGEBYSCORE job_queue 0 now LIMIT 100
  for job in jobs:
    execute(job)
    ZREM job_queue job

Pros:
✓ Sub-millisecond latency
✓ Native priority queue support
✓ Atomic operations

Cons:
✗ Not durable (need Redis persistence)
✗ Complex failover (Redis Sentinel/Cluster)
✗ Manual retry logic required
✗ No built-in visibility timeout
✗ Operational overhead

Verdict: Good for small scale, complex for 10K jobs/sec

APPROACH 2: RabbitMQ with TTL + Dead Letter Exchange

Implementation:
1. Publish to delay queue with per-message TTL
2. Message expires after TTL
3. Routes to processing queue via DLX
4. Worker consumes from processing queue

Pros:
✓ Native delayed message support
✓ Durable by default
✓ Publisher confirms

Cons:
✗ Complex setup (TTL + DLX pattern)
✗ Requires quorum queues for HA
✗ Clustering complexity
✗ Less precise than SQS (TTL is approximate)

Verdict: More complex than SQS, similar capabilities

APPROACH 3: Database Polling (Original Naive Approach)

Implementation:
Every 2 seconds:
  SELECT * FROM executions
  WHERE scheduled_time BETWEEN now AND now+2s
  AND status = PENDING

Pros:
✓ Simple to understand
✓ No additional infrastructure

Cons:
✗ High database load (0.5 QPS × 20K rows)
✗ Query overhead (500ms)
✗ Poor precision (±2 seconds)
✗ Doesn't scale beyond 1K jobs/sec

Verdict: Not suitable for 10K jobs/sec requirement
```

### Why SQS is the Best Choice:

```
1. Native Delayed Delivery
   - DelaySeconds parameter (0-900 seconds)
   - No custom implementation needed
   - Precise delivery timing

2. Fully Managed
   - No infrastructure to manage
   - Automatic scaling
   - Built-in redundancy

3. High Throughput
   - Unlimited throughput (Standard queue)
   - Batch operations (10 messages per API call)
   - Handles 10K jobs/sec easily

4. Reliability
   - At-least-once delivery
   - Visibility timeout for failure handling
   - Dead letter queue for permanent failures

5. Cost Effective
   - $0.40 per million requests
   - 1.12B requests/day = $448/day
   - Much cheaper than managing Redis/RabbitMQ cluster

6. Integration
   - Native AWS SDK support
   - Works with Lambda, ECS, EC2
   - CloudWatch metrics built-in
```



---

## PHASE 9: DEEP DIVE - SCALABILITY (4 minutes)

### What to Say:

"Let me analyze the system left-to-right to identify bottlenecks and ensure we can handle 10K jobs/second with room for growth."

### Scalability Analysis:

```
┌─────────────────────────────────────────────────────────────────┐
│  COMPONENT 1: API GATEWAY & JOB SERVICE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Load: 7K job creations/second (70% one-time jobs)             │
│                                                                 │
│  Single Instance Capacity:                                      │
│  - Request processing: 50ms                                     │
│  - Throughput: 1000ms / 50ms = 20 req/sec                      │
│  - Instances needed: 7000 / 20 = 350 instances                 │
│                                                                 │
│  With safety margin (2x): 700 instances                        │
│                                                                 │
│  Scaling Strategy:                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Auto Scaling Group (ECS/Kubernetes)                     │  │
│  │                                                          │  │
│  │  Metrics:                                                │  │
│  │  - CPU utilization > 70% → Scale up                     │  │
│  │  - Request latency > 100ms → Scale up                   │  │
│  │  - CPU utilization < 30% → Scale down                   │  │
│  │                                                          │  │
│  │  Configuration:                                          │  │
│  │  - Min instances: 100                                    │  │
│  │  - Max instances: 1000                                   │  │
│  │  - Target CPU: 60%                                       │  │
│  │  - Scale-up cooldown: 60s                                │  │
│  │  - Scale-down cooldown: 300s                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Bottleneck: ❌ None (horizontally scalable)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  COMPONENT 2: DYNAMODB - JOBS TABLE                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Load:                                                          │
│  - Writes: 7K job creations/second                             │
│  - Reads: 1K status queries/second                             │
│  - Total: 8K operations/second                                 │
│                                                                 │
│  Partition Strategy:                                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Partition Key: job_id (UUID)                            │  │
│  │                                                          │  │
│  │  Why this works:                                         │  │
│  │  - UUIDs are randomly distributed                        │  │
│  │  - No hot partitions                                     │  │
│  │  - Each write goes to different partition               │  │
│  │                                                          │  │
│  │  DynamoDB Auto-scaling:                                  │  │
│  │  - Target utilization: 70%                               │  │
│  │  - Min WCU: 1000                                         │  │
│  │  - Max WCU: 10000                                        │  │
│  │  - Min RCU: 500                                          │  │
│  │  - Max RCU: 5000                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Cost Optimization:                                             │
│  - Use on-demand pricing for unpredictable workloads           │
│  - Or provisioned capacity with auto-scaling                   │
│                                                                 │
│  Bottleneck: ❌ None (DynamoDB scales automatically)            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  COMPONENT 3: DYNAMODB - EXECUTIONS TABLE (CRITICAL)           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Load:                                                          │
│  - Writes: 30K/second (10K jobs × 3 status updates)           │
│  - Reads: 10K/second (worker fetches)                          │
│  - Total: 40K operations/second                                │
│                                                                 │
│  Problem: Hot Partition                                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Without Sharding:                                       │  │
│  │                                                          │  │
│  │  Partition Key: time_bucket (hour)                      │  │
│  │  All writes for current hour → Same partition           │  │
│  │                                                          │  │
│  │  DynamoDB limit: 1000 WCU per partition                 │  │
│  │  Our load: 30K WCU                                      │  │
│  │  Result: ❌ THROTTLING                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Solution: Write Sharding                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Partition Key: time_bucket#shard_id                    │  │
│  │                                                          │  │
│  │  Example:                                                │  │
│  │  - time_bucket = 1715547600 (hour)                      │  │
│  │  - shard_id = hash(job_id) % 10                         │  │
│  │  - partition_key = "1715547600#shard_3"                 │  │
│  │                                                          │  │
│  │  Result:                                                 │  │
│  │  - 10 partitions per hour                               │  │
│  │  - 30K WCU / 10 = 3K WCU per partition                  │  │
│  │  - Well within 1000 WCU limit ✓                         │  │
│  │                                                          │  │
│  │  Read Strategy:                                          │  │
│  │  - Scheduler queries all 10 shards in parallel          │  │
│  │  - Aggregates results                                    │  │
│  │  - Total query time: ~2 seconds                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Shard Count Calculation:                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Required WCU per partition: 1000 (DynamoDB limit)      │  │
│  │  Total WCU needed: 30,000                                │  │
│  │  Shards needed: 30,000 / 1000 = 30 shards               │  │
│  │                                                          │  │
│  │  With safety margin: 50 shards                          │  │
│  │  Per-partition load: 30,000 / 50 = 600 WCU ✓            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Implementation:                                                │
│  ```python                                                      │
│  def get_partition_key(time_bucket, job_id):                   │
│      shard_id = hash(job_id) % 50                              │
│      return f"{time_bucket}#shard_{shard_id}"                  │
│                                                                 │
│  def query_time_bucket(time_bucket):                           │
│      results = []                                              │
│      # Parallel queries across all shards                      │
│      with ThreadPoolExecutor(max_workers=50) as executor:      │
│          futures = []                                          │
│          for shard_id in range(50):                            │
│              pk = f"{time_bucket}#shard_{shard_id}"            │
│              future = executor.submit(                         │
│                  dynamodb.query,                               │
│                  KeyConditionExpression="pk = :pk"             │
│              )                                                 │
│              futures.append(future)                            │
│          for future in futures:                                │
│              results.extend(future.result())                   │
│      return results                                            │
│  ```                                                            │
│                                                                 │
│  Bottleneck: ✅ Resolved with sharding                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  COMPONENT 4: SCHEDULER SERVICE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Load: Process 3M jobs every 5 minutes                         │
│                                                                 │
│  Single Instance Performance:                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Query time: 2 seconds (parallel shard queries)          │  │
│  │  Processing: 100ms (calculate delays)                    │  │
│  │  SQS publish: 5 seconds (batch sends)                    │  │
│  │  Total: ~7 seconds                                       │  │
│  │                                                          │  │
│  │  Capacity: 3M jobs per 7 seconds = 428K jobs/sec        │  │
│  │  Required: 3M jobs per 300 seconds = 10K jobs/sec       │  │
│  │  Result: ✓ Single instance sufficient                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  High Availability:                                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Deploy 3 instances across availability zones           │  │
│  │  Use distributed lock (Redis) for coordination          │  │
│  │  Only one instance processes at a time                  │  │
│  │  Others standby for failover                            │  │
│  │                                                          │  │
│  │  Lock implementation:                                    │  │
│  │  SETNX scheduler:lock:window_id instance_id NX EX 300   │  │
│  │                                                          │  │
│  │  If primary fails:                                       │  │
│  │  - Lock expires after 5 minutes                         │  │
│  │  - Standby instance acquires lock                       │  │
│  │  - Continues processing                                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Bottleneck: ❌ None (single instance handles load)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  COMPONENT 5: AMAZON SQS                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Load:                                                          │
│  - Enqueue: 10K messages/second (new jobs)                     │
│  - Enqueue: 3K messages/second (retries)                       │
│  - Dequeue: 13K messages/second (workers)                      │
│  - Total: 26K operations/second                                │
│                                                                 │
│  SQS Capacity:                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Standard Queue:                                         │  │
│  │  - Throughput: Unlimited                                 │  │
│  │  - Latency: <10ms (p99)                                  │  │
│  │  - Message size: 256KB max                               │  │
│  │  - Retention: 14 days                                    │  │
│  │                                                          │  │
│  │  Our usage:                                              │  │
│  │  - Message size: 200 bytes                               │  │
│  │  - Throughput: 26K ops/sec                               │  │
│  │  - Well within limits ✓                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Optimization: Batch Operations                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  SendMessageBatch:                                       │  │
│  │  - Up to 10 messages per API call                       │  │
│  │  - Reduces API calls by 10x                             │  │
│  │  - Lower latency, lower cost                            │  │
│  │                                                          │  │
│  │  ReceiveMessage:                                         │  │
│  │  - MaxNumberOfMessages: 10                               │  │
│  │  - WaitTimeSeconds: 20 (long polling)                   │  │
│  │  - Reduces empty receives                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Bottleneck: ❌ None (SQS scales automatically)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  COMPONENT 6: WORKER POOL (MOST CRITICAL)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Load: 50K concurrent job executions                           │
│                                                                 │
│  Worker Capacity Calculation:                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Concurrent jobs: 50,000                                 │  │
│  │  Jobs per worker: 10 (thread pool)                      │  │
│  │  Workers needed: 50,000 / 10 = 5,000 workers            │  │
│  │                                                          │  │
│  │  With safety margin (20%): 6,000 workers                │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Instance Sizing:                                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  c5.xlarge (4 vCPU, 8GB RAM)                            │  │
│  │  - 2 workers per instance                                │  │
│  │  - Total instances: 3,000                                │  │
│  │                                                          │  │
│  │  Cost (on-demand):                                       │  │
│  │  - $0.17/hour × 3,000 = $510/hour                       │  │
│  │  - Monthly: $372,300                                     │  │
│  │                                                          │  │
│  │  Cost (spot instances, 70% discount):                   │  │
│  │  - Monthly: $111,690                                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Auto-scaling Strategy:                                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Metric: SQS ApproximateNumberOfMessages                │  │
│  │                                                          │  │
│  │  Scale-up triggers:                                      │  │
│  │  - Queue depth > 5M messages                            │  │
│  │  - Add 1000 workers                                     │  │
│  │  - Cooldown: 60 seconds                                 │  │
│  │                                                          │  │
│  │  Scale-down triggers:                                    │  │
│  │  - Queue depth < 1M messages                            │  │
│  │  - Remove 500 workers                                   │  │
│  │  - Cooldown: 300 seconds                                │  │
│  │                                                          │  │
│  │  Limits:                                                 │  │
│  │  - Min workers: 1,000 (baseline)                        │  │
│  │  - Max workers: 10,000 (10x capacity)                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Worker Implementation:                                         │
│  ```python                                                      │
│  class Worker:                                                  │
│      def __init__(self):                                        │
│          self.thread_pool = ThreadPoolExecutor(max_workers=10) │
│          self.sqs = boto3.client('sqs')                        │
│                                                                 │
│      def run(self):                                            │
│          while True:                                           │
│              # Long polling (20 seconds)                       │
│              messages = self.sqs.receive_message(              │
│                  QueueUrl=QUEUE_URL,                           │
│                  MaxNumberOfMessages=10,                       │
│                  WaitTimeSeconds=20                            │
│              )                                                 │
│                                                                 │
│              for message in messages:                          │
│                  # Submit to thread pool                       │
│                  self.thread_pool.submit(                      │
│                      self.process_job,                         │
│                      message                                   │
│                  )                                             │
│                                                                 │
│      def process_job(self, message):                           │
│          try:                                                  │
│              # Parse message                                   │
│              job_data = json.loads(message['Body'])            │
│                                                                 │
│              # Fetch job details                               │
│              job = self.fetch_job(job_data['job_id'])          │
│                                                                 │
│              # Update status: RUNNING                          │
│              self.update_status(job_data['execution_id'],      │
│                               'RUNNING')                       │
│                                                                 │
│              # Execute task                                    │
│              result = self.execute_task(job)                   │
│                                                                 │
│              # Update status: COMPLETED                        │
│              self.update_status(job_data['execution_id'],      │
│                               'COMPLETED', result)             │
│                                                                 │
│              # Delete message from SQS                         │
│              self.sqs.delete_message(                          │
│                  QueueUrl=QUEUE_URL,                           │
│                  ReceiptHandle=message['ReceiptHandle']        │
│              )                                                 │
│                                                                 │
│          except Exception as e:                                │
│              # Handle failure (see retry section)              │
│              self.handle_failure(message, e)                   │
│  ```                                                            │
│                                                                 │
│  Bottleneck: ⚠️  Worker capacity (mitigated with auto-scaling)  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Scaling Summary:

```
Component              Bottleneck?  Solution
─────────────────────────────────────────────────────────────
API Gateway            ❌ No        Horizontal scaling
Job Service            ❌ No        Auto-scaling group
Jobs Table             ❌ No        DynamoDB auto-scaling
Executions Table       ✅ Yes       Write sharding (50 shards)
Scheduler Service      ❌ No        Single instance sufficient
Amazon SQS             ❌ No        Unlimited throughput
Worker Pool            ⚠️  Maybe    Auto-scaling (1K-10K workers)
─────────────────────────────────────────────────────────────

Critical Path: Worker Pool
- Most expensive component ($111K/month)
- Requires careful capacity planning
- Auto-scaling based on queue depth
- Use spot instances for cost savings
```



---

## PHASE 10: DEEP DIVE - FAULT TOLERANCE & RETRIES (3 minutes)

### What to Say:

"Let me explain how we ensure at-least-once execution and handle various failure scenarios gracefully."

### Failure Scenarios & Solutions:

```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO 1: VISIBLE FAILURE (Task Execution Fails)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Example: HTTP callback returns 500 error                      │
│                                                                 │
│  Worker Handling:                                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  try {                                                   │  │
│  │      response = http.post(job.url, job.parameters)      │  │
│  │      if (response.status >= 500) {                      │  │
│  │          throw new RetryableException()                 │  │
│  │      }                                                   │  │
│  │  } catch (RetryableException e) {                       │  │
│  │      // Increment attempt counter                       │  │
│  │      attempt = execution.attempt + 1                    │  │
│  │                                                          │  │
│  │      if (attempt <= MAX_RETRIES) {                      │  │
│  │          // Calculate exponential backoff               │  │
│  │          delay = INITIAL_DELAY * (2 ^ attempt)          │  │
│  │          // delay = 5s, 10s, 20s for attempts 1,2,3     │  │
│  │                                                          │  │
│  │          // Update status                               │  │
│  │          update_execution(                              │  │
│  │              execution_id,                              │  │
│  │              status='RETRYING',                         │  │
│  │              attempt=attempt,                           │  │
│  │              error_message=e.message                    │  │
│  │          )                                              │  │
│  │                                                          │  │
│  │          // Re-publish to SQS with delay                │  │
│  │          sqs.send_message(                              │  │
│  │              message_body=job_data,                     │  │
│  │              delay_seconds=delay                        │  │
│  │          )                                              │  │
│  │                                                          │  │
│  │          // Delete original message                     │  │
│  │          sqs.delete_message(receipt_handle)             │  │
│  │      } else {                                           │  │
│  │          // Max retries exceeded                        │  │
│  │          update_execution(                              │  │
│  │              execution_id,                              │  │
│  │              status='FAILED',                           │  │
│  │              error_message='Max retries exceeded'       │  │
│  │          )                                              │  │
│  │                                                          │  │
│  │          // Move to Dead Letter Queue                   │  │
│  │          sqs.send_message(                              │  │
│  │              queue_url=DLQ_URL,                         │  │
│  │              message_body=job_data                      │  │
│  │          )                                              │  │
│  │                                                          │  │
│  │          // Delete original message                     │  │
│  │          sqs.delete_message(receipt_handle)             │  │
│  │      }                                                   │  │
│  │  }                                                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Timeline:                                                      │
│  T+0s:    First attempt fails                                  │
│  T+5s:    Retry attempt 1 (5s backoff)                        │
│  T+15s:   Retry attempt 2 (10s backoff)                       │
│  T+35s:   Retry attempt 3 (20s backoff)                       │
│  T+35s:   Max retries → Move to DLQ                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO 2: INVISIBLE FAILURE (Worker Crashes)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Example: Worker process crashes mid-execution                 │
│                                                                 │
│  SQS Visibility Timeout Mechanism:                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  T+0s:   Worker receives message from SQS               │  │
│  │          Message becomes invisible (timeout: 30s)        │  │
│  │                                                          │  │
│  │  T+5s:   Worker starts processing                       │  │
│  │                                                          │  │
│  │  T+10s:  Worker crashes (no delete_message call)        │  │
│  │                                                          │  │
│  │  T+30s:  Visibility timeout expires                     │  │
│  │          Message becomes visible again                  │  │
│  │                                                          │  │
│  │  T+31s:  Another worker receives message                │  │
│  │          Retries execution                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Configuration:                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  VisibilityTimeout: 30 seconds                          │  │
│  │  - Should be > max task execution time                  │  │
│  │  - Our tasks: ~5 seconds                                │  │
│  │  - 30 seconds provides buffer                           │  │
│  │                                                          │  │
│  │  MaxReceiveCount: 3                                     │  │
│  │  - After 3 visibility timeouts                          │  │
│  │  - Automatically move to DLQ                            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Preventing Duplicate Execution:                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Problem: Worker crashes after task completes but       │  │
│  │           before delete_message                         │  │
│  │                                                          │  │
│  │  Solution: Idempotency Check                            │  │
│  │                                                          │  │
│  │  def process_job(message):                              │  │
│  │      execution_id = message['execution_id']             │  │
│  │                                                          │  │
│  │      # Check if already completed                       │  │
│  │      execution = get_execution(execution_id)            │  │
│  │      if execution.status == 'COMPLETED':                │  │
│  │          # Already processed, just delete message       │  │
│  │          sqs.delete_message(receipt_handle)             │  │
│  │          return                                         │  │
│  │                                                          │  │
│  │      # Proceed with execution                           │  │
│  │      execute_task(job)                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO 3: DATABASE FAILURE                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DynamoDB High Availability:                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  - Multi-AZ replication (automatic)                     │  │
│  │  - 99.99% availability SLA                              │  │
│  │  - Automatic failover                                   │  │
│  │  - No manual intervention needed                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Transient Failure Handling:                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  def update_execution_with_retry(execution_id, status): │  │
│  │      max_retries = 3                                    │  │
│  │      for attempt in range(max_retries):                 │  │
│  │          try:                                           │  │
│  │              dynamodb.update_item(                      │  │
│  │                  Key={'execution_id': execution_id},    │  │
│  │                  UpdateExpression='SET status = :s',    │  │
│  │                  ExpressionAttributeValues={            │  │
│  │                      ':s': status                       │  │
│  │                  }                                      │  │
│  │              )                                          │  │
│  │              return                                     │  │
│  │          except ClientError as e:                       │  │
│  │              if attempt < max_retries - 1:              │  │
│  │                  time.sleep(2 ** attempt)  # Backoff    │  │
│  │              else:                                      │  │
│  │                  raise                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Graceful Degradation:                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  If database is unavailable:                            │  │
│  │  1. Worker continues executing tasks                    │  │
│  │  2. Status updates queued in memory                     │  │
│  │  3. Retry updates when database recovers                │  │
│  │  4. SQS message not deleted until status updated        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO 4: SQS FAILURE                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SQS High Availability:                                         │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  - Multi-AZ replication (automatic)                     │  │
│  │  - 99.9% availability SLA                               │  │
│  │  - Messages replicated across multiple servers          │  │
│  │  - No single point of failure                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Unlikely Scenario: SQS Unavailable                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Impact:                                                 │  │
│  │  - New jobs cannot be queued                            │  │
│  │  - Workers cannot receive messages                      │  │
│  │                                                          │  │
│  │  Mitigation:                                             │  │
│  │  1. Jobs remain in database (durable)                   │  │
│  │  2. Scheduler will retry publishing on next cycle       │  │
│  │  3. Workers retry polling with exponential backoff      │  │
│  │  4. Circuit breaker prevents cascading failures         │  │
│  │                                                          │  │
│  │  Recovery:                                               │  │
│  │  - When SQS recovers, scheduler publishes pending jobs  │  │
│  │  - Workers resume processing                            │  │
│  │  - No job loss (all in database)                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO 5: SCHEDULER SERVICE FAILURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  High Availability Setup:                                       │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Deploy 3 scheduler instances across AZs:               │  │
│  │                                                          │  │
│  │  Instance 1 (us-east-1a): PRIMARY                       │  │
│  │  Instance 2 (us-east-1b): STANDBY                       │  │
│  │  Instance 3 (us-east-1c): STANDBY                       │  │
│  │                                                          │  │
│  │  Coordination via Redis distributed lock:               │  │
│  │  - Only one instance active at a time                   │  │
│  │  - Lock TTL: 5 minutes (same as cycle)                  │  │
│  │  - Automatic failover on lock expiry                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Failure Timeline:                                              │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  T+0:00  Instance 1 acquires lock, starts processing    │  │
│  │  T+0:30  Instance 1 crashes                             │  │
│  │  T+5:00  Lock expires (TTL)                             │  │
│  │  T+5:01  Instance 2 acquires lock                       │  │
│  │  T+5:01  Instance 2 processes jobs                      │  │
│  │                                                          │  │
│  │  Impact: 30-second delay in scheduling                  │  │
│  │  - Jobs still execute within 2s of scheduled time       │  │
│  │  - No job loss (all in database)                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Idempotency Protection:                                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Problem: Both instances process same window            │  │
│  │                                                          │  │
│  │  Solution: Conditional writes                           │  │
│  │                                                          │  │
│  │  dynamodb.update_item(                                  │  │
│  │      Key={'execution_id': execution_id},                │  │
│  │      UpdateExpression='SET status = :new',              │  │
│  │      ConditionExpression='status = :old',               │  │
│  │      ExpressionAttributeValues={                        │  │
│  │          ':new': 'SCHEDULED',                           │  │
│  │          ':old': 'PENDING'                              │  │
│  │      }                                                   │  │
│  │  )                                                       │  │
│  │                                                          │  │
│  │  If already SCHEDULED: Update fails, skip publishing    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Idempotency Strategies:

```
┌─────────────────────────────────────────────────────────────────┐
│  APPROACH 1: TASK-LEVEL IDEMPOTENCY (RECOMMENDED)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Design tasks to be naturally idempotent:                      │
│                                                                 │
│  ❌ Bad: "Increment counter by 1"                               │
│  ✅ Good: "Set counter to 42"                                   │
│                                                                 │
│  ❌ Bad: "Send welcome email"                                   │
│  ✅ Good: "Send welcome email if not already sent"              │
│                                                                 │
│  ❌ Bad: "Transfer $100"                                        │
│  ✅ Good: "Transfer $100 with transaction_id=xyz"               │
│                                                                 │
│  Implementation:                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  HTTP Callback with Idempotency Key:                    │  │
│  │                                                          │  │
│  │  POST /webhook                                          │  │
│  │  Headers:                                                │  │
│  │    Idempotency-Key: execution_id                        │  │
│  │  Body:                                                   │  │
│  │    {                                                     │  │
│  │      "job_id": "job_123",                               │  │
│  │      "execution_id": "exec_456",                        │  │
│  │      "parameters": {...}                                │  │
│  │    }                                                     │  │
│  │                                                          │  │
│  │  Downstream service:                                     │  │
│  │  - Checks if execution_id already processed             │  │
│  │  - If yes: Return cached result                         │  │
│  │  - If no: Process and cache result                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  APPROACH 2: DEDUPLICATION TABLE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Track completed executions to prevent duplicates:             │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  def process_job(execution_id, job):                    │  │
│  │      # Check if already processed                       │  │
│  │      if dedup_table.exists(execution_id):               │  │
│  │          return dedup_table.get_result(execution_id)    │  │
│  │                                                          │  │
│  │      # Process job                                      │  │
│  │      result = execute_task(job)                         │  │
│  │                                                          │  │
│  │      # Store result                                     │  │
│  │      dedup_table.put(execution_id, result)              │  │
│  │                                                          │  │
│  │      return result                                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Deduplication Table Schema:                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  {                                                       │  │
│  │    "execution_id": "exec_456",  // Partition Key        │  │
│  │    "result": {...},                                     │  │
│  │    "completed_at": 1715548800,                          │  │
│  │    "ttl": 1715635200  // 24 hours                       │  │
│  │  }                                                       │  │
│  │                                                          │  │
│  │  - TTL: Auto-delete after 24 hours                      │  │
│  │  - Prevents unbounded growth                            │  │
│  │  - 24 hours sufficient for retry window                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Pros:                                                          │
│  ✅ Prevents duplicate execution                                │
│  ✅ Works with any task type                                    │
│                                                                 │
│  Cons:                                                          │
│  ❌ Additional database operations                              │
│  ❌ Race condition window (check → execute → write)             │
│  ❌ Requires cleanup (TTL)                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  APPROACH 3: STATUS-BASED IDEMPOTENCY (CURRENT DESIGN)         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Use execution status to prevent duplicates:                   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  def process_job(message):                              │  │
│  │      execution_id = message['execution_id']             │  │
│  │                                                          │  │
│  │      # Fetch current status                             │  │
│  │      execution = get_execution(execution_id)            │  │
│  │                                                          │  │
│  │      # Check if already completed                       │  │
│  │      if execution.status in ['COMPLETED', 'FAILED']:    │  │
│  │          # Already processed                            │  │
│  │          sqs.delete_message(receipt_handle)             │  │
│  │          return                                         │  │
│  │                                                          │  │
│  │      # Update status to RUNNING (atomic)                │  │
│  │      try:                                               │  │
│  │          update_status(                                 │  │
│  │              execution_id,                              │  │
│  │              new_status='RUNNING',                      │  │
│  │              condition='status = PENDING'               │  │
│  │          )                                              │  │
│  │      except ConditionalCheckFailedException:            │  │
│  │          # Another worker already processing            │  │
│  │          return                                         │  │
│  │                                                          │  │
│  │      # Execute task                                     │  │
│  │      result = execute_task(job)                         │  │
│  │                                                          │  │
│  │      # Update status to COMPLETED                       │  │
│  │      update_status(execution_id, 'COMPLETED', result)   │  │
│  │                                                          │  │
│  │      # Delete message                                   │  │
│  │      sqs.delete_message(receipt_handle)                 │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Pros:                                                          │
│  ✅ No additional table needed                                  │
│  ✅ Atomic status transitions                                   │
│  ✅ Prevents concurrent execution                               │
│                                                                 │
│  Cons:                                                          │
│  ❌ Doesn't prevent task side effects if task not idempotent    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Monitoring & Alerting:

```
Key Metrics to Track:

1. Job Success Rate
   - Metric: (completed_jobs / total_jobs) × 100
   - Alert: < 95%
   - Action: Investigate failed jobs in DLQ

2. Execution Latency
   - Metric: actual_execution_time - scheduled_time
   - Alert: P99 > 5 seconds
   - Action: Scale up workers or investigate bottlenecks

3. Retry Rate
   - Metric: (retried_jobs / total_jobs) × 100
   - Alert: > 10%
   - Action: Investigate task failures

4. Queue Depth
   - Metric: SQS ApproximateNumberOfMessages
   - Alert: > 10M messages
   - Action: Scale up workers

5. Worker Health
   - Metric: Active workers / Total workers
   - Alert: < 80%
   - Action: Investigate worker crashes

6. Database Throttling
   - Metric: DynamoDB ThrottledRequests
   - Alert: > 100 per minute
   - Action: Increase provisioned capacity or add shards
```



---

## FOLLOW-UP QUESTIONS & ADVANCED TOPICS

### Q1: "How would you handle job dependencies?"

**Answer:**
```
Approach: Directed Acyclic Graph (DAG) Execution

Data Model Extension:
{
  "job_id": "job_123",
  "dependencies": ["job_456", "job_789"],  // Must complete first
  "status": "WAITING_ON_DEPENDENCIES"
}

Execution Flow:
1. When job completes, publish event to Kafka topic
2. Dependency resolver service listens to completion events
3. For each dependent job:
   - Check if all dependencies completed
   - If yes: Update status to PENDING, schedule execution
   - If no: Keep waiting

Implementation:
- Use Redis to track dependency completion
- SADD job:123:completed_deps job_456
- SCARD job:123:completed_deps == total_deps → Ready

Challenges:
- Circular dependency detection
- Partial failure handling
- Long-running dependency chains
```

### Q2: "How would you implement job priorities?"

**Answer:**
```
Approach: Multiple SQS Queues with Priority

Queue Setup:
- High Priority Queue (P0)
- Medium Priority Queue (P1)
- Low Priority Queue (P2)

Worker Polling Strategy:
while True:
    # Poll high priority first
    messages = poll_queue(high_priority_queue, max=10)
    if messages:
        process(messages)
        continue
    
    # Then medium priority
    messages = poll_queue(medium_priority_queue, max=10)
    if messages:
        process(messages)
        continue
    
    # Finally low priority
    messages = poll_queue(low_priority_queue, max=10)
    if messages:
        process(messages)

Worker Allocation:
- 70% workers: All priorities (round-robin)
- 20% workers: High + Medium only
- 10% workers: High priority only

Prevents starvation while ensuring high-priority jobs execute first
```

### Q3: "How would you support multi-region deployment?"

**Answer:**
```
Architecture: Active-Active Multi-Region

Region Setup:
┌─────────────────────────────────────────────────────────┐
│  US-EAST-1                    US-WEST-2                 │
│  ┌──────────────┐            ┌──────────────┐          │
│  │ API Gateway  │            │ API Gateway  │          │
│  │ Job Service  │            │ Job Service  │          │
│  │ Workers      │            │ Workers      │          │
│  │ DynamoDB     │◄──────────►│ DynamoDB     │          │
│  │ (Global)     │  Replication│ (Global)     │          │
│  └──────────────┘            └──────────────┘          │
└─────────────────────────────────────────────────────────┘

DynamoDB Global Tables:
- Automatic multi-region replication
- Last-writer-wins conflict resolution
- Sub-second replication latency

Job Routing:
- User creates job in nearest region
- Job replicated to all regions
- Scheduler in each region processes local jobs
- Workers execute jobs in same region

Challenges:
- Clock skew between regions
- Duplicate execution prevention
- Cross-region latency
```

### Q4: "How would you handle very long-running jobs (hours)?"

**Answer:**
```
Problem: SQS visibility timeout max = 12 hours

Solution: Heartbeat Pattern

Implementation:
1. Worker receives job from SQS
2. Worker starts background thread:
   - Every 5 minutes: Extend visibility timeout
   - ChangeMessageVisibility(ReceiptHandle, 600)
3. Worker executes long-running task
4. On completion: Delete message

Alternative: Async Job Pattern
1. Worker receives job from SQS
2. Worker starts job in background (Lambda, ECS task)
3. Worker immediately deletes SQS message
4. Background job updates status via callback
5. Use separate tracking mechanism (not SQS)

For very long jobs (days):
- Store job state in database
- Use cron to check progress
- Resume from checkpoint on failure
```

### Q5: "How would you implement rate limiting per user?"

**Answer:**
```
Approach: Redis Token Bucket

Implementation:
def can_create_job(user_id, limit=100):
    key = f"rate_limit:{user_id}:{current_minute}"
    
    # Increment counter
    count = redis.incr(key)
    
    # Set expiry on first request
    if count == 1:
        redis.expire(key, 60)
    
    # Check limit
    return count <= limit

Enhanced: Sliding Window
def can_create_job(user_id, limit=100, window=60):
    key = f"rate_limit:{user_id}"
    now = time.time()
    window_start = now - window
    
    # Remove old entries
    redis.zremrangebyscore(key, 0, window_start)
    
    # Count current requests
    count = redis.zcard(key)
    
    if count >= limit:
        return False
    
    # Add current request
    redis.zadd(key, {str(now): now})
    redis.expire(key, window)
    
    return True

API Gateway Integration:
- Check rate limit before creating job
- Return 429 Too Many Requests if exceeded
- Include Retry-After header
```

### Q6: "How would you implement job result storage?"

**Answer:**
```
Approach: Separate Result Storage

Data Model:
{
  "execution_id": "exec_456",
  "job_id": "job_123",
  "status": "COMPLETED",
  "result_location": "s3://results/exec_456.json"
}

Storage Strategy:
1. Small results (<1KB): Store in DynamoDB
2. Medium results (1KB-1MB): Store in DynamoDB with compression
3. Large results (>1MB): Store in S3, reference in DynamoDB

Implementation:
def store_result(execution_id, result):
    result_size = len(json.dumps(result))
    
    if result_size < 1024:
        # Store directly in DynamoDB
        update_execution(execution_id, result=result)
    else:
        # Store in S3
        s3_key = f"results/{execution_id}.json"
        s3.put_object(Bucket=BUCKET, Key=s3_key, Body=json.dumps(result))
        
        # Store reference
        update_execution(execution_id, result_location=f"s3://{BUCKET}/{s3_key}")

Retrieval:
def get_result(execution_id):
    execution = get_execution(execution_id)
    
    if execution.result:
        return execution.result
    elif execution.result_location:
        # Fetch from S3
        s3_key = execution.result_location.replace(f"s3://{BUCKET}/", "")
        obj = s3.get_object(Bucket=BUCKET, Key=s3_key)
        return json.loads(obj['Body'].read())

Lifecycle:
- S3 lifecycle policy: Delete after 30 days
- DynamoDB TTL: Delete execution records after 90 days
```

---

## COMPLETE CODE IMPLEMENTATION

### Worker Service (Python)

```python
import boto3
import json
import time
import requests
from concurrent.futures import ThreadPoolExecutor
from typing import Dict, Any

class TaskSchedulerWorker:
    def __init__(self):
        self.sqs = boto3.client('sqs')
        self.dynamodb = boto3.resource('dynamodb')
        self.executions_table = self.dynamodb.Table('executions')
        self.jobs_table = self.dynamodb.Table('jobs')
        self.thread_pool = ThreadPoolExecutor(max_workers=10)
        
        self.queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789/job-queue'
        self.max_retries = 3
        self.initial_backoff = 5
        
    def run(self):
        """Main worker loop"""
        print("Worker started, polling for jobs...")
        
        while True:
            try:
                # Long polling (20 seconds)
                response = self.sqs.receive_message(
                    QueueUrl=self.queue_url,
                    MaxNumberOfMessages=10,
                    WaitTimeSeconds=20,
                    VisibilityTimeout=30
                )
                
                messages = response.get('Messages', [])
                
                if not messages:
                    continue
                
                # Process messages in parallel
                futures = []
                for message in messages:
                    future = self.thread_pool.submit(
                        self.process_message,
                        message
                    )
                    futures.append(future)
                
                # Wait for all to complete
                for future in futures:
                    future.result()
                    
            except Exception as e:
                print(f"Error in main loop: {e}")
                time.sleep(5)
    
    def process_message(self, message: Dict[str, Any]):
        """Process a single job message"""
        try:
            # Parse message
            body = json.loads(message['Body'])
            execution_id = body['execution_id']
            job_id = body['job_id']
            receipt_handle = message['ReceiptHandle']
            
            print(f"Processing execution {execution_id}")
            
            # Check if already completed (idempotency)
            execution = self.get_execution(execution_id)
            if execution['status'] in ['COMPLETED', 'FAILED']:
                print(f"Execution {execution_id} already {execution['status']}")
                self.delete_message(receipt_handle)
                return
            
            # Update status to RUNNING
            try:
                self.update_execution_status(
                    execution_id,
                    'RUNNING',
                    condition_status='PENDING'
                )
            except Exception as e:
                # Another worker already processing
                print(f"Execution {execution_id} already being processed")
                return
            
            # Fetch job details
            job = self.get_job(job_id)
            
            # Execute task
            result = self.execute_task(job, body.get('parameters', {}))
            
            # Update status to COMPLETED
            self.update_execution_status(
                execution_id,
                'COMPLETED',
                result=result
            )
            
            # Delete message from SQS
            self.delete_message(receipt_handle)
            
            print(f"Execution {execution_id} completed successfully")
            
        except Exception as e:
            print(f"Error processing message: {e}")
            self.handle_failure(message, e)
    
    def execute_task(self, job: Dict[str, Any], parameters: Dict[str, Any]) -> Dict[str, Any]:
        """Execute the actual task"""
        task_type = job['task_type']
        
        if task_type == 'HTTP_CALLBACK':
            return self.execute_http_callback(job, parameters)
        elif task_type == 'MESSAGE_PUBLISH':
            return self.execute_message_publish(job, parameters)
        else:
            raise ValueError(f"Unknown task type: {task_type}")
    
    def execute_http_callback(self, job: Dict[str, Any], parameters: Dict[str, Any]) -> Dict[str, Any]:
        """Execute HTTP callback task"""
        config = job['config']['http']
        
        response = requests.request(
            method=config['method'],
            url=config['url'],
            json=parameters,
            headers=config.get('headers', {}),
            timeout=config.get('timeout_seconds', 30)
        )
        
        if response.status_code >= 500:
            raise Exception(f"HTTP {response.status_code}: {response.text}")
        
        return {
            'status_code': response.status_code,
            'response': response.text[:1000]  # Truncate
        }
    
    def execute_message_publish(self, job: Dict[str, Any], parameters: Dict[str, Any]) -> Dict[str, Any]:
        """Execute message publish task"""
        config = job['config']['message']
        
        # Publish to SNS/SQS/Kafka
        # Implementation depends on message broker
        
        return {'status': 'published'}
    
    def handle_failure(self, message: Dict[str, Any], error: Exception):
        """Handle task failure with retry logic"""
        try:
            body = json.loads(message['Body'])
            execution_id = body['execution_id']
            receipt_handle = message['ReceiptHandle']
            
            # Get current attempt
            execution = self.get_execution(execution_id)
            attempt = execution.get('attempt', 0) + 1
            
            if attempt <= self.max_retries:
                # Calculate exponential backoff
                delay = self.initial_backoff * (2 ** (attempt - 1))
                
                print(f"Retrying execution {execution_id}, attempt {attempt}, delay {delay}s")
                
                # Update status to RETRYING
                self.update_execution_status(
                    execution_id,
                    'RETRYING',
                    attempt=attempt,
                    error_message=str(error)
                )
                
                # Re-publish to SQS with delay
                self.sqs.send_message(
                    QueueUrl=self.queue_url,
                    MessageBody=message['Body'],
                    DelaySeconds=min(delay, 900)  # Max 15 minutes
                )
                
                # Delete original message
                self.delete_message(receipt_handle)
                
            else:
                print(f"Max retries exceeded for execution {execution_id}")
                
                # Update status to FAILED
                self.update_execution_status(
                    execution_id,
                    'FAILED',
                    error_message=f"Max retries exceeded: {error}"
                )
                
                # Move to DLQ (SQS does this automatically with MaxReceiveCount)
                # Or manually send to DLQ
                
                # Delete original message
                self.delete_message(receipt_handle)
                
        except Exception as e:
            print(f"Error handling failure: {e}")
    
    def get_execution(self, execution_id: str) -> Dict[str, Any]:
        """Fetch execution from DynamoDB"""
        response = self.executions_table.get_item(
            Key={'execution_id': execution_id}
        )
        return response.get('Item', {})
    
    def get_job(self, job_id: str) -> Dict[str, Any]:
        """Fetch job from DynamoDB"""
        response = self.jobs_table.get_item(
            Key={'job_id': job_id}
        )
        return response.get('Item', {})
    
    def update_execution_status(self, execution_id: str, status: str, 
                                condition_status: str = None, **kwargs):
        """Update execution status in DynamoDB"""
        update_expr = 'SET #status = :status, updated_at = :updated_at'
        expr_values = {
            ':status': status,
            ':updated_at': int(time.time())
        }
        expr_names = {'#status': 'status'}
        
        # Add optional fields
        for key, value in kwargs.items():
            update_expr += f', {key} = :{key}'
            expr_values[f':{key}'] = value
        
        # Conditional update
        condition_expr = None
        if condition_status:
            condition_expr = '#status = :condition_status'
            expr_values[':condition_status'] = condition_status
        
        self.executions_table.update_item(
            Key={'execution_id': execution_id},
            UpdateExpression=update_expr,
            ExpressionAttributeNames=expr_names,
            ExpressionAttributeValues=expr_values,
            ConditionExpression=condition_expr
        )
    
    def delete_message(self, receipt_handle: str):
        """Delete message from SQS"""
        self.sqs.delete_message(
            QueueUrl=self.queue_url,
            ReceiptHandle=receipt_handle
        )

if __name__ == '__main__':
    worker = TaskSchedulerWorker()
    worker.run()
```

### Scheduler Service (Python)

```python
import boto3
import json
import time
from datetime import datetime, timedelta
from croniter import croniter
from concurrent.futures import ThreadPoolExecutor

class TaskSchedulerService:
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.sqs = boto3.client('sqs')
        self.redis = redis.Redis(host='localhost', port=6379)
        
        self.executions_table = self.dynamodb.Table('executions')
        self.jobs_table = self.dynamodb.Table('jobs')
        self.queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789/job-queue'
        
        self.num_shards = 50
        self.window_minutes = 5
        
    def run(self):
        """Main scheduler loop - runs every 5 minutes"""
        while True:
            try:
                # Acquire distributed lock
                lock_acquired = self.acquire_lock()
                
                if lock_acquired:
                    print("Lock acquired, processing jobs...")
                    self.process_upcoming_jobs()
                    self.release_lock()
                else:
                    print("Lock not acquired, another instance is running")
                
                # Sleep until next cycle
                time.sleep(300)  # 5 minutes
                
            except Exception as e:
                print(f"Error in scheduler loop: {e}")
                self.release_lock()
                time.sleep(60)
    
    def acquire_lock(self) -> bool:
        """Acquire distributed lock using Redis"""
        lock_key = f"scheduler:lock:{int(time.time() / 300)}"
        instance_id = f"instance_{time.time()}"
        
        # Try to acquire lock with 5-minute TTL
        acquired = self.redis.set(
            lock_key,
            instance_id,
            nx=True,
            ex=300
        )
        
        return acquired
    
    def release_lock(self):
        """Release distributed lock"""
        lock_key = f"scheduler:lock:{int(time.time() / 300)}"
        self.redis.delete(lock_key)
    
    def process_upcoming_jobs(self):
        """Process jobs due in next 5 minutes"""
        now = int(time.time())
        window_end = now + (self.window_minutes * 60)
        
        # Calculate time buckets (current hour + next hour)
        current_hour = (now // 3600) * 3600
        next_hour = current_hour + 3600
        time_buckets = [current_hour, next_hour]
        
        jobs_to_schedule = []
        
        # Query all shards in parallel
        with ThreadPoolExecutor(max_workers=self.num_shards) as executor:
            futures = []
            
            for time_bucket in time_buckets:
                for shard_id in range(self.num_shards):
                    future = executor.submit(
                        self.query_shard,
                        time_bucket,
                        shard_id,
                        now,
                        window_end
                    )
                    futures.append(future)
            
            # Collect results
            for future in futures:
                jobs_to_schedule.extend(future.result())
        
        print(f"Found {len(jobs_to_schedule)} jobs to schedule")
        
        # Publish to SQS in batches
        self.publish_to_sqs(jobs_to_schedule, now)
        
        # Handle recurring jobs
        self.schedule_next_occurrences(jobs_to_schedule)
    
    def query_shard(self, time_bucket: int, shard_id: int, 
                   window_start: int, window_end: int):
        """Query a single shard for upcoming jobs"""
        partition_key = f"{time_bucket}#shard_{shard_id}"
        
        response = self.executions_table.query(
            KeyConditionExpression='time_bucket = :pk AND execution_time_job_id BETWEEN :start AND :end',
            FilterExpression='#status = :status',
            ExpressionAttributeNames={'#status': 'status'},
            ExpressionAttributeValues={
                ':pk': partition_key,
                ':start': f"{window_start}-",
                ':end': f"{window_end}-",
                ':status': 'PENDING'
            }
        )
        
        return response.get('Items', [])
    
    def publish_to_sqs(self, jobs: list, now: int):
        """Publish jobs to SQS with appropriate delays"""
        batch = []
        
        for job in jobs:
            execution_time = int(job['scheduled_time'])
            delay = max(0, execution_time - now)
            
            # Skip if delay > 15 minutes (SQS limit)
            if delay > 900:
                continue
            
            # Update status to SCHEDULED
            try:
                self.executions_table.update_item(
                    Key={'execution_id': job['execution_id']},
                    UpdateExpression='SET #status = :new_status',
                    ConditionExpression='#status = :old_status',
                    ExpressionAttributeNames={'#status': 'status'},
                    ExpressionAttributeValues={
                        ':new_status': 'SCHEDULED',
                        ':old_status': 'PENDING'
                    }
                )
            except:
                # Already scheduled by another instance
                continue
            
            # Add to batch
            batch.append({
                'Id': job['execution_id'],
                'MessageBody': json.dumps({
                    'execution_id': job['execution_id'],
                    'job_id': job['job_id'],
                    'scheduled_time': execution_time
                }),
                'DelaySeconds': delay
            })
            
            # Send batch when full
            if len(batch) >= 10:
                self.send_batch(batch)
                batch = []
        
        # Send remaining
        if batch:
            self.send_batch(batch)
    
    def send_batch(self, batch: list):
        """Send batch of messages to SQS"""
        try:
            self.sqs.send_message_batch(
                QueueUrl=self.queue_url,
                Entries=batch
            )
            print(f"Sent batch of {len(batch)} messages")
        except Exception as e:
            print(f"Error sending batch: {e}")
    
    def schedule_next_occurrences(self, completed_jobs: list):
        """Schedule next occurrences for recurring jobs"""
        for execution in completed_jobs:
            job_id = execution['job_id']
            
            # Fetch job definition
            job = self.get_job(job_id)
            
            if job.get('schedule', {}).get('type') == 'CRON':
                # Calculate next execution time
                cron_expr = job['schedule']['expression']
                base_time = datetime.fromtimestamp(execution['scheduled_time'])
                
                cron = croniter(cron_expr, base_time)
                next_time = cron.get_next(datetime)
                next_timestamp = int(next_time.timestamp())
                
                # Create new execution record
                self.create_execution(job_id, next_timestamp)
                
                # Update job's next_execution_time
                self.jobs_table.update_item(
                    Key={'job_id': job_id},
                    UpdateExpression='SET next_execution_time = :next_time',
                    ExpressionAttributeValues={':next_time': next_timestamp}
                )
    
    def create_execution(self, job_id: str, scheduled_time: int):
        """Create new execution record"""
        execution_id = f"exec_{int(time.time() * 1000)}"
        time_bucket = (scheduled_time // 3600) * 3600
        shard_id = hash(job_id) % self.num_shards
        
        self.executions_table.put_item(
            Item={
                'execution_id': execution_id,
                'job_id': job_id,
                'time_bucket': f"{time_bucket}#shard_{shard_id}",
                'execution_time_job_id': f"{scheduled_time}-{job_id}",
                'scheduled_time': scheduled_time,
                'status': 'PENDING',
                'attempt': 0,
                'created_at': int(time.time())
            }
        )
    
    def get_job(self, job_id: str):
        """Fetch job from DynamoDB"""
        response = self.jobs_table.get_item(Key={'job_id': job_id})
        return response.get('Item', {})

if __name__ == '__main__':
    scheduler = TaskSchedulerService()
    scheduler.run()
```

---

## KEY TAKEAWAYS FOR INTERVIEW

### What Makes This Design Strong:

```
1. Two-Layered Architecture
   ✓ Separates durability (database) from precision (SQS)
   ✓ Achieves 2-second execution precision
   ✓ Handles 10K jobs/second throughput

2. Scalability
   ✓ Write sharding prevents hot partitions
   ✓ Horizontal scaling of all components
   ✓ Auto-scaling based on queue depth

3. Reliability
   ✓ At-least-once execution guarantee
   ✓ Automatic retry with exponential backoff
   ✓ Dead letter queue for permanent failures
   ✓ Idempotency protection

4. Operational Excellence
   ✓ Fully managed services (DynamoDB, SQS)
   ✓ Minimal operational overhead
   ✓ Comprehensive monitoring and alerting
   ✓ Cost-effective ($270K/month for 10K jobs/sec)
```

### Common Mistakes to Avoid:

```
❌ Using database polling for timely execution
❌ Not considering hot partitions in time-series data
❌ Forgetting idempotency protection
❌ Not planning for failure scenarios
❌ Ignoring cost implications of design choices
❌ Over-engineering with unnecessary components
```

### How to Present This Design:

```
1. Start Simple (5 min)
   - Basic architecture: API → Database → Workers
   - Explain why it doesn't scale

2. Add Complexity Gradually (10 min)
   - Introduce SQS for timely execution
   - Add write sharding for hot partitions
   - Explain retry logic and failure handling

3. Deep Dive on Request (15 min)
   - Detailed data flows
   - Failure scenarios
   - Scalability analysis
   - Alternative approaches

4. Show Trade-offs (5 min)
   - Why SQS over Redis/RabbitMQ
   - Cost vs complexity
   - Consistency vs availability
```

---

## CONCLUSION

This task scheduler design demonstrates a production-ready system that:
- Executes 10K jobs/second with 2-second precision
- Provides at-least-once execution guarantees
- Scales horizontally across all components
- Handles failures gracefully with automatic retries
- Uses managed services to minimize operational overhead
- Costs ~$270K/month for the specified scale

The key insight is the two-layered architecture that separates durable storage (DynamoDB) from timely execution (SQS), achieving both reliability and precision without sacrificing scalability.

