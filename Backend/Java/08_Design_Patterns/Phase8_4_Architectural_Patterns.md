# Phase 8.4 — Architectural Patterns in Java

> **What are Architectural Patterns?**
> While GoF design patterns solve class- and object-level problems, **architectural patterns** solve system-level problems. They define how the major components of an application relate to each other, how responsibilities are divided, and how data flows. Choosing the right architectural pattern for a system is one of the most impactful decisions a software lead makes — it shapes maintainability, testability, and scalability for years.

---

## 1. MVC — Model-View-Controller

### Theory

MVC is one of the oldest and most influential architectural patterns. It divides an application into **three distinct layers**, each with a single responsibility:

- **Model** — holds the business data and logic. Knows nothing about how data is displayed.
- **View** — is the presentation layer. Renders the Model's data to the user. Knows nothing about business rules.
- **Controller** — acts as the middleman. Receives user input, updates the Model, and decides which View to show.

The key insight behind MVC is **separation of concerns**: changing your database schema (Model) should not require touching the HTML templates (View), and vice versa.

The **flow of control** goes like this:

```
User → Controller → Model (updates state) → View (reads state) → User
```

### Spring MVC Example

Spring MVC is the most widely used implementation of this pattern in the Java world. The `DispatcherServlet` acts as a central Controller front-controller.

```java
// THE MODEL — plain Java object representing business data
// In Spring, models are typically plain POJOs or JPA entities
public class Product {
    private Long id;
    private String name;
    private double price;
    private int stockQuantity;

    // Constructors, getters, setters (or use records/Lombok)
    public Product(Long id, String name, double price, int stockQuantity) {
        this.id            = id;
        this.name          = name;
        this.price         = price;
        this.stockQuantity = stockQuantity;
    }

    public Long getId()            { return id; }
    public String getName()        { return name; }
    public double getPrice()       { return price; }
    public int getStockQuantity()  { return stockQuantity; }
    public void setStockQuantity(int qty) { this.stockQuantity = qty; }
}

// THE SERVICE (MODEL layer — business logic)
// Separating business rules from the controller keeps the controller thin
@Service
public class ProductService {
    private final Map<Long, Product> store = new HashMap<>(Map.of(
        1L, new Product(1L, "Laptop",    999.99, 10),
        2L, new Product(2L, "Headphones", 79.99, 50)
    ));

    public List<Product> getAllProducts() {
        return new ArrayList<>(store.values());
    }

    public Optional<Product> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }

    // Business rule lives here, not in the controller
    public boolean purchaseProduct(Long id, int quantity) {
        Product product = store.get(id);
        if (product == null || product.getStockQuantity() < quantity) {
            return false;
        }
        product.setStockQuantity(product.getStockQuantity() - quantity);
        return true;
    }
}

// THE CONTROLLER — receives HTTP requests, delegates to service, chooses view
@RestController           // marks this as a REST controller (View = JSON response)
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService; // injected — no 'new' here

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    // Handles GET /api/products → returns list of all products
    @GetMapping
    public ResponseEntity<List<Product>> listProducts() {
        return ResponseEntity.ok(productService.getAllProducts());
    }

    // Handles GET /api/products/{id}
    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        return productService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // Handles POST /api/products/{id}/purchase?quantity=3
    @PostMapping("/{id}/purchase")
    public ResponseEntity<String> purchase(@PathVariable Long id,
                                           @RequestParam int quantity) {
        boolean success = productService.purchaseProduct(id, quantity);
        return success
            ? ResponseEntity.ok("Purchase successful")
            : ResponseEntity.badRequest().body("Insufficient stock or product not found");
    }
}
```

In this example, the **JSON response is the View** — in a traditional web app with Thymeleaf or JSP, the view would be an HTML template that the controller selects by name.

---

### ⚠️ Important Points & Best Practices

The most common mistake in MVC is putting business logic in the controller (a "Fat Controller"). The controller should be thin — its job is to translate HTTP requests into service calls and HTTP responses. All the real work lives in the Model/Service layer.

Always inject the service into the controller through the constructor (not field injection via `@Autowired`), because constructor injection makes the dependency explicit and enables easy unit testing without a Spring container.

