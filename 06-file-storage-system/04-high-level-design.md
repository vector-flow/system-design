# Minutes 11-20: High-Level Design (10 min)

## **Why This Matters**

The high-level design is your **architectural blueprint**. This is where you:

- Show you can design distributed systems
- Demonstrate understanding of component interactions
- Prove you can communicate complex ideas visually
- Set the foundation for deep dives later

**Golden Rule:** Breadth over depth here. Cover all major components, then deep dive later.

-----

## **Your Opening (15 seconds)**

**What to say:**

> “Let me start with a high-level architecture that addresses our scale requirements. I’ll draw the major components first, then walk through the key flows: file upload, download, and synchronization. Sound good?”

-----

## **Phase 1: Draw Core Architecture (3 minutes)**

### **Step 1: Draw from Client to Server (60 seconds)**

**Start drawing on the whiteboard:**

```
┌─────────────────────────────────────────────────────────┐
│                        CLIENTS                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │   Web    │  │  Mobile  │  │ Desktop  │               │
│  │ Browser  │  │   App    │  │  Client  │               │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘               │
└───────┼─────────────┼─────────────┼───────────────-─────┘
        │             │             │
        └─────────────┼─────────────┘
                      │
                      ▼
              ┌───────────────┐
              │ Load Balancer │
              │   (Layer 7)   │
              └───────┬───────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
   ┌────────┐    ┌────────┐    ┌────────┐
   │  API   │    │  API   │    │  API   │
   │ Server │    │ Server │    │ Server │
   └────────┘    └────────┘    └────────┘
```

**Narrate as you draw:**

> “Users access the system through web, mobile, or desktop clients. All requests go through load balancers for distribution, which route to our stateless API servers.”

-----

### **Step 2: Add Backend Storage Layer (90 seconds)**

**Continue drawing:**

```
   ┌────────┐    ┌────────┐    ┌────────┐
   │  API   │    │  API   │    │  API   │
   │ Server │    │ Server │    │ Server │
   └───┬────┘    └───┬────┘    └───┬────┘
       │             │             │
       ├─────────────┼─────────────┤
       │             │             │
       ▼             ▼             ▼
   ┌─────────────────────────────────────┐
   │         METADATA CACHE              │
   │    (Redis/Memcached Cluster)        │
   └──────────────┬──────────────────────┘
                  │
                  ▼
   ┌─────────────────────────────────────┐
   │       METADATA DATABASE             │
   │    (MySQL/PostgreSQL Sharded)       │
   │  ┌──────┐  ┌──────┐  ┌──────┐       │
   │  │Shard1│  │Shard2│  │Shard3│       │
   │  └──────┘  └──────┘  └──────┘       │
   └─────────────────────────────────────┘

   ┌─────────────────────────────────────┐
   │      BLOCK/CHUNK STORAGE            │
   │   (Amazon S3 / Object Storage)      │
   │                                     │
   │  Distributed across multiple AZs    │
   └─────────────────────────────────────┘
```

**Narrate:**

> “API servers interact with two main storage systems: metadata database for file information, and object storage for actual file chunks. We use Redis for metadata caching to reduce database load.”

-----

### **Step 3: Add Supporting Services (90 seconds)**

**Add these components to the side:**

```
┌───────────────────────────────┐
│   SUPPORTING SERVICES         │
├───────────────────────────────┤
│                               │
│  ┌─────────────────────────┐  │
│  │ Synchronization Service │  │
│  │  - Change detection     │  │
│  │  - Conflict resolution  │  │
│  └─────────────────────────┘  │
│                               │
│  ┌─────────────────────────┐  │
│  │ Notification Service    │  │
│  │  - WebSocket/Long Poll  │  │
│  │  - Push notifications   │  │
│  └─────────────────────────┘  │
│                               │
│  ┌─────────────────────────┐  │
│  │  Message Queue          │  │
│  │  (Kafka/RabbitMQ)       │  │
│  └─────────────────────────┘  │
│                               │
│  ┌─────────────────────────┐  │
│  │  Analytics & Logging    │  │
│  └─────────────────────────┘  │
│                               │
└───────────────────────────────┘
```

