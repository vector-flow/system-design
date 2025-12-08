# Minutes 21-35: Deep Dive - Core Components (15 min)

## **Why This Matters**

This is where you **differentiate yourself** from other candidates. Anyone can draw boxes and arrows—senior engineers understand the nuances:

- Implementation details that matter
- Trade-offs at the component level
- Edge cases and failure scenarios
- Performance optimizations

**Golden Rule:** Go deep on 2-3 components. Show mastery, not breadth.

-----

## **Your Opening (15 seconds)**

**What to say:**

> “Now let’s dive deep into the core components. I’ll focus on three critical areas: file chunking and storage, metadata management, and synchronization. These are the heart of the system and have the most interesting technical challenges.”

-----

# **DEEP DIVE 1: File Chunking & Storage (5 minutes)**

## **Part A: Chunking Strategy (90 seconds)**

### **Why Chunk Files?**

**Start by establishing the reasoning:**

> “First, why do we chunk files at all? Four main reasons:”

```
BENEFITS OF CHUNKING:
══════════════════════════════════════════
1. DEDUPLICATION
   - Same chunk across files stored once
   - User has 100 photos with same background
   - Background chunk stored once, saves 99% space
   
2. DELTA SYNC
   - Only upload modified chunks
   - Edit 1KB in 1GB file → upload 4MB chunk
   - Saves bandwidth and time
   
3. PARALLEL UPLOADS/DOWNLOADS
   - 10 chunks × 10 parallel connections
   - Saturate bandwidth better
   - Faster for large files
   
4. RESUME CAPABILITY
   - Upload fails at 80%
   - Resume from last successful chunk
   - No need to restart entire file
```

-----

### **Chunk Size Decision (60 seconds)**

**Present the trade-off analysis:**

```
CHUNK SIZE ANALYSIS:
═══════════════════════════════════════════════════

TOO SMALL (512 KB):
✗ Too many chunks → metadata explosion
✗ More network overhead (headers per chunk)
✗ More dedup lookups required
Example: 1GB file = 2,000 chunks

TOO LARGE (64 MB):
✗ Less deduplication benefit
✗ Poor delta sync (change 1 byte → re-upload 64MB)
✗ Harder to parallelize effectively
Example: 1GB file = 16 chunks

SWEET SPOT (4 MB):
✓ Good deduplication granularity
✓ Efficient delta detection
✓ ~250 chunks for 1GB file (manageable)
✓ Standard in industry (Dropbox uses 4MB)
✓ Good parallel upload performance

DECISION: 4 MB chunks
```

**Narrate:**

> “We’ll use 4MB chunks. This is battle-tested—Dropbox, Google Drive, and OneDrive all use similar sizes. It balances deduplication benefits with metadata overhead.”

-----

### **Chunking Algorithm (90 seconds)**

**Explain the implementation:**

> “Now, how do we actually chunk files? Two approaches:”

**Approach 1: Fixed-Size Chunking (Simple)**

```python
# Pseudo-code
def fixed_chunk(file_path, chunk_size=4MB):
    chunks = []
    with open(file_path, 'rb') as f:
        chunk_id = 0
        while True:
            data = f.read(chunk_size)
            if not data:
                break
            
            # Calculate hash for deduplication
            chunk_hash = sha256(data)
            
            chunks.append({
                'id': chunk_id,
                'hash': chunk_hash,
                'size': len(data),
                'offset': chunk_id * chunk_size
            })
            chunk_id += 1
    
    return chunks
```

**Problem with fixed chunking:**

> “If user inserts 1 byte at start of file, ALL chunks shift. Every chunk hash changes → no deduplication benefit!”

```
BEFORE INSERT:
File: [AAAA][BBBB][CCCC][DDDD]
       ↓     ↓     ↓     ↓
Hash: h1    h2    h3    h4

AFTER INSERT 'X' AT START:
File: [XAAA][ABBB][BCCC][CDDD][D...]
       ↓     ↓     ↓     ↓
Hash: h5    h6    h7    h8  ← All different!
```

-----

**Approach 2: Content-Defined Chunking (CDC) (Better)**

```python
# Pseudo-code using Rabin fingerprinting
def content_defined_chunk(file_path, avg_size=4MB):
    chunks = []
    with open(file_path, 'rb') as f:
        window = []
        current_chunk = []
        
        while True:
            byte = f.read(1)
            if not byte:
                break
            
            current_chunk.append(byte)
            window.append(byte)
            
            # Rolling hash (Rabin fingerprint)
            fingerprint = rolling_hash(window)
            
            # Check if this is a chunk boundary
            # Using low-order bits for boundary detection
            if (fingerprint & 0xFFF) == 0xFFF or len(current_chunk) >= 8MB:
                # Found boundary or max size reached
                chunk_data = bytes(current_chunk)
                chunk_hash = sha256(chunk_data)
                
                chunks.append({
                    'hash': chunk_hash,
                    'size': len(chunk_data),
                    'data': chunk_data
                })
                
                current_chunk = []
        
        # Handle last chunk
        if current_chunk:
            chunks.append(create_chunk(current_chunk))
    
    return chunks
```

**How CDC works:**

