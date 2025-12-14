# Chapter 8: Why Distributed Systems Are Hard

> **First Principles Question**: What fundamentally changes when your program runs on multiple machines instead of one? Why do problems that seem trivial locally become nearly impossible in distributed systems?

---

## Chapter Overview

You've built a successful application on a single server. Now you need to scale—handle more users, provide higher availability, expand globally. The obvious solution: add more machines. But the moment your application spans multiple computers, you enter a world where the rules change completely.

This chapter explores the fundamental challenges of distributed systems—not specific technologies, but the underlying physics and logic that make distributed computing genuinely hard.

**What readers will understand after this chapter:**
- The eight fallacies of distributed computing
- Why "just add more servers" isn't simple
- Network partitions and the CAP theorem
- Consistency models and their trade-offs
- Time and ordering in distributed systems
- Failure modes unique to distributed systems

---

## Section 1: The Eight Fallacies of Distributed Computing

### 1.1 The False Assumptions

In 1994, Peter Deutsch (later extended by James Gosling) identified false assumptions that developers make when building distributed systems:

**The Eight Fallacies:**
1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

**Why These Matter:**
Code that works perfectly on a single machine fails spectacularly when any of these assumptions break—and they all will.

### 1.2 Fallacy 1: The Network Is Reliable

**The Assumption:**
When you send a message, it arrives.

**The Reality:**
```
Node A sends to Node B:
- Packet dropped by overloaded router
- Cable physically damaged
- Network interface crashed
- Destination server rebooting
- Firewall rule changed
- Cloud provider having issues
```

**The Problem This Creates:**
```java
// Local call - will either succeed or throw
result = database.save(record);

// Network call - many failure modes
result = remoteService.save(record);
// Did it:
// - Succeed?
// - Fail before processing?
// - Succeed but response was lost?
// - Timeout but actually completed?
```

**You Cannot Distinguish:**
- Server never received request
- Server received but failed
- Server succeeded but response lost

This is the fundamental ambiguity of distributed systems.

### 1.3 Fallacy 2: Latency Is Zero

**The Assumption:**
Network calls are as fast as local calls.

**The Reality:**
```
Local function call:    ~100 nanoseconds
Same datacenter:        ~500,000 nanoseconds (0.5ms)
Cross-continent:        ~150,000,000 nanoseconds (150ms)

Difference: 1,500,000x slower
```

**The Problem This Creates:**
```java
// This is fine locally
for (Order order : orders) {
    Customer customer = customerService.get(order.getCustomerId());
    // ...
}

// With 1000 orders and 50ms latency:
// 1000 * 50ms = 50 seconds!
```

**Implications:**
- Chatty protocols kill performance
- Batch operations become essential
- Data locality matters enormously

### 1.4 Fallacy 3: Bandwidth Is Infinite

**The Assumption:**
You can send as much data as you want.

**The Reality:**
- Network links have capacity limits
- Shared infrastructure means contention
- Cloud egress costs money
- Mobile networks are constrained

**The Problem This Creates:**
```java
// Seems innocent
List<Product> allProducts = catalogService.getAll();

// 100,000 products * 10KB each = 1GB transfer
// On a 100Mbps link: 80+ seconds
// Plus serialization/deserialization time
```

### 1.5 The Other Fallacies

**Fallacy 4: The Network Is Secure**
Every network call is a potential attack vector. Encrypt, authenticate, authorize—everywhere.

**Fallacy 5: Topology Doesn't Change**
Servers move, IP addresses change, DNS updates. Hardcoding addresses is fragile.

**Fallacy 6: There Is One Administrator**
Multiple teams, multiple organizations, different policies. You don't control everything.

**Fallacy 7: Transport Cost Is Zero**
Cloud egress fees, serialization CPU cost, network infrastructure expenses add up.

**Fallacy 8: The Network Is Homogeneous**
Different protocols, different speeds, different reliability across your infrastructure.

---

## Section 2: The CAP Theorem

### 2.1 The Impossible Triangle

**The Theorem (Eric Brewer, 2000):**
A distributed system can provide at most two of these three guarantees:

- **Consistency (C)**: Every read receives the most recent write
- **Availability (A)**: Every request receives a response (success or failure)
- **Partition Tolerance (P)**: System continues operating despite network partitions

**The Catch:**
Network partitions are not optional—they will happen. So you must choose between consistency and availability when partitions occur.

