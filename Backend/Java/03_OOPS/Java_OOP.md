# ☕ Java Phase 3 — Object-Oriented Programming (OOP) — Complete Deep Dive
### Theory + Examples + `final`, `sealed`, and Design Guidance

---

## 📌 Why OOP Exists — The Big Picture

Before writing a single class, understand *why* OOP was created. Early programs were written procedurally — a giant list of instructions executed top to bottom. As programs grew, this became a nightmare to maintain: changing one part broke five others, data and logic were scattered everywhere, and reuse was nearly impossible.

**OOP solves this by organizing code around objects** — self-contained units that bundle data (fields) and behavior (methods) together. The real world is naturally object-oriented: a `BankAccount` *has* a balance and *can* deposit or withdraw. OOP lets us model that directly.

Java is a **class-based, single-inheritance OOP language** built around four pillars:
1. **Encapsulation** — hide your data, expose only what's needed
2. **Inheritance** — reuse and extend existing code
3. **Polymorphism** — one interface, many behaviors
4. **Abstraction** — show what something does, hide how it does it

---

## 3.1 Classes & Objects

### Theory

A **class** is a blueprint — it defines what data an object holds and what it can do. An **object** is a concrete instance of that blueprint, living in heap memory.

Think of a class as the cookie cutter, and objects as the cookies. The cutter defines the shape; each cookie is a real, tangible thing with its own existence.

```
CLASS (Blueprint)         OBJECT (Instance in memory)
─────────────────         ───────────────────────────
class Car {               Car myCar = new Car();
  String color;           → myCar.color = "Red"
  int speed;              → myCar.speed = 0
  void accelerate() {}    → myCar lives on the HEAP
}
```

### Memory Model — Heap vs Stack

Understanding where things live is critical for debugging and performance.

```
STACK (per thread)          HEAP (shared)
──────────────────          ─────────────
main() frame:               [Car object @ 0x1A2B]
  myCar ──────────────────▶   color: "Red"
  yourCar ────────────────▶ [Car object @ 0x3C4D]
                              color: "Blue"
```

- **Stack** stores local variables and method call frames. It's fast but limited in size.
- **Heap** stores all objects created with `new`. Managed by the Garbage Collector.
- A variable like `Car myCar` on the stack holds a *reference* (memory address) to the object on the heap, not the object itself. This is crucial — when you pass an object to a method, you're passing the reference, not a copy of the object.

### The `this` Keyword

`this` refers to the current object — the instance on which a method is being called. You need it when a parameter name shadows a field name, or when passing the current object to another method.

```java
public class Car {
    private String color;
    private int speed;

    // Without 'this', 'color = color' would assign the parameter to itself
    public Car(String color, int speed) {
        this.color = color;   // 'this.color' = field, 'color' = parameter
        this.speed = speed;
    }

    // Passing current object to another method
    public void register(Registry r) {
        r.add(this);  // pass the current Car instance
    }
}
```

### `null` and `NullPointerException`

`null` means a reference points to *nothing*. Calling a method on null causes a `NullPointerException` (NPE) — the most common Java runtime error.

```java
Car myCar = null;
myCar.accelerate();  // 💥 NullPointerException — no object to call on

// Always guard when null is possible
if (myCar != null) {
    myCar.accelerate();
}
// Or use Optional (covered in Phase 4)
```

Since Java 14, NPE messages are *helpful* — they tell you exactly which variable was null. E.g., `Cannot invoke "Car.accelerate()" because "myCar" is null`.

---

## 3.2 Constructors

### Theory

A constructor is a special method that runs when an object is created. Its job is to **initialize the object into a valid, usable state**. Constructors have no return type and must match the class name exactly.

If you write no constructor, Java silently adds a default no-arg constructor. The moment you write *any* constructor, that free default disappears.

### Types of Constructors

```java
public class BankAccount {
    private String owner;
    private double balance;

    // 1. No-arg constructor — creates account with defaults
    public BankAccount() {
        this.owner = "Unknown";
        this.balance = 0.0;
    }

    // 2. Parameterized constructor — creates account with given values
    public BankAccount(String owner, double balance) {
        if (balance < 0) throw new IllegalArgumentException("Balance cannot be negative");
        this.owner = owner;
        this.balance = balance;
    }

    // 3. Constructor chaining with this() — avoids duplicating validation logic
    // this() must be the FIRST statement in the constructor
    public BankAccount(String owner) {
        this(owner, 0.0);  // delegates to the 2-param constructor above
    }

    // 4. Copy constructor — creates a new account identical to another
    public BankAccount(BankAccount other) {
        this(other.owner, other.balance);
    }
}
```

### Constructor vs Method — Key Differences

| Aspect | Constructor | Method |
|--------|-------------|--------|
| Name | Must match class name | Any valid identifier |
| Return type | None (not even `void`) | Must have one |
| Called by | `new` keyword (automatically) | Explicitly by programmer |
| Purpose | Initialize object | Perform behavior |
| Inheritance | NOT inherited | Inherited |

### Why constructors matter for design

A well-designed constructor ensures an object is **never in an invalid state**. If you have mandatory fields, require them in the constructor rather than relying on setters — this is the principle of *defensive construction*.

```java
// Bad: Object can exist in invalid state
Order o = new Order();
o.setCustomer(customer);  // What if someone forgets this?
o.setItems(items);        // Object is "half-built" between these calls

// Good: Object is fully valid the moment it's created
Order o = new Order(customer, items);
```

---

## 3.3 Access Modifiers

### Theory

Access modifiers control **visibility** — who can see and use a class member. They are the mechanism behind encapsulation, enforcing boundaries in your code.

