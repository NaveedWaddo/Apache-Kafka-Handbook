# 11 — Replication, ISR, Leader Election & Fault Tolerance
> **Phase 3 — Kafka Internals & Storage**  
> 📁 `notes/11-replication.md`

---

## 📖 1. Concept Explanation

### Why Replication Exists

Kafka runs on commodity hardware that **will fail**. Replication is Kafka's answer to hardware failure — instead of a single copy of data that can be lost, every partition is copied to multiple brokers.

> **Without replication:** One broker dies → all partitions on that broker are gone → data loss + consumer starvation.  
> **With replication:** One broker dies → replicas on other brokers take over → zero data loss + automatic recovery.

Replication is what makes Kafka a **durable, fault-tolerant** system. Understanding it deeply tells you exactly what you can and cannot lose in various failure scenarios.

---

### Core Replication Terminology

```
Replication Factor (RF):
  The number of copies of each partition.
  RF=3 → 3 copies: 1 leader + 2 followers.

Leader Replica:
  The one replica that handles ALL reads and writes for a partition.
  There is exactly 1 leader per partition at any time.

Follower Replica:
  Passive copies that replicate from the leader.
  They ONLY fetch from the leader — they don't serve client requests.
  (Kafka 2.4+ added follower fetching as an opt-in feature for latency)

ISR (In-Sync Replicas):
  The set of replicas that are fully caught up with the leader.
  A replica is "in sync" if it has fetched all messages from the leader
  within replica.lag.time.max.ms (default 30s).

AR (Assigned Replicas):
  All replicas assigned to a partition (leader + all followers).
  AR = ISR + OSR (out-of-sync replicas)
```

---

### Replication Factor Design

```
RF = 1  → No redundancy. Single point of failure. Dev/test only.
RF = 2  → One failure tolerated. Minimal. Not recommended for production.
RF = 3  → One failure tolerated with ISR still active. PRODUCTION STANDARD.
RF = 5  → Two failures tolerated. Used for critical financial topics.

Rule: Can tolerate (RF - 1) simultaneous broker failures without data loss.

With RF=3 and min.insync.replicas=2:
  Tolerate 1 broker failure: ISR=[2 brokers] → writes still accepted ✅
  Two brokers fail: ISR=[1 broker] < min.insync.replicas=2 → writes rejected ❌
  (but reads still work from remaining leader)
```

---

## 🏗️ 2. Architecture / Flow

### Replication Layout Across Brokers

```
Topic: "payments" — 3 partitions, RF=3, 3 brokers

                Broker 1        Broker 2        Broker 3
              ┌──────────┐    ┌──────────┐    ┌──────────┐
Partition 0:  │  LEADER  │    │ follower │    │ follower │
              └──────────┘    └──────────┘    └──────────┘

              ┌──────────┐    ┌──────────┐    ┌──────────┐
Partition 1:  │ follower │    │  LEADER  │    │ follower │
              └──────────┘    └──────────┘    └──────────┘

              ┌──────────┐    ┌──────────┐    ┌──────────┐
Partition 2:  │ follower │    │ follower │    │  LEADER  │
              └──────────┘    └──────────┘    └──────────┘

Each broker is leader for 1 partition, follower for 2 others.
Leadership is spread evenly → no broker is a hot spot.
This is called "preferred replica" distribution.
```

---

### The Replication Flow (Step by Step)

```
Step 1: Producer sends to Partition 0 leader (Broker 1)
  Producer → Broker 1: ProduceRequest{msgs}

Step 2: Leader writes to its local log
  Broker 1 appends to /kafka-logs/payments-0/*.log
  Broker 1 LEO advances to, say, offset 101

Step 3: Followers fetch from leader
  Broker 2 (follower): sends FetchRequest to Broker 1
    "Give me messages from offset 99 onwards for payments-P0"
  Broker 3 (follower): sends FetchRequest to Broker 1
    "Give me messages from offset 98 onwards for payments-P0"

Step 4: Leader tracks follower progress
  Broker 1 tracks each follower's LEO (from their FetchRequests)
  ISR membership based on: follower hasn't lagged > replica.lag.time.max.ms

Step 5: High Watermark advances
  Once ALL ISR members have fetched offset 101:
  Leader sets HW = 101
  Leader includes HW in next FetchResponse to followers

Step 6: Followers update their HW
  Followers learn the new HW from leader's response
  HW = 101 → consumers can now read offset 101

Step 7: Leader ACKs producer (for acks=all)
  Once all ISR members have LEO >= 101:
  Broker 1 → Producer: ProduceResponse{offset=101, success}
```