### 2.2 Understanding Partitions

**What Is a Network Partition?**
Some nodes can communicate with each other, but not with other nodes.

```
Normal state:
[Node A] ←→ [Node B] ←→ [Node C]

Partition:
[Node A] ←→ [Node B]    [Node C]
         (can't reach C)
```

**Causes:**
- Network switch failure
- Datacenter connectivity loss
- Cloud availability zone issue
- Misconfigured firewall
- Software bugs

**The Hard Question:**
Node A receives a write request. Node C has the current data but is unreachable. What do you do?

### 2.3 CP Systems: Choose Consistency

**Behavior:**
When partition occurs, refuse requests that might violate consistency.

```
Client → Node A: "Write X=5"
Node A: "I can't reach Node C to ensure consistency"
Node A → Client: "Error: Service unavailable"
```

**Examples:**
- Traditional databases (PostgreSQL, MySQL with sync replication)
- ZooKeeper
- etcd
- HBase

**Trade-off:**
System is correct but sometimes unavailable.

**When to Choose:**
- Financial transactions
- Inventory management
- Any system where incorrect data is worse than no data

### 2.4 AP Systems: Choose Availability

**Behavior:**
When partition occurs, continue serving requests, accepting temporary inconsistency.

```
Client → Node A: "Write X=5"
Node A: "I'll accept this even though I can't reach Node C"
Node A → Client: "Success"

Meanwhile, Node C still thinks X=old_value
```

**Examples:**
- Cassandra
- DynamoDB (default settings)
- CouchDB
- DNS

**Trade-off:**
System is always available but might return stale data.

**When to Choose:**
- Shopping carts (merge later)
- Social media feeds (eventual consistency is fine)
- Caching systems
- Any system where availability matters more than immediate consistency

### 2.5 CAP in Practice: It's a Spectrum

**Reality Is More Nuanced:**
- Most systems are configurable
- Different operations can have different guarantees
- "Consistency" and "availability" have degrees

**Example: Cassandra**
```java
// Per-query tuning
Statement query = QueryBuilder
    .select().from("users")
    .setConsistencyLevel(ConsistencyLevel.QUORUM);  // Stronger consistency

Statement query = QueryBuilder
    .select().from("posts")
    .setConsistencyLevel(ConsistencyLevel.ONE);     // Higher availability
```

---

## Section 3: Consistency Models

### 3.1 What Is Consistency?

**The Question:**
When you write data, who sees it and when?

**On a Single Machine:**
If you write X=5, the next read of X returns 5. Always. Simple.

**On Multiple Machines:**
If you write X=5 to Node A, what does a read from Node B return?

### 3.2 Strong Consistency

**Guarantee:**
After a write completes, all subsequent reads (from any node) return that value.

```
Time →
Write X=5 on Node A    |    Read X from Node B
        ↓                         ↓
    [completes]              returns 5 (guaranteed)
```

**How It's Achieved:**
- Synchronous replication
- Consensus protocols (Paxos, Raft)
- Two-phase commit