```
CONCEPT:
- Slide a window through file bytes
- Calculate rolling hash at each position
- When hash matches pattern (e.g., last 12 bits = all 1s)
  → This is a chunk boundary
- Probability of boundary: 1/4096 → avg 4KB chunks

ADVANTAGE:
File: [AAAA | BBBB | CCCC | DDDD]
              ↑           ↑
        Boundaries based on content

After insert:
File: [X | AAAA | BBBB | CCCC | DDDD]
       ↑    ↑           ↑
       New  Same boundaries maintained!
       
Result: Only first chunk is new, rest are deduplicated!
```

**Decision:**

> “For production, we’d use **Content-Defined Chunking with Rabin fingerprinting**. It’s more complex but gives much better deduplication, especially for incremental file changes. This is what Dropbox uses.”

-----

## **Part B: Deduplication Strategy (90 seconds)**

### **How Deduplication Works:**

```
DEDUPLICATION FLOW:
═══════════════════════════════════════════

1. CLIENT CHUNKS FILE
   vacation.jpg → 13 chunks

2. CALCULATE CHUNK HASHES
   Chunk 0: hash → a3f5c8d...
   Chunk 1: hash → 7b2e9f1...
   ...

3. CHECK WHICH CHUNKS EXIST
   API call: POST /api/chunks/check
   Request: ["a3f5c8d...", "7b2e9f1...", ...]
   
4. SERVER QUERIES DEDUP TABLE
   SELECT chunk_hash FROM chunks 
   WHERE chunk_hash IN (...)
   
5. SERVER RESPONDS
   Response: {
     "existing": ["a3f5c8d...", "9c4d2a1..."],
     "needed": ["7b2e9f1...", "e8f3b7c..."]
   }

6. CLIENT UPLOADS ONLY NEEDED CHUNKS
   - Skip existing chunks (saves bandwidth!)
   - Upload only new chunks
   
7. SERVER CREATES FILE RECORD
   file_id → [chunk_ref_1, chunk_ref_2, ...]
   Multiple files can reference same chunks!
```

-----

### **Deduplication Database Schema:**

```sql
-- Chunk storage table (deduplicated)
CREATE TABLE chunks (
    chunk_hash VARCHAR(64) PRIMARY KEY,  -- SHA-256 hash
    size_bytes INTEGER NOT NULL,
    storage_path VARCHAR(512) NOT NULL,  -- S3 key
    ref_count INTEGER DEFAULT 1,         -- Reference counting
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed TIMESTAMP,
    
    INDEX idx_last_accessed (last_accessed)  -- For cleanup
);

-- File to chunks mapping
CREATE TABLE file_chunks (
    file_id BIGINT NOT NULL,
    chunk_hash VARCHAR(64) NOT NULL,
    chunk_index INTEGER NOT NULL,       -- Order in file
    chunk_offset BIGINT NOT NULL,       -- Byte offset in file
    
    PRIMARY KEY (file_id, chunk_index),
    FOREIGN KEY (chunk_hash) REFERENCES chunks(chunk_hash),
    FOREIGN KEY (file_id) REFERENCES files(file_id) ON DELETE CASCADE
);
```

**Key design decisions:**

> “Notice a few things:
> 
> 1. **Content-addressable storage**: Chunk hash is the primary key
> 1. **Reference counting**: Track how many files use each chunk
> 1. **Garbage collection**: When ref_count hits 0, chunk can be deleted
> 1. **Separate mapping table**: Same chunk can be in multiple files/positions”

-----

### **Deduplication Effectiveness (30 seconds)**

**Show the math:**

```
DEDUPLICATION SAVINGS EXAMPLE:
══════════════════════════════════════════

Scenario: 100M users, each has:
- 5 copies of company logo (1MB)
- 3 versions of presentation (50MB each)
- Similar document templates

WITHOUT DEDUP:
100M users × 5 logos × 1MB = 500 TB
100M users × 3 presentations × 50MB = 15 PB
Total: ~15.5 PB

WITH DEDUP:
Company logo: Stored once × 1MB = 1 MB
Presentation: ~70% same chunks
Deduplicated storage: ~5 PB

SAVINGS: 67% reduction → Massive cost savings!
```

-----

## **Part C: Storage Architecture (90 seconds)**

### **Multi-Tier Storage Strategy:**

```
STORAGE TIERS:
═══════════════════════════════════════════════════

┌─────────────────────────────────────────────┐
│  HOT TIER (Recent/Frequently Accessed)      │
│  - SSD-backed storage                       │
│  - Low latency (~10ms)                      │
│  - Higher cost ($0.10/GB/month)             │
│  - TTL: 30 days since last access           │
│  - Example: S3 Standard                     │
└─────────────────────────────────────────────┘
                    │
                    │ (Auto-transition after 30 days)
                    ▼
┌─────────────────────────────────────────────┐
│  WARM TIER (Less Frequent Access)           │
│  - Standard HDD storage                     │
│  - Medium latency (~100ms)                  │
│  - Medium cost ($0.02/GB/month)             │
│  - TTL: 90 days since last access           │
│  - Example: S3 Infrequent Access            │
└─────────────────────────────────────────────┘
                    │
                    │ (Auto-transition after 90 days)
                    ▼
┌─────────────────────────────────────────────┐
│  COLD TIER (Archive)                        │
│  - Tape/slow HDD                            │
│  - High latency (~hours for retrieval)      │
│  - Low cost ($0.004/GB/month)               │
│  - Rarely accessed                          │
│  - Example: S3 Glacier                      │
└─────────────────────────────────────────────┘
```