**Narrate:**

> “We need supporting services: sync service to detect changes across devices, notification service for real-time updates, message queues for async processing, and analytics for monitoring.”

-----

### **Step 4: Add CDN Layer (30 seconds)**

**Draw at the top:**

```
                 CLIENTS
                    │
                    ▼
         ┌──────────────────┐
         │   CDN (Global)   │
         │  - CloudFront    │
         │  - Cloudflare    │
         └────────┬─────────┘
                  │
         (for hot/popular files)
                  │
          ┌───────┴────────┐
          │                │
     Load Balancer    Object Storage
```

**Narrate:**

> “For frequently accessed files, we’ll use a CDN to cache content closer to users, reducing latency and bandwidth on our origin servers.”

-----

## **Phase 2: Complete Architecture Diagram (30 seconds)**

### **Your Final Whiteboard Should Look Like This:**

```
┌─────────────────────────────────────────────────────────────────┐
│                          CLIENTS                                │
│        [Web Browser]  [Mobile App]  [Desktop Client]            │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │   CDN Layer    │
                    │  (CloudFront)  │
                    └───────┬────────┘
                            │
                    ┌───────┴────────┐
                    │ Load Balancer  │
                    └───────┬────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
   ┌────────┐          ┌────────┐         ┌────────┐
   │  API   │          │  API   │         │  API   │
   │ Server │          │ Server │         │ Server │
   └───┬────┘          └───┬────┘         └───┬────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
       ▼                   ▼                   ▼
┌──────────────┐    ┌─────────────┐    ┌─────────────┐
│ Metadata     │    │   Sync      │    │ Notification│
│ Cache        │◄───┤  Service    │───►│  Service    │
│ (Redis)      │    │             │    │ (WebSocket) │
└──────┬───────┘    └─────────────┘    └─────────────┘
       │                   │
       ▼                   ▼
┌──────────────┐    ┌─────────────┐
│ Metadata DB  │    │   Message   │
│  (Sharded    │    │   Queue     │
│  PostgreSQL) │    │  (Kafka)    │
└──────────────┘    └─────────────┘
       
┌─────────────────────────────────────-────┐
│     Block Storage (S3/Object Store)      │
│  - Distributed across availability zones │
│  - 3x replication for durability         │
└────────────────────────────────────-─────┘
```

-----

## **Phase 3: Core Data Flow - File Upload (2 minutes)**

### **Walk Through Upload Flow Step-by-Step:**

**Draw numbered arrows on your diagram as you explain:**

> “Let me walk through what happens when a user uploads a file:”

```
1. CLIENT → API SERVER
   "User selects file (e.g., 'vacation.jpg', 50MB)"
   
2. API SERVER (File Processing)
   - Break file into 4MB chunks
   - Calculate hash for each chunk (SHA-256)
   - Check for deduplication
   
3. API SERVER → METADATA DB
   - Query: "Does chunk hash XYZ already exist?"
   - If yes: Skip upload (deduplication)
   - If no: Proceed with upload
   
4. API SERVER → OBJECT STORAGE
   - Upload new chunks to S3
   - Chunks stored as: 
     s3://bucket/user123/chunks/abc123def456
   
5. API SERVER → METADATA DB
   - Write file metadata:
     * file_id, user_id, filename
     * file_size, created_time
     * chunk_ids[] array
   - Write chunk metadata:
     * chunk_id, hash, size, offset
   
6. API SERVER → MESSAGE QUEUE
   - Publish event: "File uploaded by user123"
   
7. SYNC SERVICE (subscribes to queue)
   - Receives upload event
   - Identifies user's other devices
   - Marks file for sync
   
8. NOTIFICATION SERVICE
   - Push notification to user's devices
   - "vacation.jpg uploaded from Desktop"
```

