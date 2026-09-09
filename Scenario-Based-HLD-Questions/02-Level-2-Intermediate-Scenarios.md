# 🏛️ Module 02 — Level 2: Intermediate Production Scenarios (3–6 Years Experience)

This module focuses on distributed interactions across multiple services, databases, caching layers, queues, distributed locks, real-time connections, and search engines.

Every scenario strictly follows the **12-Step FAANG/Startup Interview Blueprint**.

---

# HLD Scenario #09 — The Celebrity Problem & Hot Partitions in Distributed Databases

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 2 (3–6 Years Experience)
- **Topics**: Data Partitioning • Sharding • Fanout-on-Write vs Read • Hot Keys • NoSQL
- **Company Examples**: Twitter/X, Instagram, TikTok, LinkedIn

---

### 1. Interviewer Question
> *"You are designing a Twitter/X or Instagram feed system. Your database is sharded by `user_id`. When a normal user posts, it writes to one shard and updates 200 followers. When a celebrity with 80 million followers posts, that single database shard catches fire, CPU hits 100%, and millions of notifications stall. How do you redesign your data partitioning and fanout architecture?"*

---

### 2. What the Interviewer is Actually Testing
- **Hot Partitions & Skewed Key Distribution**: Why uniform hash sharding fails under asymmetric user followings.
- **Fanout-on-Write (Push) vs Fanout-on-Read (Pull) vs Hybrid Fanout**.
- **Partition Salting & Secondary Partitioning Keys**.
- **Distributed In-Memory Timeline Caching (Redis Sorted Sets)**.

---

### 3. First 30 Seconds Thinking Framework
> 💡 **Golden Interview Line**:  
> *"The root cause is write amplification and hot partition skew caused by naive Fanout-on-Write. If we write to 80 million inboxes for one post, we create an 80-million-to-one write amplification. My strategy is a **Hybrid Fanout Model**: Push for 99.9% of normal users, and Pull for the 0.1% of celebrities."*

---

### 4. HLD Whiteboard Architecture Diagram

```
                    [ User Creates Post ]
                              │
                              ▼
                     [ Post Service ]
                              │
               Is Author a Celebrity (> 25k followers)?
                     ├─── YES ──► [ Write to Celebrity Timeline DB / Redis ]
                     │            (Do NOT push to 80M follower inboxes!)
                     │
                     └─── NO  ──► [ Push to Kafka / Fanout Workers ]
                                                │
                                                ▼
                                  [ Write to Follower Inboxes ]
                                  (Fanout-on-Write for regular users)

                    [ Follower Opens App ]
                              │
                              ▼
                     [ Feed Home Service ]
                              │
                1. Fetch user's Precomputed Timeline (from Redis)
                2. Check user's followed celebrities list
                3. Fetch latest posts from Celebrity DB/Cache (Fanout-on-Read)
                4. Merge & Rank results in-memory
                              │
                              ▼
                     [ Rendered Timeline ]
```

---

### 5. Component-by-Component Responsibilities

| Component | Responsibility & Configuration |
| :--- | :--- |
| **Post Ingestion Service** | Classifies authors as regular users vs celebrities based on follower count threshold (e.g., 25,000 followers). |
| **Kafka Fanout Topic** | Dedicated topic for regular user fanout; decoupled from celebrity ingestion. |
| **Fanout Worker Fleet** | Pulls regular posts and inserts `post_id` into follower timeline Redis Sorted Sets (`ZADD`). |
| **Celebrity Cache (Redis)** | Holds the last 100 posts of all celebrities; highly replicated across read clusters. |
| **Timeline Aggregator** | K-way merge engine that merges a user's precomputed timeline with live celebrity posts at read time. |

---

### 6. End-to-End Request Flow (Step-by-Step)

#### Normal User Posts:
1. Normal user posts $\rightarrow$ `Post Service` writes to Posts DB.
2. Emits message to `Kafka: fanout-posts`.
3. Fanout workers fetch follower IDs (e.g. 200 followers) and execute `ZADD timeline:user_id <timestamp> <post_id>` into each follower's Redis timeline inbox.

