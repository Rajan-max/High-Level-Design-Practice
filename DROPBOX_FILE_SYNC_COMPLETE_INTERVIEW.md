# Dropbox / File Synchronization Service - Complete Interview Guide

## INTERVIEW STRUCTURE (45 minutes)

```
Phase 1: Requirements Clarification (5 min)
Phase 2: Functional & Non-Functional Requirements (3 min)
Phase 3: Back-of-Envelope Estimation (5 min)
Phase 4: High-Level Architecture (8 min)
Phase 5: API Design (3 min)
Phase 6: Data Models (4 min)
Phase 7: Core Flows - Upload/Download/Sync (8 min)
Phase 8: Deep Dive - Block-Level Deduplication (5 min)
Phase 9: Deep Dive - Large File Handling & Chunking (4 min)
```

-----------------------------------------------------------

## PHASE 1: REQUIREMENTS CLARIFICATION (5 minutes)

### What to Say:

"Thank you for the problem. I want to design a cloud-based file storage and synchronization service like Dropbox. Let me clarify the requirements and scope."

### Questions to Ask:

**Q1: Core Functionality**
- "Should users be able to upload, download, and sync files across devices?"
- "Do we need real-time sync or is eventual consistency acceptable?"

**Expected Answer:** Upload/download/sync across devices, near real-time sync (few seconds delay acceptable)

**Q2: File Operations**
- "Should users be able to edit files directly in the system?"
- "Do we need file versioning (history of changes)?"
- "What about file preview without downloading?"

**Expected Answer:** Editing out of scope, versioning nice-to-have, preview out of scope

**Q3: Sharing & Collaboration**
- "Should users be able to share files with other users?"
- "Do we need folder-level sharing or just file-level?"
- "What about public links for non-users?"

**Expected Answer:** File and folder sharing with other users, public links out of scope

**Q4: File Size Limits - CRITICAL**
- "What's the maximum file size we need to support?"
- "Average file size?"
- "Types of files (documents, images, videos)?"

**Expected Answer:** Support up to 50GB files, average 10MB, all file types

**Q5: Storage Limits**
- "Is there a storage quota per user?"
- "How much total storage do we need to support?"

**Expected Answer:** Storage limits out of scope for now, focus on architecture

**Q6: Scale - THE KEY QUESTION**
- "How many users?"
- "How many files per user on average?"
- "Expected upload/download traffic?"

**Expected Answer:** 100M users, 1000 files per user average, read-heavy (10:1 ratio)

**Q7: Sync Behavior**
- "Should sync happen automatically or manually?"
- "What triggers a sync (file change, periodic check)?"
- "How do we handle conflicts (two users edit same file)?"

**Expected Answer:** Automatic sync, file system events trigger sync, last-write-wins for conflicts

**Q8: Reliability**
- "What happens if upload fails midway?"
- "Should we support resumable uploads?"
- "Data durability requirements?"

**Expected Answer:** Resumable uploads critical, no data loss acceptable

----------------------------------------------------------------------------------------------------

## PHASE 2: FUNCTIONAL & NON-FUNCTIONAL REQUIREMENTS (3 minutes)

### What to Say:

"Based on our discussion, let me summarize the requirements."

### Functional Requirements (Priority Order):

```
1. File Upload (CORE)
   - Upload files from any device
   - Support files up to 50GB
   - Resumable uploads
   - Progress indicator

2. File Download (CORE)
   - Download files to any device
   - Fast download speeds
   - Resume interrupted downloads

3. File Synchronization (CORE)
   - Automatic sync across devices
   - Detect local file changes
   - Detect remote file changes
   - Bidirectional sync (local ↔ remote)

4. File Sharing (IMPORTANT)
   - Share files with other users
   - Share folders with other users
   - View shared files
   - Permission management (view/edit)

5. File Management (IMPORTANT)
   - List files and folders
   - Create/delete folders
   - Rename files
   - Move files between folders
   - Delete files

Nice-to-have:
- File versioning (history)
- File preview
- Conflict resolution UI
- Offline access

Out of Scope:
- In-app file editing
- Real-time collaboration
- Comments and annotations
- Mobile photo backup
- File encryption (client-side)
```

### Non-Functional Requirements:

```
1. Performance
   - Upload/download: Fast as network allows
   - Sync latency: < 5 seconds
   - API response: < 200ms
   - Support 50GB files

2. Scalability
   - 100M users
   - 100B total files
   - 1000 files per user average
   - 10TB total storage per user (theoretical)

3. Availability
   - 99.99% uptime (four nines)
   - No single point of failure
   - Graceful degradation

4. Consistency
   - Eventual consistency acceptable
   - Last-write-wins for conflicts
   - Strong consistency for metadata

5. Reliability & Durability
   - No data loss (99.999999999% durability)
   - Resumable uploads/downloads
   - Automatic retry on failure
   - Data replication

6. Efficiency
   - Minimize bandwidth usage
   - Avoid uploading duplicate data
   - Only sync changed portions of files
   - Compression where beneficial

7. Security
   - Encryption in transit (HTTPS)
   - Encryption at rest
   - Access control (ACLs)
   - Secure file sharing
```

### Why This Matters:
✓ 50GB file support requires chunking
✓ Deduplication critical for storage efficiency
✓ Resumable uploads mandatory for large files
✓ Sync is the core challenge (bidirectional)

--------------------------------------------------------------

## PHASE 3: BACK-OF-ENVELOPE ESTIMATION (5 minutes)

### What to Say:

"Let me calculate the scale to validate our architecture decisions."

### User & File Estimation:

```
Given:
- Total users: 100M
- Active users (DAU): 20M (20% of total)
- Files per user: 1000 average
- Average file size: 10MB
- Max file size: 50GB

Calculations:

1. Total Files
   Total files: 100M users × 1000 files = 100B files
   
2. Total Storage (Naive)
   Without deduplication: 100B files × 10MB = 1000 PB = 1 EB
   
   With deduplication (assume 30% savings):
   Actual storage: 1 EB × 0.7 = 700 PB
   
   This is MASSIVE! Deduplication is critical ✓

3. Storage per User
   Average: 1000 files × 10MB = 10 GB per user
   Max (theoretical): 10 TB per user
```

### Traffic Estimation:

```
Given:
- DAU: 20M
- Uploads per user per day: 5 files
- Downloads per user per day: 10 files (read-heavy)
- Sync checks per user per day: 50 (every 30 min for 24 hours)

Calculations:

1. Upload Traffic
   Daily uploads: 20M users × 5 files = 100M uploads/day
   Average upload QPS: 100M / 86,400 = 1,157 uploads/sec
   Peak upload QPS (3x): 3,500 uploads/sec
   
   Bandwidth:
   Average: 1,157 uploads/sec × 10MB = 11.57 GB/sec
   Peak: 3,500 × 10MB = 35 GB/sec

2. Download Traffic
   Daily downloads: 20M users × 10 files = 200M downloads/day
   Average download QPS: 200M / 86,400 = 2,315 downloads/sec
   Peak download QPS (3x): 7,000 downloads/sec
   
   Bandwidth:
   Average: 2,315 × 10MB = 23 GB/sec
   Peak: 7,000 × 10MB = 70 GB/sec

3. Sync Check Traffic (Metadata Only)
   Daily sync checks: 20M users × 50 = 1B checks/day
   Average sync QPS: 1B / 86,400 = 11,574 checks/sec
   Peak sync QPS (3x): 35,000 checks/sec
   
   Bandwidth (metadata only, ~1KB per check):
   Average: 11,574 × 1KB = 11.5 MB/sec
   Peak: 35,000 × 1KB = 35 MB/sec
   
   This is lightweight ✓

4. Read:Write Ratio
   Downloads:Uploads = 2,315:1,157 = 2:1
   
   Note: This is lower than typical because we're counting
   file operations, not bytes. Sync checks dominate request
   volume but are metadata-only.
```

### Storage Breakdown:

```
1. File Data (Blob Storage)
   Total: 700 PB (with deduplication)
   Replication factor: 3
   Total with replication: 700 PB × 3 = 2.1 EB
   
2. File Metadata (Database)
   Total files: 100B
   Metadata per file: 1KB
   - file_id: 16 bytes
   - file_name: 256 bytes
   - file_size: 8 bytes
   - mime_type: 50 bytes
   - created_at: 8 bytes
   - modified_at: 8 bytes
   - owner_id: 16 bytes
   - parent_folder_id: 16 bytes
   - block_hashes: 500 bytes (for deduplication)
   - permissions: 100 bytes
   
   Total: 100B × 1KB = 100 TB
   
   This is manageable for distributed database ✓

3. Block Metadata (Deduplication Index)
   Assume 4MB blocks:
   Total blocks: 700 PB / 4MB = 175 billion blocks
   
   Metadata per block: 50 bytes
   - block_hash (SHA-256): 32 bytes
   - block_size: 8 bytes
   - reference_count: 8 bytes
   - storage_location: 100 bytes (S3 path)
   
   Total: 175B × 150 bytes = 26.25 TB
   
   This fits in distributed cache (Redis) ✓

4. User Metadata
   Users: 100M
   Metadata per user: 1KB
   Total: 100M × 1KB = 100 GB
   
   Negligible ✓
```

### Chunking Analysis (THE CRITICAL CALCULATION):

