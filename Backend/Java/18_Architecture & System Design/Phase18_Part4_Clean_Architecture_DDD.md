# Phase 18 — Part 4: Clean Architecture, Hexagonal Architecture & Domain-Driven Design

> These architectural philosophies answer a question that grows more urgent as software systems age: how do you structure a codebase so that business logic remains understandable, testable, and changeable — even as the technical environment around it evolves? Frameworks change, databases change, and frontends change. The core business logic should not have to change with them.

---

## 1. The Problem These Architectures Solve

Imagine a Spring Boot application three years into its life. The business logic for processing an order is scattered across controllers, services, and JPA entities. The Order entity has annotations from Spring Data, Jackson, Hibernate, Bean Validation, and the domain model itself — all mixed together. Changing the database means changing domain objects. Testing business rules requires spinning up a Spring context. This is the "Big Ball of Mud" — technically functional but architecturally decayed.

Clean Architecture, Hexagonal Architecture, and Onion Architecture are three related but distinct answers to this problem. They all share a core insight: **the dependency rule**. Business logic should be the most stable part of your system. Everything else — frameworks, databases, user interfaces, external APIs — should depend on business logic. Business logic should never depend on any of them.

---

## 2. Hexagonal Architecture (Ports and Adapters)

**Hexagonal Architecture**, invented by Alistair Cockburn in 2005, structures a system around a central application core (the hexagon) surrounded by **ports** (interfaces that the core defines) and **adapters** (implementations of those interfaces that connect the core to the outside world).

The shape "hexagon" has no special significance — Cockburn chose it to visually distinguish the inside from the outside and to have enough sides to draw multiple ports. The meaningful metaphor is that the application core is a fortress: the outside world can only enter through defined ports, and the core does not know or care what's on the other side of those ports.

**Ports** come in two varieties. A **Driving Port** (or Primary Port) is an interface that the outside world uses to drive the application. A REST controller, a command-line interface, and a message consumer are all "drivers" — they initiate action by calling into the application through a port. A **Driven Port** (or Secondary Port) is an interface that the application uses to interact with external infrastructure. A repository interface, an email sender interface, and a payment gateway interface are all "driven ports" — the application calls through them to reach external systems.

**Adapters** implement ports. A `JpaOrderRepository` is an adapter that implements the `OrderRepository` driven port. An `OrderRestController` is an adapter that calls the `PlaceOrderUseCase` driving port. Crucially, the application core defines the port interfaces — not the infrastructure. The `OrderRepository` interface lives inside the domain, not in the Spring Data layer.

```
┌────────────────────────────────────────────────────────────────────┐
│                         OUTSIDE WORLD                              │
│  ┌──────────┐  ┌──────────┐    ┌──────────────┐  ┌─────────────┐  │
│  │  REST    │  │  Kafka   │    │  PostgreSQL  │  │  Stripe     │  │
│  │ Adapter  │  │ Adapter  │    │  Adapter     │  │  Adapter    │  │
│  └────┬─────┘  └────┬─────┘    └──────┬───────┘  └──────┬──────┘  │
│       │ calls       │ calls            │ implements      │ implements │
│  ─────┼─────────────┼──────────────────┼─────────────────┼───────── │
│  ┌────▼─────────────▼──────────────────▼─────────────────▼───────┐ │
│  │               APPLICATION CORE (HEXAGON)                       │ │
│  │   ┌──────────────────┐      ┌───────────────────────────────┐  │ │
│  │   │  Driving Ports   │      │      Driven Ports             │  │ │
│  │   │ PlaceOrderUseCase│      │ OrderRepository (interface)   │  │ │
│  │   │ GetOrderUseCase  │      │ PaymentGateway (interface)    │  │ │
│  │   └──────────────────┘      └───────────────────────────────┘  │ │
│  │                 Domain Model: Order, Customer, Payment          │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│  ─────────────────────────────────────────────────────────────────── │
└────────────────────────────────────────────────────────────────────┘
```

