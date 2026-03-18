# 12 — Log Storage, Segments, Retention & Compaction
> **Phase 3 — Kafka Internals & Storage**  
> 📁 `notes/12-log-storage.md`

---

## 📖 1. Concept Explanation

### Why Log Storage Design Matters

Kafka's storage model is fundamentally different from a traditional message broker. Messages are **not deleted after consumption** — they persist on disk according to a configurable policy. Understanding this lets you:

- Design retention policies that balance cost vs. replayability
- Choose between **deletion** and **compaction** for the right use cases
- Avoid silent disk-full incidents in production
- Implement event sourcing and CDC patterns correctly

---

### Two Retention Policies

```
Policy 1: DELETE (default)
  Messages are deleted when they get "old enough" or the partition gets "big enough"
  → Good for: event streams, logs, activity tracking
  → Consumer can replay history up to retention window

Policy 2: COMPACT
  Only the LATEST message per key is kept forever
  Older messages with the same key are deleted during compaction
  → Good for: current state / changelog topics (like DB snapshots)
  → Consumer can always rebuild current state by reading from beginning
  → Used heavily by: Kafka Streams (KTable), Schema Registry, Connect offsets
```

---

### Log Segments — Physical Storage Unit

```
Every partition is stored as a series of SEGMENT FILES on disk:

/kafka-logs/orders-0/                   ← orders topic, partition 0
  ├── 00000000000000000000.log           ← Segment 1 (base offset = 0)
  ├── 00000000000000000000.index         ← Offset index for segment 1
  ├── 00000000000000000000.timeindex     ← Timestamp index for segment 1
  ├── 00000000000000001000.log           ← Segment 2 (base offset = 1000)
  ├── 00000000000000001000.index
  ├── 00000000000000001000.timeindex
  ├── 00000000000000002000.log           ← Active segment (writes go here)
  ├── 00000000000000002000.index
  ├── 00000000000000002000.timeindex
  └── leader-epoch-checkpoint            ← Leader epoch history

Key facts:
  Segment filename = base offset (first offset in that segment)
  Only the LAST (active) segment is written to
  All other segments are SEALED (immutable, read-only)
  Retention/Compaction operates on SEALED segments only
```

---

## 🏗️ 2. Architecture / Flow

### Segment Lifecycle

```
Birth: Active Segment Created
  When a partition is first created, segment 00000...0000.log is created
  All writes go to this segment

Roll: Active Segment → Sealed Segment
  Triggered when ANY of these is true:
    ① log.segment.bytes exceeded (default 1GB)
    ② log.roll.ms exceeded (default 7 days)
    ③ log.index.size.max.bytes exceeded (index file too large)
  Old segment is closed and becomes sealed (immutable)
  New active segment created with next offset as filename

Deletion (DELETE policy):
  Sealed segments eligible for deletion when:
    ① log.retention.ms exceeded (age of last message in segment > retention)
    ② log.retention.bytes exceeded (total partition size > limit)
  Log Cleaner thread deletes the entire segment file

Compaction (COMPACT policy):
  Log Cleaner thread rewrites segments, keeping only latest value per key
  Detailed in Section 3 below
```

---

### DELETE Retention — How It Works

```
Example: log.retention.ms = 604800000 (7 days)

Timeline:

Day 1:  Segment A (offsets 0–999)     ← created, filled, sealed
Day 3:  Segment B (offsets 1000–1999) ← sealed
Day 5:  Segment C (offsets 2000–2999) ← sealed
Day 7:  Segment D (offsets 3000–3999) ← sealed
Day 8:  Segment E (offsets 4000+)     ← ACTIVE

Day 8: Retention check runs (log.retention.check.interval.ms = 5 min):
  Segment A: last message timestamp = Day 1 → age = 7 days → DELETE ✅
  Segment B: last message timestamp = Day 3 → age = 5 days → keep
  Segment C: age = 3 days → keep
  ...

Result: Segment A deleted. Consumers that haven't read segment A will get
        "OffsetOutOfRangeException" if they try to fetch offset 0.
```

---

### Compaction — How It Works

