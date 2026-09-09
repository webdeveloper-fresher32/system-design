# 🧭 Module 00 — Master Interview Thinking Canvas & Universal Framework

> **Ganesh's Golden Rule**: Never start drawing boxes or throwing out database names in the first 2 minutes of an HLD interview. Great engineers are hired for their **methodology, clarifying questions, and trade-off justification**, not for memorized buzzwords.

---

## 🏛️ The Universal 7-Step HLD Interview Execution Script

In a 45-minute FAANG or Tier-1 startup interview, manage your clock with precision:

```
┌───────────────────────────┬──────────────┬───────────────────────────────────────────────────────┐
│ Step                      │ Target Time  │ Goal & What to Say                                    │
├───────────────────────────┼──────────────┼───────────────────────────────────────────────────────┤
│ 1. Scope & Clarify        │ Min 00 – 04  │ Functional / Non-Functional boundaries & Scale        │
│ 2. Back-of-Envelope Math  │ Min 04 – 07  │ QPS, Peak QPS, Storage, Network Bandwidth             │
│ 3. API & Data Contracts   │ Min 07 – 11  │ Core endpoints, payload signatures, state definitions │
│ 4. High-Level Core Flow   │ Min 11 – 18  │ Clean happy-path diagram (Client -> LB -> Svc -> DB)  │
│ 5. Deep Dive Bottlenecks  │ Min 18 – 30  │ Concurrency, Partitioning, Caching, Locking, Queuing  │
│ 6. Failure Modes & Edge   │ Min 30 – 38  │ What if Redis dies? What if 3rd-party hangs? Split    │
│ 7. Bottlenecks & Tradeoff │ Min 38 – 43  │ CAP theorem, Cost vs Latency, Operational complexity  │
└───────────────────────────┴──────────────┴───────────────────────────────────────────────────────┘
```

---

## ⚡ Step 1: The Golden Opening Questions Matrix

When given ANY prompt, clarify these four dimensions before touching the whiteboard:

```
                 ┌────────────────────────────────────────────────┐
                 │          THE CLARIFYING QUADRANT               │
                 ├───────────────────────┬────────────────────────┤
                 │ 1. USERS & ACTORS     │ 2. TRAFFIC PROFILE     │
                 │ - Who calls this?     │ - Read vs Write Ratio? │
                 │ - Web/Mobile/Internal │ - Steady vs Spiky?     │
                 │ - Human vs Machine?   │ - Average vs Peak QPS? │
                 ├───────────────────────┼────────────────────────┤
                 │ 3. DATA & CONSISTENCY │ 4. LATENCY BUDGET      │
                 │ - Strict ACID vs      │ - P99 SLA target?      │
                 │   Eventual Converge?  │ - Sync HTTP vs Async   │
                 │ - Financial audit?    │   Event-Driven?        │
                 └───────────────────────┴────────────────────────┘
```

### Golden Clarifying Lines:
- *"Before choosing technologies, I want to clarify: is this system read-heavy (like Twitter/Instagram) or write-heavy (like IoT sensor logging or metric ingestion)?"*
- *"What is our consistency requirement? For instance, can a user tolerate seeing a like count delayed by 2 seconds (eventual consistency), or are we handling a wallet balance where phantom updates are unacceptable (strict ACID)?"*
- *"Do we need multi-region active-active availability, or is single-region active-passive with cross-region read replicas acceptable?"*

---

## 🧮 Step 2: Back-of-the-Envelope Mental Math Cheat Sheet

Commit these standard numbers to memory for rapid mental calculations:

### 1. The Rule of 86,400 (Seconds in a Day)
- There are **$\approx 86,400$ seconds in a day** ($\approx 10^5$ seconds for rough estimates).
- **1 Million requests / day** $\approx \frac{1,000,000}{86,400} \approx \mathbf{12\text{ requests/sec (RPS)}}$.
- **10 Million requests / day** $\approx \mathbf{120\text{ RPS}}$.
- **100 Million requests / day** $\approx \mathbf{1,200\text{ RPS}}$.
- **Peak Multiplier**: Always multiply average RPS by **$2\times \text{ to } 5\times$** to size for peak traffic. (e.g., $1,200\text{ avg RPS} \Rightarrow 3,000\text{ to } 6,000\text{ peak RPS}$).

