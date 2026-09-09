# 🏛️ Module 01 — Level 1: Basic Production Scenarios (2–3 Years Experience)

This module trains you on core production engineering scenarios: isolating asymmetric traffic, surviving sudden traffic spikes, breaking hot endpoint bottlenecks, triaging slow databases, preventing cascading latency failures, guaranteeing payment idempotency, and recovering from mid-flight crashes.

Every scenario strictly follows the **12-Step FAANG/Startup Interview Blueprint**.

---

# HLD Scenario #01 — Payment Traffic Isolation

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 1 (2–3 Years Experience)
- **Topics**: Scalability • Bulkhead Pattern • Microservices • Availability • Payment Systems
- **Company Examples**: Amazon, Swiggy, Flipkart, Stripe, Shopify

---

### 1. Interviewer Question
> *"Suppose your app has 10 users browsing products, but suddenly 1,000 users start making payments during a flash sale. Design the system so that payment traffic never slows down, blocks, or crashes browsing traffic."*

---

### 2. What the Interviewer is Actually Testing
- **Bulkhead Pattern & Resource Isolation**: Preventing one saturated resource from exhausting shared threads, memory, or database sockets.
- **Compute & Thread Pool Decoupling**: Understanding why running two routes on the same app process causes thread starvation.
- **Database Connection Pool Exhaustion**: Realizing that HikariCP/PostgreSQL connection saturation brings down the entire database instance for everyone.
- **Synchronous vs Asynchronous Processing**: Decoupling synchronous HTTP requests from background queue workers.

---

### 3. First 30 Seconds Thinking Framework
> 💡 **Golden Interview Line**:  
> *"Before jumping to technologies, I want to identify the shared resources between browsing and checkout: the API Gateway thread pool, the backend compute containers, and the database connection pools. My goal is to enforce strict physical and logical bulkheads across all three tiers so checkout traffic can never starve browsing traffic."*

---

### 4. HLD Whiteboard Architecture Diagram

```
                 [ 10 Browsing Users ]            [ 1,000 Payment Users ]
                           │                                 │
                           ▼                                 ▼
                     ┌─────────────────────────────────────────────┐
                     │          API Gateway (Envoy / Kong)         │
                     │  - Route: /catalog      - Route: /checkout  │
                     │  - Thread Pool A (40%)  - Thread Pool B (60%)│
                     └──────────────┬──────────────────────────────┘
                                    │
           ┌────────────────────────┴────────────────────────┐
           │ (Route: /catalog)                               │ (Route: /checkout)
           ▼                                                 ▼
┌───────────────────────┐                         ┌───────────────────────┐
│    Catalog Service    │                         │    Payment Service    │
│  - Dedicated Pods     │                         │  - Autoscaling Pods   │
│  - HPA: CPU 70%       │                         │  - Rate Limiter (429) │
└──────────┬────────────┘                         └──────────┬────────────┘
           │                                                 │
           ▼                                                 ▼
┌───────────────────────┐                         ┌───────────────────────┐
│     Redis Cache       │                         │   Kafka / SQS Queue   │
│ (Catalog Reads - 99%) │                         │ (Buffers Write Surge) │
└──────────┬────────────┘                         └──────────┬────────────┘
           │ (Cache Miss)                                    │
           ▼                                                 ▼
┌───────────────────────┐                         ┌───────────────────────┐
│ Catalog Database (PG) │                         │ Payment Database (PG) │
│ (Read Replicas)       │                         │ (Isolated Connection  │
│                       │                         │  Pool & Instance)     │
└───────────────────────┘                         └───────────────────────┘
```

---

### 5. Component-by-Component Responsibilities

| Component | Responsibility & Configuration |
| :--- | :--- |
| **API Gateway** | Enforces the **Bulkhead Pattern**. Allocates distinct worker thread pools and connection quotas for `/catalog` vs `/checkout`. |
| **Catalog Service** | Stateless microservice running in dedicated Kubernetes pods. Reads primarily from Redis cache and database read replicas. |
| **Payment Service** | Dedicated autoscaling pod fleet. Validates orders, enforces idempotency, writes to payment queue, and calls external PSP (Stripe). |
| **Kafka / SQS Queue** | Absorbs write surges during payment checkout, decoupling HTTP request ingestion from database writes. |
| **Isolated Databases** | Catalog DB and Payment DB run on separate database instances or use strict, isolated connection pool caps (e.g., PgBouncer). |

---

### 6. End-to-End Request Flow (Step-by-Step)

#### Browsing Path:
1. User sends `GET /api/v1/products/123`.
2. API Gateway directs request to **Catalog Thread Pool**.
3. Catalog Service checks **Redis Cache** (sub-2ms response).
4. On cache miss, it queries **Catalog Read Replica**, warms the cache, and returns `200 OK`.

#### Payment Path:
1. User clicks "Pay" $\rightarrow$ `POST /api/v1/checkout` with unique `idempotency_key`.
2. API Gateway routes request to **Payment Thread Pool**. If pool is full, gateway returns `429 Too Many Requests` or queues at ingress without affecting catalog threads.
3. Payment Service writes checkout intent to **Kafka / Amazon SQS**.
4. Payment Worker pulls message from queue at a controlled rate, locks funds, calls payment gateway (Stripe), and updates Payment DB.
5. Client receives `202 Accepted` with an `order_id` and polls or listens via WebSocket/SSE for final confirmation.

---

### 7. Failure Mode Analysis Matrix

