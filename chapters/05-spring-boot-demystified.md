# Chapter 5: Spring Boot Demystified

> **First Principles Question**: Why does adding `@RestController` to a class magically make it handle HTTP requests? What's actually happening under the hood?

---

## Chapter Overview

Spring Boot is so prevalent in Java development that many developers use it without understanding it. This chapter demystifies Spring—not just how to use it, but why it exists and how it actually works.

**What readers will understand after this chapter:**
- What problem dependency injection solves
- How Spring's "magic" actually works (no magic—just patterns)
- What auto-configuration does and why
- How to think in Spring idioms
- When Spring is overkill and alternatives to consider

---

## Section 1: The Problem Before Spring

### 1.1 The Dependency Nightmare

**A Simple Service:**
```java
public class OrderService {
    public void createOrder(Order order) {
        // Need to validate the order
        // Need to check inventory
        // Need to process payment
        // Need to send notification
        // Need to save to database
    }
}
```

**Without Dependency Injection:**
```java
public class OrderService {
    public void createOrder(Order order) {
        // Create all dependencies manually
        DataSource ds = new MySQLDataSource("jdbc:mysql://localhost/db");
        OrderRepository repo = new OrderRepository(ds);
        InventoryService inventory = new InventoryService(
            new InventoryRepository(ds)
        );
        PaymentService payment = new PaymentService(
            new StripeGateway("sk_live_xxx")
        );
        EmailService email = new EmailService(
            new SMTPClient("smtp.example.com", 587)
        );

        // Finally do the work
        if (inventory.check(order.getItems())) {
            payment.charge(order.getTotal());
            repo.save(order);
            email.sendConfirmation(order);
        }
    }
}
```

### 1.2 The Problems

**1. Hard-coded Dependencies:**
Every class creates its own dependencies. Changing from MySQL to PostgreSQL means changing code in dozens of places.

**2. No Testability:**
How do you test OrderService without a real database, Stripe account, and email server?

**3. Configuration Scattered:**
Database URLs, API keys, server addresses—all embedded in code.

**4. Object Lifecycle Chaos:**
Who creates what? When? Who cleans up? Creating a new DataSource for every operation wastes resources.

### 1.3 The Manual Solution

**Dependency Injection by Hand:**
```java
public class OrderService {
    private final OrderRepository repo;
    private final InventoryService inventory;
    private final PaymentService payment;
    private final EmailService email;

    // Dependencies injected through constructor
    public OrderService(OrderRepository repo,
                        InventoryService inventory,
                        PaymentService payment,
                        EmailService email) {
        this.repo = repo;
        this.inventory = inventory;
        this.payment = payment;
        this.email = email;
    }

    public void createOrder(Order order) {
        // Use injected dependencies
        if (inventory.check(order.getItems())) {
            payment.charge(order.getTotal());
            repo.save(order);
            email.sendConfirmation(order);
        }
    }
}
```

**But Now Someone Has to Wire It:**
```java
public class Application {
    public static void main(String[] args) {
        // Manual wiring
        DataSource ds = new MySQLDataSource("jdbc:mysql://localhost/db");
        OrderRepository orderRepo = new OrderRepository(ds);
        InventoryRepository invRepo = new InventoryRepository(ds);
        InventoryService inventory = new InventoryService(invRepo);
        PaymentService payment = new PaymentService(new StripeGateway("sk_live_xxx"));
        EmailService email = new EmailService(new SMTPClient("smtp.example.com", 587));

        OrderService orderService = new OrderService(orderRepo, inventory, payment, email);

        // Finally use it
        orderService.createOrder(new Order(...));
    }
}
```

**This Wiring Code:**
- Gets massive in real applications
- Must be maintained as dependencies change
- Is boilerplate that adds no business value

---

## Section 2: Inversion of Control — The Conceptual Foundation

### 2.1 The Hollywood Principle

**"Don't call us, we'll call you."**

**Traditional Programming:**
Your code controls everything. You create objects, call methods, manage lifecycle.

**Inversion of Control:**
A framework controls flow. It creates objects and calls your code when appropriate.