```
Topic: user-profiles (cleanup.policy=compact)

Initial state — many updates per user over time:

Segment 1 (old):
  [key=user1, val={"name":"Alice","age":25}]  ← offset 0
  [key=user2, val={"name":"Bob","age":30}]    ← offset 1
  [key=user1, val={"name":"Alice","age":26}]  ← offset 2 (updated age)
  [key=user3, val={"name":"Charlie"}]         ← offset 3

Segment 2 (newer):
  [key=user2, val={"name":"Bob","age":31}]    ← offset 4
  [key=user1, val={"name":"Alice","age":27}]  ← offset 5

After compaction:
  The Log Cleaner keeps only the LATEST offset per key:

  user1 → latest is offset 5 → keep offset 5, DELETE offsets 0 and 2
  user2 → latest is offset 4 → keep offset 4, DELETE offset 1
  user3 → latest is offset 3 → keep (only version)

Compacted result:
  [key=user3, val={"name":"Charlie"}]         ← offset 3
  [key=user2, val={"name":"Bob","age":31}]    ← offset 4
  [key=user1, val={"name":"Alice","age":27}]  ← offset 5

A NEW consumer starting from offset 0 will read:
  → user3's profile (only version)
  → user2's latest profile
  → user1's latest profile
This gives a COMPLETE SNAPSHOT of all current user states!
```

---

### Tombstone Records (Deletion in Compacted Topics)

```
To DELETE a key from a compacted topic, produce a NULL value:

Producer sends:
  key = "user1", value = null   ← this is a TOMBSTONE

Effect:
  Compaction keeps the tombstone temporarily
  After delete.retention.ms (default 24h), tombstone itself is deleted
  user1 is completely removed from the compacted topic

Use case: GDPR "right to be forgotten"
  User requests data deletion
  Send tombstone for user_id
  After compaction: user's data erased from Kafka permanently
```

---

### Combined Policy: DELETE + COMPACT

```
cleanup.policy=compact,delete

Both policies apply:
  COMPACT: keep only latest value per key
  DELETE: also apply time/size-based retention

Use case: User activity stream where you want:
  - Latest state per user (compaction)
  - But don't need data older than 30 days (deletion)

Effectively: "Keep the latest version of each key, but only for 30 days"
```

---

## ⚙️ 3. Log Compaction Internals

### The Log Cleaner Thread

```
Log Cleaner is a background thread pool running on each broker:

1. Identifies which partitions need compaction
   (based on dirty ratio: uncompacted / total size)

2. Builds an offset map (key → latest offset):
   Scans log to find the latest offset for every key
   This map is built in memory (log.cleaner.dedupe.buffer.size)

3. Rewrites segments:
   For each sealed segment:
     For each record:
       If record.offset == offsetMap[record.key] → KEEP
       Else → SKIP (it's been superseded)
   Creates new cleaned segment files

4. Swaps files atomically:
   Old segments deleted, cleaned segments replace them

5. Compaction never touches the ACTIVE segment
   (the one currently being written to)
```

### Key Compaction Concepts

```
Log Head: The active segment (always being written)
           Contains all messages including duplicates
           Never compacted

Log Tail: All sealed segments (the "compacted" part)
          Contains only the latest message per key
          No offset is missing from the tail's perspective

                  Log Tail (compacted)         Log Head
                  ┌──────────────────────┐    ┌──────────────────────┐
Offset:           │ 3  │ 4  │ 5  │ 12 │ │    │ 20 │ 21 │ 22 │ 23 │  │
                  │usr3│usr2│usr1│usr4│ │    │usr1│usr5│usr2│usr3│  │
                  └──────────────────────┘    └──────────────────────┘
                  ← Compacted (latest per key)  ← New writes coming in

Dirty Ratio = uncompacted bytes / total bytes
  When dirty ratio > min.cleanable.dirty.ratio → compaction triggered
```

---

## 💻 4. Code Examples (Java + Spring Boot)

### Topic Creation with Retention Policies

