# 06 — Kafka Consumers: Poll Loop, Offsets & Commits
> **Phase 2 — Producers & Consumers (Core Mechanics)**  
> 📁 `notes/06-consumers.md`

---

## 📖 1. Concept Explanation

### What is a Kafka Consumer?

A **Kafka Consumer** is a client that **reads (polls) records from Kafka topics**. Unlike traditional message queues where the broker pushes messages, Kafka consumers **pull** messages at their own pace — giving them complete control over:

- **What** to read (topic, partition, offset)
- **When** to read (polling frequency)
- **How fast** to read (batch size, processing speed)
- **What to acknowledge** (offset commit strategy)

The consumer's most critical responsibility is **offset management** — tracking which messages it has successfully processed.

---

### Consumer Responsibilities

```
┌────────────────────────────────────────────────────────────────┐
│                    CONSUMER RESPONSIBILITIES                   │
│                                                                │
│  1. Poll       Fetch batches of records from broker            │
│  2. Deserialize bytes → Java objects                           │
│  3. Process    Execute business logic                          │
│  4. Commit     Tell Kafka: "I'm done with these offsets"       │
│  5. Heartbeat  Tell Kafka: "I'm still alive"                   │
└────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ 2. Architecture / Flow — The Poll Loop

### The Core Consumer Loop

The entire Kafka consumer is built around a **single-threaded poll loop**:

```
Consumer starts
      │
      ▼
  subscribe("order-events")
  [joins consumer group, gets partitions assigned]
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                      POLL LOOP                              │
│                                                             │
│   while (true) {                                            │
│       records = consumer.poll(Duration.ofMillis(1000))      │
│                                                             │
│       for (record : records) {                              │
│           // Process record                                 │
│           processOrder(record.value())                      │
│       }                                                     │
│                                                             │
│       consumer.commitSync()  // or auto-commit              │
│   }                                                         │
└────────────────────────────────┬────────────────────────────┘
                                 │
                                 ▼
                    Send heartbeat (background thread)
                    [proves consumer is alive to group coordinator]
```

### What Happens Inside `poll()`

```
poll(timeout=1000ms) does:

  1. Check if heartbeat needs to be sent → send if needed
  2. Check if offset commit needs to be sent → send if needed
  3. Send FETCH request to partition leader brokers
  4. Wait up to `timeout` ms for response
  5. Return fetched records (up to max.poll.records)
  6. If no records → return empty collection after timeout

Key insight: poll() is NOT just "fetch data"
  It also drives heartbeating and offset commits internally.
  If you stop calling poll() → consumer appears dead to the group
  → triggers a rebalance (other consumers take over partitions)
```

---

### Full Consumer Lifecycle

```
Consumer Instance Starts
         │
         ▼
  Find Group Coordinator
  (a broker responsible for this consumer group)
         │
         ▼
  JoinGroup Request
  (tells coordinator: "I want to join group X for topic Y")
         │
         ▼
  Group Coordinator picks a Group Leader
  (the first consumer to join becomes leader)
         │
         ▼
  Group Leader runs Partition Assignment
  (assigns partitions to each consumer instance)
         │
         ▼
  SyncGroup — All consumers receive their partition assignments
         │
         ▼
  Each consumer starts polling its assigned partitions
         │
         ▼
  ┌──────── POLL LOOP ────────────────┐
  │  poll() → process → commit        │ ← steady state
  └───────────────────────────────────┘
         │
    (consumer leaves or crashes)
         │
         ▼
  Rebalance triggered
  → Group coordinator redistributes partitions
  → Remaining consumers get new assignments
```

---

## ⚙️ 3. How It Works Internally

### Offset Commit Strategies

This is the most important concept for consumer correctness.

**Committed offset** = "I have successfully processed everything BEFORE this offset"

```
Partition 0 messages:
  [off:0] [off:1] [off:2] [off:3] [off:4] [off:5]
                              ↑
                  Committed offset = 3
                  Meaning: "I'm done with 0,1,2 — next fetch starts at 3"
```

---

#### Strategy 1: Auto Commit (enable.auto.commit=true)

```
Kafka auto-commits the current offset every auto.commit.interval.ms (default 5s)

