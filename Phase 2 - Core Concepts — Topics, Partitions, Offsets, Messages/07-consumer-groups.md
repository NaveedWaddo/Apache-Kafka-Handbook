# 07 — Consumer Groups & Partition Assignment Strategies
> **Phase 2 — Producers & Consumers**  
> 📁 `notes/07-consumer-groups.md`

---

## 📖 1. Concept Explanation

### What is a Consumer Group?

A **Consumer Group** is a set of consumer instances that **cooperate to consume a topic together**. Kafka distributes partitions across all members of the group — each partition is consumed by exactly **one member** at a time.

This is the mechanism that gives Kafka both:
- **Scalability** — add more consumers to process faster
- **Fault tolerance** — if one consumer dies, others take over its partitions

```
Topic: "order-events"  (6 partitions: P0–P5)

Consumer Group: "order-processor"
  Consumer A → P0, P1
  Consumer B → P2, P3
  Consumer C → P4, P5

Each partition is assigned to exactly ONE consumer in the group.
Every message in P0 is processed ONLY by Consumer A.
```

---

### Consumer Group vs Independent Consumers

```
SAME group-id = cooperative consumption (share the load)
─────────────────────────────────────────────────────────

Topic: "order-events" (6 partitions)

Group "billing":      C1→[P0,P1]  C2→[P2,P3]  C3→[P4,P5]
                      (each partition consumed once per group)

DIFFERENT group-id = independent consumption (each gets everything)
─────────────────────────────────────────────────────────────────────

Topic: "order-events" (6 partitions)

Group "billing":      C1→[P0..P5 full stream]
Group "inventory":    C1→[P0..P5 full stream]  ← separate offset
Group "analytics":    C1→[P0..P5 full stream]  ← separate offset

Each group independently reads ALL messages from ALL partitions.
Groups don't interfere with each other.
```

---

### The Partition-to-Consumer Limit

```
Consumers in group < Partitions:
  Some consumers handle multiple partitions
  C1→[P0,P1,P2]  C2→[P3,P4,P5]

Consumers in group = Partitions (IDEAL):
  Each consumer handles exactly one partition
  C1→[P0]  C2→[P1]  C3→[P2]  C4→[P3]  C5→[P4]  C6→[P5]

Consumers in group > Partitions (WASTEFUL):
  Extra consumers sit IDLE — they get no partitions
  C1→[P0]  C2→[P1]  C3→[P2]  C4→[P3]  C5→[P4]  C6→[P5]
  C7→[IDLE]  C8→[IDLE]  ← wasted resources

Rule: Maximum parallelism = number of partitions
      You can never have more active consumers than partitions
```

---

### Group Coordinator & Group Leader

Two special roles inside a consumer group:

**Group Coordinator (Broker-side):**
- A broker elected to manage the group's lifecycle
- Tracks heartbeats from all consumers
- Detects failures (missed heartbeats)
- Triggers rebalances when membership changes
- Stores committed offsets in `__consumer_offsets`

**Group Leader (Consumer-side):**
- One consumer instance elected as leader within the group
- Receives the full list of members and their metadata
- Runs the **partition assignment algorithm**
- Sends assignments back to the Group Coordinator
- Any consumer can be the leader — it rotates on rebalance

```
Broker Side:                     Consumer Side:
┌──────────────────┐              ┌─────────────────────┐
│  Group           │              │  Group Leader (C1)   │
│  Coordinator     │◄────────────►│  Runs assignor algo  │
│  (Broker N)      │              │  Returns assignments │
│                  │              └─────────────────────┘
│  Tracks:         │              ┌─────────────────────┐
│  - Members       │◄────────────►│  Consumer C2         │
│  - Heartbeats    │              └─────────────────────┘
│  - Offsets       │              ┌─────────────────────┐
└──────────────────┘◄────────────►│  Consumer C3         │
                                  └─────────────────────┘
```

---

