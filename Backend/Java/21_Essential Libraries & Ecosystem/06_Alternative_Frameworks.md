# 21.6 — Alternative Java Frameworks

> **Frameworks covered:** Quarkus · Micronaut · Vert.x · Helidon · Dropwizard · Play Framework
>
> Spring Boot is the default Java application framework for a reason — it is mature, flexible, and has the largest ecosystem. But it has real tradeoffs: it relies heavily on runtime reflection and classpath scanning, which means slow startup times and significant memory footprints. In the cloud-native era, where functions wake up cold and containers must scale to zero, these tradeoffs matter. The frameworks in this section were all built to address specific shortcomings of Spring Boot — faster startup, lower memory, better cloud-native fit, reactive foundations, or simplified microservice packaging. Understanding them makes you a better architect, even if you build everything in Spring.

---

## ☁️ Quarkus — Supersonic Subatomic Java

Quarkus is a Kubernetes-native Java framework developed by Red Hat, designed from the ground up for containerised environments. Its headline feature is **GraalVM Native Image support** — the ability to compile a Java application ahead-of-time into a native binary that starts in milliseconds and uses a fraction of the heap memory of a JVM-based application. This directly addresses the single biggest complaint about Java in serverless and container environments: slow cold starts.

### Why Quarkus Achieves What Spring Cannot (Easily)

Spring Boot's power comes from doing things at runtime: scanning the classpath, creating proxies, configuring beans based on what it finds. This flexibility is invaluable for large applications, but it makes ahead-of-time compilation extremely difficult because native image compilation requires knowing all code paths at build time. Quarkus flips this model: it does as much work as possible **at build time** — classpath scanning, dependency injection wiring, configuration parsing — so that the resulting application has almost nothing to do at startup.

The result is dramatic. A typical Spring Boot application starts in 3–5 seconds and uses 200–400 MB of heap memory at idle. The equivalent Quarkus JVM application starts in under a second and uses ~100 MB. The Quarkus native binary starts in under 50 milliseconds and uses ~30–50 MB of RSS memory.

### 21.6.1 Quarkus — Building an Application

