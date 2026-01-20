# Common API Design Mistakes

A comprehensive catalog of API anti-patterns with explanations and fixes.

---

## 1. Inconsistent Naming Conventions

### The Problem
```yaml
paths:
  /users:              # plural
  /user/{id}:          # singular - inconsistent!
  /userProfile:        # camelCase
  /user-settings:      # kebab-case - inconsistent!
  /GetUserOrders:      # PascalCase + verb - very wrong
  /user_preferences:   # snake_case - yet another style
```

### Why It Matters
- Developers can't predict endpoint names
- Documentation becomes confusing
- SDK generation produces inconsistent method names
- Higher cognitive load when using the API

### The Fix
```yaml
paths:
  /users:
  /users/{id}:
  /users/{id}/profile:
  /users/{id}/settings:
  /users/{id}/orders:
  /users/{id}/preferences:
```

**Rules:**
- Always use plural nouns for collections
- Always use kebab-case for multi-word resources
- Never mix naming conventions

---

## 2. Verbs in URLs

### The Problem
```
POST /createUser
GET  /getUserById/{id}
POST /deleteUser/{id}
POST /updateUserEmail
GET  /fetchAllOrders
POST /doPayment
```

### Why It Matters
- URLs should identify resources, not actions
- HTTP methods already express the action
- Creates confusion about what method to use
- Breaks REST principles

### The Fix
```
POST   /users           # create
GET    /users/{id}      # read
DELETE /users/{id}      # delete
PATCH  /users/{id}      # update email (partial update)
GET    /orders          # fetch all orders
POST   /payments        # create payment
```

**Exception - Action Endpoints:**
For non-CRUD operations, verbs are acceptable as sub-resources:
```
POST /orders/{id}/cancel      ✓ (action on a specific order)
POST /orders/{id}/refund      ✓ (action on a specific order)
POST /users/{id}/verify-email ✓ (action on a specific user)
```

---

## 3. Wrong Status Codes

### The Problem
```
POST /users → 200 OK                    # Should be 201
DELETE /users/{id} → 200 OK             # Should be 204
GET /users/999 → 200 { "error": "..." } # Should be 404
POST /login (wrong pw) → 404            # Should be 401
POST /orders → 500 { "error": "out of stock" } # Should be 422
PUT /users/{id} → 200 (resource created) # Should be 201
```

### Why It Matters
- Clients rely on status codes for control flow
- Monitoring and alerting depends on accurate codes
- Caching behavior is affected by status codes
- API gateways route based on status codes

### The Fix
```
POST /users → 201 Created + Location: /users/123
DELETE /users/{id} → 204 No Content
GET /users/999 → 404 Not Found
POST /login (wrong pw) → 401 Unauthorized
POST /orders (out of stock) → 422 Unprocessable Entity
PUT /users/{id} (created) → 201 Created
```

**Quick Reference:**
| Scenario | Correct Status |
|----------|----------------|
| Resource created | 201 + Location header |
| Successful deletion | 204 |
| Resource not found | 404 |
| Authentication failed | 401 |
| Not authorized | 403 |
| Business rule violation | 422 |
| Duplicate resource | 409 |

---

## 4. Exposing Internal Implementation Details

### The Problem
```json
{
  "id": 12345,
  "_id": "507f1f77bcf86cd799439011",
  "_internalId": "mongo_5f4d3c2b1a",
  "createdBy": "admin_user_7",
  "databaseTable": "users_v2",
  "shardKey": "us-west-2-a",
  "mysqlRowId": 98765,
  "__v": 0
}
```

### Why It Matters
- Reveals database technology (security risk)
- Sequential IDs enable enumeration attacks
- Internal fields confuse API consumers
- Migration becomes harder (locked to implementation)

### The Fix
```json
{
  "id": "usr_a1b2c3d4e5f6g7h8",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

**Best Practices:**
- Use UUIDs or prefixed opaque IDs (`usr_`, `ord_`, `pay_`)
- Never expose auto-increment IDs
- Filter internal fields from responses
- Use consistent ID format across all resources

---

## 5. Inconsistent Error Responses

### The Problem
```json
// Endpoint A - validation error
{ "error": "User not found" }

