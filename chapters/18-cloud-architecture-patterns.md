# Chapter 18: Cloud Architecture Patterns

> **First Principles Question**: What are the fundamental patterns for building applications in the cloud? How do you design systems that are reliable, scalable, and cost-effective?

---

## Chapter Overview

Cloud architecture isn't just about using cloud services—it's about designing applications that leverage cloud capabilities effectively. This chapter covers the patterns that have emerged as best practices for building modern cloud-native applications.

**What readers will understand after this chapter:**
- High availability and fault tolerance patterns
- Scalability patterns: horizontal vs. vertical
- Data management patterns in distributed systems
- Cost optimization strategies
- Security architecture patterns
- Common anti-patterns to avoid

---

## Section 1: High Availability Patterns

### 1.1 What Is High Availability?

**Definition:**
The ability of a system to remain operational for a high percentage of time.

**Measuring Availability:**
```
Availability    Downtime/year    Downtime/month    Downtime/week
───────────────────────────────────────────────────────────────
99%             3.65 days        7.31 hours        1.68 hours
99.9%           8.77 hours       43.83 minutes     10.08 minutes
99.95%          4.38 hours       21.92 minutes     5.04 minutes
99.99%          52.6 minutes     4.38 minutes      1.01 minutes
99.999%         5.26 minutes     26.3 seconds      6.05 seconds
```

**The Formula:**
```
Availability = Uptime / (Uptime + Downtime)

For serial dependencies:
Total = A1 × A2 × A3
Example: 99.9% × 99.9% × 99.9% = 99.7%

For redundant components:
Total = 1 - (1 - A1) × (1 - A2)
Example: 1 - (0.001 × 0.001) = 99.9999%
```

### 1.2 Multi-AZ Architecture

**Pattern: Deploy Across Availability Zones**
```
                    ┌───────────────────────────────────────────┐
                    │                  Region                    │
                    │                                            │
                    │   ┌──────────────────────────────────┐    │
                    │   │       Application Load Balancer   │    │
                    │   └──────────────┬───────────────────┘    │
                    │                  │                         │
                    │    ┌─────────────┴─────────────┐          │
                    │    │                           │          │
                    │    ▼                           ▼          │
                    │ ┌─────────────────┐  ┌─────────────────┐ │
                    │ │      AZ-A       │  │      AZ-B       │ │
                    │ │ ┌─────────────┐ │  │ ┌─────────────┐ │ │
                    │ │ │ App Server  │ │  │ │ App Server  │ │ │
                    │ │ └─────────────┘ │  │ └─────────────┘ │ │
                    │ │ ┌─────────────┐ │  │ ┌─────────────┐ │ │
                    │ │ │ App Server  │ │  │ │ App Server  │ │ │
                    │ │ └─────────────┘ │  │ └─────────────┘ │ │
                    │ │ ┌─────────────┐ │  │ ┌─────────────┐ │ │
                    │ │ │DB (Primary) │◄┼──┼►│DB (Standby) │ │ │
                    │ │ └─────────────┘ │  │ └─────────────┘ │ │
                    │ └─────────────────┘  └─────────────────┘ │
                    │                                            │
                    └───────────────────────────────────────────┘
```

**Key Principles:**
```
├── Deploy app instances in 2+ AZs
├── Use load balancer to distribute traffic
├── Database with Multi-AZ (synchronous replication)
├── If one AZ fails, others continue serving
└── No single point of failure within region
```

### 1.3 Multi-Region Architecture

**Pattern: Active-Passive Multi-Region**
```
┌────────────────────────┐         ┌────────────────────────┐
│   Region: US-East-1    │         │   Region: US-West-2    │
│       (Primary)        │         │       (Backup)         │
│                        │         │                        │
│  ┌──────────────────┐  │         │  ┌──────────────────┐  │
│  │   Application    │  │         │  │   Application    │  │
│  │   (Active)       │  │         │  │   (Standby)      │  │
│  └────────┬─────────┘  │         │  └────────┬─────────┘  │
│           │            │         │           │            │
│  ┌────────▼─────────┐  │  Async  │  ┌────────▼─────────┐  │
│  │   Database       │──┼─────────┼──│   Database       │  │
│  │   (Read/Write)   │  │  Repli. │  │   (Read Replica) │  │
│  └──────────────────┘  │         │  └──────────────────┘  │
└────────────────────────┘         └────────────────────────┘
              │                                │
              └──────────┬─────────────────────┘
                         │
              ┌──────────▼──────────┐
              │      Route 53       │
              │  (Health Checks +   │
              │   DNS Failover)     │
              └─────────────────────┘
```

