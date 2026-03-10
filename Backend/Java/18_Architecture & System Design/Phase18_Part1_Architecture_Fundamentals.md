# Phase 18 — Part 1: Software Architecture Fundamentals

> Software architecture is the high-level structure of a system — the decisions that are hard to change later, that determine how components are organized, how they communicate, and how the system will behave under growth and failure. Understanding architectural styles and distributed systems theory is what separates a senior engineer from someone who just writes features.

---

## 1. Architectural Styles

An architectural style is a named, reusable pattern for organizing a system at the highest level. Choosing the wrong style at the start of a project can cost years of painful refactoring. Choosing the right one requires understanding what each style optimizes for and what trade-offs it forces you to accept.

---

### 1.1 Monolithic Architecture

A **monolith** is a single deployable unit where all application concerns — UI, business logic, data access — live in one codebase and are deployed together as one process.

**Theory:** Monoliths have earned a bad reputation, but they are often the correct choice for new products. A well-structured monolith is easier to develop, test, debug, and deploy than any distributed alternative. The problems with monoliths only emerge at scale: when the team grows large enough that coordination becomes a bottleneck, or when independent parts of the system need to scale at different rates.

Think of a monolith as a single Java application with separate packages for each concern:

```
com.example.ecommerce
├── order/
│   ├── OrderController.java
│   ├── OrderService.java
│   └── OrderRepository.java
├── inventory/
│   ├── InventoryService.java
│   └── InventoryRepository.java
├── payment/
│   ├── PaymentService.java
│   └── PaymentRepository.java
└── notification/
    └── NotificationService.java
```

```java
// In a monolith, an OrderService can directly call InventoryService
// This is a simple, synchronous in-process method call — no network, no serialization
@Service
public class OrderService {
    private final InventoryService inventoryService;  // Direct dependency injection
    private final PaymentService paymentService;
    private final NotificationService notificationService;

    public Order placeOrder(OrderRequest request) {
        // All these calls are local — they run in the same JVM process
        // Atomic: if any step throws an exception, the whole transaction can roll back
        inventoryService.reserve(request.getItems());    // In-process call
        Payment payment = paymentService.charge(request.getPaymentInfo()); // In-process call
        Order order = orderRepository.save(new Order(request, payment));
        notificationService.sendConfirmation(order);    // In-process call
        return order;
    }
}
```

The critical thing to appreciate here is that **a local method call costs essentially nothing**. It's a few nanoseconds. The same logic across microservices costs milliseconds per call, adds failure modes (network timeouts, service unavailability), and requires serialization/deserialization of data. The monolith eliminates all of that complexity.

**Variants of monolith:**
- **Modular Monolith:** A monolith with strong internal module boundaries, enforced by Java's module system (JPMS) or package-private visibility. Each module has a clean API and hides its internals. This is often the best first step before considering microservices.
- **Layered Monolith (N-Tier):** The traditional approach — presentation layer, service layer, data access layer. Easy to understand but often devolves into a "big ball of mud" when business logic leaks across layers.

**Strengths:** Simple to develop, easy to test end-to-end, no distributed systems complexity, single transaction scope, easy debugging (one process to attach a debugger to), low operational overhead.

**Weaknesses:** As the system grows, deployments become risky (changing one module requires deploying everything), parts of the system cannot scale independently, different teams can block each other's releases, and the technology stack is locked in for the entire application.

**When to choose monolith:** Almost always at the start. The rule of thumb from Martin Fowler: "Don't start with microservices. Migrate to them when the monolith's pain becomes real, not anticipated."

---

### 1.2 Service-Oriented Architecture (SOA)

**SOA** is an architectural style that structures an application as a collection of **coarse-grained services** that communicate over a network, typically using an **Enterprise Service Bus (ESB)**. It emerged in the 2000s as enterprises tried to integrate large, heterogeneous legacy systems.

**Theory:** SOA differs from microservices in a key dimension — SOA services are larger and more generalized, designed to be reused across many business processes. An SOA "Order Service" might contain all order-related logic for the entire enterprise. Services communicate through an ESB that handles routing, transformation, and orchestration.

