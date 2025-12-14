# Chapter 9: Microservices: The Trade-offs

> **First Principles Question**: Why would you choose to make your system distributed (with all the complexity from Chapter 8) when you could keep it as one application? What problems does microservices solve that justify this complexity?

---

## Chapter Overview

Microservices are everywhere in modern tech conversations. But the hype often obscures the reality: microservices are a trade-off, not a silver bullet. They solve specific problems at the cost of introducing others.

This chapter examines microservices through a first-principles lens—what problems they solve, what problems they create, and how to decide if they're right for your system.

**What readers will understand after this chapter:**
- What microservices actually are (and aren't)
- The specific problems microservices solve
- The specific problems microservices create
- When to use microservices vs. monoliths
- How to decompose a system into services
- The operational requirements for microservices

---

## Section 1: What Are Microservices?

### 1.1 A Definition

**Microservices:**
An architectural style where a system is composed of small, independent services that:
- Run in their own processes
- Communicate via network protocols (HTTP, gRPC, messaging)
- Can be deployed independently
- Are organized around business capabilities
- Own their own data

**What They're NOT:**
- Just "small services"
- Services with REST APIs
- Docker containers
- Any distributed system

### 1.2 The Contrast: Monolithic Architecture

**Monolith:**
A single deployment unit containing all application functionality.

```
┌─────────────────────────────────────────────┐
│                 Monolith                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────────┐ │
│  │  Users  │  │ Orders  │  │  Inventory  │ │
│  └────┬────┘  └────┬────┘  └──────┬──────┘ │
│       │            │              │         │
│       └────────────┼──────────────┘         │
│                    │                        │
│            ┌───────▼───────┐               │
│            │   Database    │               │
│            └───────────────┘               │
└─────────────────────────────────────────────┘
```

**Microservices:**
Multiple deployment units, each handling specific functionality.

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ User Service │   │Order Service │   │ Inv Service  │
│   ┌─────┐    │   │   ┌─────┐    │   │   ┌─────┐    │
│   │ DB  │    │   │   │ DB  │    │   │   │ DB  │    │
│   └─────┘    │   │   └─────┘    │   │   └─────┘    │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                   Network calls
```

### 1.3 Key Characteristics

**Independent Deployment:**
Change one service without redeploying others.

**Decentralized Data:**
Each service owns its data. No shared database.

**Technology Diversity:**
Each service can use different languages, frameworks, databases.

**Organized Around Business:**
Services map to business capabilities, not technical layers.

**Smart Endpoints, Dumb Pipes:**
Logic lives in services, communication is simple (HTTP, messaging).

---

## Section 2: Why Microservices Exist

### 2.1 The Scaling Story

**Monolith Scaling Problem:**
```
                    Traffic: 1000 req/s
                           │
                           ▼
┌─────────────────────────────────────────────┐
│                 Monolith                     │
│  Users: 10 req/s    Orders: 900 req/s      │
│  Inventory: 90 req/s                        │
└─────────────────────────────────────────────┘
                           │
                    Scale ENTIRE app
                           │
                           ▼
           [Monolith] [Monolith] [Monolith]

           Wasteful: Users module replicated
           when only Orders needed scaling
```

**Microservices Solution:**
```
Users Service: 1 instance (10 req/s)
Order Service: 10 instances (900 req/s)
Inventory Service: 2 instances (90 req/s)

Scale ONLY what needs scaling
```

### 2.2 The Team Scaling Story

**Conway's Law:**
> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations."

**Monolith with Large Team:**
```
50 developers → 1 codebase
- Merge conflicts constantly
- Changes affect everyone
- Coordination overhead dominates
- "Don't touch that file, someone else is working on it"
```

**Microservices with Teams:**
```
Team A (5 devs) → User Service
Team B (8 devs) → Order Service
Team C (6 devs) → Inventory Service

- Teams work independently
- Clear ownership
- Parallel development
- "You build it, you run it"
```

### 2.3 The Technology Evolution Story

**Monolith Technology Lock-in:**
```
2015: Built with Java 8, Spring 3, MySQL
2020: Want to use Java 11, Spring 5, PostgreSQL

Migration = Rewrite entire application
Risk = Breaking everything at once
```

**Microservices Technology Freedom:**
```
User Service: Java → Kotlin (gradual migration)
Order Service: Still Java 8 (if it works)
New Analytics Service: Python (best tool for job)

Migration = One service at a time
Risk = Contained to single service
```

### 2.4 The Deployment Story

**Monolith Deployment:**
```
1. Stop the entire application
2. Deploy new version
3. If bug: rollback EVERYTHING
4. Changes deployed together even if unrelated

Release Friday? Bold.
```

**Microservices Deployment:**
```
1. Deploy User Service v2.1
2. Other services unaffected
3. If bug: rollback only User Service
4. Independent release schedules

Release Friday? Sure, just one service.
```

---

## Section 3: What Problems Microservices Create

### 3.1 Distributed System Complexity

**Everything from Chapter 8 Applies:**
- Network failures
- Latency
- Partial failures
- Consistency challenges
- Debugging complexity

**What Was Simple Becomes Hard:**
```java
// Monolith: Simple function call
Order order = orderService.create(userId, items);
User user = userService.getById(userId);
// Instant, reliable, transactional

// Microservices: Network call
Order order = orderClient.create(userId, items);
// What if network fails?
// What if order service is down?
// What if it times out but actually succeeded?
```

### 3.2 Data Consistency Challenges

**Monolith with Single Database:**
```java
@Transactional
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    accountRepo.debit(fromId, amount);
    accountRepo.credit(toId, amount);
    // ACID transaction - both succeed or both fail
}
```

**Microservices with Separate Databases:**
```java
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    accountServiceA.debit(fromId, amount);  // Service A, Database A
    accountServiceB.credit(toId, amount);   // Service B, Database B
    // What if second call fails?
    // No transaction spanning services!
}
```

**Solutions Exist but Add Complexity:**
- Sagas (compensating transactions)
- Eventual consistency
- Event sourcing
- All require careful design and implementation

### 3.3 Operational Overhead

**Monolith Operations:**
```
Deploy: 1 application
Monitor: 1 application
Debug: 1 set of logs
Secure: 1 perimeter
```

**Microservices Operations:**
```
Deploy: 20+ services
Monitor: 20+ dashboards, distributed tracing
Debug: Logs scattered across services
Secure: Service-to-service authentication, network policies
```

**Required Infrastructure:**
- Service discovery
- Load balancing
- Configuration management
- Secret management
- Logging aggregation
- Distributed tracing
- API gateway
- Container orchestration

### 3.4 Testing Complexity

**Monolith Testing:**
```java
@Test
void testOrderCreation() {
    User user = userService.create("Alice");
    Order order = orderService.create(user.getId(), items);
    assertThat(order.getStatus()).isEqualTo("CREATED");
    // Everything in memory, fast, deterministic
}
```

**Microservices Testing:**
```java
@Test
void testOrderCreation() {
    // Need User Service running
    // Need Order Service running
    // Need Inventory Service running
    // Need databases for each
    // Network latency in tests
    // Flaky due to timing issues
}
```

**Testing Strategies Required:**
- Contract testing (Pact)
- Service virtualization
- Consumer-driven contracts
- Extensive integration test environments

### 3.5 Debugging Distributed Systems

**Monolith Debugging:**
```
Request → Stack trace → Problem found
Single thread, single process, single log file
```

**Microservices Debugging:**
```
Request → API Gateway → Service A → Service B → Service C → Database
                              ↓
                        Service D (async)
                              ↓
                        Service E