## 🏗️ 2. Rebalance — The Core Mechanism

### What Triggers a Rebalance?

A **rebalance** is the process of reassigning partitions among consumers in the group. It is triggered when:

```
1. New consumer joins the group
2. Consumer leaves gracefully (shutdown)
3. Consumer crashes (heartbeat timeout)
4. Topic partition count changes
5. Consumer subscribes to new topics (pattern subscription)
6. session.timeout.ms exceeded (consumer considered dead)
```

### Eager Rebalance (Stop-the-World) — Old Default

```
Phase 1 — REVOKE ALL
  All consumers stop consuming
  All partitions are revoked from ALL consumers
  Group enters "PreparingRebalance" state

  [C1: was P0,P1] → revoke → [idle]
  [C2: was P2,P3] → revoke → [idle]
  [C3: was P4,P5] → revoke → [idle]

Phase 2 — REJOIN
  All consumers rejoin the group
  Leader runs assignment algorithm
  New assignments distributed

  [C1: gets P0,P1,P2]
  [C2: gets P3,P4,P5]
  (C3 left, so C1 and C2 absorb its partitions)

Phase 3 — RESUME
  Consumers resume from committed offsets

Problem: During rebalance, ALL consumption stops
         Even consumers unaffected by the change stop
         This "stop the world" causes latency spikes
```

### Cooperative (Incremental) Rebalance — Modern Default

```
Kafka 2.4+ introduced CooperativeStickyAssignor:

Phase 1 — CALCULATE DIFF
  Leader computes what needs to change
  Only revokes partitions that need to move

Phase 2 — PARTIAL REVOKE
  Only affected consumers revoke specific partitions
  Unaffected consumers KEEP consuming their partitions!

Phase 3 — REASSIGN
  Revoked partitions redistributed to new owner
  Consumer that left: partitions distributed to others

C3 joins group:
  Before: C1→[P0,P1,P2]  C2→[P3,P4,P5]
  Revoke: C1 gives up P2 only
  After:  C1→[P0,P1]  C2→[P3,P4,P5]  C3→[P2] ← only one partition moved!
  C1 and C2 never stopped consuming P0,P1,P3,P4,P5

Advantage: No stop-the-world. Minimal disruption.
```

---

## ⚙️ 3. Partition Assignment Strategies

### Strategy 1: RangeAssignor (Default for older clients)

Assigns contiguous ranges of partitions per topic:

```
Topic A: 6 partitions (P0–P5)
Topic B: 4 partitions (P0–P3)
Consumers: C1, C2, C3

RangeAssignor (per topic, alphabetical order):

Topic A:  C1→[P0,P1]  C2→[P2,P3]  C3→[P4,P5]  (6÷3=2 each)
Topic B:  C1→[P0,P1]  C2→[P2,P3]  C3→[]        (4÷3, C3 gets nothing)

Problem: C1 always gets the "first" partitions of every topic
         → C1 gets more partitions when topic count doesn't divide evenly
         → UNEVEN LOAD across consumers
```

### Strategy 2: RoundRobinAssignor

Assigns partitions in round-robin across all topics:

```
All partitions across all topics, sorted:
  TopicA-P0, TopicA-P1, TopicA-P2, TopicA-P3, TopicA-P4, TopicA-P5,
  TopicB-P0, TopicB-P1, TopicB-P2, TopicB-P3

Round-robin to C1, C2, C3:
  C1 → TopicA-P0, TopicA-P3, TopicB-P0, TopicB-P3
  C2 → TopicA-P1, TopicA-P4, TopicB-P1
  C3 → TopicA-P2, TopicA-P5, TopicB-P2

Better balance! But on rebalance, many partitions move
```

### Strategy 3: StickyAssignor

Like RoundRobin but **preserves existing assignments** as much as possible during rebalance:

