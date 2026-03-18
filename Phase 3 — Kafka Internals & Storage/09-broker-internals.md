# 09 — Kafka Broker Internals: Request Handling & Log Architecture
> **Phase 3 — Kafka Internals & Storage**  
> 📁 `notes/09-broker-internals.md`

---

## 📖 1. Concept Explanation

### Why Broker Internals Matter at Senior Level

Understanding how the broker works internally is what separates a Kafka user from a Kafka engineer. It explains:
- Why Kafka is so fast (sequential I/O, page cache, zero-copy)
- Why certain configs have the impact they do
- How to debug performance issues at the broker level
- How to design topics and partitions for optimal throughput

---

### What Happens Inside a Broker?

A Kafka broker is a **JVM process** that runs several subsystems concurrently:

```
┌────────────────────────────────────────────────────────────────┐
│                        KAFKA BROKER                            │
│                                                                │
│  ┌──────────────────┐    ┌────────────────────────────────┐    │
│  │  Network Layer   │    │      Request Handler Pool      │    │
│  │  (Acceptor +     │───▶│  (RequestHandlerThread pool)   │    │
│  │   Processor      │    │  Processes decoded requests    │    │
│  │   threads)       │    └───────────────┬────────────────┘    │
│  └──────────────────┘                    │                     │
│                                          ▼                     │
│  ┌───────────────────────────────────────────────────────┐     │
│  │                  Log Manager                          │     │
│  │  Manages all partition logs on disk                   │     │
│  │  Handles: append, read, segment rolling, retention    │     │
│  └───────────────────────────────────────────────────────┘     │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Replica     │  │  Group       │  │  Controller          │  │
│  │  Manager     │  │  Coordinator │  │  (if elected)        │  │
│  │  (ISR, fetch)│  │  (offsets)   │  │  (KRaft/ZK)          │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ 2. Network Layer — How Requests Arrive

### The Reactor Pattern (Acceptor + Processors + Handlers)

Kafka uses a **non-blocking I/O reactor pattern** — a key reason for its high performance:

```
                INCOMING TCP CONNECTIONS
                          │
                    ┌─────▼──────┐
                    │  Acceptor  │  (1 thread)
                    │  Thread    │  Accepts new connections
                    └─────┬──────┘  Assigns to a Processor
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │Processor │ │Processor │ │Processor │  (num.network.threads = 3 default)
        │Thread 1  │ │Thread 2  │ │Thread 3  │  Non-blocking NIO
        │          │ │          │ │          │  Reads bytes off socket
        │ Reads &  │ │ Reads &  │ │ Reads &  │  Decodes request headers
        │ Decodes  │ │ Decodes  │ │ Decodes  │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │             │             │
             └─────────────▼─────────────┘
                     Request Queue
                           │
              ┌────────────▼────────────┐
              │    I/O Thread Pool      │  (num.io.threads = 8 default)
              │  ┌──────┐ ┌──────┐ ... │  Blocking I/O — actual disk reads/writes
              │  │ IO-1 │ │ IO-2 │     │  Handles: Produce, Fetch, Metadata, etc.
              │  └──────┘ └──────┘     │
              └─────────────────────────┘
                           │
                    Response Queue
                           │
              ┌────────────▼────────────┐
              │   Processor sends       │
              │   response back to      │
              │   client socket         │
              └─────────────────────────┘
