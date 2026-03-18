# 13 — ZooKeeper vs KRaft: Kafka's Metadata Evolution
> **Phase 3 — Kafka Internals & Storage**  
> 📁 `notes/13-zookeeper-kraft.md`

---

## 📖 1. Concept Explanation

### The Metadata Problem

Every Kafka cluster needs a **metadata layer** — a system that answers:

- Which brokers are alive right now?
- Which broker is the leader for partition X?
- What are the ISR members for topic Y?
- Which consumer group is at what offset?
- Who is the current controller?

This metadata must be **consistent, highly available, and fast** — because every producer and consumer decision depends on it.

Kafka has had two answers to this problem over its lifetime:

```
2011–2022:  ZooKeeper  (external Apache coordination service)
2022+:      KRaft      (Kafka's own built-in Raft consensus)
```

---

### What is ZooKeeper?

Apache ZooKeeper is a **distributed coordination service** — essentially a hierarchical key-value store with strong consistency guarantees, designed for distributed systems to coordinate on shared state.

```
ZooKeeper data model: znodes (like a filesystem)

/
├── /brokers
│   ├── /ids
│   │   ├── /1  ← Broker 1 registered (ephemeral node)
│   │   ├── /2  ← Broker 2 registered
│   │   └── /3  ← Broker 3 registered
│   └── /topics
│       ├── /payments
│       │   └── /partitions
│       │       ├── /0  ← {"leader":1, "isr":[1,2,3]}
│       │       └── /1  ← {"leader":2, "isr":[2,3,1]}
│       └── /orders
├── /controller  ← Which broker is the controller {"brokerid":1}
├── /consumers   ← (legacy) consumer group offsets
└── /config      ← Topic and broker configurations
```

**Ephemeral znodes** are the key mechanism: when a broker connects to ZooKeeper, it creates an ephemeral node. When it disconnects (crash or network failure), the node is **automatically deleted**, alerting the controller that the broker is gone.

---

### ZooKeeper Architecture with Kafka

```
┌─────────────────────────────────────────────────────────────┐
│                    ZooKeeper Ensemble                        │
│                                                             │
│   ┌──────────┐      ┌──────────┐      ┌──────────┐          │
│   │  ZK-1    │◄────►│  ZK-2    │◄────►│  ZK-3    │          │
│   │ (Leader) │      │(Follower)│      │(Follower)│          │
│   └──────────┘      └──────────┘      └──────────┘          │
│        ZAB Protocol (ZooKeeper Atomic Broadcast)            │
└──────────────────────────┬──────────────────────────────────┘
                           │  All brokers connect here
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
     ┌──────────┐    ┌──────────┐    ┌──────────┐
     │ Broker 1 │    │ Broker 2 │    │ Broker 3 │
     │(Controller)    │          │    │          │
     └──────────┘    └──────────┘    └──────────┘
     
Broker 1 becomes Controller by:
  1. Racing to create ephemeral /controller znode
  2. First one wins → becomes Controller
  3. Others watch the /controller znode
  4. If Controller dies → znode deleted → others race again
```

---

### Problems with ZooKeeper Mode

```
Problem 1: Operational complexity
  Two separate systems to deploy, monitor, upgrade, secure, backup
  Different configuration, different logs, different failure modes
  ZooKeeper needs its own hardware/VMs in production
  Ops cost: ~2x what it should be

Problem 2: Scalability ceiling
  ZooKeeper stores full metadata in memory
  Each partition = ~100 bytes of metadata in ZK
  100,000 partitions = ~10MB in ZK → manageable
  1,000,000 partitions = ~100MB → ZK starts struggling
  Real limit: ~200K partitions per cluster before instability
  Large organizations were hitting this wall

Problem 3: Controller startup (metadata reload)
  On controller restart or failover:
    New controller must load ALL metadata from ZooKeeper
    200K partitions × multiple metadata reads = minutes of downtime
    During this time: no leader elections, no topic creation

Problem 4: Controller failover time
  ZK session timeout: 6–30 seconds before dead controller detected
  Metadata reload: additional 30–60 seconds
  Total failover: 30–90 seconds of degraded operation
  → Unacceptable for high-availability requirements

Problem 5: Split-brain risk
  If network partitions ZooKeeper from Kafka cluster:
    Brokers can't update ZK → stale metadata
    Controller may lose ZK connection → new controller elected
    Two controllers for a brief window → dangerous
```

---

## 🏗️ 2. KRaft — Kafka's Built-In Consensus

