# Scaling Strategy — Real-Time Chat System

---

## 1. Overview

The system must serve **500 million DAU**, sustain **~5 million messages/second** at peak, and maintain **99.99% availability**. Scaling is achieved through a combination of horizontal scaling, intelligent caching, data partitioning, and geographic distribution.

---

## 2. Load Balancing

### 2.1 Global Traffic Management

```
Internet
    │
    ▼
 Anycast DNS (Route 53 / Cloud DNS)
    │   Geo-routing: route users to nearest PoP
    ▼
 CDN Edge (CloudFront / Fastly)
    │   Static assets, media thumbnails, cached API responses
    ▼
 Global Load Balancer (Layer 7 — ALB / GCP GCLB)
    │   TLS termination, path-based routing
    ▼
 ┌────────────────────────────────────────────┐
 │  Internal Services  (per region)           │
 │  API Servers │ WebSocket Servers │ Workers  │
 └────────────────────────────────────────────┘
```

### 2.2 Layer-by-Layer Strategy

**L4 / L7 Load Balancers (AWS ALB / NLB)**
- HTTP/2 and WebSocket support.
- Health checks every 5 s; unhealthy pods removed within 10 s.
- Connection draining (30 s) before pod termination for zero-downtime deploys.

**WebSocket Connection Routing (Consistent Hashing)**
- WebSocket connections are long-lived; re-balancing must be minimized.
- A **consistent hash ring** on `user_id` routes each user's connections to a stable pod.
- If a pod dies, the ring reroutes to the next node; clients reconnect with exponential back-off.

**Internal Service Mesh (Istio / Envoy)**
- mTLS between microservices.
- Circuit breakers and retries configured per service pair.
- gRPC load balancing for internal RPC calls.

### 2.3 Auto-Scaling

|          Service     |             Trigger            | Min |  Max |
|----------------------|--------------------------------|-----|------|
|      API Servers     |      CPU > 60% for 2 min       |  20 |  500 |
|    WebSocket Servers | Active connections > 50 k/pod  |  50 | 2000 |
|     Message Workers  | Kafka consumer lag > 10 k      |  10 |  200 |
| Notification Workers |       Queue depth > 50 k       |   5 |  100 |

Kubernetes HPA (Horizontal Pod Autoscaler) with KEDA for Kafka-lag-based scaling.

---

## 3. Caching Strategy

### 3.1 Cache Layers

```
Client App
  └── Local SQLite cache (last 500 messages per conv, user profiles)

CDN Edge Cache
  └── Media thumbnails (TTL: 7 days), static assets (TTL: 30 days)

Redis Cluster (Primary Cache — in-region)
  └── User sessions & presence (TTL: 30 s)
  └── Typing indicators (TTL: 5 s)
  └── Hot conversation member lists (TTL: 5 min)
  └── Rate-limit counters (TTL: 60 s)
  └── Pre-computed inbox (latest msg per conv, TTL: 30 s)

Cassandra Row Cache (L2 — on-box)
  └── Recently accessed message partitions (LRU, configurable size)
```

### 3.2 Cache-Aside Pattern (User Profiles)

```
Read:
  1. Check Redis → HIT: return; MISS → continue
  2. Query PostgreSQL → found → write to Redis (TTL 5 min) → return

Write:
  1. Update PostgreSQL (primary source of truth)
  2. Invalidate Redis key (delete, not update — avoids races)
  3. Next read will repopulate from DB
```

### 3.3 Write-Through (Presence)

Presence is updated on every heartbeat (every 15 s):
1. WebSocket server writes directly to Redis (primary).
2. A background job syncs Redis → PostgreSQL `users.last_seen_at` every 60 s.
3. On disconnect, write `last_seen_at` immediately.

### 3.4 Cache Sizing & Eviction

| Redis Cluster |        Purpose                     |    Size    |   Eviction  |
|---------------|------------------------------------|------------|-------------|
| sessions      |    Auth, presence, rate-limits     | 128 GB × 3 | allkeys-lru |
| hot-data      | Inbox, member lists, profile cache | 64 GB × 3  | allkeys-lru |
| pubsub        |      Message fan-out channels      | 32 GB × 3  | no eviction |