```
Before rebalance (C3 leaves):
  C1 → [P0, P3]
  C2 → [P1, P4]
  C3 → [P2, P5]  ← leaves

StickyAssignor tries to keep C1's P0,P3 and C2's P1,P4:
  C1 → [P0, P3, P2]   (kept P0,P3 + took P2 from C3)
  C2 → [P1, P4, P5]   (kept P1,P4 + took P5 from C3)

Minimizes partition movement → less duplicate processing after rebalance
```

### Strategy 4: CooperativeStickyAssignor ✅ (Recommended)

Same assignment logic as StickyAssignor but uses the **cooperative rebalance protocol** — partitions are only revoked if they truly need to move. **This is the recommended strategy for all new applications.**

```
partition.assignment.strategy=
  org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

### Strategy Comparison

| Strategy | Load Balance | Rebalance Impact | Partition Movement | Protocol |
|----------|-------------|-----------------|-------------------|---------|
| RangeAssignor | ⚠️ Uneven | Stop-the-world | High | Eager |
| RoundRobinAssignor | ✅ Even | Stop-the-world | High | Eager |
| StickyAssignor | ✅ Even | Stop-the-world | Low | Eager |
| **CooperativeStickyAssignor** | ✅ Even | **Incremental** | **Minimal** | **Cooperative** |

---

## 💻 4. Code Examples (Java + Spring Boot)

### `application.yml` — Consumer Group Config

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: order-processor
      auto-offset-reset: earliest
      enable-auto-commit: false
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      max-poll-records: 50
      properties:
        # Use cooperative rebalance (recommended)
        partition.assignment.strategy: >
          org.apache.kafka.clients.consumer.CooperativeStickyAssignor

        # Heartbeat & session timeouts
        heartbeat.interval.ms: 3000         # Send heartbeat every 3s
        session.timeout.ms: 45000           # Dead after 45s of no heartbeat
        max.poll.interval.ms: 300000        # Max time between polls (5 min)

        # Deserialization
        spring.json.trusted.packages: "com.example.kafka.model"
```

---

### Basic Consumer Group Listener

```java
package com.example.kafka.consumer;

import com.example.kafka.model.OrderEvent;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Service;

@Service
public class OrderEventConsumer {

    // All instances with same groupId share the partitions
    @KafkaListener(
        topics = "order-events",
        groupId = "order-processor",
        concurrency = "3"    // 3 listener threads = up to 3 partitions per instance
    )
    public void consume(
            ConsumerRecord<String, OrderEvent> record,
            Acknowledgment ack) {

        System.out.printf(
            "[Group: order-processor] Partition=%d Offset=%d Key=%s%n",
            record.partition(), record.offset(), record.key()
        );

        try {
            processOrder(record.value());
            ack.acknowledge();   // Commit offset after successful processing
        } catch (Exception e) {
            System.err.println("Processing failed: " + e.getMessage());
            // Don't ACK → message will be redelivered on restart
        }
    }

    private void processOrder(OrderEvent event) {
        // Business logic
    }
}
```

---

### Multiple Consumer Groups on Same Topic

```java
package com.example.kafka.consumer;

import com.example.kafka.model.OrderEvent;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

@Service
public class MultiGroupConsumers {

    // Group 1: Billing service reads ALL order events
    @KafkaListener(topics = "order-events", groupId = "billing-service")
    public void billingConsumer(ConsumerRecord<String, OrderEvent> record) {
        System.out.println("[BILLING] Processing: " + record.value().getOrderId());
    }

    // Group 2: Inventory service ALSO reads ALL order events (independent)
    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void inventoryConsumer(ConsumerRecord<String, OrderEvent> record) {
        System.out.println("[INVENTORY] Processing: " + record.value().getOrderId());
    }

    // Group 3: Analytics — also reads everything independently
    @KafkaListener(topics = "order-events", groupId = "analytics-service")
    public void analyticsConsumer(ConsumerRecord<String, OrderEvent> record) {
        System.out.println("[ANALYTICS] Processing: " + record.value().getOrderId());
    }
}
```

