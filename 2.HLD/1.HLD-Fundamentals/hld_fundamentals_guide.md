# 🏛️ Module 1: High-Level Design (HLD) Fundamentals & System Architecture

> **Core Philosophy:** *High-Level Design is the engineering discipline of decomposing complex business requirements into modular, scalable, fault-tolerant distributed components, establishing bounded contexts, and eliminating single points of failure under real-world physical constraints.*

---

## 📌 Table of Contents

1. [The Problem: What Happens When Scale Breaks?](#1-the-problem-what-happens-when-scale-breaks)
2. [HLD vs LLD: The Architectural Spectrum](#2-hld-vs-lld-the-architectural-spectrum)
3. [The Complete Request Lifecycle (From Click to Database &amp; Back)](#3-the-complete-request-lifecycle-from-click-to-database--back)
4. [Client-Server Evolution: Peer-to-Peer, Client-Server &amp; Microservices](#4-client-server-evolution-peer-to-peer-client-server--microservices)
5. [Monolith vs Microservices: Deep Architectural Decomposition](#5-monolith-vs-microservices-deep-architectural-decomposition)
6. [Stateless vs Stateful Services: Surviving Machine Failures](#6-stateless-vs-stateful-services-surviving-machine-failures)
7. [Synchronous vs Asynchronous Communication Models](#7-synchronous-vs-asynchronous-communication-models)
8. [Requirements Engineering: Functional vs Non-Functional (NFRs)](#8-requirements-engineering-functional-vs-non-functional-nfrs)
9. [Capacity Estimation &amp; Back-of-the-Envelope Calculations](#9-capacity-estimation--back-of-the-envelope-calculations)
10. [End-to-End Java / Spring Boot Architecture Implementation](#10-end-to-end-java--spring-boot-architecture-implementation)
11. [Side-by-Side Trade-off Comparison Table](#11-side-by-side-trade-off-comparison-table)
12. [Interview Framework &amp; Senior Architect Mental Model](#12-interview-framework--senior-architect-mental-model)

---

## 1. The Problem: What Happens When Scale Breaks?

### Real-World Domain Example: The Flash-Sale Meltdown 💥

Imagine an e-commerce platform built as a simple single-server application. At 12:00:00 PM, a new flagship product launches. 500,000 active shoppers hit `POST /orders` within 10 seconds.

```
                  500,000 CONCURRENT USERS
                             │
                             ▼
              [ Single Monolithic Server (Tomcat) ]
                             │
        ❌ Thread Pool Exhaustion (max 200 worker threads)
        ❌ CPU spikes to 100% due to GC pauses & lock contention
        ❌ Database connection pool exhausted (max 100 connections)
        ❌ Entire application crashes with HTTP 504 Gateway Timeout!
```

### The Root Failures of Naive Systems:

1. **Vertical Ceilings:** A single physical box has fixed RAM, CPU sockets, and network interface card (NIC) throughput.
2. **Coupled Failures:** A memory leak in the PDF invoice generator crashes the mission-critical checkout process.
3. **Stateful Traps:** Storing user session data in server RAM means dying servers log out 100,000 active buyers.

---

## 2. HLD vs LLD: The Architectural Spectrum

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                         SYSTEM DESIGN SPECTRUM                                 │
├──────────────────────────────────────┬─────────────────────────────────────────┤
│ High-Level Design (HLD)              │ Low-Level Design (LLD)                  │
├──────────────────────────────────────┼─────────────────────────────────────────┤
│ • Macro view of the entire system    │ • Micro view of components & classes    │
│ • Component boundaries & services    │ • Class diagrams, inheritance, patterns │
│ • Network protocols (HTTP, gRPC, WS) │ • Method signatures, design patterns    │
│ • Data stores (SQL vs NoSQL vs Cache)│ • Thread safety, locks, memory leaks    │
│ • Sharding, replication & partitions │ • SOLID principles, clean code, refactor│
│ • Scalability, availability, latency │ • Time & space complexity: O(1), O(N)   │
└──────────────────────────────────────┴─────────────────────────────────────────┘
```

---

## 3. The Complete Request Lifecycle (From Click to Database & Back)

Every distributed web interaction follows this comprehensive sequence of network hops:

```
[ Browser / Mobile Client ]
       │ 1. Types https://store.example.com/checkout
       ▼
[ DNS Resolution ] ── Queries Browser Cache ──▶ OS Cache ──▶ Local DNS Resolver ──▶ Authoritative DNS (Route53)
       │ Returns Anycast IP (e.g., 104.16.24.1)
       ▼
[ Anycast CDN Edge (Cloudflare / CloudFront) ]
       │ TLS 1.3 Termination, WAF inspection (DDoS protection), Static Assets served
       ▼
[ External Layer 7 Load Balancer (AWS ALB / Nginx) ]
       │ HTTP/2 multiplexing, Path-based routing (/api/v1/checkout ──▶ Checkout Service)
       ▼
[ API Gateway (Spring Cloud Gateway) ]
       │ JWT Authentication, Token Bucket Rate Limiting, Distributed TraceId injection
       ▼
[ Internal Microservice Fleet (Spring Boot / Netty) ]
       │ Thread receives request from non-blocking EventLoop
       ├──▶ 1. Check Redis Distributed Cache (Cache-Aside pattern)
       │         └── Cache HIT: Return immediately in < 2ms!
       │         └── Cache MISS: Proceed to database...
       ├──▶ 2. Query Read-Replica / Primary Master (PostgreSQL / MySQL)
       │         └── Row-level lock / Optimistic lock with version column
       └──▶ 3. Publish asynchronous event to Kafka Topic ("order-created")
                 └── Decouples Billing, Notification, and Inventory services
```

---

## 4. Client-Server Evolution: Peer-to-Peer, Client-Server & Microservices

- **Peer-to-Peer (P2P):** Nodes act as both clients and servers (BitTorrent, Blockchain). High resilience, high coordination complexity.
- **Classic Client-Server:** Centralized server fleet with dumb or thin clients. Simple to secure and update, but servers must scale horizontally.
- **Modern Microservices:** Clients communicate through an intelligent API Gateway into a constellation of decentralized domain-driven services.

---

## 5. Monolith vs Microservices: Deep Architectural Decomposition

### Monolithic Architecture

A single deployable unit where all domain modules (Orders, Catalog, Auth, Billing) execute within the same process and communicate via in-memory method calls.

```
┌─────────────────────────────────────────────────────────────┐
│                 MONOLITHIC RUNTIME PROCESS                  │
│                                                             │
│   ┌──────────────┐   ┌──────────────┐   ┌───────────────┐   │
│   │ OrderModule  │──▶│ BillingModule│──▶│ AuthModule    │   │
│   └──────────────┘   └──────────────┘   └───────────────┘   │
│          │                  │                   │           │
└──────────┼──────────────────┼───────────────────┼───────────┘
           ▼                  ▼                   ▼
                [ SHARED MONOLITH DATABASE ]
```

* **Advantages:** Zero network serialization overhead, single transaction manager (ACID), simple deployment.
* **Disadvantages:** Single point of failure, team code conflicts, scaling requires duplicating the entire application.

### Microservices Architecture

Independent autonomous services bounded by domain boundaries, each with its own isolated database. Communication happens exclusively over network APIs (REST/gRPC/Kafka).

```
[ API Gateway ]
       ├──▶ [ Order Service ]      ──▶ (Orders DB - PostgreSQL)
       ├──▶ [ Billing Service ]    ──▶ (Billing DB - MySQL)
       └──▶ [ Inventory Service ]  ──▶ (Inventory DB - Redis / Mongo)
```

* **Advantages:** Independent CI/CD, technology heterogeneity (Java for transactions, Python for ML), fine-grained autoscaling.
* **Disadvantages:** Network latency, distributed transactions (Sagas required), complex distributed observability.

---

## 6. Stateless vs Stateful Services: Surviving Machine Failures

```
STATEFUL (Anti-Pattern at Web Tier):
[ Client ] ── Request 1 ──▶ [ App Server 1 ] (Stores session: {"user": 42} in local JVM Heap)
[ Client ] ── Request 2 ──▶ [ App Server 2 ] ──❌ 401 Unauthorized! (Server 2 does not have the session)

STATELESS (Cloud-Native Standard):
[ Client ] ── Request 1 ──▶ [ App Server 1 ] ── Writes session token ──▶ [ Redis Cluster ]
[ Client ] ── Request 2 ──▶ [ App Server 2 ] ── Reads session token  ──▶ [ Redis Cluster ]
* Any server can crash or be autoscaled down without losing user sessions!
```

---

## 7. Synchronous vs Asynchronous Communication Models

```
SYNCHRONOUS (Blocking RPC):
Client ──────── Request ────────▶ Service A ──────── Request ────────▶ Service B
Client ◀─────── Response ◀────── Service A ◀─────── Response ◀────── Service B
* Latency adds up: Total = Latency(A) + Latency(B). If Service B fails, Service A fails!

ASYNCHRONOUS (Event-Driven via Log Broker):
Producer ───── Publish Event ─────▶ [ Kafka Topic ] ──▶ (Returns 202 Accepted in 5ms)
                                           │
                        ┌──────────────────┴──────────────────┐
                        ▼                                     ▼
             [ Consumer 1: Billing ]               [ Consumer 2: Email ]
* Producer never waits for consumers. Extreme fault isolation & traffic spike buffering!
```

---

## 8. Requirements Engineering: Functional vs Non-Functional (NFRs)

Before drawing a single box in an interview, explicitly segregate requirements:

1. **Functional Requirements (FR):**
   - User can place an order for items in their shopping cart.
   - User can view their past 90 days of order history.
   - User receives real-time delivery tracking status updates.
2. **Non-Functional Requirements (NFR):**
   - **High Availability:** 99.99% uptime (<52.6 minutes downtime per year).
   - **Low Latency SLA:** p99 read latency < 50ms, p99 write latency < 200ms.
   - **Consistency Model:** Strong consistency for financial deductions; eventual consistency for email notifications.
   - **Scalability:** Must absorb a 5x surge in traffic during promotional flash sales.

---

## 9. Capacity Estimation & Back-of-the-Envelope Calculations

### Master Formula Sheet

```
1. Requests Per Second (QPS):
   QPS = (Daily Active Users × Actions per User per Day) / 86,400 seconds
   Peak QPS = Average QPS × 2.5 to 3.0

2. Storage Calculations:
   Daily Storage = Writes per Day × Average Payload Size (Bytes)
   3-Year Storage = Daily Storage × 365 × 3 × Replication Factor (usually 3x)

3. Network Bandwidth:
   Incoming Bandwidth = QPS (writes) × Average Payload Size
   Outgoing Bandwidth = QPS (reads) × Average Response Size
```

### Real Example: Designing Twitter / X Tweet Ingestion

- **Assumptions:** 200 Million Daily Active Users (DAU).
- Each user posts 2 tweets/day -> 400,000,000 tweets/day.
- **Write QPS:** 400,000,000 / 86,400 ≈ 4,630 writes/sec. Peak QPS ≈ 4,630 × 2.5 ≈ 11,500 QPS.
- **Storage per Tweet:**
  - `tweet_id`: 8 Bytes (64-bit int)
  - `user_id`: 8 Bytes
  - `text`: 140 chars × 2 Bytes (UTF-8) = 280 Bytes
  - `metadata` (created_at, geo, client): 64 Bytes
  - Total per record ≈ 360 Bytes.
- **Daily Storage:** 400,000,000 × 360 Bytes ≈ 144 GB/day.
- **5-Year Storage with 3x Replication:** 144 GB × 365 × 5 × 3 ≈ 788.4 TB.

---

## 10. End-to-End Java / Spring Boot Architecture Implementation

### A. Production Stateless Controller with Jakarta Validation

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
@Slf4j
public class OrderController {

    private final OrderApplicationService orderService;

    public record OrderItemDto(Long itemId, Integer quantity, BigDecimal unitPrice) {}

    public record CreateOrderRequest(
        @NotNull Long userId,
        @NotEmpty List<OrderItemDto> items,
        @NotBlank String idempotencyKey
    ) {}

    public record OrderResponse(
        String orderId,
        String status,
        BigDecimal totalAmount,
        Instant createdAt
    ) {}

    @PostMapping
    public ResponseEntity<OrderResponse> placeOrder(
            @Valid @RequestBody CreateOrderRequest request,
            @RequestHeader(value = "X-Trace-Id", required = false) String traceId) {

        log.info("Processing order for user: {}, traceId: {}", request.userId(), traceId);
        OrderResponse response = orderService.processOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}
```

### B. Service Tier with Distributed Caching & Idempotency

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderApplicationService {

    private final StringRedisTemplate redisTemplate;
    private final OrderRepository orderRepository;
    private final KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;

    @Transactional
    public OrderResponse processOrder(CreateOrderRequest req) {
        // 1. Enforce Idempotency via Redis SETNX (prevents double charging on client retry)
        String idempotencyKey = "idempotency:order:" + req.idempotencyKey();
        Boolean isFirstRequest = redisTemplate.opsForValue()
            .setIfAbsent(idempotencyKey, "PROCESSING", Duration.ofMinutes(10));

        if (Boolean.FALSE.equals(isFirstRequest)) {
            log.warn("Duplicate request detected for key: {}", req.idempotencyKey());
            throw new DuplicateRequestException("Order is already being processed");
        }

        // 2. Persist to Relational Database (Source of Truth)
        BigDecimal total = req.items().stream()
            .map(item -> item.unitPrice().multiply(BigDecimal.valueOf(item.quantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        OrderEntity order = new OrderEntity(req.userId(), total, OrderStatus.CREATED);
        orderRepository.save(order);

        // 3. Publish Domain Event to Kafka for Async Processing (Billing, Notifications)
        OrderPlacedEvent event = new OrderPlacedEvent(order.getId(), req.userId(), total);
        kafkaTemplate.send("order-events", order.getId().toString(), event);

        // 4. Update Idempotency status
        redisTemplate.opsForValue().set(idempotencyKey, "COMPLETED", Duration.ofDays(1));

        return new OrderResponse(order.getId().toString(), "CONFIRMED", total, Instant.now());
    }
}
```

---

## 11. Side-by-Side Trade-off Comparison Table

| Architecture Dimension         | Monolith Architecture                   | Microservices Architecture                       |
| :----------------------------- | :-------------------------------------- | :----------------------------------------------- |
| **Development Velocity** | Extremely fast in early phase           | Slower initially due to scaffolding & CI/CD      |
| **Operational Overhead** | Low (single deployment pipeline)        | High (requires K8s, service mesh, OpenTelemetry) |
| **Failure Blast Radius** | High (one memory leak kills entire app) | Low (isolated per container/pod)                 |
| **Data Consistency**     | Strong ACID transactions                | Eventual consistency (Sagas & Compensation)      |
| **Hardware Efficiency**  | Shared heap & connection pools          | Extra overhead per container/JVM                 |
| **Team Scalability**     | Hits bottlenecks past 20 engineers      | Supports 100+ engineers across domain teams      |

---

## 12. Interview Framework & Senior Architect Mental Model

```
SENIOR SYSTEM ARCHITECT CHECKLIST:
1. Clarify Context: Never assume scale. Always ask: "What is the expected DAU and read:write ratio?"
2. Scope Strictly: Focus on the top 3 critical functional flows. Defer nice-to-haves.
3. Draw Simple First: Client ──▶ Gateway ──▶ App ──▶ DB. Do not add Kafka and Redis on minute 1!
4. Justify Every Box: When adding Redis, explain WHY (e.g. "Because read traffic is 95% and DB read pool will saturate").
5. Embrace Failure: Discuss what happens when Redis dies, the network partitions, or disk fills up.
```