#### Celebrity Posts:
1. Celebrity posts $\rightarrow$ `Post Service` writes to Posts DB.
2. Appends `post_id` strictly to the **Celebrity's Own Post List** in Redis.
3. Zero fanout workers are triggered; zero follower inboxes are touched.

#### Follower Reads Feed:
1. Follower opens app $\rightarrow$ `Feed Service` fetches user's precomputed Redis timeline.
2. Checks which celebrities the user follows.
3. Fetches recent posts for those celebrities from the **Celebrity Cache**.
4. Merges regular timeline and celebrity posts in-memory using a priority queue (k-way merge) by timestamp and returns top 20 posts.

---

### 7. Failure Mode Analysis Matrix

| Failure Point | Impact | Mitigation Strategy |
| :--- | :--- | :--- |
| **Celebrity Read Hotspot** | Millions of users read the same celebrity key | **Salt the cache key** across multiple Redis nodes (`celebrity:elono:1`, `celebrity:elono:2`). |
| **User Crosses Celebrity Threshold** | Fanout queue suddenly explodes | Asynchronously migrate user classification via background job. |
| **Redis Timeline Node Dies** | Followers lose cached timelines | Reconstruct timeline on-demand from Primary DB via write-back worker. |

---

### 8. Deep-Dive Trade-Off Analysis
- **Push vs Pull vs Hybrid**:
  - *Pure Push (Fanout-on-Write)*: Blazing fast reads ($O(1)$ lookup in Redis), but breaks under celebrity writes ($O(80\text{M})$ writes per post).
  - *Pure Pull (Fanout-on-Read)*: $O(1)$ write, but terrible read performance ($O(N)$ database queries across hundreds of followed users when generating the home feed).
  - *Hybrid*: Optimal sweet spot balancing read latency with write throughput.

---

### 9. Curveball Follow-Ups

#### 🌀 Curveball 1: *"What if a celebrity's own partition in DynamoDB or Cassandra catches fire under 100k read QPS?"*
> **Answer**: Apply **Partition Key Salting**. Append a random integer from 1 to 20 to the partition key (`user_123#salt_4`). Read requests query across all 20 salted partitions using parallel scatter-gather, distributing read IOPS evenly across 20 physical database shards.

---

### 10. Strong Interview-Style Answer (2–3 Minutes)
> *"The failure is caused by write amplification from pure Fanout-on-Write. When an account with 80 million followers posts, pushing that single post to 80 million follower inboxes causes massive database shard and queue saturation.
> 
> I solve this using a **Hybrid Fanout Architecture**:
> 
> 1. **User Classification**: Authors are classified into regular users (<25k followers) and celebrities (>25k followers).
> 2. **Fanout-on-Write for Regular Users**: For 99.9% of users, posts are pushed to followers' precomputed Redis timelines via Kafka fanout workers.
> 3. **Fanout-on-Read for Celebrities**: When a celebrity posts, we write only to the celebrity's own post list in Redis. No fanout occurs.
> 4. **Read-Time Merge**: When a user loads their feed, we pull their precomputed timeline from Redis, fetch recent posts from the few celebrities they follow, and execute an in-memory k-way merge sorted by timestamp.
> 
> This cuts write amplification from 80 million writes down to 1 write, while maintaining sub-50ms feed load times."*

---

### 11. Real-World Production Battle-Tested Case Studies
- **Twitter/X**: Twitter famously migrated from pure fanout-on-write to a hybrid architecture after celebrity tweets regularly caused cascading Kafka consumer lag and timeline cache crashes.
- **Instagram**: Employs a similar hybrid model for high-follower accounts combined with Memcached replication.

---

### 12. 60-Second Last-Minute Revision Cheat Sheet
- **Root Cause**: Write amplification from pushing to 80M inboxes.
- **Solution**: Hybrid Fanout (Push for normal users, Pull for celebrities).
- **Celebrity Partition Hotspot**: Partition Key Salting (`key#salt`).
- **Read Path**: K-way merge in memory between precomputed timeline + celebrity cache.

---

# HLD Scenario #10 — Cache Stampede / Thundering Herd When Redis Crashes

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 2 (3–6 Years Experience)
- **Topics**: Caching • Redis • Cache Stampede • Singleflight • Circuit Breakers
- **Company Examples**: Reddit, Discord, Netflix, Cloudflare