Timeline:
  T=0s   poll() → process records 0,1,2,3
  T=5s   AUTO COMMIT → offset=4 committed (background)
  T=6s   poll() → process records 4,5,6
  T=7s   CRASH before commit
  T=?    Consumer restarts → resumes from offset 4 (last committed)
         Records 4,5,6 are RE-PROCESSED → DUPLICATES

Risk: At-least-once delivery (records may be reprocessed after crash)
```

---

#### Strategy 2: Manual Sync Commit (commitSync)

```java
// After processing, commit synchronously (blocks until broker confirms)
consumer.commitSync();

Timeline:
  poll() → [off:0, off:1, off:2]
  process all three records
  commitSync() → BLOCKS until broker ACKs offset=3
  poll() → [off:3, off:4, off:5]
  process all three
  commitSync() → BLOCKS until broker ACKs offset=6

Guarantee: Only commits after successful processing
Risk: Blocking call reduces throughput
```

---

#### Strategy 3: Manual Async Commit (commitAsync)

```java
// Non-blocking commit — doesn't wait for broker ACK
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("Commit failed: {}", exception.getMessage());
    }
});

Timeline:
  poll() → process → commitAsync() → [continues immediately]
                                       [ACK arrives later]

Advantage: Non-blocking, better throughput
Risk: If commit fails and consumer crashes → reprocessing
Best practice: use commitAsync in loop, commitSync on shutdown
```

---

#### Strategy 4: Commit Specific Offsets

```java
// Commit exact offset per partition — maximum control
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
offsets.put(
    new TopicPartition("order-events", 0),
    new OffsetAndMetadata(record.offset() + 1)   // +1: "next offset to fetch"
);
consumer.commitSync(offsets);
```

> ⚠️ Always commit `offset + 1` — not the offset you just processed. The committed offset is the **next offset to fetch**, not the last processed.

---

### Heartbeat and Session Timeout

```
Two separate timers control consumer liveness:

1. heartbeat.interval.ms (default 3000ms)
   → How often consumer sends heartbeat to group coordinator
   → Should be 1/3 of session.timeout.ms

2. session.timeout.ms (default 45000ms)
   → If coordinator doesn't receive heartbeat within this time
   → Consumer is declared DEAD → rebalance triggered

3. max.poll.interval.ms (default 300000ms = 5 min)
   → Max time between two poll() calls
   → If processing takes longer than this → consumer declared dead
   → Most common cause of unexpected rebalances!

Rebalance trigger causes:
  - Consumer crashes (session.timeout.ms exceeded)
  - Consumer processing too slow (max.poll.interval.ms exceeded)
  - Consumer joins the group (new instance deployed)
  - Consumer leaves cleanly (graceful shutdown)
  - Partition count changes
```

```
┌─────────────────────────────────────────────────────────┐
│          Heartbeat Thread (background)                  │
│  Sends heartbeat every heartbeat.interval.ms            │
│  Independent of poll() call                             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│          Main Thread (your code)                        │
│  Must call poll() within max.poll.interval.ms           │
│  If processing is slow:                                 │
│    - Reduce max.poll.records (process fewer at a time)  │
│    - Increase max.poll.interval.ms                      │
│    - Move processing to async (careful with ordering)   │
└─────────────────────────────────────────────────────────┘
```

---

### Fetch Internals

```
Consumer sends FETCH request to broker with:
  fetch.min.bytes      (default 1 byte)
    → Broker waits until this many bytes available before responding
    → Increase for throughput (e.g., 1MB) — fewer, larger responses

  fetch.max.wait.ms    (default 500ms)
    → Max time broker holds the FETCH request before responding
    → Even if fetch.min.bytes not reached

  max.partition.fetch.bytes (default 1MB)
    → Max bytes fetched per partition per request

  max.poll.records     (default 500)
    → Max number of records returned per poll() call
```

---

## 💻 4. Code Examples (Java + Spring Boot)

### `application.yml` — Production Consumer Config

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: order-processor
      auto-offset-reset: earliest       # Start from beginning for new groups
      enable-auto-commit: false         # Manual commit for correctness
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      max-poll-records: 50              # Process 50 records per poll
      properties:
        spring.json.trusted.packages: "com.example.kafka.model"
        fetch.min.bytes: 1024           # 1KB minimum before broker responds
        fetch.max.wait.ms: 500
        session.timeout.ms: 45000
        heartbeat.interval.ms: 15000
        max.poll.interval.ms: 300000
    listener:
      ack-mode: MANUAL_IMMEDIATE        # Manual offset commit
      concurrency: 3                    # 3 consumer threads (for 3+ partitions)
      poll-timeout: 3000
```

