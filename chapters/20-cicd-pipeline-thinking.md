# Chapter 20: CI/CD Pipeline Thinking

> **First Principles Question**: Why do we need automated pipelines? What problems do Continuous Integration and Continuous Deployment solve, and how should you think about designing them?

---

## Chapter Overview

CI/CD has become essential to modern software development. But it's easy to set up pipelines without understanding why they matter. This chapter builds understanding from first principles: what problems CI/CD solves, how to design effective pipelines, and the practices that make them valuable.

**What readers will understand after this chapter:**
- Why CI/CD exists and what problems it solves
- The difference between Continuous Integration, Delivery, and Deployment
- Pipeline design principles and stages
- Testing strategies in pipelines
- Deployment strategies: blue-green, canary, rolling
- Common patterns and anti-patterns

---

## Section 1: The Problems CI/CD Solves

### 1.1 The Integration Hell Problem

**Before Continuous Integration:**
```
Timeline of a 3-month project:

Week 1-10:  Everyone works independently
            ├── Alice builds Feature A
            ├── Bob builds Feature B
            └── Carol builds Feature C

Week 11:    "Integration Week" begins
            ├── Merge Alice's code... conflicts!
            ├── Merge Bob's code... breaks Alice's!
            ├── Merge Carol's code... nothing works!
            └── Panic, late nights, finger-pointing

Week 12:    Still fixing integration issues
            ├── "It worked on my machine!"
            ├── Different assumptions about interfaces
            └── Incompatible changes

Week 13+:   Delayed release, technical debt
```

**The Core Problem:**
```
The longer you wait to integrate, the harder it gets.

Integration pain grows exponentially:

Pain │
     │                    ███
     │               ████████
     │          █████████████
     │     ██████████████████
     │████████████████████████
     └────────────────────────► Time since last integration
```

### 1.2 The "Works on My Machine" Problem

**Without Standardized Builds:**
```
Developer A:                     Developer B:
├── Java 17                      ├── Java 11
├── Maven 3.9                    ├── Maven 3.6
├── Node 18                      ├── Node 16
├── Windows                      ├── Mac
└── "Works perfectly!"           └── "Works perfectly!"

Production Server:
├── Java 17
├── Maven 3.8
├── Node 20
├── Linux
└── "Nothing works!"
```

### 1.3 The Manual Deployment Problem

**Manual Deployment Process:**
```
1. Developer: "Deploy complete!"
2. Ops: "Which version?"
3. Developer: "The latest one."
4. Ops: "Where is it?"
5. Developer: "I'll send you the JAR."
6. Ops: "How do I configure it?"
7. Developer: "There's a README somewhere..."
8. Ops: (copies files manually, crosses fingers)
9. Production: (crashes)
10. Everyone: (blames each other)

Problems:
├── Time-consuming (hours)
├── Error-prone (manual steps)
├── Inconsistent (different each time)
├── Undocumented (tribal knowledge)
├── Scary (nobody wants to do it)
└── Infrequent (because it's scary)
```

### 1.4 The Fear of Deployment

```
                                      ┌─────────────────────┐
                                      │ "Should we deploy?" │
                                      └──────────┬──────────┘
                                                 │
                              ┌──────────────────┼──────────────────┐
                              │                  │                  │
                              ▼                  ▼                  ▼
                     ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
                     │    "It's      │  │   "Let's      │  │    "We need   │
                     │    Friday"    │  │    wait for   │  │    more       │
                     │               │  │    Monday"    │  │    testing"   │
                     └───────┬───────┘  └───────┬───────┘  └───────┬───────┘
                             │                  │                  │
                             └──────────────────┼──────────────────┘
                                                │
                                                ▼
                                       ┌───────────────┐
                                       │  Deployment   │
                                       │   delayed     │
                                       │   (again)     │
                                       └───────────────┘

When deployments are painful, you deploy less often.
When you deploy less often, each deployment is bigger.
When deployments are bigger, they're riskier.
When they're riskier, they're more painful.

The vicious cycle.
```

