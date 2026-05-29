# Chat Application - Deep Dive Follow-ups

## Table of Contents
1. Media Files (Photos/Videos) - Complete Solution
2. Multi-Device Message Synchronization
3. Cross-Datacenter Communication & User Location Discovery

---

## 1. MEDIA FILES - COMPLETE DEEP DIVE

### Problem Statement

Text messages: ~200 bytes
Image: ~2 MB (10,000x larger!)
Video: ~50 MB (250,000x larger!)

With 10% of 50B messages being media:
- 5B media files/day
- Average 10 MB per file
- 50 PB/day of media traffic!

**Key Challenges:**
1. Storage cost (50 PB/day)
2. Bandwidth (29 GB/sec)
3. Upload/download latency
4. Thumbnail generation
5. Multiple device formats
6. CDN distribution

---

### Approach 1: Store in Database (❌ TERRIBLE)

```
Client → Server → PostgreSQL/MongoDB (store blob)
```

**Why it fails:**
- Databases not optimized for large blobs
- PostgreSQL max row size: 1 GB
- Expensive: $0.10/GB/month
- Slow: No CDN, single region
- Wastes DB connections
- 50 PB × $0.10 = $5M/month just storage!

**Verdict:** ❌ Never do this

---

### Approach 2: Server Proxies to S3 (⚠️ INEFFICIENT)

```
Upload:
Client → Server → S3

Download:
Client → Server → S3 → Server → Client
```

**Why it's bad:**
- Server handles all bytes (bottleneck)
- 50 PB/day through servers = massive bandwidth cost
- Server CPU wasted on proxying
- Latency: 2 hops instead of 1
- Can't scale horizontally easily

**Verdict:** ⚠️ Works but wasteful

---

### Approach 3: Pre-Signed URLs (✅ BEST)

```
Upload:
1. Client → Server: "I want to upload photo.jpg (2 MB)"
2. Server → Client: Pre-signed S3 URL (valid 15 min)
3. Client → S3: Direct upload
4. Client → Server: "Upload complete, URL: s3://..."
5. Server: Send message with S3 URL to recipient

Download:
1. Server → Client: Message with S3 URL
2. Client → S3: Direct download (or via CloudFront CDN)
```

**Why it's best:**
✅ Server doesn't handle media bytes
✅ Direct client ↔ S3 transfer (fast)
✅ S3 handles scale automatically
✅ Can add CloudFront CDN easily
✅ Secure (URLs expire)
✅ Cost-effective

---

### Complete Media Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    UPLOAD FLOW                                   │
└─────────────────────────────────────────────────────────────────┘

┌──────────┐
│  Client  │
│  (iOS)   │
└────┬─────┘
     │
     │ 1. POST /media/upload-url
     │    { type: "image/jpeg", size: 2048576 }
     ▼
┌─────────────────┐
│  Media Service  │
│                 │
│  - Validate size│
│  - Generate ID  │
│  - Create S3    │
│    pre-signed   │
│    URL          │
└────┬────────────┘
     │
     │ 2. Response:
     │    { 
     │      media_id: "m-123",
     │      upload_url: "https://s3.../photo.jpg?signature=...",
     │      expires_in: 900
     │    }
     ▼
┌──────────┐
│  Client  │
└────┬─────┘
     │
     │ 3. PUT to S3 upload_url
     │    (Direct upload, bypasses server)
     ▼
┌─────────────────┐
│   S3 Bucket     │
│  /media/2024/   │
│    01/15/       │
│      m-123.jpg  │
└────┬────────────┘
     │
     │ 4. S3 Event Notification
     ▼
┌─────────────────┐
│  Lambda         │
│  (Async)        │
│                 │
│  - Generate     │
│    thumbnails   │
│  - Compress     │
│  - Update DB    │
└────┬────────────┘
     │
     │ 5. Write metadata
     ▼
┌─────────────────┐
│  DynamoDB       │
│  media_metadata │
│                 │
│  - media_id     │
│  - original_url │
│  - thumbnail_url│
│  - size         │
│  - status       │
└─────────────────┘
     │
     │ 6. Client sends message
     ▼
┌──────────┐
│  Client  │ → POST /messages/send
└──────────┘    {
                  chat_id: "c-456",
                  media_id: "m-123",
                  caption: "Check this out!"
                }

┌─────────────────────────────────────────────────────────────────┐
│                    DOWNLOAD FLOW                                 │
└─────────────────────────────────────────────────────────────────┘

┌──────────┐
│ Recipient│
└────┬─────┘
     │
     │ 1. Receives message via WebSocket
     │    { media_id: "m-123", caption: "..." }
     ▼
┌──────────┐
│  Client  │
└────┬─────┘
     │
     │ 2. GET /media/m-123/download-url
     ▼
┌─────────────────┐
│  Media Service  │
│                 │
│  - Fetch from DB│
│  - Generate     │
│    CloudFront   │
│    signed URL   │
└────┬────────────┘
     │
     │ 3. Response:
     │    {
     │      thumbnail_url: "https://cdn.../m-123-thumb.jpg",
     │      full_url: "https://cdn.../m-123.jpg",
     │      expires_in: 3600
     │    }
     ▼
┌──────────┐
│  Client  │
└────┬─────┘
     │
     │ 4. Download thumbnail first (progressive loading)
     │    GET thumbnail_url
     ▼
┌─────────────────┐
│  CloudFront CDN │
│  (Edge Location)│
└────┬────────────┘
     │
     │ 5. If not cached, fetch from S3
     ▼
┌─────────────────┐
│   S3 Bucket     │
└─────────────────┘
```

---

### Data Model for Media

```sql
-- DynamoDB Table: media_metadata
{
  "media_id": "m-123",                    // Partition Key
  "user_id": "u-456",                     // Uploader
  "type": "image/jpeg",                   // MIME type
  "original_size": 2048576,               // Bytes
  "compressed_size": 512000,              // After compression
  "width": 1920,
  "height": 1080,
  "duration": null,                       // For videos
  
  "s3_key": "media/2024/01/15/m-123.jpg",
  "s3_bucket": "chat-media-prod",
  
  "thumbnail_s3_key": "media/2024/01/15/m-123-thumb.jpg",
  "thumbnail_size": 15360,
  
  "cdn_url": "https://d123.cloudfront.net/m-123.jpg",
  "thumbnail_cdn_url": "https://d123.cloudfront.net/m-123-thumb.jpg",
  
  "status": "READY",                      // UPLOADING, PROCESSING, READY, FAILED
  "created_at": 1705334400,
  "ttl": 1707926400                       // Auto-delete after 30 days
}