| Failure Point | Impact | Mitigation Strategy |
| :--- | :--- | :--- |
| **Payment Gateway (Stripe) Hangs** | Latency jumps from 200ms to 10s | Set **Socket Read Timeout to 1.5s** + **Circuit Breaker** (Envoy/Resilience4j) to fail fast. |
| **Payment Service CPU Hits 100%** | Checkout requests fail | Kubernetes **Horizontal Pod Autoscaler (HPA)** scales pods; Catalog pods remain 100% isolated. |
| **Payment DB Saturated** | Payment queries time out | Queue buffer (Kafka/SQS) acts as a **shock absorber**; workers pull only at the rate DB can write. |
| **Shared Monolith DB Exhaustion** | If DB must be shared | Enforce separate **HikariCP / PgBouncer connection pools** with strict upper limits per service. |

---

### 8. Deep-Dive Trade-Off Analysis
- **Synchronous vs Asynchronous Checkout**:
  - *Synchronous HTTP*: User knows immediately if payment succeeded, but holds open API Gateway threads and database connections during the 2–3s external bank handshake.
  - *Asynchronous Queuing (Recommended for Spikes)*: Returns `202 Accepted` immediately, freeing gateway threads. Client polls or receives a WebSocket update. High complexity, but maximum resilience.
- **Single Monolith DB vs Database-per-Service**:
  - *Single Monolith DB*: Simple queries and joins, but high risk of connection pool starvation where a payment surge takes down browsing queries.
  - *Database-per-Service*: Complete failure isolation and independent scalability, but requires event-driven data sync (CDC/Outbox) and cannot perform cross-service SQL joins.

---

### 9. Curveball Follow-Ups

#### 🌀 Curveball 1: *"What if this is an un-split legacy monolith and you cannot rewrite it into microservices before tomorrow?"*
> **Answer**: Configure the reverse proxy (Nginx or Envoy) with two separate upstream socket bindings pointing to two isolated monolith server process groups (`port 8001` for catalog, `port 8002` for checkout). In the application configuration, allocate separate database connection pools (HikariCP) for checkout vs catalog queries so checkout can never consume catalog connections.

#### 🌀 Curveball 2: *"What if the payment gateway provider (Stripe) goes completely offline?"*
> **Answer**: Trigger a Circuit Breaker immediately after 5 consecutive failures. Return a user-friendly message: *"Payment processing is temporarily degraded; your order has been saved and will be processed shortly"*. Buffer orders in a durable Dead Letter Queue (DLQ) with exponential backoff retries.

#### 🌀 Curveball 3: *"What if the traffic surges from 1,000 to 100,000 concurrent payment requests?"*
> **Answer**: Implement an **Edge Virtual Waiting Room** (e.g., Cloudflare Workers) that intercepts traffic before it reaches origin servers, admitting only 2,000 users per minute with cryptographically signed HMAC tokens while placing the rest in a fair queue.

---

### 10. Strong Interview-Style Answer (2–3 Minutes)
> *"To ensure that 1,000 concurrent payment transactions do not degrade browsing for catalog users, I implement physical and logical isolation across three distinct tiers:*
> 
> *First, at the **Ingress and API Gateway tier**, I apply the **Bulkhead Pattern**. The gateway allocates independent worker thread pools and rate limits for `/catalog` and `/checkout`. Even if all payment threads are saturated, the browsing thread pool remains 100% available with zero socket contention.*
> 
> *Second, at the **Compute tier**, Catalog and Payment run as independent microservices in isolated Kubernetes pod deployments with dedicated CPU/memory resource limits. During a flash sale, payment pods autoscale independently without stealing compute nodes from the catalog.*
> 
> *Third, at the **Data tier**, we separate databases or enforce isolated connection pools using PgBouncer. Catalog browsing queries are served from Redis and read replicas, meaning write locks on payment tables never block read queries.*
> 
> *Finally, to absorb the payment surge, the payment service writes checkout intents asynchronously to a message queue like Kafka or SQS, acknowledging the client with `202 Accepted`. This prevents database connection exhaustion and isolates the entire browsing ecosystem from payment spikes."*

---

### 11. Real-World Production Battle-Tested Case Studies
- **Amazon**: During Prime Day, Amazon's ordering pipeline completely decouples order placement from financial settlement. When you click "Buy Now", an asynchronous message is dropped into an internal queue (SQS-like), and the user is immediately shown the confirmation screen. Payment charging and credit card authorization happen asynchronously in the background.
- **Shopify**: Uses strict Nginx and Puma worker thread bulkheading to ensure flash sale storefronts never take down admin dashboards or reporting tools.

---

### 12. 60-Second Last-Minute Revision Cheat Sheet
- **Bulkhead Pattern**: Separate thread pools at API Gateway.
- **Compute Isolation**: Separate microservices / K8s deployments.
- **Connection Isolation**: Separate DB instances or separate PgBouncer pools.
- **Asynchronous Ingestion**: Ingress $\rightarrow$ Queue (Kafka/SQS) $\rightarrow$ Worker $\rightarrow$ DB.
- **Fast Fail**: 1.5s read timeouts + Circuit Breaker on external payment providers.

---

# HLD Scenario #02 — Sudden 50x Traffic Spike (1k to 50k RPS)

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 1 (2–3 Years Experience)
- **Topics**: Scalability • Autoscaling • Rate Limiting • Edge Caching • Backpressure
- **Company Examples**: Twitter/X, Ticketmaster, Netflix, Cloudflare

---

### 1. Interviewer Question
> *"Your API normally receives 1,000 requests/second. Due to a celebrity tweet or flash sale, traffic suddenly surges to 50,000 requests/second within 30 seconds. What fails first, and how would you design the system to survive?"*

---

### 2. What the Interviewer is Actually Testing
- **Capacity Planning & Bottleneck Identification**: Knowing the order of failures (DNS $\rightarrow$ Load Balancer sockets $\rightarrow$ App thread pools $\rightarrow$ Database connections).
- **Autoscaling Latency vs Pre-Warming**: Understanding that Kubernetes HPA and cloud VM provisioning take 1 to 3 minutes, while the spike arrives in 30 seconds.
- **Edge Caching & Load Shedding**: Stopping traffic before it hits origin servers.
- **Asynchronous Ingestion vs Synchronous Bottlenecks**.

