# API Design — E-Commerce Inventory & Order System

---

## 1. Versioning Strategy

All APIs are versioned via the URL path:
```
https://api.shopnow.com/v1/...
```

- **v1** is the current stable version
- Breaking changes result in a new version (`/v2/`)
- Deprecated versions remain live for 6 months with a `Deprecation` response header
- Non-breaking additions (new optional fields) are added to the same version

---

## 2. Authentication

All protected endpoints require a **Bearer token** in the `Authorization` header:
```
Authorization: Bearer <JWT_ACCESS_TOKEN>
```
- Access Token TTL: 15 minutes
- Refresh Token TTL: 7 days (stored in HttpOnly cookie)
- Token rotation on every refresh

---

## 3. Standard Response Format

### Success
```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 150
  }
}
```

### Error
```json
{
  "success": false,
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Only 2 units available for the selected variant",
    "details": {}
  }
}
```

---

## 4. REST API Endpoints

---

### 4.1 Auth Endpoints

#### POST /v1/auth/register
Register a new user.

**Request:**
```json
{
  "email": "john@example.com",
  "password": "SecurePass@123",
  "full_name": "John Doe",
  "phone": "9876543210"
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "user_id": "a3b1c2d4-...",
    "email": "john@example.com",
    "full_name": "John Doe"
  }
}
```

**Errors:**
| Code | Meaning |
|------|---------|
| 400  | Validation failed (weak password, invalid email) |
| 409  | Email already registered |

---

#### POST /v1/auth/login
Login and receive tokens.

**Request:**
```json
{
  "email": "john@example.com",
  "password": "SecurePass@123"
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGci...",
    "expires_in": 900
  }
}
```
> Refresh token is set as an **HttpOnly cookie**.

**Errors:**
| Code | Meaning |
|------|---------|
| 401  | Invalid credentials |
| 429  | Too many login attempts (rate limited) |

---

#### POST /v1/auth/refresh
Refresh the access token.

**Request:** No body (refresh token read from cookie)

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGci...",
    "expires_in": 900
  }
}
```

---

#### POST /v1/auth/logout
Invalidate refresh token.

**Response: 204 No Content**

---

### 4.2 Product Endpoints

#### GET /v1/products
List and search products with filtering.

**Query Parameters:**
| Param      | Type    | Description                        |
|------------|---------|------------------------------------|
| q          | String  | Search keyword                     |
| category   | String  | Category slug                      |
| brand      | String  | Brand name                         |
| min_price  | Number  | Minimum price filter               |
| max_price  | Number  | Maximum price filter               |
| sort       | String  | `price_asc`, `price_desc`, `rating`|
| page       | Int     | Page number (default: 1)           |
| limit      | Int     | Items per page (default: 20, max: 100)|

**Response: 200 OK**
```json
{
  "success": true,
  "data": [
    {
      "product_id": "p1...",
      "name": "Nike Air Max 270",
      "base_price": 8995.00,
      "avg_rating": 4.3,
      "thumbnail_url": "https://cdn.shopnow.com/products/p1/thumb.jpg",
      "category": "Footwear",
      "in_stock": true
    }
  ],
  "meta": { "page": 1, "limit": 20, "total": 312 }
}
```

---

#### GET /v1/products/:productId
Get full product details.

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "product_id": "p1...",
    "name": "Nike Air Max 270",
    "description": "...",
    "base_price": 8995.00,
    "avg_rating": 4.3,
    "category": { "id": "...", "name": "Footwear" },
    "variants": [
      {
        "variant_id": "v1...",
        "sku": "NIKE-AM270-BLK-10",
        "size": "10",
        "color": "Black",
        "price": 8995.00,
        "images": ["https://cdn.../v1-front.jpg"],
        "available_qty": 14
      }
    ]
  }
}
```

**Errors:**
| Code | Meaning |
|------|---------|
| 404  | Product not found |

---

### 4.3 Cart Endpoints

