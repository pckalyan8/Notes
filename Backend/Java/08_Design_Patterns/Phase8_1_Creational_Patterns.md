# Phase 8.1 — Creational Design Patterns in Java

> **What are Creational Patterns?**
> Creational patterns deal with **object creation mechanisms**. The core idea is to separate the process of creating an object from the code that uses it. This gives you flexibility in deciding *which* object to create, *how* to create it, and *when* to create it — without coupling the client code to specific classes.

---

## 1. Singleton Pattern

### Theory

The Singleton pattern ensures that **only one instance of a class** exists throughout the application's lifetime, and provides a global access point to that instance.

**When to use it:**
- Configuration managers (single config object shared across the app)
- Logger instances
- Thread pools
- Database connection pools
- Registry objects

**The core problem it solves:** Without Singleton, you risk creating multiple expensive objects (like DB connections) unnecessarily, or worse, having inconsistent state across multiple instances.

### Basic (Naive) Implementation — Not Thread-Safe

```java
public class Singleton {

    // The single instance — stored as a static field
    private static Singleton instance;

    // Private constructor prevents anyone from calling `new Singleton()`
    private Singleton() {
        System.out.println("Singleton instance created.");
    }

    // Public accessor — creates on first call, returns same instance thereafter
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton(); // PROBLEM: race condition in multithreaded env
        }
        return instance;
    }

    public void doWork() {
        System.out.println("Singleton doing work: " + this.hashCode());
    }
}
```

**Problem:** If two threads call `getInstance()` simultaneously when `instance == null`, both may pass the null check and create two separate instances — breaking the guarantee.

---

### Thread-Safe Variant 1 — Synchronized Method

```java
public class SynchronizedSingleton {
    private static SynchronizedSingleton instance;

    private SynchronizedSingleton() {}

    // `synchronized` ensures only one thread enters at a time
    // PROBLEM: every call acquires a lock — very slow under high contention
    public static synchronized SynchronizedSingleton getInstance() {
        if (instance == null) {
            instance = new SynchronizedSingleton();
        }
        return instance;
    }
}
```

---

### Thread-Safe Variant 2 — Double-Checked Locking (DCL) ✅ Recommended

```java
public class DCLSingleton {

    // `volatile` ensures visibility — all threads see the updated reference
    // Without volatile, the JVM can reorder instructions and a partially
    // constructed object might be returned to another thread
    private static volatile DCLSingleton instance;

    private DCLSingleton() {}

    public static DCLSingleton getInstance() {
        if (instance == null) {                 // First check — no lock (fast path)
            synchronized (DCLSingleton.class) { // Lock only when needed
                if (instance == null) {         // Second check — inside lock (safe)
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;
    }
}
```

**Why two null checks?** The first is for performance — the majority of calls skip the lock entirely. The second is for safety — only one thread should create the instance.

---

### Thread-Safe Variant 3 — Initialization-on-Demand Holder (Bill Pugh) ✅ Best

```java
public class HolderSingleton {

    private HolderSingleton() {}

    // Inner static class is loaded lazily by the JVM — only when first accessed
    // JVM class loading is inherently thread-safe, so no synchronization needed
    private static class SingletonHolder {
        private static final HolderSingleton INSTANCE = new HolderSingleton();
    }

    public static HolderSingleton getInstance() {
        return SingletonHolder.INSTANCE; // JVM guarantees safe publication
    }
}
```

This is the cleanest approach — lazy, thread-safe, and no synchronization overhead.

---

### Thread-Safe Variant 4 — Enum Singleton ✅ Josh Bloch's Recommendation

```java
public enum EnumSingleton {
    INSTANCE; // The single instance

    // Enum constants are initialized once by the JVM — inherently thread-safe
    // Also protects against serialization attacks (unlike other patterns)
    // Prevents reflection-based instantiation

    public void doWork() {
        System.out.println("Enum Singleton at work: " + this.hashCode());
    }
}

// Usage:
EnumSingleton.INSTANCE.doWork();
```

