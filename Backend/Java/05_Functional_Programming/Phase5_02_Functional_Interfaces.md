# ⚡ Phase 5.2 — Functional Interfaces in Java
### `java.util.function` — The Building Blocks of Functional Java

---

## 🧠 What Is a Functional Interface?

A **functional interface** is any interface that has **exactly one abstract method**.
This single-method constraint is what allows a lambda expression to be used in its
place — the lambda *becomes* the implementation of that one method.

Java ships with a rich set of ready-made functional interfaces in the
`java.util.function` package so you rarely need to write your own. These fall into
four fundamental categories, each representing a different shape of computation:

| Category | Shape | Method | Description |
|----------|-------|--------|-------------|
| `Function<T, R>` | T → R | `apply(T)` | Transforms input to output |
| `Consumer<T>` | T → void | `accept(T)` | Consumes input, no output |
| `Supplier<T>` | () → T | `get()` | Produces a value, no input |
| `Predicate<T>` | T → boolean | `test(T)` | Tests input, returns true/false |

Everything else in `java.util.function` is a variation or specialization of one of
these four shapes.

---

## 1️⃣ `Function<T, R>` — Transform Things

`Function<T, R>` represents a transformation: it takes an input of type `T` and
produces a result of type `R`. Think of it as the mathematical concept of a function:
f(x) = y.

```java
import java.util.function.Function;

// Takes a String, returns its length as Integer
Function<String, Integer> lengthOf = s -> s.length();
System.out.println(lengthOf.apply("Hello")); // 5

// Takes an Integer, returns its String representation
Function<Integer, String> intToStr = n -> "Number: " + n;
System.out.println(intToStr.apply(42)); // Number: 42
```

### Composing Functions with `andThen()` and `compose()`

This is where functions become truly powerful — you can chain them together to build
pipelines. `andThen()` runs the current function first, then the next. `compose()`
runs the other function first, then the current one. They are inverses of each other.

```java
Function<String, String> trim      = String::trim;
Function<String, String> uppercase = String::toUpperCase;
Function<String, Integer> length   = String::length;

// andThen: left-to-right composition (trim, THEN uppercase)
Function<String, String> trimThenUpper = trim.andThen(uppercase);
System.out.println(trimThenUpper.apply("  hello  ")); // "HELLO"

// Chain of three: trim → uppercase → get length
Function<String, Integer> pipeline = trim.andThen(uppercase).andThen(length);
System.out.println(pipeline.apply("  hello  ")); // 5

// compose: right-to-left (uppercase THEN trim — in this case less useful, but illustrative)
Function<String, String> upperThenTrim = trim.compose(uppercase);
System.out.println(upperThenTrim.apply("  hello  ")); // "  HELLO  " then trimmed = "HELLO"
```

### `identity()` — The No-Op Function

`Function.identity()` returns a function that just returns its input unchanged.
It sounds useless, but it's very useful in stream operations when you need to pass
a `Function` but don't actually want to transform anything.

```java
Function<String, String> noOp = Function.identity();
System.out.println(noOp.apply("unchanged")); // unchanged

// Useful in toMap() when key and value are the same thing
List<String> words = List.of("apple", "banana", "cherry");
Map<String, Integer> wordLengths = words.stream()
    .collect(Collectors.toMap(Function.identity(), String::length));
// {apple=5, banana=6, cherry=6}
```

### `BiFunction<T, U, R>` — Two Inputs

When you need two parameters instead of one, use `BiFunction<T, U, R>`.

```java
BiFunction<String, Integer, String> repeat = (s, n) -> s.repeat(n);
System.out.println(repeat.apply("Ha", 3)); // HaHaHa

// BiFunction also has andThen() but NOT compose()
BiFunction<Integer, Integer, Integer> multiply = (a, b) -> a * b;
Function<Integer, String> toString = n -> "Result: " + n;
BiFunction<Integer, Integer, String> multiplyThenFormat = multiply.andThen(toString);
System.out.println(multiplyThenFormat.apply(6, 7)); // Result: 42
```

---

## 2️⃣ `Consumer<T>` — Do Something With Things

`Consumer<T>` represents an action — it takes an input of type `T` and does something
with it, but **returns nothing** (void). It's meant for side effects: printing, saving
to a database, sending an email, adding to a list, etc.

