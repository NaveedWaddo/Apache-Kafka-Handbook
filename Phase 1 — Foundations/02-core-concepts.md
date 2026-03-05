# 02 — Core Concepts: Topics, Partitions, Offsets & Messages
> **Phase 1 — Foundations**  
> 📁 `notes/02-core-concepts.md`

---

## 📖 1. Concept Explanation

These are the **four foundational building blocks** of Kafka. Every advanced Kafka concept traces back to these. Understanding them deeply is non-negotiable.

### 1.1 — Topic

A **Topic** is a **named, logical channel** to which messages are written and from which they are read.

Think of it like a **category or feed name** to which records are published.

- Topics are identified by name (e.g., `user-events`, `order-created`, `payment-processed`)
- Producers write to a topic; consumers read from a topic
- A topic can have **many producers** and **many consumer groups**
- Topics are **multi-subscriber** — unlike a queue where one consumer takes the message, in Kafka **every consumer group** gets its own independent copy of the full stream

```
Topic: "order-created"

Producer A  ──┐
Producer B  ──┼──→  [ order-created topic ]  ──→  Consumer Group 1 (Billing Service)
Producer C  ──┘                               ──→  Consumer Group 2 (Inventory Service)
                                              ──→  Consumer Group 3 (Analytics)
```

---

### 1.2 — Partition

A **Partition** is a **single ordered, immutable sequence of records** within a topic.

Every topic is split into one or more partitions. This is the unit of **parallelism** and **scalability** in Kafka.

**Key rules of partitions:**
- Messages within a single partition are **strictly ordered**
- Messages across different partitions have **no ordering guarantee**
- Each partition is stored on **one broker** (the leader), with copies on others (replicas)
- The number of partitions determines **maximum consumer parallelism**

```
Topic: "order-created"  (3 partitions)

Partition 0: [Msg@0] → [Msg@1] → [Msg@4] → [Msg@7] ...
Partition 1: [Msg@0] → [Msg@2] → [Msg@5] → [Msg@8] ...
Partition 2: [Msg@0] → [Msg@3] → [Msg@6] → [Msg@9] ...
              ↑
        Each partition has its
        own independent offset counter
```

**Why partition?**
- A single machine can't handle unlimited write throughput
- Partitions spread load across multiple brokers
- Multiple consumers read different partitions **in parallel**

---

### 1.3 — Offset

An **Offset** is a **unique, sequential, integer ID** assigned to each message within a partition.

- Offsets start at `0` and increase by 1 for each new message
- Offsets are **partition-scoped** — Partition 0 has offsets 0,1,2,3... and Partition 1 also has its own independent offsets 0,1,2,3...
- Kafka uses offsets to track **where each consumer group has read up to**
- Consumers commit their offset to tell Kafka: "I've processed up to here"

```
Partition 0:
┌────────┬────────┬────────┬────────┬────────┬────────┐
│ Off: 0 │ Off: 1 │ Off: 2 │ Off: 3 │ Off: 4 │ Off: 5 │  → ...
└────────┴────────┴────────┴────────┴────────┴────────┘
              ↑
    Consumer Group "billing" has committed offset 1
    (meaning it has processed messages 0 and 1,
     next it will fetch offset 2)
```

**Three special offset reset positions:**
```
auto.offset.reset = "earliest"  → Start from offset 0 (oldest available message)
auto.offset.reset = "latest"    → Start from the next NEW message (skip history)
auto.offset.reset = "none"      → Throw error if no committed offset found
```

---

### 1.4 — Message (Record)

