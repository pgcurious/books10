# Chapter 16: AWS Core Services Decoded

> **First Principles Question**: AWS has hundreds of services. How do you make sense of them? What are the fundamental building blocks, and how do you decide which services to use for what purpose?

---

## Chapter Overview

AWS offers over 200 services, which can be overwhelming. This chapter cuts through the noise to focus on the core services that form the foundation of most architectures. More importantly, we'll build a mental model for understanding AWS services so you can navigate new offerings confidently.

**What readers will understand after this chapter:**
- The AWS philosophy and service naming patterns
- Core compute services: EC2, Lambda, ECS, EKS
- Storage services: S3, EBS, EFS
- Database services: RDS, DynamoDB, ElastiCache
- Networking fundamentals: VPC, subnets, security groups
- Identity and access: IAM
- How to decide which service for which use case

---

## Section 1: Understanding AWS

### 1.1 The AWS Philosophy

**Everything Is a Service:**
AWS builds discrete services that do one thing well. Need compute? That's EC2. Need object storage? That's S3. Need a queue? That's SQS.

**Services Compose:**
Services are designed to work together:
```
Lambda (compute) + S3 (storage) + DynamoDB (database)
           ↓             ↓              ↓
       Event-driven application architecture
```

**Build vs. Buy Spectrum:**
For any capability, AWS often offers multiple options:
```
More Control                                    More Managed
◄─────────────────────────────────────────────────────────────►

EC2 + MySQL      RDS MySQL      Aurora         Aurora Serverless
(you manage)     (managed)      (optimized)    (auto-scale)
```

### 1.2 Decoding AWS Service Names

**Patterns to Recognize:**
```
"Elastic" = Scalable/Auto-scaling
├── Elastic Compute Cloud (EC2)
├── Elastic Load Balancing (ELB)
├── Elastic Container Service (ECS)
└── ElastiCache

"Simple" = AWS's original naming scheme (not always simple)
├── Simple Storage Service (S3)
├── Simple Queue Service (SQS)
├── Simple Notification Service (SNS)
└── Simple Email Service (SES)

"Amazon" vs "AWS" = Historical (Amazon older, AWS newer)
├── Amazon EC2, Amazon S3 (older)
└── AWS Lambda, AWS Fargate (newer)
```

### 1.3 The AWS Global Infrastructure

**Regions:**
```
Geographic locations with multiple data centers:
├── us-east-1 (N. Virginia) ← Oldest, most services
├── us-west-2 (Oregon)
├── eu-west-1 (Ireland)
├── ap-southeast-1 (Singapore)
└── ... 30+ regions globally
```

**Availability Zones (AZs):**
```
Each region has multiple AZs:
┌─────────────────────────────────────────────────┐
│                  us-east-1                       │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐     │
│  │  us-east- │ │  us-east- │ │  us-east- │     │
│  │    1a     │ │    1b     │ │    1c     │     │
│  │           │ │           │ │           │     │
│  │ Data Ctr  │ │ Data Ctr  │ │ Data Ctr  │     │
│  │ Cluster   │ │ Cluster   │ │ Cluster   │     │
│  └───────────┘ └───────────┘ └───────────┘     │
│        ↑              ↑             ↑           │
│        └──────── Low latency ───────┘           │
│                 High bandwidth                   │
│        Isolated failure domains                  │
└─────────────────────────────────────────────────┘
```

**Design Principle:**
For high availability, deploy across multiple AZs.

---

## Section 2: Compute Services

### 2.1 EC2 — Elastic Compute Cloud

**What It Is:**
Virtual servers in the cloud. The most fundamental AWS compute service.

**Mental Model:**
"Rent a computer by the hour (or second)."

**Instance Types:**
```
Family    Purpose              Example Use Case
────────────────────────────────────────────────────
t3        General, burstable   Web servers, small apps
m6i       General, balanced    Application servers
c6i       Compute optimized    CPU-intensive processing
r6i       Memory optimized     In-memory databases
i3        Storage optimized    High I/O databases
p4        GPU instances        Machine learning
```

