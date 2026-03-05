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
- Offsets are **partition-scoped** — Partition 0 has offsets 0,1,2,3... and Partition 1 also has its own offsets 0,1,2,3...
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
     next it will read offset 2)
```

**Three special offset positions:**
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
- Same key = same partition = **ordering guaranteed** for that key

```
order_id="A" → hash → Partition 0  (always)
order_id="B" → hash → Partition 2  (always)
order_id="C" → hash → Partition 1  (always)
```

This ensures all events for `order_id="A"` are ordered — critical for event sourcing.

---

## 🏗️ 2. Architecture / Flow — How It All Fits Together

### Full Picture: Topics → Partitions → Offsets → Consumers

```
TOPIC: "payments"  (3 partitions, replication-factor=2)

                   ┌─── BROKER 1 ────────────────────────────────┐
                   │  Partition 0 (LEADER)                        │
                   │  [off:0|$50] [off:1|$30] [off:2|$80]  →     │
                   │                                              │
                   │  Partition 2 (REPLICA)                       │
                   │  [off:0|$90] [off:1|$10]               →     │
                   └──────────────────────────────────────────────┘

                   ┌─── BROKER 2 ────────────────────────────────┐
                   │  Partition 1 (LEADER)                        │
                   │  [off:0|$20] [off:1|$70] [off:2|$15]  →     │
                   │                                              │
                   │  Partition 0 (REPLICA)                       │
                   │  [off:0|$50] [off:1|$30] [off:2|$80]  →     │
                   └──────────────────────────────────────────────┘

                   ┌─── BROKER 3 ────────────────────────────────┐
                   │  Partition 2 (LEADER)                        │
                   │  [off:0|$90] [off:1|$10]               →     │
                   │                                              │
                   │  Partition 1 (REPLICA)                       │
                   │  [off:0|$20] [off:1|$70] [off:2|$15]  →     │
                   └──────────────────────────────────────────────┘


Consumer Group: "fraud-detector"
  Consumer A  →  reads Partition 0 (committed at offset 2)
  Consumer B  →  reads Partition 1 (committed at offset 1)
  Consumer C  →  reads Partition 2 (committed at offset 0)
```

### Message Flow Step-by-Step

```
Step 1: Producer sends message with key="order-123"

Step 2: Kafka client hashes the key
         hash("order-123") % 3 = Partition 1

Step 3: Producer sends to Broker 2 (leader of Partition 1)

Step 4: Broker 2 appends message to Partition 1's log
         Assigns offset = 3 (next available)

Step 5: Broker 2 replicates to Broker 3 (follower of Partition 1)

Step 6: Broker 2 acknowledges to producer (if acks=all)

Step 7: Consumer in "fraud-detector" group polls Partition 1
         Gets message at offset 3

Step 8: Consumer processes it, commits offset 4
         (meaning "I'm done with everything up to and including offset 3")
```

---

## ⚙️ 3. How It Works Internally

### Partition Assignment and Key Hashing

```python
# Kafka's default partitioner (simplified)
def get_partition(key: bytes, num_partitions: int) -> int:
    if key is None:
        # Round-robin across partitions
        return round_robin_counter % num_partitions
    else:
        # murmur2 hash of key bytes
        hash_value = murmur2(key)
        return abs(hash_value) % num_partitions
```

### Offset Commit Storage

Consumer offsets are stored in a special internal Kafka topic: **`__consumer_offsets`**

```
__consumer_offsets topic stores:
  Key:   (consumer_group_id, topic, partition)
  Value: committed_offset

Example:
  ("fraud-detector", "payments", 0) → offset: 2
  ("fraud-detector", "payments", 1) → offset: 1
  ("fraud-detector", "payments", 2) → offset: 0
```

This means even if a consumer crashes, when it restarts it asks Kafka:  
*"Where did my group last stop on this partition?"* and resumes from there.

---

### Log Segments (How Partitions are Stored on Disk)

Each partition is stored as a series of **segment files** on disk:

```
/kafka-logs/payments-0/          ← Partition 0 directory
    00000000000000000000.log     ← Segment: offsets 0 to 999
    00000000000000001000.log     ← Segment: offsets 1000 to 1999
    00000000000000002000.log     ← Segment: offsets 2000 onwards (active)
    00000000000000000000.index   ← Sparse index: offset → file position
    00000000000000000000.timeindex ← Timestamp index
```

- Only the **last (active) segment** is written to
- Kafka can delete/compact old segments based on retention policy
- The **index files** allow O(1) random access to any offset

---

## 💻 4. Code Examples

### Creating a Topic with Partitions (Admin API)

```python
from kafka.admin import KafkaAdminClient, NewTopic