---

### Partition Assignment Listener (Monitor Rebalances)

```java
package com.example.kafka.consumer;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.TopicPartition;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.listener.ConsumerSeekAware;
import org.springframework.stereotype.Service;

import java.util.Collection;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class RebalanceAwareConsumer implements ConsumerSeekAware {

    @Override
    public void onPartitionsAssigned(
            Map<TopicPartition, Long> assignments,
            ConsumerSeekCallback callback) {

        String assigned = assignments.keySet().stream()
            .map(tp -> tp.topic() + "-" + tp.partition())
            .collect(Collectors.joining(", "));

        System.out.println("✅ [REBALANCE] Partitions ASSIGNED: " + assigned);
    }

    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        String revoked = partitions.stream()
            .map(tp -> tp.topic() + "-" + tp.partition())
            .collect(Collectors.joining(", "));

        System.out.println("⚠️ [REBALANCE] Partitions REVOKED: " + revoked);
        // Important: commit offsets here before losing partition ownership!
    }

    @Override
    public void onPartitionsLost(Collection<TopicPartition> partitions) {
        // Called during cooperative rebalance when partitions lost unexpectedly
        System.out.println("❌ [REBALANCE] Partitions LOST: " + partitions);
    }

    @KafkaListener(topics = "order-events", groupId = "rebalance-demo")
    public void consume(ConsumerRecord<String, String> record) {
        System.out.printf("P%d O%d: %s%n",
            record.partition(), record.offset(), record.value());
    }
}
```

---

### Programmatic Consumer Group Inspection (Admin API)

```java
package com.example.kafka.admin;

import org.apache.kafka.clients.admin.*;
import org.apache.kafka.clients.consumer.OffsetAndMetadata;
import org.apache.kafka.common.TopicPartition;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
public class ConsumerGroupInspector {

    private final KafkaAdmin kafkaAdmin;

    public ConsumerGroupInspector(KafkaAdmin kafkaAdmin) {
        this.kafkaAdmin = kafkaAdmin;
    }

    public void describeGroup(String groupId) throws Exception {
        try (AdminClient client =
                AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            // Describe consumer group (members, state, assignment)
            Map<String, ConsumerGroupDescription> groups =
                client.describeConsumerGroups(List.of(groupId))
                      .all().get();

            ConsumerGroupDescription desc = groups.get(groupId);
            System.out.println("Group ID    : " + desc.groupId());
            System.out.println("State       : " + desc.state());     // Stable, Rebalancing, etc.
            System.out.println("Coordinator : " + desc.coordinator());
            System.out.println("Assignor    : " + desc.partitionAssignor());

            System.out.println("\nMembers:");
            for (MemberDescription member : desc.members()) {
                System.out.printf(
                    "  Member: %s | Host: %s | Partitions: %s%n",
                    member.consumerId(),
                    member.host(),
                    member.assignment().topicPartitions()
                );
            }
        }
    }

    public void printConsumerLag(String groupId, String topic) throws Exception {
        try (AdminClient client =
                AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            // Get committed offsets for this group
            Map<TopicPartition, OffsetAndMetadata> committed =
                client.listConsumerGroupOffsets(groupId)
                      .partitionsToOffsetAndMetadata().get();

            // Get latest end offsets for the topic
            List<TopicPartition> partitions = committed.keySet().stream()
                .filter(tp -> tp.topic().equals(topic))
                .toList();

            Map<TopicPartition, Long> endOffsets =
                client.listOffsets(
                    partitions.stream().collect(
                        java.util.stream.Collectors.toMap(
                            tp -> tp,
                            tp -> OffsetSpec.latest()
                        )
                    )
                ).all().get().entrySet().stream()
                .collect(java.util.stream.Collectors.toMap(
                    Map.Entry::getKey,
                    e -> e.getValue().offset()
                ));

            System.out.printf("\nConsumer Lag for group=%s topic=%s:%n",
                groupId, topic);
            System.out.printf("%-12s %-15s %-15s %-10s%n",
                "Partition", "Committed", "End Offset", "LAG");

            long totalLag = 0;
            for (TopicPartition tp : partitions) {
                long committedOffset = committed.get(tp).offset();
                long endOffset = endOffsets.get(tp);
                long lag = endOffset - committedOffset;
                totalLag += lag;
                System.out.printf("%-12d %-15d %-15d %-10d%n",
                    tp.partition(), committedOffset, endOffset, lag);
            }
            System.out.println("TOTAL LAG: " + totalLag);
        }
    }
}
```