### What is KRaft?

**KRaft** (Kafka Raft) is Kafka's self-managed metadata system, introduced in **KIP-500**. It replaces ZooKeeper by embedding a **Raft consensus algorithm** directly into Kafka brokers.

Raft is a distributed consensus algorithm designed to be more understandable than Paxos. It elects a leader among a quorum of nodes and ensures all nodes agree on a consistent log of events.

```
KRaft key insight:
  Kafka already has an excellent distributed log (for user data).
  Why not use that same log to store metadata?
  → __cluster_metadata topic IS the metadata store
  → Raft consensus ensures it's replicated and consistent
```

---

### KRaft Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                    KAFKA CLUSTER (KRaft Mode)                      │
│                                                                    │
│   ┌──────────────────────────────────────────────────────────┐     │
│   │              Controller Quorum (Raft)                    │     │
│   │                                                          │     │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │     │
│   │  │ Controller 1│  │ Controller 2│  │ Controller 3│      │     │
│   │  │  (Active)   │  │  (Voter)    │  │  (Voter)    │      │     │
│   │  │             │  │             │  │             │      │     │
│   │  │ Handles all │  │ Replicates  │  │ Replicates  │      │     │
│   │  │ metadata    │  │ metadata    │  │ metadata    │      │     │
│   │  │ changes     │  │ log         │  │ log         │      │     │
│   │  └─────────────┘  └─────────────┘  └─────────────┘      │     │
│   │         │                                                │     │
│   │         ▼  __cluster_metadata (internal topic)           │     │
│   │  [BrokerRegistration][TopicRecord][PartitionRecord]...   │     │
│   └──────────────────────────────────────────────────────────┘     │
│                    │ MetadataFetch (push to brokers)                │
│         ┌──────────┼──────────┐                                    │
│         ▼          ▼          ▼                                    │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐                          │
│   │ Broker 4 │ │ Broker 5 │ │ Broker 6 │  ← Data Brokers          │
│   │ (data)   │ │ (data)   │ │ (data)   │                          │
│   └──────────┘ └──────────┘ └──────────┘                          │
│                                                                    │
│   ✅ No ZooKeeper. Self-contained.                                 │
└────────────────────────────────────────────────────────────────────┘
```

> **Note:** In smaller clusters (dev/staging), a node can play both roles: `process.roles=broker,controller`. In large production clusters, **dedicated controller nodes** are strongly recommended.

---

### `__cluster_metadata` — The Metadata Log

```
KRaft stores all cluster metadata as records in __cluster_metadata:

Record Type               What it stores
─────────────────────     ──────────────────────────────────────────
BrokerRegistrationRecord  Broker ID, host, port, rack, capabilities
TopicRecord               Topic name → topic UUID
PartitionRecord           Partition → replicas, ISR, leader epoch
PartitionChangeRecord     Leader/ISR changes after failover
ConfigRecord              Broker and topic configs
ClientQuotasRecord        Client quota settings
AccessControlRecord       ACL security rules
ProducerIdsRecord         Transactional producer ID ranges

All records are:
  - Append-only (like a regular Kafka topic)
  - Replicated across all controller quorum members
  - Replayed from offset 0 to rebuild full cluster state (fast!)
  - Snapshotted periodically (to avoid replaying entire history)
```

---

## ⚙️ 3. How KRaft Works Internally

### Raft Consensus — Simplified

```
Raft ensures that a quorum of nodes agrees on every metadata change:

1. Active Controller (Raft Leader) receives metadata change request
   e.g., "Broker 2 died → elect new leader for Partition 0"

2. Active Controller appends record to its local __cluster_metadata log
   Assigns this record a metadata offset

3. Active Controller replicates to Voter controllers
   "Here's metadata record at offset 1042. Please append and ACK."

4. Voter controllers append and respond
   "ACK at offset 1042"

5. Once majority (quorum) ACK:
   Active Controller commits the record
   Sends LeaderAndIsr RPC to affected data brokers
   Brokers update their local metadata cache

6. Quorum sizes:
   3 controllers → tolerate 1 failure (needs 2/3 to commit)
   5 controllers → tolerate 2 failures (needs 3/5 to commit)
```

### Controller Failover in KRaft

```
Normal: Controller 1 is active (Raft leader)
        Controllers 2, 3 are voters (followers)

Controller 1 dies:

Step 1: Voters detect leader missing
  No heartbeat from Controller 1 for election timeout (default ~1s in Raft)

