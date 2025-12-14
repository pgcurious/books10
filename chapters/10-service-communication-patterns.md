# Chapter 10: Service Communication Patterns

> **First Principles Question**: When services need to exchange information, what are the fundamental ways they can communicate? What trade-offs does each approach make?

---

## Chapter Overview

In a microservices architecture, services must talk to each other. This seems simple—just make HTTP calls, right? But the choice of communication pattern has profound implications for coupling, resilience, consistency, and complexity.

This chapter explores the fundamental patterns for service communication, helping you understand when to use each and the consequences of those choices.

**What readers will understand after this chapter:**
- Synchronous vs. asynchronous communication trade-offs
- REST, gRPC, and GraphQL for synchronous communication
- Message queues and event-driven architecture
- Event patterns: notification, state transfer, sourcing
- Choreography vs. orchestration
- Practical implementation considerations

---

## Section 1: The Communication Spectrum

### 1.1 Synchronous Communication

**Definition:**
The caller waits for a response before continuing.

```
Service A                    Service B
    │                            │
    │────── Request ────────────►│
    │         (waits)            │ (processes)
    │◄───── Response ────────────│
    │                            │
    ▼ continues                  │
```

**Characteristics:**
- Simple mental model
- Immediate feedback
- Tight temporal coupling
- Caller blocked during call

**Examples:**
- HTTP/REST calls
- gRPC calls
- GraphQL queries

### 1.2 Asynchronous Communication

**Definition:**
The caller sends a message and continues without waiting for a response.

```
Service A                    Broker                    Service B
    │                           │                          │
    │──── Publish message ─────►│                          │
    │                           │                          │
    ▼ continues immediately     │──── Deliver message ────►│
                                │                          │ (processes)
```

**Characteristics:**
- Decoupled in time
- Higher complexity
- Better resilience
- Eventual consistency

**Examples:**
- Message queues (RabbitMQ, SQS)
- Event streaming (Kafka)
- Pub/sub systems

### 1.3 The Coupling Trade-off

**Synchronous (Tighter Coupling):**
```
┌───────────────────────────────────────────────────────┐
│ Service A needs Service B to be:                      │
│   - Running                                          │
│   - Responsive                                       │
│   - Compatible API                                   │
│   - Available NOW                                    │
└───────────────────────────────────────────────────────┘
```

**Asynchronous (Looser Coupling):**
```
┌───────────────────────────────────────────────────────┐
│ Service A needs:                                      │
│   - Message broker available                         │
│   - Message format agreement                         │
│   - That's it—B can process later                   │
└───────────────────────────────────────────────────────┘
```

---

## Section 2: Synchronous Patterns

### 2.1 REST (Representational State Transfer)

**The Dominant Approach:**
HTTP-based APIs using standard methods and resource-based URLs.

```http
GET /users/123
POST /orders
PUT /users/123
DELETE /orders/456
```

**REST Principles:**
- **Resources**: Nouns, not verbs (`/users`, not `/getUser`)
- **HTTP Methods**: GET (read), POST (create), PUT (update), DELETE (remove)
- **Stateless**: Each request contains all information needed
- **Uniform Interface**: Standard patterns across APIs

**Example Service:**
```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable Long id) {
        return orderService.findById(id)
            .map(OrderResponse::from)
            .orElseThrow(() -> new NotFoundException("Order not found"));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(@RequestBody CreateOrderRequest request) {
        Order order = orderService.create(request);
        return OrderResponse.from(order);
    }

    @PutMapping("/{id}/status")
    public OrderResponse updateStatus(
            @PathVariable Long id,
            @RequestBody UpdateStatusRequest request) {
        Order order = orderService.updateStatus(id, request.getStatus());
        return OrderResponse.from(order);
    }
}
```

**Calling Another Service:**
```java
@Service
public class OrderService {

    private final RestTemplate restTemplate;

    public UserInfo getUserForOrder(Long userId) {
        return restTemplate.getForObject(
            "http://user-service/users/{id}",
            UserInfo.class,
            userId
        );
    }
}
```