admin = KafkaAdminClient(bootstrap_servers="localhost:9092")

topic = NewTopic(
    name="order-created",
    num_partitions=6,         # 6 partitions for parallelism
    replication_factor=3      # 3 copies for fault tolerance
)

admin.create_topics([topic])
print("Topic created!")
```

### Producer with Explicit Key (Ordering per entity)

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    key_serializer=str.encode,
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# All events for order "O-100" go to the SAME partition
# This guarantees ordering for this specific order
for event in ['created', 'paid', 'shipped', 'delivered']:
    producer.send(
        topic='order-events',
        key='O-100',           # ← key determines partition
        value={'order_id': 'O-100', 'status': event}
    )

producer.flush()
```

### Consumer Reading with Offset Control

```python
from kafka import KafkaConsumer, TopicPartition

consumer = KafkaConsumer(bootstrap_servers=['localhost:9092'])

# Manually assign a specific partition and offset
tp = TopicPartition('order-events', partition=0)
consumer.assign([tp])
consumer.seek(tp, offset=50)   # Start reading from offset 50

for message in consumer:
    print(f"Offset {message.offset}: {message.value}")
    if message.offset >= 100:  # Read only offsets 50-100
        break
```

### Inspecting Message Metadata

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    'payments',
    bootstrap_servers=['localhost:9092'],
    group_id='my-group',
    auto_offset_reset='earliest',
    value_deserializer=lambda v: v.decode('utf-8')
)

for msg in consumer:
    print(f"""
    Topic:     {msg.topic}
    Partition: {msg.partition}
    Offset:    {msg.offset}
    Key:       {msg.key}
    Value:     {msg.value}
    Timestamp: {msg.timestamp}
    Headers:   {msg.headers}
    """)
```

---

## ⚡ 5. Key Configurations

| Config | Default | Explanation |
|--------|---------|-------------|
| `num.partitions` | 1 | Default partitions for auto-created topics |
| `replication.factor` | 1 | Number of copies per partition |
| `retention.ms` | 604800000 (7 days) | How long messages are kept |
| `retention.bytes` | -1 (unlimited) | Max size per partition before deletion |
| `auto.offset.reset` | latest | Where consumer starts if no offset exists |
| `enable.auto.commit` | true | Auto-commit offset periodically |
| `auto.commit.interval.ms` | 5000 | Frequency of auto offset commit |
| `log.segment.bytes` | 1GB | Size at which a new log segment is created |

---

## 🔄 6. Real-World Scenarios

### Scenario: Choosing the Right Number of Partitions

**Rule of thumb:**
```
Target Throughput (msgs/sec)
─────────────────────────────  =  Minimum Partitions Needed
Throughput per Partition
```

Kafka can handle ~10-100 MB/s per partition (varies by hardware).

- If you need 1 GB/s throughput and each partition handles ~50 MB/s → **20 partitions**
- More partitions = more parallelism but also more overhead (file handles, replication traffic)
- **You can increase partitions later, but you cannot decrease them**
- Adding partitions breaks key-based ordering for existing keys (keys rehash to new partitions)

**Recommended practice:** Start with more partitions than you think you need (e.g., 12 or 24), as reducing is not possible.

---

### Scenario: Null Key vs. Keyed Messages

```
Use NULL key when:
  - You don't need ordering
  - You want even load distribution
  - e.g., logs, metrics, click events

Use a KEY when:
  - You need ordering per entity
  - e.g., all events for user_id="u123" must be ordered
  - e.g., all updates for order_id="o456" must be in sequence
  - e.g., CDC (Change Data Capture) — changes per row