// Endpoint B - not found
{ "message": "Not found", "code": 404 }

// Endpoint C - validation error
{ "errors": [{ "msg": "Invalid" }] }

// Endpoint D - server error
{ "success": false, "reason": "Something went wrong" }

// Endpoint E - auth error
"Unauthorized"
```

### Why It Matters
- Clients need different error handling per endpoint
- Impossible to build generic error handling
- Poor developer experience
- Harder to debug issues

### The Fix
```json
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User with ID 'usr_abc123' was not found",
    "requestId": "req_xyz789",
    "timestamp": "2024-01-15T10:30:00Z",
    "path": "/users/usr_abc123",
    "documentation": "https://api.example.com/docs/errors#USER_NOT_FOUND"
  }
}
```

**Validation Error Format:**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "requestId": "req_xyz789",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Must be a valid email address",
        "provided": "not-an-email"
      },
      {
        "field": "age",
        "code": "OUT_OF_RANGE",
        "message": "Must be between 18 and 120",
        "provided": 15
      }
    ]
  }
}
```

---

## 6. Missing Pagination

### The Problem
```
GET /orders → Returns ALL 50,000 orders in one response
GET /users → Returns 10,000 users (3MB response)
GET /logs → Returns 1 million log entries
```

### Why It Matters
- Memory exhaustion on server and client
- Network timeouts on large responses
- Database performance issues
- Poor user experience (long wait times)

### The Fix
```
# Require pagination
GET /orders → 400 Bad Request
{
  "error": {
    "code": "PAGINATION_REQUIRED",
    "message": "The 'limit' parameter is required for this endpoint"
  }
}

# With pagination
GET /orders?limit=20&offset=0
```

**Response Format:**
```json
{
  "data": [...],
  "pagination": {
    "total": 50000,
    "limit": 20,
    "offset": 0,
    "hasMore": true
  },
  "links": {
    "self": "/orders?limit=20&offset=0",
    "next": "/orders?limit=20&offset=20",
    "last": "/orders?limit=20&offset=49980"
  }
}
```

**For Large Datasets - Use Cursor Pagination:**
```json
{
  "data": [...],
  "pagination": {
    "limit": 20,
    "hasMore": true,
    "nextCursor": "eyJpZCI6MTAwfQ=="
  },
  "links": {
    "next": "/orders?limit=20&cursor=eyJpZCI6MTAwfQ=="
  }
}
```

---

## 7. Sensitive Data in URLs

### The Problem
```
GET /users?api_key=sk_live_abc123xyz
GET /reset-password?token=secret_reset_token_789
GET /users/{id}?ssn=123-45-6789
GET /login?password=mysecretpassword
GET /search?credit_card=4111111111111111
```

### Why It Matters
- URLs are logged in server access logs
- URLs are cached by CDNs and proxies
- URLs appear in browser history
- URLs are visible in referrer headers
- URLs may be stored in analytics

### The Fix
```http
# API keys in headers
GET /users
Authorization: Bearer sk_live_abc123xyz

# Or custom header
GET /users
X-API-Key: sk_live_abc123xyz

# Sensitive operations in POST body
POST /reset-password
Content-Type: application/json

{
  "token": "secret_reset_token_789",
  "newPassword": "newSecurePassword123!"
}

# Authentication via POST
POST /auth/login
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "mysecretpassword"
}
```

**Never Put in URLs:**
- API keys and tokens
- Passwords
- SSN, credit cards, PII
- Session identifiers
- Reset/verification tokens

---

## 8. Breaking Changes Without Versioning

### The Problem
```yaml
# V1 (original)
GET /users/{id}
Response:
  name: "John Doe"

# V2 (breaking change - field renamed!)
GET /users/{id}
Response:
  firstName: "John"
  lastName: "Doe"
```

### Why It Matters
- Existing clients break immediately
- Mobile apps can't be force-updated
- Breaks trust with API consumers
- No migration path for clients