**REST Strengths:**
- Universal (HTTP everywhere)
- Human-readable (can use curl, browser)
- Well-understood caching (HTTP caching)
- Stateless scalability

**REST Weaknesses:**
- Over-fetching (get entire resource when you need one field)
- Under-fetching (need multiple calls for related data)
- No streaming
- Text-based (less efficient than binary)

### 2.2 gRPC

**Binary Protocol:**
Google's high-performance RPC framework using Protocol Buffers.

**Define the Contract (Proto file):**
```protobuf
syntax = "proto3";

package orders;

service OrderService {
    rpc GetOrder(GetOrderRequest) returns (Order);
    rpc CreateOrder(CreateOrderRequest) returns (Order);
    rpc StreamOrders(StreamRequest) returns (stream Order);
}

message GetOrderRequest {
    int64 order_id = 1;
}

message Order {
    int64 id = 1;
    string customer_id = 2;
    repeated OrderItem items = 3;
    OrderStatus status = 4;
    google.protobuf.Timestamp created_at = 5;
}

enum OrderStatus {
    PENDING = 0;
    CONFIRMED = 1;
    SHIPPED = 2;
    DELIVERED = 3;
}
```

**Server Implementation:**
```java
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {

    @Override
    public void getOrder(GetOrderRequest request, StreamObserver<Order> responseObserver) {
        Order order = orderRepository.findById(request.getOrderId());
        responseObserver.onNext(order);
        responseObserver.onCompleted();
    }

    @Override
    public void streamOrders(StreamRequest request, StreamObserver<Order> responseObserver) {
        orderRepository.findByCustomerId(request.getCustomerId())
            .forEach(responseObserver::onNext);
        responseObserver.onCompleted();
    }
}
```

**Client Usage:**
```java
public class OrderClient {

    private final OrderServiceGrpc.OrderServiceBlockingStub stub;

    public Order getOrder(long orderId) {
        return stub.getOrder(GetOrderRequest.newBuilder()
            .setOrderId(orderId)
            .build());
    }
}
```

**gRPC Strengths:**
- High performance (binary, HTTP/2)
- Strong typing (generated code)
- Streaming support (bidirectional)
- Multiple language support
- Built-in code generation

**gRPC Weaknesses:**
- Not browser-friendly (need gRPC-Web)
- Harder to debug (binary)
- More setup complexity
- Less universal than HTTP/JSON

**When to Use gRPC:**
- Internal service-to-service communication
- High-throughput requirements
- Streaming needed
- Strong contract requirements

### 2.3 GraphQL

**Query Language for APIs:**
Clients request exactly the data they need.

**Schema Definition:**
```graphql
type Query {
    user(id: ID!): User
    order(id: ID!): Order
    orders(userId: ID!, status: OrderStatus): [Order!]!
}

type User {
    id: ID!
    name: String!
    email: String!
    orders: [Order!]!
}

type Order {
    id: ID!
    user: User!
    items: [OrderItem!]!
    status: OrderStatus!
    total: Float!
    createdAt: DateTime!
}

type OrderItem {
    product: Product!
    quantity: Int!
    unitPrice: Float!
}

enum OrderStatus {
    PENDING
    CONFIRMED
    SHIPPED
    DELIVERED
}
```

**Client Query:**
```graphql
query GetOrderWithUser($orderId: ID!) {
    order(id: $orderId) {
        id
        status
        total
        user {
            name
            email
        }
        items {
            product {
                name
            }
            quantity
        }
    }
}
```

**Response (exactly what was requested):**
```json
{
    "data": {
        "order": {
            "id": "123",
            "status": "CONFIRMED",
            "total": 99.99,
            "user": {
                "name": "Alice",
                "email": "alice@example.com"
            },
            "items": [
                { "product": { "name": "Widget" }, "quantity": 2 }
            ]
        }
    }
}
```

