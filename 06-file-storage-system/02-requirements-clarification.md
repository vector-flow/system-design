# Minutes 1-5: Requirements Clarification (5 min)

## **The Critical Foundation - Get This Right!**

This phase sets the trajectory for your entire interview. A poor requirements gathering session leads to solving the wrong problem. Here’s how to nail it:

-----

## **Your Opening (30 seconds)**

**What to say:**

> “Great! Before I jump into the design, I’d like to clarify the requirements and constraints to ensure I’m solving the right problem. I’ll ask about functional requirements, non-functional requirements, and scale. Does that work?”

**Why this matters:** Shows structure and gets interviewer buy-in for your process.

-----

## **Functional Requirements (2 minutes)**

### **Core Features - Ask in Priority Order:**

**1. Essential Features (confirm these first):**

- “Should users be able to upload and download files?”
- “Do we need file synchronization across multiple devices?”
- “Should users be able to share files with other users?”

**2. Important Features (clarify scope):**

- “Do we need folder/directory support or just flat file structure?”
- “Should we support file versioning and history?”
- “Do we need real-time collaborative editing (like Google Docs) or just file storage?”
- “Should we support mobile apps, desktop clients, and web interface?”

**3. Nice-to-Have Features (ask if time permits):**

- “Do we need file preview/thumbnails?”
- “Should we support comments or notifications?”
- “Do we need trash/recovery functionality?”
- “What about search across file contents?”

### **What to Write Down:**

Create a clear list on the whiteboard:

**In Scope:**

- Upload/download files
- Sync across devices
- File sharing with permissions
- Folder hierarchy
- Basic versioning

**Out of Scope:**

- Real-time collaborative editing
- Advanced search
- File preview generation

-----

## **Non-Functional Requirements (1.5 minutes)**

### **System Qualities - Critical to Clarify:**

**Availability:**

- “What’s our target availability? 99.9% or higher?”
- “Is this a system where downtime is acceptable occasionally?”

**Consistency:**

- “How important is strong consistency? Can we tolerate eventual consistency?”
- “If two users edit the same file simultaneously, what should happen?”

**Performance:**

- “What are acceptable latencies for upload/download?”
- “How fast should sync happen? Real-time (seconds) or can it be minutes?”

**Reliability:**

- “How critical is data durability? Can we tolerate any data loss?”
- “Do we need disaster recovery capabilities?”

**Security:**

- “Do we need encryption at rest and in transit?”
- “What level of access control? (public links, user-specific, team-based?)”

### **What to Write Down:**

**Target SLAs:**

- 99.99% availability
- < 200ms latency for metadata operations
- Sync within 10 seconds across devices
- Zero data loss (high durability)
- Eventual consistency acceptable for most operations

-----

## **Scale & Capacity (1.5 minutes)**

### **User Scale:**

- “How many total users are we designing for?”
- “How many daily active users (DAU)?”
- “How many files per user on average?”

**Example numbers to extract:**

- 500M total users
- 100M DAU
- Average 200 files per user
- 10GB average storage per user

### **File Characteristics:**

- “What’s the average file size?”
- “What’s the maximum file size we should support?”
- “What’s the distribution? Lots of small files or fewer large files?”

**Example numbers:**

- Average: 10MB per file
- Max: 10GB per file
- Mix: 80% files under 1MB, 15% between 1MB-100MB, 5% over 100MB

### **Traffic Patterns:**

- “Is this read-heavy or write-heavy?”
- “What’s the read-to-write ratio?”
- “Are there peak usage times?”

**Example patterns:**

- Read-heavy: 100:1 read-to-write ratio
- Peak hours: business hours in major timezones
- Upload spike: Monday mornings

### **Growth:**

- “What’s the expected growth rate?”
- “Are we designing for current scale or 3-5 years out?”

-----

## **Quick Calculation Check (30 seconds)**

**Do a sanity check calculation immediately:**

> “Let me quickly verify the scale: 500M users × 10GB average = 5 exabytes of storage. With 100M DAU, if each user uploads 2 files/day, that’s 200M uploads daily or ~2,300 uploads/second. Does that align with your expectations?”

**Why this matters:**

- Shows quantitative thinking early
- Catches any misunderstandings about scale
- Impresses the interviewer

-----

## **Clarification Summary (30 seconds)**

**Before moving on, recap:**

> “Just to confirm, we’re building a file storage and sync system for 500M users with focus on availability and sync speed. Core features are upload/download, cross-device sync, and file sharing. We can accept eventual consistency and prioritize read performance. Sound good?”

**Wait for confirmation before proceeding!**

-----

## **Common Mistakes to Avoid:**

❌ **Don’t assume anything** - “I’ll assume we need X” (always ask!)

❌ **Don’t ask yes/no questions only** - Ask open-ended: “What features are most critical?”

❌ **Don’t spend 10 minutes here** - Be efficient, you need time for design

❌ **Don’t skip scale questions** - Numbers drive architectural decisions

❌ **Don’t ignore interviewer hints** - If they mention “millions of small files,” probe deeper

❌ **Don’t forget to write things down** - Your whiteboard should have a clear requirements section

-----

## **Pro Tips:**

✅ **Prioritize ruthlessly** - “If we had to launch with just 3 features, which would they be?”

✅ **Ask about existing systems** - “Are users migrating from another system? How much data?”

✅ **Clarify edge cases early** - “How should we handle files larger than 10GB?”

✅ **Understand the business context** - “Is this for consumers, enterprises, or both?”

✅ **Be conversational, not robotic** - This is a dialogue, not an interrogation

✅ **Show domain knowledge** - “For a system like Dropbox, I imagine sync speed is crucial…”

-----

## **What Your Whiteboard Should Look Like After 5 Minutes:**

```
REQUIREMENTS - File Storage System

FUNCTIONAL:
✓ Upload/download files (max 10GB)
✓ Sync across devices (< 10 sec)
✓ File sharing + permissions
✓ Folder hierarchy
✓ Version history (basic)
✗ Real-time collab editing
✗ Advanced search

NON-FUNCTIONAL:
- 99.99% availability
- Eventual consistency OK
- Zero data loss
- Encrypted at rest/transit

SCALE:
- 500M users, 100M DAU
- 10GB avg per user → 5EB total
- 100:1 read/write ratio
- ~2,300 uploads/sec
- Peak: business hours
```

-----

## **The Secret Weapon: The “Why” Questions**

After getting basic requirements, ask **one strategic “why” question**:

- “Why is sync speed more important than consistency?”
- “Why are we targeting consumers vs enterprises?”
- “Why do users need versioning?”

**This shows:** You think beyond requirements to understand the business problem.

-----

**Remember:** These 5 minutes determine whether you’re designing the right system. Invest the time here—it pays dividends for the remaining 55 minutes!​​​​​​​​​​​​​​​​
