# Chapter 7: Data Layer Thinking

> **First Principles Question**: Why do we need databases at all? What fundamental problems do they solve, and why are there so many different types?

---

## Chapter Overview

Data is the lifeblood of applications. How you store, retrieve, and manage data affects everything—performance, scalability, consistency, and even the features you can build. Yet many developers treat the data layer as an afterthought. This chapter builds a mental model for thinking about data persistence from first principles.

**What readers will understand after this chapter:**
- Why databases exist and what problems they solve
- The fundamental trade-offs in data storage
- When to use relational vs. NoSQL databases
- How to think about data modeling
- Transaction guarantees and why they matter
- Caching strategies and their trade-offs
- JPA/Hibernate patterns and pitfalls

---

## Section 1: The Fundamental Problem of Persistence

### 1.1 Why We Need Databases

**The Problem:**
Your Java application stores data in memory. Variables, objects, collections—all in RAM. What happens when:
- The application restarts?
- The server loses power?
- You need to scale to multiple servers?

**Memory Is Ephemeral:**
```java
List<User> users = new ArrayList<>();
users.add(new User("Alice"));
// Application restarts... users is gone
```

**The Solution: Persistence**
Write data to durable storage (disk) in a way that survives crashes and can be read back.

### 1.2 Why Not Just Use Files?

**Simplest Persistence:**
```java
// Write
Files.writeString(Path.of("users.json"), gson.toJson(users));

// Read
String json = Files.readString(Path.of("users.json"));
List<User> users = gson.fromJson(json, new TypeToken<List<User>>(){}.getType());
```

**Problems with Plain Files:**

**1. Concurrent Access:**
What if two processes write simultaneously? Data corruption.

**2. Querying:**
"Find all users named Alice who signed up last month"—you'd have to read the entire file.

**3. Relationships:**
Users have orders, orders have items. Managing cross-file references is complex.

**4. Partial Updates:**
Change one user's email? Rewrite the entire file.

**5. Crash Recovery:**
Write interrupted halfway? Corrupted file.

### 1.3 What Databases Provide

**Databases Solve:**
- **Concurrent access**: Multiple readers/writers safely
- **Efficient querying**: Indexes, query optimization
- **Data integrity**: Constraints, transactions
- **Crash recovery**: Write-ahead logging, checkpoints
- **Relationships**: Foreign keys, joins
- **Scale**: Replication, sharding

**The Core Value:**
Databases let you think about data logically while handling the physical complexity of storage.

---

## Section 2: The Relational Model — Tables, Rows, and Relationships

### 2.1 The Relational Insight

**Edgar Codd's Breakthrough (1970):**
Separate logical data structure from physical storage. Represent data as relations (tables) with rows and columns.

**A Table:**
```
users
┌────┬─────────┬───────────────────┬────────────┐
│ id │ name    │ email             │ created_at │
├────┼─────────┼───────────────────┼────────────┤
│ 1  │ Alice   │ alice@example.com │ 2024-01-15 │
│ 2  │ Bob     │ bob@example.com   │ 2024-01-16 │
│ 3  │ Carol   │ carol@example.com │ 2024-01-17 │
└────┴─────────┴───────────────────┴────────────┘
```

**Key Properties:**
- Each row is a record (entity)
- Each column has a type
- Primary key uniquely identifies rows
- No duplicate rows

### 2.2 Relationships

**One-to-Many:**
```
users                           orders
┌────┬─────────┐                ┌────┬─────────┬────────┐
│ id │ name    │                │ id │ user_id │ total  │
├────┼─────────┤                ├────┼─────────┼────────┤
│ 1  │ Alice   │◄───────────────│ 1  │ 1       │ 99.99  │
│ 2  │ Bob     │                │ 2  │ 1       │ 149.99 │
└────┴─────────┘                │ 3  │ 2       │ 29.99  │
                                └────┴─────────┴────────┘
```