```java
import java.util.function.Consumer;

Consumer<String> print = s -> System.out.println(">> " + s);
print.accept("Hello"); // >> Hello

Consumer<Integer> saveToDb = id -> System.out.println("Saving record: " + id);
saveToDb.accept(101); // Saving record: 101
```

### Chaining Consumers with `andThen()`

You can chain consumers to perform multiple actions in sequence on the same input.

```java
Consumer<String> log   = s -> System.out.println("[LOG] " + s);
Consumer<String> audit = s -> System.out.println("[AUDIT] " + s);

Consumer<String> logAndAudit = log.andThen(audit);
logAndAudit.accept("User logged in");
// [LOG] User logged in
// [AUDIT] User logged in
```

### `BiConsumer<T, U>` — Two Inputs, No Output

```java
BiConsumer<String, Integer> printNTimes = (s, n) -> {
    for (int i = 0; i < n; i++) System.out.println(s);
};
printNTimes.accept("Hello", 3); // prints Hello three times

// Very useful with Map.forEach()
Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 87, "Charlie", 92);
scores.forEach((name, score) ->
    System.out.printf("%s scored %d%n", name, score));
```

---

## 3️⃣ `Supplier<T>` — Produce Things on Demand

`Supplier<T>` takes **no input** and **produces a value** of type `T`. Think of it
as a factory or a lazy generator — it produces something when asked. This is extremely
useful for **lazy initialization** (don't compute the value until it's actually needed)
and for providing default values.

```java
import java.util.function.Supplier;

// Supplies a new ArrayList on demand
Supplier<List<String>> listFactory = ArrayList::new;
List<String> list1 = listFactory.get();
List<String> list2 = listFactory.get(); // fresh list each time

// Lazy computation — only executed when get() is called
Supplier<Double> lazyPi = () -> {
    System.out.println("Computing pi...");
    return Math.PI;
};
// Nothing printed yet — the lambda hasn't run
Double pi = lazyPi.get(); // "Computing pi..." prints NOW

// Used in Optional.orElseGet() — only creates default if needed
Optional<String> maybeValue = Optional.empty();
String result = maybeValue.orElseGet(() -> "computed default");
```

### Why `orElseGet()` Over `orElse()`

This is one of the most important practical uses of `Supplier`. Consider the
difference:

```java
// orElse(T): The value is ALWAYS computed, whether needed or not
String value1 = optional.orElse(computeExpensiveDefault()); // always runs

// orElseGet(Supplier): Only computed if the Optional is empty
String value2 = optional.orElseGet(() -> computeExpensiveDefault()); // lazy

// If optional.isPresent(), orElse wastes computation. orElseGet is almost always better.
```

---

## 4️⃣ `Predicate<T>` — Test Things

`Predicate<T>` takes an input of type `T` and returns a **boolean**. It answers a
yes/no question about its input. Predicates are the backbone of filtering in streams.

```java
import java.util.function.Predicate;

Predicate<String> isEmpty   = String::isEmpty;
Predicate<String> isLong    = s -> s.length() > 10;
Predicate<Integer> isEven   = n -> n % 2 == 0;
Predicate<Integer> isPositive = n -> n > 0;

System.out.println(isEmpty.test(""));       // true
System.out.println(isLong.test("Hi"));      // false
System.out.println(isEven.test(4));         // true
```

### Composing Predicates — `and()`, `or()`, `negate()`

Predicates can be combined with logical operators to build complex conditions without
writing nested `if` statements.

```java
Predicate<Integer> isEven     = n -> n % 2 == 0;
Predicate<Integer> isPositive = n -> n > 0;
Predicate<Integer> isSmall    = n -> n < 100;

// and(): both must be true
Predicate<Integer> isEvenAndPositive = isEven.and(isPositive);
System.out.println(isEvenAndPositive.test(4));   // true
System.out.println(isEvenAndPositive.test(-4));  // false

// or(): at least one must be true
Predicate<Integer> isEvenOrSmall = isEven.or(isSmall);
System.out.println(isEvenOrSmall.test(3));   // true (3 < 100)
System.out.println(isEvenOrSmall.test(200)); // false

// negate(): invert the result
Predicate<Integer> isOdd = isEven.negate();
System.out.println(isOdd.test(3)); // true

// Chaining: even AND positive AND small
Predicate<Integer> combined = isEven.and(isPositive).and(isSmall);
System.out.println(combined.test(42));   // true
System.out.println(combined.test(102));  // false (not small)
```