```
VISIBILITY TABLE
────────────────────────────────────────────────────────
Modifier       Same Class   Same Package   Subclass   Anywhere
─────────      ──────────   ────────────   ────────   ───────
private        ✅           ❌             ❌         ❌
(default)      ✅           ✅             ❌         ❌
protected      ✅           ✅             ✅         ❌
public         ✅           ✅             ✅         ✅
```

```java
package com.bank;

public class Account {
    private double balance;          // only Account can touch this
    double interestRate;             // package-private: any class in com.bank
    protected String accountType;   // subclasses (even in other packages) + package
    public String accountNumber;    // everyone
}
```

### Where to use each

**`private`** is your default for fields. Always ask: "does anyone outside this class *need* direct access to this data?" Almost always the answer is no.

**`protected`** is for members that subclasses need to access or override, but the outside world shouldn't touch directly. Use sparingly — it creates tight coupling between parent and child classes.

**Package-private (default)** is useful for utility classes and helpers that should only be used within a module/package. Good for internal implementation details.

**`public`** is for your intended API — the methods and constructors you want the world to use.

---

## 3.4 Encapsulation

### Theory

Encapsulation means **bundling data and the methods that operate on it together, while restricting direct access to the data**. It's often described as "data hiding," but it's really about *protecting invariants* — the rules that keep your object in a valid state.

### Why It Matters — A Concrete Example

```java
// BAD: No encapsulation — balance can be corrupted by anyone
public class BadAccount {
    public double balance;  // anyone can do: account.balance = -999999;
}

// GOOD: Encapsulated — balance only changes through controlled methods
public class GoodAccount {
    private double balance;   // hidden from outside

    public double getBalance() {
        return balance;       // read-only access
    }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        this.balance += amount;  // validation before mutation
    }

    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        if (amount > balance) throw new IllegalStateException("Insufficient funds");
        this.balance -= amount;
    }
}
```

Without encapsulation, `balance` could be set to any value by any code anywhere in the system. With encapsulation, the `BankAccount` class owns its state — it guarantees `balance` is always ≥ 0.

### Defensive Copying

When a field is a mutable object (like a `List` or `Date`), returning it directly leaks access to your internals.

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class Portfolio {
    private List<String> stocks = new ArrayList<>();

    // BAD: caller gets a reference to your internal list and can modify it
    public List<String> getStocksBad() {
        return stocks;  // caller can do getStocks().clear() — destroys your data!
    }

    // GOOD: defensive copy — caller gets their own copy
    public List<String> getStocks() {
        return Collections.unmodifiableList(stocks);  // or new ArrayList<>(stocks)
    }

    public void addStock(String symbol) {
        // validate here before adding
        if (symbol != null && !symbol.isBlank()) {
            stocks.add(symbol.toUpperCase());
        }
    }
}
```

### JavaBeans Convention

JavaBeans is a standard naming convention for encapsulated classes, used heavily by frameworks (Spring, Hibernate, Jackson):
- Fields are `private`
- Getters: `getFieldName()` or `isFieldName()` for booleans
- Setters: `setFieldName(Type value)`
- Class has a public no-arg constructor

---

## 3.5 Inheritance

### Theory

Inheritance lets one class **reuse the fields and methods of another class**, extending or specializing its behavior. The child class (`extends`) inherits everything non-private from the parent.

Java supports **single inheritance** (a class can extend only one class) but **multiple interface implementation** (a class can implement many interfaces). This avoids the "Diamond Problem" of multiple class inheritance.

```
Animal (parent/superclass)
  ├── Dog (child/subclass)
  └── Cat (child/subclass)
        └── Kitten (grandchild — multilevel inheritance)
```

### How `extends` and `super` Work

```java
// Parent class
public class Animal {
    protected String name;
    protected int age;

    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public void eat() {
        System.out.println(name + " is eating");
    }

    public String describe() {
        return name + ", age " + age;
    }
}

// Child class
public class Dog extends Animal {
    private String breed;

    // super() must be FIRST statement — calls parent constructor
    public Dog(String name, int age, String breed) {
        super(name, age);       // calls Animal(String, int)
        this.breed = breed;
    }

    // Overriding describe() to add breed info
    @Override
    public String describe() {
        return super.describe() + ", breed: " + breed;  // reuse parent's version
    }

    // New behavior only Dogs have
    public void fetch() {
        System.out.println(name + " fetches the ball!");
    }
}
```

```java
Dog dog = new Dog("Rex", 3, "Labrador");
dog.eat();          // inherited from Animal → "Rex is eating"
dog.fetch();        // Dog-specific → "Rex fetches the ball!"
dog.describe();     // overridden → "Rex, age 3, breed: Labrador"
```

### Method Overriding Rules

Overriding replaces the parent's version of a method with a more specific one:

1. The method signature (name + parameters) must be **identical**
2. The return type must be the same or a **covariant** (subtype) return — `Dog` can override a method returning `Animal` to return `Dog`
3. Access modifier must be **same or more permissive** — can't override `public` with `private`
4. Cannot throw **new or broader checked exceptions** than the parent
5. `@Override` annotation is optional but strongly recommended — it catches typos at compile time

```java
public class Shape {
    public Shape create() { return new Shape(); }
}

public class Circle extends Shape {
    @Override
    public Circle create() { return new Circle(); }  // ✅ covariant return type
}
```

### `final` with Inheritance

`final` has three uses related to inheritance:

```java
// 1. final CLASS — cannot be extended at all
// Why? Security (String, Integer), or when you know extension would break semantics
public final class String { ... }       // can't subclass String
public final class ImmutablePoint { ... }

// 2. final METHOD — cannot be overridden in subclasses
// Why? Critical behavior that must not be changed (e.g., security checks, invariants)
public class Template {
    public final void process() {       // algorithm skeleton that can't be changed
        validate();
        doProcess();    // hook for subclasses
        audit();
    }
    protected void doProcess() { }  // subclasses CAN override this part
}

