# 🏆 Module 12: Real-World System Design Interviews (From Easy to Advanced & AI)

> **Core Philosophy:** *A system design interview is not a presentation; it is an interactive engineering collaboration. Success requires navigating the universal 9-step framework while diving deep into scale bottlenecks, data models, and edge-case concurrency.*

---

## 📌 Table of Contents
1. [The Universal 9-Step System Design Interview Framework](#1-the-universal-9-step-system-design-interview-framework)
2. [Easy: URL Shortener (TinyURL)](#2-easy-url-shortener-tinyurl)
3. [Easy: Distributed Rate Limiter](#3-easy-distributed-rate-limiter)
4. [Medium: WhatsApp / WeChat Real-Time Chat System](#4-medium-whatsapp--wechat-real-time-chat-system)
5. [Medium: Distributed Notification Service (APNs, FCM, SMS, Email)](#5-medium-distributed-notification-service-apns-fcm-sms-email)
6. [Medium: Google Drive / Dropbox Chunked File Sync](#6-medium-google-drive--dropbox-chunked-file-sync)
7. [Advanced: Uber / Lyft Real-Time Geospatial Dispatch](#7-advanced-uber--lyft-real-time-geospatial-dispatch)
8. [Advanced: Twitter / X Timeline Fan-Out (The Celebrity Problem)](#8-advanced-twitter--x-timeline-fan-out-the-celebrity-problem)
9. [Advanced: Netflix Global Video Delivery & CDN Streaming](#9-advanced-netflix-global-video-delivery--cdn-streaming)
10. [AI Systems: Production RAG & Vector Search Architecture](#10-ai-systems-production-rag--vector-search-architecture)
11. [Master Interview Rapid Recall Checklist](#11-master-interview-rapid-recall-checklist)

---

## 1. The Universal 9-Step System Design Framework

```
┌───────────────┬────────┬────────────────────────────────────────────────────────┐
│ Step          │ Time   │ Deliverables                                           │
├───────────────┼────────┼────────────────────────────────────────────────────────┤
│ 1. Scope Reqs │ 05 Min │ 3-4 Core Functional Reqs, Latency SLA, Availability    │
│ 2. Estimation │ 05 Min │ DAU, Peak QPS, Daily Storage, Network Bandwidth        │
│ 3. API Design │ 05 Min │ REST / gRPC signatures, HTTP headers, payloads         │
│ 4. HLD Diagram│ 10 Min │ Client ──▶ CDN ──▶ LB ──▶ Gateway ──▶ App ──▶ DB/Cache │
│ 5. Data Model │ 05 Min │ Schemas, Primary Keys, SQL vs NoSQL, Shard Keys        │
│ 6. Bottlenecks│ 05 Min │ Identify SPOF, DB saturation, Lock contention          │
│ 7. Resilience │ 05 Min │ Add Circuit Breakers, Bulkheads, DLQs, Idempotency     │
│ 8. Trade-offs │ 03 Min │ Explain CAP choices, consistency vs latency            │
│ 9. Summary    │ 02 Min │ End-to-end walkthrough of a single user request        │
└───────────────┴────────┴────────────────────────────────────────────────────────┘
```

---

## 2. Easy: URL Shortener (TinyURL)

### A. Requirements & Scale Math
- **Scale:** 100M new URLs/month. Read-to-Write ratio = 10:1 (1 Billion reads/month).
- **Write QPS:** $100M / (30 	imes 86,400) pprox \mathbf{40	ext{ writes/sec}}$. Read QPS $pprox \mathbf{400	ext{ QPS}}$.
- **Storage:** 7 characters in Base62 ($[0-9a-zA-Z]$) $ightarrow 62^7 pprox \mathbf{3.5	ext{ Trillion unique URLs}}$.
- Record size $pprox 500	ext{ Bytes}$. 5-Year Storage $pprox 100M 	imes 12 	imes 5 	imes 500	ext{B} pprox \mathbf{3	ext{ TB}}$.

### B. High-Level Architecture & Redirect
```
[ User Browser ]
       │ 1. GET /aZ9x4Q
       ▼
[ Layer 7 Load Balancer ] ──▶ [ TinyURL Web Service ]
                                      │
              ┌───────────────────────┴───────────────────────┐
              ▼ (1. Check Cache)                              ▼ (2. Cache Miss: Query DB)
       [ Redis Cache ]                              [ Master / Replica PostgreSQL ]
              │                                               │
              └───────────────────┬───────────────────────────┘
                                  ▼
                     Return HTTP 302 Found
                     (Location: https://long-url.com/very/long/path)
```
- **301 vs 302:** Use **HTTP 302 Found** so requests pass through your server, enabling click analytics and tracking!

---

## 3. Medium: WhatsApp / WeChat Real-Time Chat System

### A. Core Architecture
```
[ Client A ] ──▶ [ WebSocket Gateway (Netty) ] ──▶ [ Chat Routing Service ]
                                                              │
                    ┌─────────────────────────────────────────┴────────────────┐
                    ▼ (User Online)                                            ▼ (User Offline)
  [ WebSocket Push directly to Client B ]                      [ Append to Cassandra DB ]
                                                               [ + Trigger Push Notification (APNs/FCM) ]
```
- **Database:** **Apache Cassandra / ScyllaDB** (Wide-Column NoSQL).
  - Sharded by `chat_id` (partition key).
  - Clustered by `message_id` (Snowflake 64-bit ID ordered by creation timestamp).
- **Online Presence:** Redis Set with heartbeats sent every 30 seconds (TTL: 60s).

---

## 4. Advanced: Uber / Lyft Real-Time Geospatial Dispatch

### A. Core Challenges
- 5 Million drivers report GPS coordinates every 4 seconds ($1,250,000	ext{ updates/sec}$).
- Rider requests a pickup: Find the 10 closest available drivers within a 3km radius.

### B. Geospatial Indexing Engine
- Traditional SQL `WHERE ST_DWithin(location, ...)` fails at 1M writes/sec!
- **Solution:** **Uber H3 (Hexagonal Hierarchical Spatial Index)** or **Google S2**.
  - Earth is divided into nested hexagonal cells.
  - Driver GPS coordinates $(	ext{lat}, 	ext{lng})$ are mapped to a 64-bit `H3Index` in memory.
  - Redis In-Memory Set: `H3_Cell_ID ──▶ Set of Driver IDs`.
  - Proximity search queries the rider's cell + its 6 immediate neighboring hexagons.

---

## 5. Advanced: Twitter / X Timeline Fan-Out (The Celebrity Problem)

### The Justin Bieber Dilemma:
- If an account with 100M followers tweets, **Write Fan-Out (Push)** requires inserting that tweet ID into 100M Redis inboxes. It takes minutes and overwhelms Redis memory!
- If an account has 10 followers, **Read Fan-Out (Pull)** wastes DB reads.

### The Hybrid Fan-Out Architecture:
```
USER POSTS A TWEET:
Is author a Celebrity? (> 100,000 followers)
  ├── NO  ──▶ WRITE FAN-OUT (Push): Insert tweet_id into all followers' Redis Timeline Lists.
  └── YES ──▶ READ FAN-OUT (Pull): Do NOT push! When a follower opens their app, fetch their
              pre-computed timeline from Redis and merge the celebrity's latest tweets on the fly!
```

---

## 6. AI Systems: Production RAG & Vector Search Architecture

```
DOCUMENT INGESTION PIPELINE (Offline / Asynchronous):
[ Enterprise PDFs / Docs ]
            │
            ▼ 1. Recursive Chunking (500 tokens, 50 token overlap)
[ Text Chunks ]
            │
            ▼ 2. Embedding Model (e.g. text-embedding-3-large via Triton)
[ 1536-Dimensional Float Vectors ]
            │
            ▼ 3. Ingest into Vector Database (Milvus / Pgvector / Pinecone)
            (Indexes vectors using Hierarchical Navigable Small World - HNSW)

QUERY & INFERENCE PIPELINE (Real-Time Online):
[ User Query: "What is our refund policy?" ]
            │
            ▼ 1. Convert Query to Vector Embedding
[ Query Vector ]
            │
            ▼ 2. Approximate Nearest Neighbors (ANN) Cosine Similarity Search
[ Top-5 Matching Document Chunks ]
            │
            ▼ 3. Augment System Prompt
[ Formatted Context + Question ] ──▶ [ LLM Serving Engine (vLLM / TensorRT-LLM) ]
                                                │
                                                ▼ 4. Stream Tokens via SSE
                                    [ User Client UI ]
```

---

## 7. Master Interview Rapid Recall Checklist
- **URL Shortener:** Base62 encoding, 302 redirect, Ticket server / Snowflake ID.
- **WhatsApp:** Netty WebSockets, Cassandra partitioned by `chat_id`, Redis heartbeat presence.
- **Uber:** Uber H3 hexagonal spatial indexing, in-memory Redis driver pools.
- **Twitter:** Hybrid push/pull fan-out, Redis home timelines.
- **Netflix:** Open Connect CDN appliances embedded in ISPs, adaptive bitrate streaming (HLS).
- **AI RAG:** Chunking, HNSW vector indexing (Pgvector/Milvus), context injection, vLLM serving.
