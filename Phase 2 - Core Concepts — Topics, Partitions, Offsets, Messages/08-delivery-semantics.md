# 08 — Message Delivery Semantics: At Most / At Least / Exactly Once
> **Phase 2 — Producers & Consumers**  
> 📁 `notes/08-delivery-semantics.md`

---

## 📖 1. Concept Explanation

**Delivery semantics** define the guarantee Kafka makes about whether a message is delivered to its consumer, and **how many times**.

This is the most important correctness concept in distributed messaging. Every Kafka system you build makes an implicit or explicit choice here.

```
At Most Once    → Message delivered 0 or 1 times. Can be LOST. Never duplicated.
At Least Once   → Message delivered 1 or more times. Never lost. Can DUPLICATE.
Exactly Once    → Message delivered exactly 1 time. Never lost. Never duplicated.
```

> **Reality check:** Exactly-once is the hardest to achieve and has performance costs. Most production systems accept at-least-once with **idempotent consumers** as the practical sweet spot.

---

### Why This is Hard in Distributed Systems

The fundamental problem is a **two-phase commit** challenge:

```
Producer sends message →

  Phase 1: Network delivers to broker?
  Phase 2: Broker writes to disk?
  Phase 3: Broker ACKs to producer?
  Phase 4: Producer receives ACK?

If ANY of these fail, the producer doesn't know what happened.
Did the broker write it or not?
→ Safe answer: retry (risks duplicate)
→ Unsafe answer: assume it succeeded (risks loss)
```

The same problem exists on the consumer side:

```
Consumer reads message →

  Phase 1: Consumer fetches from broker
  Phase 2: Consumer processes message
  Phase 3: Consumer commits offset

If Phase 3 fails (crash between 2 and 3):
  → Consumer restarts → replays from old offset → processes AGAIN → DUPLICATE

If Phase 3 happens before Phase 2:
  → Consumer crashes during processing → restarts → old offset already committed
  → Message NOT reprocessed → LOST
```

---

## 🏗️ 2. The Three Semantics — Deep Dive

### 2.1 — At Most Once (May Lose, Never Duplicate)

```
PRODUCER SIDE:
  acks=0 or acks=1
  retries=0 (no retry on failure)

CONSUMER SIDE:
  Commit offset BEFORE processing
  If crash after commit but before processing → message LOST forever

Flow:
  Consumer polls → commits offset immediately → processes message
                                                    ↑
                                          If crash here → message lost
```

**When to use:**
- Metrics, logs, telemetry where occasional loss is acceptable
- Very high throughput scenarios where some loss is a known trade-off
- Cases where reprocessing is more harmful than loss

```
Throughput:   ████████████ Highest
Durability:   ████░░░░░░░░ Low
Complexity:   ████░░░░░░░░ Simple
```

---

### 2.2 — At Least Once (Never Lose, May Duplicate)

```
PRODUCER SIDE:
  acks=all
  retries=MAX (retry on failure)
  → If ACK lost in network, producer retries → broker may write twice

CONSUMER SIDE:
  Commit offset AFTER processing
  If crash after processing but before commit → message reprocessed

Flow:
  Consumer polls → processes message → commits offset
                        ↑
              If crash here → on restart:
              offset is old → message reprocessed → DUPLICATE
```

**When to use:**
- Most Kafka use cases (with idempotent consumers)
- Downstream systems that handle duplicates naturally (idempotent DBs, dedup checks)
- Log aggregation where occasional duplicate is acceptable

```
Throughput:   ████████░░░░ High
Durability:   ████████████ Highest
Complexity:   ████████░░░░ Medium
```

> **This is the most common production pattern.** Make your consumers idempotent (check if already processed) and you get effectively exactly-once behavior at the application level.

---

### 2.3 — Exactly Once (Never Lose, Never Duplicate)

True exactly-once requires coordination at **both the producer AND consumer level**:

**Producer side: Idempotent Producer**
```
enable.idempotence=true
  → PID (Producer ID) + sequence number per partition
  → Broker deduplicates retries from same producer session
  → Exactly once within a single producer session
```

