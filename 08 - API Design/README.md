# Chapter 8: API Design

## 8.1 REST API Design

### Resource-Oriented URLs

```
# Collections
GET    /api/v1/users                    # List users
POST   /api/v1/users                    # Create user

# Single resource
GET    /api/v1/users/{id}               # Get user
PATCH  /api/v1/users/{id}               # Partial update
PUT    /api/v1/users/{id}               # Full replace
DELETE /api/v1/users/{id}               # Delete user

# Sub-resources
GET    /api/v1/users/{id}/orders        # User's orders
POST   /api/v1/users/{id}/orders        # Create order for user
GET    /api/v1/users/{id}/orders/{oid}  # Specific order
```

### HTTP Methods Semantics

| Method | Idempotent | Safe | Purpose |
|--------|------------|------|---------|
| **GET** | ✅ | ✅ | Retrieve |
| **HEAD** | ✅ | ✅ | Metadata only |
| **OPTIONS** | ✅ | ✅ | Discover capabilities |
| **POST** | ❌ | ❌ | Create, actions |
| **PUT** | ✅ | ❌ | Full replace |
| **PATCH** | ❌* | ❌ | Partial update |
| **DELETE** | ✅ | ❌ | Remove |

*PATCH can be idempotent if implemented correctly (e.g., JSON Merge Patch)*

### Status Codes

```
2xx Success
  200 OK                    GET, PUT, PATCH success
  201 Created               POST success (include Location header)
  202 Accepted              Async operation started
  204 No Content            DELETE, successful no-body response

3xx Redirection
  301 Moved Permanently
  304 Not Modified          Caching (If-None-Match)

4xx Client Error
  400 Bad Request           Validation failed
  401 Unauthorized          Auth required/invalid
  403 Forbidden             Auth valid, no permission
  404 Not Found             Resource doesn't exist
  409 Conflict              Resource conflict (duplicate)
  422 Unprocessable Entity  Semantic validation failed
  429 Too Many Requests     Rate limited (include Retry-After)

5xx Server Error
  500 Internal Server Error Unexpected error
  502 Bad Gateway           Upstream failure
  503 Service Unavailable   Overloaded, maintenance
  504 Gateway Timeout       Upstream timeout
```

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input",
    "details": [
      { "field": "email", "code": "INVALID_FORMAT", "message": "Invalid email format" },
      { "field": "age", "code": "OUT_OF_RANGE", "message": "Age must be 18-100" }
    ],
    "request_id": "req_abc123",
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

---

## 8.2 Pagination

### Cursor-Based (Recommended for Large Datasets)

```
Request:
GET /api/v1/users?limit=20&cursor=eyJpZCI6MTAwfQ==

Response:
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTIwfQ==",
    "has_more": true,
    "limit": 20
  }
}
```

| Method | Pros | Cons |
|--------|------|------|
| **Cursor** | Consistent at scale, no offset drift | Can't jump to page N |
| **Offset/Limit** | Simple, jump to page | Slow at scale (OFFSET scans), drift on writes |
| **Keyset** | Fast, consistent | Complex for multi-column sort |

> **Rule**: Use cursor/keyset for APIs. Offset only for small, static datasets (<10K rows).

---

## 8.3 API Versioning

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URL Path** | `/api/v1/users` | Explicit, easy to route | URL pollution |
| **Header** | `Accept: application/vnd.api.v1+json` | Clean URLs | Less visible, harder to test |
| **Query Param** | `/api/users?version=1` | Simple | Caching issues |

**Recommendation**: URL path for major versions (`/v1/`, `/v2/`), header for minor.

---

## 8.4 GraphQL

### When to Use GraphQL

```
✅ Multiple clients with different data needs (mobile vs web)
✅ Complex nested relationships (avoid N+1 with DataLoader)
✅ Rapid frontend iteration (schema as contract)
✅ Federated microservices (Apollo Federation)

❌ Simple CRUD (REST is simpler)
❌ Public API (caching, rate limiting harder)
❌ Team lacks GraphQL expertise
❌ High-throughput internal RPC (gRPC better)
```

### N+1 Problem & DataLoader

```
# Naive: N+1 queries
users = db.query("SELECT * FROM users LIMIT 10")
for user in users:
    posts = db.query("SELECT * FROM posts WHERE user_id = ?", user.id)  # 10 queries!

# DataLoader: Batch + Cache
posts_loader = DataLoader(lambda keys: db.query(
    "SELECT * FROM posts WHERE user_id IN ?", keys
))
for user in users:
    posts = posts_loader.load(user.id)  # Batched into 1 query
```

---

## 8.5 gRPC

### Protobuf Schema