---

### Static Group Membership (Avoid Rebalances on Restart)

```java
// application.yml — static membership
spring:
  kafka:
    consumer:
      group-id: order-processor
      properties:
        # Static member ID — this consumer always gets same partitions
        # No rebalance triggered if this consumer restarts within session.timeout.ms
        group.instance.id: order-processor-instance-1

        # Give it time to reconnect before declaring dead
        session.timeout.ms: 60000
```

```java
@KafkaListener(
    topics = "order-events",
    groupId = "order-processor",
    // In Spring Kafka, set via consumer properties
    properties = {"group.instance.id=order-processor-pod-1"}
)
public void consume(ConsumerRecord<String, OrderEvent> record) {
    // This consumer will reconnect and resume the SAME partitions
    // as long as it restarts within session.timeout.ms
}
```

> **Static membership** is critical for Kubernetes deployments — pod restarts no longer trigger expensive rebalances.

---

## ⚡ 5. Key Configurations

| Config | Default | Recommended | Explanation |
|--------|---------|-------------|-------------|
| `group.id` | `""` | Required | Consumer group name |
| `partition.assignment.strategy` | `RangeAssignor` | `CooperativeStickyAssignor` | How partitions are assigned |
| `heartbeat.interval.ms` | `3000` | `3000` | How often to send heartbeat |
| `session.timeout.ms` | `45000` | `45000` | Time before consumer considered dead |
| `max.poll.interval.ms` | `300000` | Tune to processing time | Max time between polls before kicked out |
| `group.instance.id` | `null` | Set in K8s | Static membership — prevents restart rebalances |
| `max.poll.records` | `500` | `50–200` | Records returned per poll call |

### The Timeout Triangle

```
heartbeat.interval.ms < session.timeout.ms << max.poll.interval.ms

heartbeat.interval.ms = 3000   (send heartbeat every 3s)
session.timeout.ms    = 45000  (dead if no heartbeat for 45s)
max.poll.interval.ms  = 300000 (kicked out if processing takes > 5min)

If your processing is slow (> max.poll.interval.ms):
  → Consumer is kicked from group
  → Rebalance triggered
  → Fix: reduce max.poll.records OR increase max.poll.interval.ms
```

---

## 🔄 6. Real-World Scenarios

### Scenario: Kubernetes Pod Autoscaling

```
Topic: "order-events" (12 partitions)
Service: order-processor (Kubernetes Deployment)

Normal load: 3 pods → each gets 4 partitions
  Pod1 → P0,P1,P2,P3
  Pod2 → P4,P5,P6,P7
  Pod3 → P8,P9,P10,P11

Peak load: scale to 6 pods
  HPA triggers → 3 new pods added
  Rebalance: each pod gets 2 partitions
  Pod1→[P0,P1]  Pod2→[P2,P3]  Pod3→[P4,P5]
  Pod4→[P6,P7]  Pod5→[P8,P9]  Pod6→[P10,P11]

After-hours: scale back to 3 pods
  3 pods removed → rebalance → 4 partitions each again

Key insight:
  Partition count (12) = max parallelism
  Never deploy more pods than partitions (extras sit idle)
```

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| More consumers than partitions | Extra consumers idle, wasting resources | partitions ≥ max expected consumers |
| Using RangeAssignor with multiple topics | Uneven load distribution | Use CooperativeStickyAssignor |
| Long processing in `@KafkaListener` | Exceeds `max.poll.interval.ms` → kicked from group → rebalance | Offload to async thread or increase `max.poll.interval.ms` |
| No `group.instance.id` in K8s | Every pod restart triggers full rebalance | Set static `group.instance.id` per pod |
| All consumers in one JVM process | Single process death kills all parallelism | Deploy across multiple pods/machines |
| Not handling `onPartitionsRevoked` | In-flight work lost during rebalance | Commit offsets in `onPartitionsRevoked` |
| Ignoring consumer lag | Silent backlog builds up unnoticed | Monitor `consumer_lag` metric, alert on high lag |

