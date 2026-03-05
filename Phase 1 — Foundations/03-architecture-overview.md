# 03 — Kafka Architecture Overview: Brokers, Clusters, ZooKeeper & KRaft
> **Phase 1 — Foundations**  
> 📁 `notes/03-architecture-overview.md`

---

## 📖 1. Concept Explanation

### The Big Picture

Kafka's architecture is built around **four core components** that work together to deliver a fault-tolerant, highly available, scalable distributed system:

```
┌─────────────────────────────────────────────────────────────────┐
│                    KAFKA ECOSYSTEM                              │
│                                                                 │
│   Producers  →  [ Kafka Cluster (Brokers) ]  →  Consumers      │
│                         ↕                                       │
│               [ Metadata Layer ]                                │
│               (ZooKeeper  OR  KRaft)                            │
└─────────────────────────────────────────────────────────────────┘
```

The four components:
1. **Broker** — A single Kafka server. Stores data, serves reads/writes.
2. **Cluster** — A group of brokers working together.
3. **ZooKeeper** — The old metadata coordinator (being phased out).
4. **KRaft** — Kafka's new built-in metadata system (Kafka 3.3+ production-ready).

---

### 1.1 — Broker

A **Broker** is simply a **Kafka server process** running on a machine.

Each broker:
- Has a unique **Broker ID** (integer, e.g., 1, 2, 3)
- Stores **partition data** on local disk
- Handles **read and write requests** from clients
- Participates in **replication** — storing leader and/or follower partitions
- Registers itself with the cluster metadata layer on startup

```
Broker = Kafka server process
       + unique numeric ID
       + local disk storage for partitions
       + network listener (default port 9092)
```

**What a broker does NOT do:**
- It does not coordinate with other brokers for every write (that's the client's job)
- It does not push data to consumers (consumers pull)

---

### 1.2 — Cluster

A **Cluster** is a **group of brokers** that collectively store all topic partitions and serve clients.

Key concepts within a cluster:

**Controller Broker:**
- One broker in the cluster is elected as the **Controller**
- The Controller manages partition leader elections, broker join/leave events
- If the controller dies, a new one is elected automatically
- In ZooKeeper mode: first broker to claim `/controller` znode becomes controller
- In KRaft mode: elected via the Raft consensus protocol

**Partition Leadership:**
- Each partition has exactly one **Leader broker** at any time
- All reads and writes for a partition go through its Leader
- Other brokers hold **Follower (Replica)** copies
- If a leader dies → a follower is promoted to leader (automatic failover)

```
Topic "payments" — 3 partitions, replication-factor=3

Partition 0:  Leader=Broker1,  Replicas=[Broker1, Broker2, Broker3]
Partition 1:  Leader=Broker2,  Replicas=[Broker2, Broker3, Broker1]
Partition 2:  Leader=Broker3,  Replicas=[Broker3, Broker1, Broker2]
```

Notice the **staggered leader distribution** — Kafka spreads partition leaders evenly across brokers so no single broker becomes a hot spot.

---

### 1.3 — ISR (In-Sync Replicas)

**ISR** is one of Kafka's most important durability concepts.

The **In-Sync Replica set** is the group of replicas that are **fully caught up** with the leader — they have all messages the leader has, within a configurable lag threshold.

```
Leader: Broker 1 (Partition 0)
ISR: [Broker1, Broker2, Broker3]  ← all caught up

If Broker3 falls behind (network issue):
ISR: [Broker1, Broker2]           ← Broker3 removed from ISR

If Broker3 catches up again:
ISR: [Broker1, Broker2, Broker3]  ← Broker3 re-added
```

**Why ISR matters:**
- A message is only considered **"committed"** when ALL brokers in the ISR have it
- `acks=all` means: leader waits for all ISR members to acknowledge the write
- If the leader dies, only an ISR member can be elected as the new leader — ensuring **no data loss**

---

### 1.4 — ZooKeeper (Legacy Metadata Layer)