-- Messages table reference
{
  "message_id": "msg-789",
  "chat_id": "c-456",
  "sender_id": "u-456",
  "content": "Check this out!",
  "media_ids": ["m-123"],                 // Array of media IDs
  "timestamp": 1705334400
}
```

---

### Compression Strategy

**Images:**
```
Original JPEG (2 MB, 4000×3000)
  ↓
1. Resize to max 1920×1080 (if larger)
2. Compress with quality=85
3. Convert to WebP (30% smaller)
  ↓
Compressed (500 KB)
Savings: 75%

Thumbnail:
- Resize to 200×150
- Quality=70
- Size: ~15 KB
```

**Videos:**
```
Original MP4 (50 MB, 1080p, H.264)
  ↓
1. Transcode to H.265 (50% smaller)
2. Generate multiple qualities:
   - 1080p (high)
   - 720p (medium)
   - 480p (low)
3. Adaptive bitrate streaming (HLS/DASH)
  ↓
Compressed 1080p (25 MB)
Savings: 50%

Thumbnail:
- Extract frame at 1 second
- Resize to 200×150
- Size: ~10 KB
```

**Implementation (Lambda):**
```python
import boto3
from PIL import Image
import io

def lambda_handler(event, context):
    # Triggered by S3 upload
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    s3 = boto3.client('s3')
    
    # Download original
    obj = s3.get_object(Bucket=bucket, Key=key)
    img = Image.open(io.BytesIO(obj['Body'].read()))
    
    # Compress
    img.thumbnail((1920, 1080), Image.LANCZOS)
    
    # Save compressed
    buffer = io.BytesIO()
    img.save(buffer, format='WEBP', quality=85)
    buffer.seek(0)
    
    compressed_key = key.replace('.jpg', '-compressed.webp')
    s3.put_object(Bucket=bucket, Key=compressed_key, Body=buffer)
    
    # Generate thumbnail
    img.thumbnail((200, 150), Image.LANCZOS)
    thumb_buffer = io.BytesIO()
    img.save(thumb_buffer, format='WEBP', quality=70)
    thumb_buffer.seek(0)
    
    thumb_key = key.replace('.jpg', '-thumb.webp')
    s3.put_object(Bucket=bucket, Key=thumb_key, Body=thumb_buffer)
    
    # Update DynamoDB
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('media_metadata')
    
    media_id = key.split('/')[-1].split('.')[0]
    table.update_item(
        Key={'media_id': media_id},
        UpdateExpression='SET #status = :ready, compressed_s3_key = :ckey, thumbnail_s3_key = :tkey',
        ExpressionAttributeNames={'#status': 'status'},
        ExpressionAttributeValues={
            ':ready': 'READY',
            ':ckey': compressed_key,
            ':tkey': thumb_key
        }
    )
```

---

### Progressive Loading Strategy

**Client-side implementation:**
```javascript
// 1. Show placeholder immediately
showPlaceholder(messageId);

// 2. Load thumbnail (15 KB, fast)
const thumbnailUrl = await fetchThumbnailUrl(mediaId);
displayThumbnail(messageId, thumbnailUrl);

// 3. Load full image in background
const fullUrl = await fetchFullUrl(mediaId);
preloadImage(fullUrl);

// 4. When full image loaded, swap
onImageLoaded(fullUrl, () => {
  displayFullImage(messageId, fullUrl);
});