---

### ISR Membership — Who's In, Who's Out

```
ISR membership is dynamic:

Follower REMOVED from ISR when:
  → replica.lag.time.max.ms elapsed without a fetch from that follower
  → Default: 30 seconds
  → Controller updates partition metadata: ISR shrinks

Follower RE-ADDED to ISR when:
  → Follower catches up fully (its LEO = leader's LEO)
  → It must fetch ALL missing messages and write them to disk
  → Then it sends confirmation → Controller adds it back to ISR

Tracking mechanism:
  Leader maintains lastCaughtUpTimeMs per follower
  If (now - lastCaughtUpTimeMs) > replica.lag.time.max.ms → remove from ISR
```

---

### Leader Election — When a Leader Dies

```
Normal state:
  Partition 0: Leader=Broker1  ISR=[B1, B2, B3]

Broker 1 crashes:

Step 1: Detection
  ZooKeeper/KRaft: Broker 1 session expires (no heartbeat)
  Controller notified: "Broker 1 is dead"

Step 2: Controller identifies affected partitions
  Find all partitions where Broker 1 was the leader
  → Partition 0 needs a new leader

Step 3: Elect new leader from ISR
  ISR = [B2, B3]  (B1 removed because it's dead)
  Controller picks first available ISR member: Broker 2
  (Or uses preferred replica priority if configured)

Step 4: Controller broadcasts new assignment
  Sends LeaderAndIsr request to all brokers:
  "Partition 0: new leader = Broker 2, ISR = [B2, B3]"

Step 5: Clients discover new leader
  Clients get LEADER_NOT_AVAILABLE error on next request
  They refresh metadata → learn Broker 2 is new leader
  Reconnect to Broker 2 → resume producing/consuming

Step 6: Recovery
  Broker 1 restarts
  It fetches missing messages from Broker 2 (new leader)
  Once fully caught up → re-added to ISR
  Partition 0: Leader=Broker2, ISR=[B2, B3, B1]
  (B1 is no longer the "preferred" leader — requires manual preferred-leader-election
   or auto.leader.rebalance.enable=true)
```

---

### Unclean Leader Election

```
What if ALL ISR members are down?

Only out-of-sync replicas remain.

unclean.leader.election.enable=false (DEFAULT, recommended):
  → Partition goes OFFLINE
  → Reads and writes fail with LEADER_NOT_AVAILABLE
  → Wait for an ISR member to come back
  → No data loss — never elect stale replica

unclean.leader.election.enable=true (DANGEROUS):
  → Pick the most up-to-date out-of-sync replica as new leader
  → It becomes leader even though it's missing some messages
  → Messages it doesn't have are LOST FOREVER (they existed on dead leader)
  → Partition comes back online but with DATA LOSS

Production rule: NEVER enable unclean leader election for financial/critical data
  Only consider for: metrics, logs, telemetry where some loss is acceptable
```

---

## ⚙️ 3. How It Works Internally

### Follower Fetch Loop

```
Follower runs a continuous fetch loop (ReplicaFetcherThread):

while (true) {
  1. Send FetchRequest to leader
     partition=0, fetchOffset=myLEO, maxBytes=replicaFetchMaxBytes

  2. Leader returns messages from myLEO onwards + current HW

  3. Follower appends received messages to local log
     Updates own LEO

  4. Follower updates local HW to min(own LEO, received HW)

  5. Leader updates followerLEO[followerId] from FetchRequest offset
     This is how leader knows follower is caught up

  6. repeat
}
```

### Epoch-Based Leader Tracking (Leader Epoch)

```
Problem without leader epoch:
  Old leader writes msg at offset 100
  Old leader crashes, never replicates offset 100
  New leader elected (from ISR, at offset 99)
  Old leader restarts, thinks it's still the leader
  Two "leaders" writing to the same partition → split-brain

Solution: Leader Epoch
  Every time a new leader is elected, its EPOCH increments
  Epoch = monotonically increasing integer per partition

  Old leader comes back:
    It checks its leader epoch with the controller
    Controller: "Epoch 5 is the current leader, you are epoch 4"
    Old leader: truncates log back to where epoch 5 started
    Rejoins as follower → fetches from epoch 5 leader

  This prevents log divergence (different messages at same offset)
```