**Cost calculation:**

```
COST SAVINGS WITH TIERING:
══════════════════════════════════════════

Total storage: 15 PB

Distribution:
- Hot (recent):  20% → 3 PB × $0.10 = $300K/month
- Warm:          50% → 7.5 PB × $0.02 = $150K/month  
- Cold:          30% → 4.5 PB × $0.004 = $18K/month

Total: $468K/month

Without tiering (all hot):
15 PB × $0.10 = $1.5M/month

SAVINGS: 69% reduction in storage costs!
```

-----

### **Storage Layout in S3:**

```
S3 BUCKET STRUCTURE:
═══════════════════════════════════════════

s3://dropbox-chunks/
├── chunks/
│   ├── a3/              ← First 2 chars of hash (sharding)
│   │   ├── f5/          ← Next 2 chars
│   │   │   └── a3f5c8d... (full hash as filename)
│   │   └── c8/
│   │       └── a3c8d9e...
│   ├── 7b/
│   │   └── 2e/
│   │       └── 7b2e9f1...
│   └── ...
│
└── metadata/            ← Backup of metadata
    └── shards/
        ├── shard_0/
        └── shard_1/

BENEFITS:
✓ Uniform distribution (hash-based)
✓ Avoids S3 hot partitions
✓ Easy to parallelize operations
✓ Natural sharding for operations
```

-----

# **DEEP DIVE 2: Metadata Management (4 minutes)**

## **Part A: Metadata Schema Design (90 seconds)**

### **Complete Database Schema:**

```sql
-- Users table
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100),
    storage_quota_gb INTEGER DEFAULT 10,
    storage_used_gb DECIMAL(12,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    last_login TIMESTAMP,
    
    INDEX idx_email (email)
);

-- Files table (main metadata)
CREATE TABLE files (
    file_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    parent_folder_id BIGINT,           -- NULL for root
    filename VARCHAR(512) NOT NULL,
    file_path VARCHAR(2048) NOT NULL,  -- Full path for quick lookup
    file_size_bytes BIGINT NOT NULL,
    file_type VARCHAR(50),              -- MIME type
    file_hash VARCHAR(64),              -- Hash of entire file
    version INTEGER DEFAULT 1,
    is_deleted BOOLEAN DEFAULT FALSE,
    is_shared BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    modified_at TIMESTAMP DEFAULT NOW(),
    last_accessed_at TIMESTAMP,
    
    -- Sharding key (critical!)
    SHARD_KEY: user_id
    
    INDEX idx_user_files (user_id, is_deleted, parent_folder_id),
    INDEX idx_path (user_id, file_path),
    INDEX idx_modified (user_id, modified_at),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Chunks table (already shown above)
CREATE TABLE chunks (
    chunk_hash VARCHAR(64) PRIMARY KEY,
    size_bytes INTEGER NOT NULL,
    storage_path VARCHAR(512) NOT NULL,
    ref_count INTEGER DEFAULT 1,
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed TIMESTAMP,
    
    INDEX idx_last_accessed (last_accessed)
);

-- File-to-chunks mapping
CREATE TABLE file_chunks (
    file_id BIGINT NOT NULL,
    chunk_hash VARCHAR(64) NOT NULL,
    chunk_index INTEGER NOT NULL,
    chunk_offset BIGINT NOT NULL,
    
    PRIMARY KEY (file_id, chunk_index),
    FOREIGN KEY (chunk_hash) REFERENCES chunks(chunk_hash),
    FOREIGN KEY (file_id) REFERENCES files(file_id) ON DELETE CASCADE,
    
    INDEX idx_chunk_lookup (chunk_hash)  -- For ref counting
);

-- File versions (for version history)
CREATE TABLE file_versions (
    version_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    file_id BIGINT NOT NULL,
    version_number INTEGER NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    created_by BIGINT NOT NULL,
    
    UNIQUE KEY unique_file_version (file_id, version_number),
    FOREIGN KEY (file_id) REFERENCES files(file_id) ON DELETE CASCADE
);

-- Version chunks mapping (what chunks this version has)
CREATE TABLE version_chunks (
    version_id BIGINT NOT NULL,
    chunk_hash VARCHAR(64) NOT NULL,
    chunk_index INTEGER NOT NULL,
    
    PRIMARY KEY (version_id, chunk_index),
    FOREIGN KEY (version_id) REFERENCES file_versions(version_id),
    FOREIGN KEY (chunk_hash) REFERENCES chunks(chunk_hash)
);

-- Sharing/permissions
CREATE TABLE file_permissions (
    permission_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    file_id BIGINT NOT NULL,
    shared_with_user_id BIGINT,        -- NULL for public links
    permission_type ENUM('read', 'write', 'owner'),
    share_link_token VARCHAR(64) UNIQUE,  -- For public links
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    
    INDEX idx_user_permissions (shared_with_user_id),
    INDEX idx_share_token (share_link_token),
    FOREIGN KEY (file_id) REFERENCES files(file_id) ON DELETE CASCADE
);

-- Sync state (for each device)
CREATE TABLE device_sync_state (
    device_id VARCHAR(64) NOT NULL,
    user_id BIGINT NOT NULL,
    file_id BIGINT NOT NULL,
    local_version INTEGER NOT NULL,
    server_version INTEGER NOT NULL,
    sync_status ENUM('synced', 'pending', 'conflict'),
    last_sync_at TIMESTAMP,
    
    PRIMARY KEY (device_id, file_id),
    INDEX idx_user_device (user_id, device_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (file_id) REFERENCES files(file_id) ON DELETE CASCADE
);
```