---

### 3. First 30 Seconds Thinking Framework
> 💡 **Golden Interview Line**:  
> *"Standard autoscaling will not save us here because VM and container initialization takes 45 to 120 seconds, while 50,000 RPS arrives in 30 seconds. To survive, our strategy must be layered: absorb 80% at the CDN edge, shed or rate-limit excess traffic at the gateway, and buffer all writes asynchronously through Kafka."*

---

### 4. HLD Whiteboard Architecture Diagram

```
[ 50,000 RPS Surge ]
        │
        ▼
[ Cloudflare / CloudFront CDN ] ──► (Cache Hit: Absorbs 80-90% read traffic)
        │ (Cache Miss)
        ▼
[ Edge Rate Limiter / WAF ] ──► (Drops unauthorized / abusive traffic with 429)
        │
        ▼
[ Layer 7 Load Balancer / Ingress ] (Health checks, round-robin / least conn)
        │
        ▼
[ API Gateway (Token Bucket) ] ──► (Queue excess or Load Shed with 503)
        │
   ┌────┴───────────────────────────┐
   ▼                                ▼
[ Read Path ]                 [ Write Path ]
   │                                │
[ Redis Cluster (Replicas) ]   [ Message Queue (Kafka/SQS) ]
   │ (Cache Miss)                   │
   ▼                                ▼
[ DB Read Replicas ]          [ Workers pulling at controlled rate ]
                                    │
                                    ▼
                              [ Primary DB (Protected) ]
```

---

### 5. Component-by-Component Responsibilities

| Component | Responsibility & Configuration |
| :--- | :--- |
| **CDN (Cloudflare/CloudFront)** | Caches static assets and semi-static API responses with `stale-while-revalidate`. Absorbs 80–90% of reads. |
| **WAF & Edge Rate Limiter** | Token-bucket rate limiting per IP/Client ID to stop scrapers and abusive bots with `HTTP 429`. |
| **API Gateway (Kong/Envoy)** | Enforces **Graceful Load Shedding**: drops non-critical requests (e.g., recommendations, analytics) when CPU exceeds 80%. |
| **Redis Cluster** | In-memory distributed cache handling read spikes with sub-millisecond response times. |
| **Kafka / Amazon SQS** | Queues write traffic so the database writes at a fixed, sustainable throughput without crashing. |

---

### 6. End-to-End Request Flow (Step-by-Step)
1. Traffic surges to 50k RPS. Incoming requests hit **Edge CDN**.
2. If the request is a read for public catalog data, CDN serves from cache. Origin never sees this traffic.
3. Cache misses reach the **API Gateway**. Gateway checks local token bucket:
   - Valid requests within limits pass through.
   - Excess traffic receives `HTTP 429 Too Many Requests` with a `Retry-After` header.
4. If backend CPU/thread pools exceed 85%, gateway activates **Load Shedding**, shedding low-priority endpoints.
5. Write requests are validated and pushed to a **Kafka partitioned topic**, returning `202 Accepted` to the client in <15ms.
6. A fleet of worker pods pulls messages from Kafka at a controlled pace (e.g., 2,500 writes/sec), batching updates into the Primary Database without exhausting connection pools.

---

### 7. Failure Mode Analysis Matrix

| Failure Point | What Fails First | Root Cause | Architectural Defense |
| :--- | :--- | :--- | :--- |
| **Database Pool** | App throws `ConnectionPoolTimeoutException` | 50k connections attempt to open concurrently | Add **PgBouncer** / ProxySQL; clamp maximum connections to 200; buffer writes via Kafka. |
| **App Compute Pods** | Pods run Out of Memory (OOMKilled) | Cold-start autoscaling lag takes 90s | Maintain a **20% pre-warmed standby capacity** buffer; shed load at gateway. |
| **Redis Cache** | Network bandwidth saturates on hot key | Single hot key hits one Redis node | Implement **L1 In-Process Memory Cache** (Caffeine) inside each app instance (2s TTL). |

---

### 8. Deep-Dive Trade-Off Analysis
- **Availability vs Consistency**:
  - Serving stale data via `stale-while-revalidate` from CDN/Redis guarantees 99.99% availability during spikes, but users might see inventory counts that are 5 seconds old.
- **Indiscriminate Dropping vs Fair Rate Limiting**:
  - Dropping all traffic above 20k RPS is computationally cheap.
  - Tracking per-user rate limits in distributed Redis requires network hops for every request, which can itself become a bottleneck during a 50k spike. *Solution*: Local token pre-allocation.

---

### 9. Curveball Follow-Ups

#### 🌀 Curveball 1: *"What happens during the first 60 seconds before Kubernetes autoscaling finishes spinning up new pods?"*
> **Answer**: During the 60-second cold-start window, the **API Gateway must engage Load Shedding and Request Queuing**. It prioritizes critical transactional endpoints (`/checkout`) and immediately drops or serves cached defaults for non-critical endpoints (`/recommendations`, `/search-suggestions`) with HTTP 503, preventing CPU saturation on existing pods.

#### 🌀 Curveball 2: *"What if Redis itself runs out of connections or memory under this sudden spike?"*
> **Answer**: Add an **L1 In-Memory Process Cache (e.g., Caffeine in Java, sync.Map in Go)** directly inside each service pod with a 3-second TTL. This eliminates 95% of queries to Redis, reducing Redis network load from 50k RPS to under 2.5k RPS.

---