---

### Basic Consumer — Auto Commit

```java
package com.example.kafka.consumer;

import com.example.kafka.model.OrderEvent;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

@Service
public class OrderConsumer {

    // Spring manages offset commit automatically (auto-commit or after listener returns)
    @KafkaListener(
        topics = "order-events",
        groupId = "order-processor"
    )
    public void consume(ConsumerRecord<String, OrderEvent> record) {
        System.out.printf(
            "Processing → partition=%d offset=%d orderId=%s status=%s%n",
            record.partition(),
            record.offset(),
            record.key(),
            record.value().getStatus()
        );

        processOrder(record.value());
        // Spring auto-commits offset AFTER this method returns (AckMode.BATCH default)
    }

    private void processOrder(OrderEvent event) {
        // Business logic here
        System.out.println("Order processed: " + event.getOrderId());
    }
}
```

---

### Manual Commit Consumer — Maximum Control

```java
package com.example.kafka.consumer;

import com.example.kafka.model.OrderEvent;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Service;

@Service
public class ManualCommitOrderConsumer {

    // AckMode.MANUAL_IMMEDIATE in application.yml
    @KafkaListener(topics = "order-events", groupId = "order-processor")
    public void consume(
            ConsumerRecord<String, OrderEvent> record,
            Acknowledgment ack) {       // ← Injected by Spring Kafka

        try {
            System.out.printf("Processing offset=%d key=%s%n",
                record.offset(), record.key());

            processOrder(record.value());

            // Only commit AFTER successful processing
            ack.acknowledge();
            System.out.println("Offset committed: " + record.offset());

        } catch (Exception e) {
            System.err.println("Processing failed, NOT committing offset: "
                + e.getMessage());
            // Don't call ack.acknowledge() → offset not committed
            // → On restart, this message will be reprocessed
            // Implement DLQ (dead letter queue) for repeated failures
        }
    }

    private void processOrder(OrderEvent event) {
        // May throw exception on failure
    }
}
```

---

### Batch Consumer — Process Multiple Records Together

```java
package com.example.kafka.consumer;

import com.example.kafka.model.OrderEvent;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class BatchOrderConsumer {

    // Batch listener receives a list of records per poll
    @KafkaListener(
        topics = "order-events",
        groupId = "batch-processor",
        batch = "true"                // Enable batch mode
    )
    public void consumeBatch(
            List<ConsumerRecord<String, OrderEvent>> records,
            Acknowledgment ack) {

        System.out.printf("Batch received: %d records%n", records.size());

        // Process all records in the batch
        List<OrderEvent> events = records.stream()
            .map(ConsumerRecord::value)
            .toList();

        processBatch(events);

        // Commit once for the entire batch
        ack.acknowledge();
        System.out.printf("Batch committed. Offsets: %d → %d%n",
            records.get(0).offset(),
            records.get(records.size() - 1).offset());
    }

    private void processBatch(List<OrderEvent> events) {
        // e.g., bulk insert to database, batch API call
        System.out.println("Bulk processing " + events.size() + " orders");
    }
}
```

**`application.yml` addition for batch:**
```yaml
spring:
  kafka:
    listener:
      type: batch    # Enable batch listener mode
```

---

### Seek to Specific Offset on Startup

```java
package com.example.kafka.consumer;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.TopicPartition;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.listener.ConsumerSeekAware;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
public class ReplayConsumer implements ConsumerSeekAware {

    private ConsumerSeekCallback seekCallback;

    @Override
    public void registerSeekCallback(ConsumerSeekCallback callback) {
        this.seekCallback = callback;
    }

    // Called when partitions are assigned — seek to desired position
    @Override
    public void onPartitionsAssigned(
            Map<TopicPartition, Long> assignments,
            ConsumerSeekCallback callback) {

        // Option 1: Seek to beginning of all assigned partitions
        callback.seekToBeginning(assignments.keySet());

        // Option 2: Seek to specific offset
        // assignments.keySet().forEach(tp ->
        //     callback.seek(tp.topic(), tp.partition(), 100));

        // Option 3: Seek to timestamp (replay last 1 hour)
        // long oneHourAgo = System.currentTimeMillis() - (60 * 60 * 1000);
        // callback.seekToTimestamp(tp.topic(), tp.partition(), oneHourAgo);
    }

    @KafkaListener(topics = "order-events", groupId = "replay-group")
    public void consume(ConsumerRecord<String, String> record) {
        System.out.printf("Replaying: offset=%d value=%s%n",
            record.offset(), record.value());
    }
}
```