**Instance Naming Convention:**
```
m6i.xlarge
│││  │
││└──┴── Size (nano, micro, small, medium, large, xlarge, 2xlarge...)
│└────── Generation (newer = better price/performance)
└─────── Family (m=general, c=compute, r=memory...)
```

**Pricing Models:**
```
On-Demand:      Pay by the hour/second. No commitment.
Reserved:       1-3 year commitment. Up to 72% discount.
Spot:           Bid for unused capacity. Up to 90% discount.
                Can be interrupted with 2-minute warning.
Savings Plans:  Commit to $/hour. Flexible across instance types.
```

**When to Use EC2:**
- Full control over the OS needed
- Legacy applications
- Long-running processes
- Specific instance requirements

### 2.2 Lambda — Serverless Functions

**What It Is:**
Run code without provisioning servers. Pay per invocation.

**Mental Model:**
"Give us a function. We'll run it when triggered."

**How It Works:**
```
Event Sources              Lambda Function          Destinations
─────────────            ─────────────────        ───────────
API Gateway ───────────►│                 │──────► DynamoDB
S3 events   ───────────►│   Your Code     │──────► S3
SQS messages───────────►│   (Java, Node,  │──────► SNS
EventBridge ───────────►│    Python...)   │──────► SQS
DynamoDB Streams ──────►│                 │──────► Other services
                        └─────────────────┘
```

**Example Lambda (Java):**
```java
public class Handler implements RequestHandler<APIGatewayProxyRequestEvent,
                                               APIGatewayProxyResponseEvent> {

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent input,
            Context context) {

        String body = input.getBody();
        // Process the request

        return new APIGatewayProxyResponseEvent()
            .withStatusCode(200)
            .withBody("{\"message\": \"Success\"}");
    }
}
```

**Pricing:**
```
Pay for:
├── Number of requests ($0.20 per 1M requests)
├── Duration × Memory ($0.0000166667 per GB-second)
└── Zero cost when not running
```

**Limitations:**
```
├── 15-minute maximum execution time
├── 10 GB maximum memory
├── 250 MB deployment package (or 10 GB with containers)
├── Cold starts (first invocation latency)
└── Stateless (no local state between invocations)
```

**When to Use Lambda:**
- Event-driven processing
- API backends
- Scheduled tasks
- Short-lived operations
- Variable/unpredictable traffic

### 2.3 Container Services: ECS and EKS

**ECS — Elastic Container Service:**
```
AWS's native container orchestration.

Components:
┌─────────────────────────────────────────────────────────────┐
│                          ECS Cluster                         │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                       Service                           ││
│  │  ┌───────┐  ┌───────┐  ┌───────┐                       ││
│  │  │ Task  │  │ Task  │  │ Task  │  ← Running containers  ││
│  │  └───────┘  └───────┘  └───────┘                       ││
│  └─────────────────────────────────────────────────────────┘│
│                                                              │
│  Launch Types:                                               │
│  ├── EC2: You manage the EC2 instances                      │
│  └── Fargate: AWS manages the infrastructure (serverless)   │
└─────────────────────────────────────────────────────────────┘
```