```

**Key numbers:**
- `num.network.threads` (default 3) — Processor threads for reading/writing from sockets
- `num.io.threads` (default 8) — Handler threads for actual request processing
- `queued.max.requests` (default 500) — Max requests in queue before back-pressure

---

### Request Types (KafkaRequestHandler)

Every client interaction is a typed request:

| Request Type | From | Purpose |
|-------------|------|---------|
| `Produce` | Producer | Write messages to a partition |
| `Fetch` | Consumer / Follower | Read messages from a partition |
| `Metadata` | Any client | Get broker/partition leader info |
| `FindCoordinator` | Consumer | Find Group/Transaction Coordinator |
| `JoinGroup` | Consumer | Join a consumer group |
| `SyncGroup` | Consumer | Get partition assignment |
| `OffsetCommit` | Consumer | Commit offsets to `__consumer_offsets` |
| `OffsetFetch` | Consumer | Get committed offsets |
| `ListOffsets` | Consumer/Admin | Get earliest/latest offset for partition |
| `CreateTopics` | Admin | Create new topics |
| `LeaderAndIsr` | Controller | Update partition leadership metadata |

---

## ⚙️ 3. The Log — Heart of the Broker

### Partition Log Structure

Each partition is a **directory** on disk containing **segment files**:

```
/kafka-logs/
  └── order-events-0/              ← Topic "order-events", Partition 0
        ├── 00000000000000000000.log       ← Segment 1: offsets 0–999
        ├── 00000000000000000000.index     ← Offset index for segment 1
        ├── 00000000000000000000.timeindex ← Timestamp index for segment 1
        ├── 00000000000000001000.log       ← Segment 2: offsets 1000–1999
        ├── 00000000000000001000.index
        ├── 00000000000000001000.timeindex
        ├── 00000000000000002000.log       ← Active segment (being written)
        ├── 00000000000000002000.index
        └── 00000000000000002000.timeindex

  └── order-events-1/              ← Topic "order-events", Partition 1
        └── ...
```

**Segment file naming:** The file name = **base offset** of that segment (first message's offset in that file).

---

### Inside a Log Segment (.log file)

Each `.log` file is a sequence of **record batches**:

```
.log file binary format:

┌──────────────────────────────────────────────────────────┐
│                    Record Batch 1                        │
│  baseOffset=0  lastOffset=4  magic=2  crc=xxx            │
│  attributes: compression=NONE, timestampType=CREATE      │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ │
│  │Record 0│ │Record 1│ │Record 2│ │Record 3│ │Record 4│ │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘ │
└──────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────┐
│                    Record Batch 2                        │
│  baseOffset=5  lastOffset=9  magic=2  crc=xxx            │
│  attributes: compression=LZ4                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │   [Record 5 + 6 + 7 + 8 + 9 compressed as LZ4]    │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
    ↑
    Compressed at BATCH level — all 5 records in one compressed block
```

Each **Record** inside the batch contains:
```
offsetDelta   (relative to batch baseOffset)
timestampDelta
keyLength + key bytes
valueLength + value bytes
headers[]
```

---

### Index Files — Fast Random Access

The `.index` file is a **sparse index** mapping offsets to file positions:

```
.index file (offset → file byte position):

Offset 0    → byte position 0
Offset 100  → byte position 8200
Offset 200  → byte position 16400
Offset 300  → byte position 24600
...

(Not every offset is indexed — only every Nth, controlled by index.interval.bytes)

To find offset 150:
  1. Binary search .index for largest offset ≤ 150 → find "Offset 100 → pos 8200"
  2. Seek to byte 8200 in .log file
  3. Scan forward from there until offset 150 is found
  → O(log n) index lookup + small sequential scan = very fast
```

The `.timeindex` file similarly maps timestamps to offsets — used for `--to-datetime` consumer resets.

---

### The Write Path — Why Kafka is So Fast

```
Producer sends batch to broker:

Step 1: Receive
  Network thread reads bytes from socket into ByteBuffer

Step 2: Validate
  CRC check, magic byte check, offset validation

Step 3: Append to Page Cache
  Log.append() writes to OS page cache (NOT directly to disk)
  Returns immediately after page cache write ← THIS IS THE SPEED SECRET

Step 4: Update Index
  Offset index updated in memory

Step 5: Fsync (optional)
  flush.messages or flush.ms config controls when OS flushes to physical disk
  Default: let OS decide (efficient, but recovery depends on replicas)

Step 6: Update LEO (Log End Offset)
  Leader updates its Log End Offset — followers will fetch up to this

Step 7: Replicate
  Follower FetchRequest pulls new records from leader
  Follower appends to its own log
  Follower updates HW (High Watermark) when ISR catches up
```

---

### Page Cache — The Real Reason for Speed

```
OS Page Cache sits between Kafka's process and physical disk:

Kafka write:
  producer data → OS Page Cache → (async) → Physical Disk

