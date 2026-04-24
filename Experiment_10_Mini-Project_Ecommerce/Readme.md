# ShopNow — E-Commerce Inventory & Order System
### System Design — Lab Mini Project

> A production-grade, scalable architecture for an Amazon/Flipkart-like e-commerce platform.
> Designed as part of the System Design course lab project.

---

## Project Overview

ShopNow is a microservices-based e-commerce platform supporting end-to-end product browsing, cart management, inventory tracking, order placement, and payment processing.

The system is designed to handle:
- **1 million daily active users** under normal conditions
- **50,000 concurrent users** during flash sales
- **99.99% availability** (< 52 minutes downtime per year)
- Sub-100ms product search latency (P99)

This design document covers the full architectural thinking — from high-level system decomposition down to individual class designs, API contracts, and scaling strategies — simulating how a system like this would be designed at a product-based company.

---

## Repository Structure

```
/ecommerce-system-design
├── HLD.md        ← High-Level Design (Architecture, Requirements, CAP Theorem)
├── LLD.md        ← Low-Level Design (Class Diagrams, DB Schema, Sequence Diagrams)
├── api.md        ← API Design (Endpoints, Idempotency, Rate Limiting)
├── scaling.md    ← Scalability & Reliability (Caching, Sharding, Failure Handling)
└── README.md     ← This file
```

---

## Assumptions

The following assumptions were made during design:

1. **Single seller model** — Only one seller (the platform itself). Multi-seller marketplace onboarding is out of scope.
2. **India-first deployment** — Primary region is `ap-south-1` (Mumbai); latency targets are for Indian users.
3. **Synchronous payment** — Payment is confirmed synchronously via webhook before the order is marked CONFIRMED. Real-time payment completion expected within 60 seconds.
4. **Logistics abstracted** — Shipping and delivery tracking are handled by a third-party logistics provider accessed via a simple API wrapper. Internal logistics management is not in scope.
5. **Currency is INR** — Multi-currency support is a future improvement.
6. **No real-time bidding / dynamic pricing** — Prices are set by admins and updated in batch. Flash sale prices are pre-configured.
7. **Users are authenticated** — Cart, Order, and Inventory write operations require a logged-in user. Guest checkout is not supported in v1.
8. **Monorepo structure** — All microservices live in one repository for simplicity during this phase.

---

## Tech Stack

| Layer                     | Technology              | Justification                                                                                 |
|---------------------------|-------------------------|-----------------------------------------------------------------------------------------------|
| **Backend Services**      | Node.js (Express)       | Non-blocking I/O makes it ideal for high-concurrency API workloads. Large npm ecosystem.      |
| **Primary Database**      | PostgreSQL              | ACID compliance is non-negotiable for financial transactions (orders, payments, inventory).    |
| **Cache**                 | Redis                   | Sub-millisecond reads, native support for distributed locking (SETNX), pub/sub, and atomic ops (DECR). |
| **Message Broker**        | Apache Kafka            | High-throughput, durable, replayable event log. Ideal for decoupling microservices at scale. |
| **Search Engine**         | Elasticsearch           | Purpose-built for full-text search and faceted filtering; handles 10K+ QPS far better than SQL LIKE queries. |
| **API Gateway**           | Kong                    | Open-source, plugin-based; handles rate limiting, JWT auth, and routing without custom code. |
| **Object Storage**        | AWS S3                  | Cheap, infinitely scalable, and integrates natively with CloudFront CDN.                     |
| **CDN**                   | AWS CloudFront          | Global edge caching for static assets and product images; reduces origin load dramatically.  |
| **Container Orchestration** | Kubernetes (EKS)      | Auto-scaling, self-healing pods, rolling deployments, and service discovery built-in.         |
| **Service Mesh**          | Istio                   | Handles inter-service mTLS, circuit breaking, and canary deployments at the infrastructure level. |
| **Monitoring**            | Prometheus + Grafana    | Industry-standard metrics collection and visualization. Alertmanager for on-call alerts.      |
| **Distributed Tracing**   | Jaeger                  | Traces requests across microservices; critical for debugging latency issues in distributed systems. |
| **CI/CD**                 | GitHub Actions          | Tight VCS integration, simple YAML-based pipelines, free for standard usage.                 |