### 10. Strong Interview-Style Answer (2–3 Minutes)
> *"A 50x spike in 30 seconds will break unbuffered downstream systems because standard autoscaling takes 1 to 3 minutes to react. Here is how I protect the system tier by tier:*
> 
> *First, at the **Edge (CDN)**: I absorb 80% to 90% of read traffic by configuring aggressive edge caching with `Cache-Control: public, max-age=5, stale-while-revalidate=30`. Public item details never reach origin servers.*
> 
> *Second, at the **API Gateway**: I enforce **Token Bucket Rate Limiting** per IP and user session. If backend resource utilization exceeds 85%, the gateway engages **graceful load shedding**, rejecting non-critical endpoints while reserving compute for checkout flows.*
> 
> *Third, for the **Write Path**: I decouple HTTP ingestion from database commits. The API Gateway writes validated requests into a partitioned **Kafka message queue** and returns `202 Accepted`. Background consumer workers pull from Kafka at a strictly controlled rate that the primary database can safely sustain.*
> 
> *Finally, to mitigate the autoscaling delay, we configure Kubernetes HPA with aggressive step-scaling policies and maintain a 20% pre-warmed over-provisioned buffer of standby pods."*

---

### 11. Real-World Production Battle-Tested Case Studies
- **Twitter/X**: During major sporting events (e.g., World Cup goals), tweet volume spikes by 50x in seconds. Twitter buffers incoming tweets in high-throughput Kafka clusters, returning an immediate acknowledgment to the user while downstream timeline indexing workers process the queue asynchronously.
- **Cloudflare**: Handles multi-terabit DDoS attacks and legitimate flash sales by executing rate-limiting logic directly at edge Anycast points-of-presence using eBPF and XDP (eXpress Data Path) kernel filters.

---

### 12. 60-Second Last-Minute Revision Cheat Sheet
- **Autoscaling is too slow**: Cold start takes 45–120s $\Rightarrow$ Cannot rely on HPA alone.
- **Absorb at Edge**: 80% absorbed by CDN `stale-while-revalidate`.
- **Load Shedding**: Drop non-critical APIs at gateway when CPU > 80%.
- **Buffer Writes**: Ingress $\rightarrow$ Kafka $\rightarrow$ Controlled Workers $\rightarrow$ DB.
- **Protect Redis**: L1 In-Memory Cache (Caffeine/Local RAM) ahead of Redis L2.

---

# HLD Scenario #03 — Hot Endpoint Bottleneck (The 90% Traffic Hotspot)

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 1 (2–3 Years Experience)
- **Topics**: Caching • Redis • L1/L2 Caching • Microservices • Route Splitting
- **Company Examples**: Target, Walmart, Steam, Apple

---

### 1. Interviewer Question
> *"Out of 50 different API endpoints in your system, one single endpoint (`GET /api/v1/products/{id}`) receives 90% of the entire system's traffic. How do you prevent it from starving other endpoints and crashing your backend?"*

---

### 2. What the Interviewer is Actually Testing
- **Multi-Level Caching (L1 Process Cache + L2 Distributed Cache + CDN)**
- **Route-Based Isolation & Independent Service Extraction**
- **Hot Key Mitigation & Cache Stampede Prevention**
- **Read/Write Splitting & Replica Routing**

---

### 3. First 30 Seconds Thinking Framework
> 💡 **Golden Interview Line**:  
> *"When 90% of traffic hits a single endpoint, we face two distinct problems: compute starvation of other endpoints, and hot-key saturation at the cache and database layers. I will isolate this endpoint onto dedicated compute infrastructure and implement a multi-tiered caching architecture (CDN $\rightarrow$ L1 Local Cache $\rightarrow$ L2 Redis)."*

---

### 4. HLD Whiteboard Architecture Diagram

```
Client Requests
      │
      ▼
[ CDN Edge Cache (Cloudflare) ] ── (Hit for public static/semi-static data)
      │
      ▼
[ API Gateway Route Splitter ]
      ├── (90% Traffic) ──► Route: /products/* ──► [ Dedicated Pod Fleet ]
      │                                                     │
      │                                            [ L1: Local In-Memory Cache ]
      │                                                     │ (Miss)
      │                                            [ L2: Redis Cluster ]
      │                                                     │ (Miss)
      │                                            [ Read Replicas (Aurora/PG) ]
      │
      └── (10% Traffic) ──► Route: All Others  ──► [ General Pod Fleet ]
```

---

### 5. Component-by-Component Responsibilities

| Component | Responsibility & Configuration |
| :--- | :--- |
| **API Gateway Router** | Route-based traffic splitting. Routes `/products/*` exclusively to dedicated pods. |
| **Dedicated Pod Fleet** | Autoscaling compute cluster dedicated strictly to serving product catalog requests. |
| **L1 In-Memory Cache** | In-process cache (Caffeine/Go sync.Map) with 2–5s TTL. Serves 95% of reads with 0 network hops. |
| **L2 Redis Cluster** | Shared distributed cache across all pods with 10-minute TTL. |
| **Singleflight Mutex** | Request coalescing mechanism: ensures only 1 thread queries the DB on cache misses. |

---

### 6. End-to-End Request Flow (Step-by-Step)
1. Request arrives for `GET /api/v1/products/42`.
2. CDN Edge checks its cache. If cached within 5 seconds, returns immediately.
3. On CDN miss, request arrives at API Gateway and routes to the **Dedicated Product Pod Fleet**.
4. Pod checks its **L1 Local Memory Cache** (response time $< 0.1\text{ms}$).
5. If L1 misses, it checks **L2 Redis Cluster** (response time $< 2\text{ms}$).
6. If L2 misses, the pod engages a **Singleflight Mutex**:
   - Only 1 worker thread queries the **Database Read Replica**.
   - All other 5,000 concurrent requests block and wait for that single query to return.
7. Database query returns, populates L2 Redis and L1 local cache, and unblocks all waiting requests.

---