### The Fix
```yaml
# V1 (maintained for existing clients)
GET /v1/users/{id}
Response:
  name: "John Doe"
Headers:
  Deprecation: true
  Sunset: Sat, 01 Jul 2027 00:00:00 GMT
  Link: </v2/users/{id}>; rel="successor-version"

# V2 (new version)
GET /v2/users/{id}
Response:
  firstName: "John"
  lastName: "Doe"
```

**Versioning Strategies:**
| Strategy | Example | Use Case |
|----------|---------|----------|
| URL Path | `/v1/users` | Public APIs |
| Header | `Accept-Version: v1` | Internal APIs |
| Content-Type | `Accept: application/vnd.api.v1+json` | Strict REST |

**Deprecation Timeline:**
1. Announce deprecation (6+ months ahead)
2. Add deprecation headers
3. Monitor usage of deprecated version
4. Sunset old version

---

## 9. Ignoring Idempotency

### The Problem
```
# Non-idempotent PUT (dangerous!)
PUT /orders/{id}/increment-quantity
# Called twice = quantity increased twice

# Non-idempotent DELETE
DELETE /wallet/deduct?amount=100
# Called twice = deducted $200

# No idempotency key for payments
POST /payments
{ "amount": 100, "to": "merchant_123" }
# Network retry = double charge
```

### Why It Matters
- Network retries cause duplicate operations
- Mobile apps often retry on timeout
- Users may double-click submit buttons
- Webhooks may be delivered multiple times

### The Fix
```http
# Idempotent PUT - sets absolute value
PUT /orders/{id}
{ "quantity": 5 }  # Always sets to 5, safe to retry

# Idempotency key for POST operations
POST /payments
Idempotency-Key: pay_req_unique_123
Content-Type: application/json

{
  "amount": 100,
  "to": "merchant_123"
}
# Same key = same result, no duplicate charge
```

**Idempotency Rules:**
| Method | Must Be Idempotent? |
|--------|---------------------|
| GET | Yes (no side effects) |
| PUT | Yes (same result every time) |
| DELETE | Yes (delete once or many times = deleted) |
| POST | No, but use Idempotency-Key for critical ops |
| PATCH | No* | *Can be idempotent with JSON Merge Patch |

---

## 10. Missing Request/Response Examples

### The Problem
```yaml
/users:
  post:
    summary: Create user
    requestBody:
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/User'
    responses:
      201:
        description: Created
        # No example!
```

### Why It Matters
- Developers can't understand the API quickly
- More support requests
- Higher integration errors
- Slower onboarding

### The Fix
```yaml
/users:
  post:
    summary: Create a new user account
    description: |
      Creates a new user with the provided information.
      The user will receive a verification email.
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/CreateUserRequest'
          example:
            email: "john.doe@example.com"
            name: "John Doe"
            password: "securePassword123!"
            preferences:
              newsletter: true
              language: "en"
    responses:
      201:
        description: User created successfully
        headers:
          Location:
            description: URL of the created user
            schema:
              type: string
            example: /users/usr_abc123
        content:
          application/json:
            example:
              data:
                id: "usr_abc123"
                email: "john.doe@example.com"
                name: "John Doe"
                createdAt: "2024-01-15T10:30:00Z"
                verified: false
              meta:
                requestId: "req_xyz789"
      400:
        description: Invalid request
        content:
          application/json:
            example:
              error:
                code: "VALIDATION_ERROR"
                message: "Request validation failed"
                details:
                  - field: "email"
                    code: "INVALID_FORMAT"
                    message: "Must be a valid email"
```

---

## 11. Overly Deep URL Nesting

### The Problem
```
GET /companies/{cid}/departments/{did}/teams/{tid}/members/{mid}/tasks/{taskId}/comments/{commentId}

POST /users/{uid}/projects/{pid}/boards/{bid}/lists/{lid}/cards/{cid}/attachments
```

### Why It Matters
- URLs become unwieldy and error-prone
- Implies strict hierarchy that may not exist
- Makes caching difficult
- Complicates client-side routing