### Log Truncation on Rejoin

```
After Broker 1 (old leader) restarts and rejoins as follower:

Its log might have "extra" messages that the new leader doesn't have
(messages that were written to old leader but never replicated)

These must be TRUNCATED before syncing:

Old Broker 1 log: [0][1][2][3][4][5][6][7][8][9][10]  ← had 11 messages
New Broker 2 log: [0][1][2][3][4][5][6][7][8]           ← only 9 (was the HW)

Broker 1 rejoining:
  Contacts Broker 2: "What offset did epoch 4 end at?"
  Broker 2: "Epoch 4 ended at offset 8"
  Broker 1 truncates: [0][1][2][3][4][5][6][7][8]  ← offsets 9,10 deleted
  Broker 1 fetches from offset 9 onwards from Broker 2
  Sync complete → rejoins ISR
```

---

## 💻 4. Code Examples (Java + Spring Boot)

### Application Config for Maximum Durability

```yaml
# application.yml — zero data loss configuration
spring:
  kafka:
    bootstrap-servers: broker1:9092,broker2:9092,broker3:9092
    producer:
      acks: all                        # Wait for ALL ISR to acknowledge
      retries: 2147483647              # Retry forever within delivery.timeout.ms
      delivery-timeout-ms: 120000
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
    consumer:
      properties:
        isolation.level: read_committed
```

---

### Topic Creation with Replication Config

```java
package com.example.kafka.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class ReplicatedTopicConfig {

    // Standard production topic: RF=3, min ISR=2
    @Bean
    public NewTopic paymentsTopic() {
        return TopicBuilder.name("payments")
                .partitions(12)
                .replicas(3)
                .config("min.insync.replicas", "2")
                .config("unclean.leader.election.enable", "false")
                .build();
    }

    // Critical financial topic: RF=5, min ISR=3
    @Bean
    public NewTopic ledgerTopic() {
        return TopicBuilder.name("ledger-events")
                .partitions(6)
                .replicas(5)
                .config("min.insync.replicas", "3")
                .config("unclean.leader.election.enable", "false")
                .config("retention.ms", "-1")  // Keep forever
                .build();
    }

    // High-throughput metrics topic: RF=2, lower durability for speed
    @Bean
    public NewTopic metricsTopic() {
        return TopicBuilder.name("system-metrics")
                .partitions(24)
                .replicas(2)
                .config("min.insync.replicas", "1")
                .config("retention.ms", "86400000")  // 1 day
                .build();
    }
}
```

---

### Monitor Replication Health via AdminClient

```java
package com.example.kafka.admin;

import org.apache.kafka.clients.admin.*;
import org.apache.kafka.common.TopicPartitionInfo;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.util.*;

@Component
public class ReplicationHealthMonitor {

    private final KafkaAdmin kafkaAdmin;

    public ReplicationHealthMonitor(KafkaAdmin kafkaAdmin) {
        this.kafkaAdmin = kafkaAdmin;
    }

    @Scheduled(fixedDelay = 30000) // Check every 30 seconds
    public void checkReplicationHealth() {
        try (AdminClient client =
                AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            // Get all topics
            Set<String> topics = client.listTopics().names().get();

            // Filter out internal topics
            List<String> userTopics = topics.stream()
                .filter(t -> !t.startsWith("__"))
                .toList();

            Map<String, TopicDescription> descriptions =
                client.describeTopics(userTopics).allTopicNames().get();

            int totalPartitions = 0;
            int underReplicated = 0;
            int offline = 0;

            for (TopicDescription desc : descriptions.values()) {
                for (TopicPartitionInfo partition : desc.partitions()) {
                    totalPartitions++;
                    int replicaCount = partition.replicas().size();
                    int isrCount = partition.isr().size();

                    if (partition.leader() == null) {
                        offline++;
                        System.err.printf(
                            "🔴 OFFLINE: %s-%d (no leader!)%n",
                            desc.name(), partition.partition());
                    } else if (isrCount < replicaCount) {
                        underReplicated++;
                        System.err.printf(
                            "⚠️  UNDER-REPLICATED: %s-%d " +
                            "(replicas=%d isr=%d leader=Broker%d)%n",
                            desc.name(), partition.partition(),
                            replicaCount, isrCount,
                            partition.leader().id());
                    }
                }
            }

            System.out.printf(
                "[HEALTH] partitions=%d under-replicated=%d offline=%d%n",
                totalPartitions, underReplicated, offline);

            if (underReplicated > 0 || offline > 0) {
                // In production: trigger alert (PagerDuty, Slack, etc.)
                triggerAlert(underReplicated, offline);
            }

        } catch (Exception e) {
            System.err.println("Health check failed: " + e.getMessage());
        }
    }

    private void triggerAlert(int underReplicated, int offline) {
        System.err.printf(
            "🚨 ALERT: %d under-replicated, %d offline partitions!%n",
            underReplicated, offline);
    }
}
```

