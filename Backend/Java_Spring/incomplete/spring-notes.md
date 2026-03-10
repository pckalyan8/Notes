# Comprehensive and Detailed Spring Framework Notes

## Table of Contents
1. [Spring Core](#1-spring-core)
2. [Spring Boot](#2-spring-boot)
3. [Spring MVC](#3-spring-mvc)
4. [Spring Data JPA](#4-spring-data-jpa)
5. [Spring Security](#5-spring-security)
6. [Spring REST APIs](#6-spring-rest-apis)

---

## 1. Spring Core

### 1.1 Inversion of Control (IoC) and Dependency Injection (DI)

**Inversion of Control (IoC)** is a design principle where the control of object creation, configuration, and lifecycle is passed from the application to an external container. The Spring IoC container is the concrete implementation of this principle.

**Dependency Injection (DI)** is the primary pattern used to achieve IoC. An object's dependencies (i.e., other objects it needs to work with) are supplied to it by the container, rather than the object creating them itself.

**`BeanFactory` vs. `ApplicationContext`**
- **`BeanFactory`**: The root interface for accessing the Spring bean container. It provides basic IoC features and uses lazy initialization for beans (a bean is created only when it's first requested).
- **`ApplicationContext`**: A sub-interface of `BeanFactory`. It's the most commonly used container and provides more enterprise-specific functionality, such as event propagation, declarative mechanisms, and integration with AOP. It pre-instantiates singleton beans by default upon startup.

**Types of Dependency Injection:**
- **Constructor Injection**: Dependencies are provided as arguments to the constructor. This is the **recommended approach for mandatory dependencies**, as it ensures the object is fully initialized and immutable upon creation.

    ```java
    @Service
    public class ProductService {
        private final ProductRepository productRepository; // final, ensures it's set at construction

        // @Autowired is optional on a single constructor in recent Spring versions
        public ProductService(ProductRepository productRepository) {
            this.productRepository = productRepository;
        }
    }
    ```

- **Setter Injection**: Dependencies are injected through public setter methods. This is suitable for **optional dependencies** that can be set after the bean has been constructed.

    ```java
    @Component
    public class NotificationService {
        private EmailService emailService;

        @Autowired
        public void setEmailService(EmailService emailService) {
            this.emailService = emailService;
        }
    }
    ```
- **Field Injection**: Dependencies are injected directly into the fields. While concise, it's **generally discouraged** because it makes testing more difficult (you must use reflection to set mock objects) and can hide dependencies.

    ```java
    @RestController
    public class ProductController {
        @Autowired
        private ProductService productService; // Harder to test with mocks
    }
    ```

### 1.2 Resolving Ambiguous Dependencies

When you have multiple beans of the same type, Spring doesn't know which one to inject. You can resolve this using:
- **`@Primary`**: Marks one bean as the default choice.

    ```java
    @Repository("mySqlProductRepository")
    @Primary
    public class MySqlProductRepository implements ProductRepository { /* ... */ }

    @Repository("mongoProductRepository")
    public class MongoProductRepository implements ProductRepository { /* ... */ }
    ```
- **`@Qualifier`**: Specifies which bean to inject by name.

    ```java
    @Service
    public class ProductService {
        private final ProductRepository productRepository;

        public ProductService(@Qualifier("mongoProductRepository") ProductRepository productRepository) {
            this.productRepository = productRepository;
        }
    }
    ```

### 1.3 Bean Lifecycle, Callbacks, and Scopes

**Detailed Bean Lifecycle**:
1.  **Instantiation**: The container creates the bean instance.
2.  **Populate Properties**: Dependencies are injected via setter or field injection.
3.  **Aware Interfaces**: `setBeanName()`, `setBeanFactory()`, `setApplicationContext()` are called.
4.  **`BeanPostProcessor` (pre-initialization)**: The `postProcessBeforeInitialization` method of any registered `BeanPostProcessor` is called.
5.  **Initialization Callbacks**:
    - Method annotated with `@PostConstruct`.
    - `afterPropertiesSet()` method if the bean implements `InitializingBean`.
    - Custom `init-method` specified in XML or `@Bean(initMethod=...)`.
6.  **`BeanPostProcessor` (post-initialization)**: The `postProcessAfterInitialization` method is called.
7.  **Bean is Ready**: The bean is now in use.
8.  **Destruction**: When the container is shut down:
    - Method annotated with `@PreDestroy`.
    - `destroy()` method if the bean implements `DisposableBean`.
    - Custom `destroy-method`.

**Lifecycle Callback Example**:
```java
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

@Component
public class LifecycleBean {
    public LifecycleBean() { System.out.println("1. Constructor"); }

    @PostConstruct
    public void init() { System.out.println("2. @PostConstruct: Bean is initialized"); }

    public void doWork() { System.out.println("3. Bean is in use"); }

    @PreDestroy
    public void cleanup() { System.out.println("4. @PreDestroy: Cleaning up"); }
}
```

**Bean Scopes**:
- **`singleton`**: (Default) One instance per container.
- **`prototype`**: A new instance is created for each injection or `getBean()` call. The container does not manage the full lifecycle; the destruction callbacks are not called.
- **`request`**: One instance per HTTP request.
- **`session`**: One instance per HTTP session.
- **`application`**: One instance per `ServletContext`.

```java
import org.springframework.web.context.annotation.SessionScope;
import org.springframework.stereotype.Component;

@Component
@SessionScope // A new ShoppingCart is created for each user session
public class ShoppingCart {
    // ...
}
```

---

## 2. Spring Boot

### 2.1 The `@SpringBootApplication` Annotation

This is a convenience annotation that combines three others:
- **`@Configuration`**: Tags the class as a source of bean definitions for the application context.
- **`@EnableAutoConfiguration`**: Tells Spring Boot to start adding beans based on classpath settings, other beans, and various property settings.
- **`@ComponentScan`**: Tells Spring to look for other components, configurations, and services in the specified package, allowing it to find and register controllers and other beans. By default, it scans from the package of the main application class.

### 2.2 Auto-configuration In-Depth

Spring Boot's auto-configuration attempts to configure your Spring application based on the JAR dependencies you have added. It works by checking for the presence of certain classes on the classpath. For example:
- If `spring-webmvc.jar` is on the classpath, `@EnableAutoConfiguration` triggers the `WebMvcAutoConfiguration`, which sets up a `DispatcherServlet`, a default `ViewResolver`, and other web-related beans.
- If a database driver (like `h2.jar` or `mysql-connector-java.jar`) and `spring-data-jpa` are present, `DataSourceAutoConfiguration` and `JpaRepositoriesAutoConfiguration` will attempt to configure a `DataSource` and scan for JPA repositories.

### 2.3 Externalized Configuration

Spring Boot provides a powerful mechanism for externalizing configuration using `application.properties` or `application.yml`.

**Binding to Objects (`@ConfigurationProperties`)**:
```yaml
# application.yml
app:
  name: My Awesome App
  author: "Jane Doe"
  contact:
    email: jane.doe@example.com
    phone: "123-456-7890"
```
```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private String author;
    private Contact contact;
    // Getters and Setters for all fields
}
```

### 2.4 Application Runners

If you need to execute code after the `ApplicationContext` is created but before the application starts serving requests, you can use `CommandLineRunner` or `ApplicationRunner`.

- **`CommandLineRunner`**: Provides access to application arguments as a simple string array.
- **`ApplicationRunner`**: Provides access to arguments via the `ApplicationArguments` interface, which offers more flexibility.

```java
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class DatabaseInitializer implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        System.out.println("Initializing database with default data...");
        // Your initialization logic here
    }
}
```

### 2.5 Spring Boot Actuator

Actuator brings production-ready features to your application.
Add the dependency: `<artifactId>spring-boot-starter-actuator</artifactId>`

**Key Endpoints** (exposed via `/actuator/...`):
- `health`: Shows application health information.
- `info`: Displays arbitrary application info.
- `metrics`: Shows various metrics (JVM memory, CPU usage, etc.).
- `env`: Displays the current environment properties.
- `beans`: Displays a complete list of all Spring beans in your application.
- `mappings`: Displays a collated list of all `@RequestMapping` paths.

You can configure which endpoints to expose in `application.properties`:
`management.endpoints.web.exposure.include=health,info,prometheus`

---

## 3. Spring MVC

### 3.1 `DispatcherServlet` Request Flow

The `DispatcherServlet` is the heart of Spring MVC. Here is a more detailed flow diagram:

```
[Request] -> [DispatcherServlet]
                  |
                  v
1. [HandlerMapping] -> Finds the responsible Controller
                  |
                  v
2. [Controller] -> Executes logic, returns ModelAndView (Model data + View name)
                  |
                  v
3. [ViewResolver] -> Resolves the logical View name to a physical View (e.g., a JSP or Thymeleaf template)
                  |
                  v
4. [View] -> Renders the response using the Model data
                  |
                  v
[Response] <-<-<-<-<-<-
```

### 3.2 Advanced Request Mapping

`@RequestMapping` can be configured with more than just a path:
- `params`: Match based on the presence or value of a request parameter.
- `headers`: Match based on the presence or value of a request header.
- `consumes`: Match based on the `Content-Type` header of the request.
- `produces`: Match based on the `Accept` header of the request.

```java
@GetMapping(path = "/users", params = "version=2")
public UserV2 getUsersV2() { /* ... */ }

@PostMapping(path = "/import", consumes = "text/csv")
public ResponseEntity<Void> importCsvData(@RequestBody String csv) { /* ... */ }
```

### 3.3 Model, View, and `ModelAndView`

- **`Model`**: An interface that defines a holder for model attributes. You can add attributes to it, which are then accessible in the view.
- **`ModelAndView`**: A container for both the `Model` and the `View`, allowing a controller to return both in one value.

```java
@Controller
public class AppController {
    @GetMapping("/welcome")
    public String welcome(Model model) {
        model.addAttribute("message", "Hello from the controller!");
        return "welcome"; // Returns the logical view name
    }

    @GetMapping("/info")
    public ModelAndView info() {
        ModelAndView mav = new ModelAndView("infoPage"); // Set view name
        mav.addObject("details", "This is some detailed info."); // Add model data
        return mav;
    }
}
```

### 3.4 Interceptors (`HandlerInterceptor`)

Interceptors allow you to execute custom logic before, after, or upon completion of a request. They are useful for logging, authentication checks, etc.

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        // Before the handler method is executed
        log.info("Incoming request: {} {}", request.getMethod(), request.getRequestURI());
        return true; // continue execution
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        // After the handler method, before the view is rendered
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        // After the complete request has finished
    }
}

// You must register the interceptor
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private LoggingInterceptor loggingInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor);
    }
}
```
### 3.5 CORS Configuration
Cross-Origin Resource Sharing (CORS) is a security mechanism that browsers use to restrict cross-origin HTTP requests. You can configure it globally or per-controller/method.

```java
// Per-method
@CrossOrigin(origins = "http://example.com")
@GetMapping("/api/data")
public MyData getData() { /* ... */ }

// Global configuration
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:4200", "http://my-prod-ui.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true);
    }
}
```

---

## 4. Spring Data JPA

### 4.1 Repository Hierarchy

Spring Data provides a hierarchy of repository interfaces:
- **`Repository`**: Marker interface.
- **`CrudRepository`**: Provides basic CRUD functions (`save`, `findById`, `findAll`, `delete`, etc.).
- **`PagingAndSortingRepository`**: Extends `CrudRepository` to add methods for pagination and sorting.
- **`JpaRepository`**: Extends `PagingAndSortingRepository` to add JPA-specific methods like `flush()` and `saveAndFlush()`.

### 4.2 Pagination and Sorting

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.repository.JpaRepository;

public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findByCategory(String category, Pageable pageable);
}

@Service
public class ProductService {
    @Autowired
    private ProductRepository repo;

    public Page<Product> findProducts(int page, int size, String sortBy) {
        Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
        return repo.findAll(pageable);
    }
}
```