---

## 2. MVP — Model-View-Presenter

### Theory

MVP evolved from MVC to improve testability, particularly for desktop UIs and Android applications. The key change is the role of the Presenter:

- **Model** — same as in MVC: business data and logic.
- **View** — even more passive than in MVC. In MVP, the View is a dumb interface that the Presenter drives. The View should not contain any logic — it just displays what the Presenter tells it to.
- **Presenter** — contains all the presentation logic. Unlike MVC's Controller, the Presenter communicates with the View through an interface, which makes it trivially testable (you can mock the View).

The crucial distinction: **the View and Model never communicate directly**. Everything goes through the Presenter.

```
User → View (thin) → Presenter → Model
                  ←  Presenter (updates View via interface)
```

### Example — Login Form

```java
// THE VIEW INTERFACE — defines what the Presenter can tell the View to do
// The real implementation (a Swing form, Android Activity, etc.) hides behind this
interface LoginView {
    String getUsername();
    String getPassword();
    void showLoading(boolean show);
    void showError(String message);
    void navigateToDashboard(String username);
}

// THE MODEL — authentication logic
class AuthService {
    private static final Map<String, String> users = Map.of(
        "alice", "password123",
        "admin", "securePass!"
    );

    public boolean authenticate(String username, String password) {
        return password.equals(users.get(username));
    }
}

// THE PRESENTER — all UI logic lives here, completely testable
class LoginPresenter {
    private final LoginView view;       // communicates through interface — easy to mock
    private final AuthService authService;

    public LoginPresenter(LoginView view, AuthService authService) {
        this.view        = view;
        this.authService = authService;
    }

    // Called when the user taps the login button
    public void onLoginClicked() {
        String username = view.getUsername().trim();
        String password = view.getPassword();

        // Presentation logic — validation before hitting the service
        if (username.isEmpty()) {
            view.showError("Username cannot be empty.");
            return;
        }
        if (password.isEmpty()) {
            view.showError("Password cannot be empty.");
            return;
        }

        view.showLoading(true);

        // Business rule checked in the service
        if (authService.authenticate(username, password)) {
            view.showLoading(false);
            view.navigateToDashboard(username); // tell the view what to do next
        } else {
            view.showLoading(false);
            view.showError("Invalid username or password.");
        }
    }
}

// THE REAL VIEW — would be a Swing JPanel, JavaFX scene, etc.
// For demonstration, a console-based stub:
class ConsoleLoginView implements LoginView {
    private final String usernameInput;
    private final String passwordInput;

    public ConsoleLoginView(String username, String password) {
        this.usernameInput = username;
        this.passwordInput = password;
    }

    @Override public String getUsername() { return usernameInput; }
    @Override public String getPassword() { return passwordInput; }
    @Override public void showLoading(boolean show) {
        System.out.println(show ? "Loading..." : "Done loading.");
    }
    @Override public void showError(String message) {
        System.out.println("ERROR: " + message);
    }
    @Override public void navigateToDashboard(String username) {
        System.out.println("Welcome, " + username + "! Navigating to dashboard.");
    }
}

// UNIT TEST — no UI framework needed! Just mock the interface.
class LoginPresenterTest {
    public static void main(String[] args) {
        AuthService auth = new AuthService();

        System.out.println("--- Test: valid credentials ---");
        LoginView validView = new ConsoleLoginView("alice", "password123");
        new LoginPresenter(validView, auth).onLoginClicked();

        System.out.println("\n--- Test: wrong password ---");
        LoginView badPassView = new ConsoleLoginView("alice", "wrongpass");
        new LoginPresenter(badPassView, auth).onLoginClicked();

        System.out.println("\n--- Test: empty username ---");
        LoginView emptyView = new ConsoleLoginView("", "anything");
        new LoginPresenter(emptyView, auth).onLoginClicked();
    }
}
```

The Presenter contains zero UI framework code, so you can test every login scenario with a simple JUnit test and a mock `LoginView`, without starting any UI framework.

---

### ⚠️ Important Points & Best Practices

