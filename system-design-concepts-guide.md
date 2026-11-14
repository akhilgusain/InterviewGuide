# System Design Concepts Guide (9+ YOE)

> Beyond "Design WhatsApp / Uber / Twitter" — the **building blocks** interviewers expect you to know and the **other kinds of questions** they ask.
> Every concept: simple explanation → when to use → trade-off → what to say in the interview.

---

## Table of Contents

1. [How to Approach Any System Design Question](#1-how-to-approach-any-system-design-question)
2. [Other Questions Commonly Asked (Beyond WhatsApp/Uber)](#2-other-questions-commonly-asked-beyond-whatsappuber)
3. [Fundamentals](#3-fundamentals)
4. [Networking & Communication](#4-networking--communication)
5. [Load Balancing & Traffic Management](#5-load-balancing--traffic-management)
6. [Caching](#6-caching)
7. [Databases & Storage](#7-databases--storage)
8. [Data Partitioning & Replication](#8-data-partitioning--replication)
9. [Distributed Systems Theory](#9-distributed-systems-theory)
10. [Messaging & Async Processing](#10-messaging--async-processing)
11. [Microservices Patterns](#11-microservices-patterns)
12. [Resilience & Fault Tolerance](#12-resilience--fault-tolerance)
13. [Probabilistic / Special Data Structures](#13-probabilistic--special-data-structures)
14. [Search, Geo & Real-time Concepts](#14-search-geo--real-time-concepts)
15. [Big Data & Analytics](#15-big-data--analytics)
16. [Security](#16-security)
17. [Observability](#17-observability)
18. [Deployment & Infrastructure](#18-deployment--infrastructure)
19. [Low-Level Design (LLD) Topics](#19-low-level-design-lld-topics)
20. [Back-of-the-Envelope Estimation](#20-back-of-the-envelope-estimation)
21. [Senior-Level Behavioural / Trade-off Questions](#21-senior-level-behavioural--trade-off-questions)
22. [Cheat Sheet: Problem → Concepts Used](#22-cheat-sheet-problem--concepts-used)
23. [Quick Revision Checklist](#23-quick-revision-checklist)

---

## 1. How to Approach Any System Design Question

Use the same 7 steps every time (45-60 min interview):

| Step | Time | What to do |
|------|------|-----------|
| 1. Clarify requirements | 5 min | Functional (what it does) + Non-functional (scale, latency, availability, consistency) |
| 2. Estimate scale | 3-5 min | Users, QPS, storage, bandwidth. Read-heavy or write-heavy? |
| 3. API design | 3-5 min | Core endpoints, request/response, pagination, idempotency |
| 4. Data model | 5 min | Entities, relationships, SQL vs NoSQL choice and why |
| 5. High-level design | 10 min | Boxes & arrows: client → LB → services → cache → DB → queue |
| 6. Deep dive | 15 min | 2-3 hard parts (the bottleneck, hot keys, consistency, fan-out) |
| 7. Trade-offs & wrap-up | 5 min | Failures, monitoring, scaling, what you'd improve |

**Senior-level expectations (9+ YOE):**
- You **drive** the discussion; don't wait to be asked.
- Every choice is justified with a **trade-off** ("I choose X over Y because…, cost is…").
- You think about **failure modes**, **operability** (monitoring, deployments), **cost**, and **evolution** (MVP → scale).
- Never say "it depends" without finishing the sentence.

**Non-functional requirements checklist:** scalability, availability, latency, consistency, durability, fault tolerance, security, cost, maintainability.

---

## 2. Other Questions Commonly Asked (Beyond WhatsApp/Uber)

You've done the "app clone" type. Interviewers also ask these categories:

### 2.1 Infrastructure / Building-block systems (VERY common for senior roles)
| Question | Key concepts tested |
|---|---|
| Design a **URL Shortener** (bit.ly) | Hashing, base62, ID generation, caching, DB choice, redirect 301 vs 302 |
| Design a **Rate Limiter** | Token bucket, sliding window, Redis, distributed counters, race conditions |
| Design a **Distributed Cache** (Redis/Memcached) | Consistent hashing, eviction, replication, hot keys |
| Design a **Key-Value Store** (DynamoDB) | Partitioning, replication, quorum, vector clocks, gossip, LSM tree |
| Design a **Distributed Message Queue** (Kafka/SQS) | Partitions, offsets, consumer groups, ordering, delivery semantics |
| Design a **Pub-Sub System** | Topics, fan-out, at-least-once, backpressure |
| Design a **Distributed Lock Service** (Zookeeper/Redlock) | Leases, fencing tokens, leader election |
| Design a **Unique ID Generator** (Snowflake) | Time-sortable IDs, clock skew, machine IDs |
| Design a **Job Scheduler / Cron at scale** | Delayed queue, time wheel, leader election, retries, idempotency |
| Design a **Distributed Task Queue** (Celery/Sidekiq) | Workers, retries, DLQ, priority |
| Design **API Gateway** | Auth, routing, throttling, aggregation |
| Design a **Load Balancer** | L4 vs L7, algorithms, health checks |
| Design a **Service Discovery / Config Service** | Registry, heartbeats, long-polling, versioning |
| Design a **Distributed File System** (HDFS/GFS) | Chunking, metadata server, replication |
| Design a **Blob/Object Storage** (S3) | Erasure coding, durability (11 nines), metadata separation |
| Design a **CDN** | Edge caching, cache invalidation, anycast, origin shield |
| Design a **Metrics / Monitoring System** (Prometheus/Datadog) | Time-series DB, aggregation, downsampling, alerting |
| Design a **Logging System** (ELK) | Ingestion pipeline, indexing, retention |
| Design a **Web Crawler** | BFS, URL frontier, politeness, dedup (bloom filter) |
| Design a **Notification Service** | Multi-channel, templates, retries, preferences, dedup |
| Design a **Email Service** | Queue, bounce handling, SPF/DKIM, throttling |
| Design an **Analytics / Click Counter** | Stream processing, aggregation windows, approximate counting |
| Design a **Feature Flag / A-B testing platform** | Config distribution, targeting rules, SDK caching |
| Design **Idempotent Payment System** | Idempotency keys, exactly-once effect, ledger, reconciliation |

### 2.2 Product / Application systems
| Question | Key concepts |
|---|---|
| **Twitter / News Feed / Instagram** | Fan-out on write vs read, timeline cache, celebrity problem |
| **YouTube / Netflix** | Video upload pipeline, transcoding, CDN, adaptive bitrate (HLS/DASH) |
| **Dropbox / Google Drive** | Chunking, delta sync, dedup, metadata DB, conflict resolution |
| **Google Docs (collaborative editing)** | OT / CRDT, WebSockets, versioning |
| **Typeahead / Autocomplete** | Trie, top-K, caching, offline aggregation |
| **Search Engine / Elasticsearch** | Inverted index, ranking (TF-IDF/BM25), sharding |
| **Ticket Booking (BookMyShow/Ticketmaster)** | Seat locking, double booking, TTL holds, optimistic/pessimistic locking |
| **Hotel/Flight Booking** | Inventory, overbooking, availability search, saga |
| **E-commerce (Amazon/Flipkart)** | Catalog, cart, inventory, order saga, flash sale |
| **Flash Sale / Inventory** | Hot key, queueing, Redis atomic decrement, oversell prevention |
| **Payment Gateway (Stripe/Razorpay)** | Idempotency, ledger, reconciliation, PCI, webhooks |
| **Digital Wallet / Banking (Paytm/GPay)** | Double-entry ledger, ACID, saga, consistency |
| **Food Delivery (Swiggy/Zomato)** | Geo search, matching, ETA, surge, order state machine |
| **Google Maps / Navigation** | Graph, Dijkstra/A*, contraction hierarchies, map tiles |
| **Proximity Service (Yelp/Nearby Friends)** | Geohash, quadtree, S2, H3 |
| **Leaderboard** | Redis sorted set, sharding, approximate rank |
| **Live Comments / Live Streaming** | WebSockets, SSE, fan-out, pub-sub |
| **Online Judge (LeetCode)** | Sandboxing, queue, workers, resource limits |
| **Stock Exchange / Trading** | Order book, matching engine, low-latency, sequencer |
| **Chess / Multiplayer Game backend** | State sync, matchmaking, WebSockets |
| **Video Conferencing (Zoom)** | WebRTC, SFU vs MCU, TURN/STUN |
| **Social Graph / Friends suggestions** | Graph DB, BFS, mutual friends |
| **Spotify / Music Streaming** | Chunked streaming, CDN, recommendation, offline sync |
| **Google Calendar / Meeting scheduler** | Time zones, recurring events, conflict detection |
| **Poll / Voting / Like counter** | Counters at scale, sharded counters, eventual consistency |
| **Ad Click Aggregation / Ad Serving** | Stream processing, exactly-once, auctions |
| **Recommendation System** | Offline + online, feature store, candidate generation + ranking |
| **Multi-tenant SaaS platform** | Tenant isolation, noisy neighbour, per-tenant sharding |
| **Hospital / School ERP style (enterprise)** | RBAC, audit, workflows, reporting |

### 2.3 Architectural / Conceptual questions (no "design X" — open discussion)
- Monolith vs microservices — when to migrate? How to do it safely (strangler fig)?
- How do you handle **distributed transactions**?
- How do you make an API **idempotent**?
- How do you design for **zero-downtime deployment** and **DB schema migration**?
- How do you scale a **read-heavy** vs **write-heavy** system?
- How would you handle **10x traffic** next month?
- Walk through a **production incident** you handled — how did you design to prevent recurrence?
- How do you handle **hot partition / hot key / celebrity problem**?
- How do you ensure **exactly-once** processing?
- Multi-region architecture: active-active vs active-passive, data residency.
- Cost optimization of a large cloud system.
- Data migration of a live system with no downtime.

### 2.4 Low-Level Design (OOP) — often in the same loop for Java devs
Parking lot, Elevator, Splitwise, LRU cache, Rate limiter (code), Logger framework, Vending machine, ATM, Chess, BookMyShow (class-level), Tic-Tac-Toe, Notification system (class-level), Snake & Ladder, Library management, In-memory file system, Pub-Sub (in-memory), Thread-safe bounded queue. See [section 19](#19-low-level-design-lld-topics).

---

## 3. Fundamentals

### 3.1 Scalability
**Simple words:** Ability of the system to handle more load by adding resources.

- **Vertical scaling (scale up):** bigger machine (more CPU/RAM). Simple, but has a ceiling, single point of failure, downtime on resize.
- **Horizontal scaling (scale out):** more machines. Needs stateless services + load balancer. Practically unlimited, but adds complexity (distributed state, consistency).

**Rule:** make app servers **stateless** (store session in Redis/JWT) so any server can serve any request.

### 3.2 Latency vs Throughput vs Bandwidth
- **Latency:** time for one request (ms). "How fast?"
- **Throughput:** requests/second (QPS/RPS). "How many?"
- **Bandwidth:** max data transfer rate (MB/s).
- Use **percentiles** (p50, p95, p99) — averages hide slow requests. **Tail latency** matters because one user request may call 10 services; the slowest dominates.

### 3.3 Availability vs Reliability
- **Availability:** % of time system is up. 99.9% = ~8.76 hrs downtime/year; 99.99% = ~52 min/year; 99.999% = ~5 min/year.
- **Reliability:** system does the *correct* thing over time.
- Availability in series multiplies (A × B); in parallel improves (1 − (1−A)(1−B)). Redundancy = availability.

### 3.4 SLA, SLO, SLI
- **SLI** = measured metric (e.g., p99 latency, error rate).
- **SLO** = internal target (p99 < 300 ms, 99.95% success).
- **SLA** = contract with customer, with penalties.
- **Error budget** = 100% − SLO; spend it on releases/experiments.

### 3.5 Stateless vs Stateful
Stateless service = no memory between requests → easy to scale/replace. State goes to DB/cache. Stateful (DBs, WebSocket servers) need sticky routing or replication.

### 3.6 Single Point of Failure (SPOF)
Any component whose failure kills the system. Remove by redundancy: multiple LBs, DB replicas, multi-AZ, multi-region.

### 3.7 Read-heavy vs Write-heavy
- Read-heavy (Twitter timeline, URL shortener) → cache, replicas, CDN, denormalize.
- Write-heavy (logging, IoT, chat) → queue buffering, sharding, LSM-tree DBs (Cassandra), batching.

---

## 4. Networking & Communication

### 4.1 What happens when you type a URL
DNS lookup → TCP handshake → TLS handshake → HTTP request → LB → server → response → render. Knowing this end-to-end is a common warm-up.

### 4.2 DNS
Translates domain → IP. Hierarchy: root → TLD → authoritative. Cached at browser/OS/ISP (TTL). Used for **geo-routing**, **failover**, **load balancing (round-robin DNS)**. Downside: TTL caching slows failover.

### 4.3 HTTP versions
- **HTTP/1.1:** one request per connection at a time (keep-alive helps); head-of-line blocking.
- **HTTP/2:** multiplexing many requests on one connection, header compression.
- **HTTP/3:** runs on QUIC (UDP); avoids TCP head-of-line blocking, faster handshakes.

### 4.4 TCP vs UDP
- **TCP:** reliable, ordered, connection-based (web, DB, APIs).
- **UDP:** fast, no guarantees (video calls, gaming, DNS, metrics).

### 4.5 REST vs gRPC vs GraphQL
| | REST | gRPC | GraphQL |
|---|---|---|---|
| Format | JSON/HTTP | Protobuf/HTTP2 | JSON, single endpoint |
| Strength | Simple, cacheable, universal | Fast, typed contracts, streaming | Client picks fields, avoids over/under-fetching |
| Weakness | Over/under-fetching | Not browser-friendly, harder to debug | Caching hard, N+1, query cost control |
| Use | Public APIs | Internal service-to-service | Mobile/BFF with varied clients |

### 4.6 Real-time communication options
| Technique | How | Use |
|---|---|---|
| **Short polling** | Client asks every N sec | Simple, wasteful |
| **Long polling** | Server holds request until data | Fallback, moderate |
| **SSE (Server-Sent Events)** | One-way server → client over HTTP | Live feeds, notifications |
| **WebSocket** | Full-duplex persistent connection | Chat, games, collaborative apps |
| **WebRTC** | Peer-to-peer media | Video calls |
| **Webhooks** | Server calls your URL on event | Payments, integrations |

**WebSocket scaling:** connections are stateful → keep a **connection registry** (user → server) in Redis, use pub-sub to route messages between servers, heartbeat/ping to detect dead connections.

### 4.7 API Design best practices
- **Pagination:** offset (simple, slow on deep pages) vs **cursor/keyset** (fast, stable). Prefer cursor for feeds.
- **Idempotency:** `Idempotency-Key` header; store key → response for N hours. Critical for payments/retries.
- **Versioning:** `/v1/`, header-based.
- **Rate limiting & quotas**, **bulk APIs**, **async APIs** (202 Accepted + status URL), **ETag** for conditional requests.
- HTTP verbs: GET/PUT/DELETE are idempotent; POST is not (unless you make it so).

### 4.8 Proxy vs Reverse Proxy
- **Forward proxy:** sits in front of clients (hides client identity, filtering).
- **Reverse proxy:** sits in front of servers (LB, TLS termination, caching, compression). Nginx, HAProxy, Envoy.

---

## 5. Load Balancing & Traffic Management

### 5.1 Load Balancer (LB)
Distributes traffic across servers; detects unhealthy ones (health checks).

- **L4 (transport):** routes by IP/port. Fast, dumb. (AWS NLB)
- **L7 (application):** routes by URL, headers, cookies. Smarter, can do TLS termination, path-based routing. (AWS ALB, Nginx)

### 5.2 Algorithms
| Algorithm | Idea | Good for |
|---|---|---|
| Round robin | Rotate | Equal servers |
| Weighted round robin | More to stronger servers | Mixed hardware |
| Least connections | Send to least busy | Long-lived connections |
| Least response time | Fastest server | Latency-sensitive |
| IP hash / consistent hash | Same client → same server | Sticky sessions, caches |
| Random (power of two choices) | Pick 2 random, choose less loaded | Large clusters, simple & effective |

### 5.3 Sticky sessions
Same user always to same server. Avoid if possible (hurts balance, failure loses state); use shared session store instead.

### 5.4 LB high availability
LB itself is a SPOF → run active-passive pair (floating IP / VRRP) or DNS across multiple LBs; cloud LBs are already redundant.

### 5.5 Global Server Load Balancing (GSLB) / Geo-routing
DNS or anycast routes users to nearest region; fail over if a region is down.

### 5.6 API Gateway
Single entry for clients: auth, rate limit, routing, request transformation, aggregation, TLS, logging. (Kong, AWS API Gateway, Spring Cloud Gateway). Don't put business logic here.

### 5.7 Rate Limiting (VERY common)
Protects from abuse/overload.

| Algorithm | How | Pros / Cons |
|---|---|---|
| **Token bucket** | Bucket refills at rate r; each request takes a token; allows bursts | Simple, burst-friendly (most popular) |
| **Leaky bucket** | Queue drained at fixed rate | Smooth output; no bursts |
| **Fixed window counter** | Count per minute | Simple; edge spike (2× at window boundary) |
| **Sliding window log** | Store timestamps | Accurate; memory heavy |
| **Sliding window counter** | Weighted blend of 2 windows | Good accuracy, low memory |

**Distributed implementation:** counters in **Redis** using `INCR` + `EXPIRE` or a **Lua script** (atomic) to avoid race conditions. Return `429 Too Many Requests` with `Retry-After`. Key by user ID / API key / IP. For ultra scale: local in-memory counters with periodic sync (approximate).

### 5.8 Backpressure & Load Shedding
- **Backpressure:** slow consumer tells producer to slow down (bounded queues, reactive streams).
- **Load shedding:** under overload, reject low-priority requests early to keep the system alive.

### 5.9 CDN (Content Delivery Network)
Caches static (and some dynamic) content at **edge locations** near users.
- **Push CDN:** you upload content to CDN (good for rarely changing files).
- **Pull CDN:** CDN fetches from origin on first miss (easiest).
- **Invalidation:** TTL, versioned URLs (`app.v123.js`), purge API.
- Benefits: lower latency, offloads origin, DDoS absorption.
- **Origin shield:** one mid-tier cache in front of origin to reduce miss storms.

---

## 6. Caching

### 6.1 Why & where
Cache = fast, small storage for frequently used data. Layers: browser → CDN → API gateway → app (local, e.g. Caffeine) → distributed cache (Redis) → DB buffer pool.

### 6.2 Caching strategies
| Strategy | Flow | Notes |
|---|---|---|
| **Cache-aside (lazy loading)** | App checks cache; on miss reads DB and fills cache | Most common; stale risk; app owns logic |
| **Read-through** | Cache itself loads from DB on miss | Cleaner app code |
| **Write-through** | Write to cache and DB together | Consistent, slower writes |
| **Write-behind (write-back)** | Write to cache, async flush to DB | Fast writes; data loss risk if cache dies |
| **Write-around** | Write to DB only; cache fills on read | Avoids polluting cache with write-once data |
| **Refresh-ahead** | Proactively refresh before expiry | Good for hot keys |

### 6.3 Eviction policies
LRU (least recently used – most common), LFU (least frequently used), FIFO, TTL, Random. Redis: `allkeys-lru`, `volatile-ttl`, etc.

### 6.4 Cache problems (interviewers LOVE these)
| Problem | What | Fix |
|---|---|---|
| **Cache stampede / thundering herd** | Hot key expires, thousands hit DB together | Request coalescing / single-flight, mutex lock, jittered TTL, refresh-ahead |
| **Cache penetration** | Requests for keys that **don't exist** always hit DB | Cache null values (short TTL), **Bloom filter** |
| **Cache avalanche** | Many keys expire at the same time / cache down | Random TTL jitter, multi-layer cache, replicas, circuit breaker |
| **Hot key** | One key gets huge traffic → one shard overloaded | Local in-process cache, replicate key (key#1..N), split key |
| **Stale data / inconsistency** | Cache ≠ DB | TTL, invalidate on write, CDC events, accept eventual consistency |
| **Big key** | Huge value blocks Redis | Split, compress |

### 6.5 Cache invalidation
"Two hard things in CS: cache invalidation and naming things." Options: TTL; delete on write (delete, don't update — avoids race); event-driven invalidation (CDC/Kafka); versioned keys.

### 6.6 Redis vs Memcached
Redis: rich data types (list, set, sorted set, hash, streams, pub/sub), persistence, replication, clustering, Lua. Memcached: simple KV, multithreaded, pure cache.

---

## 7. Databases & Storage

### 7.1 SQL vs NoSQL
| | SQL (RDBMS) | NoSQL |
|---|---|---|
| Model | Tables, joins, schema | KV, document, column, graph |
| Consistency | ACID | Often BASE / tunable |
| Scaling | Vertical + read replicas; sharding is hard | Built to scale horizontally |
| Use | Money, orders, relationships, complex queries | Massive scale, flexible schema, simple access patterns |

**Don't say "NoSQL is faster"** — say *why*: "access pattern is key-based, need horizontal write scale, schema evolves".

### 7.2 NoSQL types
| Type | Examples | Use |
|---|---|---|
| Key-Value | Redis, DynamoDB, Riak | Sessions, cache, carts |
| Document | MongoDB, Couchbase | Catalogs, profiles, CMS |
| Wide-column | Cassandra, HBase, Bigtable | Time-series, messaging, high write |
| Graph | Neo4j, Neptune | Social graph, fraud, recommendations |
| Time-series | InfluxDB, TimescaleDB, Prometheus | Metrics, IoT |
| Search | Elasticsearch, OpenSearch, Solr | Full-text, logs |
| Object store | S3, GCS, Azure Blob | Images, videos, backups |

### 7.3 ACID
- **Atomicity** – all or nothing.
- **Consistency** – DB moves valid state → valid state.
- **Isolation** – concurrent transactions don't interfere.
- **Durability** – committed data survives crashes (WAL).

### 7.4 Isolation levels & anomalies
| Level | Dirty read | Non-repeatable read | Phantom |
|---|---|---|---|
| Read Uncommitted | ✅ possible | ✅ | ✅ |
| Read Committed | ❌ | ✅ | ✅ |
| Repeatable Read | ❌ | ❌ | ✅ (MySQL InnoDB mostly prevents) |
| Serializable | ❌ | ❌ | ❌ |

Also know: **lost update**, **write skew**, **MVCC** (readers don't block writers, via row versions).

### 7.5 Concurrency control
- **Pessimistic locking:** `SELECT … FOR UPDATE` – lock first. Safe, lower concurrency. Use for high contention (ticket seat).
- **Optimistic locking:** version column; update `WHERE version = ?`; retry on conflict. Use for low contention (`@Version` in JPA).
- **Distributed lock:** Redis (`SET NX PX`), Zookeeper, DB row lock.

### 7.6 Indexing
- **B-Tree / B+Tree:** default; range queries, sorted; read-optimized.
- **Hash index:** equality only.
- **Composite index:** order matters (leftmost prefix).
- **Covering index:** index contains all selected columns → no table lookup.
- **Inverted index:** word → documents (search).
- **Cost:** slower writes, more storage. Index only what you query.

### 7.7 B-Tree vs LSM-Tree (storage engines)
- **B-Tree (MySQL, Postgres):** update in place; good reads; random write I/O.
- **LSM-Tree (Cassandra, RocksDB, HBase):** writes go to memtable → flushed as sorted immutable SSTables → background **compaction**. Excellent write throughput; reads may check multiple files (use **Bloom filters** to skip).

### 7.8 Normalization vs Denormalization
Normalize for integrity/less duplication (OLTP). Denormalize for read speed (avoid joins), common in NoSQL & feed systems. Cost: duplicate data → must keep in sync.

### 7.9 OLTP vs OLAP
- **OLTP:** many small transactions (orders). Row-store.
- **OLAP:** big analytical queries (reports). Column-store (Redshift, BigQuery, ClickHouse, Snowflake).

### 7.10 Storage types
- **Block** (EBS, disks): databases, VMs.
- **File** (NFS, EFS): shared file system.
- **Object** (S3): cheap, durable, flat, HTTP API — images, videos, backups, data lakes. Store **metadata in DB, blob in S3**, serve via CDN; use **pre-signed URLs** for direct upload/download.

### 7.11 Storing large files
Chunk (e.g., 4 MB), upload in parallel, resumable (multipart), hash each chunk for **dedup** and integrity, metadata DB maps file → chunks.

### 7.12 Connection pooling
Opening DB connections is expensive; pool (HikariCP). Too many app instances × pool size can exhaust DB → use PgBouncer/proxy.

---

## 8. Data Partitioning & Replication

### 8.1 Replication
Copies of data on multiple nodes for availability + read scaling.

| Mode | Description | Trade-off |
|---|---|---|
| **Leader–Follower (primary–replica)** | Writes to leader; followers replicate | Simple; replication lag; failover needed |
| **Multi-leader** | Several nodes accept writes (multi-region) | Write conflicts → need resolution (LWW, CRDT) |
| **Leaderless (Dynamo-style)** | Any node accepts writes; quorum R+W>N | High availability; eventual consistency; read-repair |

- **Sync replication:** durable & consistent, slower, blocks if replica down.
- **Async replication:** fast, may lose recent writes on leader crash; **replication lag** → stale reads.
- **Semi-sync:** at least one replica confirms.

**Fixing replication lag problems:** read-your-writes (read from leader for recent writers), monotonic reads (stick user to one replica), consistent prefix reads.

### 8.2 Sharding (Partitioning)
Splitting data across nodes so each holds a subset.

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Range** | `A–M`, `N–Z`, by date | Range queries easy | Hot spots (latest date) |
| **Hash** | `hash(key) % N` | Even distribution | Range queries hard; resharding moves lots of data |
| **Consistent hashing** | Hash ring | Minimal data movement on add/remove | Needs virtual nodes for balance |
| **Directory / lookup** | Lookup table key → shard | Flexible | Lookup is SPOF/bottleneck |
| **Geo** | By region | Latency, compliance | Uneven load |

**Shard key choice is critical:** high cardinality, even spread, matches query pattern. Bad key → **hot shard**.

**Problems with sharding:** cross-shard joins, cross-shard transactions, resharding, hot keys, global unique IDs, secondary indexes (local vs global index).

### 8.3 Consistent Hashing (must know)
Nodes and keys placed on a ring (0..2³²). A key goes to the next node clockwise. Add/remove node → only neighbors' keys move (~K/N). **Virtual nodes** give each physical node many ring points → even load, better for heterogeneous machines. Used in: DynamoDB, Cassandra, Memcached clients, CDN, LB.

### 8.4 Hot partition / Celebrity problem
One key/shard gets disproportionate traffic. Fixes: add random suffix to key (key#0-9) and aggregate on read, cache aggressively, dedicated shard for VIPs, hybrid fan-out (push for normal users, pull for celebrities).

### 8.5 Rebalancing
Fixed number of partitions > nodes (move whole partitions), dynamic splitting (HBase), proportional to nodes (Cassandra). Avoid `hash % N`.

### 8.6 Unique ID generation in distributed systems
| Approach | Notes |
|---|---|
| Auto-increment | Single DB bottleneck; guessable |
| UUID v4 | No coordination; 128-bit, random → bad index locality |
| UUID v7 / ULID | Time-ordered; index friendly |
| **Snowflake** | 64 bits = 1 sign + 41 timestamp + 10 machine + 12 sequence → ~4096 IDs/ms/machine, time-sortable. Risk: clock skew/backwards |
| DB ticket server (Flickr) | Few DBs with odd/even steps |
| Range allocation | Service hands out blocks of IDs (e.g., 1000 at a time) |

### 8.7 Data Locality / Multi-region
- **Active-passive:** standby region; simpler, failover takes time (RTO).
- **Active-active:** all regions serve; conflict resolution & global consistency issues.
- **RPO** (data loss tolerated) & **RTO** (time to recover) drive the design.
- **Data residency/GDPR** may force data to stay in region.

---

## 9. Distributed Systems Theory

### 9.1 CAP Theorem
In a network **P**artition, you must choose **C**onsistency or **A**vailability.
- **CP:** refuse/err during partition to keep data correct (HBase, MongoDB default, Zookeeper, etcd). Use: banking, inventory.
- **AP:** keep serving, possibly stale (Cassandra, DynamoDB, CouchDB). Use: social feeds, carts.
- Partitions *will* happen, so real choice is C vs A **during** failure.
- Common mistake: "pick 2 of 3". Correct: "P is unavoidable; trade C vs A".

### 9.2 PACELC
Extends CAP: if **P**artition → A or C; **E**lse (normal) → **L**atency or **C**onsistency. Even without failures, strong consistency costs latency. (DynamoDB: PA/EL; Spanner: PC/EC.)

### 9.3 Consistency models (strong → weak)
1. **Linearizability (strong):** behaves like a single copy; reads see latest write.
2. **Sequential consistency:** all see operations in same order.
3. **Causal consistency:** causally related ops are ordered.
4. **Read-your-writes / Session consistency.**
5. **Eventual consistency:** replicas converge if no new writes.

Pick the weakest that is still correct for the business case (likes count → eventual; bank balance → strong).

### 9.4 Quorum (Dynamo-style)
N replicas, W write acks, R read acks. If **R + W > N**, reads overlap the latest write → strong-ish consistency. Common: N=3, W=2, R=2. W=1,R=N → fast writes; R=1,W=N → fast reads.

### 9.5 Consensus: Paxos / Raft
How nodes agree on a value despite failures. **Raft:** leader election + log replication; commit when majority (quorum) acks. Needs odd node count (3/5). Used by etcd, Consul, Kafka KRaft. Zookeeper uses ZAB. Tolerates f failures with 2f+1 nodes.

### 9.6 Leader Election
Needed for single-writer, schedulers, shard owners. Approaches: Raft/Zookeeper ephemeral nodes/etcd lease/Redis lock. Beware **split brain** (two leaders) → use **fencing tokens** / epochs.

### 9.7 Distributed Locks
- Redis `SET key value NX PX 30000` + unique value, release with Lua compare-and-delete. **Redlock** across 5 nodes (debated).
- Problem: client pauses (GC) past lease expiry, then writes while another holds lock → use **fencing token** (monotonic number checked by the resource).
- For strong correctness use Zookeeper/etcd.

### 9.8 Time & ordering
- Physical clocks drift; NTP isn't perfect → don't rely on wall-clock for ordering.
- **Lamport clock:** logical counter for happened-before.
- **Vector clock:** detects concurrent updates (Dynamo).
- **Hybrid Logical Clock (HLC)**, Google **TrueTime** (Spanner) – bounded uncertainty.
- **Last-Write-Wins (LWW):** simple but can lose data.

### 9.9 Gossip protocol
Nodes periodically share state with random peers; info spreads exponentially. Used for membership, failure detection (Cassandra, Consul). Scalable, eventually consistent.

### 9.10 Heartbeats & failure detection
Periodic "I'm alive"; missed N beats → suspect. Phi-accrual detector adapts to network. Can't distinguish slow from dead → design for false positives.

### 9.11 Conflict resolution
LWW, version vectors + app merge, **CRDTs** (Conflict-free Replicated Data Types: counters, sets—merge automatically), **Operational Transformation** (Google Docs).

### 9.12 Merkle trees / Anti-entropy
Hash tree to compare replica data efficiently — find divergent ranges by comparing hashes top-down. Used in Cassandra, Dynamo, Git, blockchains.

### 9.13 Hinted handoff & read repair
- **Hinted handoff:** if target replica down, another node stores the write temporarily and forwards later.
- **Read repair:** on read, detect stale replica and fix it.

### 9.14 Fallacies of distributed computing
Network is reliable? Latency zero? Bandwidth infinite? Network secure? Topology fixed? One admin? Transport cost zero? Network homogeneous? — all **false**. Assume failure everywhere.

### 9.15 Idempotency & delivery semantics
- **At-most-once:** may lose, never duplicate.
- **At-least-once:** never lose, may duplicate → make consumers **idempotent** (dedupe by message ID).
- **Exactly-once:** practically = at-least-once + idempotent processing (or transactional/atomic commit, e.g., Kafka transactions). Be careful claiming true exactly-once.

---

## 10. Messaging & Async Processing

### 10.1 Why queues?
Decouple services, absorb spikes (buffer), async work, retries, scale consumers independently.

### 10.2 Queue vs Pub/Sub vs Log
| Model | Behavior | Example |
|---|---|---|
| **Point-to-point queue** | Each message consumed by one consumer; deleted after ack | SQS, RabbitMQ |
| **Pub/Sub** | Each subscriber gets a copy | SNS, Redis Pub/Sub, Google Pub/Sub |
| **Distributed log** | Append-only, replayable, consumers track offset | Kafka, Kinesis, Pulsar |

### 10.3 Kafka essentials
- **Topic → partitions** (unit of parallelism & ordering). Ordering guaranteed **only within a partition** → use a key (e.g., orderId) to route related events to the same partition.
- **Consumer group:** each partition consumed by one consumer in group; scale consumers ≤ partitions.
- **Offset commit** controls delivery semantics. **Replication factor** + `acks=all` + `min.insync.replicas` for durability.
- **Retention:** keep by time/size; **log compaction** keeps latest value per key.
- Fast because: sequential disk I/O, page cache, zero-copy, batching.
- **Rebalancing** pauses consumers; **lag** = key health metric.

### 10.4 Dead Letter Queue (DLQ) & retries
Failed messages → retry with **exponential backoff + jitter** → after N attempts go to **DLQ** for inspection/replay. Avoid **poison pill** blocking the queue.

### 10.5 Ordering
Global order doesn't scale. Guarantee per-key order via partitioning by key. Parallel consumers on same key break order — use single consumer per partition/key.

### 10.6 Outbox Pattern (VERY important)
**Problem:** update DB **and** publish event atomically (dual-write problem: one can fail).
**Solution:** in the same DB transaction, write business row + row in `outbox` table. A relay (poller or **CDC/Debezium**) publishes outbox rows to Kafka. Consumers are idempotent. Gives at-least-once with no lost events.

### 10.7 Change Data Capture (CDC)
Stream DB changes from transaction log (binlog/WAL) → Kafka. Use for cache invalidation, search index sync, data warehouse, event sourcing-lite.

### 10.8 Event-driven architecture
- **Event notification:** "OrderPlaced" (small, others fetch details).
- **Event-carried state transfer:** event has full data.
- **Event sourcing:** store *events* as source of truth; state = replay of events. Pros: audit, time travel. Cons: complexity, schema evolution, eventual consistency.
- **CQRS:** separate write model and read model(s) (often with event sourcing). Reads served from denormalized views.

### 10.9 Batch vs Stream processing
- **Batch:** process large data periodically (Spark, MapReduce).
- **Stream:** process continuously (Flink, Kafka Streams, Spark Streaming). Concepts: windows (tumbling, sliding, session), watermarks (late data), stateful ops, checkpointing.
- **Lambda architecture:** batch + speed layer. **Kappa:** stream only.

### 10.10 Fan-out
- **Fan-out on write (push):** on post, write to all followers' timelines. Fast reads; expensive for celebrities.
- **Fan-out on read (pull):** compute timeline at read time. Cheap writes; slow reads.
- **Hybrid:** push for normal users, pull for celebrities, merge on read.

### 10.11 Scheduling / delayed jobs
Options: DB polling with `run_at` index, Redis sorted set (score = timestamp), delayed queue (SQS delay, RabbitMQ TTL+DLX), **hierarchical timing wheel**, Quartz with cluster lock. Need leader election or row-level lock (`SELECT … FOR UPDATE SKIP LOCKED`) so jobs aren't run twice.

---

## 11. Microservices Patterns

### 11.1 Monolith vs Microservices
| | Monolith | Microservices |
|---|---|---|
| Deploy | One unit | Independently |
| Complexity | In code | In network/ops |
| Scale | Whole app | Per service |
| Data | One DB | DB per service |
| Good for | Early-stage, small teams | Large orgs, independent scaling |

**Advice:** start with a **modular monolith**; split when team/scale demands. Microservices cost: network failures, distributed transactions, observability, versioning.

### 11.2 Decomposition
By **business capability** / **DDD bounded context**, not by technical layer. **Strangler Fig** pattern to migrate incrementally (route traffic piece by piece to new services).

### 11.3 Database per service
Each service owns its data; others access via API/events only. Avoid shared DB (tight coupling). Reporting → CDC to data warehouse or CQRS views.

### 11.4 Distributed transactions
Two-phase commit (2PC) is blocking and doesn't scale → use:

**Saga pattern:** sequence of local transactions; on failure run **compensating transactions**.
- **Choreography:** services react to each other's events. Simple, but hard to follow with many steps.
- **Orchestration:** central orchestrator tells each service what to do. Clear flow, easier to debug (Temporal, Camunda, Step Functions).
- Example: Order → reserve inventory → charge payment → ship. If payment fails → release inventory → cancel order.
- Saga is **eventually consistent**; handle semantic locks, idempotency, and out-of-order messages.

**2PC / XA:** coordinator asks "prepare?" then "commit". Strong consistency, but blocking if coordinator dies, poor availability. **TCC** (Try-Confirm-Cancel) is an app-level variant.

### 11.5 Service Discovery
Services find each other dynamically: **client-side** (Eureka + Ribbon) or **server-side** (Kubernetes Service/DNS, LB). Registry with heartbeats.

### 11.6 Service Mesh
Sidecar proxy (Envoy) handles mTLS, retries, timeouts, circuit breaking, tracing, traffic splitting (Istio, Linkerd) — keeps that logic out of app code.

### 11.7 Other patterns
- **API Gateway / BFF (Backend for Frontend)**
- **Sidecar**, **Ambassador**, **Anti-corruption layer**
- **Config server** / externalized config
- **Bulkhead**, **Circuit breaker**, **Retry**, **Timeout** (see §12)
- **Aggregator / Composite service**
- **Claim check:** put big payload in blob store, pass reference in message.
- **Transactional outbox**, **Inbox (dedupe)**
- **Leader election**, **Sharding**, **Scatter-gather**
- **Event sourcing**, **CQRS**
- **Shadow traffic / Dark launch**, **Canary**, **Blue-green**

### 11.8 Inter-service communication
- **Sync (REST/gRPC):** simple, but coupling & cascading failure. Use timeouts + circuit breaker.
- **Async (events/queue):** decoupled, resilient, but eventual consistency & harder debugging.
- Rule of thumb: **commands/queries sync where user waits; side effects async**.

---

## 12. Resilience & Fault Tolerance

### 12.1 Timeouts
Always set them (connect + read). No timeout = threads pile up → outage. Budget timeouts down the call chain (client 3s > service A 2s > service B 1s).

### 12.2 Retries
Retry only **idempotent** ops, on transient errors, with **exponential backoff + jitter**, capped attempts. Beware **retry storms / amplification** (3 layers × 3 retries = 27×). Use a **retry budget**.

### 12.3 Circuit Breaker
States: **Closed** (normal) → **Open** (fail fast after error threshold) → **Half-open** (probe a few requests) → Closed. Prevents cascading failure and gives the downstream time to recover. (Resilience4j in Java.) Pair with a **fallback** (cached/default response).

### 12.4 Bulkhead
Isolate resources (thread pools/connection pools) per dependency so one slow dependency can't consume all threads. Like ship compartments.

### 12.5 Fallback & Graceful degradation
If recommendation service is down, show popular items. Core flow must work; non-critical features degrade.

### 12.6 Rate limiting, load shedding, backpressure
See §5.7–5.8.

### 12.7 Idempotency (critical)
Same request repeated = same effect. Implement with idempotency key + unique constraint / dedupe table; return stored response. Needed for payments, order creation, message consumers, retries.

### 12.8 Redundancy & Failover
Multi-AZ, multi-region, replicas, health checks, automatic failover. **Chaos engineering** (Netflix Chaos Monkey) proves it works.

### 12.9 Disaster Recovery
Backup/Restore (cheapest, slow) → Pilot light → Warm standby → Multi-site active-active (fastest, costly). Define **RPO/RTO**. Test restores regularly.

### 12.10 Graceful shutdown & health checks
- **Liveness** (restart me?) vs **readiness** (send me traffic?) probes.
- Drain connections on shutdown, finish in-flight requests.

### 12.11 Cascading failure & metastable failure
Overload → timeouts → retries → more overload. Break the loop with limits, shedding, breakers, autoscaling with warm-up.

### 12.12 Thundering herd
Many clients retry/refresh simultaneously after an outage. Use jitter, staggered restarts, request coalescing.

---

## 13. Probabilistic / Special Data Structures

| Structure | What it does | Use case |
|---|---|---|
| **Bloom filter** | "Definitely not present" or "maybe present"; no false negatives, some false positives; no deletes (use counting bloom) | Skip DB lookups, crawler dedup, cache penetration, LSM reads, username availability |
| **Count-Min Sketch** | Approximate frequency counts in small memory | Top-K / heavy hitters, trending, rate limiting |
| **HyperLogLog** | Approximate **unique count** (cardinality) in ~12 KB with ~1% error | Unique visitors, Redis `PFADD/PFCOUNT` |
| **Trie** | Prefix tree | Autocomplete, spell check |
| **Inverted index** | word → doc list | Search |
| **Skip list / Sorted set** | Ordered with O(log n) | Redis leaderboards |
| **Geohash / Quadtree / S2 / H3** | Spatial indexing | Nearby search (Uber, Yelp) |
| **Merkle tree** | Hash tree for diffing | Replica sync, Git |
| **Consistent hash ring** | Key→node mapping | Sharding/caching |
| **LRU (HashMap + Doubly Linked List)** | O(1) get/put eviction | Caches |
| **Heap / Priority queue** | Top-K, scheduling | Timers, top-K merge |
| **Segment tree / Fenwick tree** | Range queries/updates | Analytics counters |
| **Ring buffer** | Fixed-size circular queue | Logs, LMAX Disruptor |
| **Timing wheel** | Efficient timers at scale | Delayed jobs, TCP timeouts |
| **Rope / CRDT / OT structures** | Collaborative text | Google Docs |

**Top-K pattern (trending hashtags):** stream → Count-Min Sketch per window + min-heap of size K → merge across partitions → store result in Redis.

---

## 14. Search, Geo & Real-time Concepts

### 14.1 Full-text search
- **Inverted index:** term → postings list (doc IDs, positions).
- **Pipeline:** tokenize → lowercase → stopwords → stemming → index.
- **Ranking:** TF-IDF, **BM25**, plus signals (recency, popularity, personalization).
- **Elasticsearch architecture:** index → shards (primary + replicas); query = scatter-gather across shards then merge top results.
- **Sync strategy:** DB is source of truth; push changes to ES via CDC/Kafka (eventual consistency). Never use ES as the primary store for critical data.
- **Typeahead:** trie or prefix index; cache top suggestions per prefix; update offline from query logs; return < 100 ms; client-side debounce.

### 14.2 Geospatial
| Technique | Idea |
|---|---|
| **Geohash** | Encode lat/lng into string; shared prefix = nearby. Query cell + 8 neighbors. Edge-case at cell boundaries. |
| **Quadtree** | Recursively split map into 4 until cell has ≤N points; adaptive to density. |
| **S2 (Google) / H3 (Uber)** | Hierarchical cells on sphere / hexagons; uniform neighbor distance (H3). |
| **R-tree / PostGIS / Redis GEO** | DB-level spatial indexes. |

Uber-style: drivers send location every ~4 s → in-memory store (Redis GEO / sharded by cell) → matching service queries nearby cells. Don't write every ping to a SQL DB.

### 14.3 Real-time presence ("online/last seen")
Heartbeat every N sec → set key with TTL in Redis; expired = offline. For fan-out to friends, use pub-sub and only for those currently viewing.

### 14.4 Collaborative editing
- **OT (Operational Transformation):** transform concurrent operations (Google Docs); central server orders them.
- **CRDT:** data structure designed so concurrent edits merge deterministically; works peer-to-peer/offline (Figma, Notion-like).

### 14.5 Video streaming
Upload → store raw in object storage → **transcode** into multiple resolutions/bitrates/codecs (parallel DAG of tasks via queue) → segments (HLS/DASH, 2-10 s chunks) → CDN. Client does **adaptive bitrate (ABR)** — switches quality based on bandwidth. Thumbnails, moderation, DRM, resumable uploads.

### 14.6 Recommendation (high-level)
Offline: collect events → feature store → train models. Online: **candidate generation** (thousands) → **ranking** (hundreds) → **re-ranking** (business rules) → serve cached. Cold start problem, A/B testing.

### 14.7 Leaderboard
Redis **sorted set**: `ZADD`, `ZREVRANGE`, `ZRANK` — O(log n). For huge scale: shard by score range / user-bucket; approximate ranks for the long tail; top-N exact.

### 14.8 Counters at scale (likes, views)
Single hot row is a bottleneck → **sharded counters** (N sub-counters, sum on read), buffer increments in Redis / Kafka and batch-write, accept eventual consistency, use HyperLogLog for uniques.

### 14.9 Ticket / seat booking (double-booking prevention)
- Temporary **hold** with TTL (Redis key or DB `held_until`) → user pays within e.g. 10 min → confirm; else release.
- Prevent double-sell with DB unique constraint `(show_id, seat_id)` or `SELECT FOR UPDATE` / optimistic version.
- Flash traffic → **virtual waiting room / queue** in front.

### 14.10 Payments (very commonly asked at 9 YOE)
- **Idempotency key** on every payment request.
- **State machine:** CREATED → PENDING → SUCCESS/FAILED → REFUNDED.
- **Double-entry ledger:** every transaction has debit + credit rows that sum to zero; immutable, append-only.
- **Webhooks** from PSP + **reconciliation job** (daily compare internal ledger vs bank/PSP files).
- Handle timeouts: status unknown → query PSP, never blindly retry charge without idempotency.
- Use **saga** across order/payment/inventory; **outbox** for events.
- Security: PCI-DSS (tokenize cards, never store CVV), encryption, audit logs.

---

## 15. Big Data & Analytics

- **Data lake** (raw, cheap: S3) vs **Data warehouse** (structured, fast SQL: Redshift/BigQuery/Snowflake) vs **Lakehouse** (Delta/Iceberg).
- **ETL vs ELT:** transform before vs after loading.
- **MapReduce:** map (parallel process) → shuffle → reduce. Spark = in-memory successor.
- **Columnar formats:** Parquet/ORC — read only needed columns, great compression.
- **Partitioning** (by date) and **bucketing** speed up scans.
- **Stream processing:** Flink/Kafka Streams; **exactly-once** via checkpoints + transactional sinks.
- **Windowing:** tumbling (fixed, non-overlapping), sliding (overlapping), session (activity gap). **Watermarks** handle late events.
- **Ad click aggregation example:** clicks → Kafka (partition by adId) → Flink windowed count → OLAP store (ClickHouse/Druid) ; raw events kept in S3 for reprocessing; reconcile batch vs stream.
- **Feature store**, **data catalog**, **data lineage**, **schema registry** (Avro/Protobuf, schema evolution with backward/forward compatibility).

---

## 16. Security

### 16.1 AuthN vs AuthZ
Authentication = who are you. Authorization = what can you do.

### 16.2 Session vs Token
- **Session cookie:** server stores state (Redis); easy revoke; needs shared store.
- **JWT:** self-contained signed token; stateless verify; hard to revoke → short expiry + refresh token + denylist.

### 16.3 OAuth 2.0 & OIDC
OAuth2 = delegated **authorization** (Authorization Code + PKCE for apps, Client Credentials for service-to-service). **OIDC** adds identity (ID token) on top → "Login with Google". **SSO** via SAML/OIDC.

### 16.4 Access control
**RBAC** (roles), **ABAC** (attributes/policies), **ACL**. Principle of least privilege. Check authorization in every service (don't trust only gateway).

### 16.5 Transport & data protection
- **TLS** everywhere; **mTLS** between services.
- **Encryption at rest** (KMS), field-level encryption for PII, **hashing passwords** with bcrypt/argon2 (+ salt), never reversible.
- **Secrets** in Vault/Secrets Manager, not in code/config.

### 16.6 Common attacks & defenses
| Attack | Defense |
|---|---|
| SQL injection | Prepared statements |
| XSS | Output encoding, CSP |
| CSRF | CSRF tokens, SameSite cookies |
| DDoS | CDN/WAF, rate limiting, autoscaling, anycast |
| Brute force | Rate limit, lockout, MFA |
| IDOR | Authorization checks on object IDs |
| SSRF | Allow-list outbound calls |
| Replay | Nonce, timestamps, idempotency |

### 16.7 Other
Audit logging, API keys rotation, WAF, zero trust, multi-tenancy isolation, GDPR (right to be forgotten → delete across caches, backups, logs, search indexes).

---

## 17. Observability

**Three pillars:**
- **Logs:** structured (JSON), with correlation/trace ID; centralized (ELK/Loki).
- **Metrics:** numeric time-series (Prometheus/Grafana/Datadog). Use **RED** (Rate, Errors, Duration) for services, **USE** (Utilization, Saturation, Errors) for resources, plus **Golden Signals** (latency, traffic, errors, saturation).
- **Traces:** follow a request across services (OpenTelemetry, Jaeger, Zipkin) via propagated trace-ID.

Also: **alerting** on SLO burn rate (not every blip), **dashboards**, **health checks**, **synthetic monitoring**, **on-call runbooks**, **post-mortems (blameless)**.

**Designing a metrics system:** agents push/pull → Kafka → time-series DB (partition by metric + time) → downsampling (1s→1m→1h) → retention tiers → query + alert engine.

---

## 18. Deployment & Infrastructure

### 18.1 Deployment strategies
| Strategy | How | Rollback |
|---|---|---|
| **Rolling** | Replace instances gradually | Slow-ish |
| **Blue-Green** | Two envs; flip traffic | Instant |
| **Canary** | 1% → 10% → 100% with metrics gate | Fast, safest |
| **Shadow** | Copy prod traffic to new version, ignore response | N/A |
| **Feature flags** | Deploy code dark, enable gradually | Instant toggle |

### 18.2 Zero-downtime DB migrations
**Expand → Migrate → Contract:** add new nullable column (backward-compatible) → deploy code writing both → backfill → switch reads → drop old column. Never rename/drop in one step. Tools: Flyway/Liquibase; online schema change (gh-ost, pt-osc).

### 18.3 Containers & orchestration
Docker (package), Kubernetes (schedule, self-heal, scale, rolling updates, service discovery, config/secrets). HPA autoscaling on CPU/custom metrics. Requests/limits, readiness/liveness probes, PodDisruptionBudgets.

### 18.4 Autoscaling
Reactive (CPU/queue depth) vs scheduled vs predictive. Warm-up time matters; scale on **queue lag** for consumers. Set min/max to control cost.

### 18.5 Infrastructure as Code & CI/CD
Terraform/CloudFormation; pipelines build → test → scan → deploy; immutable artifacts; GitOps.

### 18.6 Cloud region / AZ concepts
Region (geography) contains multiple AZs (isolated data centers). Multi-AZ for HA; multi-region for DR/latency. Cross-AZ/region traffic costs money and adds latency.

### 18.7 Cost awareness (senior signal)
Cache to reduce DB size, tiered storage (hot/warm/cold; S3 lifecycle), spot instances for batch, right-sizing, reduce cross-region traffic, compress data, TTL old data.

---

## 19. Low-Level Design (LLD) Topics

Often asked in Java loops. Know **SOLID**, **design patterns**, and write clean code with interfaces.

### 19.1 SOLID in one line each
- **S**ingle Responsibility – one reason to change.
- **O**pen/Closed – extend without modifying.
- **L**iskov – subtypes substitutable.
- **I**nterface Segregation – small focused interfaces.
- **D**ependency Inversion – depend on abstractions.

### 19.2 Patterns most useful in LLD
| Pattern | Typical use |
|---|---|
| **Strategy** | Pricing, payment methods, rate-limit algorithms, parking fee |
| **Factory / Abstract Factory** | Create vehicles, notifications by channel |
| **Singleton** | Config, logger (thread-safe: enum / holder idiom) |
| **Observer** | Notifications, pub-sub, stock tickers |
| **State** | Elevator, vending machine, order lifecycle, ATM |
| **Command** | Undo/redo, job queues |
| **Decorator** | Add features (toppings, logging, caching wrappers) |
| **Builder** | Complex object creation |
| **Chain of Responsibility** | Validation, logging levels, approval flows |
| **Template Method** | Common workflow with variable steps |
| **Adapter / Facade / Proxy** | Integrate 3rd-party, simplify API, lazy/caching |

### 19.3 Typical LLD problems
Parking Lot, Elevator, Vending Machine, ATM, Splitwise, LRU/LFU Cache, Rate Limiter, Logger, BookMyShow, Chess, Tic-Tac-Toe, Snake & Ladder, Library, Hotel booking, Cab booking (class-level), Food ordering, In-memory file system, Text editor undo/redo, Pub-Sub, Task scheduler, Thread-safe blocking queue, Notification service, Stack Overflow clone, Online shopping cart, Meeting room scheduler.

### 19.4 Java concurrency to know (LLD + SD)
`synchronized`, `ReentrantLock`, `ReadWriteLock`, `volatile`, `Atomic*`, `ConcurrentHashMap`, `BlockingQueue`, `ExecutorService`, `CompletableFuture`, `Semaphore`, `CountDownLatch`, `ThreadLocal`, deadlock conditions, virtual threads (Java 21). Related guides: see [java-interview-guide.md](./java-interview-guide.md).

### 19.5 LLD answer framework
1. Clarify requirements & scope → 2. Identify entities/nouns → 3. Define relationships and interfaces → 4. Apply patterns where natural → 5. Handle concurrency & extensibility → 6. Walk through a use case/test.

---

## 20. Back-of-the-Envelope Estimation

### 20.1 Numbers to memorize
| Item | Value |
|---|---|
| 1 day | ~86,400 s ≈ 10⁵ s |
| 1 month | ~2.5 M s |
| 1 year | ~31.5 M s ≈ 3×10⁷ s |
| 1 KB / MB / GB / TB / PB | 10³ / 10⁶ / 10⁹ / 10¹² / 10¹⁵ bytes |
| QPS from DAU | `DAU × actions/day ÷ 86,400` ; peak ≈ 2–5× avg |
| Char | 1 byte (ASCII), 2–4 (UTF-8 non-latin) |
| Typical tweet/text row | ~200–500 B |
| Photo | ~200 KB–2 MB; 1 min video 720p ~ 30–60 MB |
| Machine QPS | web server ~1k–10k RPS; Redis ~100k ops/s; MySQL ~1k–10k QPS; Kafka partition ~10-50 MB/s |

### 20.2 Latency numbers (order of magnitude)
| Operation | Time |
|---|---|
| L1 cache | ~1 ns |
| RAM access | ~100 ns |
| SSD random read | ~100 µs |
| Same-DC network round trip | ~0.5 ms |
| HDD seek | ~10 ms |
| Cross-continent round trip | ~100–150 ms |

**Takeaways:** memory is ~1000× faster than SSD-ish network; avoid cross-region calls in hot path; sequential disk is fast.

### 20.3 Worked example (URL shortener)
- 100 M new URLs/month → ~40 writes/s; read:write 100:1 → ~4,000 reads/s; peak ×3 ≈ 12k RPS.
- 5 years: 100M × 12 × 5 = 6 B URLs × 500 B = **3 TB** storage.
- Short code length: base62, 7 chars = 62⁷ ≈ 3.5 trillion ≫ 6 B ✅.
- Cache 20% hot URLs (80/20 rule): 4000 rps × 86400 × 500 B × 20% ≈ ~35 GB/day → fits in Redis.

### 20.4 Tips
Round aggressively, state assumptions, show units, sanity-check ("does 3 TB fit one node? yes, but need replication/HA").

---

## 21. Senior-Level Behavioural / Trade-off Questions

Prepare 2-3 real stories for each:

1. **Largest scale system you designed/owned** — numbers (QPS, data size, team size).
2. **Production incident** — detection, mitigation, root cause, prevention (post-mortem).
3. **A design decision you regret** — what you'd do differently.
4. **Performance optimization** — profiling approach, before/after metrics.
5. **Monolith → microservices migration** (or reverse) — why, how, results.
6. **Disagreement on technical approach** — how you resolved it (data, prototypes, ADRs).
7. **Reducing cost / tech debt** — how you prioritized.
8. **Mentoring & influencing architecture** — design reviews, RFCs, ADRs.
9. **Choosing between build vs buy**; choosing a database; choosing sync vs async.
10. **Handling ambiguity / changing requirements.**

**STAR format** + always quantify impact.

### Typical "why" questions to be ready for
- Why Kafka over RabbitMQ/SQS? (replay, throughput, ordering per partition vs routing flexibility/simplicity)
- Why Cassandra over MySQL? (write scale, multi-DC, no joins/transactions)
- Why Redis here and what if Redis dies? (fallback to DB, replicas, persistence, cold-start stampede)
- Why SQL for this? (relationships, ACID)
- What breaks first when traffic 10×? (DB writes → cache → LB)
- How would you test this system? (load test, chaos, contract tests, canary)
- How do you monitor it & what alerts? (SLO-based)

---

## 22. Cheat Sheet: Problem → Concepts Used

| Problem | Must-mention concepts |
|---|---|
| URL shortener | Base62, ID generator (Snowflake/range), cache, 301 vs 302, analytics async, bloom filter, TTL/expiry |
| Rate limiter | Token bucket, Redis+Lua, sliding window counter, per-user/IP, 429, distributed sync |
| Chat (WhatsApp) | WebSocket, connection registry, message queue per user, delivery receipts, offline store, group fan-out, E2E encryption, media in blob+CDN |
| News feed | Fan-out hybrid, timeline cache (Redis list), ranking, pagination cursor, celebrity problem |
| Uber | Geo-index (H3/geohash), location ingestion, matching, ETA, surge pricing, trip state machine, WebSocket, payment saga |
| YouTube/Netflix | Chunked upload, transcoding DAG, HLS/DASH, CDN, metadata DB, recommendations, view-count counters |
| Dropbox | Chunking, delta sync, dedup via hash, metadata service, notification (long poll), conflict handling |
| Notification service | Queue per channel, templates, retries + DLQ, rate limit per user, preferences, dedupe, priority |
| Web crawler | URL frontier, politeness, robots.txt, bloom filter/URL dedup, DNS cache, content hash for dup pages, distributed workers |
| Typeahead | Trie, top-K per prefix, offline build, caching, sharding by prefix |
| Distributed cache | Consistent hashing, replication, eviction, hot key, failure handling, client vs proxy routing |
| Key-value store | Partitioning, replication, quorum, vector clocks, gossip, hinted handoff, Merkle, LSM, compaction |
| Message queue | Partitions, offsets, replication, consumer groups, ack, DLQ, ordering, retention |
| Distributed lock | Lease, TTL, fencing token, Raft/Zookeeper, Redlock caveats |
| Unique ID | Snowflake, clock skew, worker ID assignment, UUIDv7 |
| Job scheduler | Delayed queue, sorted set/timing wheel, SKIP LOCKED, leader election, idempotent tasks, retries |
| Ticket booking | Seat hold TTL, unique constraint, optimistic/pessimistic lock, waiting room, payment saga |
| Payment system | Idempotency, ledger, state machine, reconciliation, webhooks, saga, outbox |
| Flash sale | Queue admission, Redis atomic DECR, pre-warm cache, rate limit, oversell prevention, async order creation |
| E-commerce | Catalog (search+cache), cart (Redis), inventory (strong consistency), order saga, event-driven |
| Search | Inverted index, sharding, scatter-gather, CDC sync, ranking |
| Google Docs | OT/CRDT, WebSocket, op log, snapshots, presence |
| Leaderboard | Sorted set, sharding, snapshots, time windows |
| Metrics system | Time-series DB, push vs pull, downsampling, retention, alert engine |
| Ad click aggregator | Kafka, stream windowing, exactly-once, dedupe, lambda/kappa, reconciliation |
| Stock exchange | Order book, matching engine single-thread per symbol, sequencer, event log, low latency |
| Video call | WebRTC, STUN/TURN, SFU, signalling server, adaptive bitrate |
| Maps | Graph partitioning, Dijkstra/A*, contraction hierarchies, tiles, real-time traffic |
| Multi-tenant SaaS | Tenant ID everywhere, isolation levels (shared schema / schema-per / DB-per), noisy neighbour limits, per-tenant quotas |

---

## 23. Quick Revision Checklist

**Must be able to explain in 2 minutes each:**
- [ ] CAP & PACELC, consistency models, quorum
- [ ] Consistent hashing + virtual nodes
- [ ] Sharding strategies & hot partitions
- [ ] Replication (leader-follower, multi-leader, leaderless) & lag problems
- [ ] Cache strategies, stampede/penetration/avalanche
- [ ] SQL vs NoSQL choice; indexing; B-Tree vs LSM
- [ ] Isolation levels, optimistic vs pessimistic locks
- [ ] Kafka internals, ordering, delivery semantics, DLQ
- [ ] Outbox + CDC, saga, 2PC, idempotency
- [ ] Circuit breaker, retry w/ jitter, bulkhead, timeouts
- [ ] Rate limiter algorithms + distributed implementation
- [ ] Load balancing (L4/L7), CDN, API gateway
- [ ] WebSocket vs SSE vs long polling; scaling WebSockets
- [ ] Bloom filter, HyperLogLog, Count-Min Sketch, Trie, Geohash/Quadtree
- [ ] Distributed lock + fencing token; leader election; Raft
- [ ] Authentication (JWT/OAuth2/OIDC), encryption, common attacks
- [ ] Observability (logs/metrics/traces), SLO/SLI/SLA
- [ ] Deployment strategies, zero-downtime migrations
- [ ] Estimation numbers & worked examples
- [ ] 2-3 production stories using STAR

### Study plan suggestion
| Week | Focus |
|---|---|
| 1 | Fundamentals, caching, DB, sharding/replication (§3–8) |
| 2 | Distributed theory, messaging, microservices patterns (§9–12) |
| 3 | Practice building-block designs: URL shortener, rate limiter, KV store, queue, notification, scheduler |
| 4 | Practice product designs: feed, YouTube, payments, ticketing, Dropbox, search + mock interviews |
| Ongoing | LLD problems, estimation drills, behavioural stories |

### Recommended resources
- *Designing Data-Intensive Applications* — Martin Kleppmann (best single book; chapters on replication, partitioning, transactions, consensus, streams).
- *System Design Interview Vol 1 & 2* — Alex Xu.
- ByteByteGo newsletter, High Scalability blog, engineering blogs (Uber, Netflix, Meta, Stripe, Discord).
- Papers (skim): Dynamo, Bigtable, GFS, MapReduce, Kafka, Raft, Spanner.

> **Final tip:** In the interview, spend more time on **trade-offs, failure handling, and bottlenecks** than on drawing boxes. That's what separates a senior from a mid-level answer.
