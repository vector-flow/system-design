# Minutes 6-10: Back-of-Envelope Calculations (5 min)

## **Why This Matters**

Back-of-envelope calculations demonstrate:

- **Quantitative thinking** - You make data-driven decisions
- **Reality checking** - Your design must handle actual scale
- **Bottleneck identification** - Where will the system break?
- **Cost awareness** - Understanding infrastructure implications

**Golden Rule:** These estimates guide your architectural choices. Get the order of magnitude right—exact precision doesn’t matter.

-----

## **Your Opening (15 seconds)**

**What to say:**

> “Now let me do some quick capacity estimates to understand the scale we’re dealing with. I’ll calculate storage, bandwidth, and throughput requirements. These numbers will help guide our design decisions.”

-----

## **1. Storage Capacity Estimation (90 seconds)**

### **Total Storage Calculation:**

**Start with user data:**

```
Total Users:              500M
Average storage per user: 10GB
─────────────────────────────────
Total user data:          500M × 10GB = 5,000 PB = 5 EB
```

**Add replication factor:**

```
Replication factor:       3x (for durability)
Total with replication:   5 EB × 3 = 15 EB
```

**Add metadata overhead:**

```
Metadata (~2% of data):   15 EB × 0.02 = 300 PB
─────────────────────────────────
Total storage needed:     ~15.3 EB
```

### **What to say:**

> “So we’re looking at around 15 exabytes of total storage. That’s massive and immediately tells us we need distributed object storage like S3 or similar. We can’t use traditional file systems at this scale.”

-----

## **2. Traffic & Throughput Estimation (90 seconds)**

### **Daily Active Users (DAU) Activity:**

**Upload calculations:**

```
DAU:                           100M users
Files uploaded per user/day:   2 files
Average file size:             10MB
──────────────────────────────────────────
Daily upload volume:           100M × 2 × 10MB = 2 PB/day
Upload requests per day:       200M requests
Upload requests per second:    200M / 86,400 = ~2,300 QPS
```

**Download calculations (100:1 read ratio):**

```
Download requests per day:     200M × 100 = 20B requests
Download requests per second:  20B / 86,400 = ~231,000 QPS
Daily download volume:         2 PB × 100 = 200 PB/day
```

### **Peak Traffic (3x average):**

```
Peak upload QPS:     2,300 × 3 = ~7,000 QPS
Peak download QPS:   231,000 × 3 = ~700,000 QPS
```

### **What to say:**

> “We’re handling around 700K read requests per second at peak. This is read-heavy, so we’ll need aggressive caching, CDNs, and read replicas.”

-----

## **3. Bandwidth Estimation (60 seconds)**

### **Upload bandwidth:**

```
Average uploads:       2,300 QPS × 10MB = 23 GB/s = 184 Gbps
Peak uploads:          7,000 QPS × 10MB = 70 GB/s = 560 Gbps
```

### **Download bandwidth:**

```
Average downloads:     231,000 QPS × 10MB = 2,310 GB/s = 18.5 Tbps
Peak downloads:        700,000 QPS × 10MB = 7,000 GB/s = 56 Tbps
```

### **What to say:**

> “Peak download bandwidth is 56 terabits per second. This reinforces the need for CDN distribution—we can’t serve this from centralized data centers alone.”

-----

## **4. Metadata Storage Estimation (60 seconds)**

### **File metadata calculation:**

**Assume metadata schema per file:**

```
File record size estimate:
- File ID:           16 bytes (UUID)
- User ID:           8 bytes
- File name:         256 bytes (max)
- File path:         512 bytes
- File size:         8 bytes
- Hash:              32 bytes (SHA-256)
- Created time:      8 bytes
- Modified time:     8 bytes
- Permissions:       64 bytes
- Version info:      32 bytes
- Misc metadata:     56 bytes
──────────────────────────────────
Total per file:      ~1 KB
```

**Total metadata storage:**

```
Total users:             500M
Files per user:          200
Total files:             500M × 200 = 100B files
──────────────────────────────────────────────
Metadata storage:        100B × 1KB = 100 TB
With indexes (3x):       100 TB × 3 = 300 TB
```

### **What to say:**

> “Metadata is relatively small at 300TB. This can fit in a distributed SQL database with sharding, or we could use a NoSQL solution like Cassandra.”

-----

## **5. Chunk Storage Analysis (45 seconds)**

### **File chunking calculations:**

**Why chunk files?**

- Efficient delta sync
- Deduplication
- Resume capability
- Parallel uploads

**Chunk size analysis:**

```
Chosen chunk size:       4 MB (industry standard)
Average file size:       10 MB
Chunks per file:         10MB / 4MB = ~3 chunks

Total files:             100B
Total chunks:            100B × 3 = 300B chunks
```

**Chunk metadata:**

```
Metadata per chunk:      
- Chunk ID:              16 bytes
- File ID:               16 bytes
- Hash:                  32 bytes
- Sequence:              4 bytes
- Size:                  4 bytes
──────────────────────────────────
Total per chunk:         ~72 bytes

Total chunk metadata:    300B × 72 bytes = 21.6 TB
```

### **What to say:**

> “With 4MB chunks, we have 300 billion chunks. Chunk metadata is manageable but we need efficient indexing for deduplication lookups.”

-----

## **6. Database QPS Estimation (30 seconds)**

### **Metadata database queries:**

```
Each upload involves:
- 1 file metadata write
- ~3 chunk metadata writes
- 1 user quota check
- Permission checks
──────────────────────────────────
Total writes per upload:  ~6 queries

Upload QPS:               2,300
DB write QPS:             2,300 × 6 = 13,800 QPS
```