// 3. final FIELD — value cannot be changed after initialization
public class Config {
    private final String host;          // must be set in constructor, then immutable
    private final int port;

    public Config(String host, int port) {
        this.host = host;
        this.port = port;
    }
}
```

### Why Java Doesn't Have Multiple Class Inheritance — The Diamond Problem

```
         Animal
        /      \
      Dog      Cat
        \      /
         ???
       DogCat???
```

If `Dog` and `Cat` both override `Animal.eat()` differently, which version does `DogCat` inherit? Java avoids this confusion entirely by allowing only single class inheritance. Interfaces (with default methods) have rules to resolve this.

---

## 3.6 Polymorphism

### Theory

Polymorphism means "many forms." It allows **one reference type to refer to objects of different types**, and the correct behavior is selected automatically. It's what lets you write general code that works on a family of types without knowing the specific type at compile time.

There are two kinds:

**Compile-time (Static) Polymorphism — Method Overloading:** the compiler picks the right method based on the argument types at compile time.

**Runtime (Dynamic) Polymorphism — Method Overriding:** the JVM picks the right method based on the *actual object type* at runtime, not the reference type.

### Dynamic Dispatch — The Core Mechanism

```java
Animal a1 = new Dog("Rex", 3, "Lab");   // Animal reference, Dog object
Animal a2 = new Cat("Whiskers", 2);     // Animal reference, Cat object

// At compile time: Java checks if Animal has eat()
// At RUNTIME: JVM looks at actual object type (Dog, Cat) and calls THEIR eat()
a1.eat();   // Dog's eat() is called
a2.eat();   // Cat's eat() is called
```

This is the magic that makes code like this possible:

```java
// One method handles ALL animals — present and future
public void feedAll(List<Animal> animals) {
    for (Animal animal : animals) {
        animal.eat();   // correct eat() called for each, regardless of type
    }
}

// Works with any Animal subclass ever created
feedAll(List.of(new Dog(...), new Cat(...), new Bird(...)));
```

### Upcasting and Downcasting

```java
// UPCASTING (implicit, always safe) — treating a Dog as an Animal
Animal animal = new Dog("Rex", 3, "Lab");   // automatic upcast
animal.eat();        // ✅ works — eat() is in Animal

// But you LOSE access to Dog-specific methods
animal.fetch();      // ❌ Compile error — Animal doesn't have fetch()

// DOWNCASTING (explicit, can fail) — going back to Dog
Dog dog = (Dog) animal;   // manual downcast
dog.fetch();         // ✅ now works

// Unsafe downcast without checking
Animal a = new Cat("Whiskers", 2);
Dog d = (Dog) a;     // 💥 ClassCastException at runtime!
```

### `instanceof` — Safe Downcasting

Always check with `instanceof` before downcasting to avoid `ClassCastException`:

```java
// Traditional way (Java 1+)
if (animal instanceof Dog) {
    Dog dog = (Dog) animal;
    dog.fetch();
}

// Pattern matching instanceof (Java 16+) — cleaner and safer
if (animal instanceof Dog dog) {  // checks AND casts in one step
    dog.fetch();                   // 'dog' is scoped to this block
}
```

### Method Overloading — Compile-time Polymorphism

The compiler selects the method at compile time based on argument count and types:

```java
public class Calculator {
    // Same name, different parameters — compiler picks the right one
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
    public int add(int a, int b, int c) { return a + b + c; }
    public String add(String a, String b) { return a + b; }
}

Calculator calc = new Calculator();
calc.add(1, 2);         // → int version
calc.add(1.5, 2.5);     // → double version
calc.add(1, 2, 3);      // → 3-param version
calc.add("Hello", "!"); // → String version
```

---

## 3.7 Abstraction

### Theory

Abstraction means **showing only what's relevant, hiding implementation details**. When you use `List.add()`, you don't care if it's an ArrayList or LinkedList internally — you just want to add an element. That's abstraction.

Java provides two tools for abstraction:
1. **Abstract classes** — partial abstraction (can have both abstract and concrete methods)
2. **Interfaces** — complete contract (defines what, never how — until Java 8 default methods)

---

## 3.7a Abstract Classes

### Theory

An `abstract` class is a class that **cannot be instantiated** — you can never write `new AbstractClass()`. It exists to be extended. It can contain:
- Abstract methods (declared, no body — subclasses MUST implement)
- Concrete methods (with body — subclasses inherit or override)
- Fields (both instance and static)
- Constructors (called via `super()` from subclasses)

```java
// Abstract base for all database repositories
public abstract class BaseRepository<T, ID> {
    protected final String tableName;

    // Constructor — subclasses call this via super()
    protected BaseRepository(String tableName) {
        this.tableName = tableName;
    }

    // ABSTRACT — every subclass MUST implement this their own way
    public abstract T findById(ID id);
    public abstract void save(T entity);
    public abstract void delete(ID id);

    // CONCRETE — shared logic all repositories use the same way
    public void logOperation(String operation) {
        System.out.println("[" + tableName + "] " + operation + " at " + System.currentTimeMillis());
    }

    // Template method pattern — defines the algorithm skeleton
    public final T findByIdWithLogging(ID id) {   // 'final' — can't be overridden
        logOperation("findById(" + id + ")");
        T result = findById(id);                   // calls subclass implementation
        logOperation("findById completed");
        return result;
    }
}

// Concrete subclass
public class UserRepository extends BaseRepository<User, Long> {
    public UserRepository() {
        super("users");     // calls BaseRepository constructor
    }

    @Override
    public User findById(Long id) {
        // actual SQL/JPA logic here
        return new User(id, "John");
    }

