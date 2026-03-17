# 05 — Kafka Producers: Internals, Configs & Guarantees
> **Phase 2 — Producers & Consumers (Core Mechanics)**  
> 📁 `notes/05-producers.md`

---

## 📖 1. Concept Explanation

### What is a Kafka Producer?

A **Kafka Producer** is a client application that **publishes (writes) records to Kafka topics**. The producer is responsible for:

- Choosing which **topic** to write to
- Choosing which **partition** within that topic (via key or custom logic)
- **Serializing** the key and value into bytes
- **Batching** messages for efficiency
- **Retrying** on failure
- Deciding the **durability guarantee** (`acks`)

The producer is far more than a simple "send message" client — it has a sophisticated internal pipeline that dramatically affects **throughput, latency, and durability**.

---

### The Producer's Mental Model

```
Your Code
    │
    │ kafkaTemplate.send("payments", key, value)
    ↓
┌─────────────────────────────────────────────────────────────────┐
│                    PRODUCER INTERNALS                           │
│                                                                 │
│  1. Serialize        key/value → bytes                          │
│  2. Partition        which partition? (key hash / round-robin)  │
│  3. RecordAccumulator batch messages per partition              │
│  4. Sender thread    background thread flushes batches          │
│  5. NetworkClient    sends batches over TCP to broker           │
│  6. Retry logic      retries on retriable errors                │
│  7. ACK handling     waits for broker acknowledgement           │
└─────────────────────────────────────────────────────────────────┘
    │
    ↓
Kafka Broker (Partition Leader)
```

---

## 🏗️ 2. Architecture / Flow — Producer Internals

### Inside the Producer — Detailed Flow

```
Producer.send(record)
       │
       ▼
┌─────────────────────┐
│   Interceptors      │  Optional: modify/inspect records before send
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   Serializer        │  key.serializer, value.serializer
│   key   → bytes     │  e.g., StringSerializer, JsonSerializer
│   value → bytes     │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   Partitioner       │  Which partition?
│                     │  key != null → murmur2(key) % numPartitions
│                     │  key == null → StickyPartitioner (batch-first)
└────────┬────────────┘
         │
         ▼
┌───────────────────────────────────────────────────────────┐
│                  RecordAccumulator                        │
│                                                           │
│  ProducerBatch[topic-partition-0]:  [r1][r2][r3]...       │
│  ProducerBatch[topic-partition-1]:  [r4][r5]...           │
│  ProducerBatch[topic-partition-2]:  [r6]...               │
│                                                           │
│  Batch is sent when:                                      │
│    - batch.size reached (default 16KB)         OR         │
│    - linger.ms elapsed (default 0ms)           OR         │
│    - buffer.memory is full (blocks/throws)                │
└──────────────────────────┬────────────────────────────────┘
                           │  (background Sender thread)
                           ▼
                  ┌─────────────────┐
                  │  NetworkClient  │  TCP connection to broker
                  │  (async I/O)    │
                  └────────┬────────┘
                           │
                           ▼
                  Kafka Broker (Partition Leader)
                           │
                           ▼
                  ACK response → CompletableFuture resolved
```

---

### The Sender Thread — Async by Default

The producer's `send()` method is **non-blocking** by default. It places the record into the `RecordAccumulator` and returns immediately. A **background Sender thread** drains the accumulator and sends batches to brokers.

```
Main Thread:
  send(r1) → accumulator  [returns immediately]
  send(r2) → accumulator  [returns immediately]
  send(r3) → accumulator  [returns immediately]

Background Sender Thread:
  [waits for batch.size or linger.ms]
  → flushes [r1, r2, r3] as one batch to broker
  → receives ACK from broker
  → resolves CompletableFuture for each record
```

---

## ⚙️ 3. How It Works Internally

### Batching Deep Dive

Two configs control when a batch is flushed:

```
batch.size  (default: 16384 bytes = 16 KB)
  → Flush when accumulated bytes for a partition reach this threshold
  → Larger batch = better throughput, higher latency

linger.ms   (default: 0 ms)
  → Wait this long before flushing even if batch isn't full
  → linger.ms=0 → send immediately (low latency, small batches)
  → linger.ms=5 → wait 5ms (better batching, slight latency increase)
```

```
linger.ms=0 (default):
  Time: 0ms  → send(r1) → batch=[r1] → FLUSH immediately
  Time: 1ms  → send(r2) → batch=[r2] → FLUSH immediately
  Result: 2 separate network requests, low throughput

linger.ms=10:
  Time: 0ms  → send(r1) → batch=[r1]
  Time: 3ms  → send(r2) → batch=[r1, r2]
  Time: 7ms  → send(r3) → batch=[r1, r2, r3]
  Time: 10ms → FLUSH batch=[r1, r2, r3] as ONE request
  Result: 1 network request, 3x throughput
```

---

### Acknowledgement Modes (acks)

`acks` is the most critical producer config — it controls the **durability guarantee**:

```
acks=0  (Fire and forget)
  Producer sends → does NOT wait for any ACK
  Broker may not have even received it
  Highest throughput, NO guarantee
  Use case: metrics, logs where loss is acceptable

  Producer ──→ Broker (leader)
               (no response needed)

──────────────────────────────────────────────────

acks=1  (Leader ACK only)
  Producer sends → waits for Leader to write to its log
  Leader ACKs → producer considers it done
  If leader crashes BEFORE replicating → data loss possible
  Moderate throughput, moderate guarantee

  Producer ──→ Broker1 (leader writes to disk)
               ← ACK
               [Broker2, Broker3 replicate async — not waited]

──────────────────────────────────────────────────

acks=all  (or acks=-1)  ← RECOMMENDED for production
  Producer sends → waits for ALL ISR members to write
  Leader waits for every ISR follower to confirm
  No data loss as long as min.insync.replicas is met
  Lower throughput (latency = slowest ISR member's write)

  Producer ──→ Broker1 (leader)
                   ├──→ Broker2 (follower ACK)
                   └──→ Broker3 (follower ACK)
               ← ACK only after all ISR members confirm
```

---

### Retries and Idempotence

**Retries without idempotence = duplicate messages:**

```
Scenario (acks=all, retries=3):
  1. Producer sends batch to Broker1
  2. Broker1 writes, replicates, sends ACK back
  3. ACK is lost in the network
  4. Producer times out, retries the SAME batch
  5. Broker1 receives duplicate → writes it again
  Result: Message stored TWICE → consumer sees duplicates
```

**Enable idempotent producer to fix this:**

```
enable.idempotence=true

Each producer gets a unique ProducerID (PID)
Each message gets a sequence number
Broker tracks: "I've already seen PID=42, seq=7"
If duplicate arrives → broker silently drops it
Result: Exactly-once delivery at the producer level
```

```
Producer (PID=42):
  send seq=1 → Broker (stores, ACK lost) → retry seq=1 → Broker (DROPS duplicate ✅)
  send seq=2 → Broker (stores, ACK received) ✅
```

> ⚠️ `enable.idempotence=true` requires `acks=all` and `retries > 0`. Spring Kafka sets this automatically when you configure `acks=all`.

---

### Compression

Kafka supports **batch-level compression** — the entire batch is compressed as one unit:

```
compression.type = none | gzip | snappy | lz4 | zstd

Performance comparison:
  none   → Fastest write, largest disk/network usage
  lz4    → Best balance (fast compression, good ratio) ← production default
  snappy → Similar to lz4, older Google codec
  gzip   → Best compression ratio, slowest CPU
  zstd   → Best ratio with reasonable speed (Kafka 2.1+) ← recommended

Compression happens in the producer (saves network bandwidth)
Decompression happens in the consumer (transparent)
Broker stores the compressed batch as-is (no recompression)
```

---

## 💻 4. Code Examples (Java + Spring Boot)

### `pom.xml`

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- For JSON serialization -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

---