Step 2: Election
  Controller 2 increments its term, votes for itself
  Sends RequestVote RPC to Controller 3
  Controller 3 votes YES (Controller 2 has the latest log)
  Controller 2 wins with majority (2/3)

Step 3: Controller 2 becomes active
  Sends LeaderAndIsr updates to data brokers
  Total time: typically < 1 second (vs 30–90 seconds with ZooKeeper)

Step 4: Controller 1 restarts
  Detects Controller 2 has higher term
  Rejoins as voter, fetches missing metadata records
```

### KRaft Metadata Snapshots

```
Problem without snapshots:
  __cluster_metadata grows forever as records accumulate
  On new broker join or controller restart: must replay entire log
  10 million metadata records × replay time = slow startup

Solution: Snapshots
  Periodically, the active controller writes a SNAPSHOT:
  A point-in-time complete view of all cluster metadata

  Snapshot file: /var/lib/kafka/__cluster_metadata/
    00000000000001000000-0000000001.checkpoint  ← snapshot at offset 1M

  New controller or broker startup:
    1. Load latest snapshot (instant) → skip replaying old records
    2. Replay only records since the snapshot (small delta)
    → Fast startup regardless of cluster history length

Controlled by:
  metadata.log.max.record.bytes.between.snapshots (default 20MB)
```

---

### Node Roles in KRaft

```
process.roles=broker
  → Pure data broker. Handles producer/consumer requests.
  → Receives metadata updates from controller quorum.
  → Cannot participate in controller elections.

process.roles=controller
  → Pure controller. Participates in Raft quorum.
  → Stores __cluster_metadata. Handles leader elections.
  → Does NOT handle producer/consumer data requests.
  → Recommended for large clusters (dedicated controller nodes).

process.roles=broker,controller
  → Combined mode. Acts as both data broker AND controller voter.
  → Fine for small clusters (< 10 brokers).
  → Not recommended for large clusters (resource contention).
```

---

## 💻 4. Code Examples (Java + Spring Boot)

### KRaft Single-Node Docker Setup

```yaml
# docker-compose-kraft.yml
version: '3.8'

services:
  kafka:
    image: apache/kafka:3.7.0
    container_name: kafka-kraft
    ports:
      - "9092:9092"
    environment:
      # KRaft Identity
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller        # Combined mode
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"         # Fixed cluster ID

      # Listeners
      KAFKA_LISTENERS: >-
        PLAINTEXT://0.0.0.0:9092,
        CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: >-
        PLAINTEXT:PLAINTEXT,
        CONTROLLER:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER

      # KRaft Quorum (single node = only itself)
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093

      # Topic defaults
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    volumes:
      - kraft-data:/var/lib/kafka/data

volumes:
  kraft-data:
```

---

### KRaft 3-Broker Production Cluster

```yaml
# docker-compose-kraft-cluster.yml
version: '3.8'

# Shared environment template
x-kafka-common: &kafka-common
  image: apache/kafka:3.7.0
  environment: &kafka-env
    CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    KAFKA_PROCESS_ROLES: broker,controller
    KAFKA_CONTROLLER_QUORUM_VOTERS: >-
      1@kafka-1:9093,
      2@kafka-2:9093,
      3@kafka-3:9093
    KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
    KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
    KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: >-
      PLAINTEXT:PLAINTEXT,
      CONTROLLER:PLAINTEXT
    KAFKA_DEFAULT_REPLICATION_FACTOR: 3
    KAFKA_NUM_PARTITIONS: 6
    KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
    KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
    KAFKA_MIN_INSYNC_REPLICAS: 2
  networks:
    - kafka-net

services:
  kafka-1:
    <<: *kafka-common
    container_name: kafka-1
    ports: ["9092:9092"]
    environment:
      <<: *kafka-env
      KAFKA_NODE_ID: 1
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
    volumes: [kafka-1-data:/var/lib/kafka/data]

  kafka-2:
    <<: *kafka-common
    container_name: kafka-2
    ports: ["9093:9092"]
    environment:
      <<: *kafka-env
      KAFKA_NODE_ID: 2
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9093
    volumes: [kafka-2-data:/var/lib/kafka/data]

  kafka-3:
    <<: *kafka-common
    container_name: kafka-3
    ports: ["9094:9092"]
    environment:
      <<: *kafka-env
      KAFKA_NODE_ID: 3
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9094
    volumes: [kafka-3-data:/var/lib/kafka/data]

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports: ["8080:8080"]
    environment:
      KAFKA_CLUSTERS_0_NAME: kraft-cluster
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-1:9092,kafka-2:9092,kafka-3:9092
    networks: [kafka-net]
    depends_on: [kafka-1, kafka-2, kafka-3]