**GraphQL Strengths:**
- No over-fetching (get only requested fields)
- No under-fetching (get related data in one request)
- Strong typing
- Self-documenting (introspection)
- Great for frontend flexibility

**GraphQL Weaknesses:**
- Complexity on server
- N+1 query problems if not careful
- Caching more difficult
- Learning curve
- Potentially unbounded queries

**When to Use GraphQL:**
- Client needs flexibility in data requests
- Multiple clients with different data needs
- Mobile apps (bandwidth matters)
- Aggregating data from multiple services

### 2.4 Comparison Matrix

| Aspect | REST | gRPC | GraphQL |
|--------|------|------|---------|
| Performance | Good | Excellent | Good |
| Flexibility | Fixed endpoints | Fixed methods | Very flexible |
| Learning Curve | Low | Medium | Medium-High |
| Browser Support | Excellent | Limited | Excellent |
| Debugging | Easy | Hard | Medium |
| Streaming | Limited | Excellent | Via subscriptions |
| Caching | HTTP caching | Custom | Custom |
| Use Case | Public APIs | Internal services | BFF/Aggregation |

---

## Section 3: Asynchronous Patterns

### 3.1 Message Queues

**Point-to-Point Communication:**
```
Producer ──► Queue ──► Consumer

Message delivered to ONE consumer
```

**Example: Order Processing**
```
┌────────────────┐      ┌─────────────┐      ┌─────────────────┐
│ Order Service  │─────►│ Order Queue │─────►│ Fulfillment Svc │
└────────────────┘      └─────────────┘      └─────────────────┘
                         [Order 1]
                         [Order 2]
                         [Order 3]
```

**RabbitMQ Example:**
```java
// Producer
@Service
public class OrderProducer {

    private final RabbitTemplate rabbitTemplate;

    public void sendOrder(Order order) {
        rabbitTemplate.convertAndSend(
            "orders.exchange",
            "orders.created",
            order
        );
    }
}

// Consumer
@Service
public class FulfillmentConsumer {

    @RabbitListener(queues = "orders.fulfillment")
    public void handleOrder(Order order) {
        fulfillmentService.process(order);
    }
}
```

**Queue Semantics:**
- **At-most-once**: Message may be lost, never duplicated
- **At-least-once**: Message delivered, may be duplicated
- **Exactly-once**: Hardest to achieve, usually through idempotency

### 3.2 Publish/Subscribe

**One-to-Many Communication:**
```
Publisher ──► Topic ──► Subscriber A
                   ├──► Subscriber B
                   └──► Subscriber C

Message delivered to ALL subscribers
```

**Example: Order Events**
```
┌────────────────┐      ┌─────────────────┐
│ Order Service  │─────►│ OrderCreated    │
└────────────────┘      │ Topic           │
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
    │ Inventory Svc   │ │ Notification Svc│ │ Analytics Svc   │
    │ (reserve stock) │ │ (email customer)│ │ (record metrics)│
    └─────────────────┘ └─────────────────┘ └─────────────────┘
```

**Spring Cloud Stream Example:**
```java
// Publisher
@Service
public class OrderEventPublisher {

    private final StreamBridge streamBridge;

    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            Instant.now()
        );
        streamBridge.send("orderCreated-out-0", event);
    }
}

// Subscriber
@Configuration
public class InventoryEventHandler {

    @Bean
    public Consumer<OrderCreatedEvent> orderCreated() {
        return event -> {
            inventoryService.reserveItems(event.getItems());
        };
    }
}
```

### 3.3 Event Streaming (Kafka)

**Persistent, Ordered Log:**
```
Partition 0: [E1] [E2] [E3] [E4] [E5] ──►
Partition 1: [E1] [E2] [E3] ──►
Partition 2: [E1] [E2] [E3] [E4] ──►
                                    │
                            (consumers read)
```

**Key Differences from Queues:**
- Events persist (configurable retention)
- Consumers can replay from any point
- Ordering guaranteed within partition
- Consumer groups for scaling