### `application.yml` — Production-Grade Producer Config

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      # Serializers
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

      # Durability
      acks: all                        # Wait for all ISR members
      retries: 2147483647              # Max retries (Integer.MAX_VALUE)

      # Batching & Throughput
      batch-size: 32768                # 32 KB batch size
      properties:
        linger.ms: 20                  # Wait up to 20ms to fill batches
        buffer.memory: 67108864        # 64 MB producer buffer
        compression.type: lz4          # Compress batches

        # Idempotence (exactly-once at producer level)
        enable.idempotence: true

        # Ordering guarantees with retries
        max.in.flight.requests.per.connection: 5   # Safe with idempotence=true

        # Timeouts
        request.timeout.ms: 30000      # 30s per request
        delivery.timeout.ms: 120000    # 2min total delivery timeout
```

---

### Basic Producer — `OrderProducer.java`

```java
package com.example.kafka.producer;

import com.example.kafka.model.OrderEvent;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.header.internals.RecordHeader;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class OrderProducer {

    private static final String TOPIC = "order-events";
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public OrderProducer(KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // ── Basic send ──────────────────────────────────────────────
    public void sendOrder(OrderEvent event) {
        // key = orderId ensures all events for one order go to same partition
        CompletableFuture<SendResult<String, OrderEvent>> future =
            kafkaTemplate.send(TOPIC, event.getOrderId(), event);

        future.whenComplete((result, ex) -> {
            if (ex != null) {
                System.err.printf("FAILED to send order %s: %s%n",
                    event.getOrderId(), ex.getMessage());
            } else {
                System.out.printf("SENT order %s → partition=%d offset=%d%n",
                    event.getOrderId(),
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            }
        });
    }

    // ── Send with headers (tracing, versioning) ─────────────────
    public void sendWithHeaders(OrderEvent event, String traceId) {
        ProducerRecord<String, OrderEvent> record = new ProducerRecord<>(
            TOPIC, null, event.getOrderId(), event
        );
        record.headers().add(new RecordHeader("trace-id", traceId.getBytes()));
        record.headers().add(new RecordHeader("source",   "order-service".getBytes()));
        record.headers().add(new RecordHeader("version",  "v2".getBytes()));

        kafkaTemplate.send(record);
    }

    // ── Synchronous send (blocks until ACK) ─────────────────────
    public void sendSync(OrderEvent event) throws Exception {
        // .get() blocks until broker ACKs — use only for critical path
        SendResult<String, OrderEvent> result =
            kafkaTemplate.send(TOPIC, event.getOrderId(), event).get();

        System.out.printf("Synchronously confirmed: partition=%d offset=%d%n",
            result.getRecordMetadata().partition(),
            result.getRecordMetadata().offset());
    }
}
```

---

### The Event Model — `OrderEvent.java`

```java
package com.example.kafka.model;

import com.fasterxml.jackson.annotation.JsonFormat;
import java.time.LocalDateTime;

public class OrderEvent {

    private String orderId;
    private String status;          // CREATED, PAID, SHIPPED, DELIVERED
    private String customerId;
    private double amount;

    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    private LocalDateTime timestamp;

    // Constructors
    public OrderEvent() {}

    public OrderEvent(String orderId, String status, String customerId, double amount) {
        this.orderId    = orderId;
        this.status     = status;
        this.customerId = customerId;
        this.amount     = amount;
        this.timestamp  = LocalDateTime.now();
    }

    // Getters & Setters (or use Lombok @Data)
    public String getOrderId()    { return orderId; }
    public String getStatus()     { return status; }
    public String getCustomerId() { return customerId; }
    public double getAmount()     { return amount; }
    public LocalDateTime getTimestamp() { return timestamp; }

    public void setOrderId(String orderId)       { this.orderId = orderId; }
    public void setStatus(String status)         { this.status = status; }
    public void setCustomerId(String customerId) { this.customerId = customerId; }
    public void setAmount(double amount)         { this.amount = amount; }
    public void setTimestamp(LocalDateTime t)    { this.timestamp = t; }
}
```

---

### Programmatic Producer Config (Full Control)

```java
package com.example.kafka.config;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.DefaultKafkaProducerFactory;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.core.ProducerFactory;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();

        // Connection
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        // Serializers
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,   StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // Durability
        props.put(ProducerConfig.ACKS_CONFIG,    "all");
        props.put(ProducerConfig.RETRIES_CONFIG,  Integer.MAX_VALUE);

        // Batching
        props.put(ProducerConfig.BATCH_SIZE_CONFIG,       32 * 1024);   // 32 KB
        props.put(ProducerConfig.LINGER_MS_CONFIG,        20);           // 20 ms
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG,    64 * 1024 * 1024L); // 64 MB
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

        // Idempotence
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);

        // Timeouts
        props.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG,  30_000);
        props.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120_000);

        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

