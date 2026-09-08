# 🏛️ High-Level Design (HLD) & Distributed Systems Course

> **Mastery Curriculum for 3+ YOE Software Engineers & System Architects.**
> Structured into **12 cleanly separated, high-yield modules** following the exact Low-Level Design (LLD) course format, with all code samples in **Java 21 / Spring Boot 3**.

---

## 🎯 Course Overview & Module Navigation

```
2.HLD/
├── 1.HLD-Fundamentals/               # Request lifecycle, Monolith vs Microservices, Statelessness, Estimation
├── 2.Networking-Basics/              # DNS, HTTP/2 & HTTP/3, TCP vs UDP, TLS Handshake, CDN, Load Balancers
├── 3.Scalability/                    # Horizontal vs Vertical, Autoscaling, CAP & PACELC, Consistent Hashing
├── 4.Databases/                      # SQL vs NoSQL, Master-Replica, Sharding, B+ Tree vs LSM, Transactions
├── 5.Caching/                        # Cache-Aside, Write-Through/Back, Eviction (LRU/LFU), Redis Cluster
├── 6.Messaging-Systems/              # Kafka Topics & Partitions, RabbitMQ, SQS, Idempotency, DLQs
├── 7.Storage-Systems/                # Object Storage (S3), Pre-signed URLs, Multi-part uploads, CDN Edge
├── 8.Microservices/                  # API Gateway, Service Discovery, Resilience4j (Circuit Breakers)
├── 9.Security-And-Auth/              # JWT, Access vs Refresh Tokens, OAuth2, Rate Limiting (Token Bucket)
├── 10.Observability/                 # 4 Golden Signals, Prometheus, Grafana, OpenTelemetry Traces
├── 11.Design-Tradeoffs/              # Consistency vs Latency, Latency vs Throughput, Read vs Write Heavy
└── 12.Real-System-Design-Interviews/ # 9-Step Interview Framework + Real Systems (WhatsApp, Uber, Netflix)
```

---

## 📚 Module-Wise Guides

| # | Module | Core Topics | Guide Link |
| :---: | :--- | :--- | :--- |
| **01** | **HLD Fundamentals** | Client-Server, Monolith vs Microservices, Capacity Estimation | [Read Guide](./1.HLD-Fundamentals/hld_fundamentals_guide.md) |
| **02** | **Networking Basics** | DNS Resolution, HTTP/2 & 3, TCP vs UDP, TLS 1.3, Load Balancing | [Read Guide](./2.Networking-Basics/networking_basics_guide.md) |
| **03** | **Scalability** | Horizontal Scaling, Statelessness, CAP & PACELC, Consistent Hashing | [Read Guide](./3.Scalability/scalability_guide.md) |
| **04** | **Databases** | SQL vs NoSQL, Replication Lag, Sharding, B+ Tree vs LSM, Saga | [Read Guide](./4.Databases/databases_guide.md) |
| **05** | **Caching** | Cache-Aside, Redis Data Structures, LRU/LFU, Stampede/Avalanche Fixes | [Read Guide](./5.Caching/caching_guide.md) |
| **06** | **Messaging Systems** | Apache Kafka Partitions, RabbitMQ, SQS, At-Least-Once, DLQ | [Read Guide](./6.Messaging-Systems/messaging_systems_guide.md) |
| **07** | **Storage Systems** | S3 Object Storage, Pre-Signed Uploads, Multi-Part Chunking, CDN | [Read Guide](./7.Storage-Systems/storage_systems_guide.md) |
| **08** | **Microservices** | API Gateways, Circuit Breakers (Resilience4j), Bulkheads, Fallbacks | [Read Guide](./8.Microservices/microservices_guide.md) |
| **09** | **Security & Auth** | JWT Structure, Token Refresh, OAuth2, Distributed Rate Limiting | [Read Guide](./9.Security-And-Auth/security_and_auth_guide.md) |
| **10** | **Observability** | 4 Golden Signals, Prometheus, Grafana, Distributed Tracing | [Read Guide](./10.Observability/observability_guide.md) |
| **11** | **Design Trade-offs** | CAP in practice, Batching vs Streaming, Read vs Write Heavy | [Read Guide](./11.Design-Tradeoffs/design_tradeoffs_guide.md) |
| **12** | **Real System Design** | 9-Step Interview Framework, WhatsApp, Uber, Netflix, TinyURL, AI RAG | [Read Guide](./12.Real-System-Design-Interviews/real_system_design_interviews_guide.md) |
