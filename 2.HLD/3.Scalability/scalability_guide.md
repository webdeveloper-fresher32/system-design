# ⚖️ Module 3: Scalability, Consensus & Distributed System Principles

> **Core Philosophy:** *Scalability is the property of a distributed system to handle increased load simply by adding hardware resources without architectural redesign. Every scalable system must navigate the mathematical limits of the CAP and PACELC theorems.*

---

## 📌 Table of Contents
1. [The Problem: Hardware Walls & Traffic Surges](#1-the-problem-hardware-walls--traffic-surges)
2. [Vertical Scaling (Scale-Up) vs Horizontal Scaling (Scale-Out)](#2-vertical-scaling-scale-up-vs-horizontal-scaling-scale-out)
3. [Statelessness: The Foundation of Infinite Scale](#3-statelessness-the-foundation-of-infinite-scale)
4. [The CAP Theorem: Consistency, Availability & Partition Tolerance](#4-the-cap-theorem-consistency-availability--partition-tolerance)
5. [The PACELC Theorem: Trade-offs in Normal Operation](#5-the-pacelc-theorem-trade-offs-in-normal-operation)
6. [Consistent Hashing & The Virtual Node Algorithm](#6-consistent-hashing--the-virtual-node-algorithm)
7. [Distributed Consensus: Raft vs Paxos Deep Dive](#7-distributed-consensus-raft-vs-paxos-deep-dive)
8. [Auto-Scaling Architecture: Metrics & Horizontal Pod Autoscaling (HPA)](#8-auto-scaling-architecture-metrics--horizontal-pod-autoscaling-hpa)
9. [Java / Spring Boot Consistent Hash Ring Implementation](#9-java--spring-boot-consistent-hash-ring-implementation)
10. [Side-by-Side Trade-off Analysis Table](#10-side-by-side-trade-off-analysis-table)
11. [Interview Rapid Q&A Checklist](#11-interview-rapid-qa-checklist)

---

## 1. The Problem: Hardware Walls & Traffic Surges

When traffic scales from $1,000$ to $1,000,000$ concurrent requests:
1. **Vertical Scaling Ceilings:** You can buy a 128-core server with 2TB RAM, but you cannot buy a 10,000-core server. Costs scale exponentially while returns diminish.
2. **Cascading Node Death:** If a single monolithic server crashes, $100\%$ of users experience downtime.
3. **Network Sockets Exhaustion:** Operating systems limit max open TCP file descriptors (e.g. 65,535 per IP).

---

## 2. Vertical Scaling vs Horizontal Scaling

```
VERTICAL SCALING (Scale-Up):
┌───────────────────────────┐         ┌───────────────────────────┐
│ Small Server: 8 Core/32GB │  ───▶   │ High-End Box: 128C/1TB    │
└───────────────────────────┘         └───────────────────────────┘
* Pros: Zero code changes, single shared memory, ACID transactions remain simple.
* Cons: Exponential financial cost, hardware upper ceiling, Single Point of Failure (SPOF).

HORIZONTAL SCALING (Scale-Out):
                   ┌──▶ [ App Server 1 (Commodity VM / Pod) ]
[ Load Balancer ] ─┼──▶ [ App Server 2 (Commodity VM / Pod) ]
                   └──▶ [ App Server N (Elastic Auto-Scaling) ]
* Pros: Elasticity (scale to zero or 10,000 instances), zero downtime deployments, fault isolation.
* Cons: Distributed concurrency, network latency, data consistency challenges.
```

---

## 3. Statelessness: The Foundation of Infinite Scale

An application tier is **stateless** when every request can be handled by ANY server instance indiscriminately:

```
ANTI-PATTERN (Stateful Web Server):
Request 1 (Login)  ──▶ Server A (Stores session object in JVM memory)
Request 2 (Browse) ──▶ Server B (Checks memory -> Session missing! Throws 401 Unauthorized!)

PRODUCTION PATTERN (Stateless Architecture):
Request 1 (Login)  ──▶ Server A ──▶ Generates Signed JWT (or stores session in Redis)
Request 2 (Browse) ──▶ Server B ──▶ Validates JWT cryptographically (or reads Redis)
* Result: Server A can be terminated, upgraded, or crashed without user noticing!
```

---

## 4. The CAP Theorem: Consistency, Availability & Partition Tolerance

In any asynchronous distributed network, network delays or packet losses (**Network Partitions - P**) are mathematically guaranteed to occur. When a partition happens, a system MUST choose between:

```
                CONSISTENCY (C)
                     ▲
                    / \
                   /   \
                  /     \
                 /   P   \
                /         \
  AVAILABILITY (A) ════════ PARTITION TOLERANCE (P)

CP SYSTEM (Consistency over Availability):
- If Node 1 cannot talk to Node 2, REJECT writes on Node 2 rather than serve conflicting data.
- Examples: Apache ZooKeeper, Google Cloud Spanner, Banking ledgers.

AP SYSTEM (Availability over Consistency):
- Both Node 1 and Node 2 accept reads and writes, even though their data temporarily diverges.
- Data synchronizes eventually via background repair (Read Repair / Hinted Handoff).
- Examples: Amazon DynamoDB, Apache Cassandra, Couchbase, Social feeds.
```

---

## 5. The PACELC Theorem: Trade-offs in Normal Operation

The CAP theorem only describes behavior during a catastrophic network partition. The **PACELC Theorem** extends CAP by explaining what happens during normal, healthy operations:

```
IF Partition (P):
    Choose between Availability (A) and Consistency (C)
ELSE (E):
    Choose between Latency (L) and Consistency (C)

- MongoDB / PostgreSQL: PC / EC (Consistent during partition; Consistent in normal operation at the cost of replication latency).
- Cassandra / DynamoDB: PA / EL (Available during partition; Low-latency in normal operation at the cost of eventual consistency).
```

---

## 6. Consistent Hashing & The Virtual Node Algorithm

### The Modulo Hashing Flaw
If you have $N$ servers, simple hashing maps keys via:
$$	ext{serverIndex} = 	ext{hash}(	ext{key}) \pmod N$$
If a server dies and $N$ becomes $N-1$, almost **$100\%$ of keys remap to different servers**, causing a catastrophic global cache miss!

### The Consistent Hashing Ring Solution
1. Map both servers and keys onto a circular $360^\circ$ ring (range $0$ to $2^{32}-1$).
2. A key is assigned to the first server encountered moving clockwise.
3. When a server is added or removed, only $\mathbf{K/N}$ keys on average migrate!
4. **Virtual Nodes:** To prevent non-uniform distribution ("hotspots"), each physical server is mapped to 100-200 virtual points on the ring.

```
                  Server A (0°)
                 /                    Key 1    /               \    Key 2
               /                     Server C (240°) ────────── Server B (120°)
                     Key 3
```

---

## 7. Distributed Consensus: Raft vs Paxos Deep Dive

When multiple distributed nodes must agree on a single source of truth (e.g., leader election, distributed locking):
- **Paxos:** Mathematically proven consensus protocol. Notoriously difficult to understand and implement correctly.
- **Raft:** Decomposes consensus into 3 independent, understandable sub-problems:
  1. **Leader Election:** Heartbeat timers. If a follower misses a heartbeat, it increments election term and requests votes. Quorum of $\lfloor N/2 floor + 1$ required.
  2. **Log Replication:** Leader accepts writes, appends to its WAL, streams log entries to followers. Once a majority ACK, leader commits.
  3. **Safety:** A node can only be elected leader if its log contains all committed entries from past terms.

---

## 8. Java / Spring Boot Consistent Hash Ring Implementation

```java
public class ConsistentHashRing<T> {

    private final HashFunction hashFunction;
    private final int numberOfReplicas; // Number of virtual nodes per physical node
    private final SortedMap<Long, T> circle = new ConcurrentSkipListMap<>();

    public ConsistentHashRing(int numberOfReplicas) {
        this.numberOfReplicas = numberOfReplicas;
        this.hashFunction = Hashing.murmur3_128();
    }

    public void addNode(T node) {
        for (int i = 0; i < numberOfReplicas; i++) {
            long hash = hashFunction.hashString(node.toString() + "-VN-" + i, StandardCharsets.UTF_8).asLong();
            circle.put(hash, node);
        }
    }

    public void removeNode(T node) {
        for (int i = 0; i < numberOfReplicas; i++) {
            long hash = hashFunction.hashString(node.toString() + "-VN-" + i, StandardCharsets.UTF_8).asLong();
            circle.remove(hash);
        }
    }

    public T getNode(String key) {
        if (circle.isEmpty()) {
            return null;
        }
        long hash = hashFunction.hashString(key, StandardCharsets.UTF_8).asLong();
        if (!circle.containsKey(hash)) {
            // Find clockwise closest node
            SortedMap<Long, T> tailMap = circle.tailMap(hash);
            hash = tailMap.isEmpty() ? circle.firstKey() : tailMap.firstKey();
        }
        return circle.get(hash);
    }
}
```

---

## 9. Side-by-Side Trade-off Analysis Table

| Architecture Choice | Primary Benefit | Significant Cost | Ideal Production Workload |
| :--- | :--- | :--- | :--- |
| **Scale-Up (Vertical)** | Zero code rewrite, ACID intact | Hardware ceiling, high cost | Relational primary databases |
| **Scale-Out (Horizontal)**| Linear capacity expansion | Concurrency & eventual consistency | Stateless microservices |
| **Consistent Hashing** | Minimal key remapping on scaling | Virtual node memory tracking | Distributed caches (Memcached/Redis) |
| **CP System (Raft/Zk)** | Strictly consistent data | Latency penalty, availability loss | Service discovery, distributed locks |
| **AP System (Cassandra)**| Zero write downtime | Stale reads, read-repair overhead | IoT metrics, social timelines |

---

## 10. Interview Rapid Q&A Checklist
- *What is Split-Brain and how is it prevented?* (When network partitions split a cluster into two halves and both elect a leader; prevented by requiring an odd number of nodes and strict quorum: $\lfloor N/2 floor + 1$).
- *Why is Consistent Hashing critical for caching clusters?* (Without it, adding a single cache server invalidates all cached keys, causing all traffic to slam and crash the database).
- *What is Amdahl's Law?* (The theoretical limit of latency speedup when adding more parallel processors is constrained by the sequential, unparallelizable portions of the program).
