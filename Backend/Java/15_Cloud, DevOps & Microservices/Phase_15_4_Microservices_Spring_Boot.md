# Phase 15.4 — Microservices with Spring Boot

## What Are Microservices and When Should You Use Them?

A **microservice** is a small, independently deployable service that owns a single, well-defined business capability. An e-commerce system, for example, might be split into: `order-service`, `payment-service`, `inventory-service`, `notification-service`, and `user-service`. Each service has its own database, its own deployment pipeline, and communicates with others over a network (usually REST over HTTP or asynchronous messaging).

The key value proposition of microservices is **independent deployability and scalability**. If your payment processing needs to scale for Black Friday, you only scale the `payment-service` — not the entire monolith. If a bug exists in `notification-service`, you deploy a fix there without touching or risking the `order-service`.

However, microservices are NOT free. They introduce distributed systems complexity: network calls can fail, services need to discover each other, you now have distributed transactions to manage, and debugging requires correlating logs across multiple services. The general industry wisdom is: **start with a monolith, migrate to microservices when the pain points are real and clear**.

---

## Service Discovery — Eureka and Consul

In a microservice architecture, services are created and destroyed dynamically (especially with auto-scaling). You cannot hardcode IP addresses. **Service discovery** solves this by giving every service a logical name and maintaining a registry of which instances are currently healthy and available.

### Netflix Eureka with Spring Cloud

Eureka is the most common service registry in the Spring Cloud ecosystem. It has two components: the **Eureka Server** (the registry itself) and **Eureka Clients** (the microservices that register themselves).

```java
// EurekaServerApplication.java — the registry
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml for Eureka Server
server:
  port: 8761
spring:
  application:
    name: eureka-server
eureka:
  client:
    register-with-eureka: false   # Server doesn't register with itself
    fetch-registry: false
```

Each microservice becomes a Eureka Client by adding `spring-cloud-starter-netflix-eureka-client` and annotating its main class:

```java
// OrderServiceApplication.java — a microservice
@SpringBootApplication
@EnableDiscoveryClient
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

```yaml
# application.yml for order-service
spring:
  application:
    name: order-service   # This is the logical name used for discovery
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

Once registered, `order-service` can call `payment-service` using the logical name (via Spring Cloud LoadBalancer, described next), without knowing its actual IP address.

---

## API Gateway — Spring Cloud Gateway

An **API Gateway** is the single entry point for all external client requests. Instead of clients calling `order-service:8081`, `payment-service:8082`, and `user-service:8083` directly (which would require clients to know all service addresses and handle routing), they call the gateway (e.g., `api.example.com`) and the gateway routes requests to the appropriate downstream service.

The gateway also handles cross-cutting concerns: authentication/authorization, rate limiting, request/response transformation, CORS, and SSL termination. This keeps the individual microservices lean and focused on their core business logic.