**Cross-partition/cross-topic: Kafka Transactions**
```
Transactions allow:
  - Atomic writes to MULTIPLE partitions/topics
  - Atomic read-process-write (consume + produce atomically)
  - Consumer sees ONLY committed transaction data (isolation.level=read_committed)
```

**Exactly-Once Flow:**
```
transactional.id=payment-processor-1

1. beginTransaction()
2. Read from input topic (source)
3. Process the message
4. Write result to output topic
5. Commit source topic offset (as part of transaction)
6. commitTransaction()

If crash at any step:
  - Transaction is aborted
  - Output topic write is invisible (consumer isolation)
  - Source offset is NOT committed
  - On restart: re-read from same offset, re-process, re-write
  - No duplicates in output because transaction was atomic
```

```
Throughput:   ████░░░░░░░░ Lower (30–40% overhead)
Durability:   ████████████ Highest
Complexity:   ████████████ High
```

---

## ⚙️ 3. How It Works Internally

### Idempotent Producer Internals

```
Producer Registration:
  On first connection, broker assigns:
    PID = 42 (Producer ID — unique per producer instance)

Per-Partition Sequence Numbers:
  Producer maintains a sequence counter per partition:
    payments-P0: seq=0
    payments-P1: seq=0

Message flow:
  Send msg to payments-P0 → {PID=42, partition=0, seq=0, data=...}
  Broker writes it, records: {PID=42, P0, seq=0} = written

  Network timeout → Producer retries: {PID=42, partition=0, seq=0, data=...}
  Broker checks: "I already have (PID=42, P0, seq=0)" → IGNORE (deduplicate)
  Producer eventually gets ACK → success, no duplicate
```

### Kafka Transactions Internals

```
Transaction Components:
  Transaction Coordinator (TC): A broker acting as the coordinator
                                 Selected via: hash(transactional.id) % numBrokers
  Transaction Log: __transaction_state (internal topic)
  Transaction Marker: special record added to each partition to mark commit/abort

Transaction Lifecycle:

  1. initTransactions()
     → Producer registers with TC, gets epoch (fencing old zombie producers)

  2. beginTransaction()
     → Local flag on producer — no broker call yet

  3. send() to topic A  +  send() to topic B
     → TC is notified: "these partitions are part of this transaction"

  4. sendOffsetsToTransaction(offsets, groupId)
     → Atomically commit consumer offsets AS PART of the transaction

  5. commitTransaction()
     → TC writes COMMIT to __transaction_state
     → TC sends COMMIT marker to all involved partitions
     → Consumers with isolation.level=read_committed now see these messages

  OR abortTransaction()
     → TC sends ABORT marker
     → Messages written are invisible to read_committed consumers (rolled back)
```

### Consumer Isolation Levels

```
isolation.level=read_uncommitted (default):
  Consumer reads ALL messages, including those in open/aborted transactions
  → May see messages that are later aborted (phantom reads)

isolation.level=read_committed:
  Consumer reads ONLY messages from committed transactions
  + Non-transactional messages (regular produces)
  → Never sees aborted transaction data
  → Higher latency: must wait for transaction to commit before reading
```

---

## 💻 4. Code Examples (Java + Spring Boot)

### At Most Once — application.yml

```yaml
spring:
  kafka:
    producer:
      acks: "0"                 # No acknowledgement
      retries: 0               # No retry
    consumer:
      enable-auto-commit: true  # Commit before processing
      auto-commit-interval: 100 # Commit frequently
```

---

### At Least Once — application.yml + Listener

```yaml
spring:
  kafka:
    producer:
      acks: all
      retries: 2147483647
      properties:
        enable.idempotence: true
    consumer:
      enable-auto-commit: false   # Manual commit AFTER processing
      properties:
        isolation.level: read_committed
```