### 2.2 How IoC Manifests

**Without IoC:**
```java
public class MyApp {
    public static void main(String[] args) {
        // I control everything
        ServerSocket server = new ServerSocket(8080);
        while (true) {
            Socket client = server.accept();
            // I read request, I route, I respond
        }
    }
}
```

**With IoC (Spring):**
```java
@RestController
public class MyController {
    @GetMapping("/hello")
    public String hello() {
        // Framework calls this when request matches
        return "Hello!";
    }
}
```

**The Inversion:**
You don't start the server. You don't accept connections. You don't parse HTTP. You just say "when someone requests /hello, call this method." The framework handles the rest.

### 2.3 Dependency Injection as IoC

**Without DI:**
Your code creates dependencies (you control).

**With DI:**
Framework creates dependencies and gives them to you (framework controls).

```java
@Service
public class OrderService {
    private final OrderRepository repo;  // Spring provides this

    public OrderService(OrderRepository repo) {
        this.repo = repo;  // Spring calls this constructor with proper instance
    }
}
```

---

## Section 3: The Spring Container — Where the Magic Happens

### 3.1 The ApplicationContext

**What It Is:**
A container that:
1. Knows about all your components ("beans")
2. Creates them in the right order
3. Wires dependencies together
4. Manages their lifecycle

**The Container Model:**
```
┌─────────────────────────────────────────────────────────────┐
│                   ApplicationContext                         │
│                                                             │
│   ┌─────────┐    ┌─────────────┐    ┌─────────────┐        │
│   │ Service │    │ Repository  │    │ Controller  │        │
│   │  Bean   │←───│    Bean     │←───│    Bean     │        │
│   └─────────┘    └─────────────┘    └─────────────┘        │
│        ↓                                                    │
│   ┌─────────────┐                                          │
│   │  DataSource │                                          │
│   │    Bean     │                                          │
│   └─────────────┘                                          │
│                                                             │
│   Spring creates, wires, and manages all of these          │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Bean Definition

**What's a Bean?**
An object managed by Spring. Just a regular Java object, but Spring creates and manages it.

**Defining Beans:**

**Via Annotations:**
```java
@Service
public class UserService { }

@Repository
public class UserRepository { }

@Controller
public class UserController { }
```

**Via Configuration:**
```java
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(config);
    }
}
```

### 3.3 Component Scanning

**How Spring Finds Beans:**
```java
@SpringBootApplication  // Triggers scanning
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

**What Happens:**
1. Spring scans packages starting from MyApp's package
2. Finds classes with @Component, @Service, @Repository, @Controller
3. Creates bean definitions for each
4. Instantiates them in dependency order

### 3.4 Dependency Resolution

**Spring's Algorithm:**
```
1. Collect all bean definitions
2. For each bean:
   a. Check constructor parameters
   b. Find beans matching those types
   c. Create dependencies first (recursive)
   d. Call constructor with dependencies
3. Handle circular dependencies (or fail)
```

**Example:**
```java
@Service
public class OrderService {
    public OrderService(OrderRepository repo, PaymentService payment) {
        // Spring finds OrderRepository bean
        // Spring finds PaymentService bean
        // Creates both (and their dependencies)
        // Then creates OrderService
    }
}
```

---

## Section 4: Understanding Annotations — Not Magic, Just Metadata

### 4.1 What Annotations Are

**Just Metadata:**
Annotations are markers that add information to code. They don't DO anything by themselves.

```java
@Service  // Just a marker
public class UserService { }
```