```protobuf
syntax = "proto3";

package user.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (stream User);  // Server streaming
  rpc CreateUser(stream CreateUserRequest) returns (User); // Client streaming
  rpc Chat(stream Message) returns (stream Message);       // Bidirectional
}

message User {
  string id = 1;
  string email = 2;
  string name = 3;
  google.protobuf.Timestamp created_at = 4;
}

message GetUserRequest {
  string id = 1;
}
```

### gRPC vs REST

| Aspect | gRPC | REST |
|--------|------|------|
| **Contract** | .proto (code-gen) | OpenAPI / implicit |
| **Serialization** | Protobuf (binary) | JSON (text) |
| **Transport** | HTTP/2 (required) | HTTP/1.1 or HTTP/2 |
| **Streaming** | Native (4 modes) | WebSocket / SSE |
| **Browser** | gRPC-Web proxy needed | Native |
| **Caching** | Custom | Native HTTP |
| **Debugging** | Requires tools | curl, browser |

---

## 8.6 API Gateway Responsibilities

```
Request ──► [API Gateway] ──► Services
              │
              ├── Auth (JWT validation, OAuth2, mTLS)
              ├── Rate Limiting (token bucket per key)
              ├── Routing (path, header, weight)
              ├── Request/Response Transform
              ├── Aggregation (fan-out + compose)
              ├── Circuit Breaker
              ├── Request Validation (schema)
              ├── Response Caching
              ├── Observability (metrics, traces, logs)
              └── SSL Termination
```

---

## 8.7 Webhooks

```
Producer (Your System)                    Consumer (Client)
     │                                        │
     ├─ Event occurs                          │
     │                                        │
     ├─ POST https://client.com/webhook ─────►│
     │     {                                  │
     │       "event": "order.created",        │
     │       "timestamp": "...",              │
     │       "payload": {...}                 │
     │     }                                  │
     │                                        │
     │◄── 200 OK (within 5s) ─────────────────┤
     │                                        │
     │  (Retry with backoff on failure)       │
```

### Webhook Best Practices

| Practice | Reason |
|----------|--------|
| **Sign payloads** | HMAC-SHA256, verify authenticity |
| **Idempotent delivery** | Same event may deliver multiple times |
| **Retry with backoff** | Exponential, max 24-72h |
| **Ordering** | Sequence numbers if order matters |
| **Local testing** | ngrok, webhook.site, CLI tools |

---

## 8.8 API Security

### Authentication

| Method | Use Case | Token Location |
|--------|----------|----------------|
| **API Key** | Server-to-server, simple | Header: `X-API-Key` |
| **JWT (Bearer)** | Stateless auth, OAuth2 | Header: `Authorization: Bearer <jwt>` |
| **Session/Cookie** | Browser apps | Cookie (HttpOnly, Secure, SameSite) |
| **mTLS** | Service-to-service, high security | TLS cert verification |

### Authorization

```
RBAC (Role-Based):
  User ──► Role ──► Permissions
  Admin → [read, write, delete, admin]
  User  → [read, write]

ABAC (Attribute-Based):
  Policy: "Allow if user.department == resource.department AND user.level >= resource.sensitivity"
```

---

## 8.9 API Documentation & Contract Testing

### OpenAPI / Swagger

```yaml
openapi: 3.0.3
info:
  title: User API
  version: 1.0.0
paths:
  /users/{id}:
    get:
      summary: Get user by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: Not found
components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
          format: uuid
        email:
          type: string
          format: email
        name:
          type: string
```

### Contract Testing (Pact)

```
Consumer (Client)                   Provider (API)
     │                                    │
     ├─ Define contract (Pact file) ─────►│
     │                                    │
     │                                    ├─ Verify against provider code
     │                                    │     (CI pipeline)
     │                                    │
     │◄── Verification result ────────────┤
```

---

## ✅ Exercises

1. **REST Design**: Design API for a blog platform (posts, comments, tags, authors). Include pagination, filtering, sorting.
2. **Error Handling**: Define error codes and HTTP status for: duplicate email, invalid credit card, insufficient inventory, rate limited.
3. **GraphQL vs REST**: You're building a dashboard with 20 widgets, each needing different data. REST or GraphQL? Justify.
4. **Versioning**: You need to add a required field to `CreateUser`. How do you version?

---

## 🔁 Review Prompts

- When do you use PUT vs PATCH? What's the idempotency difference?
- Why is cursor pagination better than offset at scale?
- What are the trade-offs between REST, GraphQL, and gRPC?
- How do you secure webhooks against replay and forgery?

---

## 📚 Further Reading

- [REST API Tutorial](https://restfulapi.net/)
- [GraphQL Best Practices](https://graphql.org/learn/best-practices/)
- [gRPC Documentation](https://grpc.io/docs/)
- [OpenAPI Specification](https://spec.openapis.org/oas/v3.0.3)
- *Web API Design* — Apigee (Google)