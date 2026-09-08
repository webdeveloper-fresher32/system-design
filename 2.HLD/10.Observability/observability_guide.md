# 📊 Module 10: Observability, Production Telemetry & Site Reliability

> **Core Philosophy:** *In complex distributed architectures, systems fail in ways that cannot be reproduced locally. Observability is not mere logging; it is the capability to infer the internal state of any service at any microsecond based entirely on its telemetry outputs (Metrics, Logs, and Traces).*

---

## 📌 Table of Contents
1. [The Problem: Debugging Distributed Black Boxes](#1-the-problem-debugging-distributed-black-boxes)
2. [The Three Pillars of Observability: Metrics, Logs & Traces](#2-the-three-pillars-of-observability-metrics-logs--traces)
3. [The Four Golden Signals of Monitoring](#3-the-four-golden-signals-of-monitoring)
4. [Metrics Architecture: Pull (Prometheus) vs Push (StatsD / Datadog)](#4-metrics-architecture-pull-prometheus-vs-push-statsd--datadog)
5. [Structured JSON Logging & Centralized Aggregation (Loki / ELK)](#5-structured-json-logging--centralized-aggregation-loki--elk)
6. [Distributed Tracing: OpenTelemetry, W3C TraceContext & Jaeger](#6-distributed-tracing-opentelemetry-w3c-tracecontext--jaeger)
7. [Service Reliability Engineering: SLI, SLO & SLA Framework](#7-service-reliability-engineering-sli-slo--sla-framework)
8. [Health Checks: Liveness vs Readiness vs Startup Probes in K8s](#8-health-checks-liveness-vs-readiness-vs-startup-probes-in-k8s)
9. [Java / Spring Boot Micrometer & OpenTelemetry Tracing Implementation](#9-java--spring-boot-micrometer--opentelemetry-tracing-implementation)
10. [Side-by-Side Telemetry Comparison Table](#10-side-by-side-telemetry-comparison-table)
11. [Interview Rapid Q&A Checklist](#11-interview-rapid-qa-checklist)

---

## 1. The Problem: Debugging Distributed Black Boxes

In a monolithic app, debugging an error is simple: open the application log and read the stack trace.
In a microservices architecture:
- A user clicks "Buy Now" and gets a 500 error after 4 seconds.
- The request traversed 14 different microservices across 3 datacenters.
- Which microservice timed out? Which database query stalled?
- **Solution:** End-to-end **Distributed Tracing** with unified `TraceId` and structured metrics.

---

## 2. The Three Pillars of Observability

```
                       OBSERVABILITY TRIANGLE
                                ▲
                               / \
                              /   \
                             /     \
                            /       \
                 METRICS   /         \   LOGS
              (Prometheus) ═══════════ (Loki / ELK)
                         DISTRIBUTED TRACES
                        (OpenTelemetry / Jaeger)
```
- **Metrics (Aggregated Numbers):** CPU load, request QPS, p99 latency histograms. Highly compressed, low storage cost, ideal for real-time alerting.
- **Logs (Discrete Events):** Structured JSON records describing an isolated event with rich metadata (`user_id`, `error_stack`). High storage cost.
- **Distributed Traces (Request Journey):** Records the causal timeline and latency of a single request spanning multiple microservice boundaries.

---

## 3. The Four Golden Signals of Monitoring (Google SRE Book)

1. **Latency:** The time it takes to service a request. Track **p95, p99, and p99.9 percentiles**; never rely on averages!
2. **Traffic:** Demand placed on your system (HTTP requests/second, Kafka consumed msgs/sec).
3. **Errors:** The rate of requests that fail (e.g. HTTP 5xx responses or unhandled exceptions).
4. **Saturation:** How "full" your system is (CPU utilization %, JVM Heap usage %, connection pool utilization).

---

## 4. Distributed Tracing: OpenTelemetry & W3C TraceContext

```
[ Client Request ] ── (Injects Header: traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01)
       │
       ▼
[ API Gateway ] ────── (Span 1: duration = 240ms, TraceId = 4bf92f...)
       │
       ├──▶ [ Order Service ] ──── (Span 2: duration = 120ms, Parent = Span 1)
       │         │
       │         └──▶ [ Postgres Query ] ── (Span 3: duration = 40ms, Parent = Span 2)
       │
       └──▶ [ Payment Service ] ── (Span 4: duration = 80ms, Parent = Span 1)
```
- **TraceId:** Globally unique 128-bit ID identifying the overall end-to-end request.
- **SpanId:** Identifies a single unit of work (e.g. an HTTP call or a SQL query) with start time and duration.

---

## 5. Service Reliability Engineering: SLI, SLO & SLA Framework

- **SLI (Service Level Indicator):** A quantifiable metric of performance.
  - *Example:* "Percentage of successful `POST /orders` requests completing in $< 200	ext{ms}$."
- **SLO (Service Level Objective):** An internal target set by engineering.
  - *Example:* "SLI must be $\ge 99.9\%$ over a rolling 30-day window."
- **SLA (Service Level Agreement):** A legal contract with external customers with financial penalties if breached.
  - *Example:* "If uptime falls below $99.5\%$, customer receives a $20\%$ billing credit."

---

## 6. Java / Spring Boot Micrometer & Prometheus Telemetry

```java
@Service
@Slf4j
public class OrderFulfillmentService {

    private final Counter orderSuccessCounter;
    private final Counter orderFailureCounter;
    private final Timer orderProcessingTimer;

    public OrderFulfillmentService(MeterRegistry registry) {
        this.orderSuccessCounter = Counter.builder("orders_placed_total")
            .description("Count of successfully placed orders")
            .tag("status", "success")
            .register(registry);

        this.orderFailureCounter = Counter.builder("orders_placed_total")
            .description("Count of failed order attempts")
            .tag("status", "failed")
            .register(registry);

        this.orderProcessingTimer = Timer.builder("order_processing_duration_seconds")
            .description("Latency distribution of order fulfillment")
            .publishPercentiles(0.5, 0.95, 0.99) // p50, p95, p99 percentiles!
            .register(registry);
    }

    public Order processOrder(OrderRequest request) {
        return orderProcessingTimer.record(() -> {
            try {
                Order order = executeBusinessLogic(request);
                orderSuccessCounter.increment();
                return order;
            } catch (Exception ex) {
                orderFailureCounter.increment();
                log.error("Order processing failed for orderId: {}", request.orderId(), ex);
                throw ex;
            }
        });
    }
}
```

---

## 7. Side-by-Side Telemetry Comparison Table

| Telemetry Type | Data Format | Storage Retention | Best Used For |
| :--- | :--- | :--- | :--- |
| **Metrics** | Time-series float pairs | High (Months to Years) | Real-time automated alerting & Grafana dashboards |
| **Logs** | Unstructured or JSON text | Medium (7 to 30 Days) | Post-mortem root-cause debugging |
| **Traces** | Directed Acyclic Graph (Spans)| Low / Sampled (3 to 7 Days)| Pinpointing latency bottlenecks across services |

---

## 8. Interview Rapid Q&A Checklist
- *What is Trace Sampling?* (Recording only 1% or 5% of distributed traces to reduce storage and network overhead while still capturing statistically significant latency data).
- *What is the difference between Prometheus (Pull) and Datadog (Push)?* (Prometheus scrapes HTTP `/actuator/prometheus` endpoints on application pods periodically; Push systems have application agents push UDP/TCP packets to a central daemon).