---

### Programmatic Consumer Config (Full Control)

```java
package com.example.kafka.config;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.ConsumerFactory;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ContainerProperties;
import org.springframework.kafka.support.serializer.JsonDeserializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();

        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processor");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        // Fetch tuning
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 50);
        props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG,  1024);
        props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);

        // Liveness
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG,    45_000);
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 15_000);
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG,  300_000);

        // Deserializers
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
            StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
            JsonDeserializer.class);
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.kafka.model");

        return new DefaultKafkaConsumerFactory<>(props);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object>
            kafkaListenerContainerFactory() {

        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();

        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);   // 3 threads = 3 partitions processed in parallel
        factory.getContainerProperties().setAckMode(
            ContainerProperties.AckMode.MANUAL_IMMEDIATE
        );

        return factory;
    }
}
```

---

## ⚡ 5. Key Configurations Reference

| Config | Default | Recommended (Prod) | Impact |
|--------|---------|-------------------|--------|
| `enable.auto.commit` | `true` | `false` | Correctness |
| `auto.commit.interval.ms` | `5000` | — (disable auto) | Commit frequency |
| `auto.offset.reset` | `latest` | `earliest` (new groups) | Where to start |
| `max.poll.records` | `500` | `50–200` | Batch processing size |
| `max.poll.interval.ms` | `300000` | Increase if processing is slow | Rebalance sensitivity |
| `session.timeout.ms` | `45000` | `45000` | Dead consumer detection |
| `heartbeat.interval.ms` | `3000` | `15000` (1/3 of session) | Heartbeat frequency |
| `fetch.min.bytes` | `1` | `1024` (1KB) | Throughput |
| `fetch.max.wait.ms` | `500` | `500` | Latency ceiling |
| `max.partition.fetch.bytes` | `1MB` | `1MB` | Per-partition fetch limit |

---

## 🔄 6. Spring Kafka AckMode Reference

| AckMode | Behavior | Use When |
|---------|----------|---------|
| `RECORD` | Commits after each record processed | Simple, slow, safest |
| `BATCH` | Commits after each poll batch (default) | Good balance |
| `TIME` | Commits every `ackTime` ms | Time-based commit |
| `COUNT` | Commits every N records | Count-based commit |
| `MANUAL` | Commit when `ack.acknowledge()` called (deferred) | Full control |
| `MANUAL_IMMEDIATE` | Commits immediately when `ack.acknowledge()` called | Full control, immediate |

---

## 🔄 7. Real-World Scenarios

### Scenario: Slow Consumer Causing Rebalances

```
Problem:
  Consumer processes each record in 2 seconds
  max.poll.records = 500
  max.poll.interval.ms = 300000 (5 min)
  
  Poll returns 500 records
  Processing 500 × 2s = 1000 seconds
  But max.poll.interval = 300 seconds → REBALANCE triggered mid-processing!

Fix options:
  Option 1: Reduce max.poll.records
    max.poll.records = 10
    10 × 2s = 20s << 300s ✅

  Option 2: Increase max.poll.interval.ms
    max.poll.interval.ms = 1200000 (20 min)
    But: slower dead consumer detection

  Option 3: Speed up processing
    Async I/O, batching DB calls, caching
    Best long-term solution
```

---

### Scenario: At-Least-Once vs At-Most-Once Commit Timing

```
AT-MOST-ONCE (commit before processing):
  poll() → commit offset → process record
  If crash after commit but before processing → message LOST
  Use case: metrics, logs where loss is acceptable

AT-LEAST-ONCE (commit after processing):
  poll() → process record → commit offset
  If crash after processing but before commit → message REPROCESSED
  Use case: most business systems (make processing idempotent)

EXACTLY-ONCE (requires transactions):
  Use Kafka transactions (consume-transform-produce pattern)
  Or: store processed offset in same DB transaction as business data
```

---

