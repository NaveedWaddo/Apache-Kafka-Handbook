# 10 — Partitioning Strategies: Keys, Custom Partitioners & Ordering
> **Phase 3 — Kafka Internals & Storage**  
> 📁 `notes/10-partitioning.md`

---

## 📖 1. Concept Explanation

### Why Partitioning Strategy is a Senior-Level Decision

Partitioning affects:
- **Message ordering** — which messages are guaranteed to be ordered relative to each other
- **Load distribution** — whether all brokers get equal traffic or some become hot spots
- **Consumer parallelism** — how many consumer instances can work in parallel
- **Key-based routing** — whether all events for an entity land on the same partition

Choosing the wrong partitioning strategy is one of the most common and hardest-to-fix mistakes in Kafka design. You can add partitions later but you **cannot change the key design** without re-keying all historical data.

---

### The Four Partitioning Approaches

```
1. Key-based (Hash Partitioning)   → deterministic, ordering guaranteed per key
2. Null Key (Sticky/Round-Robin)   → even distribution, no ordering
3. Custom Partitioner              → full control over routing logic
4. Manual Partition Specification  → producer explicitly picks partition
```

---

### Default Partitioner Behavior (Kafka 3.x)

```
if (key != null):
    partition = abs(murmur2(keyBytes)) % numPartitions
    → Same key ALWAYS → same partition (as long as numPartitions unchanged)

if (key == null):
    Use StickyPartitionCache:
    → Fill current partition's batch until batch.size or linger.ms
    → Then switch to next partition
    → Better throughput than round-robin (larger batches per partition)
    → Pre-Kafka 2.4: was pure round-robin (one msg per partition = tiny batches)
```

---

## 🏗️ 2. Architecture / Flow

### Key-Based Partitioning — Ordering Guarantee

```
Topic: "order-events" (4 partitions)

Producer sends with key = orderId:

  orderId="A100" → murmur2("A100") % 4 = 2 → Partition 2
  orderId="B200" → murmur2("B200") % 4 = 0 → Partition 0
  orderId="C300" → murmur2("C300") % 4 = 3 → Partition 3
  orderId="A100" → murmur2("A100") % 4 = 2 → Partition 2 (SAME!)

Partition 2 log:
  [A100: CREATED] → [A100: PAID] → [A100: SHIPPED] → [A100: DELIVERED]
  ↑ All events for A100 are strictly ordered within partition 2

Consumer assigned to Partition 2 processes A100 events in correct sequence.
```

### Hot Partition Problem

```
Bad key choice: status = "CREATED" | "PAID" | "SHIPPED"

100,000 orders created per second → "CREATED" key → ALL go to Partition 1
   10 orders paid per second     → "PAID" key    → all go to Partition 2
    5 orders shipped per second  → "SHIPPED" key → all go to Partition 3

Partition 1: ██████████████████████ OVERLOADED (100K msgs/sec)
Partition 2: █░░░░░░░░░░░░░░░░░░░░░ light load  (10 msgs/sec)
Partition 3: ░░░░░░░░░░░░░░░░░░░░░░ idle         (5 msgs/sec)

→ Partition 1 broker becomes the bottleneck
→ Consumers for Partition 1 can't keep up
→ Growing lag on Partition 1 only

Fix: Use orderId as key (high cardinality = even distribution)
```

### Custom Partitioner — Routing Logic

```
Use case: VIP customers get priority partition

Partitions 0-7: Regular customers
Partitions 8-9: VIP customers (dedicated, faster consumers)

Customer "VIP-001" → routes to partition 8 or 9
Customer "REG-123" → routes to partition 0-7 (round-robin among these)
```

---

## ⚙️ 3. How It Works Internally

### murmur2 Hash Function

Kafka uses the **murmur2** non-cryptographic hash function:

```java
// Kafka's DefaultPartitioner (simplified):
public int partition(String topic, Object key, byte[] keyBytes,
                     Object value, byte[] valueBytes, Cluster cluster) {

    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();

    if (keyBytes == null) {
        // Sticky partitioning (Kafka 2.4+)
        return stickyPartitionCache.partition(topic, cluster);
    }

    // murmur2 hash → absolute value → mod numPartitions
    return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
}
```

