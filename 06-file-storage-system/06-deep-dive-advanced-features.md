# Minutes 36-45: Deep Dive - Advanced Features (10 min)

## **Why This Matters**

Advanced features demonstrate:

- **Senior-level thinking** - Beyond basic CRUD operations
- **Production experience** - Real systems need these features
- **User empathy** - Understanding what makes a product great
- **System complexity** - How features interact and impact architecture

**Golden Rule:** Pick 2-3 features based on interviewer interest. Don’t try to cover everything.

-----

## **Your Opening (15 seconds)**

**What to say:**

> “We’ve covered the core storage and sync architecture. Now let’s discuss advanced features that make this a production-ready system. I’ll focus on three areas: data consistency and reliability, security and access control, and performance optimizations. Which interests you most, or should I cover all three?”

**Wait for interviewer response, then prioritize accordingly.**

-----

# **ADVANCED FEATURE 1: Data Consistency & Reliability (3.5 minutes)**

## **Part A: Replication Strategy (90 seconds)**

### **Multi-Level Replication:**

```
REPLICATION ARCHITECTURE:
═══════════════════════════════════════════════════

LEVEL 1: CHUNK STORAGE REPLICATION (S3)
┌─────────────────────────────────────────────┐
│  US-EAST-1a          US-EAST-1b             │
│  ┌──────────┐       ┌──────────┐            │
│  │ Chunk A  │──────►│ Chunk A  │ (replica)  │
│  │ (primary)│       │          │            │
│  └──────────┘       └──────────┘            │
│                                             │
│              US-EAST-1c                     │
│              ┌──────────┐                   │
│              │ Chunk A  │ (replica)         │
│              │          │                   │
│              └──────────┘                   │
└─────────────────────────────────────────────┘

Configuration:
- 3x replication within region (99.999999999% durability)
- Cross-region replication for disaster recovery
- Versioning enabled (immutable chunks)

LEVEL 2: METADATA DATABASE REPLICATION
┌─────────────────────────────────────────────┐
│  Master DB (Shard 0)                        │
│  ┌────────────────┐                         │
│  │ Metadata       │                         │
│  │ (Read + Write) │                         │
│  └────────┬───────┘                         │
│           │                                 │
│           ├──────────────┐                  │
│           │              │                  │
│           ▼              ▼                  │
│  ┌────────────┐   ┌────────────┐            │
│  │ Read Replica│   │Read Replica│           │
│  │  (Async)   │   │  (Async)   │            │
│  └────────────┘   └────────────┘            │
└─────────────────────────────────────────────┘

Configuration:
- 1 master + 2 read replicas per shard
- Async replication (< 100ms lag)
- Automatic failover with Consul/etcd
- Read replicas for read scaling
```

-----

### **Consistency Model:**

```
CAP THEOREM CHOICE:
═══════════════════════════════════════════════════

Given: Storage system spanning multiple data centers

We choose: AP (Availability + Partition Tolerance)
           over Consistency

WHY?
✓ Users expect access even during network partitions
✓ File storage can tolerate eventual consistency
✓ Conflicts are rare and resolvable
✗ Strict consistency would sacrifice availability

CONSISTENCY LEVELS BY OPERATION:
┌───────────────────────────────────────────────┐
│ Operation          │ Consistency Level        │
├───────────────────────────────────────────────┤
│ Write chunk        │ Strong (S3 write)        │
│ Read chunk         │ Strong (immutable)       │
│ Write metadata     │ Strong (single shard)    │
│ Read metadata      │ Eventual (cached)        │
│ Cross-device sync  │ Eventual (seconds)       │
│ File permissions   │ Strong (security!)       │
└───────────────────────────────────────────────┘
```

-----

### **Handling Eventual Consistency:**

```python
class EventualConsistencyHandler:
    """
    Handle scenarios where eventual consistency matters
    """
    
    def read_with_consistency_guarantee(self, file_id, 
                                       consistency_level='eventual'):
        """
        Read file metadata with specified consistency
        """
        if consistency_level == 'strong':
            # Read from master (slow but consistent)
            return self.read_from_master(file_id)
        
        elif consistency_level == 'eventual':
            # Try cache first, then replica, fallback to master
            cached = self.read_from_cache(file_id)
            if cached and self.is_fresh_enough(cached):
                return cached
            
            replica = self.read_from_replica(file_id)
            if replica:
                self.update_cache(file_id, replica)
                return replica
            
            # Fallback to master
            return self.read_from_master(file_id)
    
    def write_with_read_after_write(self, file_id, data):
        """
        Ensure user sees their own writes immediately
        """
        # Write to master
        self.write_to_master(file_id, data)
        
        # Invalidate cache
        self.invalidate_cache(file_id)
        
        # For this user's session, pin reads to master
        # for next 5 seconds (until replicas catch up)
        session_id = get_current_session()
        self.pin_to_master(session_id, file_id, ttl=5)
        
        return {'status': 'success'}

# Usage
handler = EventualConsistencyHandler()

# User uploads file
handler.write_with_read_after_write(file_id=123, data=...)

# Immediately after, user lists files
# → Reads from master for 5 seconds
# → Then switches to replicas
files = handler.read_with_consistency_guarantee(
    file_id=123, 
    consistency_level='eventual'
)
```

-----

## **Part B: Fault Tolerance & Disaster Recovery (90 seconds)**

### **Failure Scenarios & Handling:**