```java
// pom.xml: use the Quarkus BOM and Maven plugin for build-time processing
// <artifactId>quarkus-bom</artifactId> + <artifactId>quarkus-maven-plugin</artifactId>

// ── REST endpoint — uses Jakarta RESTful Web Services (JAX-RS) ─────────────────
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;
import jakarta.inject.Inject;

@Path("/api/products")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class ProductResource {

    @Inject
    ProductService productService;   // Quarkus uses ArC, a CDI-based DI container

    @GET
    @Path("/{id}")
    public Response getProduct(@PathParam("id") Long id) {
        return productService.findById(id)
            .map(p -> Response.ok(p).build())
            .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }

    @GET
    public List<ProductDto> listProducts(@QueryParam("category") String category) {
        return productService.findByCategory(category);
    }

    @POST
    @ResponseStatus(201)
    public ProductDto createProduct(@Valid CreateProductRequest request) {
        return productService.create(request);
    }
}

// ── Reactive REST endpoint — Quarkus RESTEasy Reactive ────────────────────────
// Add: quarkus-resteasy-reactive and quarkus-resteasy-reactive-jackson
@Path("/api/users")
public class UserResource {

    @Inject ReactiveUserRepository repository;

    @GET
    @Path("/{id}")
    public Uni<Response> getUser(@PathParam("id") Long id) {
        return repository.findById(id)
            .onItem().transform(u -> Response.ok(u).build())
            .onItem().ifNull().continueWith(Response.status(404).build());
    }

    // Returns a stream of users as Server-Sent Events
    @GET
    @Produces(MediaType.SERVER_SENT_EVENTS)
    public Multi<User> streamUsers() {
        return repository.streamAll();
    }
}

// ── Dependency Injection — ArC (Quarkus CDI) ─────────────────────────────────
@ApplicationScoped   // Singleton for the application lifetime — equivalent to Spring's @Service
public class ProductService {
    @Inject ProductRepository repository;
    @Inject EventBus eventBus;           // Vert.x EventBus for async messaging

    @Transactional
    public ProductDto create(CreateProductRequest request) {
        Product product = new Product(request.name(), request.price());
        repository.persist(product);
        eventBus.send("product.created", product.getId());
        return ProductDto.from(product);
    }
}

@RequestScoped    // New instance per HTTP request
public class RequestContextService { ... }

@Dependent        // New instance per injection point (prototype scope)
public class AuditLogger { ... }

// ── Configuration — application.properties ────────────────────────────────────
// quarkus.http.port=8080
// quarkus.datasource.db-kind=postgresql
// quarkus.datasource.username=${DB_USER}
// quarkus.datasource.password=${DB_PASS}
// quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/mydb
// quarkus.hibernate-orm.database.generation=update

// Injecting config values
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class PaymentConfig {
    @ConfigProperty(name = "payment.api.url")
    String apiUrl;

    @ConfigProperty(name = "payment.timeout", defaultValue = "30")
    int timeoutSeconds;
}

// ── Panache ORM — dramatically simplified Hibernate ────────────────────────────
// Panache wraps Hibernate ORM to eliminate DAO boilerplate using the Active Record
// or Repository pattern.

// Active Record pattern: entity has persistence methods directly on it
@Entity
public class Product extends PanacheEntity {  // PanacheEntity provides id, persist(), delete() etc.
    public String name;
    public BigDecimal price;
    public String category;

    // Static finder methods defined right on the entity
    public static List<Product> findByCategory(String category) {
        return list("category", category);
    }

    public static Optional<Product> findByName(String name) {
        return find("name", name).firstResultOptional();
    }
}

// Usage: no repository class needed
Product p = new Product();
p.name = "Widget";
p.price = BigDecimal.valueOf(9.99);
p.persist();   // Saves to database

List<Product> widgets = Product.findByCategory("widgets");
Product.deleteById(42L);

// ── Building native image ─────────────────────────────────────────────────────
// mvn package -Pnative                    (requires GraalVM or a Docker container)
// mvn package -Pnative -Dquarkus.native.container-build=true  (uses Docker — no GraalVM required locally)
//
// Result: target/my-app-1.0-runner  (native binary, ~30MB, starts in <50ms)
```

> **When to choose Quarkus:** Serverless functions (AWS Lambda, Azure Functions) where cold start time determines user experience. Kubernetes environments where you're paying for memory and want to run more pods. Greenfield microservices where you want native image support from day one. Quarkus also supports a Dev Mode (`mvn quarkus:dev`) with live reload that is faster than Spring Boot DevTools.

---

## 🔬 Micronaut — Compile-Time Everything

Micronaut, created by the Groovy and Grails team at Object Computing, takes a different approach from both Spring Boot and Quarkus. Rather than doing work at runtime (Spring) or at build time from a JVM application (Quarkus), Micronaut processes dependency injection, AOP, and configuration **entirely at compile time** using annotation processors. There is no runtime reflection at all. The result is fast startup, low memory, and excellent GraalVM compatibility — without needing GraalVM to achieve competitive JVM startup times.

### 21.6.2 Micronaut — Core Concepts

