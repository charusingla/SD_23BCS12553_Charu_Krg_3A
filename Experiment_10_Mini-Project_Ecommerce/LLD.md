# Low-Level Design (LLD) — E-Commerce Inventory & Order System

---

## 1. Class Diagrams (OOP + SOLID Principles)

```
┌─────────────────────────────────────────────────────────────────────┐
│                          USER DOMAIN                                │
│                                                                     │
│  ┌──────────────────┐         ┌──────────────────────────────────┐  │
│  │      User        │         │          Address                 │  │
│  │──────────────────│         │──────────────────────────────────│  │
│  │ - userId: UUID   │1      * │ - addressId: UUID                │  │
│  │ - email: String  │─────────│ - userId: UUID                   │  │
│  │ - passwordHash   │         │ - street, city, state, pin       │  │
│  │ - role: Enum     │         │ - isDefault: Boolean             │  │
│  │──────────────────│         └──────────────────────────────────┘  │
│  │ + register()     │                                               │
│  │ + login()        │                                               │
│  │ + updateProfile()│                                               │
│  └──────────────────┘                                               │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        PRODUCT DOMAIN                               │
│                                                                     │
│  ┌───────────────────┐     ┌─────────────────┐  ┌───────────────┐  │
│  │     Product       │     │    Category     │  │    Review     │  │
│  │───────────────────│ *─1 │─────────────────│  │───────────────│  │
│  │ - productId: UUID │     │ - categoryId    │  │ - reviewId    │  │
│  │ - name: String    │     │ - name: String  │  │ - userId      │  │
│  │ - description     │     │ - parentId      │  │ - productId   │  │
│  │ - basePrice       │     └─────────────────┘  │ - rating 1-5  │  │
│  │ - categoryId      │                           │ - comment     │  │
│  │ - imageUrls[]     │     ┌─────────────────┐  └───────────────┘  │
│  │───────────────────│ 1─* │  ProductVariant │                     │
│  │ + getDetails()    │     │─────────────────│                     │
│  │ + updatePrice()   │     │ - variantId     │                     │
│  │ + addImage()      │     │ - productId     │                     │
│  └───────────────────┘     │ - sku: String   │                     │
│                             │ - size, color   │                     │
│                             │ - priceModifier │                     │
│                             └─────────────────┘                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                      INVENTORY DOMAIN                               │
│                                                                     │
│  ┌───────────────────┐     ┌─────────────────────────────────────┐  │
│  │    Warehouse      │     │         InventoryItem               │  │
│  │───────────────────│     │─────────────────────────────────────│  │
│  │ - warehouseId     │1  * │ - inventoryId: UUID                 │  │
│  │ - location: String│─────│ - variantId: UUID                   │  │
│  │ - capacity: Int   │     │ - warehouseId: UUID                 │  │
│  │───────────────────│     │ - quantityOnHand: Int               │  │
│  │ + getStock()      │     │ - quantityReserved: Int             │  │
│  │ + updateCapacity()│     │ - reorderLevel: Int                 │  │
│  └───────────────────┘     │─────────────────────────────────────│  │
│                             │ + reserveStock(qty): Boolean        │  │
│                             │ + releaseReservation(qty)           │  │
│                             │ + commitDeduction(qty)              │  │
│                             │ + getAvailableQty(): Int            │  │
│                             └─────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        ORDER DOMAIN                                 │
│                                                                     │
│  ┌──────────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │      Cart        │    │  CartItem    │    │      Order       │  │
│  │──────────────────│1 * │──────────────│    │──────────────────│  │
│  │ - cartId: UUID   │────│ - cartItemId │    │ - orderId: UUID  │  │
│  │ - userId: UUID   │    │ - variantId  │    │ - userId: UUID   │  │
│  │ - createdAt      │    │ - quantity   │    │ - status: Enum   │  │
│  │──────────────────│    │ - priceSnap  │    │ - totalAmount    │  │
│  │ + addItem()      │    └──────────────┘    │ - addressId      │  │
│  │ + removeItem()   │                        │ - placedAt       │  │
│  │ + getTotal()     │                        │──────────────────│  │
│  │ + clearCart()    │                        │ + cancel()       │  │
│  └──────────────────┘                        │ + getStatus()    │  │
│                                              └──────────────────┘  │
│   OrderStatus: PENDING → CONFIRMED → PROCESSING → SHIPPED →         │
│                DELIVERED → [RETURN_REQUESTED → RETURNED]            │
│                          ↘ CANCELLED                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Database Schema

### Users Table
```sql
CREATE TABLE users (
    user_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email       VARCHAR(255) UNIQUE NOT NULL,
    phone       VARCHAR(15),
    password_hash VARCHAR(255) NOT NULL,
    full_name   VARCHAR(255),
    role        VARCHAR(20) DEFAULT 'CUSTOMER', -- CUSTOMER | ADMIN
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_users_email ON users(email);
```

### Addresses Table
```sql
CREATE TABLE addresses (
    address_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
    street      VARCHAR(255) NOT NULL,
    city        VARCHAR(100) NOT NULL,
    state       VARCHAR(100) NOT NULL,
    pincode     VARCHAR(10) NOT NULL,
    country     VARCHAR(100) DEFAULT 'India',
    is_default  BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_addresses_user ON addresses(user_id);
```

### Categories Table
```sql
CREATE TABLE categories (
    category_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(100) NOT NULL,
    parent_id   UUID REFERENCES categories(category_id), -- supports nested categories
    slug        VARCHAR(100) UNIQUE NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW()
);
```

### Products Table
```sql
CREATE TABLE products (
    product_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(500) NOT NULL,
    description TEXT,
    base_price  NUMERIC(12, 2) NOT NULL,
    category_id UUID NOT NULL REFERENCES categories(category_id),
    brand       VARCHAR(100),
    is_active   BOOLEAN DEFAULT TRUE,
    avg_rating  NUMERIC(3,2) DEFAULT 0.00,
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_brand ON products(brand);
```

### Product Variants Table
```sql
CREATE TABLE product_variants (
    variant_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id      UUID NOT NULL REFERENCES products(product_id) ON DELETE CASCADE,
    sku             VARCHAR(100) UNIQUE NOT NULL,
    size            VARCHAR(50),
    color           VARCHAR(50),
    price_modifier  NUMERIC(10,2) DEFAULT 0.00, -- added to base_price
    image_urls      TEXT[], -- array of S3 URLs
    created_at      TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_variants_product ON product_variants(product_id);
CREATE INDEX idx_variants_sku ON product_variants(sku);
```

### Inventory Table
```sql
CREATE TABLE inventory (
    inventory_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    variant_id          UUID NOT NULL REFERENCES product_variants(variant_id),
    warehouse_id        UUID NOT NULL REFERENCES warehouses(warehouse_id),
    quantity_on_hand    INT NOT NULL DEFAULT 0,
    quantity_reserved   INT NOT NULL DEFAULT 0,
    reorder_level       INT NOT NULL DEFAULT 10,
    updated_at          TIMESTAMP DEFAULT NOW(),
    UNIQUE(variant_id, warehouse_id)
);
CREATE INDEX idx_inventory_variant ON inventory(variant_id);

-- Computed available quantity:
-- available = quantity_on_hand - quantity_reserved
```

### Carts Table
```sql
CREATE TABLE carts (
    cart_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID UNIQUE NOT NULL REFERENCES users(user_id),
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE cart_items (
    cart_item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cart_id      UUID NOT NULL REFERENCES carts(cart_id) ON DELETE CASCADE,
    variant_id   UUID NOT NULL REFERENCES product_variants(variant_id),
    quantity     INT NOT NULL CHECK (quantity > 0),
    price_snapshot NUMERIC(12,2) NOT NULL, -- locked price at add-to-cart time
    added_at     TIMESTAMP DEFAULT NOW(),
    UNIQUE(cart_id, variant_id)
);
```

### Orders Table
```sql
CREATE TABLE orders (
    order_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(user_id),
    address_id      UUID NOT NULL REFERENCES addresses(address_id),
    status          VARCHAR(30) NOT NULL DEFAULT 'PENDING',
    total_amount    NUMERIC(12,2) NOT NULL,
    discount_amount NUMERIC(12,2) DEFAULT 0.00,
    coupon_code     VARCHAR(50),
    placed_at       TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);

CREATE TABLE order_items (
    order_item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id      UUID NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
    variant_id    UUID NOT NULL REFERENCES product_variants(variant_id),
    quantity      INT NOT NULL,
    unit_price    NUMERIC(12,2) NOT NULL,
    total_price   NUMERIC(12,2) NOT NULL
);
CREATE INDEX idx_order_items_order ON order_items(order_id);
```

### Payments Table
```sql
CREATE TABLE payments (
    payment_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id        UUID NOT NULL REFERENCES orders(order_id),
    gateway_txn_id  VARCHAR(255) UNIQUE, -- from Razorpay
    amount          NUMERIC(12,2) NOT NULL,
    currency        VARCHAR(10) DEFAULT 'INR',
    status          VARCHAR(30) DEFAULT 'INITIATED', -- INITIATED | SUCCESS | FAILED | REFUNDED
    method          VARCHAR(50), -- UPI | CARD | NETBANKING | WALLET
    initiated_at    TIMESTAMP DEFAULT NOW(),
    completed_at    TIMESTAMP,
    idempotency_key VARCHAR(255) UNIQUE NOT NULL
);
CREATE INDEX idx_payments_order ON payments(order_id);
```

---

## 3. Relationships & Indexing Summary

| Relationship                         | Type       | Index                          |
|--------------------------------------|------------|--------------------------------|
| User → Addresses                     | 1 to Many  | `addresses.user_id`            |
| Product → Variants                   | 1 to Many  | `product_variants.product_id`  |
| Variant → Inventory (per warehouse)  | 1 to Many  | `inventory.variant_id`         |
| User → Cart (one active cart)        | 1 to 1     | `carts.user_id` (UNIQUE)       |
| Cart → Cart Items                    | 1 to Many  | `cart_items.cart_id`           |
| Order → Order Items                  | 1 to Many  | `order_items.order_id`         |
| Order → Payment                      | 1 to 1     | `payments.order_id`            |

### Key Index Decisions
- `users.email` — indexed for fast login lookup
- `orders.user_id + status` — composite index for "my orders" page filters
- `inventory.variant_id` — critical for stock check during checkout
- `payments.idempotency_key` — prevents duplicate payment records

---

## 4. Sequence Diagrams

### Sequence 1: Place Order (Happy Path)

```
User      API GW     OrderSvc    InventorySvc    PaymentSvc    Kafka
 │           │           │              │              │          │
 │──checkout─►           │              │              │          │
 │           │──validate─►              │              │          │
 │           │           │──reserveStock►              │          │
 │           │           │              │◄─lock(Redis) │          │
 │           │           │              │──success─────►          │
 │           │           │◄─reserved────│              │          │
 │           │           │──createPayment────────────►  │          │
 │           │           │              │              │──webhook─►│
 │           │           │              │              │          │
 │           │           │◄─────────────────payment.done──────────│
 │           │           │──CONFIRM order               │          │
 │           │           │──commitDeduct────────────────►          │
 │◄──orderID─│◄──success─│              │              │          │
 │           │           │──────────────────────────order.confirmed►│
 │           │           │              │              │          │
 │     (Notification Service consumes event → sends email/SMS)   │
```

### Sequence 2: Search & View Product

```
User       API GW     SearchSvc        ProductSvc       Redis Cache
 │            │            │                │                 │
 │──search────►            │                │                 │
 │            │──query─────►                │                 │
 │            │            │──Elasticsearch │                 │
 │            │            │◄─result IDs────│                 │
 │            │◄─product list               │                 │
 │◄──results──│            │                │                 │
 │            │            │                │                 │
 │──view productId──────────────────────────►                 │
 │            │            │                │──checkCache─────►
 │            │            │                │◄─HIT: data──────│
 │◄──product detail──────────────────────────                 │
 │            │            │                │                 │
 │     (Cache MISS path: DB → populate Redis → return)        │
```

---

## 5. Design Patterns Used

### 5.1 Singleton Pattern — Database Connection Pool
The database connection pool (e.g., `pg.Pool` in Node.js) is initialized once and shared across all service modules. A single instance prevents connection exhaustion.

```javascript
// db.js
let pool = null;
class DBPool {
  static getInstance() {
    if (!pool) {
      pool = new Pool({ connectionString: process.env.DB_URL, max: 20 });
    }
    return pool;
  }
}
module.exports = DBPool;
```

### 5.2 Factory Pattern — Notification Dispatcher
Different notification channels (Email, SMS, Push) are created through a factory, keeping the Notification Service decoupled from specific implementations.

```javascript
class NotificationFactory {
  static create(type) {
    switch(type) {
      case 'EMAIL': return new EmailNotifier();  // uses AWS SES
      case 'SMS':   return new SMSNotifier();    // uses Twilio
      case 'PUSH':  return new PushNotifier();   // uses FCM
      default: throw new Error('Unknown notification type');
    }
  }
}
```

### 5.3 Strategy Pattern — Payment Gateway
The Payment Service uses a Strategy interface, making it easy to swap or add gateways without changing core logic.

```javascript
class PaymentContext {
  constructor(strategy) { this.strategy = strategy; }
  execute(payload) { return this.strategy.pay(payload); }
}
// Usage: new PaymentContext(new RazorpayStrategy()).execute(payload)
// Swap: new PaymentContext(new StripeStrategy()).execute(payload)
```

### 5.4 Observer Pattern — Event-Driven via Kafka
Order Service (publisher) emits domain events. Notification Service, Inventory Service, and Analytics Service (observers) react independently — fully decoupled.

### 5.5 Saga Pattern — Distributed Order Transaction
Since we cannot use a single DB transaction across microservices, we use a **Choreography-based Saga**:
- Each service publishes events on success/failure
- Downstream services either proceed or trigger compensating transactions (rollback stock, cancel order)

### 5.6 Repository Pattern — Data Access Abstraction
Each service accesses its DB only through a repository class, keeping business logic isolated from query logic (adheres to **Single Responsibility Principle**).

```javascript
class OrderRepository {
  async create(orderData) { /* SQL INSERT */ }
  async findById(orderId) { /* SQL SELECT */ }
  async updateStatus(orderId, status) { /* SQL UPDATE */ }
}
```

---

## 6. SOLID Principles Applied

| Principle                      | Application in This System                                                  |
|--------------------------------|-----------------------------------------------------------------------------|
| **S** - Single Responsibility  | Each microservice owns one domain (Order Service only manages orders)       |
| **O** - Open/Closed            | NotificationFactory is open to new channel types without modifying existing |
| **L** - Liskov Substitution    | All payment strategies implement the same `PaymentStrategy` interface       |
| **I** - Interface Segregation  | Inventory Service exposes separate interfaces for read vs. write operations |
| **D** - Dependency Inversion   | Services depend on abstractions (interfaces), not concrete DB/queue clients |