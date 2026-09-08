# 🌐 Module 2: Networking, Protocols & API Gateways

> **Core Philosophy:** *Every distributed system is fundamentally bounded by the physics of the network. Understanding protocol handshakes, connection multiplexing, and traffic routing determines the performance ceiling of your architecture.*

---

## 📌 Table of Contents
1. [The Problem: Network Bottlenecks at Scale](#1-the-problem-network-bottlenecks-at-scale)
2. [DNS Architecture: From Local Cache to Root & Anycast BGP](#2-dns-architecture-from-local-cache-to-root--anycast-bgp)
3. [The Transport Layer: TCP 3-Way Handshake vs UDP](#3-the-transport-layer-tcp-3-way-handshake-vs-udp)
4. [HTTP Evolution: HTTP/1.1 vs HTTP/2 Multiplexing vs HTTP/3 (QUIC)](#4-http-evolution-http11-vs-http2-multiplexing-vs-http3-quic)
5. [TLS 1.3 Handshake & Asymmetric vs Symmetric Cryptography](#5-tls-13-handshake--asymmetric-vs-symmetric-cryptography)
6. [Real-Time Protocols: WebSockets vs SSE vs Long Polling](#6-real-time-protocols-websockets-vs-sse-vs-long-polling)
7. [API Gateways: Architecture, Rate Limiting & Routing](#7-api-gateways-architecture-rate-limiting--routing)
8. [Load Balancing: Layer 4 vs Layer 7 & Routing Algorithms](#8-load-balancing-layer-4-vs-layer-7--routing-algorithms)
9. [Java / Spring Boot High-Throughput Network Implementation](#9-java--spring-boot-high-throughput-network-implementation)
10. [Side-by-Side Protocol Comparison Table](#10-side-by-side-protocol-comparison-table)
11. [Interview Rapid Q&A Checklist](#11-interview-rapid-qa-checklist)

---

## 1. The Problem: Network Bottlenecks at Scale

### Real-World Domain Example: The Latency Compounder ⏱️
Imagine a user in Sydney accessing a backend hosted in us-east-1 (Virginia).
- Physical round-trip time (RTT) across trans-oceanic fiber cables: ~160ms.
- Under HTTP/1.1 with old TLS 1.2:
  1. TCP Handshake: 1 RTT (160ms)
  2. TLS 1.2 Handshake: 2 RTTs (320ms)
  3. HTTP GET Request + Response: 1 RTT (160ms)
  - **Total time before 1st byte of data is seen:** $160 + 320 + 160 = \mathbf{640	ext{ms}}$!
- **Solution:** Edge CDN caching + TLS 1.3 (1 RTT) or HTTP/3 QUIC (0-RTT connection resumption).

---

## 2. DNS Architecture: From Local Cache to Root & Anycast BGP

When a client resolves `api.uber.com`:

```
[ Client Browser ]
       │ 1. Checks Browser DNS Cache (TTL: 60s)
       │ 2. Checks OS /etc/hosts & OS DNS Cache
       ▼ (Cache Miss)
[ ISP Recursive Resolver (e.g. 8.8.8.8) ]
       │ 3. Query Root Server (".")
       ▼
[ Root DNS Server ] ──▶ Returns IP of ".com" TLD Name Server
       │ 4. Query ".com" TLD
       ▼
[ TLD Name Server ] ──▶ Returns Authoritative Name Server for "uber.com"
       │ 5. Query "api.uber.com"
       ▼
[ Authoritative DNS (Route53 / Cloudflare) ]
       │ ──▶ Returns A Record (IPv4: 104.16.24.1) or CNAME
       ▼
[ Client initiates TCP Connection to 104.16.24.1 ]
```

### Anycast BGP Routing
Rather than binding an IP to a single physical machine in one datacenter, **Anycast** announces the same IP address from 200+ edge datacenters worldwide via Border Gateway Protocol (BGP). The internet backbone router automatically sends the client's packet to the geographically closest datacenter!

---

## 3. The Transport Layer: TCP 3-Way Handshake vs UDP

```
TCP 3-WAY HANDSHAKE (Reliable, Ordered, Byte-Stream):
Client ──────────────── SYN ────────────────▶ Server  (Seq = 100)
Client ◀──────────── SYN + ACK ────────────── Server  (Seq = 300, Ack = 101)
Client ──────────────── ACK ────────────────▶ Server  (Ack = 301)
* Guarantees: Packet retransmission on drop, congestion control, in-order delivery.

UDP (User Datagram Protocol - Connectionless, Low-Overhead):
Client ───────────── Datagram 1 ────────────▶ Server  (Fire-and-forget!)
Client ───────────── Datagram 2 ────────────▶ Server
* Zero connection handshake, zero retransmission delay.
* Used for: Live audio/video (WebRTC/Zoom), Online FPS gaming, DNS lookups.
```

---

## 4. HTTP Evolution: HTTP/1.1 vs HTTP/2 Multiplexing vs HTTP/3 (QUIC)

### The Head-of-Line (HoL) Blocking Bottleneck
- **HTTP/1.1:** A single TCP connection handles only one HTTP request at a time. If Request 1 (a large image) is slow, Request 2 and 3 must wait in queue.
- **HTTP/2 (Binary Framing & Multiplexing):** Multiple logical bidirectional streams interleave over a single TCP connection concurrently.
  - *Remaining Flaw:* If 1 packet is dropped at the TCP layer, all streams stall until retransmission completes (TCP-level HoL blocking).
- **HTTP/3 (QUIC over UDP):** Streams are independent at the transport layer! If Stream 1 loses a packet, Stream 2 and 3 continue streaming without delay.

```
HTTP/1.1:   [ Request 1 ] ──▶ [ Response 1 ] ──▶ [ Request 2 ] ──▶ [ Response 2 ]
HTTP/2:     [ Stream 1: Chunk A ][ Stream 2: Chunk A ][ Stream 1: Chunk B ] (1 TCP Pipe)
HTTP/3:     Independent UDP Streams (Zero TCP HoL blocking, 0-RTT handshakes)
```

---

## 5. TLS 1.3 Handshake & Cryptography

```
TLS 1.3 (1-RTT Full Handshake):
Client ──────── ClientHello + KeyShare (ECDH) ───────▶ Server
Client ◀─────── ServerHello + KeyShare + Cert + Finished ─ Server
[ Encrypted Application Data can now be sent immediately! ]
```
1. **Asymmetric Cryptography (ECDH / RSA):** Authenticates the server certificate and establishes a shared ephemeral session key.
2. **Symmetric Cryptography (AES-256-GCM / ChaCha20):** High-speed encryption of the actual HTTP payload using the agreed session key.

---

## 6. Real-Time Protocols: WebSockets vs SSE vs Long Polling

```
LONG POLLING (Inefficient):
Client ── HTTP GET ──▶ Server (Holds open up to 30s)
Client ◀─ HTTP Response ─ Server (Closes socket. Client immediately opens new HTTP GET)

SERVER-SENT EVENTS (SSE - Server-to-Client Only):
Client ── HTTP GET (Accept: text/event-stream) ──▶ Server
Client ◀─ event: price-update 
 data: 124.5 ─── Server (Keeps connection open forever)
* Best for: AI Token Streaming (ChatGPT), Stock price tickers, Live sports scores.

WEBSOCKET (Full-Duplex Bidirectional):
Client ── HTTP Upgrade: websocket ──▶ Server
Client ◀─ 101 Switching Protocols ─── Server
Client ◀══════ Persistent Bidirectional TCP Socket ══════▶ Server (2-Byte Framing overhead)
* Best for: WhatsApp Chat, Multiplayer Gaming, Collaborative Whiteboards.
```

---

## 7. API Gateways: Architecture, Rate Limiting & Routing

An API Gateway sits as the single reverse-proxy entry point for external client traffic:

```
[ Mobile / Web Client ]
           │
           ▼ (HTTPS / TLS 1.3)
┌────────────────────────────────────────────────────────┐
│             API GATEWAY (Spring Cloud Gateway)         │
│  ├── 1. Global CORS & SSL Termination                  │
│  ├── 2. JWT Signature Verification (Stateless Auth)    │
│  ├── 3. Distributed Rate Limiter (Redis Token Bucket)  │
│  ├── 4. Request Header Enrichment (Inject X-User-Id)   │
│  └── 5. Dynamic Routing & Circuit Breaking             │
└────────────────────────────────────────────────────────┘
         │                   │                   │
         ▼                   ▼                   ▼
  [ Auth Service ]    [ Order Service ]   [ Payment Service ]
```

---

## 8. Load Balancing: Layer 4 vs Layer 7 & Routing Algorithms

- **Layer 4 (Transport / TCP Level):** Inspects source/dest IP and Port. Extremely high packet forwarding throughput (>1M QPS). Cannot inspect HTTP headers. (e.g. AWS NLB, Linux IPVS).
- **Layer 7 (Application / HTTP Level):** Inspects URL path (`/orders` vs `/users`), cookies, headers. Handles SSL termination and compression. (e.g. AWS ALB, Nginx, Envoy).

### Load Balancing Algorithms:
1. **Round Robin:** Equal sequential distribution. Flawed if request costs vary widely.
2. **Least Connections:** Routes to the server with the fewest active TCP sockets. Ideal for long-lived WebSocket sessions.
3. **Consistent Hashing:** Hashes client IP or `user_id` onto a ring. Guarantees that the same user always hits the same cache-warm server.

---

## 9. Java / Spring Boot High-Throughput Network Implementation

### A. Non-Blocking SSE Streaming Controller (Spring WebFlux)
```java
@RestController
@RequestMapping("/api/v1/stream")
@RequiredArgsConstructor
public class RealtimeMarketStreamController {

    private final MarketPriceBroadcastService marketService;

    public record StockQuote(String symbol, BigDecimal price, Instant timestamp) {}

    @GetMapping(value = "/quotes", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<StockQuote>> streamQuotes(@RequestParam String symbol) {
        return marketService.getQuoteStream(symbol)
            .map(quote -> ServerSentEvent.<StockQuote>builder()
                .id(UUID.randomUUID().toString())
                .event("quote-update")
                .data(quote)
                .build())
            .delayElements(Duration.ofMillis(250)); // Non-blocking push
    }
}
```

### B. Spring Cloud Gateway Route & Rate Limiting Configuration
```java
@Configuration
public class GatewayConfiguration {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder,
                                           RedisRateLimiter redisRateLimiter,
                                           JwtAuthenticationFilter authFilter) {
        return builder.routes()
            .route("order-service-route", r -> r.path("/api/v1/orders/**")
                .filters(f -> f.filter(authFilter.apply(new JwtAuthenticationFilter.Config()))
                               .requestRateLimiter(c -> c.setRateLimiter(redisRateLimiter)
                                                         .setKeyResolver(exchange -> 
                                                             Mono.just(exchange.getRequest().getRemoteAddress().getAddress().getHostAddress())))
                               .circuitBreaker(c -> c.setName("orderCircuitBreaker")
                                                     .setFallbackUri("forward:/fallback/orders")))
                .uri("lb://ORDER-SERVICE"))
            .build();
    }

    @Bean
    public RedisRateLimiter redisRateLimiter() {
        // Replenish 50 tokens/second, burst capacity of 100 tokens
        return new RedisRateLimiter(50, 100);
    }
}
```

---

## 10. Side-by-Side Protocol Comparison Table

| Protocol | Transport | Full Duplex? | Overhead per Message | Best Production Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **REST (HTTP/1.1)** | TCP | No (Request-Response) | High (Text headers ~1KB) | Public APIs, CRUD web services |
| **gRPC (HTTP/2)** | TCP | Yes (Bidirectional) | Minimal (Protobuf binary) | Internal microservice-to-microservice RPC |
| **WebSocket** | TCP | Yes (Full-Duplex) | Ultra-low (2 bytes) | Real-time chat, collaborative whiteboards |
| **Server-Sent Events**| TCP | No (Server -> Client) | Low (Plain text stream) | LLM token streaming, live sports tickers |
| **WebRTC** | UDP | Yes (Peer-to-Peer) | Minimal | Voice calling, video conferencing |

---

## 11. Interview Rapid Q&A Checklist
- *Why not use WebSockets for everything?* (WebSockets are stateful, hold persistent TCP connections, break standard HTTP caching, and complicate horizontal autoscaling behind load balancers).
- *What is SSL Termination?* (Decrypting TLS traffic at the load balancer or API Gateway so that downstream internal microservices can communicate over high-speed unencrypted internal VPC networks).
- *What is Connection Pooling?* (Reusing existing TCP connections instead of paying the 3-way handshake and TLS negotiation cost on every single HTTP request).