**Pattern: Active-Active Multi-Region**
```
┌────────────────────────┐         ┌────────────────────────┐
│   Region: US-East-1    │         │   Region: EU-West-1    │
│                        │         │                        │
│  ┌──────────────────┐  │         │  ┌──────────────────┐  │
│  │   Application    │◄─┼─────────┼──│   Application    │  │
│  │   (Active)       │  │ Traffic │  │   (Active)       │  │
│  └────────┬─────────┘  │         │  └────────┬─────────┘  │
│           │            │         │           │            │
│  ┌────────▼─────────┐  │ Global  │  ┌────────▼─────────┐  │
│  │   DynamoDB       │◄─┼─Table───┼─►│   DynamoDB       │  │
│  │   (Local Table)  │  │         │  │   (Local Table)  │  │
│  └──────────────────┘  │         │  └──────────────────┘  │
└────────────────────────┘         └────────────────────────┘
              │                                │
              └──────────┬─────────────────────┘
                         │
              ┌──────────▼──────────┐
              │      Route 53       │
              │  (Latency-based or  │
              │   Geolocation)      │
              └─────────────────────┘
```

**Trade-offs:**
```
Active-Passive:                    Active-Active:
├── Simpler                        ├── Complex
├── Lower cost                     ├── Higher cost
├── Manual/auto failover           ├── Always serving
├── RTO: minutes to hours          ├── RTO: near-zero
└── Good for DR                    └── Good for global users
```

### 1.4 Health Checks and Auto-Healing

**Pattern: Auto-Healing Infrastructure**
```
┌─────────────────────────────────────────────────────────────┐
│                    Auto-Healing Flow                         │
└─────────────────────────────────────────────────────────────┘

Load Balancer        Health Check        Auto Scaling Group
┌──────────────┐     ┌──────────────┐    ┌───────────────────┐
│              │────►│ GET /health  │    │                   │
│   Checks     │     │              │    │  Replace          │
│   every 10s  │     │ HTTP 200 OK? │────│  unhealthy        │
│              │     │              │    │  instances        │
└──────────────┘     └──────────────┘    └───────────────────┘

Scenario:
1. Instance becomes unhealthy
2. Load balancer stops sending traffic
3. Auto Scaling Group detects unhealthy instance
4. New instance launched automatically
5. Once healthy, traffic resumes
```

**Health Check Types:**
```
ELB Health Checks:
├── HTTP/HTTPS checks (response code)
├── TCP checks (port open)
└── Configurable thresholds

EC2 Health Checks:
├── Instance status checks
├── System status checks
└── EBS volume checks

Application Health Checks:
├── Shallow: "Is the process running?"
├── Deep: "Can I reach the database?"
└── Custom logic in /health endpoint
```

---

## Section 2: Scalability Patterns

### 2.1 Vertical vs Horizontal Scaling

**Vertical Scaling (Scale Up):**
```
Before:                After:
┌─────────────────┐    ┌─────────────────┐
│   t3.medium     │    │    t3.xlarge    │
│   2 vCPU        │───►│    4 vCPU       │
│   4 GB RAM      │    │    16 GB RAM    │
└─────────────────┘    └─────────────────┘

Pros:                  Cons:
├── Simple             ├── Limits exist
├── No code changes    ├── Downtime to resize
└── Immediate          └── Single point of failure
```

**Horizontal Scaling (Scale Out):**
```
Before:                After:
┌─────────────┐       ┌─────────────┐  ┌─────────────┐
│   Server    │       │   Server    │  │   Server    │
│             │  ───► │             │  │             │
└─────────────┘       └─────────────┘  └─────────────┘
                      ┌─────────────┐  ┌─────────────┐
                      │   Server    │  │   Server    │
                      │             │  │             │
                      └─────────────┘  └─────────────┘

Pros:                  Cons:
├── Near-infinite      ├── Complexity
├── No downtime        ├── Code must support it
├── Fault tolerant     └── Stateless requirement
└── Cost-efficient
```

### 2.2 Auto Scaling Patterns