---

## Trade-offs Taken

### 1. Microservices over Monolith
**Decision:** Break the system into 6+ services from the start.
- **Pro:** Independent scaling, independent deployments, team autonomy, fault isolation.
- **Con:** Operational complexity (service discovery, distributed tracing, network latency between services). Requires a mature DevOps culture.
- **Justification:** The scale targets (1M DAU, flash sale spikes) make a monolith's single deployment unit a scaling bottleneck. Microservices are worth the overhead here.

### 2. PostgreSQL over NoSQL for Core Data
**Decision:** Use relational PostgreSQL for orders, inventory, and payments.
- **Pro:** ACID transactions, foreign key integrity, rich query language, mature tooling.
- **Con:** Harder to scale horizontally than NoSQL; sharding is complex.
- **Justification:** Orders and payments are financial data. Consistency and correctness take priority over horizontal scalability. Read replicas handle the read load; sharding deferred until truly necessary.

### 3. Choreography-based Saga over 2-Phase Commit (2PC)
**Decision:** Services react to Kafka events rather than using distributed transactions.
- **Pro:** Loose coupling, no single coordinator bottleneck, services can fail independently.
- **Con:** Eventual consistency is harder to reason about; compensating transactions must be carefully designed.
- **Justification:** 2PC locks resources across multiple databases and becomes a distributed deadlock risk at scale. The saga pattern is the industry standard for distributed transactions.

### 4. Elasticsearch for Search (Dual Write)
**Decision:** Maintain a separate Elasticsearch index alongside PostgreSQL.
- **Pro:** Full-text search, faceted filtering, and ranking at millisecond latency.
- **Con:** Two data stores to keep in sync; potential for brief inconsistency between PostgreSQL and Elasticsearch.
- **Justification:** PostgreSQL full-text search degrades significantly beyond a few million rows. The slight staleness in search results (seconds) is acceptable. Kafka CDC via Debezium ensures eventual sync.

### 5. Redis for Inventory Locking over DB-level Locking
**Decision:** Use Redis atomic operations (DECR + Lua script) for inventory reservation instead of PostgreSQL `SELECT FOR UPDATE`.
- **Pro:** Orders of magnitude faster; no DB connection held during lock duration; scales horizontally.
- **Con:** If Redis fails, inventory check must fall back to DB, adding latency.
- **Justification:** During a flash sale, 10,000+ users hit inventory simultaneously. Holding a PostgreSQL row lock for each would serialize all requests and crater throughput. Redis handles this atomically in microseconds.

---

## Future Improvements

### Short-Term (3–6 months)
- [ ] **Multi-seller marketplace**: Seller onboarding portal, per-seller inventory management, commission tracking
- [ ] **Guest checkout**: Allow purchases without registration (email-based order tracking)
- [ ] **Wishlist**: Save products for later, notified on price drops
- [ ] **Coupon Engine v2**: Percentage off, buy-1-get-1, category-level discounts with proper rule engine
- [ ] **Product recommendations**: Collaborative filtering ("Customers also bought...")

### Medium-Term (6–12 months)
- [ ] **Multi-region deployment**: Active-active in Mumbai + Hyderabad for lower latency and disaster recovery
- [ ] **Elasticsearch ML ranking**: Use learning-to-rank to personalize search results per user
- [ ] **A/B Testing framework**: Test pricing strategies, homepage layouts, checkout flows
- [ ] **Fraud Detection**: ML-based model on transaction patterns; flag suspicious orders for review

### Long-Term (12+ months)
- [ ] **Global expansion**: Multi-currency, multi-language, international shipping partners
- [ ] **Real-time inventory sync with offline stores**: Omni-channel inventory management
- [ ] **Predictive restocking**: ML model to auto-raise purchase orders before stock runs out
- [ ] **Event sourcing for Orders**: Store all state transitions as immutable events, enable full audit trail and replay

---

## Author
> System Design Lab Project — Experiment 10
> E-Commerce Inventory & Order System