# 04 — Setting Up Kafka Locally: Docker & Native Install
> **Phase 1 — Foundations**  
> 📁 `notes/04-local-setup.md`

---

## 📖 1. Concept Explanation

Before diving into advanced Kafka, you need a **local environment** to experiment, run code, and validate concepts hands-on. This note covers **three ways** to run Kafka locally:

| Method | Best For | Complexity |
|--------|----------|------------|
| **Docker Compose (KRaft)** | Quickest start, modern setup | ⭐ Easy |
| **Docker Compose (Multi-broker)** | Realistic cluster simulation | ⭐⭐ Medium |
| **Native Install** | Deep OS-level understanding | ⭐⭐ Medium |

> **Recommendation:** Use **Docker Compose with KRaft** for day-to-day learning. It mirrors production setups and has zero ZooKeeper overhead.

---

## 🏗️ 2. Setup Options — Architecture

### Single-Node KRaft (Docker)

```
Your Machine
  └── Docker
        └── kafka-broker (port 9092)
              ├── Acts as both Controller + Broker
              ├── No ZooKeeper needed
              └── Data stored in named volume
```

### Multi-Broker Cluster (Docker Compose)

```
Your Machine
  └── Docker Compose
        ├── kafka-1 (port 9092)  ← Controller + Broker
        ├── kafka-2 (port 9093)  ← Broker
        └── kafka-3 (port 9094)  ← Broker
```

### With Kafka UI (Recommended for Learning)

```
Your Machine
  └── Docker Compose
        ├── kafka-1, kafka-2, kafka-3
        └── kafka-ui (port 8080)  ← Visual dashboard
```

---

## 💻 3. Setup Method 1 — Single Node KRaft (Quickest Start)

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  kafka:
    image: apache/kafka:3.7.0
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      # KRaft settings
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller          # Combined mode
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093

      # Topic defaults
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1

      # Log retention
      KAFKA_LOG_RETENTION_HOURS: 168           # 7 days
      KAFKA_LOG_SEGMENT_BYTES: 1073741824      # 1 GB

    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  kafka-data:
```

**Start it:**
```bash
docker-compose up -d
docker-compose ps      # verify it's running
docker-compose logs -f kafka   # watch logs
```

---

## 💻 4. Setup Method 2 — Multi-Broker KRaft Cluster (Realistic)

### `docker-compose-cluster.yml`

```yaml
version: '3.8'

services:

  # ── Broker 1 (also acts as Controller) ──────────────────────────
  kafka-1:
    image: apache/kafka:3.7.0
    container_name: kafka-1
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      # All 3 nodes participate in controller quorum
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"   # Fixed cluster ID (generate once)
    volumes:
      - kafka-1-data:/var/lib/kafka/data
    networks:
      - kafka-net

  # ── Broker 2 ────────────────────────────────────────────────────
  kafka-2:
    image: apache/kafka:3.7.0
    container_name: kafka-2
    ports:
      - "9093:9092"
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9093
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    volumes:
      - kafka-2-data:/var/lib/kafka/data
    networks:
      - kafka-net

  # ── Broker 3 ────────────────────────────────────────────────────
  kafka-3:
    image: apache/kafka:3.7.0
    container_name: kafka-3
    ports:
      - "9094:9092"
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9094
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_NUM_PARTITIONS: 3
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    volumes:
      - kafka-3-data:/var/lib/kafka/data
    networks:
      - kafka-net

  # ── Kafka UI ─────────────────────────────────────────────────────
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
    environment:
      KAFKA_CLUSTERS_0_NAME: local-cluster
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-1:9092,kafka-2:9092,kafka-3:9092
    networks:
      - kafka-net

volumes:
  kafka-1-data:
  kafka-2-data:
  kafka-3-data:

networks:
  kafka-net:
    driver: bridge
```

**Start the cluster:**
```bash
docker-compose -f docker-compose-cluster.yml up -d

# Verify all brokers are up
docker-compose -f docker-compose-cluster.yml ps

# Open Kafka UI dashboard
open http://localhost:8080
```

---

## 💻 5. Setup Method 3 — Native Install (macOS / Linux)

### macOS (Homebrew)

```bash
# Install Java 17+ (required)
brew install openjdk@17

# Install Kafka
brew install kafka

# Kafka binaries land in:
# /opt/homebrew/opt/kafka/bin/

# Start Kafka (KRaft mode, single node)
brew services start kafka

# Or start manually:
/opt/homebrew/opt/kafka/bin/kafka-server-start.sh \
  /opt/homebrew/etc/kafka/server.properties
```

### Linux (Manual Install)

```bash
# Download Kafka
wget https://downloads.apache.org/kafka/3.7.0/kafka_2.13-3.7.0.tgz
tar -xzf kafka_2.13-3.7.0.tgz
cd kafka_2.13-3.7.0

# Generate a cluster ID (KRaft requires this)
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"
echo "Cluster ID: $KAFKA_CLUSTER_ID"