-----

## **Part B: Database Sharding Strategy (90 seconds)**

### **Why Shard?**

> “With 500M users and 100B files, a single database can’t handle:
> 
> - 115K read QPS (even with caching)
> - 300TB of metadata
> - High availability requirements
> 
> We need horizontal sharding.”

-----

### **Sharding Approach: User-Based Sharding**

```
SHARDING STRATEGY:
═══════════════════════════════════════════

Shard Key: user_id
Hash Function: Consistent Hashing
Number of Shards: 1,024 (configurable)

SHARD ASSIGNMENT:
shard_id = hash(user_id) % 1024

Example:
user_id = 123456
hash(123456) = 7483625
shard_id = 7483625 % 1024 = 329
→ User 123456 → Shard 329
```

**Shard distribution:**

```
┌─────────────────────────────────────────┐
│  SHARD 0 (100TB capacity)               │
│  - Users: 0-488K                        │
│  - Files: ~100M                         │
│  - Master + 2 Read Replicas             │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  SHARD 1                                │
│  - Users: 488K-976K                     │
│  - Files: ~100M                         │
│  - Master + 2 Read Replicas             │
└─────────────────────────────────────────┘

... (1,022 more shards)
```

-----

### **Why User-Based Sharding?**

```
PROS:
✓ User queries never cross shards
  "Get all files for user_id=X" → Single shard lookup
✓ Related data co-located (user's files together)
✓ Easy to maintain consistency per user
✓ Simple routing logic

CONS:
✗ Power users create hot shards
  Solution: Detect and split hot users to dedicated shards
✗ Shared files span shards
  Solution: Replicate sharing metadata
✗ Chunk table not sharded by user
  Solution: Separate chunk table, global with caching
```

-----

### **Routing Layer:**

```python
class ShardRouter:
    def __init__(self, num_shards=1024):
        self.num_shards = num_shards
        self.shard_map = self._load_shard_map()
    
    def get_shard(self, user_id):
        """Route user to appropriate shard"""
        shard_id = hash(user_id) % self.num_shards
        return self.shard_map[shard_id]
    
    def _load_shard_map(self):
        """Load shard topology from config service"""
        # Returns: {shard_id: {'master': 'db1.host', 'replicas': [...]}}
        return load_from_config_service()

# Usage in API server
router = ShardRouter()

def get_user_files(user_id):
    shard = router.get_shard(user_id)
    db = connect_to_shard(shard['master'])
    
    return db.query("""
        SELECT * FROM files 
        WHERE user_id = ? AND is_deleted = FALSE
    """, user_id)
```

-----

### **Handling Cross-Shard Queries:**

```
PROBLEM: Shared files
User A (Shard 5) shares file with User B (Shard 7)

SOLUTION 1: Denormalize (Preferred)
- Store sharing record in BOTH shards
- file_permissions table replicated
- Each user queries own shard only

file_permissions in Shard 5:
  file_id | owner_id | shared_with | type
  --------|----------|-------------|------
  100     | A        | B           | read

file_permissions in Shard 7:
  file_id | owner_id | shared_with | type
  --------|----------|-------------|------
  100     | A        | B           | read  (replicated)

SOLUTION 2: Shared files service
- Separate microservice for shared files
- Handles cross-shard queries
- More complex but cleaner separation
```

-----

## **Part C: Caching Strategy (60 seconds)**

### **Multi-Layer Caching:**

```
CACHE HIERARCHY:
═══════════════════════════════════════════

LAYER 1: Client-Side Cache
├─ Location: Desktop/mobile app
├─ What: File metadata, chunk hashes
├─ TTL: Until file modified
└─ Benefit: Zero network calls for repeated access

LAYER 2: CDN Cache (Varnish/CloudFront)
├─ Location: Edge locations worldwide
├─ What: Hot file chunks, thumbnails
├─ TTL: 24 hours
└─ Benefit: Low latency downloads

LAYER 3: Application Cache (Redis)
├─ Location: Same datacenter as API servers
├─ What: File metadata, user info, permissions
├─ TTL: 5 minutes (frequent) to 1 hour (rare)
└─ Benefit: Reduce DB load by 90%

LAYER 4: Database Query Cache
├─ Location: PostgreSQL query cache
├─ What: Query results
├─ TTL: Automatic invalidation
└─ Benefit: Repeated identical queries
```

-----

### **Redis Caching Implementation:**