### 4.3 `@Transactional` and Propagation

The `propagation` attribute of `@Transactional` defines how transactions relate to each other.
- **`REQUIRED`**: (Default) Supports a current transaction; creates a new one if none exists.
- **`SUPPORTS`**: Supports a current transaction; executes non-transactionally if none exists.
- **`MANDATORY`**: Supports a current transaction; throws an exception if none exists.
- **`REQUIRES_NEW`**: Creates a new transaction, suspending the current transaction if one exists.
- **`NOT_SUPPORTED`**: Executes non-transactionally, suspending the current transaction if one exists.
- **`NEVER`**: Executes non-transactionally; throws an exception if a transaction exists.

```java
@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String message) {
        // This method will always run in its own transaction,
        // ensuring the audit log is saved even if the outer transaction fails.
    }
}
```

---

## 5. Spring Security

### 5.1 Detailed Security Configuration

Modern configuration uses a `SecurityFilterChain` bean to define firewall rules.

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true) // Enable method-level security
public class SecurityConfig {

    @Autowired
    private JwtAuthenticationFilter jwtAuthenticationFilter; // Your custom JWT filter
    @Autowired
    private UserDetailsService userDetailsService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // For stateless APIs
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class); // Add JWT filter

        return http.build();
    }
    
    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
    
    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### 5.2 `UserDetailsService`

