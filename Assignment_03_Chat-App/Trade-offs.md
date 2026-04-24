# Trade-offs & Design Decisions — Real-Time Chat System

---

## 1. WebSockets vs. Long Polling vs. Server-Sent Events

### Decision: WebSockets (primary) + REST (fallback)

| Criterion                     | WebSockets         | Long Polling | Server-Sent Events |
|-------------------------------|--------------------|--------------|--------------------|
| Latency                       | ~5 ms              | 50–500 ms    | ~10 ms             |
| Bi-directionality             | Full-duplex ✅     | Simulated (two connections) | Server→Client only |
| Connection overhead           | One persistent TCP | New HTTP per poll          | One persistent HTTP |
| Firewall / Proxy friendliness | Moderate           | High ✅      | High ✅           |
| Server resource usage         | High(one fd per user) | Very high (threads blocked) | Medium |
| Mobile battery impact         | Medium             | High (CPU wakeups) |             Medium |

**Why WebSockets:**  
Chat is fundamentally bidirectional and latency-sensitive. WebSockets give full-duplex with minimal overhead once connected. The 100 ms target is impossible with polling.

**Trade-off accepted:**  
WebSocket connections hold open file descriptors and memory on the server (~50 KB/connection). At 100 M concurrent connections, this requires careful pod sizing and connection multiplexing. Corporate firewalls or misconfigured proxies can drop WebSocket connections — the REST fallback exists for these cases.

---

## 2. SQL (PostgreSQL) vs. NoSQL (Cassandra) for Messages

### Decision: Cassandra for messages, PostgreSQL for user/conversation metadata

**Why not PostgreSQL for messages?**
- At 100 B messages/day, a single PostgreSQL cluster would need petabyte-scale tables.
- `JOIN`s across billions of rows are prohibitively expensive.
- Vertical scaling (bigger machines) hits physical limits; horizontal sharding of PostgreSQL is complex and operationally costly.
- Time-series writes (sequential appends) are not PostgreSQL's strength.

**Why Cassandra?**
- Linear horizontal scale: add nodes, increase throughput proportionally.
- `(conversation_id, bucket)` partition key perfectly matches the read pattern (all messages in a conversation, paginated by time).
- Write throughput of 1 M+ writes/s per cluster is routine.
- Multi-datacenter replication built-in.

**Trade-off accepted:**
- Cassandra provides **eventual consistency** — a message written in one node may take milliseconds to appear on another. This is acceptable for message history but not for message ordering guarantees.
- No `JOIN`s: message history and user profiles are fetched in separate queries and joined application-side.
- No secondary indexes at scale — all queries must be on the partition or clustering key. Alternate access patterns (e.g., "all messages sent by user X globally") require a separate materialized table.
- Operations complexity: Cassandra requires tuning compaction, GC pressure, and hinted handoffs.

---

## 3. Push Notifications: APNs / FCM vs. Own Notification Infrastructure

### Decision: APNs + FCM via a notification microservice

**Why not build a custom push layer?**
- APNs and FCM are deeply integrated with iOS and Android power management. Custom solutions bypass the system's ability to wake the device efficiently, causing more battery drain.
- APNs/FCM handle retry, prioritization, and collapse keys natively.
- Maintaining delivery at 500 M/day with custom socket infrastructure to billions of heterogeneous devices is an enormous operational burden.

**Trade-off accepted:**
- Dependency on Apple and Google infrastructure — if APNs or FCM goes down, notifications fail.
- Device tokens expire; the system must handle token refresh callbacks.
- APNs/FCM impose payload size limits (4 KB APNs, 4 KB FCM data payload).

---

## 4. End-to-End Encryption: Client-Side vs. Server-Side

### Decision: End-to-End Encryption (Signal Protocol / Double Ratchet)

**Why E2EE?**
- Users have a reasonable expectation that private messages are private — even from the platform operator.
- Regulatory compliance in certain jurisdictions mandates data minimization.
- Reduces the severity of server-side breaches: stolen ciphertext is useless without keys.