```java
package com.example.kafka.consumer;

import com.example.kafka.model.OrderEvent;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Service;

@Service
public class AtLeastOnceConsumer {

    @KafkaListener(topics = "order-events", groupId = "at-least-once-group")
    public void consume(
            ConsumerRecord<String, OrderEvent> record,
            Acknowledgment ack) {

        try {
            // PROCESS FIRST
            processOrder(record.value());

            // THEN COMMIT (at-least-once: crash here = reprocessed)
            ack.acknowledge();

        } catch (Exception e) {
            // Don't ACK on failure → message will be redelivered
            System.err.println("Processing failed, will retry: " + e.getMessage());
        }
    }

    private void processOrder(OrderEvent event) {
        // Idempotent logic:
        // Check if order already processed before doing work
        // e.g.: if (orderRepo.existsById(event.getOrderId())) return;
        System.out.println("Processing: " + event.getOrderId());
    }
}
```

---

### Exactly Once — Transactional Producer Config

```java
package com.example.kafka.config;

import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.core.*;
import org.springframework.kafka.support.serializer.JsonSerializer;
import org.springframework.kafka.transaction.KafkaTransactionManager;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class ExactlyOnceConfig {

    @Bean
    public ProducerFactory<String, Object> transactionalProducerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);

        // The transactional.id makes this a transactional producer
        // Must be unique per producer instance
        config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG,
            "order-processor-" + System.getenv("HOSTNAME"));

        DefaultKafkaProducerFactory<String, Object> factory =
            new DefaultKafkaProducerFactory<>(config);

        // IMPORTANT: mark factory as transactional
        factory.setTransactionIdPrefix("order-processor-");
        return factory;
    }

    @Bean
    public KafkaTemplate<String, Object> transactionalKafkaTemplate() {
        return new KafkaTemplate<>(transactionalProducerFactory());
    }

    @Bean
    public KafkaTransactionManager<String, Object> kafkaTransactionManager() {
        return new KafkaTransactionManager<>(transactionalProducerFactory());
    }
}
```

---

### Exactly Once — Read-Process-Write (Consume → Transform → Produce)

```java
package com.example.kafka.service;

import com.example.kafka.model.OrderEvent;
import com.example.kafka.model.PaymentCommand;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class ExactlyOnceOrderProcessor {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public ExactlyOnceOrderProcessor(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // @Transactional wraps the entire consume + produce in one Kafka transaction
    // If anything fails: input offset NOT committed, output NOT visible → safe retry
    @Transactional("kafkaTransactionManager")
    @KafkaListener(
        topics = "order-events",
        groupId = "exactly-once-processor",
        properties = {"isolation.level=read_committed"}
    )
    public void processAndForward(
            ConsumerRecord<String, OrderEvent> record) {

        OrderEvent order = record.value();
        System.out.printf("Processing order %s in transaction%n",
            order.getOrderId());

        // Transform: order event → payment command
        PaymentCommand payment = new PaymentCommand(
            order.getOrderId(),
            order.getUserId(),
            order.getAmount()
        );

        // Write to output topic (SAME transaction as offset commit)
        kafkaTemplate.send("payment-commands", order.getOrderId(), payment);

        // Offset committed as part of the transaction
        // If send() fails → transaction aborts → offset NOT committed
        // → Message reprocessed, but payment-commands topic stays clean
    }
}
```

---

### Exactly Once — Manual Transaction Control

```java
package com.example.kafka.service;

import com.example.kafka.model.OrderEvent;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class ManualTransactionService {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public ManualTransactionService(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void processWithManualTransaction(OrderEvent event) {
        // executeInTransaction gives you explicit control
        kafkaTemplate.executeInTransaction(operations -> {
            try {
                // Step 1: Validate business logic
                if (event.getAmount() <= 0) {
                    throw new IllegalArgumentException("Invalid amount");
                }

                // Step 2: Send to multiple topics atomically
                operations.send("payment-commands",
                    event.getOrderId(),
                    buildPaymentCommand(event));

                operations.send("audit-events",
                    event.getOrderId(),
                    buildAuditEvent(event));

                // Both sends above are ATOMIC
                // Either both committed or both aborted
                System.out.println("Transaction committed for: "
                    + event.getOrderId());
                return true;

            } catch (Exception e) {
                System.err.println("Transaction aborted: " + e.getMessage());
                throw e; // Spring Kafka will abort the transaction
            }
        });
    }

    private Object buildPaymentCommand(OrderEvent e) {
        return new Object(); // replace with actual PaymentCommand
    }

    private Object buildAuditEvent(OrderEvent e) {
        return new Object(); // replace with actual AuditEvent
    }
}
```