```
Scenario: User uploads 50GB file

Without Chunking:
- Upload time: 50GB / 100Mbps = 4000 seconds = 1.1 hours
- If connection drops at 99%: Lose 1.1 hours of work
- No progress indicator
- Timeout issues
- UNACCEPTABLE ✗

With Chunking (4MB blocks):
- Total blocks: 50GB / 4MB = 12,800 blocks
- Upload per block: 4MB / 100Mbps = 0.32 seconds
- If connection drops: Only lose current block (4MB)
- Resume from last successful block
- Progress: (blocks_uploaded / total_blocks) × 100%
- ACCEPTABLE ✓

Deduplication Benefit:
- User A uploads 50GB movie
- User B uploads same movie
- Without dedup: 100GB stored
- With dedup: 50GB stored (50% savings)
- User B's upload: Instant (hash check only)
```

### Summary Table:

```
┌─────────────────────────┬──────────────────┐
│ Metric                  │ Value            │
├─────────────────────────┼──────────────────┤
│ Total users             │ 100M             │
│ Daily active users      │ 20M              │
│ Total files             │ 100B             │
│ Total storage (dedup)   │ 700 PB           │
│ Upload QPS (peak)       │ 3,500            │
│ Download QPS (peak)     │ 7,000            │
│ Sync check QPS (peak)   │ 35,000           │
│ Upload bandwidth (peak) │ 35 GB/sec        │
│ Download bandwidth      │ 70 GB/sec        │
│ Metadata storage        │ 100 TB           │
│ Block index             │ 26 TB            │
│ Chunk size              │ 4 MB             │
│ Total blocks            │ 175B             │
│ Deduplication savings   │ 30%              │
└─────────────────────────┴──────────────────┘

Key Insights:
1. Chunking mandatory for 50GB files (1.1 hour upload)
2. Deduplication saves 30% storage (300 PB!)
3. Sync checks dominate request volume (35K QPS)
4. CDN critical for 70 GB/sec download bandwidth
5. Block metadata (26 TB) fits in distributed cache
```

-----------------------------------------------------------------

## PHASE 4: DATA MODELS / CORE ENTITIES (4 minutes)

### What to Say:

"I'll use PostgreSQL for file metadata (structured, relational), Redis for block index (fast lookups), and Cassandra as persistent fallback. The key is separating metadata from actual data."

### Database Selection:

```
┌──────────────────┬─────────────┬────────────────────────┐
│ Data Type        │ Database    │ Reason                 │
├──────────────────┼─────────────┼────────────────────────┤
│ File Metadata    │ PostgreSQL  │ ACID, relationships    │
│ Block Index      │ Redis       │ Sub-ms lookups         │
│ Block Metadata   │ Cassandra   │ Persistent fallback    │
│ Sync State       │ DynamoDB    │ High write throughput  │
│ File Blocks      │ S3          │ Object storage         │
└──────────────────┴─────────────┴────────────────────────┘
```

### 1. File Metadata (PostgreSQL):

```sql
-- Files table
CREATE TABLE files (
    file_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_name VARCHAR(255) NOT NULL,
    file_size BIGINT NOT NULL,
    mime_type VARCHAR(100),
    owner_id UUID NOT NULL REFERENCES users(user_id),
    parent_folder_id UUID REFERENCES folders(folder_id),
    status VARCHAR(20) NOT NULL, -- UPLOADING, COMPLETED, DELETED
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    modified_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP,
    version INT NOT NULL DEFAULT 1,
    
    INDEX idx_owner_parent (owner_id, parent_folder_id),
    INDEX idx_status (status),
    INDEX idx_modified (modified_at)
);

-- File blocks mapping (the "recipe")
CREATE TABLE file_blocks (
    file_id UUID NOT NULL REFERENCES files(file_id),
    block_hash CHAR(64) NOT NULL, -- SHA-256 hex
    block_order INT NOT NULL,
    block_size INT NOT NULL,
    
    PRIMARY KEY (file_id, block_order),
    INDEX idx_block_hash (block_hash)
);

-- Sharing/Permissions table
CREATE TABLE file_permissions (
    permission_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id UUID NOT NULL REFERENCES files(file_id),
    shared_with_user_id UUID NOT NULL REFERENCES users(user_id),
    permission_type VARCHAR(10) NOT NULL, -- VIEW, EDIT
    shared_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE (file_id, shared_with_user_id),
    INDEX idx_shared_with (shared_with_user_id)
);

-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    storage_used BIGINT DEFAULT 0,
    storage_quota BIGINT DEFAULT 10737418240, -- 10GB
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

Example Data:
file_id: 550e8400-e29b-41d4-a716-446655440000
file_name: "vacation_video.mp4"
file_size: 53687091200 (50GB)
owner_id: 123e4567-e89b-12d3-a456-426614174000
status: "COMPLETED"
created_at: 2024-01-15 10:30:00

file_blocks entries:
file_id: 550e8400-e29b-41d4-a716-446655440000
block_hash: "a3f5b8c9d2e1f4a7b6c5d8e9f1a2b3c4..."
block_order: 0
block_size: 4194304

block_hash: "b4g6c9d3f2e2a5b8c7d6e9f2a3b4c5d6..."
block_order: 1
block_size: 4194304
...
```

### 2. Block Index (Redis):

```
Data Structure: Hash

Key: block:{block_hash}
Value: JSON object

Example:
Key: "block:a3f5b8c9d2e1f4a7b6c5d8e9f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0"

Value: {
  "storage_location": "s3://dropbox-blocks/a3/f5/a3f5b8c9...",
  "block_size": 4194304,
  "reference_count": 1523,
  "created_at": 1705315800,
  "last_accessed": 1705318800
}

Redis Commands:
# Check if block exists
EXISTS block:a3f5b8c9d2e1...

# Get block metadata
HGETALL block:a3f5b8c9d2e1...

# Increment reference count (atomic)
HINCRBY block:a3f5b8c9d2e1... reference_count 1

# Decrement reference count
HINCRBY block:a3f5b8c9d2e1... reference_count -1

# Set expiry for cold blocks (LRU)
EXPIRE block:a3f5b8c9d2e1... 2592000  # 30 days

Memory Calculation:
- 175B blocks
- 150 bytes per entry (hash + metadata)
- Total: 175B × 150 bytes = 26.25 TB
- Distributed across 512 Redis shards
- ~50 GB per shard
```

### 3. Block Metadata (Cassandra - Persistent Fallback):

```sql
CREATE TABLE block_metadata (
    block_hash TEXT PRIMARY KEY,
    storage_location TEXT,
    block_size INT,
    reference_count COUNTER,
    created_at TIMESTAMP,
    last_accessed TIMESTAMP
);

-- Partition key: block_hash
-- No clustering keys needed

Why Cassandra?
✓ Write-optimized (LSM tree)
✓ Handles high write throughput
✓ Persistent storage (Redis fallback)
✓ Counter data type for reference_count
✓ Horizontal scaling

Query Pattern:
SELECT * FROM block_metadata WHERE block_hash = ?;
```


### 4. Sync State (DynamoDB):

```
Table: device_sync_state
Partition Key: user_id
Sort Key: device_id

Attributes:
- user_id: UUID
- device_id: UUID
- device_name: String
- last_sync_timestamp: Number (Unix timestamp)
- sync_cursor: String
- files_synced: Number
- last_sync_status: String (SUCCESS, FAILED, IN_PROGRESS)

Example:
{
  "user_id": "123e4567-e89b-12d3-a456-426614174000",
  "device_id": "device_abc123",
  "device_name": "MacBook Pro",
  "last_sync_timestamp": 1705318800,
  "sync_cursor": "1705318800:file_xyz789",
  "files_synced": 1523,
  "last_sync_status": "SUCCESS"
}

Query Patterns:
1. Get all devices for user:
   Query(user_id = ?)
   
2. Get specific device state:
   GetItem(user_id = ?, device_id = ?)
   
3. Update sync state:
   UpdateItem(user_id = ?, device_id = ?, 
              SET last_sync_timestamp = ?, sync_cursor = ?)
```

### 5. S3 Storage Structure:

```
Bucket: dropbox-blocks

Directory Structure:
/blocks/
  /{first_2_chars}/
    /{next_2_chars}/
      /{full_hash}

Example:
Block hash: a3f5b8c9d2e1f4a7b6c5d8e9f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0

S3 Path:
s3://dropbox-blocks/blocks/a3/f5/a3f5b8c9d2e1f4a7b6c5d8e9f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0

Why this structure?
✓ Avoids S3 hot partition problem
✓ Distributes load across S3 partitions
✓ Enables efficient prefix-based operations
✓ Supports lifecycle policies per prefix

Metadata:
- Content-Type: application/octet-stream
- Content-Length: 4194304
- x-amz-storage-class: STANDARD (hot) or GLACIER (cold)
- x-amz-server-side-encryption: AES256
```
----------------------------------------------------------------------------------
---

## PHASE 5: API DESIGN (3 minutes)

### What to Say:

"I'll define APIs for upload, download, and sync. The key innovation is the block existence check API that enables deduplication."

### API Endpoints:

**1. Initiate File Upload (Chunked)**
```
POST /v1/files/upload/init
Authorization: Bearer <token>

Request Body:
{
  "file_name": "movie.mp4",
  "file_size": 53687091200,  // 50GB
  "mime_type": "video/mp4",
  "parent_folder_id": "folder_123",
  "blocks": [
    {
      "block_hash": "a3f5b8c9d2e1...",  // SHA-256
      "block_size": 4194304,              // 4MB
      "block_order": 0
    },
    {
      "block_hash": "b4g6c9d3f2e2...",
      "block_size": 4194304,
      "block_order": 1
    },
    // ... 12,800 blocks total
  ]
}

Response: 200 OK
{
  "file_id": "file_xyz789",
  "upload_id": "multipart_abc123",  // S3 multipart upload ID
  "blocks_to_upload": [
    {
      "block_hash": "b4g6c9d3f2e2...",
      "block_order": 1,
      "presigned_url": "https://s3.amazonaws.com/...",
      "part_number": 2
    }
    // Only blocks that don't exist
  ],
  "blocks_already_exist": [
    {
      "block_hash": "a3f5b8c9d2e1...",
      "block_order": 0
    }
    // Blocks already in system (deduplication!)
  ]
}

Why This Design?
- Client sends ALL block hashes upfront
- Server checks which blocks already exist
- Client only uploads missing blocks
- Instant upload if all blocks exist (100% dedup)
```

