# 4.5 Optional in Java (Java 8+)

## What & Why

`NullPointerException` has been called the "billion-dollar mistake" by its inventor, Tony Hoare, because null references are so easy to forget and so catastrophic when they slip past unchecked. Java's answer in Java 8 is `Optional<T>` — a container that may or may not hold a non-null value. Rather than returning `null` when something has no value, a method can return `Optional.empty()`, and the type signature itself advertises the possibility of absence.

The key shift `Optional` promotes is moving from **imperative null checks** (`if (value != null)`) to a **declarative, pipeline style** where you describe *what to do if a value is present* and *what to return if it isn't*, all in a readable chain.

`Optional` is not a silver bullet — used badly it creates new headaches. Used well it makes the presence/absence of a value explicit at the API boundary and dramatically reduces silent null propagation.

---

## 1. Creating Optional Instances

There are three factory methods, and picking the right one matters:

```java
import java.util.Optional;

// Optional.of() — wraps a value you are CERTAIN is non-null.
// If you pass null to of(), it throws NullPointerException immediately.
Optional<String> present = Optional.of("Hello, World");

// Optional.ofNullable() — wraps a value that might be null.
// If the value is null, it returns Optional.empty(); otherwise Optional.of(value).
String maybeNull = fetchFromDatabase(); // might return null
Optional<String> safe = Optional.ofNullable(maybeNull);

// Optional.empty() — explicitly represents "no value".
Optional<String> empty = Optional.empty();

// Rule of thumb:
// Use of()         when you know the value is non-null
// Use ofNullable() when you're wrapping legacy code that might return null
// Use empty()      as the explicit "nothing here" return value in your own methods
```

---

## 2. Checking and Extracting Values

```java
Optional<String> opt = Optional.of("Java");

// isPresent() and isEmpty() — simple boolean checks
System.out.println(opt.isPresent()); // true
System.out.println(opt.isEmpty());   // false (Java 11+)

// get() — extracts the value, but throws NoSuchElementException if empty
// Use sparingly — you usually shouldn't need to call get() explicitly
String value = opt.get(); // safe here because we just checked isPresent()

// ifPresent() — execute a lambda only if a value is present
opt.ifPresent(v -> System.out.println("Found: " + v)); // Found: Java

// ifPresentOrElse() — Java 9+ — handle both the present and empty cases
opt.ifPresentOrElse(
    v -> System.out.println("Found: " + v),
    ()  -> System.out.println("Nothing found")
);

Optional.<String>empty().ifPresentOrElse(
    v -> System.out.println("Found: " + v),
    ()  -> System.out.println("Nothing found") // this branch runs
);
```

---

## 3. Providing Fallback Values

These are among the most useful `Optional` methods — they let you specify what to return when the optional is empty, without writing an if/else:

```java
Optional<String> maybe = Optional.empty();

// orElse() — returns the given default value if empty
// WARNING: the default value is ALWAYS evaluated, even if the optional is present
String result1 = maybe.orElse("default");  // "default"

// orElseGet() — returns the result of a Supplier if empty
// PREFERRED: the Supplier is only called if the optional is empty (lazy)
String result2 = maybe.orElseGet(() -> computeExpensiveDefault()); // lazy evaluation

// orElseThrow() — throws an exception if empty (Java 10+)
// This is cleaner than calling get() and hoping for the best
String result3 = maybe.orElseThrow(() -> new IllegalStateException("Value is required"));

// or() — Java 9+ — returns another Optional if this one is empty
// Perfect for chaining fallback Optional sources
Optional<String> fallback = maybe.or(() -> Optional.of("fallback from another source"));
System.out.println(fallback.get()); // "fallback from another source"
```

**`orElse` vs `orElseGet` — this is a subtle but important performance distinction:**

```java
// orElse ALWAYS evaluates the argument — even if the Optional is present
Optional<User> userOpt = Optional.of(currentUser);
User user = userOpt.orElse(createNewUser()); // createNewUser() IS called even though
                                              // userOpt has a value — wasteful!

// orElseGet uses a Supplier — the lambda only runs if the Optional is empty
User user2 = userOpt.orElseGet(() -> createNewUser()); // createNewUser() NOT called
```

---

## 4. Transforming Values

This is where `Optional` really shines — you can map, filter, and chain transformations without ever writing an explicit null check:

```java
// map() — transform the value if present, leave empty if not
Optional<String> name = Optional.of("  alice  ");
Optional<String> upper = name
    .map(String::trim)        // "alice"
    .map(String::toUpperCase); // "ALICE"

System.out.println(upper.get()); // ALICE

Optional<String> empty = Optional.<String>empty()
    .map(String::toUpperCase); // stays empty — no exception thrown
System.out.println(empty.isPresent()); // false

// filter() — keep the value only if it passes a predicate
Optional<Integer> age = Optional.of(17);
Optional<Integer> adult = age.filter(a -> a >= 18); // empty because 17 < 18
System.out.println(adult.isPresent()); // false

Optional<Integer> legalAge = Optional.of(21).filter(a -> a >= 18);
System.out.println(legalAge.get()); // 21

// flatMap() — when your transformation function itself returns an Optional
// Without flatMap you'd get Optional<Optional<String>> — ugly
public Optional<String> findCity(String userId) {
    return Optional.ofNullable(users.get(userId))   // Optional<User>
        .flatMap(User::getAddress)                  // Optional<Address>  (not Optional<Optional<Address>>)
        .flatMap(Address::getCity);                 // Optional<String>
}

// Contrast: map() gives Optional<Optional<T>>, flatMap() flattens it
Optional<Optional<String>> nested  = Optional.of("hello").map(s -> Optional.of(s.toUpperCase()));
Optional<String>           flat    = Optional.of("hello").flatMap(s -> Optional.of(s.toUpperCase()));
```