**Many-to-Many (via junction table):**
```
products                 order_items              orders
┌────┬─────────┐        ┌──────────┬────────────┐   ┌────┐
│ id │ name    │        │ order_id │ product_id │   │ id │
├────┼─────────┤        ├──────────┼────────────┤   ├────┤
│ 1  │ Widget  │◄───────│ 1        │ 1          │───►│ 1  │
│ 2  │ Gadget  │◄───────│ 1        │ 2          │───►│ 1  │
└────┴─────────┘        │ 2        │ 1          │───►│ 2  │
                        └──────────┴────────────┘   └────┘
```

### 2.3 Normalization

**The Goal:**
Eliminate data redundancy and anomalies.

**Unnormalized (Bad):**
```
orders
┌────┬───────────┬─────────────────────┬────────┐
│ id │ user_name │ user_email          │ total  │
├────┼───────────┼─────────────────────┼────────┤
│ 1  │ Alice     │ alice@example.com   │ 99.99  │
│ 2  │ Alice     │ alice@example.com   │ 149.99 │
│ 3  │ Alice     │ alice@NEW.com       │ 49.99  │ ← Email inconsistency!
└────┴───────────┴─────────────────────┴────────┘
```

**Problems:**
- Update Alice's email? Must update many rows.
- Forget one? Inconsistent data.
- Delete all Alice's orders? Lose her information entirely.

**Normalized (Good):**
```
users                           orders
┌────┬───────┬─────────────────┐  ┌────┬─────────┬────────┐
│ id │ name  │ email           │  │ id │ user_id │ total  │
├────┼───────┼─────────────────┤  ├────┼─────────┼────────┤
│ 1  │ Alice │ alice@example.com│  │ 1  │ 1       │ 99.99  │
└────┴───────┴─────────────────┘  │ 2  │ 1       │ 149.99 │
                                  │ 3  │ 1       │ 49.99  │
                                  └────┴─────────┴────────┘
```

**Benefits:**
- One place to update email
- No inconsistency possible
- Can delete orders without losing user

### 2.4 SQL — The Language of Relational Data

**Query Language, Not Programming Language:**
You describe WHAT you want, not HOW to get it.

```sql
-- The what
SELECT u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id
HAVING COUNT(o.id) > 5
ORDER BY order_count DESC;

-- The how (query optimizer decides):
-- - Which index to use?
-- - What join algorithm?
-- - In what order to filter?
```

**This Declarative Nature:**
The database can optimize based on statistics, indexes, and hardware. Your query stays the same.

---

## Section 3: ACID — The Guarantees That Matter

### 3.1 What Is a Transaction?

**Definition:**
A sequence of operations treated as a single logical unit. Either all succeed or all fail.

**Example:**
```sql
-- Transfer $100 from Alice to Bob
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 'alice';
UPDATE accounts SET balance = balance + 100 WHERE user_id = 'bob';
COMMIT;
```

**The Critical Question:**
What if the system crashes between the two updates?

### 3.2 ACID Properties

**A — Atomicity:**
All or nothing. If any part fails, nothing happens.

```
Without Atomicity:
1. Deduct from Alice ✓
2. [CRASH]
3. Add to Bob ✗
Result: Money vanished!

With Atomicity:
1. Deduct from Alice
2. [CRASH]
3. [ROLLBACK]
Result: Alice still has her money
```

**C — Consistency:**
Database moves from one valid state to another. Constraints are always satisfied.

```sql
-- Constraint: balance >= 0
UPDATE accounts SET balance = balance - 100 WHERE user_id = 'alice';
-- If Alice only has $50, this fails. Transaction aborts.
```

**I — Isolation:**
Concurrent transactions don't interfere with each other.

```
Without Isolation:
Transaction A: Read balance (100)
Transaction B: Read balance (100)
Transaction A: Set balance = 100 - 50 = 50
Transaction B: Set balance = 100 - 30 = 70
Result: Balance is 70, should be 20!

With Isolation:
Transactions see a consistent snapshot, updates serialized properly.
```

**D — Durability:**
Once committed, data survives crashes.