**Kafka Producer:**
```java
@Service
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderEvent(OrderEvent event) {
        kafkaTemplate.send(
            "orders",                    // topic
            event.getOrderId(),          // key (for partitioning)
            event                        // value
        );
    }
}
```

**Kafka Consumer:**
```java
@Service
public class OrderEventConsumer {

    @KafkaListener(
        topics = "orders",
        groupId = "inventory-service"
    )
    public void handleOrderEvent(OrderEvent event) {
        if (event instanceof OrderCreated created) {
            inventoryService.reserve(created.getItems());
        } else if (event instanceof OrderCancelled cancelled) {
            inventoryService.release(cancelled.getItems());
        }
    }
}
```

**Consumer Groups:**
```
Topic: orders (3 partitions)

Consumer Group: inventory-service
├── Instance 1: reads partition 0
├── Instance 2: reads partition 1
└── Instance 3: reads partition 2

Consumer Group: analytics-service
├── Instance 1: reads partitions 0, 1
└── Instance 2: reads partition 2

Each group sees ALL messages
Within a group, partitions distributed
```

---

## Section 4: Event Patterns

### 4.1 Event Notification

**Minimal Event:**
"Something happened, query me for details."

```json
{
    "eventType": "OrderCreated",
    "orderId": "12345",
    "timestamp": "2024-01-15T10:30:00Z"
}
```

**Subscriber Response:**
```java
public void handleOrderCreated(OrderCreatedNotification notification) {
    // Need details, so call the Order Service
    Order order = orderServiceClient.getOrder(notification.getOrderId());
    // Now process with full order data
}
```

**Pros:**
- Small event size
- Source of truth stays with publisher
- Easy to implement

**Cons:**
- Subscribers need to call back for data
- Publisher must handle callback load
- Temporal coupling (publisher must be available)

### 4.2 Event-Carried State Transfer

**Full Data in Event:**
"Here's everything you need to know."

```json
{
    "eventType": "OrderCreated",
    "orderId": "12345",
    "customerId": "cust-789",
    "items": [
        {"productId": "prod-1", "quantity": 2, "unitPrice": 29.99},
        {"productId": "prod-2", "quantity": 1, "unitPrice": 49.99}
    ],
    "shippingAddress": {
        "street": "123 Main St",
        "city": "Boston",
        "zip": "02101"
    },
    "total": 109.97,
    "timestamp": "2024-01-15T10:30:00Z"
}
```

**Subscriber Response:**
```java
public void handleOrderCreated(OrderCreatedEvent event) {
    // All data is in the event, no callback needed
    inventoryService.reserve(event.getItems());
    shippingService.prepareLabel(event.getShippingAddress());
}
```

**Pros:**
- No callback to publisher needed
- Better decoupling
- Subscribers can build local projections

**Cons:**
- Larger events
- Data might be stale by time of processing
- Versioning becomes important

### 4.3 Event Sourcing

**Events AS the Source of Truth:**
Instead of storing current state, store the sequence of events that led to it.

```
Order Aggregate Events:
┌────────────────────────────────────────────────────┐
│ 1. OrderCreated { orderId: 123, items: [...] }     │
│ 2. ItemAdded { orderId: 123, item: {...} }         │
│ 3. ItemRemoved { orderId: 123, itemId: 456 }       │
│ 4. OrderConfirmed { orderId: 123 }                 │
│ 5. PaymentReceived { orderId: 123, amount: 99.99 } │
│ 6. OrderShipped { orderId: 123, trackingNo: "..." }│
└────────────────────────────────────────────────────┘

Current State = Replay all events
```