Where did it fail? Which service? Which instance?
Need: Correlation IDs, distributed tracing, log aggregation
```

---

## Section 4: The Decision Framework

### 4.1 Start with Why

**Wrong Reason: "Everyone is doing microservices"**
Netflix, Amazon, Google use microservices because they have:
- Thousands of engineers
- Millions of users
- Global scale requirements
- Mature operational practices

Your startup with 5 engineers doesn't have the same problems.

**Right Reason: You have specific problems microservices solve**
- Team scaling bottleneck
- Different scaling requirements per component
- Need for independent deployment
- Polyglot requirements

### 4.2 The Monolith-First Approach

**Martin Fowler's Advice:**
> "Don't even consider microservices unless you have a system that's too complex to manage as a monolith."

**Start with Monolith:**
```
1. Build monolith
2. Identify boundaries through usage
3. Extract services when you NEED to
4. You now know the domain, extraction is informed
```

**vs. Starting with Microservices:**
```
1. Guess service boundaries
2. Get boundaries wrong
3. Expensive refactoring across services
4. Distributed monolith (worst of both worlds)
```

### 4.3 Signs You Might Need Microservices

**Team Scaling Issues:**
- Multiple teams stepping on each other
- Long merge conflicts
- Coordination overhead dominates work
- Teams can't work independently

**Deployment Issues:**
- Simple changes require full deployment
- Risk of deployment is high
- Deployment takes too long
- Can't deploy often enough

**Scaling Issues:**
- One component needs different resources
- Can't scale parts independently
- Wasteful resource allocation

**Technology Issues:**
- Stuck on old technology
- Best tool for job is different per component
- Can't adopt new approaches

### 4.4 Signs You Probably Don't Need Microservices

**Team Size:**
- Fewer than 20-30 developers
- Single team owns everything
- Communication is not a bottleneck

**System Size:**
- Simple domain
- Few components
- Scaling needs are uniform

**Maturity:**
- Limited operational experience
- No CI/CD pipeline
- No monitoring/logging infrastructure
- No containerization experience

### 4.5 The Distributed Monolith Anti-pattern

**What It Looks Like:**
```
Service A ←→ Service B ←→ Service C
    ↕           ↕           ↕
    └───────────┴───────────┘