**The annotation itself:**
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component  // @Service IS a @Component
public @interface Service {
    String value() default "";
}
```

### 4.2 How Spring Uses Annotations

**At Startup:**
1. Classpath scanning finds classes
2. Reflection reads annotations
3. Spring processes based on annotation type

**Example Processing:**
```java
// Pseudocode of what Spring does
for (Class<?> clazz : scannedClasses) {
    if (clazz.isAnnotationPresent(Component.class)) {
        BeanDefinition def = new BeanDefinition(clazz);
        beanFactory.registerBeanDefinition(def);
    }
}
```

### 4.3 Common Spring Annotations Decoded

**@Component Family:**
```java
@Component    // Generic managed bean
@Service      // Business logic layer
@Repository   // Data access layer (adds exception translation)
@Controller   // Web layer (MVC)
@RestController  // @Controller + @ResponseBody
```

**@Autowired:**
```java
@Autowired
private UserRepository repo;  // Spring injects matching bean
```

**@Value:**
```java
@Value("${app.name}")
private String appName;  // Injects from properties file
```

**@Qualifier:**
```java
@Autowired
@Qualifier("primaryDataSource")  // Which one when multiple match
private DataSource ds;
```

---

## Section 5: Auto-Configuration — Spring Boot's Real Innovation

### 5.1 The Problem Spring Boot Solved

**Old Spring:**
```xml
<!-- 50+ lines of XML configuration -->
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost/mydb"/>
    <!-- ... more properties ... -->
</bean>

<bean id="sessionFactory"
      class="org.springframework.orm.hibernate4.LocalSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <!-- ... more properties ... -->
</bean>

<!-- More and more boilerplate... -->
```

### 5.2 Convention Over Configuration

**Spring Boot Philosophy:**
If you have the MySQL driver on classpath and spring.datasource.url in properties, you probably want a DataSource. Let's create one automatically.

**Spring Boot:**
```properties
# application.properties
spring.datasource.url=jdbc:mysql://localhost/mydb
spring.datasource.username=root
spring.datasource.password=secret
```

**That's it.** Spring Boot auto-configures:
- DataSource
- Connection pool (HikariCP)
- Transaction manager
- JdbcTemplate

### 5.3 How Auto-Configuration Works

**The @Conditional Annotations:**
```java
@Configuration
@ConditionalOnClass(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.url")
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean  // Only if user hasn't defined one
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

**The Logic:**
- `@ConditionalOnClass`: Only activate if class is present
- `@ConditionalOnProperty`: Only if property is set
- `@ConditionalOnMissingBean`: Only if user hasn't defined their own

### 5.4 The spring.factories File

**Discovery Mechanism:**
Spring Boot finds auto-configurations via META-INF/spring.factories:

```properties
# In spring-boot-autoconfigure.jar
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
  org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration,\
  ...hundreds more...
```

**At Startup:**
1. Load all auto-configuration classes
2. Evaluate conditions
3. Apply those that match

### 5.5 Examining Auto-Configuration

**Debug What's Happening:**
```bash
java -jar myapp.jar --debug
```

**Output Shows:**
```
Positive matches:
-----------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required class 'javax.sql.DataSource'

Negative matches:
-----------------
   RabbitAutoConfiguration:
      - @ConditionalOnClass did not find required class 'RabbitTemplate'
```

---

## Section 6: Web Applications with Spring MVC

### 6.1 The DispatcherServlet

**The Front Controller:**
```
                                    ┌─────────────────────────┐
    HTTP Request                    │    DispatcherServlet    │
         │                          │                         │
         ▼                          │   1. Receive request    │
    ┌─────────┐                     │   2. Find handler       │
    │  Tomcat │ ──────────────────→ │   3. Execute handler    │
    └─────────┘                     │   4. Render response    │
                                    └────────────┬────────────┘
                                                 │
                    ┌────────────────────────────┼────────────────────────┐
                    ▼                            ▼                        ▼
            ┌─────────────┐            ┌─────────────┐          ┌─────────────┐
            │ Controller  │            │ Controller  │          │ Controller  │
            │   /users    │            │  /orders    │          │  /products  │
            └─────────────┘            └─────────────┘          └─────────────┘
```

### 6.2 Request Mapping

**How URLs Map to Methods:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping                     // GET /api/users
    public List<User> list() { }

    @GetMapping("/{id}")            // GET /api/users/123
    public User get(@PathVariable Long id) { }

    @PostMapping                    // POST /api/users
    public User create(@RequestBody User user) { }

    @PutMapping("/{id}")            // PUT /api/users/123
    public User update(@PathVariable Long id, @RequestBody User user) { }

    @DeleteMapping("/{id}")         // DELETE /api/users/123
    public void delete(@PathVariable Long id) { }
}
```

### 6.3 Request Processing Pipeline

**The Full Flow:**
```
1. Request arrives: POST /api/users

