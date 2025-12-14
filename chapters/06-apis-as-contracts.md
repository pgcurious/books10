# Chapter 6: APIs as Contracts

> **First Principles Question**: When you build an API, you're not just writing code—you're making promises. What promises should you make, and how do you keep them?

---

## Chapter Overview

An API is a contract between your service and its consumers. Bad API design leads to frustrated developers, breaking changes, and integration nightmares. Good API design creates developer experiences that feel inevitable—like this was the only way it could have worked.

**What readers will understand after this chapter:**
- Why API design is fundamentally about communication
- REST principles and when to follow them
- How to version APIs without breaking clients
- Error handling that helps rather than frustrates
- Documentation as a first-class concern
- The relationship between API design and business needs

---

## Section 1: APIs Are Communication Protocols

### 1.1 What an API Actually Is

**Application Programming Interface:**
The surface area through which one piece of software talks to another.

**The Contract Metaphor:**
```
┌─────────────────────────────────────────────────────────────┐
│                        API Contract                          │
│                                                             │
│  Provider promises:                                         │
│  - POST /orders creates an order                           │
│  - Returns order ID in response                            │
│  - Returns 201 on success, 400 on invalid input            │
│                                                             │
│  Consumer promises:                                         │
│  - Send valid order JSON in request body                   │
│  - Include authentication token                            │
│  - Handle both success and error responses                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 The Human Element

**APIs Serve Developers:**
Your API's user isn't a machine—it's a developer trying to build something. They:
- Have limited time to learn your API
- Will make mistakes
- Need clear error messages
- Want predictable behavior

**Empathy-Driven Design:**
Every API decision should answer: "Will this make sense to someone who doesn't know our system?"

### 1.3 Types of APIs

**Web APIs (HTTP-based):**
- REST (most common)
- GraphQL (query-based)
- gRPC (high-performance RPC)

**Library APIs:**
- Public methods of a package
- Same principles apply

**This Chapter's Focus:**
HTTP-based REST APIs—the lingua franca of web services.

---

## Section 2: REST Fundamentals

### 2.1 What REST Actually Means

**RE**presentational **S**tate **T**ransfer

**The Core Idea:**
Resources have representations (JSON, XML). Clients transfer these representations to read or modify state.

**Not a Standard, a Style:**
REST is an architectural style defined by Roy Fielding in his 2000 dissertation. There's no official "REST specification."

### 2.2 REST Principles

**1. Resources:**
Everything is a resource identified by a URL.
```
/users/123          → User 123
/users/123/orders   → Orders belonging to User 123
/products/456       → Product 456
```

**2. HTTP Verbs Have Meaning:**
```
GET     → Read (safe, idempotent)
POST    → Create (not idempotent)
PUT     → Replace (idempotent)
PATCH   → Partial update (not necessarily idempotent)
DELETE  → Remove (idempotent)
```

**3. Stateless:**
Each request contains all information needed. No server-side sessions between requests.

**4. Representations:**
Resources can have multiple representations (JSON, XML). Clients request what they want via Accept header.

### 2.3 Designing Resource URLs

**Good Resource Naming:**
```
# Nouns, not verbs
GET  /users           ✓ (list users)
GET  /getUsers        ✗ (verb in URL)

# Plural for collections
GET  /users           ✓
GET  /user            ✗

# Hierarchy shows relationships
GET  /users/123/orders       ✓ (orders for user 123)
GET  /orders?userId=123      ✓ (alternative, both valid)