### 7. Failure Mode Analysis Matrix

| Failure Point | Impact | Architectural Defense |
| :--- | :--- | :--- |
| **Redis Hot-Key Saturation** | Single Redis shard CPU hits 100% | Replicate hot key across shards with random suffix (`product:42#1`, `product:42#2`). |
| **Cache Stampede on Expiry** | 10,000 queries hit DB simultaneously | Use **Singleflight / Mutex locking** so only 1 query hits DB to rebuild cache. |
| **Database Read Replica Lag** | Stale product data served | Set short L1/L2 TTLs (3–5s) or push cache invalidation via Redis Pub/Sub. |

---

### 8. Deep-Dive Trade-Off Analysis
- **L1 In-Memory Cache vs L2 Redis**:
  - *L1 In-Memory*: Extremely fast ($<0.1\text{ms}$, no network serialization), but memory is duplicated across 100 pods and different pods may hold slightly different versions until TTL expires.
  - *L2 Redis*: Centralized consistency, but introduces network latency (1–3ms) and can become a network/CPU bottleneck under massive hot-key traffic.

---

### 9. Curveball Follow-Ups

#### 🌀 Curveball 1: *"What if the product price changes while millions of users are viewing it? How do you invalidate the cache across 100 app pods instantly?"*
> **Answer**: Use **Redis Pub/Sub** or a Kafka broadcast topic. When an admin updates the product price in PostgreSQL, an event is published to channel `cache:invalidate`. All 100 app pods subscribe to this channel and immediately evict `product:42` from their local L1 memory cache.

---

### 10. Strong Interview-Style Answer (2–3 Minutes)
> *"To prevent an endpoint receiving 90% of traffic from starving the system, I address this across compute isolation, multi-tier caching, and stampede protection:*
> 
> *First, **Compute Separation**: I split `/products/*` into a dedicated microservice with its own Kubernetes deployment. The API Gateway routes all catalog traffic strictly to this fleet, guaranteeing that the remaining 10% of APIs have dedicated CPU, memory, and thread allocations.*
> 
> *Second, **Multi-Tier Caching**:*
> 1. *At the edge, Cloudflare caches responses for 5 seconds using `stale-while-revalidate`.*
> 2. *Inside each application pod, an **L1 In-Process Cache** (e.g., Caffeine) caches hot products for 3 seconds. This satisfies 95% of queries directly from local RAM with zero network hops.*
> 3. *On L1 miss, we check an **L2 Redis Cluster**.*
> 
> *Finally, to protect the database from cold-start cache stampedes, we implement **Singleflight Request Coalescing**. If 10,000 requests miss the cache at the same millisecond, only a single thread queries the database read replica while the others wait for the result in memory, completely eliminating database crashes."*

---

### 11. Real-World Production Battle-Tested Case Studies
- **Steam / Valve**: During Steam Summer Sales, game store pages receive 95% of traffic. Steam uses multi-tiered Varnish reverse-proxy caches and in-process memory caching to absorb hundreds of thousands of requests per second for top-selling games.

---

### 12. 60-Second Last-Minute Revision Cheat Sheet
- **Compute Isolation**: Route splitting $\rightarrow$ Dedicated pod fleet for hot endpoint.
- **Multi-Level Cache**: Edge CDN $\rightarrow$ L1 Local RAM (Caffeine) $\rightarrow$ L2 Redis $\rightarrow$ DB.
- **Hot Key Solution**: Replicate key with random salt suffixes across Redis shards.
- **Cache Stampede Fix**: Singleflight pattern (1 query to DB, rest wait on promise).

---

# HLD Scenario #04 — Database Slowdown Under Growing User Base

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 1 (2–3 Years Experience)
- **Topics**: Database Scaling • Indexing • Slow Queries • Connection Pooling • Read Replicas
- **Company Examples**: Meta, LinkedIn, Airbnb, GitHub

---

### 1. Interviewer Question
> *"Your application has grown from 10,000 to 500,000 active users. Over the last month, API latency has doubled, and the database CPU is consistently at 85–95%. Walk me through your step-by-step investigation and remediation plan."*

---

### 2. What the Interviewer is Actually Testing
- **Production Troubleshooting Methodology**: Starting with metrics and telemetry rather than jumping to premature sharding.
- **Query Optimization & Indexing**: Diagnosing sequential table scans vs index scans.
- **Connection Pooling Limits**: Sizing connection pools (HikariCP, PgBouncer).
- **Read/Write Splitting vs Vertical Scaling vs Sharding**.

---

### 3. First 30 Seconds Thinking Framework
> 💡 **Golden Interview Line**:  
> *"I will approach this systematically: first identify whether the bottleneck is CPU, Disk I/O, or Lock Contention; second, analyze the slow query log to find high-impact queries; third, apply immediate low-hanging optimizations like indexes, connection pooling, and read replicas before considering architectural changes like sharding."*

---

### 4. Investigation & Remediation Architecture

```
[ Step 1: Metric Triangulation ]
- CPU % (Compute bound vs I/O bound)
- Memory Buffer Cache Hit Ratio (< 99% indicates disk thrashing)
- Active Connection Count & Lock Waits
               │
               ▼
[ Step 2: Query Telemetry ]
- Query pg_stat_statements / slow_query_log
- Identify queries with highest (calls * mean_exec_time)
               │
               ▼
[ Step 3: Query Plan Analysis ]
- Run EXPLAIN ANALYZE on top offenders
- Identify Seq Scans, Temporary File spills to disk, unindexed foreign keys
               │
               ▼
┌─────────────────────────────────────────────────────────────┐
│                 Step 4: Remediation Plan                    │
│                                                             │
│ Immediate (Hours):                                          │
│ - Add missing composite indexes                             │
│ - Deploy PgBouncer (Drop connection overhead)               │
│                                                             │
│ Medium-Term (Days):                                         │
│ - Route read-heavy queries to Read Replicas                 │
│ - Add Redis cache for repetitive SELECT queries             │
│                                                             │
│ Long-Term (Months):                                         │
│ - Table partitioning (e.g., date-based on audit logs)       │
│ - Sharding by user_id (Vitess / Citus)                      │
└─────────────────────────────────────────────────────────────┘
```