2. HandlerMapping finds:
   UserController.create() matches POST /api/users

3. Argument resolution:
   - @RequestBody → Read JSON body, deserialize to User
   - @PathVariable → Extract from URL
   - @RequestParam → Extract query parameters

4. Method execution:
   User result = userController.create(user);

5. Response handling:
   - @ResponseBody → Serialize result to JSON
   - Set Content-Type: application/json
   - Return 200 OK (or appropriate status)
```

### 6.4 Exception Handling

**Global Error Handling:**
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(UserNotFoundException ex) {
        return new ErrorResponse("User not found", ex.getMessage());
    }

    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(ValidationException ex) {
        return new ErrorResponse("Validation failed", ex.getErrors());
    }
}
```

---

## Section 7: Data Access with Spring Data

### 7.1 The Repository Pattern

**Before Spring Data:**
```java
public class UserRepository {
    private final JdbcTemplate jdbc;

    public User findById(Long id) {
        return jdbc.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new Object[]{id},
            (rs, rowNum) -> new User(
                rs.getLong("id"),
                rs.getString("name"),
                rs.getString("email")
            )
        );
    }

    public List<User> findByEmail(String email) {
        return jdbc.query(
            "SELECT * FROM users WHERE email = ?",
            new Object[]{email},
            (rs, rowNum) -> new User(...)
        );
    }

    // ... many more boilerplate methods
}
```

### 7.2 Spring Data Magic

**Just Define an Interface:**
```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Spring generates: SELECT * FROM users WHERE email = ?
    List<User> findByEmail(String email);

    // Spring generates: SELECT * FROM users WHERE name LIKE ?
    List<User> findByNameContaining(String name);

    // Spring generates: SELECT * FROM users WHERE active = true ORDER BY created_at DESC
    List<User> findByActiveTrueOrderByCreatedAtDesc();

    // Custom query when method names get unwieldy
    @Query("SELECT u FROM User u WHERE u.department.id = :deptId AND u.role = :role")
    List<User> findByDepartmentAndRole(@Param("deptId") Long deptId, @Param("role") String role);
}
```

### 7.3 How Spring Data Works

**The Proxy Pattern:**
```
┌─────────────────────────────────────────────────────────────┐
│                You define interface                         │
│    public interface UserRepository                          │
│        extends JpaRepository<User, Long> { }                │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│        Spring creates implementation at runtime             │
│                                                             │
│    public class UserRepositoryImpl implements UserRepo {    │
│        // Generated code for all methods                    │
│        public User findById(Long id) {                      │
│            return em.find(User.class, id);                  │
│        }                                                    │
│        public List<User> findByEmail(String email) {        │
│            return em.createQuery(...)                       │
│        }                                                    │
│    }                                                        │
└─────────────────────────────────────────────────────────────┘
```

**Method Name Parsing:**
```
findByNameContainingAndActiveTrue
│     │    │          │     │
│     │    │          │     └─ = true
│     │    │          └─ AND active
│     │    └─ LIKE '%...%'
│     └─ WHERE name
└─ SELECT
```

---

## Section 8: Configuration in Spring Boot

### 8.1 Configuration Hierarchy

**Priority Order (highest to lowest):**
1. Command line arguments
2. SPRING_APPLICATION_JSON
3. application.properties in config/
4. application.properties in classpath root
5. @PropertySource annotations
6. Default properties

### 8.2 Profile-Based Configuration

**Different Configs for Different Environments:**
```
application.properties          # Common settings
application-dev.properties      # Development overrides
application-prod.properties     # Production overrides
application-test.properties     # Test overrides
```

**Activate Profile:**
```bash
java -jar myapp.jar --spring.profiles.active=prod
```

**Profile-Specific Beans:**
```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        // In-memory database for dev
        return new EmbeddedDatabaseBuilder().build();
    }
}

@Configuration
@Profile("prod")
public class ProdConfig {
    @Bean
    public DataSource dataSource() {
        // Real database for prod
        return createProductionDataSource();
    }
}
```

