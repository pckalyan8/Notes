# 4.2 Generics in Java

## What & Why

Before Java 5, collections were collections of raw `Object` references. You could put anything into a `List`, which meant the compiler could never catch type mismatches — you'd only find out at runtime via a `ClassCastException`. Generics solved this by introducing **type parameters** that let you say "this is a `List` of `String`" at compile time, making the mismatch an error before the program ever runs.

Beyond safety, generics eliminate the constant casting that pre-Java-5 code required. The compiler inserts casts for you, and they're guaranteed correct — so you get both safety and readability in one feature.

---

## 1. Generic Classes

A generic class declares one or more **type parameters** in angle brackets after the class name. By convention, single uppercase letters are used: `T` (Type), `E` (Element), `K` (Key), `V` (Value), `N` (Number).

```java
// A simple generic box that can hold any type
public class Box<T> {
    private T value;  // T is the type parameter — resolved when Box is instantiated

    public Box(T value) {
        this.value = value;
    }

    public T getValue() {
        return value;
    }

    public void setValue(T value) {
        this.value = value;
    }

    @Override
    public String toString() {
        return "Box[" + value + "]";
    }
}
```

When you instantiate `Box`, you supply the actual type argument:

```java
Box<String> stringBox = new Box<>("Hello");
// The compiler now knows getValue() returns String — no cast needed
String content = stringBox.getValue();

Box<Integer> intBox = new Box<>(42);
int number = intBox.getValue(); // auto-unboxed to int

// The diamond operator <> (Java 7+) lets the compiler infer the type argument
Box<Double> doubleBox = new Box<>(3.14);  // compiler infers Double
```

---

## 2. Generic Methods

You can make an individual method generic even if its class is not. The type parameter is declared **before the return type**:

```java
public class GenericUtils {

    // <T> declares this method's own type parameter
    public static <T> T getFirst(List<T> list) {
        if (list.isEmpty()) {
            throw new NoSuchElementException("List is empty");
        }
        return list.get(0);
    }

    // A method with multiple type parameters
    public static <K, V> Map<V, K> invertMap(Map<K, V> original) {
        Map<V, K> inverted = new HashMap<>();
        original.forEach((k, v) -> inverted.put(v, k));
        return inverted;
    }
}

// Usage — the compiler infers T from the argument
String first = GenericUtils.getFirst(List.of("alpha", "beta", "gamma")); // "alpha"
Integer num   = GenericUtils.getFirst(List.of(10, 20, 30));              // 10
```

---

## 3. Bounded Type Parameters

By default, a type parameter `T` can be substituted by *any* type. **Bounds** restrict the set of acceptable types and, crucially, allow you to call methods that the bound type guarantees.

### Upper Bounds (`extends`)

`<T extends Number>` means T must be `Number` or any subclass of `Number` (like `Integer`, `Double`, `Long`). Inside the method, you can call any method declared by `Number`.

```java
// Without bound, we couldn't call .doubleValue() — not all T would have it
public static <T extends Number> double sum(List<T> numbers) {
    double total = 0;
    for (T n : numbers) {
        total += n.doubleValue(); // safe because T is guaranteed to be a Number
    }
    return total;
}

System.out.println(sum(List.of(1, 2, 3)));          // 6.0
System.out.println(sum(List.of(1.5, 2.5, 3.0)));    // 7.0
```

You can combine an upper bound with an interface requirement using `&`:

```java
// T must extend Animal AND implement Comparable<T>
public static <T extends Animal & Comparable<T>> T findLargest(List<T> animals) {
    return Collections.max(animals);
}
```

### Multiple Bounds

When combining bounds, the **class must come first**, followed by interfaces:

```java
// Correct: class first, then interfaces
<T extends Serializable & Comparable<T>>

// Wrong: interface before another bound type that might be a class
// <T extends Comparable<T> & SomeClass> — compile error if SomeClass is a class
```

---

## 4. Wildcards — The Most Misunderstood Feature

Wildcards (`?`) appear in *type positions* (variable declarations, method parameters) when you don't know or don't care about the exact type, but you do care about what you can *do* with the collection.

### Unbounded Wildcard `<?>`

Use when you only need methods defined on `Object`, like printing a list of any type:

```java
// Works for List<String>, List<Integer>, List<Anything>
public static void printList(List<?> list) {
    for (Object element : list) {
        System.out.println(element);
    }
    // You CANNOT add to this list (except null) because the type is unknown
    // list.add("something"); // compile error — unsafe
}
```

### Upper-Bounded Wildcard `<? extends T>` — Producer

Use when you want to **read** from a collection. The collection *produces* elements for you to consume.

```java
// Can accept List<Integer>, List<Double>, List<Float> — any subtype of Number
public static double sumAll(List<? extends Number> numbers) {
    double sum = 0;
    for (Number n : numbers) {
        sum += n.doubleValue(); // reading is safe
    }
    // numbers.add(new Integer(1)); // compile error — can't add, type is unknown
    return sum;
}
```

Think of it this way: if the list holds `List<Integer>` and you try to add a `Double`, you'd corrupt it. The compiler prevents this.