```
FAILURE MODES & RECOVERY:
═══════════════════════════════════════════════════

1. CHUNK STORAGE FAILURE (S3 outage)
┌─────────────────────────────────────────────┐
│ Scenario: S3 in US-EAST-1 degraded          │
├─────────────────────────────────────────────┤
│ Detection:                                  │
│ - Health check fails (HTTP 500s)            │
│ - Latency spike > 1 second                  │
│                                             │
│ Response:                                   │
│ - Redirect reads to cross-region replica    │
│   (US-WEST-2)                               │
│ - Queue writes, retry with backoff          │
│ - Show degraded mode banner to users        │
│                                             │
│ Recovery Time: ~5 minutes                   │
└─────────────────────────────────────────────┘

2. METADATA DATABASE MASTER FAILURE
┌─────────────────────────────────────────────┐
│ Scenario: Master DB shard crashes           │
├─────────────────────────────────────────────┤
│ Detection:                                  │
│ - Health check timeout (3 failed pings)     │
│ - Detected by Consul/etcd                   │
│                                             │
│ Response:                                   │
│ - Promote read replica to master            │
│   (automatic via Patroni/orchestrator)      │
│ - Update DNS/routing                        │
│ - Brief read-only mode (30 seconds)         │
│                                             │
│ Recovery Time: ~30 seconds                  │
└─────────────────────────────────────────────┘

3. SYNC SERVICE NODE FAILURE
┌─────────────────────────────────────────────┐
│ Scenario: Sync node crashes (100K conns)    │
├─────────────────────────────────────────────┤
│ Detection:                                  │
│ - Load balancer health check fails          │
│ - WebSocket connections dropped             │
│                                             │
│ Response:                                   │
│ - Clients auto-reconnect to healthy nodes   │
│ - Exponential backoff (1s, 2s, 4s...)       │
│ - Fetch missed updates on reconnection      │
│                                             │
│ Impact: Brief sync delay (< 30 seconds)     │
└─────────────────────────────────────────────┘

4. ENTIRE DATA CENTER FAILURE
┌─────────────────────────────────────────────┐
│ Scenario: US-EAST-1 region down             │
├─────────────────────────────────────────────┤
│ Detection:                                  │
│ - Multiple availability zone failures       │
│ - Global load balancer detects issues       │
│                                             │
│ Response:                                   │
│ - Failover to US-WEST-2 region              │
│ - All traffic redirected (DNS/Anycast)      │
│ - Cross-region replicas become primary      │
│                                             │
│ Recovery Time: ~10 minutes                  │
│ Data Loss: Zero (async replication lag)     │
└─────────────────────────────────────────────┘
```

-----

### **Disaster Recovery Implementation:**

```python
class DisasterRecoveryOrchestrator:
    """
    Coordinates failover during disasters
    """
    
    def __init__(self):
        self.primary_region = 'us-east-1'
        self.backup_region = 'us-west-2'
        self.health_checker = HealthChecker()
    
    async def monitor_and_failover(self):
        """
        Continuously monitor health and trigger failover
        """
        while True:
            health = await self.health_checker.check_region(
                self.primary_region
            )
            
            if health['status'] == 'degraded':
                # Multiple checks failed
                if health['failed_checks'] >= 3:
                    await self.initiate_failover()
            
            await asyncio.sleep(10)  # Check every 10 seconds
    
    async def initiate_failover(self):
        """
        Failover to backup region
        """
        log.critical("Initiating failover to backup region")
        
        # 1. Update global load balancer
        await self.update_glb_routing(
            primary=self.backup_region,
            backup=self.primary_region
        )
        
        # 2. Promote backup database shards
        await self.promote_backup_databases()
        
        # 3. Update DNS records
        await self.update_dns(self.backup_region)
        
        # 4. Notify operations team
        await self.send_alert(
            severity='CRITICAL',
            message='Failover completed to us-west-2'
        )
        
        # 5. Update status page
        await self.update_status_page(
            'Experiencing issues in primary region. '
            'Running on backup infrastructure.'
        )
    
    async def validate_backup_region(self):
        """
        Continuously validate backup is ready
        """
        # Check data replication lag
        lag = await self.get_replication_lag(
            self.primary_region, 
            self.backup_region
        )
        
        if lag > 5:  # More than 5 seconds behind
            await self.alert(
                'High replication lag detected: {}s'.format(lag)
            )
        
        # Verify backup can handle traffic
        capacity = await self.check_capacity(self.backup_region)
        
        if capacity['available'] < 0.5:  # Less than 50% capacity
            await self.alert('Insufficient backup capacity')

# Run continuously
orchestrator = DisasterRecoveryOrchestrator()
asyncio.create_task(orchestrator.monitor_and_failover())
asyncio.create_task(orchestrator.validate_backup_region())
```

-----

### **Data Integrity Checks:**

```
INTEGRITY VERIFICATION:
═══════════════════════════════════════════════════

1. CHUNK INTEGRITY
   - Every chunk has SHA-256 hash stored
   - On read: Verify hash matches content
   - If mismatch: Read from replica, report corruption
   
   Frequency: Every read (low overhead)

2. FILE INTEGRITY
   - Periodic background job scans files
   - Verifies all chunks present and readable
   - Checks chunk ordering and offsets
   
   Frequency: Every 30 days per file

3. METADATA CONSISTENCY
   - Verify file_chunks mapping is complete
   - Check reference counts are accurate
   - Ensure no orphaned chunks
   
   Frequency: Weekly background job

EXAMPLE INTEGRITY CHECK:
┌─────────────────────────────────────────────┐
│ async def verify_file_integrity(file_id):   │
│     # Get file metadata                     │
│     file = get_file_metadata(file_id)       │
│                                             │
│     # Get all chunks                        │
│     chunks = get_file_chunks(file_id)       │
│                                             │
│     expected_size = 0                       │
│     for chunk in chunks:                    │
│         # Download chunk                    │
│         data = download_chunk(chunk.hash)   │
│                                             │
│         # Verify hash                       │
│         computed = sha256(data)             │
│         if computed != chunk.hash:          │
│             alert_corruption(chunk)         │
│             repair_from_replica(chunk)      │
│                                             │
│         expected_size += len(data)          │
│                                             │
│     # Verify total size matches             │
│     if expected_size != file.size:          │
│         alert_size_mismatch(file_id)        │
└─────────────────────────────────────────────┘
```