    @Override
    public void save(User user) { /* SQL insert/update */ }

    @Override
    public void delete(Long id) { /* SQL delete */ }
}
```

### When to Use Abstract Classes

Use an abstract class when:

1. **You want to share code (fields + methods) among closely related classes.** Abstract classes can have instance fields and constructors, which interfaces cannot.
2. **You want to define a template/skeleton algorithm** where subclasses fill in specific steps (Template Method Pattern).
3. **The "is-a" relationship is strong** and the classes are tightly related. `Dog is-a Animal`, `UserRepository is-a BaseRepository`.
4. **You need protected or package-private members** that are part of the contract.

Do NOT use abstract classes when:
- The classes are unrelated but share behavior (use interfaces)
- You might need multiple inheritance (use interfaces)
- You just want to prevent instantiation of a utility class (use `private` constructor instead)

---

## 3.8 Interfaces (Deep Dive)

### Theory

An interface is a **contract** — it declares what methods a class must provide, without saying how. Before Java 8, interfaces could only have abstract methods. Now they can also have `default` (concrete) and `static` methods.

Interfaces enable **multiple type inheritance** — a class can implement many interfaces, giving it multiple types simultaneously.

```java
// An interface defines a contract
public interface Flyable {
    // Implicitly: public abstract
    void fly();
    double getMaxAltitude();

    // Default method (Java 8+) — provides default behavior
    // Implementing classes inherit this unless they override it
    default String flyDescription() {
        return "Flying at up to " + getMaxAltitude() + " meters";
    }

    // Static method (Java 8+) — utility method on the interface
    static boolean canFlyHigherThan(Flyable f, double altitude) {
        return f.getMaxAltitude() > altitude;
    }
}

public interface Swimmable {
    void swim();

    default String swimDescription() {
        return "Swimming";
    }
}

// A class can implement MULTIPLE interfaces
public class Duck extends Bird implements Flyable, Swimmable {
    @Override
    public void fly() { System.out.println("Duck flying"); }

    @Override
    public double getMaxAltitude() { return 150.0; }

    @Override
    public void swim() { System.out.println("Duck swimming"); }
}
```

### Default Method Conflict Resolution

When two interfaces have a `default` method with the same name, the implementing class **must override it** to resolve the ambiguity:

```java
interface A {
    default void greet() { System.out.println("Hello from A"); }
}
interface B {
    default void greet() { System.out.println("Hello from B"); }
}

class C implements A, B {
    @Override
    public void greet() {
        A.super.greet();   // explicitly choose A's version
        // or write your own implementation entirely
    }
}
```

### Functional Interfaces (Java 8+)

A functional interface has **exactly one abstract method**. This is what enables lambda expressions — a lambda is essentially a concise way to implement a functional interface.

```java
@FunctionalInterface  // optional but documents intent; compiler enforces single abstract method
public interface Validator<T> {
    boolean validate(T value);  // the one abstract method

    // Can still have default and static methods
    default Validator<T> and(Validator<T> other) {
        return value -> this.validate(value) && other.validate(value);
    }
}

// Using with a lambda
Validator<String> notEmpty = s -> !s.isEmpty();
Validator<String> notTooLong = s -> s.length() <= 100;
Validator<String> combined = notEmpty.and(notTooLong);

System.out.println(combined.validate("Hello"));  // true
System.out.println(combined.validate(""));        // false
```

### Key Interfaces to Know

**`Comparable<T>`** — for natural ordering within a class. Used by `Arrays.sort()`, `Collections.sort()`, `TreeSet`, `TreeMap`.

```java
public class Student implements Comparable<Student> {
    private String name;
    private double gpa;

    @Override
    public int compareTo(Student other) {
        // negative: this < other, 0: equal, positive: this > other
        return Double.compare(other.gpa, this.gpa);  // sort by GPA descending
    }
}

List<Student> students = new ArrayList<>(List.of(new Student("Alice", 3.8), new Student("Bob", 3.5)));
Collections.sort(students);  // uses compareTo() automatically
```

**`Comparator<T>`** — for external/alternate ordering. Doesn't require modifying the class.

```java
// Sort students by name instead
Comparator<Student> byName = Comparator.comparing(s -> s.getName());

// Or using method reference
Comparator<Student> byName2 = Comparator.comparing(Student::getName);
students.sort(byName2);
```

**`Iterable<T>`** — allows use in for-each loops. Implement `iterator()` to return an `Iterator<T>`.

### Sealed Interfaces (Java 17+)

A `sealed` interface **restricts which classes may implement it**. This is powerful for modeling a closed set of types — like an algebraic data type.

```java
// Only these three can implement Shape — no others allowed
public sealed interface Shape permits Circle, Rectangle, Triangle {
    double area();
}

// Each permitted type must be: final, sealed, or non-sealed
public final class Circle implements Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }

    @Override
    public double area() { return Math.PI * radius * radius; }
}

public final class Rectangle implements Shape {
    private final double width, height;
    public Rectangle(double w, double h) { this.width = w; this.height = h; }

    @Override
    public double area() { return width * height; }
}

public final class Triangle implements Shape {
    private final double base, height;
    public Triangle(double b, double h) { this.base = b; this.height = h; }

    @Override
    public double area() { return 0.5 * base * height; }
}
```

Now you can use **exhaustive pattern matching** (Java 21) — the compiler *knows* those are the only possible shapes:

```java
// Compiler verifies all cases are covered — no default needed!
public double describeShape(Shape shape) {
    return switch (shape) {
        case Circle c    -> c.area();
        case Rectangle r -> r.area();
        case Triangle t  -> t.area();
        // No default required — compiler knows these are ALL the cases
    };
}
```

**Why sealed is powerful:** In a non-sealed hierarchy, anyone could add a `Pentagon` tomorrow and your switch would silently miss it. With sealed, the compiler enforces completeness.

### Interface vs Abstract Class — The Decision Guide

This is one of the most important design decisions in Java:

```
Use an INTERFACE when:                    Use an ABSTRACT CLASS when:
──────────────────────────────────────    ─────────────────────────────────────────
Unrelated classes share a contract        Closely related classes share code
(Flyable applies to Plane, Bird, Superman) (Animal → Dog, Cat, Bird)