```java
// ─────────────────────────────────────────────────────────────────
// DOMAIN LAYER: Pure Java — zero framework dependencies
// This is the hexagon — it knows nothing about Spring, JPA, or Kafka
// ─────────────────────────────────────────────────────────────────

// Domain entity — encapsulates business rules
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private OrderStatus status;
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    // Business rule lives HERE in the domain — not in a service or controller
    public void confirm() {
        if (this.status != OrderStatus.PENDING) {
            throw new InvalidOrderStateException(
                "Cannot confirm an order that is not PENDING. Current status: " + this.status);
        }
        this.status = OrderStatus.CONFIRMED;
        // Raise a domain event — the domain announces what happened
        // Listeners (potentially adapters) will react to it
        domainEvents.add(new OrderConfirmedEvent(this.id, this.customerId, this.items));
    }

    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::subtotal)
            .reduce(Money.ZERO, Money::add);
    }

    public List<DomainEvent> flushEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }

    // Factory method — creation logic belongs in the domain
    public static Order place(CustomerId customerId, List<OrderItem> items) {
        if (items.isEmpty()) throw new DomainException("Cannot place an order with no items");
        Order order = new Order(OrderId.generate(), customerId, items, OrderStatus.PENDING);
        order.domainEvents.add(new OrderPlacedEvent(order.id, customerId));
        return order;
    }
}

// ─────────────────────────────────────────────────────────────────
// DRIVEN PORTS (Secondary Ports): Interfaces defined BY the domain
// The domain says: "I need something that can store and retrieve orders"
// The domain does NOT say how — that's the adapter's concern
// ─────────────────────────────────────────────────────────────────
public interface OrderRepository {   // Lives in the DOMAIN package
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
}

public interface PaymentGateway {    // Lives in the DOMAIN package
    PaymentResult charge(CustomerId customer, Money amount, PaymentMethod method);
}

public interface EventPublisher {    // Lives in the DOMAIN package
    void publish(DomainEvent event);
}

// ─────────────────────────────────────────────────────────────────
// DRIVING PORTS (Primary Ports): Use Case interfaces
// These define what the application CAN DO — its capabilities
// ─────────────────────────────────────────────────────────────────
public interface PlaceOrderUseCase {
    OrderId placeOrder(PlaceOrderCommand command);
}

public interface ConfirmOrderUseCase {
    void confirmOrder(ConfirmOrderCommand command);
}

// ─────────────────────────────────────────────────────────────────
// APPLICATION SERVICE: Orchestrates domain objects and driven ports
// Implements use cases; belongs in the APPLICATION layer (inside the hexagon)
// Has ZERO infrastructure dependencies — only domain + port interfaces
// ─────────────────────────────────────────────────────────────────
public class OrderApplicationService implements PlaceOrderUseCase, ConfirmOrderUseCase {
    private final OrderRepository orderRepository;   // Interface — not a JPA repository
    private final PaymentGateway paymentGateway;     // Interface — not Stripe directly
    private final EventPublisher eventPublisher;     // Interface — not Kafka directly
    private final CustomerRepository customerRepository;

    @Override
    public OrderId placeOrder(PlaceOrderCommand command) {
        // Validate that the customer exists (application logic)
        Customer customer = customerRepository.findById(command.customerId())
            .orElseThrow(() -> new CustomerNotFoundException(command.customerId()));

        // Create the order using domain logic (rich domain model)
        Order order = Order.place(command.customerId(), command.items());

        // Charge payment through the driven port (don't know it's Stripe — just the interface)
        PaymentResult payment = paymentGateway.charge(
            customer.getId(), order.calculateTotal(), command.paymentMethod());
        if (!payment.isSuccessful()) {
            throw new PaymentFailedException(payment.getFailureReason());
        }

        // Persist through the driven port (don't know it's PostgreSQL — just the interface)
        orderRepository.save(order);

        // Publish domain events through the driven port (don't know it's Kafka)
        order.flushEvents().forEach(eventPublisher::publish);

        return order.getId();
    }

    @Override
    public void confirmOrder(ConfirmOrderCommand command) {
        Order order = orderRepository.findById(command.orderId())
            .orElseThrow(() -> new OrderNotFoundException(command.orderId()));

        order.confirm(); // Business rule enforced in the domain object

        orderRepository.save(order);
        order.flushEvents().forEach(eventPublisher::publish);
    }
}

// ─────────────────────────────────────────────────────────────────
// ADAPTERS: Infrastructure implementations of the ports
// These DEPEND ON the domain (they implement domain interfaces)
// The domain does NOT depend on them — this is the key dependency direction
// ─────────────────────────────────────────────────────────────────

// Driven Adapter: JPA implementation of OrderRepository driven port
@Repository
public class JpaOrderRepositoryAdapter implements OrderRepository {
    private final JpaOrderEntityRepository jpaRepo;  // Spring Data JPA repository
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        // Domain Order → JPA Entity → database
        OrderJpaEntity entity = mapper.toEntity(order);
        jpaRepo.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        // Database → JPA Entity → Domain Order
        return jpaRepo.findById(id.value()).map(mapper::toDomain);
    }

    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return jpaRepo.findByCustomerId(customerId.value())
            .stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
    }
}

// Driven Adapter: Stripe implementation of PaymentGateway driven port
@Component
public class StripePaymentGatewayAdapter implements PaymentGateway {
    private final Stripe stripeClient;

    @Override
    public PaymentResult charge(CustomerId customer, Money amount, PaymentMethod method) {
        // Domain PaymentMethod → Stripe-specific parameters
        try {
            ChargeCreateParams params = ChargeCreateParams.builder()
                .setAmount(amount.getMinorUnits())     // Stripe uses cents
                .setCurrency(amount.getCurrency().getCode().toLowerCase())
                .setSource(method.getToken())
                .build();
            Charge charge = Charge.create(params);
            return PaymentResult.success(charge.getId());
        } catch (StripeException e) {
            return PaymentResult.failure(e.getMessage());
        }
    }
}

// Driving Adapter: REST controller — calls INTO the application through a driving port
@RestController
@RequestMapping("/api/orders")
public class OrderRestAdapter {
    private final PlaceOrderUseCase placeOrderUseCase;    // Interface — not the service directly
    private final ConfirmOrderUseCase confirmOrderUseCase;
    private final OrderRequestMapper mapper;

    @PostMapping
    public ResponseEntity<PlaceOrderResponse> placeOrder(
            @RequestBody @Valid PlaceOrderRequest request) {
        // REST model → Command (application boundary object)
        PlaceOrderCommand command = mapper.toCommand(request);
        // Call through the driving port interface — decoupled from the implementation
        OrderId orderId = placeOrderUseCase.placeOrder(command);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(new PlaceOrderResponse(orderId.value()));
    }
}

// Driving Adapter: Kafka consumer — also calls INTO the application through a driving port
// Notice how the same use case interface (PlaceOrderUseCase) can be driven by
// BOTH a REST controller and a Kafka consumer — the use case doesn't know or care
@Component
public class OrderKafkaConsumerAdapter {
    private final PlaceOrderUseCase placeOrderUseCase;
    private final OrderMessageMapper mapper;

    @KafkaListener(topics = "order-commands", groupId = "order-service")
    public void handleOrderCommand(OrderCommandMessage message) {
        PlaceOrderCommand command = mapper.toCommand(message);
        placeOrderUseCase.placeOrder(command);
    }
}
```