-----

# **ADVANCED FEATURE 2: Security & Access Control (3.5 minutes)**

## **Part A: Encryption (90 seconds)**

### **End-to-End Encryption Architecture:**

```
ENCRYPTION LAYERS:
═══════════════════════════════════════════════════

LAYER 1: IN-TRANSIT ENCRYPTION
┌─────────────────────────────────────────────┐
│ Client ←TLS 1.3→ Load Balancer              │
│        ←TLS 1.3→ API Server                 │
│        ←TLS 1.3→ S3                         │
│                                             │
│ - All network traffic encrypted             │
│ - Perfect Forward Secrecy (PFS)             │
│ - Strong cipher suites only                 │
└─────────────────────────────────────────────┘

LAYER 2: AT-REST ENCRYPTION (Server-Side)
┌─────────────────────────────────────────────┐
│ S3 Storage with SSE-KMS                     │
│                                             │
│ Chunk → Encrypt with DEK → Store            │
│         ↑                                   │
│         DEK encrypted by CMK (AWS KMS)      │
│                                             │
│ Benefits:                                   │
│ ✓ Automatic key rotation                    │
│ ✓ Audit logs of key usage                   │
│ ✓ Compliance (HIPAA, PCI-DSS)               │
│ ✗ Cloud provider has access                 │
└─────────────────────────────────────────────┘

LAYER 3: CLIENT-SIDE ENCRYPTION (Optional)
┌─────────────────────────────────────────────┐
│ For enterprise/privacy-focused users        │
│                                             │
│ Client:                                     │
│   1. Generate file encryption key (FEK)     │
│   2. Encrypt file with AES-256-GCM          │
│   3. Encrypt FEK with user's master key     │
│   4. Upload encrypted file + encrypted FEK  │
│                                             │
│ Server:                                     │
│   - Stores encrypted data (can't decrypt)   │
│   - Stores encrypted FEK                    │
│   - Never sees plaintext or keys            │
│                                             │
│ Benefits:                                   │
│ ✓ Zero-knowledge architecture               │
│ ✓ Maximum privacy                           │
│ ✗ Can't do server-side features (search)    │
│ ✗ Lost key = lost data                      │
└─────────────────────────────────────────────┘
```

-----

### **Encryption Key Management:**

```
KEY HIERARCHY:
═══════════════════════════════════════════════════

┌─────────────────────────────────────────────┐
│         CUSTOMER MASTER KEY (CMK)           │
│         - AWS KMS managed                   │
│         - Rotated annually                  │
│         - HSM-backed                        │
└───────────────────┬─────────────────────────┘
                    │ encrypts
                    ▼
┌─────────────────────────────────────────────┐
│      DATA ENCRYPTION KEYS (DEK)             │
│      - One per chunk or file                │
│      - Generated per encryption operation   │
│      - Cached briefly in memory             │
└───────────────────┬─────────────────────────┘
                    │ encrypts
                    ▼
┌─────────────────────────────────────────────┐
│              ACTUAL DATA                    │
│         (chunks in S3)                      │
└─────────────────────────────────────────────┘

ENCRYPTION PROCESS:
```

```python
class EncryptionService:
    """
    Handle encryption/decryption operations
    """
    
    def __init__(self):
        self.kms_client = boto3.client('kms')
        self.cmk_id = 'arn:aws:kms:us-east-1:...'
    
    def encrypt_chunk(self, chunk_data):
        """
        Encrypt chunk with envelope encryption
        """
        # 1. Generate data encryption key
        response = self.kms_client.generate_data_key(
            KeyId=self.cmk_id,
            KeySpec='AES_256'
        )
        
        plaintext_key = response['Plaintext']  # DEK
        encrypted_key = response['CiphertextBlob']  # DEK encrypted by CMK
        
        # 2. Encrypt chunk data with DEK
        cipher = AES.new(plaintext_key, AES.MODE_GCM)
        ciphertext, tag = cipher.encrypt_and_digest(chunk_data)
        
        # 3. Package encrypted chunk
        encrypted_chunk = {
            'ciphertext': ciphertext,
            'encrypted_key': encrypted_key,
            'iv': cipher.nonce,
            'tag': tag
        }
        
        # 4. Clear plaintext key from memory
        del plaintext_key
        
        return encrypted_chunk
    
    def decrypt_chunk(self, encrypted_chunk):
        """
        Decrypt chunk
        """
        # 1. Decrypt the data encryption key
        response = self.kms_client.decrypt(
            CiphertextBlob=encrypted_chunk['encrypted_key']
        )
        plaintext_key = response['Plaintext']
        
        # 2. Decrypt chunk data
        cipher = AES.new(
            plaintext_key,
            AES.MODE_GCM,
            nonce=encrypted_chunk['iv']
        )
        
        plaintext = cipher.decrypt_and_verify(
            encrypted_chunk['ciphertext'],
            encrypted_chunk['tag']
        )
        
        # 3. Clear key from memory
        del plaintext_key
        
        return plaintext
```

-----

## **Part B: Access Control & Permissions (90 seconds)**

### **Permission Model:**

