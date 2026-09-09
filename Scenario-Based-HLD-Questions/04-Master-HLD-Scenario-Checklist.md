# 🏆 Module 04 — Master HLD Scenario Checklist & Mindset Guide

This is your battle-tested reference guide. When an interviewer gives you an unfamiliar scenario under high pressure, use this framework to structure your thoughts, ask the right questions, and deliver a clean architectural solution.

---

## 🧭 The 7-Step HLD Interview Execution Framework

Never start drawing boxes or proposing databases in the first 2 minutes. Follow this systematic script:

```
[ Step 1: Scope & Clarify ] ──► [ Step 2: Capacity & Constraints ] ──► [ Step 3: High-Level Core Flow ]
                                                                                   │
[ Step 6: Failure Modes & Edge Cases ] ◄── [ Step 5: Deep Dive Components ] ◄─────┘
               │
               ▼
[ Step 7: Bottlenecks & Trade-Offs ]
```

### Step 1: Clarify Functional & Non-Functional Requirements (3–5 mins)
- *"Who is calling this API? (Mobile, Web, Internal Services, 3rd party webhooks?)"*
- *"What is the Read-to-Write ratio? (100:1 like Twitter, or 1:1 like IoT/Telemetry?)"*
- *"What are the latency expectations? (Sub-50ms P99, or asynchronous eventual delivery?)"*
- *"What is the consistency requirement? (Strict ACID financial consistency or eventual consistency?)"*

### Step 2: Back-of-the-Envelope Capacity Estimations (3 mins)
- **QPS (Queries Per Second)**:
  - $1\text{ Million requests/day} \approx 12\text{ RPS}$
  - $100\text{ Million requests/day} \approx 1,200\text{ RPS average}$, **Peak $\approx 3,000\text{ to } 5,000\text{ RPS}$**
- **Storage Calculation**:
  - Average payload size $\times$ daily write volume $\times$ retention period (e.g. 5 years).
- **Network Bandwidth**:
  - $\text{Read QPS} \times \text{Response Size}$ (watch out for egress costs!).

### Step 3: High-Level Architecture (5–8 mins)
- Start with standard building blocks: Client $\rightarrow$ CDN/WAF $\rightarrow$ Load Balancer $\rightarrow$ API Gateway $\rightarrow$ Stateless Compute Fleet $\rightarrow$ Database.
- Establish the primary happy path first before addressing edge cases.

### Step 4: Deep Dive into Core Bottlenecks (15 mins)
- Pick the 2 most difficult components (e.g., distributed locking, cache invalidation, partitioned message ordering).
- Detail data schemas, partition keys, caching strategies, and indexing.

### Step 5: Resiliency & Failure Scenarios (10 mins)
- *"What happens if the primary database dies?"*
- *"What happens if Redis dies?"*
- *"What happens if a downstream 3rd party hangs?"*

---

## 🧠 The "Think Like an Interviewer" Mindset Matrix

When an interviewer throws a curveball at you, they are looking to see if you instinctively reach for the right architectural pattern:

| What the Interviewer Says... | What They Are Really Testing... | The Architectural Pattern You Must Use |
| :--- | :--- | :--- |
| *"Traffic increased 100x instantly"* | Autoscaling lag, capacity buffers | **Edge caching (CDN), Rate limiting / Throttling, Queue-buffered asynchronous ingestion** |
| *"Users are double-clicking the button"* | Concurrency, non-idempotent endpoints | **Idempotency Keys (UUIDv4) stored in ACID DB unique index + pass to PSP** |
| *"Redis crashed, now the database is dead"* | Thundering herd, cache stampede | **Singleflight (request coalescing), L1 local in-process cache, Jittered TTLs** |
| *"One user has 50M followers and broke the feed"* | Hot partitions, write amplification | **Hybrid Fanout (Push for normal users, Pull for celebrities), Partition salting** |
| *"Payment succeeded, but inventory crashed"* | Distributed consistency, dual-write bug | **Transactional Outbox pattern + CDC (Debezium) + Saga pattern** |
| *"Downstream service is taking 10 seconds"* | Thread pool saturation, cascading outages | **Short read/connect timeouts, Circuit Breaker (Resilience4j/Envoy), Bulkheads** |
| *"Worker held lock for 10s, but paused for 15s"* | Clock drift, GC STW pauses, lock expiry | **Fencing Tokens (Monotonic sequence checked at storage layer)** |
| *"Active-active across two continents"* | WAN latency, CAP theorem, split-brain | **PACELC trade-off, Local writes with CRDTs / HLC timestamps, Asynchronous sync** |
| *"Need RPO=0 and RTO < 1 min during region loss"* | Disaster recovery, DNS caching lag | **Multi-region Raft consensus (CockroachDB/Spanner) + Anycast BGP routing (Cloudflare)** |
| *"10 million users need instant live score updates"* | C10M socket limits, server memory exhaustion | **L4 Network Load Balancers, Epoll/Goroutine socket gateways, Hierarchical Pub/Sub (NATS)** |