```java
// ── HTTP Controller ────────────────────────────────────────────────────────────
import io.micronaut.http.annotation.*;
import io.micronaut.http.HttpResponse;
import jakarta.inject.Inject;

@Controller("/api/orders")
public class OrderController {

    @Inject OrderService orderService;

    @Get("/{id}")
    public HttpResponse<OrderDto> getOrder(Long id) {
        return orderService.findById(id)
            .map(HttpResponse::ok)
            .orElse(HttpResponse.notFound());
    }

    @Post
    public HttpResponse<OrderDto> createOrder(@Body @Valid CreateOrderRequest request) {
        OrderDto created = orderService.create(request);
        return HttpResponse.created(created);
    }
}

// ── Dependency Injection — compile-time generated ─────────────────────────────
// At compile time, Micronaut's annotation processor generates concrete factory
// and injection classes. There is NO runtime classpath scanning or reflection.

@Singleton  // Singleton scope — analogous to Spring's @Service + @Singleton
public class OrderService {
    private final OrderRepository repository;
    private final EventPublisher publisher;

    // Constructor injection — the generated factory calls this constructor directly
    public OrderService(OrderRepository repository, EventPublisher publisher) {
        this.repository = repository;
        this.publisher = publisher;
    }
}

// ── Reactive HTTP Client — declarative, like Feign but generated at compile time
@Client("user-service")  // Looks up "user-service" from service discovery or config
public interface UserServiceClient {
    @Get("/api/users/{id}")
    Mono<UserDto> findUser(Long id);          // Reactive return type

    @Post("/api/users")
    Single<UserDto> createUser(@Body CreateUserRequest request);  // RxJava Single works too
}

// ── Configuration injection — type-safe, validated at startup ─────────────────
@ConfigurationProperties("payment")  // Reads properties prefixed with "payment."
public interface PaymentConfig {
    String getApiUrl();
    @Bindable(defaultValue = "30")
    int getTimeoutSeconds();
    boolean isRetryEnabled();
}
// application.yml:
// payment:
//   api-url: https://api.payment.com
//   timeout-seconds: 45
//   retry-enabled: true

// ── GraalVM Native Image ───────────────────────────────────────────────────────
// Because Micronaut uses no runtime reflection, native image compilation
// works with no special configuration for most application code.
// Simply run: mn create-app myapp --features=graalvm
// Then:       ./gradlew nativeCompile
// Result: application binary with ~50ms startup, ~30MB RSS
```

> **Micronaut vs. Quarkus:** Both achieve fast startup and GraalVM compatibility through build-time processing. Micronaut's DI is fully compile-time with no runtime component (even on the JVM, startup is sub-second without GraalVM). Quarkus does more build-time work than Spring but still has some runtime initialization on the JVM — it only truly shines in native image mode. Micronaut has a more familiar Spring-like API. Quarkus has deeper integration with the Kubernetes/OpenShift ecosystem and Red Hat support. For AWS Lambda, both are excellent choices.

---

## ⚡ Vert.x — The Reactive Toolkit

Vert.x, maintained by Eclipse Foundation, is not a framework in the traditional sense — it is a **toolkit** for building reactive, event-driven applications on the JVM. It uses an event loop model (similar to Node.js) where I/O events are processed by one of a small number of event loop threads, and handlers run in response to events without blocking.

Vert.x is polyglot (Java, Kotlin, Groovy, JavaScript, Ruby) and is used both as a standalone application framework and as the reactive engine underlying other frameworks — notably Quarkus uses Vert.x's event loop and EventBus internally.

### 21.6.3 Vert.x — Core Concepts