```java
package com.example.kafka.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class StoragePolicyTopicConfig {

    // Standard event stream — DELETE after 7 days
    @Bean
    public NewTopic orderEventsTopic() {
        return TopicBuilder.name("order-events")
                .partitions(12)
                .replicas(3)
                .config("cleanup.policy", "delete")
                .config("retention.ms", "604800000")        // 7 days
                .config("retention.bytes", "107374182400")  // 100GB per partition
                .config("segment.bytes", "536870912")       // 512MB segments
                .build();
    }

    // Current state topic — COMPACT forever
    @Bean
    public NewTopic userProfilesTopic() {
        return TopicBuilder.name("user-profiles")
                .partitions(12)
                .replicas(3)
                .config("cleanup.policy", "compact")
                .config("min.cleanable.dirty.ratio", "0.1")   // Compact aggressively
                .config("segment.ms", "86400000")             // 1 day segments
                .config("delete.retention.ms", "86400000")    // Keep tombstones 1 day
                .build();
    }

    // Combined: compact + time retention (state for last 30 days)
    @Bean
    public NewTopic recentStateTopic() {
        return TopicBuilder.name("account-state")
                .partitions(6)
                .replicas(3)
                .config("cleanup.policy", "compact,delete")
                .config("retention.ms", "2592000000")  // 30 days
                .config("min.cleanable.dirty.ratio", "0.5")
                .build();
    }

    // High-volume metrics — short retention, large segments
    @Bean
    public NewTopic metricsTopic() {
        return TopicBuilder.name("system-metrics")
                .partitions(24)
                .replicas(2)
                .config("cleanup.policy", "delete")
                .config("retention.ms", "3600000")    // 1 hour
                .config("segment.bytes", "134217728") // 128MB segments (roll fast)
                .config("compression.type", "lz4")
                .build();
    }
}
```

---

### Producer for Compacted Topic — State Updates

```java
package com.example.kafka.producer;

import com.example.kafka.model.UserProfile;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

@Service
public class UserProfileProducer {

    private static final String TOPIC = "user-profiles";
    private final KafkaTemplate<String, UserProfile> kafkaTemplate;

    public UserProfileProducer(KafkaTemplate<String, UserProfile> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    // Upsert: key = userId ensures compaction keeps only latest profile
    public void upsertProfile(String userId, UserProfile profile) {
        kafkaTemplate.send(TOPIC, userId, profile)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    System.out.printf("Profile upserted: userId=%s partition=%d%n",
                        userId, result.getRecordMetadata().partition());
                }
            });
    }

    // Tombstone: delete user's profile (GDPR erasure)
    public void deleteProfile(String userId) {
        // null value = tombstone record
        // After delete.retention.ms, this key disappears from compacted log
        kafkaTemplate.send(TOPIC, userId, null)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    System.out.printf("Tombstone sent for userId=%s%n", userId);
                }
            });
    }
}
```

---

### Consumer Rebuilding State from Compacted Topic

```java
package com.example.kafka.consumer;

import com.example.kafka.model.UserProfile;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

@Service
public class UserProfileStateBuilder {

    // In-memory state rebuilt from compacted topic
    private final Map<String, UserProfile> profileCache = new ConcurrentHashMap<>();

    // Read from beginning to rebuild full current state
    @KafkaListener(
        topics = "user-profiles",
        groupId = "profile-cache-builder",
        properties = {"auto.offset.reset=earliest"}
    )
    public void buildState(ConsumerRecord<String, UserProfile> record) {

        String userId = record.key();

        if (record.value() == null) {
            // Tombstone — delete from state
            profileCache.remove(userId);
            System.out.printf("Deleted profile for userId=%s%n", userId);
        } else {
            // Upsert — put latest profile
            profileCache.put(userId, record.value());
            System.out.printf("Loaded profile for userId=%s%n", userId);
        }
    }

    public UserProfile getProfile(String userId) {
        return profileCache.get(userId);
    }

    public Map<String, UserProfile> getAllProfiles() {
        return Map.copyOf(profileCache);
    }
}
```

---

### Dynamic Retention Management via AdminClient