```
PERMISSION HIERARCHY:
═══════════════════════════════════════════════════

1. FILE OWNERSHIP
   - Creator is owner
   - Owner has all permissions
   - Can transfer ownership

2. SHARED PERMISSIONS
   ┌─────────────────────────────────────────┐
   │ Permission Level │ Capabilities         │
   ├─────────────────────────────────────────┤
   │ VIEWER           │ - Read file          │
   │                  │ - Download           │
   │                  │ - View metadata      │
   ├─────────────────────────────────────────┤
   │ COMMENTER        │ - All VIEWER perms   │
   │                  │ - Add comments       │
   ├─────────────────────────────────────────┤
   │ EDITOR           │ - All COMMENTER perms│
   │                  │ - Modify file        │
   │                  │ - Upload versions    │
   ├─────────────────────────────────────────┤
   │ OWNER            │ - All EDITOR perms   │
   │                  │ - Delete file        │
   │                  │ - Manage permissions │
   │                  │ - Transfer ownership │
   └─────────────────────────────────────────┘

3. TEAM/ORGANIZATION PERMISSIONS
   - Company-wide shared folders
   - Department-level access
   - Role-based access control (RBAC)
```

-----

### **Sharing Implementation:**

```sql
-- Permission schema (already shown, expanded)
CREATE TABLE file_permissions (
    permission_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    file_id BIGINT NOT NULL,
    granted_to_user_id BIGINT NULL,      -- User-specific
    granted_to_team_id BIGINT NULL,       -- Team-wide
    granted_by_user_id BIGINT NOT NULL,   -- Who granted
    permission_type ENUM('viewer', 'commenter', 'editor', 'owner'),
    
    -- Public sharing
    is_public_link BOOLEAN DEFAULT FALSE,
    share_link_token VARCHAR(64) UNIQUE,
    
    -- Expiration
    expires_at TIMESTAMP NULL,
    
    -- Tracking
    created_at TIMESTAMP DEFAULT NOW(),
    last_accessed_at TIMESTAMP,
    access_count INTEGER DEFAULT 0,
    
    -- Constraints
    CHECK (
        (granted_to_user_id IS NOT NULL) OR 
        (granted_to_team_id IS NOT NULL) OR 
        (is_public_link = TRUE)
    ),
    
    INDEX idx_user_permissions (granted_to_user_id, file_id),
    INDEX idx_team_permissions (granted_to_team_id),
    INDEX idx_public_links (share_link_token, expires_at),
    FOREIGN KEY (file_id) REFERENCES files(file_id) ON DELETE CASCADE
);

-- Teams table
CREATE TABLE teams (
    team_id BIGINT PRIMARY KEY AUTO_INCREMENT,
    organization_id BIGINT NOT NULL,
    team_name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Team membership
CREATE TABLE team_members (
    team_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    role ENUM('member', 'admin') DEFAULT 'member',
    joined_at TIMESTAMP DEFAULT NOW(),
    
    PRIMARY KEY (team_id, user_id),
    FOREIGN KEY (team_id) REFERENCES teams(team_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

-----

### **Permission Checking Logic:**

```python
class PermissionChecker:
    """
    Check if user has permission to access file
    """
    
    def check_permission(self, user_id, file_id, required_permission):
        """
        Check if user has required permission
        
        Returns: (allowed: bool, reason: str)
        """
        # 1. Check if user is owner
        file = get_file_metadata(file_id)
        if file.owner_id == user_id:
            return (True, 'owner')
        
        # 2. Check direct user permissions
        direct_perm = db.query("""
            SELECT permission_type, expires_at
            FROM file_permissions
            WHERE file_id = ? AND granted_to_user_id = ?
            AND (expires_at IS NULL OR expires_at > NOW())
        """, file_id, user_id)
        
        if direct_perm:
            if self.has_sufficient_permission(
                direct_perm.permission_type, 
                required_permission
            ):
                return (True, 'direct_grant')
        
        # 3. Check team permissions
        user_teams = get_user_teams(user_id)
        for team in user_teams:
            team_perm = db.query("""
                SELECT permission_type
                FROM file_permissions
                WHERE file_id = ? AND granted_to_team_id = ?
            """, file_id, team.team_id)
            
            if team_perm and self.has_sufficient_permission(
                team_perm.permission_type,
                required_permission
            ):
                return (True, f'team:{team.team_name}')
        
        # 4. Check if file is in shared folder
        if self.is_in_shared_folder(file_id, user_id):
            return (True, 'shared_folder')
        
        # No permission found
        return (False, 'access_denied')
    
    def has_sufficient_permission(self, granted, required):
        """
        Check if granted permission >= required
        """
        hierarchy = {
            'viewer': 1,
            'commenter': 2,
            'editor': 3,
            'owner': 4
        }
        return hierarchy.get(granted, 0) >= hierarchy.get(required, 0)
    
    def check_public_link_access(self, share_token):
        """
        Validate public share link
        """
        link = db.query("""
            SELECT file_id, permission_type, expires_at, access_count
            FROM file_permissions
            WHERE share_link_token = ?
        """, share_token)
        
        if not link:
            return (False, 'invalid_token')
        
        if link.expires_at and link.expires_at < now():
            return (False, 'link_expired')
        
        # Increment access counter
        db.execute("""
            UPDATE file_permissions
            SET access_count = access_count + 1,
                last_accessed_at = NOW()
            WHERE share_link_token = ?
        """, share_token)
        
        return (True, link.file_id)

# Usage in API endpoint
@app.route('/api/files/<file_id>/download')
def download_file(file_id):
    user_id = get_current_user_id()
    
    # Check permission
    allowed, reason = permission_checker.check_permission(
        user_id, 
        file_id, 
        required_permission='viewer'
    )
    
    if not allowed:
        return {'error': 'Access denied'}, 403
    
    # Generate presigned URL
    url = generate_download_url(file_id)
    
    # Audit log
    log_access(user_id, file_id, 'download', reason)
    
    return {'download_url': url}
