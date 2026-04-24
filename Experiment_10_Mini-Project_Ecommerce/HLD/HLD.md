# High-Level Design (HLD) — E-Commerce Inventory & Order System

---

## 1. Functional Requirements

### Core Features
- **User Management**: Registration, login, profile management
- **Product Catalog**: Browse, search, and filter products with real-time inventory counts
- **Inventory Management**: Stock tracking, low-stock alerts, warehouse mapping
- **Cart & Checkout**: Add to cart, apply coupons, place orders
- **Order Management**: Create, track, cancel, and return orders
- **Payment Processing**: Integration with payment gateways (Razorpay / Stripe)
- **Notifications**: Email/SMS/push for order updates
- **Reviews & Ratings**: Post-purchase product reviews

### Out of Scope (for this design)
- Seller onboarding portal (assumed pre-seeded catalog)
- Logistics partner API integration (abstracted as a service)

---

## 2. Non-Functional Requirements

| Property       | Target                                          |
|----------------|-------------------------------------------------|
| Availability   | 99.99% uptime (< 52 min downtime/year)         |
| Latency        | Product search < 100ms (P99); Checkout < 500ms |
| Scalability    | Handle 1M DAU, 50K concurrent users during sale |
| Consistency    | Strong consistency for inventory & payments     |
| Durability     | Zero data loss for orders and transactions      |
| Security       | OWASP Top 10 compliance, PCI-DSS for payments  |

---

## 3. System Architecture Diagram

```
                          ┌──────────────────────────────────────────────┐
                          │               CLIENT LAYER                   │
                          │    Web Browser / Mobile App / Third-party    │
                          └────────────────────┬─────────────────────────┘
                                               │ HTTPS
                          ┌────────────────────▼─────────────────────────┐
                          │              CDN (CloudFront)                 │
                          │  Static assets: images, JS, CSS bundles       │
                          └────────────────────┬─────────────────────────┘
                                               │
                          ┌────────────────────▼─────────────────────────┐
                          │           API Gateway (Kong / AWS API GW)     │
                          │  Rate Limiting · Auth · SSL Termination       │
                          │  Request Routing · Load Balancing             │
                          └──────┬──────────┬──────────┬──────────┬──────┘
                                 │          │          │          │
             ┌───────────────────▼──┐  ┌───▼───┐  ┌──▼────┐  ┌──▼──────────┐
             │   User Service       │  │Product│  │Order  │  │ Inventory   │
             │  (Auth / Profile)    │  │Service│  │Service│  │ Service     │
             └──────────┬───────────┘  └───┬───┘  └──┬────┘  └──┬──────────┘
                        │                  │          │           │
             ┌──────────▼──────────────────▼──────────▼───────────▼──────────┐
             │                     MESSAGE BROKER (Kafka)                     │
             │   Topics: order.placed  |  inventory.updated  |  payment.done  │
             └──────────┬──────────────────┬──────────┬───────────┬───────────┘
                        │                  │          │           │
             ┌──────────▼──┐  ┌────────────▼──┐  ┌───▼──────┐  ┌▼──────────────┐
             │Notification │  │ Search Service │  │ Payment  │  │Analytics/     │
             │  Service    │  │ (Elasticsearch)│  │ Service  │  │Reporting Svc  │
             └─────────────┘  └───────────────┘  └──────────┘  └───────────────┘

             ┌──────────────────────────────────────────────────────────────────┐
             │                        DATA LAYER                               │
             │                                                                 │
             │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐ │
             │  │  PostgreSQL  │  │    Redis      │  │   Elasticsearch       │ │
             │  │  (Primary DB)│  │  (Cache/Lock) │  │  (Search Index)       │ │
             │  └──────────────┘  └──────────────┘  └───────────────────────┘ │
             │                                                                 │
             │  ┌──────────────┐  ┌──────────────┐                            │
             │  │   S3/Blob    │  │  TimescaleDB  │                            │
             │  │  (Images)    │  │  (Analytics)  │                            │
             │  └──────────────┘  └──────────────┘                            │
             └──────────────────────────────────────────────────────────────────┘
```

---

## 4. High-Level Components

### 4.1 API Gateway
- Single entry point for all client traffic
- Responsibilities: SSL termination, JWT validation, rate limiting, request routing
- Routes requests to appropriate microservices