#### GET /v1/cart
Get current user's cart. *(Auth required)*

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "cart_id": "c1...",
    "items": [
      {
        "cart_item_id": "ci1...",
        "variant_id": "v1...",
        "product_name": "Nike Air Max 270",
        "size": "10",
        "color": "Black",
        "quantity": 2,
        "unit_price": 8995.00,
        "line_total": 17990.00
      }
    ],
    "subtotal": 17990.00
  }
}
```

---

#### POST /v1/cart/items
Add item to cart. *(Auth required)*

**Request:**
```json
{
  "variant_id": "v1...",
  "quantity": 2
}
```

**Response: 200 OK** (returns updated cart)

**Errors:**
| Code | Meaning |
|------|---------|
| 400  | Invalid quantity (< 1 or > 10) |
| 404  | Variant not found |
| 409  | Requested quantity exceeds available stock |

---

#### PATCH /v1/cart/items/:cartItemId
Update quantity of a cart item. *(Auth required)*

**Request:**
```json
{ "quantity": 3 }
```

**Response: 200 OK**

---

#### DELETE /v1/cart/items/:cartItemId
Remove item from cart. *(Auth required)*

**Response: 204 No Content**

---

### 4.4 Order Endpoints

#### POST /v1/orders
Place an order (checkout). *(Auth required)*

**Request:**
```json
{
  "address_id": "a1...",
  "coupon_code": "SAVE10",
  "payment_method": "UPI"
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "order_id": "o1...",
    "status": "PENDING",
    "total_amount": 16191.00,
    "discount_amount": 1799.00,
    "payment_url": "https://razorpay.com/pay/..."
  }
}
```

**Errors:**
| Code | Meaning |
|------|---------|
| 400  | Empty cart / invalid address |
| 409  | Stock unavailable for one or more items |
| 422  | Invalid coupon code or expired |

> This endpoint is **idempotent** — see Section 6.

---

#### GET /v1/orders
List all orders for logged-in user. *(Auth required)*

**Query Parameters:** `status`, `page`, `limit`

**Response: 200 OK**
```json
{
  "success": true,
  "data": [
    {
      "order_id": "o1...",
      "status": "DELIVERED",
      "total_amount": 16191.00,
      "placed_at": "2025-04-01T10:30:00Z",
      "item_count": 2
    }
  ],
  "meta": { "page": 1, "total": 8 }
}
```

---

#### GET /v1/orders/:orderId
Get full order details. *(Auth required)*

**Response: 200 OK** — includes all order items, payment status, and tracking info.

---

#### POST /v1/orders/:orderId/cancel
Cancel an order. *(Auth required)*

**Request:**
```json
{ "reason": "Changed my mind" }
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": { "order_id": "o1...", "status": "CANCELLED" }
}
```

**Errors:**
| Code | Meaning |
|------|---------|
| 400  | Order cannot be cancelled (already shipped) |
| 403  | Order does not belong to this user |

---

### 4.5 Inventory Endpoints *(Admin only)*

#### GET /v1/inventory/:variantId
Check stock level for a variant across warehouses.

**Response: 200 OK**
```json
{
  "success": true,
  "data": [
    {
      "warehouse_id": "w1...",
      "location": "Mumbai",
      "quantity_on_hand": 50,
      "quantity_reserved": 8,
      "available": 42
    }
  ]
}
```

---

#### PATCH /v1/inventory/:variantId
Update stock levels. *(Admin only)*

**Request:**
```json
{
  "warehouse_id": "w1...",
  "adjustment": 100,
  "reason": "New stock received"
}
```

**Response: 200 OK**

---

## 5. HTTP Status Codes Used

| Code | Meaning                                             |
|------|-----------------------------------------------------|
| 200  | Success (GET, PATCH)                                |
| 201  | Resource created (POST — register, order)           |
| 204  | Success, no body (DELETE, logout)                   |
| 400  | Bad request / validation error                      |
| 401  | Unauthenticated (missing/expired token)             |
| 403  | Forbidden (authenticated but not authorized)        |
| 404  | Resource not found                                  |
| 409  | Conflict (duplicate email, out of stock)            |
| 422  | Unprocessable entity (invalid business logic)       |
| 429  | Too many requests (rate limited)                    |
| 500  | Internal server error                               |
| 503  | Service unavailable (circuit breaker open)          |

---

## 6. Idempotency Handling

### Problem
During checkout, network failures can cause clients to retry the `POST /v1/orders` request. Without idempotency, the same order gets placed multiple times and payment charged twice.

### Solution
Clients must send a unique `Idempotency-Key` header with every checkout request:
```
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

**Server Behavior:**
1. On first request: process normally, store `(idempotency_key → response)` in Redis (TTL: 24 hours)
2. On retry with same key: return cached response immediately without reprocessing
3. `idempotency_key` is also stored in the `payments` table (UNIQUE constraint) as a second guard

**Scope:** Idempotency keys also applied to:
- `POST /v1/cart/items` — prevents duplicate add-to-cart on retry
- Payment webhook endpoints — prevents double-crediting on duplicate webhook delivery

---

## 7. Rate Limiting Strategy

Rate limiting is enforced at the **API Gateway (Kong)** level using a token bucket algorithm backed by Redis.

### Limits by Endpoint Type

| Endpoint Group       | Limit             | Window   | Scope       |
|----------------------|-------------------|----------|-------------|
| Auth (login/register)| 10 requests       | 1 minute | Per IP      |
| Product Search/View  | 200 requests      | 1 minute | Per User    |
| Cart Operations      | 60 requests       | 1 minute | Per User    |
| Place Order          | 5 requests        | 1 minute | Per User    |
| Admin APIs           | 500 requests      | 1 minute | Per API Key |

### Rate Limit Response Headers
Every API response includes these headers so clients can self-regulate:
```
X-RateLimit-Limit: 200
X-RateLimit-Remaining: 187
X-RateLimit-Reset: 1714042860
```

### When Limit is Exceeded
```
HTTP/1.1 429 Too Many Requests
Retry-After: 45

{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please retry after 45 seconds."
  }
}
```

### Additional Protection
- **Burst allowance**: Short traffic spikes (2× limit) allowed for up to 5 seconds
- **IP blocklist**: IPs with repeated 429s escalated to temporary block (15 min)
- **Bot detection**: Unusual patterns (all requests exactly 1s apart) flagged and throttled