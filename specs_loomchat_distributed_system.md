# SPEC-1-Distributed Chat System

## Background

This project is an educational yet production-grade distributed messaging/chat system intended to simulate real-world backend challenges at scale. The system is designed for millions of registered users with tens of thousands of concurrent active users. It focuses on core backend concerns such as scalability, reliability, consistency, fault tolerance, and observability, using Go and Java as primary backend languages.

The goal is to help the developer gain hands-on experience with senior-level backend architecture patterns while still keeping the system implementable by a small team.

**Confirmed assumptions:**

* 1:1 and group chats, including very large groups (10k+ members)
* Messages are persisted (not ephemeral)
* Per-conversation message ordering
* At-least-once delivery semantics
* Presence, last-seen, read receipts, typing indicators
* Near–real-time messaging (<1s latency target)
* Eventual consistency acceptable
* Cloud-native deployment (Kubernetes)

## Requirements

### Must Have

* Real-time 1:1 and group messaging
* Support for very large groups (10k+ members)
* Message persistence and history retrieval
* Per-conversation message ordering
* Horizontal scalability to tens of thousands of concurrent users
* Fault tolerance (no single point of failure)
* User presence (online/offline/last seen)
* Authentication & authorization

### Should Have

* Read receipts and typing indicators
* Push notification integration (abstracted)
* Message search (basic)
* Observability (logs, metrics, traces)

### Could Have

* Media messages (images/files)
* Message reactions
* Admin/moderation tools for large groups

### Won’t Have (for MVP)

* End-to-end encryption
* Voice/video calls
* Federated chat (Matrix-style)

## Method

### High-Level Architecture Overview

The system follows a **service-oriented, event-driven architecture** optimized for real-time communication and large fan-out scenarios.

Clients connect via **WebSocket** to a stateless edge layer. Core responsibilities are split into specialized backend services to ensure scalability and fault isolation.

#### Core Services

1. **API Gateway / WebSocket Gateway (Go)**

   * Terminates WebSocket connections
   * Authenticates users (JWT)
   * Maintains connection ↔ user mapping
   * Routes incoming messages to backend messaging pipeline
   * Pushes outbound messages to connected clients

2. **Messaging Service (Java)**

   * Core message processing logic
   * Validates permissions (sender is member of conversation)
   * Assigns monotonically increasing message sequence IDs per conversation
   * Persists messages
   * Publishes message events to the event bus

3. **Group Fan-out Service (Go)**

   * Handles message distribution for group chats
   * Uses **lazy fan-out** for very large groups
   * Resolves online members via Presence Service
   * Pushes delivery tasks to Gateway layer

4. **Presence Service (Go)**

   * Tracks online/offline state
   * Stores ephemeral presence data in Redis
   * Exposes APIs for querying active users per group

5. **Notification Service (Java)**

   * Consumes message events
   * Sends push notifications to offline users
   * Abstracts external providers (FCM/APNs)

6. **Auth Service (Java)**

   * User authentication
   * JWT issuance & validation
   * Manages user identity

---

### Infrastructure Components

* **Kafka** – durable event bus for message propagation
* **Redis Cluster** – presence, connection mapping, ephemeral state
* **Primary Database (Cassandra / ScyllaDB)** – message storage (high write throughput)
* **Relational DB (PostgreSQL)** – users, groups, memberships
* **Kubernetes** – container orchestration
* **Envoy / NGINX** – ingress + L7 routing

---

### Client Communication Flow

```plantuml
@startuml
Client -> WebSocket Gateway: Send Message
WebSocket Gateway -> Messaging Service: Validate & Enqueue
Messaging Service -> DB: Persist Message
Messaging Service -> Kafka: Publish Message Event
Kafka -> Fan-out Service: Consume Event
Fan-out Service -> Presence Service: Resolve Online Users
Fan-out Service -> WebSocket Gateway: Deliver Message
@enduml
```

---

### Key Design Decisions

* **WebSocket at the edge**: minimizes latency and supports real-time bidirectional messaging
* **Event-driven core (Kafka)**: decouples message ingestion from delivery
* **Lazy fan-out for large groups**: avoids O(N) writes per message
* **Single-write message storage**: messages stored once per conversation
* **Per-user read offsets**: scalable read receipts without duplicating messages
* **Stateless gateways**: allows horizontal scaling without sticky sessions
* **Polyglot services (Go + Java)**:

  * Go for high-concurrency, low-latency services
  * Java for complex business logic and ecosystem maturity

---

### Data Model & Storage Strategy

#### Relational Data (PostgreSQL)

Used for strongly consistent, low-churn data.

**users**

* user_id (PK, UUID)
* username
* created_at

**conversations**

* conversation_id (PK, UUID)
* type (DIRECT | GROUP)
* created_at

**conversation_members**

* conversation_id (PK)
* user_id (PK)
* role (MEMBER | ADMIN)
* joined_at

**user_conversation_state**

* conversation_id (PK)
* user_id (PK)
* last_read_message_seq
* last_seen_at

---

#### Message Storage (Cassandra / ScyllaDB)

Optimized for high write throughput and sequential reads.

**messages_by_conversation**

* conversation_id (partition key)
* message_seq (clustering key, monotonically increasing)
* message_id (UUID)
* sender_id
* content
* created_at

**Partitioning strategy:**

* One partition per conversation
* Message ordering guaranteed by message_seq

---

#### Ephemeral Data (Redis)

* user:{user_id}:presence -> ONLINE | OFFLINE
* connection:{connection_id} -> user_id
* conversation:{id}:online_users -> set(user_id)

---

### Message Ordering Algorithm

* Each conversation has a logical **message sequence counter**
* Messaging Service uses a lightweight **per-conversation sequencer**
* Sequence generated before persistence
* Ensures strict ordering within a conversation

```plantuml
@startuml
Messaging Service -> Sequencer: nextSeq(conversation)
Sequencer --> Messaging Service: seq
Messaging Service -> Cassandra: insert(message, seq)
@enduml
```

**Why this works:**

* No distributed transactions
* Ordering scoped to a single conversation
* Scales horizontally across conversations