**2. Upload Block**
```
PUT <presigned_url>
Content-Type: application/octet-stream
Body: <binary block data>

Response: 200 OK
Headers:
  ETag: "block_etag_value"

Note: This goes directly to S3, not our API servers
```

**3. Complete File Upload**
```
POST /v1/files/upload/complete
Authorization: Bearer <token>

Request Body:
{
  "file_id": "file_xyz789",
  "upload_id": "multipart_abc123",
  "blocks": [
    {
      "block_hash": "a3f5b8c9d2e1...",
      "block_order": 0,
      "etag": "etag_value_1"
    },
    {
      "block_hash": "b4g6c9d3f2e2...",
      "block_order": 1,
      "etag": "etag_value_2"
    }
    // All blocks with their ETags
  ]
}

Response: 200 OK
{
  "file_id": "file_xyz789",
  "status": "COMPLETED",
  "file_url": "https://cdn.dropbox.com/files/xyz789"
}

Server Actions:
1. Verify all blocks uploaded (check ETags with S3)
2. Complete S3 multipart upload
3. Update file metadata (status = COMPLETED)
4. Increment reference count for each block
5. Notify other devices via WebSocket
```

**4. Download File**
```
GET /v1/files/{file_id}/download
Authorization: Bearer <token>

Response: 200 OK
{
  "file_id": "file_xyz789",
  "file_name": "movie.mp4",
  "file_size": 53687091200,
  "download_url": "https://cdn.dropbox.com/files/xyz789?token=...",
  "expires_at": "2024-01-15T11:00:00Z",  // 5 min expiry
  "blocks": [
    {
      "block_hash": "a3f5b8c9d2e1...",
      "block_order": 0,
      "download_url": "https://cdn.dropbox.com/blocks/a3f5..."
    },
    {
      "block_hash": "b4g6c9d3f2e2...",
      "block_order": 1,
      "download_url": "https://cdn.dropbox.com/blocks/b4g6..."
    }
    // All blocks with CDN URLs
  ]
}

Client Actions:
1. Download blocks in parallel (10 concurrent)
2. Verify each block hash
3. Assemble blocks in order
4. Write to local file
```

**5. Get File Changes (Sync)**
```
GET /v1/sync/changes?since=<timestamp>&device_id=<device_id>
Authorization: Bearer <token>

Query Parameters:
- since: Unix timestamp of last sync
- device_id: Unique device identifier

Response: 200 OK
{
  "changes": [
    {
      "change_type": "CREATED",
      "file_id": "file_abc123",
      "file_name": "document.pdf",
      "file_size": 1048576,
      "modified_at": "2024-01-15T10:30:00Z",
      "parent_folder_id": "folder_123"
    },
    {
      "change_type": "MODIFIED",
      "file_id": "file_xyz789",
      "file_name": "movie.mp4",
      "modified_at": "2024-01-15T10:35:00Z",
      "changed_blocks": [
        {
          "block_hash": "new_hash_1...",
          "block_order": 0
        }
      ]
    },
    {
      "change_type": "DELETED",
      "file_id": "file_def456",
      "deleted_at": "2024-01-15T10:40:00Z"
    }
  ],
  "next_cursor": "1705318800",  // Timestamp for next sync
  "has_more": false
}

Client Actions:
1. Download new/modified files
2. Delete locally deleted files
3. Update sync cursor
4. Schedule next sync check
```

**6. Share File**
```
POST /v1/files/{file_id}/share
Authorization: Bearer <token>

Request Body:
{
  "shared_with": ["user2@example.com", "user3@example.com"],
  "permission": "VIEW"  // or "EDIT"
}

Response: 200 OK
{
  "share_id": "share_abc123",
  "shared_with": [
    {
      "user_id": "user_2",
      "email": "user2@example.com",
      "permission": "VIEW"
    }
  ]
}
```

**7. WebSocket Connection (Real-time Sync)**
```
WS /v1/sync/ws?token=<auth_token>&device_id=<device_id>

Server → Client Messages:
{
  "event": "FILE_CHANGED",
  "file_id": "file_xyz789",
  "change_type": "MODIFIED",
  "modified_at": "2024-01-15T10:30:00Z"
}

{
  "event": "FILE_DELETED",
  "file_id": "file_abc123",
  "deleted_at": "2024-01-15T10:35:00Z"
}

Client Actions:
- Receive push notifications
- Trigger immediate sync
- No polling needed
```

### Why This API Design?

```
✓ Block existence check enables deduplication
✓ Presigned URLs for direct S3 access
✓ Multipart upload for large files
✓ WebSocket for real-time push
✓ Cursor-based sync (efficient)
✓ Block-level download (resume support)
```



------------------------------------------------------


## PHASE 4: HIGH-LEVEL ARCHITECTURE (8 minutes)

### What to Say:

"I'll design a system with three key components: client sync agent, API servers, and storage layer. The critical insight is using block-level deduplication to avoid storing duplicate data."

### Complete Architecture Diagram:

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLIENT DEVICES                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │   Desktop    │  │    Laptop    │  │    Mobile    │         │
│  │              │  │              │  │              │         │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │         │
│  │ │  Sync    │ │  │ │  Sync    │ │  │ │  Sync    │ │         │
│  │ │  Agent   │ │  │ │  Agent   │ │  │ │  Agent   │ │         │
│  │ └────┬─────┘ │  │ └────┬─────┘ │  │ └────┬─────┘ │         │
│  │      │       │  │      │       │  │      │       │         │
│  │ ┌────▼─────┐ │  │ ┌────▼─────┐ │  │ ┌────▼─────┐ │         │
│  │ │  Local   │ │  │ │  Local   │ │  │ │  Local   │ │         │
│  │ │  Dropbox │ │  │ │  Dropbox │ │  │ │  Dropbox │ │         │
│  │ │  Folder  │ │  │ │  Folder  │  │ │  Folder  │ │         │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
          │ HTTPS            │                  │
          └──────────────────┼──────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CDN (CloudFront)                              │
│  - Serves file downloads                                        │
│  - Caches popular files                                         │
│  - Reduces origin load by 80%                                   │
│  - Global edge locations                                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              LOAD BALANCER (Layer 7)                             │
│  - SSL termination                                               │
│  - Rate limiting                                                 │
│  - Health checks                                                 │
│  - Route to API Gateway                                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    API GATEWAY LAYER                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ API Gateway  │  │ API Gateway  │  │ API Gateway  │         │
│  │   Node 1     │  │   Node 2     │  │   Node N     │         │
│  │              │  │              │  │              │         │
│  │ - Auth       │  │ - Auth       │  │ - Auth       │         │
│  │ - Validation │  │ - Validation │  │ - Validation │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
└─────────┼──────────────────┼──────────────────┼─────────────────┘
          │                  │                  │
    ┌─────┴─────┬────────────┴────────┬─────────┴──────┐
    │           │                     │                │
    ▼           ▼                     ▼                ▼

┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ File Service │  │ Sync Service │  │ Block Service│  │ Share Service│
└──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘

═══════════════════════════════════════════════════════════════════
                        FILE SERVICE DETAIL
═══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                    FILE SERVICE                                  │
│  - Handles file metadata operations                             │
│  - Generates presigned URLs                                     │
│  - Manages file permissions                                     │
│  - Coordinates with Block Service                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│              FILE METADATA DB (PostgreSQL)                       │
│  Tables:                                                        │
│  - files (id, name, size, owner_id, parent_folder_id)          │
│  - file_blocks (file_id, block_hash, block_order)              │
│  - folders (id, name, owner_id, parent_id)                     │
│  - permissions (file_id, user_id, permission_type)             │
│  - shared_files (file_id, shared_with_user_id)                 │
└─────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
                        BLOCK SERVICE DETAIL
═══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                    BLOCK SERVICE                                 │
│  - Handles block-level deduplication                            │
│  - Checks if blocks already exist                               │
│  - Manages block reference counting                             │
│  - Coordinates multipart uploads to S3                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│              BLOCK INDEX (Redis Cluster)                         │
│  Key: block_hash (SHA-256)                                      │
│  Value: {                                                       │
│    storage_location: "s3://bucket/path",                        │
│    block_size: 4194304,                                         │
│    reference_count: 1523,                                       │
│    created_at: timestamp                                        │
│  }                                                              │
│  - 26 TB total                                                  │
│  - Distributed across 512 shards                                │
│  - LRU eviction for cold blocks                                 │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          │ Fallback
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│              BLOCK METADATA DB (Cassandra)                       │
│  - Persistent storage for block index                           │
│  - Partition key: block_hash                                    │
│  - Handles cache misses                                         │
└─────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
                        SYNC SERVICE DETAIL
═══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│                    SYNC SERVICE                                  │
│  - Handles sync requests from clients                           │
│  - Maintains WebSocket connections                              │
│  - Pushes change notifications                                  │
│  - Manages sync state                                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────┐
│              NOTIFICATION SERVICE (Redis Pub/Sub)                │
│  - Publishes file change events                                │
│  - Subscribers: Connected sync clients                          │
│  - Channel per user: "user:{user_id}:changes"                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              SYNC STATE DB (DynamoDB)                            │
│  - Tracks last sync timestamp per device                        │
│  - Partition key: user_id                                       │
│  - Sort key: device_id                                          │
│  - Attributes: last_sync_time, sync_cursor                      │
└─────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
                        STORAGE LAYER