---

### 1. Interviewer Question
> *"Your e-commerce platform relies on Redis to cache product catalog data. Redis suffers a network partition and crashes for 3 minutes. Instantly, all 20,000 requests/sec bypass the cache and hit PostgreSQL directly. The database pool saturates in 2 seconds, and the entire platform goes dark. How do you engineer the system so that losing Redis does NOT bring down the database?"*

---

### 2. What the Interviewer is Actually Testing
- **Cache Stampede / Thundering Herd Problem**.
- **Request Coalescing (Singleflight Pattern)**.
- **Multi-Layer Caching (L1 Local Memory + L2 Distributed Redis)**.
- **Probabilistic Early Expiration (XFetch Algorithm)**.
- **Database Circuit Breaking & Graceful Fallbacks**.

---

### 3. First 30 Seconds Thinking Framework
> 💡 **Golden Interview Line**:  
> *"When a shared cache goes down, the database must never be exposed to raw, unbuffered production traffic. We must implement two protective shields: an **In-Process L1 Memory Cache** with stale serving capabilities, and **Request Coalescing (Singleflight)** so that only 1 query per app server ever reaches the database."*

---

### 4. HLD Whiteboard Architecture Diagram

```
   [ 20,000 Concurrent Requests for Product #42 ]
                         │
                         ▼
               [ API Web Server Fleet ]
                         │
        Check L1 Local In-Memory Cache (5s TTL)
           ├── [HIT] ──► Return to user immediately (0 network calls)
           │
           └── [MISS] ──► Query Redis (L2)
                              │
                     [ Redis is DOWN / TIMEOUT ]
                              │
                              ▼
            ┌──────────────────────────────────────────┐
            │       Singleflight / Mutex Lock          │
            │   Only 1 single worker thread per node   │
            │       is permitted to query the DB       │
            └─────────────────────┬────────────────────┘
                                  │
                  (1 query) ──────┴────── (19,999 wait on promise)
                     │
                     ▼
             [ Primary Database ]
            (Receives only 10 QPS instead of 20,000 QPS!)
```

---

### 5. Component-by-Component Responsibilities

| Component | Responsibility & Configuration |
| :--- | :--- |
| **L1 Local Memory Cache** | Caffeine (Java) or `sync.Map` (Go) inside application memory with 5-second TTL and `stale-if-error` policy. |
| **Singleflight Mutex** | Synchronizes concurrent requests for the exact same key within an app process; passes the result of the first query to all waiting callers. |
| **Database Circuit Breaker** | Trips to OPEN if DB latency exceeds 500ms or connection pools exceed 90%, returning degraded cached fallbacks. |
| **PostgreSQL Database** | Protected behind connection poolers (PgBouncer); receives only throttled, coalesced queries. |

---

### 6. Strong Interview-Style Answer (2–3 Minutes)
> *"To guarantee that a Redis crash never takes down the underlying database, I deploy three layers of defense:
> 
> 1. **In-Process Request Coalescing (Singleflight)**: Using Go's `singleflight.Group` or Java's `LoadingCache`, duplicate concurrent requests for the same product key on an application node are collapsed into a single inflight request. Instead of 20,000 queries hitting the database, only 1 query per application pod is dispatched to PostgreSQL.
> 2. **L1 Process Cache with Stale-If-Error**: Each application pod maintains an in-memory Caffeine cache with a 5-second TTL. If Redis times out or crashes, pods automatically serve stale data from their local memory rather than hitting the database.
> 3. **Probabilistic Early Expiration (XFetch)**: To prevent keys from expiring all at once, we use the XFetch algorithm to probabilistically recalculate and re-warm cached keys in the background before they expire.
> 4. **Database Circuit Breaker**: If DB connection pool utilization reaches 85%, a circuit breaker trips immediately, returning static fallback data or HTTP 503 instead of allowing the database to crash."*

---

### 7. 60-Second Last-Minute Revision Cheat Sheet
- **Root Cause**: All traffic bypasses failed cache and hits DB concurrently.
- **Singleflight Pattern**: Collapse 10k duplicate requests into 1 DB query per app pod.
- **L1 In-Process Cache**: Caffeine in RAM with `stale-if-error` fallback.
- **TTL Jitter & XFetch**: Jitter TTLs by $\pm 20\%$ to prevent simultaneous expiration.