- Services coupled tightly
- Can't deploy independently
- Shared database
- Synchronous calls everywhere
```

**Result:**
All the complexity of distributed systems, none of the benefits of microservices.

**Cause:**
- Premature decomposition
- Wrong boundaries
- Insufficient domain understanding

---

## Section 5: Decomposition Strategies

### 5.1 Domain-Driven Design

**Bounded Contexts:**
A bounded context is a logical boundary within which a particular model is defined and applicable.

```
E-commerce Domain:
┌───────────────────────────────────────────────────────────┐
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │   Catalog   │  │   Orders    │  │   Shipping  │      │
│  │   Context   │  │   Context   │  │   Context   │      │
│  │             │  │             │  │             │      │
│  │  Product    │  │  Order      │  │  Shipment   │      │
│  │  Category   │  │  LineItem   │  │  Carrier    │      │
│  │  Price      │  │  Customer   │  │  Tracking   │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
│                                                           │
└───────────────────────────────────────────────────────────┘

Each context: Different meaning of "Product", "Customer", etc.
```

**Same Term, Different Meaning:**
```
In Catalog Context:
  Product = { name, description, price, images }

In Inventory Context:
  Product = { SKU, quantity, location, reorderPoint }

In Shipping Context:
  Product = { weight, dimensions, isFragile }
```

### 5.2 Identifying Boundaries

**Business Capability Mapping:**
1. List business capabilities (what the business does)
2. Group related capabilities
3. Each group might be a service

```
E-commerce Business Capabilities:
- Manage product catalog
- Process orders
- Handle payments
- Manage inventory
- Ship products
- Support customers

→ Potential Services:
  Catalog Service, Order Service, Payment Service,
  Inventory Service, Shipping Service, Support Service
```

**Event Storming:**
1. Map business events ("Order Placed", "Payment Received")
2. Group related events
3. Identify aggregates and bounded contexts

### 5.3 The Strangler Pattern

**Migrating from Monolith to Microservices:**

```
Phase 1: Monolith handles everything
┌─────────────────────────┐
│       Monolith          │ ← All traffic
└─────────────────────────┘