**EKS — Elastic Kubernetes Service:**
```
Managed Kubernetes.

┌─────────────────────────────────────────────────────────────┐
│                        EKS Cluster                           │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                  Control Plane (AWS manages)            ││
│  │        API Server, etcd, Controllers                    ││
│  └─────────────────────────────────────────────────────────┘│
│                              │                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                   Worker Nodes                          ││
│  │    ┌─────────┐   ┌─────────┐   ┌─────────┐            ││
│  │    │   Pod   │   │   Pod   │   │   Pod   │            ││
│  │    └─────────┘   └─────────┘   └─────────┘            ││
│  │  Node Type:                                             ││
│  │  ├── EC2 (self-managed or managed node groups)         ││
│  │  └── Fargate (serverless)                              ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

**ECS vs. EKS Decision:**
```
Choose ECS when:                Choose EKS when:
├── AWS-only environment        ├── Need Kubernetes specifically
├── Simpler is better           ├── Multi-cloud strategy
├── Smaller team                ├── K8s expertise exists
└── Don't need K8s features     └── Need K8s ecosystem
```

### 2.4 Compute Decision Framework

```
┌─────────────────────────────────────────────────────────────┐
│                    Compute Decision Tree                     │
└─────────────────────────────────────────────────────────────┘
                              │
                Need full OS control?
                     │           │
                    Yes          No
                     │           │
                    EC2          │
                              Short-lived event processing?
                                  │           │
                                 Yes          No
                                  │           │
                               Lambda         │
                                         Containerized?
                                           │        │
                                          Yes       No
                                           │        │
                                     K8s required?  │
                                      │       │     │
                                     Yes      No    └── Elastic Beanstalk
                                      │       │         (if want PaaS)
                                    EKS     ECS/Fargate
```

---

## Section 3: Storage Services

### 3.1 S3 — Simple Storage Service

**What It Is:**
Object storage with virtually unlimited capacity.

**Mental Model:**
"An infinite, durable file system accessed via HTTP."

**Key Concepts:**
```
Bucket:     A container for objects (globally unique name)
Object:     A file + metadata (up to 5 TB)
Key:        The object's path/name within a bucket

Structure:
s3://my-bucket/                    ← Bucket
├── images/                        ← Prefix (not a real folder)
│   ├── logo.png                   ← Object
│   └── banner.jpg                 ← Object
├── data/
│   └── report.csv
└── config.json
```

**Storage Classes:**
```
Class                   Use Case                      Cost
───────────────────────────────────────────────────────────────
Standard               Frequently accessed            $$$
Intelligent-Tiering    Unknown access patterns        $$ (auto-moves)
Standard-IA            Infrequent access, quick need  $$
One Zone-IA            Infrequent, single AZ ok       $
Glacier Instant        Archive, instant retrieval     $
Glacier Flexible       Archive, minutes to hours      ¢
Glacier Deep Archive   Long-term archive, 12+ hours   ¢
```

**Durability:**
99.999999999% (11 9s) — Designed to not lose your data.

**Use Cases:**
```
├── Static website hosting
├── Data lake storage
├── Backup and archive
├── Application assets (images, files)
├── Log storage
└── Big data analytics input/output
```

### 3.2 EBS — Elastic Block Store

**What It Is:**
Block storage volumes for EC2 instances. Like a virtual hard drive.

**Mental Model:**
"A hard drive that can attach to your EC2 instance."

**Key Characteristics:**
```
├── Persistent (survives instance stop/termination if configured)
├── Single AZ (must be in same AZ as instance)
├── Single attachment (one instance at a time, usually)
├── Resizable (can increase size without downtime)
└── Snapshots (backup to S3)
```

**Volume Types:**
```
Type        IOPS          Throughput    Use Case
──────────────────────────────────────────────────────────────
gp3         3,000-16,000  125-1,000 MB/s  General purpose (default)
gp2         3,000-16,000  Scales w/size   Legacy general purpose
io2         Up to 256,000 4,000 MB/s     Databases, high perf
st1         500           500 MB/s       Big data, sequential
sc1         250           250 MB/s       Cold data, infrequent
```

### 3.3 EFS — Elastic File System

**What It Is:**
Managed NFS file system that multiple instances can access.

**Mental Model:**
"A shared network drive that grows automatically."

**Key Characteristics:**
```
├── Shared access (multiple EC2 instances simultaneously)
├── Elastic (grows and shrinks automatically)
├── Multi-AZ (by default)
├── POSIX-compliant (standard file system)
└── NFS protocol
```

**When to Use:**
```
├── Shared configuration files
├── Content management
├── Home directories
├── Container persistent storage
└── When multiple instances need same files
```

### 3.4 Storage Decision Framework

```
                         Storage Decision
                              │
                 Do you need block-level access?
                        │            │
                       Yes           No
                        │            │
              EBS (single)      Do multiple instances
              or EFS (shared)   need access?
                                  │        │
                                 Yes       No
                                  │        │
                                 S3       S3
                           (via SDK)   (via SDK)