---

# HLD Scenario #11 — Cross-Service Distributed Transactions (The Saga Pattern)

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 2 (3–6 Years Experience)
- **Topics**: Distributed Transactions • Saga Pattern • Orchestration vs Choreography • Microservices
- **Company Examples**: Uber, DoorDash, Swiggy, Amazon

---

### 1. Interviewer Question
> *"In a food delivery app (DoorDash/UberEats), an order requires four operations across four microservices:
> 1. Reserve Restaurant Meal
> 2. Deduct Customer Wallet Balance
> 3. Assign Delivery Driver
> 4. Confirm Order
> 
> The first two succeed, but no drivers are available. How do you roll back the money and restaurant reservation without using distributed two-phase commit (2PC)?"*

---

### 2. What the Interviewer is Actually Testing
- **Why Two-Phase Commit (2PC / XA) Fails at Scale**: Network locks, coordinator single point of failure, latency amplification.
- **Saga Pattern**: Orchestration vs Choreography.
- **Compensating Transactions & Forward/Backward Recovery**.
- **Idempotency in Rollbacks**.

---

### 3. First 30 Seconds Thinking Framework
> 💡 **Golden Interview Line**:  
> *"Two-Phase Commit is an anti-pattern in distributed microservices because it holds database locks across network boundaries. I will implement an **Orchestrated Saga Pattern with Compensating Transactions**, ensuring that if any forward step fails, compensating undo actions restore eventual consistency."*

---

### 4. Orchestrated Saga Workflow Diagram

```
                   [ Order Orchestrator ]
                             │
     ┌───────────────────────┼───────────────────────┐
  1. Create Pending          │                       │
     │                       ▼                       ▼
     ▼              [ Wallet Service ]      [ Delivery Service ]
[ Kitchen Service ]          │                       │
     │                 2. Deduct $30                 │
     │                       │                       │
     │                 Payment OK                    │
     │                       └───────────────► 3. Find Driver
     │                                               │
     │                                         Driver NOT Found!
     │                                         (Rollback Triggered)
     │                                               │
     ◄──────────── 4. Compensate: Refund $30 ◄───────┘
  5. Compensate: Cancel Kitchen Order
```

---

### 5. Strong Interview-Style Answer (2–3 Minutes)
> *"Distributed 2PC cannot scale across microservices because it is synchronous, locks database resources across network boundaries, and introduces a single point of failure at the coordinator.
> 
> I would implement an **Orchestrated Saga Pattern with Compensating Transactions**:
> 
> 1. **Order Orchestrator**: A stateful orchestrator (using a workflow engine like Temporal or AWS Step Functions) manages the saga lifecycle and state transitions.
> 2. **Forward Workflow**:
>    - Step 1: `ReserveRestaurantOrder` $\rightarrow$ Success
>    - Step 2: `DeductWalletBalance` $\rightarrow$ Success
>    - Step 3: `AssignDeliveryDriver` $\rightarrow$ Fails (No drivers available).
> 3. **Compensating Rollback**: Upon failure of Step 3, the orchestrator initiates compensating actions in reverse order:
>    - Invokes `RefundWalletBalance(transaction_id)`
>    - Invokes `CancelRestaurantOrder(order_id)`
>    - Updates order state to `FAILED_NO_DRIVERS`.
> 4. **Strict Idempotency**: Every compensating action must be strictly idempotent. If the network drops during the refund call, the orchestrator retries until it receives an explicit acknowledgment without double-refunding."*

---

### 6. 60-Second Last-Minute Revision Cheat Sheet
- **Why NOT 2PC**: Locks rows across network, high latency, blocking coordinator.
- **Saga Pattern**: Sequence of local transactions coordinated via events or an orchestrator.
- **Compensating Action**: Undo logic (e.g., Refund for Charge, Cancel for Reserve).
- **Temporal / Step Functions**: Standard state machine engines for Orchestration.

---

# HLD Scenario #12 — Real-Time Connection Scale (10 Million Concurrent WebSockets)

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 2 (3–6 Years Experience)
- **Topics**: WebSockets • Real-Time Systems • C10M Problem • Pub/Sub • Epoll
- **Company Examples**: WhatsApp, Slack, Discord, Robinhood