Multiple inheritance is needed             You need shared fields/state
(Duck is both Flyable AND Swimmable)      (all repos share tableName field)

Defining a capability/role                 Defining a type hierarchy
("can fly" vs "is an animal")             ("is-a" relationship)

You want to keep things flexible           You want a template method skeleton
(swap implementations freely)             (define the algorithm, subclass fills steps)

API that many unrelated types              Partial implementation to reduce duplication
will implement                             among related types
```

**The "is-a" vs "can-do" test:**
- `Dog` *is-a* `Animal` → abstract class (inheritance)
- `Duck` *can* `fly` → interface (capability)
- `Order` *can be* `Comparable` → interface (role)

In modern Java, **prefer interfaces over abstract classes** whenever possible. They're more flexible (multiple implementation), and since Java 8, they can carry default method implementations too.

---

## 3.9 Static Members

### Theory

`static` means the member **belongs to the class itself, not to any instance**. Static members are shared across all instances — there is only one copy in memory, regardless of how many objects you create.

```java
public class Counter {
    // static field — ONE copy for the entire class
    private static int totalCount = 0;

    // instance field — each object has its OWN copy
    private int id;

    public Counter() {
        totalCount++;          // increment the shared counter
        this.id = totalCount;  // give this instance its unique id
    }

    // static method — no 'this', can only access static members
    public static int getTotalCount() {
        return totalCount;
    }

    // instance method — has 'this', can access both instance and static
    public int getId() {
        return id;
    }
}
```

```java
Counter c1 = new Counter();   // totalCount = 1, c1.id = 1
Counter c2 = new Counter();   // totalCount = 2, c2.id = 2
Counter c3 = new Counter();   // totalCount = 3, c3.id = 3

Counter.getTotalCount();      // 3 — accessed via class name, not instance
```

### Static Initializer Blocks

Runs once when the class is first loaded, before any constructor:

```java
public class DatabaseConfig {
    private static final String DB_URL;
    private static final int MAX_CONNECTIONS;

    // Static initializer — runs once when class loads
    static {
        String env = System.getenv("ENV");
        if ("production".equals(env)) {
            DB_URL = "jdbc:postgresql://prod-server/mydb";
            MAX_CONNECTIONS = 100;
        } else {
            DB_URL = "jdbc:h2:mem:testdb";
            MAX_CONNECTIONS = 10;
        }
    }
}
```

### Singleton Pattern Using Static

The Singleton ensures only one instance ever exists. The best modern approach uses an `enum`:

```java
// Thread-safe Singleton — recommended way
public enum AppConfig {
    INSTANCE;

    private final String configFile = "/app/config.yml";
    private Map<String, String> settings = new HashMap<>();

    public String getSetting(String key) {
        return settings.get(key);
    }
}

// Usage
AppConfig config = AppConfig.INSTANCE;
```

Or the Double-Checked Locking (DCL) pattern for class-based singletons:

```java
public class ConnectionPool {
    // volatile ensures visibility across threads
    private static volatile ConnectionPool instance;

    private ConnectionPool() { }

    public static ConnectionPool getInstance() {
        if (instance == null) {                    // first check (no lock)
            synchronized (ConnectionPool.class) {
                if (instance == null) {            // second check (with lock)
                    instance = new ConnectionPool();
                }
            }
        }
        return instance;
    }
}
```

---

## 3.10 Inner Classes

### Theory

Java allows classes nested inside other classes. There are four types, each with different purposes and access rules.

```java
public class Outer {
    private int outerField = 10;

    // ─────────────────────────────────────────────
    // 1. MEMBER INNER CLASS (non-static)
    // Has access to outer class's private members
    // Requires an instance of Outer to instantiate
    // ─────────────────────────────────────────────
    public class Inner {
        public void display() {
            System.out.println("Outer field: " + outerField);  // ✅ can access private!
        }
    }

    // ─────────────────────────────────────────────
    // 2. STATIC NESTED CLASS
    // Does NOT have access to outer instance members
    // Can be instantiated without an Outer instance
    // Use this when the nested class doesn't need outer state
    // ─────────────────────────────────────────────
    public static class StaticNested {
        public void display() {
            // System.out.println(outerField);  // ❌ Cannot access — no outer instance
            System.out.println("I'm a static nested class");
        }
    }