```
COMMIT returns → Data is safe
Even if power fails 1ms later.
(Via write-ahead logging)
```

### 3.3 Isolation Levels

**The Trade-off:**
Stronger isolation = more correct behavior, but less concurrency.

| Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads |
|-------|-------------|---------------------|---------------|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible |
| Serializable | Prevented | Prevented | Prevented |

**In Practice:**
Most applications use Read Committed (PostgreSQL default) or Repeatable Read (MySQL default). Serializable is rarely needed.

### 3.4 When ACID Isn't Enough

**Distributed Transactions:**
What if your data spans multiple databases? Multiple services?

**The CAP Theorem:**
In a distributed system, you can only guarantee two of:
- **Consistency**: Every read receives the most recent write
- **Availability**: Every request receives a response
- **Partition Tolerance**: System continues despite network failures

**Reality:**
Network partitions happen. You must choose between consistency and availability.

---

## Section 4: Beyond Relational — NoSQL Databases

### 4.1 Why NoSQL Emerged

**The 2000s Problem:**
Web-scale applications needed:
- Massive write throughput
- Horizontal scaling
- Flexible schemas
- Geographic distribution

**Relational Limitations:**
- Joins don't scale across servers
- Schema changes are expensive
- Single-server writes bottleneck

### 4.2 NoSQL Categories

**Document Databases (MongoDB, CouchDB):**
```json
{
    "_id": "user_123",
    "name": "Alice",
    "email": "alice@example.com",
    "addresses": [
        {"type": "home", "city": "Boston"},
        {"type": "work", "city": "Cambridge"}
    ],
    "orders": [
        {"id": "ord_1", "total": 99.99, "items": [...]}
    ]
}
```

**Best for:**
- Variable schema data
- Embedded relationships (denormalized)
- Content management, user profiles

**Key-Value Stores (Redis, DynamoDB):**
```
key: "user:123"
value: serialized user object

key: "session:abc"
value: session data
```

**Best for:**
- Caching
- Session storage
- Simple lookup patterns

**Wide-Column Stores (Cassandra, HBase):**
```
Row key: "user:123"
Columns: {
    "name": "Alice",
    "email": "alice@example.com",
    "orders:2024-01-01": "...",
    "orders:2024-01-15": "..."
}
```

**Best for:**
- Time-series data
- Write-heavy workloads
- High availability requirements

**Graph Databases (Neo4j, Neptune):**
```
(Alice)-[:FRIEND]->(Bob)
(Alice)-[:PURCHASED]->(Product1)
(Bob)-[:PURCHASED]->(Product1)
```

**Best for:**
- Social networks
- Recommendation engines
- Fraud detection

### 4.3 The Trade-offs

**What NoSQL Gains:**
- Horizontal scalability
- Schema flexibility
- Specialized query patterns
- High availability

**What NoSQL Loses:**
- ACID transactions (usually)
- Complex queries (joins)
- Strong consistency (usually)
- Mature tooling

### 4.4 Choosing the Right Database

**Start with the Questions:**
1. What are the access patterns?
2. What consistency requirements?
3. What scale requirements?
4. What relationships exist in the data?

**Decision Guide:**
```
Need complex queries and transactions?     → Relational (PostgreSQL, MySQL)
Need flexible schema and embedded docs?   → Document (MongoDB)
Need extreme read performance?            → Key-Value with caching (Redis)
Need high write throughput, always on?    → Wide-Column (Cassandra)
Need to traverse relationships?           → Graph (Neo4j)
```

**The Default:**
When in doubt, start with PostgreSQL. It handles most use cases well and has excellent NoSQL features (JSONB, arrays) when needed.

---

## Section 5: Data Modeling Strategies

### 5.1 Entity Relationship Modeling

**Start with Entities:**
What things exist in your domain? Users, orders, products, payments.

**Define Relationships:**
- Users *have many* orders
- Orders *have many* items
- Items *belong to* products
- Orders *have one* payment