**The profound testability benefit:** Because the domain and application service depend only on interfaces (ports), you can test the entire business logic with zero infrastructure. No Spring context, no database, no Kafka:

```java
class OrderApplicationServiceTest {
    // Pure unit tests — no Spring, no database, no Stripe
    private OrderRepository orderRepository = mock(OrderRepository.class);
    private PaymentGateway paymentGateway = mock(PaymentGateway.class);
    private EventPublisher eventPublisher = mock(EventPublisher.class);

    private OrderApplicationService service =
        new OrderApplicationService(orderRepository, paymentGateway, eventPublisher, ...);

    @Test
    void placeOrder_whenPaymentFails_shouldNotSaveOrder() {
        // Arrange: configure mocks
        when(paymentGateway.charge(any(), any(), any()))
            .thenReturn(PaymentResult.failure("Card declined"));

        // Act + Assert: exception thrown, and order was NEVER saved to the repository
        assertThrows(PaymentFailedException.class,
            () -> service.placeOrder(testCommand()));

        verify(orderRepository, never()).save(any()); // Critical assertion: no partial saves
    }
}
```

---

## 3. Clean Architecture (Uncle Bob's Rings)

**Clean Architecture** by Robert C. Martin is closely related to Hexagonal Architecture but organizes the system into concentric rings instead of a hexagon. The dependency rule is the same: code can only depend inward (toward the center). The inner rings know nothing about the outer rings.