```

-----

### **Public Link Generation:**

```python
def create_share_link(file_id, permission_type='viewer', 
                     expires_in_days=None):
    """
    Create shareable public link
    """
    # Generate secure random token
    token = secrets.token_urlsafe(32)
    
    # Calculate expiration
    expires_at = None
    if expires_in_days:
        expires_at = datetime.now() + timedelta(days=expires_in_days)
    
    # Store permission
    db.execute("""
        INSERT INTO file_permissions 
        (file_id, granted_by_user_id, permission_type, 
         is_public_link, share_link_token, expires_at)
        VALUES (?, ?, ?, TRUE, ?, ?)
    """, file_id, get_current_user_id(), permission_type, 
         token, expires_at)
    
    # Generate URL
    share_url = f"https://drive.com/s/{token}"
    
    return {
        'url': share_url,
        'token': token,
        'expires_at': expires_at,
        'permission': permission_type
    }

# Usage
link = create_share_link(
    file_id=12345,
    permission_type='viewer',
    expires_in_days=7
)
# Returns: https://drive.com/s/x7k2m9p4qb...
```

-----

# **ADVANCED FEATURE 3: Performance Optimizations (3 minutes)**

## **Part A: Intelligent Caching (60 seconds)**

### **Multi-Tier Caching Strategy:**

```
CACHING LAYERS (Detailed):
═══════════════════════════════════════════════════

LAYER 1: CLIENT-SIDE CACHE
┌─────────────────────────────────────────────┐
│ Desktop Client Cache:                       │
│ - Entire file stored locally                │
│ - Metadata cached (file tree)               │
│ - Chunk hashes for dedup                    │
│                                             │
│ Cache Size: Configurable (default 10GB)     │
│ Eviction: LRU based on last access          │
│ Invalidation: On server notification        │
│                                             │
│ Hit Rate: ~95% (most accessed files local)  │
└─────────────────────────────────────────────┘

LAYER 2: CDN CACHE (CloudFront)
┌─────────────────────────────────────────────┐
│ What's Cached:                              │
│ - Frequently accessed files (hot files)     │
│ - File chunks (especially media)            │
│ - Thumbnails and previews                   │
│                                             │
│ Cache Strategy:                             │
│ - TTL: 24 hours                             │
│ - Cache-Control headers based on file type  │
│ - Invalidation via API calls                │
│                                             │
│ Geographic Distribution:                    │
│ - 200+ edge locations worldwide             │
│ - Anycast routing to nearest edge           │
│                                             │
│ Hit Rate: ~80% for popular files            │
└─────────────────────────────────────────────┘

LAYER 3: APPLICATION CACHE (Redis)
┌─────────────────────────────────────────────┐
│ What's Cached:                              │
│ - File metadata (hot)                       │
│ - User information                          │
│ - Permissions data                          │
│ - Chunk location mappings                   │
│ - Directory listings                        │
│                                             │
│ Cache Configuration:                        │
│ - Redis Cluster (16 nodes)                  │
│ - Master-slave replication                  │
│ - Eviction: LRU with TTLs                   │
│                                             │
│ TTLs by Data Type:                          │
│ - User info: 1 hour                         │
│ - File metadata: 5 minutes                  │
│ - Permissions: 2 minutes (security)         │
│ - Directory listings: 30 seconds            │
│                                             │
│ Hit Rate: ~90% for metadata                 │
└─────────────────────────────────────────────┘

LAYER 4: DATABASE QUERY CACHE
┌─────────────────────────────────────────────┐
│ Postgre​​​​​​​​​​​​​​​​SQL built-in query cache             │
│ - Automatic for repeated queries            │
│ - Invalidated on table updates              │
│                                             │
│ Hit Rate: ~60% (lower due to writes)        │
└─────────────────────────────────────────────┘

```
---

### **Intelligent Cache Warming:**

```python
class CacheWarmer:
    """
    Proactively warm caches based on usage patterns
    """
    
    def __init__(self):
        self.redis = redis.Redis()
        self.ml_predictor = UsagePredictionModel()
    
    async def warm_user_cache(self, user_id):
        """
        Preload likely-to-be-accessed files into cache
        """
        # 1. Get user's recent access patterns
        recent_files = get_recent_access(user_id, days=7)
        
        # 2. Predict likely next accesses using ML
        predictions = self.ml_predictor.predict_next_access(
            user_id, 
            recent_files
        )
        
        # 3. Preload top predictions into Redis
        for file_id, probability in predictions[:10]:
            if probability > 0.7:  # High confidence
                metadata = fetch_from_db(file_id)
                await self.redis.setex(
                    f"file:metadata:{file_id}",
                    300,  # 5 min TTL
                    json.dumps(metadata)
                )
    
    async def warm_team_cache(self, team_id):
        """
        Warm cache for team's shared files
        """
        # Get team's most accessed files this week
        popular_files = db.query("""
            SELECT file_id, COUNT(*) as access_count
            FROM access_logs
            WHERE team_id = ? 
            AND accessed_at > NOW() - INTERVAL 7 DAY
            GROUP BY file_id
            ORDER BY access_count DESC
            LIMIT 50
        """, team_id)
        
        # Preload into cache
        for file in popular_files:
            await self.preload_file_metadata(file.file_id)
    
    async def warm_on_user_login(self, user_id):
        """
        Warm cache immediately after user logs in
        """
        # Start cache warming asynchronously
        asyncio.create_task(self.warm_user_cache(user_id))
        asyncio.create_task(self.warm_user_directory(user_id))

# Run periodic cache warming
scheduler = BackgroundScheduler()
scheduler.add_job(
    warm_popular_files,
    trigger='interval',
    minutes=15
)
```

-----

## **Part B: Compression (45 seconds)**

### **Adaptive Compression Strategy:**

```
COMPRESSION ALGORITHM SELECTION:
═══════════════════════════════════════════════════

