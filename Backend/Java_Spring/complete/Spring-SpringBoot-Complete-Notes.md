# Complete Spring & Spring Boot Notes

> **Comprehensive Guide for Spring Framework Interview Preparation**  
> **Covers: Spring Core, Spring Boot, Spring MVC, Spring Data JPA, Spring Security, Microservices, and more**  
> **Last Updated: February 2026**

---

## Table of Contents

1. [Spring Core Concepts](#1-spring-core-concepts)
2. [Spring Boot](#2-spring-boot)
3. [Spring MVC](#3-spring-mvc)
4. [Spring Data JPA](#4-spring-data-jpa)
5. [Spring Security](#5-spring-security)
6. [Spring REST APIs](#6-spring-rest-apis)
7. [Microservices with Spring Cloud](#7-microservices-with-spring-cloud)
8. [Spring AOP](#8-spring-aop)
9. [Spring Testing](#9-spring-testing)
10. [Logging and Monitoring](#10-logging-and-monitoring)
11. [Best Practices](#11-best-practices)

---

## 1. Spring Core Concepts

### 1.1 What is Spring Framework?

Spring is a lightweight, open-source framework for building Java enterprise applications. It provides comprehensive infrastructure support for developing Java applications.

**Key Features:**
- Dependency Injection (DI) / Inversion of Control (IoC)
- Aspect-Oriented Programming (AOP)
- Transaction Management
- MVC Framework
- Data Access Framework
- Security Framework

### 1.2 Dependency Injection (DI)

Dependency Injection is a design pattern where objects receive their dependencies from external sources rather than creating them.

**Benefits:**
- Loose coupling
- Easier testing
- Better code organization
- Reusability

**Types of DI:**

#### Constructor Injection (Recommended)

```java
@Component
public class UserService {
    private final UserRepository userRepository;
    
    // Constructor injection
    @Autowired  // Optional in Spring 4.3+
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public User findUser(Long id) {
        return userRepository.findById(id);
    }
}
```

#### Setter Injection

```java
@Component
public class EmailService {
    private EmailSender emailSender;
    
    @Autowired
    public void setEmailSender(EmailSender emailSender) {
        this.emailSender = emailSender;
    }
}
```

#### Field Injection (Not Recommended)

```java
@Component
public class NotificationService {
    @Autowired
    private EmailService emailService;  // Not recommended
}
```

**Why Constructor Injection is Preferred:**
1. Immutability - fields can be final
2. Testability - easy to mock dependencies
3. Required dependencies - compile-time check
4. No reflection needed

### 1.3 Inversion of Control (IoC)

IoC is a principle where the control of object creation and lifecycle is transferred to a container (Spring IoC Container).

**IoC Container:**

```
┌─────────────────────────────────┐
│   Spring IoC Container          │
│                                 │
│  ┌──────────────────────────┐   │
│  │   Bean Definitions       │   │
│  │  (Configuration)         │   │
│  └──────────────────────────┘   │
│              ↓                  │
│  ┌──────────────────────────┐   │
│  │   Bean Factory/          │   │
│  │   ApplicationContext     │   │
│  └──────────────────────────┘   │
│              ↓                  │
│  ┌──────────────────────────┐   │
│  │   Managed Beans          │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

### 1.4 Bean Scopes

```java
// Singleton (default) - One instance per Spring container
@Component
@Scope("singleton")
public class SingletonBean {
}

// Prototype - New instance each time
@Component
@Scope("prototype")
public class PrototypeBean {
}

// Request - One instance per HTTP request (Web)
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestScopedBean {
}

// Session - One instance per HTTP session (Web)
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class SessionScopedBean {
}

// Application - One instance per ServletContext
@Component
@Scope(value = WebApplicationContext.SCOPE_APPLICATION, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ApplicationScopedBean {
}
```

### 1.5 Bean Lifecycle

```
Bean Lifecycle:
┌──────────────────────────────────────────────────────┐
│ 1. Instantiation                                     │
│    ↓                                                  │
│ 2. Populate Properties (Dependency Injection)        │
│    ↓                                                  │
│ 3. setBeanName (BeanNameAware)                       │
│    ↓                                                  │
│ 4. setBeanFactory (BeanFactoryAware)                 │
│    ↓                                                  │
│ 5. setApplicationContext (ApplicationContextAware)   │
│    ↓                                                  │
│ 6. Pre-initialization (BeanPostProcessor)            │
│    ↓                                                  │
│ 7. afterPropertiesSet (InitializingBean)             │
│    ↓                                                  │
│ 8. Custom init method (@PostConstruct)               │
│    ↓                                                  │
│ 9. Post-initialization (BeanPostProcessor)           │
│    ↓                                                  │
│ 10. Bean Ready for Use                               │
│    ↓                                                  │
│ 11. destroy (DisposableBean)                         │
│    ↓                                                  │
│ 12. Custom destroy method (@PreDestroy)              │
└──────────────────────────────────────────────────────┘
```

**Example:**

```java
@Component
public class LifecycleBean implements InitializingBean, DisposableBean {
    
    private String name;
    
    public LifecycleBean() {
        System.out.println("1. Constructor called");
    }
    
    @PostConstruct
    public void init() {
        System.out.println("2. @PostConstruct - Initialization");
    }
    
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("3. afterPropertiesSet - Properties set");
    }
    
    @PreDestroy
    public void cleanup() {
        System.out.println("4. @PreDestroy - Cleanup");
    }
    
    @Override
    public void destroy() throws Exception {
        System.out.println("5. destroy - Bean destruction");
    }
}
```

### 1.6 ApplicationContext vs BeanFactory

| Feature | BeanFactory | ApplicationContext |
|---------|-------------|-------------------|
| Lazy initialization | Yes | No (by default) |
| Event publishing | No | Yes |
| Internationalization | No | Yes |
| AOP integration | Manual | Automatic |
| Enterprise services | No | Yes |
| Use case | Resource-constrained | Enterprise applications |

```java
// BeanFactory (basic container)
BeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml"));
MyBean bean = (MyBean) factory.getBean("myBean");

// ApplicationContext (advanced container)
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
MyBean bean = context.getBean(MyBean.class);
```

### 1.7 Configuration Approaches

#### 1. Java-based Configuration (Modern, Recommended)

```java
@Configuration
public class AppConfig {
    
    @Bean
    public UserService userService() {
        return new UserService(userRepository());
    }
    
    @Bean
    public UserRepository userRepository() {
        return new UserRepositoryImpl();
    }
    
    @Bean
    @Scope("prototype")
    public PrototypeBean prototypeBean() {
        return new PrototypeBean();
    }
}

// Main application
public class App {
    public static void main(String[] args) {
        ApplicationContext context = 
            new AnnotationConfigApplicationContext(AppConfig.class);
        UserService service = context.getBean(UserService.class);
    }
}
```

#### 2. Annotation-based Configuration

```java
@Component
public class UserRepository {
    public User findById(Long id) {
        // Implementation
    }
}

@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public User getUser(Long id) {
        return userRepository.findById(id);
    }
}

@Configuration
@ComponentScan("com.example")
public class AppConfig {
}
```

#### 3. XML-based Configuration (Legacy)

```xml
<!-- beans.xml -->
<beans>
    <bean id="userRepository" class="com.example.UserRepository"/>
    
    <bean id="userService" class="com.example.UserService">
        <property name="userRepository" ref="userRepository"/>
    </bean>
</beans>
```

### 1.8 Stereotypes Annotations

```java
// @Component - Generic stereotype
@Component
public class GenericComponent {
}

// @Service - Business logic layer
@Service
public class UserService {
}

// @Repository - Data access layer (adds exception translation)
@Repository
public class UserRepository {
}

// @Controller - Presentation layer (Spring MVC)
@Controller
public class UserController {
}

// @RestController - REST API controller (@Controller + @ResponseBody)
@RestController
public class UserRestController {
}

// @Configuration - Configuration class
@Configuration
public class AppConfig {
}
```

### 1.9 @Qualifier and @Primary

```java
// Multiple implementations
interface PaymentService {
    void pay(double amount);
}

@Component
@Primary  // Default choice
public class CreditCardPayment implements PaymentService {
    @Override
    public void pay(double amount) {
        System.out.println("Paying with credit card: " + amount);
    }
}

@Component
public class PayPalPayment implements PaymentService {
    @Override
    public void pay(double amount) {
        System.out.println("Paying with PayPal: " + amount);
    }
}

// Using @Qualifier to specify which implementation
@Service
public class OrderService {
    
    @Autowired
    @Qualifier("payPalPayment")
    private PaymentService paymentService;
    
    // Or using constructor
    public OrderService(@Qualifier("creditCardPayment") PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

### 1.10 @Value and Property Files

```java
// application.properties
// app.name=MyApplication
// app.version=1.0
// database.url=jdbc:mysql://localhost:3306/mydb

@Component
public class AppProperties {
    
    @Value("${app.name}")
    private String appName;
    
    @Value("${app.version}")
    private String version;
    
    @Value("${database.url}")
    private String databaseUrl;
    
    // Default value
    @Value("${app.timeout:30}")
    private int timeout;
    
    // SpEL (Spring Expression Language)
    @Value("#{systemProperties['user.name']}")
    private String userName;
    
    @Value("#{T(java.lang.Math).random() * 100}")
    private double randomNumber;
}
```

---

## 2. Spring Boot

### 2.1 What is Spring Boot?

Spring Boot is an opinionated framework built on top of Spring Framework that simplifies the development of production-ready applications.

**Key Features:**
- Auto-configuration
- Embedded servers (Tomcat, Jetty, Undertow)
- Starter dependencies
- Production-ready features (metrics, health checks)
- No XML configuration required

**Spring vs Spring Boot:**

```
Spring Framework:
┌──────────────────────────────────┐
│ Developer configures everything  │
│ - Dependencies                   │
│ - Bean definitions               │
│ - Server configuration           │
│ - Property files                 │
└──────────────────────────────────┘

Spring Boot:
┌──────────────────────────────────┐
│ Auto-configuration               │
│ - Starter dependencies           │
│ - Embedded server                │
│ - Sensible defaults              │
│ - Override when needed           │
└──────────────────────────────────┘
```

### 2.2 Creating a Spring Boot Application

#### Project Structure

```
my-spring-boot-app/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/demo/
│   │   │       ├── DemoApplication.java
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       └── model/
│   │   └── resources/
│   │       ├── application.properties
│   │       ├── application.yml
│   │       ├── static/
│   │       └── templates/
│   └── test/
│       └── java/
├── pom.xml (Maven) or build.gradle (Gradle)
└── target/
```

#### Main Application Class

```java
@SpringBootApplication
public class DemoApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}

// @SpringBootApplication is equivalent to:
// @Configuration + @EnableAutoConfiguration + @ComponentScan
```

#### pom.xml (Maven)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>1.0.0</version>
    
    <properties>
        <java.version>17</java.version>
    </properties>
    
    <dependencies>
        <!-- Web Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- JPA Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        
        <!-- MySQL Driver -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
        </dependency>
        
        <!-- Lombok (Optional) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

### 2.3 Auto-Configuration

Spring Boot automatically configures beans based on classpath dependencies.

```java
// Auto-configuration example
@Configuration
@ConditionalOnClass(DataSource.class)  // Only if DataSource is on classpath
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean  // Only if no other DataSource bean exists
    public DataSource dataSource(DataSourceProperties properties) {
        return DataSourceBuilder.create()
            .url(properties.getUrl())
            .username(properties.getUsername())
            .password(properties.getPassword())
            .build();
    }
}

// Excluding auto-configuration
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
public class Application {
}
```

### 2.4 Starter Dependencies

Common Spring Boot Starters:

```xml
<!-- Web applications -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- JPA / Hibernate -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Validation -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- Thymeleaf (Template Engine) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>

<!-- Actuator (Monitoring) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Caching -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>

<!-- Redis -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<!-- MongoDB -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>

<!-- WebFlux (Reactive) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

### 2.5 Application Properties

#### application.properties

```properties
# Server Configuration
server.port=8080
server.servlet.context-path=/api

# Database Configuration
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA/Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
spring.jpa.properties.hibernate.format_sql=true

# Logging
logging.level.root=INFO
logging.level.com.example=DEBUG
logging.file.name=app.log

# Custom Properties
app.name=My Application
app.version=1.0.0
```

#### application.yml (Alternative)

```yaml
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver
  
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQLDialect
        format_sql: true

logging:
  level:
    root: INFO
    com.example: DEBUG
  file:
    name: app.log

app:
  name: My Application
  version: 1.0.0
```

### 2.6 Profiles

Profiles allow different configurations for different environments.

**Creating Profiles:**

```properties
# application.properties (default)
app.name=My Application

# application-dev.properties
spring.datasource.url=jdbc:mysql://localhost:3306/dev_db
logging.level.root=DEBUG

# application-prod.properties
spring.datasource.url=jdbc:mysql://prod-server:3306/prod_db
logging.level.root=WARN
```

**Activating Profiles:**

```properties
# In application.properties
spring.profiles.active=dev

# Or via command line
java -jar app.jar --spring.profiles.active=prod

# Or via environment variable
export SPRING_PROFILES_ACTIVE=dev
```

**Profile-specific Beans:**

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new HikariDataSource(); // Dev configuration
    }
    
    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        return new HikariDataSource(); // Prod configuration
    }
}

@Service
@Profile("dev")
public class DevEmailService implements EmailService {
    // Development email implementation
}

@Service
@Profile("prod")
public class ProdEmailService implements EmailService {
    // Production email implementation
}
```

### 2.7 CommandLineRunner and ApplicationRunner

Execute code after application startup:

```java
@Component
public class DataLoader implements CommandLineRunner {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public void run(String... args) throws Exception {
        // Load initial data
        User user = new User("admin", "admin@example.com");
        userRepository.save(user);
        System.out.println("Initial data loaded");
    }
}

@Component
public class AppInitializer implements ApplicationRunner {
    
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("Application started!");
        System.out.println("Non-option args: " + args.getNonOptionArgs());
        System.out.println("Option names: " + args.getOptionNames());
    }
}
```

### 2.8 Actuator

Spring Boot Actuator provides production-ready features.

**Add Dependency:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

**Configuration:**

```properties
# Expose all endpoints
management.endpoints.web.exposure.include=*

# Expose specific endpoints
management.endpoints.web.exposure.include=health,info,metrics

# Customize endpoint path
management.endpoints.web.base-path=/actuator

# Show detailed health information
management.endpoint.health.show-details=always
```

**Common Endpoints:**

- `/actuator/health` - Application health
- `/actuator/info` - Application info
- `/actuator/metrics` - Application metrics
- `/actuator/env` - Environment properties
- `/actuator/loggers` - Logger configuration
- `/actuator/beans` - All Spring beans
- `/actuator/mappings` - All @RequestMapping paths

**Custom Health Indicator:**

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        boolean isHealthy = checkHealth();
        
        if (isHealthy) {
            return Health.up()
                .withDetail("service", "running")
                .withDetail("error", "none")
                .build();
        } else {
            return Health.down()
                .withDetail("service", "not running")
                .withDetail("error", "Connection failed")
                .build();
        }
    }
    
    private boolean checkHealth() {
        // Check service health
        return true;
    }
}
```

**Custom Info Contributor:**

```java
@Component
public class CustomInfoContributor implements InfoContributor {
    
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("app", Map.of(
            "name", "My Application",
            "version", "1.0.0",
            "author", "John Doe"
        ));
    }
}
```

---

## 3. Spring MVC

### 3.1 MVC Architecture

```
Client Request Flow:
┌──────────────┐
│   Client     │
└──────┬───────┘
       │ HTTP Request
       ↓
┌──────────────────────────────────┐
│   DispatcherServlet              │
│   (Front Controller)             │
└──────┬───────────────────────────┘
       │
       ↓
┌──────────────────────────────────┐
│   Handler Mapping                │
│   (Find Controller)              │
└──────┬───────────────────────────┘
       │
       ↓
┌──────────────────────────────────┐
│   Controller                     │
│   (Process Request)              │
└──────┬───────────────────────────┘
       │
       ↓
┌──────────────────────────────────┐
│   Service Layer                  │
│   (Business Logic)               │
└──────┬───────────────────────────┘
       │
       ↓
┌──────────────────────────────────┐
│   Repository/DAO                 │
│   (Data Access)                  │
└──────┬───────────────────────────┘
       │
       ↓
┌──────────────────────────────────┐
│   Database                       │
└──────────────────────────────────┘
```

### 3.2 Controllers

```java
@Controller
public class UserController {
    
    @Autowired
    private UserService userService;
    
    // Handle GET request
    @GetMapping("/users")
    public String listUsers(Model model) {
        List<User> users = userService.getAllUsers();
        model.addAttribute("users", users);
        return "users";  // View name
    }
    
    // Path variable
    @GetMapping("/users/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        User user = userService.getUserById(id);
        model.addAttribute("user", user);
        return "user-details";
    }
    
    // Request parameter
    @GetMapping("/search")
    public String searchUsers(
            @RequestParam String name,
            @RequestParam(required = false, defaultValue = "0") int page,
            Model model) {
        List<User> users = userService.searchByName(name, page);
        model.addAttribute("users", users);
        return "search-results";
    }
    
    // POST request
    @PostMapping("/users")
    public String createUser(@ModelAttribute User user) {
        userService.saveUser(user);
        return "redirect:/users";
    }
    
    // Handle form submission
    @GetMapping("/users/new")
    public String showCreateForm(Model model) {
        model.addAttribute("user", new User());
        return "user-form";
    }
    
    @PostMapping("/users/save")
    public String saveUser(@Valid @ModelAttribute User user, 
                          BindingResult result,
                          Model model) {
        if (result.hasErrors()) {
            return "user-form";
        }
        userService.saveUser(user);
        return "redirect:/users";
    }
}
```

### 3.3 REST Controllers

```java
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    
    @Autowired
    private UserService userService;
    
    // GET all users
    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        List<User> users = userService.getAllUsers();
        return ResponseEntity.ok(users);
    }
    
    // GET user by ID
    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable Long id) {
        User user = userService.getUserById(id);
        if (user == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(user);
    }
    
    // POST - Create user
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        User savedUser = userService.saveUser(user);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(savedUser.getId())
            .toUri();
        return ResponseEntity.created(location).body(savedUser);
    }
    
    // PUT - Update user
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody User user) {
        User existingUser = userService.getUserById(id);
        if (existingUser == null) {
            return ResponseEntity.notFound().build();
        }
        user.setId(id);
        User updatedUser = userService.saveUser(user);
        return ResponseEntity.ok(updatedUser);
    }
    
    // PATCH - Partial update
    @PatchMapping("/{id}")
    public ResponseEntity<User> partialUpdateUser(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {
        User user = userService.partialUpdate(id, updates);
        return ResponseEntity.ok(user);
    }
    
    // DELETE user
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }
    
    // Query parameters
    @GetMapping("/search")
    public ResponseEntity<List<User>> searchUsers(
            @RequestParam String name,
            @RequestParam(required = false) String email) {
        List<User> users = userService.search(name, email);
        return ResponseEntity.ok(users);
    }
}
```

### 3.4 Request Mapping Annotations

```java
@RestController
@RequestMapping("/api")
public class MappingExamples {
    
    // GET
    @GetMapping("/items")
    public List<Item> getItems() { }
    
    // POST
    @PostMapping("/items")
    public Item createItem(@RequestBody Item item) { }
    
    // PUT
    @PutMapping("/items/{id}")
    public Item updateItem(@PathVariable Long id, @RequestBody Item item) { }
    
    // DELETE
    @DeleteMapping("/items/{id}")
    public void deleteItem(@PathVariable Long id) { }
    
    // PATCH
    @PatchMapping("/items/{id}")
    public Item patchItem(@PathVariable Long id, @RequestBody Map<String, Object> updates) { }
    
    // Multiple paths
    @GetMapping({"/items", "/products"})
    public List<Item> getItemsOrProducts() { }
    
    // Path with regex
    @GetMapping("/items/{id:[0-9]+}")
    public Item getItem(@PathVariable Long id) { }
    
    // Consumes (Content-Type)
    @PostMapping(value = "/items", consumes = "application/json")
    public Item createJsonItem(@RequestBody Item item) { }
    
    // Produces (Accept)
    @GetMapping(value = "/items", produces = "application/json")
    public List<Item> getItemsAsJson() { }
    
    // Multiple HTTP methods
    @RequestMapping(value = "/items", method = {RequestMethod.GET, RequestMethod.POST})
    public ResponseEntity<?> handleItems() { }
}
```

### 3.5 Request/Response Handling

```java
@RestController
@RequestMapping("/api")
public class RequestResponseExamples {
    
    // @PathVariable
    @GetMapping("/users/{id}/posts/{postId}")
    public Post getUserPost(
            @PathVariable Long id,
            @PathVariable Long postId) {
        // Access path variables
    }
    
    // @RequestParam
    @GetMapping("/search")
    public List<User> search(
            @RequestParam String query,
            @RequestParam(required = false, defaultValue = "0") int page,
            @RequestParam(required = false, defaultValue = "10") int size) {
        // Access query parameters
    }
    
    // @RequestBody
    @PostMapping("/users")
    public User createUser(@Valid @RequestBody User user) {
        // user object is deserialized from JSON
    }
    
    // @RequestHeader
    @GetMapping("/header-example")
    public String getHeader(@RequestHeader("User-Agent") String userAgent) {
        return "User-Agent: " + userAgent;
    }
    
    // @CookieValue
    @GetMapping("/cookie-example")
    public String getCookie(@CookieValue("sessionId") String sessionId) {
        return "Session ID: " + sessionId;
    }
    
    // @ModelAttribute (form data)
    @PostMapping("/users/form")
    public User createUserFromForm(@ModelAttribute User user) {
        // user object populated from form fields
    }
    
    // Multiple request parameters with Map
    @GetMapping("/params")
    public Map<String, String> getAllParams(@RequestParam Map<String, String> params) {
        return params;
    }
    
    // HttpServletRequest and HttpServletResponse
    @GetMapping("/servlet-example")
    public void servletExample(HttpServletRequest request, HttpServletResponse response) {
        String method = request.getMethod();
        String userAgent = request.getHeader("User-Agent");
        response.setStatus(HttpServletResponse.SC_OK);
    }
}
```

### 3.6 Exception Handling

```java
// Custom Exception
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

// Controller Advice (Global Exception Handler)
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    // Handle specific exception
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(
            ResourceNotFoundException ex,
            WebRequest request) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
    
    // Handle validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationExceptions(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
    }
    
    // Handle all other exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(
            Exception ex,
            WebRequest request) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "An error occurred: " + ex.getMessage(),
            LocalDateTime.now()
        );
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}

// Error Response DTO
@Data
@AllArgsConstructor
public class ErrorResponse {
    private int status;
    private String message;
    private LocalDateTime timestamp;
}

// Controller-specific exception handler
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<String> handleUserNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }
}
```

### 3.7 Validation

```java
// Entity with validation annotations
@Entity
@Data
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 50, message = "Name must be between 2 and 50 characters")
    private String name;
    
    @NotBlank(message = "Email is required")
    @Email(message = "Email should be valid")
    private String email;
    
    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 100, message = "Age must be less than 100")
    private Integer age;
    
    @Pattern(regexp = "^\\+?[0-9. ()-]{7,25}$", message = "Phone number is invalid")
    private String phone;
    
    @Past(message = "Birth date must be in the past")
    private LocalDate birthDate;
    
    @Future(message = "Event date must be in the future")
    private LocalDate eventDate;
}