Block vs. Object:
─────────────────────────────────────────────────────
EBS/EFS (Block):         S3 (Object):
├── File system mount    ├── HTTP API access
├── Low latency          ├── Virtually unlimited
├── Direct I/O           ├── Per-object access
└── Size limits          └── 11 9s durability
```

---

## Section 4: Database Services

### 4.1 RDS — Relational Database Service

**What It Is:**
Managed relational databases.

**Supported Engines:**
```
├── MySQL
├── PostgreSQL
├── MariaDB
├── Oracle
├── SQL Server
└── Aurora (AWS custom)
```

**What AWS Manages:**
```
┌────────────────────────────────────────────────┐
│  You Manage:                                   │
│  ├── Database schema                           │
│  ├── Query optimization                        │
│  └── Application connection                    │
├────────────────────────────────────────────────┤
│  AWS Manages:                                  │
│  ├── Provisioning                              │
│  ├── Patching                                  │
│  ├── Backup                                    │
│  ├── Recovery                                  │
│  ├── Failure detection                         │
│  ├── Multi-AZ replication                      │
│  └── Storage scaling                           │
└────────────────────────────────────────────────┘
```

**Multi-AZ:**
```
                 Primary                    Standby
            ┌─────────────┐            ┌─────────────┐
            │    RDS      │  Sync      │    RDS      │
App ───────►│  Instance   │───────────►│  Instance   │
            │   (AZ-A)    │ Replication│   (AZ-B)    │
            └─────────────┘            └─────────────┘
                              │
                    Automatic failover
                    (2-3 minutes)
```

**Read Replicas:**
```
            ┌─────────────┐
            │   Primary   │
            │  (writes)   │
            └──────┬──────┘
          Async    │    Async
       ┌───────────┼───────────┐
       ▼           ▼           ▼
┌───────────┐ ┌───────────┐ ┌───────────┐
│  Replica  │ │  Replica  │ │  Replica  │
│  (reads)  │ │  (reads)  │ │  (reads)  │
└───────────┘ └───────────┘ └───────────┘
```

### 4.2 Aurora

**What It Is:**
AWS's cloud-native relational database. MySQL and PostgreSQL compatible.

**Why It Exists:**
```
Traditional RDS:              Aurora:
├── Compute + storage coupled  ├── Compute separate from storage
├── Single instance writes     ├── Storage is distributed
├── Replica lag (seconds)      ├── Replica lag (milliseconds)
└── 64 TB max storage          └── 128 TB, auto-scaling
```

**Aurora Architecture:**
```
┌─────────────────────────────────────────────────────────────┐
│                    Aurora Cluster                            │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐                       │
│  │    Writer    │    │    Reader    │  ← Compute layer      │
│  │   Instance   │    │   Instance   │                       │
│  └──────┬───────┘    └──────┬───────┘                       │
│         │                   │                                │
│         └─────────┬─────────┘                                │
│                   │                                          │
│  ┌────────────────┴───────────────────────────────────────┐ │
│  │           Distributed Storage (6 copies across 3 AZs)   │ │
│  │    ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                │ │
│  │    │ A │ │ B │ │ A │ │ B │ │ A │ │ B │                │ │
│  │    └───┘ └───┘ └───┘ └───┘ └───┘ └───┘                │ │
│  │     AZ-1       AZ-2        AZ-3                        │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Aurora Serverless:**
Auto-scaling compute that can scale to zero. Pay per ACU-hour.

### 4.3 DynamoDB

**What It Is:**
Managed NoSQL database. Key-value and document store.

**Mental Model:**
"A giant, infinitely scalable hash table."

**Core Concepts:**
```
Table:          Collection of items
Item:           A record (like a row)
Attributes:     Data fields (like columns, but flexible)
Primary Key:    Unique identifier
                ├── Partition key (hash)
                └── Optionally + sort key (range)
```

