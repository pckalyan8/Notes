# 📗 11.1 — Spring Core: IoC Container, Dependency Injection & Bean Lifecycle

Spring Core is the foundation beneath every other Spring module. Before studying MVC, Security, or Transactions, you must internalize what the IoC container is and why it exists — because everything else is simply a collection of pre-built beans that the container manages.

---

## 🧠 The Problem Spring Solves — Why IoC Exists

To understand Spring's value, start with the problem it solves. In traditional Java code, when class A needs an instance of class B, class A creates it directly:

```java
// Traditional approach — tight coupling
public class OrderService {
    // OrderService is responsible for creating its own dependency
    private PaymentService paymentService = new PaymentService();
    private EmailService emailService = new EmailService(new SmtpConfig("smtp.gmail.com", 587));

    public void placeOrder(Order order) {
        paymentService.charge(order);
        emailService.sendConfirmation(order);
    }
}
```

This approach has serious problems. `OrderService` is now tightly coupled to `PaymentService` — if you want to swap in a `MockPaymentService` for testing, you have to modify `OrderService`'s source code. If `EmailService` changes its constructor, every class that creates one must be updated. Creating the full object graph in the correct order becomes your responsibility. Testing in isolation is nearly impossible.

**Inversion of Control (IoC)** flips this. Instead of objects creating their dependencies, a container creates everything and hands the dependencies in. Your class just declares what it needs — the container figures out how to satisfy that need. You give up control of object creation, and in exchange you get flexibility, testability, and a clean separation of concerns.

**Dependency Injection (DI)** is the mechanism IoC uses: dependencies are "injected" into your class from outside, typically through the constructor, a setter, or a field.

```
Traditional:  OrderService → new PaymentService()   (A creates B)
IoC/DI:       Container → creates PaymentService
              Container → creates OrderService(paymentService)  (container injects B into A)
```

---

## 🏗️ The Spring IoC Container

The Spring IoC container is represented by two main interfaces. `BeanFactory` is the foundational container — it provides the core DI features but is rarely used directly. `ApplicationContext` extends `BeanFactory` and adds enterprise features: event publishing, internationalization, AOP integration, and more. You always use `ApplicationContext` in real applications.

Spring Boot auto-creates an `ApplicationContext` when your application starts. In raw Spring (without Boot), you create it yourself:

```java
// Java-based configuration (modern, preferred over XML)
@Configuration // Marks this class as a source of bean definitions
public class AppConfig {

    // @Bean tells Spring: "call this method and register the return value as a bean"
    // The bean's name defaults to the method name — "paymentService" here
    @Bean
    public PaymentService paymentService() {
        return new StripePaymentService("sk_test_abc123");
    }

    @Bean
    public EmailService emailService() {
        return new SmtpEmailService("smtp.gmail.com", 587);
    }

    @Bean
    public OrderService orderService(PaymentService paymentService, EmailService emailService) {
        // Spring injects the beans declared above into this method's parameters automatically
        return new OrderService(paymentService, emailService);
    }
}

// Starting the container manually (you won't do this in Spring Boot)
public class Main {
    public static void main(String[] args) {
        // AnnotationConfigApplicationContext reads @Configuration classes
        ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

        // Retrieve a bean from the container
        OrderService orderService = ctx.getBean(OrderService.class);
        orderService.placeOrder(new Order(...));

        // Close the container — triggers @PreDestroy callbacks
        ((ConfigurableApplicationContext) ctx).close();
    }
}
```

---

## 🔍 Component Scanning — The Automatic Approach

Manually declaring every bean with `@Bean` in a `@Configuration` class is tedious for large applications. Component scanning is the solution: you annotate your classes with stereotype annotations, and Spring discovers and registers them automatically by scanning your packages.

```java
// Activates component scanning for the package and its sub-packages
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig { }
```

The stereotype annotations are `@Component` (generic bean), `@Service` (business logic — semantic label, no behavioral difference from `@Component`), `@Repository` (data access — Spring also adds exception translation), and `@Controller`/`@RestController` (web layer). In Spring Boot, `@SpringBootApplication` includes `@ComponentScan` automatically — beans in your main class's package and sub-packages are discovered without any configuration.