### 8.3 Type-Safe Configuration

**Binding Properties to Objects:**
```java
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {
    private String host;
    private int port;
    private String username;
    private String password;

    // Getters and setters
}
```

```properties
app.mail.host=smtp.example.com
app.mail.port=587
app.mail.username=sender@example.com
app.mail.password=secret
```

```java
@Service
public class MailService {
    private final MailProperties props;

    public MailService(MailProperties props) {
        // Type-safe, validated configuration
        this.props = props;
    }
}
```

---

## Section 9: Testing Spring Applications

### 9.1 Unit Testing (No Spring)

**Test Business Logic Isolation:**
```java
class OrderServiceTest {

    @Test
    void shouldCalculateTotal() {
        // No Spring needed
        OrderService service = new OrderService(
            mock(OrderRepository.class),
            mock(PaymentService.class)
        );

        Order order = new Order(List.of(
            new Item("Widget", 10.00, 2)
        ));

        assertEquals(20.00, service.calculateTotal(order));
    }
}
```

### 9.2 Integration Testing with Spring

**Load Full Context:**
```java
@SpringBootTest
class OrderIntegrationTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldPersistOrder() {
        Order order = orderService.createOrder(...);

        Order saved = orderRepository.findById(order.getId()).orElseThrow();
        assertEquals(order.getTotal(), saved.getTotal());
    }
}
```

### 9.3 Testing Web Layer

**MockMvc for Controller Tests:**
```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Test
    void shouldReturnUser() throws Exception {
        when(userService.findById(1L))
            .thenReturn(new User(1L, "Alice", "alice@test.com"));

        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Alice"))
            .andExpect(jsonPath("$.email").value("alice@test.com"));
    }

    @Test
    void shouldReturn404WhenNotFound() throws Exception {
        when(userService.findById(999L))
            .thenThrow(new UserNotFoundException(999L));

        mockMvc.perform(get("/api/users/999"))
            .andExpect(status().isNotFound());
    }
}
```

### 9.4 Test Slices

**Load Only What You Need:**
```java
@WebMvcTest          // Only web layer
@DataJpaTest         // Only JPA layer
@JsonTest            // Only JSON serialization
@RestClientTest      // Only REST clients
```

---

## Section 10: When Spring Is Overkill

### 10.1 The Costs of Spring

**Startup Time:**
Spring Boot applications take seconds to start. For CLI tools or serverless functions, that's too slow.

**Memory:**
The framework overhead adds ~50-100MB baseline.

**Learning Curve:**
Understanding Spring takes time. For simple projects, it may not pay off.

**Magic Can Obscure:**
Auto-configuration can make debugging harder when things go wrong.

### 10.2 Alternatives to Consider

**For Simple Web Services:**
- Javalin (lightweight)
- Spark Java (micro framework)
- Helidon SE (microservices)

**For Serverless:**
- Quarkus (fast startup, low memory)
- Micronaut (compile-time DI)
- GraalVM native images

**For CLI Tools:**
- Picocli
- Plain Java with manual DI

### 10.3 When Spring Excels

**Use Spring When:**
- Building enterprise applications
- Need extensive ecosystem (security, data, batch, cloud)
- Team already knows Spring
- Long-running server applications
- Need standardized patterns across teams

**Reconsider When:**
- Startup time is critical
- Memory is constrained
- Simple CRUD with no complexity
- Learning budget is limited

---

## Section 11: Now You Understand Why...

Armed with this foundation, you can now understand:

- **Why Spring apps "just work"**: Auto-configuration reads classpath and properties, creates beans conditionally

- **Why @Autowired finds the right bean**: The container knows all beans and matches by type

- **Why you don't configure Tomcat**: Spring Boot auto-configures an embedded server

- **Why Spring Security is annoying to configure**: It's complex because security is complex; auto-configuration tries but can't know your requirements

- **Why startup is slow**: Component scanning, auto-configuration evaluation, and bean creation take time