```python
import redis
import json

redis_client = redis.Redis(host='cache.cluster', port=6379)

def get_file_metadata(file_id):
    """Get file metadata with caching"""
    cache_key = f"file:metadata:{file_id}"
    
    # Try cache first
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Cache miss - query database
    shard = router.get_shard_for_file(file_id)
    db = connect_to_shard(shard)
    
    file_data = db.query("""
        SELECT f.*, 
               GROUP_CONCAT(fc.chunk_hash ORDER BY fc.chunk_index) as chunks
        FROM files f
        LEFT JOIN file_chunks fc ON f.file_id = fc.file_id
        WHERE f.file_id = ?
        GROUP BY f.file_id
    """, file_id)
    
    if file_data:
        # Cache for 5 minutes
        redis_client.setex(
            cache_key, 
            300,  # TTL in seconds
            json.dumps(file_data)
        )
    
    return file_data

def invalidate_file_cache(file_id):
    """Invalidate cache when file changes"""
    cache_key = f"file:metadata:{file_id}"
    redis_client.delete(cache_key)
```

-----

### **Cache Invalidation Strategy:**

```
INVALIDATION PATTERNS:
═══════════════════════════════════════════

1. WRITE-THROUGH
   Update database → Update cache
   Pros: Cache always fresh
   Cons: Write latency increased

2. WRITE-BEHIND (Async)
   Update database → Invalidate cache → Lazy reload
   Pros: Fast writes
   Cons: Brief stale data possible

3. TTL-BASED (Our choice)
   Set expiration time on all cached data
   Pros: Simple, handles edge cases
   Cons: May serve stale data until expiry

DECISION: TTL-based with active invalidation
- Default TTL: 5 minutes
- On file modification: Active invalidation
- Balances consistency and performance
```

-----

# **DEEP DIVE 3: Synchronization Service (5 minutes)**

## **Part A: Sync Architecture (90 seconds)**

### **Overall Sync Flow:**

```
SYNC SERVICE ARCHITECTURE:
═══════════════════════════════════════════════

┌──────────────────────────────────────────┐
│          SYNC SERVICE CLUSTER            │
│  ┌────────────┐  ┌────────────┐          │
│  │ Sync Node 1│  │ Sync Node 2│  ...     │
│  │            │  │            │          │
│  │ Manages    │  │ Manages    │          │
│  │ 100K       │  │ 100K       │          │
│  │ connections│  │ connections│          │
│  └─────┬──────┘  └─────┬──────┘          │
└────────┼─────────────────┼───────────────┘
         │                 │
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │   Message Queue │
         │     (Kafka)     │
         │                 │
         │ Topics:         │
         │ - file.created  │
         │ - file.modified │
         │ - file.deleted  │
         │ - file.shared   │
         └────────┬────────┘
                  │
      ┌───────────┼───────────┐
      │           │           │
      ▼           ▼           ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│Device A │ │Device B │ │Device C │
│(Online) │ │(Online) │ │(Offline)│
└─────────┘ └─────────┘ └─────────┘
    ↑                         │
    └────── Polls on ─────────┘
           reconnect
```

-----

### **Connection Management:**

```python
class SyncService:
    def __init__(self):
        self.connections = {}  # device_id -> WebSocket
        self.user_devices = {}  # user_id -> [device_ids]
        
    async def handle_connection(self, websocket, device_id, user_id):
        """Handle new device connection"""
        
        # Register connection
        self.connections[device_id] = websocket
        
        if user_id not in self.user_devices:
            self.user_devices[user_id] = []
        self.user_devices[user_id].append(device_id)
        
        try:
            # Send initial sync
            await self.send_initial_sync(device_id, user_id)
            
            # Keep connection alive
            while True:
                # Heartbeat every 30 seconds
                await websocket.ping()
                await asyncio.sleep(30)
                
        except websockets.ConnectionClosed:
            # Clean up on disconnect
            self.cleanup_connection(device_id, user_id)
    
    async def send_initial_sync(self, device_id, user_id):
        """Send files that changed since last sync"""
        
        # Get device's last sync timestamp
        last_sync = get_device_last_sync(device_id)
        
        # Query files modified since last sync
        changed_files = query_db("""
            SELECT file_id, filename, file_hash, version, modified_at
            FROM files
            WHERE user_id = ? AND modified_at > ?
        """, user_id, last_sync)
        
        # Send to device
        await self.send_sync_message(device_id, {
            'type': 'initial_sync',
            'files': changed_files,
            'timestamp': now()
        })
```

-----

## **Part B: Change Detection & Propagation (90 seconds)**

### **How Changes Are Detected:**

```
CLIENT-SIDE CHANGE DETECTION:
═══════════════════════════════════════════

1. FILE SYSTEM WATCHER
   - OS-level hooks (inotify on Linux, FSEvents on Mac)
   - Detects: CREATE, MODIFY, DELETE, RENAME events
   
2. CHANGE DEBOUNCING
   - User saves file 10x in 5 seconds
   - Don't sync each save!
   - Wait for 2 seconds of inactivity
   - Then trigger sync

3. CALCULATE DELTA
   - Read modified file
   - Chunk it (content-defined chunking)
   - Compare hashes with last synced version
   - Identify changed chunks only

EXAMPLE:
File: presentation.pptx (100 MB)
User modifies slide 3 (affects 2 chunks: chunk[5], chunk[6])

OLD VERSION CHUNKS:
[chunk0][chunk1][chunk2]...[chunk5][chunk6]...[chunk24]
                            ↓       ↓
                         (old)    (old)

NEW VERSION CHUNKS:
[chunk0][chunk1][chunk2]...[chunk5'][chunk6']...[chunk24]
                            ↓        ↓
                          (new)    (new)

DELTA SYNC:
- Upload only: chunk5', chunk6' (8 MB instead of 100 MB!)
- Saves 92% bandwidth
```