Kafka read:
  Consumer requests offset 500
  → Broker asks OS: "give me bytes at position X in file"
  → If in page cache: returns immediately (no disk I/O!)
  → If not in cache: disk read, cache it, return

Why this matters:
  Producers write → data lands in page cache
  Consumers read (often recent data) → served from page cache
  → Producer → Consumer pipeline: NO DISK I/O AT ALL for hot data!

Consumer reading fresh data (within minutes of production):
  Producer writes: Page Cache ← data
  Consumer reads:  Page Cache → data
  Physical disk never touched! Zero I/O latency.
```

---

### Zero-Copy Transfer (sendfile)

For data that IS on disk, Kafka uses OS `sendfile()` syscall:

```
Traditional copy (WITHOUT zero-copy):
  Disk → Kernel buffer → User buffer (Kafka process) → Socket buffer → NIC
          (copy 1)        (copy 2)                      (copy 3)
  4 context switches, 3 data copies

Zero-copy with sendfile():
  Disk → Kernel buffer → NIC
          (1 DMA copy)   (1 DMA copy)
  2 context switches, 0 CPU copies

Kafka uses FileChannel.transferTo() which maps to sendfile() on Linux
Result: 60–70% reduction in CPU usage for consumer fetch operations
```

---

### High Watermark (HW) and Log End Offset (LEO)

```
Partition 0 on Leader (Broker 1):

  LEO = 10  (Leader has written offsets 0–9)
  HW  = 8   (All ISR members have replicated up to offset 7)

  [0][1][2][3][4][5][6][7][8][9]
                          ↑    ↑
                         HW   LEO

  Consumers can only read up to HW (offset 7)
  Offsets 8 and 9 are "in-flight" — not yet replicated to all ISR members
  → If leader crashes now, 8 and 9 might be lost → don't expose to consumers yet

Follower (Broker 2):
  Has replicated 0–7 (caught up)
  Fetches 8 and 9 from leader
  After receiving and writing: follower LEO = 10
  Leader advances HW to 10 (all ISR have offset 9)
  Consumers can now read 8 and 9
```

---

## 💻 4. Code Examples (Java + Spring Boot)

### Reading Broker-Level Metrics via AdminClient

```java
package com.example.kafka.admin;

import org.apache.kafka.clients.admin.*;
import org.apache.kafka.common.Metric;
import org.apache.kafka.common.MetricName;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.stereotype.Service;

import java.util.Collection;
import java.util.List;
import java.util.Map;

@Service
public class BrokerMetricsInspector {

    private final KafkaAdmin kafkaAdmin;

    public BrokerMetricsInspector(KafkaAdmin kafkaAdmin) {
        this.kafkaAdmin = kafkaAdmin;
    }

    public void printLogDirInfo() throws Exception {
        try (AdminClient client =
                AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            // Get log directory info for all brokers
            Collection<Integer> brokerIds = client.describeCluster()
                .nodes().get().stream()
                .map(node -> node.id())
                .toList();

            DescribeLogDirsResult logDirsResult =
                client.describeLogDirs(brokerIds);

            Map<Integer, Map<String, LogDirDescription>> logDirs =
                logDirsResult.allDescriptions().get();

            logDirs.forEach((brokerId, dirs) -> {
                System.out.println("Broker " + brokerId + ":");
                dirs.forEach((dirPath, dirDesc) -> {
                    System.out.println("  Dir: " + dirPath);
                    dirDesc.replicaInfos().forEach((tp, replicaInfo) -> {
                        System.out.printf(
                            "    %s: size=%d bytes, offsetLag=%d, isFuture=%b%n",
                            tp, replicaInfo.size(),
                            replicaInfo.offsetLag(),
                            replicaInfo.isFuture()
                        );
                    });
                });
            });
        }
    }

