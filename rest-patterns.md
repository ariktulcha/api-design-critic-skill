# REST API Patterns Reference

A comprehensive guide to REST API best practices, patterns, and conventions.

---

## URL Design Patterns

### Resource Naming

| Pattern | Example | Notes |
|---------|---------|-------|
| Collections (plural) | `/users`, `/orders` | Always plural |
| Single resource | `/users/{id}` | ID after collection |
| Sub-resources | `/users/{id}/orders` | Max 2 levels deep |
| Actions | `/orders/{id}/cancel` | Verb as sub-resource |
| Search | `/users/search` or `/search/users` | Dedicated search endpoint |

### Good URL Patterns

```
# Standard CRUD
GET    /users                    # List users
GET    /users/{id}               # Get single user
POST   /users                    # Create user
PUT    /users/{id}               # Full replacement
PATCH  /users/{id}               # Partial update
DELETE /users/{id}               # Delete user

# Relationships (shallow nesting)
GET    /users/{id}/orders        # User's orders
GET    /users/{id}/profile       # User's profile
POST   /users/{id}/addresses     # Add address to user

# Actions on resources
POST   /orders/{id}/cancel       # Cancel an order
POST   /orders/{id}/refund       # Refund an order
POST   /users/{id}/verify        # Verify user email
POST   /accounts/{id}/suspend    # Suspend account

# Filtering and querying
GET    /orders?status=pending              # Filter by status
GET    /orders?status=pending,shipped      # Multiple values
GET    /orders?created_after=2024-01-01    # Date filtering
GET    /products?price_min=10&price_max=50 # Range filtering

# Sorting
GET    /orders?sort=created_at             # Ascending
GET    /orders?sort=-created_at            # Descending (- prefix)
GET    /orders?sort=-created_at,status     # Multiple fields

# Pagination
GET    /users?limit=20&offset=0            # Offset pagination
GET    /users?limit=20&cursor=abc123       # Cursor pagination

# Field selection
GET    /users?fields=id,name,email         # Sparse fieldsets

# Related resources
GET    /orders?include=customer,items      # Expand relationships
GET    /orders?expand=customer.address     # Nested expansion
```

### Anti-Patterns to Avoid

```
# Verbs in URLs
GET    /getUser/{id}                 ✗ Use GET /users/{id}
POST   /createUser                   ✗ Use POST /users
POST   /deleteUser/{id}              ✗ Use DELETE /users/{id}
GET    /fetchAllOrders               ✗ Use GET /orders

# Inconsistent pluralization
GET    /user/{id}                    ✗ Use /users/{id}
GET    /order/{id}/item              ✗ Use /orders/{id}/items

# Deep nesting
GET    /users/{id}/orders/{oid}/items/{iid}/details/{did}  ✗
# Better: GET /order-items/{iid} or GET /items/{iid}

# RPC-style naming
PUT    /updateUserEmail              ✗ Use PATCH /users/{id}
POST   /processPayment               ✗ Use POST /payments

# IDs in query params
GET    /users?id=123                 ✗ Use /users/123

# File extensions
GET    /users.json                   ✗ Use Accept header

# Mixed case
GET    /UserProfiles                 ✗ Use /user-profiles
GET    /user_profiles                ✗ Use /user-profiles
```

---

## Response Envelope Patterns

### Standard Success Response (Single Resource)