**Implementation Pattern:**
```java
public class OrderAggregate {
    private Long id;
    private List<OrderItem> items = new ArrayList<>();
    private OrderStatus status;
    private BigDecimal total;

    // Apply events to build state
    public void apply(OrderEvent event) {
        if (event instanceof OrderCreated e) {
            this.id = e.getOrderId();
            this.items = new ArrayList<>(e.getItems());
            this.status = OrderStatus.CREATED;
            this.total = calculateTotal();
        } else if (event instanceof ItemAdded e) {
            this.items.add(e.getItem());
            this.total = calculateTotal();
        } else if (event instanceof OrderConfirmed e) {
            this.status = OrderStatus.CONFIRMED;
        }
        // ... other events
    }

    // Rebuild from event stream
    public static OrderAggregate rebuild(List<OrderEvent> events) {
        OrderAggregate order = new OrderAggregate();
        events.forEach(order::apply);
        return order;
    }
}
```

**Event Store:**
```java
public interface EventStore {

    void append(String aggregateId, List<Event> events, long expectedVersion);

    List<Event> load(String aggregateId);

    List<Event> loadFrom(String aggregateId, long fromVersion);
}
```

**Pros:**
- Complete audit trail
- Can rebuild any past state
- Natural fit for event-driven systems
- Enables temporal queries ("what was the order on Jan 1?")

**Cons:**
- Complexity (need event store, snapshots)
- Event versioning challenges
- Eventual consistency with read models
- Learning curve

### 4.4 CQRS (Command Query Responsibility Segregation)

**Separate Models for Read and Write:**
```
┌─────────────────┐     Commands      ┌─────────────────┐
│   Client        │─────────────────►│  Write Model     │
│                 │                   │  (Event Store)   │
│                 │                   └────────┬─────────┘
│                 │                            │ Events
│                 │     Queries       ┌────────▼─────────┐
│                 │◄─────────────────│  Read Model(s)   │
│                 │                   │  (Projections)   │
└─────────────────┘                   └──────────────────┘
```

**Write Side (Commands):**
```java
@Service
public class OrderCommandHandler {

    private final EventStore eventStore;

    @Transactional
    public void handle(CreateOrderCommand command) {
        OrderAggregate order = new OrderAggregate();
        List<OrderEvent> events = order.createOrder(
            command.getCustomerId(),
            command.getItems()
        );
        eventStore.append(command.getOrderId(), events, 0);
    }
}
```

**Read Side (Projections):**
```java
@Component
public class OrderProjection {

    private final OrderReadRepository readRepository;

    @EventHandler
    public void on(OrderCreated event) {
        OrderView view = new OrderView(
            event.getOrderId(),
            event.getCustomerId(),
            event.getItems(),
            OrderStatus.CREATED
        );
        readRepository.save(view);
    }

    @EventHandler
    public void on(OrderShipped event) {
        readRepository.updateStatus(
            event.getOrderId(),
            OrderStatus.SHIPPED
        );
    }
}
```

**When to Use CQRS:**
- Read and write patterns differ significantly
- Need multiple read models (list view, detail view, reporting)
- Event sourcing (natural fit)
- High-scale reads with simpler writes

---

## Section 5: Orchestration vs. Choreography

### 5.1 Orchestration

**Central Coordinator Controls Flow:**
```
                    ┌─────────────────┐
                    │   Orchestrator  │
                    │  (Saga Manager) │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │Order Service │ │Payment Svc   │ │Shipping Svc  │
    └──────────────┘ └──────────────┘ └──────────────┘
```

**Orchestrator Code:**
```java
@Service
public class OrderSagaOrchestrator {

    public void processOrder(CreateOrderRequest request) {
        // Step 1: Create order
        Order order = orderService.create(request);

        try {
            // Step 2: Reserve inventory
            inventoryService.reserve(order.getItems());

            try {
                // Step 3: Process payment
                paymentService.charge(order.getTotal(), request.getPaymentInfo());

                try {
                    // Step 4: Schedule shipping
                    shippingService.schedule(order);

                    // Success!
                    orderService.confirm(order.getId());

                } catch (ShippingException e) {
                    paymentService.refund(order.getId());
                    inventoryService.release(order.getItems());
                    orderService.cancel(order.getId());
                    throw e;
                }
            } catch (PaymentException e) {
                inventoryService.release(order.getItems());
                orderService.cancel(order.getId());
                throw e;
            }
        } catch (InventoryException e) {
            orderService.cancel(order.getId());
            throw e;
        }
    }
}
```