**Reactive Scaling (Metrics-Based):**
```yaml
# Scale based on CPU utilization
Auto Scaling Policy:
  Metric: CPUUtilization
  Target: 70%

Current State:
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  CPU: 85%  ──► Add instances                                 │
│  CPU: 70%  ──► Maintain                                      │
│  CPU: 50%  ──► Remove instances                              │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**Scheduled Scaling:**
```yaml
# Scale for known traffic patterns
Schedule:
  - "0 8 * * MON-FRI": scale to 10 instances (work hours start)
  - "0 18 * * MON-FRI": scale to 5 instances (work hours end)
  - "0 0 * * *": scale to 2 instances (night)
```

**Predictive Scaling:**
```
AWS looks at historical patterns:

Traffic Pattern (learned):
        │    ████
        │   ██████
        │  ████████    █
        │ ██████████  ███
        │████████████████
        └────────────────── Time

Scales BEFORE the spike, not during.
```

### 2.3 Database Scaling Patterns

**Read Replicas:**
```
                ┌───────────────────┐
                │    Primary DB     │
                │  (Reads + Writes) │
                └────────┬──────────┘
                         │
           Async Replication
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Replica   │  │   Replica   │  │   Replica   │
│  (Reads)    │  │  (Reads)    │  │  (Reads)    │
└─────────────┘  └─────────────┘  └─────────────┘

Use Case: Read-heavy workloads (90% reads)
```

**Database Sharding:**
```
                    Application
                         │
                  ┌──────┴──────┐
                  │   Router    │
                  │ (Shard Key) │
                  └──────┬──────┘
                         │
     ┌───────────────────┼───────────────────┐
     │                   │                   │
     ▼                   ▼                   ▼
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│   Shard 1   │   │   Shard 2   │   │   Shard 3   │
│  Users A-H  │   │  Users I-P  │   │  Users Q-Z  │
└─────────────┘   └─────────────┘   └─────────────┘

Use Case: Very large datasets, write-heavy
Complexity: High (cross-shard queries are hard)
```

**Caching Layer:**
```
Application ──► Cache (Redis/ElastiCache) ──► Database

Cache Hit:                    Cache Miss:
App → Cache → Response        App → Cache → DB → Cache → Response
(~1ms)                        (~10-100ms, then cached)

Cache Strategies:
├── Cache-aside: App manages cache
├── Write-through: Write to cache AND DB
├── Write-behind: Write to cache, async to DB
└── Refresh-ahead: Predictively refresh
```

### 2.4 Stateless Application Pattern

**Stateful (Bad for Scaling):**
```
┌──────────────┐           ┌─────────────────┐
│   Server A   │◄─────────►│    User 1       │
│  Session: X  │           │ (must use A!)   │
└──────────────┘           └─────────────────┘
                           ┌─────────────────┐
                           │    User 2       │
                      ✗    │ (Can't use A,   │
                           │  session on B)  │
                           └─────────────────┘
```

**Stateless (Good for Scaling):**
```
┌──────────────┐
│   Server A   │           ┌─────────────────┐
└──────────────┘           │    Any user     │
┌──────────────┐◄─────────►│   can use       │
│   Server B   │           │   any server    │
└──────────────┘           └─────────────────┘
┌──────────────┐
│   Server C   │
└──────────────┘
       │
       │ Sessions stored externally
       ▼
┌──────────────────────────────────────────────┐
│         Redis / DynamoDB / Session Store      │
└──────────────────────────────────────────────┘
```

---

## Section 3: Data Management Patterns

### 3.1 Event Sourcing

**Traditional State Storage:**
```
Current State Only:
┌────────────────────────────────────────┐
│ Account: 12345                         │
│ Balance: $1,500                        │
│ Last Updated: 2024-01-15               │
└────────────────────────────────────────┘

Question: "How did we get to $1,500?"
Answer: "No idea."
```

**Event Sourcing:**
```
Store Events, Derive State:
┌────────────────────────────────────────┐
│ Event 1: AccountOpened    $0           │
│ Event 2: Deposited        +$1,000      │
│ Event 3: Withdrew         -$200        │
│ Event 4: Deposited        +$700        │
│ ──────────────────────────────────     │
│ Current State (calculated): $1,500     │
└────────────────────────────────────────┘

Benefits:
├── Full audit trail
├── Can replay to any point in time
├── Debug exactly what happened
└── Can rebuild state from events
```

### 3.2 CQRS (Command Query Responsibility Segregation)

**Traditional (Same Model for Read/Write):**
```
Application
     │
     ▼