```java
import io.vertx.core.*;
import io.vertx.core.http.*;
import io.vertx.ext.web.*;
import io.vertx.ext.web.handler.*;

// ── Verticle: the basic deployment unit in Vert.x ─────────────────────────────
// A Verticle is like a microservice within a microservice — an independent unit
// of processing with its own event loop thread. You deploy multiple verticles.
public class OrderVerticle extends AbstractVerticle {

    @Override
    public void start(Promise<Void> startPromise) {
        Router router = Router.router(vertx);

        // Route with Mutiny-style async handler
        router.get("/api/orders/:id").handler(this::getOrder);
        router.post("/api/orders").handler(BodyHandler.create()).handler(this::createOrder);

        vertx.createHttpServer()
            .requestHandler(router)
            .listen(8080)
            .onSuccess(server -> {
                System.out.println("HTTP server started on port " + server.actualPort());
                startPromise.complete();
            })
            .onFailure(startPromise::fail);
    }

    private void getOrder(RoutingContext ctx) {
        Long id = Long.parseLong(ctx.pathParam("id"));
        // vertx.executeBlocking: run blocking code off the event loop
        // Returns a Future<T> — the Vert.x async type
        vertx.<OrderDto>executeBlocking(promise -> {
            OrderDto order = orderService.findByIdBlocking(id);  // Blocking DB call
            promise.complete(order);
        }).onSuccess(order -> ctx.response()
                .putHeader("Content-Type", "application/json")
                .end(Json.encode(order)))
          .onFailure(err -> ctx.response()
                .setStatusCode(404).end("Not found"));
    }

    private void createOrder(RoutingContext ctx) {
        CreateOrderRequest request = ctx.body().asPojo(CreateOrderRequest.class);
        // Using Vert.x EventBus for async communication between Verticles
        vertx.eventBus().<Long>request("orders.create", request)
            .onSuccess(message -> ctx.response()
                .setStatusCode(201)
                .end(message.body().toString()))
            .onFailure(err -> ctx.response()
                .setStatusCode(500).end("Internal error"));
    }
}

// ── EventBus — the nervous system of a Vert.x application ────────────────────
// Verticles communicate without direct method calls — via message passing on the EventBus.
// This decouples components and naturally supports distribution (messages can cross JVMs
// in a Vert.x cluster).

// Sender: sends a message to an address
vertx.eventBus().send("email.send", emailPayload);          // Fire and forget
vertx.eventBus().request("user.create", userData)           // Request-reply pattern
    .onSuccess(reply -> log.info("User created: {}", reply.body()));

// Consumer: processes messages on an address
vertx.eventBus().consumer("email.send", message -> {
    EmailPayload payload = (EmailPayload) message.body();
    emailService.send(payload);
    // No reply needed for fire-and-forget
});

vertx.eventBus().consumer("user.create", message -> {
    UserData data = (UserData) message.body();
    User created = userService.create(data);
    message.reply(created.getId());  // Reply with the new user's ID
});

// ── Deploying Verticles ────────────────────────────────────────────────────────
Vertx vertx = Vertx.vertx();
vertx.deployVerticle(new OrderVerticle(), new DeploymentOptions().setInstances(4));
// 4 instances × (Vert.x default: 2× CPU cores event loop threads) = 4 independent OrderVerticles
// Each handles its own event loop without sharing state — no thread synchronization needed
```

> **When Vert.x makes sense:** Real-time applications (chat, gaming, live dashboards), high-throughput proxies and gateways, and systems where you need fine-grained control over the event loop. Vert.x requires a more significant mindset shift than Spring or Quarkus — the "never block the event loop" rule is strict and violations cause subtle, hard-to-debug performance problems. For most business applications, Quarkus with Mutiny provides a gentler on-ramp to reactive programming while internally leveraging Vert.x's engine.

---

## 🏺 Helidon — Oracle's Lightweight Microservice Framework

Helidon is Oracle's open-source microservices framework, available in two flavours: **Helidon SE** (functional, reactive, no magic — write everything explicitly) and **Helidon MP** (MicroProfile specification — more familiar for Java EE developers, CDI-based, annotation-driven).

### 21.6.4 Helidon MP — MicroProfile Style