**Cardinality:**
```
One-to-One:     User ─── Profile
One-to-Many:    User ─┬─ Order
                      └─ Order
Many-to-Many:   Order ─┬─ Product
                       └─ Product
                Product ─┬─ Order
                         └─ Order
```

### 5.2 Designing Tables

**From Entities to Tables:**
```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    total DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    inventory_count INTEGER DEFAULT 0
);

CREATE TABLE order_items (
    order_id BIGINT REFERENCES orders(id),
    product_id BIGINT REFERENCES products(id),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);
```

### 5.3 Indexing Strategy

**Why Indexes Matter:**
Without indexes, queries scan entire tables. With indexes, they jump to relevant rows.

**The Index Trade-off:**
- **Reads faster**: Direct lookup instead of scan
- **Writes slower**: Must update index on every insert/update
- **More storage**: Index takes disk space

**What to Index:**
```sql
-- Primary keys (automatic)
-- Foreign keys (for joins)
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Frequently filtered columns
CREATE INDEX idx_orders_status ON orders(status);

-- Columns used in ORDER BY
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);

-- Composite indexes for common queries
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

**When NOT to Index:**
- Columns rarely used in WHERE/JOIN/ORDER BY
- Tables with very few rows
- Columns with low cardinality (e.g., boolean)
- Heavily written, rarely read tables

### 5.4 Denormalization

**When to Break the Rules:**
Normalization optimizes for writes and consistency. Sometimes you need to optimize for reads.

**Example: Order Totals**
```sql
-- Normalized: Calculate total every time
SELECT SUM(quantity * unit_price) FROM order_items WHERE order_id = 123;

-- Denormalized: Store calculated total
SELECT total FROM orders WHERE id = 123;
```

**When to Denormalize:**
- Read-heavy workloads
- Expensive calculations
- Performance-critical paths
- Reporting tables

**The Cost:**
You must keep denormalized data in sync. This adds complexity and potential for inconsistency.

---

## Section 6: JPA and Hibernate — Object-Relational Mapping

### 6.1 The Impedance Mismatch

**The Problem:**
Object-oriented programming and relational databases have different models.

| Objects | Tables |
|---------|--------|
| Identity by reference | Identity by primary key |
| Inheritance hierarchies | Flat tables |
| Object graphs | Foreign key relations |
| Behavior + data | Data only |

**ORM (Object-Relational Mapping):**
A layer that translates between objects and tables.

### 6.2 JPA Basics

**Entity Mapping:**
```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String name;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL)
    private List<Order> orders = new ArrayList<>();

    @Column(name = "created_at")
    private Instant createdAt;

    // Constructors, getters, setters
}
```

**Relationship Mapping:**
```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    private BigDecimal total;
}
```

### 6.3 The N+1 Problem

**The Trap:**
```java
List<User> users = userRepository.findAll();
for (User user : users) {
    System.out.println(user.getOrders().size());  // Lazy load!
}
```

**What Happens:**
```sql
SELECT * FROM users;                    -- 1 query
SELECT * FROM orders WHERE user_id = 1; -- +1 query
SELECT * FROM orders WHERE user_id = 2; -- +1 query
SELECT * FROM orders WHERE user_id = 3; -- +1 query
-- ... N more queries for N users
```

**The Fix: Fetch Joins**
```java
@Query("SELECT u FROM User u LEFT JOIN FETCH u.orders")
List<User> findAllWithOrders();
```

```sql
SELECT * FROM users u LEFT JOIN orders o ON o.user_id = u.id;
-- 1 query total
```

### 6.4 Lazy vs. Eager Loading

**Lazy Loading (Default for Collections):**
```java
@OneToMany(fetch = FetchType.LAZY)
private List<Order> orders;
// Orders loaded only when accessed
```

**Eager Loading:**
```java
@ManyToOne(fetch = FetchType.EAGER)
private User user;
// User loaded immediately with order
```

**Best Practice:**
- Default to lazy loading
- Use fetch joins when you know you need related data
- Never use eager loading on collections (N+1 risk)

### 6.5 Transactions in Spring

**Declarative Transactions:**
```java
@Service
public class OrderService {

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // Everything in this method is one transaction
        User user = userRepository.findById(request.getUserId())
            .orElseThrow();