A **Message** (also called a **Record** in Kafka's official terminology) is the **unit of data** in Kafka.

**Anatomy of a Kafka Message:**

```
┌─────────────────────────────────────────────────────────┐
│                    KAFKA MESSAGE                        │
├─────────────────┬───────────────────────────────────────┤
│   HEADERS       │  Optional key-value metadata pairs    │
│                 │  e.g., {"source": "checkout-service"} │
├─────────────────┼───────────────────────────────────────┤
│   KEY           │  Optional. Used for partitioning.     │
│                 │  e.g., "user-id:u123"                 │
│                 │  Same key → always same partition      │
├─────────────────┼───────────────────────────────────────┤
│   VALUE         │  The actual payload (bytes).          │
│                 │  Can be JSON, Avro, Protobuf, String  │
│                 │  e.g., {"order_id":"o456","amount":99}│
├─────────────────┼───────────────────────────────────────┤
│   TIMESTAMP     │  When the message was created         │
│                 │  (producer time or broker ingestion)  │
├─────────────────┼───────────────────────────────────────┤
│   OFFSET        │  Assigned by broker. Immutable.       │
│                 │  e.g., 10492                          │
├─────────────────┼───────────────────────────────────────┤
│   PARTITION     │  Which partition this message is in   │
│                 │  e.g., 2                              │
└─────────────────┴───────────────────────────────────────┘
```

**The Key is critical for partitioning:**
- If key is `null` → messages are distributed **round-robin** across partitions
- If key is set → Kafka hashes the key (murmur2 hash) and routes to a **deterministic partition**
- Same key = same partition = **ordering guaranteed for that key**

```
order_id="A" → hash → Partition 0  (always)
order_id="B" → hash → Partition 2  (always)
order_id="C" → hash → Partition 1  (always)
```

This ensures all events for `order_id="A"` are always ordered — critical for event sourcing.

---

## 🏗️ 2. Architecture / Flow — How It All Fits Together

### Full Picture: Topics → Partitions → Offsets → Consumers

```
TOPIC: "payments"  (3 partitions, replication-factor=2)

                   ┌─── BROKER 1 ─────────────────────────────────┐
                   │  Partition 0 (LEADER)                         │
                   │  [off:0|$50] [off:1|$30] [off:2|$80]  →      │
                   │                                               │
                   │  Partition 2 (REPLICA)                        │
                   │  [off:0|$90] [off:1|$10]               →      │
                   └───────────────────────────────────────────────┘

                   ┌─── BROKER 2 ─────────────────────────────────┐
                   │  Partition 1 (LEADER)                         │
                   │  [off:0|$20] [off:1|$70] [off:2|$15]  →      │
                   │                                               │
                   │  Partition 0 (REPLICA)                        │
                   │  [off:0|$50] [off:1|$30] [off:2|$80]  →      │
                   └───────────────────────────────────────────────┘

                   ┌─── BROKER 3 ─────────────────────────────────┐
                   │  Partition 2 (LEADER)                         │
                   │  [off:0|$90] [off:1|$10]               →      │
                   │                                               │
                   │  Partition 1 (REPLICA)                        │
                   │  [off:0|$20] [off:1|$70] [off:2|$15]  →      │
                   └───────────────────────────────────────────────┘


Consumer Group: "fraud-detector"
  Consumer A  →  reads Partition 0  (committed at offset 2)
  Consumer B  →  reads Partition 1  (committed at offset 1)
  Consumer C  →  reads Partition 2  (committed at offset 0)
```

### Message Flow Step-by-Step

```
Step 1: Producer sends message with key="order-123"

Step 2: Kafka client hashes the key
         hash("order-123") % 3 = Partition 1

Step 3: Producer sends to Broker 2 (leader of Partition 1)

Step 4: Broker 2 appends message to Partition 1 log
         Assigns offset = 3 (next available)

Step 5: Broker 2 replicates to Broker 3 (follower of Partition 1)

Step 6: Broker 2 ACKs to producer (if acks=all)

Step 7: Consumer in "fraud-detector" group polls Partition 1
         Receives message at offset 3

Step 8: Consumer processes it, commits offset 4
         ("I'm done with everything up to and including offset 3")
```

---

## ⚙️ 3. How It Works Internally

### Partition Key Hashing (Kafka's Default Partitioner)

```
Kafka DefaultPartitioner logic (simplified):

if (key == null) {
    use StickyPartitionCache (round-robin per batch)
} else {
    partition = abs(murmur2(keyBytes)) % numPartitions
}
```

The **murmur2** hash is fast and produces a good distribution — the same key will always yield the same partition number **as long as the partition count stays the same**.

> ⚠️ If you increase partitions later, `hash(key) % newCount` changes, breaking the old mapping.

---

### Offset Commit Storage — `__consumer_offsets`

Consumer offsets (progress tracking) are stored in a special internal Kafka topic:

```
Topic: __consumer_offsets

Key:   (consumer_group_id, topic_name, partition_number)
Value: committed_offset

Example records:
  ("fraud-detector", "payments", 0) → 2
  ("fraud-detector", "payments", 1) → 1
  ("fraud-detector", "payments", 2) → 0
```

When a consumer restarts, it queries this topic:
*"Where did my group (`fraud-detector`) last stop on `payments`, partition 0?"*  
Answer: offset 2 → so resume from offset 3.

---

### Log Segments — How Partitions Are Stored on Disk

```
/kafka-logs/payments-0/                ← Partition 0 directory
    00000000000000000000.log           ← Segment: offsets 0–999
    00000000000000001000.log           ← Segment: offsets 1000–1999
    00000000000000002000.log           ← Active segment (being written)
    00000000000000000000.index         ← Sparse offset→file-position index
    00000000000000000000.timeindex     ← Timestamp index
```

- Only the **last segment** is actively written to (append-only)
- Old segments are eligible for **deletion** (time/size retention) or **compaction**
- Index files enable **O(1)** lookup for any offset

---

## 💻 4. Code Examples (Java + Spring Boot)

### Creating a Topic Programmatically

```java
package com.example.kafka.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic orderCreatedTopic() {
        return TopicBuilder.name("order-created")
                .partitions(6)          // 6 partitions for parallelism
                .replicas(3)            // 3 replicas for fault tolerance
                .build();
    }

    @Bean
    public NewTopic paymentsTopic() {
        return TopicBuilder.name("payments")
                .partitions(12)
                .replicas(3)
                .config("retention.ms", "604800000")   // 7 days
                .config("compression.type", "lz4")
                .build();
    }
}
```
> Spring Boot auto-creates these topics on startup via `KafkaAdmin`.

---

### Producer with Explicit Key (Ordering per Entity)

```java
package com.example.kafka.producer;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class OrderEventProducer {

    private static final String TOPIC = "order-events";
    private final KafkaTemplate<String, String> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendOrderEvent(String orderId, String status) {
        String payload = String.format(
            "{\"order_id\":\"%s\", \"status\":\"%s\"}", orderId, status
        );

        // key = orderId → all events for this order go to the SAME partition
        // This guarantees: created → paid → shipped → delivered are in order
        CompletableFuture<SendResult<String, String>> future =
            kafkaTemplate.send(TOPIC, orderId, payload);

        future.whenComplete((result, ex) -> {
            if (ex == null) {
                System.out.printf(
                    "Sent [orderId=%s] → partition=%d, offset=%d%n",
                    orderId,
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset()
                );
            } else {
                System.err.println("Failed to send: " + ex.getMessage());
            }
        });
    }

    // Send a sequence of events for the same order
    public void sendOrderLifecycle(String orderId) {
        for (String status : new String[]{"CREATED", "PAID", "SHIPPED", "DELIVERED"}) {
            sendOrderEvent(orderId, status);
        }
    }
}
```

---

### Consumer with Full Message Metadata

```java
package com.example.kafka.consumer;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.header.Header;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

@Service
public class OrderEventConsumer {

    @KafkaListener(topics = "order-events", groupId = "fraud-detector")
    public void consume(ConsumerRecord<String, String> record) {

        // Access all message metadata
        System.out.println("=== Message Received ===");
        System.out.println("Topic     : " + record.topic());
        System.out.println("Partition : " + record.partition());
        System.out.println("Offset    : " + record.offset());
        System.out.println("Key       : " + record.key());
        System.out.println("Value     : " + record.value());
        System.out.println("Timestamp : " + record.timestamp());

        // Access headers
        for (Header header : record.headers()) {
            System.out.printf("Header    : %s = %s%n",
                header.key(), new String(header.value()));
        }
    }
}
```

---

### Producer with Custom Headers

```java
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.header.internals.RecordHeader;

public void sendWithHeaders(String orderId, String payload) {
    ProducerRecord<String, String> record = new ProducerRecord<>(
        "order-events",   // topic
        null,             // partition (null = let Kafka decide via key)
        orderId,          // key
        payload           // value
    );

    // Add custom headers (metadata, tracing, versioning)
    record.headers().add(new RecordHeader("source", "checkout-service".getBytes()));
    record.headers().add(new RecordHeader("version", "v2".getBytes()));
    record.headers().add(new RecordHeader("trace-id", "abc-123-xyz".getBytes()));

    kafkaTemplate.send(record);
}
```

---

### Seek to Specific Offset (Manual Offset Control)

```java
package com.example.kafka.consumer;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.TopicPartition;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.listener.ConsumerSeekAware;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class ReplayConsumer implements ConsumerSeekAware {

    // Seek to offset 50 on partition 0 when consumer starts
    @Override
    public void onPartitionsAssigned(
            Map<TopicPartition, Long> assignments,
            ConsumerSeekCallback callback) {

        assignments.forEach((partition, currentOffset) -> {
            if (partition.partition() == 0) {
                callback.seek(partition.topic(), partition.partition(), 50);
                System.out.println("Seeked partition 0 to offset 50");
            }
        });
    }

    @KafkaListener(topics = "order-events", groupId = "replay-group")
    public void consume(ConsumerRecord<String, String> record) {
        System.out.printf("Replaying: offset=%d value=%s%n",
            record.offset(), record.value());
    }
}
```

---

## ⚡ 5. Key Configurations

| Config | Default | Explanation |
|--------|---------|-------------|
| `num.partitions` | 1 | Default partition count for auto-created topics |
| `replication.factor` | 1 | Number of replica copies per partition |
| `retention.ms` | 604800000 (7 days) | How long messages are retained |
| `retention.bytes` | -1 (unlimited) | Max size per partition before deletion |
| `auto.offset.reset` | latest | Where consumer starts with no committed offset |
| `enable.auto.commit` | true | Auto-commit offset on a timer |
| `auto.commit.interval.ms` | 5000 | Frequency of auto offset commits |
| `log.segment.bytes` | 1073741824 (1 GB) | Size at which a new log segment is created |

---

## 🔄 6. Real-World Scenarios

### Choosing Partition Count

```
Formula:
  Required Partitions = Target Throughput / Throughput Per Partition

Example:
  Target: 1 GB/s
  Each partition handles ~50 MB/s
  → 1000 MB / 50 MB = 20 partitions minimum

  Add headroom for 3x spikes: 20 × 3 = 60 partitions
  Round to nearest broker multiple: 64 partitions (8 brokers × 8)
```

**Rules:**
- You can **increase** partitions later but **never decrease**
- Increasing partitions **breaks key ordering** (keys rehash to new partitions)
- Start with more than you think you need (12, 24, 48 are common starting points)

---

### Null Key vs. Keyed Messages

```
Use NULL key when:
  ✅ Ordering does not matter
  ✅ You want even load distribution
  ✅ Logs, metrics, click events, audit trails

Use a specific KEY when:
  ✅ Ordering matters per entity
  ✅ All events for user_id="u123" must be sequentially ordered
  ✅ All updates for order_id="o456" must be in sequence
  ✅ Change Data Capture (CDC) — all changes for a row/entity
```

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Using `null` keys when order matters | Same entity's events split across partitions | Use entity ID (user_id, order_id) as key |
| Too few partitions at launch | Can't scale consumers beyond partition count | Start with 6–12+ partitions, plan for growth |
| Too many partitions | Memory overhead, longer leader election on failure | Don't create thousands per topic |
| Confusing offset scope | Thinking offset 5 in P0 = same message as offset 5 in P1 | Offsets are **per-partition** and fully independent |
| Not committing offsets | On crash/restart, consumer reprocesses from scratch | Use reliable commit strategy (covered in Topic 06) |
| Increasing partitions on live topic | Key → partition routing changes, breaks ordering | Plan partition count upfront; scale via consumer groups instead |

---

## 🎯 8. Interview Questions

**Q1. What is a Kafka partition and why does it exist?**
> A partition is an ordered, immutable log subdivision of a topic. Partitions enable **horizontal scalability and parallelism** — they spread data across brokers and let multiple consumers read simultaneously, each handling a subset of partitions.

**Q2. Are messages ordered in Kafka?**
> Kafka guarantees ordering **within a partition only**. To guarantee ordering for a specific entity (e.g., all events for order_id="X"), use that entity's ID as the message key. Kafka will consistently route the same key to the same partition via murmur2 hashing.

**Q3. What is an offset and where is it stored?**
> An offset is a monotonically increasing integer that uniquely identifies a message within a partition. Consumer group offsets (tracking read progress) are stored in the internal Kafka topic `__consumer_offsets`.

**Q4. What happens if you increase partitions on an existing topic?**
> You can increase but **never decrease** partitions. The problem: existing key→partition mappings change because `hash(key) % oldCount ≠ hash(key) % newCount`. This **breaks the ordering guarantee** for previously stable keyed messages.

**Q5. What is the difference between a topic and a partition?**
> A **topic** is a logical name/category for a stream of records. A **partition** is the physical, ordered log that is the unit of storage and parallelism inside a topic. One topic = 1 to N partitions, each an independent append-only log.

**Q6. How does Kafka route a message with a key?**
> Kafka applies the **murmur2 hash** to the key's bytes, then `abs(hash) % numPartitions`. The same key always maps to the same partition, guaranteeing ordering for that key. Without a key, Spring Kafka uses sticky partitioning (fills one partition's batch before switching).

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "The Ordering Nightmare"
> **Problem:** A payment system has two events: `USER_UPGRADED` and `PAYMENT_PROCESSED`. Sometimes payment is processed before the upgrade is recorded, causing failures. Both go to the same Kafka topic with 10 partitions. How do you fix it?