```java
@Service   // Spring registers this as a bean named "orderService"
public class OrderService {

    private final PaymentService paymentService;
    private final EmailService emailService;

    // Constructor injection — the recommended style (see section below)
    public OrderService(PaymentService paymentService, EmailService emailService) {
        this.paymentService = paymentService;
        this.emailService   = emailService;
    }

    public void placeOrder(Order order) {
        paymentService.charge(order);
        emailService.sendConfirmation(order);
    }
}

@Service
public class StripePaymentService implements PaymentService {
    public void charge(Order order) { System.out.println("Charging via Stripe..."); }
}

@Service
public class SmtpEmailService implements EmailService {
    public void sendConfirmation(Order order) { System.out.println("Sending email..."); }
}
```

---

## 💉 The Three Injection Styles

Spring can inject dependencies in three ways. Each has different trade-offs, and choosing correctly matters for code quality and testability.

### Constructor Injection (Strongly Recommended)

```java
@Service
public class OrderService {

    // Dependencies are final — they cannot change after construction
    private final PaymentService paymentService;
    private final NotificationService notificationService;

    // In Spring 4.3+, @Autowired is optional if there is exactly one constructor
    // Spring automatically uses the single constructor for injection
    public OrderService(PaymentService paymentService,
                        NotificationService notificationService) {
        // You can add validation logic here — impossible with field injection
        if (paymentService == null) throw new IllegalArgumentException("PaymentService required");
        this.paymentService       = paymentService;
        this.notificationService  = notificationService;
    }
}
```

Constructor injection is preferred for three key reasons. First, dependencies are declared as `final`, making the class immutable and thread-safe by default. Second, the class is easy to test without Spring — you just call `new OrderService(mockPaymentService, mockNotificationService)`. Third, a constructor with many parameters is an immediate code smell that tells you the class has too many responsibilities — field injection hides this problem by making it trivially easy to add more dependencies.

### Setter Injection (Use for Optional Dependencies)

```java
@Service
public class ReportService {

    private final DataService dataService; // Required
    private AuditService auditService;     // Optional — may not always be present

    // Constructor for required dependencies
    public ReportService(DataService dataService) {
        this.dataService = dataService;
    }

    // Setter for optional dependency — Spring calls this only if a bean is available
    @Autowired(required = false)
    public void setAuditService(AuditService auditService) {
        this.auditService = auditService;
    }

    public void generateReport() {
        List<Data> data = dataService.fetchData();
        if (auditService != null) {
            auditService.log("Report generated");
        }
    }
}
```

### Field Injection (Avoid in Production Code)

```java
@Service
public class OrderService {

    @Autowired // Spring directly sets this field via reflection, bypassing the constructor
    private PaymentService paymentService;

    @Autowired
    private EmailService emailService;
}
```

Field injection looks clean but has serious problems. Fields are not `final`, so they can be reassigned. The class cannot be instantiated outside Spring without reflection hacks — making unit testing painful. The class hides its dependencies instead of declaring them explicitly. Most style guides and senior engineers discourage field injection in production code. You will see it frequently in tutorials because it requires less typing — but that's the only advantage it has.

---

## 🏷️ @Autowired — Wiring Rules and Qualifiers

When Spring needs to inject a `PaymentService` dependency and finds exactly one `PaymentService` bean in the container, it injects it without ambiguity. But what happens when there are multiple beans of the same type?

```java
@Service("stripePayment")
public class StripePaymentService implements PaymentService { ... }

@Service("paypalPayment")
public class PaypalPaymentService implements PaymentService { ... }

@Service
public class OrderService {

    private final PaymentService paymentService;

    // Spring finds two PaymentService beans — it cannot decide which one to inject
    // This throws NoUniqueBeanDefinitionException without further qualification
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

You resolve this ambiguity with `@Qualifier` (explicit name) or `@Primary` (default preference):

```java
@Service
@Primary // This bean is injected by default when no @Qualifier is specified
public class StripePaymentService implements PaymentService { ... }