        Order order = new Order();
        order.setUser(user);

        for (ItemRequest item : request.getItems()) {
            Product product = productRepository.findById(item.getProductId())
                .orElseThrow();
            // Deduct inventory
            product.setInventoryCount(product.getInventoryCount() - item.getQuantity());
            order.addItem(product, item.getQuantity());
        }

        return orderRepository.save(order);
        // Commit happens here (or rollback on exception)
    }
}
```

**Transaction Propagation:**
```java
@Transactional(propagation = Propagation.REQUIRED)      // Join existing or create new (default)
@Transactional(propagation = Propagation.REQUIRES_NEW)  // Always create new
@Transactional(propagation = Propagation.MANDATORY)     // Must have existing
```

### 6.6 Common Pitfalls

**1. LazyInitializationException:**
```java
User user = userRepository.findById(1L).get();
// Transaction ends here

user.getOrders().size();  // Exception! Session closed
```

**Fix:** Use fetch joins or `@Transactional` on the calling method.

**2. Detached Entity Modifications:**
```java
User user = userRepository.findById(1L).get();
// Transaction ends

user.setEmail("new@example.com");  // Nothing happens!
// Entity is detached, changes not tracked
```

**Fix:** Modify within transaction or use `save()`.

**3. Too Many Queries:**
```java
// Bad: Multiple round trips
User user = userRepository.findById(userId).get();
List<Order> orders = orderRepository.findByUserId(userId);
List<Payment> payments = paymentRepository.findByUserId(userId);

// Better: Single query with joins
User user = userRepository.findByIdWithOrdersAndPayments(userId);
```

---

## Section 7: Caching — Faster Data Access

### 7.1 Why Cache?

**The Problem:**
Database queries take time (network + disk). For frequently accessed data, this adds up.

**The Solution:**
Keep copies of data in faster storage (memory).

**The Trade-off:**
- **Faster reads**: Memory access vs. disk/network
- **Stale data risk**: Cache might be out of date
- **Memory cost**: Cache uses RAM

### 7.2 Cache Levels

**Application-Level Cache:**
```
┌─────────────┐     ┌─────────────┐     ┌────────────┐
│ Application │ ──→ │   Cache     │ ──→ │  Database  │
└─────────────┘     │   (Redis)   │     └────────────┘
                    └─────────────┘
```

**Database Query Cache:**
Database caches query results internally.

**ORM/Hibernate Cache:**
- **First-level**: Per-session entity cache
- **Second-level**: Shared across sessions

### 7.3 Caching Patterns

**Cache-Aside (Lazy Loading):**
```java
public User getUser(Long id) {
    // Check cache first
    User user = cache.get("user:" + id);
    if (user != null) {
        return user;
    }

    // Cache miss: load from database
    user = userRepository.findById(id).orElseThrow();

    // Populate cache
    cache.put("user:" + id, user, Duration.ofMinutes(10));

    return user;
}
```

**Write-Through:**
```java
public User updateUser(Long id, UpdateRequest request) {
    User user = userRepository.findById(id).orElseThrow();
    user.setEmail(request.getEmail());

    // Write to database
    user = userRepository.save(user);

    // Update cache synchronously
    cache.put("user:" + id, user);

    return user;
}
```

**Write-Behind (Async):**
```java
public User updateUser(Long id, UpdateRequest request) {
    // Update cache immediately
    User user = cache.get("user:" + id);
    user.setEmail(request.getEmail());
    cache.put("user:" + id, user);

    // Queue database write for later
    writeQueue.add(new WriteOperation(user));

    return user;
}
```

### 7.4 Cache Invalidation

**The Hard Problem:**
> "There are only two hard things in Computer Science: cache invalidation and naming things."
> — Phil Karlton

**Strategies:**

**Time-Based Expiration (TTL):**
```java
cache.put(key, value, Duration.ofMinutes(5));
// Simple but may serve stale data
```

**Event-Based Invalidation:**
```java
@EventListener
public void onUserUpdated(UserUpdatedEvent event) {
    cache.evict("user:" + event.getUserId());
}
```

**Version-Based:**
```java
cache.put("user:" + id + ":v" + user.getVersion(), user);
// New versions get new cache keys
```

### 7.5 Spring Cache Abstraction

**Declarative Caching:**
```java
@Service
public class UserService {