**Example:**
```
Table: Orders

Partition Key: customer_id
Sort Key: order_date

┌─────────────┬────────────┬─────────┬──────────┐
│ customer_id │ order_date │ total   │ status   │
├─────────────┼────────────┼─────────┼──────────┤
│ cust-123    │ 2024-01-15 │ 99.99   │ shipped  │
│ cust-123    │ 2024-02-01 │ 45.00   │ pending  │
│ cust-456    │ 2024-01-20 │ 150.00  │ delivered│
└─────────────┴────────────┴─────────┴──────────┘
```

**Capacity Modes:**
```
Provisioned:    You set read/write capacity units
                Good for: Predictable traffic

On-Demand:      Pay per request
                Good for: Variable/unpredictable traffic
```

**When to Use DynamoDB:**
```
Good for:                       Not good for:
├── Key-value lookups           ├── Complex joins
├── High scale                  ├── Ad-hoc queries
├── Predictable latency         ├── ACID transactions (complex)
├── Schemaless data             ├── Relational data models
└── Serverless architectures    └── Small, consistent workloads
```

### 4.4 ElastiCache

**What It Is:**
Managed in-memory caching. Redis or Memcached.

**Mental Model:**
"Managed Redis/Memcached for your application."

**Use Cases:**
```
├── Session storage
├── Database query caching
├── Leaderboards (Redis sorted sets)
├── Real-time analytics
└── Message queues (Redis pub/sub)
```

**Redis vs. Memcached:**
```
Redis:                         Memcached:
├── Data structures            ├── Simple key-value
├── Persistence                ├── Multi-threaded
├── Replication                ├── No persistence
├── Pub/Sub                    └── Simpler
├── Lua scripting
└── More features
```

### 4.5 Database Decision Framework

```
                      Database Decision
                            │
                  Need relational (SQL)?
                       │          │
                      Yes         No
                       │          │
              Need AWS optimization?   Need flexible schema?
                  │         │              │          │
                 Yes        No            Yes         No
                  │         │              │          │
               Aurora     RDS           DynamoDB    Consider
                                                    DocumentDB
                                                    (MongoDB)

                 Also consider:
                 ├── ElastiCache for caching
                 ├── Neptune for graph data
                 ├── Timestream for time series
                 └── Keyspaces for Cassandra
```

---

## Section 5: Networking — VPC

### 5.1 VPC — Virtual Private Cloud

**What It Is:**
Your own isolated network within AWS.

**Mental Model:**
"Your private data center network in the cloud."

**Key Components:**
```
┌─────────────────────────────────────────────────────────────┐
│                          VPC                                 │
│                     (10.0.0.0/16)                            │
│                                                              │
│  ┌────────────────────────┐  ┌────────────────────────────┐ │
│  │    Public Subnet       │  │    Public Subnet           │ │
│  │      (10.0.1.0/24)     │  │      (10.0.2.0/24)         │ │
│  │         AZ-A           │  │         AZ-B               │ │
│  │  ┌──────────────────┐  │  │  ┌──────────────────┐      │ │
│  │  │   Web Server     │  │  │  │   Web Server     │      │ │
│  │  │  (public IP)     │  │  │  │  (public IP)     │      │ │
│  │  └──────────────────┘  │  │  └──────────────────┘      │ │
│  └───────────┬────────────┘  └───────────┬────────────────┘ │
│              │                           │                   │
│  ┌───────────▼────────────┐  ┌───────────▼────────────────┐ │
│  │   Private Subnet       │  │   Private Subnet           │ │
│  │     (10.0.10.0/24)     │  │     (10.0.20.0/24)         │ │
│  │         AZ-A           │  │         AZ-B               │ │
│  │  ┌──────────────────┐  │  │  ┌──────────────────┐      │ │
│  │  │   App Server     │  │  │  │   App Server     │      │ │
│  │  │  (private IP)    │  │  │  │  (private IP)    │      │ │
│  │  └──────────────────┘  │  │  └──────────────────┘      │ │
│  │  ┌──────────────────┐  │  │  ┌──────────────────┐      │ │
│  │  │   Database       │  │  │  │   Database       │      │ │
│  │  │  (private IP)    │  │  │  │  (replica)       │      │ │
│  │  └──────────────────┘  │  │  └──────────────────┘      │ │
│  └────────────────────────┘  └────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Subnets

**Public Subnet:**
- Has route to Internet Gateway
- Instances can have public IPs
- For resources that need direct internet access

**Private Subnet:**
- No direct internet route
- Instances have only private IPs
- Access internet via NAT Gateway (if needed)

### 5.3 Security Groups

**What They Are:**
Virtual firewalls for your instances.

**Mental Model:**
"Allow lists attached to instances."

**Characteristics:**
```
├── Stateful (return traffic automatically allowed)
├── Allow rules only (no explicit deny)
├── Default: deny all inbound, allow all outbound
└── Attached to instances/resources
```

**Example:**
```
Web Server Security Group:
┌─────────────────────────────────────────────────────────────┐
│ Inbound Rules:                                              │
│ ├── Allow HTTP (80) from 0.0.0.0/0                         │
│ ├── Allow HTTPS (443) from 0.0.0.0/0                       │
│ └── Allow SSH (22) from 10.0.0.0/8 (internal only)         │
│                                                              │
│ Outbound Rules:                                             │
│ └── Allow all traffic to 0.0.0.0/0                         │
└─────────────────────────────────────────────────────────────┘