---

## Section 2: CI/CD Definitions

### 2.1 Continuous Integration (CI)

**Definition:**
The practice of frequently merging code changes into a shared repository, with automated builds and tests to verify each integration.

**Key Elements:**
```
┌─────────────────────────────────────────────────────────────────┐
│                     Continuous Integration                       │
└─────────────────────────────────────────────────────────────────┘

Developer commits code
         │
         ▼
┌─────────────────┐
│  Code merged to │
│  shared branch  │
│  (multiple      │
│  times/day)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Automated      │
│  build runs     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Automated      │
│  tests run      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Build status   │
│  reported       │
│  (pass/fail)    │
└─────────────────┘

CI Promise: You know within minutes if your changes broke something.
```

### 2.2 Continuous Delivery (CD)

**Definition:**
The practice of keeping code always ready to deploy to production. Any version that passes testing can be deployed with the push of a button.

**Continuous Delivery Pipeline:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    Continuous Delivery                           │
└─────────────────────────────────────────────────────────────────┘

Commit ─► Build ─► Test ─► Stage ─► [Manual Approval] ─► Production
                                            │
                                            │
                                    Human decides WHEN
                                    to deploy

CD Promise: Code is ALWAYS deployable.
            Deployment is a business decision, not a technical one.
```

### 2.3 Continuous Deployment

**Definition:**
Every change that passes automated testing is automatically deployed to production. No manual approval required.

**Continuous Deployment Pipeline:**
```
┌─────────────────────────────────────────────────────────────────┐
│                   Continuous Deployment                          │
└─────────────────────────────────────────────────────────────────┘

Commit ─► Build ─► Test ─► Stage ─► Production
                               │
                               │
                       All automated
                       No human gate

CD (Deployment) Promise: Every good commit goes to production.
                        Deploy 10, 50, or 100+ times per day.
```

### 2.4 The Relationship

```
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │                                                     │  │  │
│  │  │              Continuous Integration                 │  │  │
│  │  │         (Build and test on every commit)           │  │  │
│  │  │                                                     │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                                                           │  │
│  │                 Continuous Delivery                       │  │
│  │          (Always releasable, manual deploy)              │  │
│  │                                                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│                    Continuous Deployment                        │
│              (Automatic deploy on every commit)                │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

Each level builds on the previous.
Most teams do CI + Continuous Delivery.
Continuous Deployment requires significant maturity.
```

---

## Section 3: Pipeline Design

### 3.1 The Basic Pipeline Structure

**Typical Stages:**
```
┌───────────────────────────────────────────────────────────────────┐
│                        CI/CD Pipeline                              │
└───────────────────────────────────────────────────────────────────┘

┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
│  Code   │──►│  Build  │──►│  Test   │──►│ Package │──►│ Deploy  │
│         │   │         │   │         │   │         │   │         │
└─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
     │             │             │             │             │
     │             │             │             │             │
     ▼             ▼             ▼             ▼             ▼
  Commit       Compile        Unit         Docker       Staging
  Push         Check         Tests         Image        Prod
               deps          Int Tests     Artifact
                             Security
```

### 3.2 Stage: Build

**Purpose:** Compile code and verify it can be built.

```yaml
build:
  stage: build
  steps:
    - checkout code
    - restore dependencies (cached)
    - compile
    - check for warnings
    - archive artifacts

# Example Gradle
./gradlew clean build -x test

# Example Maven
mvn clean compile

Time: 1-5 minutes
```

**What Should Fail the Build:**
```
├── Compilation errors
├── Missing dependencies
├── Code style violations (optional)
└── Static analysis warnings (optional)
```

### 3.3 Stage: Test

**Testing Pyramid:**
```
                    ┌───────────────┐
                   /                 \
                  /    End-to-End    \     Slow, expensive
                 /       Tests        \    Few tests
                /─────────────────────\
               /                       \
              /    Integration Tests    \   Medium speed
             /                           \  Medium count
            /─────────────────────────────\
           /                               \
          /         Unit Tests              \  Fast, cheap
         /                                   \ Many tests
        /─────────────────────────────────────\