### `Predicate.not()` — Static Utility (Java 11+)

Before Java 11, negating a predicate was awkward in method references. `Predicate.not()`
provides a clean solution.

```java
List<String> mixed = List.of("hello", "", "world", "", "java");

// Before Java 11 — awkward lambda
List<String> nonEmpty = mixed.stream()
    .filter(s -> !s.isEmpty())
    .collect(Collectors.toList());

// Java 11+ — clean method reference with Predicate.not()
List<String> nonEmptyClean = mixed.stream()
    .filter(Predicate.not(String::isEmpty))
    .collect(Collectors.toList());
// [hello, world, java]
```

### `BiPredicate<T, U>` — Two Inputs

```java
BiPredicate<String, String> startsWith = (s, prefix) -> s.startsWith(prefix);
System.out.println(startsWith.test("Hello, World", "Hello")); // true
```

---

## 5️⃣ Operators — Self-Contained Functions

Operators are specializations of `Function` where the input and output are the
**same type**. They exist for clarity — when you see `UnaryOperator<String>`, you
immediately know the output is also a `String`.

```java
import java.util.function.UnaryOperator;
import java.util.function.BinaryOperator;

// UnaryOperator<T> extends Function<T, T>
UnaryOperator<String> shout = s -> s.toUpperCase() + "!";
System.out.println(shout.apply("hello")); // HELLO!

// BinaryOperator<T> extends BiFunction<T, T, T>
BinaryOperator<Integer> add = (a, b) -> a + b;
System.out.println(add.apply(3, 4)); // 7

// BinaryOperator has identity() and maxBy()/minBy() static helpers
BinaryOperator<Integer> max = BinaryOperator.maxBy(Integer::compareTo);
System.out.println(max.apply(10, 20)); // 20
```

---

## 6️⃣ Primitive Specializations

Generic interfaces like `Function<Integer, Integer>` require **boxing and unboxing**
every primitive value, which creates overhead. For performance-critical code, Java
provides primitive-specialized versions that work directly with `int`, `long`, and
`double`.

The naming pattern is: `[Input type][Interface name]` or `[Interface name]To[Output type]`.

```java
import java.util.function.*;

// IntFunction<R>: int input, generic output
IntFunction<String> intToStr = n -> "Value: " + n;
System.out.println(intToStr.apply(42)); // Value: 42

// ToIntFunction<T>: generic input, int output
ToIntFunction<String> strLength = String::length;
System.out.println(strLength.applyAsInt("Hello")); // 5

// IntUnaryOperator: int → int (no boxing at all)
IntUnaryOperator doubleIt = n -> n * 2;
System.out.println(doubleIt.applyAsInt(21)); // 42

// IntBinaryOperator: (int, int) → int
IntBinaryOperator addInts = (a, b) -> a + b;
System.out.println(addInts.applyAsInt(10, 32)); // 42

// IntConsumer: consumes int, no output
IntConsumer printInt = n -> System.out.println("Int: " + n);
printInt.accept(7);

// IntSupplier: produces int, no input
IntSupplier randomInt = () -> (int)(Math.random() * 100);
System.out.println(randomInt.getAsInt()); // some int

// IntPredicate: tests int, returns boolean
IntPredicate isEven = n -> n % 2 == 0;
System.out.println(isEven.test(4)); // true
```

The same patterns exist for `Long` and `Double`. Use primitive specializations
wherever you're processing large collections of numbers — they avoid autoboxing
and are measurably faster in tight loops.

---

## 7️⃣ Writing Your Own Functional Interface

Sometimes none of the built-in interfaces fit your exact need — for example, when you
need a function that throws a checked exception, or has a semantically meaningful name
in your domain.

```java
// A named functional interface for domain clarity
@FunctionalInterface
public interface EmailSender {
    void send(String recipient, String subject, String body);
    // Only one abstract method — it's a functional interface
    // Can have default methods and static methods without breaking the rule:
    default void sendWelcome(String email) {
        send(email, "Welcome!", "Thanks for joining.");
    }
}

// A functional interface that throws a checked exception
@FunctionalInterface
public interface ThrowingFunction<T, R> {
    R apply(T input) throws Exception;
    // This lets you use lambdas that throw checked exceptions
}

// Usage
ThrowingFunction<String, Integer> parse = Integer::parseInt;
```