App Server Security Group:
┌─────────────────────────────────────────────────────────────┐
│ Inbound Rules:                                              │
│ └── Allow 8080 from web-server-sg (reference by SG ID)     │
│                                                              │
│ Outbound Rules:                                             │
│ └── Allow all traffic to 0.0.0.0/0                         │
└─────────────────────────────────────────────────────────────┘
```

### 5.4 NACLs — Network Access Control Lists

**What They Are:**
Firewalls at the subnet level.

**Comparison:**
```
Security Groups:             NACLs:
├── Instance level           ├── Subnet level
├── Stateful                 ├── Stateless
├── Allow only               ├── Allow and deny
├── Evaluate all rules       ├── Rules processed in order
└── Usually sufficient       └── Additional defense layer
```

### 5.5 Load Balancing

**Application Load Balancer (ALB):**
```
├── Layer 7 (HTTP/HTTPS)
├── Path-based routing
├── Host-based routing
├── WebSocket support
└── Best for web applications
```

**Network Load Balancer (NLB):**
```
├── Layer 4 (TCP/UDP)
├── Ultra-low latency
├── Static IP support
├── Millions of requests/second
└── Best for extreme performance
```

**Pattern:**
```
                     Internet
                        │
                 ┌──────▼──────┐
                 │     ALB     │
                 └──────┬──────┘
                        │
         ┌──────────────┼──────────────┐
         │              │              │
    ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
    │ Target  │   │ Target  │   │ Target  │
    │ Group   │   │ Group   │   │ Group   │
    │ (EC2)   │   │ (Lambda)│   │ (ECS)   │
    └─────────┘   └─────────┘   └─────────┘
```

---

## Section 6: Identity and Access — IAM

### 6.1 IAM Fundamentals

**What It Is:**
Identity and Access Management. Controls who can do what in AWS.

**Core Concepts:**
```
Users:       Individual identities (people, applications)
Groups:      Collections of users
Roles:       Assumed identities (for services, cross-account)
Policies:    JSON documents defining permissions
```

### 6.2 IAM Policies

**Policy Structure:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*"
    },
    {
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
```

**Policy Elements:**
```
Effect:     Allow or Deny
Action:     What actions (s3:GetObject, ec2:StartInstance)
Resource:   Which resources (ARNs)
Condition:  Optional conditions (IP, time, MFA, etc.)
```

### 6.3 IAM Roles