    public void methodWithLocalAndAnonymous() {
        int localVar = 5;  // must be effectively final to use in inner classes

        // ─────────────────────────────────────────────
        // 3. LOCAL INNER CLASS
        // Defined inside a method, visible only within it
        // Can access effectively final local variables
        // ─────────────────────────────────────────────
        class LocalHelper {
            void help() {
                System.out.println("Local var: " + localVar);  // ✅ effectively final
            }
        }
        new LocalHelper().help();

        // ─────────────────────────────────────────────
        // 4. ANONYMOUS INNER CLASS
        // One-time use, no name, must extend a class or implement an interface
        // Largely replaced by lambdas for functional interfaces
        // Still useful for non-functional interfaces
        // ─────────────────────────────────────────────
        Runnable r = new Runnable() {
            @Override
            public void run() {
                System.out.println("Anonymous Runnable, localVar=" + localVar);
            }
        };
        r.run();
    }
}
```

```java
// Instantiation patterns
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();          // non-static inner: needs outer instance
Outer.StaticNested nested = new Outer.StaticNested();  // static: no outer needed
```

### When to Use Each

**Member inner class:** When the inner class is logically part of the outer class and needs access to its state. E.g., a `Node` class inside a `LinkedList`. However, be careful — it creates a tight coupling and holds a reference to the outer instance (can cause memory leaks).

**Static nested class:** The preferred choice when nesting is for logical grouping/namespace reasons but the inner class doesn't need outer instance state. Builder pattern implementations often use this. `Map.Entry` is a famous example.

**Local inner class:** Rarely used in modern Java. Useful when a class is needed only within a single method.

**Anonymous inner class:** Still useful for implementing non-functional interfaces with multiple methods (e.g., `Comparator` before Java 8, event listeners). For single-method interfaces (functional interfaces), prefer lambdas.

---

## 3.11 Enumerations (Enums)

### Theory

An `enum` defines a **fixed set of named constants**. But in Java, enums are full-blown classes — they can have fields, methods, constructors, and implement interfaces.

```java
public enum DayOfWeek {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY;

    // Built-in methods you get for free
    // name()    → "MONDAY"
    // ordinal() → 0 (zero-based position)
    // values()  → DayOfWeek[] of all values
    // valueOf("MONDAY") → DayOfWeek.MONDAY
}
```

### Enums with Fields and Methods

```java
public enum Planet {
    // Each constant IS an instance of Planet, constructed with these values
    MERCURY(3.303e+23, 2.4397e6),
    VENUS  (4.869e+24, 6.0518e6),
    EARTH  (5.976e+24, 6.37814e6),
    MARS   (6.421e+23, 3.3972e6);

    private final double mass;    // kilograms
    private final double radius;  // meters
    private static final double G = 6.67300E-11;

    // Enum constructors are implicitly private
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    public double surfaceGravity() {
        return G * mass / (radius * radius);
    }

    public double surfaceWeight(double otherMass) {
        return otherMass * surfaceGravity();
    }
}
```

```java
double earthWeight = 75.0;  // kg on Earth
for (Planet p : Planet.values()) {
    System.out.printf("Weight on %s: %.2f%n", p, p.surfaceWeight(earthWeight));
}
// Weight on MERCURY: 28.33
// Weight on EARTH:   75.00
// ...
```

### Enum with Abstract Methods

Each constant can provide its own implementation:

```java
public enum Operation {
    ADD("+") {
        @Override
        public double apply(double x, double y) { return x + y; }
    },
    SUBTRACT("-") {
        @Override
        public double apply(double x, double y) { return x - y; }
    },
    MULTIPLY("*") {
        @Override
        public double apply(double x, double y) { return x * y; }
    };

    private final String symbol;

    Operation(String symbol) { this.symbol = symbol; }

    public abstract double apply(double x, double y);

    @Override
    public String toString() { return symbol; }
}

// Usage
System.out.println(Operation.ADD.apply(3, 4));       // 7.0
System.out.println(Operation.MULTIPLY.apply(3, 4));  // 12.0
```

### EnumSet and EnumMap

These are highly optimized collections for enums — use them over `HashSet<Enum>` or `HashMap<Enum, V>`:

```java
import java.util.EnumSet;
import java.util.EnumMap;

// EnumSet: internally uses a bit vector — extremely fast
EnumSet<DayOfWeek> weekdays = EnumSet.range(DayOfWeek.MONDAY, DayOfWeek.FRIDAY);
EnumSet<DayOfWeek> weekend  = EnumSet.complementOf(weekdays);

// EnumMap: faster than HashMap for enum keys
EnumMap<DayOfWeek, String> schedule = new EnumMap<>(DayOfWeek.class);
schedule.put(DayOfWeek.MONDAY, "Sprint Planning");
schedule.put(DayOfWeek.FRIDAY, "Retrospective");
```

---

## 3.12 Records (Java 16+)

### Theory

A `record` is a **concise, immutable data carrier class**. It eliminates the boilerplate of a POJO (constructor, getters, `equals`, `hashCode`, `toString`) — the compiler generates all of it automatically.

```java
// Record declaration — generates everything automatically
public record Point(double x, double y) { }

// Equivalent traditional class would be ~50 lines of boilerplate
// Records give you: constructor, x(), y(), equals(), hashCode(), toString()
```

### Using Records

```java
Point p1 = new Point(1.0, 2.0);
Point p2 = new Point(1.0, 2.0);

p1.x();           // getter (not getX()!) → 1.0
p1.y();           // → 2.0
p1.toString();    // → "Point[x=1.0, y=2.0]"
p1.equals(p2);    // → true (value-based equality, not reference)
p1.hashCode();    // consistent with equals()
```

### Custom Compact Constructors

You can add validation without rewriting the full constructor:

```java
public record Range(int min, int max) {
    // Compact constructor — no params, no assignment (implicit)
    // 'this.min = min' happens automatically AFTER this block
    public Range {
        if (min > max) {
            throw new IllegalArgumentException("min (" + min + ") > max (" + max + ")");
        }
        // You can also normalize values:
        min = Math.max(0, min);  // ensure min is not negative
    }
}
```

### Records with Custom Methods

Records can have custom methods — they just can't add instance fields:

```java
public record Money(double amount, String currency) {
    // Compact constructor for validation
    public Money {
        if (amount < 0) throw new IllegalArgumentException("Negative money");
        currency = currency.toUpperCase();  // normalize
    }