---

### Trigger Preferred Leader Election

```java
package com.example.kafka.admin;

import org.apache.kafka.clients.admin.*;
import org.apache.kafka.common.ElectionType;
import org.apache.kafka.common.TopicPartition;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.stereotype.Service;

import java.util.Set;

@Service
public class LeaderElectionService {

    private final KafkaAdmin kafkaAdmin;

    public LeaderElectionService(KafkaAdmin kafkaAdmin) {
        this.kafkaAdmin = kafkaAdmin;
    }

    // After broker recovery, re-elect preferred leaders
    // to restore even leader distribution across brokers
    public void electPreferredLeaders(String topic, int partition)
            throws Exception {

        try (AdminClient client =
                AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            Set<TopicPartition> partitions =
                Set.of(new TopicPartition(topic, partition));

            // PREFERRED: elect the "preferred replica" (first in replica list)
            // This restores the original leader distribution
            client.electLeaders(ElectionType.PREFERRED, partitions)
                  .all().get();

            System.out.printf(
                "Preferred leader election triggered for %s-%d%n",
                topic, partition);
        }
    }

    // Elect preferred leaders for ALL partitions (after cluster rebalance)
    public void electAllPreferredLeaders() throws Exception {
        try (AdminClient client =
                AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            // null = all partitions
            client.electLeaders(ElectionType.PREFERRED, null)
                  .all().get();

            System.out.println("Preferred leader election triggered for all partitions");
        }
    }
}
```

---

### Producer with Retry and Durability Handling

```java
package com.example.kafka.producer;

import com.example.kafka.model.OrderEvent;
import org.apache.kafka.common.errors.NotEnoughReplicasException;
import org.apache.kafka.common.errors.TimeoutException;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class DurableOrderProducer {

    private static final String TOPIC = "payments";
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public DurableOrderProducer(KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void send(OrderEvent event) {
        kafkaTemplate.send(TOPIC, event.getOrderId(), event)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    System.out.printf("✅ Written to partition=%d offset=%d%n",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                    return;
                }

                // Specific error handling per exception type
                if (ex.getCause() instanceof NotEnoughReplicasException) {
                    // ISR < min.insync.replicas — cluster health issue
                    System.err.println(
                        "❌ NOT ENOUGH REPLICAS — cluster degraded! " +
                        "Check broker health. Message NOT sent.");
                    // Alert ops team — do NOT silently retry without checking cluster
                    alertOps("NotEnoughReplicas for topic: " + TOPIC);

                } else if (ex.getCause() instanceof TimeoutException) {
                    // Broker unresponsive — will be auto-retried by Kafka client
                    System.err.println(
                        "⚠️  Timeout — Kafka client will retry automatically");

                } else {
                    System.err.println("❌ Unexpected error: " + ex.getMessage());
                }
            });
    }

    private void alertOps(String message) {
        // PagerDuty / Slack / monitoring integration
        System.err.println("🚨 OPS ALERT: " + message);
    }
}
```

---

## ⚡ 5. Key Replication Configurations

| Config | Default | Recommended | Scope |
|--------|---------|-------------|-------|
| `replication.factor` | `1` | `3` | Topic level |
| `min.insync.replicas` | `1` | `2` | Topic/Broker level |
| `unclean.leader.election.enable` | `false` | `false` | Topic/Broker level |
| `replica.lag.time.max.ms` | `30000` | `30000` | Broker level |
| `replica.fetch.max.bytes` | `1048576` | Increase for large msgs | Broker level |
| `replica.socket.timeout.ms` | `30000` | `30000` | Broker level |
| `num.replica.fetchers` | `1` | `4–8` for high throughput | Broker level |
| `auto.leader.rebalance.enable` | `true` | `true` | Broker level |
| `leader.imbalance.check.interval.seconds` | `300` | `300` | Broker level |