    public void printUnderReplicatedPartitions(String topic) throws Exception {
        try (AdminClient client =
                AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            Map<String, TopicDescription> topics =
                client.describeTopics(List.of(topic))
                      .allTopicNames().get();

            System.out.println("Under-replicated check for: " + topic);
            topics.get(topic).partitions().forEach(partition -> {
                int replicaCount = partition.replicas().size();
                int isrCount = partition.isr().size();
                boolean underReplicated = isrCount < replicaCount;

                System.out.printf(
                    "  Partition %d → replicas=%d isr=%d %s%n",
                    partition.partition(),
                    replicaCount,
                    isrCount,
                    underReplicated ? "⚠️ UNDER-REPLICATED" : "✅ OK"
                );
            });
        }
    }
}
```

---

### Monitoring Producer Metrics

```java
package com.example.kafka.monitor;

import org.apache.kafka.common.Metric;
import org.apache.kafka.common.MetricName;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
public class ProducerMetricsMonitor {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public ProducerMetricsMonitor(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    @Scheduled(fixedDelay = 10000)  // Every 10 seconds
    public void printKeyMetrics() {
        kafkaTemplate.metrics().forEach((metricName, metric) -> {
            String name = metricName.name();

            // Filter for key metrics
            if (name.equals("record-send-rate")
                    || name.equals("record-error-rate")
                    || name.equals("batch-size-avg")
                    || name.equals("compression-rate-avg")
                    || name.equals("request-latency-avg")
                    || name.equals("buffer-available-bytes")) {

                System.out.printf("  [METRIC] %-35s = %.2f%n",
                    name, (double) metric.metricValue());
            }
        });
    }
}
```

---

### Custom Interceptor — Request Tracing

```java
package com.example.kafka.interceptor;

import org.apache.kafka.clients.producer.ProducerInterceptor;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Map;

// Intercepts every message before it's sent to broker
public class TracingProducerInterceptor
        implements ProducerInterceptor<String, Object> {

    @Override
    public ProducerRecord<String, Object> onSend(
            ProducerRecord<String, Object> record) {

        // Add trace header automatically to every message
        record.headers().add("intercepted-at",
            String.valueOf(System.currentTimeMillis()).getBytes());
        record.headers().add("app-version", "1.0.0".getBytes());

        System.out.printf("[INTERCEPTOR] Sending to %s key=%s%n",
            record.topic(), record.key());
        return record;
    }

    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception ex) {
        if (ex != null) {
            System.err.println("[INTERCEPTOR] Send failed: " + ex.getMessage());
        } else {
            System.out.printf("[INTERCEPTOR] ACK: partition=%d offset=%d%n",
                metadata.partition(), metadata.offset());
        }
    }

    @Override public void close() {}
    @Override public void configure(Map<String, ?> configs) {}
}
```

Register in config:
```java
config.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,
    TracingProducerInterceptor.class.getName());
```

---

### Broker Config Tuning via AdminClient

```java
public void updateBrokerConfig(int brokerId, String key, String value)
        throws Exception {

    try (AdminClient client =
            AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

        ConfigResource brokerResource =
            new ConfigResource(ConfigResource.Type.BROKER,
                String.valueOf(brokerId));

        AlterConfigOp op = new AlterConfigOp(
            new ConfigEntry(key, value),
            AlterConfigOp.OpType.SET
        );

        client.incrementalAlterConfigs(
            Map.of(brokerResource, List.of(op))
        ).all().get();

        System.out.printf("Updated broker %d: %s=%s%n", brokerId, key, value);
    }
}

// Example usage:
// updateBrokerConfig(1, "log.retention.hours", "72");
// updateBrokerConfig(1, "num.io.threads", "16");
```

---

## ⚡ 5. Key Broker Configurations

| Config | Default | Tuning Note |
|--------|---------|-------------|
| `num.network.threads` | `3` | Increase for many concurrent connections (set to CPU cores) |
| `num.io.threads` | `8` | Increase for high disk I/O (set to 2× CPU cores) |
| `socket.send.buffer.bytes` | `102400` | TCP send buffer. Set to -1 for OS default |
| `socket.receive.buffer.bytes` | `102400` | TCP receive buffer |
| `socket.request.max.bytes` | `104857600` | Max request size (100MB) |
| `log.dirs` | `/tmp/kafka-logs` | Use multiple dirs across disks for parallelism |
| `log.flush.interval.messages` | `Long.MAX_VALUE` | Disable explicit fsync — rely on OS + replication |
| `log.flush.interval.ms` | `Long.MAX_VALUE` | Same as above |
| `log.segment.bytes` | `1073741824` (1GB) | Segment size. Smaller = more files, faster retention |
| `log.index.interval.bytes` | `4096` | Index entry every 4KB — controls index density |
| `replica.fetch.max.bytes` | `1048576` (1MB) | Max bytes fetched per partition per replica fetch |
| `replica.lag.time.max.ms` | `30000` | Time before follower removed from ISR |

---

## 🔄 6. Real-World Scenarios

### Scenario: Tuning Broker for High Throughput

```
Hardware: 32-core machine, 256GB RAM, 12x NVMe SSDs