// Controller with validation
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        // If validation fails, MethodArgumentNotValidException is thrown
        User savedUser = userService.save(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(savedUser);
    }
}

// Custom Validator
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = AgeValidator.class)
public @interface ValidAge {
    String message() default "Invalid age";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class AgeValidator implements ConstraintValidator<ValidAge, Integer> {
    
    @Override
    public boolean isValid(Integer age, ConstraintValidatorContext context) {
        return age != null && age >= 18 && age <= 100;
    }
}

// Usage
@ValidAge
private Integer age;
```

### 3.8 Interceptors

```java
// Custom Interceptor
@Component
public class LoggingInterceptor implements HandlerInterceptor {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingInterceptor.class);
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) throws Exception {
        logger.info("Pre Handle: " + request.getMethod() + " " + request.getRequestURI());
        return true;  // Continue processing
    }
    
    @Override
    public void postHandle(HttpServletRequest request, 
                          HttpServletResponse response, 
                          Object handler, 
                          ModelAndView modelAndView) throws Exception {
        logger.info("Post Handle: " + response.getStatus());
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, 
                               Exception ex) throws Exception {
        logger.info("After Completion");
    }
}

// Register Interceptor
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Autowired
    private LoggingInterceptor loggingInterceptor;
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**");
    }
}
```

### 3.9 CORS Configuration

```java
// Method-level CORS
@RestController
@RequestMapping("/api")
public class ApiController {
    