---

### The Durability Safety Net (Memorize This)

```
Producer:    acks = all
Topic:       replication.factor = 3
Topic:       min.insync.replicas = 2
Topic:       unclean.leader.election.enable = false

What this guarantees:
  ✅ Message written to at least 2 brokers before ACK
  ✅ Even if 1 broker dies immediately after → message safe on 2nd broker
  ✅ New leader always from ISR → no data rollback
  ✅ If only 1 broker alive → writes fail (safe) rather than risking loss

What this does NOT guarantee:
  ❌ Simultaneous loss of 2 brokers → data loss (exceeded RF-1 tolerance)
  ❌ Protection from misconfigured consumers overwriting offsets
```

---

## 🔄 6. Real-World Scenarios

### Scenario: Planned Broker Maintenance (Rolling Restart)

```
Goal: Restart all 3 brokers without downtime or data loss

Step 1: Verify cluster health
  All partitions fully replicated (no under-replicated partitions)
  kafka-topics.sh --describe → ISR count = replica count everywhere

Step 2: Restart Broker 3 (not the controller)
  Stop Broker 3 gracefully (SIGTERM)
  Kafka detects: B3 session expired
  Controller: reassign B3's partition leaders to B1/B2
  Wait: all partitions have new leaders (< 30 seconds typically)

Step 3: B3 restarts, fetches missing messages from new leaders
  Wait until: B3 ISR count matches replica count for its partitions
  Monitor: UnderReplicatedPartitions = 0

Step 4: Repeat for Broker 2, then Broker 1 (the controller)
  When controller restarts: new controller elected among remaining brokers

Rule: Never restart more than 1 broker at a time
      Always wait for full ISR recovery between restarts
```

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| `replication.factor=1` in production | Single broker failure = permanent data loss | Always RF ≥ 3 in production |
| `min.insync.replicas=1` with `acks=all` | acks=all satisfied with just the leader → defeats the purpose | Set `min.insync.replicas=2` |
| `unclean.leader.election=true` | Data loss on ISR failure | Set to false for all critical topics |
| Restarting multiple brokers simultaneously | Multiple partitions go offline | Rolling restart: one at a time |
| Not monitoring UnderReplicatedPartitions | Silent durability degradation | Alert on `UnderReplicatedPartitions > 0` |
| `num.replica.fetchers=1` on busy cluster | Follower can't keep up → falls out of ISR | Increase to 4–8 |
| Ignoring preferred leader imbalance | All partition leaders pile onto recovered broker | Enable `auto.leader.rebalance.enable=true` |

---

## 🎯 8. Interview Questions

**Q1. What is ISR and how does a follower get removed from it?**
> ISR (In-Sync Replicas) is the set of replicas fully caught up with the leader. A follower is removed from ISR if it hasn't sent a FetchRequest to the leader within `replica.lag.time.max.ms` (default 30s). Once the follower catches up completely (its LEO matches the leader's), it is re-added to ISR.

**Q2. What is the difference between replication factor and min.insync.replicas?**
> `replication.factor` is the number of copies of each partition (how many brokers store it). `min.insync.replicas` is the minimum number of ISR members that must acknowledge a write for it to succeed (when `acks=all`). RF=3 means 3 copies exist; min.insync.replicas=2 means 2 must confirm writes. If ISR shrinks below min.insync.replicas, writes are rejected with `NotEnoughReplicasException`.

**Q3. How does Kafka elect a new partition leader?**
> When a leader broker fails, the controller detects it (via ZooKeeper/KRaft heartbeat timeout). The controller identifies all partitions where the failed broker was the leader, picks the first available member from each partition's ISR as the new leader, and broadcasts the LeaderAndIsr update to all brokers. Clients auto-discover the new leader on their next metadata refresh.

**Q4. What is unclean leader election and when would you enable it?**
> Unclean leader election allows an out-of-sync replica (not in ISR) to become the leader when all ISR members are unavailable. It trades **durability for availability** — the partition comes back online but recent committed messages may be lost. Only enable for non-critical data (metrics, logs) where availability outweighs durability. Never for financial or business-critical data.

**Q5. What is a leader epoch and why does it matter?**
> A leader epoch is a monotonically increasing counter that increments each time a new leader is elected for a partition. When an old leader restarts and tries to rejoin, it compares its epoch with the controller's current epoch. If its epoch is lower, it truncates any divergent log entries and re-fetches from the current leader. This prevents split-brain scenarios where two brokers believe they're the leader.