Tuning:
  # Use multiple log dirs — spreads I/O across all 12 SSDs
  log.dirs=/disk1/kafka,/disk2/kafka,...,/disk12/kafka

  # More network threads for many concurrent connections
  num.network.threads=12

  # More I/O threads for heavy disk workload
  num.io.threads=24

  # Larger buffers
  socket.send.buffer.bytes=1048576     # 1MB
  socket.receive.buffer.bytes=1048576

  # Don't fsync explicitly — trust OS page cache + replicas for durability
  log.flush.interval.messages=Long.MAX_VALUE
  log.flush.interval.ms=Long.MAX_VALUE

  # Allow large batches
  message.max.bytes=10485760   # 10MB max message size
```

### Scenario: Understanding HW and Consumer Lag

```
Scenario: New partition added mid-replication

Leader LEO = 1000
Follower LEO = 800  (still fetching)
HW = 800  (HW = min(all ISR LEOs))

Consumers can only read up to offset 799!
They appear to have lag = 200 even though leader has 1000 messages.

This is NOT a bug — it's the High Watermark safety mechanism.
Consumers will catch up as the follower catches up and HW advances.

Fix if stuck: Check if follower is actually fetching
  (under-replicated partitions alert)
```

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| `log.dirs` pointing to single disk | All partitions on one I/O path → bottleneck | Spread across multiple disks |
| Low `num.io.threads` on busy broker | Request queue fills → high latency → timeouts | Increase to 2× CPU cores |
| Explicit fsync (`log.flush.interval.messages=1`) | Catastrophic write throughput drop | Rely on OS page cache + replication for durability |
| Large `log.segment.bytes` | Old segments can't be deleted even if retention exceeded | Balance segment size with retention needs |
| Single `log.dirs` path | No parallelism across disks | Mount multiple disks, list all paths |
| Ignoring `queued.max.requests` | Request queue overflow drops connections | Increase or add brokers |

---

## 🎯 8. Interview Questions

**Q1. How does Kafka achieve such high write throughput?**
> Kafka writes sequentially to the OS page cache (not directly to disk), which is extremely fast. The OS asynchronously flushes to disk in the background. Additionally, Kafka batches multiple records into a single write operation, uses compression at batch level, and relies on replicas for durability rather than expensive fsync calls.

**Q2. What is the High Watermark (HW) and why does it matter?**
> The High Watermark is the offset up to which all ISR replicas have successfully replicated. Consumers can only read messages up to the HW. This prevents consumers from reading data that hasn't been replicated — if the leader crashes, only data up to HW is guaranteed to survive, so exposing uncommitted data would cause inconsistency.

**Q3. What is zero-copy and how does Kafka use it?**
> Zero-copy uses the OS `sendfile()` syscall (exposed as `FileChannel.transferTo()` in Java) to transfer data directly from disk buffer to network socket buffer without copying through the JVM's user space. This eliminates CPU data copies and context switches, reducing CPU usage by 60–70% for consumer fetch operations.

**Q4. What are the Acceptor and Processor threads in Kafka?**
> The Acceptor thread (1 per listener) accepts incoming TCP connections and assigns them to Processor threads. Processor threads (configurable via `num.network.threads`) use non-blocking NIO to read requests from sockets and write responses back. They don't do the actual request processing — they enqueue decoded requests for the I/O thread pool.

**Q5. What is the difference between LEO and HW?**
> LEO (Log End Offset) is the offset of the next message to be written — the very tip of a replica's log. HW (High Watermark) is the offset up to which all ISR members have the data. LEO ≥ HW always. Consumers can only read up to HW. The difference (LEO - HW) represents in-flight, not-yet-fully-replicated messages.

**Q6. Why should you avoid `log.flush.interval.messages=1` in production?**
> It forces an fsync to physical disk for every single message, bypassing the OS page cache mechanism. Disk fsync is orders of magnitude slower than page cache writes. With `acks=all` and replication-factor ≥ 3, durability is already guaranteed by replication — explicit fsync is unnecessary and devastates throughput.

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "Why Is My Consumer Stuck at an Offset?"
> **Problem:** Your consumer's latest offset is 1000 but the log end offset is 5000. The consumer isn't reading new messages. Kafka UI shows HW = 1000. What's happening?

**Answer:**
```
HW = 1000 means all ISR members have only replicated up to offset 999.
Even though the leader has up to offset 4999, consumers can only read up to HW.