**Pros:**
- Easy to understand flow
- Centralized error handling
- Clear ownership of process

**Cons:**
- Single point of failure
- Orchestrator becomes complex
- Services coupled to orchestrator
- Can become bottleneck

### 5.2 Choreography

**Services React to Events:**
```
OrderCreated ──► Inventory Service ──► InventoryReserved
                                              │
                                              ▼
                     Payment Service ◄────────┘
                           │
                           ▼
                    PaymentProcessed
                           │
                           ▼
                   Shipping Service ──► ShipmentScheduled
```

**Event-Driven Services:**
```java
// Order Service
@Service
public class OrderService {

    public Order create(CreateOrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        eventPublisher.publish(new OrderCreated(order));
        return order;
    }

    @EventHandler
    public void on(PaymentFailed event) {
        orderRepository.updateStatus(event.getOrderId(), CANCELLED);
    }
}

// Inventory Service
@Service
public class InventoryService {

    @EventHandler
    public void on(OrderCreated event) {
        boolean reserved = reserve(event.getItems());
        if (reserved) {
            eventPublisher.publish(new InventoryReserved(event.getOrderId()));
        } else {
            eventPublisher.publish(new InventoryInsufficient(event.getOrderId()));
        }
    }

    @EventHandler
    public void on(PaymentFailed event) {
        release(event.getOrderId());
    }
}

// Payment Service
@Service
public class PaymentService {

    @EventHandler
    public void on(InventoryReserved event) {
        Order order = orderClient.getOrder(event.getOrderId());
        boolean success = charge(order.getTotal());
        if (success) {
            eventPublisher.publish(new PaymentProcessed(event.getOrderId()));
        } else {
            eventPublisher.publish(new PaymentFailed(event.getOrderId()));
        }
    }
}
```

**Pros:**
- Loose coupling
- No single point of failure
- Services independently deployable
- Better scalability

**Cons:**
- Hard to understand overall flow
- Debugging is difficult
- Complex failure handling
- Risk of cyclic dependencies

### 5.3 Choosing Between Them

| Factor | Orchestration | Choreography |
|--------|---------------|--------------|
| Complexity | In orchestrator | Distributed |
| Coupling | To orchestrator | To events |
| Visibility | Centralized | Requires tracing |
| Failure Handling | Centralized | Distributed |
| Scalability | Orchestrator bottleneck | Better |
| Testing | Easier | Harder |

**Guidelines:**
- **Orchestration**: When process has clear steps, needs central control, error handling is complex
- **Choreography**: When services are truly independent, loose coupling is priority, high scalability needed

**Hybrid Approach:**
Use orchestration within bounded contexts, choreography between them.

---

## Section 6: Practical Considerations

### 6.1 Idempotency

**The Problem:**
```
Order Service ──► Payment Service
                      │
               (timeout, was it processed?)
                      │
Order Service ──► Payment Service (retry)
                      │
              (charged twice!)
```

**The Solution: Idempotent Operations**
```java
@Service
public class PaymentService {

    public PaymentResult charge(PaymentRequest request) {
        // Check if already processed
        Optional<Payment> existing = paymentRepository
            .findByIdempotencyKey(request.getIdempotencyKey());

        if (existing.isPresent()) {
            return PaymentResult.alreadyProcessed(existing.get());
        }

        // Process and store with idempotency key
        Payment payment = processPayment(request);
        payment.setIdempotencyKey(request.getIdempotencyKey());
        paymentRepository.save(payment);

        return PaymentResult.success(payment);
    }
}
```

**Idempotency Key Generation:**
```java
// Client generates key before first attempt
String idempotencyKey = UUID.randomUUID().toString();

// Retry uses SAME key
paymentClient.charge(new PaymentRequest(
    orderId,
    amount,
    idempotencyKey  // Same key on retries
));
```