```
         ┌──────────────────────────────────────────────┐
         │          Frameworks & Drivers (outer)        │
         │  ┌────────────────────────────────────────┐  │
         │  │       Interface Adapters (middle)       │  │
         │  │  ┌──────────────────────────────────┐  │  │
         │  │  │  Application Business Rules       │  │  │
         │  │  │        Use Cases                  │  │  │
         │  │  │  ┌────────────────────────────┐  │  │  │
         │  │  │  │ Enterprise Business Rules  │  │  │  │
         │  │  │  │    Entities / Domain       │  │  │  │
         │  │  │  └────────────────────────────┘  │  │  │
         │  │  └──────────────────────────────────┘  │  │
         │  └────────────────────────────────────────┘  │
         └──────────────────────────────────────────────┘
```

**Entities (innermost ring):** Enterprise-wide business rules that would be true regardless of the application being built. An `Order` domain object, a `Money` value object, a `Customer` entity — these encapsulate the most fundamental business rules.

**Use Cases (second ring):** Application-specific business rules. The `PlaceOrderUseCase` orchestrates entities to fulfill a specific user need. It doesn't know about REST, databases, or external services — it only knows about entities and abstract interfaces (ports).

**Interface Adapters (third ring):** Converts data from the format most convenient for use cases into the format most convenient for external agencies (databases, web, etc.). Controllers, presenters, gateways, and repositories live here.

**Frameworks & Drivers (outermost ring):** Spring Boot, Hibernate, Kafka, Stripe SDK. These are the details — important but changeable without affecting the inner rings.

The key practical insight of Clean Architecture beyond what we already covered in Hexagonal Architecture is the explicit concept of **boundary objects** (also called DTOs or Request/Response models). Data that crosses a ring boundary should be "boundary objects" — simple data structures with no behavior — not domain entities. You should never pass a JPA entity from the database layer into a REST controller, because that JPA entity carries database-specific annotations and lifecycle that pollutes the presentation layer.

---

## 4. Onion Architecture

**Onion Architecture**, proposed by Jeffrey Palermo in 2008, is essentially the same idea organized slightly differently. It uses the onion metaphor: the inner layers form the core, and outer layers wrap around them like onion skins.

The key distinction Onion Architecture makes explicit is that the **domain model** is at the very center — not use cases — and the domain model has absolutely zero dependencies. Use cases sit in the next ring, calling into the domain model. The principle is the same: the outermost rings (infrastructure) implement the interfaces defined by the inner rings (domain).

In practice, most Java teams use the terms "Hexagonal Architecture" and "Clean Architecture" interchangeably and treat "Onion Architecture" as a variant of the same idea. The important thing is the principle: **dependencies point inward, infrastructure is at the edge, domain is at the center.**

---

## 5. Domain-Driven Design (DDD)

**Domain-Driven Design (DDD)**, introduced by Eric Evans in his 2003 book "Domain-Driven Design: Tackling Complexity in the Heart of Software," is a software development philosophy centered on modeling the core business domain with deep fidelity. It provides a vocabulary and a set of patterns for building software that reflects the real-world domain it serves.

DDD is most valuable when the business domain is genuinely complex — when there are intricate business rules, many edge cases, and domain experts who use specific terminology. DDD is overkill for CRUD applications.

### Ubiquitous Language

The most foundational concept in DDD: developers and domain experts should use the **exact same vocabulary** to describe the domain. When a business analyst says "Customer," your code should have a `Customer` class (not `User`, not `Account`). When they say "place an order," your method should be called `placeOrder()` (not `createOrderRecord()` or `submitOrderForm()`).

When your code uses different names than the business uses, you're building a translation layer in your head every time you read or write code. The Ubiquitous Language eliminates this translation overhead and catches misunderstandings early — if your code doesn't mirror what the business says, you'll notice the mismatch during a review.

### Bounded Context

A **Bounded Context** is an explicit boundary within which a particular domain model applies. The same word can mean different things in different contexts, and a Bounded Context is where you define precisely what a word means.