## ⚠️ 8. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| `enable.auto.commit=true` for critical data | Commits before processing completes → data loss on crash | Use `MANUAL_IMMEDIATE` AckMode |
| `max.poll.interval.ms` too low | Rebalances during slow processing | Reduce `max.poll.records` or increase interval |
| Committing `offset` instead of `offset+1` | Re-reads the last processed message on restart | Always commit `record.offset() + 1` |
| Ignoring exceptions in listener | Offset commits even on failed processing | Catch exceptions, decide: retry or DLQ |
| Multi-threaded processing in listener | Offsets committed out of order → gap in commits | Use Spring's concurrency (one thread per partition) |
| Not calling `ack.acknowledge()` | Consumer eventually stalls (back-pressure fills) | Always ACK in finally block or error handler |
| Consumer group ID typo / mismatch | Creates a new consumer group, re-reads from beginning | Centralise group IDs in constants |

---

## 🎯 9. Interview Questions

**Q1. What is the Kafka consumer poll loop and why must it be called regularly?**
> The poll loop is the central mechanism by which a Kafka consumer fetches records and maintains liveness. `poll()` does three things: fetches records from brokers, sends heartbeats to the group coordinator, and commits offsets if auto-commit is enabled. If `poll()` isn't called within `max.poll.interval.ms`, the coordinator assumes the consumer is dead and triggers a rebalance.

**Q2. What is the difference between `session.timeout.ms` and `max.poll.interval.ms`?**
> `session.timeout.ms` controls how long the group coordinator waits for a **heartbeat** before declaring a consumer dead. The heartbeat thread runs in the background. `max.poll.interval.ms` controls the max time between two **`poll()` calls** — if processing is so slow that the main thread can't call `poll()` in time, the consumer is declared dead. Two separate failure modes.

**Q3. What does committing an offset mean? Why do you commit `offset + 1` and not `offset`?**
> Committing an offset tells Kafka: "The next record I want to fetch starts at this offset." So if you processed record at offset 5, you commit offset 6 — meaning "give me offset 6 next time." If you commit offset 5, you'd re-read offset 5 on restart.

**Q4. What is at-least-once delivery and how do you achieve it?**
> At-least-once delivery means every message is processed at minimum once, but may be reprocessed on failure. It's achieved by committing offsets only AFTER successfully processing records (`enable.auto.commit=false`, `AckMode.MANUAL_IMMEDIATE`). On crash/restart, uncommitted offsets are reprocessed. To handle duplicates, make consumers idempotent (e.g., use upserts in DB, check if already processed).

**Q5. What causes a consumer group rebalance?**
> A rebalance is triggered when the group membership changes: a consumer joins (new instance), a consumer leaves cleanly (shutdown), a consumer crashes (session timeout exceeded), processing takes too long (`max.poll.interval.ms` exceeded), or partition count changes on a subscribed topic. During rebalance, all consumers in the group stop consuming (stop-the-world).

**Q6. What is the difference between `AckMode.BATCH` and `AckMode.MANUAL_IMMEDIATE`?**
> `BATCH` commits after each `poll()` batch completes — Spring commits automatically after all records in the batch are processed by the listener. `MANUAL_IMMEDIATE` requires the developer to call `ack.acknowledge()` explicitly and commits immediately when called, giving full per-record control.

---

## 🧠 10. Scenario-Based Interview Problems

### Scenario 1: "Consumer Keeps Reprocessing the Same Messages"
> **Problem:** After deploying a new consumer version, it constantly reprocesses the same batch of 50 records. Logs show the consumer starts, processes 50 records, commits, then restarts and processes the same 50 again. What's wrong?

**Answer:**
```
Likely causes:

1. Exception thrown AFTER processing but inside the listener
   → Spring catches it, seeks back to last committed offset
   → Re-polls the same records
   Fix: Check for exceptions post-processing (DB save, downstream call)

2. AckMode mismatch
   → Using MANUAL but not calling ack.acknowledge()
   → Offset never advances
   Fix: Ensure ack.acknowledge() is always called (in try-finally)

3. Consumer group ID different on restart
   → New group ID = starts from auto.offset.reset = earliest
   Fix: Centralise group ID in config, don't compute it dynamically

4. enable.auto.commit=true but ACK mode is MANUAL
   → Conflicting config, unpredictable behavior
   Fix: align enable.auto.commit=false with MANUAL ack mode
```

---