---

### Idempotent Consumer (Application-Level Dedup)

```java
package com.example.kafka.consumer;

import com.example.kafka.model.OrderEvent;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.stereotype.Service;

import java.time.Duration;

@Service
public class IdempotentOrderConsumer {

    // Using Redis as deduplication store
    private final RedisTemplate<String, String> redisTemplate;

    public IdempotentOrderConsumer(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    @KafkaListener(topics = "order-events", groupId = "idempotent-consumer")
    public void consume(
            ConsumerRecord<String, OrderEvent> record,
            Acknowledgment ack) {

        String dedupeKey = String.format("kafka:processed:%s:%d:%d",
            record.topic(), record.partition(), record.offset());

        // Check if already processed (deduplication)
        Boolean alreadyProcessed = redisTemplate.hasKey(dedupeKey);

        if (Boolean.TRUE.equals(alreadyProcessed)) {
            System.out.println("⚠️ Duplicate detected, skipping: " + dedupeKey);
            ack.acknowledge();  // Commit offset, don't reprocess
            return;
        }

        try {
            // Process the message
            processOrder(record.value());

            // Mark as processed in Redis (TTL = 7 days to match Kafka retention)
            redisTemplate.opsForValue().set(dedupeKey, "done",
                Duration.ofDays(7));

            ack.acknowledge();

        } catch (Exception e) {
            System.err.println("Processing failed: " + e.getMessage());
            // Don't ACK → will be redelivered → dedup key not set → will retry
        }
    }

    private void processOrder(OrderEvent event) {
        System.out.println("Processed: " + event.getOrderId());
    }
}
```

---

## ⚡ 5. Configuration Summary

### At Most Once

```yaml
producer:
  acks: "0"
  retries: 0
consumer:
  enable-auto-commit: true
  # Commit before processing
```

### At Least Once (Recommended)

```yaml
producer:
  acks: all
  retries: 2147483647
  properties:
    enable.idempotence: true
consumer:
  enable-auto-commit: false
  # Manual commit AFTER processing
  # Make consumer idempotent at application level
```

### Exactly Once (Kafka Transactions)

```yaml
producer:
  acks: all
  retries: 2147483647
  properties:
    enable.idempotence: true
    transactional.id: my-app-${HOSTNAME}
consumer:
  enable-auto-commit: false
  properties:
    isolation.level: read_committed
```

---

## 🔄 6. Decision Framework

```
                    START
                      │
          ┌───────────▼───────────┐
          │ Can you tolerate      │
          │ message loss?         │
          └───────────┬───────────┘
                      │
            ┌─────────┴──────────┐
            YES                  NO
            │                    │
     ┌──────▼───────┐    ┌───────▼──────┐
     │ AT MOST ONCE │    │ Can you       │
     │ acks=0,      │    │ tolerate      │
     │ auto-commit  │    │ duplicates?   │
     │ before proc  │    └───────┬───────┘
     └──────────────┘            │
           (Metrics,    ┌────────┴────────┐
            Telemetry)  YES              NO
                        │                │
               ┌────────▼────────┐  ┌────▼──────────────┐
               │ AT LEAST ONCE   │  │ EXACTLY ONCE       │
               │ acks=all        │  │ Transactions       │
               │ idempotence=true│  │ read_committed     │
               │ manual commit   │  │ transactional.id   │
               │ after processing│  └───────────────────┘
               └─────────────────┘  (Payments, Accounting,
               + idempotent consumer  Inventory dedup)
               (Most production
                use cases)
```

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| `acks=1` + `retries>0` | Duplicate messages on leader failure + retry | Use `acks=all` + `enable.idempotence=true` |
| Commit before processing | Messages lost on consumer crash | Always commit AFTER successful processing |
| Transactions without `read_committed` | Consumers see uncommitted/aborted data | Set `isolation.level=read_committed` on all transactional consumers |
| Same `transactional.id` on multiple instances | Kafka fences the old producer (zombie fencing) — intentional but confusing | Append unique suffix like hostname |
| Transactions for high-throughput | 20–40% throughput reduction | Use idempotence + at-least-once + idempotent consumer instead |
| Missing idempotent consumer logic | At-least-once becomes effectively unpredictable | Always check for duplicates at application level |
| `isolation.level=read_committed` without transactions | Consumers stuck waiting for transaction markers | Only use `read_committed` when producers are transactional |