**Apache ZooKeeper** is a distributed coordination service that Kafka historically relied on for:

| Responsibility | Details |
|----------------|---------|
| **Broker Registry** | Tracks which brokers are alive (ephemeral znodes) |
| **Controller Election** | First broker to grab `/controller` znode becomes Controller |
| **Topic Metadata** | Stores partition assignments, replica lists |
| **ACL Storage** | Security access control lists |
| **Consumer Group Offsets** | (Old clients only — modern Kafka uses `__consumer_offsets`) |

**ZooKeeper Architecture with Kafka:**

```
┌─────────────────────────────────────────────────────┐
│                 ZooKeeper Ensemble                   │
│                                                     │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│   │   ZK-1   │  │   ZK-2   │  │   ZK-3   │          │
│   │ (Leader) │  │(Follower)│  │(Follower)│          │
│   └──────────┘  └──────────┘  └──────────┘          │
└──────────────────────┬──────────────────────────────┘
                       │ (coordinates)
         ┌─────────────┼─────────────┐
         ↓             ↓             ↓
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │Broker 1 │  │Broker 2 │  │Broker 3 │
    │(Kafka)  │  │(Kafka)  │  │(Kafka)  │
    └─────────┘  └─────────┘  └─────────┘
```

**Problems with ZooKeeper:**
- Requires running and managing a **separate ZK cluster** (additional ops burden)
- ZooKeeper becomes a **scalability bottleneck** with many partitions (>200K partitions was unstable)
- Controller metadata was loaded into memory on every restart → **slow recovery** on large clusters
- Two separate systems to monitor, secure, and upgrade
- Split-brain risks if ZK and Kafka clusters disagree on metadata

---

### 1.5 — KRaft (Kafka Raft — The Future)

**KRaft** (introduced in **KIP-500**) replaces ZooKeeper by embedding metadata management directly into Kafka itself using the **Raft consensus algorithm**.

**KRaft Architecture:**

```
┌────────────────────────────────────────────────────────────────┐
│                      KAFKA CLUSTER (KRaft)                     │
│                                                                │
│   ┌──────────────────────────────────────────────────────┐     │
│   │               KRaft Controller Quorum                │     │
│   │                                                      │     │
│   │  ┌──────────┐   ┌──────────┐   ┌──────────┐          │     │
│   │  │Controller│   │Controller│   │Controller│          │     │
│   │  │(Active)  │   │(Voter)   │   │(Voter)   │          │     │
│   │  └──────────┘   └──────────┘   └──────────┘          │     │
│   │       ↑ Raft consensus — 3 or 5 nodes                │     │
│   └──────────────────────────────────────────────────────┘     │
│                         ↕ (metadata replication)               │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐                   │
│   │ Broker 1 │   │ Broker 2 │   │ Broker 3 │  ← Data brokers   │
│   └──────────┘   └──────────┘   └──────────┘                   │
└────────────────────────────────────────────────────────────────┘
```

> **Note:** In smaller clusters, a broker can serve **both** roles (controller + data broker). In large production clusters, dedicated controller nodes are recommended.

**KRaft stores metadata in a special internal topic: `__cluster_metadata`**

```
__cluster_metadata  (internal Kafka topic)
  - Broker registrations
  - Topic/partition assignments
  - Leader info
  - ISR updates
  - ACLs
```

**Benefits of KRaft over ZooKeeper:**

| Feature | ZooKeeper Mode | KRaft Mode |
|---------|---------------|------------|
| Separate system to run | ✅ Yes (ZK cluster) | ❌ No |
| Partition scalability | ~200K partitions | **Millions of partitions** |
| Controller failover | 30–60 seconds | **Under 30 seconds** |
| Startup recovery | Slow (metadata reload) | Fast (metadata in Kafka log) |
| Operational complexity | High | **Low** |
| Production-ready since | Always | **Kafka 3.3 (2022)** |
| ZooKeeper removal | — | **Kafka 4.0 (ZK fully removed)** |