```java
// GatewayApplication.java
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

```yaml
# application.yml for Spring Cloud Gateway
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        # Route 1: /api/orders/** → order-service
        - id: order-service-route
          uri: lb://order-service          # "lb://" uses Spring Cloud LoadBalancer
          predicates:
            - Path=/api/orders/**          # Match requests to this path
          filters:
            - StripPrefix=1                # Remove /api prefix before forwarding
            - AddRequestHeader=X-Request-Source, gateway

        # Route 2: /api/payments/** → payment-service
        - id: payment-service-route
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**
          filters:
            - StripPrefix=1

        # Route 3: All /api/users/** requests require JWT auth
        - id: user-service-route
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter    # Built-in rate limiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

---

## Load Balancing — Spring Cloud LoadBalancer

Spring Cloud LoadBalancer is the modern replacement for Netflix Ribbon. It is a **client-side load balancer**, meaning the service making the call (the client) decides which instance of the target service to call. It fetches the list of available instances from Eureka and distributes requests across them using a configurable strategy (round-robin by default).

```java
// Configure a load-balanced RestTemplate bean
@Configuration
public class AppConfig {
    
    @Bean
    @LoadBalanced   // This annotation makes the RestTemplate use service discovery
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

// In your service, use the logical service name as the hostname
@Service
public class OrderService {
    
    @Autowired
    private RestTemplate restTemplate;

    public PaymentResponse processPayment(PaymentRequest request) {
        // "payment-service" is resolved via Eureka, not hardcoded IP
        return restTemplate.postForObject(
            "http://payment-service/payments",
            request,
            PaymentResponse.class
        );
    }
}
```

For the modern reactive stack, use **WebClient** instead:

```java
@Bean
@LoadBalanced
public WebClient.Builder webClientBuilder() {
    return WebClient.builder();
}

// Usage
webClientBuilder.build()
    .get()
    .uri("http://inventory-service/inventory/{productId}", productId)
    .retrieve()
    .bodyToMono(InventoryResponse.class);
```

---

## Circuit Breaker — Resilience4j

In a distributed system, any network call can fail. A service can be slow (latency), overloaded, or completely down. Without protection, a slow downstream service can cause a cascade of failures — the calling service's threads all block waiting for responses, the thread pool fills up, and it becomes unresponsive too. This is called a **cascading failure**.

The **Circuit Breaker** pattern prevents this. It wraps each network call in a "circuit" that monitors the failure rate. When failures exceed a threshold, the circuit "opens" — subsequent calls immediately fail-fast (return an error or a fallback) without even attempting the network call. This gives the downstream service time to recover. After a configured wait period, the circuit enters a "half-open" state and allows a few test requests through. If those succeed, the circuit closes (normal operation resumes).

```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

```yaml
# application.yml — Resilience4j configuration
resilience4j:
  circuitbreaker:
    instances:
      payment-service:                      # Named circuit breaker instance
        registerHealthIndicator: true
        slidingWindowSize: 10               # Evaluate the last 10 calls
        minimumNumberOfCalls: 5             # Need at least 5 calls before tripping
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 30s        # Stay open for 30 seconds
        failureRateThreshold: 50            # Open circuit if >50% of calls fail
        slowCallRateThreshold: 80           # Also open if >80% of calls are slow
        slowCallDurationThreshold: 3s       # "Slow" means >3 seconds

  retry:
    instances:
      payment-service:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2     # Wait 500ms, then 1s, then 2s

  ratelimiter:
    instances:
      payment-service:
        limitForPeriod: 100                 # Max 100 calls
        limitRefreshPeriod: 1s              # Per second
        timeoutDuration: 500ms
```

```java
@Service
public class OrderService {

    private final PaymentClient paymentClient;

    // @CircuitBreaker — wraps the method in a circuit breaker
    // fallbackMethod is called when the circuit is open OR the call throws an exception
    @CircuitBreaker(name = "payment-service", fallbackMethod = "paymentFallback")
    @Retry(name = "payment-service")        // Retry before the circuit breaker counts it as failure
    @RateLimiter(name = "payment-service")
    public PaymentResponse processPayment(PaymentRequest request) {
        return paymentClient.process(request);
    }

    // Fallback must have the same return type and parameters + a Throwable parameter
    public PaymentResponse paymentFallback(PaymentRequest request, Throwable ex) {
        log.warn("Payment service unavailable, using fallback. Error: {}", ex.getMessage());
        // Return a default response, queue the request for later processing, etc.
        return PaymentResponse.pending("Payment queued for processing");
    }
}
```

In addition to the circuit breaker, Resilience4j provides:
- **`@Retry`** — automatically retry a failed call a configurable number of times with backoff
- **`@RateLimiter`** — limit how many times a method can be called per time period
- **`@Bulkhead`** — limit concurrent calls to a service (prevents thread pool saturation)
- **`@TimeLimiter`** — timeout for async calls

---

## Distributed Configuration — Spring Cloud Config Server

When you have 15 microservices, managing `application.yml` inside each service's JAR becomes unmaintainable. The **Config Server** externalizes all configuration into a central Git repository. Services fetch their config at startup (and optionally refresh it at runtime without restarting).

```java
// ConfigServerApplication.java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

```yaml
# application.yml for Config Server
server:
  port: 8888
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/myorg/config-repo  # Git repo holding all configs
          default-label: main
          search-paths: '{application}'    # Each service's config is in its own folder
```

The Git repository structure mirrors your service names:
```
config-repo/
├── order-service/
│   ├── application.yml          # Common config for all environments
│   ├── application-dev.yml      # Dev-specific overrides
│   └── application-prod.yml     # Prod-specific overrides
├── payment-service/
│   └── application.yml
└── application.yml              # Shared config for ALL services
```

Each microservice connects to the Config Server using `spring-cloud-starter-config`:

```yaml
# bootstrap.yml (loaded BEFORE application.yml)
spring:
  application:
    name: order-service
  config:
    import: "configserver:http://config-server:8888"
  profiles:
    active: prod
```

---

## Distributed Tracing — Micrometer Tracing with Zipkin

When a user request flows through `api-gateway` → `order-service` → `payment-service` → `inventory-service`, a failure or slowness in one service is hard to diagnose. Which service was slow? Distributed tracing assigns a **trace ID** to the original request and a **span ID** to each service's work. By correlating spans by trace ID, you can visualize the entire call chain and see exactly where time was spent.

```xml
<!-- pom.xml — Spring Boot 3+ uses Micrometer Tracing -->
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
  <groupId>io.zipkin.reporter2</groupId>
  <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0        # Sample 100% of requests (use 0.1 for 10% in production)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

With this configuration, every log line automatically includes the `traceId` and `spanId`:

```
2025-03-01 10:23:45 INFO [order-service,3d2f1a4b8c,9e1f2d3a] Processing order #12345
```

The `3d2f1a4b8c` is the trace ID — the same ID will appear in the logs of `payment-service` and `inventory-service` for the same user request, making it trivial to correlate the entire distributed transaction.

---

## The Saga Pattern — Distributed Transactions

In a monolith, you can wrap multiple database operations in a single ACID transaction. In microservices, each service has its own database, so there is no single transaction manager. The **Saga pattern** manages multi-service workflows by breaking them into a sequence of local transactions, where each step publishes an event or sends a command to trigger the next step.

If any step fails, previously completed steps must be **compensated** (rolled back using a compensation transaction — not a true database rollback, but a business operation that reverses the effect).

**Choreography Saga** — services react to events autonomously with no central coordinator.

```
OrderService publishes OrderCreated event
    ↓
PaymentService listens, processes payment, publishes PaymentCompleted
    ↓
InventoryService listens, reserves items, publishes InventoryReserved
    ↓
OrderService listens, marks order as CONFIRMED

If PaymentService fails:
PaymentService publishes PaymentFailed
    ↓
OrderService listens, marks order as CANCELLED
```

**Orchestration Saga** — a central coordinator (the Saga Orchestrator) explicitly commands each service step-by-step. This is easier to understand and debug but introduces a centralized component.

```java
// Saga Orchestrator for order creation
@Component
public class CreateOrderSaga {

    public void execute(Order order) {
        try {
            // Step 1: Reserve inventory
            inventoryService.reserve(order.getItems());
            
            // Step 2: Process payment
            paymentService.charge(order.getCustomerId(), order.getTotal());
            
            // Step 3: Confirm order
            orderService.confirm(order.getId());
            
        } catch (PaymentFailedException e) {
            // Compensate: release inventory reservation
            inventoryService.release(order.getItems());
            orderService.cancel(order.getId());
        }
    }
}
```

---

## CQRS — Command Query Responsibility Segregation

**CQRS** separates the **write model** (commands that change state) from the **read model** (queries that return data). The write model is optimized for consistency and business rule enforcement. The read model is optimized for query performance, often as a denormalized view specifically shaped for the UI's needs.

```java
// Command side — optimized for writes, enforces business rules
@Service
public class OrderCommandService {
    
    public OrderId createOrder(CreateOrderCommand command) {
        // Validate business rules
        if (!inventoryService.isAvailable(command.getItems())) {
            throw new InsufficientInventoryException();
        }
        Order order = Order.create(command);   // Domain object, enforces invariants
        orderRepository.save(order);
        eventBus.publish(new OrderCreatedEvent(order));  // Trigger read model update
        return order.getId();
    }
}

// Query side — optimized for reads, potentially a different database/schema
@Service
public class OrderQueryService {
    
    // Uses a read-optimized view: flat table with joined customer/product data
    public List<OrderSummary> getOrdersByCustomer(CustomerId customerId) {
        return orderReadRepository.findSummariesByCustomer(customerId);
    }
}
```

CQRS is powerful but adds significant complexity. Use it only when you have genuinely different read and write characteristics — for example, when your reads far outnumber writes and require complex joins that are impractical on the normalized write model.

---

## Best Practices Summary

**Design for failure.** Every remote call can fail. Use circuit breakers, retries with exponential backoff, and fallbacks from the start — not as an afterthought.

**Each service owns its data.** Services must never directly access another service's database. All inter-service communication goes through the service's API. This is the most important rule in microservices architecture.

**Prefer asynchronous communication** for non-time-critical workflows. Synchronous HTTP calls chain services together and amplify latency and failure. Event-driven communication (Kafka, RabbitMQ) decouples services and improves resilience.

**Version your APIs carefully.** Services evolve independently, so their APIs must be backward-compatible. Use semantic versioning in your URL paths (`/v1/orders`, `/v2/orders`) and design for additive changes rather than breaking changes.

**Have a correlation ID strategy.** Every request entering the system should be assigned a unique ID at the gateway and propagated through all service calls via headers (e.g., `X-Correlation-ID`). Include it in every log line for traceability.

**Don't start with microservices.** If your team is small, the domain is not fully understood, or your traffic does not justify it, a well-structured monolith is a better choice. Microservices are an optimization for scale — organizational, technical, and operational.