---

## 🔗 Building Functional Pipelines

The real power of functional interfaces emerges when you chain them together to build
readable data transformation pipelines.

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class FunctionalPipelineDemo {

    record Employee(String name, String department, double salary) {}

    public static void main(String[] args) {
        List<Employee> employees = List.of(
            new Employee("Alice",   "Engineering", 95000),
            new Employee("Bob",     "Marketing",   72000),
            new Employee("Charlie", "Engineering", 88000),
            new Employee("Diana",   "HR",          65000),
            new Employee("Eve",     "Engineering", 102000)
        );

        // Build reusable predicates
        Predicate<Employee> inEngineering = e -> e.department().equals("Engineering");
        Predicate<Employee> highEarner    = e -> e.salary() > 90000;

        // Build reusable functions
        Function<Employee, String> getName   = Employee::name;
        Function<Employee, Double> getSalary = Employee::salary;

        // Pipeline: engineers earning > 90k, sorted by salary, get their names
        List<String> topEngineers = employees.stream()
            .filter(inEngineering.and(highEarner))
            .sorted(Comparator.comparingDouble(Employee::salary).reversed())
            .map(getName)
            .collect(Collectors.toList());

        System.out.println(topEngineers); // [Eve, Alice]

        // Compute average salary using a Supplier for the stream source
        Supplier<Double> avgSalary = () -> employees.stream()
            .mapToDouble(Employee::salary)
            .average()
            .orElse(0.0);

        System.out.printf("Average salary: $%.0f%n", avgSalary.get()); // $84400
    }
}
```

---

## ⚠️ Best Practices & Pitfalls

**Always use `@FunctionalInterface` annotation** when creating your own functional
interfaces. It gives you compile-time protection and documents intent clearly.

**Prefer the standard interfaces** in `java.util.function` over custom ones whenever
the standard interface semantically fits. This keeps your API familiar to other Java
developers. Only create custom interfaces when the standard names would be confusing
in context, or when you need checked exception support.

**Prefer `orElseGet(Supplier)` over `orElse(value)`** for expensive default computations
with `Optional`. The supplier is only invoked when the value is absent, making
`orElseGet` always at least as efficient and sometimes much more so.

**Compose predicates instead of nesting lambdas.** Deeply nested lambda conditions
are hard to read. Building up predicates with `.and()`, `.or()`, `.negate()` and
giving them meaningful variable names reads like natural language.

**Name your lambdas/predicates well when reusing them.** When you assign a predicate
or function to a variable for reuse, give it a name that reads like a sentence:
`isEligibleForPromotion`, `formatAsCurrency`, `extractUserId`. This is one of the
biggest readability wins in functional Java.

**Avoid stateful lambdas.** A lambda that reads or writes mutable shared state
(especially in stream pipelines that might run in parallel) is a source of subtle
bugs. Design lambdas to be pure functions — same input always produces same output,
no side effects.

```java
// ❌ Stateful lambda — dangerous, especially with parallel streams
List<String> sideEffectList = new ArrayList<>();
list.stream()
    .filter(s -> { sideEffectList.add(s); return s.length() > 3; }) // mutating in filter!
    .collect(Collectors.toList());

// ✅ Pure pipeline — collect results cleanly
List<String> result = list.stream()
    .filter(s -> s.length() > 3)
    .collect(Collectors.toList());
```

---

## 🔑 Key Takeaways

- All four core shapes are: **transform** (`Function`), **consume** (`Consumer`), **produce** (`Supplier`), **test** (`Predicate`).
- All four have **composition methods** — use them to build pipelines without nesting.
- `BiFunction`, `BiConsumer`, `BiPredicate` handle **two inputs**.
- `UnaryOperator` and `BinaryOperator` are Functions where **input and output types match**.
- Use **primitive specializations** (`IntFunction`, `ToIntFunction`, etc.) for numeric data to avoid boxing overhead.
- `Predicate.not()` (Java 11+) allows clean method-reference negation.
- Design your lambdas to be **pure and stateless** — same input, same output, no side effects.
