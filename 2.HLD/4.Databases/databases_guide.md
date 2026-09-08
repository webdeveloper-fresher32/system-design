# 💾 Module 4: Database Architecture, Replication & Sharding

> **Core Philosophy:** *A database is not an abstract black box; it is an I/O optimization engine balancing disk seeks, write-ahead logs, cache buffers, and network consensus. Matching storage engines to access patterns is the most consequential decision in High-Level Design.*

---

## 📌 Table of Contents
1. [The Problem: Database Saturation at Scale](#1-the-problem-database-saturation-at-scale)
2. [Storage Engine Internals: B+ Trees vs LSM Trees](#2-storage-engine-internals-b-trees-vs-lsm-trees)
3. [Relational (SQL) vs NoSQL vs NewSQL Deep Dive](#3-relational-sql-vs-nosql-vs-newsql-deep-dive)
4. [Master-Replica (Read-Replica) Architecture & Replication Lag](#4-master-replica-read-replica-architecture--replication-lag)
5. [Database Sharding Strategies: Range, Hash & Directory](#5-database-sharding-strategies-range-hash--directory)
6. [Transactions & Concurrency: ACID Isolation Levels & Locking](#6-transactions--concurrency-acid-isolation-levels--locking)
7. [Distributed Transactions: 2PC vs Saga Pattern](#7-distributed-transactions-2pc-vs-saga-pattern)
8. [Change Data Capture (CDC) with Debezium & Kafka](#8-change-data-capture-cdc-with-debezium--kafka)
9. [Java / Spring Boot Multi-DataSource & Routing Implementation](#9-java--spring-boot-multi-datasource--routing-implementation)
10. [Side-by-Side Database Trade-off Comparison](#10-side-by-side-database-trade-off-comparison)
11. [Interview Rapid Q&A Checklist](#11-interview-rapid-qa-checklist)

---

## 1. The Problem: Database Saturation at Scale

When a single relational database reaches $10,000+$ queries per second:
1. **Disk I/O Bottlenecks:** Random disk seeks on mechanical HDDs or SSD write-amplification choke query throughput.
2. **Connection Pool Exhaustion:** Postgres forks a process per connection (costing ~10MB RAM each); 2,000 connections consume 20GB RAM just on connection overhead.
3. **Lock Contention:** Concurrent transactions on hot rows (e.g. inventory counters) stall waiting on row-level exclusive locks (`SELECT FOR UPDATE`).

---

## 2. Storage Engine Internals: B+ Trees vs LSM Trees

```
B+ TREE (Read-Optimized - PostgreSQL InnoDB / MySQL):
               [ Root Node: Keys 20, 50 ]
              /            |                   [ Page 1: <20 ]  [ Page 2: 20-50 ]  [ Page 3: >50 ]
            │               │                    │
      [ Leaf Node ] ──▶ [ Leaf Node ] ────▶ [ Leaf Node ] (Doubly Linked List)
* Reads: O(log N) - fast index traversal.
* Writes: Requires random in-place disk overwrites; page splitting under heavy inserts causes write amplification!

LSM TREE (Log-Structured Merge Tree - Write-Optimized - Cassandra / RocksDB):
1. In-Memory: Writes append immediately to an in-memory sorted MemTable (ConcurrentSkipListMap).
2. Durability: Simultaneously appended to an append-only Write-Ahead Log (WAL) sequentially on disk.
3. Flushing: When MemTable fills (e.g. 64MB), it is written to disk as an immutable SSTable (Sorted String Table).
4. Compaction: Background threads merge multiple SSTables sequentially and discard deleted/stale keys.
* Writes: Ultra-fast sequential append!
* Reads: Multi-level lookup (MemTable -> Bloom Filter -> SSTable).
```

---

## 3. Relational (SQL) vs NoSQL vs NewSQL Deep Dive

| Database Type | Examples | Core Mechanism | Trade-offs | Ideal Workload |
| :--- | :--- | :--- | :--- | :--- |
| **Relational (RDBMS)** | PostgreSQL, MySQL | B+ Trees, ACID, strict schemas, foreign keys | Hard to shard across nodes | Financial ledgers, e-commerce orders |
| **Document NoSQL** | MongoDB, Couchbase | BSON/JSON documents, dynamic schemas | No cross-collection ACID | User profiles, CMS, flexible catalogs |
| **Wide-Column NoSQL** | Cassandra, ScyllaDB | LSM Trees, peer-to-peer ring, token sharding | No JOINs, complex query modeling | Messaging history, time-series metrics |
| **Key-Value NoSQL** | Redis, DynamoDB | In-memory hash tables, RocksDB engines | Limited secondary query capability | Caches, sessions, leaderboards |
| **NewSQL** | CockroachDB, Spanner | Distributed consensus (Raft) + distributed ACID | Higher write latency | Global banking requiring multi-region ACID |

---

## 4. Master-Replica Architecture & Replication Lag

```
[ Application Fleet ]
         │ (Writes: INSERT/UPDATE/DELETE)
         ▼
┌──────────────────┐      Async Binary Log Streaming       ┌──────────────────┐
│ Primary (Master) │ ────────────────────────────────────▶ │ Read Replica 1   │
│ (Single Writer)  │                                       └──────────────────┘
└──────────────────┘ ────────────────────────────────────▶ ┌──────────────────┐
                                                           │ Read Replica 2   │
                                                           └──────────────────┘
```

### The Replication Lag Trap (Read-Your-Own-Writes Inconsistency)
1. User updates their profile name from "Bob" to "Robert" $ightarrow$ writes to Primary.
2. User refreshes browser immediately $ightarrow$ read query hits Read Replica 1.
3. Read Replica 1 is 500ms behind in applying binary logs $ightarrow$ user still sees "Bob"!
- **Production Solution:**
  - **Sticky Read Routing:** If a user makes a write, route ALL reads for that specific user to the Primary Master for the next 5-10 seconds.
  - **Version Token / LSN:** Include the Write Log Sequence Number (LSN) in the client session; replica only answers once its local LSN $\ge$ client LSN.

---

## 5. Database Sharding Strategies: Range, Hash & Directory

When dataset size exceeds a single machine's disk volume ($>2	ext{ TB}$) or write throughput exceeds a single Master:

```
HORIZONTAL SHARDING:
┌───────────────────────────┐  ┌───────────────────────────┐  ┌───────────────────────────┐
│          Shard 1          │  │          Shard 2          │  │          Shard 3          │
│   Users with ID: 0 - 1M   │  │  Users with ID: 1M - 2M   │  │  Users with ID: 2M - 3M   │
└───────────────────────────┘  └───────────────────────────┘  └───────────────────────────┘
```
1. **Range-Based Sharding:** Shard by date or alphabetical range.
   - *Flaw:* Hotspots! Today's date gets 99% of all write traffic.
2. **Hash-Based Sharding:** `shard = hash(user_id) % numberOfShards`.
   - *Advantage:* Uniform write distribution across all nodes.
   - *Flaw:* Resharding requires moving data when adding shards (mitigate via Consistent Hashing).
3. **Directory-Based Sharding:** A centralized lookup service maps entity IDs to specific physical database connection strings.

---

## 6. Transactions & Concurrency: ACID Isolation Levels & Locking

### ACID Properties:
- **Atomicity:** All modifications in a transaction succeed, or all are rolled back.
- **Consistency:** Database transitions from one valid invariant state to another.
- **Isolation:** Concurrent transactions do not interfere with each other.
- **Durability:** Committed transactions persist on non-volatile disk even during power outage.

### Isolation Levels & Phenomena:
| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Mechanism |
| :--- | :---: | :---: | :---: | :--- |
| **Read Uncommitted** | Possible | Possible | Possible | No shared locks |
| **Read Committed** | **Prevented** | Possible | Possible | Short-lived read locks |
| **Repeatable Read** (MySQL default) | **Prevented** | **Prevented** | Possible | MVCC snapshot isolation |
| **Serializable** | **Prevented** | **Prevented** | **Prevented** | Two-Phase Locking (2PL) |

---

## 7. Distributed Transactions: 2PC vs Saga Pattern

### Why Two-Phase Commit (2PC) Fails in Microservices:
2PC uses a central coordinator that acquires locks across all databases before committing. If the coordinator crashes mid-flight, database rows remain locked indefinitely, degrading availability across the enterprise.

### The Saga Pattern (Orchestration-Based):
Break the distributed transaction into a series of local transactions. Each step has a corresponding **Compensating Transaction** to undo changes if a downstream step fails:

```
[ Saga Orchestrator ]
         │ 1. Create Pending Order ──▶ [ Order Service ] (Local DB Commit)
         │
         ├── 2. Reserve Stock      ──▶ [ Inventory Service ] (Local DB Commit)
         │
         ├── 3. Execute Payment    ──▶ [ Payment Service ] ──❌ Fails (Card Declined)!
         │
         └── 4. Trigger Compensation ─▶ [ Inventory Service ] (Release Reserved Stock)
```

---

## 8. Java / Spring Boot Routing DataSource Implementation

```java
// Dynamically routes read queries to Replicas and write queries to Master
public class TransactionRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
            ? DataSourceType.READ_REPLICA
            : DataSourceType.WRITE_MASTER;
    }

    public enum DataSourceType {
        WRITE_MASTER,
        READ_REPLICA
    }
}
```

---

## 9. Side-by-Side Database Trade-off Comparison

| Architecture Pattern | Read Scaling | Write Scaling | Data Consistency | Operational Complexity |
| :--- | :--- | :--- | :--- | :--- |
| **Single Primary SQL** | Limited to 1 box | Limited to 1 box | Strict ACID | Low |
| **Primary + Read Replicas**| $10	imes$ - $50	imes$ reads | Limited to 1 box | Eventual (Replication Lag)| Medium |
| **Sharded SQL Cluster** | Linear with shards | Linear with shards | Complex cross-shard ACID | High |
| **Cassandra Cluster** | High throughput | Ultra-high throughput | Tunable Consistency (Quorum) | High |

---

## 10. Interview Rapid Q&A Checklist
- *How do you eliminate cross-shard joins?* (Co-locate related entities on the same shard using a composite shard key, e.g., sharding both `Orders` and `OrderItems` by `user_id`).
- *What is Write Amplification?* (The ratio of bytes written to non-volatile storage relative to the bytes intended to be written by the application).
- *What is an Optimistic Lock in JPA?* (Using `@Version private Long version;` in an Entity. On UPDATE, Spring checks `WHERE id = ? AND version = ?`; if 0 rows update, it throws `OptimisticLockException` and retries).
