# 🧩 Module 8: Microservices Architecture, Gateways & Distributed Resilience

> **Core Philosophy:** *A distributed system is guaranteed to experience partial failure. In a microservices landscape with dozens of interconnected services, resilience patterns like Circuit Breakers, Bulkheads, and Fallbacks prevent a single failing service from dragging down the entire enterprise.*

---

## 📌 Table of Contents
1. [The Problem: Cascading Distributed Failures](#1-the-problem-cascading-distributed-failures)
2. [Microservices Decomposition: Domain-Driven Design (Bounded Contexts)](#2-microservices-decomposition-domain-driven-design-bounded-contexts)
3. [API Gateway Pattern: Responsibilities & Routing Engine](#3-api-gateway-pattern-responsibilities--routing-engine)
4. [Service Discovery & Health Check Topologies](#4-service-discovery--health-check-topologies)
5. [The Circuit Breaker Pattern (Resilience4j State Machine)](#5-the-circuit-breaker-pattern-resilience4j-state-machine)
6. [The Bulkhead Pattern: Thread Pool & Semaphore Isolation](#6-the-bulkhead-pattern-thread-pool--semaphore-isolation)
7. [Retry with Exponential Backoff & Jitter: Avoiding Thundering Herds](#7-retry-with-exponential-backoff--jitter-avoiding-thundering-herds)
8. [Java / Spring Boot Resilience4j Production Implementation](#8-java--spring-boot-resilience4j-production-implementation)
9. [Side-by-Side Trade-off Analysis Table](#9-side-by-side-trade-off-analysis-table)
10. [Interview Rapid Q&A Checklist](#10-interview-rapid-qa-checklist)

---

## 1. The Problem: Cascading Distributed Failures

```
THE CASCADING OUTAGE SCENARIO:
[ 10,000 Users ] ──▶ [ Order Service (200 worker threads) ]
                               │
                               ▼ Synchronous HTTP call
                     [ Fraud Detection Service (Hangs under 100% CPU) ]

1. Fraud Service stops responding in 50ms and now takes 60 seconds per request.
2. Order Service worker threads block waiting on socket read timeouts.
3. Within 10 seconds, all 200 Tomcat worker threads in Order Service are parked!
4. Order Service can no longer accept ANY requests (even simple reads).
5. Load Balancer health check fails -> Order Service crashes globally!
```

---

## 2. Microservices Decomposition: Domain-Driven Design (DDD)

- **Single Responsibility Principle applied to services:** Each service models one cohesive business domain (Bounded Context).
- **Database-per-Service Rule:** A microservice exclusively owns its database tables. Direct cross-database SQL queries are strictly forbidden; interaction occurs exclusively through published APIs or Kafka domain events.

---

## 3. API Gateway Pattern: Responsibilities & Routing Engine

```
[ Web / iOS / Android Clients ]
                │ (HTTPS / TLS 1.3)
                ▼
┌────────────────────────────────────────────────────────┐
│             API GATEWAY (Spring Cloud Gateway)         │
│  ├── 1. Request Validation & SSL Termination           │
│  ├── 2. Centralized Authentication (JWT Validation)   │
│  ├── 3. Token-Bucket Rate Limiting (Redis-backed)      │
│  ├── 4. Dynamic Path-based Routing (Eureka / Consul)   │
│  └── 5. Distributed Tracing Injection (X-Trace-Id)     │
└────────────────────────────────────────────────────────┘
         │                   │                   │
         ▼                   ▼                   ▼
  [ Auth Service ]    [ Order Service ]   [ Payment Service ]
```

---

## 4. Service Discovery & Health Check Topologies

In elastic cloud environments (Kubernetes, AWS ECS), service instances auto-scale, crash, and change IP addresses constantly:
- **Client-Side Discovery (Spring Cloud Eureka):** Clients query the service registry to fetch a list of healthy IPs, then use internal client-side load balancing (Spring Cloud LoadBalancer).
- **Server-Side Discovery (Kubernetes Service / AWS ALB):** Client sends requests to a stable virtual IP or DNS name; the K8s kube-proxy or ALB routes to healthy pod IPs automatically.

---

## 5. The Circuit Breaker Pattern (Resilience4j State Machine)

```
        ┌─────────────────────────────────────────────────────────────┐
        │                                                             │
        ▼ (Failure rate > 50% in sliding window of 20 calls)          │
    [ CLOSED ] ─────────────────────────────────────────────────▶ [ OPEN ]
    (Normal Flow: Calls pass)                                   (Fail Fast: Return Fallback instantly!)
        ▲                                                             │
        │ (Threshold passes, e.g. 10/10 probe calls succeed)          │ (Wait cooldown period, e.g. 30s)
        └────────────────────── [ HALF_OPEN ] ◀───────────────────────┘
                                (Allow 10 probe test calls)
```

---

## 6. The Bulkhead Pattern: Thread Pool & Semaphore Isolation

Inspired by watertight bulkheads in oceanic ships that prevent a hull breach in one compartment from sinking the entire vessel:
- If `InventoryService` becomes unresponsive, isolate it into a dedicated thread pool of max 10 threads.
- Even if all 10 threads block, the remaining 190 threads in Tomcat continue processing payments and user profiles without degradation!

---

## 7. Java / Spring Boot Resilience4j Production Implementation

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class PaymentProcessingClient {

    private final RestClient restClient;

    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
    @Bulkhead(name = "paymentPool", type = Bulkhead.Type.THREADPOOL)
    @Retry(name = "paymentRetry")
    public PaymentConfirmation executeCharge(PaymentRequest request) {
        return restClient.post()
            .uri("https://api.stripe.internal/v1/charge")
            .body(request)
            .retrieve()
            .body(PaymentConfirmation.class);
    }

    // Graceful Fallback: Executed when circuit is OPEN or timeout occurs
    public PaymentConfirmation fallbackPayment(PaymentRequest request, Throwable throwable) {
        log.warn("Payment service unavailable or circuit open: {}. Queueing payment for async retry.",
            throwable.getMessage());

        return PaymentConfirmation.pendingReview(
            request.paymentId(),
            "Payment queued for async offline processing"
        );
    }
}
```

```yaml
# application.yml - Resilience4j Configuration
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-size: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 5
  retry:
    instances:
      paymentRetry:
        max-attempts: 3
        wait-duration: 200ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2
```

---

## 8. Side-by-Side Trade-off Analysis Table

| Metric | Monolith Architecture | Microservices Architecture |
| :--- | :--- | :--- |
| **Operational Overhead** | Low (Single process deploy) | High (Requires K8s, Grafana, OpenTelemetry) |
| **Failure Isolation** | Zero (One NPE or OOM takes down all) | High (Isolated via Circuit Breakers) |
| **Data Consistency** | Strict ACID transactions | Eventual consistency (Sagas & Events) |
| **Independent Scaling** | No (Scale entire monolith) | Yes (Autoscale high-demand services independently) |

---

## 9. Interview Rapid Q&A Checklist
- *What is the Outbox Pattern?* (Writing business data and domain events into the same relational database transactionally, then using a Change Data Capture process like Debezium to reliably publish events to Kafka without dual-write race conditions).
- *What is Service Mesh (Istio / Linkerd)?* (A dedicated infrastructure layer utilizing sidecar proxies to handle service-to-service mTLS encryption, traffic routing, and observability transparently to application code).