// User experience:
// 0ms: Placeholder (gray box)
// 100ms: Blurry thumbnail visible
// 2000ms: Full quality image visible
```

---

### CDN Configuration (CloudFront)

```yaml
CloudFront Distribution:
  Origins:
    - S3 Bucket: chat-media-prod
      Origin Path: /media
  
  Behaviors:
    - Path: /thumbnails/*
      Cache TTL: 7 days
      Compress: true
    
    - Path: /images/*
      Cache TTL: 30 days
      Compress: true
    
    - Path: /videos/*
      Cache TTL: 30 days
      Compress: false  # Already compressed
  
  Edge Locations: All (global)
  
  Price Class: All edges (best performance)
  
  Signed URLs: Enabled (security)
```

**Why CDN is critical:**
- User in Tokyo downloads from Tokyo edge (20ms)
- Without CDN: Downloads from US-East (200ms)
- 10x latency improvement!
- Reduces S3 egress costs (CDN cheaper)

---

### Storage Lifecycle Policy

```yaml
S3 Lifecycle Rules:

1. Transition to Infrequent Access:
   - After 30 days → S3-IA
   - Cost: $0.023/GB (vs $0.125/GB standard)
   - Savings: 82%

2. Transition to Glacier:
   - After 90 days → Glacier
   - Cost: $0.004/GB
   - Savings: 97%

3. Delete:
   - After 1 year → Delete
   - Compliance: Keep for legal period
```

**Cost calculation:**
```
50 PB/day × 30 days = 1.5 EB/month

Standard S3: 1.5 EB × $0.023/GB = $34.5M/month
With lifecycle: 
  - 30 days standard: 1.5 PB × $0.023 = $34.5K
  - 60 days S3-IA: 3 PB × $0.0125 = $37.5K
  - Rest Glacier: 1.5 EB × $0.004 = $6M
Total: ~$6.1M/month

Savings: $28.4M/month (82%)
```

---

### Video-Specific Handling

**Adaptive Bitrate Streaming:**
```
Original video (1080p, 50 MB)
  ↓
Transcode to multiple qualities:
  - 1080p: 25 MB (H.265)
  - 720p: 12 MB
  - 480p: 6 MB
  - 360p: 3 MB
  ↓
Package as HLS:
  - master.m3u8 (playlist)
  - 1080p/segment-001.ts
  - 1080p/segment-002.ts
  - ...
  - 720p/segment-001.ts
  - ...
```

**Client automatically selects quality based on:**
- Network speed
- Device capability
- User preference

**Implementation:**
```python
# AWS MediaConvert job
import boto3

mediaconvert = boto3.client('mediaconvert')

job = {
    'Role': 'arn:aws:iam::123:role/MediaConvertRole',
    'Settings': {
        'Inputs': [{
            'FileInput': 's3://bucket/video.mp4'
        }],
        'OutputGroups': [{
            'Name': 'HLS',
            'OutputGroupSettings': {
                'Type': 'HLS_GROUP_SETTINGS',
                'HlsGroupSettings': {
                    'Destination': 's3://bucket/hls/'
                }
            },
            'Outputs': [
                {'VideoDescription': {'Width': 1920, 'Height': 1080}},
                {'VideoDescription': {'Width': 1280, 'Height': 720}},
                {'VideoDescription': {'Width': 854, 'Height': 480}}
            ]
        }]
    }
}

mediaconvert.create_job(**job)
```

---

### Security Considerations

**1. Pre-Signed URL Security:**
```python
import boto3
from datetime import timedelta

s3 = boto3.client('s3')

# Upload URL (short expiry)
upload_url = s3.generate_presigned_url(
    'put_object',
    Params={
        'Bucket': 'chat-media',
        'Key': f'media/{media_id}.jpg',
        'ContentType': 'image/jpeg'
    },
    ExpiresIn=900  # 15 minutes
)

# Download URL (longer expiry)
download_url = s3.generate_presigned_url(
    'get_object',
    Params={
        'Bucket': 'chat-media',
        'Key': f'media/{media_id}.jpg'
    },
    ExpiresIn=3600  # 1 hour
)
```

**2. Content Validation:**
```python
def validate_upload(file_type, file_size):
    # Check MIME type
    allowed_types = ['image/jpeg', 'image/png', 'video/mp4']
    if file_type not in allowed_types:
        raise ValueError("Invalid file type")
    
    # Check size
    max_sizes = {
        'image/jpeg': 10 * 1024 * 1024,  # 10 MB
        'image/png': 10 * 1024 * 1024,
        'video/mp4': 100 * 1024 * 1024   # 100 MB
    }
    if file_size > max_sizes[file_type]:
        raise ValueError("File too large")
    
    return True
```

**3. Virus Scanning:**
```
S3 Upload → Lambda → ClamAV scan → Quarantine if infected
```

---

### Cost Optimization Summary

```
┌──────────────────────┬─────────────┬─────────────┬──────────┐
│ Component            │ Without Opt │ With Opt    │ Savings  │
├──────────────────────┼─────────────┼─────────────┼──────────┤
│ Storage (S3)         │ $34.5M/mo   │ $6.1M/mo    │ 82%      │
│ Bandwidth (CDN)      │ $15M/mo     │ $3M/mo      │ 80%      │
│ Compression          │ N/A         │ 75% less    │ 75%      │
│ Thumbnails           │ N/A         │ Fast load   │ 10x      │
└──────────────────────┴─────────────┴─────────────┴──────────┘

Total monthly cost: ~$9M (vs $50M without optimization)
Savings: $41M/month (82%)
```



---

## 2. MULTI-DEVICE MESSAGE SYNCHRONIZATION - COMPLETE DEEP DIVE

### Problem Statement

User has 3 devices:
- iPhone (primary)
- MacBook (work)
- iPad (home)

**Requirements:**
1. Message sent from iPhone appears on MacBook and iPad
2. Message read on MacBook marks as read on iPhone and iPad
3. Typing on iPad shows indicator on iPhone and MacBook
4. Offline device syncs when comes online
5. No duplicate messages
6. Consistent message order across devices

---

### Naive Approach: User-Level Inbox (❌ BROKEN)

```
Table: inbox
- user_id (PK)
- message_id (SK)
- delivered (boolean)

Flow:
1. Message arrives for user-123
2. Write to inbox for user-123
3. iPhone receives message
4. iPhone sends ACK
5. Delete from inbox
6. MacBook comes online
7. Query inbox → Empty!
8. MacBook never gets message ❌
```

**Why it fails:**
- Single inbox per user
- First device to ACK deletes message
- Other devices miss it

---

### Solution: Client-Specific Inbox

**Schema:**
```sql
-- DynamoDB Table: inbox
{
  "user_id": "u-123",              // Partition Key
  "client_id#message_id": "c1#m1", // Sort Key (composite)
  
  "client_id": "c1",               // iPhone
  "message_id": "m1",
  "chat_id": "chat-456",
  "delivered": false,
  "created_at": 1705334400,
  "ttl": 1707926400                // 30 days
}

-- Separate entry for each device
{
  "user_id": "u-123",
  "client_id#message_id": "c2#m1", // MacBook
  "client_id": "c2",
  "message_id": "m1",
  ...
}

{
  "user_id": "u-123",
  "client_id#message_id": "c3#m1", // iPad
  "client_id": "c3",
  "message_id": "m1",
  ...
}
```

**Client Registration:**
```sql
-- Table: user_clients
{
  "user_id": "u-123",
  "client_id": "c1",
  "device_type": "iOS",
  "device_name": "iPhone 15 Pro",
  "last_seen": 1705334400,
  "registered_at": 1705000000
}

-- Limit: Max 5 devices per user
-- Oldest device auto-removed when 6th added
```

---

### Complete Synchronization Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: User sends message from iPhone                       │
└─────────────────────────────────────────────────────────────────┘

┌──────────┐
│ iPhone   │ (client-1, user-123)
│ (Online) │
└────┬─────┘
     │
     │ 1. Send message
     │    { chat_id: "c-456", content: "Hello" }
     ▼
┌─────────────────┐
│  Chat Gateway   │
│  (Server 1)     │
└────┬────────────┘
     │
     │ 2. Write to messages table
     ▼
┌─────────────────┐
│   DynamoDB      │
│   messages      │
│   msg-789       │
└─────────────────┘
     │
     │ 3. Fan-out to recipients
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  For each recipient in chat:                                     │
│                                                                   │
│  Recipient: user-456 (has 2 devices: MacBook, iPad)             │
│                                                                   │
│  a) Query user_clients for user-456                             │
│     → Returns: [client-5 (MacBook), client-6 (iPad)]            │
│                                                                   │
│  b) Write to inbox for EACH client:                             │
│     - inbox: user-456, client-5#msg-789                         │
│     - inbox: user-456, client-6#msg-789                         │
│                                                                   │
│  c) Publish to Redis Pub/Sub:                                   │
│     - Channel: "user-456"                                        │
│     - Message: { msg_id: "msg-789", chat_id: "c-456" }          │
└─────────────────────────────────────────────────────────────────┘
     │
     ├──────────────────┬──────────────────┐
     ▼                  ▼                  ▼
┌──────────┐      ┌──────────┐      ┌──────────┐
│ MacBook  │      │  iPad    │      │ iPhone   │
│ (Online) │      │ (Offline)│      │ (Sender) │
│ Server 2 │      │    -     │      │ Server 1 │
└────┬─────┘      └──────────┘      └────┬─────┘
     │                                    │
     │ 4. Receives via                    │ 4. Receives via
     │    Pub/Sub                         │    Pub/Sub
     │                                    │    (own message)
     │ 5. Delivers to                     │
     │    MacBook                         │ 5. Shows "sent"
     │                                    │    checkmark
     │ 6. MacBook sends ACK               │
     ▼                                    ▼
┌─────────────────┐              ┌─────────────────┐
│  Chat Gateway   │              │  Chat Gateway   │
│  (Server 2)     │              │  (Server 1)     │
└────┬────────────┘              └────┬────────────┘
     │                                │
     │ 7. Delete from inbox           │ 7. Delete from inbox
     │    user-456, client-5#msg-789  │    user-123, client-1#msg-789
     ▼                                ▼
┌─────────────────┐              ┌─────────────────┐
│   DynamoDB      │              │   DynamoDB      │
│   inbox         │              │   inbox         │
└─────────────────┘              └─────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  LATER: iPad comes online                                        │
└─────────────────────────────────────────────────────────────────┘

┌──────────┐
│  iPad    │ (client-6, user-456)
└────┬─────┘
     │
     │ 1. Connects to gateway
     │    WebSocket handshake
     ▼
┌─────────────────┐
│  Chat Gateway   │
│  (Server 3)     │
└────┬────────────┘
     │
     │ 2. Query inbox for this client
     │    SELECT * FROM inbox
     │    WHERE user_id = 'user-456'
     │    AND client_id = 'client-6'
     ▼
┌─────────────────┐
│   DynamoDB      │
│   inbox         │
│                 │
│   Returns:      │
│   - msg-789     │
│   - msg-790     │
│   - msg-791     │
└────┬────────────┘
     │
     │ 3. Fetch full messages
     ▼
┌─────────────────┐
│   DynamoDB      │
│   messages      │
└────┬────────────┘
     │
     │ 4. Deliver all pending messages
     ▼
┌──────────┐
│  iPad    │
└────┬─────┘
     │
     │ 5. iPad sends ACK for each
     ▼
┌─────────────────┐
│  Chat Gateway   │
└────┬────────────┘
     │
     │ 6. Delete from inbox
     │    user-456, client-6#msg-789
     │    user-456, client-6#msg-790
     │    user-456, client-6#msg-791
     ▼
┌─────────────────┐
│   DynamoDB      │
│   inbox         │
└─────────────────┘
```

---

### Read Receipts Across Devices

**Problem:** User reads message on MacBook, should show as read on iPhone

**Solution: Read Status Table**

```sql
-- Table: read_status
{
  "chat_id#user_id": "c-456#u-123",  // Partition Key
  "message_id": "msg-789",            // Sort Key
  
  "user_id": "u-123",
  "read_at": 1705334400,
  "read_by_client": "client-5"        // MacBook
}

-- Query: Get last read message for user in chat
SELECT message_id FROM read_status
WHERE chat_id#user_id = 'c-456#u-123'
ORDER BY message_id DESC
LIMIT 1
```

**Flow:**
```
1. User reads message on MacBook
   ↓
2. MacBook sends:
   { type: "READ", message_id: "msg-789" }
   ↓
3. Server writes to read_status table
   ↓
4. Server publishes to Redis:
   Channel: "user-123"
   Message: { type: "READ_UPDATE", message_id: "msg-789" }
   ↓
5. iPhone receives update
   ↓
6. iPhone marks message as read locally
   ↓
7. UI updates: Double blue checkmark
```

---

### Typing Indicators Across Devices

**Problem:** User types on iPad, show indicator on iPhone and MacBook

**Solution: Ephemeral State (Redis)**

```redis
# Key: typing:{chat_id}:{user_id}
# Value: timestamp
# TTL: 5 seconds

SET typing:c-456:u-123 1705334400 EX 5

# Pub/Sub for real-time
PUBLISH chat:c-456:typing {
  "user_id": "u-123",
  "typing": true
}
```

**Flow:**
```
1. User types on iPad
   ↓
2. iPad sends (throttled to 1/sec):
   { type: "TYPING", chat_id: "c-456" }
   ↓
3. Server:
   - Sets Redis key (5 sec TTL)
   - Publishes to Pub/Sub
   ↓
4. Other devices in chat receive:
   - iPhone (same user) → Don't show
   - MacBook (same user) → Don't show
   - Recipient devices → Show "User-123 is typing..."
   ↓
5. After 5 seconds of no typing:
   - Redis key expires
   - Send TYPING_STOPPED event
```

---

### Message Order Consistency

**Problem:** Messages arrive out of order on different devices

**Solution: Lamport Timestamps + Client-Side Sorting**

```sql
-- Message schema
{
  "message_id": "msg-789",
  "chat_id": "c-456",
  "sender_id": "u-123",
  "content": "Hello",
  "timestamp": 1705334400.123,      // Server timestamp (authoritative)
  "client_timestamp": 1705334399.5, // Client timestamp (for ordering)
  "sequence_number": 42              // Per-chat sequence
}

-- Sequence counter per chat (Redis)
INCR chat:c-456:sequence
→ Returns: 42
```

**Client-side sorting:**
```javascript
// Client receives messages potentially out of order
const messages = [
  { id: "m2", timestamp: 1705334401, seq: 43 },
  { id: "m1", timestamp: 1705334400, seq: 42 },
  { id: "m3", timestamp: 1705334402, seq: 44 }
];

// Sort by sequence number (authoritative)
messages.sort((a, b) => a.seq - b.seq);

// Display in order: m1, m2, m3
```

**Why this works:**
✅ Server assigns sequence numbers (atomic)
✅ Client sorts locally (fast)
✅ Handles network delays
✅ Consistent across all devices

---

### Conflict Resolution

**Scenario:** User deletes message on iPhone while offline, MacBook still shows it

**Solution: Operation Log + Sync**

```sql
-- Table: operation_log
{
  "user_id": "u-123",
  "operation_id": "op-999",
  "client_id": "client-1",
  "operation_type": "DELETE_MESSAGE",
  "message_id": "msg-789",
  "timestamp": 1705334400,
  "synced_to_clients": ["client-1"]  // Track which clients have synced
}
```

**Sync flow:**
```
1. iPhone (offline) deletes message
   - Store in local queue
   - Mark as deleted locally
   
2. iPhone comes online
   - Upload operation to server
   - Server writes to operation_log
   
3. Server publishes to other devices:
   - MacBook receives DELETE_MESSAGE
   - MacBook deletes locally
   - MacBook ACKs sync
   
4. Server updates synced_to_clients
   - When all clients synced, delete operation_log entry
```

---

### Last Seen Sync

**Problem:** User's last seen should be consistent across devices

**Solution: Max Timestamp**

```redis
# Key: last_seen:{user_id}
# Value: { timestamp, client_id }

# When any device is active:
SET last_seen:u-123 '{"ts":1705334400,"client":"c1"}' EX 300

# Query last seen:
GET last_seen:u-123
→ If exists: User is online
→ If not exists: Fetch from DB (last disconnect time)
```

**DB schema:**
```sql
-- Table: user_presence
{
  "user_id": "u-123",
  "status": "online",               // online, offline, away
  "last_seen": 1705334400,
  "active_clients": ["c1", "c2"],   // Currently connected
  "updated_at": 1705334400
}

-- Update on any device activity:
UPDATE user_presence
SET last_seen = NOW(),
    active_clients = array_append(active_clients, 'c1')
WHERE user_id = 'u-123'
```

---

### Bandwidth Optimization

**Problem:** Syncing large chat history to new device

**Solution: Incremental Sync**

```javascript
// New device connects
1. Client sends last known message_id
   { type: "SYNC", last_message_id: "msg-500" }

2. Server queries messages after msg-500
   SELECT * FROM messages
   WHERE chat_id = 'c-456'
   AND message_id > 'msg-500'
   ORDER BY message_id
   LIMIT 100

3. Send in batches of 100
   - Reduces memory usage
   - Allows progress indicator
   - Can resume if interrupted

4. Client requests next batch
   { type: "SYNC", last_message_id: "msg-600" }
```

**Optimization: Sync only metadata first**
```javascript
// Phase 1: Sync message metadata (fast)
{
  messages: [
    { id: "m1", sender: "u-123", timestamp: 1705334400, has_media: false },
    { id: "m2", sender: "u-456", timestamp: 1705334401, has_media: true }
  ]
}

// Phase 2: Lazy load content
// Only fetch when user scrolls to that message
```

---

### Storage Optimization

**Problem:** Each device stores full chat history (wasteful)

**Solution: Cloud-First Storage**

```
┌──────────────────────────────────────────────────────────────┐
│  Storage Strategy                                             │
├──────────────────────────────────────────────────────────────┤
│  Server (DynamoDB):                                           │
│  - All messages (source of truth)                            │
│  - Retention: 1 year                                          │
│                                                                │
│  Client (Local DB):                                           │
│  - Recent messages (last 7 days)                             │
│  - Frequently accessed chats                                  │
│  - Media thumbnails only                                      │
│                                                                │
│  Client (Memory):                                             │
│  - Active chat (last 100 messages)                           │
│  - Cleared when chat closed                                   │
└──────────────────────────────────────────────────────────────┘
```

**Implementation:**
```javascript
// Client-side cache strategy
class MessageCache {
  constructor() {
    this.memory = new Map();        // Active chats
    this.localDB = new IndexedDB(); // Recent messages
  }
  
  async getMessage(messageId) {
    // 1. Check memory
    if (this.memory.has(messageId)) {
      return this.memory.get(messageId);
    }
    
    // 2. Check local DB
    const local = await this.localDB.get(messageId);
    if (local) {
      this.memory.set(messageId, local);
      return local;
    }
    
    // 3. Fetch from server
    const remote = await api.getMessage(messageId);
    this.localDB.set(messageId, remote);
    this.memory.set(messageId, remote);
    return remote;
  }
  
  // Cleanup old messages
  async cleanup() {
    const sevenDaysAgo = Date.now() - 7 * 24 * 60 * 60 * 1000;
    await this.localDB.deleteOlderThan(sevenDaysAgo);
  }
}
```

---

### Summary: Multi-Device Sync Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Key Components                                                  │
├─────────────────────────────────────────────────────────────────┤
│  1. Client-Specific Inbox                                       │
│     - Separate queue per device                                 │
│     - Ensures all devices receive messages                      │
│                                                                   │
│  2. Read Status Table                                            │
│     - Tracks read state per user per chat                       │
│     - Syncs read receipts across devices                        │
│                                                                   │
│  3. Operation Log                                                │
│     - Tracks user actions (delete, edit)                        │
│     - Syncs operations to all devices                           │
│                                                                   │
│  4. Sequence Numbers                                             │
│     - Ensures consistent message ordering                       │
│     - Client-side sorting                                        │
│                                                                   │
│  5. Incremental Sync                                             │
│     - Sync only new messages                                    │
│     - Batched for efficiency                                     │
│                                                                   │
│  6. Cloud-First Storage                                          │
│     - Server is source of truth                                 │
│     - Client caches recent data                                 │
└─────────────────────────────────────────────────────────────────┘
```



---

## 3. CROSS-DATACENTER COMMUNICATION - COMPLETE DEEP DIVE

### Problem Statement

**Scenario:**
- User A in Tokyo (Asia datacenter)
- User B in New York (US datacenter)
- User A sends message to User B
- Challenge: Minimize latency, find which server User B is connected to

**Key Challenges:**
1. User location discovery (which datacenter?)
2. Cross-region message routing
3. Latency minimization
4. Data consistency across regions
5. Failover and disaster recovery

---

### Architecture: Multi-Region Deployment

```
┌─────────────────────────────────────────────────────────────────┐
│                    GLOBAL ARCHITECTURE                           │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────────┐         ┌──────────────────────────┐
│   ASIA REGION (Tokyo)    │         │   US REGION (Virginia)   │
│                          │         │                          │
│  ┌────────────────────┐ │         │  ┌────────────────────┐ │
│  │  Chat Gateways     │ │         │  │  Chat Gateways     │ │
│  │  (20K servers)     │ │         │  │  (20K servers)     │ │
│  │  - User A          │ │         │  │  - User B          │ │
│  └────────┬───────────┘ │         │  └────────┬───────────┘ │
│           │             │         │           │             │
│  ┌────────▼───────────┐ │         │  ┌────────▼───────────┐ │
│  │  Redis Pub/Sub     │ │         │  │  Redis Pub/Sub     │ │
│  │  (Regional)        │ │◄────────┼──┤  (Regional)        │ │
│  └────────────────────┘ │  Cross- │  └────────────────────┘ │
│                          │  Region │                          │
│  ┌────────────────────┐ │  Sync   │  ┌────────────────────┐ │
│  │  DynamoDB          │ │         │  │  DynamoDB          │ │
│  │  (Global Tables)   │◄┼─────────┼─►│  (Global Tables)   │ │
│  │  - messages        │ │         │  │  - messages        │ │
│  │  - user_presence   │ │         │  │  - user_presence   │ │
│  └────────────────────┘ │         │  └────────────────────┘ │
└──────────────────────────┘         └──────────────────────────┘

┌──────────────────────────┐         ┌──────────────────────────┐
│   EU REGION (Ireland)    │         │  LATAM REGION (Sao Paulo)│
│  (Similar setup)         │         │  (Similar setup)         │
└──────────────────────────┘         └──────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│              GLOBAL COORDINATION LAYER                           │
│                                                                   │
│  ┌────────────────────┐         ┌────────────────────┐         │
│  │  Global Registry   │         │  Global Load       │         │
│  │  (Consul/Etcd)     │         │  Balancer (Route53)│         │
│  │  - User locations  │         │  - Geo-routing     │         │
│  │  - Server health   │         │  - Latency-based   │         │
│  └────────────────────┘         └────────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

---

### Challenge 1: User Location Discovery

**Problem:** When User A sends message, how do we find User B's datacenter/server?

#### Solution 1: Global Registry (Consul/Etcd)

```
┌─────────────────────────────────────────────────────────────────┐
│  Global Registry (Consul)                                        │
│                                                                   │
│  Key-Value Store:                                                │
│  /users/u-123/location → { region: "us-east", server: "gw-5432" }│
│  /users/u-456/location → { region: "ap-tokyo", server: "gw-1234"}│
│                                                                   │
│  TTL: 60 seconds (refreshed by heartbeat)                       │
└─────────────────────────────────────────────────────────────────┘
```

**Flow:**
```
1. User B connects to US datacenter
   ↓
2. Gateway registers in Consul:
   PUT /users/u-456/location
   {
     "region": "us-east-1",
     "datacenter": "us-virginia",
     "server_id": "gw-5432",
     "server_ip": "10.0.1.42",
     "connected_at": 1705334400
   }
   TTL: 60 seconds
   ↓
3. Gateway sends heartbeat every 30 seconds to refresh TTL
   ↓
4. If connection drops, TTL expires, entry removed
```

**Lookup:**
```
User A (Tokyo) sends message to User B:

1. Tokyo gateway queries Consul:
   GET /users/u-456/location
   ↓
2. Response:
   {
     "region": "us-east-1",
     "server_id": "gw-5432",
     "server_ip": "10.0.1.42"
   }
   ↓
3. Tokyo gateway routes message to US region
```

**Latency:**
- Consul read: ~5ms (local replica)
- Total lookup overhead: ~5-10ms

---

#### Solution 2: Distributed Hash Table (DHT)

```
┌─────────────────────────────────────────────────────────────────┐
│  Consistent Hashing Ring                                         │
│                                                                   │
│  hash(user_id) → Determines home region                         │
│                                                                   │
│  User u-123: hash → 0x1234 → Asia region                        │
│  User u-456: hash → 0x5678 → US region                          │
└─────────────────────────────────────────────────────────────────┘
```

**Implementation:**
```python
import hashlib

def get_home_region(user_id):
    # Consistent hashing
    hash_value = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
    
    # Ring: 0 to 2^128
    regions = [
        {"name": "us-east", "start": 0, "end": 2**126},
        {"name": "eu-west", "start": 2**126, "end": 2**127},
        {"name": "ap-tokyo", "start": 2**127, "end": 2**128}
    ]
    
    for region in regions:
        if region["start"] <= hash_value < region["end"]:
            return region["name"]
```

**Pros:**
✅ No external dependency
✅ Deterministic (same user always same region)
✅ Fast (local computation)

**Cons:**
❌ User might not be in home region (traveling)
❌ Need fallback mechanism

---

#### Solution 3: Hybrid Approach (✅ BEST)

**Combine both:**
1. Use DHT to determine home region (fast, no lookup)
2. Check local cache for actual location
3. If not in cache, query global registry
4. Fallback to home region if not found

```python
class UserLocationService:
    def __init__(self):
        self.local_cache = {}  # In-memory cache
        self.consul = ConsulClient()
    
    async def find_user_location(self, user_id):
        # 1. Check local cache (0ms)
        if user_id in self.local_cache:
            if self.local_cache[user_id]['expires'] > time.time():
                return self.local_cache[user_id]['location']
        
        # 2. Query Consul (5ms)
        location = await self.consul.get(f'/users/{user_id}/location')
        if location:
            # Cache for 30 seconds
            self.local_cache[user_id] = {
                'location': location,
                'expires': time.time() + 30
            }
            return location
        
        # 3. Fallback to home region (deterministic)
        home_region = self.get_home_region(user_id)
        return {
            'region': home_region,
            'server_id': None,  # Will use regional Pub/Sub
            'online': False
        }
    
    def get_home_region(self, user_id):
        # Consistent hashing
        hash_value = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
        regions = ['us-east', 'eu-west', 'ap-tokyo', 'latam']
        return regions[hash_value % len(regions)]
```

**Latency breakdown:**
```
Cache hit: 0ms (99% of requests)
Cache miss: 5ms (Consul lookup)
Fallback: 0ms (local computation)

Average: 0.05ms (excellent!)
```

---

### Challenge 2: Cross-Region Message Routing

**Scenario:** User A (Tokyo) → User B (US)

#### Approach 1: Direct Server-to-Server (❌ Complex)

```
Tokyo Gateway → US Gateway (direct connection)
```

**Problems:**
- Need mesh network (N² connections)
- Complex routing logic
- Hard to maintain

---

#### Approach 2: Regional Pub/Sub Bridge (✅ BEST)

```
┌─────────────────────────────────────────────────────────────────┐
│  Cross-Region Message Flow                                       │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────────┐
│  User A (Tokyo)          │
└────┬─────────────────────┘
     │ 1. Send message
     ▼
┌──────────────────────────┐
│  Tokyo Gateway           │
└────┬─────────────────────┘
     │ 2. Write to DynamoDB Global Table
     ▼
┌──────────────────────────┐
│  DynamoDB (Tokyo)        │
│  - Replicates to US      │
│    (< 1 second)          │
└────┬─────────────────────┘
     │
     │ 3. Publish to local Redis
     ▼
┌──────────────────────────┐
│  Redis Pub/Sub (Tokyo)   │
└────┬─────────────────────┘
     │
     │ 4. Bridge service forwards to US
     ▼
┌──────────────────────────┐
│  Kafka (Cross-Region)    │
│  - Topic: cross-region   │
│  - Partition by region   │
└────┬─────────────────────┘
     │
     │ 5. US bridge consumes
     ▼
┌──────────────────────────┐
│  Redis Pub/Sub (US)      │
│  - Publish to user-456   │
└────┬─────────────────────┘
     │
     │ 6. US Gateway receives
     ▼
┌──────────────────────────┐
│  US Gateway              │
│  - Delivers to User B    │
└────┬─────────────────────┘
     │
     ▼
┌──────────────────────────┐
│  User B (New York)       │
└──────────────────────────┘

Total latency: ~150ms
- DynamoDB write: 10ms
- Cross-region Kafka: 100ms
- Redis Pub/Sub: 10ms
- WebSocket delivery: 30ms
```

**Bridge Service Implementation:**
```python
class CrossRegionBridge:
    def __init__(self, local_region, remote_regions):
        self.local_region = local_region
        self.redis_local = RedisClient(local_region)
        self.kafka = KafkaProducer()
        self.remote_regions = remote_regions
    
    async def start(self):
        # Subscribe to local Redis
        pubsub = self.redis_local.pubsub()
        
        # Subscribe to all user channels
        await pubsub.psubscribe('user-*')
        
        async for message in pubsub.listen():
            await self.handle_message(message)
    
    async def handle_message(self, message):
        user_id = message['channel'].split('-')[1]
        
        # Check if user is in remote region
        location = await self.find_user_location(user_id)
        
        if location['region'] != self.local_region:
            # Forward to remote region via Kafka
            await self.kafka.send(
                topic=f'cross-region-{location["region"]}',
                key=user_id,
                value=message['data']
            )
```

---

### Challenge 3: Latency Optimization

**Goal:** Minimize cross-region latency

#### Technique 1: Smart Routing

```python
class SmartRouter:
    def route_message(self, sender_region, recipient_region):
        # If same region: Use local Pub/Sub (10ms)
        if sender_region == recipient_region:
            return 'local_pubsub'
        
        # If adjacent regions: Direct connection (50ms)
        if self.are_adjacent(sender_region, recipient_region):
            return 'direct_bridge'
        
        # If far regions: Use global backbone (150ms)
        return 'kafka_bridge'
    
    def are_adjacent(self, region1, region2):
        # Define adjacency (low latency links)
        adjacent = {
            'us-east': ['us-west', 'eu-west'],
            'eu-west': ['us-east', 'ap-tokyo'],
            'ap-tokyo': ['eu-west', 'ap-singapore']
        }
        return region2 in adjacent.get(region1, [])
```

---

#### Technique 2: Message Prioritization

```python
class MessagePriority:
    def get_priority(self, message):
        # High priority: Text messages (small, urgent)
        if message['type'] == 'text':
            return 'HIGH'
        
        # Medium priority: Typing indicators
        if message['type'] == 'typing':
            return 'MEDIUM'
        
        # Low priority: Read receipts
        if message['type'] == 'read_receipt':
            return 'LOW'
        
        # Lowest: Presence updates
        if message['type'] == 'presence':
            return 'LOWEST'

# Kafka topics by priority
kafka.send(
    topic=f'cross-region-{priority}',
    value=message
)
```

---

#### Technique 3: Regional Caching

```
┌─────────────────────────────────────────────────────────────────┐
│  Regional Cache Strategy                                         │
└─────────────────────────────────────────────────────────────────┘

User profile, chat metadata → Cache in all regions (read-heavy)
Messages → Cache in home region only (write-heavy)
Presence → Cache locally, sync periodically (ephemeral)
```

**Implementation:**
```python
class RegionalCache:
    def __init__(self, region):
        self.region = region
        self.redis = RedisClient(region)
    
    async def get_user_profile(self, user_id):
        # Try local cache first
        cached = await self.redis.get(f'profile:{user_id}')
        if cached:
            return cached
        
        # Fetch from home region
        home_region = self.get_home_region(user_id)
        if home_region == self.region:
            # Local read
            profile = await self.db.get_user(user_id)
        else:
            # Cross-region read (slower)
            profile = await self.remote_db.get_user(user_id, home_region)
        
        # Cache locally for 5 minutes
        await self.redis.setex(f'profile:{user_id}', 300, profile)
        return profile
```

---

### Challenge 4: Data Consistency

**Problem:** DynamoDB Global Tables have eventual consistency (~1 second)

#### Solution: Write to Home Region, Read from Any

```
┌─────────────────────────────────────────────────────────────────┐
│  Write Strategy                                                  │
└─────────────────────────────────────────────────────────────────┘

1. Determine message home region (sender's region)
2. Write to that region's DynamoDB
3. DynamoDB replicates to other regions (async)
4. Immediate delivery via Pub/Sub (doesn't wait for replication)

┌─────────────────────────────────────────────────────────────────┐
│  Read Strategy                                                   │
└─────────────────────────────────────────────────────────────────┘

1. Read from local region (fast)
2. If message not found, read from home region (slower)
3. Cache locally
```

**Handling conflicts:**
```python
class ConflictResolver:
    def resolve_message_conflict(self, version1, version2):
        # Last-write-wins based on timestamp
        if version1['timestamp'] > version2['timestamp']:
            return version1
        return version2
    
    def resolve_read_status_conflict(self, status1, status2):
        # Max read position wins
        if status1['last_read_message_id'] > status2['last_read_message_id']:
            return status1
        return status2
```

---

### Challenge 5: Failover & Disaster Recovery

**Scenario:** Tokyo datacenter goes down

#### Solution: Automatic Failover

```
┌─────────────────────────────────────────────────────────────────┐
│  Failover Strategy                                               │
└─────────────────────────────────────────────────────────────────┘

1. Health Check (every 10 seconds)
   - Each region pings others
   - If 3 consecutive failures → Mark as down

2. DNS Failover (Route53)
   - Geo-routing with health checks
   - Automatically routes to nearest healthy region
   - TTL: 60 seconds

3. Client Reconnection
   - Client detects connection loss
   - Queries DNS (gets new region)
   - Reconnects to healthy region
   - Syncs missed messages from inbox

4. Data Recovery
   - DynamoDB Global Tables continue replicating
   - When Tokyo recovers, catches up automatically
   - No data loss (messages in other regions)
```

**Implementation:**
```python
class FailoverManager:
    def __init__(self):
        self.regions = ['us-east', 'eu-west', 'ap-tokyo']
        self.health_status = {}
    
    async def health_check_loop(self):
        while True:
            for region in self.regions:
                is_healthy = await self.ping_region(region)
                self.health_status[region] = is_healthy
                
                if not is_healthy:
                    await self.trigger_failover(region)
            
            await asyncio.sleep(10)
    
    async def trigger_failover(self, failed_region):
        # Update Route53 health check
        await self.route53.mark_unhealthy(failed_region)
        
        # Notify monitoring
        await self.alert(f'Region {failed_region} is down')
        
        # Redistribute load
        healthy_regions = [r for r in self.regions 
                          if self.health_status[r]]
        
        # Update global registry
        await self.consul.set('active_regions', healthy_regions)
```

---

### Complete Cross-Region Flow Example

```
┌─────────────────────────────────────────────────────────────────┐
│  User A (Tokyo) sends message to User B (New York)              │
└─────────────────────────────────────────────────────────────────┘

Time: 0ms
┌──────────┐
│ User A   │ Sends: "Hello from Tokyo!"
│ (Tokyo)  │
└────┬─────┘
     │
Time: 5ms
     ▼
┌─────────────────┐
│ Tokyo Gateway   │
│ - Receives msg  │
│ - Assigns ID    │
└────┬────────────┘
     │
Time: 10ms
     │ 1. Find User B location
     ▼
┌─────────────────┐
│ Local Cache     │ → Cache miss
└────┬────────────┘
     │
Time: 15ms
     ▼
┌─────────────────┐
│ Consul (Tokyo)  │ → Returns: { region: "us-east", server: "gw-5432" }
└────┬────────────┘
     │
Time: 20ms
     │ 2. Write to DynamoDB
     ▼
┌─────────────────┐
│ DynamoDB (Tokyo)│
│ - Write message │
│ - Start replica │
└────┬────────────┘
     │
Time: 25ms
     │ 3. Publish locally (for Tokyo users in same chat)
     ▼
┌─────────────────┐
│ Redis (Tokyo)   │
└────┬────────────┘
     │
Time: 30ms
     │ 4. Forward to US via Kafka
     ▼
┌─────────────────┐
│ Kafka Bridge    │
│ - Topic: us-east│
└────┬────────────┘
     │
Time: 130ms (100ms cross-region latency)
     ▼
┌─────────────────┐
│ Kafka (US)      │
│ - Consumed by   │
│   US bridge     │
└────┬────────────┘
     │
Time: 135ms
     │ 5. Publish to US Redis
     ▼
┌─────────────────┐
│ Redis (US)      │
│ - Channel:      │
│   user-456      │
└────┬────────────┘
     │
Time: 140ms
     │ 6. Deliver to User B
     ▼
┌─────────────────┐
│ US Gateway      │
│ - Server gw-5432│
│ - Has User B    │
│   connection    │
└────┬────────────┘
     │
Time: 150ms
     ▼
┌──────────┐
│ User B   │ Receives: "Hello from Tokyo!"
│ (NY)     │
└──────────┘

Total Latency: 150ms (excellent for cross-region!)
```

---

### Latency Comparison

```
┌──────────────────────┬─────────────┬─────────────┬─────────────┐
│ Scenario             │ Without Opt │ With Opt    │ Improvement │
├──────────────────────┼─────────────┼─────────────┼─────────────┤
│ Same region          │ 50ms        │ 10ms        │ 5x          │
│ Adjacent regions     │ 200ms       │ 50ms        │ 4x          │
│ Far regions          │ 500ms       │ 150ms       │ 3.3x        │
│ User lookup          │ 50ms        │ 0.05ms      │ 1000x       │
└──────────────────────┴─────────────┴─────────────┴─────────────┘

Optimizations applied:
✅ Local caching (user locations)
✅ Regional Pub/Sub (avoid global routing when possible)
✅ Kafka for cross-region (reliable, fast)
✅ DynamoDB Global Tables (automatic replication)
✅ Smart routing (adjacent regions)
```

---

### Summary: Cross-Datacenter Best Practices

```
┌─────────────────────────────────────────────────────────────────┐
│  Key Principles                                                  │
├─────────────────────────────────────────────────────────────────┤
│  1. User Location Discovery                                     │
│     - Hybrid: Cache + Consul + DHT fallback                     │
│     - Latency: < 1ms (99% cache hit)                            │
│                                                                   │
│  2. Cross-Region Routing                                         │
│     - Regional Pub/Sub for local delivery                       │
│     - Kafka bridge for cross-region                             │
│     - Smart routing based on distance                           │
│                                                                   │
│  3. Data Consistency                                             │
│     - DynamoDB Global Tables (eventual consistency)             │
│     - Write to home region, read from any                       │
│     - Pub/Sub for immediate delivery (doesn't wait)             │
│                                                                   │
│  4. Latency Optimization                                         │
│     - Regional caching (profiles, metadata)                     │
│     - Message prioritization (text > presence)                  │
│     - Adjacent region shortcuts                                 │
│                                                                   │
│  5. Failover & DR                                                │
│     - Health checks every 10 seconds                            │
│     - DNS failover (Route53)                                    │
│     - Automatic client reconnection                             │
│     - No data loss (multi-region replication)                   │
└─────────────────────────────────────────────────────────────────┘
```

This architecture achieves:
✅ 150ms cross-region latency (Tokyo → New York)
✅ 10ms same-region latency
✅ 99.99% availability (multi-region)
✅ Automatic failover (< 60 seconds)
✅ No data loss (global replication)