Decision Tree:
┌─────────────────────────────────────────────┐
│ Is file already compressed?                 │
│ (check MIME type and magic bytes)           │
└─────────┬────────────────────┬──────────────┘
          │                    │
          NO                   YES
          │                    │
          ▼                    ▼
┌──────────────────┐    ┌─────────────────┐
│ What file type?  │    │ Store as-is     │
└────┬─────────────┘    │ No recompression│
     │                  └─────────────────┘
     ├─── Text/Code
     │    ├─ Algorithm: ZSTD (fast + good ratio)
     │    └─ Level: 3 (balanced)
     │
     ├─── Media (images, video, audio)
     │    └─ No compression (already compressed)
     │
     ├─── Documents (PDF, DOCX)
     │    ├─ Algorithm: ZSTD
     │    └─ Level: 5 (better ratio)
     │
     └─── Archives (ZIP, TAR)
          └─ No compression (already compressed)

COMPRESSION STATISTICS:
File Type       | Avg Compression | Algorithm
─────────────────────────────────────────────
Source Code     | 70% reduction   | ZSTD-3
Plain Text      | 65% reduction   | ZSTD-3
JSON/XML        | 75% reduction   | ZSTD-5
Office Docs     | 30% reduction   | ZSTD-5
Images (JPEG)   | 0% (skip)       | None
Videos (MP4)    | 0% (skip)       | None
Already .gz/.zip| 0% (skip)       | None
```

-----

### **Compression Implementation:**

```python
import zstandard as zstd

class CompressionService:
    """
    Handle intelligent compression
    """
    
    # Compression levels by priority
    COMPRESSIBLE_TYPES = {
        'text/': 3,          # Text files
        'application/json': 5,
        'application/xml': 5,
        'application/javascript': 3,
        'application/pdf': 2  # Some benefit
    }
    
    # Never compress these
    SKIP_COMPRESSION = {
        'image/', 'video/', 'audio/',
        'application/zip', 'application/gzip',
        'application/x-rar'
    }
    
    def should_compress(self, mime_type, file_size):
        """
        Decide if file should be compressed
        """
        # Don't compress very small files (< 1KB)
        if file_size < 1024:
            return False, None
        
        # Check if in skip list
        for skip_type in self.SKIP_COMPRESSION:
            if mime_type.startswith(skip_type):
                return False, None
        
        # Check if compressible
        for comp_type, level in self.COMPRESSIBLE_TYPES.items():
            if mime_type.startswith(comp_type):
                return True, level
        
        # Default: try compression with sample
        return self.test_compression(file_sample), 3
    
    def compress_chunk(self, chunk_data, level=3):
        """
        Compress chunk with ZSTD
        """
        compressor = zstd.ZstdCompressor(level=level)
        compressed = compressor.compress(chunk_data)
        
        # Only use compression if saves > 10%
        compression_ratio = len(compressed) / len(chunk_data)
        
        if compression_ratio < 0.9:
            return {
                'data': compressed,
                'is_compressed': True,
                'original_size': len(chunk_data),
                'compressed_size': len(compressed),
                'algorithm': 'zstd',
                'level': level
            }
        else:
            # Compression not worthwhile
            return {
                'data': chunk_data,
                'is_compressed': False,
                'original_size': len(chunk_data)
            }
    
    def decompress_chunk(self, compressed_data, metadata):
        """
        Decompress chunk if needed
        """
        if not metadata.get('is_compressed'):
            return compressed_data
        
        if metadata['algorithm'] == 'zstd':
            decompressor = zstd.ZstdDecompressor()
            return decompressor.decompress(compressed_data)

# Usage in upload flow
compression_svc = CompressionService()

def process_upload(file_data, mime_type):
    chunks = chunk_file(file_data)
    
    # Decide compression strategy once
    should_compress, level = compression_svc.should_compress(
        mime_type, 
        len(file_data)
    )
    
    processed_chunks = []
    for chunk in chunks:
        if should_compress:
            result = compression_svc.compress_chunk(chunk, level)
            processed_chunks.append(result)
        else:
            processed_chunks.append({
                'data': chunk,
                'is_compressed': False
            })
    
    return processed_chunks

SAVINGS EXAMPLE:
1 GB source code repository
├─ Uncompressed: 1,000 MB
├─ With ZSTD: 300 MB
└─ Savings: 70% (700 MB saved × 3 replicas = 2.1 GB saved)

Across 15 PB storage:
├─ ~30% of data is compressible (4.5 PB)
├─ Average 60% compression ratio
├─ Net savings: ~2.7 PB
└─ Cost savings: ~$270K/month
```

-----

## **Part C: Parallel Operations (45 seconds)**

### **Parallel Upload/Download:**

```
PARALLELIZATION STRATEGY:
═══════════════════════════════════════════════════

PARALLEL UPLOAD (Client Side):
┌─────────────────────────────────────────────┐
│                                             │
│   Large File (1 GB, 250 chunks × 4MB)       │
│                                             │
│   ┌─────────────────────────────┐           │
│   │ Upload Queue (Priority)     │           │
│   ├─────────────────────────────┤           │
│   │ Chunk 0 → Worker 1 ────────┼─► S3       │
│   │ Chunk 1 → Worker 2 ────────┼─► S3       │
│   │ Chunk 2 → Worker 3 ────────┼─► S3       │
│   │ Chunk 3 → Worker 4 ────────┼─► S3       │
│   │ ...                         │           │
│   │ Chunk 249 → Worker N        │           │
│   └─────────────────────────────┘           │
│                                             │
│   Configuration:                            │
│   - Worker threads: 10 (configurable)       │
│   - Max concurrent: Based on bandwidth      │
│   - Retry failed chunks: Exponential backoff│
│   - Resume from last successful chunk       │
│                                             │
└─────────────────────────────────────────────┘