```java
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.*;

// ── Helidon MP REST endpoint (JAX-RS + CDI) ───────────────────────────────────
@Path("/api/products")
@ApplicationScoped
public class ProductResource {

    @Inject ProductService service;

    @GET @Path("/{id}")
    @Produces(MediaType.APPLICATION_JSON)
    public Response getProduct(@PathParam("id") Long id) {
        return service.findById(id)
            .map(Response::ok)
            .orElse(Response.status(404))
            .build();
    }
}

// ── MicroProfile Config — portable configuration ──────────────────────────────
@ApplicationScoped
public class AppConfig {
    @Inject
    @ConfigProperty(name = "database.url")
    private String dbUrl;

    @Inject
    @ConfigProperty(name = "feature.flag.new-checkout", defaultValue = "false")
    private boolean newCheckoutEnabled;
}

// ── MicroProfile Health — built-in /health endpoint ──────────────────────────
import org.eclipse.microprofile.health.*;

@Liveness   // /health/live — is the application alive?
@ApplicationScoped
public class DatabaseHealthCheck implements HealthCheck {
    @Inject DataSource dataSource;

    @Override
    public HealthCheckResponse call() {
        try (Connection conn = dataSource.getConnection()) {
            return HealthCheckResponse.named("database")
                .withData("connected", true)
                .up()
                .build();
        } catch (SQLException e) {
            return HealthCheckResponse.named("database")
                .withData("error", e.getMessage())
                .down()
                .build();
        }
    }
}

// ── MicroProfile Fault Tolerance — resilience annotations ─────────────────────
import org.eclipse.microprofile.faulttolerance.*;

@ApplicationScoped
public class UserServiceClient {

    @Retry(maxRetries = 3, delay = 500, delayUnit = ChronoUnit.MILLIS)
    @Timeout(value = 5, unit = ChronoUnit.SECONDS)
    @CircuitBreaker(requestVolumeThreshold = 10, failureRatio = 0.5, delay = 5000)
    @Fallback(fallbackMethod = "getUserFallback")
    public UserDto getUser(Long id) {
        return remoteUserService.fetchUser(id);
    }

    private UserDto getUserFallback(Long id) {
        return UserDto.anonymous();
    }
}
```

---

## 🧰 Dropwizard — Opinionated, Production-Ready from Day One

Dropwizard was the framework that popularised the "fat JAR" model — packaging your application and all dependencies into a single executable JAR. Released in 2011 by Yammer, it predates Spring Boot's fat JAR support and introduced many patterns (embedded Jetty, Jersey JAX-RS, Jackson JSON, Metrics, Hibernate Validator, Liquibase) that Spring Boot later adopted and expanded.

Dropwizard is deliberate about being opinionated: it doesn't try to be all things to all people. It uses Jersey for REST, Jackson for JSON, Hibernate for ORM, Metrics for monitoring, and JDBI or jOOQ for database access. There is one way to do each thing. This makes it fast to get started and easy to onboard new developers.

### 21.6.5 Dropwizard — Application Structure

```java
// Every Dropwizard app has three top-level components:
// 1. Configuration class — maps from YAML config file to typed Java object
// 2. Application class — wires everything together and registers resources
// 3. Resource classes — JAX-RS annotated REST endpoints

// ── Configuration ─────────────────────────────────────────────────────────────
import io.dropwizard.core.Configuration;
import com.fasterxml.jackson.annotation.*;

public class AppConfig extends Configuration {
    @NotEmpty
    private String apiKey;

    @Valid @NotNull
    private DataSourceFactory database = new DataSourceFactory();

    @JsonProperty
    public String getApiKey() { return apiKey; }

    @JsonProperty
    public DataSourceFactory getDatabase() { return database; }
}

// config.yml (maps to AppConfig above):
// server:
//   applicationConnectors:
//     - type: http
//       port: 8080
// apiKey: ${API_KEY}
// database:
//   driverClass: org.postgresql.Driver
//   url: jdbc:postgresql://localhost:5432/mydb
//   user: ${DB_USER}
//   password: ${DB_PASS}

// ── Application class — the main entry point ──────────────────────────────────
import io.dropwizard.core.Application;
import io.dropwizard.core.setup.*;

public class OrderApplication extends Application<AppConfig> {

    public static void main(String[] args) throws Exception {
        new OrderApplication().run(args);
    }

    @Override
    public void run(AppConfig config, Environment env) {
        // Set up database connection pool using HikariCP
        JdbiFactory factory = new JdbiFactory();
        Jdbi jdbi = factory.build(env, config.getDatabase(), "orders-db");

        // Create DAOs and services
        OrderDao orderDao = jdbi.onDemand(OrderDao.class);
        OrderService orderService = new OrderService(orderDao);

        // Register REST resources with Jersey
        env.jersey().register(new OrderResource(orderService));
        env.jersey().register(new HealthCheckResource());

        // Register health checks — appears at /healthcheck endpoint
        env.healthChecks().register("database", new DatabaseHealthCheck(config.getDatabase()));

        // Register Jackson module for java.time support
        env.getObjectMapper().registerModule(new JavaTimeModule());
    }
}

// ── Resource class — JAX-RS endpoint ─────────────────────────────────────────
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.*;

@Path("/api/orders")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class OrderResource {
    private final OrderService service;

    public OrderResource(OrderService service) {
        this.service = service;
    }

    @GET @Path("/{id}")
    public Response getOrder(@PathParam("id") @Min(1) long id) {
        return service.findById(id)
            .map(order -> Response.ok(order).build())
            .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }

    @POST
    public Response createOrder(@Valid CreateOrderRequest request) {
        OrderDto created = service.create(request);
        return Response.created(URI.create("/api/orders/" + created.id()))
            .entity(created)
            .build();
    }
}

// Run: java -jar app.jar server config.yml
// Dropwizard starts on two ports: 8080 (application) and 8081 (admin/metrics)
```