**Key points to emphasize:**

> “Notice a few important things:
> 
> - **Chunking** happens client-side or at API server
> - **Deduplication** via hash lookup saves storage
> - **Async processing** via message queue for sync
> - **Metadata and data** stored separately”

-----

## **Phase 4: Core Data Flow - File Download (90 seconds)**

### **Walk Through Download Flow:**

```
1. CLIENT → API SERVER
   "User clicks 'Download vacation.jpg'"
   
2. API SERVER → METADATA CACHE (Redis)
   - Query: "Get metadata for file_id=456"
   - Cache hit (90% of time): Return immediately
   - Cache miss: Query database, then cache result
   
3. METADATA CACHE/DB → API SERVER
   Returns:
   {
     file_id: 456,
     filename: "vacation.jpg",
     size: 50MB,
     chunks: [
       {chunk_id: "abc123", hash: "xyz", size: 4MB, offset: 0},
       {chunk_id: "def456", hash: "uvw", size: 4MB, offset: 4MB},
       ...
     ]
   }
   
4. API SERVER (Decision Point)
   - Check if file is "hot" (frequently accessed)
   - If hot: Redirect to CDN URL
   - If cold: Generate presigned S3 URL
   
5a. CDN PATH (Hot Files)
    CLIENT → CDN → (cache miss) → S3
    - Subsequent requests served from CDN edge
    
5b. DIRECT PATH (Cold Files)
    CLIENT → S3 (presigned URL)
    - Direct download from object storage
    
6. CLIENT (Reassembly)
   - Download chunks in parallel
   - Verify each chunk hash
   - Reassemble file locally
```

**Key optimization to highlight:**

> “For downloads, we use presigned URLs so clients can fetch directly from S3/CDN without going through our API servers. This saves massive bandwidth on our infrastructure.”

-----

## **Phase 5: Core Data Flow - Device Synchronization (2 minutes)**

### **Walk Through Sync Flow:**

**This is the most complex flow—draw it carefully:**

```
SCENARIO: User edits file on Device A, needs to sync to Device B

Device A (Desktop)                 Backend                Device B (Mobile)
     │                                │                         │
     │  1. File modified locally      │                         │
     │  "document.txt" changed        │                         │
     │                                │                         │
     │  2. Sync Client detects change │                         │
     │  (File watcher monitors dirs)  │                         │
     │                                │                         │
     │  3. Calculate delta            │                         │
     │  (Only modified chunks)        │                         │
     │                                │                         │
     │ 4. Upload delta chunks         │                         │
     ├───────────────────────────────►│                         │
     │    POST /api/sync              │                         │
     │    {file_id, chunks, version}  │                         │
     │                                │                         │
     │                        5. Validate version               │
     │                        (Optimistic locking)              │
     │                        IF version = expected:            │
     │                          Accept update                   │
     │                        ELSE:                             │
     │                          Conflict! (handle later)        │
     │                                │                         │
     │                        6. Update metadata DB             │
     │                        - New version number              │
     │                        - Updated chunk refs              │
     │                                │                         │
     │                        7. Publish to queue               │
     │                        Event: "File changed"             │
     │                                │                         │
     │                        8. Sync Service processes         │
     │                                │                         │
     │                                │  9. Query: "Which       │
     │                                │     devices need sync?" │
     │                                │                         │
     │                                │  10. Send notification  │
     │                                ├────────────────────────►│
     │                                │   WebSocket push        │
     │                                │   "document.txt updated"│
     │                                │                         │
     │                                │                      11. Device B
     │                                │                      requests delta
     │                                │◄────────────────────────┤
     │                                │  GET /api/sync?since=v5 │
     │                                │                         │
     │                                │  12. Return changed     │
     │                                │      chunks only        │
     │                                ├────────────────────────►│
     │                                │  {chunks: [...], v: 6}  │
     │                                │                         │
     │                                │                      13. Download
     │                                │                      new chunks
     │                                │                         │
     │                                │                      14. Apply
     │                                │                      changes
     │                                │                      locally
```