Run in order: Unit → Integration → E2E
Fail fast: Stop at first failure
```

**Test Stage Implementation:**
```yaml
test:
  stage: test
  parallel:
    - unit-tests:
        run: ./gradlew test
        timeout: 5m
    - integration-tests:
        run: ./gradlew integrationTest
        timeout: 15m
        services:
          - postgres:15
          - redis:7
    - security-scan:
        run: ./security-scan.sh
        timeout: 10m
```

### 3.4 Stage: Package

**Purpose:** Create deployable artifacts.

```yaml
package:
  stage: package
  steps:
    - build docker image
    - tag with version/commit
    - push to registry
    - create helm chart (if k8s)
    - sign artifacts

# Example
docker build -t myapp:${GIT_SHA} .
docker push registry.example.com/myapp:${GIT_SHA}
```

**Artifact Versioning:**
```
Good versioning:
├── myapp:1.2.3              (semantic version)
├── myapp:1.2.3-abc123       (version + commit)
├── myapp:abc123             (commit SHA)
└── myapp:main-20240115      (branch + date)

Bad versioning:
├── myapp:latest             (which version is this?)
└── myapp:new                (meaningless)
```

### 3.5 Stage: Deploy

**Environment Progression:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    Deployment Progression                        │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│     Dev      │───►│   Staging    │───►│  Production  │
│              │    │              │    │              │
│ Auto-deploy  │    │ Auto-deploy  │    │ Manual/Auto  │
│ on commit    │    │ on test pass │    │ (depends)    │
│              │    │              │    │              │
│ Smoke tests  │    │ Full tests   │    │ Canary       │
│              │    │ Performance  │    │ Monitoring   │
└──────────────┘    └──────────────┘    └──────────────┘
```

---

## Section 4: Testing Strategies

### 4.1 Unit Tests in Pipelines

**Characteristics:**
```
├── Fast (milliseconds per test)
├── No external dependencies
├── Run in isolation
├── High coverage goal (80%+)
└── Run on every commit
```

**Example Configuration:**
```yaml
unit-tests:
  script: ./gradlew test
  coverage:
    minimum: 80%
    report: build/reports/jacoco/
  cache:
    - .gradle/
  timeout: 5 minutes
```

### 4.2 Integration Tests

**Characteristics:**
```
├── Test component interactions
├── Use real(ish) dependencies
├── Slower than unit tests
├── Fewer than unit tests
└── Run on every commit (if fast enough)
```

**Patterns:**
```
┌─────────────────────────────────────────────────────────────────┐
│                 Integration Test Approaches                      │
└─────────────────────────────────────────────────────────────────┘

1. Embedded/In-Memory:
   ├── H2 instead of Postgres
   ├── Embedded Kafka
   └── Fast but not exactly like prod

2. Containers (Testcontainers):
   ├── Real Postgres in Docker
   ├── Real Redis, Kafka, etc.
   └── Slower but more realistic

3. Shared Test Environment:
   ├── Pre-provisioned services
   ├── Faster startup
   └── Risk of test interference
```

**Testcontainers Example:**
```java
@Testcontainers
class OrderServiceIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15");

    @Container
    static GenericContainer<?> redis =
        new GenericContainer<>("redis:7").withExposedPorts(6379);

    @Test
    void shouldCreateOrder() {
        // Test with real database and cache
    }
}
```

### 4.3 End-to-End Tests

**Characteristics:**
```
├── Test full user journeys
├── Use real UI and APIs
├── Slowest test type
├── Most brittle
├── Run selectively
└── Often run post-deploy
```