---

### Producer with Transaction Support (Exactly-Once)

```java
package com.example.kafka.producer;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class TransactionalOrderProducer {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public TransactionalOrderProducer(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // All sends in this method are atomic — either all succeed or none do
    @Transactional
    public void sendOrderAndPayment(Object orderEvent, Object paymentEvent) {
        kafkaTemplate.send("order-events",  "o-100", orderEvent);
        kafkaTemplate.send("payment-events","o-100", paymentEvent);
        // If any send fails → both are rolled back (not committed to Kafka)
    }
}
```

**`application.yml` addition for transactions:**
```yaml
spring:
  kafka:
    producer:
      transaction-id-prefix: order-tx-  # Enables transactional producer
```

---

### Interceptor — Cross-Cutting Concerns

```java
package com.example.kafka.interceptor;

import org.apache.kafka.clients.producer.ProducerInterceptor;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Map;

// Add this to: spring.kafka.producer.properties.interceptor.classes
public class TracingProducerInterceptor implements ProducerInterceptor<String, Object> {

    @Override
    public ProducerRecord<String, Object> onSend(ProducerRecord<String, Object> record) {
        // Add trace ID to every record automatically
        record.headers().add("sent-at",
            String.valueOf(System.currentTimeMillis()).getBytes());
        return record;
    }

    @Override
    public void onAcknowledgement(RecordMetadata metadata, Exception exception) {
        if (exception != null) {
            System.err.println("Send failed: " + exception.getMessage());
        }
    }

    @Override public void close() {}
    @Override public void configure(Map<String, ?> configs) {}
}
```

---

## ⚡ 5. Key Configurations Reference

| Config | Default | Recommended (Prod) | Impact |
|--------|---------|-------------------|--------|
| `acks` | `1` | `all` | Durability |
| `retries` | `2147483647` | `2147483647` | Reliability |
| `enable.idempotence` | `true` (Kafka 3+) | `true` | Duplicate prevention |
| `batch.size` | `16384` (16KB) | `32768–65536` | Throughput |
| `linger.ms` | `0` | `5–20` | Throughput vs latency |
| `buffer.memory` | `33554432` (32MB) | `67108864` (64MB) | Buffer for spikes |
| `compression.type` | `none` | `lz4` or `zstd` | Network + disk savings |
| `max.in.flight.requests.per.connection` | `5` | `5` (with idempotence) | Ordering + throughput |
| `request.timeout.ms` | `30000` | `30000` | Per-request timeout |
| `delivery.timeout.ms` | `120000` | `120000` | Total end-to-end timeout |

---

## 🔄 6. Real-World Scenarios

### Throughput vs Latency Tradeoff

```
Low Latency Setup (real-time alerts, trading):
  linger.ms=0
  batch.size=16384
  acks=1
  compression.type=none
  → Each message sent immediately, small batches, fast ACK

High Throughput Setup (analytics pipeline, logging):
  linger.ms=20
  batch.size=65536
  acks=all
  compression.type=lz4
  → Messages batched, fewer network calls, compressed
```

### buffer.memory Full Scenario

