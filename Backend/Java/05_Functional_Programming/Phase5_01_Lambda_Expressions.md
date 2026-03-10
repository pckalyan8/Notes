# ⚡ Phase 5.1 — Lambda Expressions in Java
### From Anonymous Classes to Concise Behavior

---

## 🧠 What Problem Do Lambdas Solve?

Before Java 8, passing **behavior** to a method required creating a full class or a
verbose anonymous inner class. This was functional but noisy — you needed six lines of
boilerplate to express a single line of intent. Lambdas eliminate that noise.

Think of a lambda as a **concise way to write a method without a name** — an anonymous
function that can be passed around like a value. The key mental model is:

> "Instead of passing an object that has a method, I can pass the method itself."

---

## 📌 Prerequisite: Functional Interface

A lambda expression can only be used where a **functional interface** is expected.
A functional interface is any interface with exactly **one abstract method**.

```java
// This is a functional interface — it has exactly one abstract method
@FunctionalInterface
public interface Greeter {
    String greet(String name);  // the one abstract method
}
```

The `@FunctionalInterface` annotation is optional but recommended — it causes the
compiler to enforce the single-abstract-method rule, protecting you from accidental
interface changes.

---

## ✍️ Lambda Syntax

The general form of a lambda is:

```
(parameters) -> body
```

Where the body can be either a single expression or a block with `{}`.

### All Syntax Variants

```java
// 1. No parameters
Runnable r = () -> System.out.println("Running!");

// 2. One parameter — parentheses are optional for single param
Consumer<String> printer = name -> System.out.println("Hello, " + name);

// 3. Multiple parameters — parentheses required
Comparator<Integer> comp = (a, b) -> a - b;

// 4. Explicit types — you CAN declare types, but it's usually unnecessary
Comparator<Integer> compTyped = (Integer a, Integer b) -> a - b;

// 5. Block body — use {} and return when logic is multi-line
Function<String, Integer> parseOrDefault = (s) -> {
    try {
        return Integer.parseInt(s);
    } catch (NumberFormatException e) {
        return 0;  // default value if parsing fails
    }
};

// 6. Single expression (no return keyword needed)
Function<Integer, Integer> square = x -> x * x;
```

The compiler infers parameter types from the target functional interface, which is why
you rarely need to write them explicitly.

---

## 🔍 How Type Inference Works

When you write a lambda, Java looks at the **target type** — the functional interface
it's being assigned to or passed into — and infers the parameter types from that
interface's method signature.

```java
// The compiler sees: BiFunction<String, Integer, String>
// So it knows: (s, n) means (String s, Integer n)
BiFunction<String, Integer, String> repeater = (s, n) -> s.repeat(n);

// In a method call, target type comes from the parameter signature
List<String> words = List.of("banana", "apple", "cherry");
words.sort((a, b) -> a.compareTo(b));
// Compiler infers a and b are String because List<String>.sort takes Comparator<String>
```

---

## 🔒 Effectively Final Variables

Lambdas can capture variables from their surrounding scope — but with a restriction:
the captured variable must be **effectively final** (either declared `final` or never
reassigned after initialization). This rule exists because lambdas may execute later
or in a different thread, and mutable captured variables would cause unpredictable behavior.

```java
String prefix = "Hello"; // effectively final — never reassigned

Consumer<String> greeter = name -> System.out.println(prefix + ", " + name);
greeter.accept("Alice"); // Prints: Hello, Alice

// This would NOT compile:
String mutablePrefix = "Hi";
mutablePrefix = "Hey";                          // reassignment breaks "effectively final"
Consumer<String> broken = name -> System.out.println(mutablePrefix + name); // ❌ compile error
```

### ⚠️ Important: Mutating Objects IS Allowed

The "effectively final" rule applies to the **variable reference**, not to the
object's internal state. You can mutate the object the variable points to — though
this is generally a bad practice in functional code.