**Pipeline Placement:**
```
Option A: Before production deploy
├── Longer pipeline
├── More confidence
└── Slower feedback

Option B: After staging deploy (smoke tests)
├── Faster pipeline
├── Test real deployment
└── Catch deployment issues

Option C: Parallel to deploy (monitoring)
├── Fastest pipeline
├── Catch issues quickly
└── Requires good rollback
```

### 4.4 Security Scanning

**Types of Security Scans:**
```
┌─────────────────────────────────────────────────────────────────┐
│                     Security Scanning                            │
└─────────────────────────────────────────────────────────────────┘

SAST (Static Application Security Testing):
├── Analyze source code
├── Find vulnerabilities in code
├── Run during build
└── Tools: SonarQube, Checkmarx

SCA (Software Composition Analysis):
├── Scan dependencies
├── Find vulnerable libraries
├── Check licenses
└── Tools: Snyk, Dependabot, OWASP Dependency-Check

DAST (Dynamic Application Security Testing):
├── Test running application
├── Find runtime vulnerabilities
├── Run against staging
└── Tools: OWASP ZAP, Burp Suite

Container Scanning:
├── Scan Docker images
├── Find OS vulnerabilities
├── Run before deploy
└── Tools: Trivy, Clair, Anchore
```

**Integration:**
```yaml
security-stage:
  parallel:
    - sast:
        run: sonar-scanner
        fail-on: high,critical
    - dependency-scan:
        run: snyk test
        fail-on: high
    - container-scan:
        run: trivy image myapp:${VERSION}
        fail-on: critical
```

---

## Section 5: Deployment Strategies

### 5.1 Rolling Deployment

**How It Works:**
```
Start:
┌─────────────────────────────────────────────────────────────────┐
│  Load Balancer                                                   │
│       │                                                          │
│   ┌───┴───┬───────┬───────┬───────┐                             │
│   ▼       ▼       ▼       ▼       │                             │
│ ┌───┐   ┌───┐   ┌───┐   ┌───┐    │                             │
│ │v1 │   │v1 │   │v1 │   │v1 │    │  4 instances v1             │
│ └───┘   └───┘   └───┘   └───┘    │                             │
└─────────────────────────────────────────────────────────────────┘

Step 1: Replace one instance
┌─────────────────────────────────────────────────────────────────┐
│   ┌───┐   ┌───┐   ┌───┐   ┌───┐                                │
│   │v2 │   │v1 │   │v1 │   │v1 │   1 v2, 3 v1                   │
│   └───┘   └───┘   └───┘   └───┘                                │
└─────────────────────────────────────────────────────────────────┘

Step 2-3: Continue rolling
┌─────────────────────────────────────────────────────────────────┐
│   ┌───┐   ┌───┐   ┌───┐   ┌───┐                                │
│   │v2 │   │v2 │   │v2 │   │v1 │   3 v2, 1 v1                   │
│   └───┘   └───┘   └───┘   └───┘                                │
└─────────────────────────────────────────────────────────────────┘

Complete:
┌─────────────────────────────────────────────────────────────────┐
│   ┌───┐   ┌───┐   ┌───┐   ┌───┐                                │
│   │v2 │   │v2 │   │v2 │   │v2 │   4 v2                         │
│   └───┘   └───┘   └───┘   └───┘                                │
└─────────────────────────────────────────────────────────────────┘
```

**Pros/Cons:**
```
Pros:                           Cons:
├── No extra infrastructure     ├── Multiple versions running
├── Gradual transition          ├── Must be backward compatible
├── Can stop/rollback mid-way   ├── Longer deployment time
└── Resource efficient          └── Testing multiple versions
```

### 5.2 Blue-Green Deployment