┌─────────────────┐
│  Single Model   │
│ (Read + Write)  │
│                 │
│  Same schema    │
│  Same database  │
└─────────────────┘
```

**CQRS (Separate Read/Write):**
```
                    Application
                         │
         ┌───────────────┴───────────────┐
         │                               │
    Commands                         Queries
    (Write)                          (Read)
         │                               │
         ▼                               ▼
┌─────────────────┐           ┌─────────────────┐
│  Write Model    │──Events──►│  Read Model     │
│ (Normalized)    │           │ (Denormalized)  │
│                 │           │ (Optimized for  │
│ - Validates     │           │  queries)       │
│ - Enforces rules│           │                 │
└─────────────────┘           └─────────────────┘

Benefits:
├── Optimize each side independently
├── Scale reads and writes separately
├── Different storage technologies
└── Read model can be eventual consistency
```

### 3.3 Saga Pattern

**Problem: Distributed Transactions**
```
Order Service       Inventory Service       Payment Service
     │                    │                       │
     │ Begin Transaction  │                       │
     │───────────────────►│                       │
     │                    │ Reserve Item          │
     │                    │──────────────────────►│
     │                    │                       │ Charge Card
     │                    │                       │
                    What if payment fails?
             How do we rollback the reservation?
```

**Saga Pattern (Choreography):**
```
┌──────────────┐     ┌────────────────┐     ┌────────────────┐
│    Order     │     │   Inventory    │     │    Payment     │
│   Service    │     │    Service     │     │    Service     │
└──────┬───────┘     └───────┬────────┘     └───────┬────────┘
       │                     │                      │
       │ OrderCreated        │                      │
       │────────────────────►│                      │
       │                     │ InventoryReserved    │
       │                     │─────────────────────►│
       │                     │                      │
       │                     │      PaymentFailed   │
       │                     │◄─────────────────────│
       │ InventoryReleased   │                      │
       │◄────────────────────│                      │
       │                     │                      │
       │ OrderCancelled      │                      │

Each service:
├── Listens for events
├── Performs local transaction
├── Publishes result event
└── Has compensating action for failures
```

**Saga Pattern (Orchestration):**
```
                    ┌─────────────────┐
                    │  Saga          │
                    │  Orchestrator  │
                    └────────┬────────┘
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
       ▼                     ▼                     ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    Order     │     │  Inventory   │     │   Payment    │
│   Service    │     │   Service    │     │   Service    │
└──────────────┘     └──────────────┘     └──────────────┘

Orchestrator:
├── Knows the steps
├── Calls each service
├── Handles failures
└── Coordinates compensations
```

### 3.4 Outbox Pattern

**Problem: Dual Write**
```
Service:
1. Save to database    ✓ (succeeds)
2. Publish to queue    ✗ (fails)

Result: Inconsistent state
        (Data saved but event not published)
```

**Outbox Pattern:**
```
┌───────────────────────────────────────────────────────┐
│                       Service                          │
│                                                        │
│  1. Single Transaction:                               │
│     ┌──────────────────────────────────────────┐     │
│     │ BEGIN TRANSACTION                         │     │
│     │   INSERT INTO orders (...)                │     │
│     │   INSERT INTO outbox (event_data)         │     │
│     │ COMMIT                                    │     │
│     └──────────────────────────────────────────┘     │
│                                                        │
│  2. Background Process:                               │
│     ┌──────────────────────────────────────────┐     │
│     │ SELECT * FROM outbox WHERE sent = false   │     │
│     │ Publish to message queue                  │     │
│     │ UPDATE outbox SET sent = true             │     │
│     └──────────────────────────────────────────┘     │
│                                                        │
└───────────────────────────────────────────────────────┘

Guarantees:
├── Data and event written atomically
├── Event eventually published
└── No lost events
```

---

## Section 4: Cost Optimization Patterns

### 4.1 Right-Sizing

**Pattern: Match Resources to Workload**
```
Step 1: Monitor actual usage
┌────────────────────────────────────────────────┐
│ Instance: t3.xlarge (4 vCPU, 16 GB)           │
│                                                │
│ CPU Usage:     ████░░░░░░░░░░░░░░░░  20%     │
│ Memory Usage:  ██████░░░░░░░░░░░░░░  30%     │
│                                                │
│ Recommendation: t3.medium (2 vCPU, 4 GB)      │
│ Savings: ~50%                                  │
└────────────────────────────────────────────────┘