**Answer:**
Root cause: **no key set** — events for the same user land on different partitions, consumed by different threads with no ordering.

**Fix in Spring Boot:**
```java
// Before (broken - null key, random partition assignment)
kafkaTemplate.send("user-events", payload);

// After (fixed - key ensures same user → same partition → ordered)
kafkaTemplate.send("user-events", userId, payload);
```
Now all events for `userId="u123"` always land on Partition 4 (for example). The partition's sequential guarantee ensures `USER_UPGRADED` is always consumed before `PAYMENT_PROCESSED` for that user.

---

### Scenario 2: "How Many Partitions?"
> **Problem:** System needs 500,000 msgs/sec. Each consumer handles 25,000 msgs/sec. Plan for 3x traffic spikes. How many partitions?

**Answer:**
```
Normal load:       500,000 / 25,000 = 20 consumers needed
3x spike headroom: 20 × 3           = 60 consumers max

→ Need at least 60 partitions
  (max parallelism = partition count)

Practical choice: 64 partitions (multiple of 8 brokers)
Also ensure: partitions % broker_count == 0 for even spread
```

---

### Scenario 3: "The Bug Reprocessing Incident"
> **Problem:** A bad deployment processed messages incorrectly for 2 hours. Bug is now fixed. How do you reprocess those 2 hours of data?