```java
package com.example.kafka.admin;

import org.apache.kafka.clients.admin.*;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;

@Service
public class RetentionManager {

    private final KafkaAdmin kafkaAdmin;

    public RetentionManager(KafkaAdmin kafkaAdmin) {
        this.kafkaAdmin = kafkaAdmin;
    }

    // Temporarily reduce retention (e.g., disk space emergency)
    public void reduceRetention(String topic, long retentionMs) throws Exception {
        try (AdminClient client =
                AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            ConfigResource resource =
                new ConfigResource(ConfigResource.Type.TOPIC, topic);

            AlterConfigOp op = new AlterConfigOp(
                new ConfigEntry("retention.ms", String.valueOf(retentionMs)),
                AlterConfigOp.OpType.SET
            );

            client.incrementalAlterConfigs(Map.of(resource, List.of(op)))
                  .all().get();

            System.out.printf("Updated %s retention to %dms%n", topic, retentionMs);
        }
    }

    // Query current topic storage configuration
    public void describeTopicStorage(String topic) throws Exception {
        try (AdminClient client =
                AdminClient.create(kafkaAdmin.getConfigurationProperties())) {

            ConfigResource resource =
                new ConfigResource(ConfigResource.Type.TOPIC, topic);

            Config config = client.describeConfigs(List.of(resource))
                                  .all().get().get(resource);

            List<String> storageKeys = List.of(
                "cleanup.policy", "retention.ms", "retention.bytes",
                "segment.bytes", "segment.ms", "compression.type",
                "min.cleanable.dirty.ratio", "delete.retention.ms"
            );

            System.out.println("Storage config for: " + topic);
            storageKeys.forEach(key -> {
                ConfigEntry entry = config.get(key);
                if (entry != null) {
                    System.out.printf("  %-35s = %s%s%n",
                        key, entry.value(),
                        entry.isDefault() ? " (default)" : "");
                }
            });
        }
    }
}
```

---

## ⚡ 5. Key Storage Configurations

### Retention Configs

| Config | Default | Scope | Explanation |
|--------|---------|-------|-------------|
| `cleanup.policy` | `delete` | Topic | `delete`, `compact`, or `compact,delete` |
| `retention.ms` | `604800000` (7d) | Topic | Max age of a message |
| `retention.bytes` | `-1` (unlimited) | Topic | Max size per partition |
| `log.retention.check.interval.ms` | `300000` (5min) | Broker | How often retention check runs |

### Segment Configs

| Config | Default | Explanation |
|--------|---------|-------------|
| `segment.bytes` | `1073741824` (1GB) | Roll segment when it reaches this size |
| `segment.ms` | `604800000` (7d) | Roll segment when it reaches this age |
| `log.index.size.max.bytes` | `10485760` (10MB) | Max index file size before roll |
| `log.index.interval.bytes` | `4096` | Add index entry every N bytes |

### Compaction Configs

| Config | Default | Explanation |
|--------|---------|-------------|
| `min.cleanable.dirty.ratio` | `0.5` | Compact when 50% of log is "dirty" |
| `delete.retention.ms` | `86400000` (1d) | How long tombstones are retained |
| `log.cleaner.threads` | `1` | Background cleaner thread count |
| `log.cleaner.dedupe.buffer.size` | `134217728` (128MB) | Memory for key→offset map during compaction |
| `min.compaction.lag.ms` | `0` | Min time before a message is eligible for compaction |
| `max.compaction.lag.ms` | `Long.MAX_VALUE` | Max time before compaction is forced |

---

## 🔄 6. Real-World Scenarios

### Scenario: Event Sourcing with Compacted Topic

```
Use case: Customer account balance (event sourcing)

Topic: account-balances (compact)
  Each message: key=accountId, value=currentBalance

Events:
  account-001 → 1000.00  (initial deposit)
  account-001 → 1500.00  (after transfer in)
  account-001 → 1350.00  (after withdrawal)
  account-002 → 500.00

After compaction:
  account-001 → 1350.00  (only latest)
  account-002 → 500.00

New service starting up:
  Reads from beginning → reconstructs current balance for all accounts
  No need for a database snapshot → Kafka IS the source of truth

Combine with a transaction log topic (delete policy):
  account-transactions (delete, 90 day retention) → full history
  account-balances (compact) → current state
  Best of both worlds!
```

---

