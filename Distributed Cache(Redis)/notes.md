# Distributed Cache System (Memcached / Redis Cluster Style)

---

## 1. Problem Statement

Design a distributed in-memory cache system that:

- Supports GET / SET / DELETE
- Supports TTL expiration
- Uses LRU eviction
- Scales to 1TB data
- Handles 100K+ RPS
- Provides high availability
- Uses eventual consistency

---

## 2. Requirements

### Functional Requirements

- Set key-value pairs
- Get key-value pairs
- Delete keys
- Configure TTL per key
- LRU eviction when memory full
- Horizontal scaling
- Optional replication

---

### Non-Functional Requirements

- Latency < 10ms (p95)
- 100K+ requests/sec
- 1TB total data capacity
- High Availability
- Eventual Consistency
- Partition Tolerance (CAP → AP system)

---

## 3. High-Level Architecture

### Components

- Client (Browser / Mobile)
- Web/Application Server
- Cache Client (SDK inside app)
- Distributed Cache Cluster (multiple nodes)
- Replica Nodes
- Zookeeper (configuration + coordination)
- Database (source of truth)

---

## 4. Entity Ownership

### Data Plane (Lives in Cache Nodes)

#### CacheEntry
- key
- value
- ttl
- lastAccessed

Stored in:
- In-memory HashMap
- Doubly Linked List (for LRU)

---

### Control Plane (Lives in Zookeeper)

#### CacheNode Metadata
- node_id
- status
- shard_range

#### Replication Metadata
- primary_node
- replicas[]
- sync_status

---

### Client-Side

- Consistent Hash Ring
- Connection Pool
- Node List (cached from Zookeeper)

---


# Distributed Cache System – APIs

---

## 1. Client-Facing APIs (Data Plane)

### GET

GET /cache/{key}

Response:
{
  "value": "...",
  "ttl_remaining": 120
}

Behavior:
- Hash(key) using consistent hashing
- Route to primary node
- Validate TTL
- Return value or 404

---

### SET

POST /cache/{key}

Body:
{
  "value": "...",
  "ttl": 3600
}

Behavior:
- Route to primary
- Store in memory
- Apply TTL
- Asynchronously replicate to replica
- Return success

---

### DELETE

DELETE /cache/{key}

Behavior:
- Route to primary
- Remove locally
- Replicate deletion to replica

---

### MGET (Optional – Bulk Fetch)

POST /cache/mget

Body:
{
  "keys": ["k1", "k2", "k3"]
}

Behavior:
- Client groups keys by shard
- Parallel requests to respective nodes
- Aggregate results

---

## 2. Internal Replication APIs

(Primary → Replica Communication)

### Replicate Write

replicate_set(key, value, ttl, version)

---

### Replicate Delete

replicate_delete(key, version)

---

### Health Check

GET /health

Used by:
- Monitoring systems
- Zookeeper heartbeat tracking

---

## 3. Configuration / Coordination Interfaces (Logical - Zookeeper)

### Register Node

/cache/nodes/{node_id}

- Created as ephemeral node
- Removed automatically if heartbeat stops

---

### Shard Mapping

/cache/shards/{shard_id}

Data:
{
  "primary": "node_x",
  "replicas": ["node_y"]
}

Clients watch this path for topology changes.



## 5. Data Flow

### Read Pattern: Cache-Aside (Recommended)

This is the most common production pattern.

#### Cache Hit

Client → Cache Client → Hash(key) → Primary Node  
Node checks TTL → return value  

Latency: Memory lookup (~ few ms)

---

#### Cache Miss

1. Cache returns miss
2. Application fetches from Database
3. Application writes value back to cache
4. Future reads become hit

Pattern: Cache-Aside (Lazy Loading)

---

### Write Pattern: Write-Through (Recommended)

1. Application writes to Database
2. Application updates cache
3. Primary replicates async to replica

Ensures DB remains source of truth.

---

### Delete

Route to primary  
Delete locally  
Replicate deletion  

---

## 6. Node Failure Handling

### Primary Failure

- Zookeeper detects heartbeat loss
- Replica promoted to primary
- Clients refresh hash ring
- Traffic resumes

Possible small data loss (async replication)

---

### Zookeeper Failure

If quorum maintained → system continues  
If quorum lost → topology frozen  
Existing cache traffic continues  
Failover stops until quorum restored  

---

## 7. Consistent Hashing

Why used?

- Distributes keys across nodes
- Minimizes key movement on scale-out
- Clients compute hash locally
- No central router in hot path

---

## 8. TTL Strategy

TTL enforced inside cache node because:

- Node owns data
- Avoid clock skew
- Avoid client complexity
- Centralized expiration logic

Expiration mechanisms:

- Lazy expiration (on read)
- Background cleanup thread

Best practice: Use both

---

## 9. Eviction Policy

LRU:

- HashMap + Doubly Linked List
- O(1) operations
- Evict least recently used key

---

## 10. Replication Trade-Offs

### Async Replication
✔ Fast writes  
✖ Possible small data loss  

### Sync Replication
✔ Stronger consistency  
✖ Higher latency  

Most distributed caches use async replication.

---

## 11. CAP Tradeoff

Partition tolerance required.

We choose:
- High Availability
- Eventual Consistency

Cache is not source of truth.

---

## 12. Thundering Herd Problem

If many requests hit a missing key:

- All requests go to DB
- DB overloads

### Mitigations:

- Request coalescing (only one request fetches DB)
- Mutex lock per key
- Probabilistic early refresh
- Soft TTL + background refresh

---

## 13. Interview Elevator Pitch

> We design a distributed in-memory cache using consistent hashing for sharding.  
> Clients route requests directly to primary nodes.  
> We use Cache-Aside for reads and Write-Through for writes.  
> Writes are asynchronously replicated to replicas for high availability.  
> Zookeeper manages cluster membership and leader election but is not in the hot path.  
> TTL and LRU eviction are handled inside each node.  
> The system prioritizes availability and low latency over strong consistency.

---

## 14. Common Follow-Up Questions

Be ready for:

- What happens if Zookeeper fails?
- How do you prevent hot keys?
- Can reads go to replicas?
- How do you rebalance when adding nodes?
- How do you avoid thundering herd?
- What if replication lag is high?
- When would you use Write-Back?