**Why murmur2?**
- Very fast (no cryptographic overhead)
- Good distribution (low collision rate)
- Deterministic (same input = same hash always)

### Sticky Partitioner (Null Key Behavior Since Kafka 2.4)

```
Old behavior (pre-2.4): round-robin
  Message 1 → Partition 0
  Message 2 → Partition 1
  Message 3 → Partition 2
  → Each partition gets tiny 1-message batches → poor throughput

New behavior (sticky):
  Messages 1-50   → Partition 0  (until batch full or linger.ms)
  Messages 51-100 → Partition 1  (then switch)
  Messages 101-150 → Partition 2
  → Larger batches per partition → better compression + throughput
```

### What Happens When Partition Count Changes

```
Before change: 4 partitions
  Key "user-123" → murmur2("user-123") % 4 = Partition 2

After adding partitions: 8 partitions
  Key "user-123" → murmur2("user-123") % 8 = Partition 6  ← DIFFERENT!

Impact:
  All new messages for "user-123" go to Partition 6
  All old messages for "user-123" are in Partition 2
  Consumers now see ordering BROKEN for existing keys

This is why increasing partitions on a live topic is dangerous!
```

---

## 💻 4. Code Examples (Java + Spring Boot)

### Keyed Producer — Ordering Guarantee

```java
package com.example.kafka.producer;

import com.example.kafka.model.OrderEvent;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class KeyedOrderProducer {

    private static final String TOPIC = "order-events";
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public KeyedOrderProducer(KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // All events for the same orderId → same partition → ordered
    public void sendOrderLifecycle(String orderId) {
        String userId = "u-" + orderId;
        double amount = 99.99;

        // These 4 events will always be in THIS exact order per partition
        for (String status : new String[]{"CREATED", "PAID", "SHIPPED", "DELIVERED"}) {
            OrderEvent event = new OrderEvent(orderId, userId, status, amount);

            // KEY = orderId → deterministic partition
            kafkaTemplate.send(TOPIC, orderId, event)
                .whenComplete((result, ex) -> {
                    if (ex == null) {
                        System.out.printf(
                            "Sent [%s/%s] → partition=%d offset=%d%n",
                            orderId, status,
                            result.getRecordMetadata().partition(),
                            result.getRecordMetadata().offset()
                        );
                    }
                });
        }
    }

    // Composite key: route by region + customerId for geo-aware partitioning
    public void sendWithCompositeKey(String region, String customerId,
                                     OrderEvent event) {
        // Composite key ensures all events for same customer in same region
        // land on same partition
        String compositeKey = region + ":" + customerId;
        kafkaTemplate.send(TOPIC, compositeKey, event);
    }
}
```

---

### Custom Partitioner — VIP Priority Routing

```java
package com.example.kafka.partitioner;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;
import org.apache.kafka.common.PartitionInfo;
import org.apache.kafka.common.utils.Utils;

import java.util.List;
import java.util.Map;

/**
 * Routes VIP customer events to dedicated high-priority partitions.
 * Assumes topic has 12 partitions:
 *   Partitions 0-9  → regular customers
 *   Partitions 10-11 → VIP customers (faster consumer group assigned)
 */
public class VIPCustomerPartitioner implements Partitioner {

    private static final int VIP_PARTITION_START = 10;

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {

        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int totalPartitions = partitions.size();           // 12
        int regularPartitions = VIP_PARTITION_START;       // 0-9
        int vipPartitions = totalPartitions - regularPartitions; // 10-11

        if (keyBytes == null) {
            // No key → round-robin in regular partitions
            return (int) (System.currentTimeMillis() % regularPartitions);
        }

        String keyStr = new String(keyBytes);

        // Route VIP customers to dedicated partitions
        if (keyStr.startsWith("VIP-")) {
            // Hash within VIP partition range (10-11)
            int hash = Utils.toPositive(Utils.murmur2(keyBytes));
            return VIP_PARTITION_START + (hash % vipPartitions);
        }

        // Regular customers: hash within partitions 0-9
        int hash = Utils.toPositive(Utils.murmur2(keyBytes));
        return hash % regularPartitions;
    }

    @Override public void close() {}

    @Override public void configure(Map<String, ?> configs) {}
}
```