---

## 🏗️ 2. Architecture Flow — Complete Picture

### ZooKeeper Mode (Legacy)

```
CLIENT (Producer/Consumer)
        │
        │ 1. Connect to any broker, ask for metadata
        ↓
┌───────────────────────────────────────────────────────────┐
│                     KAFKA CLUSTER                         │
│                                                           │
│  ┌──────────────────────────────────────────────────┐     │
│  │  Broker 1 (Controller)                           │     │
│  │   - Elected via ZooKeeper /controller znode      │     │
│  │   - Manages partition leadership                 │     │
│  │   - Topic: payments, Partition 0 LEADER          │     │
│  └──────────────────────────────────────────────────┘     │
│                                                           │
│  ┌──────────────────────────────────────────────────┐     │
│  │  Broker 2                                        │     │
│  │   - Topic: payments, Partition 1 LEADER          │     │
│  │   - Topic: payments, Partition 0 FOLLOWER        │     │
│  └──────────────────────────────────────────────────┘     │
│                                                           │
│  ┌──────────────────────────────────────────────────┐     │
│  │  Broker 3                                        │     │
│  │   - Topic: payments, Partition 2 LEADER          │     │
│  │   - Topic: payments, Partition 1 FOLLOWER        │     │
│  └──────────────────────────────────────────────────┘     │
└─────────────────────────────┬─────────────────────────────┘
                              │ (all brokers connect to ZK)
                              ↓
              ┌───────────────────────────┐
              │    ZooKeeper Ensemble     │
              │  (3 or 5 nodes)           │
              │  Stores: broker list,     │
              │  topic configs, ACLs,     │
              │  controller election      │
              └───────────────────────────┘
```

### KRaft Mode (Modern)

```
CLIENT (Producer/Consumer)
        │
        │ 1. Connect to any broker, ask for metadata
        ↓
┌───────────────────────────────────────────────────────────┐
│                  KAFKA CLUSTER (KRaft)                    │
│                                                           │
│  ┌──────────────────────────────────────────────────┐     │
│  │  Controller Quorum (3 nodes with Raft)           │     │
│  │   Active Controller: Broker 1                    │     │
│  │   Voters: Broker 2, Broker 3                     │     │
│  │   Metadata log: __cluster_metadata               │     │
│  └──────────────────────────────────────────────────┘     │
│            ↕ metadata pushed to all brokers               │
│  ┌────────────┐   ┌────────────┐   ┌────────────┐         │
│  │  Broker 1  │   │  Broker 2  │   │  Broker 3  │         │
│  │ (also ctrl)│   │            │   │            │         │
│  └────────────┘   └────────────┘   └────────────┘         │
│                                                           │
│           NO ZOOKEEPER — Self-contained ✅                │
└───────────────────────────────────────────────────────────┘
```

---

## ⚙️ 3. How It Works Internally

### Producer Write Path (Step by Step)

```
Producer wants to write to topic "payments", key="user-123"

Step 1: Bootstrap
  Producer connects to bootstrap-servers (any broker)
  Fetches cluster metadata: "Which broker is leader for each partition?"

Step 2: Partition Selection
  hash("user-123") % 3 = Partition 1
  Metadata says: Partition 1 leader = Broker 2

Step 3: Send to Leader
  Producer sends message directly to Broker 2

Step 4: Leader Appends
  Broker 2 appends message to Partition 1's log
  Assigns offset = next available integer

Step 5: Replication
  Broker 2 (leader) pushes to Broker 3 (follower)
  Broker 3 acknowledges when it has written to disk

Step 6: Acknowledgement (acks=all)
  Broker 2 waits until ALL ISR members ACK
  Then responds to producer: "offset=1042, success"

Step 7: Consumer Reads
  Consumer polls Broker 2 (leader of Partition 1)
  Kafka returns messages starting from consumer's last committed offset
```