```
Each download involves:
- 1 file metadata read
- ~3 chunk metadata reads
- 1 permission check
──────────────────────────────────
Total reads per download: ~5 queries

Download QPS:             231,000
DB read QPS:              231,000 × 5 = 1.15M QPS
```

**With caching (90% hit rate):**

```
Actual DB read QPS:       1.15M × 0.1 = 115,000 QPS
```

### **What to say:**

> “Even with 90% cache hit rate, we need 115K database read QPS. We’ll need read replicas and aggressive caching of metadata.”

-----

## **7. Number of Servers Estimation (45 seconds)**

### **Application servers:**

**Assumptions:**

- Each server handles 1,000 QPS
- Need headroom for peak traffic

```
Peak total QPS:          700,000 (downloads) + 7,000 (uploads) = 707,000
Servers needed:          707,000 / 1,000 = 707 servers
With 2x redundancy:      707 × 2 = ~1,500 servers
```

### **Storage servers:**

```
Total storage:           15 EB
Storage per server:      100 TB (with large HDDs)
──────────────────────────────────────────────
Servers needed:          15,000 PB / 100 TB = 150,000 servers
```

**Reality check:**

> “In practice, we’d use object storage services (S3, GCS) that abstract this. But this shows the massive scale—equivalent to 150,000 physical servers.”

-----

## **Summary Table (30 seconds)**

**Create this visual on whiteboard:**

```
CAPACITY ESTIMATES
═══════════════════════════════════════════════
STORAGE
  User data:           5 EB
  With replication:    15 EB
  Metadata:            300 TB
  
TRAFFIC (Peak)
  Upload QPS:          7,000
  Download QPS:        700,000
  Total QPS:           707,000
  
BANDWIDTH (Peak)
  Upload:              560 Gbps
  Download:            56 Tbps
  
DATABASE
  Write QPS:           13,800
  Read QPS (cached):   115,000
  Metadata storage:    300 TB
  
FILES
  Total files:         100B
  Total chunks:        300B
  Chunk size:          4 MB

SERVERS
  App servers:         ~1,500
  Storage (equiv):     ~150,000
```

-----

## **Key Insights to Highlight (30 seconds)**

**After calculations, explicitly state the implications:**

> “Based on these numbers, here are the key architectural decisions:
> 
> 1. **Must use object storage** - 15 EB is beyond traditional file systems
> 1. **CDN is critical** - 56 Tbps peak bandwidth requires edge distribution
> 1. **Metadata caching essential** - 115K DB QPS even with caching means we need Redis/Memcached
> 1. **Sharding required** - Single database can’t handle this scale
> 1. **Chunking is beneficial** - Enables deduplication, parallel uploads, and efficient syncing”

-----

## **Common Calculation Mistakes to Avoid:**

❌ **Wrong units:** Mixing MB/GB/TB/PB - be consistent and convert clearly

❌ **Forgetting replication:** Storage is always 3x+ for durability

❌ **Ignoring peak traffic:** Average QPS doesn’t reveal bottlenecks

❌ **No cache assumptions:** Real systems cache heavily—factor this in

❌ **Unrealistic numbers:** “Each user uploads 1000 files/day” - sanity check!

❌ **Too much precision:** “2,314.28 QPS” - round to 2,300, easier to work with

❌ **Skipping metadata:** Often forgotten but critical for system design

❌ **Not showing work:** Write formulas clearly so interviewer can follow

-----

## **Pro Tips for This Section:**

### **1. Use Standard Numbers:**

```
1 million = 10^6 = 1M
1 billion = 10^9 = 1B
1 KB = 10^3 bytes (or 2^10 for precise)
1 MB = 10^6 bytes
1 GB = 10^9 bytes
1 TB = 10^12 bytes
1 PB = 10^15 bytes

1 day = 86,400 seconds ≈ 100,000 (for rough calcs)
```

### **2. Common Approximations:**

- **Cache hit rate:** 80-90%
- **Replication factor:** 3x
- **Peak to average ratio:** 2-3x
- **Metadata overhead:** 1-2% of data
- **Server capacity:** 1,000-10,000 QPS per server

### **3. Validate Your Numbers:**

> “Does 700K QPS sound reasonable for this scale?”
> 
> Invite the interviewer to correct you—shows collaboration.

### **4. Connect to Real Systems:**

> “For context, Netflix serves about 100M hours of video daily, which is similar magnitude of bandwidth…”

Shows you understand real-world systems.

-----

## **What Your Whiteboard Should Look Like:**

```
┌─────────────────────────────────────────┐
│ BACK-OF-ENVELOPE CALCULATIONS           │
├─────────────────────────────────────────┤
│ STORAGE:                                │
│   500M users × 10GB = 5 EB              │
│   × 3 replication = 15 EB               │
│                                         │
│ TRAFFIC:                                │
│   Uploads:  2,300 QPS (7K peak)         │
│   Downloads: 231K QPS (700K peak)       │
│                                         │
│ BANDWIDTH:                              │
│   Peak download: 56 Tbps → CDN needed!  │
│                                         │
│ DATABASE:                               │
│   Metadata: 300 TB                      │
│   Read QPS: 115K (w/ 90% cache)         │
│   → Need sharding + caching             │
│                                         │
│ KEY INSIGHTS:                           │
│   ✓ Object storage mandatory            │
│   ✓ CDN critical for bandwidth          │
│   ✓ Heavy caching required              │
│   ✓ Database sharding needed            │
└─────────────────────────────────────────┘
```

-----

## **Transition to Next Section:**

> “Now that we understand the scale—15 exabytes of storage, 700K QPS at peak, and 56 terabits of bandwidth—let me sketch out a high-level architecture that can handle these requirements…”

**This creates a natural bridge to the design phase while showing your calculations inform your architecture!**