---

## 🎯 8. Interview Questions

**Q1. What is a consumer group and what problem does it solve?**
> A consumer group is a set of consumer instances sharing a `group.id` that cooperatively consume a topic. Kafka assigns each partition to exactly one consumer in the group, enabling **parallel processing** at scale. Multiple groups can independently consume the same topic — each group gets its own complete copy of the data.

**Q2. What happens during a Kafka consumer group rebalance?**
> A rebalance redistributes topic partitions among consumers in the group. With eager rebalance (old default), all consumers stop, all partitions are revoked, and then reassigned — causing a "stop the world" pause. With cooperative/incremental rebalance (Kafka 2.4+, CooperativeStickyAssignor), only the affected partitions are moved, and unaffected consumers keep processing.

**Q3. What is the difference between `session.timeout.ms` and `max.poll.interval.ms`?**
> `session.timeout.ms` is the time Kafka waits for a consumer's heartbeat before declaring it dead (heartbeats are sent by a background thread). `max.poll.interval.ms` is the maximum time between two `poll()` calls before the consumer is kicked from the group. The second catches cases where the consumer is alive but stuck processing a batch (no `poll()` call).

**Q4. What is static group membership and when should you use it?**
> Static membership assigns a permanent `group.instance.id` to a consumer. When it restarts within `session.timeout.ms`, it rejoins with the same ID and is reassigned its previous partitions — **no rebalance triggered**. This is essential in Kubernetes where pod restarts are frequent and every rebalance causes processing delays.

**Q5. What happens if you have more consumers than partitions?**
> Extra consumers in the group sit completely idle — they receive no partition assignments and process no messages. This is wasted resources. The maximum useful parallelism equals the partition count.

**Q6. What is the role of the Group Coordinator vs the Group Leader?**
> The **Group Coordinator** is a broker responsible for managing group membership, tracking heartbeats, and storing committed offsets. The **Group Leader** is one consumer instance (elected by the coordinator) responsible for running the partition assignment algorithm and sending the assignment plan back to the coordinator.

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "Rebalance Storm in Production"
> **Problem:** Your Kubernetes service has 10 pods consuming from a 20-partition topic. During deployments, every rolling restart triggers a rebalance, causing 10–30 seconds of processing downtime. How do you eliminate this?

**Answer:**
```
Problem: Every pod restart = leave group = rebalance for everyone

Solution 1: Static Group Membership
  Set group.instance.id = pod name (unique per pod, stable across restarts)
  properties:
    group.instance.id: ${HOSTNAME}   # Kubernetes sets HOSTNAME = pod name
    session.timeout.ms: 60000        # Give pod 60s to restart and rejoin

  Result: Pod restarts don't trigger rebalance as long as they come back
          within session.timeout.ms

Solution 2: CooperativeStickyAssignor (always use this)
  Even when rebalances DO happen, only affected partitions move
  Other consumers keep processing uninterrupted

Solution 3: Rolling deploy with maxUnavailable=1
  Only one pod down at a time
  Minimal partition reshuffling

Best Practice: Use ALL THREE together.
```

---