---

## ⚡ Curveball Reaction Cheatsheet

### Level 1 Curveballs:
- **"What if the database CPU hits 100%?"**
  - *Answer*: Check `slow_query_log` and `EXPLAIN ANALYZE` for missing composite indexes $\rightarrow$ Add PgBouncer connection pooler $\rightarrow$ Route reads to Read Replicas $\rightarrow$ Cache hot reads in Redis $\rightarrow$ Shard by `user_id`.
- **"What if your API Gateway becomes the single point of failure?"**
  - *Answer*: Deploy multi-zone active-active gateway clusters behind Layer 4 Network Load Balancers (NLB) with Anycast DNS routing and automatic health checks.

### Level 2 Curveballs:
- **"What if the message queue gets backlogged by 50 million messages?"**
  - *Answer*: Scale consumer worker pods horizontally $\rightarrow$ Increase consumer batch size (`fetch.min.bytes`) $\rightarrow$ Check if consumer is blocked on downstream DB row locks $\rightarrow$ Divert poison pills to Dead Letter Queue (DLQ).
- **"What if a user edits a listing, and search results don't show it immediately?"**
  - *Answer*: Use Write-Ahead Log CDC (Debezium) instead of application-level dual-writes $\rightarrow$ Lower Elasticsearch `index.refresh_interval` from default to 1s $\rightarrow$ Read-your-own-writes bypass (serve profile page directly from DB).

### Level 3 Curveballs:
- **"What if an undersea cable cuts between US and Europe during active-active operations?"**
  - *Answer*: Isolate regions to accept local writes; tag records with Hybrid Logical Clocks; use state-based CRDTs for convergent merging once the link heals, or restrict writes to single-leader partitions per tenant.
- **"What if two transfers concurrently swap money between Account A and Account B?"**
  - *Answer*: Circular deadlock! Always sort account IDs deterministically before acquiring row locks (`FOR UPDATE`).

---

## 📋 75-Topic HLD Syllabus Rapid Revision Checklist

Use this checklist to track your preparation across all major High-Level Design domains:

### 1. Fundamentals & Traffic Management
- [ ] **Scalability**: Horizontal (scale-out) vs Vertical (scale-up) limits & stateless tiers
- [ ] **Availability & Reliability**: SLO, SLA, SLI, MTBF, MTTR, calculating nines (99.9% vs 99.99%)
- [ ] **Load Balancing**: L4 (TCP/UDP) vs L7 (HTTP/gRPC), Round Robin, Weighted, Consistent Hashing
- [ ] **API Gateway**: Reverse proxy, TLS termination, path routing, JWT validation, authentication
- [ ] **Rate Limiting & Throttling**: Token Bucket, Leaky Bucket, Fixed Window, Sliding Window Log/Counter
- [ ] **Backpressure & Load Shedding**: Dropping low-priority traffic, bounded worker queues, HTTP 429 vs 503
- [ ] **Edge Acceleration & CDN**: Cloudflare/CloudFront, edge caching, `stale-while-revalidate`, cache purge