**How It Works:**
```
Before:
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Load Balancer ─────────────────────► Blue Environment (v1)     │
│       │                               ┌───┐ ┌───┐ ┌───┐         │
│       │                               │v1 │ │v1 │ │v1 │         │
│       │                               └───┘ └───┘ └───┘         │
│       │                                                          │
│       │                               Green Environment (idle)  │
│       └─────────── X ─────────────►   ┌───┐ ┌───┐ ┌───┐         │
│                                       │   │ │   │ │   │         │
│                                       └───┘ └───┘ └───┘         │
└─────────────────────────────────────────────────────────────────┘

Deploy v2 to Green:
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Load Balancer ─────────────────────► Blue Environment (v1)     │
│       │                               ┌───┐ ┌───┐ ┌───┐         │
│       │                               │v1 │ │v1 │ │v1 │         │
│       │                               └───┘ └───┘ └───┘         │
│       │                                                          │
│       │                               Green Environment (v2)    │
│       └─────────── X ─────────────►   ┌───┐ ┌───┐ ┌───┐         │
│                 (test here)           │v2 │ │v2 │ │v2 │         │
│                                       └───┘ └───┘ └───┘         │
└─────────────────────────────────────────────────────────────────┘

Switch traffic:
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Load Balancer ──────── X ────────►   Blue Environment (v1)     │
│       │                               ┌───┐ ┌───┐ ┌───┐         │
│       │                               │v1 │ │v1 │ │v1 │         │
│       │             (standby for      └───┘ └───┘ └───┘         │
│       │              rollback)                                   │
│       │                               Green Environment (v2)    │
│       └─────────────────────────────► ┌───┐ ┌───┐ ┌───┐         │
│                                       │v2 │ │v2 │ │v2 │         │
│                                       └───┘ └───┘ └───┘         │
└─────────────────────────────────────────────────────────────────┘
```

**Pros/Cons:**
```
Pros:                           Cons:
├── Instant rollback            ├── Double infrastructure cost
├── Zero downtime               ├── Database migrations complex
├── Test before switch          ├── Stateful apps challenging
└── Clean environment           └── Not always cost-effective
```

### 5.3 Canary Deployment

**How It Works:**
```
Step 1: Deploy to small canary (1-5%)
┌─────────────────────────────────────────────────────────────────┐
│  Traffic: 95% ────────────────────────────────────►  v1 (95%)   │
│                                                      ┌───┐      │
│  Traffic: 5%  ─────────────────────────────────────► │v2 │      │
│                                                      └───┘      │
│                                                    Canary (5%)  │
└─────────────────────────────────────────────────────────────────┘

Step 2: Monitor metrics (errors, latency, etc.)
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Error Rate:  v1: 0.1%    v2: 0.2%  ← Watching closely          │
│  Latency:     v1: 50ms    v2: 55ms  ← Acceptable?               │
│  CPU:         v1: 30%     v2: 35%   ← Within bounds             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Step 3a: If good, gradually increase
┌─────────────────────────────────────────────────────────────────┐
│  5% ──► 10% ──► 25% ──► 50% ──► 100%                           │
│                                                                  │
│  At each step, verify metrics                                   │
└─────────────────────────────────────────────────────────────────┘

Step 3b: If bad, rollback canary
┌─────────────────────────────────────────────────────────────────┐
│  Kill canary, 100% back to v1                                   │
│  Only 5% of users affected briefly                              │
└─────────────────────────────────────────────────────────────────┘
```

**Pros/Cons:**
```
Pros:                           Cons:
├── Limited blast radius        ├── Complex monitoring needed
├── Real production testing     ├── Longer deployment time
├── Data-driven decisions       ├── Multiple versions running
└── Catch issues early          └── Requires traffic splitting
```

### 5.4 Feature Flags

**Pattern: Decouple Deploy from Release**
```
Traditional:
Deploy code = Release feature

With Feature Flags:
Deploy code ≠ Release feature

┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  if (featureFlags.isEnabled("new-checkout")) {                  │
│      return newCheckoutFlow();                                  │
│  } else {                                                        │
│      return oldCheckoutFlow();                                  │
│  }                                                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Benefits:
├── Deploy anytime (code is disabled)
├── Enable for % of users (canary)
├── Enable for specific users (beta)
├── Instant disable if problems
└── A/B testing built-in
```