    @CrossOrigin(origins = "http://localhost:3000")
    @GetMapping("/data")
    public List<Data> getData() {
        return dataService.findAll();
    }
}

// Controller-level CORS
@RestController
@RequestMapping("/api/users")
@CrossOrigin(origins = "http://localhost:3000", maxAge = 3600)
public class UserController {
}

// Global CORS Configuration
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("http://localhost:3000", "https://example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
                .allowedHeaders("*")
                .allowCredentials(true)
                .maxAge(3600);
    }
}
```

---

## 4. Spring Data JPA

### 4.1 What is Spring Data JPA?

Spring Data JPA simplifies data access layer implementation by reducing boilerplate code.

**Architecture:**

```
┌──────────────────────────────────┐
│   Repository Interface           │
│   (JpaRepository, CrudRepository)│
└─────────────┬────────────────────┘
              │
              ↓
┌──────────────────────────────────┐
│   Spring Data JPA                │
│   (Auto-implementation)          │
└─────────────┬────────────────────┘
              │
              ↓
┌──────────────────────────────────┐
│   JPA Provider (Hibernate)       │
└─────────────┬────────────────────┘
              │
              ↓
┌──────────────────────────────────┐
│   Database                       │
└──────────────────────────────────┘
```

### 4.2 Entity Mapping

```java
@Entity
@Table(name = "users")
@Data  // Lombok
@NoArgsConstructor
@AllArgsConstructor
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "username", nullable = false, unique = true, length = 50)
    private String username;
    
    @Column(nullable = false)
    private String email;
    
    @Column(name = "full_name")
    private String fullName;
    
    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "created_at", updatable = false)
    private Date createdAt;
    
    @Temporal(TemporalType.TIMESTAMP)
    @Column(name = "updated_at")
    private Date updatedAt;
    
    @PrePersist
    protected void onCreate() {
        createdAt = new Date();
        updatedAt = new Date();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = new Date();
    }
}
```

### 4.3 Relationships

#### One-to-One

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String username;
    
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private Profile profile;
}

@Entity
public class Profile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String bio;
    
    @OneToOne
    @JoinColumn(name = "user_id", referencedColumnName = "id")
    private User user;
}
```