**Q6. What does High Watermark (HW) have to do with replication?**
> HW is the offset up to which ALL ISR members have replicated. Consumers can only read messages at or below the HW. This ensures that if the leader crashes, the new leader (elected from ISR) will have all messages consumers have already read — preventing consumers from seeing data that later disappears.

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "Silent Data Loss in Production"
> **Problem:** Your Kafka cluster had a network partition 2 months ago. One broker was isolated, continued accepting writes (it thought it was still the leader), then reconnected. Some consumer downstream reported data inconsistencies. What happened and how do you prevent it?

**Answer:**
```
What happened (split-brain without leader epochs — old Kafka behavior):
  Broker 1 was network-isolated but still thought it was leader
  New leader Broker 2 was elected by controller (Broker 1 unreachable)
  Both Broker 1 and Broker 2 accepted writes simultaneously
  Same partition offsets → different messages on different brokers
  When B1 reconnected: divergent log, some messages overwritten

Modern Kafka prevention (leader epoch):
  When B1 reconnects: controller tells it "epoch is now 5, you were epoch 4"
  B1 truncates back to epoch 4/5 boundary
  B1 fetches from current leader (B2) from that point
  No more split-brain

Your cluster was likely running old Kafka without leader epoch support.

Prevention:
  1. Upgrade to Kafka 2.x+ (leader epoch supported)
  2. acks=all + min.insync.replicas=2
     → Isolated broker would reject writes (can't reach quorum)
  3. Enable network monitoring / broker connectivity checks
  4. Alert on under-replicated partitions > 0 (first sign of isolation)
```

---

### Scenario 2: "Replication Factor Tradeoff — Cost vs Durability"
> **Problem:** Your startup is cost-conscious. The team proposes RF=1 for all topics to halve storage costs. You're the senior engineer — what do you say?

**Answer:**
```
RF=1 impact:
  Any single broker failure = ALL partitions on that broker are OFFLINE
  Even brief maintenance window = producer errors, consumer lag
  If disk fails = permanent, unrecoverable data loss

Counterproposal: Tiered replication by business criticality

  Tier 1 - Business-critical (orders, payments, users):
    RF=3, min.insync.replicas=2
    Cost: 3x storage for ~20% of data
    
  Tier 2 - Operational (notifications, emails, analytics):
    RF=2, min.insync.replicas=1
    Cost: 2x storage for ~50% of data
    
  Tier 3 - Ephemeral (metrics, logs, debug events):
    RF=1, retention.ms=3600000 (1 hour)
    Cost: 1x storage for ~30% of data

Net result: ~30–40% storage savings vs blanket RF=3,
            while protecting all customer data with RF≥2.
            Business-critical data is fully protected.
```

---

## 📝 10. Quick Revision Summary

```
✅ Replication Factor = number of copies of each partition
   RF=3 → 1 leader + 2 followers
   Tolerates (RF-1) simultaneous failures

✅ Leader = handles ALL reads and writes (1 per partition at all times)
✅ Follower = replicates from leader via continuous FetchRequest loop
✅ ISR = followers fully caught up (within replica.lag.time.max.ms)

✅ Replication flow:
   Producer → Leader (write) → Followers fetch → HW advances → consumers read

✅ High Watermark = min(LEO of all ISR members)
   Consumers CANNOT read beyond HW
   Ensures data survives leader failover without consumer inconsistency

✅ Leader election on failure:
   Controller detects dead broker → picks first ISR member → broadcasts LeaderAndIsr
   Clients auto-discover new leader on metadata refresh

✅ Leader Epoch = monotonic counter per partition
   Prevents split-brain and log divergence on rejoin
   Old leader truncates divergent entries before rejoining as follower

✅ Unclean leader election = availability over durability
   false (default) = never elect stale replica → partition goes offline
   true = stale replica elected → DATA LOSS → use only for non-critical topics

✅ The production safety net:
   acks=all + RF=3 + min.insync.replicas=2 + unclean=false

✅ Monitor always:
   UnderReplicatedPartitions = 0 → cluster healthy
   UnderReplicatedPartitions > 0 → immediate alert!
```

---

**← Previous:** [10 — Partitioning Strategies](./10-partitioning.md)  
**Next Topic →** [12 — Log Storage, Segments, Retention & Compaction](./12-log-storage.md)