Step 2: Resize or use auto-scaling
```

### 4.2 Spot Instances

**Pattern: Use Spot for Fault-Tolerant Workloads**
```
Spot Instance Characteristics:
├── Up to 90% discount
├── Can be interrupted (2-minute warning)
├── Variable availability
└── Same performance as On-Demand

Good For:                    Bad For:
├── Batch processing         ├── Databases
├── CI/CD builds             ├── Stateful applications
├── Dev/test environments    ├── User-facing with strict SLA
├── Stateless web servers    └── Long-running transactions
│   (with capacity buffer)
```

**Mixed Instance Strategy:**
```
Auto Scaling Group:
├── 30% On-Demand (baseline, guaranteed)
├── 70% Spot (cost savings)
└── Multiple instance types for Spot diversity

If Spot interrupted:
├── Auto Scaling launches On-Demand replacement
└── Or launches different Spot type
```

### 4.3 Reserved Capacity

**Pattern: Reserve for Predictable Workloads**
```
Reservation Types:
┌─────────────────────────────────────────────────────────┐
│                      Commitment                          │
├───────────────┬─────────────────┬───────────────────────┤
│ Reserved      │ Savings Plans   │ On-Demand             │
│ Instances     │                 │                       │
├───────────────┼─────────────────┼───────────────────────┤
│ Specific      │ $/hour commit   │ No commitment         │
│ instance type │ Flexible use    │ Full flexibility      │
│ 1-3 years     │ 1-3 years       │                       │
│ Up to 72% off │ Up to 72% off   │ Full price            │
└───────────────┴─────────────────┴───────────────────────┘

Strategy:
├── Reserve baseline capacity (what you always need)
├── Use Savings Plans for predictable growth
├── Use On-Demand/Spot for variable workloads
└── Review and adjust quarterly
```

### 4.4 Storage Tiering

**Pattern: Move Data Based on Access Patterns**
```
S3 Lifecycle Policy Example:
┌─────────────────────────────────────────────────────────┐
│                                                          │
│  Day 0-30:     S3 Standard         $0.023/GB            │
│                    │                                     │
│                    ▼ (after 30 days)                    │
│                                                          │
│  Day 30-90:    S3 Standard-IA      $0.0125/GB           │
│                    │                                     │
│                    ▼ (after 90 days)                    │
│                                                          │
│  Day 90-365:   S3 Glacier Instant  $0.004/GB            │
│                    │                                     │
│                    ▼ (after 1 year)                     │
│                                                          │
│  Day 365+:     S3 Glacier Deep     $0.00099/GB          │
│                                                          │
└─────────────────────────────────────────────────────────┘

Automatic transition based on age.
Significant cost savings for aging data.
```

### 4.5 Serverless for Variable Workloads

**Pattern: Pay Per Use**
```
Traditional (Always On):             Serverless (Pay Per Use):
┌────────────────────────┐          ┌────────────────────────┐
│                        │          │                        │
│  Running 24/7          │          │  Run only when needed  │
│  $50/month             │          │  $5/month              │
│                        │          │  (for same workload)   │
│  ████░░░░████░░░░████  │          │  ████░░░░████░░░░████  │
│  Actual usage: 20%     │          │  Pay for: 20%          │
│                        │          │                        │
└────────────────────────┘          └────────────────────────┘

Serverless Options:
├── Lambda (compute)
├── API Gateway (API management)
├── DynamoDB On-Demand (database)
├── Aurora Serverless (relational)
├── Fargate (containers)
└── S3 (storage, per request)
```

---

## Section 5: Security Patterns

### 5.1 Defense in Depth

**Pattern: Multiple Security Layers**
```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                              │
└─────────────────────────────┬───────────────────────────────┘
                              │
              ┌───────────────▼───────────────┐
              │         CloudFront            │  ← DDoS protection
              │         (WAF Rules)           │  ← Web app firewall
              └───────────────┬───────────────┘
                              │
              ┌───────────────▼───────────────┐
              │    Application Load Balancer  │  ← TLS termination
              │    (Public Subnet)            │  ← Security groups
              └───────────────┬───────────────┘
                              │
              ┌───────────────▼───────────────┐
              │      Application Servers      │  ← Security groups
              │      (Private Subnet)         │  ← Instance roles
              └───────────────┬───────────────┘
                              │
              ┌───────────────▼───────────────┐
              │          Database             │  ← Security groups
              │      (Private Subnet)         │  ← Encryption
              │      (No public access)       │  ← IAM auth
              └───────────────────────────────┘