### Controller Responsibilities (KRaft)

```
Active Controller handles:
  ├── Broker join events        → assign partitions to new broker
  ├── Broker leave/crash        → trigger leader election for affected partitions
  ├── Topic creation/deletion   → assign partitions across brokers
  ├── Partition reassignment    → when manually rebalancing cluster
  └── ISR change events         → update ISR list when replicas fall behind/catch up

Controller stores everything in: __cluster_metadata topic
All brokers get metadata updates via MetadataFetch RPC
```

### Broker Failure & Leader Election

```
Normal State:
  Partition 0: Leader=Broker1, ISR=[Broker1, Broker2, Broker3]

Broker1 crashes:
  ZooKeeper/KRaft detects: Broker1 session expired
  Controller triggered: need new leader for Partition 0
  Controller picks: first ISR member that's still alive → Broker2
  Controller updates metadata: Partition 0 Leader=Broker2
  All brokers get updated metadata
  Clients auto-discover new leader on next metadata refresh

Recovery:
  Broker1 restarts
  Fetches missing messages from new leader (Broker2)
  Once caught up → re-added to ISR
  Partition 0: Leader=Broker2, ISR=[Broker2, Broker3, Broker1]
  (Leadership stays with Broker2 until manual preferred-replica-election)
```

---

## 💻 4. Code Examples (Java + Spring Boot)

### `application.yml` — Broker Connection Config

```yaml
spring:
  kafka:
    # Multiple brokers for fault tolerance
    # Client will discover full cluster from any one of these
    bootstrap-servers: broker1:9092,broker2:9092,broker3:9092

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      acks: all                  # Wait for all ISR members to acknowledge
      retries: 3
      properties:
        enable.idempotence: true # Exactly-once producer guarantee

    consumer:
      group-id: payment-processor
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```

---

### Fetch Cluster Metadata Programmatically

```java
package com.example.kafka.admin;

import org.apache.kafka.clients.admin.*;
import org.apache.kafka.common.Node;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.stereotype.Service;

import java.util.Map;
import java.util.concurrent.ExecutionException;

@Service
public class ClusterInspector {

    private final KafkaAdmin kafkaAdmin;

    public ClusterInspector(KafkaAdmin kafkaAdmin) {
        this.kafkaAdmin = kafkaAdmin;
    }

    public void printClusterInfo() throws ExecutionException, InterruptedException {
        try (AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            // Describe the cluster
            DescribeClusterResult cluster = adminClient.describeCluster();

            System.out.println("Cluster ID  : " + cluster.clusterId().get());
            System.out.println("Controller  : " + cluster.controller().get());

            System.out.println("\nBrokers in cluster:");
            for (Node node : cluster.nodes().get()) {
                System.out.printf("  Broker ID=%d  Host=%s  Port=%d  Rack=%s%n",
                    node.id(), node.host(), node.port(), node.rack());
            }
        }
    }

    public void printTopicDetails(String topicName)
            throws ExecutionException, InterruptedException {

        try (AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            Map<String, TopicDescription> topics =
                adminClient.describeTopics(java.util.List.of(topicName)).allTopicNames().get();

            topics.forEach((name, desc) -> {
                System.out.println("\nTopic: " + name);
                desc.partitions().forEach(partition -> {
                    System.out.printf(
                        "  Partition %d → Leader: Broker%d | Replicas: %s | ISR: %s%n",
                        partition.partition(),
                        partition.leader().id(),
                        partition.replicas(),
                        partition.isr()
                    );
                });
            });
        }
    }
}
```