Diagnosis:
  kafka-topics.sh --describe → check ISR size
  If ISR = [leader only] → follower(s) have fallen behind

Root cause options:
  1. Follower broker is slow/overloaded → can't keep up with leader
  2. Follower broker is down/crashed → removed from ISR
  3. Network issue between leader and follower
  4. replica.lag.time.max.ms exceeded → follower removed from ISR

Fix:
  1. Check follower broker health: docker logs / JVM metrics
  2. If follower crashed: restart it, let it catch up
  3. Monitor UnderReplicatedPartitions metric → should be 0
  4. Once follower rejoins ISR and catches up → HW advances → consumers unblocked
```

---

### Scenario 2: "Broker Disk Full"
> **Problem:** One of your Kafka brokers is reporting disk full. It's the leader for 40 partitions. What are your immediate steps?

**Answer:**
```
Immediate (stop the bleeding):
  1. Temporarily reduce retention:
     kafka-configs.sh --alter --entity-type topics --entity-name <critical-topics>
       --add-config retention.ms=3600000  (1 hour instead of 7 days)
     → Kafka will delete old segments immediately

  2. Delete obsolete topics (if any)

  3. Trigger log cleanup:
     kafka-log-dirs.sh → identify largest partitions consuming space

Short-term:
  4. Add a new disk → add path to log.dirs → rebalance partitions to new disk

Medium-term:
  5. Add a new broker → reassign some of this broker's partitions to it
  6. Review retention policy — should it really be 7 days for all topics?
  7. Enable compression on heavy topics → saves 30–70% disk space

Alert setup (prevention):
  Alert on broker disk usage > 70%  (before it hits 100%)
  Monitor: kafka.log:type=LogFlushStats
```

---

## 📝 10. Quick Revision Summary

```
✅ Broker network architecture: Acceptor → Processors → Request Queue → I/O Threads
   num.network.threads = NIO processors (read/write sockets)
   num.io.threads      = actual request handlers (disk I/O)

✅ Log structure per partition:
   Directory → Segment files (.log, .index, .timeindex)
   .log      = record batches (compressed at batch level)
   .index    = sparse offset → file position index (O(log n) lookup)
   .timeindex = timestamp → offset index (for datetime-based resets)

✅ Why Kafka is fast:
   Sequential writes → OS page cache → async to disk
   Zero-copy sendfile() for consumer fetches (no JVM copy)
   Batch-level compression
   Index-based random access (no full scan)

✅ LEO (Log End Offset) = tip of partition log (latest written offset)
✅ HW  (High Watermark) = highest offset replicated by ALL ISR members
   Consumers can only read up to HW (never beyond)
   HW advances as followers replicate and acknowledge

✅ Page cache:
   Hot data (recently written) served from RAM → zero disk I/O
   Producer→Consumer pipeline: often never hits physical disk

✅ Key tuning:
   log.dirs = multiple disks (parallel I/O)
   num.io.threads = 2× CPU cores
   NEVER set log.flush.interval.messages=1 in production
```

---

**← Previous:** [08 — Message Delivery Semantics](./08-delivery-semantics.md)  
**Next Topic →** [10 — Partitioning Strategies: Keys, Custom Partitioners & Ordering](./10-partitioning.md)