@Service
public class PaypalPaymentService implements PaymentService { ... }

@Service
public class OrderService {

    private final PaymentService defaultPayment; // Gets StripePaymentService (@Primary)
    private final PaymentService paypalPayment;  // Gets PaypalPaymentService (@Qualifier)

    public OrderService(
            PaymentService defaultPayment,
            @Qualifier("paypalPaymentService") PaymentService paypalPayment) {
        this.defaultPayment = defaultPayment;
        this.paypalPayment  = paypalPayment;
    }
}
```

---

## 🌍 Bean Scopes — How Many Instances?

Scope controls how many instances of a bean the container creates and how long they live.

```java
// SINGLETON (default) — one shared instance for the entire application lifetime
// Stateless services should always be singleton
@Service
@Scope("singleton") // Redundant — singleton is the default
public class PaymentService { ... }

// PROTOTYPE — a fresh instance is created every time you request the bean
// Use for stateful objects that must not be shared
@Component
@Scope("prototype")
public class ShoppingCart {
    private final List<Item> items = new ArrayList<>();
    public void addItem(Item item) { items.add(item); }
}

// REQUEST — one instance per HTTP request (only in web applications)
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String requestId = UUID.randomUUID().toString();
    public String getRequestId() { return requestId; }
}

// SESSION — one instance per HTTP session
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserSession {
    private String username;
    private List<String> recentSearches = new ArrayList<>();
    // getters and setters...
}
```

A critical subtlety: if a singleton bean depends on a prototype-scoped bean, Spring injects the prototype bean once at startup — meaning the singleton always uses the same prototype instance, defeating the purpose of prototype scope. The solution is to use `@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)`, which injects a proxy that creates a fresh instance on each access.

---

## 🔄 Bean Lifecycle — From Creation to Destruction

Understanding the bean lifecycle lets you run initialization and cleanup logic at the right moments. Spring calls specific methods as beans move through their lifecycle.

```java
@Component
public class DatabaseConnectionPool implements InitializingBean, DisposableBean {

    private final DataSourceConfig config;
    private HikariDataSource pool;

    public DatabaseConnectionPool(DataSourceConfig config) {
        this.config = config;
        System.out.println("1. Constructor called");
    }

    // ── Approach 1: @PostConstruct (JSR-250 — preferred, standard Java annotation) ──
    // Called AFTER all dependencies have been injected and BEFORE the bean is returned to callers
    @PostConstruct
    public void initPool() {
        System.out.println("2. @PostConstruct: initializing connection pool");
        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(config.getUrl());
        hikariConfig.setMaximumPoolSize(config.getPoolSize());
        this.pool = new HikariDataSource(hikariConfig);
    }

    // ── Approach 2: InitializingBean.afterPropertiesSet() ──
    // Called by Spring's lifecycle machinery — more intrusive (couples to Spring)
    @Override
    public void afterPropertiesSet() {
        System.out.println("2b. afterPropertiesSet() called (if not using @PostConstruct)");
    }

    // ── Approach 3: @Bean(initMethod = "customInit") in @Configuration class ──
    // Useful when you don't control the class source code
    public void customInit() { System.out.println("2c. Custom init method"); }

    public DataSource getPool() { return pool; }

    // ── Destruction callbacks ──

    // @PreDestroy is called just BEFORE the bean is removed from the container
    // (on application shutdown or when the context is closed)
    @PreDestroy
    public void shutdownPool() {
        System.out.println("3. @PreDestroy: closing connection pool");
        if (pool != null) pool.close();
    }