**Why Enum is best in most cases:** Java guarantees an enum value is instantiated exactly once. It also survives serialization/deserialization correctly (other variants don't, unless you implement `readResolve()`).

---

### ⚠️ Important Points & Best Practices

- Always use `volatile` with DCL — without it, the JVM's instruction reordering can cause subtle bugs.
- The Enum singleton is immune to reflection attacks (calling a private constructor via reflection throws an `IllegalArgumentException` for enums).
- Be cautious using Singleton in a dependency-injection framework like Spring — Spring manages beans as Singletons by default via its container, so you rarely need to implement the pattern manually.
- Singleton is often considered an anti-pattern in testable code because it introduces hidden global state and makes mocking difficult. Prefer injecting dependencies.

---

## 2. Factory Method Pattern

### Theory

The Factory Method pattern defines an **interface for creating an object**, but lets **subclasses decide which class to instantiate**. The factory method defers instantiation to subclasses.

Think of it like a franchise restaurant — the parent company defines the process ("make a burger"), but each franchise location decides the exact recipe.

**When to use it:**
- When the exact type of object to create isn't known until runtime
- When you want to give subclasses control over what they create
- To decouple the client from the concrete product classes

### Example — Document Creator

```java
// Step 1: Define the Product interface
interface Document {
    void open();
    void save();
}

// Step 2: Concrete Products
class PDFDocument implements Document {
    @Override
    public void open()  { System.out.println("Opening PDF document"); }
    @Override
    public void save()  { System.out.println("Saving PDF document"); }
}

class WordDocument implements Document {
    @Override
    public void open()  { System.out.println("Opening Word document"); }
    @Override
    public void save()  { System.out.println("Saving Word document"); }
}

// Step 3: Creator — declares the factory method
abstract class DocumentEditor {

    // The factory method — subclasses override this
    public abstract Document createDocument();

    // Template behavior that uses the factory method
    public void editDocument() {
        Document doc = createDocument(); // calls subclass implementation
        doc.open();
        System.out.println("Editing...");
        doc.save();
    }
}

// Step 4: Concrete Creators — each overrides the factory method
class PDFEditor extends DocumentEditor {
    @Override
    public Document createDocument() {
        return new PDFDocument(); // decides which product to create
    }
}

class WordEditor extends DocumentEditor {
    @Override
    public Document createDocument() {
        return new WordDocument();
    }
}

// Step 5: Client code — works with DocumentEditor, not concrete classes
public class FactoryMethodDemo {
    public static void main(String[] args) {
        DocumentEditor editor = new PDFEditor();
        editor.editDocument();

        editor = new WordEditor();
        editor.editDocument();
    }
}
```

**Output:**
```
Opening PDF document
Editing...
Saving PDF document
Opening Word document
Editing...
Saving Word document
```

The client only knows about `DocumentEditor` — it never calls `new PDFDocument()` directly.

---

### ⚠️ Important Points & Best Practices

- Factory Method follows the **Open/Closed Principle** — you can add new document types without changing existing code.
- It's very common in Java's standard library: `Calendar.getInstance()`, `NumberFormat.getInstance()`, `java.util.Iterator`.
- Don't confuse Factory Method (pattern with inheritance) with the Simple Factory (a helper class with a static method — which is not a GoF pattern).

---

## 3. Abstract Factory Pattern

### Theory

Abstract Factory provides an interface for creating **families of related or dependent objects** without specifying their concrete classes. If Factory Method creates one product, Abstract Factory creates a whole **suite of products** that belong together.

Think of it as a factory of factories. A "Dark Theme" factory creates a dark button, dark checkbox, and dark dialog — all visually consistent.

**When to use it:**
- When a system needs to be independent of how its products are created
- When you want to enforce families of related objects being used together
- UI toolkit theming, cross-platform UI, database drivers for multiple vendors

### Example — UI Component Factory

```java
// Product interfaces — define what each component can do
interface Button {
    void render();
    void onClick();
}

interface Checkbox {
    void render();
    void onCheck();
}

// Concrete Products for Windows family
class WindowsButton implements Button {
    @Override public void render()  { System.out.println("Rendering Windows-style button"); }
    @Override public void onClick() { System.out.println("Windows button clicked"); }
}

class WindowsCheckbox implements Checkbox {
    @Override public void render()  { System.out.println("Rendering Windows-style checkbox"); }
    @Override public void onCheck() { System.out.println("Windows checkbox checked"); }
}

// Concrete Products for Mac family
class MacButton implements Button {
    @Override public void render()  { System.out.println("Rendering Mac-style button"); }
    @Override public void onClick() { System.out.println("Mac button clicked"); }
}

class MacCheckbox implements Checkbox {
    @Override public void render()  { System.out.println("Rendering Mac-style checkbox"); }
    @Override public void onCheck() { System.out.println("Mac checkbox checked"); }
}

// Abstract Factory — declares methods to create each product in the family
interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

// Concrete Factories — each creates one consistent family
class WindowsFactory implements GUIFactory {
    @Override public Button   createButton()   { return new WindowsButton(); }
    @Override public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}

class MacFactory implements GUIFactory {
    @Override public Button   createButton()   { return new MacButton(); }
    @Override public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// Client — uses factory, never knows which OS it's running on
class Application {
    private final Button button;
    private final Checkbox checkbox;

    // Factory is injected — client is decoupled from concrete classes
    public Application(GUIFactory factory) {
        this.button   = factory.createButton();
        this.checkbox = factory.createCheckbox();
    }

    public void renderUI() {
        button.render();
        checkbox.render();
    }
}

// Entry point
public class AbstractFactoryDemo {
    public static void main(String[] args) {
        String os = "Windows"; // could come from config or environment

        GUIFactory factory = os.equals("Windows")
            ? new WindowsFactory()
            : new MacFactory();

        Application app = new Application(factory);
        app.renderUI();
    }
}
```

**Key insight:** The `Application` class is completely unaware of whether it's running on Windows or Mac. Swapping the entire UI family only requires changing the factory — nothing else.

---

### ⚠️ Important Points & Best Practices

- Abstract Factory vs Factory Method: Factory Method uses **inheritance** (subclass decides); Abstract Factory uses **composition** (factory object is injected).
- Supporting a new product family means adding a new concrete factory — easy. But adding a new product *type* (e.g., a `Slider`) requires changing the `GUIFactory` interface and all its implementations — harder.
- JDBC is a real-world example: `Connection`, `Statement`, `ResultSet` are created by the same driver factory, ensuring they're compatible.

---

## 4. Builder Pattern

### Theory

The Builder pattern **constructs a complex object step by step**. It separates the construction of an object from its representation, so the same construction process can create different representations.

**The problem it solves — Telescoping Constructor Anti-Pattern:**

```java
// Painful to use — what does true, false, true mean?
new Pizza("Large", "Thin", true, false, true, "Mozzarella", "Tomato");
```

With Builder, you make the construction readable and flexible — you only specify what you need.

**When to use it:**
- Object has many optional parameters
- Object requires a multi-step construction process
- You want immutable objects but with complex initialization

### Example — Building a Pizza

```java
// The product — immutable, all fields set via Builder
public class Pizza {
    // Required fields
    private final String size;
    private final String crustType;

    // Optional fields — have sensible defaults
    private final boolean extraCheese;
    private final boolean mushrooms;
    private final boolean pepperoni;
    private final String sauce;

    // Private constructor — only the Builder can call this
    private Pizza(Builder builder) {
        this.size        = builder.size;
        this.crustType   = builder.crustType;
        this.extraCheese = builder.extraCheese;
        this.mushrooms   = builder.mushrooms;
        this.pepperoni   = builder.pepperoni;
        this.sauce       = builder.sauce;
    }

    @Override
    public String toString() {
        return String.format(
            "Pizza[size=%s, crust=%s, cheese=%b, mushrooms=%b, pepperoni=%b, sauce=%s]",
            size, crustType, extraCheese, mushrooms, pepperoni, sauce
        );
    }

    // Static nested Builder class
    public static class Builder {
        // Required
        private final String size;
        private final String crustType;

        // Optional — with defaults
        private boolean extraCheese = false;
        private boolean mushrooms   = false;
        private boolean pepperoni   = false;
        private String  sauce       = "Tomato";

        // Builder constructor takes only required parameters
        public Builder(String size, String crustType) {
            this.size      = size;
            this.crustType = crustType;
        }

        // Each setter returns `this` so calls can be chained fluently
        public Builder extraCheese(boolean val) { this.extraCheese = val; return this; }
        public Builder mushrooms(boolean val)   { this.mushrooms   = val; return this; }
        public Builder pepperoni(boolean val)   { this.pepperoni   = val; return this; }
        public Builder sauce(String val)        { this.sauce       = val; return this; }

        // Terminal method — validates and creates the Pizza
        public Pizza build() {
            // Could add validation here, e.g.:
            if (size == null || size.isBlank()) {
                throw new IllegalStateException("Pizza size is required");
            }
            return new Pizza(this);
        }
    }
}

// Client code — reads almost like natural language
public class BuilderDemo {
    public static void main(String[] args) {
        Pizza pizza = new Pizza.Builder("Large", "Thin")
            .extraCheese(true)
            .pepperoni(true)
            .sauce("BBQ")
            .build();

        System.out.println(pizza);

        // Minimal pizza — only required fields
        Pizza simplePizza = new Pizza.Builder("Small", "Thick").build();
        System.out.println(simplePizza);
    }
}
```

**Output:**
```
Pizza[size=Large, crust=Thin, cheese=true, mushrooms=false, pepperoni=true, sauce=BBQ]
Pizza[size=Small, crust=Thick, cheese=false, mushrooms=false, pepperoni=false, sauce=Tomato]
```

---

### Builder with Lombok (Real-World Shortcut)

```java
import lombok.Builder;
import lombok.ToString;

// Lombok generates the entire Builder automatically at compile time
@Builder
@ToString
public class UserProfile {
    private final String username;
    private final String email;
    private final int age;
    private final String bio;
}

// Usage:
UserProfile user = UserProfile.builder()
    .username("alice")
    .email("alice@example.com")
    .age(30)
    .build();
```

---

### ⚠️ Important Points & Best Practices

- The built object should ideally be **immutable** — don't expose setters on the product.
- You can add **validation** inside `build()` before returning the object.
- Lombok's `@Builder` is excellent for reducing boilerplate, but understand the manual implementation first.
- Java's `StringBuilder` is the classic example of the Builder pattern in the standard library.
- Spring's `MockMvcRequestBuilders` is another real-world example.

---

## 5. Prototype Pattern

### Theory

The Prototype pattern creates new objects by **cloning an existing object** (the prototype) rather than constructing from scratch. This is useful when object creation is expensive and you need many similar objects.

**When to use it:**
- Object creation is costly (e.g., loading from a database or making a network call)
- You need many objects that differ only slightly from each other
- You want to avoid subclassing just to create variations

### Example — Cloning a Game Character

```java
// Step 1: Implement Cloneable (or provide your own copy method)
class GameCharacter implements Cloneable {
    private String name;
    private String weaponType;
    private int health;
    private List<String> inventory; // mutable — shallow clone would share this reference!

    public GameCharacter(String name, String weaponType, int health) {
        this.name       = name;
        this.weaponType = weaponType;
        this.health     = health;
        this.inventory  = new ArrayList<>();
    }

    public void addItem(String item) { inventory.add(item); }

    // Deep clone — creates a truly independent copy
    @Override
    public GameCharacter clone() {
        try {
            GameCharacter cloned = (GameCharacter) super.clone(); // copies primitives + references
            cloned.inventory = new ArrayList<>(this.inventory);   // deep copy mutable field
            return cloned;
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException("Clone failed", e);
        }
    }

    @Override
    public String toString() {
        return String.format("GameCharacter[name=%s, weapon=%s, hp=%d, inventory=%s]",
            name, weaponType, health, inventory);
    }

    // Setters for post-clone customization
    public void setName(String name)             { this.name = name; }
    public void setHealth(int health)            { this.health = health; }
    public void setWeaponType(String weaponType) { this.weaponType = weaponType; }
}

public class PrototypeDemo {
    public static void main(String[] args) {
        // Create the expensive-to-build prototype
        GameCharacter baseWarrior = new GameCharacter("Warrior", "Sword", 100);
        baseWarrior.addItem("Shield");
        baseWarrior.addItem("Health Potion");

        // Clone and customize — much cheaper than building from scratch
        GameCharacter warrior1 = baseWarrior.clone();
        warrior1.setName("Arthur");

        GameCharacter warrior2 = baseWarrior.clone();
        warrior2.setName("Lancelot");
        warrior2.setWeaponType("Axe");
        warrior2.addItem("Magic Scroll"); // Only warrior2's inventory gets this

        System.out.println(baseWarrior);
        System.out.println(warrior1);
        System.out.println(warrior2);

        // Verify deep copy: warrior1's inventory is independent
        System.out.println("\nAre inventories the same object? "
            + (warrior1.inventory == warrior2.inventory)); // false — good!
    }
}
```

---

### ⚠️ Important Points & Best Practices

- **Shallow clone vs Deep clone:** `super.clone()` only copies the object's field values. For primitive fields that's fine, but for object references (like the `List`), you must manually deep copy them — otherwise both original and clone share the same list object.
- `Cloneable` is a marker interface but is widely considered poorly designed in Java. An alternative is a **copy constructor** or a static `copy()` factory method.
- Serialization-based deep copy is another technique: serialize then deserialize to get a completely independent copy.

---

## 6. Object Pool Pattern

### Theory

The Object Pool pattern **reuses a set of pre-created expensive objects** instead of creating and destroying them on demand. A pool manages a collection of reusable objects — when a client needs one, it borrows from the pool; when done, it returns it.

**When to use it:**
- Object creation is very expensive (DB connections, thread creation, large buffers)
- Objects are needed frequently but only for short durations
- System resources are limited

**Real-world examples:** HikariCP (database connection pools), Java thread pools (`ExecutorService`), ByteBuffer pools in Netty.

### Example — Simple Database Connection Pool

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

// Simulated expensive resource
class DatabaseConnection {
    private final int id;
    private boolean inUse = false;

    public DatabaseConnection(int id) {
        this.id = id;
        System.out.println("Creating expensive DB connection #" + id);
        // Imagine this takes 500ms — expensive!
    }

    public void executeQuery(String sql) {
        System.out.println("Connection #" + id + " executing: " + sql);
    }

    public void setInUse(boolean inUse) { this.inUse = inUse; }
    public boolean isInUse()           { return inUse; }
    public int getId()                 { return id; }
}

// The Pool — manages lifecycle of connections
class ConnectionPool {
    private final BlockingQueue<DatabaseConnection> pool;
    private final int poolSize;

    public ConnectionPool(int poolSize) {
        this.poolSize = poolSize;
        this.pool     = new ArrayBlockingQueue<>(poolSize);

        // Pre-create all connections at startup (eager initialization)
        for (int i = 1; i <= poolSize; i++) {
            pool.offer(new DatabaseConnection(i));
        }
        System.out.println("Pool initialized with " + poolSize + " connections\n");
    }

    // Borrow a connection — blocks if none available
    public DatabaseConnection acquire() throws InterruptedException {
        DatabaseConnection conn = pool.take(); // blocks until one is available
        conn.setInUse(true);
        System.out.println("Acquired connection #" + conn.getId());
        return conn;
    }

    // Return connection to pool for reuse
    public void release(DatabaseConnection conn) {
        conn.setInUse(false);
        pool.offer(conn);
        System.out.println("Released connection #" + conn.getId());
    }

    public int availableConnections() {
        return pool.size();
    }
}

public class ObjectPoolDemo {
    public static void main(String[] args) throws InterruptedException {
        ConnectionPool pool = new ConnectionPool(3);

        // Borrow connections
        DatabaseConnection c1 = pool.acquire();
        DatabaseConnection c2 = pool.acquire();

        c1.executeQuery("SELECT * FROM users");
        c2.executeQuery("UPDATE orders SET status='shipped'");

        System.out.println("Available: " + pool.availableConnections()); // 1

        // Return them — they go back to the pool, ready for reuse
        pool.release(c1);
        pool.release(c2);

        System.out.println("Available: " + pool.availableConnections()); // 3

        // Re-acquire — gets an existing connection, not a new one
        DatabaseConnection c3 = pool.acquire();
        c3.executeQuery("DELETE FROM logs WHERE age > 30");
        pool.release(c3);
    }
}
```

---

### ⚠️ Important Points & Best Practices

- **Always return objects to the pool** — a common mistake is acquiring without releasing, causing pool exhaustion. Use try-finally or try-with-resources.
- Set **maximum pool sizes** carefully — too large wastes resources; too small creates bottlenecks.
- Consider using a **timeout** when acquiring from the pool to avoid indefinitely blocking threads.
- In production, prefer battle-tested libraries: **HikariCP** for DB connections, Java's `ExecutorService` for threads.
- Object Pool is specifically useful for objects that are *expensive to create* but *cheap to reset/reuse*. Don't pool lightweight objects.

---

## Summary Table — Creational Patterns

| Pattern | Intent | Key Mechanism | Java Example |
|---------|--------|---------------|--------------|
| Singleton | One instance only | Private constructor + static field | Logger, Spring beans |
| Factory Method | Subclass decides what to create | Abstract method overridden | `Calendar.getInstance()` |
| Abstract Factory | Family of related objects | Interface of factory methods | JDBC Driver, UI toolkits |
| Builder | Step-by-step construction | Fluent API, inner Builder class | `StringBuilder`, Lombok `@Builder` |
| Prototype | Clone existing objects | `clone()` or copy constructor | Game object spawning |
| Object Pool | Reuse expensive objects | Pool queue with acquire/release | HikariCP, `ExecutorService` |

---

*Next: Phase 8.2 — Structural Patterns →*