---

## 5. Optional in Streams

`Optional` and `Stream` integrate neatly. The `stream()` method (Java 9+) turns an `Optional` into a zero-or-one-element stream, which is perfect for filtering or flat-mapping collections of Optional values:

```java
import java.util.List;
import java.util.stream.Collectors;

List<Optional<String>> optionals = List.of(
    Optional.of("apple"),
    Optional.empty(),
    Optional.of("banana"),
    Optional.empty(),
    Optional.of("cherry")
);

// Collect only the present values
List<String> fruits = optionals.stream()
    .flatMap(Optional::stream) // Optional.stream() is Java 9+
    .collect(Collectors.toList());

System.out.println(fruits); // [apple, banana, cherry]

// Before Java 9, you'd use filter(Optional::isPresent).map(Optional::get)
List<String> fruitsBefore9 = optionals.stream()
    .filter(Optional::isPresent)
    .map(Optional::get)
    .collect(Collectors.toList());
```

---

## 6. Common Anti-Patterns

### Anti-Pattern 1: Using Optional as a Field

```java
// BAD — Optional should not be a field. It adds boxing overhead and isn't Serializable.
public class Person {
    private Optional<String> nickname; // Don't do this

    public Optional<String> getNickname() { return nickname; }
}

// GOOD — use null internally, expose Optional in the getter
public class Person {
    private String nickname; // nullable field is fine internally

    public Optional<String> getNickname() {
        return Optional.ofNullable(nickname); // wrap at the API boundary
    }
}
```

### Anti-Pattern 2: isPresent() + get() — The Null-Check Reborn

```java
// BAD — you've just rewritten the null check with extra steps
if (opt.isPresent()) {
    String value = opt.get();
    doSomething(value);
}

// GOOD — use ifPresent, map, or orElse
opt.ifPresent(value -> doSomething(value));
// or
opt.map(this::processValue).orElse(defaultResult);
```

### Anti-Pattern 3: Optional for Primitive Types

Avoid `Optional<Integer>`, `Optional<Long>`, `Optional<Double>` — they box the primitive. Java provides specialised variants:

```java
import java.util.OptionalInt;
import java.util.OptionalLong;
import java.util.OptionalDouble;

OptionalInt   optInt  = OptionalInt.of(42);
OptionalLong  optLong = OptionalLong.empty();
OptionalDouble avg    = OptionalDouble.of(3.14);

System.out.println(optInt.getAsInt());   // 42
System.out.println(optLong.isPresent()); // false
```

### Anti-Pattern 4: Optional as a Method Parameter

```java
// BAD — forces callers to wrap values in Optional, which is cumbersome
public void process(Optional<String> name) { ... }
process(Optional.of("Alice")); // awkward call site

// GOOD — use method overloading or a nullable parameter + annotation
public void process(String name) { ... }   // name may be null
public void process() { process(null); }   // or overloaded for "absent" case
```

---

## 7. A Real-World Example

```java
// Domain model
public class Address {
    private String street;
    private String city;
    private String postalCode;

    public Optional<String> getCity()       { return Optional.ofNullable(city); }
    public Optional<String> getPostalCode() { return Optional.ofNullable(postalCode); }
}

public class Customer {
    private String name;
    private Address shippingAddress;

    public Optional<Address> getShippingAddress() {
        return Optional.ofNullable(shippingAddress);
    }
}

// Usage — deep navigation with no null checks, no NPE risk
public String getCustomerCity(Customer customer) {
    return Optional.of(customer)        // wrap the customer (guaranteed non-null here)
        .flatMap(Customer::getShippingAddress) // Optional<Address>
        .flatMap(Address::getCity)             // Optional<String>
        .orElse("Unknown City");               // fallback if any step is empty
}

// The imperative equivalent — same logic, far noisier
public String getCustomerCityOldWay(Customer customer) {
    if (customer != null) {
        Address address = customer.getShippingAddress().orElse(null);
        if (address != null) {
            String city = address.getCity().orElse(null);
            if (city != null) {
                return city;
            }
        }
    }
    return "Unknown City";
}
```

---

## ⚡ Best Practices & Important Points

**`Optional` is designed for return types.** Its primary purpose is to make a method's contract explicit: "this may or may not return a value." It is not designed for field storage, method parameters, or collection elements.

**The `flatMap` vs `map` rule is simple:** use `map` when your transformation returns a plain value (`T → U`). Use `flatMap` when your transformation returns another `Optional` (`T → Optional<U>`). Getting this wrong produces unwanted `Optional<Optional<T>>` nesting.

**`orElseGet` is almost always better than `orElse`** when the fallback involves any computation — method calls, object creation, or database queries. Since `orElse` eagerly evaluates its argument regardless of whether the Optional is empty, it can cause unnecessary work or even side effects.

**Return `Optional.empty()` rather than `null` from a method that returns `Optional`.** Returning `null` from an `Optional`-returning method completely defeats the purpose and gives callers a new NPE to deal with.

**`Optional` doesn't eliminate all null issues.** It helps at API boundaries between methods and callers. Inside a method body, you still need to handle null carefully. Think of Optional as a contract signal, not a null-safety guarantee.