**Sample output:**
```
Cluster ID  : abc123-xyz
Controller  : Node(id=1, host=broker1, port=9092)

Brokers in cluster:
  Broker ID=1  Host=broker1  Port=9092  Rack=us-east-1a
  Broker ID=2  Host=broker2  Port=9092  Rack=us-east-1b
  Broker ID=3  Host=broker3  Port=9092  Rack=us-east-1c

Topic: payments
  Partition 0 → Leader: Broker1 | Replicas: [Broker1, Broker2, Broker3] | ISR: [Broker1, Broker2, Broker3]
  Partition 1 → Leader: Broker2 | Replicas: [Broker2, Broker3, Broker1] | ISR: [Broker2, Broker3, Broker1]
  Partition 2 → Leader: Broker3 | Replicas: [Broker3, Broker1, Broker2] | ISR: [Broker3, Broker1, Broker2]
```

---

### Topic Creation with Rack-Aware Replication

```java
package com.example.kafka.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic paymentsTopic() {
        return TopicBuilder.name("payments")
                .partitions(6)
                .replicas(3)    // Kafka will distribute replicas across racks
                                // if broker.rack is configured on each broker
                .build();
    }
}
```

---

### Listening to Partition Metadata Events

```java
package com.example.kafka.listener;

import org.apache.kafka.common.TopicPartition;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.listener.ConsumerSeekAware;
import org.springframework.stereotype.Component;

import java.util.Collection;
import java.util.Map;

@Component
public class PartitionAwareConsumer implements ConsumerSeekAware {

    // Called when partitions are assigned to this consumer instance
    @Override
    public void onPartitionsAssigned(
            Map<TopicPartition, Long> assignments,
            ConsumerSeekCallback callback) {

        System.out.println("Partitions assigned to this instance:");
        assignments.forEach((tp, offset) ->
            System.out.printf("  Topic=%s Partition=%d CurrentOffset=%d%n",
                tp.topic(), tp.partition(), offset)
        );
    }

    // Called when partitions are revoked (before rebalance)
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        System.out.println("Partitions being revoked (rebalance triggered):");
        partitions.forEach(tp ->
            System.out.printf("  Revoking: %s-%d%n", tp.topic(), tp.partition())
        );
    }

    @KafkaListener(topics = "payments", groupId = "my-group")
    public void consume(org.apache.kafka.clients.consumer.ConsumerRecord<String, String> record) {
        System.out.printf("B%d P%d O%d: %s%n",
            // Note: actual broker ID not in ConsumerRecord; partition & offset are
            0, record.partition(), record.offset(), record.value());
    }
}
```

---

## ⚡ 5. Key Configurations

### Broker-Level Configs (`server.properties` / environment variables)

| Config | Example | Explanation |
|--------|---------|-------------|
| `broker.id` | `1` | Unique broker integer ID |
| `listeners` | `PLAINTEXT://0.0.0.0:9092` | Address the broker listens on |
| `advertised.listeners` | `PLAINTEXT://broker1:9092` | Address clients use to connect |
| `log.dirs` | `/var/kafka/logs` | Where partition data is stored on disk |
| `num.partitions` | `6` | Default partition count for new topics |
| `default.replication.factor` | `3` | Default replica count for new topics |
| `min.insync.replicas` | `2` | Min ISR size for a write to succeed (with acks=all) |
| `unclean.leader.election.enable` | `false` | Allow out-of-sync replica to become leader (risks data loss) |
| `broker.rack` | `us-east-1a` | Rack label for rack-aware replica placement |
| `process.roles` | `broker,controller` | KRaft only: roles this node plays |
| `controller.quorum.voters` | `1@b1:9093,2@b2:9093,3@b3:9093` | KRaft only: controller quorum members |

---

## 🔄 6. Real-World Scenarios

### Scenario: Rack-Aware Cluster Design

In production, brokers are typically spread across **availability zones** (AZs):

```
AZ us-east-1a:  Broker 1  (rack=a)
AZ us-east-1b:  Broker 2  (rack=b)
AZ us-east-1c:  Broker 3  (rack=c)

With rack-aware replication (broker.rack configured):
  Partition 0 replicas → [Broker1(rack=a), Broker2(rack=b), Broker3(rack=c)]

Now even if an entire AZ goes down, the partition still has 2 replicas.
Without rack awareness, all 3 replicas could end up in the same AZ.
```