**Key sync mechanisms to explain:**

### **A. Change Detection (30 seconds)**

> “Clients detect changes through file system watchers. When a file changes, we:
> 
> 1. Calculate which chunks changed (delta detection)
> 1. Only upload modified chunks, not entire file
> 1. Include version number for conflict detection”

### **B. Notification Mechanism (30 seconds)**

> “For real-time sync, we use persistent connections:
> 
> - **WebSockets** for web/desktop clients
> - **Long polling** as fallback
> - **Push notifications** for mobile (APNs/FCM)
> 
> When a file changes, Sync Service immediately notifies all connected devices.”

### **C. Conflict Resolution (30 seconds)**

> “When two devices modify the same file simultaneously:
> 
> **Strategy 1: Last Write Wins (Simple)**
> 
> - Use version numbers
> - Later timestamp wins
> - Losing edit becomes conflict copy
> 
> **Strategy 2: Operational Transform (Complex)**
> 
> - Merge concurrent edits if possible
> - Used by Google Docs
> - Overkill for simple file storage
> 
> We’ll use Last Write Wins with conflict copies for simplicity.”

-----

## **Phase 6: Key Design Decisions & Trade-offs (90 seconds)**

### **Explicitly Call Out Your Design Choices:**

**1. Chunking Strategy**

```
Decision: 4MB chunk size
Why:
✓ Good balance for network efficiency
✓ Enables partial sync (delta updates)
✓ Allows deduplication
✗ Small overhead for tiny files (<4MB)
```

**2. Metadata vs Data Separation**

```
Decision: Separate metadata DB and object storage
Why:
✓ Different access patterns (metadata: frequent, small)
✓ Different scaling needs
✓ SQL for metadata (structured queries)
✓ Object storage for chunks (cheap, durable)
```

**3. Sync Architecture**

```
Decision: Push-based sync with persistent connections
Why:
✓ Real-time updates (< 10 sec requirement)
✓ Better UX than polling
✗ More complex infrastructure
✗ Connection management overhead

Alternative: Pull-based (polling every 30s)
✓ Simpler implementation
✗ Higher latency
✗ Wastes resources on empty polls
```

**4. Consistency Model**

```
Decision: Eventual consistency with version vectors
Why:
✓ Better availability (CAP theorem)
✓ Handles network partitions gracefully
✓ Acceptable for file storage use case
✗ Potential for conflicts (rare)
```

-----

## **Phase 7: Technology Choices Justification (60 seconds)**

### **Quickly Justify Your Stack:**

**Present this table:**

```
COMPONENT            TECHNOLOGY          WHY?
─────────────────────────────────────────────────────────
API Servers         Node.js/Go          - High concurrency
                                        - Async I/O
                                        - Fast for I/O-bound ops

Metadata DB         PostgreSQL          - ACID transactions
                    (sharded)           - Rich querying
                                        - Proven at scale

Metadata Cache      Redis               - In-memory speed
                                        - 100K+ QPS per node
                                        - TTL support

Object Storage      AWS S3/GCS          - Infinite scale
                                        - 99.999999999% durability
                                        - Cost-effective

Message Queue       Kafka               - High throughput
                                        - Persistent
                                        - Replay capability

Sync Service        Custom (Go)         - Stateful connections
                                        - WebSocket support
                                        - Efficient resource use

CDN                 CloudFront          - Global edge presence
                                        - Low latency
                                        - DDoS protection
```

**Narrate:**

> “I’ve chosen technologies based on our requirements: PostgreSQL for metadata gives us ACID properties and rich queries. S3 for storage is proven at exabyte scale. Redis caching reduces database load by 90%. Kafka handles async processing reliably.”