```java
List<String> results = new ArrayList<>(); // reference is effectively final

Consumer<String> collector = item -> results.add(item); // ✅ mutating the object is allowed
collector.accept("one");
collector.accept("two");
// But this is generally a code smell in stream pipelines — prefer collect()
```

---

## 📎 Method References

Method references are a shorthand for a lambda that does nothing but call an existing
method. They make code even more readable when the lambda body is just a single
method call.

The syntax is: `ClassName::methodName` or `object::methodName`

There are four kinds of method references:

### 1. Static Method Reference

When the lambda just calls a static method.

```java
// Lambda form
Function<String, Integer> parser = s -> Integer.parseInt(s);

// Method reference form — cleaner
Function<String, Integer> parserRef = Integer::parseInt;

// Another example
List<Double> numbers = List.of(1.5, -2.3, 3.7);
numbers.stream()
       .map(Math::abs)   // equivalent to: d -> Math.abs(d)
       .forEach(System.out::println);
```

### 2. Instance Method Reference on a Specific Object

When the lambda calls a method on a *particular, pre-existing* object.

```java
String prefix = "Hello, ";

// Lambda form
Predicate<String> startsWithHello = s -> prefix.startsWith(s);

// Method reference on specific instance 'prefix'
Predicate<String> startsRef = prefix::startsWith;

// Very common with System.out
Consumer<String> print = System.out::println;  // specific instance: System.out
```

### 3. Instance Method Reference on Any Object of That Type

When the lambda takes an object as its first parameter and calls a method on it.
The object is not fixed — it comes from the stream or the input.

```java
// Lambda form: takes a String, calls .toUpperCase() on it
Function<String, String> upper = s -> s.toUpperCase();

// Method reference form — Java infers the receiver from the parameter
Function<String, String> upperRef = String::toUpperCase;

// Comparator example
List<String> names = List.of("Charlie", "alice", "Bob");
names.sort(String::compareToIgnoreCase);
// equivalent to: (a, b) -> a.compareToIgnoreCase(b)
```

The distinction between case 2 and case 3 can be confusing at first.
Think of it this way: if the object the method is called on is *already known*,
it's case 2. If that object is the first *parameter* to the lambda, it's case 3.

### 4. Constructor Reference

When the lambda creates a new instance of a class.

```java
// Lambda form
Supplier<ArrayList<String>> listMaker = () -> new ArrayList<>();

// Constructor reference form
Supplier<ArrayList<String>> listRef = ArrayList::new;

// With a parameter — picks the constructor that matches
Function<String, StringBuilder> sbMaker = StringBuilder::new;
// equivalent to: s -> new StringBuilder(s)

// Used frequently in stream collection
List<String> source = List.of("a", "b", "c");
List<String> copy = source.stream()
                          .collect(Collectors.toCollection(ArrayList::new));
```

---

## 🔄 Lambdas vs. Anonymous Inner Classes

Understanding the differences helps you know when each is appropriate.

```java
// Anonymous inner class
Runnable anonClass = new Runnable() {
    @Override
    public void run() {
        System.out.println("I'm an anon class. 'this' refers to THIS Runnable instance.");
    }
};

// Lambda
Runnable lambda = () -> System.out.println("I'm a lambda. 'this' refers to the ENCLOSING class.");
```

The key differences are:

**`this` keyword**: In an anonymous class, `this` refers to the anonymous class instance
itself. In a lambda, `this` refers to the enclosing class instance. This makes lambdas
behave more naturally in most cases.

**Memory**: Anonymous inner classes always create a new class file and instance.
Lambdas use `invokedynamic` bytecode and are more efficient — they don't necessarily
create a new object on every call (JVM can optimize this).

**State**: Anonymous inner classes can have fields and state. Lambdas cannot — they
are stateless by design (captured variables must be effectively final).

**When to still use anonymous classes**: When you need to implement an interface with
*more than one method*, when you need to track internal state, or when you need to
extend a class rather than implement an interface.

---

## 🏆 Best Practices