### 4.2 User Service
- Handles registration, login (JWT-based), profile updates
- Stores user data in PostgreSQL
- Issues access tokens (short-lived, 15min) + refresh tokens (7 days)

### 4.3 Product Service
- Manages product catalog: metadata, pricing, images
- Product images stored in S3; URLs served via CDN
- Product data cached in Redis (TTL: 5 min) to handle high read traffic

### 4.4 Inventory Service
- Tracks stock levels per product per warehouse
- Uses **pessimistic locking** (Redis distributed lock) during checkout to prevent overselling
- Publishes `inventory.updated` events to Kafka after every stock change

### 4.5 Order Service
- Orchestrates the checkout flow: validate cart → reserve inventory → initiate payment → confirm order
- Implements the **Saga pattern** for distributed transaction management
- Stores orders in PostgreSQL with ACID guarantees

### 4.6 Payment Service
- Integrates with external gateway (Razorpay)
- Handles webhook callbacks for payment confirmation/failure
- Idempotent by design — duplicate webhooks are safely ignored

### 4.7 Search Service
- Elasticsearch index built from the product catalog
- Supports full-text search, faceted filters (price range, category, brand, rating)
- Re-indexed asynchronously when product data changes via Kafka consumer

### 4.8 Notification Service
- Consumes events from Kafka (order placed, shipped, delivered)
- Sends emails (AWS SES), SMS (Twilio), and push notifications (FCM)
- Fire-and-forget; failures retried with exponential backoff

### 4.9 CDN
- Caches static assets and product images globally
- Reduces origin server load and improves latency for geo-distributed users

---

## 5. Data Flow

### 5.1 Product Search Flow
```
User → API Gateway → Search Service (Elasticsearch query)
                   ← Returns ranked product list with metadata
```

### 5.2 Place Order Flow
```
1. User submits checkout → API Gateway → Order Service
2. Order Service calls Inventory Service → acquires Redis lock → decrements stock
3. Order Service calls Payment Service → creates payment intent
4. Payment Gateway processes payment → sends webhook → Payment Service
5. Payment Service publishes `payment.done` to Kafka
6. Order Service consumes event → marks order CONFIRMED → stores in PostgreSQL
7. Notification Service consumes event → sends confirmation email/SMS
8. Inventory Service commits stock deduction permanently
```

### 5.3 Failure in Order Flow (Saga Rollback)
```
If payment fails:
  - Payment Service publishes `payment.failed`
  - Inventory Service consumes event → releases lock → restores stock
  - Order Service marks order CANCELLED
  - Notification Service sends failure notification
```

---

## 6. CAP Theorem Considerations

In a distributed system, we cannot guarantee all three of **Consistency**, **Availability**, and **Partition Tolerance** simultaneously.

| Component          | Choice  | Justification                                                         |
|--------------------|---------|-----------------------------------------------------------------------|
| Inventory / Orders | **CP**  | We cannot oversell; strong consistency is mandatory. Short downtime > wrong data |
| Product Catalog    | **AP**  | Slightly stale product info (price cache miss) is acceptable; availability is critical |
| Search Service     | **AP**  | Search showing slightly stale results is acceptable; must stay available |
| User Auth          | **CP**  | Stale tokens = security risk; consistency required                    |
| Notifications      | **AP**  | Eventual delivery is fine; notification delay is acceptable           |

> **Summary**: We lean **CP** where money or stock is involved, and **AP** for read-heavy, low-risk services. This hybrid approach balances user experience with business correctness.

---

## 7. Technology Choices Summary

| Layer              | Technology          | Reason                                           |
|--------------------|---------------------|--------------------------------------------------|
| API Gateway        | Kong                | Open-source, plugin ecosystem, rate limiting OOB |
| Backend Services   | Node.js (Express)   | High I/O throughput, large ecosystem             |
| Primary DB         | PostgreSQL          | ACID, relational integrity for orders/inventory  |
| Cache              | Redis               | Sub-ms reads, distributed locking support        |
| Message Broker     | Apache Kafka        | High throughput, durable event log, replay-able  |
| Search             | Elasticsearch       | Full-text + faceted search at scale              |
| Object Storage     | AWS S3              | Cheap, durable, CDN-compatible                   |
| Container Orchestration | Kubernetes    | Auto-scaling, self-healing, service discovery    |