#### One-to-Many / Many-to-One

```java
@Entity
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Employee> employees = new ArrayList<>();
    
    // Helper methods
    public void addEmployee(Employee employee) {
        employees.add(employee);
        employee.setDepartment(this);
    }
    
    public void removeEmployee(Employee employee) {
        employees.remove(employee);
        employee.setDepartment(null);
    }
}

@Entity
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;
}
```

#### Many-to-Many

```java
@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String title;
    
    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

### 4.4 Repository Interfaces

```java
// CrudRepository - basic CRUD operations
public interface UserCrudRepository extends CrudRepository<User, Long> {
    // Inherits: save, saveAll, findById, findAll, count, delete, deleteById, etc.
}

// JpaRepository - extends CrudRepository with JPA-specific methods
public interface UserRepository extends JpaRepository<User, Long> {
    // Inherits: flush, saveAndFlush, deleteInBatch, findAll with Sort/Pageable
}

// Custom queries
public interface UserRepository extends JpaRepository<User, Long> {
    
    // Query methods (method name parsing)
    User findByUsername(String username);
    User findByEmail(String email);
    List<User> findByFullNameContaining(String name);
    List<User> findByAgeGreaterThan(int age);
    List<User> findByAgeBetween(int start, int end);
    List<User> findByCreatedAtAfter(Date date);
    List<User> findByUsernameAndEmail(String username, String email);
    List<User> findByUsernameOrEmail(String username, String email);
    List<User> findByOrderByCreatedAtDesc();
    List<User> findTop10ByOrderByCreatedAtDesc();
    
    boolean existsByUsername(String username);
    long countByAge(int age);
    void deleteByUsername(String username);
    
    // @Query annotation - JPQL
    @Query("SELECT u FROM User u WHERE u.email = ?1")
    User findByEmailJPQL(String email);
    
    @Query("SELECT u FROM User u WHERE u.username = :username")
    User findByUsernameJPQL(@Param("username") String username);
    
    @Query("SELECT u FROM User u WHERE u.fullName LIKE %:name%")
    List<User> searchByName(@Param("name") String name);
    
    // Native SQL query
    @Query(value = "SELECT * FROM users WHERE email = ?1", nativeQuery = true)
    User findByEmailNative(String email);
    
    // Modifying query
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.email = :email WHERE u.id = :id")
    int updateEmail(@Param("id") Long id, @Param("email") String email);
    
    @Modifying
    @Transactional
    @Query("DELETE FROM User u WHERE u.createdAt < :date")
    void deleteOldUsers(@Param("date") Date date);
}
```

### 4.5 Query Methods Keywords

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    
    // Equality
    List<Product> findByName(String name);
    List<Product> findByNameIs(String name);
    List<Product> findByNameEquals(String name);
    
    // Null check
    List<Product> findByDescriptionIsNull();
    List<Product> findByDescriptionIsNotNull();
    
    // Like / Containing
    List<Product> findByNameLike(String pattern);
    List<Product> findByNameContaining(String name);
    List<Product> findByNameStartingWith(String prefix);
    List<Product> findByNameEndingWith(String suffix);
    
    // Comparison
    List<Product> findByPriceGreaterThan(Double price);
    List<Product> findByPriceLessThan(Double price);
    List<Product> findByPriceGreaterThanEqual(Double price);
    List<Product> findByPriceLessThanEqual(Double price);
    List<Product> findByPriceBetween(Double start, Double end);
    
    // In / Not In
    List<Product> findByCategoryIn(List<String> categories);
    List<Product> findByCategoryNotIn(List<String> categories);
    
    // Boolean
    List<Product> findByActiveTrue();
    List<Product> findByActiveFalse();
    
    // Ordering
    List<Product> findByOrderByPriceAsc();
    List<Product> findByOrderByPriceDesc();
    List<Product> findByCategoryOrderByPriceAsc(String category);
    
    // Limiting
    List<Product> findTop10ByOrderByPriceDesc();
    List<Product> findFirst5ByCategory(String category);
    
    // Combining conditions
    List<Product> findByNameAndCategory(String name, String category);
    List<Product> findByNameOrCategory(String name, String category);
    List<Product> findByPriceGreaterThanAndCategory(Double price, String category);
    
    // Distinct
    List<Product> findDistinctByCategory(String category);
    
    // Ignore case
    List<Product> findByNameIgnoreCase(String name);
    List<Product> findByNameContainingIgnoreCase(String name);
}
```

### 4.6 Pagination and Sorting

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserRepository userRepository;
    
    // Simple pagination
    @GetMapping
    public Page<User> getAllUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        Pageable pageable = PageRequest.of(page, size);
        return userRepository.findAll(pageable);
    }
    
    // Pagination with sorting
    @GetMapping("/sorted")
    public Page<User> getUsersSorted(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "id") String sortBy,
            @RequestParam(defaultValue = "ASC") String direction) {
        
        Sort sort = direction.equalsIgnoreCase("DESC") ?
                    Sort.by(sortBy).descending() :
                    Sort.by(sortBy).ascending();
        
        Pageable pageable = PageRequest.of(page, size, sort);
        return userRepository.findAll(pageable);
    }
    
    // Multiple sort fields
    @GetMapping("/multi-sort")
    public Page<User> getUsersMultiSort(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        
        Sort sort = Sort.by("createdAt").descending()
                       .and(Sort.by("username").ascending());
        
        Pageable pageable = PageRequest.of(page, size, sort);
        return userRepository.findAll(pageable);
    }
}

// Repository with custom query
public interface UserRepository extends JpaRepository<User, Long> {
    
    Page<User> findByFullNameContaining(String name, Pageable pageable);
    
    @Query("SELECT u FROM User u WHERE u.age > :age")
    Page<User> findUsersOlderThan(@Param("age") int age, Pageable pageable);
}
```

### 4.7 Specifications (Dynamic Queries)

```java
// Enable JPA Specifications
public interface UserRepository extends JpaRepository<User, Long>, 
                                       JpaSpecificationExecutor<User> {
}

// Specification class
public class UserSpecifications {
    
    public static Specification<User> hasUsername(String username) {
        return (root, query, cb) -> 
            username == null ? null : cb.equal(root.get("username"), username);
    }
    
    public static Specification<User> hasEmail(String email) {
        return (root, query, cb) -> 
            email == null ? null : cb.equal(root.get("email"), email);
    }
    
    public static Specification<User> ageGreaterThan(Integer age) {
        return (root, query, cb) -> 
            age == null ? null : cb.greaterThan(root.get("age"), age);
    }
    
    public static Specification<User> createdAfter(Date date) {
        return (root, query, cb) -> 
            date == null ? null : cb.greaterThan(root.get("createdAt"), date);
    }
}

// Service using Specifications
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public List<User> searchUsers(String username, String email, Integer minAge) {
        Specification<User> spec = Specification.where(null);
        
        if (username != null) {
            spec = spec.and(UserSpecifications.hasUsername(username));
        }
        if (email != null) {
            spec = spec.and(UserSpecifications.hasEmail(email));
        }
        if (minAge != null) {
            spec = spec.and(UserSpecifications.ageGreaterThan(minAge));
        }
        
        return userRepository.findAll(spec);
    }
}
```

### 4.8 Transactions

```java
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private AuditLogRepository auditLogRepository;
    
    // Declarative transaction
    @Transactional
    public User createUser(User user) {
        User savedUser = userRepository.save(user);
        auditLogRepository.save(new AuditLog("User created: " + user.getUsername()));
        return savedUser;
    }
    
    // Read-only transaction (optimization)
    @Transactional(readOnly = true)
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }
    
    // Transaction with timeout
    @Transactional(timeout = 5)
    public void processLongRunningOperation() {
        // Operation must complete within 5 seconds
    }
    
    // Transaction with propagation
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void independentTransaction() {
        // Always creates new transaction
    }
    
    // Transaction with rollback rules
    @Transactional(rollbackFor = Exception.class)
    public void methodWithRollback() throws Exception {
        // Rollback on any Exception
    }
    
    @Transactional(noRollbackFor = IllegalArgumentException.class)
    public void methodWithoutRollback() {
        // Don't rollback on IllegalArgumentException
    }
    
    // Transaction isolation level
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void highIsolationLevel() {
        // Highest isolation level
    }
}

// Programmatic transaction
@Service
public class ProgrammaticTransactionService {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    public User createUser(User user) {
        return transactionTemplate.execute(status -> {
            try {
                User savedUser = userRepository.save(user);
                auditLogRepository.save(new AuditLog("User created"));
                return savedUser;
            } catch (Exception e) {
                status.setRollbackOnly();
                throw e;
            }
        });
    }
}
```

**Transaction Propagation:**

```
REQUIRED (default) - Use existing or create new
REQUIRES_NEW - Always create new transaction
MANDATORY - Must be called within existing transaction
NEVER - Must not be called within transaction
NOT_SUPPORTED - Suspend current transaction
SUPPORTS - Use transaction if exists, otherwise non-transactional
NESTED - Execute within nested transaction
```

### 4.9 Auditing

```java
// Enable JPA Auditing
@Configuration
@EnableJpaAuditing
public class JpaConfig {
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.of("system");  // Or get from SecurityContext
    }
}