    // Custom methods are fine
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount + other.amount, this.currency);
    }

    public boolean isZero() {
        return amount == 0.0;
    }
}
```

### Records vs POJOs vs Lombok

```
RECORDS           LOMBOK @Data              POJO (manual)
────────────      ──────────────────────    ─────────────────────────
Immutable         Mutable (by default)      Your choice
Java built-in     Requires dependency       No dependency
Concise syntax    Annotation-based          Verbose
No extra fields   Can add fields            Can add fields
Great for DTOs    Great for entities        Full flexibility
Value semantics   Can have value semantics  You implement equals/hashCode
```

**Use records for:** DTOs (Data Transfer Objects), value objects, query results, configuration data, anything where immutability is desired.

**Use Lombok @Data for:** JPA entities (which need to be mutable), complex domain objects.

---

## 3.13 The Object Class — Root of All Classes

### Theory

Every class in Java implicitly extends `Object` — it's the root of the entire class hierarchy. `Object` provides methods that every object inherits, and some of them you *must* override correctly for your classes to behave properly in collections and comparisons.

### `toString()`

```java
// Default: "ClassName@hashCodeInHex" — useless for debugging
Object o = new Object();
System.out.println(o);   // Object@1b6d3586

// Override to give meaningful output
public class User {
    private String name;
    private int age;

    @Override
    public String toString() {
        return "User{name='" + name + "', age=" + age + "}";
    }
}
System.out.println(new User("Alice", 30));  // User{name='Alice', age=30}
```

### `equals()` and `hashCode()` — The Contract

This is the most critical pair of methods to understand. They have a **binding contract:**
1. If `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` **must** also be `true`
2. The reverse is NOT required — same hashCode doesn't mean equal (hash collision)

**Always override them together.** If you override one without the other, your objects will behave incorrectly in `HashMap`, `HashSet`, and similar collections.

```java
public class Product {
    private String sku;
    private String name;

    // Default equals() checks REFERENCE equality (==)
    // We want VALUE equality — two Products are equal if their SKUs match

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                      // same reference? shortcut
        if (o == null || getClass() != o.getClass()) return false;  // null/type check
        Product product = (Product) o;                   // safe cast
        return Objects.equals(sku, product.sku);         // compare by SKU
    }

    @Override
    public int hashCode() {
        return Objects.hash(sku);  // MUST use same fields as equals()!
    }
}
```

```java
Product p1 = new Product("SKU-001", "Widget");
Product p2 = new Product("SKU-001", "Widget");

System.out.println(p1 == p2);         // false — different references
System.out.println(p1.equals(p2));    // true — same SKU

// Without correct equals/hashCode, Set and Map break:
Set<Product> set = new HashSet<>();
set.add(p1);
set.contains(p2);  // true only if hashCode and equals are correct
```

**The equals() contract (must be:**
- *Reflexive:* `x.equals(x)` is true
- *Symmetric:* if `x.equals(y)` then `y.equals(x)`
- *Transitive:* if `x.equals(y)` and `y.equals(z)` then `x.equals(z)`
- *Consistent:* repeated calls return same result
- `x.equals(null)` is always false

### `clone()` — Shallow vs Deep Copy

```java
// Shallow copy: new object, but references inside still point to same objects
// Deep copy: new object AND new copies of all referenced objects

public class Team implements Cloneable {
    private String name;
    private List<String> members;  // mutable — shallow copy would share this!

    @Override
    public Team clone() {
        try {
            Team copy = (Team) super.clone();    // shallow clone
            copy.members = new ArrayList<>(this.members);  // deep copy the list
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError("Never happens", e);
        }
    }
}
```

In practice, `clone()` is considered poorly designed — prefer copy constructors, static factory methods, or serialization-based deep copy for complex objects.

---

## 3.14 SOLID Principles

The SOLID principles are the five most important guidelines for OOP design. They help you write code that is easy to understand, maintain, and extend over time.

---

### S — Single Responsibility Principle (SRP)

**"A class should have only one reason to change."**

A class should do one thing and do it well. If a class is responsible for multiple concerns, changes in one concern ripple and affect the other.

```java
// BAD: UserService handles user logic AND email AND database
public class UserService {
    public void createUser(User user) {
        // validate user
        // save to database
        // send welcome email
        // log the event
    }
}

// GOOD: Each class has one responsibility
public class UserValidator {
    public void validate(User user) { /* validation logic */ }
}

public class UserRepository {
    public void save(User user) { /* database logic */ }
}

public class EmailService {
    public void sendWelcomeEmail(User user) { /* email logic */ }
}

public class UserService {
    private final UserValidator validator;
    private final UserRepository repository;
    private final EmailService emailService;

    public UserService(UserValidator v, UserRepository r, EmailService e) {
        this.validator = v;
        this.repository = r;
        this.emailService = e;
    }

    public void createUser(User user) {
        validator.validate(user);
        repository.save(user);
        emailService.sendWelcomeEmail(user);
    }
}
```

Now `UserValidator` only changes if validation rules change. `EmailService` only changes if email behavior changes. They don't interfere.

---

### O — Open/Closed Principle (OCP)

**"Classes should be open for extension, closed for modification."**

You should be able to add new behavior without changing existing, tested code. Achieve this through polymorphism and abstraction.

```java
// BAD: Adding a new discount type requires modifying this method
public class DiscountCalculator {
    public double calculate(Order order, String discountType) {
        if ("PERCENTAGE".equals(discountType)) {
            return order.getTotal() * 0.1;
        } else if ("FIXED".equals(discountType)) {
            return 5.0;
        } else if ("SEASONAL".equals(discountType)) {  // have to add this IF every time
            return order.getTotal() * 0.2;
        }
        return 0;
    }
}

// GOOD: New discount types extend, never modify
public interface DiscountStrategy {
    double calculate(Order order);
}

public class PercentageDiscount implements DiscountStrategy {
    private final double rate;
    public PercentageDiscount(double rate) { this.rate = rate; }
    @Override public double calculate(Order order) { return order.getTotal() * rate; }
}