# Format storage directory
bin/kafka-storage.sh format \
  -t $KAFKA_CLUSTER_ID \
  -c config/kraft/server.properties

# Start Kafka
bin/kafka-server-start.sh config/kraft/server.properties
```

---

## 💻 6. Kafka CLI — Essential Commands

Once Kafka is running, use these commands to interact with it:

### Topic Management

```bash
# Create a topic
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic order-events \
  --partitions 3 \
  --replication-factor 1

# List all topics
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list

# Describe a topic (shows partitions, leaders, ISR)
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic order-events

# Delete a topic
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --delete \
  --topic order-events

# Increase partitions (can only increase, never decrease)
kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --alter \
  --topic order-events \
  --partitions 6
```

### Produce Messages (Console)

```bash
# Simple producer — type messages, press Enter to send
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events

# Producer with key (key and value separated by ':')
kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --property "key.separator=:" \
  --property "parse.key=true"

# Input format: key:value
# order-1:{"status":"CREATED"}
# order-1:{"status":"PAID"}
```

### Consume Messages (Console)

```bash
# Consume from beginning
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --from-beginning

# Consume with key display
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --from-beginning \
  --property print.key=true \
  --property key.separator=" | "

# Consume in a specific group
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --group my-test-group

# Read only N messages then stop
kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic order-events \
  --from-beginning \
  --max-messages 10
```

### Consumer Group Management

```bash
# List all consumer groups
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --list

# Describe a group (shows lag per partition)
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --group my-test-group

# Reset offsets to earliest
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-test-group \
  --topic order-events \
  --reset-offsets \
  --to-earliest \
  --execute

# Reset to specific datetime
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group my-test-group \
  --topic order-events \
  --reset-offsets \
  --to-datetime 2024-01-15T08:00:00.000 \
  --execute
```

### Cluster & Broker Info

```bash
# Describe the cluster (brokers, controller)
kafka-broker-api-versions.sh \
  --bootstrap-server localhost:9092

# Check broker configs
kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --broker 1

# Check topic configs
kafka-configs.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic order-events \
  --entity-type topics
```

---

## 💻 7. Spring Boot Connection to Local Kafka

### `application.yml` for local dev

```yaml
spring:
  kafka:
    # Single node local
    bootstrap-servers: localhost:9092

    # Multi-broker local cluster
    # bootstrap-servers: localhost:9092,localhost:9093,localhost:9094

    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      acks: all
      retries: 3
      batch-size: 16384          # 16 KB batch size
      linger-ms: 5               # Wait up to 5ms to fill a batch
      buffer-memory: 33554432    # 32 MB producer buffer

    consumer:
      group-id: local-test-group
      auto-offset-reset: earliest
      enable-auto-commit: false  # Manual commit for learning
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      max-poll-records: 10       # Process 10 records per poll

# Topic auto-creation config (for local dev only)
app:
  kafka:
    topics:
      order-events:
        partitions: 3
        replication-factor: 1
```

---

### Auto-Create Topics on Startup (Local Dev)

```java
package com.example.kafka.config;

import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
@Profile("local")   // Only create topics when running locally
public class LocalKafkaTopicConfig {

    @Bean
    public NewTopic orderEventsTopic() {
        return TopicBuilder.name("order-events")
                .partitions(3)
                .replicas(1)     // single replica for local
                .build();
    }

    @Bean
    public NewTopic paymentsTopic() {
        return TopicBuilder.name("payments")
                .partitions(3)
                .replicas(1)
                .build();
    }

    @Bean
    public NewTopic userEventsTopic() {
        return TopicBuilder.name("user-events")
                .partitions(3)
                .replicas(1)
                .build();
    }
}
```

---

### Health Check — Verify Kafka Connection

```java
package com.example.kafka.health;

import org.apache.kafka.clients.admin.AdminClient;
import org.apache.kafka.clients.admin.DescribeClusterResult;
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.kafka.core.KafkaAdmin;
import org.springframework.stereotype.Component;

@Component
public class KafkaHealthIndicator implements HealthIndicator {

    private final KafkaAdmin kafkaAdmin;

    public KafkaHealthIndicator(KafkaAdmin kafkaAdmin) {
        this.kafkaAdmin = kafkaAdmin;
    }

    @Override
    public Health health() {
        try (AdminClient client = AdminClient.create(
                kafkaAdmin.getConfigurationProperties())) {

            DescribeClusterResult result = client.describeCluster();
            int brokerCount = result.nodes().get().size();
            String clusterId = result.clusterId().get();

            return Health.up()
                    .withDetail("clusterId", clusterId)
                    .withDetail("brokerCount", brokerCount)
                    .build();

        } catch (Exception e) {
            return Health.down()
                    .withDetail("error", e.getMessage())
                    .build();
        }
    }
}
```

Access at: `GET http://localhost:8080/actuator/health`

---

## ⚡ 8. Key Environment Variables (Docker)