```
Producer sends faster than broker can accept:

  buffer.memory fills up (default 32MB)
  Producer blocks for max.block.ms (default 60s)
  If still full after max.block.ms → throws BufferExhaustedException

Fixes:
  1. Increase buffer.memory
  2. Increase batch throughput (linger.ms, batch.size)
  3. Add more partitions to distribute load
  4. Investigate broker bottleneck (slow disk, network)
```

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| `acks=1` in production | Leader crash after ACK but before replication = data loss | Use `acks=all` |
| `retries>0` without `enable.idempotence` | Duplicate messages on retry | Always set `enable.idempotence=true` |
| `linger.ms=0` for high-throughput pipeline | Tiny batches, poor throughput | Set `linger.ms=5–20` |
| Calling `.get()` on every send | Synchronous, kills throughput | Use async callbacks, only `.get()` when strictly needed |
| Not handling `CompletableFuture` failures | Silent message loss | Always attach `.whenComplete()` handler |
| `max.in.flight.requests.per.connection > 5` with idempotence | Not supported, throws exception | Keep at ≤ 5 |
| Ignoring `delivery.timeout.ms` | Retries may happen longer than expected | Set it consciously based on SLA |
| Sending huge messages (>1MB) | Broker rejects, OOM risk | Use `max.request.size`, or store payload in S3/DB and send a reference |

---

## 🎯 8. Interview Questions

**Q1. How does the Kafka producer decide which partition to send a message to?**
> If a key is provided, the **DefaultPartitioner** applies murmur2 hash on the key bytes and takes `abs(hash) % numPartitions`. The same key always maps to the same partition. If no key is provided, the **StickyPartitioner** (default since Kafka 2.4) fills one partition's batch before switching — this improves batching efficiency over pure round-robin.