```json
{
  "data": {
    "id": "usr_abc123",
    "type": "user",
    "email": "john@example.com",
    "name": "John Doe",
    "createdAt": "2024-01-15T10:30:00Z",
    "updatedAt": "2024-01-15T10:30:00Z"
  },
  "meta": {
    "requestId": "req_xyz789",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Collection Response with Pagination

```json
{
  "data": [
    {
      "id": "usr_abc123",
      "type": "user",
      "email": "john@example.com",
      "name": "John Doe"
    },
    {
      "id": "usr_def456",
      "type": "user",
      "email": "jane@example.com",
      "name": "Jane Smith"
    }
  ],
  "pagination": {
    "total": 150,
    "count": 20,
    "limit": 20,
    "offset": 0,
    "hasMore": true,
    "totalPages": 8,
    "currentPage": 1
  },
  "links": {
    "self": "/users?limit=20&offset=0",
    "first": "/users?limit=20&offset=0",
    "prev": null,
    "next": "/users?limit=20&offset=20",
    "last": "/users?limit=20&offset=140"
  },
  "meta": {
    "requestId": "req_xyz789",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

### Cursor-Based Pagination (for Large Datasets)

```json
{
  "data": [...],
  "pagination": {
    "limit": 20,
    "hasMore": true,
    "nextCursor": "eyJpZCI6MTAwLCJjcmVhdGVkQXQiOiIyMDI0LTAxLTE1In0=",
    "prevCursor": "eyJpZCI6ODAsImNyZWF0ZWRBdCI6IjIwMjQtMDEtMTQifQ=="
  },
  "links": {
    "self": "/users?limit=20&cursor=eyJpZCI6ODB9",
    "next": "/users?limit=20&cursor=eyJpZCI6MTAwfQ==",
    "prev": "/users?limit=20&cursor=eyJpZCI6ODB9&direction=prev"
  }
}
```

### Created Resource Response (201)

```json
{
  "data": {
    "id": "usr_abc123",
    "email": "john@example.com",
    "name": "John Doe",
    "createdAt": "2024-01-15T10:30:00Z"
  },
  "meta": {
    "requestId": "req_xyz789"
  }
}
```

With headers:
```http
HTTP/1.1 201 Created
Location: /users/usr_abc123
Content-Type: application/json
```

### Error Response

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Must be a valid email address",
        "provided": "not-an-email"
      },
      {
        "field": "password",
        "code": "TOO_SHORT",
        "message": "Must be at least 8 characters",
        "provided": "abc"
      }
    ],
    "requestId": "req_xyz789",
    "timestamp": "2024-01-15T10:30:00Z",
    "path": "/users",
    "documentation": "https://api.example.com/docs/errors#VALIDATION_ERROR"
  }
}
```

### Async Operation Response (202)

```json
{
  "data": {
    "jobId": "job_abc123",
    "status": "pending",
    "createdAt": "2024-01-15T10:30:00Z",
    "estimatedCompletion": "2024-01-15T10:35:00Z"
  },
  "links": {
    "self": "/jobs/job_abc123",
    "cancel": "/jobs/job_abc123/cancel"
  },
  "meta": {
    "requestId": "req_xyz789"
  }
}
```

---

## Status Code Reference

### Success Codes (2xx)

| Code | Name | When to Use | Response Body |
|------|------|-------------|---------------|
| 200 | OK | Successful GET, PUT, PATCH | Resource data |
| 201 | Created | Resource successfully created | Created resource + Location header |
| 202 | Accepted | Async operation accepted | Job/task reference |
| 204 | No Content | Successful DELETE, or update with no body | None |

### Client Error Codes (4xx)

| Code | Name | When to Use | Example |
|------|------|-------------|---------|
| 400 | Bad Request | Malformed syntax, invalid JSON | `{ "foo": invalid }` |
| 401 | Unauthorized | Missing or invalid authentication | No token, expired token |
| 403 | Forbidden | Authenticated but not authorized | User can't access resource |
| 404 | Not Found | Resource doesn't exist | `/users/999` not found |
| 405 | Method Not Allowed | HTTP method not supported | DELETE on read-only resource |
| 409 | Conflict | State conflict, duplicate | Email already registered |
| 410 | Gone | Resource permanently deleted | Archived content |
| 415 | Unsupported Media Type | Wrong Content-Type | Sending XML to JSON-only endpoint |
| 422 | Unprocessable Entity | Valid syntax, semantic error | Business rule violation |
| 429 | Too Many Requests | Rate limit exceeded | Include Retry-After header |

### Server Error Codes (5xx)

| Code | Name | When to Use | Notes |
|------|------|-------------|-------|
| 500 | Internal Server Error | Unexpected server error | Never expose stack traces |
| 502 | Bad Gateway | Upstream service failed | Proxy/gateway errors |
| 503 | Service Unavailable | Server temporarily overloaded | Include Retry-After |
| 504 | Gateway Timeout | Upstream service timeout | Proxy timeout |

### Status Code Decision Tree

```
Is the request successful?
├── Yes
│   ├── Creating a resource? → 201 Created + Location
│   ├── Deleting a resource? → 204 No Content
│   ├── Async operation? → 202 Accepted
│   └── Otherwise → 200 OK
│
└── No (Error)
    ├── Is syntax valid?
    │   └── No → 400 Bad Request
    │
    ├── Is user authenticated?
    │   └── No → 401 Unauthorized
    │
    ├── Is user authorized?
    │   └── No → 403 Forbidden
    │
    ├── Does resource exist?
    │   └── No → 404 Not Found
    │
    ├── Is there a conflict?
    │   └── Yes → 409 Conflict
    │
    ├── Is it a business rule violation?
    │   └── Yes → 422 Unprocessable Entity
    │
    ├── Rate limit exceeded?
    │   └── Yes → 429 Too Many Requests
    │
    └── Server error → 500 Internal Server Error