### The Fix
```
# Flatten to max 2 levels
GET /comments/{commentId}
GET /tasks/{taskId}/comments

# Use query params for context
GET /comments?taskId={taskId}

# Direct access to nested resources
GET /attachments/{attachmentId}
POST /cards/{cardId}/attachments
```

**Rule:** Maximum 2 levels of nesting: `/resource/{id}/sub-resource`

---

## 12. Returning 200 for Errors

### The Problem
```json
// HTTP 200 OK
{
  "success": false,
  "error": "User not found"
}

// HTTP 200 OK
{
  "status": "error",
  "message": "Invalid credentials"
}
```

### Why It Matters
- HTTP clients treat 200 as success
- Monitoring shows false positives
- Caching may cache error responses
- Breaks standard error handling

### The Fix
```json
// HTTP 404 Not Found
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User not found"
  }
}

// HTTP 401 Unauthorized
{
  "error": {
    "code": "INVALID_CREDENTIALS",
    "message": "Invalid email or password"
  }
}
```

---

## 13. No Rate Limiting Information

### The Problem
```http
HTTP/1.1 200 OK
Content-Type: application/json

{"data": [...]}
# No rate limit headers - client has no idea about limits
```

Then suddenly:
```http
HTTP/1.1 429 Too Many Requests
# Client had no warning
```

### Why It Matters
- Clients can't implement backoff strategies
- No way to optimize request patterns
- Sudden failures surprise users
- Makes capacity planning impossible

### The Fix
```http
HTTP/1.1 200 OK
Content-Type: application/json
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 998
X-RateLimit-Reset: 1640000000
X-RateLimit-Policy: "1000;w=3600"

{"data": [...]}
```

When rate limited:
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 3600
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Retry after 3600 seconds.",
    "retryAfter": 3600
  }
}
```

---

## 14. Inconsistent Null/Empty Handling

### The Problem
```json
// Sometimes null
{ "middleName": null }

// Sometimes missing
{ }

// Sometimes empty string
{ "middleName": "" }

// Sometimes "null" string
{ "middleName": "null" }
```

### Why It Matters
- Clients need different handling for each case
- JSON parsers treat these differently
- Database mapping becomes inconsistent
- Bugs from null pointer exceptions

### The Fix

Pick ONE strategy and document it:

**Option A: Omit null fields (recommended)**
```json
// Field not applicable - omit it
{ "name": "John Doe" }

// Field applicable but empty
{ "name": "John Doe", "middleName": "" }
```

**Option B: Always include all fields**
```json
// Explicit null for optional fields
{ "name": "John Doe", "middleName": null }
```

Document your choice in API documentation.

---

## 15. Using GET for State Changes

### The Problem
```
GET /users/{id}/activate
GET /orders/{id}/cancel
GET /send-email?to=user@example.com
GET /delete-cache
```

### Why It Matters
- GET requests can be cached
- Browsers prefetch GET links
- Crawlers may trigger state changes
- Violates HTTP specification

### The Fix
```
POST /users/{id}/activate
POST /orders/{id}/cancel
POST /emails
DELETE /cache
```

**Rule:** GET and HEAD must be safe (no side effects)

---

## Quick Smell Test Checklist

Run through these questions during any API review:

| # | Question | If No... |
|---|----------|----------|
| 1 | Can I guess the HTTP method from the URL? | Remove verbs from URLs |
| 2 | Are all collection resources plural? | Standardize to plurals |
| 3 | Is there pagination for list endpoints? | Add pagination |
| 4 | Do errors return proper status codes? | Fix status code usage |
| 5 | Is the error format consistent? | Standardize error schema |
| 6 | Is there a version in the URL or headers? | Add versioning |
| 7 | Is authentication documented per endpoint? | Document auth requirements |
| 8 | Are there request/response examples? | Add examples |
| 9 | Are sensitive values in headers, not URLs? | Move to headers/body |
| 10 | Do PUT/DELETE operations behave idempotently? | Fix idempotency issues |
