# Redis Interview Guide — Java / Spring Boot Developer (9 Years Experience)

> **How to use this doc:** Every answer follows the pattern **one-line answer → why it matters → detail → memory hook**. In an interview, say the one-liner first, then expand only if the interviewer leans in. The 🧠 lines are deliberate mnemonics — memorise those, and the rest reconstructs itself.

---

## Table of Contents

1. [Fundamentals & Architecture](#1-fundamentals--architecture)
2. [Data Structures and When to Use Them](#2-data-structures-and-when-to-use-them)
3. [Persistence: RDB, AOF, Hybrid](#3-persistence-rdb-aof-hybrid)
4. [Expiration, Eviction & Memory Management](#4-expiration-eviction--memory-management)
5. [Transactions, Lua Scripting & Pipelining](#5-transactions-lua-scripting--pipelining)
6. [Pub/Sub vs Streams](#6-pubsub-vs-streams)
7. [Replication, Sentinel & Cluster](#7-replication-sentinel--cluster)
8. [Distributed Locking](#8-distributed-locking)
9. [Caching Patterns](#9-caching-patterns)
10. [The Four Classic Cache Failures](#10-the-four-classic-cache-failures)
11. [Cache–Database Consistency](#11-cachedatabase-consistency)
12. [Spring Boot + Redis (Deep Dive)](#12-spring-boot--redis-deep-dive)
13. [Rate Limiting, Sessions & Real Use Cases](#13-rate-limiting-sessions--real-use-cases)
14. [Performance, Troubleshooting & Production Ops](#14-performance-troubleshooting--production-ops)
15. [Security](#15-security)
16. [Scenario-Based Questions (30)](#16-scenario-based-questions)
17. [Rapid-Fire Cheat Sheet](#17-rapid-fire-cheat-sheet)
18. [Questions to Ask the Interviewer](#18-questions-to-ask-the-interviewer)

---

## 1. Fundamentals & Architecture

### Q1. What is Redis and why is it so fast?

**One-liner:** Redis is an in-memory, single-threaded, key-value data-structure server that reaches sub-millisecond latency because it avoids disk I/O, lock contention and context switching.

**Detail — the four reasons for speed:**

| Reason | Explanation |
|---|---|
| In-memory | All data lives in RAM; disk is only for durability, not for reads. |
| Single-threaded command execution | No locks, no mutexes, no race conditions, no context switching between threads for the data path. |
| I/O multiplexing (epoll/kqueue) | One thread handles tens of thousands of sockets via an event loop; no thread-per-connection. |
| Optimised data structures + RESP protocol | Purpose-built encodings (listpack, intset, skiplist) and a simple, cheap-to-parse wire protocol. |

**Important nuance to mention (this is what separates a senior from a junior answer):** Redis is *not* fully single-threaded any more. Since Redis 4.0 there are background threads for lazy freeing (`UNLINK`, async flush), and since Redis 6.0 there is **threaded I/O** — reading/writing sockets and parsing can be parallelised across threads, but **command execution is still serialised on one thread**. So atomicity guarantees are unchanged.

🧠 **Memory hook:** *"RAM + one thread + epoll + smart encodings."*

---

### Q2. If Redis is single-threaded, doesn't it waste a multi-core machine?

Yes, one Redis process can only saturate one core for command execution. The standard answers:

1. **Run multiple Redis instances per machine** (e.g. 4 instances pinned to 4 cores) — vertical sharding.
2. **Use Redis Cluster** — horizontal sharding across nodes/instances.
3. **Enable threaded I/O** (`io-threads 4` in redis.conf) — helps when the bottleneck is socket I/O with large payloads, not CPU-bound commands.
4. In practice, the bottleneck in most Spring Boot apps is **network round-trips and payload size**, not the Redis CPU. Pipelining and batching help far more than more cores.

**Why the single thread is a deliberate choice, not a limitation:** Redis commands are tiny in-memory operations (microseconds). Adding threads would mean locking every data structure, and the lock overhead would cost more than the work itself. One thread also gives you free atomicity — `INCR` can never interleave with another `INCR`.

**Example:** a 16-core box running one Redis uses ~1 core for commands. To use the rest you run, say, 6 instances on ports 6379–6384 (each ~1 core) or a 6-shard Cluster, and the client routes each key to the right instance. The trade-off is that multi-key operations (`MGET`, Lua, `MULTI`) only work within a single instance/slot.

**Follow-up probe:** *"When does `io-threads` help?"* — only when the CPU is burned on reading/writing sockets (many clients, large values). It does nothing for slow commands like `SORT` or big Lua scripts, because execution stays on the main thread.

---

### Q3. Redis vs Memcached — how do you choose?

| Aspect | Redis | Memcached |
|---|---|---|
| Data types | Rich (list, set, zset, hash, stream, bitmap, HLL, geo) | Strings/blobs only |
| Persistence | RDB + AOF | None |
| Replication / HA | Replication, Sentinel, Cluster | None built-in |
| Scripting / Transactions | Lua, MULTI/EXEC | No |
| Threading | Single-threaded exec (multi-threaded I/O) | Truly multi-threaded |
| Eviction | 8 policies incl. LFU | LRU only |
| Best for | Anything beyond plain caching: leaderboards, queues, locks, rate limits, sessions, pub/sub | Pure, huge, simple string cache with very high multi-core throughput |

**Answer to give:** "For a plain memoisation cache with huge multi-core boxes, Memcached is marginally more efficient. For everything I've built in the last several years — sessions, distributed locks, rate limiting, leaderboards, dedupe sets, event streams — Redis is the only realistic choice because those are data-structure problems, not blob-storage problems."

---

### Q4. What is the Redis event loop / `aeEventLoop`?

Redis runs a loop that (a) polls file descriptors for readiness using the best OS multiplexer available, (b) executes ready commands one at a time, (c) runs time events (`serverCron`) roughly every 100 ms for expiry sampling, eviction, replication checks, rehashing, and stats.

**In plain words — how one loop serves thousands of clients:** the OS "multiplexer" (`epoll` on Linux, `kqueue` on BSD/macOS) lets Redis hand over *all* open sockets in a single call and be told which ones have data ready. The loop then takes a ready socket, reads the command, runs it to completion, writes the reply, and moves to the next. No thread is ever blocked waiting on one slow client. `aeEventLoop` is simply the name of this small event library inside the Redis source (`ae.c`). `serverCron` is the periodic "housekeeping" callback (the `hz` setting, default 10 times/second, controls how often it runs).

**Why this matters in interviews:** it explains why **one slow command blocks everything**. A `KEYS *` on 10M keys, a `HGETALL` on a 500k-field hash, a big `Lua` script, or `FLUSHALL` on a huge dataset stalls every other client. That single fact drives half of Redis production best practice.

🧠 **Memory hook:** *"One queue, one server. A fat customer at the counter blocks the whole line."*

---

### Q5. What is RESP?

REdis Serialization Protocol — a simple, human-readable, binary-safe request/response protocol. RESP2 is the classic; **RESP3** (Redis 6+) adds typed replies (maps, sets, doubles, big numbers), push messages (needed for client-side caching / invalidation) and out-of-band data. Lettuce supports RESP3; enabling it lets you use client-side caching and better pub/sub-on-same-connection behaviour.

**Why it exists:** a client and server need an agreed format for "what is a command and what is a reply". RESP is deliberately trivial so parsing costs almost nothing (one of the reasons Redis is fast) and you can debug it with `telnet`/`nc`.

**How it looks on the wire** — each value starts with a type byte and lines end with `\r\n`: `+` simple string, `-` error, `:` integer, `$` bulk string (length-prefixed, so it is binary-safe), `*` array. The command `SET user:1 Bob` is sent as:

```
*3\r\n$3\r\nSET\r\n$6\r\nuser:1\r\n$3\r\nBob\r\n
```

and the server replies `+OK\r\n`. Because commands are just arrays of bulk strings, a client can **pipeline** many of them in one TCP write and read the replies back in order.

**Interview probe:** *"Why does the protocol version matter to you?"* — with RESP2, a connection in Pub/Sub mode could only do subscribe-related commands, so you needed a separate connection; RESP3 push messages allow server-initiated invalidation messages on the normal connection.

---

### Q6. Is Redis ACID?

Partially, and you should be precise:

- **Atomicity** — yes, per command, and per `MULTI/EXEC` block or Lua script (all commands run without interleaving). **But there is no rollback**: if command 3 of 5 fails at runtime, commands 1, 2, 4, 5 still apply. Redis treats runtime errors as programmer bugs.
- **Consistency** — only in the "no invalid state" sense; there are no constraints/foreign keys. (Nothing stops you writing `order:1 → customer:999` when customer 999 does not exist — referential integrity is the application's job.)
- **Isolation** — effectively serialisable, because execution is single-threaded.
- **Durability** — configurable and **not guaranteed by default**. With `appendfsync everysec` you can lose up to ~1 second of writes. Replication is **asynchronous**, so a failover can lose acknowledged writes.

🧠 **Memory hook:** *"Atomic and isolated by design; durable only if you pay for it; no rollback ever."*

---

## 2. Data Structures and When to Use Them

### Q7. Walk me through the Redis data types and a real use case for each.

| Type | Key commands | Real use case |
|---|---|---|
| **String** | `SET/GET/INCR/SETEX/SETNX/APPEND/GETRANGE` | Cached JSON/serialized DTO, counters, feature flags, distributed lock token |
| **Hash** | `HSET/HGET/HMGET/HINCRBY/HGETALL/HSCAN` | Object with independently-updated fields — user profile, shopping cart (`cart:{userId}` → productId → qty), config map |
| **List** | `LPUSH/RPOP/LRANGE/LTRIM/BLPOP/BRPOP` | Simple FIFO/LIFO queue, "latest N activity feed" with `LPUSH` + `LTRIM 0 99`, worker queue via `BRPOPLPUSH` |
| **Set** | `SADD/SISMEMBER/SREM/SINTER/SUNION/SCARD/SPOP` | Unique visitors, tags, "has user already voted?", mutual-friends via `SINTER`, random winner via `SPOP` |
| **Sorted Set (ZSet)** | `ZADD/ZRANGE/ZREVRANGE/ZRANGEBYSCORE/ZINCRBY/ZRANK/ZREMRANGEBYSCORE` | Leaderboards, priority queue, delayed job scheduler (score = execute-at epoch), sliding-window rate limiter, time-series index |
| **Bitmap** | `SETBIT/GETBIT/BITCOUNT/BITOP/BITPOS` | Daily active users, 365-day login calendar per user (46 bytes/user/year), A/B flags |
| **HyperLogLog** | `PFADD/PFCOUNT/PFMERGE` | Approximate unique counts (unique IPs/page views) — 12 KB fixed, ~0.81% error, cannot list members |
| **Stream** | `XADD/XREAD/XREADGROUP/XACK/XAUTOCLAIM/XPENDING` | Durable event log, Kafka-lite with consumer groups, at-least-once job processing |
| **Geospatial** | `GEOADD/GEOSEARCH/GEODIST` | "Drivers within 3 km", store locator (internally a ZSet with geohash scores) |
| **Bitfield** | `BITFIELD` | Packed multi-counter (e.g. many small counters in one key) |
| **JSON / TS / Search** (modules) | `JSON.SET`, `FT.SEARCH`, `TS.ADD` | Native JSON docs, secondary indexes, full-text search, time series (Redis Stack) |

**How to read the table (plain-language additions):** every value in Redis lives under a *key* (a string name) and has a *type* that decides which commands work on it. Pick the type by the question you'll ask later:
- *"Give me this whole thing by key"* → String. `SET user:1:name Asha`, `INCR page:views`.
- *"Give me one field of a record"* → Hash. `HSET user:1 name Asha city Pune` then `HGET user:1 city`.
- *"What came first/last?"* → List (insertion order, fast at both ends; poor for searching the middle).
- *"Is X in the group? Any duplicates?"* → Set (unique, unordered; `SISMEMBER` is O(1)).
- *"Who's top/bottom, or what's due next?"* → Sorted Set (unique members ordered by a numeric score).
- *"Yes/no per numbered user"* → Bitmap (1 bit each; user 1,000,000 online = `SETBIT online 1000000 1`).
- *"Roughly how many distinct?"* → HyperLogLog (Q11). *"Durable event log with groups"* → Stream (Q12). *"Near me"* → Geo.
**Pitfall:** one wrong choice hurts later — e.g. storing a big collection as one JSON String means every small change rewrites all of it (Q8), and a List used as a lookup makes `LREM`/`LINDEX` O(N). Also mention that a **key is bound to one type**: `SET` on a key holding a List overwrites it, and using the wrong command gives a `WRONGTYPE` error.

🧠 **Memory hook — "SH-L-SZ-B-H-S-G":** *String, Hash, List, Set, ZSet, Bitmap, HyperLogLog, Stream, Geo.* Or remember by intent: **cache (String) → object (Hash) → order (List) → uniqueness (Set) → ranking (ZSet) → presence (Bitmap) → cardinality (HLL) → events (Stream) → location (Geo).**

---

### Q8. String vs Hash for caching a Java object — which do you pick?

**String (serialized JSON/Protobuf):**
- ✅ One round trip, simplest serializer story, easy TTL, easy versioning
- ✅ Compresses well; can be gzipped
- ❌ Must read/write the whole object to change one field

**Hash (field per property):**
- ✅ Partial reads (`HMGET name,email`) and partial atomic updates (`HINCRBY balance 100`)
- ✅ Memory-efficient for small objects thanks to **listpack** encoding
- ❌ TTL only on the whole key, not per field (until Redis 7.4, which added `HEXPIRE`)
- ❌ Nested objects need flattening; type info is lost (everything is a string)

**Rule of thumb I give:** *"Read-mostly, whole-object → String. Fields updated independently or partially read → Hash."* For example: product catalogue entry → String; shopping cart or a counter-heavy aggregate → Hash.

**Why it matters — a concrete contrast.** Suppose a `User` has 10 fields and you only increment `loginCount`:

```
# String: read JSON, deserialize, change one field, serialize, write back (2 round trips, and a race if two threads do it)
GET user:42   →  {"name":"Asha","email":"...","loginCount":7, ...}
SET user:42 '{"name":"Asha","email":"...","loginCount":8, ...}'

# Hash: one atomic command, no read-modify-write race
HINCRBY user:42 loginCount 1
```

**Pitfalls / probes:** a *listpack* is Redis's compact array-like encoding used for small hashes (default up to 128 fields / 64-byte values via `hash-max-listpack-entries/value`); beyond that Redis converts to a real hash table and the memory advantage shrinks. With Spring, a String is what `GenericJackson2JsonRedisSerializer` gives you; a Hash is what `opsForHash()` / `Jackson2HashMapper` gives you.

---

### Q9. How would you implement a leaderboard with millions of players?

```
ZADD game:leaderboard 4500 "user:101"          # set score
ZINCRBY game:leaderboard 50 "user:101"          # increment
ZREVRANGE game:leaderboard 0 9 WITHSCORES       # top 10
ZREVRANK game:leaderboard "user:101"            # my rank (0-based)
ZCOUNT game:leaderboard 4000 5000               # players in a band
```

Points to raise unprompted:
- ZSet is a **skiplist + hash map**: O(log N) insert/rank, O(1) score lookup.
- **Ties:** Redis orders equal scores lexicographically. To break ties by earliest achievement, encode a composite score: `score * 10^10 + (MAX_TS - timestamp)` inside a double's 53-bit safe integer range.
- **Rank pages** for millions of users: use `ZREVRANGE` with limits; never `ZRANGE 0 -1`.
- **Time windows:** keep `leaderboard:daily:20260907`, `weekly:2026W36` with TTLs; merge with `ZUNIONSTORE`.
- **Cluster note:** all these keys must be tagged (`{game1}`) to live in one slot if you want `ZUNIONSTORE` across them.

**Why a Sorted Set fits:** a ZSet stores *unique members*, each with a numeric *score*, always kept sorted by score. The "skiplist" is a layered linked list that allows binary-search-like jumps, so adding a player or asking "what's my rank?" costs O(log N) — about 20 steps for a million players — rather than re-sorting. The "hash map" part lets `ZSCORE` return a player's score in O(1).

**Example flow:** a game server calls `ZINCRBY game:leaderboard 50 user:101` when a match ends; the lobby screen calls `ZREVRANGE ... 0 9 WITHSCORES` for the top 10; the profile screen calls `ZREVRANK` for "You are #4,231" (add 1 because ranks start at 0). **Tie-break detail:** if two players both have 4500, Redis orders them alphabetically by member name, which feels random to users; encoding the achievement time into the score fixes it (an older timestamp should rank higher, hence `MAX_TS - timestamp`). **Pitfall:** doubles are exact only up to 2^53, so don't pack too many digits into the score. **Spring:** `redisTemplate.opsForZSet().incrementScore(key, userId, 50)` and `.reverseRangeWithScores(key, 0, 9)`.

---

### Q10. How do you build a delayed / scheduled job queue in Redis?

Use a ZSet where the **score is the execution timestamp**:

```
ZADD delayed:jobs 1757260800 "job:{payload-id}"
# poller, every second, atomically claim due jobs via Lua:
ZRANGEBYSCORE delayed:jobs -inf <now> LIMIT 0 100
ZREM ... (in the same Lua script)  → push to a ready List
LPUSH ready:jobs <jobId>
# workers: BRPOPLPUSH ready:jobs processing:jobs 5
```

Why the Lua: `ZRANGEBYSCORE` + `ZREM` must be atomic, otherwise two pollers claim the same job. Why `BRPOPLPUSH`: it gives you a **reliable queue** — the job sits in `processing:jobs` until acked, and a reaper can requeue stale entries. In production I would prefer **Redis Streams with consumer groups** (native ack, pending-entries list, `XAUTOCLAIM`) or **Redisson's `RDelayedQueue`**, which implements exactly this pattern for you.

**Step by step (plain language):**
1. *Producer* schedules "send reminder in 10 minutes": `ZADD delayed:jobs <now+600> jobId`. The ZSet keeps jobs sorted by due time, so the earliest job is always at the front.
2. *Poller* wakes every second and asks "which jobs have score ≤ now?" (`ZRANGEBYSCORE`). Those are due.
3. It removes them from the ZSet and pushes them to a normal List (`ready:jobs`) — in **one Lua script** so the "find + remove" can't be split by another poller.
4. *Workers* block on the list (`BRPOPLPUSH` = pop from `ready:jobs` and simultaneously push onto `processing:jobs`). If a worker crashes mid-job, the job is still visible in `processing:jobs` instead of being lost.

**Pitfalls:** plain `BRPOP` loses the job if the worker dies after popping; polling adds up to 1 s of latency; make job handlers **idempotent** because a reaper requeue can cause a retry. (`BRPOPLPUSH` is deprecated since Redis 6.2 in favour of `BLMOVE`, same idea.)

---

### Q11. When would you use HyperLogLog over a Set?

When you need **cardinality only** and the set is huge. A Set storing 10 million unique visitor IDs costs hundreds of MB; an HLL costs **12 KB** with ~0.81% standard error, and merges across days with `PFMERGE`. Trade-off: you can never enumerate or check membership. Classic answer: "unique visitors per page per day → HLL; 'has *this* user seen it?' → Set or Bitmap."

**How it works (intuition):** HLL (HyperLogLog) hashes each item and looks at the pattern of the hash bits — e.g. seeing a hash that starts with many leading zeros is rare, so the longest run of leading zeros observed across all items lets you *estimate* how many distinct items there must have been. Redis keeps this in 16,384 small registers (hence the fixed ~12 KB), and it never stores the items themselves — which is why you can't list them.

```
PFADD uv:page:home:2026-09-07 user:1 user:2 user:1   # duplicates are ignored
PFCOUNT uv:page:home:2026-09-07                      # ≈ 2
PFMERGE uv:page:home:week uv:page:home:2026-09-07 uv:page:home:2026-09-08
PFCOUNT uv:page:home:week                            # unique across both days
```

**Pitfalls:** the answer is approximate (~0.81% error), so do not use it for billing or exact-once logic; and `PFMERGE` is the reason HLL beats summing daily counts — the same user visiting on two days is counted once.

---

### Q12. Explain Redis Streams vs a List-based queue.

| | List queue | Stream |
|---|---|---|
| Consumption | Destructive pop | Log; entries stay until trimmed |
| Multiple consumer groups | No | Yes, independent offsets |
| Ack / retry | Manual (`BRPOPLPUSH` + reaper) | Built-in PEL, `XACK`, `XAUTOCLAIM` |
| Replay / audit | No | Yes, by ID or time |
| Fan-out | No | Yes |
| Memory control | `LTRIM` | `XADD MAXLEN ~ 10000` / `XTRIM MINID` |

Streams are "Kafka-lite": ordered, durable-ish, with consumer groups. Use Streams for at-least-once event processing; Lists only for the simplest fire-and-forget worker queues.

**Key terms explained:**
- **Stream** — an append-only log; each entry gets an ID like `1757260800000-0` (millisecond time + sequence) and holds field/value pairs. Reading does **not** remove entries.
- **Consumer group** — a named set of workers that *share* a stream: Redis hands each entry to exactly one consumer in the group, so work is load-balanced. A second group gets its own independent copy of the whole stream (fan-out).
- **PEL (Pending Entries List)** — entries delivered to a consumer but not yet `XACK`ed. If the consumer dies, the entries stay pending and another worker can take them over with `XAUTOCLAIM` after an idle timeout. That is how you get at-least-once delivery.

```
XADD orders MAXLEN ~ 10000 * orderId 42 amount 99      # append, cap length approx.
XGROUP CREATE orders billing $ MKSTREAM                  # group starts at new entries
XREADGROUP GROUP billing worker-1 COUNT 10 BLOCK 5000 STREAMS orders >
XACK orders billing 1757260800000-0                      # done processing
```

**Pitfalls:** unbounded streams eat RAM (always trim); at-least-once means handlers must be idempotent; a List `BRPOP` queue is fine only when losing a job on worker crash is acceptable or you build the reaper yourself. (Section 6, Q26 shows the Spring Boot version.)

---

## 3. Persistence: RDB, AOF, Hybrid

### Q13. Compare RDB and AOF.

**RDB (Redis Database snapshot)**
- Point-in-time binary dump (`dump.rdb`) via `SAVE` (blocking — never in prod) or `BGSAVE` (forks a child).
- Config: `save 900 1 / save 300 10 / save 60 10000`.
- ✅ Compact, fast to load, ideal for backups and disaster recovery, minimal effect on read throughput.
- ❌ Data loss window = time since last snapshot (minutes). `fork()` on a large dataset costs memory (copy-on-write) and can cause a latency spike.

**AOF (Append Only File)**
- Appends every write command to a log; replayed on restart.
- `appendfsync always` (safest, slowest) | `everysec` (default, ≤1 s loss) | `no` (OS decides, ~30 s risk). (`fsync` is the OS call that forces buffered data onto the physical disk; until it happens, a power cut can lose the data even though Redis "wrote" it.)
- Rewrite (`BGREWRITEAOF`, or automatic at `auto-aof-rewrite-percentage 100`) compacts the log; Redis 7 uses a **multi-part AOF** (base RDB + incremental files + manifest) in an `appendonlydir`.
- ✅ Much better durability, human-readable, self-healing (`redis-check-aof --fix`).
- ❌ Bigger files, slower restart, slightly lower write throughput.

**Recommended production setup:** enable **both**. On restart Redis prefers AOF (more complete). With `aof-use-rdb-preamble yes` you get the hybrid format: fast RDB-style base + command tail = fast load *and* small loss window.

🧠 **Memory hook:** *RDB = photograph (small, occasional). AOF = CCTV recording (complete, bulky). Prod = keep both.*

---

### Q14. What is the fork/copy-on-write problem?

`BGSAVE` and AOF rewrite call `fork()`. The child shares memory pages copy-on-write. If the parent keeps writing heavily, pages get copied and memory usage can approach **2× the dataset** in the worst case. Also, on a large instance the fork itself takes tens/hundreds of milliseconds and stalls the event loop.

**Mitigations:**
- Size instances so the box has headroom (keep `used_memory` under ~50–60% of RAM if you snapshot).
- Set `vm.overcommit_memory = 1`; disable transparent huge pages (THP) — THP makes COW copy 2 MB pages instead of 4 KB, badly worsening latency.
- Do snapshots on a **replica**, not the primary.
- Watch `latest_fork_usec` in `INFO stats`.

**Plain-language explanation:** `fork()` is the OS call that clones a process. Redis uses it so a child can write the snapshot while the parent keeps serving clients. The OS doesn't actually copy RAM up front — parent and child *share* the same memory pages, marked read-only. This trick is called **copy-on-write (COW)**: the first time either side modifies a page, the OS duplicates just that page. So:
- Few writes during the snapshot → few pages copied → cheap.
- Heavy writes during a long snapshot → most pages get duplicated → memory can nearly double and the box can hit OOM-kill or start swapping.

**Example:** a 20 GB dataset on a 32 GB machine looks fine until a `BGSAVE` coincides with a write burst; the extra copied pages push RAM over 32 GB, the Linux OOM killer terminates `redis-server`, and you lose the instance. The "stall" part is separate: `fork()` must copy the page tables, which for 20 GB can take hundreds of ms during which no commands are served.

**Probe:** *"Why disable THP?"* — Transparent Huge Pages make the OS track memory in 2 MB chunks, so one small write copies 2 MB instead of 4 KB.

---

### Q15. If you use Redis only as a cache, do you need persistence?

Usually no — and turning it off avoids fork spikes and disk I/O. But consider **cold-start (thundering herd on restart)**: an empty cache after a restart can flatten your database. Options: keep RDB just for warm restarts, use a replica to take over, or implement staged cache warming. Say this explicitly in interviews — it shows you think about blast radius, not just config flags.

**Why it matters, with numbers:** suppose Redis absorbs 50,000 reads/sec at a 95% hit rate, so the database only sees ~2,500 reads/sec. After a restart with an empty cache the hit rate is 0%, so the database suddenly receives all 50,000 reads/sec — a 20× jump — and may time out, which makes the app retry and worsens it. That is the **thundering herd** (many requests hitting the origin at once).

**Ways to soften it:** (1) keep `save` snapshots so Redis reloads data on restart (stale but warm); (2) run a replica and fail over instead of restarting the primary; (3) warm the cache before sending traffic (preload top-N hot keys); (4) protect the DB with request coalescing, per-key locks and rate limits (see Section 10). If the cache truly is disposable and the DB can cope, `save ""` and `appendonly no` is the right, fastest config.

---

## 4. Expiration, Eviction & Memory Management

### Q16. How does Redis expire keys?

Two mechanisms working together:

1. **Lazy (passive) expiration** — when a key is accessed, Redis checks the expiry and deletes it then.
2. **Active expiration** — `activeExpireCycle` in `serverCron` samples 20 random keys from the expires dict ~10 times/second; if more than 25% were expired, it repeats immediately. This is probabilistic, bounded by a CPU-time budget so it never stalls the server.

Consequences worth mentioning:
- An expired key can occupy memory after its TTL until sampled — so `used_memory` may lag.
- On a **replica**, keys are *not* independently deleted; the replica waits for the primary's `DEL`/`UNLINK` propagation, but logically hides expired keys from reads (since 3.2). This is why replica reads can't be blamed for "stale TTL" bugs.
- Redis 7 propagates expiry as `DEL` in the replication stream, keeping replicas deterministic.

**Why two mechanisms?** Checking every key's TTL on a timer would cost O(N) and freeze the server; waiting only for access would leave never-read keys in RAM forever. Lazy expiry gives *correctness* (you never read an expired value), active expiry gives *memory reclamation* (garbage that nobody touches is eventually cleaned) at a bounded CPU cost.

**Example:**
```
SET otp:9876 123456 EX 300     # EX = seconds; same as SETEX
TTL otp:9876                   # seconds remaining; -1 = no TTL, -2 = key gone
```
At 301 s the key is logically gone: `GET` returns nil immediately (lazy check). If nobody reads it, it still sits in RAM until the sampler happens to pick it.

**Pitfalls:** `SET key newValue` **removes** the TTL unless you pass `KEEPTTL`/`EX` again (a classic bug where a "cache entry" silently becomes immortal); `EXPIRE` on millions of keys with the *same* second causes a burst of expiries (avalanche — see Section 10), so add random jitter.

🧠 **Memory hook:** *"Deleted when touched, or when the janitor's random sample finds it."*

---

### Q17. List the eviction policies and when to use each.

Trigger: `used_memory` reaches `maxmemory`.

| Policy | Behaviour |
|---|---|
| `noeviction` | Reject writes with OOM error (default). Correct for queues/locks/source-of-truth data. |
| `allkeys-lru` | Evict least-recently-used across all keys. **Best default for a pure cache.** |
| `allkeys-lfu` | Evict least-frequently-used (Redis 4+). Better when access is skewed and some keys are hot but not recent. |
| `allkeys-random` | Random. Cheap; use when access is uniform. |
| `volatile-lru` / `volatile-lfu` / `volatile-random` | Same, but only among keys **with a TTL**. |
| `volatile-ttl` | Evict the key with the nearest expiry. |

**Trap to avoid:** with a `volatile-*` policy and no TTLs set, Redis behaves like `noeviction` and starts erroring. Common production incident.

**How to apply it:** eviction is off until you cap memory.
```
maxmemory 4gb
maxmemory-policy allkeys-lru
```
When a write would push `used_memory` over 4 GB, Redis first evicts keys per the policy until it fits, then runs the write. **LRU** = "Least Recently Used" (evict what hasn't been touched for longest); **LFU** = "Least Frequently Used" (evict what is touched rarely overall).

**Choosing, by example:** a product-page cache where everything is rebuildable → `allkeys-lru`. A cache where a few viral items are hit constantly but a one-off full-catalogue crawl would pollute LRU → `allkeys-lfu` (the crawl's single hits don't make keys "frequent"). Redis also holding distributed locks or queues alongside cache → don't mix; use a separate instance with `noeviction`, or `volatile-lru` with TTLs only on cache keys (but see the trap above). Monitor `evicted_keys` in `INFO stats` — a steadily rising number means the cache is too small.

**LRU implementation detail (good senior colour):** Redis uses **approximated LRU** — it samples `maxmemory-samples` keys (default 5) and evicts the oldest of the sample, rather than maintaining a global linked list. Raising it to 10 gets close to true LRU with slightly more CPU. LFU uses an 8-bit logarithmic counter with a decay period (`lfu-log-factor`, `lfu-decay-time`).

🧠 **Memory hook:** *"noeviction = fail fast. allkeys-lru = classic cache. allkeys-lfu = hot-key aware. volatile-* = only TTL'd keys."*

---

### Q18. `DEL` vs `UNLINK`? `SCAN` vs `KEYS`?

- **`DEL`** frees memory synchronously → deleting a 5 GB hash blocks the event loop for seconds. **`UNLINK`** removes the key from the keyspace immediately and frees memory in a background thread. **Rule: always use `UNLINK` for potentially big keys** (and `lazyfree-lazy-*` configs to make eviction/expiry async too).
- **`KEYS pattern`** is O(N) over the whole keyspace and blocks. **Never run it in production.** Use **`SCAN`** — a cursor-based, incremental iterator with `COUNT` and `MATCH`, guaranteeing that keys present the whole time are returned at least once (may return duplicates; doesn't guarantee a snapshot). Same for `HSCAN`, `SSCAN`, `ZSCAN`.

In Spring Data Redis:

**How `SCAN` works, plainly:** `SCAN 0` returns a batch of keys plus a new cursor number; you call `SCAN <cursor>` again until the cursor returns to `0`. Each call does a small, bounded amount of work, so other clients can run between calls — that is why it is safe where `KEYS` is not. `COUNT` is only a *hint* for how many slots to examine per call, not a guaranteed page size, and `MATCH` filters *after* the batch is fetched (so a call may return zero keys but a non-zero cursor — keep looping). In Cluster, SCAN only covers one node, so you scan each master.

```java
ScanOptions options = ScanOptions.scanOptions().match("session:*").count(500).build();
try (Cursor<byte[]> cursor = redisTemplate.executeWithStickyConnection(
        conn -> conn.keyCommands().scan(options))) {
    while (cursor.hasNext()) {
        byte[] key = cursor.next();
        // process incrementally, e.g. batch UNLINK
    }
}
```

🧠 **Memory hook:** *"KEYS and DEL are blocking siblings; SCAN and UNLINK are their polite versions."*

---

### Q19. What's a "big key" and a "hot key", and how do you fix them?

**Big key** = a single key with a huge value (a 200 MB string, a list with 5M elements, a hash with 1M fields).
- Symptoms: latency spikes, uneven cluster slot memory, network saturation, blocking on delete/expiry, slow `BGSAVE`/migration.
- Detect: `redis-cli --bigkeys`, `redis-cli --memkeys`, `MEMORY USAGE key`, offline RDB analysis (`rdb_bigkeys`, `redis-rdb-tools`).
- Fix: **shard the key** (`user:cart:{id}:part1..N`, or hash-of-field modulo buckets), store blobs in S3/object storage with only a pointer in Redis, compress values, and read with `HSCAN`/`LRANGE` pages instead of `HGETALL`/`LRANGE 0 -1`.
- *Why it hurts:* Redis is single-threaded, so moving or deleting a 200 MB value occupies the one thread (and the network) for that whole time — every other client waits. *Example fix:* instead of one hash `followers:celebrity` with 5M fields, use `followers:celebrity:{0..99}` and pick the bucket with `crc32(userId) % 100` — each bucket is ~50k fields.

**Hot key** = one key receiving a disproportionate share of traffic (a celebrity profile, a flash-sale SKU). In Cluster it pins one shard to 100% CPU.
- Detect: `redis-cli --hotkeys` (needs LFU), `MONITOR` sampling (careful — expensive), client-side metrics, `OBJECT FREQ`.
- Fix: **local L1 cache** (Caffeine) in front of Redis; **key replication** — write the value to `hot:key:0..N` and have clients read a random replica; read from Redis replicas; or use client-side caching (RESP3 tracking / Lettuce cache-invalidation support).
- *Why it hurts:* a key lives on exactly one shard, so adding nodes does not help — all requests for that key still hit one CPU. *Example of key replication:* a flash-sale product `sku:9987` read 100k/s becomes `sku:9987#0..9`; each client reads `sku:9987#<random 0-9>`. In Cluster those 10 names hash to different slots/shards, so load is spread ~10×. The cost is that writes/invalidations must update all 10 copies. Caffeine with a 1–5 s TTL in each pod is often the simplest fix.

🧠 **Memory hook:** *"Big key = split it. Hot key = copy it (or cache it locally)."*

### Q20. How do you reduce Redis memory usage?

1. **Shorten key names** — `u:1001:p` instead of `user:1001:profile` (matters at 100M keys).
2. **Use Hashes for small objects** — under `hash-max-listpack-entries` (128) and `hash-max-listpack-value` (64 bytes), a hash is stored as a compact listpack, dramatically cheaper than many top-level keys.
3. **Efficient serialization** — JSON → Protobuf/MessagePack/Kryo, plus compression (LZ4/Snappy) for values >1 KB. Also avoid `JdkSerializationRedisSerializer` (huge and Java-only).
4. **Integers are shared/encoded** — `int` encoding and shared integer objects (0–9999) are nearly free; `intset` for all-integer sets.
5. **Set TTLs on everything cacheable**, plus a sane eviction policy.
6. **Bitmaps/HLL instead of Sets** where semantics allow.
7. Diagnose with `INFO memory`, `MEMORY DOCTOR`, `MEMORY STATS`, and watch `mem_fragmentation_ratio` (enable `activedefrag yes` if it's >1.5).

**Why each helps, with an example:** every Redis key carries fixed overhead (roughly 50–100 bytes for the dict entry, key object and expiry), so 100M tiny keys can cost ~10 GB before any value is stored — hence (1) and (2). Example for (2): instead of 1M keys `user:{id}:name`, store `HSET user:bucket:{id/100} {id} name` so each bucket is a small listpack hash; memory often drops by half or more. `mem_fragmentation_ratio` = RSS (memory the OS gave Redis) ÷ `used_memory` (memory Redis actually uses): >1.5 means wasted fragmented space, <1 means Redis is being **swapped to disk** — a far worse emergency. Always measure before/after with `MEMORY USAGE key`; don't guess.

---

## 5. Transactions, Lua Scripting & Pipelining

### Q21. Explain `MULTI` / `EXEC` / `DISCARD` / `WATCH`.

- `MULTI` starts queuing commands on that connection. Each returns `QUEUED`.
- `EXEC` executes the whole queue **atomically and in order** — nothing interleaves because the server is single-threaded.
- `DISCARD` throws the queue away.
- `WATCH key...` gives **optimistic locking**: if any watched key is modified by anyone before `EXEC`, `EXEC` returns nil and nothing runs. You then retry (Check-And-Set loop).

**Two failure modes to distinguish (classic follow-up):**
1. **Syntax/unknown-command error at queue time** → Redis refuses the whole transaction at `EXEC`. Safe.
2. **Runtime error** (e.g. `INCR` on a string that holds "abc") → that command fails, **the rest still execute, and there is no rollback**. Redis considers this a programming error.

**Spring Boot example (`SessionCallback` — must reuse one connection):**

```java
List<Object> results = redisTemplate.execute(new SessionCallback<List<Object>>() {
    @Override
    public List<Object> execute(RedisOperations ops) {
        ops.watch("account:1:balance");
        Integer balance = (Integer) ops.opsForValue().get("account:1:balance");
        if (balance == null || balance < 100) {
            ops.unwatch();
            return Collections.emptyList();
        }
        ops.multi();
        ops.opsForValue().decrement("account:1:balance", 100);
        ops.opsForValue().increment("account:2:balance", 100);
        return ops.exec();   // null/empty ⇒ watched key changed ⇒ retry
    }
});
```

⚠️ **Gotchas:** `enableTransactionSupport(true)` on `RedisTemplate` binds the connection to the thread and participates in Spring's `@Transactional` — but then read commands inside the MULTI also return null (they're queued). In **Redis Cluster**, all keys in a transaction must be in the same hash slot. And Spring `@Transactional` around Redis does **not** give you DB+Redis atomicity.

🧠 **Memory hook:** *"MULTI = a queue, not a lock. WATCH = optimistic CAS. No rollback, ever."*

---

### Q22. Lua scripting — why prefer it over MULTI/EXEC?

`EVAL script numkeys key... arg...` runs the script **atomically** on the server. Advantages over MULTI:

- **Conditional logic and computed values** — you can read a value and branch on it *inside* the atomic unit, which MULTI cannot do.
- **One round-trip** instead of many.
- **`EVALSHA`** reuses a cached script by SHA1, so you don't resend the body.
- Redis 7 adds **Functions** (`FUNCTION LOAD`, `FCALL`) — persisted, replicated, named libraries; better than ad-hoc scripts for long-lived logic.

**Rules for scripts:** keep them short (they block the event loop), always declare keys in `KEYS[]` (required for Cluster routing), keep them **deterministic** (scripts are replicated by effects since Redis 5, but avoid random/time-dependent logic anyway), never loop over unbounded data.

**Spring Boot: atomic "check stock and decrement" example**

```java
@Bean
public DefaultRedisScript<Long> deductStockScript() {
    DefaultRedisScript<Long> s = new DefaultRedisScript<>();
    s.setScriptText("""
        local stock = tonumber(redis.call('GET', KEYS[1]))
        local qty = tonumber(ARGV[1])
        if stock == nil then return -1 end
        if stock < qty then return 0 end
        redis.call('DECRBY', KEYS[1], qty)
        return 1
        """);
    s.setResultType(Long.class);
    return s;
}

Long result = redisTemplate.execute(deductStockScript,
        List.of("stock:sku:9987"), "2");
// 1 = deducted, 0 = insufficient, -1 = key missing
```

🧠 **Memory hook:** *"Lua = MULTI with an if-statement, in one round trip."*

---

### Q23. Pipelining vs transactions vs Lua — what's the difference?

| | Pipelining | MULTI/EXEC | Lua / Function |
|---|---|---|---|
| Purpose | Reduce network RTT | Atomic group | Atomic group + logic |
| Atomic? | **No** — other clients can interleave | Yes | Yes |
| Conditional logic? | No | No | Yes |
| Round trips | 1 | 1 (after queuing) | 1 |

Pipelining is purely a **network optimisation**: send N commands without waiting for each reply. It turns 1,000 × 1 ms RTT into ~1 × a few ms. This is the single biggest easy win in most Spring Boot apps.

```java
List<Object> results = redisTemplate.executePipelined((RedisCallback<Object>) conn -> {
    for (int i = 0; i < 10_000; i++) {
        conn.stringCommands().set(("k:" + i).getBytes(), ("v" + i).getBytes());
    }
    return null;   // must return null
});
```

Also mention **`MSET`/`MGET`** for bulk string ops, and that in **Cluster** pipelining/MGET must be split per-slot (Lettuce/Redisson do this for you with cross-slot support; Jedis Cluster is stricter).

---

### Q24. How do you do atomic counters / distributed sequence numbers?

`INCR`, `INCRBY`, `HINCRBY`, `INCRBYFLOAT` — all atomic. For sequences, `INCR` is safe across all clients; grab ranges (`INCRBY 100`) client-side to reduce round trips. For a Java example:

```java
Long views = redisTemplate.opsForValue().increment("post:42:views");
// or RedissonClient:
RAtomicLong seq = redisson.getAtomicLong("order:seq");
long id = seq.incrementAndGet();
```

Trap: `INCR` on a missing key initialises to 0 first, so counters never need pre-creation — but that also means an evicted counter silently restarts at 1. If loss is unacceptable, don't rely on eviction-eligible counters (use `noeviction` or persist periodically).

**Why atomic counters matter:** the naive Java way — `GET`, add 1 in code, `SET` — is a *read-modify-write* race: two pods read 10, both write 11, one increment is lost. `INCR` does the read+add+write inside Redis in one uninterruptible step, so 1,000 concurrent increments always yield exactly +1,000.

**Sequence-number example (range allocation):** to issue order IDs without a Redis round trip each time, a pod calls `INCRBY order:seq 100` once, gets e.g. 500, and hands out 401–500 from memory. IDs are unique and increasing per pod but **not globally ordered**, and a pod crash wastes its unused range — acceptable for IDs, not for gap-free invoice numbers. Pitfalls: `INCR` on a non-integer value errors; values are 64-bit signed; if Redis fails over asynchronously, a counter can roll back slightly, so use a DB sequence if you need a strict guarantee.

---

## 6. Pub/Sub vs Streams

### Q25. How does Redis Pub/Sub work and what are its limits?

`SUBSCRIBE channel` / `PUBLISH channel msg` / `PSUBSCRIBE news.*`. Delivery is **fire-and-forget, at-most-once**:

- No persistence — a subscriber that is offline **misses messages permanently**.
- No acks, no replay, no consumer groups, no backpressure (slow consumers get disconnected via `client-output-buffer-limit`).
- In Cluster, `PUBLISH` is broadcast to all nodes (use `SPUBLISH`/`SSUBSCRIBE` — sharded pub/sub, Redis 7 — to avoid the fan-out cost).

**Legitimate uses:** cache-invalidation broadcasts, live config reload, presence/typing indicators, WebSocket fan-out across app instances, Spring Session/Actuator events.
**Not for:** anything you must not lose → use Streams, Kafka, or RabbitMQ.

**How it works, plainly:** a *channel* is just a name. Subscribers register interest in it on their connection; when someone runs `PUBLISH`, Redis immediately pushes the message to every connection currently subscribed and then forgets it — nothing is stored. `PSUBSCRIBE news.*` is a **pattern** subscription (glob match on channel names). A connection in subscribe mode can't run normal commands (under RESP2), so Spring uses a dedicated connection for the listener container.

```
# terminal A                      # terminal B
SUBSCRIBE cache:invalidate:product
                                  PUBLISH cache:invalidate:product 9987   → (integer) 1   (1 = receivers)
→ "message" "cache:invalidate:product" "9987"
```

**Example scenario:** 10 pods each hold a Caffeine L1 cache. When product 9987 changes, one pod publishes `9987` on `cache:invalidate:product`; all 10 pods' listeners evict it from their local cache. If a pod is restarting at that moment it misses the message — which is why L1 entries also get a short TTL as a safety net.

**Probe:** *"Why does a slow subscriber get disconnected?"* — Redis buffers pending messages in memory per client; if the buffer exceeds `client-output-buffer-limit pubsub` it drops the client rather than exhaust RAM.

**Spring Boot subscriber:**

```java
@Bean
RedisMessageListenerContainer container(RedisConnectionFactory cf, MessageListenerAdapter adapter) {
    RedisMessageListenerContainer c = new RedisMessageListenerContainer();
    c.setConnectionFactory(cf);
    c.addMessageListener(adapter, new PatternTopic("cache:invalidate:*"));
    return c;
}

@Bean
MessageListenerAdapter adapter(CacheInvalidationHandler h) {
    return new MessageListenerAdapter(h, "handleMessage");
}
```

🧠 **Memory hook:** *"Pub/Sub is a radio broadcast — if your radio is off, you missed the song. Streams are a DVR."*

---

### Q26. Show a Redis Streams consumer group in Spring Boot.

```java
// producer
ObjectRecord<String, OrderEvent> record =
        StreamRecords.newRecord().ofObject(event).withStreamKey("orders.stream");
RecordId id = redisTemplate.opsForStream().add(record);

// consumer group (create once, ignore BUSYGROUP)
redisTemplate.opsForStream().createGroup("orders.stream", ReadOffset.from("0"), "order-workers");

// listener container
StreamMessageListenerContainerOptions<String, ObjectRecord<String, OrderEvent>> options =
    StreamMessageListenerContainerOptions.builder()
        .pollTimeout(Duration.ofSeconds(2))
        .targetType(OrderEvent.class)
        .build();

StreamMessageListenerContainer<String, ObjectRecord<String, OrderEvent>> container =
        StreamMessageListenerContainer.create(connectionFactory, options);

container.receive(                      // manual ack variant: receiveAutoAck vs receive
        Consumer.from("order-workers", "worker-1"),
        StreamOffset.create("orders.stream", ReadOffset.lastConsumed()),
        msg -> {
            handle(msg.getValue());
            redisTemplate.opsForStream().acknowledge("order-workers", msg);
        });
container.start();
```

Mention: **`XPENDING`/`XAUTOCLAIM`** to recover messages from a dead consumer, `XADD ... MAXLEN ~ 100000` to cap memory, and that Streams give **at-least-once** delivery — so consumers must be **idempotent**.

**Walking through the code:**
1. *Producer* — `add(record)` is `XADD`; Redis returns an auto-generated `RecordId` (time-seq).
2. *`createGroup(..., ReadOffset.from("0"), ...)`* — creates the consumer group `order-workers`; `"0"` means "deliver the stream from the beginning" (use `$` to get only new messages). Creating it twice throws `BUSYGROUP`, so catch and ignore it at startup.
3. *Listener container* — Spring runs a background polling loop that issues `XREADGROUP ... BLOCK`, waits up to `pollTimeout` for data, and calls your lambda for each record. `ReadOffset.lastConsumed()` is the `>` marker: "only messages never delivered to anyone in this group".
4. *Consumer name* (`worker-1`) must be **unique per pod** (e.g. hostname) so Redis can track each worker's pending entries.
5. *Acknowledge* — after successful handling you `XACK`; skip it and the entry stays in the **PEL** (pending list) for redelivery. `receiveAutoAck` acks on delivery (at-most-once) — avoid it for important work.

**Pitfalls:** if `handle()` throws, no ack happens and the message sits pending until someone `XAUTOCLAIM`s it — run a scheduled reclaimer, and move messages with too many deliveries (`XPENDING` shows delivery count) to a dead-letter stream.

---

## 7. Replication, Sentinel & Cluster

### Q27. How does Redis replication work?

- Asynchronous, primary→replica, one primary can have many replicas; replicas can chain (sub-replicas).
- **Full sync:** replica sends `PSYNC`; primary forks and streams an RDB, buffering subsequent writes, then sends the buffer.
- **Partial resync:** using the **replication ID + offset** and the primary's in-memory **replication backlog** (`repl-backlog-size`), a briefly disconnected replica catches up without a full RDB. Redis 4+ keeps the replication ID across failover so a promoted replica doesn't force full syncs.
- Replicas are read-only by default (`replica-read-only yes`).
- **`WAIT numreplicas timeout`** blocks until N replicas ack the write — gives you tunable durability, but it is *not* consensus and does not prevent all loss.

**What this means in plain language:** replication exists for (1) **high availability** — a copy to promote if the primary dies, and (2) **read scaling** — spread reads over replicas. *Asynchronous* means the primary replies "OK" to your client **before** replicas have the write; that keeps writes fast but opens a small loss window.

**Step by step when a replica joins:** (1) `REPLICAOF primary 6379`; (2) replica sends `PSYNC` with its replication ID and offset; (3) if the primary can't serve the missing range from its backlog, it does a **full sync** — forks, writes an RDB, ships it, and meanwhile buffers new writes; (4) replica loads the RDB, then replays the buffered writes; (5) from then on the primary streams every write command continuously. If the link drops for a few seconds, the replica reconnects and asks "I'm at offset X" — if X is still within the backlog buffer, only the missing commands are sent (**partial resync**), cheaply.

**Pitfalls:** a backlog that's too small turns every blip into a costly full sync (fork + big transfer, which can overload the primary); reading from a replica can return **stale** data (replication lag — check `INFO replication` `master_repl_offset` vs replica offset); a write you just made may not be visible when you read it from a replica (read-your-own-writes issue).

**Critical honesty point:** because replication is async, a failover can lose writes already acknowledged to your client. Redis is not a linearizable store. If a question asks "can you use Redis as the source of truth for money?", the answer is *no* — use it as a cache/coordination layer, with an authoritative DB behind it.

---

### Q28. What is Redis Sentinel and how does failover work?

Sentinel is a separate set of processes providing **monitoring, notification, automatic failover and service discovery** for a primary/replica setup (not sharding).

Flow: a Sentinel marks the primary **SDOWN** (subjectively down) after `down-after-milliseconds`; when a **quorum** of Sentinels agrees it becomes **ODOWN** (objectively down); Sentinels elect a leader (Raft-like); the leader picks the best replica (by `replica-priority`, replication offset, run-id), issues `REPLICAOF NO ONE`, repoints the other replicas, and pushes the new topology to clients.

**Deploy at least 3 Sentinels across different failure domains** (quorum needs a majority; 2 gives you nothing).

**Why Sentinel exists:** plain replication does not promote anyone when the primary dies — a human would have to. Sentinel automates that and also acts as the "phone book": clients ask Sentinel *"who is the current primary of `mymaster`?"* instead of hard-coding an IP, so after failover they find the new one.

**Terms:** *SDOWN* = one Sentinel thinks the primary is down (its own opinion). *ODOWN* = at least `quorum` Sentinels agree, so failover may start. **Quorum** is the number of Sentinels that must agree the primary is unreachable (e.g. 2 of 3); a **majority** of all Sentinels is separately required to authorise the failover. Example: 3 Sentinels, quorum 2 — if the primary's network is cut, Sentinels A and B both stop getting PONGs, mark ODOWN, elect A as leader, A promotes the replica with the highest replication offset (least data loss), and tells clients.

**Pitfalls:** all 3 Sentinels on the same host/AZ defeats the purpose; async replication means the promoted replica may lack the last writes; a partitioned old primary can keep accepting writes from isolated clients (split-brain) — mitigate with `min-replicas-to-write 1` and `min-replicas-max-lag 10`, which make an isolated primary refuse writes.

**Spring Boot config:**

```yaml
spring:
  data:
    redis:
      sentinel:
        master: mymaster
        nodes: 10.0.1.10:26379,10.0.1.11:26379,10.0.1.12:26379
      password: ${REDIS_PASSWORD}
      lettuce:
        pool:
          max-active: 32
```

Lettuce subscribes to Sentinel and reconnects to the new primary automatically. To read from replicas: `LettuceClientConfiguration` with `ReadFrom.REPLICA_PREFERRED`.

---

### Q29. Explain Redis Cluster: hash slots, resharding, limitations.

- **16384 hash slots**; slot = `CRC16(key) mod 16384`. Each primary owns a slot range; each primary has ≥1 replica.
- Clients are **cluster-aware**: on a wrong-node request they get a **`MOVED`** redirect (permanent, update the map) or **`ASK`** (temporary, during slot migration).
- **Resharding** moves slots between nodes online (`redis-cli --cluster reshard`), key by key with `MIGRATE`.
- **Failover:** replicas detect primary failure via gossip; a majority of primaries must agree; a replica promotes itself. Needs ≥3 primaries for a meaningful majority (so **6 nodes minimum** with replicas).
- `cluster-require-full-coverage yes` (default) makes the cluster refuse writes if any slot is uncovered; many teams set it to `no` for partial availability.

**Why hash slots?** One machine can't hold or serve everything, so data must be split (sharded). Instead of mapping each key to a node directly (which would force huge reshuffles when nodes are added), Redis first maps every key to one of 16,384 fixed **slots**, then maps slots to nodes. Adding a node just means moving some *slots* (and the keys in them) to it — online, while serving traffic.

**Example:** three primaries own slots 0–5460, 5461–10922, 10923–16383. `CRC16("user:1001") % 16384 = 7000` → node B. A client that sends `GET user:1001` to node A gets `-MOVED 7000 10.0.0.2:6379`; smart clients (Lettuce, Jedis Cluster) cache the slot→node map and go straight to B next time. During a migration of slot 7000, a key already moved yields `-ASK`, meaning "try the other node just for this one request".

**Hash-tag example:** `{order:55}:items` and `{order:55}:total` both hash only `order:55`, so a `MULTI` or Lua script touching both works. Pitfall: tag everything with one constant (like `{app}`) and you put all data in one slot — you've recreated a single node with extra steps.

**Limitations (the part interviewers probe):**
- **Multi-key operations must be in one slot** — `MGET`, `SUNION`, `ZUNIONSTORE`, `MULTI`, Lua with multiple `KEYS`. Solve with **hash tags**: `{user:1001}:profile` and `{user:1001}:cart` hash only the part in braces, so they land in the same slot.
- Only **database 0** — no `SELECT`.
- Pub/Sub fans out cluster-wide unless you use sharded pub/sub.
- Cross-slot pipelining must be split by the client.

🧠 **Memory hook:** *"16384 slots, CRC16, MOVED/ASK, and hash tags `{}` to keep friends together."*

---

### Q30. Cluster vs Sentinel vs single instance — how do you decide?

| Need | Choice |
|---|---|
| Small dataset, HA not critical, dev/test | Single instance (+ RDB) |
| Dataset fits in one node's RAM, need automatic failover | **Primary/replica + Sentinel** |
| Dataset > one node's RAM, or write throughput > one node | **Cluster** |
| Managed, minimal ops | ElastiCache / MemoryDB / Azure Cache / Redis Enterprise Cloud |

Say the quiet part: "I default to Sentinel unless the data genuinely doesn't fit or the write rate demands sharding, because Cluster adds multi-key constraints that leak into application code."

**Reasoning behind each row:** the two problems are different — **availability** (survive a node dying → replicas + Sentinel/Cluster failover) and **scale** (more RAM or write throughput than one node → sharding, only Cluster does that). Sentinel does not split data: every node holds the *whole* dataset, so capacity is limited by one machine's RAM. Cluster splits data but costs you: multi-key ops limited to one slot, no `SELECT` databases, harder local dev/testing, more complex client config.

**Examples:** a session store of 8 GB with 5k ops/sec → one primary + 2 replicas + 3 Sentinels, simple. A 300 GB feature store with 400k writes/sec → Cluster (e.g. 12 primaries). Pitfall: choosing Cluster "for scale" up front and then writing code with cross-key Lua or `MULTI` that you must rework with hash tags.

---

### Q31. Does Redis satisfy CAP? What consistency does it give?

Redis Cluster is effectively **AP-leaning with asynchronous replication**: during a partition, the majority side stays available and the minority side stops accepting writes after `cluster-node-timeout`, but writes acknowledged on a failing primary can be lost. There is no quorum write path. **MemoryDB** (AWS) is the exception — it uses a multi-AZ transaction log for durable, consistent writes.

Practical framing: *"Redis gives you availability and speed with eventual consistency and a small write-loss window. Design for it: idempotent consumers, a durable DB of record, and reconciliation."*

**Terms and example:** **CAP** says that when a network **P**artition happens you must choose between **C**onsistency (every read sees the latest write) and **A**vailability (every request gets an answer). *Example failure:* the primary is cut off from its replicas, clients on its side keep writing `SET balance 50` and get "OK"; the majority side promotes a replica that never saw it; when the partition heals, the old primary is demoted and its extra writes are **discarded**. The client was told "OK" but the data is gone — that's the write-loss window. `cluster-node-timeout` (e.g. 15 s) bounds how long the minority keeps accepting writes. For tighter guarantees you can use `WAIT 1 100` after critical writes or `min-replicas-to-write`, but neither makes Redis linearizable.

---

## 8. Distributed Locking

### Q32. How do you implement a distributed lock in Redis? Walk through the pitfalls.

**Naïve and wrong:**
```
SETNX lock:order:1 1        # then EXPIRE separately  ← NOT atomic; crash between them = permanent deadlock
```

**Correct primitive:**
```
SET lock:order:1 <randomUUID> NX PX 30000
```
- `NX` = only if absent (mutual exclusion), `PX` = TTL (deadlock protection), **value = unique token** (so you only release *your* lock).

**Release must be atomic** — compare-and-delete via Lua, otherwise you delete a lock that another client acquired after yours expired:

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
else
  return 0
end
```

**The pitfalls to name (this is the differentiator):**
1. **No TTL** → deadlock on crash.
2. **TTL too short** → lock expires mid-work → two workers run concurrently. Fix: a **watchdog** that renews while work continues (Redisson does this automatically).
3. **Deleting someone else's lock** → must use a unique token + Lua CAD.
4. **Non-reentrancy** → nested calls self-deadlock. Redisson's `RLock` is reentrant (hash + counter).
5. **Async replication** → primary acquires the lock, dies before replicating, replica is promoted, second client acquires the same lock. **A single-instance Redis lock is not safe under failover.**
6. **GC pause / process freeze** → even a correct algorithm can't stop a stalled JVM from thinking it still holds the lock. Hence: make the protected operation **idempotent** or use **fencing tokens** (monotonic token checked by the downstream resource).

**Redisson (what I'd actually use in Spring Boot):**

```java
RLock lock = redisson.getLock("lock:order:" + orderId);
try {
    if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {   // wait 5s, lease 30s
        processOrder(orderId);
    } else {
        throw new LockAcquisitionException(...);
    }
} finally {
    if (lock.isHeldByCurrentThread()) lock.unlock();
}
```

Also mention `RedissonRedLock`/`RedissonMultiLock`, `RReadWriteLock`, `RSemaphore`, `RCountDownLatch`, and Spring Integration's `RedisLockRegistry` if you don't want Redisson.

🧠 **Memory hook — lock checklist "U-T-A-R-F":** **U**nique token, **T**TL, **A**tomic release (Lua), **R**enewal/watchdog, **F**encing/idempotency.

---

### Q33. What is Redlock and is it safe?

Redlock acquires the lock on a **majority of N independent Redis primaries** (typically 5), each with a TTL, and considers the lock held if the majority succeeded within a validity window minus clock drift. It removes the single-primary failover hole.

**Algorithm in steps:** (1) note the start time; (2) try `SET lock <token> NX PX <ttl>` on each of the 5 independent nodes in turn, using a short per-node timeout (so a dead node doesn't stall you); (3) the lock is acquired only if **≥3 of 5** succeeded **and** the elapsed time is still less than the TTL (the remaining time is the *validity window*); (4) if it fails, release on *all* nodes. The 5 nodes are separate masters (no replication between them), so one dying or failing over can't hand the lock to a second client.

**Why "independent" matters:** with one primary + replica, async replication can lose the lock on failover (Q32, pitfall 5). With 5 independent nodes, two clients would both need a majority, and any two majorities overlap in at least one node, which refuses the second client. **Why people still doubt it:** the safety argument assumes clocks tick at roughly the same rate and that a client won't be paused (GC, VM freeze) between acquiring and using the lock. In Redisson it's `RedissonRedLock` (or `RedissonMultiLock`) over several `RLock` objects from different clients; note newer Redisson docs deprecate RedLock in favour of plain `RLock`.

Then give the balanced view: **Martin Kleppmann's critique** — Redlock relies on bounded clock drift and bounded process pauses, so it is not safe as a *correctness* lock; he recommends fencing tokens from a consensus system (ZooKeeper/etcd). **Salvatore Sanfilippo's response** is that with reasonable assumptions it's fine for efficiency-oriented locking.

**My stance to state in an interview:** *"Use Redis locks for efficiency (don't do the same work twice), not for correctness (don't corrupt data). If correctness truly depends on mutual exclusion, put the guard where the state lives — a DB unique constraint, optimistic version column, or conditional write — or use ZooKeeper/etcd."*

---

### Q34. Idempotency with Redis — how?

```java
// idempotency key from the client / message id
Boolean first = redisTemplate.opsForValue()
        .setIfAbsent("idem:" + requestId, "1", Duration.ofHours(24));
if (Boolean.FALSE.equals(first)) {
    return cachedResponseOrConflict();
}
```
Store the *result* too (`SET idem:{id} <responseJson> NX EX 86400`) so retries return the same response rather than a 409. This is the standard payments/webhook pattern and a very common scenario question.

**What idempotency means:** an operation is *idempotent* if running it twice has the same effect as once. Networks retry (client timeouts, webhook redelivery, Kafka at-least-once), so "charge the card" must not charge twice. The client sends a unique `Idempotency-Key` header per logical action; the server uses Redis `SET NX` as an atomic "have I seen this key?" check — only the first caller gets `true`.

**Fuller flow (three states):**
1. `SET idem:{id} IN_PROGRESS NX EX 60` → wins? you own the work. Loses? read the value:
2. value is `IN_PROGRESS` → a concurrent duplicate; return `409`/"retry shortly".
3. value is a stored response → return it verbatim.
4. On success overwrite with the response JSON and a long TTL (24 h); on failure `DEL` the key so the client can retry.

**Pitfalls:** if the process crashes after charging but before storing the result, the short `IN_PROGRESS` TTL expires and a retry could double-charge — so make the downstream call itself idempotent too (pass the same key to the payment gateway) and keep a **DB unique constraint** on the request id as the real guard; Redis eviction or failover can lose keys, so don't treat it as the only protection for money.

---

## 9. Caching Patterns

### Q35. Explain the main caching strategies and their trade-offs.

| Pattern | Flow | Pros | Cons |
|---|---|---|---|
| **Cache-Aside (Lazy Loading)** — 90% of real systems | App reads cache → miss → read DB → write cache | Simple, only caches what's used, cache failure is survivable | First request slow; stale data window; stampede risk on miss |
| **Read-Through** | App asks the cache library; it loads from DB on miss | Clean app code (`@Cacheable` is essentially this) | Needs a loader; same staleness issues |
| **Write-Through** | Write goes to cache and DB synchronously | Cache always fresh; no stale reads | Higher write latency; caches data that may never be read |
| **Write-Behind (Write-Back)** | Write to cache, async flush to DB | Fastest writes, absorbs bursts, batches DB writes | **Data loss if Redis dies before flush**; complex; ordering issues |
| **Write-Around** | Writes go only to DB; cache filled on read | Good for write-heavy, read-rare data | First read always misses |
| **Refresh-Ahead** | Proactively refresh hot keys before TTL | Avoids stampede for predictable hot keys | Wasted refreshes; needs prediction |

**What I say:** *"Default to cache-aside with TTL plus explicit invalidation on write. I only reach for write-behind when the write rate genuinely exceeds the DB's capacity and losing a few seconds of non-critical data (like view counts) is acceptable."*

🧠 **Memory hook:** *Aside = lazy, Through = eager, Behind = risky, Around = skip, Ahead = predictive.*

**Cache-Aside in code (the one you'll actually write):**

```java
public Product getProduct(long id) {
    String key = "product:" + id;
    Product cached = (Product) redisTemplate.opsForValue().get(key);   // 1. look in cache
    if (cached != null) return cached;                                  // hit
    Product p = productRepository.findById(id).orElseThrow();           // 2. miss → DB
    redisTemplate.opsForValue().set(key, p, Duration.ofMinutes(10));    // 3. populate with TTL
    return p;
}
public void updateProduct(Product p) {
    productRepository.save(p);                                          // DB first
    redisTemplate.delete("product:" + p.getId());                       // then invalidate (see Q41)
}
```

**Why so many patterns?** They differ in *who is responsible* for keeping cache and DB in sync (the app, the cache library, or a background flusher) and in whether they favour read speed, write speed or safety. **Write-Behind example:** a "likes" counter is incremented in Redis and a job flushes totals to the DB every 5 s — fast, but a Redis crash loses up to 5 s of likes (fine for likes, not for payments). **Write-Through example:** a user-profile update writes DB and cache together so the next read is always fresh, at the cost of slower writes.

---

### Q36. How do you choose TTLs?

- Base it on **tolerance for staleness × cost to recompute × rate of change**.
- Reference/master data → hours to days. User-session-ish data → minutes. Prices/inventory → seconds or event-driven invalidation only.
- **Always add jitter**: `ttl = base + random(0, base * 0.1)` to prevent synchronised mass expiry (avalanche).
- Layer TTLs: L1 Caffeine 30 s + L2 Redis 10 min, so a stale L1 self-heals quickly.
- Negative caching (cache "not found" for 30–60 s) prevents penetration.

**How to apply the formula:** ask three questions — *How wrong can this data be before a user notices or money is lost? How expensive is a miss (DB query time, downstream API cost)? How often does it really change?* A country list (never changes, cheap) → 24 h. A product price (changes a few times a day, wrong price is costly) → 30–60 s, or no TTL but invalidate on every price update. A user's session → sliding TTL of 30 min that is refreshed on activity.

**Jitter in Spring Boot:**
```java
Duration ttl = Duration.ofMinutes(10).plusSeconds(ThreadLocalRandom.current().nextInt(0, 60));
redisTemplate.opsForValue().set(key, value, ttl);
```
**Why:** if 100k keys are loaded at the same moment with an identical 10-min TTL, they all expire at the same moment, producing a DB spike (see Q39). **Pitfall:** a TTL is a *safety net* for staleness, not the primary consistency mechanism — pair it with explicit invalidation (Q41).

---

## 10. The Four Classic Cache Failures

This set is asked in almost every senior interview. Memorise the four names and one fix each.

### Q37. Cache Penetration (穿透) — queries for data that doesn't exist

**Problem:** Requests for keys that exist neither in cache nor in DB (often malicious: `GET /user/-99999`) hit the database every single time.

**Fixes:**
1. **Cache the null** — `SET user:-99999 "__NULL__" EX 60` (short TTL to bound wrongness).
2. **Bloom filter** in front (RedisBloom `BF.EXISTS`, or Guava BloomFilter locally): "definitely not present" → reject without touching the DB.
3. **Input validation** — reject impossible IDs at the edge; auth/rate-limit the endpoint.

**Why it's a problem:** the cache only helps for keys that *have* a value. A key that doesn't exist is never stored, so each request is a guaranteed miss and goes to the DB. One attacker (or buggy client) looping over random IDs bypasses the cache entirely.

**How the Bloom filter works:** a **Bloom filter** is a compact bit array plus several hash functions. Adding an ID sets a few bits; checking an ID tests those bits. If any bit is 0 the ID is *definitely not* present; if all are 1 it is *probably* present (small false-positive rate, **never** false negatives). So you preload all valid IDs, and requests for IDs the filter rejects never reach the DB. Pitfalls: standard Bloom filters can't delete items (rebuild periodically) and new rows must be added to the filter when created.

```java
Object v = redisTemplate.opsForValue().get(key);
if ("__NULL__".equals(v)) return Optional.empty();   // negative-cache hit, no DB call
```

### Q38. Cache Breakdown / Stampede (击穿) — one hot key expires

**Problem:** A single very hot key expires and thousands of concurrent requests all miss and hammer the DB simultaneously.

**Fixes:**
1. **Mutex / single-flight** — first thread takes a distributed lock (`SET NX PX`) and rebuilds; others briefly sleep-and-retry or serve stale.
2. **Logical expiry** — never physically expire the hot key; store `{value, logicalExpireAt}` and let one thread refresh asynchronously while others read the slightly stale value.
3. **Refresh-ahead** background job for known hot keys.

**Why it's different from penetration:** here the data *exists* — it's just that the moment its TTL hits, every concurrent request misses together. If 5,000 req/s read `homepage:banner` and the DB query takes 200 ms, about 1,000 requests all run the same expensive query before the first one finishes re-populating the cache.

**Mutex example (double-checked):**
```java
String v = get(key);
if (v == null) {
    String lockKey = "lock:" + key;
    if (Boolean.TRUE.equals(redisTemplate.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(10)))) {
        try {
            v = loadFromDb();                    // only ONE caller rebuilds
            set(key, v, ttlWithJitter());
        } finally { redisTemplate.delete(lockKey); }
    } else {
        Thread.sleep(50);                        // others wait briefly, then re-read the cache
        return get(key);
    }
}
```
**Logical expiry explained:** the Redis TTL is removed; the value carries its own "fresh until" timestamp. A reader that sees it's past that time returns the old value immediately and triggers one background refresh — users never wait, at the price of brief staleness. (In Spring, `@Cacheable(sync = true)` gives per-JVM single-flight only, not cluster-wide.)

### Q39. Cache Avalanche (雪崩) — many keys expire at once, or Redis goes down

**Problem:** Mass simultaneous expiry (e.g. everything loaded at deploy with the same 1-hour TTL) or a full Redis outage sends 100% of traffic to the DB, which collapses.

**Fixes:**
1. **TTL jitter / staggered warming.**
2. **Redis HA** — Sentinel/Cluster so a node loss isn't a cache loss.
3. **Circuit breaker + bulkhead + rate limiting** (Resilience4j) between app and DB; fail fast with degraded responses instead of queueing.
4. **Multi-level cache** — a local Caffeine L1 survives a Redis outage.
5. **Graceful degradation** — serve stale data, a static fallback, or a "try again shortly" response rather than melting the DB.

**Two flavours, one result:** (a) *many keys expire together* — a scheduled/deploy-time warm-up loaded 1M keys with the same TTL; (b) *Redis itself is gone* — every request becomes a miss. In both cases the DB, sized for ~5% of traffic, suddenly gets ~100% and fails, and then the cache can't be re-filled because the DB is down — a **cascading failure**.

**How the defences fit:** jitter prevents (a); HA reduces (b); a **circuit breaker** (Resilience4j) stops calling the DB after repeated failures so it can recover, and returns a fallback; a **bulkhead** caps concurrent DB calls so one hot path can't exhaust the connection pool. Example Resilience4j config: `@CircuitBreaker(name="productDb", fallbackMethod="stale")` with `failureRateThreshold=50`, `waitDurationInOpenState=10s`.

### Q40. Hot Key / Big Key

Covered in Q19: hot key → local cache, key replication, replica reads, client-side caching. Big key → shard, compress, externalise, and read in pages.

**One-line distinction:** a *hot* key has too many **requests**; a *big* key has too many **bytes**. They need opposite fixes — hot keys are *copied/cached* to spread reads, big keys are *split* so no single operation is heavy. A key can be both (a celebrity's 100 MB follower list), in which case split it first and then cache the hot slices.

🧠 **Memory hook — "P-B-A-H":** **P**enetration (doesn't exist → null-cache/bloom), **B**reakdown (one hot key expired → mutex/logical expiry), **A**valanche (many expired / Redis down → jitter + HA + circuit breaker), **H**ot key (too popular → local cache + replicas).

---

## 11. Cache–Database Consistency

### Q41. Update the DB and the cache — what order, and why?

Four candidate orders. Know why three are worse:

1. **Update DB, then update cache** ❌ — concurrent writers can apply cache updates out of order (writer A's DB write lands first but its cache write lands last), leaving the cache permanently wrong. Also caches values nobody may read.
2. **Update cache, then update DB** ❌❌ — worst: if the DB write fails, the cache holds data that never existed.
3. **Delete cache, then update DB** ⚠️ — a reader between the delete and the commit reloads the *old* value and re-caches it → long-lived staleness. Mitigated by **delayed double delete**.
4. **Update DB, then delete cache (Cache-Aside invalidation)** ✅ — the industry default. The failure window is tiny (only a reader that loaded before the commit and writes after the delete), and it doesn't waste work.

**So: write to the DB, then `DEL`/`UNLINK` the key. Not `SET`.** Deleting is idempotent and order-insensitive; setting is not.

**Handling the remaining edge cases:**
- **Delete fails / app crashes after commit** → retry via an outbox + message queue, or subscribe to the DB binlog/WAL (Debezium/Canal → Kafka → cache invalidator). This is the robust production answer.
- **Delayed double delete**: `DEL` → update DB → sleep ~500 ms asynchronously → `DEL` again, to kill any stale value re-cached in between.
- **TTL as the safety net** — always. Even if invalidation logic has a bug, the wrongness is bounded.
- **Read-your-own-writes** for the writing user: route them to the DB (or update their local view) for a short window.
- **Strong consistency needed?** Then don't cache it, or serialise reads/writes on a lock — and admit the throughput cost.

🧠 **Memory hook:** *"DB first, then delete (never update) the cache; TTL is the seatbelt; binlog/outbox is the airbag."*

---

### Q42. How do you keep two-level (Caffeine + Redis) caches consistent?

L1 in each JVM will drift. Standard solution: on invalidation, `PUBLISH cache:invalidate:{cacheName} {key}`; every instance's `RedisMessageListenerContainer` evicts its local entry. Keep L1 TTL short (10–60 s) as a fallback in case a pub/sub message is missed (it's at-most-once!). Redis 6 **client-side caching (tracking/invalidation via RESP3)** does this natively and is supported by Lettuce.

**Why two levels?** L1 (Caffeine, in the JVM heap) answers in ~100 ns with no network; L2 (Redis) is shared by all pods at ~0.5–1 ms. Together they cut Redis load and survive Redis blips. The catch: with 10 pods you have 10 independent L1 copies, and Redis can't reach into them. If pod 3 updates product 9987 and evicts it from Redis, pods 1–2 and 4–10 keep serving the old value from heap until it expires.

**Flow to describe:** (1) writer updates the DB; (2) deletes key in Redis (L2) and its own L1; (3) publishes `9987` on `cache:invalidate:product`; (4) every pod's listener evicts `9987` from its L1; (5) next read repopulates L1 from L2/DB. Spring's `CacheManager` doesn't do this out of the box — libraries such as J2Cache/JetCache, or your own `CompositeCacheManager` plus the listener from Q25, implement it. **Pitfall:** pub/sub is at-most-once, so the short L1 TTL is mandatory, and the eventual-consistency window equals that TTL.

---

## 12. Spring Boot + Redis (Deep Dive)

### Q43. Jedis vs Lettuce vs Redisson — which and why?

| | Jedis | **Lettuce** (Spring Boot default since 2.0) | Redisson |
|---|---|---|---|
| Model | Blocking, **not thread-safe** → needs a connection pool (`JedisPool`) | Netty-based, **thread-safe**, one connection multiplexed; sync/async/reactive | High-level: distributed objects & locks |
| Reactive | No | Yes (`ReactiveRedisTemplate`, WebFlux) | Partially |
| Cluster/Sentinel | Yes | Yes, with topology refresh & `ReadFrom` | Yes |
| Best for | Simple/legacy, fine-grained control | Default choice for Spring Boot | Locks, semaphores, `RMap`, `RBucket`, delayed queues, JCache |

**Say:** *"Lettuce by default — it's non-blocking, thread-safe and Spring Boot's default. I add Redisson alongside it when I need real distributed primitives (reentrant locks with watchdog, rate limiters, delayed queues) because hand-rolling those correctly is where bugs live. Jedis only in legacy code."*

Note: with Lettuce you often don't need a pool at all (shared multiplexed connection), **except** for blocking commands (`BLPOP`), transactions (`MULTI` binds a connection), and `SELECT`-per-tenant usage — then enable `spring.data.redis.lettuce.pool`.

**Why the thread-safety row matters:** a Redis connection carries a strict request→reply stream. Jedis's connection object isn't thread-safe, so two threads sharing one would interleave bytes and corrupt replies — each thread must borrow its own connection from a pool (`JedisPool`), costing a pooled socket per concurrent caller. Lettuce is built on **Netty** (an async network framework): many threads can submit commands to one connection, which are pipelined and matched to replies in order. That's why 200 Tomcat threads can share a single Lettuce connection. A blocking command like `BLPOP` would hold that shared connection hostage, hence the exception above. **Redisson** sits on top as a "Java collections over Redis" library: `RMap<String,User>` behaves like a `ConcurrentMap` stored in Redis.

---

### Q44. Show a production-grade `RedisTemplate` / cache configuration.

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory cf) {
        RedisTemplate<String, Object> t = new RedisTemplate<>();
        t.setConnectionFactory(cf);

        ObjectMapper mapper = JsonMapper.builder()
                .addModule(new JavaTimeModule())                 // Instant/LocalDate support
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
                .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
                .activateDefaultTyping(                          // keeps polymorphic types
                        BasicPolymorphicTypeValidator.builder()
                                .allowIfSubType("com.mycompany.")
                                .allowIfSubType("java.util.")
                                .build(),
                        ObjectMapper.DefaultTyping.NON_FINAL,
                        JsonTypeInfo.As.PROPERTY)
                .build();

        GenericJackson2JsonRedisSerializer json = new GenericJackson2JsonRedisSerializer(mapper);
        t.setKeySerializer(new StringRedisSerializer());
        t.setValueSerializer(json);
        t.setHashKeySerializer(new StringRedisSerializer());
        t.setHashValueSerializer(json);
        t.afterPropertiesSet();
        return t;
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory cf, RedisTemplate<String, Object> t) {
        RedisCacheConfiguration base = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .disableCachingNullValues()          // or keep enabled for penetration defence
                .prefixCacheNameWith("app:v1:")      // versioned prefix = instant mass invalidation
                .serializeKeysWith(SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(SerializationPair.fromSerializer(
                        (RedisSerializer<Object>) t.getValueSerializer()));

        Map<String, RedisCacheConfiguration> perCache = Map.of(
                "products",  base.entryTtl(Duration.ofHours(6)),
                "inventory", base.entryTtl(Duration.ofSeconds(30)),
                "users",     base.entryTtl(Duration.ofMinutes(15)));

        return RedisCacheManager.builder(cf)
                .cacheDefaults(base)
                .withInitialCacheConfigurations(perCache)
                .transactionAware()                  // only write cache after TX commit
                .build();
    }

    /** Don't let a Redis outage break the app: log and fall through to the DB. */
    @Bean
    public CacheErrorHandler errorHandler() {
        return new SimpleCacheErrorHandler() {
            @Override public void handleCacheGetError(RuntimeException e, Cache c, Object k) {
                log.warn("Redis GET failed, falling through cache={} key={}", c.getName(), k, e);
            }
            @Override public void handleCachePutError(RuntimeException e, Cache c, Object k, Object v) {
                log.warn("Redis PUT failed cache={} key={}", c.getName(), k, e);
            }
        };
    }
}
```

**Why each line matters (be ready for this):** `StringRedisSerializer` for keys so keys are readable in `redis-cli` (JDK serialization produces `\xac\xed...` garbage); JSON for values so they're language-neutral and inspectable; `activateDefaultTyping` **with a `PolymorphicTypeValidator`** because unrestricted default typing is a deserialization-RCE risk; `prefixCacheNameWith("app:v1:")` so bumping `v1→v2` invalidates everything on deploy; `transactionAware()` so a rolled-back DB transaction doesn't leave a populated cache; `CacheErrorHandler` so Redis is a *performance* dependency, not an *availability* dependency.

---

### Q45. `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching` — explain with gotchas.

```java
@Service
public class ProductService {

    @Cacheable(cacheNames = "products", key = "#id", unless = "#result == null")
    public Product findById(Long id) { return repo.findById(id).orElse(null); }

    @Cacheable(cacheNames = "products", key = "'search:' + #q + ':' + #page.pageNumber",
               condition = "#page.pageNumber < 3")            // only cache the first pages
    public Page<Product> search(String q, Pageable page) { ... }

    @CachePut(cacheNames = "products", key = "#p.id")          // always executes, updates cache
    public Product update(Product p) { return repo.save(p); }

    @CacheEvict(cacheNames = "products", key = "#id")
    public void delete(Long id) { repo.deleteById(id); }

    @CacheEvict(cacheNames = "products", allEntries = true)    // e.g. after a bulk import
    public void reindexAll() { ... }

    @Caching(evict = {
        @CacheEvict(cacheNames = "products", key = "#p.id"),
        @CacheEvict(cacheNames = "productsByCategory", key = "#p.categoryId")
    })
    public Product save(Product p) { return repo.save(p); }
}
```

**Gotchas an interviewer will fish for:**
1. **Self-invocation bypasses the proxy** — calling `this.findById()` from another method in the same class skips the cache. Fix: inject the bean into itself, use `@Resource`/`ObjectProvider`, split classes, or (last resort) `AopContext.currentProxy()`.
2. **Only works on public methods** with Spring's default proxy-based AOP; `final`/`private`/`static` methods aren't advised.
3. **Default key generator** uses all params (`SimpleKey`), which silently collides if you overload methods across the same cache — always set `key` explicitly.
4. `condition` is evaluated **before** invocation (can't see `#result`); `unless` is evaluated **after** (can see `#result`).
5. `@Cacheable` **doesn't cache exceptions** — a failing DB call will be retried every request (that's usually desirable, but it means no negative caching unless you cache nulls).
6. **`allEntries = true` with `RedisCacheManager`** issues a `SCAN`+delete over the prefix — expensive on big caches; prefer a versioned key prefix.
7. `beforeInvocation = false` (default) on `@CacheEvict` means eviction happens *after* success; set `true` if you must evict even when the method throws.
8. Cached values must be **serializable by your configured serializer** — `Page<T>`, `Optional`, JPA lazy proxies and Hibernate collections are frequent explosion sources. Cache DTOs, not entities.

🧠 **Memory hook:** *Cacheable = read, CachePut = write-through, CacheEvict = invalidate, Caching = combine. Self-calls don't count.*

**How it works under the hood:** `@EnableCaching` makes Spring wrap each bean that has these annotations in a **proxy** (an interposed object). A call from outside goes proxy → cache lookup → (on miss) real method → store result. Flow for `findById(5)`: build key `products::5` → `GET` in Redis → hit returns immediately, **method body never runs**; miss runs the body and stores the return value. `@CachePut` always runs the method and overwrites the entry (use it on update/create to refresh); `@CacheEvict` removes it; `@Caching` simply groups several annotations on one method. This is why the self-invocation gotcha exists — `this.findById()` doesn't pass through the proxy.

**Typical wrong/right example (gotcha 1):**
```java
public Product getTwice(Long id) { return this.findById(id); }   // ❌ cache bypassed
// ✅ call through another bean, or inject self: @Lazy @Autowired ProductService self; self.findById(id)
```

---

### Q46. Why shouldn't you cache JPA entities directly?

- Lazy-loaded associations throw `LazyInitializationException` (or serialise the whole graph accidentally).
- Hibernate proxies/`PersistentBag` don't serialise cleanly.
- Detached entities re-attached from cache cause optimistic-lock and identity confusion.
- Schema changes break stored payloads (add a `@JsonIgnoreProperties(ignoreUnknown=true)` and a version prefix).

**Always cache immutable DTOs/records.** This is a great senior-level point to volunteer.

**Why each bullet bites in practice:** a JPA entity is a *live object managed by Hibernate*, not plain data. A `Order` with `@OneToMany List<Item> items` fetched lazily holds a proxy that is only a placeholder until a session loads it. Serialising it to JSON either triggers N extra queries (or an infinite loop through bidirectional links), or throws `LazyInitializationException` because the Hibernate session is already closed. On the way back, the deserialised object is a *detached copy* — its `@Version` field may be stale, so a later `save()` can fail with `OptimisticLockException` or overwrite newer data.

**Better pattern:**
```java
public record ProductDto(Long id, String name, BigDecimal price, int version) {}

@Cacheable(cacheNames = "products", key = "#id")
public ProductDto findById(Long id) {
    return repo.findById(id).map(p -> new ProductDto(p.getId(), p.getName(), p.getPrice(), p.getVersion())).orElse(null);
}
```
The DTO is flat, has no Hibernate dependency, serialises cleanly, and you control exactly what leaves the DB layer (no accidental PII leak into the cache). Include a version in the cache name prefix (`app:v2:`) so a DTO shape change doesn't deserialise old payloads.

---

### Q47. `spring.data.redis` config you should know by heart

```yaml
spring:
  data:
    redis:
      host: redis.internal
      port: 6379
      password: ${REDIS_PASSWORD}
      database: 0
      timeout: 2s                 # command timeout
      connect-timeout: 1s
      client-type: lettuce
      ssl:
        enabled: true
      lettuce:
        shutdown-timeout: 200ms
        pool:
          enabled: true
          max-active: 32          # -1 = unlimited
          max-idle: 16
          min-idle: 4
          max-wait: 500ms
      cluster:
        nodes: n1:6379,n2:6379,n3:6379
        max-redirects: 3
  cache:
    type: redis
    redis:
      time-to-live: 600000
      cache-null-values: true
      use-key-prefix: true
```

Also mention programmatic tuning that isn't in YAML: **cluster topology refresh** and **timeout options**:

```java
ClusterTopologyRefreshOptions refresh = ClusterTopologyRefreshOptions.builder()
        .enablePeriodicRefresh(Duration.ofSeconds(30))
        .enableAllAdaptiveRefreshTriggers()
        .build();
LettuceClientConfiguration cfg = LettuceClientConfiguration.builder()
        .clientOptions(ClusterClientOptions.builder()
                .topologyRefreshOptions(refresh)
                .timeoutOptions(TimeoutOptions.enabled(Duration.ofSeconds(2)))
                .build())
        .readFrom(ReadFrom.REPLICA_PREFERRED)
        .commandTimeout(Duration.ofSeconds(2))
        .build();
```

🧠 **Memory hook:** *Timeouts short, pool bounded, topology refreshed, reads from replicas if you can tolerate staleness.*

**What the important settings actually do:**
- `timeout` (command timeout): how long the client waits for a reply before throwing `RedisCommandTimeoutException`. Too long → threads pile up when Redis is slow; too short → false failures during a GC/fork pause. 1–2 s is typical.
- `connect-timeout`: max time to open the TCP connection — keep it small so a dead node fails fast.
- `pool.max-active / max-idle / min-idle / max-wait`: only used with a pool (needs `commons-pool2` on the classpath). `max-wait` is how long a thread waits for a free connection before failing; `-1` waits forever, which hides outages as thread starvation.
- `ssl.enabled`: TLS to Redis (needed on managed services like ElastiCache with in-transit encryption).
- `cluster.max-redirects`: how many `MOVED/ASK` hops the client follows per command.
- `spring.cache.redis.time-to-live`: default TTL for `@Cacheable` entries in **milliseconds** (600000 = 10 min); without it entries never expire.
- `use-key-prefix`: prefixes keys with the cache name (`products::42`), which avoids collisions between caches.

**Topology refresh, why:** after a Cluster failover or reshard, Lettuce's cached slot map is wrong; without refresh settings it keeps hitting the old node and gets `MOVED` until it eventually corrects — a common cause of error spikes during failover. Adaptive triggers refresh on `MOVED`/reconnect events and periodic refresh covers the rest.

---

### Q48. What is `@RedisHash` / Spring Data Redis repositories, and would you use them?

```java
@RedisHash(value = "person", timeToLive = 3600)
public class Person {
    @Id String id;
    @Indexed String email;      // creates a secondary index set
    String name;
}
public interface PersonRepository extends CrudRepository<Person, String> {
    List<Person> findByEmail(String email);
}
```
Under the hood: each object is a Hash at `person:{id}`, with helper Sets (`person:email:foo@bar.com`) for `@Indexed` fields and a `person:{id}:phantom` key for TTL bookkeeping.

**What this gives you:** the same repository-style API you know from Spring Data JPA (`save`, `findById`, derived `findByEmail`), but backed by Redis. `save(new Person(...))` roughly runs `HSET person:1 name ... email ...` + `SADD person:email:foo@bar.com 1` + `SADD person (the id set)`. `@Indexed` is what makes `findByEmail` possible — Redis has no native secondary index, so Spring maintains a Set per distinct value itself. `timeToLive` sets the key's expiry; the *phantom* copy (a shadow key with a slightly longer TTL) lets Spring still read the old values when the expiry event fires so it can clean up the index Sets.

**Pitfalls:** index cleanup relies on keyspace expiry events (Q51) — if an event is missed, stale entries linger in the index Sets; queries on non-indexed fields aren't supported; writes touch several keys (more round trips; in Cluster they may span slots); enable it explicitly with `@EnableRedisRepositories`.

**My honest take for an interview:** convenient for simple key-value objects and session-like data, but the secondary indexes are maintained by the framework (not the server), so they can drift, and it hides the actual access patterns. For anything performance-critical I use `RedisTemplate`/Redisson with explicit keys.

---

## 13. Rate Limiting, Sessions & Real Use Cases

### Q49. Implement rate limiting in Redis. Compare the algorithms.

| Algorithm | Implementation | Trade-off |
|---|---|---|
| **Fixed window** | `INCR rl:{user}:{minute}` + `EXPIRE 60` | Simplest; allows 2× burst at window boundaries |
| **Sliding window log** | ZSet of request timestamps: `ZREMRANGEBYSCORE` old, `ZCARD`, `ZADD` | Accurate; memory grows with request count |
| **Sliding window counter** | Weighted current + previous fixed window | Good accuracy, O(1) memory |
| **Token bucket** | Hash `{tokens, lastRefill}` + Lua refill | Allows controlled bursts — most "real world" choice |
| **Leaky bucket** | Queue-based, constant drain | Smooths traffic, harder to implement |

**Fixed window in Lua (atomic):**
```lua
local current = redis.call('INCR', KEYS[1])
if current == 1 then redis.call('EXPIRE', KEYS[1], ARGV[2]) end
if current > tonumber(ARGV[1]) then return 0 end
return 1
```

**Sliding window log in Lua:**
```lua
local now, window, limit = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3])
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, now - window)
local count = redis.call('ZCARD', KEYS[1])
if count >= limit then return 0 end
redis.call('ZADD', KEYS[1], now, ARGV[4])          -- ARGV[4] = unique request id
redis.call('PEXPIRE', KEYS[1], window)
return 1
```

**Off-the-shelf options to name:** Redisson `RRateLimiter` (`trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.SECONDS)`), Bucket4j with a Redis backend, Spring Cloud Gateway's `RequestRateLimiterGatewayFilterFactory` (which ships a Redis token-bucket Lua script), or Resilience4j for client-side limits.

**Why rate-limit in Redis (not in app memory)?** With 20 pods, a per-pod counter lets a client make 20× the intended calls. Redis is the shared counter that all pods consult, and Lua makes check-and-increment atomic so concurrent requests can't both slip under the limit.

**Algorithms in plain words:**
- *Fixed window* — "100 requests per calendar minute": the counter key includes the minute, so it resets abruptly. **Boundary problem:** 100 requests at 12:00:59 and 100 more at 12:01:00 = 200 within two seconds, though each window is "legal".
- *Sliding window log* — remember the timestamp of every request in a ZSet and count only those in the last 60 s. Exact, but a user with a 10k limit stores 10k entries.
- *Token bucket* — a bucket holds up to N tokens and refills at R tokens/sec; each request spends one token. A client who has been idle has a full bucket and may **burst** N calls, then is held to the steady rate R. Example: capacity 20, refill 5/s allows a 20-call spike and then 5/s sustained.
- *Leaky bucket* — requests enter a queue that drains at a constant rate; output is perfectly smooth.

**Usage example:** key `rl:{apiKey}:{epochMinute}`, call the Lua script from Spring (`redisTemplate.execute(script, List.of(key), "100", "60")`); on `0` return **HTTP 429 Too Many Requests** with a `Retry-After` header. **Pitfall:** `INCR` then a *separate* `EXPIRE` from Java is not atomic — a crash in between leaves a counter with no TTL (a permanently blocked user); the Lua above avoids that. In Cluster, keep each limiter's keys in one slot.

🧠 **Memory hook:** *"Counter for cheap, ZSet for exact, token bucket for burst-friendly."*

---

### Q50. How does Spring Session with Redis work?

Add `spring-session-data-redis` + `@EnableRedisHttpSession` (or `spring.session.store-type=redis`). Then:
- `HttpSession` is backed by a Redis Hash `spring:session:sessions:{id}`, plus an expirations ZSet/Set for cleanup and `sessionAttr:` fields.
- The servlet container's session is replaced by a `SessionRepositoryFilter`, so **no sticky sessions needed** and app instances become stateless/horizontally scalable — this is the main selling point.
- Configure `maxInactiveIntervalInSeconds`, and `FlushMode.ON_SAVE` (default) vs `IMMEDIATE`.
- **Watch out:** everything in the session must be serializable; keep sessions small; `SessionDestroyedEvent` requires keyspace notifications (`notify-keyspace-events Egx`).

Follow-up they often ask: *"How do you invalidate all sessions for a user?"* → maintain `user:{id}:sessions` as a Set of session IDs, then delete each (Spring Session's `findByPrincipalName` does this via an index).

**Why it exists:** a normal `HttpSession` lives in one server's memory. With several pods behind a load balancer you either pin each user to one pod (*sticky sessions* — uneven load, and the session is lost if that pod dies/redeploys) or share the session store. Spring Session replaces the container's storage with Redis, so any pod can serve any request and a pod restart doesn't log users out.

**How a request flows:** (1) the login response sets a cookie `SESSION=<id>` (a base64 session id); (2) on each request `SessionRepositoryFilter` reads the cookie, loads `spring:session:sessions:<id>` from Redis into a session object, and hands your code a normal `HttpSession`; (3) when the request ends, changed attributes are written back (`HSET`) and the TTL is reset (sliding expiration); (4) after the idle timeout, Redis expires the keys and the user must log in again.

```yaml
spring:
  session:
    timeout: 30m
    redis:
      namespace: myapp:session
      flush-mode: on-save
```
**Pitfalls:** storing big objects in the session inflates every request's Redis traffic; non-serialisable attributes fail on save; Redis eviction under `allkeys-lru` can silently log users out — give the session store its own instance or `noeviction`; for expiry events to work on managed Redis you may need to enable `notify-keyspace-events` yourself (Spring Session can do this unless you disable it with `ConfigureRedisAction.NO_OP`).

---

### Q51. What are keyspace notifications and when are they useful?

Enable with `notify-keyspace-events Ex` (E = keyevent, x = expired). Redis then publishes to `__keyevent@0__:expired` when a key expires. Uses: session-timeout hooks, "cart abandoned after 30 minutes" triggers, cache-invalidation cascades.

**Caveats to mention:** delivery is **pub/sub, so at-most-once and lost if no subscriber**; the notification fires when the key is *actually removed* (lazy/active expiry), so it can be late; and in Cluster the event fires only on the owning node. For reliability use a ZSet-based scheduler or Streams instead.

**Parameter decoding:** the flag string is a set of letters — `K` = keyspace events (channel `__keyspace@0__:<key>`, message = the event name), `E` = keyevent events (channel `__keyevent@0__:<event>`, message = the key), `x` = expired events, `g` = generic commands (`DEL`, `EXPIRE`), `A` = all. `Ex` therefore means "publish a message on `__keyevent@0__:expired` whenever a key expires". The `0` is the database number. It is **off by default** because it costs CPU.

**Example:** a cart key `cart:42` is set with a 30-minute TTL. When it expires, subscribers to `__keyevent@0__:expired` receive `cart:42` and can send an "you left items in your cart" email. Note you receive only the key *name* — the value is already gone, so keep any needed data in a separate shadow key or the DB.

**Pitfall:** with 10 pods all listening, **every pod** gets the event, so the action runs 10× unless you guard it with a lock (`SET NX`) or a single consumer. Also, since the event fires at actual deletion (lazy/sampled), it can come minutes after the nominal TTL on a quiet server.

```java
@Bean
KeyExpirationEventMessageListener expirationListener(RedisMessageListenerContainer c) {
    return new KeyExpirationEventMessageListener(c);   // publishes RedisKeyExpiredEvent
}

@EventListener
public void onExpired(RedisKeyExpiredEvent<?> e) {
    log.info("Key expired: {}", new String(e.getId()));
}
```

---

### Q52. Name real production use cases you've implemented with Redis.

Have 6–8 ready; specificity wins interviews:

1. **Read-through cache** for product/catalogue/config APIs → p99 from 180 ms to 6 ms, DB QPS down ~85%.
2. **Distributed locks** to make scheduled jobs run once across N pods (Redisson `RLock` / ShedLock with a Redis provider).
3. **Idempotency keys** for payment and webhook endpoints (`SET NX EX 24h`).
4. **API rate limiting** per API key at the gateway (token bucket Lua).
5. **Spring Session** for stateless auth across an autoscaled deployment; JWT denylist for logout/revocation (`SET jwt:blacklist:{jti} 1 EX <remaining-exp>`).
6. **Leaderboards / trending** with ZSets and time-bucketed keys.
7. **Redis Streams** as a lightweight event bus between services where Kafka was overkill.
8. **OTP / temporary token store** with natural TTL, plus attempt counters (`INCR` + `EXPIRE`).
9. **Dedupe / seen-set** for message consumers (Set or Bloom filter).
10. **Feature flags & runtime config** with pub/sub-driven refresh.

**How to turn each bullet into a 60-second story (Situation → Redis feature → result):**
- *Distributed scheduled job:* 12 pods each run `@Scheduled(cron="0 0 * * * *")`; without coordination the job runs 12×. Each pod tries `SET lock:job:hourly <podId> NX PX 600000`; only the winner runs. ShedLock does exactly this with a Redis provider (`@SchedulerLock(name="hourly", lockAtMostFor="10m")`).
- *JWT denylist:* JWTs are stateless and can't be "un-issued", so on logout store the token id (`jti`) with a TTL equal to its remaining lifetime; the auth filter does one `EXISTS` per request. The key cleans itself up when the token would have expired anyway.
- *OTP:* `SET otp:{phone} 482913 EX 300` (valid 5 min) plus `INCR otp:tries:{phone}` with `EXPIRE` to cap guesses; the TTL provides expiry for free (see S18).
- *Dedupe:* `SADD processed:{day} messageId` returns `0` if already present — a one-call "have I handled this message?" check for at-least-once consumers.
- *Feature flags:* store flags in a Hash, publish a change on a channel, and each pod updates its in-memory copy; reads never touch Redis.
Always add **numbers** (latency, QPS, cost) and **what went wrong and how you fixed it** — that is what makes it sound real, not rehearsed.

---

## 14. Performance, Troubleshooting & Production Ops

### Q53. Redis latency spiked in production. Walk me through your investigation.

Structure the answer as a funnel — this is exactly what they want to hear:

**1. Confirm where the latency is**
- `redis-cli --latency` and `--latency-history` (client-observed RTT), `--latency-dist`.
- `redis-cli --intrinsic-latency 100` on the host → measures the OS/hardware floor. If this is already high, it's the box, not Redis.
- Compare client-side p99 vs server-side: if the server is fine, the problem is network, connection pool exhaustion, GC pauses, or serialization on the Java side.

**2. Find slow commands**
- `SLOWLOG GET 25`, `SLOWLOG RESET`, `slowlog-log-slower-than 10000` (µs).
- `INFO commandstats` / `LATENCY LATEST` / `LATENCY DOCTOR`.
- Usual culprits: `KEYS`, `HGETALL`/`SMEMBERS`/`LRANGE 0 -1` on big keys, `ZRANGE` with huge ranges, unbounded Lua, `FLUSHALL`, `DEL` on big keys, `SORT`.

**3. Check memory & eviction**
- `INFO memory`: `used_memory`, `maxmemory`, `mem_fragmentation_ratio`, `evicted_keys`, `keyspace_misses`.
- Rising `evicted_keys` → too small / bad policy → hit rate collapses → DB load climbs → apparent "Redis slowness".
- Swap is fatal: check `/proc/{pid}/smaps` for swapped pages; disable swap for Redis.

**4. Check forks & persistence**
- `latest_fork_usec`, `rdb_bgsave_in_progress`, `aof_rewrite_in_progress`. Disable THP, move snapshots to a replica.

**5. Check the client layer**
- Connection pool saturation (`max-active` too low, `max-wait` timeouts), too many connections (`INFO clients`, `maxclients`), `blocked_clients`, `client-output-buffer-limit` kills for pub/sub consumers.
- Big payloads: 1 MB values at 5k rps = 5 GB/s — you're saturating NIC, not CPU. Check `INFO stats` → `instantaneous_input_kbps` / `output_kbps`.

**6. Check topology**
- Hot key pinning one shard (`--hotkeys`, per-node CPU), cross-slot chatter, a resharding/migration in progress, replica sync storms.

**Then the fix menu:** pipeline/batch, shrink payloads, split big keys, add local L1 cache, add replicas + `ReadFrom.REPLICA_PREFERRED`, raise `maxmemory` or shard, replace blocking commands, tune timeouts and pool.

🧠 **Memory hook — "S-M-F-C-T":** **S**lowlog, **M**emory, **F**ork, **C**lient/pool, **T**opology.

---

### Q54. Which Redis metrics do you alert on?

| Metric | Why |
|---|---|
| Hit rate (`keyspace_hits / (hits+misses)`) | Below ~80% means the cache isn't earning its keep |
| `used_memory` / `maxmemory` | Approaching the limit → evictions or OOM |
| `evicted_keys`, `expired_keys` | Unexpected eviction = undersized instance |
| `connected_clients`, `blocked_clients`, `rejected_connections` | Pool/`maxclients` problems |
| `instantaneous_ops_per_sec` | Capacity trending |
| Command latency p99 (per command) | Regression detection |
| `mem_fragmentation_ratio` | >1.5 → enable `activedefrag` |
| `master_link_status`, `master_repl_offset` lag | Replication health |
| `latest_fork_usec` | Persistence-induced stalls |
| `rdb_last_bgsave_status`, `aof_last_write_status` | Silent persistence failures |

Expose via **Micrometer + Actuator** (`spring-boot-starter-actuator` auto-registers Lettuce metrics; Spring Cache metrics with `@Cacheable` need `cacheManager` binding + `management.metrics.enable.cache=true`) and scrape with the Prometheus `redis_exporter`.

**Where these numbers come from:** run `INFO` (or `INFO stats`, `INFO memory`, `INFO replication`) — Redis prints counters as `key:value` text; the exporter just scrapes and publishes them. Hit rate example: `keyspace_hits=900,000` and `keyspace_misses=100,000` → 90%. Counters are cumulative since start, so alert on the *rate over a window* (e.g. Prometheus `rate(redis_keyspace_hits_total[5m])`), not the raw value.

**How to set alerts (example thresholds, tune to your workload):** hit rate < 80% for 10 min → warn; `used_memory/maxmemory` > 85% → warn, > 95% → page; any increase in `rejected_connections` → page; `master_link_status:down` > 1 min → page; `evicted_keys` rising on a cache that "shouldn't evict" → investigate sizing. Treat them as **symptom → cause** pairs: low hit rate + high evictions = cache too small; low hit rate + low evictions = poor key design or TTLs too short; high `blocked_clients` = lots of `BLPOP`/`XREAD BLOCK` waits or a slow command holding the line.

---

### Q55. How do you migrate / warm / flush a cache safely?

- **Warming:** background job or startup task that loads the top-N hot keys (derived from analytics), throttled so it doesn't hammer the DB; or replica-based warm restart with RDB.
- **Never `FLUSHALL` in production.** If you must invalidate everything, **bump a version prefix** (`app:v2:*`) so old keys age out via TTL/eviction while new reads populate fresh keys — zero-downtime, zero stampede.
- **Migration between clusters:** `redis-cli --cluster import`, `MIGRATE`, RIOT/`redis-shake`, or dual-write with read-fallback then cut over.
- `FLUSHALL ASYNC` if you truly must (non-blocking free), and disable dangerous commands in prod via `rename-command FLUSHALL ""`, `rename-command KEYS ""`, `rename-command CONFIG ""`.

**Why each is done that way:**
- *Warming* — an empty cache means 100% misses (Q15). Example: at startup, load the 5,000 most-viewed product IDs (from yesterday's analytics) in batches of 100 with a short sleep between batches, using a **pipeline** (`executePipelined`) so the warm-up is fast but doesn't swamp the DB.
- *Version prefix* — `FLUSHALL` deletes everything at once, so all pods miss simultaneously (stampede) *and* the delete may block Redis. Switching the prefix from `app:v1:` to `app:v2:` (config change + rolling deploy) makes every key a miss **gradually as traffic arrives**, while old keys are never read again and quietly expire.
- *Migration with dual-write* — (1) write to old and new Redis; (2) reads go to the new one with fallback to the old on miss (which also lets the new one warm); (3) once hit rate on the new one is steady, drop the old. For a pure cache you can often skip data copy and just let the new instance fill.
- *`rename-command … ""`* — renaming to an empty string disables the command entirely (it's set in `redis.conf`; on Redis 6+ prefer **ACLs**: `-@dangerous`). Managed services (ElastiCache) don't let you rename, but offer parameter groups/ACLs.

---

### Q56. Redis is down. What happens to your application?

The answer they want: **nothing fatal, by design.**

1. **Bounded timeouts** (1–2 s command timeout) so threads don't pile up.
2. **`CacheErrorHandler`** so `@Cacheable` failures log-and-fall-through to the DB instead of throwing.
3. **Circuit breaker (Resilience4j)** around cache access — once open, skip Redis entirely for N seconds instead of paying the timeout on every request.
4. **Local L1 cache (Caffeine)** absorbs the hottest traffic.
5. **DB protection** — connection-pool limits, a bulkhead/semaphore, and read-only degraded mode so the DB doesn't collapse (this is cache avalanche).
6. **HA first** — Sentinel/Cluster so a single node loss doesn't equal an outage.
7. **Fail differently for different data**: cache miss → degrade gracefully; but a **rate-limiter or lock** backed by a dead Redis needs an explicit fail-open (allow traffic) vs fail-closed (reject) decision. State that decision consciously — for a lock, fail closed; for a rate limiter on a login endpoint, usually fail closed; for a rate limiter on a read API, fail open.

🧠 **Memory hook:** *"Redis must be a performance dependency, not an availability dependency — except for locks, where it's a correctness dependency and you fail closed."*

**What this looks like in code:** Resilience4j wraps the Redis call, and the fallback returns "miss" so the caller goes to the DB:
```java
@CircuitBreaker(name = "redis", fallbackMethod = "cacheDown")
public Product getCached(long id) { return (Product) redisTemplate.opsForValue().get("product:" + id); }
private Product cacheDown(long id, Throwable t) { return null; }   // null = treat as a cache miss
```
With `failureRateThreshold=50` and `waitDurationInOpenState=10s`, after a burst of timeouts the circuit **opens** and calls fail instantly (no 2-second wait each), then it periodically lets one trial call through (*half-open*) to see if Redis is back. **Why this matters:** without it, 200 threads each blocking 2 s on a dead Redis exhausts the Tomcat pool and the *whole* app goes down even though the DB is healthy — an outage caused by the cache. **Fail-open vs fail-closed** means: when the guard (rate limiter, lock) can't be checked, do you let the request through (open — keeps users happy but loses protection) or reject it (closed — safe but causes errors)?

---

## 15. Security

### Q57. How do you secure a Redis deployment?

1. **Network isolation first** — private subnet/VPC, security groups, bind to a private interface (`bind 10.0.1.5`), never expose 6379 publicly. `protected-mode yes` is a backstop, not a strategy.
2. **Authentication** — `requirepass` (Redis <6) or **ACL users** (Redis 6+): `ACL SETUSER app on >secret ~app:* +@read +@write -@dangerous`. Least privilege per service.
3. **TLS** — `tls-port 6380`, `tls-cert-file`, `tls-ca-cert-file`; in Spring Boot `spring.data.redis.ssl.enabled: true`.
4. **Rename/disable dangerous commands** — `FLUSHALL`, `FLUSHDB`, `KEYS`, `CONFIG`, `DEBUG`, `SHUTDOWN`, `MODULE`, `SCRIPT`.
5. **Secrets management** — password from Vault/Secrets Manager, not `application.yml`.
6. **Deserialization safety** — never use JDK serialization for values; if using Jackson default typing, always supply a `PolymorphicTypeValidator` (unrestricted typing is a known RCE vector).
7. **Encrypt sensitive values at the application layer** before storing; don't put PII/PAN in the cache at all if you can avoid it.
8. **Audit & limits** — `maxclients`, `maxmemory`, `timeout` for idle clients, monitoring for `MONITOR`/`CONFIG` usage.

🧠 **Memory hook — "N-A-T-R-S":** **N**etwork, **A**uth/ACL, **T**LS, **R**ename commands, **S**ecrets + **S**erialization.

**Why each layer exists (with the story behind it):** Redis was designed to run inside a trusted network, so by default it has no auth and will execute whatever it receives. Exposed instances are routinely hijacked: an attacker who can reach port 6379 can run `CONFIG SET dir /root/.ssh` + `CONFIG SET dbfilename authorized_keys` + `SET` + `SAVE` to write an SSH key to disk and get a shell. That's why *network isolation is first* and `CONFIG`/`DEBUG`/`MODULE` are dangerous.

**ACL example explained:** `ACL SETUSER app on >secret ~app:* +@read +@write -@dangerous` means: create user `app`, **on** (enabled), password `secret` (`>` prefix), may only touch keys matching `~app:*`, may run commands in the read and write categories, but not the `@dangerous` category (`FLUSHALL`, `KEYS`, `CONFIG`, etc.). A reporting service would get `+@read` only. Spring config: `spring.data.redis.username: app`, `password: ${REDIS_PASSWORD}`.

**TLS** encrypts traffic between app and Redis so passwords and cached data can't be sniffed on the network (important in shared VPCs/cloud); it costs a little CPU/latency. **Deserialization risk:** if an attacker can write a crafted value into Redis and your app deserialises it with JDK serialization or unrestricted Jackson default typing, it can instantiate arbitrary classes → remote code execution. Keeping JSON with an allow-listed type validator (Q44) closes this. Typical interview follow-up: *"What if the Redis password leaks?"* → rotate via ACL (create a second user, roll the app over, remove the old one) rather than a restart.

---

## 16. Scenario-Based Questions

These are the interview differentiators. Each answer follows: **clarify → design → trade-offs → failure modes.**

---

### S1. Flash sale: 10,000 units, 500,000 concurrent users. Don't oversell.

**Design:**
1. **Pre-load inventory into Redis** before the sale: `SET stock:sku:123 10000`.
2. **Atomic decrement in Lua** (check + decrement in one atomic step — never `GET` then `DECR` from Java):
```lua
local stock = tonumber(redis.call('GET', KEYS[1]))
if not stock or stock <= 0 then return -1 end
if stock < tonumber(ARGV[1]) then return 0 end
return redis.call('DECRBY', KEYS[1], ARGV[1])
```
3. **Per-user dedupe** in the same script with a Set (`SADD buyers:sku:123 userId` → returns 0 if already bought) to enforce one-per-customer.
4. On success, **enqueue the order** to a Stream/Kafka and return "reserved" immediately; a consumer writes the order to the DB asynchronously (write-behind). The user polls or gets a websocket push.
5. **Shed load early**: return "sold out" from a local flag/Bloom check once stock hits 0, so 490,000 requests never reach Redis; rate-limit per user/IP; put a CDN/queue (waiting room) in front.
6. **Hot key mitigation**: split stock into N buckets (`stock:sku:123:{0..9}` with 1000 each) and hash the user to a bucket; roll over to another bucket if empty. This removes the single-shard bottleneck in Cluster.

**Trade-offs / failure modes to volunteer:**
- Redis is now the source of truth for stock during the sale → **reconcile with the DB** afterwards, and use `noeviction` + persistence so stock can't be evicted.
- If Redis fails mid-sale, you must **fail closed** (stop selling) rather than fall back to the DB and oversell.
- Reserved-but-unpaid orders need a **TTL-based release** (ZSet of reservation deadlines) so inventory returns to the pool.

---

### S2. Two users book the last seat at the same instant. How do you prevent double-booking?

**Layered answer (they're testing whether you know a Redis lock alone isn't enough):**
1. **Redis lock** (`SET seat:lock:{id} uuid NX PX 10000`, Redisson `RLock`) to serialise the *attempt* — this is an efficiency optimisation that prevents 99.9% of contention.
2. **The real guarantee lives in the database**: a `UNIQUE` constraint on `(event_id, seat_id)` or a conditional update — `UPDATE seats SET user_id=? WHERE seat_id=? AND user_id IS NULL` and check the affected-row count. Or optimistic locking with a `@Version` column.
3. Because of GC pauses, clock drift and failover, the lock **can** be held by two clients simultaneously; the DB constraint is what makes correctness unconditional.
4. Make the booking API **idempotent** by request key so client retries don't create duplicates.

**One-liner to remember:** *"Lock in Redis for throughput, enforce in the database for correctness."*

---

### S3. Your cache hit rate is only 40%. How do you diagnose and improve it?

**Diagnose:**
- `INFO stats` → `keyspace_hits`/`keyspace_misses`; per-cache metrics via Micrometer.
- Check `evicted_keys` — if high, the working set doesn't fit → memory is the problem, not the strategy.
- Check TTLs — too short means you evict data still being read.
- Check **key cardinality**: are you caching per-request keys that never repeat (e.g. keys including a timestamp, a random sort order, or a full free-text query)? Unique keys can't be reused.
- Check the access distribution: if traffic is uniform over 100M items, caching is inherently ineffective — you need a different approach.

**Improve:**
1. Right-size memory / add shards so the working set fits.
2. Lengthen TTLs where staleness is tolerable; switch to event-based invalidation so you don't need short TTLs.
3. **Normalise keys** — sort and canonicalise query params, round pagination, drop irrelevant params.
4. Cache at a **coarser granularity** (cache the assembled response, not 20 fragments) or **finer** (cache per-entity so different queries share entries) depending on the pattern.
5. Switch `allkeys-lru` → `allkeys-lfu` if the workload is skewed with occasional scans polluting the cache.
6. **Warm** the top-N keys and use refresh-ahead for them.
7. Accept reality: exclude genuinely uncacheable endpoints so they don't pollute the metric.

---

### S4. Design a "who's online" / presence feature for 5 million users.

**Options, ranked:**
1. **Bitmap per minute/5-minute bucket**: `SETBIT online:20260907:1430 {userIdOffset} 1`. 5M bits = ~610 KB per bucket. `BITCOUNT` gives concurrent users instantly; `BITOP OR` merges buckets for "active in the last 15 min"; `GETBIT` answers "is user X online?" in O(1). Requires dense integer user IDs (keep a `userId → offset` mapping).
2. **ZSet of heartbeats**: `ZADD presence {epochSeconds} {userId}` on each heartbeat; `ZRANGEBYSCORE presence (now-120) +inf` = online users; `ZREMRANGEBYSCORE presence -inf (now-300)` prunes. More memory but gives you the *list* and last-seen time.
3. **Per-user key with TTL**: `SET online:{userId} 1 EX 120` refreshed by heartbeat — trivial for "is X online", but counting requires `SCAN` (bad) so pair it with an approximate counter or HLL.

**What I'd actually ship:** ZSet for the authoritative last-seen + a bitmap per time bucket for cheap aggregate counts, with a 30–60 s client heartbeat and a Lua script that updates both atomically. Keys per day get a TTL so history self-cleans.

---

### S5. Session data must survive a Redis restart and a node failure. How?

1. **HA:** primary + ≥2 replicas + 3 Sentinels (or Cluster). Client configured with the Sentinel/Cluster topology so failover is transparent.
2. **Persistence:** AOF `everysec` + RDB (hybrid), so a restart replays sessions.
3. **Accept the async-replication window**: a failover can lose the last few writes. If a lost session is unacceptable, keep sessions **idempotently re-derivable** — e.g. a signed JWT for identity with Redis only holding revocation/denylist state, so a lost Redis entry degrades to "still logged in" rather than "logged out".
4. **Graceful UX**: on a session miss, re-authenticate via a refresh token rather than dumping the user to a login page.
5. For truly zero-loss requirements, use **AWS MemoryDB** (durable multi-AZ log) or store sessions in the DB with Redis as a read cache.

---

### S6. You have 50 Spring Boot pods running the same `@Scheduled` job. It must run once per cluster.

**Answer:** Use a distributed lock keyed by job name + scheduled instant.

```java
@Scheduled(cron = "0 0 2 * * *")
public void nightlyReconciliation() {
    RLock lock = redisson.getLock("job:nightly-reconciliation");
    if (!lock.tryLock(0, 30, TimeUnit.MINUTES)) return;   // don't wait; another pod has it
    try { doWork(); }
    finally { if (lock.isHeldByCurrentThread()) lock.unlock(); }
}
```

Points to add:
- **ShedLock** with the Redis provider (`@SchedulerLock(name="nightlyReconciliation", lockAtMostFor="30m", lockAtLeastFor="1m")`) is the idiomatic library; `lockAtLeastFor` prevents a fast job from running twice due to clock skew.
- The lease must exceed the worst-case runtime, or use a watchdog; otherwise two pods run concurrently → **make the job idempotent anyway** (upserts, `ON CONFLICT`, marker rows).
- Better still for heavy jobs: elect a leader or use a real scheduler (Quartz with a JDBC store, Kubernetes CronJob, or a queue where one consumer picks up the task).

**Why this problem exists:** `@Scheduled` is purely local — every JVM that has the bean fires at 02:00, so 50 pods means 50 copies of the reconciliation, duplicate emails, or double-billing. The scheduler itself has no idea other pods exist, so we need a shared "who's running it?" decision: a lock held in a store all pods can see (Redis).

**How `tryLock(0, 30, MINUTES)` behaves:** `0` = wait time (don't queue up — if someone else has it, return `false` immediately and skip); `30 min` = **lease** (the lock auto-expires after 30 min even if the owner crashes, so the job isn't blocked forever). Since all pods wake at the same second, one wins and 49 skip. With a lease given explicitly, Redisson's watchdog does *not* auto-extend it — so choose a lease longer than the longest realistic run, or omit it to use the watchdog.

**Common pitfall:** a job that finishes in 2 s releases the lock; a pod whose clock is a few seconds late then starts it again and acquires it. That's what ShedLock's `lockAtLeastFor` solves — the lock is kept for a minimum period (e.g. 1 min) even after the job completes.

---

### S7. A `HGETALL` on a hash with 1 million fields is timing out and spiking latency for everyone.

**Immediate:** stop calling `HGETALL`. Use `HSCAN` with `COUNT 500` to page through, or `HMGET` for the specific fields actually needed.

**Root cause:** a big key + an O(N) command on a single-threaded server — one call blocks the whole instance and ships tens of MB over the wire.

**Structural fix:** shard the hash into buckets (`h:{id}:{fieldHash % 64}`), or move it to a proper store if it's really a table. If you must delete it, use `UNLINK`, not `DEL`. Add guardrails: a code-review rule and/or a client interceptor banning `KEYS`, `HGETALL`, `SMEMBERS`, `LRANGE 0 -1` on unbounded collections; alert on `MEMORY USAGE` > threshold and run `--bigkeys` nightly.

**Why it blocks everyone:** `HGETALL` is O(N): Redis must walk 1M fields, build a ~tens-of-MB reply, and write it to the socket — all on the single command thread. For those hundreds of milliseconds, no other client's `GET` is served, so unrelated services see timeouts. Then the Java side must also parse a huge response (GC pressure).

**Example of the paged alternative:**
```java
ScanOptions opts = ScanOptions.scanOptions().count(500).build();
try (Cursor<Map.Entry<Object,Object>> c = redisTemplate.opsForHash().scan("big:hash", opts)) {
    while (c.hasNext()) { process(c.next()); }   // each underlying HSCAN call is a few hundred fields
}
```
Each `HSCAN` call is cheap and lets other commands run in between. If you only need three fields, `HMGET big:hash f1 f2 f3` is O(3). **Bucketing example:** `h:42:{crc32(field) % 64}` splits 1M fields into 64 hashes of ~16k, which also spreads them across Cluster slots. **Probe:** *"Why not just run it on a replica?"* — it helps the primary, but the replica's single thread blocks the same way, and replication lag can grow.

---

### S8. After a deploy, users intermittently see stale prices. Diagnose.

**Hypotheses to check, in order:**
1. **Invalidation not firing** — the write path updates the DB but the `@CacheEvict` is on a self-invoked method (proxy bypass), or it evicts the wrong `cacheNames`/key (key generator mismatch between read and write paths). *Most common cause.*
2. **Multiple cache layers** — a Caffeine L1 wasn't invalidated across pods (no pub/sub broadcast), so only the pod that handled the write is correct.
3. **Serializer/prefix change** in the deploy created a second key namespace, so reads hit old entries.
4. **Replica reads** — `ReadFrom.REPLICA` serving a lagging replica.
5. **Order-of-operations bug** — code does "update cache then DB" or "delete cache then update DB" (see Q41).
6. **Long TTL with no invalidation** on a fast-changing field.

**Verification:** reproduce with `redis-cli MONITOR` (briefly) or `OBJECT ENCODING`/`TTL` on the specific key; add a `cacheKey` + `sourceTimestamp` to the cached payload and log it in the response header so you can prove staleness age in production.

**Fix:** DB-first-then-delete, broadcast invalidation to L1s, versioned key prefixes on deploy, short TTL safety net, and pin the read path for the writing user for a few seconds.

---

### S9. Your Redis memory keeps growing and you eventually get OOM errors on writes.

1. **Confirm the policy:** if `maxmemory-policy` is `noeviction` (default!) or a `volatile-*` policy while keys have no TTL, Redis returns `OOM command not allowed when used memory > 'maxmemory'`. Set `allkeys-lru`/`allkeys-lfu` for a cache.
2. **Find keys without TTL:** `SCAN` + `TTL`, look for `-1`. Very common bug: `opsForValue().set(k,v)` without a TTL somewhere in the code.
3. **Find big keys:** `--bigkeys`, `MEMORY USAGE`.
4. **Check for a leak pattern:** ever-growing Lists/ZSets/Streams never trimmed (`LTRIM`, `ZREMRANGEBYRANK`, `XTRIM MAXLEN`), or unbounded per-request keys (session-per-request, lock keys never released — check that locks always have `PX`).
5. **Fragmentation:** `mem_fragmentation_ratio` > 1.5 → `activedefrag yes`, or restart on a replica and failover.
6. **Client buffers:** `client-output-buffer-limit` — a slow pub/sub consumer can hold hundreds of MB; check `INFO clients` → `client_recent_max_output_buffer`.
7. **Capacity:** if the working set genuinely grew, scale up or shard. Enforce a policy: **every cache write must have a TTL** (wrap `RedisTemplate` or use `RedisCacheManager` with a default TTL so it can't be forgotten).

---

### S10. Design a Redis-backed shopping cart.

- **Structure:** Hash per cart — `cart:{userId}` → field `sku:{id}` = quantity. `HINCRBY` for +/−, `HDEL` to remove, `HGETALL` to render (small enough here), `EXPIRE cart:{userId} 30d` refreshed on every mutation.
- **Guest carts:** key by a cookie/device id, then **merge** on login with a Lua script that `HINCRBY`s guest fields into the user cart and deletes the guest key atomically.
- **Persistence:** carts are business-valuable, so treat Redis as authoritative-with-backup: enable AOF, use `noeviction` (or a dedicated instance/logical DB with `allkeys-lru` *off*), and asynchronously mirror changes to the DB via a Stream so a Redis loss doesn't lose revenue.
- **Price/stock:** never store prices in the cart; resolve them at checkout from the source of truth (prices change; carts live for weeks).
- **Abandoned-cart trigger:** ZSet `cart:activity` scored by last-updated epoch; a job scans for carts idle >2 h and emits a marketing event (more reliable than keyspace-expiry notifications).

---

### S11. Requests to a legacy downstream API must be throttled to 100/sec globally across all pods.

Global limit ⇒ must be centralised ⇒ Redis. Use a **token bucket in Lua** so refill and consume are atomic:

```lua
-- KEYS[1]=bucket, ARGV: 1=rate/sec 2=capacity 3=now(ms) 4=requested
local b = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(b[1]) or tonumber(ARGV[2])
local ts     = tonumber(b[2]) or tonumber(ARGV[3])
local delta  = math.max(0, tonumber(ARGV[3]) - ts) / 1000
tokens = math.min(tonumber(ARGV[2]), tokens + delta * tonumber(ARGV[1]))
local need = tonumber(ARGV[4])
local allowed = tokens >= need
if allowed then tokens = tokens - need end
redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', ARGV[3])
redis.call('PEXPIRE', KEYS[1], 10000)
return allowed and 1 or 0
```

Add: **Redisson `RRateLimiter`** if you don't want to own the script; on rejection either queue the request (Redis List / Stream as a buffer, drained by a fixed-rate worker) or return 429 with `Retry-After`; and decide the **Redis-down behaviour** — for protecting a fragile legacy system, fail *closed* and let a local Resilience4j limiter act as a fallback ceiling.

**Walking through the script with numbers:** call with `rate=100`, `capacity=100`, `requested=1`. First call: no bucket exists, so `tokens=100`, spend 1 → 99, store `{tokens:99, ts:now}`. If 250 ms later another call arrives: `delta=0.25 s`, refill `0.25×100=25`, so `tokens=min(100, 99+25)=100`; spend 1. If a burst of 150 calls hits in the same millisecond, the first ~100 succeed, the rest get `0` (bucket empty) until time passes and refill adds ~1 token per 10 ms. **Why Lua:** the read-refill-spend-write sequence must be atomic, or two pods read the same `tokens` and both spend it. **Why a Hash + `PEXPIRE`:** state is two fields in one key, and the idle bucket cleans itself up after 10 s. Pass `now` from the app only if clocks are in sync; safer is to use Redis `TIME` inside the script (allowed since Redis 5 with effect replication).

---

### S12. You must cache a paginated, filterable search endpoint. How?

- **Don't cache whole result pages keyed by raw query strings** — cardinality explodes and hit rate stays near zero.
- Better: **cache IDs, not objects.** Cache the ordered ID list for a *normalised* query (sorted params, capped page depth) with a short TTL, then `MGET`/pipeline the individual entity caches (`product:{id}`, long TTL). Entity caches are shared across all queries → high hit rate, and invalidating one product doesn't invalidate every page.
- Only cache the first 2–3 pages (`condition = "#page.pageNumber < 3"`); deep pagination is rare and usually bot traffic.
- Invalidate ID lists by **version bump** (`search:v{n}:...` where `n` increments on catalogue changes) rather than trying to find every affected query key.
- If search is the core product, put it in Elasticsearch/OpenSearch and use Redis only for the entity layer.

**Concrete example of the ID-list approach:** request `GET /search?color=red&size=M&page=1&sort=price`. Normalise to `search:v7:color=red|size=M|sort=price|page=1`, store the value `[101,205,87,...]` (20 IDs) with a 60 s TTL. Rendering the page: `GET` the ID list, then one pipelined `MGET product:101 product:205 ...` for the entities. Another query that returns product 101 reuses the same `product:101` entry. When product 101's price changes you `DEL product:101` — the ID lists remain valid (they're only ordering), though sort-by-price lists can briefly be out of order, which is what the short TTL and version bump handle.

**Why raw-query-string caching fails:** `?q=shoe&page=2&sort=asc` vs `?sort=asc&q=shoe&page=2` are the same search but different strings; add free-text, filters and pages and the number of distinct keys is huge while each is requested once. **Version bump explained:** the `v7` is read from a small key (`INCR search:version` on catalogue changes); old `v6` keys are never read again and expire by TTL — no need to find and delete them.

---

### S13. A rogue client ran `KEYS *` in production and everything froze. How do you prevent recurrence?

1. **Disable it:** `rename-command KEYS ""` (and `FLUSHALL`, `FLUSHDB`, `CONFIG`, `DEBUG`, `MONITOR`) in redis.conf; enforce with **ACLs**: `ACL SETUSER app ... -keys -flushall -flushdb -config`.
2. **Least privilege** — separate ACL users for app vs ops vs analytics; no shared admin password.
3. **Guardrails in code** — a shared Redis client wrapper/library that forbids unbounded commands; a lint/PR rule; integration test that fails on `keys(` usage.
4. **Observability** — `SLOWLOG` alerting, `INFO commandstats` diffing, and a runbook.
5. **Culture/process** — no ad-hoc prod `redis-cli` access; use a read-only replica for exploration.

**Why it freezes:** `KEYS *` is O(N) over every key (10M keys ≈ seconds) on the one command thread — no other command runs meanwhile (Q4/Q18). Clients time out, retry, pools exhaust and the incident cascades into the apps.

**What each step looks like:**
- *ACL syntax:* `-keys` removes that one command; or whole categories: `-@dangerous -@admin`. Check with `ACL LIST`/`ACL WHOAMI`; log denials with `ACL LOG`. Prefer ACLs on Redis 6+ because `rename-command` is static, needs a restart and isn't supported on managed Redis.
- *Detection after the fact:* `SLOWLOG GET` shows the `keys *` entry with its duration and client address (`CLIENT LIST` helps map to a pod/user) — that's how you find the culprit.
- *Safe replacement:* tell people to use `SCAN 0 MATCH pattern COUNT 500` (Q18), or better, design key sets (e.g. keep `SADD sessions:index id`) so you never need to search the keyspace.
- *Prevention test:* a unit test or ArchUnit rule that fails the build if `redisTemplate.keys(` appears.

---

### S14. Two services need to coordinate a multi-step workflow. Would you use Redis or Kafka?

Frame it as a decision, not a preference:

- **Redis Streams** if: volume is moderate, retention is short (hours/days), you already run Redis, you want consumer groups + acks without new infrastructure, and latency matters more than durability guarantees.
- **Kafka** if: you need long retention/replay, high throughput with strict ordering per partition, exactly-once-ish semantics with transactions, multiple independent downstream teams, or a schema registry and compaction.
- **RabbitMQ** if: complex routing, per-message TTL/DLQ, and RPC-style patterns.
- **Redis Pub/Sub** only for ephemeral notifications.

Add the honest caveat: *"Redis Streams data lives in RAM and replication is async, so I don't use it as the only record of a financial workflow. I'd write the state transition to the DB and use the Stream for propagation, with idempotent consumers."*

**Terms and the decision in plain language:** a *workflow* here means a chain like Order Created → Payment Taken → Inventory Reserved → Shipped, where each service reacts to the previous step. Services communicate by publishing events to a broker. The question is how much durability, replay and scale you need. **Kafka** writes every event to disk, replicated across brokers and kept for days/weeks, so a new service can replay history and consumers scale by partition; the cost is operating it. **Redis Streams** gives you consumer groups and acks in an already-deployed system, with sub-millisecond latency, but capacity is bounded by RAM and a failover can drop the last few entries.

**Example:** an internal "send welcome email after signup" flow with a few hundred events/min and no regulatory audit → Redis Streams is enough. "Every payment state change must be auditable for 7 years and replayable to rebuild a ledger" → Kafka (or a DB outbox + Kafka). For multi-step flows spanning services, also mention the **saga/outbox pattern**: write the state change and an outbox row in one DB transaction, then publish from the outbox so no event is lost between DB and broker.

---

### S15. Users report that after login they occasionally see the previous user's data.

This screams **cache-key bug** — the highest-severity class of caching defect.

**Root causes to check:**
1. A `@Cacheable` key that omits the tenant/user (e.g. `key = "#page"` on a per-user method) → cross-user leakage. **This is the classic.**
2. A ThreadLocal/`SecurityContext` used to build the key but read after the request thread was recycled or in an async executor without context propagation.
3. A shared `RedisTemplate` connection bound to the wrong logical DB after `SELECT` (with `enableTransactionSupport`/thread-bound connections).
4. A CDN/HTTP cache issue rather than Redis (check `Cache-Control: private`, `Vary`).

**Fix and prevent:** include user/tenant in **every** key for user-scoped data (`key = "#root.methodName + ':' + @sec.currentUserId() + ':' + #id"`), add a mandatory tenant prefix in a custom `KeyGenerator`, write a test that asserts two users get different keys, and never cache user-scoped data in a shared cache name without that prefix. Then run an incident review: this is a data-privacy issue, so also assess whether it needs disclosure.

---

### S16. How would you migrate a single Redis instance to Redis Cluster with zero downtime?

1. **Audit the code first** — find every multi-key command, `MULTI`, Lua with multiple keys, `SELECT` on non-zero DBs, and cross-key pipelining. Introduce **hash tags** for keys that must co-locate.
2. Stand up the cluster (≥3 primaries + replicas).
3. **Dual-write** from the app (feature-flagged) to old and new, or use `redis-shake`/RIOT to replicate the dataset, then keep it in sync.
4. **Shadow reads** — read from both, compare, log mismatches, without serving from the new cluster.
5. **Cut over reads** gradually (percentage rollout), then writes; keep the old instance warm for rollback.
6. For a pure cache the simple option is legitimate: **switch and let it warm up** behind a rate-limited DB fallback and a warming job — no migration needed. Say this out loud; it shows judgment about cost vs risk.

---

### S17. Your Lua script occasionally causes `BUSY Redis is busy running a script`.

- The script is running too long and blocking the event loop. `lua-time-limit` (default 5000 ms) doesn't kill it; it just starts returning `BUSY` to other clients, and you can only `SCRIPT KILL` (if it hasn't written) or `SHUTDOWN NOSAVE` (if it has).
- **Fixes:** bound every loop in the script; never iterate a whole keyspace or a million-element collection; chunk the work across multiple `EVALSHA` calls with a cursor; move heavy logic into the application; and measure with `SLOWLOG`.
- Redis 7 Functions have the same constraint — server-side code must be short.

**Why it happens:** Lua scripts run atomically on the single command thread (Q22), so while a script runs *nothing else* executes. `BUSY` is the reply other clients get once the script has run longer than `lua-time-limit` (a.k.a. `busy-reply-threshold`, default 5 s) — Redis then lets a few admin commands through but still can't interrupt the script itself.

**Example of a bad script and its fix:**
```lua
-- BAD: loops over an unbounded set
for _, m in ipairs(redis.call('SMEMBERS', KEYS[1])) do redis.call('DEL', 'user:'..m) end
-- BETTER: process a bounded chunk per call and return a cursor; the app calls again until done
local r = redis.call('SSCAN', KEYS[1], ARGV[1], 'COUNT', 200)
for _, m in ipairs(r[2]) do redis.call('UNLINK', 'user:'..m) end
return r[1]   -- next cursor, "0" when finished
```
**Recovery ladder:** `SCRIPT KILL` works only while the script hasn't yet written anything (to avoid half-applied state); if it already wrote, the only option is `SHUTDOWN NOSAVE` (losing writes since the last save) — which is why prevention matters. Note a bug like an infinite `while true do end` loop in a script is the classic cause; code-review every script.

---

### S18. Design an OTP service (generate, verify, throttle) with Redis.

```
SET otp:{phone} {hash(otp)} EX 300 NX        # NX blocks re-request until expiry (or use a separate cooldown key)
SET otp:cooldown:{phone} 1 EX 60 NX          # 1 request/min
INCR otp:attempts:{phone} + EXPIRE 300        # max 5 verify attempts, then invalidate
INCR otp:sendcount:{phone}:{yyyyMMdd} + EXPIRE 86400   # daily cap
```
Verify in **one Lua script**: check attempts, compare hash, delete on success, increment on failure — atomic so parallel brute-force attempts can't exceed the limit. Store a **hash of the OTP**, not the OTP itself. Also: constant-time comparison, per-IP limits, and a `DEL` on successful login so an OTP is strictly single-use.

**What problem each key solves:** *OTP* = One-Time Password (the 6-digit code SMS'd to a user). The `EX 300` TTL makes the code expire by itself after 5 minutes — no cleanup job. The *cooldown* key stops a bot or impatient user from triggering unlimited SMS (which costs money). The *attempts* counter stops brute force: 6 digits = 1,000,000 possibilities, so without a cap an attacker could guess it within the validity window; 5 attempts ≈ 0.0005% chance. The *daily sendcount* caps abuse per phone number.

**Verify-script sketch:**
```lua
local attempts = tonumber(redis.call('GET', KEYS[2]) or '0')
if attempts >= 5 then return -1 end                    -- locked
if redis.call('GET', KEYS[1]) == ARGV[1] then          -- ARGV[1] = hash of the submitted code
  redis.call('DEL', KEYS[1], KEYS[2]); return 1        -- success: single use
end
redis.call('INCR', KEYS[2]); redis.call('EXPIRE', KEYS[2], 300); return 0
```
**Why hash the OTP:** if someone can read Redis (backup, `MONITOR`, a leak), they still can't see live codes. Because the code space is tiny, use an HMAC with a server secret, not a plain unsalted hash.

---

### S19. Your team wants to store 500 GB of data in Redis. Push back or proceed?

**Push back with numbers, then offer alternatives.** 500 GB of RAM (×2 for replicas, ×~1.5 headroom for fork/COW/fragmentation) is very expensive and slow to recover. Questions I'd ask: what fraction is actually hot? What's the required latency? Is Redis being used as a database because the real DB is slow?

**Alternatives:** keep only the hot ~5–10% in Redis with LFU eviction; move the rest to the right store (Postgres/DynamoDB/Cassandra/S3); use **Redis on flash** (Redis Enterprise Flash) or AWS MemoryDB tiering; consider compression and better serialization (often a 3–5× win); reconsider the data model (are you storing 500 GB because of JSON bloat and no TTLs?).

**The arithmetic behind "push back":** 500 GB of data + 1 replica = 1 TB RAM; add headroom for fork/COW and fragmentation (Q14) and you need ~1.5 TB across machines. After a crash, loading 500 GB from RDB/AOF takes many minutes (tens of minutes at ~1 GB/s disk), during which that shard is down, and a full replica resync of 500 GB can saturate the network and the primary. In Cluster you'd shard (e.g. 20 primaries × 25 GB) to keep each node small and recoverable — Redis works best with nodes of roughly ≤25–50 GB.

**Questions that reveal the right answer:** *What's the access pattern?* (if 95% of reads touch 5% of data, a 25 GB hot cache + a DB gives the same speed). *Do you need durability?* (Redis is not a primary database for this volume.) *What does it cost?* (compare RAM price/GB vs SSD-based stores). A good answer is a recommendation with trade-offs, not a flat "no".

---

### S20. Explain a time cache invalidation went wrong and what you learned. (Behavioural + technical)

Have a real story with this shape: **context → what broke → how you detected it → immediate mitigation → root cause → permanent fix → what changed in process.** Example skeleton:

> We cached pricing with a 1-hour TTL and evicted on update, but the eviction was in a method called internally within the same service class, so the Spring proxy never applied `@CacheEvict`. Merchants updated prices and saw no change for up to an hour; support tickets spiked. We detected it by adding the cache key and cached-at timestamp to a response header. Mitigation was bumping the cache version prefix to flush and temporarily dropping TTL to 60 s. The real fix was moving eviction to the transactional service boundary, adding an integration test that asserts eviction happens, wiring cache metrics into Grafana with a staleness alert, and adopting a rule that every cache has both event-based invalidation *and* a TTL safety net.

**Lesson to state:** *"TTL is not a substitute for invalidation, and invalidation is not a substitute for TTL. You need both, plus observability that makes staleness visible."*

---

### S21. How do you test Redis-dependent code?

- **Unit tests:** mock `RedisTemplate`/repository, or use **Embedded Redis** (`it.ozimov:embedded-redis`) for fast local tests — but note it's an older Redis version.
- **Integration tests:** **Testcontainers** with the real image — this is the right answer.
```java
@Testcontainers
@SpringBootTest
class CacheIT {
    @Container static GenericContainer<?> redis =
        new GenericContainer<>("redis:7.2-alpine").withExposedPorts(6379);

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.data.redis.host", redis::getHost);
        r.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
    }
}
```
- **What to actually assert:** that the second call doesn't hit the repository (verify with a Mockito spy), that eviction happens on update, that keys include the tenant, that TTLs are set, and that a Redis outage degrades instead of failing (stop the container mid-test and assert the endpoint still returns from the DB).
- **Concurrency tests** for locks/limiters: fire N threads with a `CountDownLatch` and assert the invariant (exactly one winner, at most X allowed).
- Clean state between tests (`FLUSHDB` on the test container, unique key prefixes per test) to avoid order-dependent flakiness.

---

### S22. Cluster: one shard is at 95% CPU while the others idle. What now?

**Diagnose:** it's almost always a **hot key** or a **hot slot** (all keys sharing a hash tag). Confirm with per-node `INFO commandstats`, `--hotkeys`, and slot-level key counts (`CLUSTER COUNTKEYSINSLOT`).

**Fix, cheapest first:**
1. **Local L1 cache** (Caffeine, 1–5 s TTL) in the app — instantly removes most reads for a hot key.
2. **Client-side caching** with RESP3 invalidation (Lettuce) for correctness with freshness.
3. **Read replicas** for that shard + `ReadFrom.REPLICA_PREFERRED`.
4. **Key replication/fan-out**: store N copies (`hot:{k}:0..9`) and randomise reads; write to all N on update.
5. **Remove over-broad hash tags** so keys spread across slots again.
6. **Reshard** to give that shard fewer other slots, if the hot key is genuinely irreducible.

**Why adding nodes doesn't help:** a key lives in exactly one slot, owned by exactly one primary. If `sku:9987` takes 80% of the traffic, that primary's single command thread is saturated no matter how many other shards exist — horizontal scaling only spreads *different* keys. A **hot slot** is the same problem at slot level, e.g. a hash tag like `{tenant1}` placed on every key of a large tenant forces them all onto one shard. **Quick triage commands:** `redis-cli -c --cluster info` (keys per node), `INFO commandstats` on each node, `--hotkeys` (requires an LFU policy), and compare `used_cpu_sys/user` between nodes. **Example outcome:** after adding a 2 s Caffeine cache for `sku:9987` in each of 40 pods, Redis sees at most 40 reads per 2 s for that key instead of 50k/s.

---

### S23. Design a URL shortener's Redis layer for 100k reads/sec.

- `GET short:{code}` → long URL. String values, no TTL for permanent links (or long TTL + DB fallback).
- **ID generation:** `INCR url:seq` then Base62-encode; grab ranges (`INCRBY 1000`) per pod to avoid a round trip per write.
- **Reads:** this is a read-mostly, small-value workload — ideal for Redis. Add replicas and read from them; add a CDN/edge cache for the 301 redirect; Caffeine L1 in each pod for the top codes.
- **Analytics:** `HINCRBY stats:{code} clicks 1` and `PFADD stats:{code}:uniq {visitorId}` (HLL), flushed to a warehouse periodically. Do this **asynchronously** so it never adds latency to the redirect.
- **Penetration defence:** a Bloom filter or cached-null for non-existent codes (scanners probe random codes constantly).
- **Cluster:** keys are naturally well-distributed by code; no hash tags needed.

**Why these choices:** at 100k reads/s the only cost that matters is a round trip, so each redirect must be *one* `GET`. **Base62** means using characters `0-9a-zA-Z` (62 symbols) to write a number compactly: 62⁷ ≈ 3.5 trillion, so a 7-character code covers any realistic volume. **Example:** counter value 125 → `2B` in Base62 → `sho.rt/2B`. Sequential IDs are guessable (people can enumerate URLs); if that matters, scramble the counter (e.g. a reversible permutation/Hashids) or use random codes with a `SET NX` collision check. A `301` is cached by browsers (fewer hits, but you lose click counts); `302` always hits you, which gives analytics. **Back-of-envelope:** 100M links × ~100 bytes ≈ 10 GB — fits comfortably in one primary with replicas.

---

### S24. How would you implement "trending articles in the last hour"?

Time-bucketed ZSets with a sliding union:
```
ZINCRBY trending:202609071430 1 article:88      # 1-minute bucket
EXPIRE trending:202609071430 7200
# every minute, union the last 60 buckets:
ZUNIONSTORE trending:1h 60 trending:{...x60}
EXPIRE trending:1h 120
ZREVRANGE trending:1h 0 19 WITHSCORES
```
Notes: pre-compute `trending:1h` on a schedule (not per request) because `ZUNIONSTORE` over 60 keys is expensive; in Cluster all buckets need the same hash tag (`trending:{t}:...`); consider decay-weighted scores (`score * 0.9` per bucket) for smoother rankings; for very high write volume, batch increments in the app and flush with a pipeline.

**Why buckets instead of one ZSet:** a single `trending` ZSet only grows; you can't tell which counts happened in the last hour. By writing each minute's views into its own small ZSet (`trending:<yyyyMMddHHmm>`) with a TTL, old minutes simply expire, and "last hour" = sum of the 60 newest buckets. `ZUNIONSTORE dest 60 k1 ... k60` adds the scores per member (the default aggregate is SUM) and writes a new ZSet; `ZREVRANGE 0 19` then reads the top 20 articles. **Example:** article 88 gets 5 views in minute 14:30 and 3 in 14:31 → its union score is 8, ranked against all other articles.

**Spring sketch:** `redisTemplate.opsForZSet().incrementScore("trending:" + minuteKey, "article:" + id, 1)` on each view, and a `@Scheduled(fixedRate = 60_000)` job that builds the 60 key names and calls `unionAndStore(first, others, "trending:1h")`. **Pitfall:** because the precomputed list is refreshed once a minute, "trending" is up to a minute stale — that's the deliberate trade-off for cheap reads.

---

### S25. A Spring Boot pod logs `RedisCommandTimeoutException` under load, but Redis CPU is 20%.

So the server is fine — look at the client and the network:
1. **Connection pool exhaustion / blocking commands** hogging the shared Lettuce connection (`BLPOP`, `MULTI`, long Lua). Give blocking operations a dedicated connection.
2. **JVM GC pauses** — a 3-second stop-the-world looks exactly like a Redis timeout. Check GC logs.
3. **Large payloads** — a 5 MB value at high concurrency saturates the NIC or the Netty event loop; check `INFO stats` throughput and payload sizes; compress or shrink.
4. **Too-aggressive timeout** (e.g. 50 ms) vs real p99 across an AZ boundary; measure and set realistically (usually 1–2 s with retries only for idempotent reads).
5. **Netty event-loop starvation** — doing blocking work inside Lettuce callbacks (`.thenApply` chains that call the DB). Never block in the Lettuce I/O thread.
6. **Network/topology** — cross-AZ or cross-region hops, DNS/failover in progress, `MOVED` storms after a topology change (enable adaptive topology refresh), packet loss (`redis-cli --latency` from the pod's node).
7. **Thread-pool queuing in the app** — the timeout is where the symptom surfaces, not where the queue is.

**Structure the answer:** *"Server-side fine + client-side timeouts = client, GC, network, or payload. I'd bisect by running `redis-cli --latency` from the same pod: if that's clean, the problem is inside my JVM."*

---

### S26. You need to cache data that differs per tenant in a multi-tenant SaaS.

- **Key strategy:** `tenant:{tenantId}:entity:{id}` for everything, enforced by a custom `KeyGenerator` or a `CacheKeyPrefix` that reads the tenant from a request-scoped context — never optional.
- **Isolation options:** shared instance with tenant prefixes (cheapest); separate logical DBs (❌ not available in Cluster, and no isolation guarantees); separate instances/clusters per large tenant (noisy-neighbour and compliance isolation).
- **Noisy neighbours:** per-tenant memory budgets are not enforceable natively — use `allkeys-lfu`, per-tenant rate limits, and monitor per-prefix memory (`--memkeys`, or track sizes in a counter).
- **Eviction fairness:** a single huge tenant can evict everyone else's data; that's the argument for dedicated instances for enterprise tiers.
- **Compliance:** data residency and encryption per tenant; ability to purge one tenant completely (`SCAN` by prefix + `UNLINK`, or a per-tenant key version).

**What "multi-tenant" means here:** one application serves many customers (tenants), and their data must never mix — same class of bug as S15, but by design. **Example:** two tenants both have a product with id 7; if the key is just `product:7` they overwrite each other (or leak). With `tenant:acme:product:7` and `tenant:globex:product:7` they're isolated. A Spring `KeyGenerator` can read the tenant from a request-scoped holder:
```java
@Bean KeyGenerator tenantKeyGen() {
    return (target, method, params) -> TenantContext.current() + ":" + method.getName() + ":" + Arrays.deepToString(params);
}
```
**Trade-off summary:** a shared instance is cheapest but has noisy-neighbour and blast-radius risks; dedicated instances cost more but give isolation. **Purging one tenant:** the version trick — keep `tenant:acme:ver` and include it in keys; `INCR` it to logically wipe all of that tenant's cache instantly, and let TTL clean the rest. **Pitfall:** async threads (`@Async`, reactive) lose the tenant `ThreadLocal`, so propagate context explicitly or the key may silently use a wrong/empty tenant.

---

### S27. Interviewer: "Can we just use Redis instead of our relational database?"

Give a structured no:
- **No ad-hoc queries or joins** — you must design a key for every access pattern up front, and every new query means a new index you maintain by hand.
- **No transactions across entities with rollback**, no constraints, no referential integrity.
- **Durability is a window, not a guarantee**; replication is async.
- **RAM cost** — 100× the price per GB vs disk.
- **No mature reporting/BI/analytics ecosystem.**

But say where Redis *is* the primary store legitimately: ephemeral state (sessions, locks, rate limits, presence), high-velocity counters, queues/streams with short retention, and precomputed views. And mention Redis Stack (JSON + Search + TimeSeries) narrows the gap for document-style workloads — with the caveat that it's still memory-resident.

**Example that makes the "no ad-hoc queries" point concrete:** in SQL, "orders over ₹5,000 placed last week by customers in Pune" is one `SELECT … JOIN … WHERE`. In plain Redis you must have planned for it: maintain a ZSet `orders:by_date`, a Set `customers:city:pune`, and a Hash per order, then intersect and filter in application code — and keep all those structures consistent yourself on every write. **Rollback example:** a `MULTI` that debits account A, credits B, and fails on step 3 leaves A debited (Q6). **Better framing for the interviewer:** *"Redis is the fast layer in front of / beside the database, chosen per access pattern; the relational DB stays the system of record."*

---

### S28. How do you handle a cache stampede when a deploy invalidates everything?

1. **Don't invalidate everything** — versioned prefixes let old and new coexist; old keys expire naturally.
2. If you must, **warm before cutover** — populate the new namespace from a batch job, then flip the read prefix.
3. **Single-flight per key** on the read path (`SET NX` mutex, or Caffeine's `LoadingCache` which coalesces concurrent loads per key).
4. **Rate-limit / bulkhead the DB** so at worst you slow down instead of collapsing.
5. **Serve stale while revalidating** — store `{value, softExpiry, hardExpiry}` and refresh asynchronously past soft expiry.
6. **Stagger** by using jittered TTLs so you never get synchronised expiry again.

**Why a deploy causes this:** if the new release changes the serializer, key format or class shape, old cached entries become unreadable (deserialisation errors) so every read is a miss — or the team flushes the cache to be safe. Every pod then hits the DB for the same hot keys at once (stampede, Q38/Q39), exactly when the app is also cold (JIT warm-up, empty L1 caches).

**Soft/hard expiry example:** store `{value, softExpiry = now+5min, hardExpiry = now+30min}` with Redis TTL = 30 min. A read before 5 min returns it. A read between 5–30 min **still returns the stale value immediately** but one request (guarded by `SET NX`) refreshes it in the background; after 30 min it truly misses. Users never wait, and the DB sees one query per key per refresh instead of thousands. **Single-flight in Caffeine:** `Cache.get(key, k -> loadFromDb(k))` guarantees one loader per key *per JVM* — combine with a Redis lock for a cluster-wide guarantee.

---

### S29. How would you build a "recently viewed products" list (last 20, no duplicates)?

**ZSet** with the timestamp as score — it deduplicates and orders in one structure:
```
ZADD recent:{userId} {now} product:{id}      # re-viewing just updates the score
ZREMRANGEBYRANK recent:{userId} 0 -21         # keep only the newest 20
EXPIRE recent:{userId} 2592000                # 30 days
ZREVRANGE recent:{userId} 0 19                # read newest first
```
Why not a List: `LPUSH` + `LTRIM` is simpler but allows duplicates and needs an O(N) `LREM` to dedupe. Why not a Set: no ordering. Mention that both writes should be in one pipeline/Lua so the trim can't be skipped, and that the resulting IDs are then hydrated from the shared `product:{id}` entity cache with `MGET`.

**How it works:** a ZSet stores unique members, each with a numeric score, kept sorted by score. Using the *view timestamp* as the score gives "newest first" ordering; adding the same product again only updates its score (so it moves to the top instead of duplicating). `ZREMRANGEBYRANK recent:{u} 0 -21` deletes everything except the 20 highest-ranked (ranks are ascending by score; `-21` means "21st from the end", so ranks 0 to -21 are the old ones). **Example:** user views products A, B, C, then A again → list reads A, C, B. In Spring:
```java
ZSetOperations<String,Object> z = redisTemplate.opsForZSet();
z.add(key, productId, System.currentTimeMillis());
z.removeRange(key, 0, -21);
redisTemplate.expire(key, Duration.ofDays(30));
Set<Object> ids = z.reverseRange(key, 0, 19);
```

---

### S30. Give me your Redis code-review checklist.

Say you keep one — this lands very well:

1. Does every cached key have a **TTL** (or a documented reason not to)?
2. Is the key **namespaced and versioned** (`app:v1:entity:id`) and does it include **tenant/user** where the data is scoped?
3. Any **unbounded commands** (`KEYS`, `HGETALL`, `SMEMBERS`, `LRANGE 0 -1`, `ZRANGE 0 -1`)? Replace with `SCAN`/paged reads.
4. Any **read-modify-write** across two round trips that should be `INCR`/Lua?
5. Locks: unique token, TTL, atomic release, renewal, and is the operation **idempotent** anyway?
6. Are **N+1 round trips** batched into a pipeline or `MGET`?
7. Are values **DTOs, not JPA entities**, with a schema-tolerant serializer?
8. **Failure behaviour:** is there a timeout, and does a Redis outage degrade (cache) or fail closed (lock/limiter) — deliberately?
9. Is **invalidation on the transactional boundary** (DB-first, then delete) and not blocked by proxy self-invocation?
10. **Cluster safety:** multi-key ops in one slot via hash tags; no `SELECT`.
11. **Observability:** hit/miss metrics, key-size guardrails, slowlog alerting.
12. **Security:** no PII/secrets cached unnecessarily; no unrestricted polymorphic deserialization.

---

## 17. Rapid-Fire Cheat Sheet

| Question | Answer in one breath |
|---|---|
| Default port / DBs | 6379; 16 logical DBs (0–15); Cluster has only DB 0 |
| Max value size | 512 MB per string (but keep values < 100 KB in practice) |
| Max keys | ~2^32 per instance, memory-bound in reality |
| Is Redis single-threaded? | Command execution yes; I/O threads and lazy-free are separate (6.0+) |
| `SETNX` vs `SET NX PX` | Use the latter — atomic set-with-expiry; `SETNX`+`EXPIRE` can deadlock |
| `TTL` return values | `-1` no expiry, `-2` key doesn't exist |
| Does writing a key clear its TTL? | `SET` clears it; `INCR`/`HSET`/`APPEND`/`LPUSH` preserve it. **Classic gotcha.** |
| `EXPIRE` on a non-existent key | Returns 0, does nothing |
| `PERSIST` | Removes the TTL |
| `SCAN` guarantees | Full iteration of keys present throughout; duplicates possible; no snapshot |
| `TYPE` / `OBJECT ENCODING` | Inspect logical type and internal encoding (listpack, intset, skiplist, quicklist, embstr, int, raw) |
| Blocking commands | `BLPOP`, `BRPOP`, `BRPOPLPUSH`, `BLMOVE`, `BZPOPMIN`, `XREAD BLOCK` |
| Atomic units | Single command, `MULTI/EXEC`, Lua script, Function |
| Does MULTI roll back? | No |
| Cluster slots | 16384, `CRC16(key) mod 16384`; hash tags `{...}` co-locate |
| Sentinel minimum | 3 sentinels; quorum = majority |
| Default eviction | `noeviction` |
| Recommended cache eviction | `allkeys-lru` or `allkeys-lfu` |
| AOF fsync default | `everysec` (≤1s loss) |
| Spring Boot default client | Lettuce |
| Spring cache annotations | `@EnableCaching`, `@Cacheable`, `@CachePut`, `@CacheEvict`, `@Caching`, `@CacheConfig` |
| Key serializer to use | `StringRedisSerializer` for keys, JSON for values; never JDK serialization |
| How to invalidate everything safely | Bump a key/prefix version — not `FLUSHALL` |
| Hit rate formula | `keyspace_hits / (keyspace_hits + keyspace_misses)` |
| Redis version highlights | 4: modules, `UNLINK`, LFU · 5: Streams · 6: ACL, TLS, RESP3, threaded I/O, client-side caching · 7: Functions, sharded pub/sub, multi-part AOF, `OBJECT FREQ` improvements · 7.4: hash field TTLs (`HEXPIRE`) |

**The 10 sentences that carry most Redis interviews:**
1. Single-threaded execution is why atomicity is free and why one slow command hurts everyone.
2. Always `SET key val NX PX ttl` for locks, release with a Lua compare-and-delete.
3. Cache-aside + TTL + delete-after-DB-write is the default; never "update cache then DB".
4. `SCAN`/`UNLINK`, never `KEYS`/`DEL` on big keys.
5. Hash tags are how you survive Redis Cluster's multi-key restrictions.
6. Replication is asynchronous, so Redis is not a system of record for money.
7. Pipelining is the cheapest performance win in almost every Spring Boot app.
8. Penetration → null-cache/Bloom; breakdown → mutex/logical expiry; avalanche → jitter + HA + circuit breaker; hot key → local cache.
9. Cache DTOs, versioned key prefixes, and a `CacheErrorHandler` so Redis can't take your app down.
10. Locks give efficiency; databases give correctness.

---

## 18. Questions to Ask the Interviewer

Asking these signals seniority:

- What's your Redis topology — Sentinel, Cluster, or managed (ElastiCache/MemoryDB)? How many shards, and what drove that choice?
- What's the current cache hit rate and p99, and what's the biggest source of cache-related incidents?
- Is Redis a pure cache, or does anything treat it as a system of record?
- How do you handle cache invalidation today — TTL, application events, or CDC/binlog?
- Do you use Redis for locking or rate limiting, and how do you handle the failure modes?
- What does your Redis observability look like — slowlog alerts, big-key scans, per-cache metrics?
- What's the migration/upgrade story, and are you on Redis 7 / Valkey?

---

*Prep tip: don't memorise this end to end. Read it once, then rehearse out loud the 🧠 hooks, the four cache failures, the lock checklist (U-T-A-R-F), the latency funnel (S-M-F-C-T), and three scenarios from your own experience. Interviewers reward structure and trade-off awareness far more than command trivia.*