**Trade-off accepted:**
- **No server-side spam/abuse filtering** of message content (content is opaque to the server). Abuse mitigation must rely on metadata (rate limits, user reports, account behavior).
- **Message search is hard**: full-text search must happen on-device. Server-side Elasticsearch indexes only metadata (sender, timestamp), not content.
- **Key management complexity**: lost devices mean no recovery of old messages; key rotation requires coordination; multi-device support requires per-device key bundles (Signal's Sealed Sender model).
- **Backup complexity**: if a user backs up messages to iCloud or Google Drive, those backups may or may not be E2EE depending on user settings — a UX and security communication challenge.

---

## 5. Message Fan-out: Push vs. Pull

### Decision: Push (server-initiated delivery) with Pull fallback

**Push model:** When a message arrives, the server immediately pushes it to all online recipients via WebSocket.

**Pull model:** Recipients periodically poll for new messages.

| Criterion                   | Push                      | Pull          |
|-----------------------------|---------------------------|---------------|
| Latency                     | ~5 ms ✅                  | 100 ms – 30 s |
| Server-side complexity      | High (fan-out logic)       | Low           |
| Works when connection drops | No (needs reconnect)       | Yes ✅       |
| Scales to large groups      | Hard (1024-member fan-out) | Easier        |

**Why Push primary:**  
Sub-100 ms delivery is a core requirement. Push is the only model that achieves it.

**Pull fallback:**  
On reconnect, the client pulls missed messages via REST `GET /conversations/{id}/messages?after={last_seen_id}`. This closes any gaps caused by WebSocket disconnects.

**Trade-off for large group fan-out:**  
Fan-out for a 1,024-member group requires writing to up to 1,024 Redis pub/sub channels. A naive implementation creates a thundering herd. The system mitigates this with a **tiered fan-out service**: first push to online members in the same region; then queue push notifications for offline members, with a rate limit (max 1 notification per member per 30 s per group).

---

## 6. Message Ordering: Client-side Sequence Numbers vs. Server Timestamps

### Decision: Server-assigned `TIMEUUID` (Cassandra v1 UUID) + client sequence numbers for display ordering

**Challenges with ordering in distributed systems:**
- Two users can send messages within the same millisecond.
- Clock skew between servers can cause out-of-order writes.
- Network delays can deliver messages out of order.

**Decision breakdown:**
- **Storage order**: `TIMEUUID` in Cassandra ensures total ordering within a partition (monotonically increasing via hybrid logical clock).
- **Display order**: Client uses message `sent_at` timestamp for visual ordering, with a secondary sort on `message_id` to break ties.
- **Causal ordering**: If user B replies to user A's message, B's client attaches `replied_to_id`, establishing causal relationship regardless of delivery order.

**Trade-off accepted:**  
Messages can appear slightly out of order on the recipient's screen if two messages arrive within the same clock tick. This is a cosmetic artifact tolerated in favor of system simplicity — strong global ordering (e.g., Spanner-style TrueTime) would require globally synchronized clocks and significantly more infrastructure cost.

---

## 7. Storage: Hot vs. Cold Media

### Decision: Tiered S3 storage with intelligent lifecycle policies

| Age         | Storage Class                | Cost     | Access Time |
|-------------|------------------------------|--------- |-------------|
| 0–30 days   | S3 Standard                  | High     | ms          |
| 30–90 days  | S3 Intelligent-Tiering       | Medium   | ms–s        |
| 90–365 days | S3 Standard-IA               | Low      | ms          |
| > 1 year    | S3 Glacier Instant Retrieval | Very low | ms          |

**Trade-off accepted:**  
Media older than 90 days takes longer (and costs more per request) to retrieve. For a chat app where users rarely access year-old media, this is acceptable.

---

## 8. Consistency Model: Strong vs. Eventual

### Decision: Eventual consistency for messages and presence; strong consistency for auth and user identity

| Feature | Model | Rationale |
|---------|-------|-----------|
| Message delivery | Eventual | Acceptable for chat; 99.9% of messages delivered within 100 ms |
| Read receipts | Eventual (lag < 5 s) | Minor UX trade-off for massive scale savings |
| Presence / last seen | Eventual (lag < 30 s) | Exact precision not needed |
| User registration / login | Strong | Cannot allow duplicate accounts or auth bugs |
| Conversation membership | Strong (within region) | Adding/removing a member must be immediately consistent for security |

**Trade-off accepted:**  
Eventual consistency means a user might briefly see an outdated unread count, or a read receipt might appear a few seconds late. These are acceptable UX imperfections at the benefit of eliminating distributed coordination overhead (no 2PC, no Paxos on the critical write path).

---

## 9. Microservices vs. Monolith

### Decision: Domain-driven microservices from day one

| Service              | Responsibility                        |
|----------------------|---------------------------------------|
| Auth Service         | Registration, login, token management |
| User Service         | Profiles, search, blocks              |
| Conversation Service | Conversation CRUD, group management   |
| Messaging Service    | Send, receive, status                 |
| Notification Service | Push notification orchestration       |
| Media Service        | Upload URL generation, metadata       |
| Presence Service     | Online status, typing indicators      |
| Search Service       | Elasticsearch read/write              |

**Trade-off accepted:**  
Microservices add operational overhead: service discovery, distributed tracing, inter-service latency, and the need for a mature DevOps culture. For a company at WhatsApp scale this is non-negotiable, but for an early startup, a well-structured monolith that is split later is a valid alternative ("majestic monolith" → extract services on pain points).

---

## Summary Table

| Decision | Choice | Key Trade-off |
|----------|--------|---------------|
| Real-time transport | WebSockets | fd overhead; REST fallback needed |
| Message storage | Cassandra | Eventual consistency; no JOINs |
| User/metadata storage | PostgreSQL | Must shard at extreme scale |
| Encryption | E2EE (Signal) | No server-side content moderation |
| Fan-out | Push (WebSocket) + Pull (REST fallback) | Group fan-out complexity |
| Message ordering | TIMEUUID + causal reply chains | Not globally strongly ordered |
| Media storage | Tiered S3 | Cold retrieval latency |
| Consistency | Eventual (messages) + Strong (auth) | Stale read receipts |
| Architecture | Microservices | Operational complexity |