# Scalability & Reliability — E-Commerce Inventory & Order System

---

## 1. Load Balancing Strategy

### Layer 1: DNS Load Balancing
- **AWS Route 53** with latency-based routing directs users to the nearest regional deployment
- Geo-routing sends users in Mumbai → ap-south-1, users in Delhi → ap-south-2

### Layer 2: Application Load Balancer (ALB)
- **Algorithm**: Round-Robin for stateless services (Product, Search)
- **Algorithm**: Least-connections for stateful-heavy services (Order, Payment)
- Health checks every 10 seconds; unhealthy instances removed from pool within 30 seconds
- **Sticky sessions disabled** — all services are stateless (state lives in Redis or DB)

### Layer 3: API Gateway (Kong)
- Acts as a reverse proxy before traffic reaches microservices
- Handles SSL termination, JWT validation, and routing centrally
- Can be horizontally scaled independently of backend services

### Layer 4: Service Mesh (Istio)
- Internal service-to-service traffic is load-balanced via Istio sidecar proxies
- Supports weighted routing for canary deployments (e.g., 5% traffic to v2)

---

## 2. Horizontal vs Vertical Scaling

### Our Approach: Horizontal Scaling First

| Service           | Scaling Strategy       | Justification                                              |
|-------------------|------------------------|------------------------------------------------------------|
| Product Service   | Horizontal             | Stateless; just reads from DB/cache — easy to replicate    |
| Search Service    | Horizontal             | Elasticsearch nodes scale horizontally by design           |
| Order Service     | Horizontal             | Stateless; saga state persisted in DB                      |
| Notification Svc  | Horizontal             | Kafka consumer groups naturally scale by adding instances  |
| PostgreSQL        | Vertical + Read Replicas| Primary scales vertically; reads offloaded to replicas    |
| Redis             | Horizontal (Cluster)   | Redis Cluster shards data across nodes                     |

### When Vertical Scaling is Used
- **PostgreSQL primary** — write throughput is hard to distribute; a larger instance (higher CPU/RAM) is simpler and safer for the primary node
- **Elasticsearch master nodes** — metadata operations are not parallelized; bigger instances help

### Auto-Scaling Policy (Kubernetes HPA)
```yaml
# Example: Product Service auto-scaling
minReplicas: 2
maxReplicas: 20
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        averageUtilization: 60    # scale up when CPU > 60%
```
During flash sales, we **pre-warm** instances 30 min before the event rather than waiting for reactive scaling.

---

## 3. Caching Strategy

### 3.1 What We Cache

| Data                | Cache Layer  | TTL      | Invalidation Strategy               |
|---------------------|--------------|----------|-------------------------------------|
| Product details     | Redis        | 5 min    | Event-driven (product updated → delete key) |
| Search results      | Redis        | 2 min    | TTL only (acceptable staleness)     |
| User session token  | Redis        | 15 min   | On logout / expiry                  |
| Inventory count     | Redis        | 30 sec   | Write-through on every stock change |
| Homepage featured   | Redis + CDN  | 10 min   | Manual purge on catalog change      |
| Static assets (JS, CSS, images) | CDN (CloudFront) | 7 days | Cache-busting via content hash in filename |

### 3.2 Cache Patterns Used

**Cache-Aside (Lazy Loading)** — for product details:
```
1. Service checks Redis for product:{id}
2. HIT  → return cached value
3. MISS → query PostgreSQL → store in Redis with TTL → return value
```

**Write-Through** — for inventory counts:
```
1. Every stock change updates both PostgreSQL and Redis simultaneously
2. Ensures cache and DB are never out of sync for stock data
```

**Read-Through via CDN** — for product images:
```
1. First request fetches from S3 origin
2. CloudFront caches at edge for subsequent requests globally
3. Dramatically reduces S3 GET costs and latency
```

### 3.3 Cache Stampede Prevention
When a popular cached key expires and thousands of requests hit the DB simultaneously:
- **Solution**: Use **probabilistic early expiration** (PER) — a small % of requests refresh the cache slightly before TTL expires, preventing the thundering herd

### 3.4 Redis Cluster Setup
- 6 nodes: 3 masters + 3 replicas
- Data sharded across masters using consistent hashing
- Automatic failover: replica promoted to master within 5–10 seconds of master failure

---

## 4. Database Scaling

### 4.1 Read Replicas (Replication)
```
                  ┌───────────────────┐
                  │  PostgreSQL       │
                  │  PRIMARY (writes) │
                  └───────┬───────────┘
                          │ WAL streaming
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ Replica 1│   │ Replica 2│   │ Replica 3│
    │ (reads)  │   │ (reads)  │   │(analytics│
    └──────────┘   └──────────┘   └──────────┘
```
- All writes go to primary
- Read-heavy queries (product list, order history) routed to replicas via PgBouncer
- Replication lag monitored — if lag > 1 second, reads fall back to primary

### 4.2 Sharding Strategy
At scale (100M+ orders), the orders table will need sharding:

- **Sharding Key**: `user_id` (range-based sharding by user ID prefix)
- Users with IDs 000000–3FFFFF → Shard 1, 400000–7FFFFF → Shard 2, etc.
- Orders, cart, and payments all sharded by same user_id for data locality