---

## Section 6: Pipeline as Code

### 6.1 Why Pipeline as Code?

**Benefits:**
```
├── Version controlled (history, review)
├── Reproducible (same config = same pipeline)
├── Portable (move between projects)
├── Self-documenting (code is documentation)
└── Testable (can validate pipeline logic)
```

### 6.2 GitHub Actions Example

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Build
        run: ./gradlew build -x test

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar

  test:
    needs: build
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'gradle'

      - name: Run tests
        run: ./gradlew test
        env:
          DATABASE_URL: postgresql://localhost:5432/test

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: build/reports/tests/

  security-scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Snyk
        uses: snyk/actions/gradle@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  deploy-staging:
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to staging
        run: |
          ./deploy.sh staging
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to production
        run: |
          ./deploy.sh production
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### 6.3 GitLab CI Example

```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - security
  - deploy

variables:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false"

build:
  stage: build
  image: gradle:8-jdk17
  script:
    - gradle build -x test
  artifacts:
    paths:
      - build/libs/*.jar
    expire_in: 1 hour
  cache:
    key: gradle
    paths:
      - .gradle/

unit-test:
  stage: test
  image: gradle:8-jdk17
  script:
    - gradle test
  artifacts:
    reports:
      junit: build/test-results/test/*.xml
  cache:
    key: gradle
    paths:
      - .gradle/

integration-test:
  stage: test
  image: gradle:8-jdk17
  services:
    - postgres:15
  variables:
    POSTGRES_PASSWORD: test
    DATABASE_URL: postgresql://postgres:5432/test
  script:
    - gradle integrationTest
  cache:
    key: gradle
    paths:
      - .gradle/

sast:
  stage: security
  image: sonarsource/sonar-scanner-cli
  script:
    - sonar-scanner

dependency-scan:
  stage: security
  image: snyk/snyk:gradle
  script:
    - snyk test

deploy-staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - ./deploy.sh staging
  only:
    - main

deploy-production:
  stage: deploy
  environment:
    name: production
    url: https://example.com
  script:
    - ./deploy.sh production
  when: manual
  only:
    - main
```

---

## Section 7: Best Practices

### 7.1 Pipeline Performance

**Keep Pipelines Fast:**
```
Target times:
├── Build: < 5 minutes
├── Unit tests: < 5 minutes
├── Integration tests: < 15 minutes
├── Full pipeline: < 30 minutes
└── Deploy: < 10 minutes

Techniques:
├── Parallelize stages
├── Cache dependencies
├── Only run what changed
├── Use faster runners
└── Split large test suites
```

**Caching:**
```yaml
# Cache dependencies
cache:
  key: $CI_COMMIT_REF_SLUG
  paths:
    - .gradle/
    - ~/.m2/
    - node_modules/

# Reuse built artifacts
artifacts:
  paths:
    - build/
  expire_in: 1 hour
```

### 7.2 Fail Fast

**Order Matters:**
```
Good order:
1. Lint/Format check (seconds)
2. Compile (1-2 min)
3. Unit tests (2-5 min)
4. Integration tests (5-15 min)
5. E2E tests (10-30 min)
6. Deploy

Fail at step 1 = Feedback in < 1 minute
Fail at step 5 = Feedback in 30+ minutes

Run cheap/fast checks first!
```

### 7.3 Trunk-Based Development Support

**Short-Lived Branches:**
```
┌─────────────────────────────────────────────────────────────────┐
│               Pipeline for Trunk-Based Development               │
└─────────────────────────────────────────────────────────────────┘

PR Pipeline (fast):
├── Build
├── Unit tests
├── Lint/Security (parallel)
└── Time: < 10 minutes

Main Branch Pipeline (complete):
├── Build
├── All tests
├── Security scans
├── Deploy to staging
├── Smoke tests
└── Deploy to production (if passing)
```