═══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│              OBJECT STORAGE (S3)                                 │
│  - Stores actual file blocks (4MB chunks)                       │
│  - 700 PB total (with deduplication)                            │
│  - Replication factor: 3 (built-in)                             │
│  - Lifecycle policies:                                          │
│    * Hot: Standard S3 (accessed in last 30 days)                │
│    * Warm: S3 Infrequent Access (30-90 days)                    │
│    * Cold: S3 Glacier (> 90 days)                               │
│  - Multipart upload support                                     │
│  - Presigned URLs for direct client access                      │
└─────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
                        SUPPORTING SERVICES
═══════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│              GARBAGE COLLECTION SERVICE                          │
│  - Runs periodically (daily)                                    │
│  - Identifies blocks with reference_count = 0                   │
│  - Deletes orphaned blocks from S3                              │
│  - Frees up storage space                                       │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              COMPRESSION SERVICE                                 │
│  - Client-side compression before upload                        │
│  - Algorithms: Gzip, Brotli, Zstandard                          │
│  - Smart decision based on file type                            │
│  - Text files: High compression ratio                           │
│  - Media files: Skip (already compressed)                       │
└─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities:

**1. Sync Agent (Client)**
```
- Monitor local folder for changes (FileSystemWatcher)
- Chunk files into 4MB blocks
- Calculate SHA-256 hash for each block
- Check with server which blocks already exist
- Upload only new/changed blocks
- Download remote changes
- Maintain WebSocket connection for push notifications
- Handle conflict resolution (last-write-wins)
```

**2. File Service**
```
- CRUD operations on file metadata
- Generate presigned URLs for S3
- Manage file permissions and sharing
- Coordinate with Block Service for uploads
- Handle folder operations
```

**3. Block Service**
```
- Implement block-level deduplication
- Check if block hash exists in index
- Increment/decrement reference counts
- Initiate S3 multipart uploads
- Return presigned URLs for block uploads
```

**4. Sync Service**
```
- Maintain WebSocket connections with clients
- Push change notifications in real-time
- Handle sync requests (get changes since timestamp)
- Manage sync state per device
```

**5. Share Service**
```
- Handle file/folder sharing
- Manage permissions (view/edit)
- Generate share links
- Enforce access control
```


### Data Flow Summary:

```
Upload Flow:
  Client → File Service → PostgreSQL (file metadata)
                       → Block Service → Redis (block index check)
                                      → S3 (block upload)
                                      → Redis (update ref count)
                                      → Cassandra (persist)

Download Flow:
  Client → File Service → PostgreSQL (get file_blocks)
                       → Redis (get block locations)
                       → CDN/S3 (download blocks)

Sync Flow:
  Client → Sync Service → DynamoDB (get last sync state)
                       → PostgreSQL (get changed files)
                       → DynamoDB (update sync state)
```

---

## PHASE 7: CORE FLOWS - UPLOAD/DOWNLOAD/SYNC (8 minutes)

### What to Say:

"Let me walk through the three core flows: upload with deduplication, download with CDN, and bidirectional sync. The key innovation is the block existence check that enables instant uploads."

### Flow 1: File Upload with Block-Level Deduplication

**Scenario:** User uploads 50GB movie file "Batman.mp4"

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Client-Side Chunking & Hashing                          │
└─────────────────────────────────────────────────────────────────┘

Sync Agent (Client):
  1. Detect file added to Dropbox folder
     Event: FileSystemWatcher triggers on "Batman.mp4"
  
  2. Read file and chunk into 4MB blocks
     Total size: 50GB = 51,200 MB
     Block size: 4MB
     Total blocks: 51,200 / 4 = 12,800 blocks
     
  3. Calculate SHA-256 hash for each block
     Block 0: hash("first 4MB") = "a3f5b8c9d2e1..."
     Block 1: hash("next 4MB") = "b4g6c9d3f2e2..."
     ...
     Block 12,799: hash("last 4MB") = "z9y8x7w6v5u4..."
     
     Time: ~30 seconds (parallel hashing on 8 cores)
  
  4. Calculate file fingerprint (hash of all block hashes)
     file_hash = hash(concat(all_block_hashes))
     This becomes the file_id

┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Initiate Upload (The Handshake)                         │
└─────────────────────────────────────────────────────────────────┘

Client → File Service:
  POST /v1/files/upload/init
  Body: {
    file_name: "Batman.mp4",
    file_size: 53687091200,
    blocks: [
      {block_hash: "a3f5b8c9...", block_order: 0, block_size: 4194304},
      {block_hash: "b4g6c9d3...", block_order: 1, block_size: 4194304},
      ...
      // All 12,800 block hashes
    ]
  }

File Service:
  1. Create file metadata entry
     INSERT INTO files (file_id, file_name, file_size, owner_id, status)
     VALUES (file_hash, "Batman.mp4", 53687091200, user_id, "UPLOADING")
     
  2. Forward to Block Service for deduplication check

Block Service:
  1. Check Redis for each block hash (parallel, 100 at a time)
     Pipeline:
       EXISTS block:a3f5b8c9...
       EXISTS block:b4g6c9d3...
       ...
     
     Time: 12,800 checks / 100 per batch × 1ms = 128ms
  
  2. Categorize blocks:
     blocks_exist = []      // Already in system
     blocks_missing = []    // Need to upload
     
     Result (First Upload):
       blocks_exist: 0 blocks
       blocks_missing: 12,800 blocks (all)
     
     Result (Second Upload - Same File):
       blocks_exist: 12,800 blocks (all!)
       blocks_missing: 0 blocks
  
  3. For missing blocks:
     - Initiate S3 multipart upload
     - Generate presigned URLs for each part
     
     S3 API:
       upload_id = s3.create_multipart_upload(
         bucket="dropbox-blocks",
         key="blocks/a3/f5/a3f5b8c9..."
       )
       
       presigned_urls = []
       for i, block in enumerate(blocks_missing):
         url = s3.generate_presigned_url(
           'upload_part',
           Params={
             'Bucket': 'dropbox-blocks',
             'Key': block.storage_path,
             'UploadId': upload_id,
             'PartNumber': i + 1
           },
           ExpiresIn: 3600  # 1 hour
         )
         presigned_urls.append(url)

Response to Client:
  {
    file_id: "file_xyz789",
    upload_id: "multipart_abc123",
    blocks_to_upload: [
      {
        block_hash: "a3f5b8c9...",
        block_order: 0,
        presigned_url: "https://s3.amazonaws.com/...",
        part_number: 1
      },
      // ... 12,800 URLs (first upload)
      // OR 0 URLs (duplicate file - instant!)
    ],
    blocks_already_exist: [
      // Empty for first upload
      // OR all 12,800 blocks for duplicate
    ]
  }

┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Upload Blocks (Parallel)                                │
└─────────────────────────────────────────────────────────────────┘

Client (First Upload):
  1. Upload blocks in parallel (10 concurrent)
     for each block in blocks_to_upload:
       PUT presigned_url
       Body: <4MB binary data>
       
     Progress: (uploaded_blocks / total_blocks) × 100%
     
     Time: 50GB / 100Mbps = 4000 seconds = 1.1 hours
     
     With 10 parallel uploads:
     Time: 1.1 hours / 10 = 6.6 minutes (network bound)
  
  2. Collect ETags from S3 responses
     etags = [
       {part_number: 1, etag: "etag_1"},
       {part_number: 2, etag: "etag_2"},
       ...
     ]

Client (Duplicate Upload):
  Skip! No blocks to upload.
  Time: 0 seconds (instant!)

┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Complete Upload                                         │
└─────────────────────────────────────────────────────────────────┘

Client → File Service:
  POST /v1/files/upload/complete
  Body: {
    file_id: "file_xyz789",
    upload_id: "multipart_abc123",
    blocks: [
      {block_hash: "a3f5b8c9...", block_order: 0, etag: "etag_1"},
      ...
    ]
  }

File Service:
  1. Verify all blocks uploaded
     For each block:
       - Check ETag matches S3
       - Verify block exists in S3
  
  2. Complete S3 multipart upload
     s3.complete_multipart_upload(
       bucket="dropbox-blocks",
       key=file_path,
       upload_id=upload_id,
       parts=etags
     )
  
  3. Update file metadata
     UPDATE files 
     SET status = "COMPLETED", modified_at = NOW()
     WHERE file_id = "file_xyz789"
     
     INSERT INTO file_blocks (file_id, block_hash, block_order, block_size)
     VALUES 
       ("file_xyz789", "a3f5b8c9...", 0, 4194304),
       ("file_xyz789", "b4g6c9d3...", 1, 4194304),
       ...
  
  4. Update block reference counts (atomic)
     For each block:
       Redis: HINCRBY block:a3f5b8c9... reference_count 1
       Cassandra: UPDATE block_metadata 
                  SET reference_count = reference_count + 1
                  WHERE block_hash = "a3f5b8c9..."
  
  5. Notify other devices via WebSocket
     Publish to Redis Pub/Sub:
       PUBLISH user:123:changes {
         event: "FILE_CREATED",
         file_id: "file_xyz789",
         file_name: "Batman.mp4",
         modified_at: timestamp
       }

Response to Client:
  {
    file_id: "file_xyz789",
    status: "COMPLETED",
    file_url: "https://cdn.dropbox.com/files/xyz789"
  }