---

## 🎯 8. Interview Questions

**Q1. What are the three message delivery semantics in Kafka?**
> **At most once**: messages may be lost, never duplicated (commit before processing, acks=0). **At least once**: messages never lost, may duplicate (commit after processing, acks=all + retries). **Exactly once**: messages never lost, never duplicated — requires idempotent producers + Kafka transactions.

**Q2. How does `enable.idempotence=true` achieve exactly-once at the producer level?**
> Kafka assigns a unique Producer ID (PID) and sequence number to each message per partition. If the producer retries due to a timeout, the broker detects the same (PID, partition, sequence) and discards the duplicate. This guarantees no duplicates **within a single producer session** — but not across restarts or multiple producers.

**Q3. What is `isolation.level=read_committed` and why does it matter?**
> Without it (`read_uncommitted`, the default), consumers can read messages that are part of an open or later-aborted transaction — these messages will never be committed and shouldn't be processed. `read_committed` ensures consumers only see messages from successfully committed transactions, which is essential for exactly-once consume-transform-produce pipelines.

**Q4. What is "zombie fencing" in Kafka transactions?**
> If a transactional producer (e.g., a crashed pod) restarts with the same `transactional.id`, Kafka uses an epoch counter to fence out the old (zombie) instance. The old producer's ongoing transactions are aborted, and the new instance takes ownership. This prevents two producers with the same transactional.id from producing simultaneously.

**Q5. When should you use Kafka transactions vs. idempotent consumers?**
> Use **Kafka transactions** when you need atomic writes across multiple topics/partitions, or atomic consume-transform-produce pipelines (read from topic A, write to topic B, commit both atomically). Use **idempotent consumers** (at-least-once + dedup logic) when the overhead of transactions is too high — it's simpler, faster, and achieves the same result for most use cases.

**Q6. What happens if a transaction is aborted mid-way?**
> Kafka writes an ABORT marker to all partitions involved in the transaction. Consumers with `isolation.level=read_committed` skip all messages associated with the aborted transaction. The consumer offsets for the source topic are not committed, so the input message will be reprocessed in the next attempt.

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "Double-Charged Customers"
> **Problem:** Your payment service consumes from `order-events`, deducts from customer balance, and produces to `payment-confirmed`. During a Kafka rebalance, some messages were processed but offset commit failed. Customers got charged twice. How do you fix this permanently?

**Answer:**
```
Root cause: At-least-once processing without idempotent consumer
  Process → charge customer ✅
  Offset commit fails
  Rebalance → another consumer picks up same partition
  Reprocesses same message → charges customer AGAIN ❌

Solution 1: Kafka Transactions (Exactly Once)
  @Transactional("kafkaTransactionManager")
  public void process(ConsumerRecord<...> record) {
      chargeCustomer(record.value());
      kafkaTemplate.send("payment-confirmed", ...);
      // offset committed atomically with the send
  }
  → If chargeCustomer fails: transaction aborts, offset not committed → retry
  → If send fails: transaction aborts, offset not committed → retry
  → Only way to commit offset: entire transaction succeeds

Solution 2: Idempotent Consumer (Simpler, Recommended)
  public void process(ConsumerRecord<...> record) {
      String paymentId = record.value().getOrderId();

      // Check if payment already made (in your DB)
      if (paymentRepo.existsByOrderId(paymentId)) {
          ack.acknowledge();  // Already done, skip
          return;
      }

      // Use DB transaction: insert + charge in one atomic operation
      paymentRepo.chargeWithIdempotencyCheck(paymentId, amount);
      ack.acknowledge();
  }
  → DB uniqueness constraint prevents double charge even on replay
```

---