### 7.4 Secrets Management

**Never Do:**
```yaml
# BAD: Secrets in pipeline code
env:
  API_KEY: "sk-abc123secretkey"
  DB_PASSWORD: "supersecret"
```

**Do Instead:**
```yaml
# GOOD: Secrets from secure storage
env:
  API_KEY: ${{ secrets.API_KEY }}
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

# Configure in:
# - GitHub Secrets
# - GitLab CI Variables
# - AWS Secrets Manager
# - HashiCorp Vault
```

---

## Section 8: Common Anti-Patterns

### 8.1 The "It Worked Locally" Pipeline

**Anti-Pattern:**
```
Pipeline uses different:
├── Versions of tools
├── Operating systems
├── Configurations
├── Dependencies
└── Than local development

Result: "But it works on my machine!"
```

**Solution:**
```
├── Docker for consistent environments
├── Same build tool versions
├── Pin dependency versions
├── Document local setup
└── Test locally with same commands
```

### 8.2 The "Manual Steps" Pipeline

**Anti-Pattern:**
```
Pipeline:
1. Build ✓ (automated)
2. Test ✓ (automated)
3. "SSH to server and run deploy.sh" ← Manual!
4. "Update the config file" ← Manual!
5. "Restart the service" ← Manual!
```

**Solution:**
Automate everything. If there's a manual step, it will be:
- Skipped
- Done wrong
- Forgotten

### 8.3 The "1000 Line" Pipeline

**Anti-Pattern:**
```yaml
# ci.yml - 1000 lines of copy-pasted steps
build-service-a:
  script:
    - # 50 lines
build-service-b:
  script:
    - # same 50 lines, slightly different
# ... repeated 20 times
```

**Solution:**
```yaml
# Use templates/includes
.build-template: &build
  script:
    - ./gradlew build
  artifacts:
    paths:
      - build/

build-service-a:
  <<: *build
  variables:
    SERVICE: service-a

build-service-b:
  <<: *build
  variables:
    SERVICE: service-b
```

### 8.4 The "Flaky Test" Pipeline

**Anti-Pattern:**
```
Pipeline results:
├── Monday: ✓ Pass
├── Tuesday: ✗ Fail (test_user_login)
├── Wednesday: ✓ Pass
├── Thursday: ✗ Fail (test_user_login)
└── Team: "Just re-run it..."
```

**Solution:**
```
├── Fix flaky tests immediately
├── Quarantine until fixed
├── Add retry logic for infrastructure issues
├── Track flakiness metrics
└── Don't let flaky become normal
```

---

## Summary: CI/CD Pipeline Thinking

**Why CI/CD:**
```
├── Catch problems early
├── Consistent, reproducible builds
├── Fast feedback on changes
├── Safe, frequent deployments
└── Reduced manual work and errors
```

**Key Concepts:**
```
CI: Integrate code frequently, verify with automated builds/tests
CD (Delivery): Always keep code deployable
CD (Deployment): Automatically deploy every passing change
```

**Pipeline Design:**
```
├── Build → Test → Package → Deploy
├── Fail fast (quick checks first)
├── Parallelize where possible
├── Cache for speed
└── Pipeline as code
```

**Deployment Strategies:**
```
├── Rolling: Gradual replacement
├── Blue-Green: Instant switch
├── Canary: Progressive rollout
└── Feature Flags: Decouple deploy from release
```

**Best Practices:**
```
├── Keep pipelines fast (< 30 min)
├── Automate everything
├── Secure secrets properly
├── Fix flaky tests immediately
└── Treat pipeline code like app code
```

---

## What's Next?

CI/CD gets your code to production. But how do you know it's working? How do you find problems before users do? Chapter 21 explores observability—the practice of understanding what your system is doing through metrics, logs, and traces.