Total Time:
  First upload: 6.6 minutes (network bound)
  Duplicate upload: < 1 second (instant!)
```

---

### Flow 2: File Download with CDN

**Scenario:** User downloads 50GB movie on different device

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Request Download                                        │
└─────────────────────────────────────────────────────────────────┘

Client → File Service:
  GET /v1/files/file_xyz789/download

File Service:
  1. Verify permissions
     SELECT * FROM file_permissions
     WHERE file_id = "file_xyz789" 
     AND (owner_id = user_id OR shared_with_user_id = user_id)
  
  2. Get file blocks
     SELECT block_hash, block_order, block_size
     FROM file_blocks
     WHERE file_id = "file_xyz789"
     ORDER BY block_order
     
     Result: 12,800 blocks
  
  3. Get block locations from Redis
     Pipeline:
       HGET block:a3f5b8c9... storage_location
       HGET block:b4g6c9d3... storage_location
       ...
     
     Time: 128ms (same as upload check)
  
  4. Generate CDN URLs (signed)
     For each block:
       cdn_url = generate_signed_url(
         domain="cdn.dropbox.com",
         path="/blocks/a3/f5/a3f5b8c9...",
         expires=3600,
         signature=hmac_sha256(secret, path + expires)
       )

Response to Client:
  {
    file_id: "file_xyz789",
    file_name: "Batman.mp4",
    file_size: 53687091200,
    blocks: [
      {
        block_hash: "a3f5b8c9...",
        block_order: 0,
        download_url: "https://cdn.dropbox.com/blocks/a3/f5/a3f5b8c9...?sig=...",
        block_size: 4194304
      },
      // ... 12,800 blocks
    ]
  }

┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Download Blocks (Parallel)                              │
└─────────────────────────────────────────────────────────────────┘

Client:
  1. Download blocks in parallel (10 concurrent)
     for each block in blocks:
       GET block.download_url
       
       CDN Flow:
         a. CDN checks cache
            - Hit (80% of time): Serve from edge
            - Miss: Fetch from S3, cache, serve
         
         b. Return 4MB block
     
     Progress: (downloaded_blocks / total_blocks) × 100%
  
  2. Verify each block hash
     downloaded_hash = sha256(block_data)
     if downloaded_hash != expected_hash:
       retry_download(block)
  
  3. Assemble blocks in order
     Write blocks to local file sequentially
     
     file = open("Batman.mp4", "wb")
     for block in sorted_blocks:
       file.write(block.data)
     file.close()

Total Time:
  With CDN (80% hit rate):
    - 80% from CDN edge: ~50ms per block
    - 20% from S3: ~200ms per block
    - Average: 0.8 × 50ms + 0.2 × 200ms = 80ms per block
    - Total: 12,800 blocks × 80ms / 10 parallel = 102 seconds = 1.7 minutes
  
  Without CDN (all from S3):
    - 12,800 blocks × 200ms / 10 parallel = 256 seconds = 4.3 minutes
  
  CDN saves 2.6 minutes (60% faster!)
```

---

### Flow 3: Bidirectional Sync

**Scenario:** User edits file on Device A, sync to Device B

```
┌─────────────────────────────────────────────────────────────────┐
│ Local → Remote Sync (Device A)                                  │
└─────────────────────────────────────────────────────────────────┘

Device A (Sync Agent):
  1. Detect local file change
     FileSystemWatcher event: "Batman.mp4" modified
  
  2. Re-chunk and re-hash file
     - Only first 10 blocks changed (user edited intro)
     - Blocks 11-12,800 unchanged
     
     New hashes:
       Block 0: "new_hash_0..." (changed)
       Block 1: "new_hash_1..." (changed)
       ...
       Block 9: "new_hash_9..." (changed)
       Block 10: "a3f5b8c9..." (same!)
       Block 11: "b4g6c9d3..." (same!)
       ...
  
  3. Initiate upload (same as Flow 1)
     POST /v1/files/upload/init
     
     Block Service checks:
       - Blocks 0-9: Missing (need upload)
       - Blocks 10-12,799: Exist (skip!)
     
     Result: Only upload 10 blocks (40MB)
     Time: 40MB / 100Mbps = 3.2 seconds
     
     This is DELTA SYNC! ✓
  
  4. Complete upload
     - Update file_blocks table (replace first 10 entries)
     - Increment ref count for new blocks
     - Decrement ref count for old blocks
     - Update file modified_at timestamp
  
  5. Publish change notification
     Redis Pub/Sub:
       PUBLISH user:123:changes {
         event: "FILE_MODIFIED",
         file_id: "file_xyz789",
         modified_at: timestamp,
         changed_blocks: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
       }

┌─────────────────────────────────────────────────────────────────┐
│ Remote → Local Sync (Device B)                                  │
└─────────────────────────────────────────────────────────────────┘

Device B (Sync Agent):
  1. Receive WebSocket notification
     WebSocket message:
       {
         event: "FILE_MODIFIED",
         file_id: "file_xyz789",
         modified_at: timestamp
       }
  
  2. Fetch file changes
     GET /v1/sync/changes?since=<last_sync_time>
     
     Response:
       {
         changes: [
           {
             change_type: "MODIFIED",
             file_id: "file_xyz789",
             file_name: "Batman.mp4",
             modified_at: timestamp,
             changed_blocks: [
               {block_hash: "new_hash_0...", block_order: 0},
               {block_hash: "new_hash_1...", block_order: 1},
               ...
               {block_hash: "new_hash_9...", block_order: 9}
             ]
           }
         ]
       }
  
  3. Download only changed blocks
     for each changed_block:
       GET block.download_url
       
     Download: 10 blocks × 4MB = 40MB
     Time: 3.2 seconds
  
  4. Update local file (in-place)
     file = open("Batman.mp4", "r+b")
     for block in changed_blocks:
       file.seek(block.block_order × 4MB)
       file.write(block.data)
     file.close()
  
  5. Update sync state
     UPDATE device_sync_state
     SET last_sync_timestamp = NOW(),
         sync_cursor = "timestamp:file_xyz789"
     WHERE user_id = user_id AND device_id = device_b

Total Sync Time:
  Upload (Device A): 3.2 seconds (40MB)
  Notification: < 1 second (WebSocket)
  Download (Device B): 3.2 seconds (40MB)
  
  Total: ~7 seconds (near real-time!)
  
  Without delta sync: Would need to upload/download entire 50GB
  Time saved: 1.1 hours - 7 seconds = 99.8% faster!
```

### Conflict Resolution:

```
Scenario: Both devices edit same file offline

Device A:
  - Edits "Batman.mp4" at 10:00 AM
  - Goes offline
  - Comes online at 11:00 AM
  - Uploads changes

Device B:
  - Edits "Batman.mp4" at 10:30 AM
  - Uploads changes immediately

Conflict Detection:
  Device A tries to upload at 11:00 AM
  Server checks: file.modified_at = 10:30 AM (Device B's edit)
  Device A's base version: 10:00 AM
  
  Conflict! Device A's changes are based on stale version

Resolution (Last-Write-Wins):
  1. Accept Device A's upload (11:00 AM > 10:30 AM)
  2. Save Device B's version as conflict copy
     - Rename: "Batman (Device B's conflicted copy 2024-01-15).mp4"
  3. Notify both devices
  4. User manually resolves conflict

Alternative (Version History):
  - Keep both versions
  - User can restore either version
  - Requires versioning system (out of scope)
```



---

## PHASE 8: DEEP DIVE - BLOCK-LEVEL DEDUPLICATION (5 minutes)

### What to Say:

"Let me explain how block-level deduplication works in detail. This is the core innovation that saves 300 PB of storage and enables instant uploads."

### The Deduplication Algorithm:

**Step-by-Step Process:**

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Chunking Strategy                                             │
└─────────────────────────────────────────────────────────────────┘

Why 4MB blocks?

Too Small (e.g., 1KB):
  - 50GB file = 52,428,800 blocks
  - Metadata explosion (52M hashes to check)
  - Database overwhelmed
  - Network overhead (52M HTTP requests)
  ✗ Unworkable

Too Large (e.g., 100MB):
  - 50GB file = 512 blocks
  - Low deduplication ratio
  - If user changes 1 byte, entire 100MB re-uploaded
  - Poor delta sync efficiency
  ✗ Inefficient

Sweet Spot (4MB):
  - 50GB file = 12,800 blocks
  - Manageable metadata (12,800 hashes)
  - Good deduplication ratio
  - Efficient delta sync
  - Industry standard (Dropbox, Google Drive use 4MB)
  ✓ Optimal

┌─────────────────────────────────────────────────────────────────┐
│ 2. Hash Function Selection                                      │
└─────────────────────────────────────────────────────────────────┘

Why SHA-256?

Requirements:
  - Deterministic (same input → same output)
  - Collision-resistant (different inputs → different outputs)
  - Fast to compute
  - Fixed output size

Options:
  MD5:
    - Fast (500 MB/sec)
    - 128-bit output
    - Known collisions ✗
    - Not secure
  
  SHA-1:
    - Medium speed (400 MB/sec)
    - 160-bit output
    - Collision attacks exist ✗
    - Deprecated
  
  SHA-256:
    - Fast enough (300 MB/sec)
    - 256-bit output (64 hex chars)
    - No known collisions ✓
    - Cryptographically secure ✓
    - Industry standard ✓

Collision Probability:
  SHA-256 has 2^256 possible outputs
  = 115,792,089,237,316,195,423,570,985,008,687,907,853,269,984,665,640,564,039,457,584,007,913,129,639,936
  
  With 175 billion blocks:
  Collision probability ≈ 0 (practically impossible)