Phase 2: New service handles some routes
         ┌─────────────────┐
         │   New Service   │ ← /api/orders/*
         └─────────────────┘
                │
┌───────────────┼───────────┐
│       Monolith            │ ← Everything else
└───────────────────────────┘

Phase 3: Continue extracting
         ┌─────────────────┐
         │ Service A       │
         └─────────────────┘
         ┌─────────────────┐
         │ Service B       │
         └─────────────────┘
┌───────────────────────────┐
│  Monolith (shrinking)     │
└───────────────────────────┘

Phase N: Monolith gone
```

**Key Principles:**
- Never rewrite from scratch
- Extract one piece at a time
- Route traffic gradually
- Keep both running during transition

### 5.4 Service Size Guidelines

**Too Big (Mini-Monolith):**
- Multiple teams needed to work on it
- Changes in one area require testing everything
- Takes days to understand

**Too Small (Nano-Services):**
- Every service calls multiple others for simple operations
- Massive overhead
- No service can do anything useful alone

**Right Size:**
- One team can own it
- Single deployable unit makes sense
- Limited dependencies on other services
- Can be understood quickly

**Rule of Thumb:**
> "A service should be as big as my head" — James Lewis

If you can't hold the entire service's behavior in your head, it's too big.

---

## Section 6: Service Communication

### 6.1 Synchronous vs. Asynchronous

**Synchronous (Request/Response):**
```
Service A → Request → Service B
Service A ← Response ← Service B
         (waits)
```

**Pros:**
- Simple mental model
- Immediate response
- Easy to understand flow

**Cons:**
- Tight coupling
- A waits for B (latency adds up)
- If B is slow or down, A is affected

**Asynchronous (Events/Messages):**
```
Service A → Event → Message Queue → Service B
Service A continues working
         (doesn't wait)
```

**Pros:**
- Loose coupling
- A doesn't wait for B
- B can process at its own pace
- Better resilience

**Cons:**
- More complex
- Eventual consistency
- Harder to debug

### 6.2 Communication Patterns

**API Gateway:**
```
Client → API Gateway → Service A
                    → Service B
                    → Service C

Gateway handles:
- Authentication
- Rate limiting
- Request routing
- Protocol translation
```

**Service Mesh:**
```
┌────────────────┐      ┌────────────────┐
│   Service A    │      │   Service B    │
│   ┌────────┐   │      │   ┌────────┐   │
│   │ Proxy  │───┼──────┼───│ Proxy  │   │
│   └────────┘   │      │   └────────┘   │
└────────────────┘      └────────────────┘

Sidecar proxies handle:
- Service discovery
- Load balancing
- TLS encryption
- Retries, timeouts
- Observability
```

### 6.3 API Design for Services

**Backward Compatibility:**
```
// Version 1
GET /users/123
{ "name": "Alice", "email": "alice@example.com" }

// Version 2 (additive - safe)
GET /users/123
{ "name": "Alice", "email": "alice@example.com", "phone": "555-1234" }

// Breaking change (dangerous!)
GET /users/123
{ "fullName": "Alice Smith", "contactEmail": "alice@example.com" }
```

**Strategies:**
- Additive changes only (add fields, not remove)
- Version your APIs when breaking changes needed
- Consumer-driven contracts
- Deprecation periods

### 6.4 Event-Driven Communication

**Event Notification:**
```
Order Service publishes: "OrderCreated" { orderId: 123 }
Other services react by querying Order Service for details
```

**Event-Carried State Transfer:**
```
Order Service publishes: "OrderCreated" {
    orderId: 123,
    customerId: 456,
    items: [...],
    total: 99.99
}
Other services have all data they need
```

**Event Sourcing:**
```
Store events as source of truth:
- "OrderCreated"
- "ItemAdded"
- "ItemRemoved"
- "OrderConfirmed"
- "OrderShipped"

Current state = replay all events
```

---

## Section 7: Operational Requirements

### 7.1 Service Discovery

**The Problem:**
Service A needs to call Service B. Where is Service B?

**Static Configuration (Don't Do This):**
```yaml
services:
  order-service: 10.0.0.1:8080  # What if it moves?
```

**DNS-Based:**
```yaml
services:
  order-service: order-service.internal  # DNS resolves to current IPs
```

**Service Registry (Consul, Eureka):**
```java
// Service registers itself
registry.register("order-service", myHost, myPort);

// Client discovers
List<ServiceInstance> instances = registry.discover("order-service");
```

### 7.2 Configuration Management

**The Problem:**
20 services × 4 environments = 80 configurations

**Solutions:**
- Spring Cloud Config
- HashiCorp Consul
- Kubernetes ConfigMaps
- Environment variables from secrets manager

```yaml
# application.yml (externalized)
database:
  url: ${DB_URL}
  password: ${DB_PASSWORD}  # From secrets manager

feature-flags:
  new-checkout: ${FEATURE_NEW_CHECKOUT:false}
```

### 7.3 Observability

**Three Pillars:**

**Logs:**
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "service": "order-service",
  "level": "INFO",
  "message": "Order created",
  "orderId": "12345",
  "traceId": "abc-def-123"  // Correlation!
}
```

**Metrics:**
```
order_service_requests_total{status="200"} 15234
order_service_request_duration_seconds{quantile="0.99"} 0.45
order_service_errors_total{type="timeout"} 12
```

**Traces:**
```
Trace abc-def-123:
├── API Gateway (2ms)
│   └── Order Service (45ms)
│       ├── Inventory Service (15ms)
│       └── Payment Service (25ms)
│           └── Database (10ms)
```

### 7.4 Resilience Patterns

**Circuit Breaker:**
```java
@CircuitBreaker(name = "inventory", fallbackMethod = "fallback")
public InventoryStatus checkInventory(String productId) {
    return inventoryClient.check(productId);
}

public InventoryStatus fallback(String productId, Exception e) {
    return InventoryStatus.UNKNOWN;  // Degrade gracefully
}
```

**Retry with Backoff:**
```java
@Retry(name = "inventory", maxAttempts = 3, backoff = @Backoff(delay = 100))
public InventoryStatus checkInventory(String productId) {
    return inventoryClient.check(productId);
}
```

**Bulkhead:**
```java
// Limit concurrent calls to prevent cascade failures
@Bulkhead(name = "inventory", maxConcurrent = 10)
public InventoryStatus checkInventory(String productId) {
    return inventoryClient.check(productId);
}
```

### 7.5 Deployment Strategies

**Blue-Green:**
```
             ┌─────────────────┐
             │  Load Balancer  │
             └────────┬────────┘
                      │ (switch)
        ┌─────────────┴─────────────┐
        ▼                           ▼
┌──────────────┐            ┌──────────────┐
│  Blue (v1)   │            │  Green (v2)  │
│   (live)     │            │  (standby)   │
└──────────────┘            └──────────────┘
```

**Canary:**
```
             ┌─────────────────┐
             │  Load Balancer  │
             └────────┬────────┘
                      │ (percentage)
        ┌─────────────┴─────────────┐
        ▼ 90%                   10% ▼
┌──────────────┐            ┌──────────────┐
│  Stable (v1) │            │  Canary (v2) │
└──────────────┘            └──────────────┘
```

---

## Section 8: Real-World Considerations

### 8.1 Data Ownership

**Rule: Each Service Owns Its Data**

**Wrong (Shared Database):**
```
Order Service ─┐
               ├──→ Shared Database
User Service  ─┘

Problems:
- Schema changes affect multiple services
- No independent deployment
- Tight coupling
```

**Right (Separate Databases):**
```
Order Service ──→ Order Database
User Service  ──→ User Database

How to get user info for order?
- API call to User Service
- Event-based data sync
- Read-only replica (carefully!)
```

### 8.2 Managing Shared Code

**The Temptation:**
```
common-library/
├── models/
├── utils/
└── services/

Every service depends on common-library
```

**The Problem:**
Change to common-library requires updating ALL services. You've recreated monolith coupling.

**Better Approach:**
- Copy code (yes, duplication is okay)
- Small, focused libraries with strict versioning
- Service-specific DTOs (not shared models)
- Consumer-driven contracts

### 8.3 Security

**Service-to-Service Authentication:**
```
Service A → "I am Service A" → Service B
            (prove it)

Methods:
- mTLS (mutual TLS)
- JWT tokens
- API keys
- Service mesh handles it
```

**Defense in Depth:**
```
┌─────────────────────────────────────────┐
│  Network Policies (allow-list traffic)  │
│  ┌───────────────────────────────────┐  │
│  │  mTLS (encrypted, authenticated)  │  │
│  │  ┌─────────────────────────────┐  │  │
│  │  │  Authorization (can A call B?)│  │  │
│  │  │  ┌───────────────────────┐  │  │  │
│  │  │  │  Service Logic        │  │  │  │
│  │  │  └───────────────────────┘  │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### 8.4 Team Organization

**Inverse Conway Maneuver:**
Organize teams to match desired architecture.

```
Desired Architecture:
┌────────────┐ ┌────────────┐ ┌────────────┐
│ Order Svc  │ │ User Svc   │ │ Inv Svc    │
└────────────┘ └────────────┘ └────────────┘

Team Organization:
┌────────────┐ ┌────────────┐ ┌────────────┐
│ Order Team │ │ User Team  │ │ Inv Team   │
│ (owns svc) │ │ (owns svc) │ │ (owns svc) │
└────────────┘ └────────────┘ └────────────┘
```

**"You Build It, You Run It":**
Team owns service from development through production operations.

---

## Section 9: Now You Understand Why...

Armed with this foundation, you can now understand:

- **Why companies move to microservices slowly**: It's a trade-off, not a destination. Extract when you have the problem.

- **Why microservices "failures" happen**: Usually premature decomposition, wrong boundaries, or insufficient operational maturity.

- **Why Netflix/Amazon succeed with microservices**: They have the scale to need it and the engineering culture to support it.

- **Why "monolith" isn't a dirty word**: A well-structured monolith can be excellent for many teams.

- **Why service boundaries matter so much**: Wrong boundaries mean distributed monolith—worst of both worlds.

- **Why operations becomes so important**: Microservices move complexity from code to operations.

---

## Practical Exercises

### Exercise 1: Evaluate Your System
For your current system:
1. How many developers work on it?
2. What are the deployment pain points?
3. What would benefit from independent scaling?
4. What are the bounded contexts?

Should it be microservices?

### Exercise 2: Draw the Boundaries
For an e-commerce system:
1. List business capabilities
2. Group into potential services
3. Identify data owned by each
4. Map communication between them

### Exercise 3: Simulate Failure
If using a microservices system:
1. Kill one service
2. What happens to dependent services?
3. Does the circuit breaker work?
4. What do users see?

### Exercise 4: Trace a Request
For a distributed request:
1. Set up distributed tracing
2. Make a request that spans services
3. Identify the critical path
4. Where is time spent?

---

## Key Takeaways

1. **Microservices are a trade-off**, not an upgrade. They solve specific problems and create others.

2. **Start with a monolith** unless you have clear reasons not to. Extract services when you understand the domain.

3. **Team structure matters**. Microservices work best when teams own services end-to-end.

4. **Operational maturity is required**. Without good CI/CD, monitoring, and deployment practices, microservices multiply problems.

5. **Boundaries are critical**. Wrong boundaries lead to distributed monoliths. Use domain-driven design.

6. **Data ownership is non-negotiable**. Shared databases couple services and prevent independent deployment.

---

## Looking Ahead

Understanding the trade-offs of microservices prepares you for the practical question: how do services communicate? Chapter 10 dives into service communication patterns—synchronous, asynchronous, and everything in between.

---

## Chapter 9 Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     MICROSERVICES: THE TRADE-OFFS                            │
│                                                                             │
│  WHAT MICROSERVICES SOLVE                 WHAT THEY CREATE                  │
│  ───────────────────────                  ────────────────────              │
│  ✓ Independent deployment                 ✗ Network complexity              │
│  ✓ Team autonomy                          ✗ Data consistency               │
│  ✓ Targeted scaling                       ✗ Operational overhead           │
│  ✓ Technology diversity                   ✗ Testing complexity             │
│  ✓ Fault isolation                        ✗ Debugging difficulty           │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  DECISION FRAMEWORK                                                         │
│  ──────────────────                                                         │
│                                                                             │
│  Consider Microservices If:              Stick with Monolith If:            │
│  • 20+ developers                        • < 20 developers                  │
│  • Teams blocking each other             • Single team                      │
│  • Need independent deployment           • Simple deployment needs          │
│  • Different scaling needs               • Uniform scaling                  │
│  • Strong operational maturity           • Limited ops experience           │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  DECOMPOSITION STRATEGIES                                                    │
│  ────────────────────────                                                    │
│                                                                             │
│  1. Domain-Driven Design                                                    │
│     Identify bounded contexts → Services                                    │
│                                                                             │
│  2. Business Capability Mapping                                             │
│     What does the business do? → Group into services                       │
│                                                                             │
│  3. Strangler Pattern                                                       │
│     Gradually extract from monolith → Never big-bang rewrite               │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  OPERATIONAL REQUIREMENTS                                                    │
│  ─────────────────────────                                                   │
│                                                                             │
│  ┌─────────────────┬────────────────────┬─────────────────┐                │
│  │Service Discovery│ Config Management  │  Observability  │                │
│  │                 │                    │                 │                │
│  │ Consul/Eureka   │ Spring Cloud Config│ Logs + Metrics  │                │
│  │ DNS             │ ConfigMaps         │ + Traces        │                │
│  └─────────────────┴────────────────────┴─────────────────┘                │
│                                                                             │
│  ┌─────────────────┬────────────────────┬─────────────────┐                │
│  │ Circuit Breaker │ Retry + Backoff    │    Bulkhead     │                │
│  │                 │                    │                 │                │
│  │ Stop calling    │ Retry transient    │ Limit impact of │                │
│  │ failing service │ failures           │ slow services   │                │
│  └─────────────────┴────────────────────┴─────────────────┘                │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  THE FUNDAMENTAL RULE                                                        │
│  ────────────────────                                                        │
│                                                                             │
│        ┌──────────────────────────────────────────────────────┐            │
│        │  Monolith First: Extract services when you have      │            │
│        │  specific problems that microservices solve.          │            │
│        │                                                       │            │
│        │  Wrong boundaries = Distributed Monolith              │            │
│        │  (Worst of both worlds)                               │            │
│        └──────────────────────────────────────────────────────┘            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Next Chapter: Service Communication Patterns — How services talk to each other effectively*