**Q2. What is the difference between `acks=1` and `acks=all`?**
> `acks=1` means only the partition leader acknowledges the write — followers replicate asynchronously. If the leader crashes before replication, data is lost. `acks=all` (or -1) means the leader waits for ALL in-sync replicas to acknowledge — no committed data is ever lost, but latency is higher (equals slowest ISR member's write time).

**Q3. What problem does `enable.idempotence=true` solve?**
> It prevents **duplicate messages** caused by producer retries. Without idempotence, a retry after a lost ACK causes the broker to store the message twice. With idempotence enabled, each producer gets a unique `ProducerID` and each message gets a sequence number — the broker deduplicates based on these, guaranteeing exactly-once delivery at the producer level.

**Q4. What is the relationship between `batch.size`, `linger.ms`, and throughput?**
> The producer accumulates messages into batches per partition. A batch is sent when either `batch.size` bytes are filled OR `linger.ms` milliseconds have elapsed. `linger.ms=0` sends immediately (low latency, poor batching). Increasing `linger.ms` allows more messages to accumulate per batch, reducing network round trips and dramatically improving throughput at the cost of a small latency increase.

**Q5. What is `delivery.timeout.ms` and how does it relate to `retries`?**
> `delivery.timeout.ms` is the total time the producer will spend trying to deliver a message (including retries and waits). Even with `retries=MAX_INT`, if `delivery.timeout.ms` expires, the send fails. It acts as the **outer time boundary** for the entire delivery attempt, making retry behavior predictable.

**Q6. When would you use a transactional producer?**
> When you need **atomic writes across multiple topics or partitions** — either all messages are committed or none are. Common in event sourcing (write event + update state atomically) and consume-transform-produce pipelines where a consumer reads, transforms, and produces — and you want exactly-once end-to-end semantics.

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "Messages Are Being Duplicated"
> **Problem:** Your payment service uses Kafka to publish payment events. After deploying a new version, you notice each payment event appears twice in the downstream consumer. The producer has `retries=3`, `acks=all`. What happened and how do you fix it?

**Answer:**
```
Root Cause:
  1. Producer sends payment event to broker
  2. Broker writes to disk, replicates to ISR
  3. Broker sends ACK back to producer
  4. ACK is lost in transit (network blip)
  5. Producer times out, retries (sends same message again)
  6. Broker has no duplicate detection → stores it twice

Fix:
  Add to producer config:
    enable.idempotence: true

  Now each message has:
    ProducerID (PID) = unique per producer instance
    Sequence number = per-partition incrementing counter

  Broker checks: "PID=42, seq=7 already seen? YES → drop duplicate"
  Result: Exactly one message stored, regardless of retries
```

---

### Scenario 2: "Throughput Dropped by 80% After Adding acks=all"
> **Problem:** Team switched from `acks=1` to `acks=all` for compliance. Throughput dropped from 500K msgs/sec to 100K msgs/sec. How do you recover throughput while keeping `acks=all`?

**Answer:**
```
The bottleneck: acks=all means waiting for ISR follower writes.
Each message waits individually → high latency per message → low throughput.

Fix — Improve batching to compensate:
  1. Increase linger.ms: 0 → 20
     → More messages batched per request, amortizing round-trip cost

  2. Increase batch.size: 16KB → 64KB
     → Larger batches = more messages per ACK wait

  3. Increase max.in.flight.requests.per.connection: 1 → 5
     → Multiple batches in flight simultaneously (with idempotence=true)

  4. Enable compression:
     compression.type: lz4
     → Smaller batches = faster network transfer = faster ACK

  5. Increase buffer.memory if needed:
     buffer.memory: 128MB
     → Handle traffic spikes without blocking

Result: acks=all durability + recovered throughput via better batching
```

---

### Scenario 3: "Order of Messages is Wrong in Consumer"
> **Problem:** An order goes through states: CREATED → PAID → SHIPPED. Consumers sometimes see PAID before CREATED. The producer sends all three with `orderId` as the key. `retries=3`, `max.in.flight.requests.per.connection=5`. What's the issue?

**Answer:**
```
Root Cause:
  With retries > 0 and max.in.flight > 1 (without idempotence):

  Batch 1: [CREATED] → sent → ACK fails → queued for retry
  Batch 2: [PAID]    → sent → ACK succeeds → offset 5
  Batch 1: [CREATED] → retried → ACK succeeds → offset 6
  
  Consumer sees: PAID (offset 5) before CREATED (offset 6) → OUT OF ORDER!

Fix:
  enable.idempotence=true
  → Kafka guarantees ordering even with retries when idempotence is on
  → Internally uses sequence numbers to reject out-of-order delivery
  → max.in.flight stays at 5 (idempotence makes it safe)

Alternatively (older Kafka):
  max.in.flight.requests.per.connection=1  (but kills throughput)
```

---

## 📝 10. Quick Revision Summary

```
✅ Producer Pipeline:
   Your code → Interceptor → Serializer → Partitioner
   → RecordAccumulator (batch) → Sender thread → NetworkClient → Broker

✅ acks modes:
   acks=0   → fire and forget (no guarantee)
   acks=1   → leader ACK only (possible data loss)
   acks=all → all ISR ACK (no data loss) ← production default

✅ Batching:
   batch.size  = max bytes per batch (16KB default, use 32-64KB)
   linger.ms   = max wait time to fill batch (0 default, use 5-20ms)
   Higher both = better throughput, slightly more latency

✅ Idempotence:
   enable.idempotence=true → deduplicates retries via PID + seq number
   Requires: acks=all, max.in.flight ≤ 5
   Kafka 3+ enables this by default

✅ Compression:
   lz4 or zstd → best production choice
   Applied per batch by producer, decompressed by consumer
   Reduces network + disk usage significantly

✅ Key Spring Boot wiring:
   KafkaTemplate.send(topic, key, value)  → async, returns CompletableFuture
   .get() on future                       → sync (blocks, use sparingly)
   @Transactional + transaction-id-prefix → atomic multi-topic writes
   ProducerInterceptor                    → cross-cutting (tracing, logging)

✅ Golden prod config:
   acks=all + enable.idempotence=true + retries=MAX
   + linger.ms=20 + batch.size=32KB + compression.type=lz4
```

---

**← Previous:** [04 — Setting Up Kafka Locally](./04-local-setup.md)  
**Next Topic →** [06 — Kafka Consumers: Poll Loop, Offsets & Commits](./06-consumers.md)