# Filter via query parameters
GET  /orders?status=pending&limit=10
```

**URL Design Patterns:**
```
/resources                    → Collection
/resources/{id}              → Specific resource
/resources/{id}/subresources → Related collection
/resources/{id}/actions      → Actions (sparingly)
```

### 2.4 CRUD Mapping

**The Standard Mapping:**
| Operation | HTTP Method | URL | Response |
|-----------|-------------|-----|----------|
| List | GET | /users | 200 + array |
| Read | GET | /users/123 | 200 + object |
| Create | POST | /users | 201 + object |
| Replace | PUT | /users/123 | 200 + object |
| Update | PATCH | /users/123 | 200 + object |
| Delete | DELETE | /users/123 | 204 (no content) |

---

## Section 3: Designing for Real-World Complexity

### 3.1 When CRUD Isn't Enough

**The Problem:**
Some operations don't fit the resource model cleanly.

**Examples:**
- "Archive an order" (not delete, just change status?)
- "Send password reset email" (what resource is this?)
- "Calculate shipping cost" (read-only computation)

### 3.2 Approaches for Actions

**Option 1: Treat as State Change**
```http
PATCH /orders/123
{
    "status": "archived"
}
```

**Option 2: Sub-resource**
```http
POST /users/123/password-reset-requests
{
    "email": "user@example.com"
}
```

**Option 3: Action Endpoint (RPC-style)**
```http
POST /orders/123/archive
```

**Guidance:**
- Prefer state changes when possible
- Sub-resources for things that create side effects
- Action endpoints as last resort

### 3.3 Pagination

**The Problem:**
`GET /orders` returns 1 million orders? That's not going to work.

**Common Patterns:**

**Offset-based:**
```http
GET /orders?offset=100&limit=20
```
```json
{
    "data": [...],
    "pagination": {
        "total": 1000,
        "offset": 100,
        "limit": 20
    }
}
```

**Cursor-based:**
```http
GET /orders?cursor=eyJpZCI6MTIzfQ&limit=20
```
```json
{
    "data": [...],
    "pagination": {
        "next_cursor": "eyJpZCI6MTQzfQ",
        "has_more": true
    }
}
```

**Trade-offs:**
| Pattern | Pros | Cons |
|---------|------|------|
| Offset | Simple, supports "page N" | Slow on large datasets, inconsistent if data changes |
| Cursor | Fast, consistent | Can't jump to page N, cursor management |

### 3.4 Filtering, Sorting, and Search

**Filtering:**
```http
GET /orders?status=pending&created_after=2024-01-01
```

**Sorting:**
```http
GET /orders?sort=created_at&order=desc
# Or
GET /orders?sort=-created_at  # Prefix minus for descending
```

**Full-text Search:**
```http
GET /products?q=wireless+headphones
```

### 3.5 Expanding Related Resources

**The N+1 Problem:**
```http
GET /orders           → List of orders
GET /users/1          → User for order 1
GET /users/2          → User for order 2
...                   → 100 more requests
```

**Solutions:**

**Include/Expand:**
```http
GET /orders?include=user
```
```json
{
    "data": [{
        "id": 1,
        "total": 100,
        "user": {
            "id": 1,
            "name": "Alice"
        }
    }]
}
```

**Compound Documents (JSON:API style):**
```json
{
    "data": [{
        "id": 1,
        "total": 100,
        "relationships": {
            "user": {"id": 1}
        }
    }],
    "included": [
        {"type": "user", "id": 1, "name": "Alice"}
    ]
}
```

---

## Section 4: Request and Response Design

### 4.1 Request Bodies

**Content-Type:**
```http
POST /orders
Content-Type: application/json

{
    "items": [
        {"product_id": 123, "quantity": 2}
    ],
    "shipping_address": {
        "street": "123 Main St",
        "city": "Boston"
    }
}
```

**Design Principles:**
- Use clear field names
- Nest logically related fields
- Accept IDs for relationships, not embedded objects
- Be consistent with naming conventions (camelCase or snake_case, pick one)

### 4.2 Response Bodies

**Successful Response:**
```json
{
    "id": 456,
    "status": "pending",
    "total": 199.99,
    "items": [...],
    "created_at": "2024-01-15T10:30:00Z"
}
```

**Design Principles:**
- Return the created/updated resource
- Include computed fields (totals, timestamps)
- Use consistent date format (ISO 8601)
- Include resource URL for discoverability

### 4.3 Envelope vs. Raw Response

**Raw (no envelope):**
```json
{"id": 1, "name": "Alice"}
```

**Enveloped:**
```json
{
    "data": {"id": 1, "name": "Alice"},
    "meta": {"request_id": "abc123"}
}
```

**Trade-offs:**
- Raw: Simpler, cleaner for single resources
- Envelope: Consistent structure, room for metadata

**Recommendation:**
Envelopes for collections (need pagination metadata), raw for single resources.

### 4.4 Null, Missing, and Empty

**Be Clear About Meaning:**
```json
// Field is null - explicitly set to nothing
{"middle_name": null}