MVP's main advantage over MVC is **testability** — the Presenter is just a plain Java class. The tradeoff is more boilerplate: every View needs an interface, and every user action needs to be channeled through the Presenter. For large UIs this discipline pays off enormously in test coverage.

---

## 3. MVVM — Model-View-ViewModel

### Theory

MVVM is designed for **data-binding frameworks** where the View can automatically observe and react to changes in data. It's the dominant pattern in JavaFX, Android (with LiveData/ViewModel), and frameworks like Angular or React.

- **Model** — business data and logic, same as always.
- **View** — observes the ViewModel through data bindings. When ViewModel properties change, the View updates automatically.
- **ViewModel** — exposes the Model's data as observable properties that the View binds to. It transforms raw Model data into forms that are easy for the View to display (e.g., converting a `LocalDate` to a formatted `String`). It also handles user commands.

The key innovation over MVP is that **the ViewModel has no reference to the View**. The View reaches out to the ViewModel through bindings — the ViewModel doesn't need to know if any View is listening.

### JavaFX Example

```java
import javafx.beans.property.*;
import javafx.collections.*;

// THE MODEL
public class User {
    private String username;
    private int age;
    private boolean active;

    public User(String username, int age, boolean active) {
        this.username = username;
        this.age      = age;
        this.active   = active;
    }

    // Getters and setters...
    public String getUsername() { return username; }
    public int getAge()         { return age; }
    public boolean isActive()   { return active; }
    public void setActive(boolean active) { this.active = active; }
}

// THE VIEWMODEL — exposes JavaFX observable properties for the View to bind to
public class UserViewModel {
    // Observable properties — the View binds to these, not to raw Model fields
    private final StringProperty  username    = new SimpleStringProperty();
    private final IntegerProperty age         = new SimpleIntegerProperty();
    private final BooleanProperty active      = new SimpleBooleanProperty();
    private final StringProperty  displayName = new SimpleStringProperty();
    private final StringProperty  statusText  = new SimpleStringProperty();
    private final StringProperty  errorMessage = new SimpleStringProperty("");

    private User currentUser;

    // Load a user from the Model into observable properties
    public void loadUser(User user) {
        this.currentUser = user;
        username.set(user.getUsername());
        age.set(user.getAge());
        active.set(user.isActive());

        // Transform raw data for display — ViewModel's key responsibility
        displayName.set(user.getUsername().substring(0, 1).toUpperCase()
                      + user.getUsername().substring(1)); // capitalize
        updateStatusText();
    }

    private void updateStatusText() {
        statusText.set(active.get()
            ? "Account is ACTIVE"
            : "Account is DEACTIVATED");
    }

    // Command method — called when user clicks "Toggle Active" button
    public void toggleActive() {
        if (currentUser == null) {
            errorMessage.set("No user loaded.");
            return;
        }
        boolean newStatus = !currentUser.isActive();
        currentUser.setActive(newStatus);
        active.set(newStatus);
        updateStatusText();
        errorMessage.set(""); // clear any previous error
    }

    // Property accessors — the View binds to these
    public StringProperty  usernameProperty()    { return username; }
    public IntegerProperty ageProperty()         { return age; }
    public BooleanProperty activeProperty()      { return active; }
    public StringProperty  displayNameProperty() { return displayName; }
    public StringProperty  statusTextProperty()  { return statusText; }
    public StringProperty  errorMessageProperty(){ return errorMessage; }
}

// THE VIEW (FXML-based in a real app — shown here as pseudo-code for clarity)
//
// In FXML, bindings look like:
//   <Label text="${viewModel.displayName}" />
//   <Label text="${viewModel.statusText}" />
//   <Button onAction="#toggleActive" />
//
// In code-behind (Controller in FXML terminology):
// label.textProperty().bind(viewModel.displayNameProperty());
// statusLabel.textProperty().bind(viewModel.statusTextProperty());
// errorLabel.textProperty().bind(viewModel.errorMessageProperty());
// toggleButton.setOnAction(e -> viewModel.toggleActive());
```

---

### ⚠️ Important Points & Best Practices