-----

### **Server-Side Change Propagation:**

```python
async def handle_file_upload(user_id, file_id, chunks, version):
    """Handle file upload and propagate to other devices"""
    
    # 1. Validate version (optimistic locking)
    current_version = get_file_version(file_id)
    if version != current_version:
        # Conflict detected!
        return {'status': 'conflict', 'current_version': current_version}
    
    # 2. Store new chunks
    for chunk in chunks:
        store_chunk_if_new(chunk)
    
    # 3. Update metadata
    new_version = current_version + 1
    update_file_metadata(file​​​​​​​​​​​​​​​​_id, new_version, chunks)

    # 4. Publish change event to Kafka
    await kafka_producer.send('file.modified', {
        'user_id': user_id,
        'file_id': file_id,
        'version': new_version,
        'modified_by_device': request.device_id,
        'timestamp': now(),
        'changed_chunks': [c['hash'] for c in chunks]
    })

    # 5. Sync service consumes event and notifies devices
    # (happens asynchronously)

    return {'status': 'success', 'version': new_version}

# Sync service consumer
async def process_file_change_event(event):
    """Process file change and notify all user’s devices"""

    user_id = event['user_id']
    file_id = event['file_id']
    source_device = event['modified_by_device']

    # Get all devices for this user (except the source)
    target_devices = [
        d for d in get_user_devices(user_id) 
        if d != source_device
    ]

    # Send notification to each online device
    for device_id in target_devices:
        if device_id in self.connections:
            # Device is online - push immediately
            await self.send_sync_message(device_id, {
                'type': 'file_changed',
                'file_id': file_id,
                'version': event['version'],
                'changed_chunks': event['changed_chunks']
            })
        else:
            # Device offline - mark for next sync
            mark_device_needs_sync(device_id, file_id)
```

-----

## **Part C: Conflict Resolution (90 seconds)**

### **Conflict Scenarios:**

```
CONFLICT TYPES:
═══════════════════════════════════════════

1. SIMPLE CONFLICT (Most Common)
   Time: T0                T1              T2
   Device A: Edit file → Upload (v2)
   Device B:             Edit file    → Upload (v2) ❌
   
   Result: Device B has stale version!
   Resolution: Last Write Wins
1. SIMULTANEOUS EDITS (Rare but happens)
   Time: T0              T1
   Device A: Edit → Upload (arrives first)  → v2
   Device B: Edit → Upload (arrives second) → conflict!
   
   Result: Both edited same version
   Resolution: Create conflict copy
1. OFFLINE EDITING
   Device A: Online, edits file → v2, v3, v4
   Device B: Offline for 2 days, edits v1 → wants to upload
   
   Result: Device B is far behind
   Resolution: Create conflict copy + notify user
```

-----

### **Conflict Resolution Strategy:**

```
VERSIONING SYSTEM:
═══════════════════════════════════════════

Each file has:

- version_number: Integer, monotonically increasing
- last_modified_timestamp: For tie-breaking
- last_modified_by: Device ID

ALGORITHM:
┌─────────────────────────────────────────┐
│ On Upload Request:                      │
├─────────────────────────────────────────┤
│ IF client_version == server_version:    │
│    → Accept upload                      │
│    → Increment version                  │
│    → Notify other devices               │
│                                         │
│ ELSE IF client_version < server_version:│
│    → Reject upload                      │
│    → Return conflict error              │
│    → Client must resolve                │
│                                         │
│ ELSE: (shouldn’t happen)                │
│    → Log error                          │
│    → Manual intervention                │
└─────────────────────────────────────────┘

CLIENT-SIDE RESOLUTION:
┌─────────────────────────────────────────┐
│ On Conflict Error:                      │
├─────────────────────────────────────────┤
│ 1. Download latest version from server  │
│ 2. Save local version as:               │
│    “document (conflicted copy).txt”     │
│ 3. Show notification to user:           │
│    “Conflict detected - review copies”  │
│ 4. User manually merges if needed       │
└─────────────────────────────────────────┘
```

-----

### **Implementation with Vector Clocks (Advanced):**