### Scenario: GDPR Compliance with Compacted Topics

```
Requirement: Delete all user data within 30 days of request

With compacted topics:
  1. User requests deletion
  2. Producer sends tombstone: key=userId, value=null
  3. Compaction runs: tombstone replaces all previous records for userId
  4. After delete.retention.ms (e.g., 7 days): tombstone itself deleted
  5. userId completely erased from Kafka log

With delete topics:
  1. Separate user-data-deletions topic (user IDs to delete)
  2. All downstream systems subscribe and purge their stores
  3. Kafka data expires naturally via retention.ms

Note: Kafka encryption + key rotation (per-field encryption) is the
      most rigorous GDPR approach for sensitive data in Kafka.
```

---

## ⚠️ 7. Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| `retention.bytes=-1` on high-volume topic | Disk fills up silently | Set `retention.bytes` per partition based on disk capacity |
| Small `segment.bytes` on large topic | Thousands of small files → high inode usage | Keep segments 256MB–1GB depending on retention |
| Using `compact` without keys | ALL messages have null key → compaction keeps only 1 message! | Compacted topics MUST use meaningful keys |
| Consuming tombstones without null check | NullPointerException in consumer | Always null-check record.value() on compacted topics |
| `min.cleanable.dirty.ratio=0.9` (too high) | Compaction rarely runs → partition grows unbounded | Use 0.1–0.5 for active compaction |
| Not setting `retention.ms` + `retention.bytes` | Only one limit active → can still run out of disk | Set BOTH as a safety net |
| Expecting compaction to be immediate | Log cleaner runs asynchronously → lag exists | Compaction is eventual, not real-time |

---

## 🎯 8. Interview Questions

**Q1. What is the difference between DELETE and COMPACT retention policies?**
> DELETE removes messages based on age (`retention.ms`) or size (`retention.bytes`) — entire segments are deleted when they exceed the limit. COMPACT keeps only the latest message per key, allowing older versions to be garbage collected. Use DELETE for event streams and COMPACT for current-state topics (like a database changelog).

**Q2. What is a tombstone record in Kafka?**
> A tombstone is a message with a non-null key and a null value. In a compacted topic, it signals that the key should be deleted. The tombstone temporarily exists (for `delete.retention.ms`) and then disappears along with all previous records for that key. Used for GDPR erasure and entity deletion in event-sourced systems.

**Q3. When does a log segment roll (get sealed)?**
> A segment rolls when: its size exceeds `segment.bytes` (default 1GB), its age exceeds `segment.ms` (default 7 days), or the index file exceeds `log.index.size.max.bytes`. Only sealed (non-active) segments are eligible for deletion or compaction.

**Q4. Why can't compaction run on the active segment?**
> The active segment is still being written to — its last message may not be the "final" value for a key. Compacting it prematurely could delete a value that's about to be superseded by a newer write. Compaction only runs on sealed (immutable) segments where the log head is definitively established.

**Q5. What is `min.cleanable.dirty.ratio`?**
> It's the ratio of "dirty" (uncompacted) bytes to total partition bytes that must be exceeded before the log cleaner runs compaction on a partition. A ratio of 0.5 means: compact when more than 50% of the data hasn't been compacted yet. Lower values = more aggressive (frequent) compaction; higher values = lazy compaction.

**Q6. What is `log.retention.bytes` and how does it interact with `retention.ms`?**
> `retention.bytes` limits the total size per partition; `retention.ms` limits the age of messages. Both apply simultaneously — a segment is deleted if either limit is exceeded. Setting both is a best practice: `retention.ms` handles time-based cleanup while `retention.bytes` acts as a safety net to prevent disk overflow.

---

## 🧠 9. Scenario-Based Interview Problems

### Scenario 1: "Kafka Disk Full at 3 AM"
> **Problem:** You get paged at 3 AM. Broker 2's disk is 99% full. It's the leader for 18 partitions of the `clickstream` topic. Writes are being rejected. What do you do?