**Answer (using Spring Boot + Kafka CLI):**

Since Kafka **retains** messages (unlike traditional MQs):

```bash
# Step 1: Stop consumer group (or scale to 0 replicas in k8s)

# Step 2: Reset offsets to 2 hours ago
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group billing-service \
  --topic payments \
  --reset-offsets \
  --to-datetime 2024-01-15T08:00:00.000 \
  --execute

# Step 3: Redeploy consumers with the fix
# They will resume from the reset offset and reprocess
```

**In Spring Boot** you can also implement a `ConsumerSeekAware` listener to seek to a specific timestamp on startup for programmatic replay.

---

## 📝 10. Quick Revision Summary

```
✅ TOPIC      = Named logical channel. Multi-subscriber.
               Every consumer GROUP gets all messages independently.

✅ PARTITION  = Ordered, immutable log. Unit of parallelism.
               Ordering guaranteed WITHIN partition only.
               Max parallelism = number of partitions.

✅ OFFSET     = Sequential ID per message, scoped PER partition.
               Stored in __consumer_offsets internal topic.
               Committed offset = "processed everything before this"

✅ MESSAGE    = Key + Value + Headers + Timestamp + Offset + Partition
               Key  → murmur2 hash % numPartitions = deterministic partition
               null → sticky/round-robin partitioning (no ordering)
               Same key → same partition → ordering per key guaranteed

✅ Key Design Rules:
   Use keys    → when per-entity ordering matters (user_id, order_id)
   Null keys   → when distribution matters more than order (logs, metrics)
   Never increase partitions without a migration plan

✅ Spring Boot Quick Reference:
   TopicBuilder.name().partitions().replicas().build() → create topic
   kafkaTemplate.send(topic, key, value)               → produce
   @KafkaListener(topics, groupId)                     → consume
   ConsumerRecord<K,V>                                 → full message access
   ConsumerSeekAware                                   → manual offset control
```

---

**← Previous:** [01 — Introduction to Apache Kafka](./01-introduction.md)  
**Next Topic →** [03 — Kafka Architecture Overview: Brokers, Clusters, ZooKeeper/KRaft](./03-architecture-overview.md)