```

### 5.2 Network Segmentation

**Pattern: Isolate by Trust Level**
```
┌─────────────────────────────────────────────────────────────┐
│                           VPC                                │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                  Public Subnets                         ││
│  │  ┌─────────────────┐  ┌─────────────────┐               ││
│  │  │  Load Balancer  │  │    Bastion      │               ││
│  │  │   (ALB/NLB)     │  │     Host        │               ││
│  │  └─────────────────┘  └─────────────────┘               ││
│  └─────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐│
│  │                Private Subnets (App)                    ││
│  │  ┌─────────────────┐  ┌─────────────────┐               ││
│  │  │   App Server    │  │   App Server    │               ││
│  │  │  (No public IP) │  │  (No public IP) │               ││
│  │  └─────────────────┘  └─────────────────┘               ││
│  └─────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────────────────────────────────────┐│
│  │                Private Subnets (Data)                   ││
│  │  ┌─────────────────┐  ┌─────────────────┐               ││
│  │  │    Database     │  │     Cache       │               ││
│  │  │  (No public IP) │  │  (No public IP) │               ││
│  │  └─────────────────┘  └─────────────────┘               ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘

Security Group Rules:
├── ALB: Allow 443 from internet
├── App: Allow 8080 from ALB only
├── DB: Allow 5432 from App only
└── Each layer only talks to adjacent layers
```

### 5.3 Secrets Management

**Pattern: Centralized Secrets Store**
```
                    ┌─────────────────────┐
                    │   Secrets Manager   │
                    │   or Parameter Store│
                    │                     │
                    │ ├── db-password     │
                    │ ├── api-key         │
                    │ └── tls-cert        │
                    └──────────┬──────────┘
                               │
         IAM Role permissions  │
                               │
    ┌──────────────────────────┼──────────────────────────┐
    │                          │                          │
    ▼                          ▼                          ▼
┌─────────┐              ┌─────────┐              ┌─────────┐
│ Lambda  │              │  EC2    │              │   ECS   │
│ Function│              │ Instance│              │  Task   │
└─────────┘              └─────────┘              └─────────┘

Benefits:
├── Secrets not in code/config files
├── Automatic rotation
├── Audit trail (who accessed what)
├── Encryption at rest
└── Fine-grained access control
```

### 5.4 Zero Trust

**Pattern: Never Trust, Always Verify**
```
Traditional (Perimeter):           Zero Trust:
┌──────────────────────────┐      ┌──────────────────────────┐
│  Trusted Network         │      │  Every request verified  │
│  ┌────────────────────┐  │      │                          │
│  │ All internal       │  │      │  ┌────────────────────┐  │
│  │ traffic trusted    │  │      │  │ Service A          │  │
│  │                    │  │      │  │ ├── Authenticated  │  │
│  │                    │  │      │  │ ├── Authorized     │  │
│  │                    │  │      │  │ └── Encrypted      │  │
│  └────────────────────┘  │      │  └─────────┬──────────┘  │
│                          │      │            │mTLS         │
│  Firewall ───────────────│      │  ┌─────────▼──────────┐  │
│                          │      │  │ Service B          │  │
└──────────────────────────┘      │  │ ├── Authenticated  │  │
                                  │  │ ├── Authorized     │  │
                                  │  │ └── Encrypted      │  │
                                  │  └────────────────────┘  │
                                  └──────────────────────────┘

Principles:
├── Verify every request (no implicit trust)
├── Least privilege access
├── Assume breach (limit blast radius)
└── Encrypt everything
```

---

## Section 6: Anti-Patterns to Avoid

### 6.1 The Monolithic Database

**Anti-Pattern:**
```
┌─────────────────────────────────────────────────────────────┐
│                    All Microservices                         │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │Service A│ │Service B│ │Service C│ │Service D│           │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘           │
│       │          │          │          │                    │
│       └──────────┴──────────┴──────────┘                    │
│                         │                                    │
│                         ▼                                    │
│              ┌─────────────────────┐                        │
│              │   Single Database   │  ← Single point of     │
│              │                     │     failure            │
│              │  All tables mixed   │  ← Can't scale         │
│              │  together           │     independently      │
│              └─────────────────────┘  ← Tight coupling      │
└─────────────────────────────────────────────────────────────┘