volumes:
  kafka-1-data:
  kafka-2-data:
  kafka-3-data:

networks:
  kafka-net:
    driver: bridge
```

---

### Spring Boot Connection to KRaft Cluster

```yaml
# application.yml — same as before, KRaft is transparent to clients!
spring:
  kafka:
    # KRaft is completely transparent to Spring Boot / Kafka clients
    # No configuration change needed when switching ZK → KRaft
    bootstrap-servers: localhost:9092,localhost:9093,localhost:9094
    producer:
      acks: all
      properties:
        enable.idempotence: true
    consumer:
      group-id: my-service
      auto-offset-reset: earliest
```

> **Key point:** KRaft is **completely transparent to application code**. No changes to `KafkaTemplate`, `@KafkaListener`, or any Spring Kafka code. The only changes are in broker configuration and ops tooling.

---

### Inspect KRaft Metadata via CLI

```bash
# Generate a new cluster ID (do this ONCE per cluster)
kafka-storage.sh random-uuid
# Output: MkU3OEVBNTcwNTJENDM2Qk

# Format storage (run on EACH broker before first start)
kafka-storage.sh format \
  --cluster-id MkU3OEVBNTcwNTJENDM2Qk \
  --config /etc/kafka/kraft/server.properties

# Describe the current cluster metadata (KRaft)
kafka-metadata-quorum.sh \
  --bootstrap-server localhost:9092 \
  describe --status

# Output:
# ClusterId: MkU3OEVBNTcwNTJENDM2Qk
# LeaderId:  1
# LeaderEpoch: 5
# HighWatermark: 1042
# MaxFollowerLag: 0
# MaxFollowerLagTimeMs: 0
# CurrentVoters: [1,2,3]
# CurrentObservers: []

# Describe quorum replication status
kafka-metadata-quorum.sh \
  --bootstrap-server localhost:9092 \
  describe --replication

# List metadata log (advanced debugging)
kafka-dump-log.sh \
  --files /var/lib/kafka/__cluster_metadata/00000000000000000000.log \
  --print-data-log
```

---

### Migrate ZooKeeper Cluster to KRaft (Production Steps)

```bash
# ── Step 1: Verify Kafka version ≥ 3.3 ──────────────────────────
kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# ── Step 2: Generate cluster ID ──────────────────────────────────
CLUSTER_ID=$(kafka-storage.sh random-uuid)
echo "New Cluster ID: $CLUSTER_ID"

# ── Step 3: Update broker configs (server.properties) ────────────
# Add KRaft settings, keep ZK settings for migration mode
# process.roles=broker,controller
# controller.quorum.voters=1@b1:9093,2@b2:9093,3@b3:9093

# ── Step 4: Format storage on each controller node ───────────────
kafka-storage.sh format \
  --cluster-id $CLUSTER_ID \
  --config /etc/kafka/kraft/server.properties

# ── Step 5: Enable migration mode ────────────────────────────────
# In server.properties temporarily:
# zookeeper.metadata.migration.enable=true
# This runs ZK and KRaft in parallel during migration

# ── Step 6: Rolling restart all brokers ──────────────────────────
# One broker at a time, verify health after each

# ── Step 7: Finalize migration ───────────────────────────────────
# Once all metadata migrated to KRaft:
kafka-features.sh --bootstrap-server localhost:9092 \
  upgrade --feature metadata.version=<latest>