Register the custom partitioner:

```java
package com.example.kafka.config;

import com.example.kafka.partitioner.VIPCustomerPartitioner;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.*;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class CustomPartitionerProducerConfig {

    @Bean
    public ProducerFactory<String, Object> vipProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        // Register the custom partitioner
        config.put(ProducerConfig.PARTITIONER_CLASS_CONFIG,
            VIPCustomerPartitioner.class.getName());

        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean("vipKafkaTemplate")
    public KafkaTemplate<String, Object> vipKafkaTemplate() {
        return new KafkaTemplate<>(vipProducerFactory());
    }
}
```

---

### Manual Partition Selection

```java
package com.example.kafka.producer;

import org.apache.kafka.clients.producer.ProducerRecord;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class ManualPartitionProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public ManualPartitionProducer(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // Explicitly specify partition — bypasses key-based routing
    public void sendToSpecificPartition(String topic, int partition,
                                        String key, Object value) {

        ProducerRecord<String, Object> record = new ProducerRecord<>(
            topic,
            partition,   // explicit partition
            key,
            value
        );

        kafkaTemplate.send(record)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    System.out.printf(
                        "Manual send → partition=%d offset=%d%n",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset()
                    );
                }
            });
    }

    // Geo-based routing: partition by region
    public void sendByRegion(String region, String customerId, Object event) {
        // Map region to partition range
        int partitionBase = switch (region) {
            case "us-east" -> 0;
            case "us-west" -> 3;
            case "eu"      -> 6;
            case "apac"    -> 9;
            default        -> 0;
        };

        // Hash customerId within the region's partition range (3 partitions each)
        int partitionOffset = Math.abs(customerId.hashCode()) % 3;
        int partition = partitionBase + partitionOffset;

        sendToSpecificPartition("customer-events", partition, customerId, event);
    }
}
```

---

### Custom Partitioner with Config Parameters

```java
package com.example.kafka.partitioner;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;
import org.apache.kafka.common.utils.Utils;

import java.util.Map;
import java.util.Set;

/**
 * Configurable partitioner that reads tenant tier from config.
 * Premium tenants get dedicated partitions.
 */
public class TenantAwarePartitioner implements Partitioner {

    private Set<String> premiumTenants;
    private int premiumPartitionCount;

    @Override
    public void configure(Map<String, ?> configs) {
        // Read config injected at producer factory creation
        String tenants = (String) configs.getOrDefault(
            "premium.tenants", "");
        this.premiumTenants = Set.of(tenants.split(","));

        this.premiumPartitionCount = Integer.parseInt(
            (String) configs.getOrDefault("premium.partition.count", "2"));
    }

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {

        int totalPartitions = cluster.partitionsForTopic(topic).size();
        int regularPartitions = totalPartitions - premiumPartitionCount;

        if (keyBytes == null) {
            return (int) (System.nanoTime() % regularPartitions);
        }

        String tenantId = new String(keyBytes).split(":")[0]; // "tenantId:entityId"

        if (premiumTenants.contains(tenantId)) {
            // Route premium tenant to last N partitions
            int hash = Utils.toPositive(Utils.murmur2(keyBytes));
            return regularPartitions + (hash % premiumPartitionCount);
        }

        int hash = Utils.toPositive(Utils.murmur2(keyBytes));
        return hash % regularPartitions;
    }

    @Override public void close() {}
}
```

---

## ⚡ 5. Key Partitioning Configs

| Config | Default | Explanation |
|--------|---------|-------------|
| `partitioner.class` | `DefaultPartitioner` | Class implementing `Partitioner` interface |
| `partitioner.ignore.keys` | `false` | If true, always use sticky partitioner (ignore keys) |
| `num.partitions` | `1` | Default partition count for auto-created topics |

**Choosing the right number of partitions:**

```
Factors to consider:
  1. Target throughput / throughput per partition
  2. Number of consumer instances planned (max = partition count)
  3. Number of brokers (partitions should be multiple of broker count)
  4. Key cardinality (low cardinality → skew risk)

Formula:
  partitions = max(
    targetThroughput / throughputPerPartition,
    maxConsumerInstances
  )

Round up to next multiple of broker count for even distribution.
```

