# 🚀 Apache Kafka — Complete Learning Guide
### From Basics to Senior-Level Architecture

> A structured, in-depth learning path for backend engineers targeting **senior/architect-level** Kafka mastery.  
> Each topic is covered in a dedicated `.md` file with explanations, diagrams, examples, interview questions, and scenario-based problems.

---

## 📌 How to Use This Repo

- Topics are learned **one by one**, in order.
- Each topic has its own dedicated `.md` note file.
- Notes include: deep explanations, flow diagrams, real-world examples, code snippets, interview Q&A, and scenario-based problems.
- Revise any topic anytime by jumping to its file.

---

## 🗺️ Learning Path Overview

```
Fundamentals → Core Internals → Producers & Consumers
→ Cluster & Replication → Storage & Performance
→ Streams & Connect → Schema & Security
→ Monitoring → Architecture Patterns → Real-World Systems
```

---

## 📚 Syllabus

### 🟢 Phase 1 — Foundations

| # | Topic | File | Status |
|---|-------|------|--------|
| 01 | Introduction to Apache Kafka — What, Why & Where | [01-introduction.md](./notes/01-introduction.md) | ⬜ |
| 02 | Core Concepts — Topics, Partitions, Offsets, Messages | [02-core-concepts.md](./notes/02-core-concepts.md) | ⬜ |
| 03 | Kafka Architecture Overview — Brokers, Clusters, ZooKeeper/KRaft | [03-architecture-overview.md](./notes/03-architecture-overview.md) | ⬜ |
| 04 | Setting Up Kafka Locally — Docker & Native Install | [04-local-setup.md](./notes/04-local-setup.md) | ⬜ |

---

### 🔵 Phase 2 — Producers & Consumers (Core Mechanics)

| # | Topic | File | Status |
|---|-------|------|--------|
| 05 | Kafka Producers — Internals, Configs & Guarantees | [05-producers.md](./notes/05-producers.md) | ⬜ |
| 06 | Kafka Consumers — Poll Loop, Offsets & Commits | [06-consumers.md](./notes/06-consumers.md) | ⬜ |
| 07 | Consumer Groups & Partition Assignment Strategies | [07-consumer-groups.md](./notes/07-consumer-groups.md) | ⬜ |
| 08 | Message Delivery Semantics — At Most / At Least / Exactly Once | [08-delivery-semantics.md](./notes/08-delivery-semantics.md) | ⬜ |

---

### 🟡 Phase 3 — Kafka Internals & Storage

| # | Topic | File | Status |
|---|-------|------|--------|
| 09 | Kafka Broker Internals — Request Handling & Log Architecture | [09-broker-internals.md](./notes/09-broker-internals.md) | ⬜ |
| 10 | Partitioning Strategies — Keys, Custom Partitioners & Ordering | [10-partitioning.md](./notes/10-partitioning.md) | ⬜ |
| 11 | Replication, ISR, Leader Election & Fault Tolerance | [11-replication.md](./notes/11-replication.md) | ⬜ |
| 12 | Log Storage, Segments, Retention & Compaction | [12-log-storage.md](./notes/12-log-storage.md) | ⬜ |
| 13 | ZooKeeper vs KRaft — Kafka's Metadata Evolution | [13-zookeeper-kraft.md](./notes/13-zookeeper-kraft.md) | ⬜ |

---

### 🟠 Phase 4 — Configuration, Tuning & Performance

| # | Topic | File | Status |
|---|-------|------|--------|
| 14 | Producer Configurations Deep Dive — batching, compression, acks, retries | [14-producer-configs.md](./notes/14-producer-configs.md) | ⬜ |
| 15 | Consumer Configurations Deep Dive — fetch size, lag, rebalancing | [15-consumer-configs.md](./notes/15-consumer-configs.md) | ⬜ |
| 16 | Broker & Topic Configurations — throughput, retention, replication | [16-broker-topic-configs.md](./notes/16-broker-topic-configs.md) | ⬜ |
| 17 | Performance Tuning & Benchmarking Kafka | [17-performance-tuning.md](./notes/17-performance-tuning.md) | ⬜ |

---

### 🔴 Phase 5 — Kafka Streams & Connect

| # | Topic | File | Status |
|---|-------|------|--------|
| 18 | Kafka Streams — Stream Processing Fundamentals | [18-kafka-streams-basics.md](./notes/18-kafka-streams-basics.md) | ⬜ |
| 19 | Kafka Streams — Stateful Operations, KTables & Windowing | [19-kafka-streams-advanced.md](./notes/19-kafka-streams-advanced.md) | ⬜ |
| 20 | Kafka Connect — Source & Sink Connectors | [20-kafka-connect.md](./notes/20-kafka-connect.md) | ⬜ |
| 21 | ksqlDB — Streaming SQL on Kafka | [21-ksqldb.md](./notes/21-ksqldb.md) | ⬜ |

---

### 🟣 Phase 6 — Schema, Security & Multi-Tenancy

| # | Topic | File | Status |
|---|-------|------|--------|
| 22 | Schema Registry — Avro, Protobuf & JSON Schema | [22-schema-registry.md](./notes/22-schema-registry.md) | ⬜ |
| 23 | Schema Evolution — Compatibility Strategies | [23-schema-evolution.md](./notes/23-schema-evolution.md) | ⬜ |
| 24 | Kafka Security — SSL/TLS, SASL & Encryption | [24-security-ssl-sasl.md](./notes/24-security-ssl-sasl.md) | ⬜ |
| 25 | Kafka Authorization — ACLs & Multi-Tenancy | [25-authorization-acls.md](./notes/25-authorization-acls.md) | ⬜ |