    // ── DisposableBean.destroy() — alternative to @PreDestroy ──
    @Override
    public void destroy() {
        System.out.println("3b. destroy() called");
    }
}
```

The full lifecycle order is: Constructor → Dependency injection → `@PostConstruct` → Bean in use → `@PreDestroy` → Bean destroyed.

---

## 📋 Configuration & Externalized Properties

Spring provides rich support for reading configuration from external sources — properties files, YAML, environment variables, and command-line arguments. This is central to the **12-Factor App** principle of externalizing configuration.

```java
// application.properties
app.payment.provider=stripe
app.payment.api-key=sk_live_abc123
app.payment.timeout-seconds=30
app.feature.dark-mode=true
```

```java
// Approach 1: @Value — inject a single property value
@Service
public class PaymentService {

    @Value("${app.payment.provider}")                  // Required — throws if missing
    private String provider;

    @Value("${app.payment.timeout-seconds:10}")        // Default value of 10 if not set
    private int timeoutSeconds;

    @Value("${app.feature.dark-mode:false}")           // Boolean with default false
    private boolean darkModeEnabled;

    @Value("#{${app.retry.delays:1000,2000,4000}}")    // SpEL — parse a comma-separated list
    private List<Integer> retryDelays;
}
```

```java
// Approach 2: @ConfigurationProperties — bind a group of properties to a POJO (preferred)
// More maintainable for groups of related settings, supports nested objects, and validates types

@ConfigurationProperties(prefix = "app.payment")
@Component // Or declare as @Bean in a @Configuration class
@Validated  // Enables Bean Validation on the properties
public class PaymentProperties {

    @NotBlank
    private String provider;

    @NotBlank
    private String apiKey;

    @Min(1) @Max(300)
    private int timeoutSeconds = 30; // Default value

    private Retry retry = new Retry(); // Nested object binding

    // All getters and setters are required for binding
    public String getProvider()      { return provider; }
    public void setProvider(String v){ this.provider = v; }
    public String getApiKey()        { return apiKey; }
    public void setApiKey(String v)  { this.apiKey = v; }
    public int getTimeoutSeconds()   { return timeoutSeconds; }
    public void setTimeoutSeconds(int v) { this.timeoutSeconds = v; }
    public Retry getRetry()          { return retry; }
    public void setRetry(Retry v)    { this.retry = v; }

    public static class Retry {
        private int maxAttempts = 3;
        private long backoffMs  = 1000;
        // getters and setters...
    }
}
```

---

## 🏷️ Profiles — Environment-Specific Configuration

Profiles let you define different beans or properties for different environments (dev, test, staging, prod) and activate the correct one at runtime.

```java
// Define different DataSource beans for dev and production
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev") // Only registered when the "dev" profile is active
    public DataSource devDataSource() {
        // H2 in-memory database for fast local development
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }

    @Bean
    @Profile("prod") // Only registered when the "prod" profile is active
    public DataSource prodDataSource(@Value("${db.url}") String url,
                                     @Value("${db.user}") String user,
                                     @Value("${db.password}") String password) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setUsername(user);
        config.setPassword(password);
        return new HikariDataSource(config);
    }
}
```

```properties
# application-dev.properties (activated by spring.profiles.active=dev)
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true

# application-prod.properties (activated by spring.profiles.active=prod)
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
```

Activate a profile through the property `spring.profiles.active=dev`, as an environment variable `SPRING_PROFILES_ACTIVE=prod`, or via a command-line argument `--spring.profiles.active=staging`.

---

## 🔤 Spring Expression Language (SpEL)

SpEL is a powerful expression language that can be used anywhere Spring accepts an expression — `@Value`, `@ConditionalOnExpression`, security annotations, and more.

```java
@Component
public class SpelDemoBean {

    // Access another bean's property
    @Value("#{paymentService.provider}")
    private String paymentProvider;

    // Conditional expression
    @Value("#{environment['app.feature.enabled'] == 'true' ? 'ON' : 'OFF'}")
    private String featureStatus;

    // Math and collections
    @Value("#{T(Math).random() * 100}")
    private double randomValue;

    @Value("#{{'alice', 'bob', 'carol'}}")
    private List<String> adminUsers;