---

## 🔄 6. Real-World Scenarios

### Scenario: Choosing the Right Key

```
Domain: E-commerce platform

BAD KEYS (low cardinality → hot partitions):
  ❌ event_type ("ORDER_CREATED", "ORDER_PAID") → 3 values, massive skew
  ❌ country ("US", "UK", "DE") → 3 values, US gets 80% of traffic
  ❌ status ("ACTIVE", "INACTIVE") → binary, extreme skew

GOOD KEYS (high cardinality → even distribution):
  ✅ order_id   → millions of unique orders → even spread
  ✅ user_id    → millions of users → even spread
  ✅ product_id → thousands of products → good spread
  ✅ session_id → unique per session → perfect distribution

SPECIAL CASES:
  ✅ composite: region + user_id → geo-aware, still high cardinality
  ✅ time-bucketed: hour + user_id → time-aware ordering
```

### Scenario: Hot Partition from Viral Event

```
Problem:
  Celebrity tweet mentions your product
  10M users all view product_id="FAMOUS-ITEM" in 1 hour
  Key = product_id → all 10M events → 1 partition → overloaded

Solutions:

Option 1: Salted Key
  Instead of key = product_id
  Use key = product_id + "_" + (random 0-9)
  → 10 sub-partitions get the traffic
  Cost: ordering lost for that product_id (all variants mix across partitions)

Option 2: Two-Tier Topics
  hot-products topic  (12 partitions, heavy consumers)
  cold-products topic (3 partitions, light consumers)
  Router logic determines which topic based on real-time popularity

Option 3: Time-Bucketed Key
  key = product_id + ":" + currentHourEpoch
  Different hour buckets → spread across partitions over time
  Ordering within an hour is maintained
```

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Low-cardinality key | Hot partitions, uneven load | Use high-cardinality keys (IDs, not status/type) |
| Changing partition count on live topic | Key routing breaks → ordering lost for existing keys | Plan partition count upfront; don't change on keyed topics |
| No key when ordering matters | Events for same entity split across partitions | Always use entity ID as key when ordering needed |
| Custom partitioner not handling nulls | NullPointerException in production | Always null-check key in custom partitioner |
| Hardcoded partition numbers | Brittle if partition count changes | Use key-based or custom logic, not hardcoded numbers |
| Ignoring key cardinality | Designing key that seems right but causes skew | Analyze key distribution before going to production |

---

## 🎯 8. Interview Questions

**Q1. How does Kafka decide which partition a message goes to?**
> If a key is provided, Kafka uses the murmur2 hash of the key bytes modulo the number of partitions: `abs(murmur2(key)) % numPartitions`. The same key always maps to the same partition (as long as partition count is unchanged). If no key is provided, Kafka 2.4+ uses sticky partitioning — filling one partition's batch before moving to the next — for better throughput.

**Q2. What is a hot partition and how do you fix it?**
> A hot partition is one that receives disproportionately more traffic than others, typically because the key has low cardinality (e.g., using "status" or "type" as key). Fixes include: choosing a higher-cardinality key (e.g., entity ID), adding a random salt suffix to spread traffic, or implementing a custom partitioner for business-specific routing.

**Q3. What happens to ordering when you increase the partition count?**
> The hash function changes because `hash(key) % oldPartitions ≠ hash(key) % newPartitions`. Keys that were consistently routed to partition N are now routed to a different partition. Ordering within a key is broken for messages produced after the change — old messages are in old partitions, new messages in new partitions. This is why partition count should be decided upfront.