-----

## **Phase 8: Scalability & Fault Tolerance Preview (60 seconds)**

### **Briefly Touch on How System Scales:**

> “Let me quickly highlight how this scales:

**Horizontal Scaling:**

- API servers: Stateless, add more behind load balancer
- Metadata DB: Shard by user_id (consistent hashing)
- Object storage: S3 handles this automatically
- Cache: Redis cluster with replication

**Fault Tolerance:**

- Multiple availability zones for all services
- 3x replication for object storage
- Database replicas (master-slave)
- Circuit breakers and retry logic

**Bottlenecks We’ll Address:**

- Metadata DB hot shards → will discuss sharding strategy
- Sync service connection limits → will cover scaling WebSockets
- Network bandwidth → CDN and compression

I’ll dive deeper into these in the next section.”

-----

## **Summary Checklist (30 seconds)**

### **Before Moving On, Verify You’ve Covered:**

**Quick verbal summary:**

> “To recap our high-level design:
> 
> ✓ Clients connect via load balancer to stateless API servers
> ✓ Metadata stored in sharded PostgreSQL with Redis caching
> ✓ File chunks stored in S3 with 4MB chunk size
> ✓ Sync service with WebSockets for real-time updates
> ✓ CDN for frequently accessed files
> ✓ Message queue for async processing
> ✓ Handles 700K QPS and 15 EB of storage
> 
> Which component would you like me to dive deeper into?”

-----

## **Common Mistakes to Avoid:**

❌ **Drawing too detailed too early** - Keep it high-level, resist diving into implementation

❌ **Forgetting to show data flow** - Architecture diagram alone isn’t enough

❌ **Not labeling components** - “Database” vs “Metadata Database (PostgreSQL)”

❌ **Skipping sync explanation** - This is core to the system, must explain

❌ **No technology justification** - Don’t just say “Redis,” explain why Redis

❌ **Ignoring the interviewer** - Pause periodically: “Does this make sense so far?”

❌ **Messy whiteboard** - Use consistent shapes, clear arrows, neat handwriting

❌ **No trade-offs mentioned** - Every design decision has pros/cons

-----

## **Pro Tips:**

✅ **Draw incrementally** - Build up the diagram piece by piece, don’t draw everything at once

✅ **Use consistent notation:**

- Rectangles for services
- Cylinders for databases
- Clouds for storage
- Arrows with labels for data flow

✅ **Color code if possible:**

- Blue for read paths
- Red for write paths
- Green for async flows

✅ **Number the flows** - “Step 1, Step 2…” makes it easy to follow

✅ **Keep it clean** - Leave space for annotations, don’t cram

✅ **Engage the interviewer:**

- “Would you like me to explain the sync flow in more detail?”
- “Are you comfortable with this level of detail, or should I go deeper?”

✅ **Reference your calculations:**

- “Remember we calculated 700K QPS? That’s why we need caching here.”

-----

## **What Success Looks Like:**

By the end of this 10 minutes, your interviewer should:

1. ✅ Understand the complete system architecture
1. ✅ See how upload/download/sync flows work
1. ✅ Know why you made key technology choices
1. ✅ Have confidence you can design at scale
1. ✅ Be ready to dive into specific components

**The interviewer should be nodding along, not confused. If they look lost, slow down and clarify before proceeding!**

-----

## **Transition to Next Phase:**

> “This gives us a solid high-level architecture that handles our scale requirements. Now I’d like to deep dive into the most critical components. Based on our discussion, I think we should focus on:
> 
> 1. **File chunking and storage** - How we optimize for deduplication
> 1. **Metadata database design** - Schema and sharding strategy
> 1. **Synchronization service** - Real-time sync and conflict resolution
> 
> Which of these interests you most, or is there another area you’d like to explore?”

**Let the interviewer guide which deep dive to pursue—shows flexibility and collaboration!**