    @Cacheable(value = "users", key = "#id")
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }

    @CachePut(value = "users", key = "#user.id")
    public User update(User user) {
        return userRepository.save(user);
    }

    @CacheEvict(value = "users", key = "#id")
    public void delete(Long id) {
        userRepository.deleteById(id);
    }

    @CacheEvict(value = "users", allEntries = true)
    public void clearCache() {
        // Clears all entries in "users" cache
    }
}
```

**Configuration:**
```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()
                )
            );

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
    }
}
```

---

## Section 8: Database Migrations

### 8.1 The Schema Evolution Problem

**Reality:**
Your database schema will change. New columns, new tables, renamed fields.

**The Challenge:**
How do you update production databases safely? Roll back if something goes wrong?

### 8.2 Migration Tools

**Flyway:**
```
resources/db/migration/
├── V1__Create_users_table.sql
├── V2__Create_orders_table.sql
├── V3__Add_email_verified_column.sql
└── V4__Create_products_table.sql
```

**Migration File:**
```sql
-- V3__Add_email_verified_column.sql
ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT FALSE;
CREATE INDEX idx_users_email_verified ON users(email_verified);
```

**How It Works:**
1. Flyway tracks applied migrations in a table
2. On startup, applies new migrations in order
3. Each migration runs exactly once

### 8.3 Migration Best Practices

**1. Forward-Only Migrations:**
Don't modify old migration files. Create new ones.

**2. Small, Incremental Changes:**
```sql
-- Bad: One giant migration
-- Good: Many small migrations
V1__Create_users_table.sql
V2__Add_users_email_index.sql
V3__Add_users_phone_column.sql
```

**3. Non-Destructive Changes:**
```sql
-- Safe: Add column with default
ALTER TABLE users ADD COLUMN status VARCHAR(50) DEFAULT 'active';

-- Dangerous: Drop column (data loss!)
ALTER TABLE users DROP COLUMN old_field;
```

**4. Backwards-Compatible Schema:**
When deploying with zero downtime:
1. Add new column (nullable or with default)
2. Deploy code that writes to both old and new
3. Migrate existing data
4. Deploy code that reads from new
5. Remove old column (later)

### 8.4 Liquibase Alternative

**XML-Based Changesets:**
```xml
<changeSet id="1" author="developer">
    <createTable tableName="users">
        <column name="id" type="BIGINT" autoIncrement="true">
            <constraints primaryKey="true"/>
        </column>
        <column name="email" type="VARCHAR(255)">
            <constraints nullable="false" unique="true"/>
        </column>
    </createTable>
</changeSet>
```

**Benefits:**
- Database-agnostic syntax
- Rollback support
- More features (preconditions, contexts)

---

## Section 9: Practical Patterns

### 9.1 Repository Pattern

**Abstraction Over Data Access:**
```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    List<Order> findByUserIdAndStatus(Long userId, OrderStatus status);

    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
    Optional<Order> findByIdWithItems(@Param("id") Long id);

    @Query("SELECT o FROM Order o WHERE o.createdAt > :since AND o.status = :status")
    Page<Order> findRecentByStatus(
        @Param("since") Instant since,
        @Param("status") OrderStatus status,
        Pageable pageable
    );
}
```

### 9.2 Specification Pattern

**Dynamic Queries:**
```java
public class OrderSpecifications {