**Q4. When would you write a custom partitioner?**
> Custom partitioners are used when business logic should drive routing: VIP customers to dedicated partitions, geo-based routing, tiered SLA routing, or when you need to co-locate related entities (e.g., all orders for a customer with that customer's profile events). The interface is `org.apache.kafka.clients.producer.Partitioner`.

**Q5. What is sticky partitioning and why was it introduced?**
> Before Kafka 2.4, null-key messages were distributed round-robin — one message per partition — resulting in many tiny batches with poor compression and throughput. Sticky partitioning fills a single partition's batch (up to `batch.size` or `linger.ms`) before switching to the next, producing larger, more efficient batches while still achieving even long-term distribution.

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "Design Partitioning for a Multi-Tenant SaaS"
> **Problem:** Your Kafka platform serves 500 tenants. 10 are "enterprise" (80% of traffic). 490 are "free tier" (20% of traffic). How do you design the partitioning strategy to give enterprise tenants low latency while free-tier tenants don't starve?

**Answer:**
```
Design: Separate Topics per Tier

Topic: enterprise-events (20 partitions, 20 fast consumers, acks=all)
Topic: free-events       (5 partitions,  5 slower consumers, acks=1)

Routing at producer level:
  if (tenant.tier == ENTERPRISE) → send to enterprise-events
  else → send to free-events

Within each topic: use tenantId as key
  → Each enterprise tenant's events are ordered
  → Even distribution across partitions

Consumer setup:
  Enterprise consumers: max.poll.records=50, fast processing SLA
  Free consumers: max.poll.records=500, batch processing acceptable

Alternative: Single topic with custom partitioner
  Topic: all-events (25 partitions)
    Partitions 0-19 → enterprise (TenantAwarePartitioner routes here)
    Partitions 20-24 → free tier
  Consumer group A: assigned to partitions 0-19 (enterprise)
  Consumer group B: assigned to partitions 20-24 (free tier)
  Note: manual partition assignment needed for consumer groups
```

---

### Scenario 2: "Global Ordering on a High-Volume Topic"
> **Problem:** You have a trading system where you need strict global ordering of all trades (not just per-symbol). The topic receives 1M msgs/sec. How do you achieve this with Kafka?

**Answer:**
```
True global ordering in Kafka = 1 partition = no parallelism.

Kafka guarantees ordering within a partition only.
To get global ordering: 1 partition → 1 consumer → 1M msgs/sec bottleneck.

Real answer: Kafka is the wrong tool for strict global ordering at 1M/sec.
But here's what you can do:

Option 1: Accept per-key ordering (usually sufficient)
  Each trade keyed by symbol: AAPL → same partition → ordered per symbol
  Most trading systems need per-symbol ordering, not global
  → 99% of use cases solved

Option 2: Sequence numbers at application level
  Producer assigns global sequence number (via distributed counter, Redis INCR)
  Consumer reorders by sequence number in a local buffer before processing
  → Soft global ordering with bounded reorder buffer
  → Handles gaps and out-of-order delivery

Option 3: Sequencer service
  All events pass through a single-threaded sequencer service
  Sequencer assigns monotonic seq# and writes to Kafka
  → Global ordering guaranteed, sequencer is the throughput bottleneck
  → Typical solution in financial systems for critical order flow
```

---

## 📝 10. Quick Revision Summary

```
✅ Key-based: abs(murmur2(key)) % numPartitions
   Same key → same partition (always, unless partition count changes)
   Use for: ordering per entity (orderId, userId, accountId)

✅ Null key: sticky partitioning (Kafka 2.4+)
   Fills one batch before switching partition
   Use for: logs, metrics, events where ordering doesn't matter

✅ Custom Partitioner: implements Partitioner interface
   Use for: VIP routing, geo-partitioning, tenant isolation, SLA tiers
   Always null-check key bytes in custom partitioner

✅ Manual partition: ProducerRecord(topic, partition, key, value)
   Bypasses all partitioner logic
   Use sparingly — brittle if partition count changes

✅ Hot partition causes:
   Low-cardinality keys (status, type, country)
   Fix: high-cardinality keys / salt suffix / custom partitioner

✅ Partition count rules:
   Can only INCREASE partitions (never decrease)
   Increasing breaks existing key→partition mapping
   Plan upfront: target_throughput / per_partition_throughput
   Multiple of broker count for even distribution

✅ Ordering rules:
   Within partition: GUARANTEED (strict sequence)
   Across partitions: NOT guaranteed
   Global ordering: requires 1 partition (no parallelism)
   Practical: use per-entity key for per-entity ordering
```

---

**← Previous:** [09 — Kafka Broker Internals](./09-broker-internals.md)  
**Next Topic →** [11 — Replication, ISR, Leader Election & Fault Tolerance](./11-replication.md)