public class FixedDiscount implements DiscountStrategy {
    private final double amount;
    public FixedDiscount(double amount) { this.amount = amount; }
    @Override public double calculate(Order order) { return amount; }
}

// Adding seasonal discount: create a NEW class, touch NOTHING existing
public class SeasonalDiscount implements DiscountStrategy {
    @Override public double calculate(Order order) { return order.getTotal() * 0.2; }
}

public class DiscountCalculator {
    public double calculate(Order order, DiscountStrategy strategy) {
        return strategy.calculate(order);  // never needs to change
    }
}
```

---

### L — Liskov Substitution Principle (LSP)

**"Objects of a subclass should be substitutable for objects of the superclass without breaking the program."**

If `S` extends `T`, then anywhere you use `T`, you should be able to use `S` without any surprises. Subclasses must honor the parent's contract — they should not weaken preconditions or strengthen postconditions.

```java
// BAD: Famous violation — Square "is-a" Rectangle mathematically, but not in code
public class Rectangle {
    protected int width;
    protected int height;
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int w) {
        this.width = w;
        this.height = w;  // maintains square constraint
    }
    @Override
    public void setHeight(int h) {
        this.width = h;
        this.height = h;
    }
}

// This code ASSUMES it's working with Rectangle, but Square breaks it
public void resize(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    // Expected: 5 * 10 = 50
    // With Square: 10 * 10 = 100 — SURPRISE! LSP violated
    System.out.println(r.area());
}

// GOOD: Model them as separate concepts — both implement a Shape interface
public interface Shape {
    int area();
}
public final class Rectangle implements Shape { ... }  // independent
public final class Square implements Shape { ... }      // independent
```

---

### I — Interface Segregation Principle (ISP)

**"Clients should not be forced to depend on interfaces they don't use."**

Don't create fat interfaces with too many methods. Break them into smaller, focused ones.

```java
// BAD: One fat interface forces implementors to implement methods they don't need
public interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
    void writeReport();
}

// A Robot worker doesn't eat or sleep, but is forced to implement them
public class Robot implements Worker {
    @Override public void work() { /* actual work */ }
    @Override public void eat() { /* meaningless empty implementation */ }
    @Override public void sleep() { /* meaningless empty implementation */ }
    @Override public void attendMeeting() { /* maybe? */ }
    @Override public void writeReport() { /* maybe? */ }
}

// GOOD: Segregated interfaces
public interface Workable { void work(); }
public interface Feedable { void eat(); }
public interface Sleepable { void sleep(); }
public interface MeetingAttendee { void attendMeeting(); }

public class Human implements Workable, Feedable, Sleepable, MeetingAttendee {
    // implements all — makes sense for humans
}

public class Robot implements Workable {
    // only implements what makes sense for robots
}
```

---

### D — Dependency Inversion Principle (DIP)

**"High-level modules should not depend on low-level modules. Both should depend on abstractions."**

Your business logic shouldn't be tightly coupled to specific implementations (like MySQL or a file system). It should depend on interfaces.

```java
// BAD: OrderService directly depends on MySQLOrderRepo — hard to change, hard to test
public class OrderService {
    private MySQLOrderRepository repo = new MySQLOrderRepository();  // concrete dep!

    public void placeOrder(Order order) {
        repo.save(order);  // tightly coupled to MySQL
    }
}

// GOOD: Depend on an abstraction
public interface OrderRepository {
    void save(Order order);
    Order findById(Long id);
}

public class MySQLOrderRepository implements OrderRepository { ... }
public class InMemoryOrderRepository implements OrderRepository { ... }  // for tests

public class OrderService {
    private final OrderRepository repo;  // depends on INTERFACE, not implementation

    // Dependency is INJECTED — not created internally (enables testing & swapping)
    public OrderService(OrderRepository repo) {
        this.repo = repo;
    }

    public void placeOrder(Order order) {
        repo.save(order);  // works with ANY implementation
    }
}

// In production:
OrderService service = new OrderService(new MySQLOrderRepository());

// In tests:
OrderService service = new OrderService(new InMemoryOrderRepository());
```

This is the foundation of **Dependency Injection (DI)**, which Spring Boot automates for you.

---

## 🗺️ OOP Decision Map — Quick Reference

```
WHAT DO YOU NEED?                       USE
─────────────────────────────────────   ──────────────────────────────
Model a "type" hierarchy (is-a)         Abstract class or class
Share code among related classes        Abstract class
Define a contract multiple types obey  Interface
Multiple type roles for one class       Multiple interfaces
Fixed set of constants (with behavior)  Enum
Immutable data holder (no behavior)     Record
Restrict who can extend/implement       sealed class / sealed interface
One-time implementation of an interface Anonymous inner class (or lambda)
Shared state across all instances       static field
Behavior on the class, not instances    static method
Group related type without outer state  Static nested class
Internal type needing outer state       Member inner class
```

---

## 📋 Summary of Key Rules

**Always remember:**
- Override `equals()` and `hashCode()` together or not at all.
- Use `@Override` always — it catches mistakes at compile time.
- Start with `private` for fields, loosen access only when required.
- Prefer composition over inheritance when the "is-a" relationship is unclear.
- Use interfaces for types (what something can do), abstract classes for implementation sharing (what something is).
- `final` prevents extension (`final class`), overriding (`final method`), or reassignment (`final field`).
- `sealed` gives you a controlled, exhaustive type hierarchy — great for domain modeling.
- Records are Java's concise answer to immutable data types — use them for DTOs and value objects.
- SOLID principles are not rules to follow blindly — they are guidelines that improve flexibility and testability.

---

*Next: Phase 4 — Core APIs (Collections Framework, Generics, Exception Handling, and more)*