### Scenario 2: "Exactly-Once Order Processing with Database"
> **Problem:** You're processing payment events from Kafka and saving to a PostgreSQL database. You need to guarantee exactly-once — no duplicate payments. `acks=all` and `enable.idempotence=true` are set on the producer. Is that enough?

**Answer:**
```
No — producer idempotence only covers producer → broker.
Consumer side still has at-least-once (reprocessing after crash).

Solutions:

Option A: Idempotent Consumer (recommended)
  Use payment event's unique ID as DB primary key or idempotency key
  INSERT INTO payments ON CONFLICT (event_id) DO NOTHING
  → Duplicate processing = harmless duplicate insert attempt

Option B: Offset in same DB transaction
  BEGIN TRANSACTION
    process payment
    INSERT INTO payments VALUES (...)
    INSERT INTO kafka_offsets (group, topic, partition, offset) 
      VALUES ('billing', 'payments', 2, 1043)
      ON CONFLICT DO UPDATE SET offset = 1043
  COMMIT
  On restart: read offset from DB, seek consumer to that offset
  → Transactional consistency between DB state and Kafka offset

Option C: Kafka Transactions (exactly-once for Kafka-to-Kafka)
  Works when output is another Kafka topic, not an external DB
```

---

### Scenario 3: "Consumer Lag is Growing — How to Fix?"
> **Problem:** Your consumer group for `payment-events` has a lag of 2 million records and growing. The topic has 6 partitions. You have 3 consumer instances running. How do you reduce lag?

**Answer:**
```
Current state: 3 instances × (up to 2 partitions each) = all 6 covered
Lag is growing → consumers can't keep up with producers

Step 1: Identify the bottleneck
  - Is processing slow? (DB calls, external API)
  - Is poll() rate slow? (max.poll.records too low)
  - Is it CPU-bound? Memory-bound?
  kafka-consumer-groups.sh --describe → check per-partition lag

Step 2: Scale consumers
  Add 3 more instances → 6 total (1 per partition, max parallelism)
  Can't exceed partition count (extra consumers sit idle)

Step 3: If still lagging → increase partitions
  Increase topic to 12 partitions, scale to 12 consumer instances
  Warning: key ordering breaks for existing keys

Step 4: Optimize processing
  - Batch DB writes (one INSERT for 50 records vs 50 INSERTs)
  - Async I/O for downstream calls
  - Increase max.poll.records (process more per poll cycle)
  - Use in-memory cache to avoid repeated lookups

Step 5: Emergency — skip to latest (if lag is unrecoverable)
  kafka-consumer-groups.sh --reset-offsets --to-latest
  Warning: messages between now and skip point are lost
  Only acceptable for non-critical data (metrics, logs)
```

---

## 📝 11. Quick Revision Summary

```
✅ Consumer = pull-based. Controls its own pace. Tracks own offset.

✅ Poll loop: must be called regularly or consumer is declared dead
   poll() = fetch records + send heartbeat + maybe commit offsets

✅ Offset commit strategies:
   AUTO      → simple, risk of duplicates on crash
   MANUAL    → full control, commit only after successful processing
   Commit offset+1, not the processed offset

✅ Liveness timers:
   heartbeat.interval.ms  → how often background thread sends heartbeat
   session.timeout.ms     → max time without heartbeat before declared dead
   max.poll.interval.ms   → max time between poll() calls (processing time limit)

✅ Spring Kafka AckModes:
   BATCH             → default, commits after poll batch
   MANUAL_IMMEDIATE  → developer calls ack.acknowledge(), commits immediately
   RECORD            → per-record commit (slowest, safest)

✅ Delivery semantics:
   At-most-once  → commit before processing (possible loss)
   At-least-once → commit after processing (possible duplicates) ← most common
   Exactly-once  → transactions OR idempotent consumer logic

✅ Common pitfalls:
   Auto-commit = at-least-once risk
   max.poll.interval too low = rebalance loops
   Not calling ack.acknowledge() = consumer stalls
   Committing offset not offset+1 = re-reads last message

✅ Scaling consumers: max parallel consumers = partition count
   Add partitions to scale beyond current consumer count
```

---

**← Previous:** [05 — Kafka Producers: Internals, Configs & Guarantees](./05-producers.md)  
**Next Topic →** [07 — Consumer Groups & Partition Assignment Strategies](./07-consumer-groups.md)