```

---

## Versioning Strategies

### URL Path Versioning (Recommended for Public APIs)

```http
GET /v1/users
GET /v2/users
```

**Pros:**
- Highly visible and explicit
- Easy to route at load balancer/gateway level
- Simple to document and understand

**Cons:**
- "Pollutes" the URL
- Harder to share URLs across versions

### Header Versioning (Recommended for Internal APIs)

```http
GET /users
Accept-Version: v1
# or
X-API-Version: 2024-01-15
```

**Pros:**
- Clean URLs
- Version can be optional (defaults to latest)

**Cons:**
- Less visible in documentation
- Harder to test in browser

### Content-Type Versioning

```http
GET /users
Accept: application/vnd.myapi.v1+json
```

**Pros:**
- Most "RESTful" approach
- Supports content negotiation

**Cons:**
- Complex to implement
- Non-standard media types

### Query Parameter Versioning

```http
GET /users?version=1
```

**Pros:**
- Easy to test
- Optional parameter

**Cons:**
- Caching complications
- Not truly RESTful

### Deprecation Headers

When deprecating a version, include:

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jul 2025 00:00:00 GMT
Link: </v2/users>; rel="successor-version"
```

---

## Query Parameter Conventions

### Filtering

```http
# Exact match
GET /orders?status=pending

# Multiple values (OR)
GET /orders?status=pending,shipped,delivered

# Range filtering
GET /products?price_min=10&price_max=100
GET /orders?created_after=2024-01-01&created_before=2024-12-31

# Null filtering
GET /users?manager=null
GET /users?manager=!null

# Pattern matching (if supported)
GET /users?email=*@example.com
```

### Sorting

```http
# Single field ascending
GET /orders?sort=created_at

# Single field descending (- prefix)
GET /orders?sort=-created_at

# Multiple fields
GET /orders?sort=-created_at,status

# Alternative syntax
GET /orders?sort=created_at:desc,status:asc
```

### Pagination

```http
# Offset-based (simple, good for small datasets)
GET /users?limit=20&offset=40
GET /users?page=3&per_page=20

# Cursor-based (scalable, good for large datasets)
GET /users?limit=20&cursor=eyJpZCI6MTAwfQ==

# Keyset-based (most performant)
GET /users?limit=20&after_id=usr_abc123
```

### Field Selection (Sparse Fieldsets)

```http
# Select specific fields
GET /users?fields=id,name,email

# Per-resource fields
GET /orders?fields[order]=id,total&fields[customer]=name
```

### Search

```http
# Simple full-text search
GET /users?q=john

# Field-specific search
GET /users?search[name]=john&search[email]=@example.com

# Dedicated search endpoint
POST /users/search
{
  "query": "john",
  "filters": {
    "status": "active",
    "created_after": "2024-01-01"
  }
}
```

### Expansion/Embedding

```http
# Include related resources
GET /orders?include=customer,items

# Nested expansion
GET /orders?include=customer.address,items.product

# Specific fields from related
GET /orders?include=customer(name,email)
```

---

## HTTP Headers Reference

### Request Headers

| Header | Purpose | Example |
|--------|---------|---------|
| `Authorization` | Authentication | `Bearer eyJhbGc...` |
| `Content-Type` | Request body format | `application/json` |
| `Accept` | Preferred response format | `application/json` |
| `Accept-Version` | API version | `v1` or `2024-01-15` |
| `Idempotency-Key` | Prevent duplicates | `pay_req_abc123` |
| `If-None-Match` | Conditional GET (caching) | `"etag123"` |
| `If-Match` | Conditional PUT (concurrency) | `"etag123"` |
| `X-Request-ID` | Client-generated request ID | `req_client_123` |

### Response Headers

| Header | Purpose | Example |
|--------|---------|---------|
| `Content-Type` | Response format | `application/json` |
| `Location` | Created resource URL | `/users/usr_abc123` |
| `ETag` | Resource version for caching | `"etag123"` |
| `Last-Modified` | Last update timestamp | `Wed, 15 Jan 2024 10:30:00 GMT` |
| `X-Request-ID` | Server request ID | `req_xyz789` |
| `X-RateLimit-Limit` | Rate limit ceiling | `1000` |
| `X-RateLimit-Remaining` | Requests remaining | `999` |
| `X-RateLimit-Reset` | Reset timestamp (Unix) | `1705320000` |
| `Retry-After` | Wait time (429/503) | `3600` or datetime |
| `Deprecation` | Version deprecated | `true` |
| `Sunset` | End-of-life date | `Sat, 01 Jul 2025 00:00:00 GMT` |
| `Link` | Related resources | `</v2/users>; rel="successor-version"` |

