# Kafka Interview Guide — Java / Spring Boot Developer (9 Years Experience)

> **How to use this file:** Each answer has a **one-line "say this first"** hook, then depth.
> In interviews, lead with the hook, then expand only if they probe. The 🧠 boxes are memory tricks.

---

## Table of Contents

1. [Core Architecture](#1-core-architecture)
2. [Producer Internals](#2-producer-internals)
3. [Consumer & Consumer Groups](#3-consumer--consumer-groups)
4. [Delivery Semantics & Exactly-Once](#4-delivery-semantics--exactly-once)
5. [Replication, Durability & ISR](#5-replication-durability--isr)
6. [Storage, Retention & Log Compaction](#6-storage-retention--log-compaction)
7. [Spring Boot + Kafka (the section they will grill you on)](#7-spring-boot--kafka)
8. [Error Handling, Retries & DLQ](#8-error-handling-retries--dlq)
9. [Schema Registry & Serialization](#9-schema-registry--serialization)
10. [Kafka Streams & Kafka Connect](#10-kafka-streams--kafka-connect)
11. [Performance Tuning](#11-performance-tuning)
12. [Security & Operations](#12-security--operations)
13. [Design Patterns for Microservices](#13-design-patterns-for-microservices)
14. [Scenario-Based Questions (30)](#14-scenario-based-questions)
15. [Rapid-Fire Round](#15-rapid-fire-round)
16. [Config Cheat Sheet](#16-config-cheat-sheet)
17. [Common Exceptions → Root Cause](#17-common-exceptions--root-cause)
18. [Last-Minute Revision Sheet](#18-last-minute-revision-sheet)

---

## 1. Core Architecture

### Q1. What is Kafka and why would you pick it over RabbitMQ / ActiveMQ?

**Say this first:** "Kafka is a distributed, append-only, replicated commit log. A traditional MQ deletes a message once consumed; Kafka *retains* it and each consumer group tracks its own position."

**Detail:**

| Dimension | Kafka | RabbitMQ / ActiveMQ |
|---|---|---|
| Model | Log (pull-based) | Queue (push-based) |
| Message after consume | Retained until retention expires | Deleted on ack |
| Replay | Native — just reset offset | Not possible |
| Throughput | Millions/sec (sequential disk + zero-copy) | Tens of thousands/sec |
| Ordering | Guaranteed **per partition** | Per queue |
| Routing | Dumb broker, smart consumer | Rich routing (exchanges, headers, topics) |
| Best for | Event streaming, log aggregation, event sourcing, CDC, fan-out | Complex routing, RPC-style request/reply, per-message TTL/priority |

**Pick Kafka when:** high volume, multiple independent consumers of the same data, replay/audit needed, stream processing downstream.
**Pick RabbitMQ when:** complex routing rules, per-message priority, low volume with request/reply semantics.

🧠 **Memory hook:** *Kafka = DVR (rewind anytime). RabbitMQ = postal service (delivered, gone).*

---

### Q2. Explain the full anatomy: broker, topic, partition, offset, segment.

**Say this first:** "Topic is a logical name, partition is the unit of parallelism and ordering, offset is the position inside a partition, segment is the physical file on disk."

```
Topic: "orders"  (replication.factor=3, partitions=3)

  Partition 0: [0][1][2][3][4]...   leader on Broker-1, followers B2,B3
  Partition 1: [0][1][2]...         leader on Broker-2, followers B1,B3
  Partition 2: [0][1][2][3]...      leader on Broker-3, followers B1,B2

  Each partition on disk =
      00000000000000000000.log      <- segment (default 1 GB / 7 days)
      00000000000000000000.index    <- offset -> byte position
      00000000000000000000.timeindex<- timestamp -> offset
      leader-epoch-checkpoint
```

Key facts you should be able to rattle off:

- **Offset** is unique only *within a partition*, monotonically increasing, never reused.
- **Ordering** is guaranteed only within a partition — never across a topic.
- Only the **leader** replica serves reads/writes (unless you enable follower fetching for rack locality, KIP-392).
- Writes are **append-only sequential I/O**, which is why spinning disks were fine.
- Reads use **zero-copy** (`sendfile`) — page cache → NIC without user-space copy. This breaks if you enable SSL or broker-side decompression.
- Partition count can be **increased but never decreased**.

🧠 **Memory hook:** *T-P-O-S → Topic, Partition, Offset, Segment — "Kafka's TPOS terminal."*

---

### Q3. ZooKeeper vs KRaft — what changed?

**Say this first:** "ZooKeeper stored cluster metadata externally; KRaft moves metadata into Kafka itself as an internal Raft-replicated log, removing the external dependency."

| | ZooKeeper mode | KRaft mode |
|---|---|---|
| Metadata store | External ZK ensemble | Internal `__cluster_metadata` topic |
| Controller | One broker elected as controller | Quorum of controllers (Raft) |
| Metadata propagation | Controller pushes RPCs to brokers | Brokers *pull* the metadata log |
| Partition scale | ~200k partitions/cluster | Millions |
| Recovery from controller failover | Seconds to minutes (reload full state from ZK) | Near-instant (log replay from cache) |
| Status | Deprecated in 3.5, **removed in Kafka 4.0** | Production-ready since 3.3, default in 4.0 |

**What ZooKeeper used to hold:** broker registration, topic configs, partition assignment, controller election, ACLs, (very old versions: consumer offsets).

**Interview trap:** "Where are consumer offsets stored?" → In the internal compacted topic `__consumer_offsets` (50 partitions by default), **not** in ZooKeeper. That moved in 0.9.

**Plain-language explanation:**
- *Metadata* = the cluster's "who/where" information: which brokers exist, which topics/partitions exist, who leads each partition, topic configs, ACLs. Every broker needs a consistent view of it.
- *ZooKeeper mode:* a separate 3–5 node ZooKeeper cluster stored this. You had to deploy, secure, monitor and upgrade two systems. One broker (the controller, Q5) read everything from ZK and pushed changes to the others — slow on failover for big clusters.
- *KRaft mode:* the metadata is just another Kafka log (`__cluster_metadata`), replicated among a small group of controller nodes using **Raft** (a consensus algorithm: nodes elect a leader and a change counts as committed once a majority has it). Other brokers read that log like any consumer, so a new controller already has the state in memory and takes over almost instantly.
- *Why it matters in practice:* one system to run, faster controller failover, and a much higher partition ceiling. Deployment choices: **combined** nodes (broker+controller, dev/small) vs **dedicated** controllers (production, typically 3).

```properties
# KRaft node (server.properties)
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@host1:9093,2@host2:9093,3@host3:9093
```

🧠 **Memory hook:** *KRaft = "Kafka Raft" = Kafka finally stops outsourcing its memory.*

---

### Q4. How does a producer know which broker to send to?

Producer bootstraps with `bootstrap.servers` (any subset), fetches cluster **metadata** (topic → partition → leader mapping), caches it, then sends directly to each partition **leader**. Metadata refreshes every `metadata.max.age.ms` (5 min) or immediately on a `NOT_LEADER_OR_FOLLOWER` error.

`bootstrap.servers` is only a discovery seed — give it 2–3 brokers for redundancy, not the whole cluster.

**Why it works this way:** every partition has exactly one **leader** broker that handles all writes for it, and a topic's partitions are spread over many brokers. So the producer cannot just "send to Kafka" — it must reach the *specific* broker leading the target partition. Since the producer can't know that up front, it asks any broker for the cluster map first.

**Step by step:**
1. Producer connects to one of the `bootstrap.servers` and sends a `Metadata` request.
2. The broker replies with every broker's *advertised* host:port and, for each partition of the topic, who the leader is (e.g. `orders-0 → broker 2`, `orders-1 → broker 3`).
3. Producer caches this map. For each record it picks a partition (key hash / sticky), looks up that partition's leader, and sends the batch straight there.
4. If the leader moved (broker crash, rebalance of leadership), the broker answers `NOT_LEADER_OR_FOLLOWER`; the producer refreshes metadata and retries against the new leader — no application code needed.

```properties
bootstrap.servers=kafka1:9092,kafka2:9092,kafka3:9092   # seeds only
```

**Pitfalls / follow-ups:**
- **`advertised.listeners` misconfiguration** is the classic Docker/Kubernetes bug: the client reaches the bootstrap broker, gets back an address like `localhost:9092` or an internal hostname it can't resolve, and then fails/times out on the *second* hop. Bootstrap works, sends don't.
- A producer sends to the leader only — followers are never written to directly by clients; they replicate from the leader.
- Interview probe: "What if all bootstrap brokers are down at startup?" → the client can't fetch metadata and fails (`TimeoutException`), which is why you list more than one.

---

### Q5. What is the controller and what does it do?

One broker (or a quorum in KRaft) is the **controller**. Responsibilities:
- Elect partition leaders when a broker dies.
- Maintain and propagate the ISR list.
- Handle topic creation/deletion, partition reassignment.

If the controller dies, a new one is elected (ZK ephemeral node race, or Raft election in KRaft). It's a **metadata** role — it does not sit in the data path.

**Why it exists:** in a cluster, *someone* has to make cluster-wide decisions consistently — "broker 2 died, so who becomes leader for the 40 partitions it led?" If every broker decided on its own you'd get two leaders for one partition (split brain). The controller is the single decision-maker, so decisions are serialized and consistent.

**Example — a broker dies:**
1. The controller notices the broker is gone (ZK session expiry, or missed heartbeats in KRaft).
2. For every partition that broker led, it picks a new leader from that partition's **ISR** (in-sync replica set — replicas fully caught up with the leader).
3. It records the new leader/ISR in the metadata and tells the other brokers (pushed via RPC in ZK mode; brokers read the metadata log in KRaft).
4. Producers/consumers get `NOT_LEADER_OR_FOLLOWER`, refresh metadata, and continue with the new leader.

**Key points:**
- **Data path vs control path:** producing and consuming never go through the controller; it is only involved when something *changes* (broker joins/leaves, topic created, ISR shrinks).
- **Don't confuse** the *controller* (one per cluster, metadata) with a partition *leader* (one per partition, data) and the *group coordinator* (a broker per consumer group, handles rebalances/offset commits).
- In ZK mode a controller failover meant reloading all state from ZooKeeper — slow with 100k+ partitions; KRaft keeps a hot standby quorum, so failover is near-instant (see Q3).

---

## 2. Producer Internals

### Q6. Walk through the producer send path.

**Say this first:** "`send()` is asynchronous — it serializes, partitions, and appends to an in-memory batch; a background I/O thread ships batches to leaders."

```
send(record)
  → Interceptors
  → Serializer (key, value)
  → Partitioner  → decides partition
  → RecordAccumulator (per-partition batch buffer, size = buffer.memory 32MB)
  → [Sender thread] groups batches by broker → one request per broker
  → Broker leader appends → replicates to ISR → sends back ack
  → Callback / Future completes
```

Critical knobs on this path:

- `batch.size` (16 KB) — max bytes per partition batch. Bigger = better compression + throughput.
- `linger.ms` (0) — how long to wait to fill a batch. **Setting 5–20 ms is the single biggest throughput win.**
- `buffer.memory` (32 MB) — total producer buffer. When full, `send()` blocks up to `max.block.ms` (60 s) then throws `TimeoutException`.
- `compression.type` (none) — use `lz4` or `zstd`; compression happens **per batch**, so batching and compression amplify each other.
- `max.request.size` (1 MB) and broker `message.max.bytes` (1 MB) must be raised together for large payloads.

🧠 **Memory hook:** *"Batch, Linger, Buffer, Compress" = BLBC = "Big Loads Become Cheap."*

---

### Q7. Explain `acks` in depth.

| `acks` | Meaning | Durability | Latency |
|---|---|---|---|
| `0` | Fire and forget, no response | Can lose everything | Lowest |
| `1` | Leader wrote to its log | Loses data if leader dies before followers replicate | Medium |
| `all` / `-1` | All **in-sync replicas** acked | No loss if `min.insync.replicas >= 2` | Highest |

**The nuance that impresses interviewers:** `acks=all` alone is **not** enough. If `min.insync.replicas=1` and all followers fall out of the ISR, then "all in-sync replicas" = just the leader, and you're effectively back to `acks=1`.

**The durable combo:**
```
replication.factor = 3
min.insync.replicas = 2      (topic-level)
acks = all                   (producer)
unclean.leader.election.enable = false
```
This tolerates **1** broker loss with zero data loss, and stays available. With RF=3/minISR=2, losing 2 brokers makes the partition **unavailable for writes** (`NotEnoughReplicasException`) — that's the deliberate CP tradeoff.

🧠 **Memory hook:** *3-2-all-false* — chant it. RF 3, minISR 2, acks all, unclean false.

**Default note:** since Kafka 3.0 the Java producer defaults to `acks=all` and `enable.idempotence=true`.

---

### Q8. What is producer idempotence and how does it work internally?

**Say this first:** "It de-duplicates producer retries at the broker so a network-level retry doesn't create a duplicate record."

**The problem it solves:** the broker writes your record, but the ack is lost on the network. The producer can't tell "write failed" from "ack lost", so it retries — and without idempotence the broker appends the record a *second* time (a duplicate). Idempotence gives each batch a number so the broker can recognise "I've already seen this one".

Mechanics:
- Broker assigns the producer a **PID** (producer ID).
- Every record batch carries `(PID, epoch, sequence number)` per partition.
- The broker keeps the last 5 sequence numbers per PID/partition. If it sees a sequence it already committed → discards and returns success. If it sees a gap → `OutOfOrderSequenceException`.

Requirements (auto-enforced when `enable.idempotence=true`): `acks=all`, `retries > 0`, `max.in.flight.requests.per.connection <= 5`.

**Scope of the guarantee:** exactly-once *per partition, per producer session*. It does **not** protect against:
- Application-level double `send()` (your code calling send twice).
- **Producer restart (new PID)** — that's what transactional `transactional.id` fixes.

**Worked example:** producer sends batch with sequence 7 → broker appends it → ack lost → producer retries the same batch (still sequence 7) → broker sees 7 was already committed, skips the append and replies success. One copy stored. If instead sequence 9 arrived when the broker expected 8, something was lost in between, so it raises `OutOfOrderSequenceException` (see S30).

```properties
enable.idempotence=true     # default since Kafka 3.0; no other code changes
```

🧠 **Memory hook:** *PID + Epoch + Sequence = "PES" — the broker's "PES-t control" for duplicates.*

---

### Q9. Can retries reorder messages? How do you prevent it?

Yes — classically. With `max.in.flight=5` and no idempotence: batch1 fails, batch2 succeeds, batch1 succeeds on retry → order is 2,1.

**Why it happens:** `max.in.flight.requests.per.connection` is how many un-acked requests the producer may have on the wire to one broker at once. With several in flight, the producer doesn't wait for batch 1's ack before sending batch 2. If batch 1 hits a transient error (e.g. leader election) and is retried *after* batch 2 already landed, the log ends up `[B2, B1]`. Example: `ORDER_CREATED` then `ORDER_CANCELLED` ends up stored as cancelled-then-created — a consumer would see an order "created" after it was cancelled.

**Fixes, best to worst:**
1. `enable.idempotence=true` — broker rejects out-of-sequence batches, so ordering is preserved *even with 5 in-flight*. This is the modern answer.
2. `max.in.flight.requests.per.connection=1` — safe but ~5x throughput hit. Legacy answer.
3. `retries=0` — preserves order but you now lose messages.

🧠 **Memory hook:** *Idempotence buys you order AND throughput. `max.in.flight=1` is the old, slow way.*

---

### Q10. How does partitioning work? Explain the default partitioner and sticky partitioning.

- **Key present:** `partition = murmur2(keyBytes) % numPartitions`. Same key → same partition → ordering per key. Note it's `murmur2`, not `hashCode()` — deliberately consistent across languages.
- **Key null (Kafka ≥ 2.4):** **Sticky partitioner** — sends to one partition until the batch is full/linger expires, then switches. This improves batching vs. old round-robin per record.
- **Explicit partition** in `ProducerRecord` overrides everything.

**Why sticky?** Old behaviour for null keys was round-robin *per record*, so a burst of 100 records became 100 tiny one-record batches (one per partition) — lots of requests, poor compression. Sticky sends the whole burst to one partition until its batch is full (`batch.size`) or `linger.ms` expires, then picks another. Distribution is still even over time, but batches are full and requests are fewer. (Since 3.3 the default is further refined with a load-aware "adaptive" variant that favours faster brokers.)

**Why keys matter:** use a business key (`customerId`, `orderId`) when you need ordering for that entity; use `null` keys when order doesn't matter and you want best throughput/balance. A skewed key (one huge customer) creates a hot partition — see S9.

**Custom partitioner** — implement `Partitioner`, set `partitioner.class`. Real use case: tenant isolation, or routing VIP customers to dedicated partitions.

```java
public class TenantPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        int n = cluster.partitionsForTopic(topic).size();
        if (key == null) return 0;
        String tenant = key.toString();
        // reserve last partition for premium tenants
        return tenant.startsWith("PREMIUM_") ? n - 1
             : Math.abs(Utils.murmur2(keyBytes)) % (n - 1);
    }
    @Override public void close() {}
    @Override public void configure(Map<String, ?> configs) {}
}
```

**Interview trap:** "If you add partitions, does key→partition mapping stay stable?" → **No.** `hash % N` changes when N changes, so existing keys move to different partitions and per-key ordering breaks across the boundary.

---

### Q11. `delivery.timeout.ms` vs `request.timeout.ms` vs `retries` — which wins?

- `request.timeout.ms` (30 s) — timeout for **one** network request.
- `retry.backoff.ms` (100 ms) — wait between retries.
- `delivery.timeout.ms` (120 s) — **total** upper bound from `send()` returning until success/failure. This is the *authoritative* cap.

Since 2.1, `retries` defaults to `Integer.MAX_VALUE` and is effectively irrelevant — the producer retries until `delivery.timeout.ms` expires. **Tune `delivery.timeout.ms`, not `retries`.**

Constraint: `delivery.timeout.ms >= request.timeout.ms + linger.ms`.

**How they fit together (timeline of one record):**
```
send() returns ──[ linger.ms in buffer ]──[ attempt 1: up to request.timeout.ms ]─ fail ─[ retry.backoff.ms ]─[ attempt 2 ... ]
|<──────────────────────── delivery.timeout.ms (120 s total) ─────────────────────────────────────────────────>|
```
Each attempt is capped by `request.timeout.ms` (30 s); the producer keeps retrying until the *whole* 120 s budget is spent, then fails the record with a `TimeoutException` delivered to your callback/future. So with defaults you get roughly 3–4 attempts, regardless of what `retries` says.

**Practical tuning:** want to fail fast (e.g. an HTTP request waits on the send)? Lower `delivery.timeout.ms` (say 10 s) and keep `request.timeout.ms` below it. Want to ride out a 5-minute broker outage without dropping data? Raise `delivery.timeout.ms` — but remember `buffer.memory` fills meanwhile and `send()` will block (Q6).

🧠 **Memory hook:** *delivery.timeout is the "hard deadline"; request.timeout is "one attempt."*

---

### Q12. Sync vs async send in Spring — show the code.

```java
// Fire-and-forget (fastest, no error visibility) — rarely correct
kafkaTemplate.send("orders", key, event);

// Async with callback (recommended default)
kafkaTemplate.send("orders", key, event)
    .whenComplete((result, ex) -> {
        if (ex != null) {
            log.error("Publish failed key={}", key, ex);
            outboxRepo.markFailed(key);            // compensating action
        } else {
            RecordMetadata md = result.getRecordMetadata();
            log.debug("Sent to {}-{}@{}", md.topic(), md.partition(), md.offset());
        }
    });

// Synchronous (blocking) — only when caller must know before responding
try {
    SendResult<String, OrderEvent> r =
        kafkaTemplate.send("orders", key, event).get(5, TimeUnit.SECONDS);
} catch (ExecutionException e) {
    throw new PublishFailedException(e.getCause());
}
```

Note: Spring Kafka 3.x returns `CompletableFuture` (2.x returned `ListenableFuture`). Blocking with `.get()` inside a loop destroys throughput because it defeats batching — that's a classic follow-up.

---

## 3. Consumer & Consumer Groups

### Q13. Explain consumer groups and the partition-to-consumer rule.

**Say this first:** "Within a group, each partition is assigned to exactly one consumer. Consumers scale up to the partition count and no further."

```
Topic orders (4 partitions)

Group A (2 consumers):  C1 → P0,P1   C2 → P2,P3
Group A (4 consumers):  C1→P0 C2→P1 C3→P2 C4→P3
Group A (5 consumers):  5th sits IDLE  ← partitions are the ceiling
Group B (1 consumer):   gets ALL 4 — independent offsets from Group A
```

Rules to state crisply:
1. Max useful consumers in a group = number of partitions.
2. Different groups get **all** messages independently (pub/sub fan-out).
3. Same group = competing consumers (queue semantics).
4. `group.id` identity determines offset ownership. Change it → you're a new group → `auto.offset.reset` applies.

🧠 **Memory hook:** *Partitions are seats; consumers are guests. Extra guests stand.*

---

### Q14. What triggers a rebalance and why should you fear it?

**Triggers:**
- Consumer joins / leaves gracefully.
- Consumer times out (`session.timeout.ms` missed heartbeat, or `max.poll.interval.ms` exceeded).
- Partition count of a subscribed topic increases.
- Subscribed topics change (regex subscription matching a new topic).
- Group coordinator (broker) changes.

**Why it hurts:** with the default **eager** protocol, *all* consumers stop consuming ("stop-the-world"), revoke everything, and re-assign. On a big group with long assignment this can pause processing for seconds — and lag spikes.

**Mitigations (know all four):**
1. **`CooperativeStickyAssignor`** — incremental rebalance; only moved partitions are revoked, everyone else keeps consuming.
2. **Static membership** (`group.instance.id`, KIP-345) — a pod restart within `session.timeout.ms` does **not** trigger rebalance. Perfect for Kubernetes StatefulSets.
3. Tune `max.poll.records` down / `max.poll.interval.ms` up so slow processing doesn't look like death.
4. Graceful shutdown (`consumer.close()`) so the coordinator learns immediately instead of waiting for the timeout.

```yaml
spring:
  kafka:
    consumer:
      group-id: order-service
      properties:
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
        group.instance.id: ${HOSTNAME}      # static membership
        session.timeout.ms: 45000
        max.poll.interval.ms: 300000
```

🧠 **Memory hook:** *"Cooperative + Static = calm groups."*

---

### Q15. Compare the partition assignment strategies.

| Strategy | Behaviour | Notes |
|---|---|---|
| `RangeAssignor` | Per topic, assigns contiguous ranges | Default (with Cooperative) in 3.x; skews badly with multiple topics |
| `RoundRobinAssignor` | Round-robins all partitions across all consumers | Better balance; full revoke on rebalance |
| `StickyAssignor` | Balanced + minimizes movement | Still eager (stop-the-world) |
| `CooperativeStickyAssignor` | Sticky + **incremental** rebalance | **Recommended default today** |

**Migration gotcha:** you cannot jump straight from eager to cooperative in one deploy. Two-step rolling upgrade: deploy with `[CooperativeSticky, Range]`, then remove the eager one.

**What the terms mean:**
- **Assignor** = the algorithm that decides which consumer gets which partitions; the group leader (one consumer) runs it on each rebalance.
- **Eager** = every consumer gives up *all* its partitions, then everything is re-assigned (stop-the-world). **Cooperative** = only partitions that actually need to move are revoked, in a second round; the rest keep being processed.
- **Sticky** = try to keep each partition with its previous owner, so local caches/state aren't thrown away.

**Example — topic with 6 partitions, consumers C1, C2; C3 joins:**
- Range/RoundRobin/Sticky (eager): all of P0–P5 are revoked from C1, C2 → reassigned (2 each); processing pauses for everyone.
- CooperativeSticky: C1 and C2 each give up just 1 partition (the ones going to C3) and keep working on the other 2–3.

**Why Range skews:** it computes ranges *per topic*, so with 3 topics × 4 partitions and 3 consumers, the first consumer always gets the extra partition of each topic → it ends up with more work.

---

### Q16. `session.timeout.ms` vs `max.poll.interval.ms` vs `heartbeat.interval.ms`

This is asked constantly. Get it right:

- **`heartbeat.interval.ms` (3 s)** — how often the **background heartbeat thread** pings the coordinator.
- **`session.timeout.ms` (45 s)** — coordinator declares the consumer dead if no heartbeat within this window. Detects *crashed process / network partition*.
- **`max.poll.interval.ms` (5 min)** — max time between two `poll()` calls. Detects *live process, stuck processing*. Exceeding it makes the consumer proactively leave the group; the subsequent commit fails with `CommitFailedException`.

Rule of thumb: `heartbeat.interval.ms ≈ session.timeout.ms / 3`.

🧠 **Memory hook:** *Session = "are you alive?" (heartbeat thread). MaxPollInterval = "are you making progress?" (main thread).*

**Since heartbeats are on a separate thread, a hung processing loop keeps you "alive" but not "progressing" — that's exactly why `max.poll.interval.ms` exists.**

**Failure scenarios — which timer fires?**
| What happened | Detected by | Result |
|---|---|---|
| Pod killed / network cut | `session.timeout.ms` (no heartbeats for 45 s) | Coordinator removes consumer, rebalance |
| App alive, but one record takes 10 min (DB stuck) | `max.poll.interval.ms` (no `poll()` for 5 min) | Consumer leaves group, rebalance; its next commit fails |
| Healthy | — | Heartbeat every 3 s, poll often |

**Typical fix for "rebalance every few minutes":** the batch from `poll()` (`max.poll.records` × per-record time) takes longer than `max.poll.interval.ms`. Either lower `max.poll.records`, speed up processing, or raise `max.poll.interval.ms` (S4).

---

### Q17. How does offset commit work? Auto vs manual, and the modes in Spring.

Offsets live in `__consumer_offsets`, keyed by `(group, topic, partition)`, compacted.

**Committed offset = next offset to read**, i.e. `lastProcessed + 1`. (Frequent trick question.)

**Auto-commit** (`enable.auto.commit=true`, every `auto.commit.interval.ms=5000`): commits happen inside `poll()` for the *previous* batch. Danger: records fetched but not yet processed can be committed → **message loss** on crash. Spring Boot sets this to `false` and manages commits itself.

**Spring `AckMode` values:**

| AckMode | Commits when |
|---|---|
| `BATCH` (default) | After all records from one `poll()` are processed |
| `RECORD` | After each record's listener returns |
| `TIME` | Every `ackTime` ms |
| `COUNT` | Every `ackCount` records |
| `COUNT_TIME` | Whichever comes first |
| `MANUAL` | Your `ack.acknowledge()` queued, committed at end of batch |
| `MANUAL_IMMEDIATE` | Your `ack.acknowledge()` commits immediately |

```java
@KafkaListener(topics = "orders", containerFactory = "manualAckFactory")
public void consume(ConsumerRecord<String, OrderEvent> rec, Acknowledgment ack) {
    try {
        orderService.process(rec.value());
        ack.acknowledge();                 // commit only after success
    } catch (TransientException e) {
        ack.nack(Duration.ofSeconds(5));   // re-deliver after pause, no commit
    }
}
```

🧠 **Memory hook:** *"Commit AFTER processing = at-least-once. Commit BEFORE = at-most-once."*

---

### Q18. Explain `auto.offset.reset`.

Applies **only** when there is no valid committed offset for the group (brand-new group, or the offset expired/was deleted).

- `latest` (default) — start at the end; you skip existing backlog.
- `earliest` — start from the beginning; you reprocess everything.
- `none` — throw `NoOffsetForPartitionException`.

**Why it exists:** a consumer needs a starting point. Normally it resumes from the group's committed offset in `__consumer_offsets`. But a brand-new `group.id` has no entry — so Kafka needs a rule: start at the live edge (`latest`) or from the oldest retained message (`earliest`).

**Example:** topic holds offsets 0–9,999. A new group starts:
- `latest` → sees only messages produced *after* it starts (offset 10,000+). Anything sent earlier is silently skipped — a common "my consumer receives nothing" surprise in dev.
- `earliest` → reads all 10,000 retained messages first.

```yaml
spring.kafka.consumer.auto-offset-reset: earliest   # typical for services that must not miss data
```

Also applies if the committed offset is *out of range* (e.g. the segment holding it was deleted by retention). Use `none` when you'd rather fail loudly than silently skip or replay.

**Interview trap:** "My consumer restarted and reprocessed everything — why?" → Its committed offsets were gone (`offsets.retention.minutes`, default 7 days of group inactivity) *and* `auto.offset.reset=earliest`. Or someone changed the `group.id`.

---

### Q19. How do you seek to a specific offset or timestamp?

**Why you'd do it:** replay after a bug fix, skip a poisoned region, or start from "yesterday" for a new feature. Normally the consumer continues from its committed offset; *seeking* overrides the position the next `poll()` reads from.

**Two ways:** inside the app (Spring's `ConsumerSeekAware`, below) or externally with the CLI (further below). Timestamp seeks work because Kafka keeps a `.timeindex` per segment (Q2) that maps timestamp → offset; the plain-consumer API is `offsetsForTimes()` followed by `seek()`.

```java
@Component
public class ReplayListener implements ConsumerSeekAware {

    @Override
    public void onPartitionsAssigned(Map<TopicPartition, Long> assignments,
                                     ConsumerSeekCallback cb) {
        // replay last 24 hours
        long ts = System.currentTimeMillis() - Duration.ofDays(1).toMillis();
        assignments.keySet().forEach(tp -> cb.seekToTimestamp(tp, ts));
    }
}
```
Other options: `cb.seekToBeginning(...)`, `cb.seekToEnd(...)`, `cb.seek(tp, offset)`, `cb.seekRelative(topic, partition, -1000, true)`.

Or externally, without code: `kafka-consumer-groups.sh --reset-offsets --to-datetime 2026-09-01T00:00:00.000 --group order-service --topic orders --execute` (group must be **inactive**).

**Pitfalls:** seeking in `onPartitionsAssigned` runs on *every* rebalance, so the replay will repeat each time a partition is reassigned — guard it with a flag/config if it should happen once. Seeking changes only where you *read*; it doesn't commit anything until the next commit. Replays mean duplicates, so the consumer must be idempotent (Q26). Run the CLI with `--dry-run` first to preview.

---

### Q20. What is consumer lag and how do you monitor it?

**Lag = log-end-offset (LEO) − committed offset**, per partition.

In plain words: LEO is the offset of the next message to be written (how much the producers have written); the committed offset is how far the group has processed. The gap is the backlog of unprocessed messages. **Example:** LEO=10,500, committed=10,000 → lag=500. Total group lag is the sum across partitions; one partition with huge lag while the rest are zero points to a hot key or a stuck consumer on that partition.

```bash
kafka-consumer-groups.sh --bootstrap-server broker:9092 \
  --describe --group order-service
```

In production: Burrow, Kafka Exporter → Prometheus → Grafana, or Confluent Control Center. Alert on **lag derivative** (is it growing?) not just absolute value — a batch job legitimately has high lag momentarily.

Micrometer exposes `kafka.consumer.fetch.manager.records.lag.max` automatically in Spring Boot when you register the metrics.

**Reading the `--describe` output:** columns `CURRENT-OFFSET` (committed), `LOG-END-OFFSET`, `LAG`, plus `CONSUMER-ID`/`HOST` showing which instance owns each partition. A partition with no consumer id means nobody owns it (group has too few consumers or is rebalancing).

**Lag in time, not just messages:** 10,000 messages of lag is trivial at 50k msg/s and serious at 10 msg/s. Convert to "minutes behind" (lag ÷ consumption rate) for alerting and SLAs.

🧠 **Memory hook:** *Lag = "how far behind the live edge." Rising lag = consumer slower than producer. Period.*

---

### Q21. Why is `KafkaConsumer` not thread-safe, and how do you parallelize?

`KafkaConsumer` is **single-threaded by contract** (only `wakeup()` is safe from another thread).

**Why not thread-safe:** the consumer holds mutable state — current positions, group membership, the fetch buffer, offsets to commit — and is designed around one thread driving `poll()`. Making it lock-protected would slow every call, so the client simply forbids concurrent access and throws `ConcurrentModificationException` ("KafkaConsumer is not safe for multi-threaded access") if you try.

Options for parallelism:

1. **More partitions + more consumer instances** — the idiomatic scale path.
2. **`concurrency` in Spring** — creates N consumers in one app, each with its own thread and partition subset.
   ```java
   factory.setConcurrency(3);   // ≤ partition count, else idle threads
   ```
3. **Hand off to a worker pool** — fastest, but you break offset ordering and at-least-once unless you carefully track completion. Use `ContainerProperties.AckMode.MANUAL` plus a per-partition ordering guard. Mention `pause()`/`resume()` for backpressure.

🧠 **Memory hook:** *One consumer, one thread. Want parallelism? Add partitions, not threads.*

---

### Q22. What happens on `poll()`?

1. Join group / get assignment if needed (first poll).
2. Send fetch requests to leaders of assigned partitions.
3. Return up to `max.poll.records` (500) records from the internal buffer.
4. Trigger auto-commit of the previous batch (if enabled).
5. Trigger `ConsumerRebalanceListener` callbacks if a rebalance happened.

Fetch tuning: `fetch.min.bytes` (1) — broker waits for this much data; `fetch.max.wait.ms` (500) — cap on that wait; `max.partition.fetch.bytes` (1 MB); `fetch.max.bytes` (50 MB).

**Note:** `max.poll.records` limits what's *returned*, not what's *fetched*. Records already in the buffer are returned across subsequent polls without new network calls.

**Mental model:** `poll()` is the consumer's heartbeat of work — your whole app is `while (true) { records = poll(); process(records); }`. It does the network I/O, group membership and delivering results. Hence two rules: call it *regularly* (else `max.poll.interval.ms` kicks you out, Q16), and never block inside the loop for long.

```java
while (running) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(500)); // waits up to 500 ms for data
    for (ConsumerRecord<String, String> r : records) handle(r);
    consumer.commitSync();   // after processing → at-least-once
}
```
The `Duration` argument is the max time to *wait for data* when the buffer is empty — not a processing timeout. Spring's listener container runs this loop for you.

---

## 4. Delivery Semantics & Exactly-Once

### Q23. Explain at-most-once, at-least-once, exactly-once with the exact config for each.

| Semantic | Producer | Consumer | Result |
|---|---|---|---|
| **At-most-once** | `acks=0` or `1`, `retries=0` | Commit offset **before** processing | May lose, never duplicates |
| **At-least-once** | `acks=all`, retries on | Commit offset **after** processing | Never loses, may duplicate ← **99% of real systems** |
| **Exactly-once** | `enable.idempotence=true` + `transactional.id` | `isolation.level=read_committed`, offsets committed inside the transaction | No loss, no duplicates — but only Kafka→Kafka |

**What each looks like when a crash happens** (consumer reads message M, calls the DB, then dies):
- *At-most-once:* offset was committed **before** processing → after restart M is skipped. **Lost**, but never repeated. Fine for metrics/log sampling.
- *At-least-once:* offset is committed **after** processing → if the crash falls between "DB written" and "offset committed", M is redelivered and the DB write runs again. **Duplicate**, but never lost. Needs an idempotent consumer (Q26).
- *Exactly-once:* the output records *and* the consumed offset are committed atomically in one Kafka transaction, so after a crash either both happened or neither did. Only possible because input, output and offsets all live in Kafka.

**Config sketch for at-least-once (the one you'll actually use):**
```properties
# producer
acks=all
enable.idempotence=true
# consumer
enable.auto.commit=false        # commit manually after the work is done
```

**The honest senior answer:** "Exactly-once in Kafka is real but only end-to-end *within* Kafka (consume → process → produce → commit offsets, all in one transaction). The moment you write to an external system — a DB, a REST call, an email — you're back to at-least-once, and you achieve effective exactly-once by making the **consumer idempotent**. In practice I design for at-least-once delivery plus idempotent consumers."

🧠 **Memory hook:** *"Kafka gives EOS inside its own walls. Outside the walls, you bring your own idempotency."*

---

### Q24. How do Kafka transactions actually work?

Components: **Transaction Coordinator** (a broker) + internal topic `__transaction_state`.

**The problem:** a stream processor does "read order → write invoice → write audit → mark order as consumed". If it crashes midway you'd get an invoice with no audit record, or a re-processed order producing a second invoice. A transaction makes the whole group of writes (including the consumer offset commit) **all-or-nothing**.

```
initTransactions()            → registers transactional.id, fences old PID by bumping epoch
beginTransaction()
send(...) send(...)           → records written to partitions immediately, marked "uncommitted"
sendOffsetsToTransaction(...) → offsets go to __consumer_offsets as part of the txn
commitTransaction()           → coordinator writes COMMIT markers to every touched partition
                                (or ABORT markers on abort)
```

- Records are written to the log **eagerly**; commit/abort is decided later by a **control marker**.
- A consumer with `isolation.level=read_committed` filters out aborted records and stops reading at the **LSO** (Last Stable Offset) — it will not read past an open transaction. That's why a hung transaction stalls consumers.
- `transactional.id` must be **stable across restarts** to get zombie fencing. If each pod generates a random one, you lose the guarantee.
- `transaction.timeout.ms` (60 s) — coordinator aborts a hanging transaction after this.

**Zombie fencing:** an old instance that comes back with the same `transactional.id` but a stale epoch gets `ProducerFencedException` and must die. This is what prevents a split-brain double-write.

**Plain-Java shape:**
```java
producer.initTransactions();
while (true) {
    var records = consumer.poll(Duration.ofMillis(500));
    producer.beginTransaction();
    try {
        for (var r : records) producer.send(new ProducerRecord<>("invoices", r.key(), transform(r.value())));
        producer.sendOffsetsToTransaction(currentOffsets(records), consumer.groupMetadata());
        producer.commitTransaction();
    } catch (ProducerFencedException e) { producer.close(); throw e; }   // zombie: must stop
    catch (KafkaException e) { producer.abortTransaction(); /* seek back and retry */ }
}
```
**Cost:** transactions add latency (extra commit round trips per batch), so commit per *batch* of records rather than per record; and `read_committed` consumers see data only after commit.

🧠 **Memory hook:** *Records first, verdict later. The COMMIT marker is the judge; read_committed consumers wait for the verdict.*

---

### Q25. Show exactly-once in Spring Boot.

```yaml
spring:
  kafka:
    producer:
      transaction-id-prefix: order-tx-       # Spring appends a suffix per producer
      acks: all
      properties:
        enable.idempotence: true
    consumer:
      isolation-level: read_committed
      enable-auto-commit: false
```

```java
@Service
public class OrderProcessor {

    private final KafkaTemplate<String, Object> template;

    // Setting transaction-id-prefix auto-creates a KafkaTransactionManager;
    // @Transactional on a listener makes consume+produce+offset-commit atomic.
    @KafkaListener(topics = "orders")
    @Transactional("kafkaTransactionManager")
    public void handle(OrderEvent event) {
        InvoiceEvent invoice = invoiceService.build(event);
        template.send("invoices", event.getOrderId(), invoice);
        template.send("audit",    event.getOrderId(), event.toAudit());
        // offsets committed by the container inside the same transaction
    }
}
```

**What Spring is doing for you:** the listener container opens a Kafka transaction before calling `handle()`, runs your method, then sends the consumed offsets *into* that transaction and commits it. If `handle()` throws, the transaction aborts: `invoices`/`audit` records are marked aborted (invisible to `read_committed` readers) and the offset isn't committed, so the record is redelivered. Downstream consumers **must** use `isolation-level: read_committed`, otherwise they will still see the aborted records. With `transaction-id-prefix` set, Spring also uses a stable per-instance id, so fencing works across restarts.

**Follow-up they love:** "What if you also write to Postgres in there?" → Kafka and JDBC are separate resource managers. Options:
- `ChainedKafkaTransactionManager` (deprecated in Spring Kafka 2.7+) — best-effort 1PC; a crash between the two commits still breaks atomicity.
- **Correct answer: Transactional Outbox pattern** (see Q60) or an idempotent DB write keyed on a business/message ID.

---

### Q26. How do you build an idempotent consumer? (Asked in almost every senior interview.)

**Say this first:** "I give every event a stable unique ID and make the write side reject or absorb repeats."

Four techniques, from cheapest to strongest:

1. **Natural idempotency** — the operation is already safe to repeat (`SET status='PAID'`, `PUT` semantics). Free. Prefer this.
2. **Unique constraint / upsert:**
   ```sql
   INSERT INTO payments (event_id, order_id, amount)
   VALUES (?, ?, ?) ON CONFLICT (event_id) DO NOTHING;
   ```
3. **Processed-events table in the same DB transaction** — the durable, general solution:
   ```java
   @Transactional
   public void handle(PaymentEvent e) {
       if (processedRepo.existsById(e.getEventId())) return;    // dedupe
       ledger.apply(e);
       processedRepo.save(new Processed(e.getEventId(), Instant.now()));
   }
   ```
   Both writes commit or roll back together → safe to redeliver. Purge the table on a TTL.
4. **Version / optimistic concurrency** — carry an entity version in the event, apply only if `event.version > current.version`. Also fixes out-of-order updates.

Redis SETNX is fine as a *fast path* but not authoritative — if Redis loses the key you double-process. Say that.

🧠 **Memory hook:** *"Same DB transaction as the business write, or it isn't a guarantee."*

---

## 5. Replication, Durability & ISR

### Q27. What is the ISR and when does a replica get kicked out?

**ISR (In-Sync Replicas)** = replicas that have caught up with the leader within `replica.lag.time.max.ms` (default **30 s**). The leader is always in the ISR.

A follower is removed when it hasn't sent a fetch request, or hasn't caught up to the leader's log end offset, within that window. It rejoins once it catches up.

**How replication works (the context):** each partition has one leader and RF−1 followers. Followers continuously *fetch* from the leader just like consumers do and append to their own copy. The ISR is the live list of "followers that are keeping up". It exists so the leader can answer "is this write safely stored on enough copies?" without waiting for a slow or dead broker forever.

**Example (RF=3, `min.insync.replicas=2`):**
- ISR = {B1(leader), B2, B3}. `acks=all` waits for all three.
- B3 hits a long GC pause and falls >30 s behind → ISR shrinks to {B1, B2}. Writes continue, now acked by two copies.
- B3 catches up → rejoins the ISR.
- B2 also dies → ISR = {B1} < `min.insync.replicas` → writes with `acks=all` fail with `NotEnoughReplicasException`, protecting you from a single-copy write.

**Common causes of ISR shrink:** slow disks, long GC pauses, network saturation, a broker under-provisioned for its partition count. Frequent `IsrShrinksPerSec`/`IsrExpandsPerSec` flapping = unhealthy broker.

**Why it matters:** `acks=all` waits for the ISR, and only ISR members are eligible to become leader (when unclean election is disabled).

**Metrics to name:** `UnderReplicatedPartitions` (ISR < RF — investigate), `OfflinePartitionsCount` (no leader — outage!), `IsrShrinksPerSec`.

---

### Q28. What is unclean leader election?

`unclean.leader.election.enable=false` (default): if no ISR replica is available, the partition goes **offline** — choose consistency, sacrifice availability.

If `true`: an out-of-sync replica becomes leader → **silent data loss** (records the old leader had are truncated away). Only enable it for data where availability trumps correctness (e.g., metrics firehose).

**Scenario that shows why:** RF=3, ISR={B1 leader, B2}; B3 is lagging at offset 80 while the leader is at offset 100. B1 and B2 both crash.
- *Clean (default):* the partition stays offline until B1 or B2 returns. No data lost, but producers/consumers are blocked.
- *Unclean:* B3 is promoted and accepts new writes starting at offset 81. When B1/B2 return, they must truncate offsets 81–100 to match the new leader → those 20 acknowledged records are gone forever, and consumers who already read them now see a different history.

It's the **CP vs AP** knob for a partition: consistency (default) vs availability. Combined with `acks=all` + `min.insync.replicas=2` it's what makes "3-2-all-false" (Q7) lossless.

🧠 **Memory hook:** *Unclean = "dirty leader" = accepts a replica that missed messages = data loss.*

---

### Q29. What are HW, LEO and LSO?

- **LEO (Log End Offset)** — next offset to be written on that replica.
- **HW (High Watermark)** — highest offset replicated to **all** ISR members. **Consumers can only read up to HW−1.** This prevents reading records that could be lost in a leader failover.
- **LSO (Last Stable Offset)** — with transactions, the offset before the first open transaction. `read_committed` consumers stop here.

**Picture (one partition, leader view):**
```
offset:   0 ... 95  96  97  98  99 | 100 101 102
                        ↑                  ↑
                   HW = 98                LEO = 103 (next write goes here)
          consumers may read up to offset 97 only; 98–102 are not yet on all ISR members
```
**Why the HW exists:** if the leader crashed, a follower missing offsets 98–102 becomes leader and those records vanish. Had a consumer already read them, it would have seen data that "never existed". Holding readers back to the HW guarantees everything a consumer sees survives failover.

**How the HW advances:** followers report their fetch position to the leader; the HW is the minimum LEO across the ISR. Followers learn the new HW in their next fetch response — which is why the HW on a follower briefly lags the leader's (and why epochs, Q30, were introduced).

**LSO example:** a producer has an open transaction starting at offset 120; messages 121+ from other producers are already written. LSO = 120, so a `read_committed` consumer stalls at 120 until that transaction commits or aborts. A *hung transaction* therefore blocks `read_committed` consumers — it's governed by `transaction.timeout.ms`.

🧠 **Memory hook:** *LEO = written. HW = safely replicated (readable). LSO = safely committed (txn-readable).*

---

### Q30. What is leader epoch and what problem did it solve?

Each leader change increments an **epoch** number stored in `leader-epoch-checkpoint`. Followers truncate their log using epoch + offset rather than the high watermark alone.

This fixed two real data-loss/divergence bugs (KIP-101) where HW-based truncation could either lose committed records or leave replicas divergent after simultaneous restarts. If asked, this is a nice "I've read the KIPs" flex.

**Plain-language version:** the **epoch** is a "term number" for the leader of a partition (leader #1, leader #2, …). Each message batch is tagged with the epoch of the leader that wrote it, and the broker remembers "epoch N started at offset X".

**The problem:** after a crash a restarted follower must discard any suffix of its log that the new leader doesn't have. Old logic truncated to the *high watermark* (HW), but the HW on a follower can be stale. Truncating too far lost data that was actually committed; not truncating enough left a follower holding records the new leader never had (divergent logs, same offset → different data).

**The fix:** the restarted follower asks the new leader "what is the end offset of the epoch I last knew about?" (`OffsetForLeaderEpoch` request) and truncates exactly to that point. No guessing from the HW.

**Side benefit:** consumers and followers also use epochs to detect a stale leader (`FENCED_LEADER_EPOCH`) — a "zombie" old leader's responses are rejected.

---

### Q31. How many partitions and what replication factor should a topic have?

**Partitions — formula to say out loud:**
```
partitions = max( targetThroughput / producerThroughputPerPartition,
                  targetThroughput / consumerThroughputPerPartition )
```
Then add headroom because you can't easily shrink. Also: partitions ≥ max expected consumer instances.

**Worked example:** target 100 MB/s. Measure one producer thread to a partition at ~10 MB/s and one consumer at ~5 MB/s (slow because of DB calls). → max(100/10, 100/5) = **20 partitions**. Round up for growth (e.g. 24–30). Measure with `kafka-producer-perf-test.sh` and your real consumer logic rather than guessing.

**Why not just "a lot"?** Partitions are Kafka's unit of parallelism, but each is a directory of files on a broker, a leader to elect, and replication streams. And adding partitions later changes key→partition mapping (Q10) and breaks per-key ordering, so choose upfront with ~2–3× headroom.

**Costs of over-partitioning:** more open file handles, more memory for buffers on both sides, longer leader-election time on broker failure, more end-to-end latency (replication is per-partition), larger metadata.

Practical guidance: start with 6–12 for a mid-volume topic; 30 partitions per topic and ≤ ~4,000 partitions per broker is a comfortable zone in KRaft. **RF = 3** in production (2 gives no tolerance for a rolling restart + a failure), RF = 1 only for dev.

**Why RF=3:** with RF=3 and `min.insync.replicas=2` you can lose one broker (or take one down for a rolling upgrade) and still accept `acks=all` writes with zero data loss. RF=2 with minISR=2 becomes unavailable whenever either broker restarts; RF=2 with minISR=1 can lose data. Each extra replica costs disk + network (RF=3 = 3× storage and replication traffic), so go higher only for critical data. For rack/AZ awareness set `broker.rack` so replicas land in different zones.

---

## 6. Storage, Retention & Log Compaction

### Q32. Retention: time vs size vs compaction.

```yaml
cleanup.policy: delete            # or "compact", or "compact,delete"
retention.ms: 604800000           # 7 days (default)
retention.bytes: -1               # per partition, -1 = unlimited
segment.bytes: 1073741824         # 1 GB
segment.ms: 604800000             # force roll after 7 days
```

Key facts:
- Deletion happens at **segment granularity**, and the **active segment is never deleted**. So a topic can hold data older than `retention.ms` until the segment rolls. Great trick question.
- `retention.bytes` is **per partition**, not per topic. A 10 GB limit on 10 partitions = 100 GB.
- Whichever limit hits first wins.

**Three cleanup behaviours in plain words:**
- `delete` — old data is thrown away after a time (`retention.ms`) or size (`retention.bytes`) limit. Use for event streams ("orders in the last 7 days").
- `compact` — keep only the latest record per key, indefinitely. Use for "current state" topics (Q33).
- `compact,delete` — compacted *and* anything older than the retention window is deleted. Use for latest-state-per-key with an upper age bound.

**Why segments matter:** Kafka can't delete the middle of a file, so it deletes whole segment files. A segment becomes eligible only once it is *closed* (rolled at `segment.bytes` or `segment.ms`). Example: low-traffic topic, `retention.ms`=1 day, `segment.ms`=7 days → the active segment keeps rolling only weekly, so you may still see week-old messages. Lower `segment.ms` if you need retention to be accurate (e.g. GDPR deletion windows).

```bash
kafka-configs.sh --bootstrap-server broker:9092 --alter --entity-type topics \
  --entity-name orders --add-config retention.ms=86400000     # 1 day, no restart needed
```

---

### Q33. Explain log compaction with a concrete example.

Compaction retains the **latest value per key**, forever (or until you also set `delete`).

**Why:** many topics are really "the current state of X" (customer profile, product price). You don't need all 50 historic versions of key `customer-42` — only the newest — but you still want the topic replayable from offset 0 to rebuild a full table. Compaction bounds the size to "one record per key" instead of growing forever.

**How it works:** a background **log cleaner** thread periodically scans closed segments, builds a map of key → latest offset, and rewrites the segments keeping only records that are still the latest for their key. Offsets of surviving records never change (so gaps appear), and ordering is preserved.

```
Before:  (k1,v1) (k2,v1) (k1,v2) (k3,v1) (k1,v3) (k2,null)
After:                    (k1,v3) (k3,v1) (k2,null → tombstone)
```

- A **tombstone** (null value) marks a key for deletion; it's retained for `delete.retention.ms` (24 h) so consumers get a chance to see the delete, then removed.
- The **active segment is never compacted**, so the newest data always shows every version.
- `min.cleanable.dirty.ratio` (0.5) — cleaner runs when 50% of the log is "dirty".
- `min.compaction.lag.ms` — guarantees a message stays uncompacted for at least this long (so consumers can read intermediate values).

**Use cases:** changelog topics for Kafka Streams state stores, `__consumer_offsets`, CDC "latest row state", user profile/config topics — anything where you want a **snapshot of current state** rebuildable by replaying from offset 0.

🧠 **Memory hook:** *Delete = "keep the last 7 days." Compact = "keep the last value per key."*

**Trap:** "Why isn't my compacted topic shrinking?" → key is null (compaction requires keys), everything is still in the active segment, dirty ratio not reached, `min.compaction.lag.ms` too high, or the log cleaner thread is dead (`log.cleaner.enable`, check `LogCleanerManager` metrics).

---

### Q34. Why is Kafka so fast? Name the mechanisms.

1. **Sequential I/O** — append-only log; sequential disk writes rival random RAM access.
2. **Page cache** — Kafka doesn't cache in JVM heap; it lets the OS cache. That's why brokers run with a small heap (6 GB) and huge RAM.
3. **Zero-copy** (`sendfile`) — page cache → socket without copying into user space.
4. **Batching** — on producer, broker and consumer.
5. **Compression per batch** — better ratio than per message; passed through the broker without re-compressing (when producer and broker codecs match).
6. **Binary protocol / no per-message broker state** — offsets are tracked by consumers, so the broker stays dumb and cheap.
7. **Partitioning** — horizontal scale.

**What each means, and why it helps:**
1. *Sequential I/O:* disks are slow at random seeks but fast at streaming in order. Kafka only ever appends to the end of a file and reads forward, so it gets near-disk-bandwidth speed (hundreds of MB/s even on HDDs).
2. *Page cache:* the OS keeps recently written/read file pages in free RAM. A consumer reading recent data is usually served from memory without touching disk, and a broker restart keeps the cache warm (it belongs to the OS, not the JVM). Also avoids long GC pauses from a giant heap.
3. *Zero-copy:* normally disk → kernel buffer → app buffer → socket buffer → NIC (4 copies, 2 context switches). `sendfile` lets the kernel push file pages straight to the network card. Lost if the broker must touch the bytes, e.g. TLS encryption or format conversion/decompression.
4. *Batching:* one network request/disk write carries hundreds of records, amortising per-request overhead.
5. *Compression:* smaller batches = less network and disk; the broker stores the compressed batch as-is.
6. *Dumb broker:* no per-consumer acks or message-by-message tracking; a consumer's position is one number.
7. *Partitioning:* spread load across brokers, disks and consumers.

🧠 **Memory hook:** *"SPZ-BCP" → Sequential, Page cache, Zero-copy, Batch, Compress, Partition.*

---

## 7. Spring Boot + Kafka

### Q35. Show a complete, production-grade configuration.

```yaml
spring:
  kafka:
    bootstrap-servers: b1:9092,b2:9092,b3:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
        linger.ms: 20
        compression.type: lz4
        delivery.timeout.ms: 120000
      batch-size: 65536
    consumer:
      group-id: order-service
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: "com.acme.events"
        partition.assignment.strategy: org.apache.kafka.clients.consumer.CooperativeStickyAssignor
        isolation.level: read_committed
      max-poll-records: 100
    listener:
      ack-mode: manual_immediate
      concurrency: 3
      observation-enabled: true      # micrometer tracing
```

Point out `ErrorHandlingDeserializer` — without it a single malformed payload throws inside `poll()`, before your listener, and **no error handler can catch it**: the container retries the same record forever. That's the classic "poison pill" outage.

**What the key settings do (why they're set this way):**
- `acks: all` + `enable.idempotence` — no loss on leader failure and no duplicates from retries (Q7, Q8).
- `linger.ms: 20`, `batch-size: 65536`, `compression.type: lz4` — wait up to 20 ms to fill 64 KB batches and compress them: far fewer, smaller requests for a few ms of latency (Q6).
- `max.in.flight...: 5` — safe with idempotence; keeps ordering while pipelining requests.
- `enable-auto-commit: false` + `ack-mode: manual_immediate` — *you* decide when a record counts as processed, so a crash causes redelivery instead of loss (Q17).
- `auto-offset-reset: earliest` — a new group starts from the beginning rather than skipping the backlog (Q18).
- `isolation.level: read_committed` — ignore records from aborted transactions (only matters if producers use transactions).
- `max-poll-records: 100` — smaller batches per poll so processing finishes well within `max.poll.interval.ms` (Q16).
- `CooperativeStickyAssignor` — rebalances don't pause the whole group (Q14).
- `spring.json.trusted.packages` — JSON deserialization only instantiates classes from these packages; `"*"` would allow any class (a deserialization-security risk).
- `concurrency: 3` — 3 consumer threads per app instance; must fit within the partition count (Q37).

---

### Q36. `@KafkaListener` — all the variations you should know.

```java
// 1. Simple
@KafkaListener(topics = "orders", groupId = "order-service")
public void basic(OrderEvent event) { }

// 2. Full record + headers + metadata
@KafkaListener(topics = "orders")
public void rich(@Payload OrderEvent event,
                 @Header(KafkaHeaders.RECEIVED_KEY) String key,
                 @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                 @Header(KafkaHeaders.OFFSET) long offset,
                 @Header(name = "traceId", required = false) String traceId,
                 Acknowledgment ack) { }

// 3. Batch listener (set factory.setBatchListener(true))
@KafkaListener(topics = "metrics")
public void batch(List<ConsumerRecord<String, Metric>> records, Acknowledgment ack) {
    metricRepo.saveAll(records.stream().map(ConsumerRecord::value).toList());
    ack.acknowledge();                          // one commit per batch
}

// 4. Explicit partitions + starting offsets
@KafkaListener(topicPartitions = @TopicPartition(topic = "orders",
        partitionOffsets = @PartitionOffset(partition = "0", initialOffset = "0")))
public void fromStart(OrderEvent e) { }

// 5. Pattern subscription
@KafkaListener(topicPattern = "audit\\..*")
public void allAudit(String payload) { }

// 6. Reply (request/response over Kafka)
@KafkaListener(topics = "price-requests")
@SendTo("price-replies")
public PriceResponse quote(PriceRequest req) { return pricer.quote(req); }

// 7. Controllable listener (pause/resume by id)
@KafkaListener(id = "orderListener", topics = "orders", autoStartup = "false")
public void controlled(OrderEvent e) { }
// registry.getListenerContainer("orderListener").pause() / .resume() / .start()
```

**When to use which:**
1. *Simple* — payload only; Spring's `MessageConverter`/deserializer turns the bytes into `OrderEvent`.
2. *Rich* — when you need the key, partition, offset or custom headers (e.g. a correlation id for logs/tracing). `required = false` avoids failing when a header is missing. Injecting `Acknowledgment` requires a manual `AckMode`.
3. *Batch* — receive the whole poll result as a list, ideal for bulk DB inserts; one failure affects the whole batch, so error handling is trickier (use `BatchListenerFailedException` to point at the bad record).
4. *Explicit partitions* — bypasses group management (**no rebalancing**, no automatic assignment); used for replaying from a known offset or pinning a consumer to a partition.
5. *Pattern* — subscribes to all topics matching a regex, and picks up new matching topics later (which triggers a rebalance).
6. *`@SendTo`* — the return value is published to the reply topic; used for request/reply patterns.
7. *Controllable* — `autoStartup=false` plus `KafkaListenerEndpointRegistry` lets you start/pause/resume at runtime (e.g. pause during a downstream outage or maintenance window, S26).

---

### Q37. What does `concurrency` really do, and how does it differ from `partitions`?

`ConcurrentKafkaListenerContainerFactory.setConcurrency(n)` creates **n** `KafkaMessageListenerContainer`s, each with its own `KafkaConsumer` and thread, all in the same group. The group coordinator splits partitions across them.

- `concurrency > partitions` → extra threads idle. Wasted.
- Total consumers across **all instances** must be ≤ partitions. With 3 pods × concurrency 4 = 12 consumers, you need ≥ 12 partitions.

**Plain-language difference:** *partitions* are a property of the **topic** (how the data is split on the brokers); *concurrency* is a property of **your listener** (how many consumer threads you run). Partitions set the ceiling on parallelism; concurrency is how much of it you actually use inside one JVM.

**Example:** topic `orders` has 6 partitions, listener `concurrency=3`, one pod → 3 threads each own 2 partitions. Add a second pod (also 3) → 6 consumers, each owns 1 partition (perfect). Add a third pod → 9 consumers for 6 partitions; 3 sit idle. Setting `concurrency=6` on a topic with 3 partitions likewise wastes 3 threads.

```java
factory.setConcurrency(3);                     // global, or per listener:
@KafkaListener(topics = "orders", concurrency = "3")
```
**Caveats:** each concurrent consumer holds its own connections/memory, and ordering is only per partition — records in different partitions are handled in parallel by different threads. If a single *slow* message handler is the bottleneck, concurrency above the partition count won't help; you'd need more partitions or async hand-off (Q21, S11).

🧠 **Memory hook:** *concurrency = consumers inside one JVM. Partitions = the global ceiling.*

---

### Q38. How do you test Kafka code in Spring Boot?

**1. Embedded Kafka (fast, in-process):**
```java
@SpringBootTest
@EmbeddedKafka(partitions = 1, topics = {"orders"},
               brokerProperties = {"listeners=PLAINTEXT://localhost:9092"})
class OrderListenerTest {

    @Autowired KafkaTemplate<String, OrderEvent> template;
    @Autowired OrderRepository repo;

    @Test
    void shouldPersistOrder() {
        template.send("orders", "o1", new OrderEvent("o1", 100));
        await().atMost(Duration.ofSeconds(10))
               .untilAsserted(() -> assertThat(repo.findById("o1")).isPresent());
    }
}
```

**2. Testcontainers (closest to prod — my preferred answer):**
```java
@Testcontainers
@SpringBootTest
class KafkaIT {
    @Container
    static KafkaContainer kafka =
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
}
```

**3. Unit-test the listener method directly** — it's just a bean; call it with a hand-built `ConsumerRecord`. Fastest, no broker.

**4. `MockProducer` / `MockConsumer`** from `kafka-clients` for pure-unit producer/consumer logic.

Always use **Awaitility**, never `Thread.sleep()` — flaky-test smell that interviewers notice.

**Why Awaitility:** Kafka processing is asynchronous — the listener runs on another thread some milliseconds after `send()`. `Thread.sleep(2000)` is either too short (flaky) or too long (slow suite). `await().atMost(10s).untilAsserted(...)` polls the assertion and returns the instant it passes.

**Choosing between them:** unit-test your business logic without Kafka (3, 4); use EmbeddedKafka for fast wiring checks (serializers, listener config); use Testcontainers when behaviour depends on real broker features (transactions, compaction, security, Schema Registry). Pitfalls: tests sharing a topic/`group.id` see each other's messages — use unique topic or group names per test, and set `auto-offset-reset=earliest` so a consumer started after the message was sent still receives it. `@EmbeddedKafka` can expose its address via `spring.embedded.kafka.brokers` (set `spring.kafka.bootstrap-servers: ${spring.embedded.kafka.brokers}`).

---

### Q39. How do you propagate tracing/correlation IDs through Kafka?

Headers. Set `observation-enabled: true` (Spring Kafka 3.x + Micrometer Tracing) and W3C `traceparent` propagates automatically. Manual version:

```java
ProducerRecord<String, OrderEvent> rec = new ProducerRecord<>("orders", key, event);
rec.headers().add("X-Correlation-Id", MDC.get("correlationId").getBytes(UTF_8));
kafkaTemplate.send(rec);
```

```java
@KafkaListener(topics = "orders")
public void handle(OrderEvent e, @Header("X-Correlation-Id") String cid) {
    MDC.put("correlationId", cid);
    try { service.process(e); } finally { MDC.clear(); }
}
```

Use a `RecordInterceptor` (consumer side) or `ProducerInterceptor` to do this globally instead of per-listener.

**Why it's needed:** with HTTP, tracing libraries pass the trace id in request headers. A Kafka message decouples producer and consumer in time, so unless the id travels *with the message*, the trace breaks at the topic and you can't follow one order across services in your logs.

**How it works:** Kafka records have **headers** (key → byte[] metadata, separate from key/value, since Kafka 0.11). The producer side copies the current trace/correlation id into a header; the consumer reads it and restores it into the logging context (**MDC** — SLF4J's thread-local map, which makes `%X{correlationId}` appear in every log line). With `observation-enabled`, Micrometer does both steps and also creates a span per send/receive, with the W3C `traceparent` header (`00-<traceId>-<spanId>-01`).

**Pitfalls:** always `MDC.clear()` in `finally` (consumer threads are reused, so a stale id leaks into the next record); if you forward to a retry/DLT topic, headers must be copied too (`DeadLetterPublishingRecoverer` and `@RetryableTopic` copy the original headers); header values are bytes, so decide an encoding (UTF-8).

---

## 8. Error Handling, Retries & DLQ

### Q40. Walk through Spring Kafka error handling end-to-end.

Layers, in the order failures occur:

1. **Deserialization failure** → `ErrorHandlingDeserializer` wraps the delegate, converts the failure into a `DeserializationException` in a header so the error handler *can* see it. **Non-negotiable in production.**
2. **Listener exception** → `DefaultErrorHandler` (blocking retries with a `BackOff`), then the **recoverer** (usually `DeadLetterPublishingRecoverer`).
3. **Non-blocking retries** → `@RetryableTopic` / `RetryTopicConfiguration`, which moves failures to `topic-retry-0`, `-1`, ... then `topic-dlt`.
4. **Container-level** → `CommonContainerStoppingErrorHandler` for fatal, unrecoverable conditions.

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {

    var recoverer = new DeadLetterPublishingRecoverer(template,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

    // exponential backoff: 1s, 2s, 4s, 8s ... cap 10s, give up after 30s total
    ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(4);
    backOff.setInitialInterval(1000);
    backOff.setMultiplier(2.0);
    backOff.setMaxInterval(10_000);

    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);

    // never retry these — straight to DLT
    handler.addNotRetryableExceptions(
        ValidationException.class,
        DeserializationException.class,
        NullPointerException.class);

    // always retry these
    handler.addRetryableExceptions(SocketTimeoutException.class);

    handler.setRetryListeners((rec, ex, attempt) ->
        log.warn("Retry {} for {}-{}@{}", attempt, rec.topic(), rec.partition(), rec.offset()));

    return handler;
}
```

**Crucial distinction to state:** `DefaultErrorHandler` does **blocking** retries — the whole partition is stalled while it sleeps. Fine for 3 quick retries; catastrophic for a 5-minute backoff. For long backoffs use **non-blocking** retry topics.

**What happens to a failing record (step by step):**
1. Listener throws → `DefaultErrorHandler` **seeks back** to that record's offset (so the next `poll()` returns it again) and sleeps per the `BackOff`.
2. The record is retried; each failure repeats step 1 until the `BackOff` is exhausted.
3. The **recoverer** runs once: here it publishes the record plus exception details to `<topic>.DLT` (dead-letter topic = parking lot for records that can't be processed).
4. The offset is committed past the bad record and the partition moves on.

**Terms:** *backoff* = delay policy between attempts (fixed/exponential). *Not-retryable exceptions* = deterministic failures (validation error, bad JSON) where retrying can never succeed — they skip the retries and go straight to the DLT. *Retryable* = transient (timeouts, deadlocks, downstream 503).

**Pitfalls:** if retries take longer than `max.poll.interval.ms` the consumer is kicked out and a rebalance starts — keep total backoff well under it (the example's ~30 s is safe versus the 5 min default). The recoverer's own `send` can fail; monitor the DLT publish. And the DLT should have as many partitions as the source topic if you keep `record.partition()` as in the example, otherwise publishing fails.

🧠 **Memory hook:** *Blocking retry = "hold the line." Non-blocking retry topic = "step aside, I'll come back."*

---

### Q41. Non-blocking retries with `@RetryableTopic`.

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 2000, multiplier = 3.0, maxDelay = 60000),
    topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE,
    dltStrategy = DltStrategy.FAIL_ON_ERROR,
    autoCreateTopics = "true",
    exclude = {ValidationException.class})
@KafkaListener(topics = "payments", groupId = "payment-service")
public void handle(PaymentEvent event) {
    gateway.charge(event);          // may throw on transient gateway outage
}

@DltHandler
public void dlt(PaymentEvent event,
                @Header(KafkaHeaders.ORIGINAL_TOPIC) String topic,
                @Header(KafkaHeaders.EXCEPTION_MESSAGE) String msg) {
    alertService.page("Payment permanently failed: " + event.getId() + " / " + msg);
    failedPaymentRepo.save(event);
}
```

Produces: `payments` → `payments-retry-0` → `payments-retry-1` → `payments-retry-2` → `payments-dlt`.

**How it works:** when `handle()` throws, instead of sleeping in place, the framework **publishes the record to the next retry topic** (with a header saying when it becomes due) and immediately moves on to the next record on `payments`. A separate consumer on each retry topic waits (by pausing that partition, not busy-looping) until the record is due, then calls the same method again. After the last attempt the record lands in the DLT and `@DltHandler` is called. Net effect: one failing record doesn't hold up the healthy ones behind it.

**With the example's settings:** attempts=4 (1 original + 3 retries), delay 2 s → 6 s → 18 s (`multiplier=3`), capped by `maxDelay` 60 s. `exclude = ValidationException` skips retries for that type and goes straight to the DLT. `FAIL_ON_ERROR` means if the `@DltHandler` itself throws, the record isn't silently dropped.

**Note:** Spring creates the retry/DLT topics automatically (`autoCreateTopics`) — in production, with topic auto-creation restricted by ACLs, pre-create them. Each retry topic adds a consumer and partitions to manage.

**Trade-off you must mention:** retry topics **break ordering** for the retried key — the failed record is now processed later than records that followed it. If ordering per key is a hard requirement, you need blocking retries or a per-key "parking" mechanism.

---

### Q42. What headers does `DeadLetterPublishingRecoverer` add?

`KafkaHeaders.DLT_ORIGINAL_TOPIC`, `DLT_ORIGINAL_PARTITION`, `DLT_ORIGINAL_OFFSET`, `DLT_ORIGINAL_TIMESTAMP`, `DLT_EXCEPTION_FQCN`, `DLT_EXCEPTION_MESSAGE`, `DLT_EXCEPTION_STACKTRACE`, `DLT_KEY_EXCEPTION_*`.

That's what makes a DLQ actually *operable* — you can see what failed, why, and where it came from.

**Why headers (not the payload):** the recoverer republishes the *original, unmodified* record to the DLT so it can be replayed as-is; diagnostics go into headers so the payload stays untouched. Example: to find why record `payments-2@5531` failed, read the DLT record's `DLT_EXCEPTION_MESSAGE` (human-readable reason) and `DLT_EXCEPTION_STACKTRACE` (full trace); `DLT_ORIGINAL_TOPIC/PARTITION/OFFSET` tells you the exact source position, and `DLT_ORIGINAL_TIMESTAMP` when it was first produced. Header names are `kafka_dlt-original-topic` etc. on the wire.

**DLQ replay strategy:** a separate listener on the `.DLT` topic with `autoStartup=false`, started on demand via `KafkaListenerEndpointRegistry`, that republishes to the original topic after the bug is fixed. Always cap replay and de-dupe.

---

### Q43. How do you handle a poison pill?

Layered defence:
1. `ErrorHandlingDeserializer` — malformed bytes never blow up `poll()`.
2. `DefaultErrorHandler` with `addNotRetryableExceptions(DeserializationException.class)` → straight to DLT, no retry loop.
3. Schema Registry + backwards-compatible schemas so poison pills are rare by construction.
4. Emergency lever: `kafka-consumer-groups.sh --reset-offsets --shift-by 1` to step over one stuck record (group must be stopped).

🧠 **Memory hook:** *Poison pill = infinite retry loop = lag alarm. The cure is ErrorHandlingDeserializer + non-retryable + DLT.*

**What a poison pill is:** a record that can *never* be processed successfully — corrupt bytes, wrong schema, or data that triggers a bug. Because a partition is consumed in order, the consumer can't skip it, so it retries forever; everything behind it in that partition is stuck and lag climbs while the app looks "healthy" (logs repeating the same error and offset).

**Two flavours:** (a) *deserialization* failure — happens inside `poll()` before your code runs, so a plain `try/catch` in the listener never sees it (hence `ErrorHandlingDeserializer`); (b) *processing* failure — your code throws on this specific data (NPE, constraint violation), handled by the error handler + DLT.

**Emergency example:** logs show `Error deserializing key/value for partition orders-3 at offset 4812` repeating. Stop the group, then
`kafka-consumer-groups.sh --bootstrap-server b:9092 --group order-service --topic orders:3 --reset-offsets --to-offset 4813 --execute`
(or `--shift-by 1`), restart, and afterwards inspect offset 4812 with `kafka-console-consumer.sh --partition 3 --offset 4812 --max-messages 1`. Fix the root cause so it doesn't return (S5).

---

## 9. Schema Registry & Serialization

### Q44. Why Avro/Protobuf over JSON?

- **Compact binary** — 3–10× smaller than JSON, big saving on network/disk.
- **Schema enforced at write time** — producers can't publish garbage.
- **Schema evolution with compatibility checks** — the registry rejects a breaking change *at deploy time* instead of at 3 a.m.
- Payload carries only a **5-byte header** (magic byte + 4-byte schema ID); the schema itself lives in the registry.

JSON is fine for low-volume internal topics and much easier to debug. Be pragmatic — say "JSON with a versioned envelope for simple internal events, Avro/Protobuf where volume or contract stability matters."

**Terms:** *Serialization* = turning an object into bytes for the topic; *schema* = the formal definition of a message's fields/types; *Schema Registry* = a separate service that stores schemas and assigns each a numeric ID; *Avro/Protobuf* = compact binary formats driven by a schema.

**How the pieces fit:** the producer's serializer registers/looks up the schema in the registry, gets e.g. ID 42, and writes `[0x0][00 00 00 2A][binary data]`. The consumer reads the ID, fetches schema 42 (cached after the first time), and decodes. Field *names* aren't repeated in every message as they are in JSON, which is where most of the size saving comes from.

```json
JSON : {"orderId":"o-1","amount":100.5,"currency":"INR"}          // ~55 bytes, every message
Avro : [magic][schemaId][o-1][100.5][INR] as binary                // ~15 bytes, names live in the registry
```
**Trade-offs:** needs the registry as a new dependency (HA, caching, availability on cold start); binary data can't be read with plain `kafka-console-consumer` (use `kafka-avro-console-consumer`); stricter process for schema changes — which is the point.

---

### Q45. Explain the compatibility modes.

| Mode | You can... | Consumers/Producers upgrade order |
|---|---|---|
| `BACKWARD` (default) | Delete fields, add **optional** fields (with defaults) | **Consumers first** |
| `FORWARD` | Add fields, delete optional fields | **Producers first** |
| `FULL` | Both | Any order |
| `NONE` | Anything | Good luck |
| `*_TRANSITIVE` | Checked against **all** past versions, not just the latest | Safest |

**Rules that keep you safe:** always give new fields a **default**; never rename (that's delete + add); never change a type; never remove a required field.

**Why direction matters — definitions:** *"Backward"* is judged from the **consumer's** view: a consumer using the **new** schema must still read **old** messages already in the topic. *"Forward"*: a consumer on the **old** schema must still read **new** messages (so old consumers keep working while producers upgrade).

**Example (Avro, `Order` v1 = `{id, amount}`):**
- v2 adds `currency` **with default "INR"** → BACKWARD-compatible: new consumer reading v1 data (no currency) fills in the default. Also FORWARD-compatible: an old consumer simply ignores the extra field → so it's FULL.
- v2 adds `currency` **without** a default → rejected under BACKWARD: the new reader has no value to use for old records.
- v2 *removes* `amount` (which had no default) → BACKWARD OK (reader ignores absent writer data... new schema simply doesn't read it) but not FORWARD: old consumers still expect `amount`.
- v2 renames `amount` → `price` → effectively delete + add-without-default → breaks.

**Practical:** the registry enforces the chosen mode when you register a new version (HTTP 409 if incompatible) — wire the check into CI (`mvn schema-registry:test-compatibility`) so it fails the build, not production (S15). *Transitive* matters when old messages are retained for a long time: with plain BACKWARD only the latest version is compared.

🧠 **Memory hook:** *BACKWARD = new schema can read OLD data → upgrade CONSUMERS first. FORWARD = old schema can read NEW data → upgrade PRODUCERS first.*

---

### Q46. Avro in Spring Boot?

```yaml
spring:
  kafka:
    producer:
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
      properties:
        schema.registry.url: http://schema-registry:8081
        auto.register.schemas: false     # register via CI, not at runtime
    consumer:
      value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
      properties:
        schema.registry.url: http://schema-registry:8081
        specific.avro.reader: true       # get generated classes, not GenericRecord
```
Generate classes with `avro-maven-plugin` from `.avsc` files kept in a shared schema repo. Set `auto.register.schemas=false` in prod so a rogue deploy can't push an incompatible schema.

**What each setting does:**
- `KafkaAvroSerializer` / `KafkaAvroDeserializer` (Confluent) — talk to the Schema Registry: on send, look up the schema's ID and prefix it to the bytes; on receive, fetch the schema by ID and decode (Q44).
- `schema.registry.url` — where the registry lives (must be set on both sides).
- `auto.register.schemas` — if `true`, the producer registers its schema on first send, convenient in dev but means any code change can alter the contract. With `false`, the schema must already exist (registered by CI after its compatibility check) or the send fails.
- `specific.avro.reader: true` — deserialize into the **generated** class (`OrderEvent`) instead of a generic map-like `GenericRecord` — type-safe.

**Example `.avsc`:**
```json
{"type":"record","name":"OrderEvent","namespace":"com.acme.events",
 "fields":[{"name":"orderId","type":"string"},
           {"name":"amount","type":"double"},
           {"name":"currency","type":"string","default":"INR"}]}
```
**Pitfall:** the registry becomes a runtime dependency — clients cache schemas, so it's only hit on first use of each ID, but a cold-starting pod with the registry down can't decode. Also pair the Avro deserializer with `ErrorHandlingDeserializer` so a bad record doesn't become a poison pill (Q43).

---

## 10. Kafka Streams & Kafka Connect

### Q47. Kafka Streams vs a plain consumer — when do you reach for Streams?

Use **Streams** when you need stateful operations: joins, windowed aggregations, deduplication over time, event-time processing with out-of-order handling. It gives you local state stores (RocksDB) backed by **compacted changelog topics**, so state survives restarts and rebalances.

Use a **plain consumer** for stateless "read → call service → write DB" work. Simpler, easier to reason about, fits Spring's programming model.

Core concepts to name: `KStream` (record stream / inserts), `KTable` (changelog / upserts, latest per key), `GlobalKTable` (fully replicated to every instance, no co-partitioning needed), state store, changelog topic, repartition topic, **co-partitioning requirement** for KStream-KTable joins (same key, same partition count).

**Plain-language explanation:**
- *Kafka Streams* is a **library** (not a cluster): you embed it in your Spring Boot app; running more instances of the app scales it out, and partitions are split across instances just like a consumer group.
- *Why stateful is hard with a plain consumer:* "sum of amounts per customer in the last 5 minutes" needs memory of earlier records. With a plain consumer you'd hand-roll storage, recovery after crash, and handling of late events. Streams gives you a local **state store** (embedded RocksDB key-value DB) plus a **changelog topic** (every state change is also written to a compacted Kafka topic), so a replacement instance rebuilds the state by replaying it.
- *KStream vs KTable:* stream = every event matters ("a payment happened"); table = latest value per key ("current balance"). Stream ⇄ table duality: a table is a stream aggregated by key.
- *Co-partitioning:* a join needs the matching keys on the same instance, so both topics need the same key and the same partition count — otherwise a `repartition topic` (re-shuffle by key) is needed, or you use a `GlobalKTable` for small reference data.
- *Windows:* group events by time buckets (here 5 min); `ofSizeWithNoGrace` = late events after the window closes are dropped.

**Rule of thumb for the answer:** "plain consumer for per-record work with side effects; Streams for transform/aggregate/join Kafka→Kafka." Streams isn't ideal for calling slow external services per record.

```java
@Configuration
@EnableKafkaStreams
public class TopologyConfig {

    @Bean
    public KStream<String, OrderEvent> topology(StreamsBuilder builder) {
        KStream<String, OrderEvent> orders = builder.stream("orders");

        orders.filter((k, v) -> v.getAmount() > 1000)
              .groupByKey()
              .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
              .aggregate(() -> 0.0,
                         (k, v, agg) -> agg + v.getAmount(),
                         Materialized.as("high-value-5min"))
              .toStream()
              .to("high-value-totals");
        return orders;
    }
}
```

Exactly-once in Streams is one line: `processing.guarantee=exactly_once_v2`.

---

### Q48. What is Kafka Connect and when do you use it instead of writing code?

A distributed framework for moving data in/out of Kafka via config, not code. **Source** connectors (DB → Kafka) and **Sink** connectors (Kafka → Elasticsearch/S3/JDBC). It handles offsets, retries, scaling, and schema conversion for you.

**Best senior talking point — Debezium CDC:** instead of dual-writing to DB and Kafka, Debezium tails the DB write-ahead log and publishes row changes. That gives you the outbox/CDC pattern with zero application code and no dual-write inconsistency.

Single Message Transforms (SMTs) handle light per-record work (mask a field, route by field, add a header). Anything heavier belongs in Streams.

**Why use it instead of code:** moving data between a DB/S3/Elasticsearch and Kafka is always the same problem — track progress, retry, parallelise, convert formats, restart safely. Connect solves it once; you write JSON config instead of a custom consumer app and its operational headaches.

**Terms:** a **connector** is the configured job; its **tasks** do the parallel work (e.g. one per table/partition) and run on a cluster of **workers** (JVM processes). Progress is stored in Kafka itself (internal topics for offsets/config/status), so a failed worker's tasks move to another one. A **converter** (JSON/Avro/Protobuf) sets how bytes are encoded.

**Example — sink Kafka `orders` into Postgres (submitted via the REST API at `:8083/connectors`):**
```json
{ "name": "orders-sink",
  "config": { "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
              "topics": "orders", "tasks.max": "3",
              "connection.url": "jdbc:postgresql://db:5432/shop",
              "insert.mode": "upsert", "pk.mode": "record_key", "pk.fields": "orderId",
              "errors.tolerance": "all", "errors.deadletterqueue.topic.name": "orders-sink-dlq" } }
```
`upsert` keeps the sink idempotent; the DLQ settings park bad records instead of failing the connector.

**Debezium in one line:** it reads the database's transaction log (MySQL binlog / Postgres WAL) so it captures every committed change in order, with no polling and no impact on application code. **When *not* to use Connect:** when you need real business logic per record — use a consumer or Streams.

---

## 11. Performance Tuning

### Q49. Producer throughput is too low. What do you change, in order?

1. `linger.ms: 10–50` — the biggest single win; lets batches actually fill.
2. `batch.size: 64KB–256KB` — more records per request.
3. `compression.type: lz4` (throughput) or `zstd` (best ratio) — fewer bytes on the wire, better batching.
4. `buffer.memory: 64–128MB` if you see `send()` blocking.
5. `acks=1` **only** if the data tolerates loss.
6. Stop calling `.get()` on every send (sync sends kill batching).
7. Multiple producer instances / more partitions if a single partition leader is the bottleneck.

Measure `record-send-rate`, `batch-size-avg`, `record-queue-time-avg`, `request-latency-avg`.

**Why this order:** the producer's cost is mostly per *request*, not per byte. Records are grouped in batches (per partition) and shipped as requests; if batches go out nearly empty (default `linger.ms=0` sends immediately) you pay full overhead for little data. So first make batches fuller (1–2), then make them smaller on the wire (3), then remove buffering/blocking limits (4, 6). Trading durability (5) is last and rarely acceptable.

**Example config for a throughput-oriented producer:**
```yaml
spring.kafka.producer:
  batch-size: 131072          # 128 KB
  compression-type: lz4
  properties: { linger.ms: 20 }
```
**How to tell it worked:** `batch-size-avg` rises toward `batch.size`, `records-per-request-avg` goes up, and request rate drops for the same record rate. If `record-queue-time-avg` ≈ `linger.ms` constantly, the batches fill on time rather than size — the cost is that latency up to `linger.ms` is added to every send; fine for pipelines, not for latency-critical paths.

🧠 **Memory hook:** *"Linger, Batch, Compress" — in that order, always.*

---

### Q50. Consumer is too slow. What do you change?

1. **Add partitions + consumers** (the real fix if the work is parallelizable).
2. `max.poll.records` — *lower* it if processing per record is slow (avoid `max.poll.interval` breaches); *raise* it for cheap per-record work + batch DB writes.
3. **Batch listener** + `saveAll()` — turning 500 single-row inserts into one batch insert is often a 10–50× win.
4. `fetch.min.bytes` up (e.g. 64 KB) to reduce round trips.
5. Move blocking I/O off the consumer thread with a bounded worker pool + `pause()`/`resume()`.
6. Profile first — 90% of "Kafka is slow" turns out to be a slow DB query or an unindexed lookup in the listener.

**First, decide where the time goes:** consumption rate = records per poll ÷ time to process them. If `poll()` returns quickly and processing dominates, the bottleneck is *your code* (DB/API calls), not Kafka; if the consumer is mostly waiting on fetches, tune the fetch settings. Check lag per partition (Q20): lag on all partitions → add capacity; lag on one → hot key/stuck consumer.

**Example (batch writes):**
```java
@KafkaListener(topics = "events", containerFactory = "batchFactory")
public void onBatch(List<EventDto> batch) {
    jdbcTemplate.batchUpdate("INSERT INTO events(id, payload) VALUES (?, ?)", /* one round trip */ ...);
}   // 500 round trips → 1; the container commits after the method returns
```
**Reasoning for step 5:** if each message needs a 2-second API call (S11), one thread handles only 0.5 msg/s. Hand the calls to a bounded thread pool, `pause()` the consumer when the pool is full so `poll()` keeps being called (avoiding `max.poll.interval.ms`), and commit only offsets of finished work — ordering/at-least-once need care (Q21).

---

### Q51. Broker-side tuning highlights.

- JVM heap **6 GB**, rest of RAM for page cache. Use G1GC.
- `num.network.threads` (3), `num.io.threads` (8, ≈ number of disks × 2).
- `log.dirs` across multiple disks (JBOD), not RAID-5.
- `num.replica.fetchers` up if replication lags.
- OS: raise file descriptor limit (100k+), `vm.swappiness=1`, use XFS/ext4, `noatime`.
- Rack awareness: `broker.rack` so replicas spread across AZs.

**Why each one:**
- *Small heap + big page cache:* Kafka's speed comes from the OS page cache (Q34); a large heap steals RAM from it and causes long GC pauses. G1GC keeps pauses short so followers don't fall out of the ISR.
- *Network vs I/O threads:* network threads read/write sockets; I/O threads do the disk work. If `RequestHandlerAvgIdlePercent` drops below ~30% the I/O threads are saturated — raise `num.io.threads` or add brokers.
- *JBOD, not RAID-5:* Kafka already replicates across brokers; RAID-5's parity writes slow sequential appends and add redundancy you don't need. Spread `log.dirs` across disks for throughput (a disk failure takes down only its partitions on modern versions).
- *`num.replica.fetchers`:* threads followers use to copy from leaders; more threads help when replication lags and ISR shrinks.
- *OS:* every segment and socket is a file descriptor (low defaults → "Too many open files"); `vm.swappiness=1` stops the OS swapping out the page cache/heap; `noatime` avoids a metadata write on each read.
- *Rack awareness:* replicas of a partition are placed in different racks/AZs, so a whole-AZ outage doesn't remove all copies.

---

## 12. Security & Operations

### Q52. How do you secure a Kafka cluster?

Three independent layers:

1. **Encryption in transit** — TLS (`SSL` or `SASL_SSL` listeners). Note this **disables zero-copy**, costing throughput.
2. **Authentication** — SASL mechanisms: `PLAIN` (only over TLS), `SCRAM-SHA-512` (recommended without Kerberos), `GSSAPI/Kerberos` (enterprise), `OAUTHBEARER` (modern/cloud), or mTLS client certs.
3. **Authorization** — ACLs on Resource (Topic/Group/Cluster/TransactionalId) × Operation (Read/Write/Create/Describe/Alter) × Principal × Host. Or RBAC in Confluent.

```yaml
spring:
  kafka:
    security:
      protocol: SASL_SSL
    properties:
      sasl.mechanism: SCRAM-SHA-512
      sasl.jaas.config: >
        org.apache.kafka.common.security.scram.ScramLoginModule required
        username="${KAFKA_USER}" password="${KAFKA_PASSWORD}";
      ssl.truststore.location: /etc/kafka/truststore.jks
      ssl.truststore.password: ${TRUSTSTORE_PASSWORD}
```

Plus: encryption at rest at the volume level, and **field-level encryption in the payload** for PII (because retention means the data sits there for days — mention GDPR/right-to-be-forgotten, which is why PII topics are often compacted so a tombstone can actually erase a key).

**The three layers in plain words (and the question each answers):**
- *Encryption (TLS):* "can someone sniff the traffic?" Clients verify the broker's certificate using a **truststore**; with mTLS the broker also verifies the client's certificate (from a **keystore**).
- *Authentication (SASL):* "who are you?" Client proves identity with a username/password (SCRAM stores salted hashes, never plain passwords), a Kerberos ticket, an OAuth token, or a certificate. The authenticated name is the **principal**.
- *Authorization (ACLs):* "what may you do?" Example granting an app read access:
```bash
kafka-acls.sh --bootstrap-server b:9092 --add --allow-principal User:order-service \
  --operation Read --topic orders --group order-service
```
The consumer needs Read on the **topic and the group**; a transactional producer additionally needs Write on its `TransactionalId`.

**Pitfalls:** `PLAIN` without TLS sends passwords in clear text; keep secrets in env vars/a vault (as `${KAFKA_USER}` above), not in git; ACL denials surface as `TopicAuthorizationException`/`GroupAuthorizationException`; and remember TLS costs zero-copy (Q2).

---

### Q53. Essential CLI commands.

```bash
# Topics
kafka-topics.sh --bootstrap-server b:9092 --create --topic orders \
  --partitions 6 --replication-factor 3 --config min.insync.replicas=2
kafka-topics.sh --bootstrap-server b:9092 --describe --topic orders
kafka-topics.sh --bootstrap-server b:9092 --alter --topic orders --partitions 12

# Configs
kafka-configs.sh --bootstrap-server b:9092 --entity-type topics --entity-name orders \
  --alter --add-config retention.ms=259200000

# Consumer groups
kafka-consumer-groups.sh --bootstrap-server b:9092 --describe --group order-service
kafka-consumer-groups.sh --bootstrap-server b:9092 --group order-service \
  --topic orders --reset-offsets --to-earliest --execute      # group must be inactive

# Produce / consume
kafka-console-producer.sh   --bootstrap-server b:9092 --topic orders \
  --property parse.key=true --property key.separator=:
kafka-console-consumer.sh   --bootstrap-server b:9092 --topic orders \
  --from-beginning --property print.key=true

# Perf test
kafka-producer-perf-test.sh --topic orders --num-records 1000000 \
  --record-size 1024 --throughput -1 --producer-props bootstrap.servers=b:9092

# Reassign partitions / check under-replicated
kafka-reassign-partitions.sh --bootstrap-server b:9092 --reassignment-json-file plan.json --execute
kafka-topics.sh --bootstrap-server b:9092 --describe --under-replicated-partitions
```

**What the commands tell you / when to use them:**
- `--describe --topic orders` — shows each partition's **Leader**, **Replicas** and **Isr**. If `Isr` has fewer entries than `Replicas`, a follower is lagging (Q27). Typical first step when debugging.
- `--alter --partitions 12` — can only *increase* partition count and re-maps keys (Q10, S13).
- `kafka-configs.sh --alter --add-config` — changes topic configs dynamically with no restart (here: 3-day retention).
- `kafka-consumer-groups.sh --describe` — per-partition `CURRENT-OFFSET`, `LOG-END-OFFSET`, `LAG` (Q20). `--reset-offsets` needs one of `--to-earliest/--to-latest/--to-offset/--to-datetime/--shift-by` and `--execute` (without `--execute` it only *previews*, i.e. a dry run).
- Console producer/consumer — quick smoke tests; `parse.key`/`print.key` let you see keys, `--from-beginning` replays from offset 0.
- `kafka-producer-perf-test.sh` — benchmark a topic/config; `--throughput -1` means "as fast as possible".
- `kafka-reassign-partitions.sh` — moves replicas between brokers using a JSON plan (e.g. after adding brokers); throttle it so the copy doesn't starve live traffic. `--under-replicated-partitions` lists unhealthy partitions after an incident.

---

### Q54. What do you monitor?

**Broker:** `UnderReplicatedPartitions` (should be 0), `OfflinePartitionsCount` (0), `ActiveControllerCount` (exactly 1 cluster-wide), `RequestHandlerAvgIdlePercent` (>30%), `IsrShrinksPerSec`, disk usage, `LeaderElectionRateAndTimeMs`.

**Producer:** `record-error-rate`, `record-retry-rate`, `request-latency-avg`, `buffer-available-bytes`, `record-send-rate`.

**Consumer:** `records-lag-max` ← the headline metric, `commit-latency-avg`, `fetch-rate`, `records-consumed-rate`, `rebalance-rate-per-hour`.

**App:** DLQ topic message count (should be ~0 — alert on any increase), processing time p99.

**What each signal tells you:**
- `UnderReplicatedPartitions` > 0 → some follower is behind (slow/dead broker, network). It's the first health signal of the cluster. `OfflinePartitionsCount` > 0 → partitions with *no* leader = outage for those partitions. `ActiveControllerCount` must be exactly 1 (0 = no controller; 2 = split brain).
- `RequestHandlerAvgIdlePercent` low → broker I/O threads saturated (Q51). `IsrShrinksPerSec` flapping → unstable follower.
- Producer `record-error-rate` / `record-retry-rate` rising → broker or network trouble; `buffer-available-bytes` near 0 → `send()` is about to block.
- Consumer `records-lag-max` rising → falling behind (Q20); `rebalance-rate-per-hour` high → instability (Q14, S4); `commit-latency-avg` high → coordinator trouble.

**How to collect them:** brokers/clients expose **JMX** MBeans → Prometheus JMX/Kafka exporter → Grafana dashboards with alerts; in Spring Boot, Micrometer exposes the client metrics on `/actuator/prometheus`. Alert on **trends and symptoms** (lag growing for 10 min, any offline partition) rather than every blip.

---

## 13. Design Patterns for Microservices

### Q55. Event Notification vs Event-Carried State Transfer vs Event Sourcing.

| Pattern | Payload | Pro | Con |
|---|---|---|---|
| **Event notification** | Just an ID: `{orderId: 123, type: ORDER_CREATED}` | Tiny, low coupling on schema | Consumer must call back → chatty, temporal coupling |
| **Event-carried state transfer** | Full snapshot of the entity | Consumers are autonomous, no callbacks | Big payloads, schema coupling, stale copies |
| **Event sourcing** | The event log **is** the source of truth; state = fold(events) | Full audit, time travel, rebuild any projection | Complex, needs snapshots, schema evolution is hard forever |

Real-world answer: "I default to event-carried state transfer with a thin, versioned envelope — enough state that consumers don't call back, but not the whole aggregate."

**Examples with an `Order` service and a `Shipping` service:**
- *Notification:* `{ "type":"OrderCreated", "orderId":123 }` → Shipping must call `GET /orders/123` to learn the address. Small event, but Shipping now depends on Order's API being up (temporal coupling) and may get traffic spikes on it.
- *Event-carried state transfer:* `{ "type":"OrderCreated", "orderId":123, "address":{...}, "items":[...] }` → Shipping keeps its own local copy and works even if Order is down. Cost: bigger messages and every consumer depends on the event's shape.
- *Event sourcing:* instead of storing "order = SHIPPED" in a table, store the events `Created, Paid, Shipped` and compute the current state by replaying them (a "fold"). Kafka (or an event store) is the primary database. To avoid replaying millions of events, periodically save a **snapshot**. Not to be confused with merely *using* Kafka for events — most systems don't need full event sourcing.

**Choosing:** notification for "something happened, go look"; ECST for read-heavy consumers that need data locally; event sourcing only where the history itself is the business value (ledgers, audit).

---

### Q56. Explain the Saga pattern with Kafka.

Distributed transaction across services without 2PC — a sequence of local transactions, each publishing an event that triggers the next, with **compensating** transactions on failure.

```
OrderCreated → [Payment] PaymentCompleted → [Inventory] StockReserved → [Shipping] Shipped
                              ↓ fail
                        PaymentFailed → [Order] OrderCancelled  (compensation)
```

- **Choreography** — each service listens and reacts. Decoupled, but the overall flow is implicit and hard to trace.
- **Orchestration** — a central saga orchestrator issues commands and tracks state. Explicit and debuggable; the orchestrator is a new dependency.

Say: "I'd use orchestration for anything with more than 3–4 steps, because choreographed sagas become impossible to reason about."

**Why it exists:** each microservice owns its own database, so a single ACID transaction across Order, Payment and Inventory isn't possible (2PC = distributed locks and tight coupling). A saga replaces one big transaction with a chain of small local ones; if step N fails you can't roll back the earlier commits, so you run **compensating** actions that semantically undo them (refund the payment, release the stock).

**Walk-through (choreography):**
1. Order service saves the order as `PENDING` and publishes `OrderCreated` (via the outbox, S7).
2. Payment consumes it, charges, publishes `PaymentCompleted` (or `PaymentFailed`).
3. Inventory consumes `PaymentCompleted`, reserves stock, publishes `StockReserved`/`StockFailed`.
4. On `PaymentFailed`/`StockFailed`, upstream services react: Order marks `CANCELLED`; if stock failed after payment, Payment issues a refund.

**Orchestration variant:** an `OrderSagaOrchestrator` sends commands (`ChargePayment`, `ReserveStock`) to command topics and listens to replies, keeping state (`PAYMENT_DONE`, …) in its own table — one place to see where each order is stuck.

**Requirements/pitfalls:** every step must be **idempotent** (events are redelivered), every step needs a defined compensation, use a correlation/saga id on all messages, and handle timeouts ("no reply in 5 min → compensate"). Sagas give *eventual* consistency, not isolation — other users can see intermediate states (`PENDING`).

---

### Q57. CQRS with Kafka.

Write model publishes events → Kafka → read-model services build denormalized projections (Elasticsearch, Redis, a read replica). Benefits: independent scaling of reads, purpose-built read stores. Cost: **eventual consistency** — you must handle "user saves, immediately reads, doesn't see it." Mitigations: return the write-model result optimistically, or version/ETag the read and poll.

**CQRS = Command Query Responsibility Segregation:** use separate models for *changing* data (commands, normalized, enforce business rules) and *reading* data (queries, shaped exactly like the screen needs). The problem it solves: one relational model rarely serves both well — complex joins for list screens slow the write database and lock up under load.

**Flow example:** `POST /orders` → Order service writes to its DB and publishes `OrderCreated` → a projection consumer upserts a flat document `{orderId, customerName, total, status}` into Elasticsearch → `GET /orders?customer=…` reads that document with no joins. Rebuilding a projection = reset its consumer group to offset 0 (or a new group) and replay — Kafka's retention/compaction make this possible.

**Pitfalls:** projections must be idempotent and handle out-of-order/duplicate events (version number check, Q26); lag means stale reads — expose it as a metric; don't use CQRS for simple CRUD apps, the extra moving parts aren't worth it.

---

### Q58. How do you version events?

1. **Envelope with explicit version:**
   ```json
   { "eventType": "OrderCreated", "version": 2, "occurredAt": "...",
     "eventId": "uuid", "payload": { ... } }
   ```
2. **Additive-only changes** with defaults; never remove or retype a field (Schema Registry BACKWARD).
3. **New topic for breaking changes** — `orders.v2`, dual-publish during migration, retire v1 once all consumers move. Clean, boring, works.

`eventId` and `occurredAt` in every event, always — they power idempotency and out-of-order detection.

**Why versioning is needed:** events are a *contract* with consumers you don't control, and old events stay in the topic for days or forever. If the shape changes carelessly, either old consumers break on new events or new consumers break while replaying old ones.

**Example evolution:** `OrderCreated` v1 has `amount`. You need currency → v2 adds `currency` with default `"INR"`; old consumers ignore it, new consumers use it (compatible, same topic, `version` bumped). Later you need to split `amount` into `net`/`tax` — a *breaking* change → publish to `orders.v2` as well as `orders.v1` (dual-publish), migrate consumers one at a time, then retire v1.

**What the envelope fields are for:** `eventType`+`version` let a consumer pick the right parsing code (an "upcaster" converts v1 → latest in one place); `eventId` is the idempotency key; `occurredAt` lets you detect/handle late or out-of-order events (don't use the Kafka record timestamp for business time).

---

### Q59. Topic naming and design conventions.

`<domain>.<entity>.<event-type>.<version>` → `sales.order.created.v1`. Or `<env>.<team>.<entity>` — the point is consistency, documented, and enforced in CI.

**One topic per event type vs one per aggregate?** Per aggregate (`orders` with a `type` header) preserves **ordering across event types for the same entity** — usually what you want. Per event type gives finer-grained ACLs and independent retention. State the trade-off; it's a design-sense question.

**Why bother with naming:** topics are shared, long-lived and hard to rename (you'd have to create a new one and migrate). A predictable name tells everyone who owns it, what it holds, and its contract version, and lets ACLs/prefix rules work (`sales.*`).

**Example of the ordering argument:** `OrderCreated`, `OrderPaid`, `OrderCancelled` on three separate topics have no relative ordering, so a consumer could see "cancelled" before "created". In a single `sales.order.v1` topic keyed by `orderId`, they arrive in order.

**Other conventions worth naming:** lowercase with dots/hyphens only (avoid mixing `.` and `_` — Kafka metric names collapse them and can collide); separate topics for DLT/retry (`.DLT`, `-retry-0`); explicit partition count, RF and retention per topic created via IaC (not auto-create, `auto.create.topics.enable=false`); key = the entity id you need ordering for; document each topic's owner, schema, retention and consumers.

---

## 14. Scenario-Based Questions

### S1. Consumer lag is growing steadily in production. Walk me through your diagnosis.

**Answer structure — say you go top-down:**

1. **Confirm the shape.** `kafka-consumer-groups.sh --describe`: is lag on **all** partitions or **one**? One partition → hot key / skew. All → consumer capacity or a downstream bottleneck.
2. **Check the producer side.** Did input rate genuinely spike (marketing campaign, batch upstream job, retry storm)? Compare `records-consumed-rate` with the produce rate.
3. **Check consumer health.** Is any consumer stuck? Look at `rebalance-rate-per-hour` — repeated rebalances mean nobody makes progress. Thread dump for a blocked thread.
4. **Check downstream.** 95% of the time it's the DB or a REST call in the listener: p99 latency, connection pool exhaustion, missing index, an external API rate-limiting us.
5. **Check for a poison pill** — same offset repeatedly in error logs = infinite blocking retry.

**Immediate mitigation:** scale consumer pods up to the partition count; raise `max.poll.records` + batch the DB writes; temporarily raise `max.poll.interval.ms` so it stops rebalancing.
**Structural fix:** add partitions (with a key-remap plan), batch inserts, offload slow I/O to a worker pool with `pause()`/`resume()`, or split the slow step into its own topic + service.

🧠 **Say the order out loud:** *Skew → Input rate → Consumer health → Downstream → Poison pill.*

---

### S2. Downstream reports duplicate order confirmations. How do you fix it?

**Answer:** "Duplicates are expected — at-least-once is the default. The fix belongs on the consumer, not the broker."

1. **Find the source.** Check for repeated rebalances (records processed, commit fails, redelivered), a producer retrying without idempotence, or a listener throwing *after* the side effect but *before* commit.
2. **Immediate hardening:** `enable.idempotence=true` on the producer; commit offsets **after** processing (`MANUAL_IMMEDIATE`).
3. **Real fix — idempotent consumer:** processed-events table keyed by `eventId`, written **inside the same DB transaction** as the business change (see Q26).
4. **For the email itself:** a unique constraint on `(orderId, notificationType)` in the notifications table means the second attempt is a no-op.

**Key sentence:** *"I don't try to guarantee once-delivery; I guarantee once-effect."*

---

### S3. You must guarantee that all events for a given customer are processed in order.

**Answer:**
- **Key by customer ID.** Same key → same partition → single consumer → ordered. This is the whole answer to 90% of ordering questions.
- **Consequences to volunteer:**
  - You cannot use a multi-threaded worker pool without a per-key ordering guard.
  - **Retry topics break ordering** — for this topic use blocking retries (`DefaultErrorHandler`) instead of `@RetryableTopic`, or park the whole key.
  - Adding partitions later re-maps keys and breaks ordering across the change — plan a drain-then-migrate, or use a custom partitioner with a stable mapping (e.g. consistent hashing / lookup table).
  - A hot customer creates a hot partition (see S9).
- **If you need in-order *and* parallel:** hash the key to a worker within the consumer (`workerIndex = hash(key) % poolSize`), keeping per-key order while parallelizing across keys. Track completion to commit safely.

---

### S4. The consumer group rebalances every few minutes and processing stalls.

**Answer — diagnose then fix:**

Causes, in likelihood order:
1. Processing a batch takes longer than `max.poll.interval.ms` → consumer is evicted mid-batch. **Signature: `CommitFailedException` in logs.**
2. Long GC pause or CPU starvation → heartbeats missed → `session.timeout.ms` exceeded.
3. Pods restarting (OOMKilled, failing liveness probes, K8s rescheduling).
4. Network blips to the coordinator.

Fixes:
```yaml
max.poll.records: 50            # smaller batches finish inside the interval
max.poll.interval.ms: 600000    # give slow processing room
session.timeout.ms: 45000
heartbeat.interval.ms: 3000
partition.assignment.strategy: ...CooperativeStickyAssignor   # incremental
group.instance.id: ${HOSTNAME}  # static membership: restarts don't rebalance
```
Plus: fix the GC/memory limits, make the listener faster, and ensure graceful shutdown so leaves are clean.

🧠 **Two-line summary:** *`CommitFailedException` → too slow, shrink the batch. Heartbeat timeout → the JVM is starved.*

---

### S5. One malformed message is blocking the whole partition forever. Fix it — now and permanently.

**Now (production is down):**
- Identify the offset from logs, then either (a) let it hit the DLT if a handler exists, or (b) stop the consumer group and `--reset-offsets --shift-by 1 --execute` to step over it.

**Permanently:**
1. `ErrorHandlingDeserializer` wrapping the real deserializer — deserialization failures become catchable instead of exploding inside `poll()`.
2. `DefaultErrorHandler` with `addNotRetryableExceptions(DeserializationException.class, ValidationException.class)` → immediate DLT, no retry loop.
3. `DeadLetterPublishingRecoverer` so the bad record is preserved with diagnostic headers.
4. Schema Registry with BACKWARD compatibility so this can't be produced in the first place.
5. Alert on DLQ count > 0.

---

### S6. You need retries with a 30-minute backoff for a flaky third-party API. How?

**Answer:** "Never with blocking retries — a 30-minute sleep in the listener stalls the entire partition and triggers a rebalance."

Use **non-blocking retry topics** with tiered delays:
```java
@RetryableTopic(
    attempts = "5",
    backoff = @Backoff(delay = 60_000, multiplier = 3, maxDelay = 1_800_000),
    // 1m → 3m → 9m → 27m → 30m(cap), then DLT
    dltStrategy = DltStrategy.FAIL_ON_ERROR,
    exclude = {ValidationException.class})       // 4xx = permanent, don't retry
@KafkaListener(topics = "payment-requests")
public void charge(PaymentEvent e) { gateway.charge(e); }
```

**How the delay actually works (good detail to add):** the retry-topic consumer checks the record's timestamp; if the backoff hasn't elapsed it **pauses** the partition and re-seeks rather than sleeping — so the container isn't blocked and no rebalance occurs.

Also mention: distinguish retryable (5xx, timeout, 429) from non-retryable (4xx validation) so you don't burn 30 minutes on a permanently bad request. Add a circuit breaker (Resilience4j) so you stop hammering a dead API.

---

### S7. You must save to Postgres and publish to Kafka atomically. How?

**Say this first:** "That's the dual-write problem. There is no distributed transaction between Postgres and Kafka, so I use the Transactional Outbox pattern."

```java
@Transactional
public void createOrder(OrderRequest req) {
    Order order = orderRepo.save(new Order(req));         // business write
    outboxRepo.save(new OutboxEvent(                      // event in the SAME txn
        UUID.randomUUID(), "Order", order.getId(),
        "OrderCreated", toJson(order)));
}
```
```sql
CREATE TABLE outbox (
  id UUID PRIMARY KEY, aggregate_type TEXT, aggregate_id TEXT,
  event_type TEXT, payload JSONB,
  created_at TIMESTAMPTZ DEFAULT now(), published_at TIMESTAMPTZ
);
```
Then publish with **either**:
- **Debezium CDC** on the outbox table (preferred — no polling, no extra code, uses the WAL), or
- a **polling publisher** (`@Scheduled`, `SELECT ... WHERE published_at IS NULL FOR UPDATE SKIP LOCKED LIMIT 100`) that sends and marks published.

**Guarantee:** at-least-once publication (a crash after send but before marking published → duplicate), so consumers must be idempotent. State this — it shows you understand the boundary.

**Why not `@Transactional` over both?** `ChainedKafkaTransactionManager` is best-effort 1PC: a crash in the window between the two commits leaves them inconsistent. It's deprecated and it's the wrong answer.

🧠 **Memory hook:** *Dual write → Outbox. One DB transaction, one source of truth, CDC does the rest.*

---

### S8. You need to publish 20 MB PDF documents through Kafka.

**Answer:** "Don't. Kafka is tuned for small records; defaults cap at 1 MB and large records wreck latency, page-cache efficiency and replication."

**Preferred: claim-check pattern.** Store the blob in S3/blob storage, publish a small event with the object key, checksum and metadata. Consumers fetch on demand. Mention lifecycle policy alignment so the blob outlives the topic retention.

**If you truly must** (say you know the configs):
```
# broker
message.max.bytes = 25000000
replica.fetch.max.bytes = 25000000
# producer
max.request.size = 25000000
buffer.memory = 134217728
compression.type = zstd
# consumer
max.partition.fetch.bytes = 25000000
fetch.max.bytes = 52428800
max.poll.records = 1
```
And isolate it in a **dedicated topic/cluster** so it can't degrade latency-sensitive topics.

---

### S9. One partition has 10× the traffic of the others (hot key). What do you do?

**Diagnose:** per-partition lag/offset rate shows the skew; usually one tenant/customer dominates.

**Fixes, in order:**
1. **Composite/salted key** — `customerId + "#" + (seq % 10)` spreads one customer across 10 partitions. **Cost: you lose strict per-customer ordering** — only acceptable if the operations are commutative or you re-order downstream by sequence number.
2. **Custom partitioner** that routes the whale tenant to a dedicated set of partitions (see Q10) — preserves ordering, isolates noise.
3. **Separate topic** for high-volume tenants with its own consumer group and scaling profile.
4. **More partitions** helps only if the skew is hash collision, not a genuinely dominant key. Say this — it's the trap answer.

🧠 **Memory hook:** *Hot partition = key design problem, not a scaling problem.*

---

### S10. Production bug corrupted data. You need to reprocess the last 7 days.

**Answer — plan it as a controlled replay:**

1. **Check retention first.** `retention.ms` must still cover 7 days or the data is gone (this is why we set 7-day retention on event topics).
2. **Fix and deploy the code.**
3. **Choose the replay method:**
   - **New consumer group** (safest): deploy a one-off instance with `group-id: order-service-replay-20260907` and `auto-offset-reset: earliest`, or seek-by-timestamp via `ConsumerSeekAware`. The live group is untouched — no risk to steady state.
   - **Reset the existing group** (only in a maintenance window): stop all consumers, `kafka-consumer-groups.sh --reset-offsets --to-datetime 2026-08-31T00:00:00.000 --execute`, restart.
4. **Make it safe:** consumers must be idempotent, or write to a *shadow* table/topic and cut over after validation.
5. **Guard side effects:** replaying will re-send emails/payments unless you gate them — pass a `replayMode` flag or check `occurredAt` age and suppress notifications.
6. **Throttle** the replay (`max.poll.records`, rate limiter) so you don't melt the downstream DB while live traffic is flowing.

🧠 **Memory hook:** *New group + idempotent + suppress side effects + throttle.*

---

### S11. Your listener calls an API that takes 2 seconds. You get 1,000 msg/sec. Design it.

**Do the maths out loud:** 1,000/sec × 2 s = **2,000 concurrent operations** needed. With one thread per partition you'd need 2,000 partitions. Not viable.

**Options:**
1. **Async/reactive** — `WebClient` + Reactor, so a thread isn't parked on I/O. Use `reactor-kafka` or a bounded async pipeline with manual acks after completion. Highest throughput.
2. **Worker pool per consumer** — consumer thread dispatches to a bounded `ThreadPoolExecutor` (say 100 threads); when the queue is full, `consumer.pause()` for backpressure, `resume()` when it drains. Commit only after a batch fully completes; keep per-key ordering by hashing key → worker.
3. **Batch the API calls** — if the API has a bulk endpoint, a batch listener of 500 records becomes 1 call instead of 500. Biggest possible win; always ask if a bulk endpoint exists.
4. **Reduce the work** — cache, or does every message really need the call?

Add a **circuit breaker + bulkhead** (Resilience4j) so a slow API doesn't cascade, and mention that you'd pick partitions ≈ 30–60 with concurrency so each consumer runs a pool.

---

### S12. A broker crashed and we lost messages. How is that possible and how do you prevent it?

**Root causes to enumerate:**
- `acks=1` and the leader died before followers replicated → the acked records were never on another replica.
- `acks=all` **but** `min.insync.replicas=1` and followers had drifted out of the ISR → "all" meant just the leader.
- `unclean.leader.election.enable=true` → an out-of-sync replica became leader and truncated the newer records.
- Consumer-side "loss": `enable.auto.commit=true` committed offsets for records that were fetched but not yet processed.

**Prevention (recite the combo):**
```
replication.factor       = 3
min.insync.replicas      = 2
acks                     = all
enable.idempotence       = true
unclean.leader.election.enable = false
enable.auto.commit       = false   (commit after processing)
broker.rack              = <az>    (spread replicas across AZs)
```
Monitor `UnderReplicatedPartitions` and `IsrShrinksPerSec` so drift is caught before a failure.

---

### S13. You need to add partitions to a live topic. What breaks?

**Answer:**
1. **Key→partition mapping changes** (`hash % N`), so per-key ordering breaks across the change: old events for key K are in P2, new ones in P5, and two consumers may process them concurrently.
2. **Existing data does not move** — old partitions keep their data; only new writes spread out. So skew persists for the retention window.
3. **Triggers a rebalance** of every consumer group on that topic.
4. **Kafka Streams** breaks harder: co-partitioning assumptions and state-store changelogs are invalidated. Usually you must reset the application (`kafka-streams-application-reset`) and rebuild state.

**Safe procedure:** announce it → if strict ordering matters, drain the topic (stop producers, let consumers reach lag 0) → add partitions → restart producers. Or create `orders.v2` with the right partition count and migrate. **Best practice: over-provision partitions at creation** — you can never shrink.

---

### S14. How do you replicate data across two data centres / regions?

- **MirrorMaker 2** (built on Kafka Connect) — active-passive or active-active, with remote topics prefixed by cluster alias (`us-east.orders`), offset translation via `MirrorCheckpointConnector` so consumers can fail over near their last position.
- **Confluent Cluster Linking / Replicator** — commercial, byte-for-byte with offset preservation.
- **Stretch cluster** across AZs (not regions) with `broker.rack` and `min.insync.replicas=2` — works only with sub-10ms latency.

Points that mark a senior: offsets are **not** identical across clusters (hence offset translation), replication is **asynchronous** so failover means potential data loss (RPO > 0), and active-active needs a loop-prevention strategy plus conflict resolution.

**Why you'd do this:** disaster recovery (a whole region goes down), locality (users in EU read EU data), or migration to a new cluster. A single Kafka cluster can't span regions well because replication and `acks=all` would pay cross-region latency on every write.

**How MirrorMaker 2 works:** it runs as Kafka Connect connectors that *consume* from the source cluster and *produce* to the target. Topic `orders` in `us-east` appears as `us-east.orders` in `eu-west` (the prefix prevents replication loops and shows origin). A checkpoint connector records "source offset X ≈ target offset Y" for each consumer group so, on failover, consumers can resume near where they stopped instead of from scratch (usually with a small replay → idempotent consumers again).

**Terms:** *RPO* (Recovery Point Objective) = how much data you can afford to lose (async replication → some seconds of records not yet copied). *RTO* (Recovery Time Objective) = how long failover takes. *Active-passive* = one region serves, the other is a standby; *active-active* = both serve and replicate to each other (needs prefixed topics or conflict handling).

---

### S15. A new field was added to the event schema and consumers started failing. What went wrong and how do you prevent it?

**What went wrong:** producer deployed first with a field consumers couldn't handle — either a strict deserializer (Jackson `FAIL_ON_UNKNOWN_PROPERTIES=true`), a required field without a default, or no compatibility checking at all.

**Fixes:**
1. Schema Registry with `BACKWARD_TRANSITIVE`, `auto.register.schemas=false`, compatibility check in the CI pipeline → the breaking change fails the build, not production.
2. New fields **always optional with a default**.
3. Deploy order: BACKWARD → **consumers first**, always.
4. For JSON: `spring.json.value.default.type` and lenient mapping (`FAIL_ON_UNKNOWN_PROPERTIES=false`).
5. Genuinely breaking change → new topic version `orders.v2`, dual-publish, migrate, retire.

---

### S16. Design an order-processing system for 50,000 events/sec. Talk me through it.

**Say the numbers first.** 50k/sec × 1 KB ≈ 50 MB/s ingress; × RF 3 ≈ 150 MB/s replication; 7-day retention ≈ 30 TB × 3 = 90 TB.

**Topics:** `orders.created` (60 partitions, RF 3, minISR 2, keyed by `orderId`), plus `orders.paid`, `orders.shipped`, `orders.dlt`.

**Partition sizing:** if one consumer handles ~2k/sec, you need ≥ 25 consumers → 60 partitions gives headroom and room to scale to 60 pods.

**Producers:** `acks=all`, idempotence on, `linger.ms=20`, `batch.size=128KB`, `compression=lz4`. Outbox + Debezium if the order service also writes to a DB.

**Consumers:** 10 pods × concurrency 6 = 60 consumers, `max.poll.records=200`, batch listener with `saveAll()`, manual immediate acks, `CooperativeStickyAssignor` + static membership, idempotent writes keyed on `eventId`.

**Cluster:** 6+ brokers across 3 AZs (`broker.rack`), 6 GB heap, NVMe, JBOD.

**Reliability:** DLQ per topic, `@RetryableTopic` for transient failures, circuit breakers, lag alerts on derivative, DLQ-count alert at > 0.

**Observability:** Micrometer + Prometheus + Grafana, `traceparent` header propagation, Kafka Exporter for lag.

---

### S17. Producers are up but a consumer group shows zero lag and processes nothing. What's happening?

Zero lag with no throughput usually means **there's nothing new to read for that group's assignment**, so check:
1. Consumers subscribed to the **wrong topic** (typo) or a **regex** matching nothing.
2. Producers writing to a **different cluster/environment** (bootstrap servers mismatch) — very common.
3. Group has **more consumers than partitions**; the extra ones show nothing (though others should still work).
4. `auto.offset.reset=latest` on a brand-new group: it skipped the backlog and is legitimately caught up — data is there, but "old" data was never read.
5. Consumers assigned partitions but the **container isn't started** (`autoStartup=false`, or listener registry paused).
6. Transactions: `isolation.level=read_committed` and a **hung open transaction** blocks reading past the LSO — lag looks small but nothing advances. Check `transaction.timeout.ms` and abandoned transactional producers.

🧠 **Memory hook:** *Wrong topic → wrong cluster → latest reset → container not running → open transaction.*

---

### S18. How do you implement delayed/scheduled message processing (e.g. "cancel order if unpaid in 30 min")?

Kafka has **no native delayed delivery**. Options:

1. **Tiered delay topics** (`delay-1m`, `delay-5m`, `delay-30m`): consumer reads, checks `record.timestamp() + delay > now`, and if not due it **pauses the partition and seeks back** instead of sleeping. This is exactly what Spring's non-blocking retry does — reuse `RetryTopicConfiguration` machinery.
2. **DB + scheduler** (most common in practice): persist a `scheduled_action` row with `due_at`, and a `@Scheduled`/Quartz job (with `SKIP LOCKED`) publishes when due. Simple, queryable, cancellable.
3. **Kafka Streams punctuator** — keep pending orders in a state store and `context.schedule(...)` a wall-clock punctuation to emit timeouts. Elegant, and it also handles the "payment arrived, cancel the timer" case naturally via the store.

Always mention **cancellation**: the timeout must be suppressed if payment arrives — that's why a state store or DB row beats a naive delay queue.

---

### S19. Kafka cluster is completely down. What should your producer do?

**Behaviour:** `send()` buffers into `buffer.memory` (32 MB) and, once full, blocks up to `max.block.ms` (60 s) then throws `TimeoutException`. Each record fails after `delivery.timeout.ms` (120 s). Left alone, this cascades into request-thread exhaustion and your API goes down with Kafka.

**Design so it doesn't:**
1. **Never block the request thread** — async `send()` with a callback; short `max.block.ms` (e.g. 2000).
2. **Circuit breaker** around the publish; when open, fall back immediately.
3. **Local fallback** — write to the outbox table (S7) and drain when Kafka recovers. This is the strongest answer: the DB write is the source of truth, so publishing can lag without data loss.
4. **Degrade gracefully** — return 202 Accepted, queue locally, alert.
5. Decide explicitly whether the operation is **critical** (fail the request) or **best-effort** (log and continue) — e.g. payment events fail loudly, analytics events are dropped.

---

### S20. How do you achieve both high throughput and strict ordering?

**Answer:** "These are in tension; the resolution is that you only need ordering *per key*, not globally."

- Key by the entity that needs ordering → many partitions → parallel across keys, ordered within a key. Throughput scales with key cardinality.
- Inside a consumer, if you use a worker pool, route by `hash(key) % workers` so a key always lands on the same worker.
- Keep `enable.idempotence=true` so producer retries don't reorder even with 5 in-flight requests (avoid the old `max.in.flight=1` throughput penalty).
- Avoid retry topics on ordered topics; use blocking retry or key-parking.
- If someone insists on **global** ordering: that's 1 partition, 1 consumer, and a hard throughput ceiling. Push back and ask what the real business requirement is — usually it's per-account.

---

### S21. Your compacted topic isn't shrinking. Debug it.

Checklist (say them fast, it shows operational depth):
1. Records have **null keys** — compaction needs keys.
2. Everything is still in the **active segment**, which is never compacted. Lower `segment.bytes`/`segment.ms`.
3. `min.cleanable.dirty.ratio` (0.5) not yet reached.
4. `min.compaction.lag.ms` set high, holding records back deliberately.
5. `cleanup.policy` is actually `delete` (check `kafka-configs --describe`, not just your assumption).
6. **Log cleaner thread died** — check broker logs for `log-cleaner` errors and the `max-clean-time` / `LogCleanerManager` metrics; a single corrupt segment can kill the cleaner for the whole broker.
7. Only the **latest value per key** is kept — if every record has a unique key, compaction can't reduce anything. That's a data-modelling issue.

---

### S22. How do you do a zero-downtime consumer deployment?

- **Rolling update** with `maxUnavailable: 1`. Each pod termination triggers a rebalance; use `CooperativeStickyAssignor` so unaffected pods keep consuming.
- **Static membership** + `terminationGracePeriodSeconds` > shutdown time: if the pod returns within `session.timeout.ms`, **no rebalance happens at all**.
- **Graceful shutdown**: Spring's `SmartLifecycle` stops containers on SIGTERM; ensure in-flight records finish and offsets commit (`setStopImmediate(false)`, sensible `shutdownTimeout`).
- **Backwards-compatible schemas** so old and new pods can both read the topic during the roll.
- For risky logic changes, run a **shadow consumer group** on the new version writing to a scratch destination, compare outputs, then promote.

---

### S23. Two services both need order events, but one is 100× slower. Design it.

**Answer:** separate **consumer groups** — that's the whole point of Kafka's fan-out. Each group has independent offsets, so the slow consumer's lag never affects the fast one.

Additional measures for the slow service:
- Its own partition count requirement — if the topic has 6 partitions but the slow consumer needs 60 workers, either raise partitions or fan out into a dedicated topic keyed for its needs.
- Longer retention on the topic so the slow consumer can fall behind safely.
- Separate lag alert thresholds per group (don't page for the batch consumer being 10 minutes behind).
- If the slow consumer only needs a subset, a filter/projection topic (Streams or a small forwarder) reduces its volume.

---

### S24. How would you migrate from RabbitMQ to Kafka with no downtime?

1. **Dual-write** phase: the producer publishes to both RabbitMQ and Kafka. Kafka consumers run in **shadow mode** (process to a scratch store, no side effects).
2. **Validate**: reconcile counts and outputs between the two paths for a full business cycle.
3. **Cut over consumers** one service at a time; Rabbit consumers idle but ready to re-enable.
4. **Stop publishing to Rabbit**, drain its queues, decommission.

Design differences to call out: Rabbit's per-message ack/nack and routing keys map to **partitions + offsets + consumer-side filtering**; Rabbit's DLX maps to a DLT topic; per-message TTL and priority have **no Kafka equivalent** — that's usually the hardest part of the migration and needs a redesign (delay topics, or keep Rabbit for those flows).

---

### S25. Consumer processes a message, calls a payment API successfully, then crashes before committing. What happens and how do you prevent a double charge?

**What happens:** the offset was never committed → on restart the record is redelivered → the payment API is called again → **double charge**.

**Prevention (layered):**
1. **Idempotency key on the API call** — pass `Idempotency-Key: <eventId>` (Stripe-style); the provider de-dupes. Best answer, because the guarantee lives where the money moves.
2. **Local processed-events table** written in the same DB transaction as recording the charge, checked before calling (Q26). Note the residual window: crash *between* the API call and the DB commit still leaves ambiguity — which is exactly why #1 matters.
3. **Two-phase intent**: persist `payment_attempt(eventId, status=PENDING)` first, call the API, then mark `COMPLETED`. On redelivery, a PENDING row means "query the provider for status" rather than blindly retrying.
4. Keep the window small: `MANUAL_IMMEDIATE` ack right after the transaction commits.

**Interview-winning line:** *"For money, I never rely on Kafka's delivery semantics — I make the payment operation itself idempotent."*

---

### S26. How do you handle a consumer that must stop consuming during a maintenance window?

```java
@Autowired KafkaListenerEndpointRegistry registry;

public void pause()  { registry.getListenerContainer("orderListener").pause(); }
public void resume() { registry.getListenerContainer("orderListener").resume(); }
public void stop()   { registry.getListenerContainer("orderListener").stop(); }
```
**`pause()` vs `stop()`:** `pause()` keeps the group membership and assignment (no rebalance) but stops fetching — the right choice for a short window. `stop()` closes the consumer and leaves the group → rebalance. For long windows, `stop()` frees resources.

Expose these behind an admin endpoint or Actuator, and confirm your retention window is longer than the maintenance window so nothing expires while paused.

---

### S27. Disk on brokers is filling up fast. What do you do?

**Immediate:**
1. Find the biggest topics (`kafka-log-dirs.sh --describe`).
2. Reduce `retention.ms` / `retention.bytes` on the offenders via `kafka-configs.sh` — takes effect at the next segment roll/cleanup, so also lower `segment.bytes` if you need it fast.
3. Delete genuinely unused topics.

**Then structurally:**
- Enable `compression.type=zstd` on high-volume topics (producer-side).
- Switch state-like topics to `cleanup.policy=compact`.
- Add brokers and reassign partitions (`kafka-reassign-partitions.sh`) to spread load.
- Tiered storage (KIP-405 / Confluent) to offload old segments to S3.
- Per-topic retention policy governance — event topics 7 days, audit 30 days, debug 1 day.

**Watch out:** don't just delete `.log` files from disk — that corrupts the partition. Always go through Kafka.

---

### S28. Explain a Kafka production incident you personally debugged. (Behavioural — prepare one!)

**Use STAR and pick a story with a metric.** Template you can adapt:

> **Situation:** "Our order-fulfilment consumer group's lag went from near-zero to 2 million during a Black Friday promo, and confirmations were delayed by ~40 minutes."
> **Task:** "I was on call and owned restoring SLA within the hour and preventing recurrence."
> **Action:** "Per-partition lag showed all 12 partitions lagging equally, so not skew. `records-consumed-rate` had halved while produce rate tripled. Thread dumps showed consumer threads parked on a JDBC call — the listener did one `INSERT` per record and the connection pool was saturated at 10. Short term I scaled from 4 to 12 pods and raised the pool to 40, which cut lag growth. Then I converted the listener to a batch listener with `max.poll.records=200` and `saveAll()` with JDBC batching, and added `CooperativeStickyAssignor` because the scale-up had caused eager rebalance storms."
> **Result:** "Per-record processing dropped from ~11 ms to ~0.7 ms, lag cleared in 25 minutes, and we've handled 4× that peak since. I also added a lag-derivative alert so we catch it at 10k instead of 2M."

Have **one** such story rehearsed. Interviewers at 9 YOE weight this heavily.

---

### S29. How do you decide between more partitions and more consumer groups?

- **More partitions** → more parallelism **within** one logical consumer. Use when a single service can't keep up.
- **More consumer groups** → more independent **readers** of the same data. Use when a *different* service needs the same events.

They solve orthogonal problems, and confusing them is a common junior mistake. Bonus: mention that partitions are hard to change later and cost broker resources, whereas adding a consumer group is free and instant.

**Decision example:** the `orders` topic feeds Billing (needs ~20 msg/s) and Analytics (needs 5,000 msg/s). Billing keeps up with a single consumer → give it its **own group** (it reads all orders independently). Analytics is slow → keep *its* group, but add consumers up to the partition count, and if 10 partitions aren't enough, add partitions (accepting the key-mapping impact, S13). Wrongly giving both services the *same* `group.id` would split the orders between them so each sees only part of the data.

**Quick test:** "Should this message be processed once by *any one* of the instances (scale-out → partitions/consumers) or once by *every* service (fan-out → groups)?"

---

### S30. You're seeing `OutOfOrderSequenceException` in producer logs. What is it and what do you do?

Idempotent producer detected a **gap** in the sequence numbers the broker has for its PID/partition — the broker expected sequence N and got N+2.

Causes:
- Retention/segment deletion or a topic-level cleanup removed records the producer state depended on.
- Broker failover with the producer-state snapshot lost, or an unclean leader election.
- `transactional.id` reuse across instances, or `delivery.timeout.ms` expiring so a batch was dropped mid-sequence — leaving a hole.

Handling: it's a **fatal, non-retryable** error for that producer — you must **close and recreate the producer** (Spring: the `DefaultKafkaProducerFactory` will do this if you call `reset()`, or configure a producer-per-thread and let it be recreated). Then verify: `unclean.leader.election.enable=false`, `delivery.timeout.ms` generous enough, unique stable `transactional.id` per logical producer, and no gaps caused by dropped batches.

**Background (see Q8):** every batch from an idempotent producer carries a sequence number per partition (0, 1, 2, …). The broker only accepts "the next expected" number. A *duplicate* (older number) is silently ignored as a retry; a *gap* (expected 8, got 10) means batch 8–9 never reached the broker yet later ones did — accepting it would silently lose data or break ordering, so the broker rejects it with this exception.

**What to do in practice:** (1) don't just retry the same send — the producer's sequence state is poisoned; recreate it. (2) Look at what happened right before: broker restarts/leader changes, `TimeoutException: Expiring N records` (a batch dropped after `delivery.timeout.ms` while later batches survived), or topic data/state loss. (3) Re-publish the records that failed; since the producer is idempotent after recreation (new PID), consumers should still be idempotent for any batch that did land. It's rare in healthy clusters, so treat a burst as a symptom of broker instability.

---

## 15. Rapid-Fire Round

| Question | Answer |
|---|---|
| Can you reduce partitions? | No. Create a new topic and migrate. |
| Can two consumers in one group read the same partition? | No. One partition → one consumer per group. |
| Where are consumer offsets stored? | `__consumer_offsets` (internal, compacted, 50 partitions). |
| What does the committed offset value mean? | The **next** offset to read (`lastProcessed + 1`). |
| Is ordering guaranteed across a topic? | No — only within a partition. |
| Default `acks` in modern clients? | `all` (since 3.0); previously `1`. |
| Default `enable.idempotence`? | `true` (since 3.0). |
| Max in-flight with idempotence? | 5. |
| What's the default retention? | 7 days (`log.retention.hours=168`). |
| Can a consumer read from a follower? | Yes with rack-aware follower fetching (KIP-392); otherwise leader only. |
| What happens if a consumer never commits? | On restart it re-reads from the last committed offset (or `auto.offset.reset` if none). |
| Does Kafka guarantee no duplicates? | Only with transactions + `read_committed`, and only Kafka→Kafka. |
| How do you delete a specific message? | You can't (immutable log). Compacted topic + tombstone, or encrypt-and-drop-the-key for GDPR. |
| Consumer group with 0 members — do offsets survive? | Yes, for `offsets.retention.minutes` (7 days). |
| Can a producer write to a specific partition? | Yes — pass the partition in `ProducerRecord`. |
| Is `KafkaTemplate` thread-safe? | Yes. `KafkaConsumer` is not. |
| What's the smallest unit of parallelism? | The partition. |
| What breaks zero-copy? | TLS encryption and broker-side re-compression. |
| Max message size by default? | 1 MB (`message.max.bytes` / `max.request.size`). |
| What is `__transaction_state`? | Internal topic holding transaction coordinator state. |
| Difference between `pause()` and `stop()` on a listener container? | `pause()` keeps assignment (no rebalance); `stop()` leaves the group. |
| `KStream` vs `KTable`? | Stream = inserts (facts); Table = upserts (latest state per key). |
| What is co-partitioning? | Joined topics must share key and partition count. |
| Kafka 4.0 headline change? | ZooKeeper removed; KRaft only. |

---

## 16. Config Cheat Sheet

### Producer

| Config | Default | Production | Why |
|---|---|---|---|
| `acks` | all | `all` | No data loss |
| `enable.idempotence` | true | `true` | No duplicate retries, order preserved |
| `retries` | MAX_VALUE | leave | Bounded by `delivery.timeout.ms` |
| `delivery.timeout.ms` | 120000 | 120000 | Real deadline |
| `request.timeout.ms` | 30000 | 30000 | Per-request |
| `max.in.flight...` | 5 | `5` | Max allowed with idempotence |
| `linger.ms` | 0 | `10–50` | Biggest throughput lever |
| `batch.size` | 16384 | `65536+` | Fewer, bigger requests |
| `compression.type` | none | `lz4` / `zstd` | Network + disk savings |
| `buffer.memory` | 33554432 | `67108864` | Absorb bursts |
| `max.block.ms` | 60000 | `2000–5000` | Don't hang request threads |
| `max.request.size` | 1048576 | as needed | Must match broker |

### Consumer

| Config | Default | Production | Why |
|---|---|---|---|
| `group.id` | — | required | Offset identity |
| `enable.auto.commit` | true | `false` | Commit after processing |
| `auto.offset.reset` | latest | `earliest` | Don't silently skip backlog |
| `max.poll.records` | 500 | `50–500` | Match processing speed |
| `max.poll.interval.ms` | 300000 | tune | Progress detector |
| `session.timeout.ms` | 45000 | 45000 | Liveness detector |
| `heartbeat.interval.ms` | 3000 | 3000 | ≈ session/3 |
| `fetch.min.bytes` | 1 | `1–65536` | Fewer round trips |
| `isolation.level` | read_uncommitted | `read_committed` if txns | Skip aborted records |
| `partition.assignment.strategy` | [Range, CooperativeSticky] | `CooperativeStickyAssignor` | Incremental rebalance |
| `group.instance.id` | — | `${HOSTNAME}` | Static membership |

### Topic / Broker

| Config | Default | Production |
|---|---|---|
| `replication.factor` | 1 | `3` |
| `min.insync.replicas` | 1 | `2` |
| `unclean.leader.election.enable` | false | `false` |
| `retention.ms` | 604800000 | per topic policy |
| `segment.bytes` | 1073741824 | `1GB` (smaller for compacted) |
| `cleanup.policy` | delete | `delete` or `compact` |
| `replica.lag.time.max.ms` | 30000 | 30000 |
| `auto.create.topics.enable` | true | **`false`** |
| `offsets.topic.replication.factor` | 3 | `3` |

🧠 **The five numbers to never forget:** *1 MB message, 7 day retention, 1 GB segment, 500 max.poll.records, 5 min max.poll.interval.*

**How to use the tables:** "Default" is what you get with no config; "Production" is the safe starting point, not a law — measure before tuning. The pairs that must move together: `max.request.size` (producer) ↔ `message.max.bytes` (broker) ↔ `max.partition.fetch.bytes` (consumer) for big records (S8); `max.poll.records` ↔ `max.poll.interval.ms` (batch must finish within the interval, Q16); `replication.factor` ↔ `min.insync.replicas` ↔ `acks` (durability triple, Q7). `auto.create.topics.enable=false` stops a typo in a topic name from silently creating a new topic with default 1 partition / RF 1.

---

## 17. Common Exceptions → Root Cause

| Exception | Root cause | Fix |
|---|---|---|
| `CommitFailedException` | Processing exceeded `max.poll.interval.ms`; partitions revoked | Lower `max.poll.records`, raise interval, speed up listener |
| `TimeoutException: Expiring N records` | Broker unreachable or overloaded; `delivery.timeout.ms` hit | Check broker health, raise timeout, check network/ACLs |
| `RecordTooLargeException` | Payload > `max.request.size` / `message.max.bytes` | Raise both, or use claim-check |
| `NotLeaderOrFollowerException` | Leader moved (transient) | Auto-retried; just log |
| `NotEnoughReplicasException` | ISR < `min.insync.replicas` | Restore brokers; check under-replicated partitions |
| `ProducerFencedException` | Another producer took the same `transactional.id` | Close this producer; ensure unique stable IDs |
| `OutOfOrderSequenceException` | Sequence gap for idempotent producer | Recreate producer; check unclean election / timeouts |
| `SerializationException` | Bad payload / schema mismatch | `ErrorHandlingDeserializer` + DLT |
| `InvalidGroupIdException` | Missing `group.id` on a committing consumer | Set `group.id` |
| `UnknownTopicOrPartitionException` | Topic doesn't exist; `auto.create` off | Create topic; check name and ACLs |
| `GroupAuthorizationException` / `TopicAuthorizationException` | Missing ACL | Grant Read/Write/Describe to the principal |
| `NoOffsetForPartitionException` | No committed offset + `auto.offset.reset=none` | Set earliest/latest |
| `WakeupException` | `consumer.wakeup()` called during poll | Expected during shutdown; handle and close |
| `IllegalStateException: Consumer is not subscribed` | Manual assign/subscribe mix-up | Use one or the other |

**Plain-language notes on the less obvious ones:**
- `CommitFailedException` — the consumer was evicted from the group (too slow between polls, Q16), so by the time it tried to commit, someone else owned its partitions. The work was done, but the commit is rejected → those records will be reprocessed by the new owner.
- `TimeoutException: Expiring N records` — N records sat in the producer buffer longer than `delivery.timeout.ms` without being acknowledged, e.g. broker down or a wrong `advertised.listeners` (Q4).
- `NotLeaderOrFollowerException` — you sent to a broker that is no longer the partition's leader; the client refreshes metadata and retries by itself (Q4, Q5).
- `NotEnoughReplicasException` — fewer in-sync replicas than `min.insync.replicas` while `acks=all`: Kafka refuses the write rather than store it under-replicated (Q7, Q27).
- `ProducerFencedException` — a newer producer with the same `transactional.id` took over, so this one is a "zombie" and must stop (Q24).
- `WakeupException` — not an error: it's how another thread (e.g. a shutdown hook) interrupts a blocked `poll()` to stop the loop cleanly. Catch it, `close()` the consumer.

---

## 18. Last-Minute Revision Sheet

**The 10 sentences that carry most interviews:**

1. "Kafka is a distributed append-only commit log; ordering is per partition, and the partition is the unit of parallelism."
2. "Durability = RF 3, minISR 2, acks=all, unclean leader election off. All four, or you have a gap."
3. "`acks=all` with `min.insync.replicas=1` is secretly `acks=1`."
4. "Idempotent producer de-dupes retries via PID + epoch + sequence, and preserves order even with 5 in-flight requests."
5. "Session timeout asks 'are you alive'; max.poll.interval asks 'are you making progress'. Different threads, different failures."
6. "Real systems run at-least-once with idempotent consumers. Exactly-once is Kafka-to-Kafka only."
7. "Dual writes to a DB and Kafka are solved with the transactional outbox, not a chained transaction manager."
8. "`ErrorHandlingDeserializer` first, then `DefaultErrorHandler` with non-retryable exceptions, then a DLT with diagnostic headers. Without step one, a poison pill loops forever."
9. "Blocking retries stall the partition; retry topics don't, but they break per-key ordering. Pick deliberately."
10. "Adding partitions re-maps keys and breaks per-key ordering, and you can never shrink — so over-provision at creation."

**Questions to ask *them* (shows seniority):**
- Are you on KRaft or still ZooKeeper? Any migration planned?
- What are your delivery-semantics expectations — do consumers assume idempotency?
- How do you handle schema evolution today? Schema Registry with CI checks?
- What does your DLQ replay process look like?
- What's your typical partition count and how do you decide it?
- Do you use Kafka Streams anywhere, or is it all plain consumers?

---

*Good luck. Lead with the hook, expand on demand, and always volunteer the trade-off — that's what separates 9 years from 3.*