**Answer:**
```
Immediate (restore writes in < 5 min):

Step 1: Emergency retention reduction
  kafka-configs.sh --bootstrap-server localhost:9092 \
    --entity-type topics --entity-name clickstream \
    --alter --add-config retention.ms=3600000  # 1 hour

  Kafka immediately starts deleting segments older than 1 hour
  Monitor: du -sh /kafka-logs/ → should start shrinking

Step 2: If still critical — reduce retention.bytes too
  --add-config retention.bytes=10737418240  # 10GB per partition

Step 3: Verify writes resume
  Watch producer error rate in monitoring → should drop to 0
  Check under-replicated partitions → should stay at 0

Within the hour (prevent recurrence):
  Identify which topics are consuming the most space:
    du -sh /kafka-logs/* | sort -rh | head -20

  Set permanent retention policy appropriate to the topic's SLA
  Set BOTH retention.ms and retention.bytes on high-volume topics

Long-term:
  Alert on disk usage > 70% (not 99%!)
  Add capacity: new broker, expand disk
  Consider tiered storage (Kafka 3.6+ feature) for cold data
```

---

### Scenario 2: "Design Storage for an E-Commerce Platform"
> **Problem:** Design the storage strategy for: (1) order events, (2) product catalog updates, (3) real-time click events. Each has different requirements.

**Answer:**
```
1. Order Events
   cleanup.policy=delete
   retention.ms=7776000000  (90 days — regulatory requirement)
   retention.bytes=53687091200  (50GB per partition safety cap)
   replication.factor=3
   min.insync.replicas=2
   Rationale: Full audit trail needed. DELETE old orders after 90 days.

2. Product Catalog Updates
   cleanup.policy=compact
   min.cleanable.dirty.ratio=0.1  (compact aggressively)
   delete.retention.ms=86400000   (1 day tombstone retention)
   replication.factor=3
   Rationale: We need CURRENT product state, not full history.
   New services can bootstrap by reading from start of compacted log.
   Tombstones handle product discontinuation (GDPR: product removal).

3. Real-time Click Events
   cleanup.policy=delete
   retention.ms=86400000    (1 day — real-time analysis window only)
   retention.bytes=5368709120  (5GB per partition — high volume)
   replication.factor=2     (some loss tolerable, it's analytics)
   compression.type=lz4     (high volume → compression critical)
   segment.bytes=268435456  (256MB — roll segments quickly for fast deletion)
   Rationale: High volume, short-lived. Only need today's clicks.
   RF=2 acceptable — minor loss in analytics is tolerable.
```

---

## 📝 10. Quick Revision Summary

```
✅ Log structure: partition = directory of segment files (.log, .index, .timeindex)
   Active segment: only one, currently being written
   Sealed segments: immutable, eligible for retention/compaction

✅ Segment rolls when:
   size > segment.bytes (1GB default)
   OR age > segment.ms (7 days default)

✅ DELETE policy (default):
   Deletes ENTIRE sealed segments when age > retention.ms
   OR partition size > retention.bytes
   Best for: event streams, logs, time-series data

✅ COMPACT policy:
   Keeps only LATEST message per key (forever)
   Old versions of same key → deleted during compaction
   Tombstone (null value) → signals key deletion
   Best for: current state, changelogs, CDC, event sourcing

✅ COMPACT+DELETE:
   Keep latest per key AND apply time/size limits
   Best for: recent state only (last 30 days of user profiles)

✅ Log Cleaner: background thread that runs compaction
   Triggered when dirty ratio > min.cleanable.dirty.ratio
   Never compacts the active segment

✅ Tombstone: key + null value
   Marks a key for deletion in compacted topics
   Kept for delete.retention.ms then permanently removed
   Used for GDPR erasure

✅ Key configs:
   cleanup.policy: delete | compact | compact,delete
   retention.ms: max message age (DELETE)
   retention.bytes: max partition size (DELETE)
   segment.bytes: when to roll a new segment
   min.cleanable.dirty.ratio: how dirty before compaction
   delete.retention.ms: how long tombstones live (COMPACT)
```

---

**← Previous:** [11 — Replication, ISR, Leader Election & Fault Tolerance](./11-replication.md)  
**Next Topic →** [13 — ZooKeeper vs KRaft: Kafka's Metadata Evolution](./13-zookeeper-kraft.md)