---

### 1. Interviewer Question
> *"You are building a live sports scoring or trading system where 10 million concurrent users receive price updates every second. How do you architect the gateway, maintain open persistent connections without running out of server memory, and broadcast updates with sub-200ms latency?"*

---

### 2. What the Interviewer is Actually Testing
- **C10K / C10M Socket Concurrency Limits**: Memory overhead per TCP socket, OS file descriptors.
- **Stateful Connection Gateways vs Stateless Business Logic**.
- **Hierarchical Pub/Sub Fanout Layer (NATS / Redis Cluster)**.
- **Reconnect Storm Mitigation (Jittered Exponential Backoff)**.

---

### 3. Architecture Diagram

```
[ 10 Million Mobile / Web Clients ]
                 │ (Persistent TLS WebSockets)
                 ▼
[ Layer 4 TCP Load Balancer (AWS NLB) ]
                 │
  ┌──────────────┼──────────────┐
  ▼              ▼              ▼
[ WS Node 1 ]  [ WS Node 2 ]  [ WS Node N ]  (Fleet of ~100 servers, 100k conns each)
  │              │              │
  └──────────────┼──────────────┘
                 │
          (Subscribe to topic: "match_42")
                 │
                 ▼
    [ Distributed Pub/Sub Layer ]
     (NATS / Redis Cluster / Kafka)
                 ▲
                 │ (Publish score update: "GOAL! 1-0")
    [ Match Event Ingestion Service ]
```

---

### 4. Strong Interview-Style Answer (2–3 Minutes)
> *"To support 10 million concurrent WebSockets, I separate stateful connection gateways from stateless business logic:
> 
> 1. **L4 Load Balancing**: Place AWS Network Load Balancers (NLB) in front to distribute raw TCP connections via round-robin or least-connections to a fleet of lightweight WebSocket gateway nodes.
> 2. **Optimized Socket Gateways**: Build gateways in Go (using non-blocking epoll / Goroutines) or Rust/Netty. We tune Linux OS kernel limits (`fs.file-max`, ephemeral port ranges, and TCP read/write buffer minimization to 4KB per socket). Each server handles 100k active connections using ~30GB RAM, requiring a fleet of 100 servers.
> 3. **Hierarchical Pub/Sub Fanout**: Use a high-throughput messaging bus like **NATS Core**. Gateway nodes aggregate subscriptions: an app server subscribes to `match:123` *once* at the broker, and fans out the received message locally in-memory to all 5,000 local sockets watching that match.
> 4. **Reconnect Storm Prevention**: If a gateway crashes, 100,000 clients will reconnect instantly. Clients must implement **Exponential Backoff with Full Jitter** to smoothly distribute reconnection requests across several minutes."*

---

### 5. 60-Second Last-Minute Revision Cheat Sheet
- **OS Tuning**: `ulimit -n 1048576`, tune `tcp_rmem` and `tcp_wmem` to 4KB.
- **Fleet Sizing**: 100k connections per node $\times$ 100 nodes = 10 Million connections.
- **Fanout Optimization**: Server subscribes *once* to message bus; fans out in RAM to local sockets.
- **Thundering Reconnect Fix**: Exponential backoff with random jitter on client reconnect.

---

# HLD Scenario #13 — Distributed Rate Limiting at Scale (Distributed Token Bucket)

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 2 (3–6 Years Experience)
- **Topics**: Rate Limiting • Redis • Lua Scripting • Token Bucket • API Gateway
- **Company Examples**: Stripe, Cloudflare, Twilio, GitHub

---

### 1. Interviewer Question
> *"You need to rate limit API consumers to 1,000 requests per minute across a cluster of 50 API Gateway instances. If you store rate counts in Redis, the network latency of making 2 Redis calls (`GET` + `INCR`) per HTTP request doubles your API latency and causes race conditions. How do you design this?"*

---

### 2. What the Interviewer is Actually Testing
- **Distributed Token Bucket vs Sliding Window Counter**.
- **Atomic Operations in Redis using Lua Scripting**.
- **Batching & Local Token Pre-Allocation**.
- **Fail-Open vs Fail-Closed Strategy**.