### Security Headers

```http
# HTTPS enforcement
Strict-Transport-Security: max-age=31536000; includeSubDomains

# Prevent MIME sniffing
X-Content-Type-Options: nosniff

# Frame protection
X-Frame-Options: DENY

# XSS protection (legacy browsers)
X-XSS-Protection: 1; mode=block

# Content Security Policy
Content-Security-Policy: default-src 'self'

# CORS headers
Access-Control-Allow-Origin: https://example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

---

## Authentication Patterns

### Bearer Token (JWT/OAuth2)

```http
GET /users
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### API Key

```http
# In header (preferred)
GET /users
X-API-Key: sk_live_abc123xyz

# Never in URL
GET /users?api_key=sk_live_abc123xyz  ✗ NEVER DO THIS
```

### Basic Auth

```http
GET /users
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

### OAuth2 Scopes

Document required scopes per endpoint:

```yaml
/users:
  get:
    security:
      - oauth2: [users:read]
  post:
    security:
      - oauth2: [users:write]
/users/{id}:
  delete:
    security:
      - oauth2: [users:delete, admin]
```

---

## HATEOAS (Hypermedia)

### Resource Links

```json
{
  "data": {
    "id": "ord_abc123",
    "status": "pending",
    "total": 99.99
  },
  "links": {
    "self": "/orders/ord_abc123",
    "customer": "/customers/cust_xyz789",
    "items": "/orders/ord_abc123/items",
    "cancel": "/orders/ord_abc123/cancel",
    "payment": "/orders/ord_abc123/payment"
  },
  "actions": {
    "cancel": {
      "href": "/orders/ord_abc123/cancel",
      "method": "POST",
      "title": "Cancel this order"
    },
    "pay": {
      "href": "/orders/ord_abc123/payment",
      "method": "POST",
      "title": "Submit payment"
    }
  }
}
```

### Collection Navigation

```json
{
  "data": [...],
  "links": {
    "self": "/orders?page=2",
    "first": "/orders?page=1",
    "prev": "/orders?page=1",
    "next": "/orders?page=3",
    "last": "/orders?page=10"
  }
}
```

---

## Idempotency

### Idempotent Operations

| Method | Idempotent | Notes |
|--------|------------|-------|
| GET | Yes | Must not have side effects |
| HEAD | Yes | Must not have side effects |
| PUT | Yes | Same request = same result |
| DELETE | Yes | Deleting twice = still deleted |
| OPTIONS | Yes | No side effects |
| POST | No | Use Idempotency-Key for critical ops |
| PATCH | No | Can be designed to be idempotent |

### Idempotency-Key Header

For non-idempotent operations (payments, orders):

```http
POST /payments
Idempotency-Key: pay_req_abc123
Content-Type: application/json

{
  "amount": 100,
  "currency": "USD"
}
```

Server behavior:
1. First request: Process and store result with key
2. Subsequent requests with same key: Return stored result
3. Key expires after 24 hours (typical)

---

## Error Codes Catalog

### Standard Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `VALIDATION_ERROR` | 400/422 | Request validation failed |
| `INVALID_JSON` | 400 | Malformed JSON in request |
| `MISSING_FIELD` | 400 | Required field not provided |
| `INVALID_FORMAT` | 400 | Field format invalid |
| `UNAUTHORIZED` | 401 | Authentication required |
| `INVALID_TOKEN` | 401 | Token expired or invalid |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `METHOD_NOT_ALLOWED` | 405 | HTTP method not supported |
| `CONFLICT` | 409 | Resource conflict (duplicate) |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unexpected server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily down |

### Domain-Specific Codes

```json
{
  "error": {
    "code": "INSUFFICIENT_FUNDS",
    "message": "Account balance is too low for this transaction"
  }
}
```

```json
{
  "error": {
    "code": "ORDER_ALREADY_SHIPPED",
    "message": "Cannot cancel an order that has already shipped"
  }
}
```

---

## Content Negotiation

### Request

```http
GET /users/123
Accept: application/json
Accept-Language: en-US
Accept-Encoding: gzip, deflate
```

### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Language: en-US
Content-Encoding: gzip
Vary: Accept, Accept-Language, Accept-Encoding
```

### Multiple Formats

```http
# JSON (default)
GET /report
Accept: application/json

# CSV export
GET /report
Accept: text/csv

# PDF export
GET /report
Accept: application/pdf
```