**Keep lambdas short.** If a lambda body is more than 2–3 lines, extract it into a
named private method and use a method reference instead. This improves readability and
makes the code unit-testable.

```java
// ❌ Too long for an inline lambda
list.stream()
    .filter(user -> {
        if (user.getAge() < 18) return false;
        if (!user.isActive()) return false;
        return user.getEmail() != null && user.getEmail().contains("@");
    })
    .collect(Collectors.toList());

// ✅ Extract to a named method
list.stream()
    .filter(this::isEligibleUser)
    .collect(Collectors.toList());

private boolean isEligibleUser(User user) {
    if (user.getAge() < 18) return false;
    if (!user.isActive()) return false;
    return user.getEmail() != null && user.getEmail().contains("@");
}
```

**Prefer method references over trivial lambdas.** `String::toUpperCase` is more
readable than `s -> s.toUpperCase()`. Use lambdas when the method reference form
would require explaining the mapping, but favor references when the intent is obvious.

**Never catch checked exceptions inside lambdas silently.** If you must handle a
checked exception, either wrap it in a utility or use a helper method that wraps the
exception in a `RuntimeException`.

```java
// ❌ Swallowing exceptions silently
list.stream()
    .map(path -> {
        try { return Files.readString(path); }
        catch (IOException e) { return ""; }  // silently swallowed
    });

// ✅ Use a helper wrapper
list.stream()
    .map(path -> readFileSafely(path))
    .filter(Optional::isPresent)
    .map(Optional::get);
```

**Don't use lambdas just to look "modern."** If a traditional `for` loop or `if` block
is clearer for a given situation, use it. Functional style improves clarity when the
transformation logic is naturally pipeline-shaped — it isn't always the right tool.

---

## 🧪 Complete Working Example

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class LambdaDemo {

    // A simple functional interface
    @FunctionalInterface
    interface StringTransformer {
        String transform(String input);
    }

    public static void main(String[] args) {

        // --- Basic lambda usage ---
        StringTransformer shout = s -> s.toUpperCase() + "!";
        System.out.println(shout.transform("hello")); // HELLO!

        // --- Method reference: static ---
        Function<String, Integer> parseNum = Integer::parseInt;
        System.out.println(parseNum.apply("42")); // 42

        // --- Method reference: instance on specific object ---
        String greeting = "Hello, World!";
        Supplier<String> getGreeting = greeting::toLowerCase;
        System.out.println(getGreeting.get()); // hello, world!

        // --- Method reference: instance on any object of type ---
        List<String> names = Arrays.asList("Charlie", "Alice", "Bob");
        names.sort(String::compareToIgnoreCase);
        System.out.println(names); // [Alice, Bob, Charlie]

        // --- Constructor reference ---
        Function<String, StringBuilder> sbBuilder = StringBuilder::new;
        StringBuilder sb = sbBuilder.apply("Initial text");
        System.out.println(sb); // Initial text

        // --- Capturing effectively final variable ---
        String separator = " | ";
        Function<List<String>, String> joiner = list ->
            String.join(separator, list);  // 'separator' is captured
        System.out.println(joiner.apply(names)); // Alice | Bob | Charlie

        // --- Composing lambdas ---
        StringTransformer trim = String::trim;
        StringTransformer shoutTrimmed = s -> trim.transform(shout.transform(s));
        System.out.println(shoutTrimmed.transform("  hi  ")); // HI!
    }
}
```

---

## 🔑 Key Takeaways

- A lambda is an anonymous function that implements a **functional interface**.
- The compiler infers parameter types from the **target type context**.
- Captured variables must be **effectively final** (the reference, not the object state).
- **Method references** are a concise form of lambdas — four kinds: static, instance-specific, instance-any, constructor.
- Lambdas use `invokedynamic` under the hood — they are more efficient than anonymous classes.
- `this` inside a lambda refers to the **enclosing class**, not the lambda itself.
- Keep lambdas short — extract long logic into named methods and reference them.