---

### 3. Redis Lua Script (Atomic Token Replenishment & Consumption)

```lua
-- Atomic Token Bucket Execution in Redis
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call("HMGET", key, "tokens", "last_updated")
local tokens = tonumber(data[1])
local last_updated = tonumber(data[2])

if not tokens then
    tokens = limit
    last_updated = now
else
    local delta = math.max(0, now - last_updated)
    tokens = math.min(limit, tokens + delta * rate)
    last_updated = now
end

if tokens >= requested then
    tokens = tokens - requested
    redis.call("HMSET", key, "tokens", tokens, "last_updated", last_updated)
    redis.call("EXPIRE", key, 3600)
    return 1 -- ALLOW
else
    return 0 -- REJECT (HTTP 429)
end
```

---

### 4. Strong Interview-Style Answer (2–3 Minutes)
> *"To implement rate limiting across 50 gateway nodes without race conditions or latency penalties:
> 
> 1. **Atomic Token Bucket with Redis Lua**: A naive `GET` and `INCR` suffers from a classic race condition and requires two round trips. Using an atomic **Redis Lua script**, token replenishment and consumption occur in a single atomic server-side execution without distributed locks.
> 2. **Token Batching for Ultra-Low Latency**: Under extreme traffic, querying Redis for every request introduces latency. Gateway pods pre-allocate a chunk of tokens (e.g., 50 tokens at a time) from Redis and consume them locally in memory.
> 3. **Sliding Window Counter Alternative**: If strict window boundaries are required, use Redis Sorted Sets (`ZREMRANGEBYSCORE`, `ZCARD`, `ZADD`) executed in a pipeline to prune expired timestamps and verify capacity.
> 4. **Fail-Open Resilience**: If Redis goes down completely, the rate limiter must **fail open** (log a warning and let requests pass) rather than blocking all legitimate production traffic."*

---

### 5. 60-Second Last-Minute Revision Cheat Sheet
- **Lua Script**: Eliminates Read-Modify-Write race conditions in Redis.
- **Local Pre-allocation**: Gateway fetches 50 tokens at once to minimize Redis network hops.
- **Fail-Open**: Never bring down production if Redis crashes; fail open with warning logs.

---

# HLD Scenario #14 — Distributed Lock Leasing, Fencing Tokens & Clock Drift

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 2 (3–6 Years Experience)
- **Topics**: Distributed Locks • Redlock • Fencing Tokens • Concurrency • Clock Drift
- **Company Examples**: Airbnb, Google, AWS, Stripe

---

### 1. Interviewer Question
> *"You use Redis (Redlock or `SETNX with TTL`) to ensure only one worker processes a specific recurring bank transfer at a time. Worker A acquires the lock for 10 seconds. However, Worker A suffers a 15-second Stop-The-World (STW) Garbage Collection pause. The lock expires. Worker B acquires the lock and begins writing to the database. Worker A wakes up and also writes to the database, corrupting the transfer. How do you solve this?"*

---

### 2. What the Interviewer is Actually Testing
- **Martin Kleppmann's Redlock Critique**.
- **Process Pauses (GC STW), Clock Drift, and Unreliable Locks**.
- **Fencing Tokens (Monotonically Increasing Sequences)**.
- **Optimistic Concurrency Control at the Storage Layer**.

---

### 3. Fencing Token Architectural Solution

```
[ Worker A ]                         [ Redis Lock ]                    [ Storage / DB ]
     │                                     │                                  │
     │ 1. Acquire lock (Gets Token = 33) ──►                                  │
     │                                     │                                  │
[ Long GC STW Pause! (15s) ]               │                                  │
     │   (Lock TTL expires in Redis)       │                                  │
     │                                     │                                  │
     │                           [ Worker B ]                                 │
     │                                │                                       │
     │                                ├── 2. Acquire lock (Token = 34) ───────┤
     │                                │   (Lock Granted)                      │
     │                                ├── 3. Write to DB (Token = 34) ────────┼─► [ DB: Current Token = 34 ]
     │                                │      (Write Accepted!)                │
[ Worker A Wakes Up! ]                │                                       │
     │                                                                        │
     └── 4. Write to DB (Token = 33) ─────────────────────────────────────────┼─► [ REJECTED! ]
                                                                                  DB checks: 33 < 34
                                                                                  Write Dropped!
```