┌─────────────────────────────────────────────────────────────────┐
│ 3. Deduplication Scenarios                                      │
└─────────────────────────────────────────────────────────────────┘

Scenario A: Identical Files (100% Deduplication)
  User A uploads "Batman.mp4" (50GB)
  User B uploads same "Batman.mp4"
  
  Result:
    - All 12,800 blocks already exist
    - User B uploads 0 bytes
    - Storage used: 50GB (not 100GB)
    - Savings: 50GB (50%)
    - Upload time: < 1 second (instant!)

Scenario B: Similar Files (Partial Deduplication)
  User A uploads "Batman_1080p.mp4" (50GB)
  User B uploads "Batman_4K.mp4" (80GB)
  
  Same movie, different quality:
    - Audio tracks identical (20% of file)
    - Video different (80% of file)
  
  Result:
    - 20% blocks match (audio)
    - 80% blocks new (video)
    - User B uploads 64GB (80% of 80GB)
    - Storage used: 50GB + 64GB = 114GB (not 130GB)
    - Savings: 16GB (12%)

Scenario C: Modified File (Delta Sync)
  User edits first 5 minutes of "Batman.mp4"
  
  5 minutes of 4K video ≈ 2GB
  Changed blocks: 2GB / 4MB = 512 blocks
  
  Result:
    - 512 blocks new
    - 12,288 blocks unchanged
    - Upload: 2GB (not 50GB)
    - Savings: 48GB (96%)
    - Time: 16 seconds (not 1.1 hours)

┌─────────────────────────────────────────────────────────────────┐
│ 4. Reference Counting & Garbage Collection                      │
└─────────────────────────────────────────────────────────────────┘

The Problem:
  Block "a3f5b8c9..." is used by:
    - User A's "Batman.mp4"
    - User B's "Batman.mp4"
    - User C's "Movie_Collection.zip"
  
  If User A deletes their file:
    - Can we delete the block? NO!
    - User B and C still need it

Solution: Reference Counting

Block Metadata:
  {
    block_hash: "a3f5b8c9...",
    storage_location: "s3://...",
    reference_count: 3,  // Used by 3 files
    created_at: timestamp
  }

Operations:
  File Upload:
    For each block:
      HINCRBY block:a3f5b8c9... reference_count 1
  
  File Delete:
    For each block:
      HINCRBY block:a3f5b8c9... reference_count -1
      
      if reference_count == 0:
        Mark block for garbage collection

Garbage Collection Process:
  Daily job runs at 2 AM:
  
  1. Scan Redis for blocks with reference_count = 0
     SCAN 0 MATCH block:* COUNT 1000
     
     For each block:
       if HGET block:hash reference_count == 0:
         orphaned_blocks.add(block)
  
  2. Verify with Cassandra (double-check)
     SELECT reference_count 
     FROM block_metadata 
     WHERE block_hash IN (orphaned_blocks)
  
  3. Delete from S3
     for block in orphaned_blocks:
       s3.delete_object(
         Bucket="dropbox-blocks",
         Key=block.storage_location
       )
  
  4. Delete from Redis and Cassandra
     DEL block:hash
     DELETE FROM block_metadata WHERE block_hash = ?
  
  5. Update metrics
     storage_freed += block_size
     blocks_deleted += 1

Safety Measures:
  - Grace period: Wait 7 days before deleting
  - Backup: Keep block metadata for 30 days
  - Verification: Check reference_count in both Redis and Cassandra
  - Audit log: Record all deletions

┌─────────────────────────────────────────────────────────────────┐
│ 5. Deduplication Ratio Analysis                                 │
└─────────────────────────────────────────────────────────────────┘

Real-World Data (Industry Average):

File Type Distribution:
  - Documents (PDF, DOCX): 30% of files
    Dedup ratio: 40% (templates, forms)
  
  - Images (JPG, PNG): 25% of files
    Dedup ratio: 20% (photos shared)
  
  - Videos (MP4, AVI): 15% of files
    Dedup ratio: 50% (movies, shows)
  
  - Archives (ZIP, RAR): 10% of files
    Dedup ratio: 60% (software, backups)
  
  - Code (JS, PY, JAVA): 10% of files
    Dedup ratio: 70% (libraries, dependencies)
  
  - Other: 10% of files
    Dedup ratio: 30%

Weighted Average:
  Overall dedup ratio = 
    0.30 × 0.40 + 0.25 × 0.20 + 0.15 × 0.50 + 
    0.10 × 0.60 + 0.10 × 0.70 + 0.10 × 0.30
  = 0.12 + 0.05 + 0.075 + 0.06 + 0.07 + 0.03
  = 0.415 ≈ 40%

But wait! Cross-user deduplication adds more:
  - Popular files (movies, software): +10%
  - Shared folders: +5%
  - Backups: +5%

Total deduplication: 40% + 20% = 60%

Conservative estimate: 30% (accounting for unique files)

Storage Savings:
  Without dedup: 1 EB
  With 30% dedup: 700 PB
  Savings: 300 PB
  
  Cost savings (at $0.023/GB/month for S3):
    300 PB = 300,000,000 GB
    Monthly: 300M × $0.023 = $6.9M
    Annual: $6.9M × 12 = $82.8M
  
  Deduplication saves $82.8M per year! ✓
```

### Implementation Details:

```java
class BlockDeduplicationService {
    private RedisClient redis;
    private S3Client s3;
    private static final int BLOCK_SIZE = 4 * 1024 * 1024; // 4MB
    
    public UploadPlan checkBlockExistence(List<BlockHash> blockHashes) {
        List<BlockHash> blocksToUpload = new ArrayList<>();
        List<BlockHash> blocksExist = new ArrayList<>();
        
        // Batch check in Redis (100 at a time)
        for (int i = 0; i < blockHashes.size(); i += 100) {
            List<BlockHash> batch = blockHashes.subList(
                i, 
                Math.min(i + 100, blockHashes.size())
            );
            
            // Pipeline for efficiency
            Pipeline pipeline = redis.pipelined();
            Map<BlockHash, Response<Boolean>> responses = new HashMap<>();
            
            for (BlockHash hash : batch) {
                responses.put(hash, pipeline.exists("block:" + hash));
            }
            
            pipeline.sync();
            
            // Categorize blocks
            for (Map.Entry<BlockHash, Response<Boolean>> entry : responses.entrySet()) {
                if (entry.getValue().get()) {
                    blocksExist.add(entry.getKey());
                } else {
                    blocksToUpload.add(entry.getKey());
                }
            }
        }
        
        return new UploadPlan(blocksToUpload, blocksExist);
    }
    
    public void incrementReferenceCount(BlockHash blockHash) {
        // Atomic increment in Redis
        redis.hincrBy("block:" + blockHash, "reference_count", 1);
        
        // Persist to Cassandra (async)
        cassandra.executeAsync(
            "UPDATE block_metadata SET reference_count = reference_count + 1 " +
            "WHERE block_hash = ?",
            blockHash
        );
    }
    
    public void decrementReferenceCount(BlockHash blockHash) {
        long newCount = redis.hincrBy("block:" + blockHash, "reference_count", -1);
        
        if (newCount == 0) {
            // Mark for garbage collection
            redis.sadd("gc:orphaned_blocks", blockHash);
        }
        
        // Persist to Cassandra
        cassandra.executeAsync(
            "UPDATE block_metadata SET reference_count = reference_count - 1 " +
            "WHERE block_hash = ?",
            blockHash
        );
    }
}
```

### Why Block-Level Deduplication Wins:

```
┌──────────────────────┬──────────────┬──────────────────┐
│ Approach             │ Storage      │ Upload Time      │
├──────────────────────┼──────────────┼──────────────────┤
│ No Deduplication     │ 1 EB         │ 1.1 hours        │
│ File-Level Dedup     │ 900 PB       │ 1.1 hours        │
│ Block-Level Dedup    │ 700 PB       │ < 1 second       │
└──────────────────────┴──────────────┴──────────────────┘