> **When Dropwizard shines:** Small-to-medium REST services where you want a simple, opinionated stack with built-in production features (metrics, health checks, connection pooling) and minimal magic. It is also an excellent choice for teams that want to understand exactly what their framework does — there is very little auto-configuration.

---

## 🎮 Play Framework — Full-Stack MVC for Scala and Java

Play Framework is a reactive, stateless MVC framework that runs on Akka (for Scala) or Play's own async layer (for Java). It uses SBT or Maven for build, has built-in hot-reload in development mode, and follows a convention-over-configuration model similar to Ruby on Rails.

Play is less commonly chosen for new greenfield Java microservices today — Spring Boot, Quarkus, and Micronaut have largely captured that space. But Play is still widely used in organisations with existing Play 2.x codebases, and understanding it is valuable when you encounter it.

### 21.6.6 Play Framework — Java Application Structure

```java
// routes file (conf/routes) — defines URL patterns mapped to controller actions
// GET    /api/users/:id     controllers.UserController.getUser(id: Long)
// POST   /api/users         controllers.UserController.createUser()
// GET    /                  controllers.HomeController.index()

// ── Controller — handles HTTP requests ────────────────────────────────────────
import play.mvc.*;
import play.libs.concurrent.HttpExecutionContext;
import javax.inject.Inject;
import java.util.concurrent.CompletionStage;

public class UserController extends Controller {

    private final UserRepository repository;
    private final HttpExecutionContext ec;
    private final play.libs.Json json;  // Play includes Jackson via its Json helper

    @Inject
    public UserController(UserRepository repository, HttpExecutionContext ec) {
        this.repository = repository;
        this.ec = ec;
    }

    // Synchronous action (blocking — acceptable for simple cases)
    public Result getUser(Long id) {
        return repository.findById(id)
            .map(user -> ok(Json.toJson(user)))
            .orElse(notFound());
    }

    // Asynchronous action — returns CompletionStage for non-blocking I/O
    public CompletionStage<Result> createUser(Http.Request request) {
        JsonNode body = request.body().asJson();
        if (body == null) return completedFuture(badRequest("Expecting JSON"));

        return repository.create(Json.fromJson(body, CreateUserRequest.class))
            .thenApplyAsync(user -> created(Json.toJson(user)), ec.current());
    }

    // ── Response helpers provided by Controller base class ────────────────────
    // ok(content)         — 200 OK
    // created(content)    — 201 Created
    // badRequest(content) — 400 Bad Request
    // notFound()          — 404 Not Found
    // forbidden()         — 403 Forbidden
    // redirect(url)       — 302 Redirect
}

// ── Dependency Injection — Google Guice ───────────────────────────────────────
// Play uses Guice by default. Define bindings in a Module class.
import play.api.inject.*;
import com.google.inject.*;

public class AppModule extends AbstractModule {
    @Override
    protected void configure() {
        bind(UserRepository.class).to(PostgresUserRepository.class);
        bind(EmailService.class).to(SmtpEmailService.class);
    }
}

// ── Configuration — Typesafe Config (HOCON format) ────────────────────────────
// conf/application.conf:
// play.http.secret.key = "changeme"
// db.default.driver    = org.postgresql.Driver
// db.default.url       = "jdbc:postgresql://localhost/mydb"
// my.feature.enabled   = true

// In a controller or service:
import play.api.Configuration;
@Inject Configuration config;
boolean featureEnabled = config.getOptional("my.feature.enabled", Boolean.class).orElse(false);

// ── Running in development ─────────────────────────────────────────────────────
// sbt run       → starts with hot-reload (changes to .java files are reloaded automatically)
// sbt dist      → builds a distribution ZIP with startup scripts
// sbt assembly  → builds a fat JAR (with sbt-assembly plugin)
```