// Field is missing - not provided/unknown
{"first_name": "Alice", "last_name": "Smith"}

// Field is empty - explicitly empty
{"tags": []}
```

**Document the difference.** Is a missing field "use default" or "unchanged"?

---

## Section 5: Error Handling That Helps

### 5.1 Status Codes Convey Meaning

**Use Status Codes Correctly:**

**Success:**
- 200 OK - Request succeeded
- 201 Created - Resource created
- 204 No Content - Success, no body (DELETE)

**Client Errors:**
- 400 Bad Request - Malformed request
- 401 Unauthorized - Authentication required
- 403 Forbidden - Authenticated but not allowed
- 404 Not Found - Resource doesn't exist
- 409 Conflict - State conflict (e.g., duplicate)
- 422 Unprocessable Entity - Valid syntax, invalid semantics
- 429 Too Many Requests - Rate limited

**Server Errors:**
- 500 Internal Server Error - Unexpected failure
- 502 Bad Gateway - Upstream failure
- 503 Service Unavailable - Temporarily down
- 504 Gateway Timeout - Upstream timeout

### 5.2 Error Response Design

**Minimal Error:**
```json
{
    "error": "Invalid order"
}
```
*Not helpful. Which field? What's wrong?*

**Informative Error:**
```json
{
    "error": {
        "code": "VALIDATION_FAILED",
        "message": "The request contains invalid fields",
        "details": [
            {
                "field": "items[0].quantity",
                "message": "Quantity must be positive",
                "code": "POSITIVE_REQUIRED"
            },
            {
                "field": "shipping_address.zip",
                "message": "Invalid ZIP code format",
                "code": "INVALID_FORMAT"
            }
        ],
        "request_id": "req_abc123"
    }
}
```

### 5.3 Error Design Principles

**1. Machine-Readable Codes:**
```json
"code": "INSUFFICIENT_INVENTORY"
```
Clients can programmatically handle specific errors.

**2. Human-Readable Messages:**
```json
"message": "Only 3 units of SKU-123 are available"
```
Developers can understand during debugging.

**3. Point to the Problem:**
```json
"field": "items[2].quantity"
```
Exact location of the issue.

**4. Suggest Solutions:**
```json
"suggestion": "Reduce quantity or check back later"
```
When possible, help users fix the problem.

**5. Include Request ID:**
```json
"request_id": "req_abc123"
```
Enables correlation with logs for support.

### 5.4 Validation Errors

**Comprehensive Validation Response:**
```http
POST /users
Content-Type: application/json

{"email": "invalid", "age": -5}
```

```http
HTTP/1.1 422 Unprocessable Entity