---

### 4. Strong Interview-Style Answer (2–3 Minutes)
> *"This is the fundamental limitation of distributed locks: **a distributed lock alone cannot guarantee safety against process pauses (like GC pauses or network freezes)** because the worker has no way of knowing its lease has expired without contacting the lock manager before every write.
> 
> To guarantee correctness, we must use **Fencing Tokens**:
> 
> 1. **Monotonic Sequence Token**: Whenever a worker acquires a lock, the lock service increments and returns a monotonically increasing sequence number (e.g. Lock Server returns Token 33 for Worker A, Token 34 for Worker B).
> 2. **Storage-Level Fencing**: The storage layer (PostgreSQL, DynamoDB, or S3) enforces that it will only accept a write if the incoming request's fencing token is **strictly greater** than the highest token it has already processed:
>    ```sql
>    UPDATE bank_transfers 
>    SET status = 'PROCESSED', last_token = 34 
>    WHERE id = 101 AND last_token < 34;
>    ```
> 3. **Result**: When Worker A wakes up from its GC pause and attempts to write with Token 33, the database checks `33 < 34` and rejects the write, eliminating corruption."*

---

### 5. 60-Second Last-Minute Revision Cheat Sheet
- **The Core Flaw**: GC pauses and network freezes invalidate distributed locks without the worker knowing.
- **Fencing Token**: Monotonically increasing number returned with every lock lease.
- **Storage-Level Check**: Database rejects any write whose fencing token is older than the current token.

---

# HLD Scenario #15 — Search Index Lag & Asynchronous Elasticsearch Sync

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 2 (3–6 Years Experience)
- **Topics**: Elasticsearch • Search Systems • CDC • Debezium • Read-Your-Own-Writes
- **Company Examples**: Shopify, Elastic, Etsy, Amazon

---

### 1. Interviewer Question
> *"Your e-commerce application stores product listings in PostgreSQL and syncs them to Elasticsearch for full-text search. Customers complain that when they create or edit a listing, it takes 30 seconds to appear in search results, or sometimes disappears completely. What causes this lag, and how do you achieve near-real-time search indexing?"*

---

### 2. What the Interviewer is Actually Testing
- **Dual-Write Anti-Pattern vs Change Data Capture (CDC)**.
- **Elasticsearch Refresh Interval vs Lucene Segment Flushes**.
- **Bulk Indexing (`_bulk`) Throughput**.
- **Read-Your-Own-Writes Consistency for Creators**.

---

### 3. Architecture: Reliable CDC Pipeline to Elasticsearch

```
[ Merchant Updates Product ]
            │
            ▼
┌───────────────────────────┐
│     PostgreSQL (ACID)     │
│   (Write-Ahead Log: WAL)  │
└───────────┬───────────────┘
            │
            ▼
[ Debezium / Kafka Connect ] ── (Captures WAL row changes with zero app lag)
            │
            ▼
  [ Kafka: product-events ]
            │
            ▼
[ Elasticsearch Indexer Worker ] ── (Bulk indexing API: /_bulk)
            │
            ▼
   [ Elasticsearch Cluster ] ── (Index refresh_interval: 1s)
```

---

### 4. Strong Interview-Style Answer (2–3 Minutes)
> *"Search indexing lag is caused by application-level dual writes and sub-optimal Elasticsearch refresh configurations:
> 
> 1. **CDC via Debezium**: Instead of writing to PostgreSQL and then calling Elasticsearch synchronously in an HTTP handler, use **Change Data Capture (Debezium)** to read PostgreSQL's Write-Ahead Log (WAL) and stream changes to Kafka. This decouples database commits from search indexing.
> 2. **Tune Lucene Refresh Interval**: Elasticsearch memory buffers only become searchable after a refresh. Configure `index.refresh_interval: "1s"` on active search indices.
> 3. **Batch Bulk Indexing**: Indexer workers consume from Kafka and write to Elasticsearch using the `_bulk` API in batches of 500 documents or 5MB, maximizing ingestion throughput.
> 4. **Read-Your-Own-Writes User Experience**: When a merchant saves an edit, serve their immediate confirmation view directly from PostgreSQL, while public global search queries use the near-real-time (<1 second) Elasticsearch index."*