    public static Specification<Order> hasStatus(OrderStatus status) {
        return (root, query, cb) ->
            status == null ? null : cb.equal(root.get("status"), status);
    }

    public static Specification<Order> createdAfter(Instant date) {
        return (root, query, cb) ->
            date == null ? null : cb.greaterThan(root.get("createdAt"), date);
    }

    public static Specification<Order> belongsToUser(Long userId) {
        return (root, query, cb) ->
            userId == null ? null : cb.equal(root.get("user").get("id"), userId);
    }
}

// Usage
List<Order> orders = orderRepository.findAll(
    Specification.where(hasStatus(PENDING))
        .and(createdAfter(lastWeek))
        .and(belongsToUser(userId))
);
```

### 9.3 Pagination

**Spring Data Pagination:**
```java
@GetMapping("/orders")
public Page<OrderResponse> listOrders(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt,desc") String[] sort) {

    Pageable pageable = PageRequest.of(page, size, Sort.by(parseSort(sort)));
    Page<Order> orders = orderRepository.findAll(pageable);

    return orders.map(OrderResponse::from);
}
```

**Cursor-Based Pagination:**
```java
public record OrderPage(List<Order> orders, String nextCursor) {}

public OrderPage findOrdersAfter(String cursor, int limit) {
    Long lastId = decodeCursor(cursor);

    List<Order> orders = orderRepository.findByIdGreaterThanOrderByIdAsc(
        lastId, PageRequest.of(0, limit + 1)
    );

    boolean hasMore = orders.size() > limit;
    if (hasMore) {
        orders = orders.subList(0, limit);
    }

    String nextCursor = hasMore ? encodeCursor(orders.get(limit - 1).getId()) : null;

    return new OrderPage(orders, nextCursor);
}
```

### 9.4 Soft Deletes

**Don't Actually Delete:**
```java
@Entity
@SQLDelete(sql = "UPDATE orders SET deleted_at = CURRENT_TIMESTAMP WHERE id = ?")
@Where(clause = "deleted_at IS NULL")
public class Order {

    @Column(name = "deleted_at")
    private Instant deletedAt;

    // Regular delete() calls set deleted_at instead
    // Queries automatically filter out deleted records
}
```

### 9.5 Auditing

**Track Who Changed What:**
```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Order {

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @CreatedBy
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}

@Configuration
@EnableJpaAuditing
public class AuditConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .map(Authentication::getName);
    }
}
```

---

## Section 10: Now You Understand Why...

Armed with this foundation, you can now understand:

- **Why databases have different types**: Different access patterns have different optimal solutions

- **Why normalization matters**: Redundant data leads to inconsistency. But sometimes denormalization is the right trade-off.

- **Why transactions exist**: Without atomic operations, partial failures corrupt data

- **Why ORM can be both helpful and harmful**: It abstracts away SQL but can hide performance problems (N+1)

- **Why caching is tricky**: Faster reads at the cost of potential staleness. Invalidation is hard.

- **Why database migrations are version controlled**: Schema changes need to be reproducible and ordered

- **Why "just use MongoDB" isn't always the answer**: NoSQL trades consistency and query flexibility for scale and flexibility

---

## Practical Exercises

### Exercise 1: Design a Schema
Design tables for a blog system:
- Users have many posts
- Posts have many comments
- Posts have many tags (many-to-many)
- Comments can be nested (replies)

### Exercise 2: Spot the N+1
```java
List<Post> posts = postRepository.findAll();
for (Post post : posts) {
    System.out.println(post.getTitle() + " by " + post.getAuthor().getName());
    System.out.println("Tags: " + post.getTags().size());
    System.out.println("Comments: " + post.getComments().size());
}
```
How many queries does this execute? How would you fix it?

### Exercise 3: Write a Migration
Write Flyway migrations to:
1. Add a `views_count` column to posts
2. Create an index on it
3. Backfill existing posts with 0 views

### Exercise 4: Implement Caching
Add caching to a user lookup service:
- Cache user by ID for 10 minutes
- Invalidate on update
- Handle the case where user doesn't exist

### Exercise 5: Compare Query Approaches
```java
// Approach A
List<Order> orders = orderRepository.findByStatus(OrderStatus.PENDING);
List<OrderDTO> dtos = orders.stream()
    .map(o -> new OrderDTO(o.getId(), o.getUser().getName(), o.getTotal()))
    .toList();