```

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Using `null` keys when order matters | Events for the same entity can go to different partitions, breaking order | Always use a meaningful key (user_id, order_id) |
| Too few partitions | Can't scale consumers beyond partition count | Plan for growth; start with 6-12+ partitions |
| Too many partitions | Increased memory, replication overhead, longer failover | Don't create thousands of partitions per topic |
| Misunderstanding offset scope | Thinking offset 5 in partition 0 = offset 5 in partition 1 | Offsets are **per-partition**; they are independent |
| Not committing offsets | On restart, consumer reprocesses from start | Ensure reliable offset commit strategy |
| Using earliest without idempotency | On restart, duplicate messages get processed | Make consumers idempotent or use exactly-once |

---

## 🎯 8. Interview Questions

**Q1. What is a Kafka partition and why does it exist?**
> A partition is an ordered, immutable log that is a subdivision of a topic. Partitions exist for **scalability and parallelism** — they allow a topic's data to be spread across multiple brokers and consumed in parallel by multiple consumers simultaneously.

**Q2. Are messages ordered in Kafka?**
> Kafka guarantees ordering **within a partition**, not across partitions. If you need all events for a specific entity to be ordered (e.g., all updates for order_id="X"), use that entity's ID as the message key — Kafka will always route the same key to the same partition.

**Q3. What is an offset and where is it stored?**
> An offset is a monotonically increasing integer that uniquely identifies a message within a partition. Consumer offsets (tracking how far each consumer group has read) are stored in the internal Kafka topic `__consumer_offsets`.

**Q4. What happens if you increase the number of partitions on an existing topic?**
> You can increase partitions but **never decrease**. The problem is that existing key-to-partition mappings change — `hash(key) % old_partitions` ≠ `hash(key) % new_partitions`. This breaks ordering guarantees for keyed messages that were previously routed consistently.

**Q5. What is the difference between a topic and a partition?**
> A **topic** is a logical name/category for a stream of records. A **partition** is the physical unit of storage and parallelism within a topic. One topic has 1 to N partitions, each being an independent ordered log.

**Q6. How does Kafka route messages with a key?**
> Kafka uses the **murmur2 hash** of the key's bytes, then `abs(hash) % num_partitions` to determine the partition. The same key always produces the same partition assignment (for a given number of partitions).

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "The Ordering Nightmare"
> **Problem:** You have a payment processing system. A user upgrades their plan, then immediately makes a payment. Sometimes the payment is processed before the upgrade, causing it to fail. Both events go to the same Kafka topic with 10 partitions. How do you fix this?

**Answer:**  
The root cause is **no key set** — events for the same user are landing on different partitions and being consumed by different consumer instances in any order.

**Fix:** Use `user_id` as the Kafka message key.
```python
producer.send('user-events', key=str(user_id), value=event_data)
```
Now all events for `user_id=123` always go to the same partition (e.g., Partition 4). The partition's sequential guarantee ensures the upgrade event is always consumed before the payment event for that user.

---

### Scenario 2: "Exactly How Many Partitions?"
> **Problem:** Your system needs to process 500,000 messages/sec. Each consumer instance can process 25,000 messages/sec. You also want headroom for 3x traffic spikes. How many partitions?

**Answer:**
```
Normal throughput:  500,000 / 25,000 = 20 consumers needed
Spike headroom:     20 × 3 = 60 consumers max

→ You need AT LEAST 60 partitions
  (max parallelism = number of partitions)

Practical choice: Round up to 64 or 72 (nice multiples)
Also account for: replication overhead, broker count
Rule: partitions should be a multiple of your broker count
  for even distribution
```

---

### Scenario 3: "The Offset Reset Incident"
> **Problem:** A bug was deployed that processed messages incorrectly for 2 hours. You've fixed the bug. How do you reprocess those 2 hours of messages?

**Answer:**
1. **Stop the consumer group** (or ensure all consumers are down)
2. **Find the start offset** corresponding to 2 hours ago using `kafka-consumer-groups.sh --reset-offsets --to-datetime 2024-01-15T08:00:00.000`
3. **Apply the reset** to set the committed offset back to that point
4. **Redeploy** consumers with the fix — they'll reread from the reset offset

Since Kafka **retains** messages (unlike traditional MQs), this is possible as long as the messages are within the retention period.

```bash
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group billing-service \
  --topic payments \
  --reset-offsets \
  --to-datetime 2024-01-15T08:00:00.000 \
  --execute
```

---

## 📝 10. Quick Revision Summary

```
✅ TOPIC    = Named logical channel for messages. Multi-subscriber.
✅ PARTITION = Ordered, immutable log. Unit of parallelism.
             Ordering guaranteed WITHIN partition only.
✅ OFFSET   = Sequential ID per message within a partition.
             Scoped to partition (each partition has its own 0,1,2...)
             Stored in __consumer_offsets internal topic.
✅ MESSAGE  = Has: Key, Value, Headers, Timestamp, Offset, Partition
             Key → determines partition (murmur2 hash % num_partitions)
             Null key → round-robin distribution
             Same key → always same partition → ordering per key

✅ Key Design Rules:
   - Use keys when order matters per entity (user_id, order_id)
   - Use null keys for even load distribution (logs, metrics)
   - Adding partitions breaks existing key assignments — plan ahead

✅ Offset Rules:
   - earliest = start from offset 0
   - latest   = start from next new message
   - Committed offset = "I've processed everything before this"
```

---

**← Previous:** [01 — Introduction to Apache Kafka](./01-introduction.md)  
**Next Topic →** [03 — Kafka Architecture Overview: Brokers, Clusters, ZooKeeper/KRaft](./03-architecture-overview.md)