---

### 5. 60-Second Last-Minute Revision Cheat Sheet
- **Anti-Pattern**: Synchronous dual-writes from API server to both DB and Elasticsearch.
- **Best Practice**: Postgres WAL $\rightarrow$ Debezium $\rightarrow$ Kafka $\rightarrow$ Bulk Indexer $\rightarrow$ Elasticsearch.
- **Refresh Interval**: Set `refresh_interval: "1s"`.
- **User UX**: Serve creator's edit confirmation from primary DB; serve public search from Elasticsearch.

---

# HLD Scenario #16 — Resilient Direct-to-Storage Media Uploads (S3 Presigned URLs)

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 2 (3–6 Years Experience)
- **Topics**: Object Storage • Amazon S3 • Presigned URLs • Multipart Uploads • Video Processing
- **Company Examples**: YouTube, TikTok, Netflix, Google Drive

---

### 1. Interviewer Question
> *"Users need to upload 2GB video files to your platform. If uploads go through your application backend servers, server memory explodes, connections time out, and AWS bandwidth egress bills soar. How do you design a resilient, high-speed upload architecture?"*

---

### 2. What the Interviewer is Actually Testing
- **Bypassing Application Servers via S3 Presigned URLs**.
- **Multipart Uploads & Resumable Upload Protocols (TUS)**.
- **Asynchronous Processing via S3 Event Notifications**.
- **Cost & Bandwidth Egress Optimization**.

---

### 3. Architecture Flow

```
[ Client App / Browser ]
      │
      │ 1. POST /api/v1/videos/upload-init { filename: "vid.mp4", size: 2GB }
      ▼
[ API Gateway / Backend ]
      │ 2. Authenticate user & generate S3 Multi-part Presigned URLs
      │    (Part 1 URL, Part 2 URL, ... Part N URL)
      ▼
[ Client receives Presigned URLs ]
      │
      │ 3. Upload chunks (5MB each) DIRECTLY to S3 in parallel
      ├───────────────────────────────────────────────┐
      ▼                                               ▼
[ S3 Storage Bucket ] ◄───────────────────────────────┘
      │
      │ 4. Client sends: CompleteMultipartUpload
      │ 5. S3 triggers ObjectCreated Event
      ▼
[ AWS SQS / EventBridge ]
      │
      ▼
[ Transcoding Worker Fleet (FFmpeg) ] ──► Compresses to 1080p, 720p, HLS/DASH
      │
      ▼
[ S3 Processed Media Bucket ] ──► [ CloudFront CDN ]
```

---

### 4. Strong Interview-Style Answer (2–3 Minutes)
> *"Routing multi-gigabyte video uploads through backend application servers exhausts server memory and doubles cloud egress costs. I design a **Direct-to-Object-Storage Architecture**:
> 
> 1. **Presigned Multipart Upload Authorization**: The client requests an upload token. The API server authenticates the user, verifies storage quotas, calls Amazon S3 to initialize a multipart upload, and returns a list of signed presigned URLs for 5MB chunks.
> 2. **Direct Browser-to-S3 Parallel Upload**: The client uploads chunks directly to Amazon S3 in parallel over HTTP/2, completely bypassing our backend servers.
> 3. **Resumability**: If network connectivity drops at 1.8GB, the client only retries the failed 5MB chunks rather than restarting from 0MB.
> 4. **Event-Driven Asynchronous Processing**: Upon completion, S3 emits an `ObjectCreated` event to AWS SQS. A fleet of transcoding workers running FFmpeg pulls messages, encodes the video into multiple bitrates (HLS/DASH), and stores outputs in a public S3 bucket fronted by CloudFront CDN."*

---

### 5. 60-Second Last-Minute Revision Cheat Sheet
- **Never stream large files through app servers**: Direct-to-S3 presigned URLs.
- **Multipart Upload**: Split 2GB file into 5MB chunks; enables parallel uploads and resumability.
- **Event-Driven Transcoding**: S3 `ObjectCreated` event $\rightarrow$ SQS $\rightarrow$ Worker fleet.