// Approach B
@Query("SELECT new com.example.OrderDTO(o.id, o.user.name, o.total) " +
       "FROM Order o WHERE o.status = :status")
List<OrderDTO> findOrderDTOsByStatus(@Param("status") OrderStatus status);
```
When would you use each approach? What are the trade-offs?

---

## Key Takeaways

1. **Databases solve fundamental problems**. Concurrent access, durability, querying—don't reinvent these.

2. **Choose the right database for your access patterns**. Relational for complex queries and transactions, NoSQL for specific scale/flexibility needs.

3. **Understand your ORM**. JPA/Hibernate is powerful but hides complexity. Learn what queries it generates.

4. **Caching is a trade-off**. Faster reads vs. potential staleness. Have a clear invalidation strategy.

5. **Schema evolution requires discipline**. Use migration tools, make backwards-compatible changes, test migrations.

6. **Indexes are essential but not free**. Index what you query, but remember write overhead.

---

## Looking Ahead

With the data layer understood, Part 3 explores **Distributed Systems**: What happens when your application spans multiple servers? Chapter 8 examines why distributed systems are fundamentally hard.

---

## Chapter 7 Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DATA LAYER THINKING                                   │
│                                                                             │
│  DATABASE SELECTION                                                          │
│  ──────────────────                                                          │
│                                                                             │
│   Complex queries + Transactions?      → Relational (PostgreSQL, MySQL)    │
│   Document-oriented + Flexible schema? → Document (MongoDB)                 │
│   Simple key-value + Speed?            → Key-Value (Redis)                  │
│   Write-heavy + High availability?     → Wide-Column (Cassandra)            │
│   Graph relationships?                 → Graph (Neo4j)                      │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ACID PROPERTIES                                                             │
│  ───────────────                                                             │
│                                                                             │
│   Atomicity:   All or nothing                                               │
│   Consistency: Valid state to valid state                                   │
│   Isolation:   Transactions don't interfere                                 │
│   Durability:  Committed data survives crashes                              │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  JPA/HIBERNATE ESSENTIALS                                                    │
│  ────────────────────────                                                    │
│                                                                             │
│   @Entity + @Table     →  Map class to table                                │
│   @ManyToOne/@OneToMany →  Define relationships                             │
│   FetchType.LAZY       →  Load on demand (default for collections)          │
│   JOIN FETCH           →  Load eagerly in query (avoid N+1)                 │
│   @Transactional       →  Define transaction boundaries                     │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  CACHING STRATEGY                                                            │
│  ────────────────                                                            │
│                                                                             │
│   ┌─────────────┐      ┌─────────────┐      ┌────────────┐                 │
│   │   Request   │ ──→  │    Cache    │ ──→  │  Database  │                 │
│   └─────────────┘      │   (Redis)   │      └────────────┘                 │
│                        └─────────────┘                                      │
│                        Hit: Return     Miss: Query DB, populate cache       │
│                                                                             │
│   Invalidation: TTL | Event-based | Version-based                          │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  THE N+1 PROBLEM                                                             │
│  ───────────────                                                             │
│                                                                             │
│   Bad:  SELECT * FROM users;           (1 query)                            │
│         SELECT * FROM orders WHERE user_id = 1;  (N queries)                │
│         SELECT * FROM orders WHERE user_id = 2;                             │
│         ...                                                                  │
│                                                                             │
│   Good: SELECT * FROM users u                                               │
│         LEFT JOIN orders o ON o.user_id = u.id;  (1 query)                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Next Chapter: Why Distributed Systems Are Hard — Understanding the fundamental challenges when code runs on multiple machines*