### Lower-Bounded Wildcard `<? super T>` — Consumer

Use when you want to **write** to a collection. The collection *consumes* elements you produce.

```java
// Can accept List<Number>, List<Object> — any supertype of Integer
public static void addIntegers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
    list.add(3);
    // Reading gives Object — you can't know the exact type
    Object first = list.get(0); // only Object, not Integer
}
```

### The PECS Rule

This is the golden rule for wildcards — **P**roducer `E`xtends, **C**onsumer `S`uper:

```java
// PECS in action: copy from src (producer) to dest (consumer)
public static <T> void copy(List<? extends T> src,   // produces T
                             List<? super T>   dest) { // consumes T
    for (T element : src) {
        dest.add(element);
    }
}

List<Integer> integers = List.of(1, 2, 3);
List<Number>  numbers  = new ArrayList<>();
copy(integers, numbers); // compiles — Integer extends Number
System.out.println(numbers); // [1, 2, 3]
```

---

## 5. Type Erasure — What Actually Happens at Runtime

Java generics are a **compile-time feature only**. The JVM has no knowledge of type parameters. The compiler:

1. Checks all generic types are used correctly (type safety).
2. **Erases** all type parameter information, replacing `T` with its bound (or `Object` if unbounded).
3. Inserts **casts** wherever necessary.

```java
// What you write:
Box<String> box = new Box<>("hello");
String value = box.getValue();

// What the compiler generates (approximately):
Box box = new Box("hello");
String value = (String) box.getValue(); // cast inserted automatically
```

This explains several puzzling rules in Java generics:

**You cannot create generic arrays:**
```java
T[] array = new T[10];  // compile error — T is erased, JVM can't create T[]
// Workaround:
T[] array = (T[]) new Object[10]; // unchecked cast — generally avoid
```

**You cannot use instanceof with a parameterised type:**
```java
if (obj instanceof List<String>) { }  // compile error — type info gone at runtime
if (obj instanceof List<?>)       { }  // ok — unbounded wildcard is fine
```

**Generic type info is not available via `getClass()` at runtime:**
```java
List<String> strings  = new ArrayList<>();
List<Integer> integers = new ArrayList<>();
System.out.println(strings.getClass() == integers.getClass()); // true — both ArrayList
```

---

## 6. Generic Interfaces and Inheritance

```java
// A generic interface
public interface Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    void save(T entity);
    void delete(ID id);
}

// Concrete implementation specifies the type arguments
public class UserRepository implements Repository<User, Long> {
    @Override
    public Optional<User> findById(Long id) { /* ... */ return Optional.empty(); }

    @Override
    public List<User> findAll() { /* ... */ return new ArrayList<>(); }

    @Override
    public void save(User user) { /* ... */ }

    @Override
    public void delete(Long id) { /* ... */ }
}
```

A class can also extend another generic class with its own type parameter flowing through:

```java
public abstract class BaseService<T, ID> {
    protected abstract Repository<T, ID> getRepository();

    public Optional<T> findById(ID id) {
        return getRepository().findById(id);
    }
}

public class UserService extends BaseService<User, Long> {
    @Override
    protected Repository<User, Long> getRepository() {
        return new UserRepository();
    }
}
```

---

## 7. Raw Types and Why You Should Never Use Them

A **raw type** is a generic class used without type arguments. Raw types exist solely for backward compatibility with pre-Java-5 code. Using them defeats the entire purpose of generics:

```java
// Raw type — the compiler doesn't know what's inside
List rawList = new ArrayList();
rawList.add("hello");
rawList.add(42);          // no compile error — no type checking at all

String s = (String) rawList.get(1); // compiles but throws ClassCastException at runtime

// Always use parameterised types
List<String> safeList = new ArrayList<>();
safeList.add("hello");
// safeList.add(42);  // compile error — caught early, where it belongs
```

---

## ⚡ Best Practices & Important Points

**Favour generic methods over generic classes** when the generality is needed only for one or two methods. This keeps the class interface cleaner.

**Use bounded wildcards in API parameters, not in return types.** A return type of `List<? extends Number>` forces callers to deal with the wildcard; returning `List<Number>` is cleaner. Wildcards belong on the *consuming* side (parameters).

**Apply PECS every time you design a method that takes a collection.** If the method reads from the collection, use `? extends T`. If it writes to it, use `? super T`. If it does both, use `T` directly.

**Do not mix raw types and generics.** If you see compiler warnings about "unchecked operations", treat them as errors. They are hiding potential `ClassCastException` bugs that will surface at runtime.

**Generic types cannot be instantiated directly.** Because of erasure, `new T()` is illegal. Common workarounds include passing a `Class<T>` parameter and using reflection (`clazz.getDeclaredConstructor().newInstance()`), or using a `Supplier<T>` lambda.

**Understand that `List<Integer>` is NOT a `List<Number>`.** This surprises many developers. Even though `Integer extends Number`, `List<Integer>` does not extend `List<Number>`. This is why wildcards exist — `List<? extends Number>` is the type that captures both.