// Base auditing entity
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Auditable {
    
    @CreatedBy
    @Column(updatable = false)
    private String createdBy;
    
    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdDate;
    
    @LastModifiedBy
    private String lastModifiedBy;
    
    @LastModifiedDate
    private LocalDateTime lastModifiedDate;
}

// Entity extending auditable
@Entity
public class Product extends Auditable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private Double price;
}
```

---

## 5. Spring Security

### 5.1 Security Architecture

```
Security Filter Chain:
┌──────────────────────────────────┐
│   Client Request                 │
└────────────┬─────────────────────┘
             │
    ┌────────▼─────────┐
    │ SecurityFilter   │
    │ Chain            │
    └────────┬─────────┘
             │
    ┌────────▼──────────────────┐
    │ Authentication Filters    │
    │ - UsernamePassword       │
    │ - JWT                    │
    │ - OAuth2                 │
    └────────┬──────────────────┘
             │
    ┌────────▼──────────────────┐
    │ Authentication Manager   │
    └────────┬──────────────────┘
             │
    ┌────────▼──────────────────┐
    │ Authentication Provider  │
    └────────┬──────────────────┘
             │
    ┌────────▼──────────────────┐
    │ UserDetailsService       │
    └────────┬──────────────────┘
             │
    ┌────────▼──────────────────┐
    │ Authorization            │
    └────────┬──────────────────┘
             │
    ┌────────▼──────────────────┐
    │ Controller               │
    └──────────────────────────┘
```

### 5.2 Basic Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
                .permitAll()
            )
            .csrf(csrf -> csrf.disable())  // Disable for REST APIs
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 5.3 UserDetailsService

```java
// User Entity
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String username;
    
    @Column(nullable = false)
    private String password;
    
    @Column(nullable = false)
    private String email;
    
    private boolean enabled = true;
    
    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
}

// Role Entity
@Entity
@Table(name = "roles")
public class Role {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String name;  // ROLE_USER, ROLE_ADMIN
}

// Custom UserDetails
public class UserPrincipal implements UserDetails {
    
    private User user;
    
    public UserPrincipal(User user) {
        this.user = user;
    }
    
    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority(role.getName()))
            .collect(Collectors.toList());
    }
    
    @Override
    public String getPassword() {
        return user.getPassword();
    }
    
    @Override
    public String getUsername() {
        return user.getUsername();
    }
    
    @Override
    public boolean isAccountNonExpired() {
        return true;
    }
    
    @Override
    public boolean isAccountNonLocked() {
        return true;
    }
    
    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }
    
    @Override
    public boolean isEnabled() {
        return user.isEnabled();
    }
}

// UserDetailsService Implementation
@Service
public class CustomUserDetailsService implements UserDetailsService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) 
            throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> 
                new UsernameNotFoundException("User not found: " + username));
        return new UserPrincipal(user);
    }
}
```

### 5.4 JWT Authentication

```java
// JWT Utility Class
@Component
public class JwtUtil {
    
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration}")
    private Long expiration;
    
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        return createToken(claims, userDetails.getUsername());
    }
    
    private String createToken(Map<String, Object> claims, String subject) {
        return Jwts.builder()
            .setClaims(claims)
            .setSubject(subject)
            .setIssuedAt(new Date(System.currentTimeMillis()))
            .setExpiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(SignatureAlgorithm.HS256, secret)
            .compact();
    }
    
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }
    
    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
    
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }
    
    private Claims extractAllClaims(String token) {
        return Jwts.parser()
            .setSigningKey(secret)
            .parseClaimsJws(token)
            .getBody();
    }
    
    private Boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
    
    public Boolean validateToken(String token, UserDetails userDetails) {
        final String username = extractUsername(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }
}

// JWT Request Filter
@Component
public class JwtRequestFilter extends OncePerRequestFilter {
    
    @Autowired
    private CustomUserDetailsService userDetailsService;
    
    @Autowired
    private JwtUtil jwtUtil;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                   HttpServletResponse response, 
                                   FilterChain chain)
            throws ServletException, IOException {
        
        final String authorizationHeader = request.getHeader("Authorization");
        
        String username = null;
        String jwt = null;
        
        if (authorizationHeader != null && authorizationHeader.startsWith("Bearer ")) {
            jwt = authorizationHeader.substring(7);
            username = jwtUtil.extractUsername(jwt);
        }
        
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            
            if (jwtUtil.validateToken(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        chain.doFilter(request, response);
    }
}

// Security Configuration with JWT
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Autowired
    private JwtRequestFilter jwtRequestFilter;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}

// Authentication Controller
@RestController
@RequestMapping("/api/auth")
public class AuthController {
    
    @Autowired
    private AuthenticationManager authenticationManager;
    
    @Autowired
    private JwtUtil jwtUtil;
    
    @Autowired
    private CustomUserDetailsService userDetailsService;
    
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody LoginRequest request) {
        try {
            authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                    request.getUsername(), 
                    request.getPassword()
                )
            );
        } catch (BadCredentialsException e) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body("Invalid credentials");
        }
        
        final UserDetails userDetails = userDetailsService
            .loadUserByUsername(request.getUsername());
        final String jwt = jwtUtil.generateToken(userDetails);
        
        return ResponseEntity.ok(new AuthResponse(jwt));
    }
}
```

### 5.5 Method-Level Security

```java
// Enable method security
@Configuration
@EnableMethodSecurity
public class MethodSecurityConfig {
}

@Service
public class UserService {
    
    // Pre-authorization
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) {
        // Only ADMIN can execute
    }
    
    @PreAuthorize("hasAuthority('USER_READ')")
    public User getUserById(Long id) {
        // Only users with USER_READ authority
    }
    
    @PreAuthorize("#username == authentication.principal.username")
    public User getUser(String username) {
        // Only the user themselves
    }
    
    @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
    public void updateUser(Long id, User user) {
        // ADMIN or own profile
    }
    
    // Post-authorization
    @PostAuthorize("returnObject.username == authentication.principal.username")
    public User loadUserByUsername(String username) {
        // Check after method execution
    }
    
    // Secured annotation
    @Secured("ROLE_ADMIN")
    public void adminOnlyMethod() {
    }
    
    @Secured({"ROLE_USER", "ROLE_ADMIN"})
    public void userOrAdminMethod() {
    }
}
```

---

Due to length constraints, I'll continue the comprehensive Spring notes in the next part. Let me save this first part.


## 6. Spring REST APIs

### 6.1 RESTful Principles

**REST** (Representational State Transfer) principles:

1. **Client-Server Architecture** - Separation of concerns
2. **Stateless** - Each request contains all necessary information
3. **Cacheable** - Responses must define themselves as cacheable or not
4. **Uniform Interface** - Standard HTTP methods
5. **Layered System** - Client doesn't know if connected to end server
6. **Code on Demand** (Optional) - Server can send executable code

**HTTP Methods:**

| Method | Purpose | Idempotent | Safe |
|--------|---------|------------|------|
| GET | Retrieve resource | Yes | Yes |
| POST | Create resource | No | No |
| PUT | Update/Replace resource | Yes | No |
| PATCH | Partial update | No | No |
| DELETE | Delete resource | Yes | No |
| HEAD | Get headers only | Yes | Yes |
| OPTIONS | Get supported methods | Yes | Yes |

### 6.2 REST Controller Best Practices

```java
@RestController
@RequestMapping("/api/v1/products")
@Validated
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    // GET all products with pagination
    @GetMapping
    public ResponseEntity<Page<ProductDTO>> getAllProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "id") String sortBy) {
        
        Page<ProductDTO> products = productService.getAllProducts(page, size, sortBy);
        return ResponseEntity.ok(products);
    }
    
    // GET single product
    @GetMapping("/{id}")
    public ResponseEntity<ProductDTO> getProduct(@PathVariable Long id) {
        ProductDTO product = productService.getProductById(id);
        return ResponseEntity.ok(product);
    }
    
    // POST - Create product
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<ProductDTO> createProduct(
            @Valid @RequestBody ProductDTO productDTO) {
        ProductDTO created = productService.createProduct(productDTO);
        
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.getId())
            .toUri();
        
        return ResponseEntity.created(location).body(created);
    }
    
    // PUT - Full update
    @PutMapping("/{id}")
    public ResponseEntity<ProductDTO> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody ProductDTO productDTO) {
        ProductDTO updated = productService.updateProduct(id, productDTO);
        return ResponseEntity.ok(updated);
    }
    
    // PATCH - Partial update
    @PatchMapping("/{id}")
    public ResponseEntity<ProductDTO> partialUpdate(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {
        ProductDTO updated = productService.partialUpdate(id, updates);
        return ResponseEntity.ok(updated);
    }
    
    // DELETE
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        productService.deleteProduct(id);
        return ResponseEntity.noContent().build();
    }
}
```

### 6.3 DTO Pattern

```java
// Entity
@Entity
@Table(name = "products")
@Data
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String name;
    private String description;
    private Double price;
    private Integer stock;
    
    @ManyToOne
    @JoinColumn(name = "category_id")
    private Category category;
    
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// DTO
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ProductDTO {
    private Long id;
    
    @NotBlank(message = "Name is required")
    private String name;
    
    private String description;
    
    @NotNull(message = "Price is required")
    @Min(value = 0, message = "Price must be positive")
    private Double price;
    
    @Min(value = 0, message = "Stock must be non-negative")
    private Integer stock;
    
    private Long categoryId;
    private String categoryName;
}