```
                          ┌──────────────────────────────┐
                          │     Enterprise Service Bus    │
                          │  (routing, transformation,    │
                          │   orchestration, security)    │
                          └──────────────┬───────────────┘
                                        │
             ┌──────────────┬───────────┴─────────┬──────────────┐
             │              │                     │              │
      ┌──────┴──────┐ ┌─────┴──────┐ ┌────────────┴────┐ ┌──────┴──────┐
      │Order Service│ │  Customer  │ │Inventory Service│ │  Payment    │
      │             │ │  Service   │ │                 │ │  Service    │
      └─────────────┘ └────────────┘ └─────────────────┘ └─────────────┘
```

The ESB is simultaneously SOA's greatest selling point (it centralizes cross-cutting concerns like security and logging) and its greatest liability (it becomes a single point of failure and a bottleneck for all communication).

**In Java, SOA typically used SOAP web services:**

```java
// SOA typically uses SOAP/WSDL for service contracts
// The service definition is formal and strict
@WebService(name = "OrderService", targetNamespace = "http://enterprise.example.com/orders")
@SOAPBinding(style = Style.DOCUMENT, use = Use.LITERAL)
public interface OrderServiceWS {

    @WebMethod(operationName = "submitOrder")
    @WebResult(name = "orderResponse")
    OrderResponse submitOrder(
        @WebParam(name = "orderRequest") OrderRequest request
    );
}

@WebService(endpointInterface = "com.example.OrderServiceWS")
public class OrderServiceImpl implements OrderServiceWS {
    @Override
    public OrderResponse submitOrder(OrderRequest request) {
        // Business logic here
        return new OrderResponse("ORDER-001", "CONFIRMED");
    }
}
```

**When SOA made sense:** Large enterprises with many existing legacy systems that needed to be integrated. The ESB abstracted away the differences between a COBOL mainframe system, a .NET application, and a Java EE server.

**Why SOA fell out of favor:** The ESB became a bottleneck and single point of failure. The heavy XML/SOAP protocol added latency and complexity. Services were too coarse-grained — changing one service often required changing others. The organizational model didn't match the technical model.

---

### 1.3 Microservices Architecture

**Microservices** decompose an application into small, independently deployable services, each owning its own data and focused on a single business capability. Unlike SOA, microservices communicate directly (no ESB), prefer lightweight protocols (HTTP/REST, gRPC, async messaging), and each service is designed to be small enough that a small team can own it entirely.