# ── Step 8: Remove ZooKeeper settings ────────────────────────────
# Remove zookeeper.connect from all server.properties
# Restart brokers
# Decommission ZooKeeper ensemble
```

---

## ⚡ 5. Key KRaft Configurations

| Config | Example | Explanation |
|--------|---------|-------------|
| `process.roles` | `broker,controller` | Roles this node plays |
| `node.id` | `1` | Unique node ID (replaces broker.id in KRaft) |
| `controller.quorum.voters` | `1@b1:9093,2@b2:9093,3@b3:9093` | All controller quorum members |
| `controller.listener.names` | `CONTROLLER` | Which listener is for controller comms |
| `metadata.log.dir` | `/var/lib/kafka/metadata` | Where `__cluster_metadata` is stored |
| `metadata.log.max.record.bytes.between.snapshots` | `20971520` (20MB) | Snapshot frequency |
| `controller.quorum.election.timeout.ms` | `1000` | Time before starting an election |
| `controller.quorum.fetch.timeout.ms` | `2000` | Time before follower considers leader dead |

---

## 🔄 6. ZooKeeper vs KRaft — Full Comparison

| Feature | ZooKeeper Mode | KRaft Mode |
|---------|---------------|------------|
| **External dependency** | ✅ ZooKeeper ensemble required | ❌ None — self-contained |
| **Max partitions** | ~200K (practical limit) | **Millions** |
| **Controller failover** | 30–90 seconds | **< 1 second** |
| **Startup time** (large cluster) | Minutes (metadata reload) | **Seconds** (snapshot + delta) |
| **Operational complexity** | High (2 systems) | **Low** (1 system) |
| **Split-brain risk** | Present (ZK + Kafka can disagree) | **Eliminated** (single consensus) |
| **Security surface** | ZK ACLs + Kafka ACLs (separate) | **Unified** Kafka ACLs |
| **Monitoring** | 2 sets of metrics/alerts | **1 system** |
| **Production ready** | Always | **Kafka 3.3+ (Oct 2022)** |
| **ZK removed** | N/A | **Kafka 4.0** |
| **Migration path** | — | `kafka-storage.sh format` + rolling restart |

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Different `CLUSTER_ID` across brokers | Brokers refuse to join cluster | Generate once, use same ID everywhere |
| Not formatting storage before first start | Broker fails to start with `StorageException` | Run `kafka-storage.sh format` on each node |
| `process.roles=broker,controller` in large cluster | Controller GC pauses affect metadata | Use dedicated controller nodes |
| Even number of controller nodes | Can't form quorum majority (e.g., 2 nodes = 1+1 split) | Always use odd numbers: 3 or 5 |
| Missing `CONTROLLER` in listener security protocol map | Controller-to-controller comms fail | Map all listener names explicitly |
| Forgetting to remove ZK config after migration | Both ZK and KRaft active → confusion | Clean removal of ZK settings post-migration |
| Not upgrading metadata.version after migration | New KRaft features unavailable | Run `kafka-features.sh upgrade` |

---

## 🎯 8. Interview Questions

**Q1. Why was ZooKeeper replaced in Kafka?**
> ZooKeeper required running and managing a separate distributed system, adding operational complexity. It had a practical partition limit of ~200K (metadata stored in memory). Controller failover took 30–90 seconds due to metadata reload. KRaft (KIP-500) solved all of this by embedding consensus directly in Kafka using the Raft protocol and storing metadata in an internal Kafka topic (`__cluster_metadata`).

**Q2. What is KRaft and how does it work?**
> KRaft is Kafka's built-in Raft consensus implementation for metadata management. A subset of brokers (the "controller quorum") use the Raft protocol to agree on metadata changes — broker registrations, leader elections, topic/partition assignments. All metadata is stored as records in `__cluster_metadata`, an internal Kafka topic replicated across quorum members.

**Q3. What is the `__cluster_metadata` topic?**
> It's an internal Kafka topic that serves as the metadata log in KRaft mode. It stores records for every metadata event: broker registrations, topic creation, partition leader changes, ISR updates, ACLs, and configs. It's replicated across all controller quorum members and periodically snapshotted for fast startup recovery.

**Q4. What is the difference between `process.roles=broker`, `controller`, and `broker,controller`?**
> `broker` — a data broker that handles producer/consumer requests. `controller` — participates in the Raft controller quorum, manages cluster metadata, does NOT serve client data requests. `broker,controller` — combined mode, acts as both. Combined mode is fine for small clusters but for large production clusters, dedicated controller nodes are recommended to avoid resource contention.

**Q5. How many controller nodes should a production KRaft cluster have?**
> Always an odd number: 3 for most production clusters (tolerates 1 failure — needs 2/3 majority). Use 5 for very large clusters or where 2-failure tolerance is required (needs 3/5 majority). Even numbers are problematic because split votes can prevent quorum formation.

**Q6. Is application code affected when migrating from ZooKeeper to KRaft?**
> No. KRaft is completely transparent to Kafka clients. `KafkaTemplate`, `@KafkaListener`, and all Spring Kafka code works identically. Only broker-side configuration changes — `process.roles`, `controller.quorum.voters`, and storage formatting. No client-side changes needed.

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "Controller Failover Is Taking 45 Seconds"
> **Problem:** Your ZooKeeper-mode Kafka cluster has 500 topics and 10,000 partitions. When the controller broker restarts, it takes 45 seconds before the cluster is fully operational again. Teams are complaining about the downtime. How do you fix this?

**Answer:**
```
Root cause (ZooKeeper mode):
  Controller restart triggers full metadata reload from ZooKeeper
  10,000 partitions × multiple ZK reads = slow sequential loading
  This is a fundamental limitation of ZooKeeper mode