{
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Request validation failed",
        "errors": [
            {
                "field": "email",
                "code": "INVALID_EMAIL",
                "message": "Must be a valid email address"
            },
            {
                "field": "age",
                "code": "MIN_VALUE",
                "message": "Must be at least 0",
                "context": {"min": 0, "actual": -5}
            }
        ]
    }
}
```

---

## Section 6: Versioning — The Inevitability of Change

### 6.1 Why Versioning Matters

**The Reality:**
Requirements change. You will need to modify your API. Clients depend on current behavior.

**The Challenge:**
How do you evolve without breaking existing integrations?

### 6.2 Versioning Strategies

**1. URL Path Versioning:**
```http
GET /v1/users
GET /v2/users
```

*Pros:* Explicit, easy to route, cache-friendly
*Cons:* URL pollution, implies complete API replacement

**2. Query Parameter:**
```http
GET /users?version=2
```

*Pros:* Clean URLs
*Cons:* Easy to forget, caching complications

**3. Header Versioning:**
```http
GET /users
Accept: application/vnd.myapi.v2+json
```

*Pros:* Clean URLs, proper use of HTTP
*Cons:* Hidden, harder to test, easy to forget

**4. No Versioning (Evolve Carefully):**
Add fields, never remove. Deprecate gracefully.

*Pros:* Simplest when possible
*Cons:* Accumulates cruft, some changes impossible

**Recommendation:**
URL versioning for major changes, careful evolution for minor changes.

### 6.3 What Constitutes a Breaking Change?

**Breaking Changes:**
- Removing a field
- Renaming a field
- Changing a field's type
- Changing error codes
- Changing URL structure
- Requiring new fields

**Non-Breaking Changes:**
- Adding optional fields
- Adding new endpoints
- Adding new optional parameters
- Adding new error codes (if clients handle unknown)

### 6.4 Deprecation Strategy

**Good Deprecation:**
1. **Announce** in documentation and headers
2. **Set timeline** (e.g., 6 months)
3. **Monitor usage** of deprecated endpoints
4. **Contact heavy users** directly
5. **Remove** after period

**Deprecation Headers:**
```http
HTTP/1.1 200 OK
Deprecation: Sun, 01 Jun 2025 00:00:00 GMT
Sunset: Sun, 01 Dec 2025 00:00:00 GMT
Link: </v2/users>; rel="successor-version"
```

### 6.5 Version Coexistence

**Running Multiple Versions:**
```
┌─────────────────────────────────────────────────────────────┐
│                       API Gateway                            │
├─────────────────────────────────────────────────────────────┤
│                           │                                  │
│    /v1/*  ────────────────┤                                  │
│                           ▼                                  │
│                    ┌─────────────┐                          │
│                    │  v1 Service │                          │
│                    │ (deprecated)│                          │
│                    └─────────────┘                          │
│                                                             │
│    /v2/*  ────────────────┤                                  │
│                           ▼                                  │
│                    ┌─────────────┐                          │
│                    │  v2 Service │                          │
│                    │  (current)  │                          │
│                    └─────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

---

## Section 7: Authentication and Authorization

### 7.1 Authentication vs. Authorization

**Authentication:** Who are you?
**Authorization:** What can you do?

### 7.2 Common Authentication Methods

**API Keys:**
```http
GET /orders
X-API-Key: sk_live_abc123
```

*Best for:* Server-to-server, simple integrations
*Risk:* If leaked, full access until revoked

**JWT (JSON Web Tokens):**
```http
GET /orders
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

*Best for:* Stateless auth, contains claims
*Risk:* Can't revoke until expiration

**OAuth 2.0:**
Full framework for delegated authorization.

*Best for:* Third-party access, user consent
*Complexity:* Significant implementation overhead

### 7.3 Authorization Patterns

**Role-Based:**
```
Admin: all operations
Manager: read/write own department
User: read/write own resources
Guest: read only
```

**Scope-Based (OAuth):**
```
orders:read - Read orders
orders:write - Create/modify orders
users:read - Read user data
```

**Resource-Based:**
```
Can user X perform action Y on resource Z?
```

### 7.4 Security Headers

**Always Include:**
```http
# Prevent credentials in responses from being cached
Cache-Control: no-store

# Prevent MIME type sniffing
X-Content-Type-Options: nosniff

# Rate limit information
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

---

## Section 8: Documentation as Product

### 8.1 Documentation Is Not Optional

**Reality:**
If it's not documented, it doesn't exist to your users.

**Good Documentation Includes:**
- Endpoint reference (every endpoint, every parameter)
- Authentication guide
- Error code reference
- Quick start guide
- Use case examples
- Changelog

### 8.2 OpenAPI (Swagger)

**Define API in YAML/JSON:**
```yaml
openapi: 3.0.0
info:
  title: Orders API
  version: 1.0.0
paths:
  /orders:
    get:
      summary: List orders
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, completed, cancelled]
      responses:
        '200':
          description: List of orders
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Order'
```

**Benefits:**
- Auto-generate documentation UI
- Generate client SDKs
- Enable testing tools
- Single source of truth

### 8.3 Interactive Documentation

**Swagger UI / ReDoc:**
- Try endpoints directly
- See request/response examples
- View schemas interactively

**Postman Collections:**
- Shareable API collections
- Environment variables
- Automated testing

### 8.4 Example-First Documentation

**Bad Documentation:**
```
POST /orders
Creates a new order.
Parameters: See schema.
```

**Good Documentation:**
```
POST /orders
Creates a new order with the specified items.

Example Request:
POST /orders
Content-Type: application/json
Authorization: Bearer <token>

{
    "items": [
        {"product_id": "prod_123", "quantity": 2},
        {"product_id": "prod_456", "quantity": 1}
    ],
    "shipping_address": {
        "name": "Alice Smith",
        "street": "123 Main St",
        "city": "Boston",
        "state": "MA",
        "zip": "02101"
    }
}

Example Response (201 Created):
{
    "id": "ord_789",
    "status": "pending",
    "total": 149.97,
    "items": [...],
    "created_at": "2024-01-15T10:30:00Z"
}

Common Errors:
- 400: Invalid request body
- 401: Missing or invalid authentication
- 422: Validation failed (see error details)
```

---

## Section 9: Rate Limiting and Quotas

### 9.1 Why Rate Limit?

**Protection Against:**
- Abuse and attacks
- Runaway clients (bugs that hammer your API)
- Resource exhaustion
- Cost management

### 9.2 Rate Limit Design

**Communicate Clearly:**
```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1640000000
```

**When Exceeded:**
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60

{
    "error": {
        "code": "RATE_LIMITED",
        "message": "Rate limit exceeded. Try again in 60 seconds."
    }
}
```

### 9.3 Rate Limit Strategies

**Per-Key Limits:**
```
Free tier: 100 requests/hour
Basic: 1,000 requests/hour
Pro: 10,000 requests/hour
```

**Per-Endpoint Limits:**
```
GET /users: 1000/hour (read-heavy is fine)
POST /orders: 100/hour (writes are expensive)
```

**Burst vs. Sustained:**
Allow short bursts but limit sustained rate.

---

## Section 10: Implementing in Spring Boot

### 10.1 Controller Design

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final OrderService orderService;

    @GetMapping
    public ResponseEntity<Page<OrderResponse>> listOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) OrderStatus status) {

        Page<Order> orders = orderService.findOrders(status, PageRequest.of(page, size));
        return ResponseEntity.ok(orders.map(OrderResponse::from));
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable Long id) {
        Order order = orderService.findById(id)
            .orElseThrow(() -> new OrderNotFoundException(id));
        return ResponseEntity.ok(OrderResponse.from(order));
    }

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request) {

        Order order = orderService.create(request);
        URI location = URI.create("/api/v1/orders/" + order.getId());
        return ResponseEntity.created(location).body(OrderResponse.from(order));
    }
}
```

### 10.2 Request/Response DTOs

```java
public record CreateOrderRequest(
    @NotEmpty List<OrderItemRequest> items,
    @Valid ShippingAddress shippingAddress
) {}

public record OrderItemRequest(
    @NotNull Long productId,
    @Min(1) int quantity
) {}

public record OrderResponse(
    Long id,
    String status,
    BigDecimal total,
    List<OrderItemResponse> items,
    Instant createdAt
) {
    public static OrderResponse from(Order order) {
        return new OrderResponse(
            order.getId(),
            order.getStatus().name(),
            order.getTotal(),
            order.getItems().stream().map(OrderItemResponse::from).toList(),
            order.getCreatedAt()
        );
    }
}
```

### 10.3 Exception Handling

```java
@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(
                "ORDER_NOT_FOUND",
                "Order not found: " + ex.getOrderId(),
                null
            ));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {

        List<FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> new FieldError(
                error.getField(),
                error.getDefaultMessage(),
                error.getCode()
            ))
            .toList();

        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(new ErrorResponse(
                "VALIDATION_ERROR",
                "Request validation failed",
                fieldErrors
            ));
    }
}
```

### 10.4 OpenAPI Integration

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Orders API")
                .version("1.0.0")
                .description("API for managing customer orders"))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")));
    }
}
```

---

## Section 11: Now You Understand Why...

Armed with this foundation, you can now understand:

- **Why consistent APIs feel good**: Predictability reduces cognitive load—developers learn patterns, not exceptions

- **Why error messages matter**: A good error message is documentation at the moment of need

- **Why versioning is hard**: You're balancing evolution with stability for unknown consumers

- **Why "RESTful" arguments happen**: REST is a style, not a specification—reasonable people disagree

- **Why documentation is never "done"**: APIs evolve; documentation must keep pace

- **Why rate limiting is essential**: Without limits, one bad client can take down your service

---

## Practical Exercises

### Exercise 1: Design Review
Take an existing API you use and evaluate:
- Are resources named consistently?
- Are error messages helpful?
- Is pagination consistent?
- How is versioning handled?

### Exercise 2: Error Redesign
```json
// Given this error:
{"error": "Bad request"}

// Redesign it for a failed order creation with:
// - Invalid email
// - Negative quantity
// - Missing required field
```

### Exercise 3: Version Migration Plan
You need to rename `userName` to `username` in responses.
Create a migration plan that doesn't break existing clients.

### Exercise 4: Document an Endpoint
Write complete documentation for this endpoint:
```java
@PostMapping("/users/{userId}/addresses")
public Address addAddress(@PathVariable Long userId, @RequestBody AddressRequest request)
```

Include: description, parameters, request/response examples, error cases.

---

## Key Takeaways

1. **APIs are promises**. Breaking changes break trust. Design carefully, evolve thoughtfully.

2. **Consistency trumps perfection**. A consistent 80% solution beats an inconsistent 100% solution.

3. **Errors are features**. Good error responses help developers succeed.

4. **Documentation is product**. Undocumented features don't exist to users.

5. **Plan for change**. Versioning strategy should exist before you need it.

---

## Looking Ahead

With APIs understood, Chapter 7 explores the **Data Layer**: How to think about persistence, when to use different databases, and how data access patterns affect your architecture.

---

## Chapter 6 Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         APIs AS CONTRACTS                                    │
│                                                                             │
│  REST RESOURCE PATTERNS                                                     │
│  ─────────────────────                                                      │
│                                                                             │
│   GET    /users          → List resources                                   │
│   GET    /users/{id}     → Get specific resource                           │
│   POST   /users          → Create resource (201 Created)                   │
│   PUT    /users/{id}     → Replace resource                                │
│   PATCH  /users/{id}     → Partial update                                  │
│   DELETE /users/{id}     → Delete resource (204 No Content)                │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ERROR RESPONSE DESIGN                                                      │
│  ────────────────────                                                       │
│                                                                             │
│   {                                                                         │
│     "error": {                                                              │
│       "code": "VALIDATION_ERROR",          ← Machine-readable              │
│       "message": "Request validation failed", ← Human-readable             │
│       "details": [                                                          │
│         {                                                                   │
│           "field": "email",                ← Where the problem is          │
│           "message": "Invalid email format" ← What's wrong                 │
│         }                                                                   │
│       ],                                                                    │
│       "request_id": "req_abc123"          ← For debugging                  │
│     }                                                                       │
│   }                                                                         │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  VERSIONING STRATEGIES                                                      │
│  ────────────────────                                                       │
│                                                                             │
│   URL Path:    GET /v1/users    GET /v2/users                              │
│   Header:      Accept: application/vnd.api.v2+json                         │
│   Query:       GET /users?version=2                                         │
│   Evolution:   Add fields, never remove (when possible)                    │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  THE API LIFECYCLE                                                          │
│  ─────────────────                                                          │
│                                                                             │
│   Design → Document → Implement → Test → Deploy → Monitor → Evolve         │
│     ↑                                                          │            │
│     └──────────────────────────────────────────────────────────┘            │
│                     (feedback loop)                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Next Chapter: Data Layer Thinking — Persistence strategies and database patterns*