### Scenario 2: "Inventory Oversell"
> **Problem:** Order service publishes to `order-created`. Inventory service consumes, checks stock, and reserves items. With at-least-once delivery, duplicate messages cause the same item to be reserved twice, leading to oversell. Design a solution.

**Answer:**
```
Option A: Idempotent reservation with DB unique constraint
  Table: reservations (order_id PRIMARY KEY, item_id, quantity)

  @KafkaListener(...)
  public void reserve(ConsumerRecord<...> record) {
      try {
          // INSERT IGNORE or ON CONFLICT DO NOTHING
          inventoryRepo.insertReservation(
              record.value().getOrderId(),
              record.value().getItemId(),
              record.value().getQuantity()
          );
          ack.acknowledge();
      } catch (DuplicateKeyException e) {
          // Already reserved — idempotent, just ACK
          ack.acknowledge();
      }
  }

  DB unique constraint on order_id guarantees:
  Even if same message replayed 100 times → only 1 reservation inserted

Option B: Redis-based distributed dedup
  Before any DB operation:
    key = "reserved:order:" + orderId
    if (redis.setIfAbsent(key, "1", Duration.ofDays(1))) {
        // First time → do the reservation
    } else {
        // Already done → skip
    }
```

---

### Scenario 3: "Exactly Once Across DB and Kafka"
> **Problem:** You need to save an order to Postgres AND publish to Kafka. Both must happen atomically — no partial success. How?

**Answer:**
```
The Dual-Write Problem:
  Option A: Save DB first, then Kafka
    DB succeeds, Kafka fails → event never published → downstream starved

  Option B: Kafka first, then DB
    Kafka succeeds, DB fails → event published but order not saved → inconsistency

Pure Kafka transactions don't help here because Postgres is not part of Kafka's
transaction coordinator.

REAL SOLUTION: Outbox Pattern (covered in detail in Topic 33)

  1. Save order + outbox record in SAME DB transaction:
     BEGIN;
       INSERT INTO orders (id, ...) VALUES (...);
       INSERT INTO outbox (id, topic, key, payload) VALUES (...);
     COMMIT;

  2. Separate Outbox Poller / CDC reads outbox table
     Publishes to Kafka exactly once (using idempotent producer)
     Marks outbox record as published

  This guarantees:
  - DB write and Kafka publish are eventually consistent
  - No message ever lost (DB transaction is the source of truth)
  - No duplicate (outbox row = exactly one Kafka publish with dedup)
```

---

## 📝 10. Quick Revision Summary

```
✅ AT MOST ONCE:
   acks=0, retries=0, commit BEFORE processing
   → Loss possible, no duplicates
   → Use for: metrics, non-critical logs

✅ AT LEAST ONCE:
   acks=all, retries=MAX, enable.idempotence=true, commit AFTER processing
   → No loss, duplicates possible
   → Use for: most production systems (+ idempotent consumer)
   → This is the recommended default

✅ EXACTLY ONCE:
   Idempotent producer + Kafka Transactions + isolation.level=read_committed
   → No loss, no duplicates
   → Use for: payments, financial ledgers, inventory
   → 20–40% throughput overhead

✅ Idempotent Producer:
   PID + sequence numbers → broker deduplicates retries
   Scope: within one producer session only

✅ Kafka Transactions:
   transactional.id → atomic multi-partition writes
   beginTransaction → send → commitTransaction (or abortTransaction)
   Zombie fencing via epoch counter

✅ Isolation levels:
   read_uncommitted → sees all messages (default)
   read_committed   → only sees committed transaction data

✅ Practical recommendation:
   AT LEAST ONCE + IDEMPOTENT CONSUMER = 95% of production needs
   EXACTLY ONCE (transactions) = payment systems, financial ledgers

✅ Outbox Pattern = the real solution for DB + Kafka atomicity
   (saves DB row + outbox record in one DB tx, then CDC publishes to Kafka)
```

---

**← Previous:** [07 — Consumer Groups & Partition Assignment Strategies](./07-consumer-groups.md)  
**Next Topic →** [09 — Kafka Broker Internals: Request Handling & Log Architecture](./09-broker-internals.md)