| Variable | Example | Purpose |
|----------|---------|---------|
| `KAFKA_NODE_ID` | `1` | Unique broker/controller ID |
| `KAFKA_PROCESS_ROLES` | `broker,controller` | KRaft roles for this node |
| `KAFKA_CONTROLLER_QUORUM_VOTERS` | `1@kafka:9093` | KRaft quorum member list |
| `KAFKA_ADVERTISED_LISTENERS` | `PLAINTEXT://localhost:9092` | Address clients connect to |
| `KAFKA_DEFAULT_REPLICATION_FACTOR` | `3` | RF for auto-created topics |
| `KAFKA_NUM_PARTITIONS` | `3` | Partitions for auto-created topics |
| `KAFKA_MIN_INSYNC_REPLICAS` | `2` | Min ISR for write acceptance |
| `KAFKA_LOG_RETENTION_HOURS` | `168` | How long to keep messages |
| `CLUSTER_ID` | `MkU3OEVBNTcwNTJENDM2Qk` | Fixed cluster identity |

---

## 🔄 9. Useful Docker Commands for Day-to-Day

```bash
# Start / Stop / Restart
docker-compose up -d
docker-compose down
docker-compose restart kafka

# View logs
docker-compose logs -f kafka
docker-compose logs -f kafka-1 kafka-2

# Exec into container (run CLI commands)
docker exec -it kafka bash

# Once inside, run CLI:
kafka-topics.sh --bootstrap-server localhost:9092 --list

# Remove all data (full reset)
docker-compose down -v   # -v removes named volumes
```

---

## ⚠️ 10. Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| `ADVERTISED_LISTENERS` set to container hostname | Spring Boot can't connect from host | Set to `localhost` for local dev |
| No `CLUSTER_ID` in multi-broker setup | Brokers can't form quorum | Generate once with `kafka-storage.sh random-uuid` and hardcode |
| Forgetting `-v` on `docker-compose down` | Old data persists, causes startup errors | Use `docker-compose down -v` for clean reset |
| Spring Boot connecting before Kafka is ready | `Connection refused` on startup | Add `depends_on` or use retry config (`reconnect.backoff.ms`) |
| Port conflicts | Container fails to start | Check `lsof -i :9092` and kill conflicting processes |
| `replication.factor > broker count` | Topic creation fails | RF must be ≤ number of live brokers |
| Using `localhost` in container-to-container | Container can't reach Kafka | Use service name (`kafka-1`) for inter-container communication |

---

## 🎯 11. Interview Questions

**Q1. What is `ADVERTISED_LISTENERS` and why does it matter?**
> `ADVERTISED_LISTENERS` is the address that Kafka brokers tell clients to use when connecting. In Docker, the broker runs inside a container (hostname = container name), but clients on the host machine need `localhost`. Getting this wrong is the #1 cause of connection failures in local setups.

**Q2. What is the difference between `LISTENERS` and `ADVERTISED_LISTENERS`?**
> `LISTENERS` is the address Kafka binds to and listens on internally (e.g., `0.0.0.0:9092` — all interfaces). `ADVERTISED_LISTENERS` is what gets sent to clients in metadata responses — the address clients should use to connect. They differ when the server is behind NAT, Docker, or a load balancer.

**Q3. Why do you need a `CLUSTER_ID` in KRaft mode?**
> KRaft requires a stable cluster identity to prevent different clusters from accidentally joining each other. The cluster ID is generated once with `kafka-storage.sh random-uuid`, written to the storage directory, and must match across all brokers in the same cluster.

**Q4. How would you completely reset a local Kafka environment?**
> Run `docker-compose down -v` (the `-v` flag removes Docker named volumes where Kafka stores its data). Then `docker-compose up -d` starts fresh with no prior topics, offsets, or consumer group state.

---

## 📝 12. Quick Revision Summary

```
✅ 3 local setup options:
   Single-node KRaft Docker  → quickest, best for learning
   Multi-broker Docker       → realistic cluster simulation
   Native install            → deep OS-level access

✅ Most important Docker env vars:
   KAFKA_NODE_ID              → unique ID per node
   KAFKA_PROCESS_ROLES        → broker | controller | broker,controller
   KAFKA_CONTROLLER_QUORUM_VOTERS → quorum membership list
   KAFKA_ADVERTISED_LISTENERS → what clients see (use localhost for local dev)
   CLUSTER_ID                 → fixed per cluster, generate once

✅ Essential CLI commands:
   kafka-topics.sh --create / --list / --describe / --delete
   kafka-console-producer.sh → send test messages
   kafka-console-consumer.sh --from-beginning → verify messages
   kafka-consumer-groups.sh --describe → check lag
   docker exec -it kafka bash → get inside container

✅ Spring Boot local config:
   bootstrap-servers: localhost:9092
   @Profile("local") for dev-only topic creation
   KafkaHealthIndicator for connection verification

✅ Clean reset: docker-compose down -v && docker-compose up -d
```

---

**← Previous:** [03 — Kafka Architecture Overview](./03-architecture-overview.md)  
**Next Topic →** [05 — Kafka Producers: Internals, Configs & Guarantees](./05-producers.md)