---

### 5. Likely Root Causes & Architectural Solutions

| Metric Symptom | Root Cause | Architectural Solution |
| :--- | :--- | :--- |
| **High CPU, Normal Disk I/O** | Missing index causing full table scan across 10M rows | Add composite B-Tree index matching `WHERE` / `ORDER BY` clauses. |
| **High Disk I/O, Low Buffer Hit** | Working dataset exceeds RAM; swapping to disk | Upgrade instance RAM (vertical scale) or cache hot rows in Redis. |
| **High Active Connections** | Backend opening hundreds of unpooled connections | Deploy **PgBouncer** connection pooler in transaction pooling mode. |
| **High Lock Wait Times** | Long-running write transactions blocking reads | Shorten transactions; use optimistic concurrency control (`OCC`). |

---

### 6. Strong Interview-Style Answer (2–3 Minutes)
> *"I would diagnose and resolve the database saturation in four systematic steps:
> 
> 1. **Triangulate the Bottleneck**: Check CloudWatch or Datadog for database CPU, Memory Buffer Cache Hit Ratio, Disk IOPS, and Connection Count. If buffer cache hit ratio is below 99%, the working dataset exceeds RAM and is thrashing the disk.
> 2. **Analyze Slow Query Telemetry**: Query `pg_stat_statements` (PostgreSQL) to rank queries by total execution time (`calls * mean_exec_time`). Run `EXPLAIN (ANALYZE, BUFFERS)` on the top three offenders to check for sequential scans on large tables.
> 3. **Immediate Remediation**:
>    - Add missing B-Tree composite indexes, ensuring column order matches equality filters first, then range filters.
>    - Place **PgBouncer** in front of PostgreSQL to reuse connections, capping backend DB connections at ~100 and eliminating connection memory overhead.
> 4. **Horizontal Offloading**:
>    - Spin up **Read Replicas** and update ORM/database routing to send all read-only `SELECT` queries to replicas, freeing the Primary DB strictly for writes.
>    - Introduce a Redis Cache-Aside layer for hot, repetitive queries."*

---

### 7. 60-Second Last-Minute Revision Cheat Sheet
- **Order of Action**: Metrics $\rightarrow$ Slow Query Log $\rightarrow$ `EXPLAIN ANALYZE` $\rightarrow$ Indexes $\rightarrow$ PgBouncer $\rightarrow$ Replicas $\rightarrow$ Sharding.
- **Never shard first**: Sharding is a last resort after indexing, pooling, caching, and read replicas have been exhausted.

---

# HLD Scenario #05 — Downstream Latency Degradation & Cascading Outages

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 1 (2–3 Years Experience)
- **Topics**: Reliability • Timeouts • Circuit Breakers • Bulkheads • Cascading Failures
- **Company Examples**: Netflix, Uber, Airbnb, Amazon

---

### 1. Interviewer Question
> *"Service A calls Service B synchronously. Service B starts taking 10 seconds to respond instead of its normal 100ms. Within 2 minutes, Service A and all other upstream services crash. Why did this happen, and how do you prevent it?"*

---

### 2. What the Interviewer is Actually Testing
- **Cascading Failures & Thread Pool Starvation**: How slow downstream sockets exhaust upstream HTTP worker threads and OS file descriptors.
- **Client-Side Timeout Discipline**: Setting aggressive connect and read timeouts.
- **Circuit Breaker Pattern**: States (Closed $\rightarrow$ Open $\rightarrow$ Half-Open).
- **Bulkheading & Graceful Fallbacks**.

---

### 3. First 30 Seconds Thinking Framework
> 💡 **Golden Interview Line**:  
> *"Service A did not crash because Service B failed; Service A crashed because Service B was SLOW. Slow responses are far more dangerous than immediate errors because they keep threads blocked. Incoming traffic accumulates, worker thread pools saturate, memory explodes, and Service A crashes."*

---

### 4. Circuit Breaker State Machine & Architecture

```
              ┌───────────────────────────────┐
              │            CLOSED             │
              │ (Normal: All requests pass)   │
              └───────────────┬───────────────┘
                              │
               Failure / Timeout rate > 50%
                              │
                              ▼
              ┌───────────────────────────────┐
              │             OPEN              │
              │ (Fast Fail: Return 503 /      │
              │  fallback immediately)        │
              └───────────────┬───────────────┘
                              │
                 Wait sleep window (e.g. 30s)
                              │
                              ▼
              ┌───────────────────────────────┐
              │          HALF-OPEN            │
              │ (Send 5% probe traffic)       │
              └───────┬───────────────┬───────┘
                      │               │
            Probes succeed       Probes fail
                      │               │
                      ▼               ▼
                   [CLOSED]        [OPEN]
```

---

### 5. Strong Interview-Style Answer (2–3 Minutes)
> *"This crash is a textbook **cascading failure caused by thread pool exhaustion**. When Service B's response time degraded from 100ms to 10 seconds, Service A's HTTP worker threads remained blocked waiting on socket reads. Because incoming traffic continued at normal volume, all worker threads in Service A's pool were consumed within seconds, causing Service A to reject all incoming requests and crash.
> 
> To prevent this, I implement a four-pillar resilience strategy:
> 
> 1. **Strict Timeouts**:
>    - **Connect Timeout**: Set to 200–500ms.
>    - **Read Timeout**: Set to 1.5x of P99 latency (e.g., 300ms instead of 10s). A fast failure is always better than a slow hang.
> 2. **Circuit Breaker (Envoy / Resilience4j)**: If timeouts or 5xx errors exceed a 50% threshold over a 10-second rolling window, the breaker trips to **OPEN**. Service A immediately stops making network calls to Service B and fails fast.
> 3. **Bulkheading**: Allocate a dedicated, bounded connection pool strictly for Service B so that issues with Service B never starve threads dedicated to Service C or Service D.
> 4. **Graceful Degradation / Fallback**: Return cached stale data, default recommendations, or queue the action asynchronously instead of throwing 500 errors to the user."*