```python
class VectorClock:
    """
    More sophisticated conflict detection
    Used by systems like Dropbox
    """
    def __init__(self):
        self.clock = {}  # device_id -> counter
    
    def increment(self, device_id):
        """Increment this device's counter"""
        self.clock[device_id] = self.clock.get(device_id, 0) + 1
    
    def merge(self, other_clock):
        """Merge two vector clocks (for conflict detection)"""
        for device_id, counter in other_clock.items():
            self.clock[device_id] = max(
                self.clock.get(device_id, 0),
                counter
            )
    
    def happens_before(self, other_clock):
        """Check if this clock happened before other"""
        # This < Other if all counters <= and at least one <
        all_lte = all(
            self.clock.get(d, 0) <= other_clock.get(d, 0)
            for d in set(self.clock) | set(other_clock)
        )
        any_lt = any(
            self.clock.get(d, 0) < other_clock.get(d, 0)
            for d in set(self.clock) | set(other_clock)
        )
        return all_lte and any_lt
    
    def is_concurrent(self, other_clock):
        """Check if two clocks are concurrent (conflict!)"""
        return (not self.happens_before(other_clock) and 
                not other_clock.happens_before(self))

# Usage in conflict detection
def detect_conflict(file_id, client_clock, client_chunks):
    """Detect conflicts using vector clocks"""
    
    # Get server's vector clock for this file
    server_clock = get_file_vector_clock(file_id)
    
    if client_clock.happens_before(server_clock):
        # Client is behind - reject upload
        return {
            'conflict': True,
            'resolution': 'reject',
            'message': 'File was modified by another device'
        }
    
    elif server_clock.happens_before(client_clock):
        # Client is ahead - accept upload (normal case)
        return {
            'conflict': False,
            'resolution': 'accept'
        }
    
    elif client_clock.is_concurrent(server_clock):
        # Concurrent edits - conflict!
        return {
            'conflict': True,
            'resolution': 'create_copy',
            'message': 'Conflicting changes detected'
        }

EXAMPLE:
Device A clock: {A: 5, B: 2, C: 1}
Device B clock: {A: 4, B: 3, C: 1}

Compare:
- A: 5 vs 4 (A ahead)
- B: 2 vs 3 (B ahead)
- C: 1 vs 1 (equal)

Result: Concurrent! → Conflict
```

-----

## **Part D: Real-Time Synchronization Performance (60 seconds)**

### **WebSocket Connection Scaling:**

```
CONNECTION SCALING:
═══════════════════════════════════════════

CHALLENGE: 100M DAU, assume 2 devices each
           = 200M concurrent WebSocket connections

SOLUTION: Connection Pooling per Sync Node

Each Sync Node (server):
- Handles 100,000 concurrent connections
- Requires ~200GB RAM (2KB per connection)
- Uses epoll/kqueue for efficient I/O

Total Sync Nodes Needed:
200M connections / 100K per node = 2,000 nodes

With 2x redundancy: 4,000 sync nodes

OPTIMIZATION: Not all users online simultaneously
- Peak online: 20% of DAU = 20M users
- Active connections: 40M (2 devices per user)
- Sync nodes needed: 400 nodes
- With 2x redundancy: 800 nodes
```

-----

### **Message Queue Throughput:**

```
KAFKA CONFIGURATION:
═══════════════════════════════════════════

Topics:
- file.created (low volume)
- file.modified (high volume)
- file.deleted (low volume)
- file.shared (medium volume)

Partitioning Strategy:
- Partition by user_id (hash-based)
- 1,000 partitions per topic
- Ensures all user's events ordered
- Parallel processing across consumers

Throughput:
- Upload QPS: 7,000 (peak)
- Each upload → 1 event
- Events per second: 7,000 EPS

Kafka capacity:
- Modern Kafka: 1M messages/sec per cluster
- Our need: 7K messages/sec
- Utilization: 0.7% (lots of headroom!)

Consumer Groups:
- Sync Service: 100 consumers
- Analytics: 50 consumers
- Each consumer handles ~70 events/sec
```

-----

### **Latency Breakdown:**

```
SYNC LATENCY TARGET: < 10 seconds
═══════════════════════════════════════════

ACTUAL LATENCY BREAKDOWN:
┌─────────────────────────────────────────┐
│ Step                        │ Time      │
├─────────────────────────────┼───────────┤
│ 1. File watcher detects     │ ~500ms    │
│    change (debouncing)      │           │
├─────────────────────────────┼───────────┤
│ 2. Calculate delta chunks   │ ~200ms    │
│    (for 10MB file)          │           │
├─────────────────────────────┼───────────┤
│ 3. Upload chunks to S3      │ ~2s       │
│    (network latency)        │           │
├─────────────────────────────┼───────────┤
│ 4. Update metadata DB       │ ~50ms     │
├─────────────────────────────┼───────────┤
│ 5. Publish to Kafka         │ ~10ms     │
├─────────────────────────────┼───────────┤
│ 6. Kafka → Sync Service     │ ~50ms     │
│    (consumer lag)           │           │
├─────────────────────────────┼───────────┤
│ 7. WebSocket push           │ ~100ms    │
│    to device                │           │
├─────────────────────────────┼───────────┤
│ 8. Device downloads delta   │ ~2s       │
│    chunks                   │           │
├─────────────────────────────┼───────────┤
│ 9. Device applies changes   │ ~200ms    │
│                             │           │
├─────────────────────────────┼───────────┤
│ TOTAL:                      │ ~5.1s ✓   │
└─────────────────────────────┴───────────┘

Result: Well under 10 second target!
```

-----

# **Summary of Deep Dive Components (30 seconds)**

**Quickly recap the three deep dives:**

> “Let me summarize the core components we’ve covered:

**1. File Chunking & Storage:**

- 4MB chunks using content-defined chunking
- SHA-256 hashing for deduplication
- Multi-tier storage (hot/warm/cold)
- Reference counting for garbage collection

**2. Metadata Management:**

- Sharded PostgreSQL by user_id
- 300TB metadata across 1,024 shards
- Redis caching with 90% hit rate
- TTL-based invalidation

**3. Synchronization:**

- WebSocket connections for real-time sync
- Kafka for event propagation
- Vector clocks for conflict detection
- < 5 second sync latency

These three components work together to provide fast, reliable file storage and synchronization at massive scale.”

-----

# **Transition Questions (15 seconds)**

**Ask the interviewer:**

> “We’ve covered the core technical components in depth. Would you like me to:
> 
> A) Dive into advanced features like sharing, permissions, or versioning?
> B) Discuss scalability bottlenecks and how to address them?
> C) Talk about security and encryption?
> D) Cover monitoring and observability?
> 
> Or is there a specific component you’d like me to elaborate on further?”

-----

# **Common Deep Dive Mistakes to Avoid:**

❌ **Too shallow** - “We use chunks” without explaining why or how

❌ **Too much code** - Don’t write full implementations, pseudo-code is enough

❌ **No trade-offs** - Every decision has pros/cons, mention them

❌ **Ignoring scale** - “We’ll use a single database” → Shows lack of experience

❌ **Over-engineering** - Don’t propose blockchain for conflict resolution

❌ **No numbers** - “It’ll be fast” vs “5 second latency under p99”

❌ **Forgetting edge cases** - What if user uploads same file 1000x?

❌ **No failure scenarios** - What happens when Kafka is down?

-----

# **Pro Tips for Deep Dives:**

✅ **Use real-world examples:**

- “Similar to how Dropbox handles this…”
- “Amazon S3 uses this approach…”
- Shows you’ve studied production systems

✅ **Quantify everything:**

- Not “fast” but “< 100ms p99 latency”
- Not “lots of connections” but “100K per node”

✅ **Draw while explaining:**

- Schemas on whiteboard
- Data flow diagrams
- State transitions

✅ **Acknowledge alternatives:**

- “We could use MySQL or Cassandra. MySQL gives us ACID…”
- Shows breadth of knowledge

✅ **Connect to earlier sections:**

- “Remember we calculated 700K QPS? That’s why we need this caching layer”
- Shows coherent thinking

✅ **Prepare for follow-ups:**

- Interviewer might ask: “What if deduplication has hash collision?”
- Have answers ready for obvious questions

✅ **Be honest about limitations:**

- “This approach works for files under 10GB. For larger files, we’d need to…”
- Better than pretending you’ve solved everything

-----

# **What Your Whiteboard Should Look Like After Deep Dives:**

```
┌─────────────────────────────────────────────────────────┐
│ DEEP DIVE: FILE CHUNKING                                │
├─────────────────────────────────────────────────────────┤
│ • 4MB chunks (content-defined, Rabin fingerprinting)    │
│ • SHA-256 for deduplication (67% storage savings)       │
│ • Reference counting in chunks table                    │
│ • Multi-tier: Hot (S3) → Warm (IA) → Cold (Glacier)     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ DEEP DIVE: METADATA DB                                  │
├─────────────────────────────────────────────────────────┤
│ • PostgreSQL sharded by user_id (1,024 shards)          │
│ • Schema: users, files, chunks, file_chunks, versions   │
│ • Redis cache (90% hit rate, 5 min TTL)                 │
│ • Reduces DB load: 1.15M → 115K QPS                     │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ DEEP DIVE: SYNC SERVICE                                 │
├─────────────────────────────────────────────────────────┤
│ • WebSocket connections (800 nodes, 100K conn/node)     │
│ • Kafka for events (7K EPS, 1000 partitions)            │
│ • Vector clocks for conflict detection                  │
│ • Total latency: ~5 seconds (target: <10s) ✓            │
└─────────────────────────────────────────────────────────┘
```

-----

# **Interviewer Signals to Watch:**

**Good signs:**

- 😊 Nodding along
- 🤔 Asking detailed follow-ups
- ✍️ Taking notes
- 💬 “Interesting approach…”
- ⏰ Letting you talk longer than planned

**Warning signs:**

- 😐 Blank stare (you’ve lost them)
- ⏭️ “Let’s move on…” (they’re not interested)
- 📱 Checking time frequently (too slow)
- 🤨 “Are you sure about that?” (you made an error)

**Adjust accordingly:**

- If lost → slow down, use simpler examples
- If bored → speed up, move to next component
- If engaged → go deeper, show more expertise

-----

# **Time Check:**

At 35 minutes, you should have:

- ✅ Gathered requirements (5 min)
- ✅ Done capacity estimates (5 min)
- ✅ Drawn high-level design (10 min)
- ✅ Deep-dived 3 components (15 min)

**Remaining: 25 minutes**

**What’s next:**

- Advanced features (10 min)
- Scalability & bottlenecks (7 min)
- Trade-offs (5 min)
- Wrap-up (3 min)

You’re right on schedule! 🎯​​​​​​​​​​​​​​​​