Redis Cluster deployed with 3 shards × 3 replicas (1 primary + 2 replicas per shard) across 3 AZs.

---

## 4. Sharding Strategy

### 4.1 PostgreSQL — Vertical + Horizontal

**Vertical split (functional partitioning):**
- `users`, `user_devices`, `user_blocks` → **Users DB**
- `conversations`, `conversation_members` → **Conversations DB**
- `media_metadata` → **Media DB**

**Horizontal sharding (when a DB exceeds 5 TB):**
- Shard `users` by `user_id % N` (consistent hashing with virtual nodes).
- Shard `conversations` by `conversation_id % N`.
- Use **Citus** (PostgreSQL extension) or **PlanetScale** for transparent sharding.
- A routing layer maps `user_id → shard_id` via a small lookup table in Redis.

**Read replicas:**
- Each primary has ≥ 2 read replicas in separate AZs.
- 90% of reads routed to replicas; writes to primary only.
- Replica lag monitored; reads fall back to primary if lag > 5 s.

### 4.2 Cassandra — Native Partition-based Sharding

Cassandra shards automatically via its **consistent hash ring** on the partition key.

**Partition key:** `(conversation_id, bucket)` where `bucket = YYYY-MM`

- A busy group chat with millions of messages per month stays within one time-bucket partition.
- `MURMUR3` token-based distribution spreads partitions evenly across nodes.
- Each data center runs ≥ 6 nodes; replication factor = 3.
- **Compaction strategy:** `TimeWindowCompactionStrategy` (TWCS) — optimal for append-heavy time-series; expired buckets compact and purge efficiently.

### 4.3 Kafka — Topic Partitioning

```
Topic: chat.messages
Partitions: 1024
Partition key: conversation_id   ← orders messages within a conversation
Replication factor: 3
Retention: 7 days (messages confirmed consumed → no longer needed in Kafka)
```

Message workers consume in consumer groups:
- `group: fan-out-workers` — deliver to recipient WebSocket servers / push.
- `group: persistence-workers` — write to Cassandra.
- `group: search-indexer` — index into Elasticsearch.
- `group: analytics-workers` — emit to data warehouse (BigQuery / Redshift).

---

## 5. Geographic Distribution (Multi-Region)

```
Regions: us-east-1 | eu-west-1 | ap-south-1 | ap-southeast-1

Each region has:
  - Full application stack (API + WS servers + workers)
  - Regional Redis cluster
  - Regional Cassandra ring (RF=3 local + cross-region async replication)
  - Regional PostgreSQL primary (with async replication to other regions)
  - Regional Kafka cluster (MirrorMaker 2 for cross-region replication)
  - Regional CDN PoP
```

**Message routing for cross-region conversations:**
1. Sender's message hits their nearest WebSocket server (region A).
2. Kafka in region A replicates to recipient's region (region B) via MirrorMaker 2.
3. Fan-out worker in region B delivers to recipient's WebSocket server.
4. Total cross-region latency: < 200 ms (same planet).

---

## 6. Connection Management at Scale

**WebSocket servers handle 50,000 concurrent connections each.**  
At 100 M concurrent connections: 100M / 50K = **2,000 pods** across all regions.

**Presence fan-out optimization:**  
Rather than sending presence updates to all contacts (which explodes to O(contacts × presence_updates)), the system uses a **subscription model**:
- Client explicitly subscribes to presence of users in currently open conversations.
- Server only fans out to subscribers → reduces fan-out by 10–100×.

**Message fan-out for large groups (1,024 members):**
- Direct WebSocket push for online members (via Redis pub/sub `user:{user_id}` channels).
- Push notification for offline members (batched, max 1 notification per 30 s per group).
- A dedicated **Group Fan-out Service** handles groups > 100 members using a tree-based fan-out.

---

## 7. Observability for Scale

- **Distributed tracing** (OpenTelemetry → Jaeger): trace every message from send → Kafka → persistence → delivery.
- **SLO dashboards**: Message delivery latency p50/p95/p99, broken down by region and conversation type.
- **Capacity forecasting**: 90-day rolling trend analysis for pod count, storage, and network egress.
- **Chaos engineering** (Netflix Chaos Monkey-style): weekly randomized pod kills and AZ failures in staging; monthly in production.