For example, "Customer" in the **Sales** context means "a person who makes purchases and has a purchasing history." "Customer" in the **Shipping** context means "a delivery address and contact details." "Customer" in the **Support** context means "a ticket holder with a support tier." These are three different models, and forcing them to share one `Customer` class creates a monster class that tries to serve all three contexts and serves none of them well.

```java
// Bounded Context: Order Management — "Customer" means purchasing behavior
package com.example.ordering.domain;

public class Customer {
    private final CustomerId id;
    private CustomerTier tier;         // BRONZE, SILVER, GOLD — drives discount eligibility
    private CreditLimit creditLimit;   // Relevant for order approval
    private List<LoyaltyPoints> points;

    public boolean isEligibleForDiscount() {
        return tier != CustomerTier.BRONZE;
    }
}

// Bounded Context: Shipping — "Customer" means delivery logistics
package com.example.shipping.domain;

public class Customer {          // SAME NAME, completely different class, different package
    private final CustomerId id; // Same ID — but only this is shared between contexts
    private PostalAddress shippingAddress;
    private DeliveryPreferences preferences; // Leave at door? Signature required?
    private List<DeliveryHistory> history;   // Relevant for estimating delivery windows
    // NO tier, NO credit limit, NO loyalty points — not relevant here
}
```

The Bounded Context boundary is enforced by the **Anti-Corruption Layer** (covered in Part 3) when these contexts need to communicate. Each context has its own model and its own team; they integrate at the boundaries, not within the core.

### Aggregates, Entities, and Value Objects

**Entity:** An object defined by its identity — two entities with the same attributes are still different objects if they have different IDs. An `Order` is an entity: Order#1001 and Order#1002 might have the same items and same customer, but they are distinct orders.

**Value Object:** An object defined entirely by its attributes — two value objects with the same attributes are considered equal and interchangeable. A `Money(100, USD)` is a value object. It has no identity of its own — any `Money(100, USD)` is the same as any other `Money(100, USD)`. Value objects should be **immutable** — rather than modifying a `Money` object, you create a new one.

**Aggregate:** A cluster of entities and value objects that are treated as a single unit for data changes, with one **Aggregate Root** that controls all access. The invariants of the aggregate (its business rules) must always be consistent within the aggregate boundary. You always access an aggregate through its root — never directly modify inner entities by bypassing the root.

```java
// ORDER AGGREGATE: Order is the Aggregate Root
// All changes to OrderItems must go through the Order — never modify OrderItem directly
public class Order {  // Aggregate Root
    private final OrderId id;   // Entity identity
    private OrderStatus status;
    private final List<OrderItem> items = new ArrayList<>();  // Part of the aggregate

    // All mutations go through the aggregate root's methods
    // The root enforces all invariants — no external code can break them
    public void addItem(ProductId productId, int quantity, Money unitPrice) {
        if (status != OrderStatus.DRAFT) {
            throw new InvalidOrderStateException("Cannot add items to a non-DRAFT order");
        }
        // Find existing item for this product (aggregate root manages this logic)
        items.stream()
            .filter(item -> item.getProductId().equals(productId))
            .findFirst()
            .ifPresentOrElse(
                item -> item.increaseQuantity(quantity), // Existing item
                () -> items.add(new OrderItem(productId, quantity, unitPrice)) // New item
            );
        // Invariant check: order total must never exceed the customer's credit limit
        validateTotalDoesNotExceedCreditLimit();
    }

    public void removeItem(ProductId productId) {
        items.removeIf(item -> item.getProductId().equals(productId));
        if (items.isEmpty()) {
            throw new DomainException("Order must have at least one item");
        }
    }

    public Money calculateTotal() {
        return items.stream().map(OrderItem::subtotal).reduce(Money.ZERO, Money::add);
    }
}

// OrderItem is an Entity within the aggregate — but accessed only through Order
public class OrderItem {
    private final OrderItemId id;   // Has its own identity within the aggregate
    private final ProductId productId;
    private int quantity;
    private final Money unitPrice;

    // Package-private: only the Order (in the same package) can modify OrderItem
    void increaseQuantity(int additional) {
        if (additional <= 0) throw new DomainException("Quantity increase must be positive");
        this.quantity += additional;
    }

    public Money subtotal() {
        return unitPrice.multiply(quantity);
    }
}

// MONEY Value Object: defined by its value, not its identity
// Two Money(100, USD) objects are equal — no distinguishing identity
public final class Money {  // final: immutable value objects should be non-inheritable
    private final long minorUnits;  // Amount in cents (avoids floating point issues)
    private final Currency currency;

    public static final Money ZERO = new Money(0, Currency.USD);

    private Money(long minorUnits, Currency currency) {
        if (minorUnits < 0) throw new DomainException("Money cannot be negative");
        this.minorUnits = minorUnits;
        this.currency = currency;
    }

    public static Money of(long amount, Currency currency) {
        return new Money(amount, currency);
    }

    // Operations return NEW Money objects — never mutate the existing one
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new DomainException("Cannot add money in different currencies");
        }
        return new Money(this.minorUnits + other.minorUnits, this.currency);
    }

    public Money multiply(int factor) {
        return new Money(this.minorUnits * factor, this.currency);
    }

    // Value objects are equal when their VALUES are equal — not their references
    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof Money other)) return false;
        return minorUnits == other.minorUnits && currency.equals(other.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(minorUnits, currency);
    }
}
```