**Why Roles Matter:**
```
Bad Practice:                   Good Practice:
EC2 instance with              EC2 instance assumes role:
access keys stored:
                               ┌─────────────────┐
┌─────────────────┐           │   IAM Role      │
│     EC2         │           │  (S3 access)    │
│  ┌───────────┐  │           └────────┬────────┘
│  │ .aws/     │  │                    │ assumes
│  │credentials│  │           ┌────────▼────────┐
│  └───────────┘  │           │     EC2         │
└─────────────────┘           │ (no credentials)│
                              └─────────────────┘
Keys can be leaked!           Temporary, rotated automatically
```

**Common Role Patterns:**
```
├── EC2 instance roles (EC2 → S3, DynamoDB, etc.)
├── Lambda execution roles (Lambda → other services)
├── ECS task roles (containers → services)
├── Cross-account access (Account A → Account B)
└── Service-linked roles (AWS services → your resources)
```

### 6.4 Least Privilege Principle

**Start Restrictive:**
```
1. Start with no permissions
2. Add only what's needed
3. Use specific resource ARNs
4. Avoid wildcard (*) when possible
5. Review and audit regularly
```

**Example Evolution:**
```json
// Too permissive:
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}

// Better:
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-app-bucket/*"
}

// Best (with conditions):
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-app-bucket/*",
  "Condition": {
    "IpAddress": {"aws:SourceIp": "10.0.0.0/8"}
  }
}
```

---

## Section 7: Messaging and Integration

### 7.1 SQS — Simple Queue Service

**What It Is:**
Managed message queue service.

**Mental Model:**
"A reliable inbox for messages between services."

**Queue Types:**
```
Standard Queue:                 FIFO Queue:
├── Nearly unlimited throughput ├── 300 msg/sec (3000 with batching)
├── At-least-once delivery      ├── Exactly-once processing
├── Best-effort ordering        ├── Strict ordering
└── Higher throughput           └── Guaranteed order
```

**Pattern:**
```
Producer                Queue                 Consumer
┌─────────┐         ┌──────────┐          ┌─────────┐
│  Order  │────────►│   SQS    │─────────►│ Order   │
│ Service │ send    │  Queue   │ receive  │Processor│
└─────────┘         └──────────┘          └─────────┘

Benefits:
├── Decoupling (producer doesn't wait)
├── Buffer (handle traffic spikes)
└── Reliability (messages persist until processed)
```

### 7.2 SNS — Simple Notification Service

**What It Is:**
Pub/sub messaging service.

**Mental Model:**
"Send a message to many subscribers at once."

**Pattern:**
```
Publisher              Topic              Subscribers
┌─────────┐        ┌──────────┐        ┌─────────────┐
│ Payment │───────►│   SNS    │───────►│ Email       │
│ Service │ publish│  Topic   │        ├─────────────┤
└─────────┘        └──────────┘───────►│ SMS         │
                                       ├─────────────┤
                        │─────────────►│ SQS Queue   │
                                       ├─────────────┤
                        └─────────────►│ Lambda      │
                                       └─────────────┘
```

### 7.3 EventBridge

**What It Is:**
Serverless event bus for application integration.

**Mental Model:**
"Route events between services with rules."

**Pattern:**
```
Event Sources          Event Bus              Targets
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ AWS Services │───►│              │───►│ Lambda       │
├──────────────┤    │  EventBridge │    ├──────────────┤
│ Custom Apps  │───►│    Rules     │───►│ Step Functions│
├──────────────┤    │              │    ├──────────────┤
│ SaaS Apps    │───►│  (filter &   │───►│ SQS          │
└──────────────┘    │   route)     │    ├──────────────┤
                    └──────────────┘───►│ API Gateway  │
                                        └──────────────┘
```

---

## Section 8: Putting It Together

