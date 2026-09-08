# ⚡ Module 5: Distributed Caching Architecture & Memory Engines

> **Core Philosophy:** *The fastest query is the one that never touches the disk. Caching is the art of trading expensive computation and disk I/O for fast memory lookups, balanced against the complexity of keeping cached data in sync with reality.*

---

## 📌 Table of Contents
1. [The Problem: Database I/O Saturation & The Memory Hierarchy](#1-the-problem-database-io-saturation--the-memory-hierarchy)
2. [The Multi-Tier Caching Topology (L1 vs L2 vs Distributed)](#2-the-multi-tier-caching-topology-l1-vs-l2-vs-distributed)
3. [Caching Patterns: Cache-Aside vs Write-Through vs Write-Back](#3-caching-patterns-cache-aside-vs-write-through-vs-write-back)
4. [Cache Invalidation & Eviction Algorithms (LRU, LFU, ARC)](#4-cache-invalidation--eviction-algorithms-lru-lfu-arc)
5. [The Three Classic Cache Disasters & Production Fixes](#5-the-three-classic-cache-disasters--production-fixes)
6. [Redis Internal Architecture: Single-Threaded Event Loop & In-Memory Data Structures](#6-redis-internal-architecture-single-threaded-event-loop--in-memory-data-structures)
7. [Redis High Availability: Sentinel vs Redis Cluster Sharding](#7-redis-high-availability-sentinel-vs-redis-cluster-sharding)
8. [Java / Spring Boot Industrial Cache Implementation with Redisson](#8-java--spring-boot-industrial-cache-implementation-with-redisson)
9. [Side-by-Side Trade-off Analysis Table](#9-side-by-side-trade-off-analysis-table)
10. [Interview Rapid Q&A Checklist](#10-interview-rapid-qa-checklist)

---

## 1. The Problem: Database I/O Saturation & The Memory Hierarchy

```
MEMORY LATENCY HIERARCHY:
CPU L1 Cache:        0.5 ns
CPU L2 Cache:        7.0 ns
Main Memory (RAM):   100 ns      (0.0001 ms)
NVMe SSD Read:       10,000 ns   (0.01 ms - 100x slower than RAM!)
Spinning Disk (HDD): 10,000,000 ns (10 ms - 100,000x slower than RAM!)
```
- A database index stored on NVMe or magnetic disk requires physical I/O sweeps.
- Redis serves data straight from RAM using non-blocking I/O multiplexing (`epoll`), answering up to **100,000+ operations/second per core with sub-millisecond response latency**.

---

## 2. The Multi-Tier Caching Topology (L1 vs L2)

```
[ Client Browser / Mobile ] ──▶ Local Storage / Service Worker Cache (0ms)
              │
              ▼
[ Edge CDN (Cloudflare / Akamai) ] ──▶ Edge Static & API Cache (10-20ms)
              │
              ▼
[ Application Server (Spring Boot) ]
  ├── L1 In-Memory Cache (Caffeine / Guava in JVM Heap: ~0.05ms)
  │         └── Cache MISS?
  └── L2 Distributed Cache (Redis Cluster over Network: 1-2ms)
              │ (Cache MISS?)
              ▼
[ Primary Database Tier (PostgreSQL) ] (10-50ms)
```

---

## 3. Caching Patterns

```
CACHE-ASIDE (Lazy Loading - Industry Standard):
[ App ] ── 1. Read Key ──▶ [ Redis Cache ]
   │                              │
   │ (Cache Miss)                 │ (Cache Hit: Return Data)
   ▼                              ▼
[ Database ] ── 2. Query DB & Set Cache with Jittered TTL ──▶ [ Redis Cache ]

WRITE-THROUGH:
[ App ] ── 1. Write Data ──▶ [ Cache ] ── 2. Synchronous Write ──▶ [ Database ]
* Pros: Cache is never stale.
* Cons: Slower write latency (waits for both cache and DB to commit).

WRITE-BACK (Write-Behind):
[ App ] ── 1. Write Data ──▶ [ Cache ] ── (Returns 200 OK immediately!)
                               │
                               ▼ 2. Background worker flushes batched writes asynchronously
                             [ Database ]
* Pros: Ultra-fast writes! (Used in real-time gaming, telemetry).
* Cons: Risk of permanent data loss if the cache node crashes before the flush.
```

---

## 4. Cache Invalidation & Eviction Algorithms

When Redis reaches `maxmemory`, it must discard keys based on an eviction policy:
1. **LRU (Least Recently Used):** Evicts keys that have not been accessed for the longest time.
2. **LFU (Least Frequently Used):** Tracks an access counter and evicts keys with the lowest hit frequency.
3. **Volatile-TTL:** Evicts keys with an explicit expiration date first.

---

## 5. The Three Classic Cache Disasters & Production Fixes

### A. Cache Penetration (Malicious / Non-Existent Keys)
- **Problem:** Attackers request non-existent IDs (`/users/-999999`). Cache misses every time, slamming the DB directly.
- **Fix 1: Bloom Filter.** A space-efficient probabilistic data structure placed in front of cache. If Bloom filter returns `false`, the key definitely does not exist $ightarrow$ return 404 immediately!
- **Fix 2: Cache Empty Objects.** Store `{"status": "NOT_FOUND"}` in Redis with a short 60s TTL.

### B. Cache Avalanche (Mass Expiration Spike)
- **Problem:** Millions of product keys seeded with a 24-hour TTL simultaneously expire at midnight. All traffic crashes into the database at once!
- **Fix: Jittered Expiration.** Add random noise to TTL:
  $$	ext{TTL} = 	ext{baseTTL} + 	ext{random}(0, 300	ext{ seconds})$$

### C. Cache Breakdown / Stampede (Hot Key Expiry)
- **Problem:** A super-hot key (e.g. World Cup live score with 100,000 QPS) expires. 100,000 requests miss simultaneously and trigger 100,000 duplicate DB queries!
- **Fix: Distributed Mutex Lock (Redisson / SingleFlight).** Only the first thread that misses cache acquires a distributed lock to query DB and populate Redis. The other 99,999 threads wait or return stale data.

---

## 6. Redis Internal Architecture

```
REDIS ARCHITECTURE:
- Single-threaded event loop driven by epoll / kqueue multiplexing.
- Zero thread context-switching overhead, zero lock contention!
- In-Memory Data Structures:
  • Strings (Simple Dynamic String - SDS)
  • Hashes (ziplist -> hashtable)
  • Lists (quicklist: linked list of ziplists)
  • Sets (intset -> hashtable)
  • Sorted Sets (ZSET): SkipList + HashTable (O(log N) leaderboard lookups!)
  • HyperLogLog: Estimates cardinality of billions of items with 12KB RAM.
```

---

## 7. Java / Spring Boot Cache Stampede Protection with Redisson

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductCacheService {

    private final StringRedisTemplate redisTemplate;
    private final ProductRepository productRepository;
    private final RedissonClient redissonClient;
    private final ObjectMapper objectMapper;

    public Product getProduct(Long productId) {
        String cacheKey = "product:" + productId;
        String cachedJson = redisTemplate.opsForValue().get(cacheKey);

        if (cachedJson != null) {
            return deserialize(cachedJson); // Cache HIT
        }

        // Cache MISS -> Prevent Cache Breakdown / Stampede via Distributed Lock
        String lockKey = "lock:product:" + productId;
        RLock lock = redissonClient.getLock(lockKey);

        try {
            // Wait up to 500ms to acquire lock, hold for max 2 seconds
            if (lock.tryLock(500, 2000, TimeUnit.MILLISECONDS)) {
                try {
                    // Double-check cache (another thread might have populated it while waiting)
                    cachedJson = redisTemplate.opsForValue().get(cacheKey);
                    if (cachedJson != null) {
                        return deserialize(cachedJson);
                    }

                    // Query Database (Single thread only!)
                    Product product = productRepository.findById(productId)
                        .orElseThrow(() -> new ResourceNotFoundException("Product not found"));

                    // Write to Redis with Jittered TTL (10 mins + 0-60s random jitter)
                    long jitter = ThreadLocalRandom.current().nextLong(0, 60);
                    redisTemplate.opsForValue().set(
                        cacheKey, serialize(product), Duration.ofSeconds(600 + jitter)
                    );

                    return product;
                } finally {
                    lock.unlock();
                }
            } else {
                // Could not acquire lock: wait briefly and retry cache read
                Thread.sleep(50);
                return getProduct(productId);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Interrupted during cache retrieval", e);
        }
    }
}
```

---

## 8. Side-by-Side Trade-off Analysis Table

| Caching Pattern | Read Latency | Write Latency | Stale Data Risk | Complexity |
| :--- | :--- | :--- | :--- | :--- |
| **Cache-Aside** | Fast on hit, slow on miss | Fast (normal DB write)| Medium (until TTL or manual evict) | Low |
| **Write-Through** | Fast always | Slower (2 sync writes)| Low (immediate update) | Medium |
| **Write-Back** | Fastest | Fastest (writes to RAM)| High (risk of data loss on crash) | High |

---

## 9. Interview Rapid Q&A Checklist
- *How does Redis persist in-memory data to disk?* (Via **RDB snapshots** taken at intervals and **AOF (Append-Only File)** logging every write command; Redis 4+ uses hybrid RDB + AOF persistence).
- *What is Redis Sentinel vs Redis Cluster?* (Sentinel provides High Availability failover for a single master-replica set; Redis Cluster provides automatic horizontal sharding across 16,384 hash slots).