### Scenario 2: "Consumer Keeps Getting Kicked from Group"
> **Problem:** Your consumer processes each message in 5 minutes (heavy ML inference). It keeps getting kicked from the consumer group with the error `Consumer poll timeout`. How do you fix it?

**Answer:**
```
Root cause: max.poll.interval.ms (default 5 min) exceeded
  Consumer fetches batch → takes too long to process → misses next poll()
  Kafka coordinator declares consumer dead → rebalance

Solutions:

Option A: Increase max.poll.interval.ms
  max.poll.interval.ms: 600000   # 10 minutes
  Risk: slow failure detection if consumer truly dies

Option B: Reduce max.poll.records (process fewer at a time)
  max.poll.records: 1            # One at a time
  Each poll is fast → poll() called again quickly

Option C: Async processing (BEST)
  @KafkaListener receives record
  Submits to thread pool for processing
  poll() returns immediately → no timeout
  Use manual offset commit AFTER async task completes

  // Pseudocode
  @KafkaListener(...)
  public void consume(ConsumerRecord record, Acknowledgment ack) {
      CompletableFuture
          .runAsync(() -> heavyMLInference(record.value()), executorPool)
          .thenRun(() -> ack.acknowledge());
  }
  // Warning: ordering not guaranteed with this approach
```

---

### Scenario 3: "Consumer Lag Growing Indefinitely"
> **Problem:** Your consumer group has 3 instances consuming a 12-partition topic. Consumer lag is growing at 50,000 msgs/sec. What are your options?

**Answer:**
```
Diagnosis:
  Production rate > consumption rate = lag grows

Option 1: Scale out (if partitions allow)
  Currently:  3 consumers × 4 partitions each
  Scale to:   12 consumers × 1 partition each (max parallelism)
  → 4x more consumers → 4x throughput

  But: topic has 12 partitions → max 12 consumers
  If already at 12 consumers and still lagging → Option 2

Option 2: Increase partition count
  kafka-topics.sh --alter --partitions 24
  → Now can scale to 24 consumers
  Warning: key-based ordering disrupted during transition

Option 3: Optimize consumer processing
  Profile what takes time in the listener
  Batch DB writes instead of one-by-one inserts
  Use async I/O (WebClient instead of RestTemplate)

Option 4: Separate fast and slow consumers
  Create separate consumer group for real-time (SLA-sensitive)
  Create separate consumer group for batch/analytics (lag-tolerant)
  Fast group gets priority resources

Monitoring:
  Alert when lag > 100,000 messages or lag is growing over 5 min window
```

---

## 📝 10. Quick Revision Summary

```
✅ Consumer Group = set of consumers sharing group.id
   Each partition → assigned to exactly ONE consumer in group
   Max parallelism = number of partitions (extras are idle)

✅ Multiple groups = each gets independent copy of full stream
   Different group.id = separate offsets = independent processing

✅ Rebalance triggers: join, leave, crash, timeout, partition change
   Eager: stop-the-world (all partitions revoked and reassigned)
   Cooperative: incremental (only moved partitions affected)

✅ Assignment Strategies:
   RangeAssignor        → uneven load, eager (avoid)
   RoundRobinAssignor   → even load, eager (better)
   StickyAssignor       → even + minimal movement, eager
   CooperativeStickyAssignor → even + minimal + cooperative ✅ USE THIS

✅ Timeout trio:
   heartbeat.interval.ms (3s) < session.timeout.ms (45s) << max.poll.interval.ms (5min)

✅ Static membership (group.instance.id):
   Pod restarts don't trigger rebalance within session.timeout.ms
   Critical for Kubernetes deployments

✅ Group Coordinator = broker managing membership + offsets
✅ Group Leader     = one consumer running partition assignment algorithm
```

---

**← Previous:** [06 — Kafka Consumers: Poll Loop, Offsets & Commits](./06-consumers.md)  
**Next Topic →** [08 — Message Delivery Semantics: At Most / At Least / Exactly Once](./08-delivery-semantics.md)