Key Advantages:
✓ 30% storage savings (300 PB = $82.8M/year)
✓ Instant uploads for duplicate files
✓ Efficient delta sync (only changed blocks)
✓ Cross-user deduplication
✓ Bandwidth savings (don't upload existing blocks)
```



---

## PHASE 9: DEEP DIVE - LARGE FILE HANDLING & CHUNKING (4 minutes)

### What to Say:

"Let me explain how we handle 50GB files with resumable uploads, progress tracking, and multipart upload coordination."

### The Large File Problem:

```
Challenge: Upload 50GB file via single HTTP POST

Problems:
1. Timeouts
   - Upload time: 50GB / 100Mbps = 1.1 hours
   - Most servers timeout after 30 minutes
   - Request fails, user loses 30 minutes of work

2. No Progress Indicator
   - User sees spinning wheel for 1.1 hours
   - No idea if upload is working
   - Poor user experience

3. Network Interruptions
   - Connection drops at 99% complete
   - Must restart from 0%
   - Extremely frustrating

4. Browser/Server Limits
   - API Gateway: 10MB max payload
   - Nginx: 1GB default limit
   - Cannot upload 50GB in single request

Solution: Chunking + Multipart Upload
```

### Multipart Upload Flow:

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Client-Side Chunking                                    │
└─────────────────────────────────────────────────────────────────┘

Client (Sync Agent):
  function chunkFile(file) {
    const CHUNK_SIZE = 4 * 1024 * 1024; // 4MB
    const chunks = [];
    
    for (let offset = 0; offset < file.size; offset += CHUNK_SIZE) {
      const chunk = file.slice(offset, offset + CHUNK_SIZE);
      const hash = await sha256(chunk);
      
      chunks.push({
        data: chunk,
        hash: hash,
        order: chunks.length,
        size: chunk.size
      });
    }
    
    return chunks;
  }

Result:
  50GB file → 12,800 chunks of 4MB each

┌─────────────────────────────────────────────────────────────────┐
│ Step 2: Initiate Multipart Upload                               │
└─────────────────────────────────────────────────────────────────┘

Client → Server:
  POST /v1/files/upload/init
  Body: {
    file_name: "large_file.zip",
    file_size: 53687091200,
    blocks: [/* 12,800 block hashes */]
  }

Server (Block Service):
  1. Check which blocks exist (deduplication)
     blocks_to_upload = checkBlockExistence(block_hashes)
     
  2. For missing blocks, initiate S3 multipart upload
     upload_id = s3.createMultipartUpload({
       Bucket: "dropbox-blocks",
       Key: "blocks/a3/f5/a3f5b8c9...",
       ServerSideEncryption: "AES256"
     })
     
  3. Generate presigned URLs for each part
     presigned_urls = []
     for (i = 0; i < blocks_to_upload.length; i++) {
       url = s3.generatePresignedUrl('uploadPart', {
         Bucket: "dropbox-blocks",
         Key: block.key,
         UploadId: upload_id,
         PartNumber: i + 1,
         Expires: 3600  // 1 hour
       })
       presigned_urls.push(url)
     }

Response to Client:
  {
    upload_id: "multipart_abc123",
    blocks_to_upload: [
      {
        block_hash: "a3f5b8c9...",
        part_number: 1,
        presigned_url: "https://s3.amazonaws.com/..."
      },
      // ... only missing blocks
    ]
  }

┌─────────────────────────────────────────────────────────────────┐
│ Step 3: Upload Parts with Progress Tracking                     │
└─────────────────────────────────────────────────────────────────┘

Client:
  class ChunkedUploader {
    async uploadFile(file, uploadPlan) {
      const totalBlocks = uploadPlan.blocks_to_upload.length;
      let uploadedBlocks = 0;
      const etags = [];
      
      // Upload 10 blocks in parallel
      const concurrency = 10;
      const queue = [...uploadPlan.blocks_to_upload];
      
      while (queue.length > 0) {
        const batch = queue.splice(0, concurrency);
        
        const promises = batch.map(async (block) => {
          try {
            // Upload block to S3
            const response = await fetch(block.presigned_url, {
              method: 'PUT',
              body: block.data,
              headers: {
                'Content-Type': 'application/octet-stream'
              }
            });
            
            // Get ETag from response
            const etag = response.headers.get('ETag');
            
            // Update progress
            uploadedBlocks++;
            const progress = (uploadedBlocks / totalBlocks) * 100;
            this.updateProgressBar(progress);
            
            // Save state for resume
            this.saveUploadState({
              upload_id: uploadPlan.upload_id,
              completed_parts: uploadedBlocks,
              etags: etags
            });
            
            return {
              part_number: block.part_number,
              etag: etag
            };
            
          } catch (error) {
            // Retry failed upload
            console.error(`Failed to upload block ${block.part_number}:`, error);
            queue.push(block); // Re-queue for retry
          }
        });
        
        const results = await Promise.all(promises);
        etags.push(...results);
      }
      
      return etags;
    }
  }

Progress Display:
  Uploading large_file.zip...
  [████████████████░░░░░░░░] 65% (8,320 / 12,800 blocks)
  Speed: 12.5 MB/s
  Time remaining: 4 minutes 32 seconds

┌─────────────────────────────────────────────────────────────────┐
│ Step 4: Resumable Upload (Connection Drops)                     │
└─────────────────────────────────────────────────────────────────┘

Scenario: Upload fails at 65% (8,320 blocks uploaded)

Client (on restart):
  1. Load saved state from local storage
     state = localStorage.getItem('upload_state')
     {
       upload_id: "multipart_abc123",
       completed_parts: 8320,
       etags: [/* 8,320 ETags */]
     }
  
  2. Verify uploaded parts with server
     POST /v1/files/upload/verify
     Body: {
       upload_id: "multipart_abc123",
       file_id: "file_xyz789"
     }
     
     Server checks S3:
       parts = s3.listParts({
         Bucket: "dropbox-blocks",
         Key: block.key,
         UploadId: upload_id
       })
       
       Returns: Parts 1-8,320 confirmed
  
  3. Resume from block 8,321
     remaining_blocks = blocks[8320:]  // 4,480 blocks left
     
     Upload only remaining blocks
     Time: 4,480 blocks × 4MB / 100Mbps = 143 seconds
     
     User saved: 8,320 blocks × 4MB / 100Mbps = 266 seconds
     
     Resume saves 4.4 minutes! ✓

┌─────────────────────────────────────────────────────────────────┐
│ Step 5: Complete Multipart Upload                               │
└─────────────────────────────────────────────────────────────────┘

Client → Server:
  POST /v1/files/upload/complete
  Body: {
    upload_id: "multipart_abc123",
    file_id: "file_xyz789",
    parts: [
      {part_number: 1, etag: "etag_1"},
      {part_number: 2, etag: "etag_2"},
      ...
      {part_number: 12800, etag: "etag_12800"}
    ]
  }

Server:
  1. Verify all parts uploaded
     for each part:
       verify_part_exists(upload_id, part_number, etag)
  
  2. Complete S3 multipart upload
     s3.completeMultipartUpload({
       Bucket: "dropbox-blocks",
       Key: block.key,
       UploadId: upload_id,
       MultipartUpload: {
         Parts: parts.map(p => ({
           PartNumber: p.part_number,
           ETag: p.etag
         }))
       }
     })
     
     S3 assembles all parts into single object
  
  3. Update file metadata
     UPDATE files SET status = 'COMPLETED' WHERE file_id = ?
  
  4. Update block reference counts
     For each block: increment reference_count
  
  5. Clean up upload state
     DELETE FROM upload_state WHERE upload_id = ?

Success!
```

### Compression Strategy:

```
┌─────────────────────────────────────────────────────────────────┐
│ Smart Compression Decision                                      │
└─────────────────────────────────────────────────────────────────┘

Client Logic:
  function shouldCompress(file) {
    const compressibleTypes = [
      'text/plain',
      'text/html',
      'text/css',
      'application/javascript',
      'application/json',
      'application/xml'
    ];
    
    const alreadyCompressed = [
      'image/jpeg',
      'image/png',
      'video/mp4',
      'application/zip',
      'application/gzip'
    ];
    
    // Don't compress if already compressed
    if (alreadyCompressed.includes(file.type)) {
      return false;
    }
    
    // Compress text files
    if (compressibleTypes.includes(file.type)) {
      return true;
    }
    
    // For unknown types, test compression ratio
    const sample = file.slice(0, 1024 * 1024); // 1MB sample
    const compressed = gzip(sample);
    const ratio = compressed.size / sample.size;
    
    // Compress if ratio < 0.8 (20% savings)
    return ratio < 0.8;
  }

Compression Results:
  Text file (5GB):
    - Compressed: 1GB (80% savings)
    - Upload time: 1GB / 100Mbps = 80 seconds
    - Without compression: 5GB / 100Mbps = 400 seconds
    - Time saved: 320 seconds (5.3 minutes)
  
  Video file (50GB):
    - Compressed: 49.5GB (1% savings)
    - Compression overhead: 30 seconds
    - Not worth it! Skip compression ✓

Algorithm Choice:
  - Gzip: Standard, widely supported
  - Brotli: Better compression, slower
  - Zstandard: Best balance, modern
  
  Recommendation: Zstandard (zstd)
    - 20-30% better than gzip
    - Fast compression/decompression
    - Tunable compression levels
```

### Bandwidth Optimization:

```
┌─────────────────────────────────────────────────────────────────┐
│ Adaptive Chunk Size                                             │
└─────────────────────────────────────────────────────────────────┘

Problem: Fixed 4MB chunks not optimal for all networks

Slow Network (1 Mbps):
  - 4MB chunk takes 32 seconds
  - Too long per chunk
  - Better: 1MB chunks (8 seconds each)

Fast Network (1 Gbps):
  - 4MB chunk takes 0.032 seconds
  - Too many HTTP requests
  - Better: 16MB chunks (0.128 seconds each)

Solution: Adaptive Chunking
  function getOptimalChunkSize(bandwidth) {
    const targetTime = 5; // 5 seconds per chunk
    const chunkSize = bandwidth * targetTime;
    
    // Clamp between 1MB and 16MB
    return Math.max(1MB, Math.min(16MB, chunkSize));
  }

Network Detection:
  1. Upload test chunk (1MB)
  2. Measure time taken
  3. Calculate bandwidth
  4. Adjust chunk size
  5. Re-measure every 10 chunks

Result:
  - Slow networks: Smaller chunks, faster feedback
  - Fast networks: Larger chunks, fewer requests
  - Optimal user experience ✓
```

---

## FOLLOW-UP QUESTIONS & DEEP DIVES

### Follow-up 1: How do you handle file versioning?

**Answer:**

```
Approach: Copy-on-Write with Block Reuse

Data Model:
  files table:
    - file_id (primary key)
    - current_version_id (foreign key)
  
  file_versions table:
    - version_id (primary key)
    - file_id (foreign key)
    - version_number (1, 2, 3, ...)
    - created_at
    - created_by
  
  version_blocks table:
    - version_id (foreign key)
    - block_hash
    - block_order

Flow:
  User edits file:
    1. Create new version entry
       INSERT INTO file_versions (file_id, version_number)
       VALUES (file_id, current_version + 1)
    
    2. Copy unchanged blocks (just references!)
       INSERT INTO version_blocks (version_id, block_hash, block_order)
       SELECT new_version_id, block_hash, block_order
       FROM version_blocks
       WHERE version_id = old_version_id
       AND block_order NOT IN (changed_blocks)
    
    3. Add new blocks
       INSERT INTO version_blocks (version_id, block_hash, block_order)
       VALUES (new_version_id, new_block_hash, block_order)
    
    4. Update current version
       UPDATE files SET current_version_id = new_version_id

Storage Impact:
  - Only changed blocks stored
  - Unchanged blocks reused (reference counting)
  - 10 versions with 1% changes each = 10% extra storage
  - Much better than 1000% (10 full copies)

Retention Policy:
  - Keep last 30 versions
  - Or versions from last 90 days
  - Delete old versions (decrement block ref counts)
```

### Follow-up 2: How do you handle concurrent edits?

**Answer:**

```
Problem: Two users edit same shared file simultaneously

Scenario:
  User A and User B both have "document.txt" open
  Both make changes offline
  Both try to sync

Detection:
  Server tracks file version/timestamp
  
  User A uploads:
    - Base version: v5 (timestamp: 10:00 AM)
    - New version: v6 (timestamp: 11:00 AM)
    - Server accepts ✓
  
  User B uploads:
    - Base version: v5 (timestamp: 10:00 AM)
    - New version: v6 (timestamp: 11:05 AM)
    - Server detects conflict! ✗
      (v6 already exists from User A)

Resolution Options:

Option 1: Last-Write-Wins (Simple)
  - Accept User B's version (11:05 > 11:00)
  - Save User A's version as conflict copy
  - Notify both users

Option 2: Operational Transform (Complex)
  - Merge both changes automatically
  - Used by Google Docs
  - Requires tracking individual operations
  - Out of scope for file sync

Option 3: Lock-Based (Pessimistic)
  - User A locks file while editing
  - User B cannot edit until A releases lock
  - Poor user experience
  - Not suitable for offline-first system

Recommendation: Last-Write-Wins + Conflict Copies
  - Simple to implement
  - Works offline
  - User can manually resolve conflicts
```

### Follow-up 3: How do you optimize for mobile devices?

**Answer:**

```
Challenges:
  - Limited bandwidth (4G: 10 Mbps)
  - Expensive data plans
  - Battery constraints
  - Storage limitations

Optimizations:

1. Selective Sync
   - Don't sync all files to mobile
   - User chooses which folders to sync
   - "Available offline" flag per file
   
   Storage savings: 90% (only 10% of files synced)

2. Thumbnail Generation
   - Generate low-res thumbnails on server
   - Mobile downloads thumbnails only
   - Full file downloaded on demand
   
   Example:
     - Original photo: 5MB
     - Thumbnail: 50KB (100x reduction)

3. Adaptive Quality
   - Detect network type (WiFi vs 4G)
   - WiFi: Full quality
   - 4G: Compressed/lower quality
   
   Bandwidth savings: 50-70%

4. Background Sync
   - Sync only when charging + WiFi
   - Avoid draining battery
   - User can override for urgent files

5. Delta Sync Priority
   - Mobile benefits most from delta sync
   - Only sync changed blocks
   - Reduces data usage by 90%+

6. Compression
   - Always compress on mobile
   - Even for images (slight quality loss acceptable)
   - Saves 20-30% bandwidth
```

---

## EVALUATION CRITERIA MAPPING

### How This Design Demonstrates Excellence:

**1. Functional Requirements (25%)**
```
✓ Upload files up to 50GB with chunking
✓ Download files with CDN acceleration
✓ Bidirectional sync (local ↔ remote)
✓ File sharing with permissions
✓ Resumable uploads/downloads
✓ Progress tracking

Score: Outstanding (25/25)
```

**2. Non-Functional Requirements (25%)**
```
✓ Performance: < 5 sec sync latency
✓ Scalability: 100M users, 100B files
✓ Availability: 99.99% (multi-region)
✓ Durability: 99.999999999% (S3)
✓ Efficiency: 30% storage savings (dedup)

Score: Outstanding (25/25)
```

**3. System Design & Architecture (20%)**
```
✓ Block-level deduplication (core innovation)
✓ Separation of metadata and data
✓ Direct S3 access (presigned URLs)
✓ WebSocket for real-time sync
✓ Multi-database strategy
✓ CDN for downloads

Score: Outstanding (20/20)
```

**4. Scalability & Performance (15%)**
```
✓ Handles 50GB files (chunking + multipart)
✓ Instant uploads (100% dedup)
✓ Delta sync (only changed blocks)
✓ 30% storage savings ($82.8M/year)
✓ Horizontal scaling

Score: Outstanding (15/15)
```

**5. Trade-off Analysis (15%)**
```
✓ 4MB chunk size justification
✓ SHA-256 vs MD5 vs SHA-1 comparison
✓ Deduplication ratio analysis
✓ Compression decision logic
✓ Last-write-wins vs OT for conflicts

Score: Outstanding (15/15)
```

**Total Score: 100/100 (Top 5% Performance)**

---

## INTERVIEW TIPS & STRATEGY

### Time Management:

```
0-5 min:   Requirements (focus on 50GB files!)
5-8 min:   Functional/non-functional requirements
8-13 min:  Back-of-envelope (chunking analysis critical)
13-21 min: High-level architecture (deduplication!)
21-24 min: API design (block existence check)
24-28 min: Data models (separation of concerns)
28-36 min: Core flows (upload/download/sync)
36-41 min: Deep dive 1 (deduplication algorithm)
41-45 min: Deep dive 2 (large file handling)
```

### Strong Signals to Send:

```
1. Identify Core Innovation Early
   ✓ "Block-level deduplication is the key - saves 300 PB"
   ✓ "Chunking enables resumable uploads for 50GB files"
   ✓ "Delta sync only uploads changed blocks"

2. Quantitative Analysis
   ✓ "50GB file takes 1.1 hours without chunking (unacceptable)"
   ✓ "4MB chunks optimal (12,800 blocks manageable)"
   ✓ "30% deduplication saves $82.8M/year"

3. Real-World Awareness
   ✓ "Dropbox uses 4MB blocks"
   ✓ "Google Drive uses similar deduplication"
   ✓ "S3 multipart upload is industry standard"

4. Trade-off Discussions
   ✓ "SHA-256 vs MD5: Security vs speed"
   ✓ "Compression: Text files yes, videos no"
   ✓ "Last-write-wins vs OT: Simplicity vs automation"
```

### Common Pitfalls to Avoid:

```
✗ Forgetting chunking (50GB single upload fails)
✗ File-level deduplication only (misses delta sync)
✗ No reference counting (orphaned blocks)
✗ Synchronous replication (slow uploads)
✗ Polling for sync (WebSocket better)
✗ No compression strategy (waste bandwidth)
✗ Ignoring mobile optimizations
✗ No conflict resolution plan
```

---

## FINAL CHECKLIST

```
□ Clarified 50GB file requirement
□ Explained chunking necessity (1.1 hour problem)
□ Calculated optimal chunk size (4MB)
□ Designed block-level deduplication
□ Showed 30% storage savings (300 PB)
□ Explained SHA-256 hash function
□ Implemented reference counting
□ Designed garbage collection
□ Showed instant upload for duplicates
□ Explained delta sync (only changed blocks)
□ Designed multipart upload flow
□ Implemented resumable uploads
□ Added progress tracking
□ Designed WebSocket for real-time sync
□ Explained CDN for downloads
□ Discussed compression strategy
□ Handled conflict resolution
□ Addressed mobile optimizations
□ Covered versioning (follow-up)
```

---

## SUMMARY

### What We Built:

```
A Dropbox-like file synchronization system that:
- Handles 100M users with 100B files
- Supports files up to 50GB
- Saves 30% storage via deduplication (300 PB)
- Enables instant uploads for duplicate files
- Syncs only changed blocks (delta sync)
- Provides resumable uploads with progress
- Achieves < 5 second sync latency
- Costs $82.8M less per year
```

### Key Design Decisions:

```
1. Block-Level Deduplication
   - 4MB chunks with SHA-256 hashing
   - Saves 300 PB storage ($82.8M/year)
   - Enables instant uploads
   - Efficient delta sync

2. Chunking Strategy
   - Solves 50GB upload problem (1.1 hours → resumable)
   - Progress tracking
   - Resumable uploads
   - Parallel uploads (10 concurrent)

3. Direct S3 Access
   - Presigned URLs bypass API servers
   - Reduces server load
   - Faster transfers
   - CDN for downloads (80% hit rate)

4. WebSocket for Sync
   - Real-time push notifications
   - No polling overhead
   - < 5 second sync latency

5. Reference Counting
   - Tracks block usage
   - Enables safe garbage collection
   - Prevents orphaned blocks
```

### Why This Design Wins:

```
✓ Solves 50GB file problem (chunking + multipart)
✓ Saves 300 PB storage (deduplication)
✓ Instant uploads for duplicates (100% dedup)
✓ Efficient delta sync (only changed blocks)
✓ Resumable uploads (connection drops OK)
✓ Real-time sync (WebSocket push)
✓ Scalable (100M users, 100B files)
✓ Cost-effective ($82.8M annual savings)
✓ Production-proven (Dropbox, Google Drive use similar)
```

**This design demonstrates Top 5% system design skills through identifying block-level deduplication as the core innovation, showing quantitative analysis (300 PB savings, 1.1 hour problem), and explaining the complete chunking + hashing + deduplication + multipart upload flow.**