### 2. Storage Estimation Math
- **1 Byte**: A single ASCII character.
- **1 KB**: $1,000$ Bytes (A typical JSON payload or metadata row).
- **1 MB**: $1,000$ KB (A short audio file or high-res image).
- **1 GB**: $1,000$ MB (A standard video file).
- **1 TB**: $1,000$ GB.
- **1 PB**: $1,000$ TB.

> **Example**: $100\text{ Million daily posts} \times 1\text{ KB per post metadata} = \mathbf{100\text{ GB/day}}$.  
> Over 5 years: $100\text{ GB} \times 365 \times 5 \approx \mathbf{182.5\text{ TB}}$. Add $2\times$ for indexes and replication $\approx \mathbf{365\text{ TB}}$.

### 3. Latency Numbers Every Systems Architect Must Know (Jeff Dean Numbers)
- **L1 cache reference**: $0.5\text{ ns}$
- **L2 cache reference**: $7\text{ ns}$
- **Main memory (RAM) reference**: $100\text{ ns}$
- **Read 1 MB sequentially from RAM**: $250,000\text{ ns} = 0.25\text{ ms}$
- **Read 1 MB sequentially from NVMe SSD**: $1\text{ ms}$
- **Round trip within same datacenter**: $0.5\text{ ms}$
- **Round trip across continents (US to EU)**: $150\text{ ms}$

---

## 🧠 The "Interviewer Curveball" Reaction Matrix

When the interviewer changes conditions halfway through, navigate using this mapping:

| Interviewer Curveball | What They Are Testing | Instant Pattern Reaction |
| :--- | :--- | :--- |
| *"Redis cluster crashes completely"* | Cache stampede, thundering herd | **Singleflight (Request coalescing), L1 in-process memory cache, Stale-while-revalidate** |
| *"Traffic jumps from 1k to 100k RPS in 10s"* | Autoscaling lag, capacity buffer | **Edge CDN caching, Token Bucket rate limiting, Queue-buffered asynchronous ingestion** |
| *"Downstream 3rd-party slows from 50ms to 8s"* | Thread pool starvation, cascading crash | **Aggressive read timeouts, Circuit Breaker (Envoy/Resilience4j), Bulkheading** |
| *"User clicks 'Pay' 4 times repeatedly"* | Non-idempotent endpoints, race condition | **Idempotency Key (UUIDv4) stored in DB unique constraint + forwarded to PSP** |
| *"Service crashes after DB write, before Kafka"* | Dual-write problem, partial failure | **Transactional Outbox Pattern + Debezium (CDC from database WAL)** |
| *"Celebrity with 80M followers posts"* | Hot partition, write amplification | **Hybrid Fanout (Push for regular users, Pull for celebrities), Partition salting** |
| *"Two users edit same file concurrently across WAN"*| CAP theorem, partition split-brain | **Local writes with CRDTs / Hybrid Logical Clocks (HLC) + async convergence** |
| *"Worker holding lock pauses for 20s (GC STW)"* | Unreliable distributed locks, clock drift | **Fencing Tokens (Monotonically increasing sequence checked at storage layer)** |
| *"Money transfer deadlocks under high concurrency"* | Circular row lock acquisition | **Deterministic sorted account ID locking (`SELECT ... FOR UPDATE ORDER BY id`)** |

---

## 🎨 Master Template for Any System Design Response

When you need to deliver a complete scenario, always execute in this order:

1. **Acknowledge and Frame**: Restate constraints in numbers.
2. **First 30 Seconds**: State your Golden Opening Line.
3. **Draft Architecture**: Draw the boxes cleanly with single responsibilities.
4. **Step-by-step Flow**: Number the arrows $1 \rightarrow 2 \rightarrow 3 \rightarrow 4$.
5. **Failure Modes**: State what happens if the database, cache, or network fails.
6. **Trade-offs**: Explain why you chose PostgreSQL over Cassandra, or Kafka over RabbitMQ.
7. **Curveball Ready**: Anticipate the $10\times$ and $100\times$ scale twists.
