# RabbitMQ Interview Guide — Java / Spring Boot Developer (9 Years Experience)

> **How to use this file:** Each answer has a **short answer** (what you say in the first 20 seconds) and a **deep dive** (what you say when they push). Memory hooks are marked with 🧠. At a 9-year level, interviewers care less about definitions and more about *reliability, ordering, retries, scaling, and failure modes* — so the scenario section (Part 10) carries the most weight.

---

## Table of Contents

1. [Core Concepts & AMQP Model](#part-1--core-concepts--amqp-model)
2. [Exchanges & Routing](#part-2--exchanges--routing)
3. [Queues, Durability & Message Guarantees](#part-3--queues-durability--message-guarantees)
4. [Acknowledgements & Publisher Confirms](#part-4--acknowledgements--publisher-confirms)
5. [Dead Letter Exchanges, TTL & Retries](#part-5--dead-letter-exchanges-ttl--retries)
6. [Queue Types: Classic, Quorum, Streams](#part-6--queue-types-classic-quorum-streams)
7. [Clustering, HA & Flow Control](#part-7--clustering-ha--flow-control)
8. [Spring Boot / Spring AMQP Deep Dive](#part-8--spring-boot--spring-amqp-deep-dive)
9. [Performance, Monitoring & Ops](#part-9--performance-monitoring--ops)
10. [Scenario-Based Questions](#part-10--scenario-based-questions)
11. [Design Patterns You Should Be Able to Draw](#part-11--design-patterns-you-should-be-able-to-draw)
12. [RabbitMQ vs Kafka vs ActiveMQ vs SQS](#part-12--rabbitmq-vs-kafka-vs-activemq-vs-sqs)
13. [Rapid-Fire Cheat Sheet](#part-13--rapid-fire-cheat-sheet)
14. [Common Pitfalls Interviewers Love](#part-14--common-pitfalls-interviewers-love)

---

## Part 1 — Core Concepts & AMQP Model

### Q1. What is RabbitMQ and where does it fit in an architecture?

**Short answer:** RabbitMQ is an open-source message broker that implements AMQP 0-9-1. It sits between services as an intermediary so producers and consumers are decoupled in time, location, and throughput. Written in Erlang/OTP, which is why it handles tens of thousands of concurrent connections with lightweight processes and gives us clustering almost for free.

**Deep dive — why we reach for it:**

| Need | How RabbitMQ helps |
|---|---|
| **Decoupling** | Producer doesn't know who consumes, or how many |
| **Load levelling** | Queue absorbs traffic spikes; consumers drain at their own pace |
| **Reliability** | Persistence + acks + confirms means work is not lost on crash |
| **Async processing** | Return HTTP 202 immediately, do heavy work in background |
| **Fan-out** | One event → many independent subscribers |
| **Work distribution** | Round-robin across a consumer pool (competing consumers) |

🧠 **Memory hook: "DLRAFW"** — Decouple, Level, Reliable, Async, Fan-out, Work-distribute.

---

### Q2. Explain the full AMQP model — every component and how a message flows.

**The flow:**

```
Producer → [Connection → Channel] → Exchange → (Binding + Routing Key) → Queue → Channel → Consumer
                                        |
                                   (no match)
                                        ↓
                            dropped / alternate-exchange / returned
```

**Components:**

- **Producer** — publishes messages. Never publishes to a queue directly; it always publishes to an **exchange** (the "default exchange" `""` just makes it *look* like direct-to-queue).
- **Exchange** — the router. Receives messages, applies routing rules, copies them to zero or more queues. It has *no storage*.
- **Binding** — the rule linking an exchange to a queue (or exchange to exchange), usually with a routing key or header pattern. Think of it as a routing table entry.
- **Routing key** — a string attribute on the message the exchange uses to decide the destination. Convention: `order.created.us`, dot-separated, most-general → most-specific.
- **Queue** — the buffer. FIFO, lives on a node, holds messages until acknowledged. Only queues store messages.
- **Consumer** — subscribes to a queue and acknowledges messages.
- **Connection** — a single TCP connection to the broker. Expensive: TLS handshake, auth, heartbeats.
- **Channel** — a lightweight virtual connection *multiplexed* over one TCP connection. All AMQP operations happen on a channel. **Channels are NOT thread-safe** — one channel per thread.
- **Virtual host (vhost)** — a namespace for exchanges/queues/bindings/permissions. Logical multi-tenancy inside a single broker. Default is `/`.
- **Node / Cluster** — an Erlang VM instance; nodes cluster to share metadata (and, for quorum queues, data).

🧠 **Memory hook:** *Exchanges route, queues store, bindings decide, channels talk.*

---

### Q3. Connection vs Channel — why does the distinction matter in production?

**Short answer:** A connection is a real TCP socket; a channel is a logical session multiplexed over it. Opening a connection per operation is a classic production killer — TCP + TLS + SASL auth per message destroys throughput and exhausts file descriptors on the broker.

**Rules I follow:**

1. **One long-lived connection per application** (or a small pool: one for publishing, one for consuming — so publisher flow-control blocking never stalls consumers).
2. **One channel per thread.** Channels are cheap but not thread-safe. Sharing a channel across threads causes frame interleaving and `unexpected command` errors.
3. **Never open a channel per message.** Use caching.
4. **Separate publisher and consumer connections.** When the broker hits a memory alarm it blocks *publishing* connections; if consumers share that connection, you deadlock — consumers can't drain the queue that would clear the alarm.

**Spring:** `CachingConnectionFactory` does this for you. Default `CacheMode.CHANNEL` = one connection, cached channels. `CacheMode.CONNECTION` = pool of connections each with its own channel cache (needed if you want per-connection isolation).

```java
@Bean
public CachingConnectionFactory connectionFactory() {
    CachingConnectionFactory cf = new CachingConnectionFactory("rabbit-host");
    cf.setUsername("app");
    cf.setPassword("secret");
    cf.setChannelCacheSize(50);          // >= number of concurrent publishing threads
    cf.setPublisherReturns(true);
    cf.setPublisherConfirmType(ConfirmType.CORRELATED);
    return cf;
}
```

⚠️ **Gotcha to mention:** if `channelCacheSize` is smaller than your concurrent publisher threads, extra channels are created and **destroyed** each time — you'll see channel churn in the management UI and latency spikes. Also `setPublisherConfirmType` on a connection factory used by consumers is harmless but pointless.

---

### Q4. What is a vhost and how have you used it?

**Short answer:** A logical namespace — its own exchanges, queues, bindings and permission set, isolated from other vhosts on the same broker. Users are granted permissions *per vhost* with regex on configure/write/read.

**Practical use:** separating environments on a shared broker (`/dev`, `/qa`), or separating teams/tenants so `order-service` cannot accidentally consume from `payment-service` queues. Names can collide across vhosts safely.

**Why it exists / how it works:** one broker usually serves many applications. Without a namespace, two teams could both create a queue called `orders` and trample each other, and every user could read every queue. A vhost is like a separate "database" inside the same server: a connection is opened *against one vhost* (it is part of the connection URL, e.g. `amqp://rabbit-host:5672/qa`), and from then on it can only see that vhost's exchanges and queues. Permissions are three regexes per user per vhost — **configure** (create/delete), **write** (publish/bind), **read** (consume/get) — e.g. `order-svc` gets `^orders\..*` on all three but `^$` (nothing) elsewhere.

```bash
rabbitmqctl add_vhost qa
rabbitmqctl set_permissions -p qa order-svc "^orders\..*" "^orders\..*" "^orders\..*"
```

```yaml
spring.rabbitmq.virtual-host: qa      # Spring Boot picks the vhost here
```

**Interview probe:** "Can a message cross vhosts?" — not natively; use a **Shovel/Federation** link, which connects to both vhosts as a normal client.

⚠️ Caveat I'd raise: vhosts isolate *naming and permissions*, **not resources**. Memory, disk, CPU, and file descriptors are shared broker-wide. A noisy vhost will still trigger a broker-wide memory alarm. For real resource isolation you need separate clusters.

---

### Q5. Is a queue FIFO? Does RabbitMQ guarantee message ordering?

**Short answer:** A single queue with a **single consumer** and no redelivery preserves publish order. Ordering breaks the moment you add: multiple consumers, `basic.nack`/`reject` with requeue, priorities, or multiple queues.

**Why redelivery breaks order:** requeued messages go back near the **head** of the queue (position isn't strictly guaranteed), so a failed message #5 can be redelivered *after* #6 and #7 were already delivered.

**How to actually get ordering when the business needs it:**

- **Consistent-hash exchange plugin** — route by a key (e.g. `customerId`) so all messages for one entity always land on the same queue, one consumer each. Ordering per-key, parallelism across keys. This is the standard answer.
- **Single active consumer** (quorum & classic queues, `x-single-active-consumer=true`) — many consumers connect but only one receives; automatic failover. Gives ordering plus HA.
- **Version/sequence numbers in the payload** and reject out-of-order at the consumer.
- **Design it away** — make handlers commutative/idempotent so order doesn't matter.

**Concrete picture of how order breaks:** messages `M1, M2, M3` sit in a queue with two consumers A and B. A gets M1, B gets M2; B is faster and finishes M2 before A finishes M1 — so M2's effects land *before* M1's even though M1 was published first. Add a failure on M1 with `requeue=true` and it is redelivered after M3 too. Example of real damage: `AccountOpened` must be processed before `AccountClosed`; reversed, you'd close a non-existent account.

**How the options above work:**
- *Consistent-hash exchange* — hashes `customerId` to choose one of N queues, so a customer's events share one queue (see Q9).
- *Single active consumer* — the broker only delivers to one registered consumer at a time (others are hot standbys). Declare with `QueueBuilder.durable("q").singleActiveConsumer().build()`. If the active one dies, another takes over — order is kept while you still get failover.
- *Sequence numbers* — producer stamps each event with a per-entity increasing version (`v=7`); the consumer applies it only if `v == lastApplied + 1`, otherwise parks/requeues it until the gap is filled.

🧠 **Memory hook:** *Order needs a single lane. Hash to pick the lane, one driver per lane.*

---

## Part 2 — Exchanges & Routing

### Q6. Explain all exchange types with a concrete use case each.

| Type | Routing logic | Use case |
|---|---|---|
| **Direct** | Routing key must **exactly equal** binding key | Task routing by severity: `error` → error-queue |
| **Fanout** | Ignores routing key, copies to **all** bound queues | Broadcast: `user.deleted` → search, audit, email services |
| **Topic** | Wildcard match on dot-separated key | `order.created.in` matched by `order.created.*` and `order.#` |
| **Headers** | Matches on message **headers**, not routing key | Multi-attribute routing: `{format=pdf, type=report}` |
| **Default (`""`)** | Implicit direct exchange; routing key = queue name | Quick point-to-point, prototypes |

**Topic wildcards — memorize precisely:**
- `*` = exactly **one** word. `order.*.in` matches `order.created.in`, not `order.created`.
- `#` = **zero or more** words. `order.#` matches `order`, `order.created`, `order.created.in.mumbai`.

🧠 **Memory hook:** *Star = single, Hash = heap.*

**Headers exchange `x-match`:**
- `x-match=all` → AND (all listed headers must match) — default
- `x-match=any` → OR (at least one matches)

⚠️ Headers exchanges are noticeably slower than direct/topic — I'd only use them when routing genuinely depends on multiple orthogonal attributes.

**A fanout is faster than a topic** with `#` because there's no pattern matching — mention this if asked about broadcast performance.

---

### Q7. What happens to a message that matches no queue?

Three possibilities, in order of what you should be doing:

1. **Silently dropped** — the default. This is how messages vanish and you get a 3 AM page.
2. **Returned to publisher** — set the `mandatory` flag. Broker sends `basic.return`, and in Spring your `ReturnsCallback` fires.
3. **Routed to an alternate exchange** — set `alternate-exchange` on the exchange. Unroutable messages go there, typically bound to a `unroutable.queue` you monitor.

```java
@Bean
public TopicExchange ordersExchange() {
    return ExchangeBuilder.topicExchange("orders.exchange")
            .durable(true)
            .alternate("orders.unroutable.exchange")   // safety net
            .build();
}

// In config
rabbitTemplate.setMandatory(true);
rabbitTemplate.setReturnsCallback(returned ->
    log.error("UNROUTABLE: exchange={} rk={} code={} text={} body={}",
        returned.getExchange(), returned.getRoutingKey(),
        returned.getReplyCode(), returned.getReplyText(),
        new String(returned.getMessage().getBody())));
```

🧠 **Memory hook:** *Mandatory tells you, Alternate saves you.* In production, do both.

---

### Q8. Exchange-to-exchange bindings — why bother?

You can bind an exchange to another exchange. This gives you a **routing topology** you can change without touching consumers.

**Real use:** a `global.events` topic exchange that all producers publish to, bound to per-domain exchanges (`orders.exchange`, `payments.exchange`). Each team owns their exchange and their queues. Adding a new consumer domain = one new binding, zero producer changes. Also lets you insert an audit fanout at the top that captures everything.

**Why it exists:** normally a binding connects an exchange to a *queue*, so every routing rule is welded to a specific queue. With exchange-to-exchange (E2E) bindings the message is routed through *two hops of rules* before reaching a queue — the first hop can be a coarse company-wide rule and the second a team-specific one, so each layer can be changed independently.

**How it works (example):**

```
producer ──► global.events (topic)
                 ├─ binding "order.#"   ──► orders.exchange (topic) ──► orders.email.queue (order.created.*)
                 │                                                  └► orders.billing.queue (order.paid)
                 └─ binding "payment.#" ──► payments.exchange (topic) ──► payments.ledger.queue
```

```java
@Bean Binding domainBinding(TopicExchange globalEvents, TopicExchange ordersExchange) {
    return BindingBuilder.bind(ordersExchange).to(globalEvents).with("order.#");  // exchange → exchange
}
```

The producer only knows `global.events`; the orders team re-arranges its own queues without a producer deploy.

⚠️ Watch out for **binding loops** — RabbitMQ detects cycles and won't deliver infinitely, but the topology becomes very hard to reason about. Keep it a DAG, max 2 levels deep.

---

### Q9. What's the consistent-hash exchange and when did you need it?

A plugin exchange (`rabbitmq_consistent_hash_exchange`) that hashes the routing key (or a header, or message property) and distributes to bound queues by weight — so the *same* key always goes to the *same* queue.

**Why it matters:** it's the standard solution to "I need ordering *and* parallelism." Sharding by `accountId` gives you N queues processed in parallel, but all events for one account are strictly ordered in one queue.

```java
@Bean
public CustomExchange hashExchange() {
    Map<String, Object> args = Map.of("hash-header", "accountId");
    return new CustomExchange("orders.hash", "x-consistent-hash", true, false, args);
}
// bind N queues with binding key = weight, e.g. "1"
```

⚠️ **Rebalancing caveat:** adding/removing a queue re-maps a fraction of keys. During the transition, two queues may briefly hold messages for the same key — so ordering can break at the moment you scale. Plan resharding during a drain window.

---

## Part 3 — Queues, Durability & Message Guarantees

### Q10. Explain every queue property.

| Property | Meaning | Production default |
|---|---|---|
| `durable` | Queue **definition** survives broker restart | `true` |
| `exclusive` | Only usable by the declaring connection; deleted when it closes | `false` (except RPC reply queues) |
| `autoDelete` | Deleted when the **last** consumer unsubscribes | `false` |
| `x-message-ttl` | Per-message expiry in ms | set it, or queues grow forever |
| `x-expires` | Queue auto-deleted after being unused for N ms | useful for temp/session queues |
| `x-max-length` / `x-max-length-bytes` | Cap queue size | set as a circuit breaker |
| `x-overflow` | What to do when full: `drop-head`, `reject-publish`, `reject-publish-dlx` | `reject-publish` for critical data |
| `x-dead-letter-exchange` | Where rejected/expired messages go | always set |
| `x-dead-letter-routing-key` | Override routing key on dead-lettering | set for clarity |
| `x-max-priority` | Enables priority queue (1–255, use ≤10) | avoid unless needed |
| `x-queue-type` | `classic`, `quorum`, `stream` | `quorum` for anything important |
| `x-single-active-consumer` | Only one consumer active at a time | `true` when order matters |

```java
@Bean
public Queue ordersQueue() {
    return QueueBuilder.durable("orders.queue")
            .withArgument("x-queue-type", "quorum")
            .withArgument("x-dead-letter-exchange", "orders.dlx")
            .withArgument("x-dead-letter-routing-key", "orders.dead")
            .withArgument("x-message-ttl", 3_600_000)         // 1 hour
            .withArgument("x-max-length", 500_000)
            .withArgument("x-overflow", "reject-publish-dlx")
            .withArgument("x-delivery-limit", 5)              // quorum-only poison pill guard
            .build();
}
```

⚠️ **Huge gotcha:** queue arguments are **immutable after declaration**. If you change `x-message-ttl` in code and redeploy, Spring's `RabbitAdmin` tries to redeclare, the broker replies `PRECONDITION_FAILED (406)`, and the channel is closed — your listeners fail to start. Fix by using **policies** for changeable settings (TTL, DLX, length) instead of queue arguments, or by declaring a new queue name and migrating with a Shovel.

🧠 **Memory hook:** *Arguments are tattoos, policies are clothes.*

**Plain-language walkthrough of the confusing ones:**

- **`exclusive`** — the queue belongs to *one connection* and vanishes when that connection closes. Perfect for a private reply queue in RPC; wrong for shared work queues (other services get `RESOURCE_LOCKED`).
- **`autoDelete`** vs **`x-expires`** — `autoDelete` deletes the queue once the last consumer *leaves* (and only after at least one has attached). `x-expires` deletes it after N ms with *no consumers, no `basic.get` and no redeclare* — handy for per-session/per-user temp queues that clients abandon.
- **`x-message-ttl`** — each message is discarded (or dead-lettered) if still unconsumed after N ms. Without it a stalled consumer lets the queue grow until memory/disk alarms hit.
- **`x-max-length` + `x-overflow`** — the cap and what happens when it is reached: `drop-head` silently discards the *oldest* messages (fine for "latest value wins" data, dangerous for orders); `reject-publish` refuses the *new* message and the publisher gets a `nack` confirm (back-pressure); `reject-publish-dlx` does the same but also dead-letters the rejected message so you can inspect it.
- **`x-delivery-limit`** — quorum queues count redeliveries per message; once the count exceeds the limit the message is dead-lettered, which stops a poison message from looping forever.
- **Declaring it twice** — declaration is *idempotent only if every property is identical*; any difference gives the 406 error above.

---

### Q11. What are the three things needed for a message to survive a broker restart?

This is asked constantly. **All three legs must be true** — miss one and you lose messages.

1. **Durable exchange** — `durable=true`
2. **Durable queue** — `durable=true`
3. **Persistent message** — `deliveryMode = 2` (PERSISTENT)

🧠 **Memory hook: "E-Q-M" — Exchange, Queue, Message. A three-legged stool.**

**Why each leg matters:** on restart the broker rebuilds its topology from disk. A non-durable *exchange* is simply gone, so publishes fail or go unrouted. A non-durable *queue* is gone along with everything in it. And even in a durable queue, a *transient* message (`deliveryMode=1`) is never written to disk, so it is discarded. Concretely: durable queue + default `MessageProperties` in Spring = fine; durable queue + a producer built with `MessageDeliveryMode.NON_PERSISTENT` = silent loss of every message on restart.

Spring sets `deliveryMode=PERSISTENT` by default in `MessageProperties`, and `@Queue`/`QueueBuilder.durable()` handles the rest. But if someone hand-builds `MessageProperties` they can silently make it transient.

**And a fourth leg for real guarantees:** durability alone doesn't help if the broker ACKs the publish before writing to disk. You need **publisher confirms** to know the broker actually persisted it. Without confirms, "durable" just means "will survive *if* it made it to disk."

⚠️ Also: persistence is **not free**. Every persistent message is written to the message store and indexed. Persistent throughput on classic queues can be 5–10× lower than transient. For high-volume, low-value data (metrics, clickstream), transient + a stream is often the right trade-off.

---

### Q12. What is a lazy queue? Is it still relevant?

**Classic behaviour:** messages are kept in RAM for fast delivery, paged to disk only under memory pressure.

**Lazy queue (`x-queue-mode=lazy`):** writes messages to disk as soon as possible, keeps almost nothing in RAM. Higher and more *predictable* latency, but the queue can grow to millions of messages without a memory alarm.

**When I used it:** a nightly batch queue that accumulates ~2M messages before a worker fleet spins up. Without lazy mode, the broker paged frantically and the whole cluster stuttered.

**Relevance note (score points here):** as of RabbitMQ 3.12+, classic queues use a **unified single-queue-type v2 store** and `lazy` mode is effectively deprecated — the new behaviour is lazy-like by default. For new work I'd use **quorum queues**, which are on-disk by default and use `x-max-in-memory-length` to bound memory instead.

**Why it existed (the trade-off in plain words):** RAM is fast but finite. A classic queue holding millions of messages in RAM pushes the broker past its memory high-watermark, which triggers a *memory alarm* and blocks all publishers. Paging to disk only "when needed" happens in big sudden bursts, so latency spikes. Lazy mode trades a little steady latency for stability: everything goes to disk immediately, so memory use stays flat no matter how deep the queue is. Enabling it was done by a policy (`rabbitmqctl set_policy lazy ".*batch.*" '{"queue-mode":"lazy"}' --apply-to queues`) rather than code, so it could be changed without redeclaring the queue.

---

### Q13. Priority queues — how do they work and what's the catch?

Declare with `x-max-priority: 10`; publish with `MessageProperties.setPriority(n)`. Higher = delivered first.

**Catches to raise:**
- Each priority level is internally a **separate sub-queue** — high `x-max-priority` (say 255) burns memory and CPU. Keep it ≤ 10, realistically 3–5 levels.
- **Already-prefetched messages don't reorder.** If a consumer has prefetch=100 and 100 low-priority messages are in flight, an urgent one waits. **Priority queues require low prefetch (1–5) to actually work.**
- Priority + requeue interacts badly with ordering.
- Not supported on **streams**; supported on quorum queues only in a limited form (3.12+ has no full priority support for quorum — verify against your version).

🧠 **Memory hook:** *Priority is useless with a big prefetch.*

**Why / how in plain words:** a normal queue is first-in-first-out, so an urgent "password reset email" waits behind 50,000 newsletter messages. A priority queue lets the urgent message jump ahead: the broker keeps one internal list per priority level and always serves the highest non-empty one first.

```java
@Bean Queue emailQueue() {
    return QueueBuilder.durable("email.queue").withArgument("x-max-priority", 5).build();
}

rabbitTemplate.convertAndSend("email.exchange", "email", body, m -> {
    m.getMessageProperties().setPriority(5);   // 5 = urgent, default is 0
    return m;
});
```

**Why prefetch kills it:** the broker has *already pushed* up to `prefetch` messages to the consumer's local buffer before the urgent one arrives; the consumer works through that buffer in the order received. Also, priority only reorders messages that are *waiting in the queue* — if consumers keep up and the queue is nearly empty, priority has nothing to reorder. A common alternative is **separate queues per priority** (`email.high`, `email.low`) with more consumers on the high one — simpler and easier to monitor.

---

## Part 4 — Acknowledgements & Publisher Confirms

### Q14. Explain the acknowledgement modes.

**Consumer side (`AcknowledgeMode`):**

**What an "ack" is and why it exists:** after the broker delivers a message it doesn't delete it yet — it marks it **unacked** (in flight) and keeps it until the consumer says "done". If the consumer's connection dies before acking, the broker **requeues** the message and redelivers it to another consumer. That is how RabbitMQ gives *at-least-once* delivery: work is never lost, but it may be processed twice, so handlers must be **idempotent** (safe to run twice). The `deliveryTag` is just a per-channel counter identifying the delivery; acks must be sent on the same channel that received it.

| Mode | Behaviour | Risk |
|---|---|---|
| `NONE` (auto-ack) | Broker deletes on delivery — fire and forget | **Message loss** if consumer crashes mid-processing |
| `AUTO` (Spring default) | Spring acks after the listener method returns normally, nacks on exception | Good default |
| `MANUAL` | You call `channel.basicAck(tag, multiple)` yourself | Full control; must handle every path or messages leak as unacked |

**The three consumer verbs:**

- `basicAck(deliveryTag, multiple)` — success. `multiple=true` acks everything up to that tag (batch optimization).
- `basicNack(deliveryTag, multiple, requeue)` — failure; supports batching. **RabbitMQ extension**, not core AMQP.
- `basicReject(deliveryTag, requeue)` — failure, single message only.

🧠 **Memory hook:** *Nack = reject in bulk.*

**`requeue=true` vs `false`:**
- `true` → back to the queue, redelivered. **Infinite loop risk** if the failure is deterministic.
- `false` → dead-lettered (if DLX configured) or dropped. **This is what you want for poison messages.**

```java
@RabbitListener(queues = "orders.queue", ackMode = "MANUAL")
public void handle(Order order, Channel channel,
                  @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
    try {
        orderService.process(order);
        channel.basicAck(tag, false);
    } catch (TransientException e) {           // DB down, downstream 503
        channel.basicNack(tag, false, true);   // retry
    } catch (Exception e) {                    // bad payload, validation
        channel.basicNack(tag, false, false);  // → DLQ, do NOT retry
    }
}
```

⚠️ **The distinction that gets you hired:** *transient failures requeue, permanent failures dead-letter.* Blindly requeueing everything is the #1 cause of RabbitMQ CPU meltdowns.

---

### Q15. What are publisher confirms and why aren't they optional?

**The problem:** `channel.basicPublish()` returns void and is asynchronous. It only means "handed to the TCP buffer." The broker may be down, the exchange may not exist, the disk may be full — and your code thinks it succeeded.

**Publisher confirms** put the channel in confirm mode; the broker sends `basic.ack` (or `basic.nack`) back per message once it has taken responsibility — meaning: routed to all queues, and for persistent messages, fsynced to disk (or, for quorum queues, replicated to a majority).

**Spring's three confirm types:**

| Type | Behaviour |
|---|---|
| `NONE` | No confirms (default — don't use for critical data) |
| `SIMPLE` | Enables `waitForConfirms()` / `waitForConfirmsOrDie()` — **synchronous, slow** |
| `CORRELATED` | Async callback with your correlation data — **the production choice** |

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated
    publisher-returns: true
    template:
      mandatory: true
```

```java
@PostConstruct
void init() {
    rabbitTemplate.setConfirmCallback((correlation, ack, cause) -> {
        String id = correlation != null ? correlation.getId() : "unknown";
        if (ack) {
            outboxRepository.markConfirmed(id);
        } else {
            log.error("Publish NACKed id={} cause={}", id, cause);
            outboxRepository.markFailed(id);   // retried by scheduler
        }
    });
}

public void publish(Order order) {
    CorrelationData cd = new CorrelationData(order.getId().toString());
    rabbitTemplate.convertAndSend("orders.exchange", "order.created", order, cd);
}
```

🧠 **Memory hook:** *Confirm = "did the broker take it?" Return = "did any queue want it?"* You need both to prove a message landed.

⚠️ **Correlated confirms are asynchronous** — the callback fires on a listener thread, *not* your publishing thread. Don't assume the message is safe when `convertAndSend` returns. If you need synchronous certainty, use `CorrelationData` with `getFuture().get(timeout)` (Spring AMQP 2.2+) or `waitForConfirmsOrDie`.

---

### Q16. Publisher confirms vs transactions — which and why?

| | Transactions (`tx.select`/`tx.commit`) | Confirms |
|---|---|---|
| Nature | Synchronous, blocking | Asynchronous, pipelined |
| Throughput | ~**250× slower** (fsync per commit) | Very high |
| Scope | Atomic batch of publishes | Per-message acknowledgement |
| Consumer acks | Can be transactional | N/A |

**Answer:** Use **confirms**, essentially always. AMQP transactions require a disk sync per commit and destroy throughput. The only real use is when you need publish + ack to be atomic on the same channel, and even then I'd prefer redesigning for idempotency.

**Important clarification to volunteer:** neither gives you a distributed transaction with your database. RabbitMQ cannot participate in a JTA/XA transaction with your RDBMS meaningfully. For DB + broker atomicity, use the **Transactional Outbox pattern** (see Part 11).

---

### Q17. Explain prefetch / QoS. How do you tune it?

`basicQos(prefetchCount)` limits how many **unacknowledged** messages the broker will push to one consumer.

**Effects:**

- **prefetch = 1** → perfect fairness, slowest throughput (one network round-trip per message). Use for long, uneven tasks (video encoding) and for priority queues.
- **prefetch = 0** (unlimited) → **never do this.** Broker pushes the entire queue to the first consumer, which OOMs while others idle.
- **prefetch too high** → one consumer hoards messages, others starve, and a crash means a big redelivery burst.

**Rule of thumb formula:**

```
prefetch ≈ (total round-trip latency / message processing time) × concurrent consumers
```

Practically: **fast messages (<5 ms) → 100–250. Medium → 20–50. Slow (>1s) → 1–5.**

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 20
        concurrency: 5
        max-concurrency: 20
```

🧠 **Memory hook:** *Fast work, big prefetch. Slow work, prefetch 1.*

⚠️ In Spring, `prefetch` must be **≥ batch-size** and, for `SimpleMessageListenerContainer` with transactions, ≥ `txSize`, or the container will stall waiting for messages that will never arrive.

---

## Part 5 — Dead Letter Exchanges, TTL & Retries

### Q18. What is a DLX/DLQ and what are the four ways a message gets dead-lettered?

A **Dead Letter Exchange** is a normal exchange that a queue forwards "failed" messages to. The queue bound to it is the DLQ.

**A message is dead-lettered when:**

1. Rejected with `basic.reject`/`basic.nack` and `requeue=false`
2. Its **TTL expires** (message TTL or queue TTL)
3. The queue **exceeds max-length** (with `x-overflow: reject-publish-dlx` or `drop-head`)
4. It exceeds `x-delivery-limit` (**quorum queues only** — built-in poison message protection)

🧠 **Memory hook: "R-T-L-D" — Rejected, TTL'd, Length-exceeded, Delivery-limit.**

**Headers RabbitMQ adds (invaluable for debugging):**
- `x-death` — array with `reason` (`rejected` / `expired` / `maxlen` / `delivery_limit`), `queue`, `exchange`, `routing-keys`, `count`, `time`
- `x-first-death-reason`, `x-first-death-queue`, `x-first-death-exchange`

```java
@Bean public DirectExchange dlx() { return new DirectExchange("orders.dlx", true, false); }

@Bean public Queue dlq() {
    return QueueBuilder.durable("orders.dlq")
            .withArgument("x-queue-type", "quorum")
            .build();   // NEVER give the DLQ its own DLX pointing back — infinite loop
}

@Bean public Binding dlqBinding() {
    return BindingBuilder.bind(dlq()).to(dlx()).with("orders.dead");
}
```

⚠️ **Critical gotchas:**
- Don't set a TTL on the DLQ unless you genuinely want to discard failures. People do this and quietly lose all their error data.
- Don't create a **DLX cycle** (DLQ dead-letters back to the main queue) — you build an infinite loop that saturates the broker.
- `x-dead-letter-routing-key` matters: without it the original routing key is preserved, which may not match your DLQ binding, and the message is dropped.

---

### Q19. TTL — what are the levels and which wins?

Three places:

1. **Queue-level message TTL** — `x-message-ttl` on the queue; applies to all messages.
2. **Per-message TTL** — `messageProperties.setExpiration("60000")` (a **String** in ms — classic mistake to set an int).
3. **Queue TTL** — `x-expires`; deletes the *queue itself* after N ms of no consumers and no operations.

**Which wins:** the **lower** of queue TTL and message TTL.

**Why TTL exists:** stale messages are often worse than no messages (a "your OTP is 123456" SMS delivered 2 hours late, a price quote that expired). TTL lets the broker discard — or better, dead-letter — messages that waited too long, and stops a dead consumer from letting a queue grow unbounded.

**Example — all three levels:**

```java
// 1. Queue-level: every message in this queue expires after 60 s
QueueBuilder.durable("otp.queue").withArgument("x-message-ttl", 60_000).build();

// 2. Per-message: this one expires after 10 s (note: String, not int)
rabbitTemplate.convertAndSend("otp.exchange", "otp", body, m -> {
    m.getMessageProperties().setExpiration("10000");
    return m;
});

// 3. Queue TTL: the queue itself is deleted after 30 min of being unused
QueueBuilder.durable("session.42.queue").withArgument("x-expires", 1_800_000).build();
```

When a message expires it is dropped, or dead-lettered if the queue has a DLX (reason `expired` in `x-death`). Setting `x-message-ttl` to `0` means "expire unless a consumer can take it immediately".

⚠️ **The subtle behaviour interviewers probe:** message TTL is only enforced when the message reaches the **head** of the queue (for classic queues). A message with a 1s TTL sitting behind a million messages isn't expired at 1s — it's expired when it's inspected at the front. So "expired" ≠ "removed immediately." Queue-level TTL does have a background expiry reaper, per-message does not.

---

### Q20. How do you implement retry with exponential backoff? Compare the approaches.

**Approach 1 — Spring Retry (in-memory, local to the consumer)**

```java
@Bean
public SimpleRabbitListenerContainerFactory retryFactory(
        ConnectionFactory cf, SimpleRabbitListenerContainerFactoryConfigurer configurer) {
    SimpleRabbitListenerContainerFactory f = new SimpleRabbitListenerContainerFactory();
    configurer.configure(f, cf);
    f.setAdviceChain(RetryInterceptorBuilder.stateless()
            .maxAttempts(4)
            .backOffOptions(1000, 2.0, 10_000)   // 1s, 2s, 4s, capped 10s
            .recoverer(new RepublishMessageRecoverer(rabbitTemplate(), "orders.dlx", "orders.dead"))
            .build());
    return f;
}
```

Or purely in YAML:

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          initial-interval: 1s
          multiplier: 2
          max-attempts: 4
          max-interval: 10s
        default-requeue-rejected: false
```

- ✅ Simple, keeps ordering, no extra topology.
- ❌ **Blocks the consumer thread** while sleeping. With prefetch, you stall the whole queue. Retry state is lost on pod restart. Never use long backoffs (minutes) here.

**Approach 2 — TTL + DLX "delay ladder" (the classic RabbitMQ way)**

```
main.queue --nack(requeue=false)--> retry.dlx --> retry.5s.queue  (TTL 5s,  DLX=main.exchange)
                                              --> retry.30s.queue (TTL 30s, DLX=main.exchange)
                                              --> retry.5m.queue  (TTL 5m,  DLX=main.exchange)
                                              --> parking.lot.queue (no consumer, manual triage)
```

Consumer reads the retry count from `x-death`, picks the next ladder rung, republishes. After the last rung → parking lot.

- ✅ **Non-blocking** — waiting happens inside the broker. Survives restarts. Precise control.
- ❌ More topology to manage; ordering is lost.

**Approach 3 — Delayed Message Exchange plugin** (`x-delayed-message`)

```java
Map<String,Object> args = Map.of("x-delayed-type", "direct");
new CustomExchange("delayed.exchange", "x-delayed-message", true, false, args);
// publish with header: x-delay = 30000
```

- ✅ Arbitrary per-message delay, no ladder needed.
- ❌ Plugin, delays are held in a **Mnesia table on one node** — not replicated, so a node failure loses scheduled messages. Poor for very high volumes of delayed messages.

**Approach 4 — Quorum queue `x-delivery-limit`** — broker-native poison protection. Simplest safety net; combine with one of the above.

🧠 **Memory hook:** *Short retries in memory (Spring Retry), long retries in the broker (TTL ladder or delayed plugin).*

**What I'd say I actually use:** Spring Retry for 2–3 fast attempts (covers blips), then `RepublishMessageRecoverer` into a TTL ladder for longer backoff, then a parking lot with alerting. And `x-delivery-limit` as a belt-and-braces guard.

---

### Q21. What is a "parking lot" queue?

The terminal DLQ after retries are exhausted — a queue with **no consumer**, deliberately. Messages sit there until a human or a tool triages them.

Why it matters: a DLQ that's automatically drained hides bugs. A parking lot with an alert on `queue depth > 0` makes failures **visible**. I'd pair it with a small admin endpoint or Shovel config to replay messages back to the main exchange after the underlying bug is fixed.

**The idea step by step:**

1. A message fails → retried a few times (Spring Retry / TTL ladder, see Q20).
2. Retries exhausted → the message is routed to `orders.parkinglot` instead of being retried forever or thrown away.
3. An alert fires (`rabbitmq_queue_messages_ready{queue="orders.parkinglot"} > 0`).
4. An engineer reads the headers (`x-death`, `x-exception-message` if `RepublishMessageRecoverer` was used), fixes the root cause (bad data, bug, downstream outage), then **replays** by moving messages back to the main exchange.

```bash
# replay with the Shovel plugin from the CLI (dynamic shovel, one-off)
rabbitmqctl set_parameter shovel replay-orders \
 '{"src-protocol":"amqp091","src-uri":"amqp://","src-queue":"orders.parkinglot",
   "dest-protocol":"amqp091","dest-uri":"amqp://","dest-exchange":"orders.exchange",
   "src-delete-after":"queue-length"}'
```

**Pitfall:** replaying *before* fixing the bug just sends the messages back to the parking lot — and replayed messages must be idempotent because some may have partially succeeded earlier.

---

## Part 6 — Queue Types: Classic, Quorum, Streams

### Q22. Compare classic, quorum and stream queues. Which do you choose?

| | **Classic** | **Quorum** | **Stream** |
|---|---|---|---|
| Replication | Mirroring (deprecated) | **Raft consensus** | Raft-based log replication |
| Data safety | Weak under partition | Strong (majority commit) | Strong |
| Storage | Memory-first, pages out | Disk-first (WAL + segments) | Append-only log on disk |
| Consumers | Destructive read | Destructive read | **Non-destructive, offset-based** |
| Replay | ❌ | ❌ | ✅ Multiple readers, rewind |
| Message TTL | ✅ | ✅ | Retention by size/time only |
| Priority | ✅ | Limited | ❌ |
| Poison protection | Manual | ✅ `x-delivery-limit` | N/A |
| Best for | Transient, non-critical, priority | **Anything durable & important** | High-throughput, fan-out, replay, audit |
| Weakness | Data loss risk | Slower for very long queues; needs odd node count | No selective ack, no TTL-per-message |

**My answer:** **Quorum queues by default** for business-critical messaging — they're the modern replacement for mirrored classic queues and have safe, well-defined failure behaviour. **Classic** only for short-lived, low-value, or priority-requiring queues. **Streams** when I need Kafka-like semantics inside RabbitMQ: replay, multiple independent consumer groups reading the same data, or very high ingest (millions/sec) like event sourcing and audit logs.

🧠 **Memory hook:** *Classic = fast and forgetful. Quorum = safe and steady. Stream = log you can rewind.*

**Jargon decoded:**
- **Raft** — a consensus algorithm: each quorum queue has one *leader* and several *followers*. The leader appends a message to its log and sends it to followers; the message counts as committed (and the publisher confirm is sent) only when a **majority** (2 of 3, 3 of 5) has written it. If the leader dies, followers hold an election and the one with the most up-to-date log wins — so no acknowledged message is lost.
- **Destructive read** — once consumed and acked, the message is *deleted*. **Non-destructive (stream)** — reading just moves your offset; the data stays until retention (age/size) removes it, so a new consumer can start from the beginning.
- **WAL (write-ahead log)** — messages are first appended sequentially to a log file and fsynced, which is fast and crash-safe.

**Declaring each type:**

```java
QueueBuilder.durable("q.quorum").quorum().build();                       // x-queue-type=quorum
QueueBuilder.durable("q.stream").stream()
        .withArgument("x-max-length-bytes", 20_000_000_000L).build();     // stream, retention by size
QueueBuilder.durable("q.classic").build();                                // classic (default)
```

---

### Q23. Why were mirrored classic queues deprecated?

- **Split-brain / data loss:** mirror promotion under partition could lose confirmed messages; the `ha-promote-on-failure` semantics were confusing. (*Split-brain* = after a network partition two nodes both believe they are the leader and accept writes, so their data diverges and one side's messages are discarded on healing.)
- **Sync cost:** adding a mirror triggered a full queue sync that *blocked the queue* — a 5M message queue could be unavailable for minutes.
- **No flow control between leader and mirrors**, so a slow mirror degraded the whole queue.

**Background:** "mirroring" meant a classic queue had one *master* and N *mirrors* (copies on other nodes) so that a node crash didn't lose the queue. It was configured with an `ha-mode` policy and was the standard HA answer for years — which is why interviewers ask what replaced it.

Quorum queues fix these with Raft: a write is confirmed only when a **majority** has it, promotion is deterministic, and new members catch up incrementally without blocking.

⚠️ **Quorum caveats to mention (shows depth):** deploy an **odd number** of replicas (3 or 5); with 3 you tolerate 1 failure. If a majority is down, the queue becomes **unavailable** (correctly — CP over AP). Memory use grows with queue length, so bound it with `x-max-in-memory-length` / `x-max-in-memory-bytes`. And quorum queues **don't support** `x-max-priority` fully or transient messages — everything is persistent.

---

### Q24. What are RabbitMQ Streams and when would you pick one over Kafka?

A stream is an immutable, append-only, replicated log. Consumers track an **offset** and read without deleting. Accessible via AMQP 0-9-1 (as a queue) *or* the dedicated binary **stream protocol** (much faster, supports batching, sub-entries, filtering).

**Pick a stream over Kafka when:** you already run RabbitMQ, need replay/fan-out for a subset of flows, and don't want a second platform with its own ops burden. Streams give you time-travel, super-fast fan-out (one copy on disk read by N consumers), and huge queues without memory pressure.

**Stick with Kafka when:** you need the wider ecosystem (Kafka Streams, ksqlDB, Connect, Schema Registry), log compaction, or genuinely enormous multi-petabyte retention.

**How it works / example:** think of a stream as a numbered notebook. Producers append lines (offset 0, 1, 2 …). Each consumer remembers which line it is on and can jump back to any earlier line, so ten services can each read the whole notebook independently and a new service can replay history. Contrast with a normal queue, where a message is torn out of the notebook once read.

```java
@Bean Queue auditStream() {
    return QueueBuilder.durable("audit.stream").stream()
            .withArgument("x-max-age", "7D")                  // retention by age
            .withArgument("x-stream-max-segment-size-bytes", 100_000_000)
            .build();
}

// A consumer chooses where to start via the consumer argument "x-stream-offset":
// "first" | "last" | "next" | <numeric offset> | <timestamp>
```


**Pitfalls to mention:** consuming a stream over AMQP 0-9-1 *requires* a prefetch (QoS) and manual/auto ack of the offset; there is no per-message TTL, no requeue and no priority; and deleting data happens by whole segments when retention is exceeded, not per message.

---

## Part 7 — Clustering, HA & Flow Control

### Q25. How does RabbitMQ clustering work?

Nodes form a cluster sharing an **Erlang distribution** connection, replicating *metadata* (users, vhosts, exchanges, bindings, policies, queue definitions) across all nodes via Mnesia (or Khepri in newer versions).

**Key facts:**
- **Metadata is on every node. Message data is not** — a classic queue lives on exactly one node ("home node"); quorum queues replicate to their member nodes.
- Connecting to node B for a queue on node A works — node B **proxies** to A. That's a hidden network hop; hence "connect close to the leader" tuning.
- Nodes must share an **Erlang cookie** and be reachable on ports 4369 (epmd), 25672 (dist), 5672 (AMQP), 15672 (management).
- All nodes should run the **same Erlang and RabbitMQ major version**.

**Why cluster, and how it works in plain terms:** a single broker is a single point of failure and has a ceiling on connections and throughput. Clustering lets several nodes act as one logical broker: declare an exchange on any node and every node knows about it. What is *not* automatic is data redundancy — if a classic queue's home node dies, the queue is unavailable (or its messages lost) unless you chose a replicated type (quorum/stream). So "cluster" gives you **scalability and shared metadata**, while "replicated queue type" gives you **HA for data**; interviewers like to hear you separate the two.

**Example:** 3 nodes, quorum queue `orders` with leader on node 1 and followers on 2 and 3. A producer connected to node 3 publishes → node 3 forwards to the leader on node 1 → leader replicates to followers → confirm returns to the producer. If node 1 crashes, node 2 or 3 is elected leader in seconds and consumers reconnect.

**Pitfalls:** `rabbitmqctl join_cluster` wipes the joining node's existing data; node names/hostnames must resolve (the Erlang cookie file must be identical, permissions `400`); and a 2-node cluster cannot form a majority after a split, so use 3.

**Client-side HA:** give the connection factory **all** node addresses so it can fail over.

```yaml
spring:
  rabbitmq:
    addresses: rabbit1:5672,rabbit2:5672,rabbit3:5672
    connection-timeout: 10s
```

Plus a load balancer (HAProxy/NLB) in front for connection distribution, with health checks on `/api/health/checks/...` rather than a raw TCP check.

---

### Q26. Explain network partitions and the handling strategies.

When nodes can't see each other, each side may think the other died → **split brain**, divergent metadata.

**Concrete scenario:** 3 nodes (A, B, C). The network between A and {B, C} fails. A is alone; B and C still talk. Without any handling, A keeps accepting connections and declaring queues while B and C do the same — when the network heals the cluster has two conflicting versions of the truth and an operator must pick a side by restarting nodes. The `cluster_partition_handling` setting decides what the nodes do automatically. (It is a *CAP-theorem* choice: `pause_minority` = consistency over availability; `autoheal`/`ignore` = availability over consistency.)

`cluster_partition_handling` modes:

| Mode | Behaviour | When |
|---|---|---|
| `ignore` | Do nothing; you resolve manually | Single-AZ, or you use quorum queues everywhere |
| `pause_minority` | Nodes in the **minority** pause themselves until the partition heals | **Recommended** — chooses consistency |
| `autoheal` | On heal, pick a winning partition; **restart losers** (losing their data) | Prioritizes availability |

**Answer:** `pause_minority` with an **odd** node count. It's the only mode that prevents divergence, and it pairs naturally with quorum queues (which need a majority anyway). Also: never stretch a cluster across regions/high-latency links — Erlang distribution heartbeats will false-positive. Use **Federation** or **Shovel** across regions instead.

🧠 **Memory hook:** *Odd nodes, pause the minority, federate across regions.*

**How `pause_minority` plays out in the scenario above:** A (1 node of 3) is the minority, so it pauses itself — refusing connections so clients fail over to B/C. B and C (2 of 3 = majority) carry on. When the network heals, A rejoins and syncs. Result: no divergence, but A's clients saw a brief outage. With an *even* count (e.g. 2+2) neither side is a majority and **both** pause — another reason for odd numbers.

---

### Q27. Federation vs Shovel — what's the difference?

| | **Federation** | **Shovel** |
|---|---|---|
| Level | Exchange or queue level, declarative | Point-to-point, queue → destination |
| Topology | Links follow bindings automatically | You define source and destination explicitly |
| Direction | Upstream pulls into downstream | Push from source to sink |
| Use case | Multi-region event distribution, WAN | Migrations, draining a DLQ, moving between clusters/versions |

**Practical:** I've used a **Shovel** to migrate a classic queue's backlog into a new quorum queue with zero downtime, and to replay a DLQ back into the main exchange. **Federation** for a "publish in Mumbai, consume in Frankfurt" event bus.

**What they are, in plain words:** both are plugins that make the broker act as a *client* of another broker (or another vhost) and copy messages over a normal AMQP connection — which is why they tolerate slow, flaky WAN links where clustering would not. Unlike a cluster, the two sides are **independent brokers** with separate metadata and can run different versions.

- **Shovel** — "a pipe": consume from a source queue/exchange and republish to a destination, reconnecting automatically. It is dumb and explicit: you name the source and the target.
- **Federation** — "a subscription": a downstream exchange/queue declares an *upstream* and the plugin pulls matching messages over only when there is a local binding/consumer interested. It is topology-aware, so adding a binding downstream automatically widens what flows.

```bash
# Shovel: copy everything from a queue on cluster A to an exchange on cluster B
rabbitmqctl set_parameter shovel mumbai-to-frankfurt \
 '{"src-protocol":"amqp091","src-uri":"amqp://mumbai-host","src-queue":"orders.out",
   "dest-protocol":"amqp091","dest-uri":"amqp://frankfurt-host","dest-exchange":"orders.in"}'
```

**Rule of thumb:** moving/draining/migrating = Shovel; continuously sharing an event stream across sites = Federation.

---

### Q28. What is flow control? What are memory and disk alarms?

RabbitMQ protects itself by **blocking publishers**, not by dropping messages.

- **Memory alarm** (`vm_memory_high_watermark`, default **0.4** = 40% of RAM): when exceeded, all connections that are *publishing* get `connection.blocked`. Consumers keep working so the queues drain.
- **Disk free alarm** (`disk_free_limit`, default 50 MB — set it to at least 1–2× RAM): same blocking behaviour. Prevents the broker corrupting itself when it can't page out.
- **Internal credit-based flow control:** even without alarms, a fast publisher outrunning a queue process gets throttled per-connection. In the UI the connection state shows `flow`.

**Handling it in Spring:**

```java
connectionFactory.addConnectionListener(new ConnectionListener() {
    @Override public void onCreate(Connection c) {}
    @Override public void onShutDown(ShutdownSignalException s) {}
});
// And on the low-level client:
connection.addBlockedListener(
    reason -> log.error("PUBLISHER BLOCKED: {}", reason),
    () -> log.info("Publisher unblocked"));
```

⚠️ **The interview trap:** if publishers and consumers share the same connection, a memory alarm blocks the connection, consumers stop consuming, the queue never drains, and the alarm never clears — **a self-inflicted deadlock**. This is the single strongest argument for separate connections.

🧠 **Memory hook:** *Rabbit doesn't drop, it blocks. Separate your connections or you deadlock.*

---

### Q29. What is `basic.qos` `global` flag, and heartbeats?

**`global` flag on QoS:** `false` (default) = prefetch applies **per consumer**; `true` = per **channel** (shared across all consumers on that channel). Almost always leave it false.

*Example:* channel with 2 consumers and `prefetch=10`. With `global=false`, each consumer can hold up to 10 unacked messages (20 total). With `global=true`, the two consumers *share* a single budget of 10. (Note: quorum queues do not support the channel-global behaviour the same way — prefer per-consumer.)

**Heartbeats** (default 60s, negotiated): AMQP-level keepalive so both sides detect dead peers even when the OS TCP stack hasn't noticed. If two consecutive heartbeats are missed, the connection is closed.

*Why needed:* if a client's machine loses power or a firewall silently drops an idle connection, no TCP FIN/RST is ever sent — the broker would hold the "ghost" connection and its unacked messages forever. Heartbeats are tiny frames sent every N/2 seconds; silence means the peer is gone, so the broker closes the connection and **requeues** its unacked messages. They also keep NAT/load-balancer idle timeouts (often 60–350 s) from killing healthy-but-quiet connections. Tune via `spring.rabbitmq.requested-heartbeat=30s`; the lower of client and server values wins.

⚠️ **Classic production bug:** a consumer doing long blocking work on the connection's I/O thread can miss heartbeats → broker closes the connection → all in-flight messages are requeued → the consumer reconnects and reprocesses → duplicate work and a growing "unacked" count. Spring's listener containers run handlers on separate threads so this is rare, but it happens with hand-rolled clients or with very long `txSize` batches. Fix: shorter work units, or raise `requestedHeartbeat`, never disable it.

---

## Part 8 — Spring Boot / Spring AMQP Deep Dive

### Q30. Walk through a production-grade Spring Boot RabbitMQ setup.

**Dependency:** `spring-boot-starter-amqp` (brings in `spring-rabbit`, `amqp-client`).

**`application.yml`:**

```yaml
spring:
  rabbitmq:
    addresses: rabbit1:5672,rabbit2:5672,rabbit3:5672
    virtual-host: /orders
    username: ${RABBIT_USER}
    password: ${RABBIT_PASS}
    connection-timeout: 10s
    requested-heartbeat: 30s
    publisher-confirm-type: correlated
    publisher-returns: true
    ssl:
      enabled: true
    template:
      mandatory: true
      retry:
        enabled: true
        initial-interval: 500ms
        max-attempts: 3
        multiplier: 2
      receive-timeout: 5s
      reply-timeout: 10s
    listener:
      type: simple
      simple:
        acknowledge-mode: auto
        prefetch: 20
        concurrency: 5
        max-concurrency: 20
        default-requeue-rejected: false     # send failures to DLQ, don't loop
        retry:
          enabled: true
          initial-interval: 1s
          max-attempts: 3
          multiplier: 2
          max-interval: 10s
```

**Key beans:**

```java
@Configuration
public class RabbitConfig {

    @Bean
    public MessageConverter jsonMessageConverter(ObjectMapper mapper) {
        return new Jackson2JsonMessageConverter(mapper);
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory cf, MessageConverter conv) {
        RabbitTemplate t = new RabbitTemplate(cf);
        t.setMessageConverter(conv);
        t.setMandatory(true);
        t.setConfirmCallback(...);
        t.setReturnsCallback(...);
        t.setObservationEnabled(true);       // Micrometer tracing
        return t;
    }

    @Bean
    public RabbitAdmin rabbitAdmin(ConnectionFactory cf) {
        RabbitAdmin admin = new RabbitAdmin(cf);
        admin.setIgnoreDeclarationExceptions(true);   // don't fail startup on 406
        return admin;
    }

    // Group topology so it's declared atomically
    @Bean
    public Declarables orderTopology() {
        TopicExchange ex   = ExchangeBuilder.topicExchange("orders.exchange").durable(true).build();
        DirectExchange dlx = ExchangeBuilder.directExchange("orders.dlx").durable(true).build();
        Queue main = QueueBuilder.durable("orders.queue")
                .withArgument("x-queue-type", "quorum")
                .withArgument("x-dead-letter-exchange", "orders.dlx")
                .withArgument("x-dead-letter-routing-key", "orders.dead")
                .withArgument("x-delivery-limit", 5)
                .build();
        Queue dlq = QueueBuilder.durable("orders.dlq")
                .withArgument("x-queue-type", "quorum").build();
        return new Declarables(ex, dlx, main, dlq,
                BindingBuilder.bind(main).to(ex).with("order.#"),
                BindingBuilder.bind(dlq).to(dlx).with("orders.dead"));
    }
}
```

**Listener:**

```java
@Component
@Slf4j
public class OrderListener {

    @RabbitListener(queues = "orders.queue", concurrency = "5-20")
    public void onOrder(@Payload OrderCreated event,
                        @Header(name = "x-death", required = false) List<Map<String,?>> death,
                        Message raw) {
        log.info("Processing order {} redelivered={}",
                 event.orderId(), raw.getMessageProperties().isRedelivered());
        orderService.handle(event);          // must be idempotent
    }

    @RabbitListener(queues = "orders.dlq")
    public void onDead(Message message) {
        alerting.raise("Order failed permanently", message);
    }
}
```

**How the pieces fit together (read it as a flow):**

1. **`RabbitAdmin`** declares exchanges/queues/bindings (the `@Bean` `Queue`/`Exchange`/`Binding` objects) when the first connection opens — that is how the topology "creates itself" on startup.
2. **`RabbitTemplate`** is the *producer* API (`convertAndSend`); the `MessageConverter` turns your Java record into JSON bytes; confirm/returns callbacks report broker-side outcomes.
3. **`@RabbitListener`** methods are the *consumers*; Spring builds a listener container (threads + channels) per listener, deserializes the body, calls your method, and **acks on normal return**.
4. If the method throws, the container's retry (`listener.simple.retry`) re-invokes it up to 3 times with 1s→2s→4s back-off (blocking that consumer thread); after the last attempt, with `default-requeue-rejected: false`, the message is rejected **without requeue** and the queue's DLX routes it to `orders.dlq`, where `onDead` alerts.
5. `concurrency: 5` / `max-concurrency: 20` = the container starts with 5 consumer threads and scales up to 20 under load; each holds up to `prefetch: 20` unacked messages.

---

### Q31. `SimpleMessageListenerContainer` vs `DirectMessageListenerContainer` — when do you use each?

| | **SimpleMessageListenerContainer (SMLC)** | **DirectMessageListenerContainer (DMLC)** |
|---|---|---|
| Threading | Each consumer has its **own thread**, internal queue between broker and listener | Listener runs **directly on the AMQP client's** consumer thread pool |
| Scaling | `concurrentConsumers` / `maxConcurrentConsumers` → **dynamic** scaling | `consumersPerQueue` → **fixed** per queue |
| Multiple queues | One consumer per container handles all its queues | One consumer **per queue** |
| Transactions/batching | Better support (`txSize`, batching) | Simpler |
| Overhead | Higher (extra thread + internal handoff) | Lower latency, fewer threads |
| Adding queues at runtime | Requires consumer restart | Just adds a consumer |

**My rule:** **SMLC** when I want elastic scaling with load (variable traffic) or transactional/batched consumption. **DMLC** when I have many queues and want low overhead and predictable latency, or when I need to add/remove queues dynamically without disruption. SMLC is the Spring Boot default (`listener.type: simple`).

🧠 **Memory hook:** *Simple = elastic but heavier. Direct = lean but fixed.*

**Plain-language picture:** a *listener container* is the Spring component that connects to the broker, receives messages and calls your `@RabbitListener` method. SMLC works like a **thread pool with a hand-off queue**: AMQP client thread → internal blocking queue → your consumer thread → listener, and it can add/remove consumer threads while running. DMLC skips the hand-off: the AMQP client's own thread calls your listener, one consumer per queue, which is cheaper but you set the count up-front.

```yaml
spring.rabbitmq.listener.type: direct          # switch the Boot default
spring.rabbitmq.listener.direct.consumers-per-queue: 3
```

**Pitfall:** with DMLC a slow listener blocks the client's thread pool (`taskExecutor`), so heavy work should use a dedicated executor; with SMLC a slow listener only holds its own consumer thread.

---

### Q32. How do message converters work? What breaks in production?

`MessageConverter` bridges Java objects ↔ AMQP byte bodies.

- `SimpleMessageConverter` (default) — Strings, byte[], and **Java serialization** for everything else. ⚠️ Java serialization is a **deserialization-attack vector** and couples services to class versions. Replace it.
- `Jackson2JsonMessageConverter` — JSON, the standard choice. Interoperable with non-Java services.
- `MarshallingMessageConverter`, Avro/Protobuf via `ContentTypeDelegatingMessageConverter` — for schema-enforced contracts.

**The classic production break:** `Jackson2JsonMessageConverter` writes the FQCN into the `__TypeId__` header. If the consumer has the class in a different package, deserialization fails with `ClassNotFoundException`. Fix with a `DefaultJackson2JavaTypeMapper` mapping, or better — **infer the type from the listener method signature**:

```java
@Bean
public Jackson2JsonMessageConverter converter() {
    Jackson2JsonMessageConverter c = new Jackson2JsonMessageConverter();
    DefaultJackson2JavaTypeMapper mapper = new DefaultJackson2JavaTypeMapper();
    mapper.setTypePrecedence(Jackson2JavaTypeMapper.TypePrecedence.INFERRED);
    mapper.setTrustedPackages("com.mycompany.events");   // security
    c.setJavaTypeMapper(mapper);
    return c;
}
```

⚠️ Also: a deserialization failure throws `MessageConversionException` **before** your listener runs. Spring wraps it in `ListenerExecutionFailedException` and, importantly, treats it as **fatal/non-requeueable** by default (`ConditionalRejectingErrorHandler`) so it goes straight to the DLQ rather than looping forever. Know that class name — it's a favourite question.

---

### Q33. What is `ConditionalRejectingErrorHandler` and why does it matter?

The default `errorHandler` on listener containers. It inspects the exception and decides: **requeue or reject permanently**.

By default it treats these as **fatal** (reject, don't requeue → DLQ):
- `MessageConversionException`
- `MessageConversionException` from the converter
- `MethodArgumentResolutionException`, `NoSuchMethodException`, `ClassCastException`

Anything else → normal `AmqpRejectAndDontRequeueException` handling per `defaultRequeueRejected`.

**Customizing it — the two exceptions you must remember:**

```java
// Force reject → DLQ, regardless of defaultRequeueRejected
throw new AmqpRejectAndDontRequeueException("bad payload");

// Force requeue, regardless of defaultRequeueRejected
throw new ImmediateRequeueAmqpException("downstream temporarily down");
```

```java
@Bean
public ErrorHandler errorHandler() {
    return new ConditionalRejectingErrorHandler(new FatalExceptionStrategy() {
        @Override public boolean isFatal(Throwable t) {
            Throwable cause = t.getCause();
            return cause instanceof MessageConversionException
                || cause instanceof ValidationException      // business-invalid → DLQ
                || cause instanceof IllegalArgumentException;
        }
    });
}
```

🧠 **Memory hook:** *`AmqpRejectAndDontRequeue` = give up. `ImmediateRequeue` = try again now.*

---

### Q34. `@RabbitListener` — the annotations you should know cold.

```java
@RabbitListener(
    bindings = @QueueBinding(
        value = @Queue(name = "orders.queue", durable = "true",
                       arguments = {
                         @Argument(name = "x-dead-letter-exchange", value = "orders.dlx"),
                         @Argument(name = "x-queue-type", value = "quorum")
                       }),
        exchange = @Exchange(name = "orders.exchange", type = ExchangeTypes.TOPIC),
        key = "order.created.#"),
    containerFactory = "retryFactory",
    concurrency = "3-10",
    ackMode = "MANUAL",
    id = "orderListener",
    autoStartup = "false")
public void handle(@Payload @Valid OrderCreated event,
                   @Headers Map<String, Object> headers,
                   @Header(AmqpHeaders.RECEIVED_ROUTING_KEY) String rk,
                   Channel channel) { ... }
```

- `@SendTo("reply.exchange/reply.rk")` — return value is published there (request-reply).
- `@RabbitHandler` on methods inside a `@RabbitListener`-annotated **class** — polymorphic dispatch by payload type.
- `id` + `RabbitListenerEndpointRegistry` → **start/stop listeners at runtime** (crucial for graceful shutdown, maintenance modes, and blue-green deploys).

**What each attribute in the example does:**

- **`bindings = @QueueBinding(...)`** — declares the queue, exchange and binding right on the listener, so the topology exists without separate `@Bean`s. Handy for small services; for shared/prod topology, prefer central `@Bean` definitions so ownership is clear.
- **`containerFactory`** — picks which `RabbitListenerContainerFactory` bean to use, i.e. which retry/advice chain, converter and ack settings apply (e.g. a `retryFactory` vs a plain one).
- **`concurrency = "3-10"`** — min–max consumer threads for this listener only (overrides the factory default).
- **`ackMode = "MANUAL"`** — you must call `channel.basicAck/basicNack`; hence the `Channel` parameter (see Q14).
- **`autoStartup = "false"`** — container is created but not started; you start it later via the registry (e.g. after cache warm-up).
- **`@Payload @Valid`** — body converted to `OrderCreated` and bean-validated; a validation failure becomes a fatal, non-requeued error (see Q33).
- **`@Header` / `@Headers`** — inject a single AMQP header or all of them (`AmqpHeaders.RECEIVED_ROUTING_KEY`, `DELIVERY_TAG`, `x-death` …).

**`@SendTo` example (request-reply):**

```java
@RabbitListener(queues = "price.request")
@SendTo                                   // empty → replies to the caller's replyTo/correlationId
public PriceResponse price(PriceRequest req) { return pricing.quote(req); }
```

**`@RabbitHandler` example:** one queue carries different event types (told apart by `__TypeId__`/content type):

```java
@RabbitListener(queues = "events.queue")
public class EventListener {
    @RabbitHandler public void on(OrderCreated e) { ... }
    @RabbitHandler public void on(OrderCancelled e) { ... }
    @RabbitHandler(isDefault = true) public void unknown(Object o) { ... }
}
```

```java
@Autowired RabbitListenerEndpointRegistry registry;

public void pauseConsumption() {
    registry.getListenerContainer("orderListener").stop();
}
```

---

### Q35. How do you test RabbitMQ code in Spring Boot?

**Three layers:**

1. **Unit** — mock `RabbitTemplate`, verify `convertAndSend(exchange, rk, payload, correlation)` arguments. Call the `@RabbitListener` method directly with a POJO.
2. **Integration with Testcontainers** — the real answer for 9 YOE:

```java
@SpringBootTest
@Testcontainers
class OrderFlowIT {
    @Container
    static RabbitMQContainer rabbit = new RabbitMQContainer("rabbitmq:3.13-management");

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.rabbitmq.host", rabbit::getHost);
        r.add("spring.rabbitmq.port", rabbit::getAmqpPort);
    }

    @Autowired RabbitTemplate template;

    @Test
    void failedMessageLandsInDlq() {
        template.convertAndSend("orders.exchange", "order.created", invalidOrder());
        await().atMost(10, SECONDS).untilAsserted(() -> {
            Object dead = template.receiveAndConvert("orders.dlq");
            assertThat(dead).isNotNull();
        });
    }
}
```

3. **Contract/schema tests** — Spring Cloud Contract for message contracts, so a producer changing a field breaks the build, not production.

⚠️ Also mention: `@RabbitListenerTest` + `TestRabbitTemplate` for verifying listener invocation without a broker, and `Awaitility` instead of `Thread.sleep` for async assertions.

---

## Part 9 — Performance, Monitoring & Ops

### Q36. What metrics do you monitor, and what does each tell you?

| Metric | What it means | Alert on |
|---|---|---|
| `queue depth` (ready) | Backlog | Sustained growth = consumers too slow / down |
| `unacked` count | In-flight messages | High + flat = **stuck consumer** (blocked thread, missing ack) |
| `publish rate` vs `ack rate` | Producer/consumer balance | Publish > ack for N minutes |
| `consumer utilisation` | % of time consumers were busy vs waiting | <100% means prefetch too low / network bound |
| `redelivered` rate | Retry churn | Spikes = poison messages or nack loops |
| DLQ / parking lot depth | Permanent failures | **> 0** should page someone |
| Memory & disk free | Alarm proximity | Approaching watermark |
| `connection.blocked` events | Flow control active | Any occurrence |
| File descriptors / sockets used | Connection leaks | >80% of limit |
| Erlang process count | Broker health | Trend |
| Quorum: `leader`, `members`, `in-sync` | Raft health | Any member out of sync |

**Tooling:** management plugin (`rabbitmq_management`), Prometheus plugin (`rabbitmq_prometheus`, `/metrics`) → Grafana, plus **Micrometer** on the app side (`spring.rabbitmq.template.observation-enabled`) for publish/consume latency and error rates, and `rabbitmq-diagnostics` CLI for on-box checks.

**Reading the table in plain words:**
- A queue's total = **ready** (waiting, not yet delivered) + **unacked** (delivered to a consumer, awaiting its ack). Ready growing = nobody is taking messages fast enough; unacked high and not moving = consumers took messages but aren't finishing them.
- **Consumer utilisation** is the share of time the queue could push a message to a consumer immediately. Near 100% = consumers are the bottleneck-free; low values mean consumers sit idle because prefetch is too small or the network is slow.
- **Redelivered rate** = how many deliveries are repeat attempts; a steady non-zero value is retry churn, a sudden spike usually means a deploy or a poison message.
- Alert on **trends** (growth rate for 5–10 min), not single samples — a short spike that drains is healthy.

```bash
rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers
rabbitmq-diagnostics check_running && rabbitmq-diagnostics alarms
```

🧠 **The one metric to name if they ask for just one:** *unacked count that isn't moving* — it's the clearest signal of a stuck consumer, and unlike queue depth it can't be explained away by a traffic spike.

---

### Q37. How do you tune RabbitMQ throughput?

**Ordered by impact:**

1. **Reuse connections and channels** — biggest single win.
2. **Right-size prefetch** — too low = latency-bound, too high = unfair and risky.
3. **Batch acks** — `basicAck(tag, true)` with `multiple=true`; in Spring, `txSize`/`batchSize`.
4. **Choose transient over persistent** where the data allows — persistence costs an fsync.
5. **Keep queues short.** RabbitMQ is a *broker*, not a database. A short queue stays in memory and is fast; a deep queue pages to disk and slows everything down. **Design for empty queues.**
6. **Fewer, coarser messages** — thousands of tiny messages have huge per-message overhead. Batch at the application level if semantics allow.
7. **Scale consumers horizontally** with the competing-consumers pattern.
8. **Shard queues** (consistent hash) — a single queue is a single Erlang process and thus a **single-core bottleneck** (~tens of thousands msg/s). Multiple queues use multiple cores. *This is the answer people miss.*
9. **Avoid headers exchanges and high `x-max-priority`.**
10. **Disable unnecessary plugins**; use the Prometheus plugin instead of polling the management HTTP API aggressively (that API is expensive).

🧠 **Memory hook:** *One queue = one CPU core. Want more throughput? Add queues, not just consumers.*

**Why the top items work (so you can justify them):**
- **Connection reuse** — opening a connection costs a TCP handshake, TLS negotiation and auth (tens of ms); a channel open is one cheap frame. Reusing amortises that to ~zero.
- **Batching acks** — each ack is a network frame; `multiple=true` acks 50 messages in one frame. Trade-off: if the consumer crashes, all 50 unacked are redelivered, so handlers must be idempotent.
- **Short queues** — messages in RAM are served without disk I/O; once a queue is millions deep, it pages to disk and every operation slows. If a queue is *always* deep, add consumers or shed load instead of tuning the broker.
- **One queue = one process** — each queue is handled by a single Erlang process (≈ one core). Ten consumers on one queue still funnel through that one process; ten queues (sharded by key) spread over ten cores and across nodes.

**Quick example — turning on Spring batching on the consumer side:**

```yaml
spring.rabbitmq.listener.simple:
  batch-size: 50              # deliver up to 50 messages at once...
  consumer-batch-enabled: true
  prefetch: 100               # ...so prefetch must be >= batch-size
```
```java
@RabbitListener(queues = "metrics.queue")
public void onBatch(List<Metric> batch) { repository.saveAll(batch); }   // one DB round-trip per batch
```

**Measure first:** change one thing at a time and compare publish/ack rates (`rabbitmq-perf-test` is the official load tool) — most "slow RabbitMQ" problems are really slow consumers or too-deep queues.

---

### Q38. How do you secure a RabbitMQ deployment?

- **TLS** for AMQP (5671) and management (15671); verify hostnames; rotate certs. Optionally **mTLS** with client cert auth.
- **No `guest` user**; per-application users with least privilege.
- **Permissions are regex per vhost** on three axes: `configure` (declare/delete), `write` (publish into), `read` (consume from).
  ```
  rabbitmqctl set_permissions -p /orders order-svc "^orders\..*" "^orders\..*" "^orders\..*"
  ```
- **Topic authorisation** for fine-grained routing-key control on topic exchanges.
- **vhost separation** per team/tenant.
- **Trusted packages** on the JSON converter (prevents deserialization of arbitrary classes).
- **Don't put secrets in message bodies** — use a claim-check (store payload in a vault/S3, pass a reference).
- Secrets from Vault/K8s secrets, never `application.yml` in git.
- Restrict management UI by network policy; use tags (`management`, `monitoring`, `administrator`) not blanket admin.

**Why each matters / how to apply it:**
- **TLS** — AMQP is plaintext by default, so credentials (SASL PLAIN) and payloads cross the network readable. TLS encrypts them; *hostname verification* stops a man-in-the-middle presenting any valid cert. In Spring: `spring.rabbitmq.ssl.enabled=true`, `ssl.verify-hostname=true`, `ssl.trust-store=...`. **mTLS** = the broker also demands a client certificate, so identity comes from the cert instead of a password.
- **`guest` user** — ships with password `guest` and by default may only connect from localhost; delete it or restrict it so a misconfiguration can't expose an admin account.
- **Least privilege** — a publish-only service gets `write` on its exchange and `""` (empty = nothing) for `configure`/`read`; then a compromised service can't drain or delete other teams' queues. Also stops code from accidentally declaring/deleting topology in prod (declare via a deployment pipeline with a privileged user, and run apps with `configure` limited).
- **Trusted packages** — Jackson/Java-serialization could otherwise instantiate attacker-chosen classes from the `__TypeId__` header (a deserialization attack).
- **Claim-check** — broker data is stored unencrypted on disk and visible in the management UI, so put sensitive/large payloads in a vault/S3 and send only an id or pre-signed reference.

---

### Q39. How do you do a zero-downtime deployment of a consumer?

1. **Make consumers idempotent** first — nothing else works without this.
2. **Graceful shutdown:** stop the listener container, let in-flight messages finish, then exit.
   ```yaml
   spring:
     lifecycle:
       timeout-per-shutdown-phase: 30s
     rabbitmq:
       listener:
         simple:
           # container shutdown timeout
   server:
     shutdown: graceful
   ```
   Set `setShutdownTimeout(...)` on the container factory; Kubernetes `terminationGracePeriodSeconds` must be **longer** than that, and use a `preStop` hook so the pod stops receiving before it shuts down.
3. **Backward-compatible message schemas** — additive changes only; new consumers must read old messages and old consumers must tolerate new fields (`@JsonIgnoreProperties(ignoreUnknown = true)`).
4. **Rolling deploy** — old and new consumers coexist on the same queue, so version N and N+1 must both handle every in-flight message.
5. For **breaking** schema changes: publish to a **new routing key / new queue** (`order.created.v2`), run both consumers, drain v1, then remove it. Never mutate a queue's arguments in place.

🧠 **Memory hook:** *Idempotent, graceful, additive, versioned.*

---

## Part 10 — Scenario-Based Questions

> This is where a 9-year interview is won or lost. Each answer follows: **diagnose → immediate mitigation → permanent fix.**

---

### S1. Production: a queue has grown to 2 million messages and is still climbing. Walk me through it.

**Diagnose (first 5 minutes):**
1. Management UI / Prometheus: is `consumer count` **zero**? → consumers are down or crash-looping. Check pod status and logs.
2. Is consumer count normal but `ack rate` ≈ 0? → consumers are **stuck**: check `unacked`. High-and-flat unacked = blocked threads (a downstream DB/API hanging with no timeout) or a deadlock.
3. Is `ack rate` healthy but `publish rate` much higher? → genuine capacity problem, or an upstream bug/retry storm producing duplicates.
4. Is `redelivered` rate high? → nack-requeue loop; the same messages spinning forever.
5. Broker side: memory/disk alarms? `connection.blocked`?

**Immediate mitigation:**
- Scale consumer replicas out (if the downstream can take it).
- Raise `prefetch` if consumer utilisation is low and messages are fast.
- If it's a nack loop: fix `default-requeue-rejected: false` and let poison messages go to the DLQ.
- If memory alarm is imminent: enable lazy behaviour / move to a bounded queue with `x-overflow: reject-publish-dlx` so you shed load in a controlled way rather than dying.
- Worst case, **Shovel** the backlog into a holding queue to restore normal latency, and drain it separately.

**Permanent fix:**
- Timeouts on **every** downstream call (this is usually the root cause).
- Bound the queue (`x-max-length` + overflow) so it can never threaten the broker.
- Alert on queue depth *rate of change*, not just absolute depth.
- Shard the queue with a consistent-hash exchange if a single queue process is the ceiling.
- Autoscale consumers on queue depth (KEDA RabbitMQ scaler in K8s).

🧠 **The framework: Consumers alive? Consumers stuck? Genuinely overloaded? Looping?**

---

### S2. The same message is being processed over and over, forever. What's happening and how do you fix it?

**Cause:** a **poison message**. The consumer throws on it, nacks with `requeue=true`, the broker puts it back, the consumer picks it up again — a hot loop that pins CPU and starves every other message.

**Why it happens:** `defaultRequeueRejected` is `true` (Spring's default is `true` unless you change it!), or a blanket `catch (Exception e) { channel.basicNack(tag, false, true); }`.

**Immediate:** stop the consumer, purge or Shovel the offending message out, restart.

**Permanent — layered defence:**
1. `spring.rabbitmq.listener.simple.default-requeue-rejected: false`
2. Distinguish exception types: transient → requeue; permanent (validation, parse, business-rule) → `AmqpRejectAndDontRequeueException` → DLQ.
3. Configure a **DLX** so rejected messages have somewhere to go.
4. `x-delivery-limit` on a quorum queue as a broker-level backstop.
5. Bounded retries with `RepublishMessageRecoverer` into a retry ladder, then a parking lot.
6. Alert on the parking lot.

🧠 **Memory hook:** *Never requeue blindly. A poison message with `requeue=true` is a denial-of-service on yourself.*

---

### S3. Your payment service processed the same payment twice. The message was delivered once as far as the producer knows. Explain.

**RabbitMQ guarantees at-least-once, never exactly-once.** Duplicates are *normal*, not exceptional. Sources:

- Consumer processed the message, then crashed **before** acking → broker redelivers.
- Network partition / heartbeat timeout → channel closed → all unacked messages requeued.
- Producer got no confirm (timeout), retried, but the original *had* landed → two copies.
- A requeue after a partial-success handler.

**The fix is idempotency at the consumer — always.**

```java
@Transactional
public void handle(PaymentCommand cmd) {
    // 1. Unique constraint on the dedup table is the real guarantee
    if (!processedRepo.tryInsert(cmd.messageId())) {   // INSERT ... ON CONFLICT DO NOTHING
        log.info("Duplicate {} ignored", cmd.messageId());
        return;
    }
    // 2. Business work in the SAME transaction as the dedup insert
    ledger.debit(cmd.accountId(), cmd.amount());
}
```

**Key points to make:**
- Producer sets a stable **`messageId`** (business idempotency key — e.g. `paymentId`, not a random UUID per publish attempt). Spring: `MessageProperties.setMessageId(...)`, and `RabbitTemplate` with a `MessagePostProcessor`.
- The dedup check and the business write must be in **one local DB transaction**. A Redis `SETNX` check is a nice fast path but is not a guarantee — it can succeed and then the DB write can fail.
- Prefer **naturally idempotent operations**: `SET status = 'PAID'` (idempotent) over `balance = balance - 100` (not).
- Use **conditional updates / optimistic locking** with a version so a stale duplicate is a no-op.
- Retention: keep dedup keys as long as your maximum possible redelivery window (TTL + retry ladder + replay), typically 7–30 days.

🧠 **Memory hook:** *At-least-once delivery + idempotent consumer = effectively-once processing.* That phrase — "effectively once" — is exactly what interviewers want to hear.

---

### S4. You need to guarantee that a DB row and a message are both written, or neither. RabbitMQ can't join your DB transaction. What do you do?

**Answer: the Transactional Outbox pattern.**

```
1. In ONE local DB transaction:
     INSERT INTO orders (...)
     INSERT INTO outbox (id, aggregate_id, type, payload, created_at, status='NEW')
   COMMIT   ← the only atomic thing you need

2. A separate relay (scheduled poller or CDC/Debezium on the outbox table)
   reads NEW rows → publishes to RabbitMQ with publisher confirms
   → marks SENT on confirm, leaves NEW on nack (retried next cycle)
```

```java
@Transactional
public Order createOrder(CreateOrderCmd cmd) {
    Order order = orderRepo.save(new Order(cmd));
    outboxRepo.save(new OutboxEvent(order.getId(), "order.created", toJson(order)));
    return order;   // NO publish here
}

@Scheduled(fixedDelay = 500)
@Transactional
public void relay() {
    List<OutboxEvent> batch = outboxRepo.lockNewBatch(100);   // SELECT ... FOR UPDATE SKIP LOCKED
    for (OutboxEvent e : batch) {
        CorrelationData cd = new CorrelationData(e.getId().toString());
        rabbitTemplate.convertAndSend("orders.exchange", e.getType(), e.getPayload(), cd);
    }
    // confirm callback flips status to SENT
}
```

**Why the naive approaches fail — say this, it shows experience:**
- **Publish inside the `@Transactional` method:** the publish happens before commit; if the commit fails, you've announced an order that doesn't exist. *Phantom event.*
- **Publish after commit:** if the app crashes between commit and publish, the order exists but nobody knows. *Lost event.*
- **`ChainedTransactionManager` (DB + Rabbit):** best-effort 1PC only — the commits are sequential, so there's still a window. It reduces the risk, doesn't remove it. Also deprecated in recent Spring.

**Trade-off:** outbox gives at-least-once, so the **consumer must still be idempotent** (S3). Also mention the read-side twin — the **Inbox pattern** — for storing processed message IDs.

🧠 **Memory hook:** *Outbox on the write side, inbox on the read side, idempotency in the middle.*

---

### S5. Business requirement: all events for a given customer must be processed in strict order, but you need to handle 50k events/sec across 10 million customers.

**The tension:** strict global ordering means one queue and one consumer = no scale. Full parallelism = no ordering.

**Solution — partition by key:**

1. **Consistent-hash exchange** keyed on `customerId` → 32 queues.
2. **One consumer per queue** (or `x-single-active-consumer=true` with several standby consumers for HA).
3. Ordering is guaranteed *per customer* (all their events hash to the same queue, processed serially); parallelism is 32×.

**Additional guardrails I'd insist on:**
- `prefetch` can be > 1 **only if** the consumer processes strictly sequentially — with SMLC and one consumer per queue, that's true. Never set `concurrency > 1` on an ordered queue.
- Requeue breaks ordering. So on failure: **stop the partition** (nack with requeue and pause the container) rather than skip ahead, or dead-letter and record the customer as "needs repair." Decide which the business prefers — *blocking* (correct but stalls one customer) vs *skipping* (available but out of order). Interviewers love that you name this trade-off.
- Include a monotonic **sequence number** per customer so the consumer can detect and reject out-of-order arrivals defensively.
- Resharding re-maps keys, so plan a drain window (see Q9).

**Alternative:** a **RabbitMQ Stream** with `super streams` (partitioned streams) — gives Kafka-style partition ordering with offset tracking and replay, and better ingest at 50k/sec.

🧠 **Memory hook:** *Order per key, parallel across keys. Hash the key, one consumer per partition.*

---

### S6. A downstream third-party API is down for 2 hours. Messages keep arriving. Design the handling.

**What NOT to do:** requeue in a tight loop (hammers the dead API and pins your CPU), or use Spring Retry with a 2-hour backoff (blocks consumer threads for hours).

**Design:**

1. **Circuit breaker** (Resilience4j) around the API call. On OPEN:
   - **Stop the listener container** via `RabbitListenerEndpointRegistry` — messages simply accumulate in the queue, which is exactly what a queue is *for*. Zero CPU, zero API load.
   - A scheduler probes the API; when healthy, restart the container.
2. If you can't pause (mixed workload on the queue), route the failing messages to a **TTL retry ladder** (5s → 1m → 5m → 30m) so waiting happens **inside the broker**, not on your threads.
3. Bound the main queue (`x-max-length`, `x-overflow: reject-publish-dlx`) so a long outage can't threaten the broker.
4. **Distinguish failure types:** 5xx/timeout → retry. 4xx (bad request) → DLQ immediately, retrying will never help.
5. Alert on queue depth so ops knows the backlog exists, and make sure consumers can drain faster than the arrival rate once recovery happens (temporarily scale out, mindful of the third party's rate limits — add a rate limiter so you don't DDoS them on recovery).

🧠 **Memory hook:** *Pause the consumer, let the queue do its job. Waiting belongs in the broker, not in a thread.*

---

### S7. Your team reports "we lost messages." How do you prove where, and prevent it?

**Audit the whole path — there are exactly five places a message can be lost:**

| Stage | Loss cause | Prevention |
|---|---|---|
| **1. Producer → Broker** | Publish failed silently (broker down, connection blocked) | **Publisher confirms** (`correlated`) + retry + outbox |
| **2. Exchange → Queue** | No matching binding; routing key typo | **`mandatory` + ReturnsCallback**, **alternate exchange** |
| **3. In the queue** | Broker restart with non-durable queue/transient message; TTL expiry; max-length `drop-head` | **Durable exchange + durable queue + persistent message**; quorum queue; `reject-publish` instead of `drop-head`; no TTL on critical queues |
| **4. Broker → Consumer** | `autoAck` and consumer crashed mid-processing | **Manual/AUTO ack after** successful processing |
| **5. Consumer processing** | Exception swallowed, message acked anyway; nack with `requeue=false` and **no DLX** | Never ack in a `finally`; **always configure a DLX**; alert on DLQ |

**Investigation steps:** correlate producer logs (with `messageId`/`correlationId`) against consumer logs; check the `unroutable` queue and DLQ depths; check broker logs for restarts and alarms; check whether any queue was redeclared/purged; look at `messages_ratio` and `drop` counters.

🧠 **Memory hook: five gates — Publish, Route, Store, Deliver, Process.** Name them in order and you look like you've done this before, because that's exactly how you actually debug it.

---

### S8. During a rolling deploy you added a required field to your event. Old consumers are crash-looping. Fix it now, and prevent it forever.

**Now:**
- Roll back the producer if possible — that's the fastest stop.
- If messages are already in the queue: stop the listeners, Shovel the incompatible messages to a holding queue, deploy the new consumers, replay.
- Ensure `MessageConversionException` is fatal so they go to the DLQ instead of looping (this should already be true via `ConditionalRejectingErrorHandler`).

**Forever — schema evolution rules:**
1. **Never** add a required field or remove/rename an existing one. Only **additive, optional** changes.
2. Consumers: `@JsonIgnoreProperties(ignoreUnknown = true)` so new fields don't break old code.
3. Include a **`schemaVersion`** header on every message.
4. **Deploy consumers before producers.** Consumers must be able to read the new format *before* anything writes it. (This ordering question is a common follow-up — get it right.)
5. For breaking changes, use **parallel routing keys**: `order.created.v1` and `order.created.v2`, both bound; run both consumers; drain v1; retire it.
6. Enforce with **Spring Cloud Contract** or a schema registry (Avro/Protobuf) in CI so an incompatible change fails the build.

🧠 **Memory hook:** *Consumers first, producers second. Additive only.*

---

### S9. You must send an order-confirmation email 24 hours after an order, and cancel it if the order is cancelled meanwhile. Design it.

**Options:**

| Approach | Verdict |
|---|---|
| Delayed Message Exchange plugin with `x-delay: 86400000` | Works, but 24h of scheduled messages live in a **non-replicated Mnesia table on one node**; a node loss loses them. Also memory-heavy at volume. |
| TTL queue (`x-message-ttl: 86400000`) + DLX to the email queue | No plugin needed, durable, replicated. **But** TTL is head-of-queue based, so a mixed-TTL queue has the "head blocking" problem — needs a dedicated queue per delay value. |
| **DB + scheduler (my recommendation for 24h)** | Store `send_at` in a table, poll every minute with `SELECT ... WHERE send_at <= now() AND status='PENDING' FOR UPDATE SKIP LOCKED`, publish then. Cancellation is a simple `UPDATE status='CANCELLED'`. |

**Why the DB wins here:** RabbitMQ has **no way to cancel or look up an individual queued message**. Messages are opaque and only accessible in order. The cancellation requirement is what rules the broker out — say this explicitly, it's the whole point of the question.

**Hybrid** that people like: DB as the source of truth for scheduling + a short-delay queue for the final hop, so retry/DLQ semantics still apply to the email send itself.

🧠 **Memory hook:** *RabbitMQ cannot cancel or query a message. If you need to cancel it, schedule it in a database.*

---

### S10. Broker memory is at 90%, publishers are blocked, and the system is frozen. Walk me through recovery.

**Diagnose:** `rabbitmq-diagnostics memory_breakdown` / management UI → where is the memory?
- `queue_procs` / message bodies → deep queues.
- `connection_readers/writers` → too many connections (each costs ~100 KB+).
- `binary` → large message bodies held in memory.
- `mnesia` → huge topology (thousands of queues/bindings) or the delayed-message plugin.

**Immediate:**
1. **Get consumers draining.** If consumers share a blocked connection with publishers, they're stuck too — that's the deadlock. Restart consumers on a *separate* connection.
2. Temporarily raise `vm_memory_high_watermark` (e.g. 0.4 → 0.6) via `rabbitmqctl set_vm_memory_high_watermark 0.6` to unblock and let the queue drain. **Temporary only** — you're borrowing headroom from the OS.
3. Set the offending classic queue to lazy mode / paging so bodies go to disk.
4. Purge or Shovel out non-critical backlog if the business allows.
5. Scale consumers hard.

**Permanent:**
- **Bound every queue**: `x-max-length` + `x-overflow: reject-publish-dlx`. Load shedding beats broker death.
- Quorum queues with `x-max-in-memory-length`.
- **Separate publisher and consumer connections** (removes the deadlock class entirely).
- Move large payloads out of messages — **claim-check pattern** (put the blob in S3, send the key).
- Alert at 60% memory, not at the alarm threshold.
- Right-size: RabbitMQ wants **short queues**; a persistent deep backlog means the architecture is wrong (consumers under-provisioned or the workload belongs in a stream/Kafka).

🧠 **Memory hook:** *Rabbit blocks publishers to save itself. Your job is to make sure consumers aren't blocked too.*

---

### S11. You need to send a 50 MB PDF through RabbitMQ. How?

**Don't.** RabbitMQ is optimised for small messages (KB range). Large messages:
- are held in memory as Erlang binaries → memory alarms,
- block the single-threaded queue process → head-of-line blocking for everyone,
- risk frame-size and heartbeat issues on slow links,
- make redelivery hugely expensive.

**Use the Claim Check pattern:**

```
Producer → upload PDF to S3/blob store → publish a small message:
   { "documentId": "...", "location": "s3://bucket/key", "checksum": "...", "sizeBytes": 52428800 }
Consumer → reads the reference → downloads → processes → (optionally) deletes the blob
```

**Extras to mention:** set a lifecycle/TTL on the blob so orphans are cleaned up; include a checksum so you detect corruption; the reference must be valid for at least as long as your retry window (don't delete the blob before retries are exhausted); presigned URLs expire — store the key, not the URL.

**Practical limit:** keep messages under ~128 KB, ideally a few KB. If you must send bigger, chunk and reassemble, but the claim check is almost always better.

---

### S12. Implement synchronous request-reply over RabbitMQ. What are the pitfalls?

```java
// Simple: Spring handles the reply queue and correlation for you
PriceQuote quote = (PriceQuote) rabbitTemplate.convertSendAndReceive(
        "pricing.exchange", "price.request", new PriceRequest(sku));

// Responder
@RabbitListener(queues = "price.request.queue")
public PriceQuote onRequest(PriceRequest req) {
    return pricingService.quote(req);      // return value auto-published to replyTo
}
```

**How it works:** the template sets `replyTo` (a **Direct Reply-To** pseudo-queue, `amq.rabbitmq.reply-to`, which needs no queue declaration) and a `correlationId`; the responder publishes the return value to `replyTo` with the same correlation ID.

**Pitfalls — the reason this question is asked:**
1. **You're doing sync over async.** You've added a broker to a request-response path: more hops, more failure modes, and the client thread is blocked. Ask *why not just call HTTP/gRPC?* Often the right answer is "we shouldn't do this."
2. **Timeouts are mandatory.** `setReplyTimeout(10_000)`. Without it, threads leak forever. A timeout returns `null` — handle it.
3. **Don't create a reply queue per request** — that's exclusive-queue churn that hammers the broker. Use **Direct Reply-To** or one shared reply queue with correlation-ID matching.
4. **Zombie replies:** the caller timed out, then the reply arrives — the consumer of the shared reply queue must discard unknown correlation IDs.
5. Prefer **`AsyncRabbitTemplate`** returning a `CompletableFuture` so you don't block a thread.
6. Cascading failure: if the responder slows, callers back up and thread pools exhaust. Add a bulkhead/circuit breaker.

🧠 **Memory hook:** *Request-reply over a queue = a distributed synchronous call with extra steps. Always timeout, never a queue per request.*

---

### S13. One RabbitMQ node in a 3-node cluster dies. What happens to your queues and your app?

**Depends entirely on queue type:**

- **Classic non-mirrored queue** whose home node died → the queue is **unavailable**; its messages are inaccessible until the node returns. If the queue was durable and the node comes back, messages are intact. If the node is gone forever, the messages are gone. Producers publishing to it get errors or (worse) silent drops.
- **Quorum queue** with 3 replicas → a new **leader is elected** from the remaining 2 (a majority of 3). No data loss for confirmed messages. Brief unavailability during election (usually < 5s). This is why quorum queues are the answer.
- **Quorum queue with only 2 replicas** and one dies → 1 of 2 is not a majority → **unavailable**. Hence *odd numbers*.
- **Stream** → same Raft-based failover.

**Client behaviour:** if `addresses` lists all three nodes, the client reconnects to a surviving node automatically. Spring's `CachingConnectionFactory` retries per `connection-timeout` and the listener container retries per `recovery-interval` (default 5s). In-flight unacked messages are **requeued** → duplicates → *idempotency again*.

**What you should have in place:**
```yaml
spring:
  rabbitmq:
    addresses: rabbit1:5672,rabbit2:5672,rabbit3:5672
    listener:
      simple:
        missing-queues-fatal: false     # don't kill the container if a queue is briefly gone
```
Plus `cluster_partition_handling: pause_minority`, quorum queues with 3 or 5 replicas, and consumers that tolerate redelivery.

🧠 **Memory hook:** *Classic queue = single point of failure. Quorum queue = majority survives. Odd node counts always.*

---

### S14. You have to migrate 200 existing classic mirrored queues to quorum queues with no downtime.

**Constraint:** you **cannot change a queue's type in place.** `x-queue-type` is immutable.

**Plan:**

1. **Inventory & assess:** which queues actually need quorum (durable, business-critical)? Which use features quorum lacks (priority, per-message TTL specifics, transient)? Those stay classic or get redesigned.
2. **Declare the new queues alongside** the old: `orders.queue.q` (or use a versioned name), bound to the **same exchange with the same routing key**. Now both queues receive every new message.
3. **Deploy consumers that read from both** queues (a `@RabbitListener(queues = {"orders.queue", "orders.queue.q"})`). Processing is idempotent, so double-delivery during the overlap is safe. If the messages must not be processed twice, instead: bind only the old queue, and use a **Shovel** to move from old → new after cutover.
4. **Drain the old queue** to zero (monitor depth).
5. **Remove the old binding**, stop consuming from it, delete it.
6. **Rename** (optional): if the name matters, do a second pass with a Shovel into a freshly created queue with the original name.

**Alternatives:** blue-green at the **vhost** level (build the whole new topology in a new vhost, switch app config, Shovel the residual backlog) — cleaner for large migrations.

**Also mention:** quorum queues need `x-delivery-limit` (poison protection changes behaviour — messages now dead-letter after N deliveries where classic looped forever), memory limits (`x-max-in-memory-length`), and a check that no code depends on features quorum drops.

🧠 **Memory hook:** *Can't change type — must create, dual-bind or shovel, drain, delete.*

---

### S15. A consumer's queue shows 0 ready but 500 unacked, and nothing is progressing. Diagnose.

**This is the "stuck consumer" signature.** 500 unacked = the broker delivered them and is waiting for acks that never come.

**Causes, in likelihood order:**
1. **Blocked thread** — a downstream HTTP/DB call with **no timeout**. The consumer thread is parked forever. → thread dump confirms it (`jstack`, look for threads in `SocketRead` / `park`).
2. **Missing ack path** in MANUAL mode — some code path returns or throws without calling `basicAck`/`basicNack`. Classic bug: acking only in the `try` block, exception path acks nothing.
3. **Deadlock** in application code, or an exhausted DB connection pool (all connections held by the same consumer threads).
4. **Prefetch too high** and messages are just very slow — check if `ack rate` is nonzero-but-tiny; then it's not stuck, it's slow.
5. **Infinite loop / GC thrashing** in the handler.

**Immediate:** thread dump + heap check, then restart the consumer — unacked messages are requeued and redelivered (which is why idempotency matters).

**Permanent:**
- **Timeouts everywhere**: HTTP connect/read, DB query timeout, `@Transactional(timeout = N)`.
- Reduce `prefetch` so a stuck consumer holds fewer hostages.
- Add a **consumer-side watchdog**: metric on time-since-last-ack, alert if it exceeds a threshold.
- In MANUAL mode, use a `finally` that guarantees exactly one of ack/nack — or just use Spring's `AUTO` mode and let the container handle it (my preference; manual acking is where bugs live).
- Consider `consumer_timeout` on the broker (default 30 minutes in 3.8.15+) — the broker will close the channel of a consumer that doesn't ack in time, forcing a requeue. Know that this exists; it's why very long-running handlers see mysterious channel closures.

🧠 **Memory hook:** *Unacked high and flat = a thread is stuck. Ready high = consumers are too few or too slow.*

---

### S16. Design an event-driven order flow: order placed → payment → inventory → shipping → notification. Include failures.

**Topology:**

```
                        ┌────────────────────────────┐
   order-service ──────►│ orders.exchange  (topic)   │
     (outbox relay)     └────┬──────────┬──────────┬──┘
                             │          │          │
              order.created  │          │          │
                       ┌─────▼───┐ ┌────▼─────┐ ┌──▼─────────┐
                       │payment.q│ │analytics.q│ │audit.stream│
                       └─────┬───┘ └──────────┘ └────────────┘
                             │ publishes payment.succeeded / payment.failed
                       ┌─────▼─────────────┐
                       │ payments.exchange │
                       └────┬──────────┬───┘
                    ┌───────▼──┐   ┌───▼──────────────┐
                    │inventory.q│   │notification.q    │
                    └───────┬───┘   └──────────────────┘
                            │ inventory.reserved / inventory.rejected
                       ┌────▼──────────┐
                       │shipping.q     │
                       └───────────────┘

Every queue: quorum + DLX → <service>.dlx → <service>.dlq → parking lot
```

**Design decisions to articulate:**
- **Topic exchange per domain**, not one giant exchange — each team owns their exchange and topology.
- **Events, not commands**, between services (`order.created`, past tense) → new subscribers need zero producer changes.
- **Outbox pattern** in each service so DB state and published events can't diverge.
- **Idempotent consumers** with a dedup/inbox table keyed on `messageId`.
- **Saga / choreography** for the distributed transaction: no 2PC. Each step publishes its outcome; failures trigger **compensating events** (`payment.failed` → `order.cancelled`; `inventory.rejected` → `payment.refund.requested`). Mention **orchestration** (a saga coordinator service holding state) as the alternative when the flow gets complex — easier to reason about and monitor, at the cost of a central component.
- **Correlation ID** propagated in headers through every hop for distributed tracing (Micrometer/OpenTelemetry auto-instruments this for Spring AMQP).
- **Streams for audit** — non-destructive, replayable, so you can rebuild read models.
- **Timeouts as events:** if payment doesn't respond in N minutes, a scheduled process emits `payment.timeout` → compensate. Never let a saga hang forever.

🧠 **Memory hook:** *Events + outbox + idempotency + DLQ per queue + compensating transactions. That's the whole pattern.*

---

### S17. How would you replay 3 days of events to rebuild a broken read model?

**With classic/quorum queues you can't** — reads are destructive; once acked, the message is gone. Say this first; it's the insight being tested.

**Options:**

1. **RabbitMQ Streams** — if audit events were published to a stream with 7-day retention, just attach a new consumer at `offset: timestamp(3 days ago)` and read forward. Non-destructive, so it doesn't affect existing consumers. **This is why you put an audit stream in the topology up front.**
   ```java
   @RabbitListener(queues = "audit.stream",
       arguments = @Argument(name = "x-stream-offset", value = "1700000000000"))
   ```
2. **Event store in the DB** (the outbox table with a longer retention) — re-publish from there via a replay job. Works even without streams.
3. **DLQ / parking lot replay** — a Shovel or an admin endpoint that moves messages from the parking lot back to the main exchange, after the bug is fixed.
4. **Snapshot + forward replay** — restore the read model from a snapshot, then replay only events after the snapshot timestamp.

**Guardrails for any replay:**
- Replay into a **separate queue/consumer** so live traffic isn't disrupted.
- **Rate-limit** the replay — 3 days of events at full speed will melt your database.
- Mark replayed messages with a header (`x-replay: true`) so consumers can skip side-effects like sending emails. *This is the detail that shows you've actually done a replay.*
- Consumers must be idempotent (again).

🧠 **Memory hook:** *You can't replay a queue. You can replay a stream or a database. Plan for replay before you need it.*

---

### S18. How do you autoscale consumers in Kubernetes based on RabbitMQ?

- **KEDA** with the `rabbitmq` scaler: scales a Deployment on `queueLength` or `messageRate`, including **scale-to-zero**.
  (KEDA = Kubernetes Event-Driven Autoscaling: an add-on that watches an external system — here the queue — and tells Kubernetes how many replicas to run. *Why not plain CPU-based HPA?* A consumer waiting on a slow DB uses little CPU even when the backlog is huge, so CPU is a poor signal; queue depth is the real one.)
  ```yaml
  triggers:
    - type: rabbitmq
      metadata:
        protocol: amqp
        queueName: orders.queue
        mode: QueueLength
        value: "100"        # target messages per replica
  ```
- Or **HPA with a custom/external metric** from the Prometheus adapter (`rabbitmq_queue_messages_ready`).

**Things that go wrong — mention these:**
- **Scaling on queue depth alone oscillates.** Add stabilization windows and use `messageRate` or a longer averaging window.
- **The downstream is the real constraint.** Scaling consumers 10× just moves the bottleneck to your database. Cap `maxReplicaCount` based on DB connection pool capacity.
- **Graceful shutdown is mandatory** — scale-down must let in-flight messages finish (`terminationGracePeriodSeconds` > container shutdown timeout, `preStop` hook).
- **Connection churn:** rapid scaling creates and destroys connections; each is expensive on the broker. Keep the scaling less twitchy.
- **Prefetch × replicas** determines how many messages are in flight; a scale-down event requeues all of them.
- For **ordered/partitioned** queues, you can't autoscale freely — replica count is tied to partition count.

*Worked example:* target `value: "100"` with 2,000 ready messages → KEDA asks for ⌈2000/100⌉ = 20 replicas (capped by `maxReplicaCount`, e.g. 12 because the DB pool allows only that many). When the queue drains it scales back down after the cooldown period, and each terminating pod finishes its in-flight messages first.

---

### S19. Producer is publishing but nothing arrives in the queue. Debug it.

**Checklist in order — this is a "do you actually operate this?" question:**

1. **Is the exchange right?** Typo in exchange name → messages go to a nonexistent exchange → the channel is **closed with a 404** (`NOT_FOUND - no exchange`). Check application logs for channel shutdown, not just "no errors."
2. **Does the routing key match a binding?** Check the exchange's bindings in the management UI, and use the **"Publish message" + bindings view**. Topic wildcard confusion (`*` vs `#`) is the most common cause.
3. **Is `mandatory` on with a ReturnsCallback?** If not, unroutable messages are **silently dropped** — turn it on and you'll instantly see the problem.
4. **Is the message actually being published?** Enable publisher confirms; are you getting acks? Is the outbox row marked SENT?
5. **Is the connection blocked?** Memory/disk alarm → publishes hang. Check `connection.blocked`.
6. **Right vhost?** Publishing to `/` while the queue is in `/orders` is a silent no-op from the producer's perspective.
7. **Permissions?** A user without `write` on the exchange gets `ACCESS_REFUSED` and a closed channel.
8. **Is a consumer draining it instantly?** Queue depth 0 doesn't mean nothing arrived — check the `publish` and `deliver` rates on the queue, not the depth.

🧠 **Memory hook:** *Silent loss = missing `mandatory`. Turn on mandatory + confirms and the mystery solves itself.*

---

### S20. Your service consumes from RabbitMQ and writes to a database. Traffic is 10× on Black Friday. What breaks first and what do you do?

**What breaks, in order:**
1. **DB connection pool** — consumers × prefetch exceed pool size → threads block waiting for connections → unacked climbs → looks like a RabbitMQ problem but isn't.
2. **DB write contention / lock waits** — hot rows, index contention.
3. **Broker memory** — queue depth grows because consumers can't keep up.
4. **Connection/channel churn** if autoscaling is aggressive.

**Preparation:**
- **Batch the writes.** Consume in batches (`BatchingRabbitTemplate` / `SimpleMessageListenerContainer` with `batchSize` + `consumerBatchEnabled`) and do a single multi-row insert per batch, acking the batch together. This is usually a 5–10× throughput win and is the answer they want.
- Size `prefetch × concurrency ≤ dbPoolSize` — write that inequality on the whiteboard.
- **Bound the queue** so a runaway backlog can't kill the broker; overflow to a DLQ you can drain later.
- **Load-test the whole path** beforehand, not just the broker.
- **Shard the queue** (consistent hash) so no single queue process is the ceiling.
- Pre-scale rather than relying on reactive autoscaling for a known event.
- Have a **degradation plan**: which message types can be shed or deferred (analytics yes, payments no)? Use separate queues per criticality so low-value traffic can't starve high-value traffic. **Priority by topology, not by `x-max-priority`.**

🧠 **Memory hook:** *`prefetch × concurrency ≤ DB pool size`. Batch the writes. Separate queues by criticality.*

---

### S21. How do you handle a message that fails validation vs one that fails because Redis was down? Same code path?

**No — and knowing the difference is the whole answer.**

| | **Permanent (non-retryable)** | **Transient (retryable)** |
|---|---|---|
| Examples | Malformed JSON, schema violation, business rule broken, referenced entity permanently absent, 400/404 from downstream | DB connection refused, timeout, 503, Redis down, deadlock, optimistic lock conflict, rate limited (429) |
| Action | **Reject → DLQ immediately** | **Retry with backoff**, then DLQ |
| Spring | `throw new AmqpRejectAndDontRequeueException(e)` | let it propagate to the retry interceptor / `ImmediateRequeueAmqpException` |

```java
@RabbitListener(queues = "orders.queue")
public void handle(OrderCreated event) {
    try {
        validator.validate(event);                 // throws ValidationException
        orderService.process(event);
    } catch (ValidationException | JsonProcessingException e) {
        throw new AmqpRejectAndDontRequeueException("Permanent failure", e);   // → DLQ
    } catch (DataAccessResourceFailureException | ResourceAccessException e) {
        throw e;                                   // → retry interceptor → backoff
    }
}
```

**Why it matters:** retrying a permanent failure wastes resources and delays good messages; DLQ-ing a transient failure loses work that would have succeeded 200 ms later. Getting this classification right is the single highest-leverage design decision in a consumer.

🧠 **Memory hook:** *Retry the environment, reject the payload.*

---

### S22. Two teams need the same events, but one is 100× slower than the other. How do you stop the slow one from affecting the fast one?

**The mistake:** both consuming from the **same queue**. A slow consumer holds messages unacked, competes for them, and now the fast team's latency depends on the slow team's throughput.

**The fix:** **one queue per consumer group.** Bind both queues to the same exchange with the same routing key. Each queue gets its own **copy** of every message, its own depth, its own DLQ, its own prefetch, its own failure domain.

```java
return new Declarables(
    ExchangeBuilder.topicExchange("orders.exchange").durable(true).build(),
    QueueBuilder.durable("orders.search.queue").withArgument("x-queue-type","quorum").build(),
    QueueBuilder.durable("orders.reporting.queue").withArgument("x-queue-type","quorum")
        .withArgument("x-max-length", 1_000_000).build(),   // slow one gets a bigger buffer
    BindingBuilder.bind(searchQueue()).to(exchange()).with("order.#"),
    BindingBuilder.bind(reportingQueue()).to(exchange()).with("order.#"));
```

**Additional points:**
- This is RabbitMQ's equivalent of Kafka consumer groups: **Kafka = one topic, N groups with independent offsets. RabbitMQ = one exchange, N queues.** Make that comparison — it lands well.
- Bound the slow queue and give it an overflow strategy so its backlog can't threaten the broker.
- If the slow consumer needs replay or falls days behind, put it on a **stream** instead of a queue.
- Cost note: N queues = N copies on disk. A stream stores **one** copy read by N consumers — better for wide fan-out.

🧠 **Memory hook:** *One queue per consumer group. Never share a queue between teams.*

---

## Part 11 — Design Patterns You Should Be Able to Draw

### 1. Competing Consumers (Work Queue)
One queue, N consumers, round-robin distribution. Scales horizontally. **Requires:** prefetch tuning for fairness, idempotency for redelivery. *Use for:* task processing, image resizing, report generation.

*How it works:* the producer sends to one queue; the broker hands each message to exactly **one** of the connected consumers in turn, so adding consumers raises throughput without code changes. *Example:* 10,000 "resize image" jobs — start 5 worker pods on `images.queue`; if the backlog grows, start 5 more. *Pitfall:* order is no longer guaranteed across consumers (see Q5).

```
Producer ──► [ images.queue ] ──┬─► Worker 1
                                ├─► Worker 2   (each message goes to ONE worker)
                                └─► Worker 3
```

### 2. Publish-Subscribe (Fan-out)
One exchange, one queue per subscriber. Each subscriber gets its own copy and its own failure domain. *Use for:* domain events consumed by multiple services.

*How it works:* a fanout (or topic) exchange copies each message into **every** bound queue. `order.created` → `email.queue`, `inventory.queue`, `analytics.queue`; each service consumes its own queue at its own speed, and if analytics is down only its queue backs up. *Contrast with Pattern 1:* there, consumers *share* a queue (split the work); here, each consumer group has its *own* queue (everyone gets everything).

```
Producer ──► [ orders.exchange (fanout/topic) ] ──┬─► email.queue ──► Email svc
                                                    ├─► inventory.queue ──► Inventory svc
                                                    └─► analytics.queue ──► Analytics svc
```

### 3. Routing / Selective Consumption
Topic or direct exchange; consumers bind only to what they care about (`order.*.premium`). Filtering happens **in the broker**, so consumers don't waste cycles discarding messages. *Better than* consuming everything and filtering in code.

*Example:* the producer publishes `order.created.premium` and `order.created.standard`. The VIP-support service binds its queue with `order.*.premium` and only ever receives premium orders; billing binds `order.created.*` and receives both. Changing who gets what is a binding change, not a code change.

### 4. Request-Reply (RPC)
`replyTo` + `correlationId`, ideally Direct Reply-To. **Always timeout.** Prefer async (`AsyncRabbitTemplate`) over blocking. *Question whether you need it at all.*

*Flow:* client publishes a request carrying `replyTo=<reply queue>` and `correlationId=abc`; the server processes it and publishes the answer to `replyTo` with the same `correlationId`; the client matches the reply to its waiting call by that ID. Full code and pitfalls are in S12.

### 5. Dead Letter + Retry Ladder + Parking Lot
`main → dlx → retry.5s/30s/5m → main → parking lot`. Non-blocking backoff, bounded retries, human triage at the end. **The single most important production pattern.**

*Plain steps:* (1) consumer fails and rejects without requeue → (2) the DLX sends the message to `retry.5s`, a queue with **no consumers** and a 5-second TTL → (3) after 5 s the message expires and is dead-lettered *back to the main exchange* → (4) the consumer tries again; if it fails again it goes to `retry.30s`, then `retry.5m` (counted via the `x-death` header) → (5) after the last rung it goes to the parking lot for a human. The waiting happens inside the broker, so no consumer thread is blocked. Details in Q20/Q21.

### 6. Transactional Outbox (+ Inbox)
DB and message publish made consistent without 2PC. Outbox for the write side, inbox (dedup table) for the read side. *See S4.*

*In one breath:* write the business row **and** an "event to send" row in the same DB transaction; a separate relay publishes unsent rows to RabbitMQ and marks them sent. The consumer records each processed `messageId` in an inbox table (in the same transaction as its own work) so duplicates are skipped. (2PC = two-phase commit, the distributed-transaction protocol this avoids.)

### 7. Saga / Choreography with Compensating Transactions
Distributed transactions via events and compensations. No 2PC. Add timeouts-as-events so sagas can't hang. *See S16.*

*What it means:* a business process spanning services (order → payment → inventory) is broken into local transactions, each publishing an event on success. If a later step fails, earlier steps are **undone with a compensating action** rather than a rollback — e.g. inventory fails → publish `payment.refund.requested`. *Choreography* = services react to each other's events with no central controller; *orchestration* = a coordinator tells each service what to do next.

### 8. Claim Check
Large payload in blob storage, small reference in the message. Keeps the broker fast. *See S11.*

*Example:* instead of publishing a 50 MB PDF, upload it to S3 and publish `{"documentId":"42","location":"s3://bucket/42.pdf"}`; the consumer downloads it when it processes the message.

### 9. Consistent-Hash Sharding
Ordering per key with parallelism across keys. *See S5.*

*How:* hash the key (e.g. `customerId`) to pick one of N queues, each with one consumer. Customer 17 always lands in queue 5, so their events are processed in order, while customers in other queues are handled in parallel. Remember resharding (changing N) temporarily breaks the key→queue mapping.

### 10. Priority by Topology
Instead of `x-max-priority`, use **separate queues per criticality** (`orders.critical.queue`, `orders.bulk.queue`) with separate consumer pools. Simpler, more predictable, and low-priority traffic can never starve high-priority traffic. *This is the mature answer when someone asks about priority.*

### 11. Circuit Breaker + Consumer Pause
Stop consuming when downstream is down; let the queue buffer. *See S6.*

*How:* a circuit breaker (e.g. Resilience4j) counts failures calling the downstream; when it "opens", you call `registry.getListenerContainer("id").stop()`, messages pile up safely in the queue, and a health probe restarts the container once the downstream recovers.

### 12. Idempotent Receiver
Dedup table with a unique constraint, in the same transaction as the business write. *See S3.*

*Why:* "at-least-once" delivery means duplicates happen. The receiver tries to insert the `messageId` into a `processed_messages` table with a unique constraint; if the insert fails (already there) it skips the work, otherwise it does the work — both in one DB transaction so they succeed or fail together.

---

## Part 12 — RabbitMQ vs Kafka vs ActiveMQ vs SQS

| | **RabbitMQ** | **Kafka** | **ActiveMQ Artemis** | **AWS SQS** |
|---|---|---|---|---|
| Model | **Smart broker**, dumb consumer | **Dumb broker**, smart consumer | Broker (JMS) | Managed queue |
| Protocol | AMQP 0-9-1 (+MQTT, STOMP, streams) | Custom binary | JMS/AMQP/STOMP/MQTT | HTTP API |
| Routing | **Rich** (exchanges, wildcards, headers) | Topic + partition key only | Selectors, JMS | Minimal (FIFO/standard) |
| Retention | Delete on ack (queues) / retention (streams) | **Time/size retention, replay** | Delete on ack | 14 days max |
| Ordering | Per queue | **Per partition** | Per queue/group | FIFO queues only |
| Throughput | High (100k+/s) | **Very high (millions/s)** | High | High (managed) |
| Consumers | Push-based | **Pull-based** | Push/pull | Poll-based |
| Replay | Only with streams | **Native** | No | No |
| Per-message TTL, priority, DLX | ✅ Rich | ❌ Mostly not | ✅ | Limited |
| Ops burden | Moderate (Erlang) | High (ZK/KRaft, partitions) | Moderate | **None** |

**How to answer "RabbitMQ or Kafka?"** — never as a winner/loser:

> **RabbitMQ when the message is a *task*.** Complex routing, per-message TTL/priority/DLQ, request-reply, work distribution, low-latency point-to-point, moderate volume. It's a *message broker*.
>
> **Kafka when the message is a *fact*.** Event sourcing, replay, stream processing, multiple independent consumer groups over the same high-volume log, retention as a feature. It's a *distributed log*.
>
> Concretely: "process this payment" → RabbitMQ. "user clicked a button" (10M/day, consumed by analytics, ML, and audit) → Kafka. And plenty of architectures run both.

🧠 **Memory hook:** *RabbitMQ delivers work. Kafka stores history.*

**Terms in the table, decoded:**
- **Smart broker / dumb consumer (RabbitMQ):** the broker does the routing, tracks which messages each consumer has acked, and **pushes** messages out; consumers just process what they are given. **Dumb broker / smart consumer (Kafka):** the broker only appends to a log; each consumer **pulls** and remembers its own *offset* (position), so it can re-read old data.
- **Partition (Kafka):** a topic is split into ordered logs; ordering is guaranteed only inside one partition, which is why Kafka's ordering is "per partition" (compare consistent-hash sharding in RabbitMQ).
- **JMS:** the Java Messaging Service API (queues/topics, selectors) that ActiveMQ implements — convenient for Java-only shops, less so for polyglot.
- **SQS:** fully managed by AWS (no servers to run), but routing is minimal; fan-out is usually done by pairing it with SNS.
- **Push vs pull consequence:** push gives low latency and prefetch-based flow control; pull lets slow consumers fall behind without the broker caring and enables easy replay.

---

## Part 13 — Rapid-Fire Cheat Sheet

**Numbers to know**
- Ports: **5672** AMQP, **5671** AMQPS, **15672** management, **15692** Prometheus, **4369** epmd, **25672** Erlang dist, **61613** STOMP, **1883** MQTT
- `vm_memory_high_watermark` default **0.4**
- `disk_free_limit` default **50 MB** (set ≥ 1–2× RAM)
- Heartbeat default **60 s**
- `consumer_timeout` default **30 min** (3.8.15+)
- Quorum queue replicas: **odd — 3 or 5**
- Practical `x-max-priority`: **≤ 10**
- Recommended max message size: **≤ 128 KB**
- AMQP `deliveryMode`: **1 = transient, 2 = persistent**

**One-line answers**
- *Exchange types?* Direct, Fanout, Topic, Headers (+ default).
- *`*` vs `#`?* One word vs zero-or-more words.
- *Durability requires?* Durable exchange + durable queue + persistent message.
- *Delivery guarantee?* **At-least-once.** Exactly-once = at-least-once + idempotent consumer.
- *4 dead-letter reasons?* Rejected, TTL expired, max-length exceeded, delivery-limit exceeded.
- *Confirms vs returns?* Confirms = broker got it. Returns = no queue wanted it.
- *Confirms vs transactions?* Confirms — transactions are ~250× slower.
- *Prefetch 0?* Unlimited — never do it.
- *Poison message fix?* `default-requeue-rejected: false` + DLX + `x-delivery-limit`.
- *Is a channel thread-safe?* **No.** One channel per thread.
- *Can you change queue arguments?* **No** — immutable. Use policies or a new queue.
- *Where does message data live in a cluster?* On the queue's node (classic) or its replica set (quorum). Metadata is everywhere.
- *Partition handling?* `pause_minority` with an odd node count.
- *Ordering with parallelism?* Consistent-hash exchange + one consumer per queue.
- *Big payloads?* Claim check.
- *DB + message atomicity?* Transactional outbox.
- *Replay?* Streams or an event table — not queues.
- *SMLC vs DMLC?* Elastic-but-heavier vs lean-but-fixed.
- *Fatal-by-default exception?* `MessageConversionException` → straight to DLQ via `ConditionalRejectingErrorHandler`.
- *Force DLQ / force retry?* `AmqpRejectAndDontRequeueException` / `ImmediateRequeueAmqpException`.

**The five loss gates:** Publish → Route → Store → Deliver → Process.

**The three-legged stool:** Exchange, Queue, Message — all durable/persistent.

---

## Part 14 — Common Pitfalls Interviewers Love

| # | Pitfall | Why it hurts | Fix |
|---|---|---|---|
| 1 | Connection or channel per message | TCP/TLS/auth churn, FD exhaustion | `CachingConnectionFactory`, one channel per thread |
| 2 | Sharing a connection between publishers and consumers | Memory alarm blocks publishing → consumers stall → **deadlock** | Separate connections |
| 3 | `requeue=true` on every failure | Infinite hot loop, CPU pinned | Classify exceptions; DLX; `x-delivery-limit` |
| 4 | No DLX configured | Rejected messages silently vanish | Always declare a DLX + DLQ + alerting |
| 5 | No `mandatory` / no confirms | Silent message loss you discover in a reconciliation report | `publisher-confirm-type: correlated`, `mandatory: true`, alternate exchange |
| 6 | Unbounded queues | Broker memory alarm takes the whole system down | `x-max-length` + `x-overflow: reject-publish-dlx` |
| 7 | prefetch = 0 or very high | One consumer hoards, others idle, big redelivery bursts | Tune to processing time |
| 8 | Assuming exactly-once | Double charges, duplicate emails | Idempotent consumers with a dedup table |
| 9 | Assuming global ordering | Subtle data corruption under retry | Consistent-hash + single consumer per partition |
| 10 | Changing queue arguments in place | `PRECONDITION_FAILED 406`, listeners won't start | Policies, or new queue + shovel |
| 11 | Publishing inside `@Transactional` | Phantom events on rollback | Transactional outbox |
| 12 | Java serialization converter | Security hole + version coupling | `Jackson2JsonMessageConverter` + trusted packages |
| 13 | TTL on the DLQ | Silently discards all your failure evidence | No TTL on DLQ; alert on depth |
| 14 | Large messages through the broker | Memory alarms, head-of-line blocking | Claim check |
| 15 | Long-running work without timeouts | Stuck consumers, unacked pileup, `consumer_timeout` channel closures | Timeouts on every I/O call |
| 16 | Deep queues treated as normal | Rabbit is not a database; latency and memory degrade | Design for empty queues; use a stream if you need retention |
| 17 | Producer deployed before consumer | Consumers crash on unknown schema | **Consumers first**, additive changes only |
| 18 | No graceful shutdown | In-flight messages requeued, duplicate work every deploy | `server.shutdown: graceful`, container shutdown timeout, `preStop` |
| 19 | Even number of cluster nodes | No majority possible on failure | Odd counts: 3 or 5 |
| 20 | Polling the management HTTP API for metrics | Expensive, degrades the broker | `rabbitmq_prometheus` plugin |

---

## Closing: How to Frame Answers at 9 Years' Experience

For every question, layer your answer in this order:

1. **The mechanism** — what RabbitMQ actually does (30 seconds).
2. **The trade-off** — what it costs, what it doesn't guarantee.
3. **The production story** — "on my last project we hit X, and we solved it with Y."
4. **The failure mode** — "the thing to watch out for is…"

Three sentences that make you sound senior, used at the right moment:

> *"RabbitMQ gives at-least-once delivery, so we designed the consumers to be idempotent and got effectively-once processing."*

> *"We couldn't make the DB write and the publish atomic, so we used a transactional outbox."*

> *"We needed ordering per customer and throughput across customers, so we sharded with a consistent-hash exchange and ran one consumer per queue."*

Good luck. 🐇