BANDWIDTH OPTIMIZATION:
┌─────────────────────────────────────────────┐
│ Available Bandwidth: 100 Mbps               │
│                                             │
│ Single-threaded:                            │
│   1 GB file ÷ 100 Mbps = ~80 seconds        │
│                                             │
│ 10 parallel uploads:                        │
│   Saturates bandwidth more efficiently      │
│   1 GB file = ~12 seconds (6.6x faster)     │
│                                             │
│ Dynamic Thread Adjustment:                  │
│   - Measure actual throughput               │
│   - Increase threads if bandwidth unused    │
│   - Decrease if too much contention         │
└─────────────────────────────────────────────┘
```

-----

### **Implementation:**

```python
import asyncio
import aiohttp
from concurrent.futures import ThreadPoolExecutor

class ParallelUploader:
    """
    Upload file chunks in parallel
    """
    
    def __init__(self, max_workers=10):
        self.max_workers = max_workers
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
    
    async def upload_file(self, file_path):
        """
        Upload file with parallel chunk uploads
        """
        # 1. Chunk the file
        chunks = await self.chunk_file(file_path)
        
        # 2. Calculate optimal parallelism
        optimal_workers = self.calculate_optimal_workers(
            len(chunks),
            await self.measure_bandwidth()
        )
        
        # 3. Create upload tasks
        semaphore = asyncio.Semaphore(optimal_workers)
        
        async def upload_with_semaphore(chunk):
            async with semaphore:
                return await self.upload_chunk(chunk)
        
        # 4. Upload all chunks in parallel
        tasks = [
            upload_with_semaphore(chunk) 
            for chunk in chunks
        ]
        
        # 5. Wait for all uploads with progress tracking
        results = []
        for coro in asyncio.as_completed(tasks):
            result = await coro
            results.append(result)
            self.update_progress(len(results), len(chunks))
        
        # 6. Verify all chunks uploaded
        if len(results) != len(chunks):
            raise UploadError("Some chunks failed")
        
        return results
    
    async def upload_chunk(self, chunk):
        """
        Upload single chunk with retry
        """
        max_retries = 3
        backoff = 1
        
        for attempt in range(max_retries):
            try:
                # Get presigned URL
                url = await self.get_upload_url(chunk.hash)
                
                # Upload chunk
                async with aiohttp.ClientSession() as session:
                    async with session.put(
                        url,
                        data=chunk.data,
                        headers={'Content-Type': 'application/octet-stream'}
                    ) as response:
                        if response.status == 200:
                            return {
                                'chunk_hash': chunk.hash,
                                'status': 'success'
                            }
                
            except Exception as e:
                if attempt < max_retries - 1:
                    await asyncio.sleep(backoff)
                    backoff *= 2
                else:
                    raise
    
    def calculate_optimal_workers(self, num_chunks, bandwidth_mbps):
        """
        Calculate optimal number of parallel uploads
        """
        # Rule of thumb: 1 worker per 10 Mbps, max 20
        optimal = min(
            bandwidth_mbps // 10,
            20,
            num_chunks  # Don't exceed number of chunks
        )
        return max(optimal, 1)

# Usage
uploader = ParallelUploader(max_workers=10)
await uploader.upload_file('large_video.mp4')

PERFORMANCE COMPARISON:
┌─────────────────────────────────────────────┐
│ File Size: 1 GB (250 chunks)                │
├─────────────────────────────────────────────┤
│ Serial Upload (1 thread):                   │
│   Time: ~80 seconds                         │
│   Throughput: 12.5 MB/s                     │
├─────────────────────────────────────────────┤
│ Parallel Upload (10 threads):               │
│   Time: ~12 seconds                         │
│   Throughput: 83 MB/s                       │
│   Improvement: 6.6x faster                  │
└─────────────────────────────────────────────┘
```

-----

## **Part D: Smart Sync (30 seconds)**

### **Predictive Synchronization:**

```
SMART SYNC FEATURES:
═══════════════════════════════════════════════════

1. SELECTIVE SYNC
   User configures which folders to sync per device
   
   Example:
   Desktop: Sync "Work" folder only
   Mobile: Sync "Photos" only (WiFi-only)
   Laptop: Sync everything
   
   Benefit: Saves bandwidth and storage

2. BANDWIDTH THROTTLING
   ┌─────────────────────────────────────────┐
   │ Time-based throttling:                  │
   │ - Business hours: 1 Mbps max            │
   │ - After hours: Unlimited                │
   │                                         │
   │ Network-based throttling:               │
   │ - WiFi: Full speed                      │
   │ - Cellular: 256 Kbps max                │
   └─────────────────────────────────────────┘

3. SMART TIMING
   Defer non-urgent syncs to optimal times:
   - Large files: Sync overnight
   - Video files: WiFi only
   - Photos: Immediate (high priority)

4. DELTA SYNC OPTIMIZATION
   ┌─────────────────────────────────────────┐
   │ Traditional sync:                       │
   │   Entire 100MB file uploaded            │
   │                                         │
   │ Delta sync:                             │
   │   Only 4MB changed chunk uploaded       │
   │   Savings: 96%                          │
   └─────────────────────────────────────────┘