MVVM shines when the UI framework supports **two-way data binding**. Without binding support, you end up writing boilerplate that manually copies data between the ViewModel and View — at that point MVP is simpler.

The ViewModel must never import or reference UI classes (`javafx.scene.*`, `android.widget.*`). This is the line that ensures testability — a ViewModel test is a plain JUnit test with no UI involved.

---

## 4. Repository Pattern

### Theory

The Repository pattern **abstracts the data access layer** behind an interface. The business logic (services) talks to a Repository interface; the actual persistence mechanism (SQL database, MongoDB, in-memory cache, external API) is hidden behind that interface. Business code should not know or care how data is stored.

Think of a Repository as a collection interface for domain objects — to the service, it looks like a simple in-memory collection with `findById`, `save`, and `delete`. What happens behind the scenes is the Repository's concern alone.

```
Service → Repository Interface → [JPA Implementation | MongoDB Implementation | In-Memory Test Implementation]
```

### Example

```java
// DOMAIN ENTITY
public class Order {
    private final Long id;
    private final String customerId;
    private final double totalAmount;
    private String status; // PENDING, PROCESSING, SHIPPED, DELIVERED

    public Order(Long id, String customerId, double totalAmount) {
        this.id          = id;
        this.customerId  = customerId;
        this.totalAmount = totalAmount;
        this.status      = "PENDING";
    }

    public Long getId()           { return id; }
    public String getCustomerId() { return customerId; }
    public double getTotalAmount(){ return totalAmount; }
    public String getStatus()     { return status; }
    public void setStatus(String status) { this.status = status; }

    @Override
    public String toString() {
        return String.format("Order[id=%d, customer=%s, total=%.2f, status=%s]",
            id, customerId, totalAmount, status);
    }
}

// REPOSITORY INTERFACE — pure contract, no implementation details leak out
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(Long id);
    List<Order> findByCustomerId(String customerId);
    List<Order> findByStatus(String status);
    void delete(Long id);
    List<Order> findAll();
}

// PRODUCTION IMPLEMENTATION — could use JPA, JDBC, etc.
// (Shown here as an in-memory store for clarity)
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<Long, Order> store = new ConcurrentHashMap<>();

    @Override
    public void save(Order order) {
        store.put(order.getId(), order);
    }

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<Order> findByCustomerId(String customerId) {
        return store.values().stream()
            .filter(o -> o.getCustomerId().equals(customerId))
            .collect(java.util.stream.Collectors.toList());
    }

    @Override
    public List<Order> findByStatus(String status) {
        return store.values().stream()
            .filter(o -> o.getStatus().equals(status))
            .collect(java.util.stream.Collectors.toList());
    }

    @Override
    public void delete(Long id) {
        store.remove(id);
    }

    @Override
    public List<Order> findAll() {
        return new ArrayList<>(store.values());
    }
}

// BUSINESS SERVICE — depends only on the interface, never the implementation
public class OrderService {
    private final OrderRepository orderRepository; // injected

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public Order createOrder(Long id, String customerId, double amount) {
        Order order = new Order(id, customerId, amount);
        orderRepository.save(order);
        System.out.println("Created: " + order);
        return order;
    }

    public void shipOrder(Long orderId) {
        orderRepository.findById(orderId).ifPresentOrElse(order -> {
            if (!"PROCESSING".equals(order.getStatus())) {
                throw new IllegalStateException("Can only ship PROCESSING orders");
            }
            order.setStatus("SHIPPED");
            orderRepository.save(order);
            System.out.println("Shipped: " + order);
        }, () -> {
            throw new RuntimeException("Order not found: " + orderId);
        });
    }

    public List<Order> getPendingOrders() {
        return orderRepository.findByStatus("PENDING");
    }
}
```

---

### ⚠️ Important Points & Best Practices

The true value of the Repository pattern shows at test time: you can pass an `InMemoryOrderRepository` to your service tests, making them blazing fast and completely independent of any database. No Docker, no test containers, no H2 setup needed for unit tests.

In Spring Data JPA, when you extend `JpaRepository`, Spring auto-generates the implementation at runtime — you get the Repository pattern without writing SQL or boilerplate for common operations.

