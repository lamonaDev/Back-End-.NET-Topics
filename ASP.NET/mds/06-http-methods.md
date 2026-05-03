# HTTP Methods — Real Business Cases

## Overview

HTTP defines a set of request methods (also called HTTP verbs) that indicate the desired action to be performed on a resource. Understanding these methods correctly is fundamental to building RESTful APIs that follow industry standards.

## Properties: Safety and Idempotency

Two critical properties define HTTP methods:

- **Safe**: Does not modify server state (read-only). Browsers may call safe methods freely without causing side effects.
- **Idempotent**: Calling multiple times produces the same result as calling once. Safe for retries — if a request fails, you can safely retry it.

### Why These Properties Matter

```
Safe Methods:
  - Can be cached by browsers and CDNs
  - Can be pre-fetched without risk
  - Search engines can crawl them safely
  - No side effects on the server

Idempotent Methods:
  - Network errors don't require special handling
  - Safe to retry on failure
  - Load balancers can retry automatically
  - Client doesn't need complex retry logic
```

| Method | Safe | Idempotent | Has Body | Business Case |
|--------|------|------------|----------|---------------|
| GET | Yes | Yes | No | Retrieve product list, user profile, order history |
| POST | No | No | Yes | Create new order, register user, upload invoice |
| PUT | No | Yes | Yes | Replace entire product record (full update) |
| PATCH | No | No* | Yes | Update only order status or user email (partial update) |
| DELETE | No | Yes | Optional | Delete a draft, remove item from cart, cancel appointment |
| HEAD | Yes | Yes | No | Check if file exists / get size before downloading |
| OPTIONS | Yes | Yes | No | CORS preflight — browser checks allowed methods before real request |

*PATCH is not guaranteed idempotent — e.g., `PATCH /counter { "op": "increment" }`

## HTTP Methods Explained

### GET

Retrieves data from the server. The request should not have a body, and the response contains the requested resource.

```
GET /api/products?category=electronics&page=1&pageSize=10 HTTP/1.1
Host: api.example.com
Accept: application/json

Response:
HTTP/1.1 200 OK
Content-Type: application/json

{
  "data": [...],
  "pagination": { "page": 1, "totalPages": 10 }
}
```

**Best Practices**:
- Use query parameters for filtering, sorting, and pagination
- Support caching with ETag headers
- Return appropriate status codes (200 for success, 404 if resource not found)

### POST

Creates a new resource. The request includes the data for the new resource in the body.

```
POST /api/orders HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "productId": 42,
  "quantity": 2,
  "shippingAddress": {
    "street": "123 Main St",
    "city": "Boston"
  }
}

Response:
HTTP/1.1 201 Created
Location: /api/orders/12345
Content-Type: application/json

{
  "id": 12345,
  "status": "pending",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

**Best Practices**:
- Return 201 Created with Location header
- Include the created resource in the response body
- Don't use POST for idempotent operations

### PUT

Replaces an existing resource entirely. The client sends the complete representation of the resource.

```
PUT /api/products/42 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "Laptop Pro",
  "price": 1299.99,
  "stock": 50,
  "description": "15-inch pro laptop",
  "category": "electronics"
}

Response:
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 42,
  "name": "Laptop Pro",
  ...
}
```

**Key Point**: If you omit any field, it may be set to null or default value. Always send the complete resource.

### PATCH

Partially updates a resource. Only the specified fields are modified; other fields remain unchanged.

```
PATCH /api/orders/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "status": "shipped",
  "shippedAt": "2024-01-15T14:00:00Z"
}

Response:
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 123,
  "status": "shipped",
  "createdAt": "2024-01-14T10:00:00Z",
  "shippedAt": "2024-01-15T14:00:00Z"
}
```

**PATCH Format**: Two common formats exist:
1. **JSON Patch** (RFC 6902): `[{ "op": "replace", "path": "/status", "value": "shipped" }]`
2. **Merge Patch**: Simple JSON with partial fields (as shown above)

### DELETE

Removes a resource from the server.

```
DELETE /api/cart/items/7 HTTP/1.1
Host: api.example.com

Response:
HTTP/1.1 204 No Content
```

**Best Practices**:
- Return 204 No Content for successful deletion
- Return 404 if resource doesn't exist (idempotent)
- Consider soft delete instead of hard delete for important data

### HEAD

Identical to GET but returns only headers, no body. Useful for checking if a resource exists or getting metadata without downloading.

```
HEAD /api/reports/monthly-2024-01.pdf HTTP/1.1
Host: api.example.com

Response:
HTTP/1.1 200 OK
Content-Length: 1048576
Content-Type: application/pdf
Last-Modified: Mon, 15 Jan 2024 10:00:00 GMT
ETag: "abc123"
```

**Use Cases**:
- Check if file exists before downloading
- Get file size before download
- Check if resource has been modified (with ETag)
- CDN cache validation

### OPTIONS

Returns information about the server's supported methods for a resource. Used primarily for CORS preflight requests.

```
OPTIONS /api/users HTTP/1.1
Host: api.example.com
Origin: https://example.com

Response:
HTTP/1.1 200 OK
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 86400
```

## Real Business Scenarios

### E-commerce Order Processing

```
POST /api/orders           → Create new order (201 Created)
GET  /api/orders/123       → View order details (200 OK)
PATCH /api/orders/123      → Update order status (200 OK)
DELETE /api/orders/123     → Cancel order (204 No Content)
```

### User Management

```
POST   /api/users          → Register new user (201 Created)
GET    /api/users/5        → Get user profile (200 OK)
PUT    /api/users/5        → Update entire profile (200 OK)
PATCH  /api/users/5        → Update email only (200 OK)
DELETE /api/users/5        → Delete user account (204 No Content)
```

## Common Misuse Examples

- **GET with body** — Used to pass filters but breaks caching, CDNs ignore it. Use query params instead.
- **POST for everything** — Missing idempotency means duplicate submissions cause duplicate records (double orders). Use PUT/PATCH appropriately.
- **PUT with partial data** — Forgetting to send all fields causes partial nulling of data. Use PATCH instead.
- **DELETE returning 200 with body** — Correct response is 204 No Content, not 200 with deleted object.
- **Non-idempotent PUT** — e.g., `PUT /api/counter/increment` — violates the contract; use POST.
- **Using GET for actions** — GET should only retrieve data, not perform actions

## HTTP Method Selection Guide

```
                    ┌─────────────────────────────────────────┐
                    │     What operation do you need?          │
                    └─────────────────────────────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
              v                   v                   v
     ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
     │  Read Data     │   │  Modify Data   │   │  Delete Data   │
     │  (No changes)  │   │  (Create/Upd) │   │                │
     └────────────────┘   └────────────────┘   └────────────────┘
              │                   │                   │
              │                   │                   │
              v                   v                   v
        ┌───────────┐       ┌───────────┐        ┌───────────┐
        │    GET    │       │ POST PUT  │        │  DELETE   │
        │    HEAD   │       │   PATCH   │        │           │
        └───────────┘       └───────────┘        └───────────┘
              │                   │                   │
              │        ┌──────────┼──────────┐        │
              │        │          │          │        │
              │        v          v          v        │
              │   ┌────────┐ ┌────────┐ ┌────────┐    │
              │   │ Create │ │ Replace │ │ Update │    │
              │   │  New   │ │  Entire │ │  Part  │    │
              │   └────────┘ └────────┘ └────────┘    │
              │      POST     PUT       PATCH        │
              └──────────────────────────────────────┘
```

## References

- [MDN — HTTP Request Methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods)
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)