---

## ⚖️ Framework Decision Guide

Understanding the tradeoffs between frameworks is essential for making good architecture decisions. The following guidance is meant to help you reason through the choice clearly.

**Spring Boot** remains the default choice for most Java enterprise applications. It has the richest ecosystem, the most comprehensive documentation, the largest community, and the most integrations. Unless you have a specific reason to choose otherwise, start with Spring Boot.

**Quarkus** is the right choice when startup time and memory footprint are primary concerns — serverless functions, densely-packed Kubernetes clusters, or environments where you want native image compilation as a first-class outcome. Its developer experience (continuous testing, dev services that auto-start databases and message brokers) is outstanding.

**Micronaut** is the best choice when you want compile-time DI and fast JVM startup without needing native image compilation. It is also an excellent choice for libraries and frameworks that will be embedded in other applications, because its lack of runtime reflection makes it predictable and lightweight.

**Vert.x** suits systems where maximum reactive throughput and fine-grained event loop control are required — real-time features, proxies, message routers. It is also the right choice when you want a toolkit rather than a framework — minimum magic, maximum control.

**Helidon** is the natural choice for teams coming from a Java EE or MicroProfile background who want a standards-based (JAX-RS, CDI, MicroProfile) approach to microservices without the full weight of WildFly or GlassFish.

**Dropwizard** is the best choice for simple, opinionated REST services where you value clarity and convention over flexibility, and where the built-in production-readiness features (metrics, health checks, connection pool management) are important from day one.

**Play Framework** is the framework to use if you are maintaining an existing Play application or working in a Scala-heavy organisation where Play's Scala support is valued. For new greenfield Java microservices, the alternatives above are generally more suitable.

---

## 💡 Phase 21.6 — Key Takeaways

The existence of these frameworks reflects a mature recognition that no single tool fits every problem. Spring Boot's flexibility and ecosystem breadth make it the right default. Quarkus and Micronaut demonstrate that fast startup and low memory are achievable in Java — the JVM's reputation for "slow and heavy" is no longer deserved with modern frameworks. Vert.x shows that Java can compete with Node.js on reactive throughput. Helidon keeps the standards-based Java EE spirit alive in the microservices era. Dropwizard proves that opinionated simplicity is a legitimate architectural value.

As a Java professional, understanding this landscape means you can evaluate a new project's requirements — startup time, throughput profile, team familiarity, operational environment — and make an informed recommendation rather than defaulting to whatever you've used before. Even if you build everything in Spring Boot, understanding why Quarkus compiles to a native binary in 50 milliseconds, why Micronaut avoids runtime reflection, and why Vert.x never blocks the event loop makes you a more thoughtful architect and a more effective Spring developer.