### Scenario: KRaft Controller Quorum Sizing

```
Always use ODD numbers for controller quorum (Raft needs majority):

  3 controllers → tolerates 1 failure  (2/3 majority needed)
  5 controllers → tolerates 2 failures (3/5 majority needed)

For most production clusters: 3 dedicated controllers is standard.
For very large clusters (1000+ brokers): 5 controllers recommended.
```

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Single broker in production | Any broker failure = cluster down | Always run minimum 3 brokers |
| `replication.factor=1` | Partition data lost if that broker dies | Use RF=3 in production |
| `unclean.leader.election=true` | Out-of-sync replica becomes leader → **data loss** | Keep it `false` in production |
| Not configuring `broker.rack` | All replicas may land in same AZ | Set rack labels, enable rack awareness |
| Forgetting `min.insync.replicas` | Even with RF=3, writes succeed with only 1 ISR replica | Set `min.insync.replicas=2` |
| Large ZooKeeper ensemble | More ZK nodes = more consensus overhead | Use 3 or 5 ZK nodes only |
| Running ZK on same hosts as Kafka | Resource contention | Use dedicated machines/containers |
| Not monitoring ISR shrinkage | Silent data durability degradation | Alert on `UnderReplicatedPartitions > 0` |

---

## 🎯 8. Interview Questions

**Q1. What is a Kafka broker?**
> A broker is a single Kafka server process that stores partition data on disk, handles read/write requests from clients, and participates in replication. Each broker has a unique integer ID and joins the cluster on startup by registering with the metadata layer (ZooKeeper or KRaft).

**Q2. What is the role of the Controller in a Kafka cluster?**
> The Controller is one broker elected to manage cluster-level administrative tasks: partition leader elections when brokers fail, handling broker join/leave events, and propagating metadata changes to all other brokers. In ZooKeeper mode it's elected via a ZK znode; in KRaft mode via Raft consensus.

**Q3. What is ISR and why is it important?**
> ISR (In-Sync Replicas) is the set of partition replicas fully caught up with the leader. With `acks=all`, a write is only acknowledged after all ISR members confirm it. A new leader can only be elected from the ISR, ensuring **no committed data is ever lost** during failover.

**Q4. What is the difference between ZooKeeper mode and KRaft mode?**
> ZooKeeper mode uses an external Apache ZooKeeper cluster to store Kafka metadata and coordinate controller election. KRaft mode (Kafka 3.3+) eliminates ZooKeeper entirely — Kafka uses its own internal Raft consensus protocol, storing metadata in `__cluster_metadata`. KRaft offers simpler operations, faster failover, and scales to millions of partitions.

**Q5. What does `unclean.leader.election.enable=true` do and why is it dangerous?**
> It allows a replica that is NOT in the ISR to become the partition leader. This can restore availability when all ISR members are down, but at the cost of **data loss** — the out-of-sync replica may be missing recent committed messages. In financial or critical systems, keep this `false`.

**Q6. What is `min.insync.replicas`?**
> It defines the minimum number of ISR replicas that must acknowledge a write when `acks=all`. If ISR shrinks below this threshold, the broker rejects writes with a `NotEnoughReplicasException`. This prevents data loss by refusing writes that can't be durably committed. Common setting: RF=3, `min.insync.replicas=2`.

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "One Broker Down — What Happens?"
> **Problem:** You have a 3-broker cluster, RF=3, `min.insync.replicas=2`, `acks=all`. Broker 2 goes down. What happens to reads and writes?