Avoid letting the Repository leak persistence details into its interface. Methods like `findWithJoinFetchOnProducts()` or `findUsingNativeQuery()` are implementation details that don't belong on the interface.

---

## 5. Service Layer Pattern

### Theory

The Service Layer pattern defines an **application's boundary and its set of available operations** from the perspective of interfacing clients. It encapsulates the application's business logic and controls transactions and coordination responses in the implementation of its operations.

It's the answer to the question: *"Where does business logic live?"* Not in controllers (they're for HTTP plumbing). Not in repositories (they're for data access). The Service Layer is the place.

```
HTTP Request → Controller → Service → Repository → Database
                  ↓ (thin)     ↑ (thick)    ↑ (thin)
```

### Example — Order Processing Service with Transaction Management

```java
// The service layer orchestrates multiple repositories and enforces business rules
@Service
@Transactional // Spring manages transactions at the service layer
public class OrderProcessingService {

    private final OrderRepository    orderRepository;
    private final InventoryRepository inventoryRepository;
    private final PaymentService     paymentService;
    private final NotificationService notificationService;

    // All dependencies injected through constructor — easy to test
    public OrderProcessingService(
            OrderRepository orderRepository,
            InventoryRepository inventoryRepository,
            PaymentService paymentService,
            NotificationService notificationService) {
        this.orderRepository      = orderRepository;
        this.inventoryRepository  = inventoryRepository;
        this.paymentService       = paymentService;
        this.notificationService  = notificationService;
    }

    // A "use case" — one complete business operation
    // The @Transactional ensures ALL steps complete or ALL roll back
    public OrderConfirmation placeOrder(PlaceOrderRequest request) {
        // Step 1: Validate the order
        if (request.getItems().isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one item");
        }

        // Step 2: Check inventory for each item (calls repository)
        for (OrderItem item : request.getItems()) {
            int available = inventoryRepository.getAvailableQuantity(item.getProductId());
            if (available < item.getQuantity()) {
                throw new InsufficientStockException(
                    "Not enough stock for product: " + item.getProductId());
            }
        }

        // Step 3: Reserve inventory
        inventoryRepository.reserve(request.getItems());

        // Step 4: Process payment (calls external service)
        PaymentResult payment = paymentService.charge(
            request.getCustomerId(), request.getTotalAmount());
        if (!payment.isSuccessful()) {
            // reservation will be rolled back by @Transactional if exception thrown
            throw new PaymentFailedException("Payment declined: " + payment.getReason());
        }

        // Step 5: Create and save the order
        Order order = new Order(
            generateOrderId(), request.getCustomerId(), request.getTotalAmount());
        order.setStatus("PROCESSING");
        orderRepository.save(order);

        // Step 6: Send confirmation (async — doesn't block the transaction)
        notificationService.sendOrderConfirmation(order, request.getCustomerEmail());

        return new OrderConfirmation(order.getId(), "Order placed successfully");
    }

    private Long generateOrderId() {
        return System.currentTimeMillis(); // simplified
    }
}
```

The service coordinates across multiple repositories and external services. If payment fails, the `@Transactional` annotation ensures the inventory reservation is rolled back automatically.

---

### ⚠️ Important Points & Best Practices

The service layer is the right home for `@Transactional` — not in controllers (too high) and not in repositories (too low). Transaction boundaries should align with complete business operations.

Keep services free of HTTP concerns (no `HttpServletRequest`, no `ResponseEntity`). A service should be callable from a REST controller, a message queue consumer, a scheduler job, or a test — it shouldn't care about the transport layer.

---

## 6. DAO — Data Access Object Pattern

### Theory

The DAO pattern is specifically focused on **abstracting and encapsulating all database access**. A DAO provides CRUD operations for a single data entity and hides the SQL or ORM details from the rest of the application.

DAO is similar to Repository, but with a subtle difference in intent: a DAO is closer to the database (one DAO per table/entity), while a Repository is closer to the domain (one Repository per aggregate root, which may span multiple tables). In practice, many applications use the terms interchangeably.

### Example — JDBC-Based DAO

```java
import java.sql.*;
import java.util.*;

// ENTITY
public class CustomerEntity {
    private Long id;
    private String name;
    private String email;

    public CustomerEntity(Long id, String name, String email) {
        this.id    = id;
        this.name  = name;
        this.email = email;
    }

    public Long getId()    { return id; }
    public String getName() { return name; }
    public String getEmail(){ return email; }

    @Override
    public String toString() {
        return String.format("Customer[id=%d, name=%s, email=%s]", id, name, email);
    }
}

// DAO INTERFACE — defines the data access contract
public interface CustomerDAO {
    void insert(CustomerEntity customer) throws SQLException;
    Optional<CustomerEntity> findById(Long id) throws SQLException;
    List<CustomerEntity> findAll() throws SQLException;
    void update(CustomerEntity customer) throws SQLException;
    void delete(Long id) throws SQLException;
}

// JDBC IMPLEMENTATION — all SQL is contained here, business code stays clean
public class JdbcCustomerDAO implements CustomerDAO {

    private final DataSource dataSource; // connection pool (e.g., HikariCP)

    public JdbcCustomerDAO(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @Override
    public void insert(CustomerEntity customer) throws SQLException {
        String sql = "INSERT INTO customers (id, name, email) VALUES (?, ?, ?)";

        // try-with-resources ensures connection and statement are always closed
        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setLong(1, customer.getId());
            stmt.setString(2, customer.getName());
            stmt.setString(3, customer.getEmail());
            stmt.executeUpdate();
            System.out.println("Inserted: " + customer);
        }
        // Connection automatically returned to pool when this block exits
    }

    @Override
    public Optional<CustomerEntity> findById(Long id) throws SQLException {
        String sql = "SELECT id, name, email FROM customers WHERE id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setLong(1, id);

            try (ResultSet rs = stmt.executeQuery()) {
                if (rs.next()) {
                    return Optional.of(mapRow(rs)); // mapping stays in DAO
                }
            }
        }
        return Optional.empty();
    }

    @Override
    public List<CustomerEntity> findAll() throws SQLException {
        String sql = "SELECT id, name, email FROM customers ORDER BY id";
        List<CustomerEntity> result = new ArrayList<>();

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql);
             ResultSet rs = stmt.executeQuery()) {

            while (rs.next()) {
                result.add(mapRow(rs));
            }
        }
        return result;
    }

    @Override
    public void update(CustomerEntity customer) throws SQLException {
        String sql = "UPDATE customers SET name = ?, email = ? WHERE id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setString(1, customer.getName());
            stmt.setString(2, customer.getEmail());
            stmt.setLong(3, customer.getId());
            int rows = stmt.executeUpdate();
            System.out.println("Updated " + rows + " row(s) for customer id=" + customer.getId());
        }
    }

    @Override
    public void delete(Long id) throws SQLException {
        String sql = "DELETE FROM customers WHERE id = ?";

        try (Connection conn = dataSource.getConnection();
             PreparedStatement stmt = conn.prepareStatement(sql)) {

            stmt.setLong(1, id);
            stmt.executeUpdate();
        }
    }

    // Private helper — maps a ResultSet row to an entity (no leaking of DB structure)
    private CustomerEntity mapRow(ResultSet rs) throws SQLException {
        return new CustomerEntity(
            rs.getLong("id"),
            rs.getString("name"),
            rs.getString("email")
        );
    }
}
```

---

### DAO vs Repository: When to Use Which

Both patterns serve the same broad purpose, but the choice reflects your architecture's maturity:

Use a **DAO** when your application is fairly CRUD-oriented, you're working directly with JDBC or simple ORM, and you don't have a rich domain model. The DAO maps closely to your database schema — essentially one class per table.

Use a **Repository** when you're applying Domain-Driven Design (DDD), your domain has Aggregates (clusters of entities that are always managed together), and you want your business code to think in terms of business objects rather than database rows. The Repository is a higher-level abstraction.

---

## 7. Domain-Driven Design (DDD) — Core Concepts

### Theory

Domain-Driven Design is an approach to software development that **centers on the core domain** — the business problem you're solving — rather than on technology. Eric Evans' seminal book coined the core vocabulary. DDD is relevant once your codebase becomes large enough that the biggest challenge is managing complexity, not just writing code.

The foundational idea is: **your code should speak the same language as your domain experts**. When business people talk about "Order", "Invoice", "Customer", your code should have classes named `Order`, `Invoice`, `Customer` — not `OrderBean`, `InvoiceDTO`, `CustomerRecord`.

---

### Core DDD Building Blocks

**Ubiquitous Language** is the agreement between developers and domain experts to use the same words for the same concepts. It gets embedded in class names, method names, and variable names. If the business says "fulfill an order", your code should have `order.fulfill()` — not `order.setStatusToShipped()`.

**Bounded Context** is perhaps the most powerful concept in DDD. A large system is divided into multiple bounded contexts — each with its own model and language. The word "Customer" can mean different things in the Sales context (a lead, a prospect) versus the Billing context (a payer) versus the Shipping context (a delivery address). Each context has its own `Customer` class with only the fields and methods that matter in that context. Bounded contexts communicate through well-defined interfaces (APIs, events).

```java
// In the BILLING bounded context:
public class Customer {
    private final String id;
    private final BillingAddress billingAddress;
    private final PaymentMethod  preferredPaymentMethod;
    private final CreditLimit    creditLimit;
    // No shipping address here — that belongs in the Shipping context
}

// In the SHIPPING bounded context — a different Customer class:
public class Customer {
    private final String id;
    private final String name;
    private final List<ShippingAddress> addresses;
    private final ShippingPreference    preference;
    // No payment methods here — irrelevant in this context
}
```

**Entities** are objects with a **unique identity** that persists over time. Even if all of an entity's attributes change, it's still the same entity. A `Customer` is an entity — even if Alice changes her name and address, she's still Alice (identity = `customerId`). Entities are compared by their ID, not their fields.

```java
// Entity — identity is the id, not the field values
public class Customer {
    private final CustomerId id; // Value Object for the ID
    private String name;
    private Email  email;

    // Equality based on identity ONLY
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Customer other)) return false;
        return id.equals(other.id); // identity check, not field comparison
    }

    @Override
    public int hashCode() {
        return id.hashCode();
    }
}
```

**Value Objects** have **no identity** — they are defined entirely by their attributes. Two `Money` objects representing $50 are interchangeable — there's no notion of "which $50 is this?" Value Objects should be immutable.

```java
// Value Object — immutable, equality by value, no identity
public final class Money {
    private final BigDecimal amount;
    private final Currency   currency;

    public Money(BigDecimal amount, Currency currency) {
        Objects.requireNonNull(amount,   "Amount cannot be null");
        Objects.requireNonNull(currency, "Currency cannot be null");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Money cannot be negative");
        }
        this.amount   = amount;
        this.currency = currency;
    }

    // Business behavior lives on the Value Object
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency); // new instance — immutable
    }

    public Money multiply(int factor) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(factor)), this.currency);
    }

    public boolean isGreaterThan(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot compare different currencies");
        }
        return this.amount.compareTo(other.amount) > 0;
    }

    // Equality by value — not identity
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money other)) return false;
        return amount.compareTo(other.amount) == 0
            && currency.equals(other.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }

    @Override
    public String toString() {
        return currency.getSymbol() + amount.toPlainString();
    }
}
```

**Aggregates** are clusters of related entities and value objects that are treated as a single unit for data changes. The **Aggregate Root** is the only object in the cluster that the outside world can hold a reference to. All changes to the cluster must go through the root, which enforces invariants (business rules) across the whole aggregate.

```java
// Aggregate Root — controls the entire Order aggregate
// Invariant: total order amount must equal the sum of line items
public class Order {
    private final OrderId        id;
    private final CustomerId     customerId;
    private final List<OrderLine> orderLines = new ArrayList<>();
    private OrderStatus          status;

    public Order(OrderId id, CustomerId customerId) {
        this.id         = id;
        this.customerId = customerId;
        this.status     = OrderStatus.PENDING;
    }

    // Outside world calls addLine() on the root — root enforces rules
    public void addLine(ProductId productId, int quantity, Money unitPrice) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Cannot modify a non-pending order");
        }
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        orderLines.add(new OrderLine(productId, quantity, unitPrice));
    }

    // Business logic encapsulated in the root
    public Money calculateTotal() {
        return orderLines.stream()
            .map(OrderLine::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    public void confirm() {
        if (orderLines.isEmpty()) {
            throw new IllegalStateException("Cannot confirm an empty order");
        }
        this.status = OrderStatus.CONFIRMED;
        // Could register a domain event here: new OrderConfirmedEvent(this.id)
    }

    public OrderId getId()           { return id; }
    public OrderStatus getStatus()   { return status; }
    public List<OrderLine> getLines() { return Collections.unmodifiableList(orderLines); }
}

// OrderLine is INSIDE the Order aggregate — never referenced directly from outside
class OrderLine {
    private final ProductId productId;
    private final int       quantity;
    private final Money     unitPrice;

    OrderLine(ProductId productId, int quantity, Money unitPrice) {
        this.productId = productId;
        this.quantity  = quantity;
        this.unitPrice = unitPrice;
    }

    public Money getSubtotal() {
        return unitPrice.multiply(quantity);
    }
}
```

**Domain Events** are things that happen in the domain that domain experts care about — "OrderPlaced", "PaymentReceived", "ItemShipped". Publishing domain events allows bounded contexts to communicate without direct coupling.

**Application Services** vs **Domain Services**: an Application Service orchestrates the use case flow (transaction, security, loading aggregates). A Domain Service contains domain logic that doesn't naturally belong to any single entity (e.g., a `TransferService` that moves money between two `Account` aggregates).

---

### ⚠️ Important Points & Best Practices

DDD is not about using all these patterns everywhere — that would be massive overkill for a simple CRUD application. Apply DDD where the domain is genuinely complex and worth modeling carefully. For simple CRUD screens, a straightforward 3-layer architecture (Controller → Service → Repository) is the right choice.

The most actionable DDD practice for most teams is adopting Ubiquitous Language and Bounded Contexts. Even without implementing Aggregates and Value Objects rigorously, using the domain's vocabulary in your code and keeping contexts cleanly separated will dramatically improve your codebase's clarity and maintainability.

---

## Choosing the Right Architectural Pattern

One of the most important judgment calls a software lead makes is matching the pattern to the problem. Here is a practical heuristic:

A **simple CRUD application** (admin panel, basic REST API) benefits most from a clean 3-layer architecture: Controller, Service, and Repository. Applying DDD here would add complexity without benefit.

A **medium-sized business application** with meaningful business rules benefits from adding the Repository pattern, a clear Service Layer with transaction management, and well-defined DTO/Entity separation.

A **large enterprise application** with a complex domain, many teams, and independent scaling requirements benefits from DDD with Bounded Contexts, possibly CQRS, domain events, and eventually microservices — but only when the complexity genuinely justifies the overhead.

For **UI-driven applications**, choose MVC for server-side rendered web apps (Spring MVC + Thymeleaf), MVP for testable desktop or form-based UIs (JavaFX with manual binding), and MVVM for data-binding-heavy UIs (JavaFX properties, Android ViewModel).

The common thread across all of these is the same principle: **separate what changes for different reasons** (data access from business logic from presentation), and **keep each layer ignorant of the layers it doesn't need to know about**.

---

## Summary Table — Architectural Patterns

| Pattern | Primary Goal | Best Fit |
|---------|-------------|----------|
| MVC | Separate UI, logic, and data | Spring MVC, server-side web apps |
| MVP | Maximize UI testability | Desktop UIs, Android without data binding |
| MVVM | Reactive UI via data binding | JavaFX, Android ViewModel, reactive frontends |
| Repository | Abstract data access behind an interface | Any app touching a database |
| Service Layer | Define application boundary and business operations | All non-trivial business apps |
| DAO | Isolate raw database/SQL operations | JDBC-heavy apps, simple CRUD systems |
| DDD | Manage complex domain logic | Large, complex business domains with rich rules |

---

*End of Phase 8 — Design Patterns*