### Domain Events

A **Domain Event** represents something meaningful that happened within the domain — something the business cares about. `OrderPlaced`, `PaymentProcessed`, `OrderShipped`. Domain events are named in the past tense because they represent facts that have already occurred.

```java
// Domain events are immutable records of what happened — they are facts, not commands
public record OrderPlacedEvent(
    OrderId orderId,
    CustomerId customerId,
    List<OrderItem> items,
    Money totalAmount,
    Instant occurredAt
) implements DomainEvent {
    public OrderPlacedEvent(OrderId orderId, CustomerId customerId,
                             List<OrderItem> items, Money total) {
        this(orderId, customerId, items, total, Instant.now());
    }
}

// Publishing domain events from the aggregate root
// The aggregate collects events during its lifetime; the application service publishes them
public class Order {
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public void place() {
        // Business logic...
        domainEvents.add(new OrderPlacedEvent(this.id, this.customerId, this.items, calculateTotal()));
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(this.domainEvents);
        this.domainEvents.clear(); // Clear after pulling — events should only be processed once
        return events;
    }
}

// In the application service:
public OrderId placeOrder(PlaceOrderCommand command) {
    Order order = Order.place(command.customerId(), command.items());
    orderRepository.save(order);

    // Pull and publish all events raised during this transaction
    order.pullDomainEvents().forEach(eventPublisher::publish);
    // The Notification Service, Analytics Service, etc. will react to OrderPlacedEvent
    // The Order Service has no knowledge of who subscribes — pure decoupling

    return order.getId();
}
```

### Repositories in DDD

In DDD, a **Repository** is a collection-like interface that provides access to aggregate roots. It abstracts the persistence mechanism entirely. The domain sees the repository as an in-memory collection of domain objects; the fact that it's backed by PostgreSQL is an implementation detail.

```java
// DDD Repository: looks like a collection from the domain's perspective
public interface OrderRepository {
    // Collection-like interface: add, find, remove
    void add(Order order);                              // Not "save" — "add to the collection"
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomer(CustomerId customerId);
    List<Order> findPendingOrdersOlderThan(Duration age);
    void remove(Order order);
}

// Application Service: Repositories in DDD
public interface CustomerRepository {
    Optional<Customer> findById(CustomerId id);
    void add(Customer customer);
    // Rich query methods that reflect domain concepts, not database operations
    List<Customer> findGoldCustomersWithPendingOrders();
}
```

---

## 6. Project Structure — Putting It All Together

Here is a complete package structure that embodies Hexagonal Architecture + DDD for a Java/Spring Boot service:

```
com.example.ordering/
├── domain/                     ← Enterprise Business Rules (innermost ring)
│   ├── model/
│   │   ├── Order.java          ← Aggregate Root
│   │   ├── OrderItem.java      ← Entity within aggregate
│   │   ├── Customer.java       ← Aggregate Root
│   │   ├── Money.java          ← Value Object
│   │   ├── OrderId.java        ← Value Object (typed ID)
│   │   └── CustomerId.java     ← Value Object (typed ID)
│   ├── event/
│   │   ├── DomainEvent.java    ← Marker interface
│   │   ├── OrderPlacedEvent.java
│   │   └── OrderConfirmedEvent.java
│   └── repository/
│       ├── OrderRepository.java    ← Driven Port (interface in domain layer)
│       └── CustomerRepository.java ← Driven Port (interface in domain layer)
│
├── application/                ← Application Business Rules (use cases)
│   ├── port/
│   │   ├── in/                 ← Driving Ports
│   │   │   ├── PlaceOrderUseCase.java
│   │   │   └── ConfirmOrderUseCase.java
│   │   └── out/                ← Driven Ports (additional infrastructure interfaces)
│   │       ├── PaymentGateway.java
│   │       └── EventPublisher.java
│   ├── command/
│   │   ├── PlaceOrderCommand.java  ← Boundary object (no domain annotations)
│   │   └── ConfirmOrderCommand.java
│   └── service/
│       └── OrderApplicationService.java ← Implements use cases, orchestrates domain
│
├── adapter/                    ← Interface Adapters + Infrastructure (outer rings)
│   ├── in/                     ← Driving Adapters
│   │   ├── rest/
│   │   │   ├── OrderRestController.java
│   │   │   ├── OrderRequest.java    ← REST DTO (no domain objects in REST layer)
│   │   │   └── OrderResponse.java
│   │   └── messaging/
│   │       └── OrderKafkaConsumer.java
│   └── out/                    ← Driven Adapters
│       ├── persistence/
│       │   ├── JpaOrderRepositoryAdapter.java  ← Implements OrderRepository
│       │   ├── OrderJpaEntity.java  ← JPA-annotated entity (stays in infrastructure)
│       │   └── OrderMapper.java     ← Maps between domain Order and JPA OrderJpaEntity
│       ├── payment/
│       │   └── StripePaymentGatewayAdapter.java ← Implements PaymentGateway
│       └── messaging/
│           └── KafkaEventPublisherAdapter.java ← Implements EventPublisher
│
└── config/                     ← Spring wiring — assembles the hexagon
    └── ApplicationConfig.java  ← @Configuration, @Bean, dependency injection
```

---

## 7. Important Points & Best Practices

**Don't let JPA entities escape the adapter layer.** A JPA entity annotated with `@Entity`, `@Table`, `@Column`, and `@OneToMany` is an infrastructure object — it reflects database schema, not domain logic. If you use your JPA entity as your domain model, you'll find yourself adding `@JsonIgnore` for serialization concerns, `@Transient` for computed fields, and `@NotNull` for validation — all mixing infrastructure concerns into your domain. Keep them separate. Use a mapper to convert between domain objects and JPA entities.

**Value Objects are your best friend for type safety.** Instead of `void processOrder(Long customerId, Long productId, Long orderId)` (easy to pass arguments in the wrong order), use `void processOrder(CustomerId customerId, ProductId productId, OrderId orderId)`. The compiler will catch the mistake of swapping two IDs. Typed identifiers cost almost nothing to implement (a record wrapping a UUID or Long) and prevent an entire class of bugs.

**DDD tactical patterns are not all-or-nothing.** You don't have to implement every DDD pattern in every part of your codebase. Apply Aggregates and Value Objects rigorously in your core domain. Use simpler CRUD patterns for ancillary concerns like configuration management or audit logging. The goal is appropriate modeling, not dogmatic pattern application.

**Start with the Use Cases.** When building a new feature, start by writing the use case interface and implementing the application service. Mock all the ports. This forces you to define the domain model and business rules first, before thinking about REST endpoints or database schema. The infrastructure (REST, JPA) should emerge to support the use case — not drive its design.

**Ubiquitous Language requires ongoing maintenance.** Hold regular review sessions with domain experts where you look at the code together and ask "does this match how you think about the business?" The language evolves as understanding deepens. Renaming a class from `User` to `Customer` or splitting `Order` into `DraftOrder` and `ConfirmedOrder` based on domain expert feedback is not refactoring — it's alignment. It makes the code easier for everyone to reason about.