Immediate fixes (staying on ZooKeeper):
  1. Dedicated controller broker:
     Assign controller responsibilities to a broker with no data partitions
     No user traffic → faster restarts, less GC pressure
  2. Tune ZK session timeout:
     zookeeper.session.timeout.ms=15000  (faster failure detection)
  3. Increase ZK ensemble performance:
     Use SSDs for ZK data directory
     Dedicated ZK machines (no co-location with Kafka)

Long-term fix: Migrate to KRaft
  KRaft controller failover: < 1 second (Raft leader election)
  No full metadata reload (snapshot + delta replay)
  10,000 partitions is trivial for KRaft

  Steps:
    Upgrade to Kafka 3.3+
    Generate cluster ID, format KRaft storage
    Enable migration mode, rolling restart
    Finalize migration, remove ZK dependency
    Decommission ZooKeeper ensemble

  Result: 45-second failover → sub-second failover
```

---

### Scenario 2: "Hitting the Partition Limit"
> **Problem:** Your platform has grown to 180,000 partitions across the cluster. ZooKeeper is showing high latency and occasional connection timeouts. What's happening and what do you do?

**Answer:**
```
What's happening:
  ZooKeeper stores all partition metadata in memory
  ~100 bytes per partition × 180,000 = ~18MB in ZK
  Near the practical 200K partition limit
  ZK garbage collection pauses affecting metadata operations
  Each metadata update (leader election, ISR change) takes longer

Symptoms:
  Producer: intermittent LEADER_NOT_AVAILABLE errors
  Consumer: metadata refresh latency spikes
  ZK: high JVM heap usage, GC pauses visible in logs

Short-term mitigations:
  Increase ZK JVM heap: -Xmx4g (from default 1g)
  Reduce partition count: consolidate low-traffic topics
  Reduce replication factor on non-critical topics

Real fix: Migrate to KRaft
  KRaft tested and confirmed working with 1M+ partitions
  Metadata stored on disk (not just in memory)
  No ZooKeeper memory constraints

  After KRaft migration:
    Can grow to millions of partitions
    Sub-millisecond metadata operations
    No more ZK memory tuning
```

---

## 📝 10. Quick Revision Summary

```
✅ ZooKeeper (legacy):
   External coordination service using ephemeral znodes
   Brokers register → Controller elected → Metadata stored in /brokers, /topics
   Problems: ops complexity, ~200K partition limit, slow failover (30–90s)

✅ KRaft (modern, Kafka 3.3+ production-ready):
   Built-in Raft consensus, no external dependency
   Metadata stored in __cluster_metadata (internal Kafka topic)
   Controller quorum (3 or 5 nodes) using Raft leader election
   Failover: < 1 second (vs 30–90s ZooKeeper)
   Scales to millions of partitions

✅ Node roles:
   broker            → data only
   controller        → metadata/quorum only (recommended for large clusters)
   broker,controller → combined (fine for small clusters)

✅ Always odd number of controllers:
   3 → tolerates 1 failure
   5 → tolerates 2 failures

✅ KRaft is transparent to application code:
   No Spring Boot / Kafka client changes needed
   Same bootstrap-servers, same @KafkaListener, same KafkaTemplate

✅ Key CLI:
   kafka-storage.sh random-uuid      → generate cluster ID
   kafka-storage.sh format           → initialize KRaft storage
   kafka-metadata-quorum.sh describe → inspect quorum status

✅ Migration path: ZK → KRaft
   Upgrade Kafka ≥ 3.3
   Format KRaft storage
   Enable migration mode
   Rolling restart
   Finalize + remove ZK
   Decommission ZooKeeper

✅ Greenfield project today? Always KRaft.
   ZooKeeper removed in Kafka 4.0.
```

---

**← Previous:** [12 — Log Storage, Segments, Retention & Compaction](./12-log-storage.md)  
**Next Topic →** [14 — Producer Configurations Deep Dive](./14-producer-configs.md)