This is a core interface for loading user-specific data. You must provide a bean that implements it to load user details from your database.

```java
@Service
public class MyUserDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
                .orElseThrow(() -> new UsernameNotFoundException("User not found"));

        return new org.springframework.security.core.userdetails.User(
            user.getUsername(), 
            user.getPassword(), 
            // map roles to GrantedAuthority
            user.getRoles().stream()
                .map(role -> new SimpleGrantedAuthority(role.getName()))
                .collect(Collectors.toList())
        );
    }
}
```

### 5.3 JWT (JSON Web Tokens) Implementation Sketch

A filter is needed to process JWTs on incoming requests.

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    // Inject your JWT service and UserDetailsService
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        final String authHeader = request.getHeader("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
        
        final String jwt = authHeader.substring(7);
        final String username = jwtService.extractUsername(jwt);
        
        // If user is not yet authenticated
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(username);
            if (jwtService.isTokenValid(jwt, userDetails)) {
                // Create authentication token and set it in the security context
                UsernamePasswordAuthenticationToken authToken = new UsernamePasswordAuthenticationToken(
                        userDetails, null, userDetails.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

---

## 6. Spring REST APIs

### 6.1 HATEOAS (Hypermedia as the Engine of Application State)

HATEOAS means that a REST API response should include links to other actions or resources, making the API self-discoverable. Spring HATEOAS makes this easy.

Add dependency: `<artifactId>spring-boot-starter-hateoas</artifactId>`

```java
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

@GetMapping("/{id}")
public EntityModel<Order> getOrder(@PathVariable Long id) {
    Order order = orderService.findOrderById(id);
    
    // Create an EntityModel to wrap the order and add links
    return EntityModel.of(order,
            linkTo(methodOn(OrderController.class).getOrder(id)).withSelfRel(),
            linkTo(methodOn(CustomerController.class).getCustomer(order.getCustomerId())).withRel("customer")
    );
}
```
The JSON response will now include a `_links` object:
```json
{
  "orderId": 123,
  "amount": 99.99,
  "_links": {
    "self": { "href": "/orders/123" },
    "customer": { "href": "/customers/45" }
  }
}
```

### 6.2 API Versioning Strategies

- **URI Versioning**: (e.g., `/api/v1/users`, `/api/v2/users`) - Simple, explicit, but pollutes URI space.
- **Query Parameter Versioning**: (e.g., `/api/users?version=1`) - Easy to implement, but can be messy.
- **Custom Header Versioning**: (e.g., `X-API-VERSION: 1`) - Keeps URIs clean, but less visible to clients.
- **Media Type Versioning (Content Negotiation)**: (e.g., `Accept: application/vnd.company.app-v1+json`) - Considered a "purer" REST approach, but can be complex.

### 6.3 `ResponseEntity` for Full Control

`ResponseEntity` gives you full control over the HTTP response.

```java
@PostMapping("/users")
public ResponseEntity<User> createUser(@RequestBody User user) {
    User createdUser = userService.create(user);
    
    URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(createdUser.getId())
            .toUri();
            
    return ResponseEntity.created(location).body(createdUser); // Returns 201 Created with Location header
}
```