### 2. Caching & Memory Architecture
- [ ] **Caching Strategies**: Cache-Aside, Read-Through, Write-Through, Write-Behind (Write-Back)
- [ ] **Eviction Policies**: LRU, LFU, FIFO, TTL-based expiration with random jitter
- [ ] **Cache Pitfalls**: Cache Stampede / Thundering Herd, Cache Penetration (Bloom filters), Cache Breakdown
- [ ] **Multi-Tier Caching**: L1 In-Memory (Caffeine/Local RAM) + L2 Distributed (Redis/Memcached)
- [ ] **Redis Deep Dive**: Data types, Redis Cluster, Sentinel failover, Lua scripts for atomic operations

### 3. Database Scaling & Storage
- [ ] **SQL vs NoSQL**: Relational ACID vs Document, Key-Value, Columnar, Graph models
- [ ] **Database Indexing**: B-Tree, B+Tree, Hash index, LSM-Tree (SSTables), Composite index order
- [ ] **Replication**: Synchronous vs Asynchronous vs Semi-synchronous, Replication Lag handling
- [ ] **Read Replicas & Connection Pooling**: PgBouncer, ProxySQL, HikariCP sizing
- [ ] **Database Partitioning & Sharding**: Range, Hash, List sharding, Directory-based, Re-sharding challenges
- [ ] **Transactions & Isolation Levels**: Read Uncommitted, Read Committed, Repeatable Read, Serializable
- [ ] **Distributed Consistency**: CAP Theorem, PACELC Theorem, Strong vs Eventual vs Causal consistency

### 4. Distributed Systems & Microservices
- [ ] **Service Discovery & Mesh**: Consul, Eureka, Envoy proxy, Istio service mesh
- [ ] **Resilience Patterns**: Timeouts, Retries with Exponential Backoff & Full Jitter, Circuit Breakers, Bulkheads
- [ ] **Distributed Idempotency**: Idempotency Keys, Unique Constraints, Redis SETNX safety
- [ ] **Distributed Locks**: Redis Redlock critique, ZooKeeper / etcd consensus leases, Fencing Tokens
- [ ] **Concurrency & Race Conditions**: Optimistic (OCC with version column) vs Pessimistic Locking (`FOR UPDATE`)
- [ ] **Deadlock Mitigation**: Deterministic sorted lock acquisition, lock timeouts

### 5. Messaging & Asynchronous Event Processing
- [ ] **Message Queues**: RabbitMQ (AMQP push-based) vs Apache Kafka (Log pull-based) vs AWS SQS
- [ ] **Kafka Architecture**: Topics, Partitions, Consumer Groups, Offsets, Rebalance protocol
- [ ] **Stream Processing & Ordering**: In-partition ordering, Reorder buffers, Watermarks, Sliding windows
- [ ] **Transactional Outbox & CDC**: Debezium, Postgres WAL decoding, preventing dual-write bugs
- [ ] **Distributed Transactions**: Saga Pattern (Orchestration vs Choreography), Compensating actions

### 6. Real-Time, Search & Storage
- [ ] **Real-Time Protocols**: WebSockets, Server-Sent Events (SSE), Long Polling, WebRTC
- [ ] **Connection Scaling**: Linux file descriptors (`ulimit -n`), socket buffers, epoll non-blocking I/O
- [ ] **Object Storage**: Amazon S3, Presigned URLs, Multipart uploads, Resumable uploads (TUS protocol)
- [ ] **Search Systems**: Elasticsearch / OpenSearch, Inverted index, Lucene segment flushing, CDC indexing

### 7. Reliability, Observability & Cloud Operations
- [ ] **Observability**: Metrics (Prometheus), Distributed Tracing (OpenTelemetry/Jaeger), Centralized Logging (ELK)
- [ ] **Disaster Recovery**: RPO (Recovery Point Objective), RTO (Recovery Time Objective), Multi-Region Active-Active
- [ ] **Zero-Downtime Deployment**: Blue-Green, Canary deployments, Rolling updates, Database schema migrations (Expand/Contract)
- [ ] **Security & Auth**: OAuth2, OpenID Connect, JWT signing & revocation, mTLS between microservices
- [ ] **Cost Optimization**: Hot-Warm-Cold storage tiers, S3 Glacier, cloud egress mitigation, log sampling
- [ ] **AI & GenAI Serving**: LLM Gateway, Semantic caching, Token-based rate limiting, SSE streaming tokens