**Cost:**
- Higher latency (must wait for acknowledgment)
- Lower availability (can't proceed if nodes unreachable)

### 3.3 Eventual Consistency

**Guarantee:**
If no new updates are made, eventually all reads return the last written value.

```
Time →
Write X=5 on Node A
        ↓
Read from Node B might return old value
        ↓
... time passes, replication happens ...
        ↓
Read from Node B returns 5
```

**Key Insight:**
"Eventually" is usually milliseconds to seconds, not hours or days.

**The Challenge:**
What happens if you read your own write from a different node?

```java
// User updates profile on Node A
profileService.update(userId, newProfile);

// Load balancer routes next request to Node B
Profile profile = profileService.get(userId);
// Might return OLD profile! User sees their change "lost"
```

### 3.4 In-Between: Session Guarantees

**Read Your Writes:**
A client always sees their own writes.

```
Session X: Write(A=1) → Read(A) returns 1 (guaranteed)
Session Y: Read(A) might return old value (okay)
```

**Monotonic Reads:**
If you've seen value X, you won't see an older value later.

```
Read returns version 5
Later read won't return version 4
```

**Monotonic Writes:**
Your writes are applied in order.

```
Write X=1, then Write X=2
All nodes eventually have X=2, never X=1 after seeing X=2
```

### 3.5 Causal Consistency

**Guarantee:**
Operations that are causally related are seen in the same order everywhere.

```
Alice: "Let's meet at 3pm"    (message 1)
Bob:   "Sure, sounds good"    (message 2, responds to 1)

Causal consistency ensures everyone sees message 1 before message 2
```

**Without Causal Consistency:**
```
Charlie might see:
"Sure, sounds good"
[later]
"Let's meet at 3pm"
```

---

## Section 4: The Problem of Time

### 4.1 Clocks Are Unreliable

**The Assumption:**
Computers have accurate clocks.

**The Reality:**
```
Computer clock accuracy: typically ±10ms to ±100ms
Network time sync (NTP): ±1ms to ±10ms best case
Clock drift: up to 100ms per day without sync
```

**The Problem:**
If Node A says "this happened at 10:00:00.000" and Node B says "this happened at 10:00:00.001", which actually happened first?

You don't know. The clocks might be wrong.

### 4.2 Types of Clocks

**Wall Clock (Time of Day):**
- What time is it?
- Can jump backward (NTP correction)
- Not suitable for measuring durations

**Monotonic Clock:**
- How much time has elapsed?
- Only moves forward
- Good for timeouts, not coordination

**Neither Helps:**
When comparing events across machines, physical time is fundamentally unreliable.

### 4.3 Logical Clocks

**Lamport Timestamps:**
Instead of physical time, count events:

```
Node A: event happens, counter = 1
Node A: sends message to B, includes counter
Node B: receives message, sets counter = max(local, received) + 1
```

**Guarantee:**
If event A caused event B, timestamp(A) < timestamp(B).

**Limitation:**
If timestamp(A) < timestamp(B), A might have caused B (or they might be concurrent).

### 4.4 Vector Clocks

**Better Causality Tracking:**
Each node maintains a vector of counters:

```
[A:0, B:0, C:0]  Initial state

Node A does work:
[A:1, B:0, C:0]

Node A sends to B:
Node B receives, updates:
[A:1, B:1, C:0]

Node C does independent work:
[A:0, B:0, C:1]
```

**Detecting Conflicts:**
If neither vector dominates the other, events are concurrent (conflict).

```
[A:1, B:0] vs [A:0, B:1]  → Concurrent! Need conflict resolution
[A:2, B:1] vs [A:1, B:1]  → First dominates, no conflict
```

### 4.5 Time in Practice

**Google's TrueTime:**
GPS and atomic clocks in datacenters, providing bounded uncertainty.

```java
TrueTime.now() returns: [earliest, latest]
// Actual time is somewhere in that interval
```

**Spanner's Approach:**
Wait for uncertainty interval to pass before committing.
Expensive but provides external consistency.

**For Most Systems:**
- Use logical clocks for ordering
- Use physical time only for human display
- Accept that "happened before" has limitations

---

## Section 5: Failure Modes

### 5.1 Types of Failures

**Crash Failures:**
Node stops completely, doesn't respond.
- Easiest to handle
- Can use timeouts to detect
- Restart and recover

**Omission Failures:**
Node fails to send or receive some messages.
- Network drops packets
- Server too busy to respond
- Hard to distinguish from crash

**Timing Failures:**
Node responds, but too slowly.
- Missed timeout doesn't mean node failed
- "Is it dead or just slow?"

**Byzantine Failures:**
Node behaves arbitrarily—corrupts data, lies, acts maliciously.
- Hardest to handle
- Requires specialized protocols
- Usually only in high-security contexts

### 5.2 The Two Generals Problem

**The Scenario:**
Two armies (databases) must attack (commit) at the same time. They communicate by messenger (network). Messages can be lost.

```
Army A: "Attack at dawn?"
[messenger crosses enemy territory]
Army B: "Agreed!"
[messenger returns]
Army A: "But did they get my confirmation of their agreement?"
[infinite regress]
```

**The Theorem:**
There is no protocol that guarantees both armies attack if any message can be lost.

**Implication:**
You cannot achieve perfect agreement in an unreliable network. All distributed protocols make trade-offs.

### 5.3 The Byzantine Generals Problem

**Harder Version:**
Some generals (nodes) might be traitors (byzantine failures). They might send different messages to different armies.

**Requirement:**
Loyal generals must agree on a plan even if some generals lie.

**Solution:**
Byzantine fault tolerance requires 3f+1 nodes to tolerate f failures.
For 1 traitor, need at least 4 generals.

**Where This Matters:**
- Blockchain consensus
- Safety-critical systems
- Multi-tenant cloud infrastructure

### 5.4 Detecting Failures

**The Fundamental Problem:**
```
Node A sends request to Node B, no response.
Is Node B:
- Crashed?
- Network partitioned?
- Just slow?
- Received request but response lost?
```

**You cannot know.**

**Approaches:**

**Timeouts:**
```java
if (no response within 5 seconds) {
    assume node failed
    // But maybe it's just slow!
}
```

**Heartbeats:**
```java
// Node sends periodic "I'm alive" messages
// Absence of heartbeats suggests failure
// But network might be dropping heartbeats
```

**Phi Accrual Failure Detection:**
Instead of binary (alive/dead), calculate probability of failure based on heartbeat patterns.

### 5.5 Handling Failures

**Retry with Idempotency:**
```java
// If operation might have succeeded, safe to retry IF idempotent
void setUserEmail(userId, email) {
    // Idempotent: calling twice has same effect as once
}

void incrementCounter(counterId) {
    // NOT idempotent: calling twice increments twice!
    // Need different strategy
}
```

**Circuit Breakers:**
```java
// Stop calling failed service, give it time to recover
if (recentFailures > threshold) {
    return fallbackResponse();
}
```

**Bulkheads:**
Isolate components so one failure doesn't cascade.

---

## Section 6: Consensus

### 6.1 The Consensus Problem

**Goal:**
Multiple nodes agree on a single value, even if some nodes fail.

**Requirements:**
- **Agreement**: All non-faulty nodes decide the same value
- **Validity**: The decided value was proposed by some node
- **Termination**: All non-faulty nodes eventually decide

**Why It's Hard:**
Network partitions, message delays, and failures combine to make this surprisingly difficult.

### 6.2 Paxos

**The Classic Algorithm (Leslie Lamport, 1989):**

**Roles:**
- Proposers: Suggest values
- Acceptors: Vote on values
- Learners: Learn the decided value

**Basic Flow:**
1. **Prepare**: Proposer sends proposal number to acceptors
2. **Promise**: Acceptors promise not to accept older proposals
3. **Accept**: Proposer sends value with proposal number
4. **Accepted**: Acceptors accept if they haven't promised to ignore it

**Why It Works:**
Any two majorities overlap, so once a majority accepts, the value is decided.

**Why It's Hard:**
The full protocol handles many edge cases. "Paxos is simple" is a common joke among distributed systems engineers.

### 6.3 Raft

**Designed for Understandability (2014):**

**Key Insight:**
Leader-based consensus is easier to understand.

**How It Works:**
1. Elect a leader
2. Leader handles all writes
3. Leader replicates to followers
4. If leader fails, elect new leader

**Leader Election:**
- Nodes start as followers
- If no heartbeat from leader, become candidate
- Request votes from other nodes
- Majority wins

**Log Replication:**
```
Leader receives write
Leader appends to log
Leader sends to followers
Majority acknowledge
Leader commits
Leader notifies followers to commit
```

**Why Raft Is Popular:**
- Easier to understand than Paxos
- Widely implemented (etcd, Consul, CockroachDB)
- Well-tested in production

### 6.4 When You Need Consensus

**Leader Election:**
Who's in charge?

**Configuration Changes:**
Cluster membership, settings.

**Distributed Locks:**
Ensure only one node holds a lock.

**Atomic Broadcast:**
All nodes see messages in the same order.

**When You Don't:**
Consensus is expensive. For many use cases, eventual consistency is sufficient and much simpler.

---

## Section 7: Coordination Services

### 7.1 What They Provide

**Building Blocks for Distributed Systems:**
- Key-value storage with strong consistency
- Watches/notifications
- Distributed locks
- Leader election
- Sequential consistency

### 7.2 ZooKeeper

**Data Model:**
Hierarchical namespace (like a file system):
```
/
├── services
│   ├── users
│   │   ├── node1 (ephemeral)
│   │   └── node2 (ephemeral)
│   └── orders
└── config
    └── database-url
```

**Key Features:**
- **Ephemeral nodes**: Disappear when session ends
- **Sequential nodes**: Automatic numbering
- **Watches**: Get notified on changes

**Common Patterns:**

**Service Discovery:**
```java
// Service registers itself
zk.create("/services/users/node-" + myId, myAddress, EPHEMERAL_SEQUENTIAL);

// Client watches for changes
List<String> nodes = zk.getChildren("/services/users", watchCallback);
```

**Leader Election:**
```java
// Each node creates sequential ephemeral node
String path = zk.create("/election/node-", data, EPHEMERAL_SEQUENTIAL);

// Node with lowest sequence number is leader
List<String> nodes = zk.getChildren("/election");
if (path.equals(lowestNode(nodes))) {
    becomeLeader();
} else {
    watchPreviousNode();  // Avoid herd effect
}
```

### 7.3 etcd

**Modern Alternative:**
- Raft-based consensus
- gRPC API
- Kubernetes' backing store
- Simpler data model (flat key-value)

```go
// Put a value
client.Put(ctx, "/services/users/1", "10.0.0.1:8080")

// Watch for changes
watchChan := client.Watch(ctx, "/services/users/", clientv3.WithPrefix())
for response := range watchChan {
    for _, event := range response.Events {
        // Handle service registration/deregistration
    }
}
```

### 7.4 When to Use Coordination Services

**Good Use Cases:**
- Leader election
- Service discovery
- Distributed configuration
- Distributed locks (with caution)

**Bad Use Cases:**
- High-volume data storage
- Caching
- Message queuing
- Anything that needs high throughput

**The Warning:**
Coordination services are a scalability bottleneck by design. They prioritize correctness over throughput.

---

## Section 8: Distributed Transactions

### 8.1 The Problem

**Single Database:**
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- Either both happen or neither
```

**Multiple Databases/Services:**
```java
accountService.debit(account1, 100);   // Service A
accountService.credit(account2, 100);  // Service B
// What if second call fails?
```

### 8.2 Two-Phase Commit (2PC)

**The Protocol:**

**Phase 1 - Prepare:**
```
Coordinator → Participants: "Can you commit?"
Participants: Do all work, write to log
Participants → Coordinator: "Yes, I'm prepared" or "No, abort"
```

**Phase 2 - Commit/Abort:**
```
If all said yes:
    Coordinator → Participants: "Commit"
    Participants: Make changes permanent
Else:
    Coordinator → Participants: "Abort"
    Participants: Rollback
```

**The Problem: Blocking:**
If coordinator fails after sending "prepare" but before sending "commit/abort", participants are stuck. They can't commit (might need to abort) and can't abort (might need to commit).

### 8.3 Sagas

**Alternative: Compensating Transactions:**

Instead of holding locks across services, execute operations and compensate if later steps fail.

```
T1: Create order
T2: Reserve inventory
T3: Charge payment
T4: Ship order

If T3 fails:
C2: Release inventory
C1: Cancel order
```

**Two Coordination Styles:**

**Choreography:**
Each service publishes events, others react.
```
Order Service → publishes "OrderCreated"
Inventory Service → listens, reserves stock, publishes "InventoryReserved"
Payment Service → listens, charges, publishes "PaymentCompleted"
```

**Orchestration:**
Central coordinator tells each service what to do.
```java
public void createOrder(OrderRequest request) {
    Order order = orderService.create(request);
    try {
        inventoryService.reserve(order.getItems());
        paymentService.charge(order.getTotal());
        shippingService.schedule(order);
    } catch (InventoryException e) {
        orderService.cancel(order);
        throw e;
    } catch (PaymentException e) {
        inventoryService.release(order.getItems());
        orderService.cancel(order);
        throw e;
    }
}
```

### 8.4 Practical Advice

**Avoid Distributed Transactions When Possible:**
- Design bounded contexts to minimize cross-service transactions
- Accept eventual consistency where appropriate
- Use idempotency to make retries safe

**When You Must:**
- Sagas are usually better than 2PC
- Make compensations idempotent
- Plan for partial failures
- Monitor saga state carefully

---

## Section 9: Now You Understand Why...

Armed with this foundation, you can now understand:

- **Why microservices are harder than monoliths**: Every service boundary is a network boundary with all associated problems

- **Why "just add caching" isn't simple**: Cache consistency, invalidation, and coordination are distributed systems problems

- **Why databases have "weird" consistency levels**: They're managing trade-offs between consistency, availability, and performance

- **Why exactly-once delivery is called impossible**: Network failures create fundamental ambiguity about whether operations succeeded

- **Why Kubernetes exists**: Running distributed systems requires coordination, consensus, and failure handling—which is exactly what Kubernetes provides

- **Why cloud services have SLAs with "9s"**: In distributed systems, partial failures are constant. The question is how many 9s of availability you need.

---

## Practical Exercises

### Exercise 1: Simulate Network Partitions
Use tools like `tc` (traffic control) on Linux to add latency and packet loss:
```bash
# Add 100ms latency
tc qdisc add dev eth0 root netem delay 100ms

# Add 10% packet loss
tc qdisc add dev eth0 root netem loss 10%
```
Test how your application behaves.

### Exercise 2: Demonstrate CAP
Set up a 3-node database cluster (e.g., PostgreSQL with streaming replication or Cassandra). Introduce a network partition and observe:
- Does the system stay available?
- Can you read stale data?
- What happens when the partition heals?

### Exercise 3: Implement a Saga
Build a simple order processing saga with:
- Order creation
- Inventory reservation
- Payment processing
- Compensating transactions for failures

### Exercise 4: Leader Election
Using ZooKeeper or etcd, implement a simple leader election:
- Multiple instances register
- One becomes leader
- Verify failover when leader dies

### Exercise 5: Measure Clock Drift
On two machines:
```bash
# Check NTP offset
ntpq -p

# Compare times
date +%s.%N
```
Run for a day, measure drift.

---

## Key Takeaways

1. **The network is hostile**. It fails, partitions, loses messages, delivers duplicates, reorders—plan for all of it.

2. **You must choose trade-offs**. CAP theorem means you can't have everything. Understand your requirements.

3. **Time is unreliable**. Don't trust wall clocks for ordering events across machines.

4. **Failures are partial**. Not "system works" or "system fails," but "some parts work, some fail, you're not sure which."

5. **Consensus is expensive**. Use it where correctness requires it, accept eventual consistency elsewhere.

6. **Distributed transactions are hard**. Design to minimize them. When needed, prefer sagas over 2PC.

---

## Looking Ahead

Understanding WHY distributed systems are hard prepares you to make informed decisions about WHEN to distribute. Chapter 9 examines microservices—a popular architectural style that embraces distribution—and the trade-offs involved.

---

## Chapter 8 Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WHY DISTRIBUTED SYSTEMS ARE HARD                          │
│                                                                             │
│  THE EIGHT FALLACIES                                                        │
│  ──────────────────                                                         │
│  1. Network is reliable        5. Topology doesn't change                   │
│  2. Latency is zero            6. One administrator                         │
│  3. Bandwidth is infinite      7. Transport cost is zero                    │
│  4. Network is secure          8. Network is homogeneous                    │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  CAP THEOREM                                                                │
│  ───────────                                                                │
│                                                                             │
│         Consistency ──── Availability                                       │
│              \              /                                               │
│               \    CAP    /                                                 │
│                \        /                                                   │
│            Partition Tolerance                                              │
│                                                                             │
│   Network partitions WILL happen → Choose C or A when they do              │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  CONSISTENCY MODELS                                                         │
│  ─────────────────                                                          │
│                                                                             │
│   Strong          Eventual        Causal                                    │
│   ──────          ────────        ──────                                    │
│   Every read      Eventually      Causally related                          │
│   sees latest     converges       ops seen in order                         │
│   write                                                                     │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  FAILURE DETECTION                                                          │
│  ─────────────────                                                          │
│                                                                             │
│   No response from Node B means:                                            │
│   - Node B crashed?                                                         │
│   - Network partition?                                                      │
│   - Node B is slow?                                                         │
│   - Response was lost?                                                      │
│   YOU CANNOT KNOW.                                                          │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  CONSENSUS PROTOCOLS                                                        │
│  ───────────────────                                                        │
│                                                                             │
│   Problem: Multiple nodes must agree on a value                             │
│                                                                             │
│   Paxos: Classic, complex, proven                                           │
│   Raft: Understandable, leader-based, widely used                          │
│                                                                             │
│   Used for: Leader election, config, distributed locks                      │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  DISTRIBUTED TRANSACTIONS                                                   │
│  ────────────────────────                                                   │
│                                                                             │
│   2PC: Prepare → Commit (blocking, coordinator is single point of failure) │
│   Sagas: Execute → Compensate on failure (more resilient)                  │
│                                                                             │
│   Best advice: Avoid when possible, design bounded contexts                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Next Chapter: Microservices: The Trade-offs — When distribution is the right choice, and when it isn't*