- **Why your tests are slow**: @SpringBootTest loads the full context; use slices instead

---

## Practical Exercises

### Exercise 1: Explore Auto-Configuration
```bash
# Create a minimal Spring Boot app
# Add --debug to see auto-configuration report
java -jar myapp.jar --debug

# Find which auto-configurations are active
```

### Exercise 2: Manual Bean Wiring
```java
// Disable component scanning
@SpringBootApplication(scanBasePackages = "nonexistent")
public class ManualApp {

    @Bean
    public UserService userService(UserRepository repo) {
        return new UserService(repo);
    }

    @Bean
    public UserRepository userRepository() {
        return new InMemoryUserRepository();
    }
}
```

### Exercise 3: Conditional Bean
```java
@Configuration
public class StorageConfig {

    @Bean
    @ConditionalOnProperty(name = "storage.type", havingValue = "s3")
    public StorageService s3Storage() {
        return new S3StorageService();
    }

    @Bean
    @ConditionalOnProperty(name = "storage.type", havingValue = "local")
    public StorageService localStorage() {
        return new LocalStorageService();
    }
}
```

### Exercise 4: Compare Startup Times
```bash
# Standard Spring Boot
time java -jar spring-app.jar &
# Note startup time

# Same functionality in Javalin
time java -jar javalin-app.jar &
# Compare
```

---

## Key Takeaways

1. **Spring solves the wiring problem**. Dependency injection eliminates manual object creation and configuration.

2. **Annotations are metadata, not magic**. Spring reads them at runtime via reflection and acts accordingly.

3. **Auto-configuration is conditional**. It checks what's on classpath and in properties, creating beans only when appropriate.

4. **The container manages lifecycle**. Beans are created, wired, and destroyed by Spring, not your code.

5. **Spring has overhead**. For simple cases, consider lighter alternatives. For enterprise applications, Spring's ecosystem is unmatched.

---

## Looking Ahead

With Spring Boot understood, Chapter 6 explores **APIs as Contracts**: How to design APIs that clients can depend on, versioning strategies, and documentation.

---

## Chapter 5 Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SPRING BOOT DEMYSTIFIED                                   │
│                                                                             │
│  THE CORE CONCEPT: Dependency Injection                                     │
│  ────────────────────────────────────────                                   │
│                                                                             │
│    Without DI:                          With DI:                            │
│    class Service {                      class Service {                     │
│      void work() {                        final Repo repo;                  │
│        Repo r = new Repo();  ←─ Hard      Service(Repo repo) {              │
│        r.save(data);            coded       this.repo = repo;  ←─ Injected  │
│      }                                    }                                 │
│    }                                      void work() {                     │
│                                             repo.save(data);                │
│                                           }                                 │
│                                         }                                   │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  THE CONTAINER                                                              │
│  ─────────────                                                              │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │                   ApplicationContext                             │       │
│  │                                                                  │       │
│  │   @Service        @Repository        @Controller                │       │
│  │   UserService ←── UserRepository ←── UserController             │       │
│  │        │                                                        │       │
│  │        └──────→ PasswordEncoder                                 │       │
│  │                                                                  │       │
│  │   Spring creates all beans, resolves dependencies, manages life │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  AUTO-CONFIGURATION                                                         │
│  ──────────────────                                                         │
│                                                                             │
│   MySQL driver on classpath?  ─┬→  YES  ──→  Create DataSource bean        │
│                                │                                            │
│   spring.datasource.url set?  ─┤                                            │
│                                │                                            │
│   User defined DataSource?    ─┴→  NO   ──→  Skip (don't override user)    │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  REQUEST FLOW                                                               │
│  ────────────                                                               │
│                                                                             │
│   HTTP Request → Tomcat → DispatcherServlet → Handler Mapping →            │
│                                                     │                       │
│   ┌──────────────────────────────────────────────────┘                      │
│   ▼                                                                         │
│   Controller Method → Service → Repository → Database                       │
│        │                                                                    │
│        ▼                                                                    │
│   Response ← JSON Serialization ← Return Value                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Next Chapter: APIs as Contracts — Designing APIs that clients can depend on*