// Mapper (Manual)
@Component
public class ProductMapper {
    
    public ProductDTO toDTO(Product product) {
        ProductDTO dto = new ProductDTO();
        dto.setId(product.getId());
        dto.setName(product.getName());
        dto.setDescription(product.getDescription());
        dto.setPrice(product.getPrice());
        dto.setStock(product.getStock());
        
        if (product.getCategory() != null) {
            dto.setCategoryId(product.getCategory().getId());
            dto.setCategoryName(product.getCategory().getName());
        }
        
        return dto;
    }
    
    public Product toEntity(ProductDTO dto) {
        Product product = new Product();
        product.setName(dto.getName());
        product.setDescription(dto.getDescription());
        product.setPrice(dto.getPrice());
        product.setStock(dto.getStock());
        return product;
    }
}

// Using MapStruct (Recommended)
@Mapper(componentModel = "spring")
public interface ProductMapper {
    
    @Mapping(source = "category.id", target = "categoryId")
    @Mapping(source = "category.name", target = "categoryName")
    ProductDTO toDTO(Product product);
    
    @Mapping(source = "categoryId", target = "category.id")
    Product toEntity(ProductDTO dto);
    
    List<ProductDTO> toDTOList(List<Product> products);
}
```

### 6.4 API Versioning

```java
// 1. URI Versioning
@RestController
@RequestMapping("/api/v1/users")
public class UserV1Controller {
    @GetMapping("/{id}")
    public UserV1DTO getUser(@PathVariable Long id) {
        // Version 1 implementation
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserV2Controller {
    @GetMapping("/{id}")
    public UserV2DTO getUser(@PathVariable Long id) {
        // Version 2 implementation
    }
}

// 2. Request Parameter Versioning
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(params = "version=1")
    public UserV1DTO getUserV1(@RequestParam Long id) {
        // Version 1
    }
    
    @GetMapping(params = "version=2")
    public UserV2DTO getUserV2(@RequestParam Long id) {
        // Version 2
    }
}

// 3. Header Versioning
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(headers = "X-API-VERSION=1")
    public UserV1DTO getUserV1(@RequestParam Long id) {
        // Version 1
    }
    
    @GetMapping(headers = "X-API-VERSION=2")
    public UserV2DTO getUserV2(@RequestParam Long id) {
        // Version 2
    }
}

// 4. Media Type Versioning (Content Negotiation)
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(produces = "application/vnd.company.app-v1+json")
    public UserV1DTO getUserV1(@RequestParam Long id) {
        // Version 1
    }
    
    @GetMapping(produces = "application/vnd.company.app-v2+json")
    public UserV2DTO getUserV2(@RequestParam Long id) {
        // Version 2
    }
}
```

### 6.5 API Documentation with Swagger/OpenAPI

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

```java
// Configuration
@Configuration
public class OpenAPIConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Product API")
                .version("1.0")
                .description("Product Management API Documentation")
                .contact(new Contact()
                    .name("API Support")
                    .email("support@example.com")
                    .url("https://example.com"))
                .license(new License()
                    .name("Apache 2.0")
                    .url("https://www.apache.org/licenses/LICENSE-2.0")))
            .servers(List.of(
                new Server().url("http://localhost:8080").description("Development"),
                new Server().url("https://api.example.com").description("Production")
            ));
    }
}

// Controller with Swagger annotations
@RestController
@RequestMapping("/api/products")
@Tag(name = "Product", description = "Product management APIs")
public class ProductController {
    
    @Operation(
        summary = "Get all products",
        description = "Retrieve a paginated list of all products"
    )
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Successfully retrieved"),
        @ApiResponse(responseCode = "400", description = "Invalid parameters"),
        @ApiResponse(responseCode = "500", description = "Internal server error")
    })
    @GetMapping
    public ResponseEntity<Page<ProductDTO>> getAllProducts(
            @Parameter(description = "Page number", example = "0")
            @RequestParam(defaultValue = "0") int page,
            
            @Parameter(description = "Page size", example = "10")
            @RequestParam(defaultValue = "10") int size) {
        // Implementation
    }
    
    @Operation(summary = "Get product by ID")
    @GetMapping("/{id}")
    public ResponseEntity<ProductDTO> getProduct(
            @Parameter(description = "Product ID", required = true)
            @PathVariable Long id) {
        // Implementation
    }
    
    @Operation(summary = "Create new product")
    @PostMapping
    public ResponseEntity<ProductDTO> createProduct(
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "Product to create",
                required = true)
            @Valid @RequestBody ProductDTO productDTO) {
        // Implementation
    }
}

// DTO with schema annotations
@Schema(description = "Product Data Transfer Object")
@Data
public class ProductDTO {
    
    @Schema(description = "Product ID", example = "1", accessMode = Schema.AccessMode.READ_ONLY)
    private Long id;
    
    @Schema(description = "Product name", example = "Laptop", required = true)
    @NotBlank
    private String name;
    
    @Schema(description = "Product price", example = "999.99", required = true)
    @NotNull
    @Min(0)
    private Double price;
}
```

**Access Swagger UI:** `http://localhost:8080/swagger-ui.html`

### 6.6 HATEOAS

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
</dependency>
```

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/{id}")
    public EntityModel<UserDTO> getUser(@PathVariable Long id) {
        UserDTO user = userService.getUserById(id);
        
        EntityModel<UserDTO> resource = EntityModel.of(user);
        
        // Add self link
        resource.add(linkTo(methodOn(UserController.class).getUser(id)).withSelfRel());
        
        // Add related links
        resource.add(linkTo(methodOn(UserController.class).getAllUsers()).withRel("all-users"));
        resource.add(linkTo(methodOn(OrderController.class).getUserOrders(id)).withRel("orders"));
        
        return resource;
    }
    
    @GetMapping
    public CollectionModel<EntityModel<UserDTO>> getAllUsers() {
        List<EntityModel<UserDTO>> users = userService.getAllUsers().stream()
            .map(user -> {
                EntityModel<UserDTO> resource = EntityModel.of(user);
                resource.add(linkTo(methodOn(UserController.class)
                    .getUser(user.getId())).withSelfRel());
                return resource;
            })
            .collect(Collectors.toList());
        
        CollectionModel<EntityModel<UserDTO>> resources = CollectionModel.of(users);
        resources.add(linkTo(methodOn(UserController.class).getAllUsers()).withSelfRel());
        
        return resources;
    }
}
```

### 6.7 Content Negotiation

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    // Support both JSON and XML
    @GetMapping(produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE})
    public List<Product> getProducts() {
        return productService.getAllProducts();
    }
    
    // Accept both JSON and XML
    @PostMapping(
        consumes = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE},
        produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE}
    )
    public Product createProduct(@RequestBody Product product) {
        return productService.save(product);
    }
}

// Configuration
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(true)
            .parameterName("mediaType")
            .ignoreAcceptHeader(false)
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

---

## 7. Microservices with Spring Cloud

### 7.1 Microservices Architecture

```
Microservices Ecosystem:
┌──────────────────────────────────────────────────────────┐
│                    API Gateway                           │
│               (Spring Cloud Gateway)                      │
└───────────────────┬──────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
    ┌───▼────┐  ┌──▼─────┐  ┌─▼──────┐
    │Service │  │Service │  │Service │
    │   A    │  │   B    │  │   C    │
    └───┬────┘  └──┬─────┘  └─┬──────┘
        │          │          │
    ┌───▼──────────▼──────────▼───────┐
    │     Service Discovery            │
    │      (Eureka Server)             │
    └──────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
    ┌───▼────┐  ┌──▼─────┐  ┌─▼──────┐
    │Config  │  │Circuit │  │Distrib.│
    │Server  │  │Breaker │  │Tracing │
    └────────┘  └────────┘  └────────┘
```

### 7.2 Service Discovery - Eureka

**Eureka Server:**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
  server:
    wait-time-in-ms-when-sync-empty: 0
```

**Eureka Client (Microservice):**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```yaml
# application.yml
spring:
  application:
    name: user-service

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

### 7.3 API Gateway - Spring Cloud Gateway

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