**Theory:** The key insight of microservices is the **alignment between team structure and system structure** (Conway's Law). If you want two teams to work independently, their systems should be able to deploy independently. This requires clear service boundaries, isolated data stores, and asynchronous communication wherever possible.

The defining principle of microservices is that each service **owns its data**. You cannot have one service reading another service's database — that creates hidden coupling that defeats the purpose of the decomposition.

```
┌────────────────────────────────────────────────────────────────────┐
│                           API Gateway                              │
│           (routing, auth, rate limiting, SSL termination)          │
└────────┬───────────────┬──────────────────────┬────────────────────┘
         │               │                      │
┌────────▼────────┐  ┌───▼─────────────┐  ┌────▼────────────────┐
│  Order Service  │  │Inventory Service│  │  Payment Service    │
│  ─────────────  │  │ ─────────────   │  │  ──────────────     │
│  Orders DB      │  │  Inventory DB   │  │  Payments DB        │
│  (PostgreSQL)   │  │  (PostgreSQL)   │  │  (PostgreSQL)       │
└────────┬────────┘  └────────────────-┘  └─────────────────────┘
         │
    (publishes events)
         │
┌────────▼────────────────┐
│   Message Broker         │
│   (Kafka / RabbitMQ)     │
└────────┬────────────────┘
         │
┌────────▼────────┐
│ Notification    │
│ Service         │
│ (MongoDB)       │
└─────────────────┘
```

```java
// Each microservice is a standalone Spring Boot application
// Order Service: knows nothing about how notifications are sent
@SpringBootApplication
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}

@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orderService;

    @PostMapping
    public ResponseEntity<OrderResponse> placeOrder(@RequestBody @Valid OrderRequest request) {
        OrderResponse response = orderService.placeOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}

@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;  // HTTP client for Inventory Service
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate; // Async notification

    @Transactional
    public OrderResponse placeOrder(OrderRequest request) {
        // Synchronous call to Inventory Service via HTTP
        // This is a network call — can fail, can be slow, needs retry logic
        inventoryClient.reserve(request.getItems());

        Order order = orderRepository.save(Order.from(request));

        // Publish event to Kafka — asynchronous, decoupled
        // Notification Service subscribes to this topic independently
        kafkaTemplate.send("order.placed", new OrderPlacedEvent(order.getId(), order.getCustomerEmail()));

        return OrderResponse.from(order);
    }
}

// Inventory Service client — generated using Feign (declarative HTTP client)
@FeignClient(name = "inventory-service", url = "${inventory.service.url}")
public interface InventoryClient {
    @PostMapping("/api/inventory/reserve")
    void reserve(@RequestBody List<OrderItem> items);
}
```

**The inter-service communication challenge:** When `OrderService` calls `InventoryService` over HTTP, that call can fail. The network is unreliable. The Inventory Service might be temporarily down. The Order Service must handle this gracefully — with timeouts, retries, and circuit breakers (covered in Part 3). This complexity is the price of microservices independence.

**Strengths:** Independent deployment of each service, independent scaling, technology heterogeneity (each service can use the best tech for its job), clear ownership, and fault isolation (one service's failure doesn't necessarily bring down the whole system).

**Weaknesses:** Distributed systems complexity (network failures, latency, partial failures), data consistency challenges across services (no single transaction boundary), operational overhead (many services to deploy, monitor, and debug), and debugging is much harder (a user request may span 10 services).

**When to choose microservices:** When you have multiple independent teams that need to release independently, when different parts of the system have radically different scaling needs, or when you're refactoring a monolith that is genuinely hindering development speed.

---

### 1.4 Serverless Architecture

**Serverless** takes the decomposition further — instead of always-running services, you deploy individual **functions** that are invoked on-demand and billed only for the time they actually run. The cloud provider handles all infrastructure management, scaling, and availability.

**Theory:** The name is misleading — there are still servers, but you don't manage them. You write a function, upload it, and the cloud runs it when triggered. This removes entire categories of operational concern: no server provisioning, no OS patching, no capacity planning. But it introduces new constraints: functions must be stateless, have limited execution time (15 minutes in AWS Lambda), and cold starts add latency.

```java
// AWS Lambda function in Java — implements RequestHandler
// This entire class is a "serverless function" — no server, no always-on process
public class OrderProcessorFunction
        implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    // Dependencies are initialized once per container (cold start pays this cost once)
    private final ObjectMapper objectMapper = new ObjectMapper();
    private final OrderRepository orderRepository;

    public OrderProcessorFunction() {
        // Initialize dependencies in the constructor
        // These will be reused across "warm" invocations of the same container
        this.orderRepository = new OrderRepository(
            System.getenv("DATABASE_URL"),
            System.getenv("DATABASE_PASSWORD")
        );
    }

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent event,
            Context context) {

        try {
            OrderRequest request = objectMapper.readValue(event.getBody(), OrderRequest.class);

            // Business logic — must complete within Lambda's timeout (up to 15 minutes)
            Order order = orderRepository.save(Order.from(request));

            return new APIGatewayProxyResponseEvent()
                .withStatusCode(201)
                .withBody(objectMapper.writeValueAsString(OrderResponse.from(order)));

        } catch (JsonProcessingException e) {
            return new APIGatewayProxyResponseEvent()
                .withStatusCode(400)
                .withBody("{\"error\": \"Invalid request body\"}");
        }
    }
}
```

**Cold start problem:** When a Lambda function hasn't been invoked recently, AWS must spin up a new container — loading the JVM, running the Spring context initialization, opening database connections. For a Spring Boot Lambda, this can take 3-10 seconds. This is why GraalVM Native Image is compelling for serverless Java — it compiles to a native binary with near-instant startup, eliminating the cold start penalty.

**Strengths:** Zero infrastructure management, automatic and infinite scaling, pay-per-execution billing (excellent for spiky or unpredictable workloads), true isolation per function.

**Weaknesses:** Cold start latency, statelessness requirement, vendor lock-in, limited execution time, complex local development/testing, and debugging distributed serverless systems is challenging.

---

### 1.5 Event-Driven Architecture

**Event-driven architecture (EDA)** structures a system around the production, detection, and consumption of events. Components communicate by publishing events to a broker, and other components subscribe to events they care about, with no direct coupling between producer and consumer.

**Theory:** EDA is a style, not a deployment model — you can use it inside a monolith, between microservices, or in serverless functions. The key characteristic is **temporal decoupling**: the producer does not wait for the consumer to respond (and often doesn't even know the consumer exists). This is fundamentally different from synchronous REST calls where the caller waits for a response.

There are two main variants:
- **Event Notification:** Publishing that something happened. `OrderPlaced` event with just the order ID. Consumers fetch details themselves if needed.
- **Event-Carried State Transfer:** The event contains all the data consumers need. `OrderPlaced` event with full order details. Consumers don't need to call back.

```java
// Event producer — Order Service
// After saving an order, publish an event to Kafka
// The Order Service does NOT know or care who consumes this event
@Service
public class OrderService {
    private final KafkaTemplate<String, OrderPlacedEvent> kafka;

    public Order placeOrder(OrderRequest request) {
        Order order = orderRepository.save(Order.from(request));

        // Publish event — fire and forget (async)
        // OrderService doesn't wait for Notification or Analytics to respond
        kafka.send("orders.placed", order.getId().toString(),
            OrderPlacedEvent.builder()
                .orderId(order.getId())
                .customerId(order.getCustomerId())
                .items(order.getItems())
                .totalAmount(order.getTotalAmount())
                .placedAt(Instant.now())
                .build()
        );

        return order; // Return immediately — don't wait for consumers
    }
}

// ─────────────────────────────────────────────────────────
// Event consumer — Notification Service
// Completely independent service; subscribes to the same Kafka topic
// If Order Service is updated, Notification Service doesn't need to change
@Service
public class OrderNotificationConsumer {

    @KafkaListener(topics = "orders.placed", groupId = "notification-service")
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Process the event — could happen seconds or minutes after it was published
        emailService.sendOrderConfirmation(event.getCustomerId(), event.getOrderId());
        // Notification Service failure does NOT affect Order placement
    }
}

// ─────────────────────────────────────────────────────────
// Another independent consumer — Analytics Service
@Service
public class OrderAnalyticsConsumer {

    @KafkaListener(topics = "orders.placed", groupId = "analytics-service")
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Multiple consumers can independently consume the same event
        // Kafka retains the event — consumers read at their own pace
        metricsService.recordOrder(event.getTotalAmount(), event.getPlacedAt());
    }
}
```

The profound advantage here is **extensibility**: you can add new consumers (a fraud detection service, a loyalty points service, a shipping service) without touching the Order Service at all. The Order Service publishes its event into the void and lets the rest of the system react however it needs to.

---

## 2. CAP Theorem

The **CAP Theorem** (Brewer's Theorem, 2000) states that a distributed data store can provide at most **two** of the following three guarantees simultaneously:

- **Consistency (C):** Every read receives the most recent write or an error. All nodes in the system see the same data at the same time.
- **Availability (A):** Every request receives a response (not an error), though the data may not be the most recent.
- **Partition Tolerance (P):** The system continues to operate even when network partitions occur (messages between nodes are lost or delayed).

**Theory — the hard truth:** In any real distributed system, network partitions **will** happen. Network cables get cut, switches fail, AWS has AZ outages. Partition tolerance is not optional — you must choose it. So the real choice is: when a partition occurs, do you sacrifice Consistency or Availability?

```
                    Consistency
                         ▲
                         │
              CA          │          CP
         (Traditional    │    (MongoDB in
          RDBMS single   │    strict mode,
          node)          │    HBase, Zookeeper)
                         │
─────────────────────────┼─────────────────────── Partition
                         │                         Tolerance
              AP          │
         (Cassandra,     │
          DynamoDB,      │
          CouchDB)       │
                         ▼
                    Availability
```

**CP Systems (Consistency + Partition Tolerance):** When a partition occurs, the system refuses to answer (returns an error) rather than potentially returning stale data. Example: a banking system that refuses to process a transaction when it can't confirm the current balance. In Java, Zookeeper is a classic CP system — it refuses reads/writes during leader election (a partition event) rather than risk inconsistency.

**AP Systems (Availability + Partition Tolerance):** When a partition occurs, the system continues to serve requests but may return stale data from different nodes. Example: a shopping cart — it's better to show a slightly outdated cart than to refuse to show the cart at all. Cassandra, DynamoDB, and CouchDB are AP systems.

**A critical nuance:** CAP is not a binary switch. Modern systems use "tunable consistency" — you can dial between stronger and weaker consistency per-operation based on need. Cassandra's consistency levels (`ONE`, `QUORUM`, `ALL`) let you choose per-query.

```java
// Cassandra in Java — tunable consistency per query
// This demonstrates the practical application of CAP theorem
import com.datastax.oss.driver.api.core.CqlSession;
import com.datastax.oss.driver.api.core.DefaultConsistencyLevel;

CqlSession session = CqlSession.builder().build();

// QUORUM consistency: majority of replicas must agree before returning
// More consistent but slower — use for reads that must be accurate (account balance)
SimpleStatement balanceQuery = SimpleStatement.builder(
    "SELECT balance FROM accounts WHERE id = ?")
    .addPositionalValue(accountId)
    .setConsistencyLevel(DefaultConsistencyLevel.QUORUM) // Strong consistency
    .build();

// ONE consistency: only one replica needs to respond
// Faster and highly available, but may return stale data — use for less-critical reads
SimpleStatement productQuery = SimpleStatement.builder(
    "SELECT name, description FROM products WHERE id = ?")
    .addPositionalValue(productId)
    .setConsistencyLevel(DefaultConsistencyLevel.ONE) // Eventual consistency
    .build();
```

---

## 3. ACID vs. BASE

These are two competing philosophies for how distributed databases handle transactions and consistency.

### ACID

**ACID** is the traditional transactional model, optimized for correctness:

- **Atomicity:** All operations in a transaction either fully succeed or fully fail — no partial updates. Either the money leaves your account AND arrives in mine, or neither happens.
- **Consistency:** A transaction brings the database from one valid state to another. Constraints (foreign keys, unique indexes) are always enforced.
- **Isolation:** Concurrent transactions execute as if they were serial. One transaction's intermediate state is invisible to other transactions.
- **Durability:** Once a transaction is committed, it remains committed even if the system crashes immediately afterward.

```java
// ACID transaction in Java with Spring — the database guarantees all-or-nothing
@Service
public class BankTransferService {

    @Transactional // Spring wraps this entire method in a database transaction
    public void transfer(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        Account from = accountRepository.findByIdForUpdate(fromAccountId); // Lock the row
        Account to = accountRepository.findByIdForUpdate(toAccountId);

        if (from.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException("Not enough balance");
            // Transaction automatically ROLLS BACK — no money is debited
        }

        from.debit(amount);
        to.credit(amount);

        accountRepository.save(from);
        accountRepository.save(to);
        // Transaction COMMITs here — both changes are permanent
        // If the system crashes between debit and credit, BOTH operations are rolled back
        // ACID guarantees atomicity: you never get a state where money is debited but not credited
    }
}
```

### BASE

**BASE** is the model favored by many distributed NoSQL systems, optimized for availability and scalability:

- **Basically Available:** The system guarantees availability, but it may respond with stale or partial data.
- **Soft State:** The state of the system can change over time even without new input — data is converging, not immediately consistent.
- **Eventually Consistent:** If no new updates are made, all replicas will eventually converge to the same value.

BASE accepts temporary inconsistency in exchange for higher availability and performance. It's not that BASE systems are incorrect — it's that they operate under a different model of "correctness" that's acceptable for certain business domains.

```java
// BASE / Eventually Consistent example — shopping cart in DynamoDB
// Optimistic locking with version numbers instead of database locks
@Repository
public class CartRepository {

    public Cart getCart(String userId) {
        // This read might return slightly stale data if a replica is behind
        // That's acceptable — the user seeing a slightly old cart is better than an error
        return dynamoDbMapper.load(Cart.class, userId);
    }

    public void addItem(String userId, CartItem item) {
        Cart cart = getCart(userId);
        cart.addItem(item);
        cart.incrementVersion(); // Optimistic locking — use version number instead of row lock

        try {
            dynamoDbMapper.save(cart,
                new DynamoDBSaveExpression()
                    .withExpectedEntry("version",
                        new ExpectedAttributeValue()
                            .withValue(new AttributeValue().withN(
                                String.valueOf(cart.getVersion() - 1))) // Expect old version
                            .withComparisonOperator(ComparisonOperator.EQ)));
        } catch (ConditionalCheckFailedException e) {
            // Another process modified the cart concurrently — retry the operation
            // This is BASE's way of handling conflicts: detect and retry rather than lock
            addItem(userId, item); // Recursive retry (in production, add exponential backoff)
        }
    }
}
```

**Choosing between ACID and BASE:** Financial transactions, inventory reservation, and anything involving money absolutely need ACID guarantees. User profiles, shopping carts, social media likes, analytics — these can tolerate eventual consistency and benefit from the scale that BASE systems offer.

---

## 4. Eventual Consistency

**Eventual consistency** is a specific type of weak consistency guarantee: if no new updates are made to an item, all reads will eventually return the last updated value. The word "eventually" is doing a lot of work here — it could mean milliseconds (typical for within a datacenter) or seconds/minutes (for cross-region replication).

**Theory:** In an eventually consistent system, different nodes may hold different values for the same data at the same moment. Over time, the nodes gossip with each other (or the write is propagated by a coordinator) and they converge to the same value. The challenge is designing your system to handle **read-your-own-writes anomalies** (you write a value and immediately can't see your own write because you're routed to a different replica), and **concurrent write conflicts**.

```java
// Handling eventual consistency in application code
// Strategy 1: Read-your-own-writes with sticky sessions
// After a write, route the same user to the same replica for a short window

@Service
public class UserProfileService {
    private final UserRepository userRepository;
    private final Cache<String, UserProfile> readYourWritesCache;

    public UserProfile updateProfile(String userId, ProfileUpdate update) {
        UserProfile updated = userRepository.save(update.applyTo(getProfile(userId)));

        // Cache the updated profile locally for 5 seconds
        // Even if the database replica hasn't caught up yet, this user will see their update
        readYourWritesCache.put(userId, updated, 5, TimeUnit.SECONDS);
        return updated;
    }

    public UserProfile getProfile(String userId) {
        // Check local cache first — ensures read-your-own-writes consistency
        UserProfile cached = readYourWritesCache.getIfPresent(userId);
        if (cached != null) return cached; // Return the version we know is current
        return userRepository.findById(userId).orElseThrow();
    }
}

// Strategy 2: Conflict resolution using Last-Write-Wins (LWW) with timestamps
// When two nodes have conflicting versions, keep the most recent one
public class ConflictResolver {
    public UserProfile resolve(UserProfile version1, UserProfile version2) {
        // Last-Write-Wins: highest timestamp wins
        // Simple but can silently discard updates — only appropriate for idempotent data
        return version1.getUpdatedAt().isAfter(version2.getUpdatedAt())
            ? version1 : version2;
    }
}

// Strategy 3: Vector clocks / CRDTs for more nuanced conflict resolution
// A CRDT (Conflict-free Replicated Data Type) is a data structure designed so
// that concurrent updates can always be merged automatically without conflicts.
// Example: a counter that can only be incremented (G-Counter)
// Incrementing at two nodes simultaneously can be merged by adding the increments
public class GCounter {
    private final Map<String, Long> nodeCounters = new ConcurrentHashMap<>();
    private final String nodeId;

    public void increment() {
        nodeCounters.merge(nodeId, 1L, Long::sum);
    }

    // Merge with the state from another node — always safe, no conflicts possible
    public void merge(GCounter other) {
        other.nodeCounters.forEach((node, count) ->
            nodeCounters.merge(node, count, Math::max)); // Take max per node
    }

    public long value() {
        return nodeCounters.values().stream().mapToLong(Long::longValue).sum();
    }
}
```

---

## 5. Important Points & Best Practices

**Don't choose microservices prematurely.** The distributed systems complexity that microservices introduce is real and significant. A team of 5 engineers will almost always be more productive with a well-structured modular monolith than with 10 microservices. The organizational motivation (independent teams, independent deployments) is what justifies the complexity — not the technical architecture itself.

**CAP Theorem is a framework for thinking, not a recipe.** Real systems don't perfectly fit CP or AP — they make different consistency/availability trade-offs for different operations within the same system. A payment service might be strongly consistent for charges but eventually consistent for reporting dashboards.

**ACID is not "slow" and BASE is not "fast" by definition.** ACID transactions in a well-tuned PostgreSQL on modern hardware can handle tens of thousands of transactions per second. The performance trade-off is real but often overstated. Only move to BASE/eventual consistency when you've actually measured that ACID is your bottleneck, not out of premature optimization.

**Eventual consistency is a UX problem as much as a technical one.** When you build an eventually consistent system, you need to design the user experience around it. Showing a user their own submitted review before all replicas have it (optimistic updates in the UI) is a design pattern, not a hack. Be explicit about this in your team — "this feature is eventually consistent" is architectural documentation that must be shared.

**Event-driven architecture is best for write operations, not reads.** Events naturally model "something happened" — order placed, payment processed. They are not a natural fit for query operations where you need current state. A system that uses events for writes (commands/domain events) but direct service calls or materialized views for reads is following the CQRS pattern, which is the mature approach to EDA.