```

```python
class SmartSyncEngine:
    """
    Intelligent sync decisions
    """
    
    def should_sync_now(self, file_id, device_id):
        """
        Decide if file should sync now or defer
        """
        device = get_device_info(device_id)
        file = get_file_metadata(file_id)
        
        # Check user's sync settings
        if not self.is_in_synced_folder(file, device):
            return False, "Not in selective sync folders"
        
        # Check network conditions
        if device.network_type == 'cellular':
            if file.size > 10 * MB and not file.is_high_priority:
                return False, "Large file on cellular, defer to WiFi"
        
        # Check bandwidth throttling
        current_hour = datetime.now().hour
        if 9 <= current_hour <= 17:  # Business hours
            if device.throttle_during_work:
                return False, "Throttled during business hours"
        
        # Check battery (mobile devices)
        if device.platform == 'mobile':
            if device.battery_level < 20:
                return False, "Low battery, defer sync"
        
        # All checks passed
        return True, "Ready to sync"
    
    def prioritize_sync_queue(self, pending_syncs):
        """
        Order sync operations by priority
        """
        def priority_score(sync_item):
            score = 0
            
            # File type priority
            if sync_item.mime_type.startswith('image/'):
                score += 100  # Photos high priority
            elif sync_item.mime_type.startswith('text/'):
                score += 80   # Documents high priority
            elif sync_item.mime_type.startswith('video/'):
                score += 20   # Videos low priority
            
            # Recency
            age_minutes = (now() - sync_item.modified_at).minutes
            score += max(0, 100 - age_minutes)  # Newer = higher
            
            # Size (smaller first)
            if sync_item.size < 1 * MB:
                score += 50
            elif sync_item.size > 100 * MB:
                score -= 50
            
            return score
        
        return sorted(pending_syncs, key=priority_score, reverse=True)
```

-----

## **Summary of Advanced Features (30 seconds)**

**Quick recap:**

> “Let me summarize the advanced features we’ve covered:

**1. Data Consistency & Reliability:**

- 3x replication with 99.999999999% durability
- Eventual consistency with strong consistency where needed
- Multi-region disaster recovery (< 10 min RTO)
- Automatic failover and integrity checks

**2. Security & Access Control:**

- End-to-end encryption with envelope encryption (AES-256)
- Granular permissions (viewer, commenter, editor, owner)
- Public share links with expiration
- Team and organization-level access control

**3. Performance Optimizations:**

- Multi-layer caching (95% client, 80% CDN, 90% Redis hit rates)
- Adaptive compression (60-70% reduction for text, ZSTD)
- Parallel uploads/downloads (6.6x faster)
- Smart sync with selective folders and bandwidth management

These features ensure the system is reliable, secure, and performant at scale.”

-----

## **Transition to Next Phase (15 seconds)**

**Ask the interviewer:**

> “We’ve covered the advanced features. Would you like me to:
> 
> A) Discuss specific scalability bottlenecks and mitigation strategies?
> B) Cover monitoring, observability, and operational concerns?
> C) Talk about additional features like versioning or collaboration?
> D) Move to discussing trade-offs and alternative approaches?”

-----

## **Common Mistakes to Avoid:**

❌ **Feature dumping** - Don’t list 20 features superficially

❌ **No justification** - “We’ll add encryption” → Why? How?

❌ **Ignoring trade-offs** - Every feature has costs

❌ **Over-engineering** - Don’t propose blockchain for file sharing

❌ **Unrealistic claims** - “100% uptime” or “Zero latency”

❌ **Skipping security** - Always address security explicitly

❌ **No metrics** - “Fast caching” vs “90% hit rate”

❌ **Feature creep** - Stay focused on core features

-----

## **Pro Tips:**

✅ **Connect to earlier design:**

- “Remember our 700K QPS? That’s why this caching layer is critical”

✅ **Use real-world examples:**

- “Similar to how Dropbox handles end-to-end encryption…”

✅ **Quantify impact:**

- Not “good compression” but “60% storage savings = $270K/month”

✅ **Show production thinking:**

- Consider monitoring, alerting, rollback strategies

✅ **Be pragmatic:**

- “We’d start with server-side encryption, add client-side in v2”

✅ **Address security seriously:**

- Never hand-wave: “We’ll encrypt everything”
- Be specific about algorithms, key management

✅ **Think about operations:**

- How do you debug issues?
- How do you monitor performance?
- How do you handle incidents?

-----

## **What Your Whiteboard Should Look Like:**

```
┌───────────────────────────────────────────────┐
│ ADVANCED FEATURES                             │
├───────────────────────────────────────────────┤
│                                               │
│ RELIABILITY:                                  │
│ • 3x replication (S3), 99.999999999% durable  │
│ • Multi-region DR (US-EAST ←→ US-WEST)        │
│ • Auto-failover < 10 min                      │
│                                               │
│ SECURITY:                                     │
│ • Envelope encryption (AES-256-GCM + KMS)     │
│ • TLS 1.3 in transit                          │
│ • RBAC: viewer/commenter/editor/owner         │
│ • Public links with expiration                │
│                                               │
│ PERFORMANCE:                                  │
│ • Caching: Client(95%) + CDN(80%) + Redis(90%)│
│ • Compression: ZSTD (60-70% for text)         │
│ • Parallel I/O: 10 workers (6.6x faster)      │
│ • Smart sync: Selective folders, throttling   │
│                                               │
│ IMPACT:                                       │
│ • Storage savings: 2.7 PB ($270K/month)       │
│ • Latency: P99 < 100ms (metadata)             │
│ • Availability: 99.99% SLA                    │
└───────────────────────────────────────────────┘
```

-----

## **Time Check:**

At 45 minutes, you should have covered:

- ✅ Requirements (5 min)
- ✅ Capacity estimates (5 min)
- ✅ High-level design (10 min)
- ✅ Core components deep dive (15 min)
- ✅ Advanced features (10 min)

**Remaining: 15 minutes**

**What’s next:**

- Scalability & bottlenecks (7 min)
- Trade-offs & alternatives (5 min)
- Wrap-up & questions (3 min)

You’re on track! 🎯​​​​​​​​​​​​​​​​