---

### 6. 60-Second Last-Minute Revision Cheat Sheet
- **Root Cause**: Thread pool exhaustion from un-timed socket reads.
- **Defense 1**: Strict timeouts (Connect: 200ms, Read: 300ms).
- **Defense 2**: Circuit Breaker (Closed $\rightarrow$ Open $\rightarrow$ Half-Open).
- **Defense 3**: Bulkhead connection pool per downstream dependency.
- **Defense 4**: Return cached fallback or default response.

---

# HLD Scenario #06 — Payment Double-Click & Idempotency Guarantee

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 1 (2–3 Years Experience)
- **Topics**: Idempotency • Transactions • Distributed Systems • Payment Systems • Race Conditions
- **Company Examples**: Stripe, PayPal, Uber, Razorpay

---

### 1. Interviewer Question
> *"A user on a slow mobile connection clicks 'Pay $500'. The spinner hangs, so they tap 'Pay' three more times. How do you guarantee the user is charged exactly once, even if requests arrive concurrently across different servers?"*

---

### 2. What the Interviewer is Actually Testing
- **API Idempotency**: Ensuring the same operation executed multiple times produces the exact same result.
- **Idempotency Key Generation & Scope**: Client-generated vs server-generated UUIDs.
- **Database Unique Constraints vs Redis Locks**: Understanding why Redis alone is insufficient for financial state.
- **Pass-through Idempotency to Payment Service Providers (Stripe/Adyen)**.

---

### 3. Architecture & Idempotency Flow

```
[ Client / Mobile App ]
      │
      │ 1. Generate unique Idempotency-Key (UUIDv4) on checkout screen load
      │ 2. POST /api/v1/payments { key: "uuid-123", amount: 500 }
      ▼
[ API Gateway / Payment Service ]
      │
      │ 3. Atomic INSERT INTO idempotency_keys (key, status, response)
      │    VALUES ('uuid-123', 'PROCESSING', NULL)
      ├───► [ If duplicate key error / 409 Conflict ]:
      │         Poll or return "Transaction in progress"
      ▼
[ Call External PSP (Stripe / Adyen) with Idempotency Key ]
      │
      │ 4. PSP processes and returns charge_id: "ch_999"
      ▼
[ Update DB in Transaction ]
      │ UPDATE idempotency_keys 
      │ SET status = 'COMPLETED', response = '{"status":"SUCCESS"}', charge_id = 'ch_999'
      │ WHERE key = 'uuid-123';
      ▼
[ Return 200 OK to Client ]
```

---

### 4. Strong Interview-Style Answer (2–3 Minutes)
> *"To ensure strict exactly-once payment execution, I implement client-generated idempotency keys backed by database-level uniqueness guarantees:
> 
> 1. **Client Token Generation**: When the user opens the checkout modal, the client generates a unique `Idempotency-Key: UUIDv4` tied to that specific order intent. Even if the user taps 'Pay' four times, all four HTTP requests carry the exact same idempotency key header.
> 2. **Atomic State Check**: The payment service initiates a database transaction inserting the key into an `idempotency_records` table with status `PROCESSING`. If a concurrent request arrives with the same key, it fails with a unique constraint violation (`409 Conflict`) and is rejected.
> 3. **Pass-Through to Payment Gateway**: We forward the same idempotency key to Stripe/Adyen. Payment aggregators natively support idempotency keys to prevent duplicate bank charges at their level.
> 4. **Store Result**: Once Stripe responds, the DB record is updated to `COMPLETED` along with the response payload. Any future retries with this key immediately return the stored successful response without hitting payment rails."*

---

### 5. 60-Second Last-Minute Revision Cheat Sheet
- **Client generates UUIDv4**: Tied to the specific checkout session.
- **ACID DB Unique Constraint**: `INSERT INTO idempotency_keys` fails on duplicates.
- **Forward Key to Stripe**: Stripe prevents duplicate card authorizations.
- **Cache Final Response**: Subsequent duplicate requests return the cached original response.

---

# HLD Scenario #07 — Mid-Flight Service Crash During Distributed Multi-Step Action

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 1 (2–3 Years Experience)
- **Topics**: Distributed Transactions • Dual-Write Problem • Outbox Pattern • CDC • Kafka
- **Company Examples**: Airbnb, Uber, DoorDash, Shopify

---

### 1. Interviewer Question
> *"A user places an order. Your Order Service successfully creates the order in the database, but crashes halfway through right before publishing the 'OrderPlaced' event to Kafka. Inventory was reserved, but payment and notifications were never triggered. How do you design for this?"*

---

### 2. What the Interviewer is Actually Testing
- **The Dual-Write Problem**: Why writing to a database and publishing to a message queue in the same HTTP handler is fundamentally unreliable.
- **Transactional Outbox Pattern**: Using local ACID transactions to guarantee message publishing.
- **Change Data Capture (CDC / Debezium)**: Reading database Write-Ahead Logs (WAL).
- **At-Least-Once Delivery & Downstream Consumer Idempotency**.

---