    // Safe navigation operator — avoids NullPointerException
    @Value("#{userService?.currentUser?.email}")
    private String currentUserEmail; // Returns null instead of throwing NPE
}
```

---

## 📦 ApplicationContext Events

Spring's event system enables decoupled communication between beans without direct dependencies. A bean publishes an event; any number of listeners respond to it independently.

```java
// Define a custom event
public class OrderPlacedEvent extends ApplicationEvent {
    private final Order order;
    public OrderPlacedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
    public Order getOrder() { return order; }
}

// Publish the event from a service
@Service
public class OrderService {

    private final ApplicationEventPublisher eventPublisher;

    public OrderService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public void placeOrder(Order order) {
        // ... business logic, save to DB ...
        // Publish an event — the service doesn't know who listens or what they do
        eventPublisher.publishEvent(new OrderPlacedEvent(this, order));
    }
}

// Listen for the event — completely decoupled from OrderService
@Component
public class OrderNotificationListener {

    @EventListener // Spring registers this method as a listener automatically
    public void onOrderPlaced(OrderPlacedEvent event) {
        System.out.println("Sending confirmation email for order: " + event.getOrder().getId());
    }
}

@Component
public class OrderAnalyticsListener {

    @EventListener
    @Async // Handle the event asynchronously in a different thread
    public void onOrderPlaced(OrderPlacedEvent event) {
        System.out.println("Recording analytics for order: " + event.getOrder().getId());
    }
}
```

Spring also publishes its own lifecycle events (`ContextRefreshedEvent`, `ContextStartedEvent`, `ContextClosedEvent`, `ApplicationReadyEvent`) that you can listen to for application startup/shutdown logic.

---

## ✅ Best Practices & Important Points

**Always prefer constructor injection over field injection.** Constructor injection makes dependencies explicit, enables immutability (`final` fields), and allows easy unit testing without a Spring container. Field injection is seductive because it's shorter, but it hides coupling and makes testing harder.

**Singletons must be stateless.** Since a singleton bean is shared across all threads, any mutable instance state is a concurrency bug waiting to happen. If your bean needs to hold state per-request or per-user, use request-scoped or session-scoped beans, or store state in a `ThreadLocal`.

**Use `@ConfigurationProperties` over `@Value` for groups of settings.** `@ConfigurationProperties` gives you type safety, IDE autocomplete support (with `spring-boot-configuration-processor`), default values in one place, nested object support, and Bean Validation. `@Value` is fine for a single occasional property but becomes messy at scale.

**Use `@PostConstruct` for initialization logic, not the constructor.** In the constructor, dependencies have not yet been injected. If your initialization depends on injected values (e.g., opening a connection pool using an injected URL), run it in `@PostConstruct`, which is guaranteed to run after all injections are complete.

**Do not call overridable methods from `@PostConstruct` or constructors.** Spring creates CGLIB subclasses of beans for AOP proxying — if you call a virtual method in these lifecycle callbacks before the proxy is fully initialized, you may bypass the proxy and miss important behavior (like transaction management).

**Name beans thoughtfully.** When using component scanning, the default name is the class name in camelCase. For `@Bean` methods, the default name is the method name. Explicit naming with `@Service("paymentService")` or `@Bean("primaryDataSource")` prevents ambiguity in large applications.

---

## 🔑 Key Summary

```
IoC           → Container controls object creation and lifecycle; you declare what you need
DI            → Container injects dependencies through constructor, setter, or field
@Component    → Generic bean; @Service, @Repository, @Controller are specialized versions
@Autowired    → Triggers dependency injection; optional if only one constructor exists
@Qualifier    → Resolves ambiguity when multiple beans of the same type exist
@Primary      → Default bean chosen when no @Qualifier is specified
@Scope        → singleton (default, shared), prototype (new per request), request, session
@PostConstruct → Runs after injection is complete; use for initialization logic
@PreDestroy   → Runs before bean is destroyed; use for cleanup/resource release
@Value        → Inject a single property value; supports SpEL expressions
@ConfigurationProperties → Bind a group of properties to a typed POJO
@Profile      → Activate beans only in specific environments (dev, prod, test)
ApplicationContext → The container itself; create via SpringApplication.run() in Spring Boot
```