### 6.2 Message Ordering

**The Problem:**
```
Events published:
1. OrderCreated { orderId: 123 }
2. OrderUpdated { orderId: 123, status: "CONFIRMED" }

Consumer might receive:
2. OrderUpdated { orderId: 123, status: "CONFIRMED" }  (first!)
1. OrderCreated { orderId: 123 }                        (second)

What status should order have?
```

**Solutions:**

**Kafka Partition Key:**
```java
// Same order ID → same partition → ordered
kafkaTemplate.send("orders", order.getId(), event);
```

**Event Versioning:**
```java
public class OrderEvent {
    private String orderId;
    private long version;  // Increment with each event
    private Instant timestamp;
}

// Consumer
public void handleEvent(OrderEvent event) {
    Order order = orderRepository.findById(event.getOrderId());
    if (event.getVersion() <= order.getLastEventVersion()) {
        // Already processed or out of order, skip
        return;
    }
    // Process event
}
```

### 6.3 Dead Letter Queues

**The Problem:**
Message processing fails repeatedly. Don't lose it, don't block queue.

**The Solution:**
```
Main Queue ──► Consumer ──► (fails) ──► Retry Queue
                                              │
                                              ▼
                                        (fails again)
                                              │
                                              ▼
                                      Dead Letter Queue
                                              │
                                              ▼
                                    (manual investigation)
```

**Configuration (RabbitMQ):**
```java
@Configuration
public class QueueConfig {

    @Bean
    public Queue orderQueue() {
        return QueueBuilder.durable("orders")
            .withArgument("x-dead-letter-exchange", "orders.dlx")
            .withArgument("x-dead-letter-routing-key", "orders.dlq")
            .build();
    }

    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable("orders.dlq").build();
    }
}
```

### 6.4 Schema Evolution

**The Problem:**
You publish events with a schema. Consumers depend on it. How do you evolve?

**Backward Compatible Changes (Safe):**
- Add optional fields
- Add new event types

```json
// v1
{ "orderId": "123", "total": 99.99 }

// v2 (backward compatible)
{ "orderId": "123", "total": 99.99, "currency": "USD" }

// Old consumers ignore new fields
```

**Breaking Changes (Dangerous):**
- Remove fields
- Rename fields
- Change types

**Strategies:**
1. **Versioned Topics**: `orders.v1`, `orders.v2`
2. **Schema Registry**: Enforce compatibility (Confluent Schema Registry)
3. **Consumer-Driven Contracts**: Consumers specify what they need

**Avro Schema Evolution:**
```json
{
    "type": "record",
    "name": "OrderCreated",
    "fields": [
        {"name": "orderId", "type": "string"},
        {"name": "total", "type": "double"},
        {"name": "currency", "type": "string", "default": "USD"}  // New with default
    ]
}
```

### 6.5 Monitoring and Observability

**Key Metrics:**

**Synchronous:**
- Request latency (p50, p95, p99)
- Error rate
- Throughput
- Circuit breaker state

**Asynchronous:**
- Queue depth
- Consumer lag (Kafka)
- Processing latency
- Dead letter queue size
- Message age

**Distributed Tracing:**
```java
// Propagate trace context through messages
public void publishEvent(OrderEvent event) {
    Message<OrderEvent> message = MessageBuilder
        .withPayload(event)
        .setHeader("traceparent", tracer.currentSpan().context().toString())
        .build();
    kafkaTemplate.send("orders", message);
}

// Consumer extracts context
@KafkaListener(topics = "orders")
public void handleEvent(
        @Payload OrderEvent event,
        @Header("traceparent") String traceContext) {
    Span span = tracer.spanBuilder("process-order")
        .setParent(extractContext(traceContext))
        .startSpan();
    // Process with trace context
}
```

---

## Section 7: Now You Understand Why...

Armed with this foundation, you can now understand:

- **Why REST is so popular**: Universal, simple, works everywhere—good enough for most cases