Problems:
├── Schema changes affect everyone
├── Can't scale services independently
├── One service can impact others
└── Defeats microservices purpose
```

### 6.2 Synchronous Everything

**Anti-Pattern:**
```
Order Request
     │
     ▼
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  Order  │────►│Inventory│────►│ Payment │────►│Shipping │
│ Service │wait │ Service │wait │ Service │wait │ Service │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
     │
     ▼
Response (after ALL services respond)

Problems:
├── Slow (total time = sum of all services)
├── Fragile (any failure = total failure)
├── Cascading failures
└── Tight coupling
```

**Better: Async Where Possible:**
```
Order Request
     │
     ▼
┌─────────┐     ┌────────────────────────────────────────┐
│  Order  │────►│            Message Queue               │
│ Service │     └────────────────────────────────────────┘
     │                │              │              │
     ▼                ▼              ▼              ▼
Response          ┌────────┐   ┌────────┐   ┌────────┐
(immediate)       │Inventory│   │Payment │   │Shipping│
                  │(async)  │   │(async) │   │(async) │
                  └────────┘   └────────┘   └────────┘
```

### 6.3 Ignoring Failure Modes

**Anti-Pattern:**
```java
// Assuming everything works
public Order createOrder(OrderRequest request) {
    inventoryService.reserve(request.getItems());  // What if this fails?
    paymentService.charge(request.getPayment());   // What if this fails?
    shippingService.schedule(request.getAddress()); // What if this fails?
    return new Order(request);
}
```

**Better: Design for Failure:**
```java
public Order createOrder(OrderRequest request) {
    try {
        // Circuit breaker prevents cascade
        boolean reserved = circuitBreaker.run(
            () -> inventoryService.reserve(request.getItems()),
            () -> false  // Fallback
        );

        if (!reserved) {
            return Order.backordered(request);
        }

        // Retry with backoff
        PaymentResult payment = retry(
            () -> paymentService.charge(request.getPayment()),
            3,  // max attempts
            Duration.ofSeconds(1)  // backoff
        );

        // Compensate if needed
        if (!payment.isSuccess()) {
            inventoryService.release(request.getItems());
            return Order.paymentFailed(request);
        }

        return Order.success(request);

    } catch (Exception e) {
        // Compensating transaction
        compensate(request);
        throw new OrderFailedException(e);
    }
}
```

### 6.4 Lift and Shift Without Optimization

**Anti-Pattern:**
```
On-Premises:                    Cloud (Lift & Shift):
┌─────────────────┐            ┌─────────────────┐
│  Big Server     │            │  Big EC2        │
│  $5,000/month   │   ────►    │  $3,000/month   │
│  (owned)        │            │  (rented)       │
└─────────────────┘            └─────────────────┘

Result: Higher costs, no cloud benefits
```

**Better: Modernize:**
```
Cloud-Optimized:
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  Auto-scaling group                                         │
│  ├── Scale based on demand                                  │
│  ├── Multiple small instances                               │
│  └── Mix of Spot and On-Demand                             │
│                                                              │
│  Managed services                                           │
│  ├── RDS instead of self-managed DB                        │
│  ├── ElastiCache instead of self-managed Redis             │
│  └── CloudWatch instead of self-managed monitoring         │
│                                                              │
│  Result: $1,000/month average, auto-scaling, managed        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary: Cloud Architecture Patterns

**High Availability:**
```
├── Multi-AZ for regional resilience
├── Multi-Region for global resilience
├── Auto-healing with health checks
└── Redundancy at every layer
```

**Scalability:**
```
├── Horizontal scaling (scale out, not up)
├── Auto Scaling based on metrics
├── Stateless applications
├── Read replicas and caching
└── Database sharding for extreme scale
```

**Data Management:**
```
├── Event Sourcing for audit and replay
├── CQRS for optimized read/write
├── Saga pattern for distributed transactions
├── Outbox pattern for reliable publishing
```

**Cost Optimization:**
```
├── Right-size resources
├── Spot for fault-tolerant workloads
├── Reserved for baseline
├── Storage tiering
└── Serverless for variable load
```

**Security:**
```
├── Defense in depth (multiple layers)
├── Network segmentation
├── Centralized secrets management
├── Zero trust (always verify)
```

---

## What's Next?

Cloud architecture doesn't exist in isolation—it's built and evolved through collaboration and change. Chapter 19 explores version control, the foundation of modern software development that enables teams to work together and track every change to both code and infrastructure.