### 8.1 A Typical Web Application Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                              VPC                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                     Public Subnets                              │ │
│  │                                                                  │ │
│  │    ┌──────────────────────────────────────────────────────┐    │ │
│  │    │               Application Load Balancer               │    │ │
│  │    └───────────────────────┬──────────────────────────────┘    │ │
│  │                            │                                    │ │
│  └────────────────────────────┼────────────────────────────────────┘ │
│                               │                                      │
│  ┌────────────────────────────┼────────────────────────────────────┐ │
│  │                     Private Subnets                             │ │
│  │                            │                                    │ │
│  │    ┌───────────────┬───────┴────────┬───────────────┐          │ │
│  │    │               │                │               │          │ │
│  │    ▼               ▼                ▼               ▼          │ │
│  │ ┌──────┐       ┌──────┐        ┌──────┐       ┌──────┐        │ │
│  │ │ ECS  │       │ ECS  │        │ ECS  │       │ ECS  │        │ │
│  │ │ Task │       │ Task │        │ Task │       │ Task │        │ │
│  │ └──┬───┘       └──┬───┘        └──┬───┘       └──┬───┘        │ │
│  │    │              │               │              │             │ │
│  │    └──────────────┴───────┬───────┴──────────────┘             │ │
│  │                           │                                    │ │
│  │    ┌──────────────────────┴─────────────────────────┐         │ │
│  │    │                                                 │         │ │
│  │    ▼                                                 ▼         │ │
│  │ ┌────────────────┐                      ┌────────────────┐    │ │
│  │ │   Aurora       │                      │  ElastiCache   │    │ │
│  │ │   (Primary)    │                      │    (Redis)     │    │ │
│  │ └────────────────┘                      └────────────────┘    │ │
│  │                                                                │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                        ┌──────────────┐
                        │      S3      │
                        │   (assets,   │
                        │    logs)     │
                        └──────────────┘
```

### 8.2 Service Selection Cheat Sheet

```
┌────────────────────────────────────────────────────────────────────┐
│                    AWS Service Selection Guide                      │
├─────────────────────┬──────────────────────────────────────────────┤
│ Need                │ Service                                       │
├─────────────────────┼──────────────────────────────────────────────┤
│ Run a server        │ EC2, ECS, EKS, Fargate                       │
│ Run code on demand  │ Lambda                                        │
│ Store objects/files │ S3                                            │
│ Block storage       │ EBS                                           │
│ Shared file storage │ EFS                                           │
│ SQL database        │ RDS, Aurora                                   │
│ NoSQL database      │ DynamoDB                                      │
│ In-memory cache     │ ElastiCache                                   │
│ Message queue       │ SQS                                           │
│ Pub/sub             │ SNS                                           │
│ Event routing       │ EventBridge                                   │
│ Load balancing      │ ALB, NLB                                      │
│ CDN                 │ CloudFront                                    │
│ DNS                 │ Route 53                                      │
│ Container registry  │ ECR                                           │
│ CI/CD               │ CodePipeline, CodeBuild, CodeDeploy           │
│ Monitoring          │ CloudWatch                                    │
│ Secrets             │ Secrets Manager, Parameter Store              │
└─────────────────────┴──────────────────────────────────────────────┘
```

---

## Summary: AWS Core Services

**The Mental Model:**
- AWS is a collection of discrete, composable services
- Choose the right abstraction level for your needs
- More managed = less control, less operational burden
- Everything connects via APIs, IAM, and networking

**The Essential Services:**
```
Compute:    EC2 (servers), Lambda (functions), ECS/EKS (containers)
Storage:    S3 (objects), EBS (blocks), EFS (files)
Database:   RDS/Aurora (SQL), DynamoDB (NoSQL), ElastiCache (cache)
Network:    VPC, ALB/NLB, Route 53
Security:   IAM (identity), Security Groups (firewalls)
Messaging:  SQS (queues), SNS (pub/sub), EventBridge (events)
```

**Decision Framework:**
1. What abstraction level do I need? (IaaS vs PaaS vs Serverless)
2. What are my scaling requirements?
3. What operational burden am I willing to accept?
4. What's my cost model? (Steady vs variable)

---

## What's Next?

You now understand AWS's core services. But clicking through the console to create resources is slow and error-prone. Chapter 17 introduces Infrastructure as Code—treating your infrastructure like software: versioned, tested, and reproducible.