### 3. Architecture: Transactional Outbox Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                      Order Service                          │
│                                                             │
│  BEGIN TRANSACTION;                                         │
│    1. INSERT INTO orders (...) VALUES (...);                │
│    2. INSERT INTO outbox_table (event_type, payload, status)│
│       VALUES ('OrderPlaced', '{order_id: 101}', 'PENDING'); │
│  COMMIT; ◄─── (Atomic: Both succeed or both rollback)       │
└──────────────┬───────────────────────────────┬──────────────┘
               │                               │
               ▼                               ▼
       [ Orders Table ]                [ Outbox Table ]
                                               │
                                               ▼
                             [ CDC Debezium / Poller Service ]
                             (Reads DB Write-Ahead-Log / WAL)
                                               │
                                               ▼
                                   [ Kafka: order-events ]
                                               │
                                               ▼
                                    [ Payment / Email Svc ]
```

---

### 4. Strong Interview-Style Answer (2–3 Minutes)
> *"This is the classic **Dual-Write Problem**: an application cannot write to a database and publish to a message queue in a single atomic transaction without distributed transactions. If the server crashes between step 1 and step 2, state becomes permanently inconsistent.
> 
> To solve this reliably, I use the **Transactional Outbox Pattern**:
> 
> 1. **Atomic Local Commit**: In the exact same database transaction that creates the order, we insert an event record into an `outbox` table. Since both writes occur within the same local ACID transaction, it is mathematically impossible for the order to exist without the outbox record.
> 2. **Reliable Message Relay (CDC via Debezium)**: A separate Change Data Capture service (Debezium) monitors PostgreSQL's Write-Ahead Log (WAL) and streams new outbox records into Kafka with zero application-level polling overhead.
> 3. **At-Least-Once Delivery & Deduplication**: If the publisher crashes after sending to Kafka but before committing its offset, it may republish the message upon recovery. Downstream consumers (Payment, Inventory) maintain consumer idempotency tables to ignore duplicate event IDs."*

---

### 5. 60-Second Last-Minute Revision Cheat Sheet
- **Dual-Write Anti-Pattern**: Never call `db.commit()` and `kafka.send()` sequentially.
- **Transactional Outbox**: Save event in an `outbox` table in the same local DB transaction.
- **CDC / Debezium**: Read DB Write-Ahead Log (WAL) to publish to Kafka.
- **At-Least-Once**: Downstream consumers must be idempotent.

---

# HLD Scenario #08 — Cache Invalidation & The "Stale Read" Race Condition

### Difficulty • Topic • Company Examples
- **Difficulty**: Level 1 (2–3 Years Experience)
- **Topics**: Caching • Redis • Cache-Aside • Race Conditions • Replication Lag
- **Company Examples**: Meta, Twitter, Amazon, Reddit

---

### 1. Interviewer Question
> *"You use a standard Cache-Aside pattern with Redis and MySQL. An admin updates a product price from $100 to $80. Users in another region are still seeing $100 for minutes. In some cases, a user sees $80, refreshes, and sees $100 again. Why does this happen and how do you resolve it?"*

---

### 2. What the Interviewer is Actually Testing
- **Cache-Aside Race Conditions (Read-Repair Race)**
- **Cache Eviction vs Cache Update**: Why updating a cache directly causes race conditions.
- **Read-After-Write Consistency across DB Replicas**: Replication lag poisoning the cache.

---

### 3. The Classic Race Condition

```
Thread 1 (Read Request)           Thread 2 (Write Request - Admin)
───────────────────────           ────────────────────────────────
1. Cache Miss for Product 101
2. Reads DB (Price = $100)
                                  3. Updates DB (Price = $80)
                                  4. Evicts Redis Cache for 101
5. Writes $100 to Redis Cache!
   (Redis now holds STALE $100 indefinitely!)
```

---

### 4. Solutions & Best Practices
1. **Cache Eviction Over Update**: Always delete (`DEL`) the cache key instead of updating it, forcing the next read to fetch fresh data.
2. **Delayed Double Deletion**:
   - Delete cache key.
   - Update DB.
   - Sleep 500ms (to let any ongoing read queries finish).
   - Delete cache key again.
3. **Replication Lag Awareness**: If the reader reads from a *Read Replica* that is 2 seconds behind the Primary DB, it will fetch stale data and populate the cache with it. Reads after writes should either query the Primary DB or wait for replication sync.
4. **Short TTLs as Safety Net**: Always attach an expiration TTL (e.g., 60 seconds) so stale states auto-heal.

---

### 5. Strong Interview-Style Answer (2–3 Minutes)
> *"This issue is caused by a race condition between concurrent reads and writes, compounded by database replication lag:
> 
> When Thread 1 experiences a cache miss, it reads the old price ($100) from a Read Replica that is lagging behind the Primary. Before Thread 1 can write to the cache, Thread 2 updates the Primary to $80 and invalidates Redis. Thread 1 then writes its stale $100 into Redis, overwriting the invalidation!
> 
> To resolve this:
> 1. **Cache Invalidation via CDC**: We don't invalidate caches in application code. Instead, Debezium streams committed Primary WAL events to an invalidation worker that evicts the Redis key *only after* the write is durably committed.
> 2. **Delayed Double Deletion**: If doing app-level eviction, delete the key, update the DB, wait 500ms, and delete the key again.
> 3. **Short Safety TTL**: Always attach a 60-second TTL to cached entities so any race condition auto-heals rapidly."*

---

### 6. 60-Second Last-Minute Revision Cheat Sheet
- **Root Cause**: Stale read from lagging replica overwrites cache eviction.
- **Rule 1**: Always delete cache keys (`DEL`), never update them in-place.
- **Rule 2**: Delayed Double Deletion (delete $\rightarrow$ write DB $\rightarrow$ wait $\rightarrow$ delete).
- **Rule 3**: Set short TTLs (60s) on all cache keys as an auto-healing safety net.