```java
@SpringBootApplication
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

```yaml
# application.yml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/users/**
          filters:
            - RewritePath=/api/users/(?<segment>.*), /${segment}
            - AddRequestHeader=X-Request-Source, API-Gateway
        
        - id: product-service
          uri: lb://PRODUCT-SERVICE
          predicates:
            - Path=/api/products/**
          filters:
            - RewritePath=/api/products/(?<segment>.*), /${segment}
        
        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/api/orders/**
          filters:
            - name: CircuitBreaker
              args:
                name: orderServiceCircuitBreaker
                fallbackUri: forward:/fallback/orders

server:
  port: 8080

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

**Custom Gateway Filters:**

```java
@Component
public class LoggingFilter implements GlobalFilter, Ordered {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingFilter.class);
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        logger.info("Request: " + exchange.getRequest().getURI());
        
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            logger.info("Response: " + exchange.getResponse().getStatusCode());
        }));
    }
    
    @Override
    public int getOrder() {
        return -1;  // Higher priority
    }
}

// Authentication Filter
@Component
public class AuthenticationFilter implements GlobalFilter, Ordered {
    
    @Autowired
    private JwtUtil jwtUtil;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        if (!request.getHeaders().containsKey("Authorization")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        
        String token = request.getHeaders().get("Authorization").get(0);
        
        if (token != null && token.startsWith("Bearer ")) {
            token = token.substring(7);
            
            if (!jwtUtil.validateToken(token)) {
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }
        }
        
        return chain.filter(exchange);
    }
    
    @Override
    public int getOrder() {
        return -2;
    }
}
```

### 7.4 Config Server

**Config Server:**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8888

spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: main
          search-paths: '{application}'
```

**Config Client:**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  application:
    name: user-service
  config:
    import: optional:configserver:http://localhost:8888
  cloud:
    config:
      fail-fast: true
      retry:
        max-attempts: 5
```

### 7.5 Circuit Breaker - Resilience4j

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
</dependency>
```

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private CircuitBreakerFactory circuitBreakerFactory;
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderDTO> getOrder(@PathVariable Long id) {
        CircuitBreaker circuitBreaker = circuitBreakerFactory.create("orderService");
        
        OrderDTO order = circuitBreaker.run(
            () -> orderService.getOrder(id),
            throwable -> getFallbackOrder(id)
        );
        
        return ResponseEntity.ok(order);
    }
    
    private OrderDTO getFallbackOrder(Long id) {
        OrderDTO fallback = new OrderDTO();
        fallback.setId(id);
        fallback.setStatus("Service Unavailable");
        return fallback;
    }
}

// Using @CircuitBreaker annotation
@Service
public class ProductService {
    
    @CircuitBreaker(name = "productService", fallbackMethod = "getProductFallback")
    @Retry(name = "productService")
    @RateLimiter(name = "productService")
    public Product getProduct(Long id) {
        // Call external service
        return restTemplate.getForObject(
            "http://PRODUCT-SERVICE/api/products/" + id,
            Product.class
        );
    }
    
    private Product getProductFallback(Long id, Exception e) {
        Product fallback = new Product();
        fallback.setId(id);
        fallback.setName("Product Unavailable");
        return fallback;
    }
}
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      productService:
        register-health-indicator: true
        sliding-window-size: 10
        minimum-number-of-calls: 5
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
        wait-duration-in-open-state: 5s
        failure-rate-threshold: 50
        event-consumer-buffer-size: 10
  
  retry:
    instances:
      productService:
        max-attempts: 3
        wait-duration: 2s
  
  ratelimiter:
    instances:
      productService:
        limit-for-period: 10
        limit-refresh-period: 1s
        timeout-duration: 0s
```

### 7.6 Inter-Service Communication

**RestTemplate (Synchronous):**

```java
@Configuration
public class RestTemplateConfig {
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class OrderService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    public UserDTO getUser(Long userId) {
        String url = "http://USER-SERVICE/api/users/" + userId;
        return restTemplate.getForObject(url, UserDTO.class);
    }
    
    public ProductDTO getProduct(Long productId) {
        String url = "http://PRODUCT-SERVICE/api/products/" + productId;
        ResponseEntity<ProductDTO> response = restTemplate.getForEntity(url, ProductDTO.class);
        return response.getBody();
    }
    
    public OrderDTO createOrder(OrderDTO order) {
        String url = "http://ORDER-SERVICE/api/orders";
        return restTemplate.postForObject(url, order, OrderDTO.class);
    }
}
```

**OpenFeign (Declarative HTTP Client):**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableFeignClients
public class Application {
}

// Feign Client
@FeignClient(name = "USER-SERVICE", fallback = UserClientFallback.class)
public interface UserClient {
    
    @GetMapping("/api/users/{id}")
    UserDTO getUserById(@PathVariable Long id);
    
    @GetMapping("/api/users")
    List<UserDTO> getAllUsers();
    
    @PostMapping("/api/users")
    UserDTO createUser(@RequestBody UserDTO user);
}

// Fallback implementation
@Component
public class UserClientFallback implements UserClient {
    
    @Override
    public UserDTO getUserById(Long id) {
        UserDTO fallback = new UserDTO();
        fallback.setId(id);
        fallback.setName("User Service Unavailable");
        return fallback;
    }
    
    @Override
    public List<UserDTO> getAllUsers() {
        return Collections.emptyList();
    }
    
    @Override
    public UserDTO createUser(UserDTO user) {
        return new UserDTO();
    }
}

// Using Feign Client
@Service
public class OrderService {
    
    @Autowired
    private UserClient userClient;
    
    public OrderDTO createOrder(CreateOrderRequest request) {
        UserDTO user = userClient.getUserById(request.getUserId());
        // Create order with user info
        return order;
    }
}
```

### 7.7 Distributed Tracing - Sleuth & Zipkin

```xml
<!-- Spring Cloud Sleuth -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>

<!-- Zipkin -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  sleuth:
    sampler:
      probability: 1.0  # 100% sampling for development
  zipkin:
    base-url: http://localhost:9411
```

**Run Zipkin:**
```bash
docker run -d -p 9411:9411 openzipkin/zipkin
```

---

## 8. Spring AOP

### 8.1 AOP Concepts

**Aspect-Oriented Programming** separates cross-cutting concerns from business logic.

**Key Concepts:**
- **Aspect**: Module containing cross-cutting concern
- **Join Point**: Point in program execution (method call)
- **Advice**: Action taken at join point
- **Pointcut**: Expression matching join points
- **Weaving**: Linking aspects with application

```
┌──────────────────────────────────┐
│   Business Logic                 │
│                                  │
│   ┌────────────────────┐        │
│   │  Method A()        │        │
│   └────────────────────┘        │
│          │                       │
│          ▼                       │
│   ┌─────────────────────────┐  │
│   │  ASPECT (Logging,       │  │
│   │  Security, Transaction) │  │
│   └─────────────────────────┘  │
└──────────────────────────────────┘
```

### 8.2 Advice Types

```java
@Aspect
@Component
public class LoggingAspect {
    
    private static final Logger logger = LoggerFactory.getLogger(LoggingAspect.class);
    
    // Before Advice - executes before method
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        logger.info("Before method: " + joinPoint.getSignature().getName());
    }
    
    // After Returning - executes after successful method completion
    @AfterReturning(
        pointcut = "execution(* com.example.service.*.*(..))",
        returning = "result"
    )
    public void logAfterReturning(JoinPoint joinPoint, Object result) {
        logger.info("Method returned: " + result);
    }
    
    // After Throwing - executes if method throws exception
    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "error"
    )
    public void logAfterThrowing(JoinPoint joinPoint, Throwable error) {
        logger.error("Method threw exception: " + error.getMessage());
    }
    
    // After - executes after method (finally)
    @After("execution(* com.example.service.*.*(..))")
    public void logAfter(JoinPoint joinPoint) {
        logger.info("After method: " + joinPoint.getSignature().getName());
    }
    
    // Around - most powerful, wraps method execution
    @Around("execution(* com.example.service.*.*(..))")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        
        logger.info("Method started: " + joinPoint.getSignature().getName());
        
        Object result = joinPoint.proceed();  // Execute method
        
        long duration = System.currentTimeMillis() - start;
        logger.info("Method completed in " + duration + "ms");
        
        return result;
    }
}
```

### 8.3 Pointcut Expressions

```java
@Aspect
@Component
public class PointcutExamples {
    
    // Execute for all methods in service package
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}
    
    // Execute for all methods in repository package
    @Pointcut("execution(* com.example.repository.*.*(..))")
    public void repositoryLayer() {}
    
    // Execute for specific method
    @Pointcut("execution(* com.example.service.UserService.createUser(..))")
    public void createUserMethod() {}
    
    // Execute for methods returning specific type
    @Pointcut("execution(com.example.dto.UserDTO com.example.service.*.*(..))")
    public void returnsUserDTO() {}
    
    // Execute for methods with specific parameter
    @Pointcut("execution(* com.example.service.*.*(Long, ..))")
    public void firstParamLong() {}
    
    // Within - all methods in class/package
    @Pointcut("within(com.example.service.*)")
    public void withinService() {}
    
    // @annotation - methods with specific annotation
    @Pointcut("@annotation(com.example.annotation.LogExecutionTime)")
    public void logExecutionTimeAnnotation() {}
    
    // @within - classes with specific annotation
    @Pointcut("@within(org.springframework.stereotype.Service)")
    public void serviceAnnotation() {}
    
    // Combining pointcuts
    @Pointcut("serviceLayer() && !createUserMethod()")
    public void serviceLayerExceptCreateUser() {}
    
    @Before("serviceLayer() || repositoryLayer()")
    public void logBeforeServiceOrRepository(JoinPoint joinPoint) {
        // Log before service or repository methods
    }
}
```

### 8.4 Custom Annotations with AOP

```java
// Custom annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogExecutionTime {
}

// Aspect
@Aspect
@Component
public class ExecutionTimeAspect {
    
    private static final Logger logger = LoggerFactory.getLogger(ExecutionTimeAspect.class);
    
    @Around("@annotation(LogExecutionTime)")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        
        Object proceed = joinPoint.proceed();
        
        long duration = System.currentTimeMillis() - start;
        logger.info(joinPoint.getSignature() + " executed in " + duration + "ms");
        
        return proceed;
    }
}

// Usage
@Service
public class UserService {
    
    @LogExecutionTime
    public User createUser(User user) {
        // Method logic
        return userRepository.save(user);
    }
}
```

---

## 9. Spring Testing

### 9.1 Unit Testing

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
// Service Test with Mockito
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void testGetUserById() {
        // Arrange
        User user = new User(1L, "john", "john@example.com");
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));
        
        // Act
        User result = userService.getUserById(1L);
        
        // Assert
        assertNotNull(result);
        assertEquals("john", result.getUsername());
        verify(userRepository, times(1)).findById(1L);
    }
    
    @Test
    void testCreateUser() {
        User user = new User(null, "jane", "jane@example.com");
        User savedUser = new User(1L, "jane", "jane@example.com");
        
        when(userRepository.save(any(User.class))).thenReturn(savedUser);
        
        User result = userService.createUser(user);
        
        assertNotNull(result.getId());
        assertEquals("jane", result.getUsername());
        verify(userRepository).save(user);
    }
    
    @Test
    void testGetUserById_NotFound() {
        when(userRepository.findById(999L)).thenReturn(Optional.empty());
        
        assertThrows(UserNotFoundException.class, () -> {
            userService.getUserById(999L);
        });
    }
}
```

### 9.2 Integration Testing

```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Autowired
    private UserRepository userRepository;
    
    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }
    
    @Test
    void testGetAllUsers() throws Exception {
        // Arrange
        User user1 = userRepository.save(new User(null, "john", "john@example.com"));
        User user2 = userRepository.save(new User(null, "jane", "jane@example.com"));
        
        // Act & Assert
        mockMvc.perform(get("/api/users"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.length()").value(2))
            .andExpect(jsonPath("$[0].username").value("john"))
            .andExpect(jsonPath("$[1].username").value("jane"));
    }
    
    @Test
    void testCreateUser() throws Exception {
        UserDTO userDTO = new UserDTO(null, "bob", "bob@example.com");
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(userDTO)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(jsonPath("$.username").value("bob"));
        
        assertEquals(1, userRepository.count());
    }
    
    @Test
    void testGetUserById_NotFound() throws Exception {
        mockMvc.perform(get("/api/users/999"))
            .andExpect(status().isNotFound());
    }
}
```

### 9.3 Repository Testing

```java
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testFindByUsername() {
        // Arrange
        User user = new User(null, "john", "john@example.com");
        entityManager.persist(user);
        entityManager.flush();
        
        // Act
        User found = userRepository.findByUsername("john");
        
        // Assert
        assertNotNull(found);
        assertEquals("john", found.getUsername());
    }
    
    @Test
    void testFindByEmail() {
        User user = new User(null, "jane", "jane@example.com");
        entityManager.persist(user);
        
        User found = userRepository.findByEmail("jane@example.com");
        
        assertNotNull(found);
        assertEquals("jane", found.getUsername());
    }
}
```

### 9.4 Testing with Test Containers

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mysql</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@SpringBootTest
@Testcontainers
class UserServiceIntegrationTest {
    
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testDatabaseOperations() {
        User user = new User(null, "test", "test@example.com");
        User saved = userRepository.save(user);
        
        assertNotNull(saved.getId());
        
        User found = userRepository.findById(saved.getId()).orElseThrow();
        assertEquals("test", found.getUsername());
    }
}
```

---

## 10. Logging and Monitoring

### 10.1 Logging with SLF4J and Logback

```xml
<!-- Included in spring-boot-starter-web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
</dependency>
```

```java
@Service
public class UserService {
    
    private static final Logger logger = LoggerFactory.getLogger(UserService.class);
    
    public User createUser(User user) {
        logger.debug("Creating user: {}", user.getUsername());
        
        try {
            User savedUser = userRepository.save(user);
            logger.info("User created successfully: {}", savedUser.getId());
            return savedUser;
        } catch (Exception e) {
            logger.error("Error creating user: {}", user.getUsername(), e);
            throw e;
        }
    }
}

// Using Lombok
@Slf4j
@Service
public class ProductService {
    
    public Product getProduct(Long id) {
        log.debug("Fetching product with ID: {}", id);
        return productRepository.findById(id).orElseThrow();
    }
}
```

**logback-spring.xml:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    
    <property name="LOG_PATH" value="logs"/>
    <property name="LOG_FILE" value="application"/>
    
    <!-- Console Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- File Appender -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${LOG_FILE}.log</file>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
        
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${LOG_FILE}.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>
    
    <!-- Error File Appender -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/error.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/error.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
    </appender>
    
    <!-- Specific logger levels -->
    <logger name="com.example" level="DEBUG"/>
    <logger name="org.springframework" level="INFO"/>
    <logger name="org.hibernate" level="INFO"/>
    
    <!-- Root logger -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="ERROR_FILE"/>
    </root>
    
</configuration>
```

### 10.2 Application Monitoring with Actuator

```properties
# application.properties
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
management.endpoint.shutdown.enabled=true

# Custom info
info.app.name=@project.name@
info.app.description=@project.description@
info.app.version=@project.version@
```

**Custom Metrics:**

```java
@Component
public class CustomMetrics {
    
    private final Counter userCreatedCounter;
    private final Timer orderProcessingTimer;
    
    public CustomMetrics(MeterRegistry registry) {
        this.userCreatedCounter = Counter.builder("users.created")
            .description("Number of users created")
            .register(registry);
        
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Order processing time")
            .register(registry);
    }
    
    public void incrementUserCreated() {
        userCreatedCounter.increment();
    }
    
    public void recordOrderProcessing(Runnable task) {
        orderProcessingTimer.record(task);
    }
}
```

---

## 11. Best Practices

### 11.1 Project Structure

```
src/main/java/com/example/myapp/
├── config/
│   ├── SecurityConfig.java
│   ├── SwaggerConfig.java
│   └── DatabaseConfig.java
├── controller/
│   ├── UserController.java
│   └── ProductController.java
├── dto/
│   ├── UserDTO.java
│   └── ProductDTO.java
├── entity/
│   ├── User.java
│   └── Product.java
├── repository/
│   ├── UserRepository.java
│   └── ProductRepository.java
├── service/
│   ├── UserService.java
│   ├── UserServiceImpl.java
│   ├── ProductService.java
│   └── ProductServiceImpl.java
├── exception/
│   ├── CustomException.java
│   └── GlobalExceptionHandler.java
├── mapper/
│   └── UserMapper.java
└── MyAppApplication.java
```

### 11.2 Coding Standards

1. **Use Constructor Injection**
2. **Keep Controllers Thin** - delegate to services
3. **Use DTOs** - don't expose entities directly
4. **Validate Input** - use @Valid annotations
5. **Handle Exceptions Properly** - use @ControllerAdvice
6. **Use Meaningful Names** - clear and descriptive
7. **Write Tests** - aim for good coverage
8. **Document APIs** - use Swagger/OpenAPI
9. **Use Profiles** - separate dev, test, prod configs
10. **Log Appropriately** - use correct log levels

### 11.3 Performance Tips

1. **Use Pagination** - for large datasets
2. **Enable Caching** - for frequently accessed data
3. **Optimize Queries** - avoid N+1 problems
4. **Use Lazy Loading** - carefully
5. **Connection Pooling** - configure properly
6. **Async Processing** - for long-running tasks
7. **Database Indexing** - on frequently queried fields

---

## Summary

This comprehensive guide covered:

1. ✅ **Spring Core** - DI, IoC, Bean lifecycle, Configuration
2. ✅ **Spring Boot** - Auto-configuration, Starters, Actuator, Profiles
3. ✅ **Spring MVC** - Controllers, Validation, Exception Handling
4. ✅ **Spring Data JPA** - Entities, Repositories, Queries, Transactions
5. ✅ **Spring Security** - Authentication, Authorization, JWT
6. ✅ **REST APIs** - Best practices, Documentation, Versioning
7. ✅ **Microservices** - Eureka, Gateway, Config Server, Circuit Breaker
8. ✅ **AOP** - Aspects, Advice types, Custom annotations
9. ✅ **Testing** - Unit, Integration, Repository tests
10. ✅ **Logging & Monitoring** - SLF4J, Logback, Actuator, Metrics

---

## Additional Resources

- [Spring Framework Documentation](https://docs.spring.io/spring-framework/docs/current/reference/html/)
- [Spring Boot Documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [Spring Data JPA Documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
- [Spring Security Documentation](https://docs.spring.io/spring-security/reference/)
- [Spring Cloud Documentation](https://spring.io/projects/spring-cloud)

**Good luck with your Spring interviews!** 🚀