---

### ⚫ Phase 7 — Observability & Operations

| # | Topic | File | Status |
|---|-------|------|--------|
| 26 | Monitoring Kafka — JMX, Prometheus & Grafana | [26-monitoring.md](./notes/26-monitoring.md) | ⬜ |
| 27 | Consumer Lag — Detection, Alerting & Remediation | [27-consumer-lag.md](./notes/27-consumer-lag.md) | ⬜ |
| 28 | Kafka Operations — Topic Management, Rolling Restarts & Upgrades | [28-operations.md](./notes/28-operations.md) | ⬜ |
| 29 | Disaster Recovery — Backup, MirrorMaker 2 & Multi-DC Replication | [29-disaster-recovery.md](./notes/29-disaster-recovery.md) | ⬜ |

---

### 🏗️ Phase 8 — Architecture Patterns & System Design

| # | Topic | File | Status |
|---|-------|------|--------|
| 30 | Event-Driven Architecture with Kafka | [30-event-driven-architecture.md](./notes/30-event-driven-architecture.md) | ⬜ |
| 31 | CQRS & Event Sourcing with Kafka | [31-cqrs-event-sourcing.md](./notes/31-cqrs-event-sourcing.md) | ⬜ |
| 32 | Saga Pattern — Distributed Transactions via Kafka | [32-saga-pattern.md](./notes/32-saga-pattern.md) | ⬜ |
| 33 | Outbox Pattern — Reliable Event Publishing | [33-outbox-pattern.md](./notes/33-outbox-pattern.md) | ⬜ |
| 34 | Kafka in Microservices — Patterns & Anti-Patterns | [34-kafka-microservices.md](./notes/34-kafka-microservices.md) | ⬜ |

---

### 🌐 Phase 9 — Deployment & Cloud

| # | Topic | File | Status |
|---|-------|------|--------|
| 35 | Kafka on Docker & Docker Compose | [35-kafka-docker.md](./notes/35-kafka-docker.md) | ⬜ |
| 36 | Kafka on Kubernetes — Strimzi & Helm | [36-kafka-kubernetes.md](./notes/36-kafka-kubernetes.md) | ⬜ |
| 37 | Managed Kafka — Confluent Cloud, AWS MSK, Azure Event Hubs | [37-managed-kafka.md](./notes/37-managed-kafka.md) | ⬜ |

---

### 🎯 Phase 10 — Senior/Architect Level Mastery

| # | Topic | File | Status |
|---|-------|------|--------|
| 38 | Kafka System Design — Designing Real-World Systems | [38-system-design.md](./notes/38-system-design.md) | ⬜ |
| 39 | Kafka Anti-Patterns & Common Mistakes | [39-anti-patterns.md](./notes/39-anti-patterns.md) | ⬜ |
| 40 | Senior-Level Interview Prep — 50 Questions & Scenario Bank | [40-interview-prep.md](./notes/40-interview-prep.md) | ⬜ |

---

## 📁 Repo Structure

```
kafka-notes/
│
├── README.md                  ← You are here (Syllabus)
│
└── notes/
    ├── 01-introduction.md
    ├── 02-core-concepts.md
    ├── 03-architecture-overview.md
    ├── ... (one file per topic)
    └── 40-interview-prep.md
```

---

## 🧩 What Each Note File Contains

Every `.md` topic file is structured as follows:

```
1. 📖 Concept Explanation       — Deep dive into the topic
2. 🏗️  Architecture / Flow       — Diagrams (ASCII/Mermaid) showing how it works
3. ⚙️  How It Works Internally   — Under-the-hood mechanics
4. 💻 Code Examples             — Real working code snippets
5. ⚡ Key Configurations        — Important settings and their impact
6. 🔄 Real-World Scenarios      — How this is used in production systems
7. ⚠️  Common Pitfalls          — Mistakes to avoid
8. 🎯 Interview Questions       — Conceptual Q&A
9. 🧠 Scenario-Based Problems   — Architect/senior-level problem solving
10. 📝 Quick Revision Summary   — Key takeaways in bullet points
```

---

## 🧭 Progress Tracker

- ⬜ Not Started
- 🔄 In Progress  
- ✅ Completed

---

## 🛠️ Prerequisites

Before starting, you should be comfortable with:
- Basic understanding of distributed systems concepts
- Any backend programming language (Java / Python / Go / Node.js)
- Basic Docker knowledge (for hands-on labs)
- Understanding of TCP/IP networking basics

---

## 📖 Reference Resources

| Resource | Type |
|----------|------|
| [Apache Kafka Official Docs](https://kafka.apache.org/documentation/) | Documentation |
| [Confluent Developer](https://developer.confluent.io/) | Tutorials & Courses |
| [Kafka: The Definitive Guide (O'Reilly)](https://www.confluent.io/resources/kafka-the-definitive-guide/) | Book |
| [Designing Data-Intensive Applications](https://dataintensive.net/) | Book |
| [Confluent Blog](https://www.confluent.io/blog/) | Articles |

---

> 💡 **Tip:** Update the **Status** column in the table above as you complete each topic to track your progress visually on GitHub.

---

*Notes maintained by learning through structured, topic-by-topic deep dives.*