**Trade-off**: Cross-user queries (admin reports) require scatter-gather across shards. This is acceptable since admin queries are batch operations, not user-facing.

### 4.3 Connection Pooling
- **PgBouncer** sits between services and PostgreSQL
- Mode: **Transaction pooling** — connections released back to pool after each transaction
- Pool size: 100 connections (avoids PostgreSQL's max_connections limit)

### 4.4 Database for Search
- Product catalog data is **dual-written**: PostgreSQL (source of truth) + Elasticsearch (search index)
- A Kafka consumer syncs changes from PostgreSQL CDC (Change Data Capture via Debezium) to Elasticsearch asynchronously

---

## 5. Failure Handling

### 5.1 Retry Logic with Exponential Backoff
For transient failures (network timeout, 5xx from downstream), services retry automatically:

```
Attempt 1: immediate
Attempt 2: wait 1 second
Attempt 3: wait 2 seconds
Attempt 4: wait 4 seconds
Max attempts: 4
Jitter: ±30% added to wait time (prevents thundering herd on retries)
```
**Applied to:** Payment gateway calls, Kafka produce failures, inter-service HTTP calls

### 5.2 Circuit Breaker Pattern
Implemented using a **state machine** (library: `opossum` in Node.js):

```
States:
  CLOSED   → normal operation, requests pass through
  OPEN     → failure threshold exceeded; requests fail fast (no call made)
  HALF-OPEN → probe request allowed; if successful, return to CLOSED

Configuration:
  Failure threshold:  50% of requests in a 10-second window fail
  Timeout:            3 seconds per request
  Reset timeout:      30 seconds before moving CLOSED → HALF-OPEN
```

**Benefit:** If Payment Service is down, Order Service opens the circuit and immediately returns a "payment unavailable" error rather than queuing thousands of waiting requests.

### 5.3 Saga Compensating Transactions
When order placement fails mid-flow, we execute compensation:

| Step Failed           | Compensating Action                         |
|-----------------------|---------------------------------------------|
| Payment failed        | Release inventory reservation               |
| Inventory check fails | Return 409, no payment initiated            |
| Notification fails    | Retry silently (fire-and-forget, non-critical)|
| Order DB write fails  | Release inventory + refund via gateway API  |

### 5.4 Dead Letter Queue (DLQ)
Kafka messages that fail processing after all retries go to a **Dead Letter Topic**:
- Alerts sent to on-call engineer
- Messages replayed manually after root cause is fixed
- Prevents a single bad message from blocking the entire consumer

### 5.5 Graceful Degradation
| Service Down        | User Experience                                             |
|---------------------|-------------------------------------------------------------|
| Search Service      | Fallback to basic DB query (slower, but functional)         |
| Notification Svc    | Orders still placed; user gets notification later           |
| Redis Cache         | Fall through to DB (higher latency, but correct)            |
| Analytics Service   | No user-facing impact; data re-processed from Kafka replay  |

---

## 6. Identified Bottlenecks & Optimizations

### Bottleneck 1: Inventory Race Condition During Flash Sale (CRITICAL)
**Problem:** 10,000 users simultaneously trying to buy the last 100 units. Without protection, 10,000 DB writes may all see stock = 100 and proceed, resulting in 10,000 orders for 100 items (overselling).

**Solution:**
1. **Redis DECR atomic operation** — use `DECR inventory:{variantId}` in Redis (atomic, single-threaded in Redis). If result < 0, reject the request. No DB hit until purchase is confirmed.
2. **Lua script in Redis** to atomically check-and-decrement in one operation:
   ```lua
   local stock = redis.call('GET', KEYS[1])
   if tonumber(stock) >= tonumber(ARGV[1]) then
     return redis.call('DECRBY', KEYS[1], ARGV[1])
   else
     return -1
   end
   ```
3. Confirmed purchases then persist to PostgreSQL asynchronously via Kafka

---

### Bottleneck 2: Product Search Latency Under Heavy Load
**Problem:** Full-text search queries against PostgreSQL will degrade significantly at 10K+ QPS.

**Solution:**
- Offload all search to **Elasticsearch** (purpose-built for full-text search at scale)
- Add **Redis caching** for top 1000 most-searched queries (covers ~80% of traffic — Pareto principle)
- Pre-compute faceted counts (by category/brand) periodically rather than per-request

---

### Bottleneck 3: Order Service Becomes a God Service
**Problem:** Order Service orchestrating inventory + payment + notifications creates a tight coupling bottleneck.

**Solution:**
- Switch from **orchestration** (Order Service calls everyone) to **choreography** (services react to events)
- Each service subscribes to Kafka topics and acts independently
- Order Service only emits `order.initiated` and waits for confirmation events

---

### Bottleneck 4: Single PostgreSQL Write Node
**Problem:** All writes (orders, inventory updates, payments) go to one DB node. At scale this becomes a bottleneck.

**Solutions (applied progressively):**
1. **Connection pooling** via PgBouncer (immediate, reduces connection overhead)
2. **Vertical scaling** of primary instance (temporary relief)
3. **CQRS** (Command Query Responsibility Segregation): separate write model (PostgreSQL) from read model (read replicas + Elasticsearch)
4. **Sharding** by user_id (long-term, when data exceeds 500GB)