- **Why internal services often use gRPC**: Performance matters when services call each other frequently

- **Why Kafka isn't "just another message queue"**: The log-based design enables replay, consumer groups, and ordering guarantees

- **Why event-driven systems are harder to debug**: No central flow to follow, events scattered across services

- **Why idempotency is mentioned constantly**: At-least-once delivery is common, duplicates must be handled

- **Why "eventual consistency" isn't scary**: Most systems can tolerate brief inconsistency; the benefits of async are worth it

---

## Practical Exercises

### Exercise 1: Compare REST vs. gRPC
Implement the same service in both:
- Measure latency and throughput
- Compare code complexity
- Try streaming with each

### Exercise 2: Build an Event-Driven Flow
Implement order processing with choreography:
- Order Service → Inventory Service → Payment Service
- Handle failures at each step
- Add observability

### Exercise 3: Implement Idempotency
Create a payment service that:
- Accepts an idempotency key
- Handles retries correctly
- Returns same result for duplicate requests

### Exercise 4: Schema Evolution
Using Kafka and Avro:
- Start with a simple event schema
- Evolve the schema (add field, remove field)
- Test backward and forward compatibility

### Exercise 5: Compare Orchestration vs. Choreography
Implement the same saga both ways:
- Measure complexity
- Introduce failures
- Compare debugging experience

---

## Key Takeaways

1. **Synchronous is simpler, asynchronous is more resilient**. Start synchronous, move to async when you need the benefits.

2. **REST for simplicity, gRPC for performance, GraphQL for flexibility**. Choose based on your specific needs.

3. **Events decouple services** but add complexity. Use event-carried state transfer to reduce callbacks.

4. **Idempotency is essential** in distributed systems. Design for at-least-once delivery.

5. **Orchestration is easier to understand, choreography scales better**. Often a hybrid works best.

6. **Schema evolution is a distributed systems problem**. Plan for it from the start.

---

## Looking Ahead

With distributed systems and communication patterns understood, Part 4 explores how to deploy and manage these systems. Chapter 11 starts from first principles: what are containers, really?

---

## Chapter 10 Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  SERVICE COMMUNICATION PATTERNS                              │
│                                                                             │
│  SYNCHRONOUS                                                                │
│  ───────────                                                                │
│  REST: Universal, simple, HTTP caching                                      │
│  gRPC: High performance, streaming, strong contracts                        │
│  GraphQL: Flexible queries, no over/under-fetching                         │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ASYNCHRONOUS                                                               │
│  ────────────                                                               │
│  Message Queues: Point-to-point, at-least-once delivery                    │
│  Pub/Sub: One-to-many, multiple subscribers                                │
│  Event Streaming: Persistent log, replay, ordering                         │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  EVENT PATTERNS                                                             │
│  ──────────────                                                             │
│                                                                             │
│  Notification    → "Something happened, ask me for details"                │
│  State Transfer  → "Here's everything you need in the event"               │
│  Event Sourcing  → "Events ARE the source of truth"                        │
│  CQRS            → "Separate read and write models"                        │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ORCHESTRATION vs CHOREOGRAPHY                                              │
│  ──────────────────────────────                                             │
│                                                                             │
│  Orchestration:           Choreography:                                     │
│  ┌───────────────┐        Event ──► Service A                              │
│  │  Coordinator  │              └──► Service B                              │
│  └───────┬───────┘                   └──► Service C                         │
│          │                                                                  │
│    ┌─────┼─────┐          Decentralized, harder to follow                  │
│    ▼     ▼     ▼          but more resilient                               │
│   A      B     C                                                            │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ESSENTIAL PATTERNS                                                         │
│  ──────────────────                                                         │
│                                                                             │
│  Idempotency:     Same request → same result (handle retries)              │
│  Message Ordering: Partition by key, version events                         │
│  Dead Letter Queue: Don't lose failed messages                             │
│  Schema Evolution: Plan for change from day one                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Next Chapter: Containers from Scratch — Understanding what containers really are, from first principles*