**Answer:**
```
Before failure:
  Partition 0: Leader=Broker1, ISR=[B1, B2, B3]
  Partition 1: Leader=Broker2, ISR=[B1, B2, B3]

Broker 2 dies:
  1. Controller detects Broker2 is gone (via ZK/KRaft heartbeat timeout)
  2. Controller elects new leader for Partition 1 → picks Broker1 or Broker3
     (both are in ISR, so no data loss)
  3. ISR updated: [B1, B3] for all partitions that had B2
  4. min.insync.replicas=2: ISR=[B1,B3] has 2 members → writes still accepted ✅
  5. Reads and writes continue on all partitions without interruption

If Broker3 also dies:
  ISR = [B1] only → 1 < min.insync.replicas=2
  → Writes REJECTED with NotEnoughReplicasException
  → Reads still work (from leader)
  This is the intended safety trade-off: consistency over availability
```

---

### Scenario 2: "Design a 3-Region Active-Active Kafka Cluster"
> **Problem:** You need Kafka to survive an entire region failure with zero data loss. How?

**Answer:**
```
Option A — MirrorMaker 2 (Active-Active Replication):
  Region 1: Kafka Cluster A  ←→  MirrorMaker 2  ←→  Region 2: Kafka Cluster B
  - Each region is a full independent cluster
  - MirrorMaker replicates topics bidirectionally
  - On region failure, redirect clients to surviving region
  - Challenge: offset translation, deduplication

Option B — Confluent's Multi-Region Clusters:
  - Single logical cluster spanning regions
  - Synchronous replication for critical partitions
  - Async replication for others (performance tradeoff)

For zero data loss:
  - Use synchronous replication (acks=all, min.insync.replicas spans regions)
  - This adds cross-region latency to every write
  - Trade-off: durability vs. write latency
```

---

### Scenario 3: "ZooKeeper vs KRaft — Which for a Greenfield Project?"
> **Problem:** Starting a new Kafka deployment today. ZooKeeper or KRaft?

**Answer: KRaft, unequivocally.**
```
Reasons:
  ✅ ZooKeeper is deprecated — removed in Kafka 4.0
  ✅ KRaft is production-ready since Kafka 3.3 (2022)
  ✅ Simpler ops — one system instead of two
  ✅ Faster controller failover (<30 sec vs 30-60 sec)
  ✅ Scales to millions of partitions
  ✅ Faster cluster startup (no full metadata reload)
  ✅ Better security model (no separate ZK ACLs)

Only use ZooKeeper if:
  - Running Kafka < 3.3 for compatibility reasons
  - Migrating a legacy system (use kafka-storage tool to migrate to KRaft)
```

---

## 📝 10. Quick Revision Summary

```
✅ BROKER      = Single Kafka server. Has ID, stores partitions, serves clients.
✅ CLUSTER     = Group of brokers. One is elected Controller.
✅ CONTROLLER  = Manages leader elections, broker join/leave, metadata updates.
✅ LEADER      = One broker per partition that handles all reads & writes.
✅ FOLLOWER    = Replica broker that replicates from leader.
✅ ISR         = In-Sync Replicas — replicas fully caught up with leader.
               Message is "committed" only when all ISR members have it.
               New leader ONLY elected from ISR → no data loss guarantee.

✅ ZOOKEEPER   = External metadata coordinator (LEGACY, deprecated in Kafka 4.0)
               Handles: broker registry, controller election, topic metadata
               Problems: separate system, 200K partition limit, slow recovery

✅ KRAFT       = Built-in Raft consensus (Kafka 3.3+ production-ready)
               Stores metadata in __cluster_metadata internal topic
               Benefits: no ZK, millions of partitions, fast failover

✅ Key Safety Config Trio (memorize this):
   replication.factor    = 3      (3 copies of every partition)
   min.insync.replicas   = 2      (need 2 ACKs for a write to succeed)
   acks                  = all    (producer waits for ISR acknowledgement)
   unclean.leader.election = false (never elect a stale replica as leader)
```

---

**← Previous:** [02 — Core Concepts: Topics, Partitions, Offsets & Messages](./02-core-concepts.md)  
**Next Topic →** [04 — Setting Up Kafka Locally: Docker & Native Install](./04-local-setup.md)
