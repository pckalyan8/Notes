# ⚡ Phase 5.3 — The Streams API in Java
### Data Processing Pipelines Done Right

---

## 🧠 What Is a Stream?

A **Stream** is not a data structure — it holds no data. Instead, it's a **pipeline
for processing data** that already lives in a collection, an array, or some generator.
Think of a stream like a conveyor belt in a factory: raw materials (your data) enter
one end, pass through a series of processing stations (operations), and a finished
product exits the other end.

The three pillars of every stream pipeline are:

1. **Source** — where the data comes from (a list, array, file, generator)
2. **Intermediate operations** — transformations that are **lazy** (they describe *what* to do, but do nothing yet)
3. **Terminal operation** — the trigger that actually **executes** the pipeline and produces a result

```
Source ──► filter() ──► map() ──► sorted() ──► collect()
             lazy        lazy      lazy         TRIGGERS execution
```

The "lazy" nature of intermediate operations is a crucial optimization: if you have
`filter().map().findFirst()`, the stream processes elements **one at a time** and stops
the moment it finds the first match — it doesn't need to filter or map the entire list.

---

## 🏭 Creating Streams (Sources)

### From Collections

The most common way — every `Collection` has `.stream()` and `.parallelStream()`.

```java
List<String> names = List.of("Alice", "Bob", "Charlie");
Stream<String> stream      = names.stream();
Stream<String> parallel    = names.parallelStream();

Set<Integer> numbers = Set.of(1, 2, 3, 4, 5);
Stream<Integer> fromSet = numbers.stream();
```

### From Arrays

```java
String[] arr = {"Java", "Python", "Go"};
Stream<String>  fromArray    = Arrays.stream(arr);
Stream<String>  fromRange    = Arrays.stream(arr, 1, 3); // ["Python", "Go"]

// Primitive arrays directly produce primitive streams (no boxing)
int[] ints = {1, 2, 3, 4, 5};
IntStream intStream = Arrays.stream(ints); // IntStream, not Stream<Integer>
```

### `Stream.of()` — From Explicit Values

```java
Stream<String> s = Stream.of("a", "b", "c");
Stream<String> single = Stream.of("only one");
Stream<Object> empty = Stream.empty(); // zero elements
```

### Infinite Streams: `generate()` and `iterate()`

These create unbounded streams, so you must always pair them with a limiting operation
like `limit()` or `takeWhile()`.

```java
// generate(): each element from a Supplier — stateless
Stream<Double> randoms = Stream.generate(Math::random);
randoms.limit(5).forEach(System.out::println); // 5 random numbers

Stream<String> hellos = Stream.generate(() -> "hello");
hellos.limit(3).forEach(System.out::println); // hello hello hello

// iterate(seed, UnaryOperator): each element derived from the previous
Stream<Integer> evens = Stream.iterate(0, n -> n + 2);
evens.limit(5).forEach(System.out::println); // 0 2 4 6 8

// Java 9+ iterate with predicate — like a for loop with a stop condition
Stream<Integer> under100 = Stream.iterate(0, n -> n < 100, n -> n + 2);
// 0, 2, 4, ..., 98  (stops before 100)
```

### Primitive Streams: `IntStream`, `LongStream`, `DoubleStream`

For numeric work, always prefer these over `Stream<Integer>` etc. — they avoid
boxing, carry extra numeric methods, and are significantly more efficient.

```java
IntStream oneToTen     = IntStream.range(1, 11);    // 1 to 10 (exclusive end)
IntStream oneToTenIncl = IntStream.rangeClosed(1, 10); // 1 to 10 (inclusive end)
LongStream bigRange    = LongStream.rangeClosed(1L, 1_000_000L);
DoubleStream doubles   = DoubleStream.of(1.1, 2.2, 3.3);
```

### From Files and Other Sources

```java
// Each line of a file as a stream element
try (Stream<String> lines = Files.lines(Path.of("data.txt"))) {
    lines.filter(line -> !line.isBlank())
         .forEach(System.out::println);
} // stream automatically closed (it implements AutoCloseable)

// BufferedReader.lines()
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    reader.lines().map(String::trim).forEach(System.out::println);
}
```

---

## 🔄 Intermediate Operations (Lazy Transformations)

Intermediate operations return a new stream. They are lazy — calling them does
nothing by itself. They just describe what the pipeline *will* do when a terminal
operation is invoked.

### `filter(Predicate<T>)` — Keep Matching Elements

Arguably the most used stream operation. It discards elements that don't pass the test.

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList()); // [2, 4, 6, 8, 10]

// Combining predicates
Predicate<String> notEmpty = Predicate.not(String::isEmpty);
Predicate<String> longWord  = s -> s.length() > 4;

List<String> words = List.of("hi", "", "hello", "world", "", "ok");
List<String> result = words.stream()
    .filter(notEmpty.and(longWord))
    .collect(Collectors.toList()); // [hello, world]
```

### `map(Function<T, R>)` — Transform Elements

Transforms each element into something else — the output type can be different from
the input type.

```java
List<String> names = List.of("alice", "bob", "charlie");

// String → String (same type)
List<String> upper = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList()); // [ALICE, BOB, CHARLIE]

// String → Integer (different type)
List<Integer> lengths = names.stream()
    .map(String::length)
    .collect(Collectors.toList()); // [5, 3, 7]
```

### `mapToInt()`, `mapToLong()`, `mapToDouble()` — Map to Primitives

When your transformation produces a number, immediately switch to a primitive stream
to unlock numeric operations like `sum()`, `average()`, `min()`, `max()`.

```java
List<String> words = List.of("apple", "banana", "cherry");

int totalLength = words.stream()
    .mapToInt(String::length)  // produces IntStream — no boxing
    .sum();                    // 5 + 6 + 6 = 17

OptionalDouble avgLength = words.stream()
    .mapToInt(String::length)
    .average(); // OptionalDouble: 5.666...
```

### `flatMap(Function<T, Stream<R>>)` — Flatten Nested Structures

`map()` transforms each element into one output. `flatMap()` transforms each element
into a **stream of outputs**, then flattens all those streams into a single stream.
This is essential for working with nested collections.

```java
// Scenario: each order has multiple items — we want a flat list of all items

List<List<String>> nestedList = List.of(
    List.of("apple", "banana"),
    List.of("cherry"),
    List.of("date", "elderberry", "fig")
);

// Without flatMap — produces Stream<List<String>>, not what we want:
// nestedList.stream().map(Collection::stream) ← a stream of streams!

// With flatMap — flattens to Stream<String>
List<String> allFruits = nestedList.stream()
    .flatMap(Collection::stream) // each inner list → stream, then all merged
    .collect(Collectors.toList());
// [apple, banana, cherry, date, elderberry, fig]


// Real-world example: extract all words from multiple sentences
List<String> sentences = List.of(
    "Hello World",
    "Java Streams are powerful",
    "Learn and practice"
);

List<String> allWords = sentences.stream()
    .flatMap(sentence -> Arrays.stream(sentence.split(" ")))
    .map(String::toLowerCase)
    .distinct()
    .sorted()
    .collect(Collectors.toList());
// [and, are, hello, java, learn, powerful, practice, streams, world]
```

The mental model for `flatMap`: map each element to a *collection/stream* of things,
then pull all those things out and put them in one big stream.

### `distinct()` — Remove Duplicates

Uses `.equals()` to identify duplicates. For custom objects, make sure you've
implemented `equals()` and `hashCode()` properly.

```java
List<Integer> withDupes = List.of(1, 3, 2, 3, 4, 1, 5, 2);
List<Integer> unique = withDupes.stream()
    .distinct()
    .collect(Collectors.toList()); // [1, 3, 2, 4, 5] (order preserved from source)
```

### `sorted()` and `sorted(Comparator)` — Sort Elements

Without a comparator, elements must implement `Comparable`. With a comparator, you
control the order completely.

```java
List<String> names = List.of("Charlie", "Alice", "Bob", "Diana");

// Natural order (Comparable — alphabetical for String)
List<String> alpha = names.stream()
    .sorted()
    .collect(Collectors.toList()); // [Alice, Bob, Charlie, Diana]

// Custom order — by length, then alphabetically
List<String> byLength = names.stream()
    .sorted(Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder()))
    .collect(Collectors.toList()); // [Bob, Alice, Diana, Charlie]

// Reverse alphabetical
List<String> reversed = names.stream()
    .sorted(Comparator.reverseOrder())
    .collect(Collectors.toList()); // [Diana, Charlie, Bob, Alice]
```

### `peek(Consumer<T>)` — Debug Side Effects

`peek()` performs an action on each element *without changing the element*. It's
primarily a debugging tool — never use it for actual business logic. Note that because
streams are lazy, `peek()` only fires for elements that actually reach it, which makes
it an excellent way to understand what a pipeline is processing.

```java
List<String> result = List.of("hello", "world", "java", "streams")
    .stream()
    .peek(s -> System.out.println("Before filter: " + s))
    .filter(s -> s.length() > 4)
    .peek(s -> System.out.println("After filter:  " + s))
    .map(String::toUpperCase)
    .collect(Collectors.toList());
// Before filter: hello   After filter: hello
// Before filter: world   After filter: world
// Before filter: java    (no "After" — filtered out)
// Before filter: streams After filter: streams
// result: [HELLO, WORLD, STREAMS]
```

### `limit(n)` and `skip(n)` — Windowing

`limit(n)` takes the first n elements. `skip(n)` discards the first n elements and
takes the rest. Together, they implement pagination.

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

List<Integer> firstThree = numbers.stream()
    .limit(3)
    .collect(Collectors.toList()); // [1, 2, 3]

List<Integer> afterThree = numbers.stream()
    .skip(3)
    .collect(Collectors.toList()); // [4, 5, 6, 7, 8, 9, 10]

// Pagination: page size = 3, get page 2 (0-indexed)
int pageSize = 3;
int pageIndex = 1; // second page
List<Integer> page2 = numbers.stream()
    .skip((long) pageIndex * pageSize)
    .limit(pageSize)
    .collect(Collectors.toList()); // [4, 5, 6]
```

### `takeWhile()` and `dropWhile()` — Conditional Windowing (Java 9+)

`takeWhile()` keeps elements from the front *as long as* the predicate is true, then
stops. `dropWhile()` skips elements from the front while the predicate is true, then
takes the rest. Unlike `filter()`, these only examine elements from the beginning
and stop/start at the first failure.

```java
List<Integer> sorted = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9);

// takeWhile: take elements while they are less than 5
List<Integer> taken = sorted.stream()
    .takeWhile(n -> n < 5)
    .collect(Collectors.toList()); // [1, 2, 3, 4] — stops at 5

// dropWhile: skip elements while they are less than 5
List<Integer> dropped = sorted.stream()
    .dropWhile(n -> n < 5)
    .collect(Collectors.toList()); // [5, 6, 7, 8, 9] — starts from 5
```

---

## 🏁 Terminal Operations (Execution Triggers)

Terminal operations consume the stream and produce a result (or a side effect). Once
a terminal operation runs, the stream is **exhausted** and cannot be reused.

### `forEach(Consumer<T>)` — Consume Each Element

The simplest terminal operation. Use for side effects only — logging, printing,
saving to a database. Do not use it as a replacement for `collect()` to build a list.

```java
List<String> names = List.of("Alice", "Bob", "Charlie");
names.stream().forEach(System.out::println);

// forEachOrdered: guarantees order in parallel streams
names.parallelStream().forEachOrdered(System.out::println);
```

### `collect(Collector)` — The Most Powerful Terminal Operation

`collect()` accumulates stream elements into a mutable result container — typically
a list, set, map, or string. See the dedicated Collectors section below.

### `toList()` — Java 16+ Shorthand

Java 16 added a direct `.toList()` method on streams, returning an **unmodifiable**
list. It's shorter than `collect(Collectors.toList())`, but note that the returned
list is unmodifiable (you can't add or remove elements).

```java
List<String> upper = List.of("a", "b", "c").stream()
    .map(String::toUpperCase)
    .toList(); // [A, B, C] — unmodifiable

// If you need a mutable list, use collect(Collectors.toList())
List<String> mutable = List.of("a", "b", "c").stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList()); // mutable ArrayList
```

### `count()` — Count Elements

```java
long count = List.of("apple", "banana", "cherry", "avocado").stream()
    .filter(s -> s.startsWith("a"))
    .count(); // 2
```

### `findFirst()` and `findAny()` — Short-Circuit Searching

Both return `Optional<T>`. `findFirst()` returns the first element in encounter order
(deterministic). `findAny()` may return any element — useful in parallel streams
where "any" is more efficient than "the first one".

```java
Optional<String> first = List.of("banana", "apple", "cherry").stream()
    .filter(s -> s.startsWith("a"))
    .findFirst();
System.out.println(first.orElse("none")); // apple

// findAny: better for parallel streams — returns whatever thread finds first
Optional<Integer> any = IntStream.range(1, 1000)
    .parallel()
    .filter(n -> n % 7 == 0)
    .boxed()
    .findAny(); // might be 7, 14, 42... any multiple of 7
```

### Matching: `anyMatch()`, `allMatch()`, `noneMatch()`

These short-circuit as soon as the answer is determined — very efficient for large
streams. They all take a `Predicate<T>` and return `boolean`.

```java
List<Integer> numbers = List.of(2, 4, 6, 7, 8, 10);

boolean anyOdd  = numbers.stream().anyMatch(n -> n % 2 != 0); // true (7 is odd)
boolean allEven = numbers.stream().allMatch(n -> n % 2 == 0); // false (7 is odd)
boolean noneNeg = numbers.stream().noneMatch(n -> n < 0);     // true
```

### `min()` and `max()` — Extremes

These return `Optional<T>` because the stream might be empty.

```java
Optional<String> shortest = List.of("apple", "kiwi", "banana", "fig").stream()
    .min(Comparator.comparingInt(String::length));
System.out.println(shortest.get()); // fig

Optional<String> alphabetLast = List.of("apple", "kiwi", "banana", "fig").stream()
    .max(Comparator.naturalOrder());
System.out.println(alphabetLast.get()); // kiwi
```

### `reduce()` — Fold / Aggregate

`reduce()` is the generalization behind `sum()`, `max()`, `min()`, `count()`, and
string joining. It takes a **binary operator** that combines two elements into one,
repeatedly, until only one value remains.

```java
// With identity (starting value) — always produces a value
int sum = List.of(1, 2, 3, 4, 5).stream()
    .reduce(0, (a, b) -> a + b); // 0+1+2+3+4+5 = 15
// Same as:
int sum2 = List.of(1, 2, 3, 4, 5).stream()
    .reduce(0, Integer::sum);

// Without identity — returns Optional (stream might be empty)
Optional<Integer> product = List.of(1, 2, 3, 4, 5).stream()
    .reduce((a, b) -> a * b); // 1*2*3*4*5 = 120

// String concatenation using reduce (use Collectors.joining() in practice — it's faster)
Optional<String> joined = List.of("Hello", " ", "World").stream()
    .reduce((a, b) -> a + b); // "Hello World"
```

### `toArray()` — Convert to Array

```java
String[] arr = List.of("a", "b", "c").stream()
    .map(String::toUpperCase)
    .toArray(String[]::new); // pass constructor reference for typed array
// [A, B, C]
```

### Numeric Terminal Operations (on Primitive Streams)

These are only available on `IntStream`, `LongStream`, and `DoubleStream`. They are
the main reason to use primitive streams for numeric processing.

```java
IntStream nums = IntStream.rangeClosed(1, 10);
System.out.println(nums.sum());     // 55

IntStream.rangeClosed(1, 10).average().ifPresent(System.out::println); // 5.5

int max = IntStream.of(3, 1, 4, 1, 5, 9, 2, 6).max().getAsInt(); // 9

// summaryStatistics() gives you everything at once
IntSummaryStatistics stats = IntStream.rangeClosed(1, 100).summaryStatistics();
System.out.println(stats.getCount());   // 100
System.out.println(stats.getSum());     // 5050
System.out.println(stats.getMin());     // 1
System.out.println(stats.getMax());     // 100
System.out.println(stats.getAverage()); // 50.5
```

---

## 🧺 Collectors — The Powerhouse of `collect()`

The `Collectors` utility class provides factory methods for all the common collection
strategies. These are the ones you'll use most.

### Basic Collection

```java
List<String> words = List.of("apple", "banana", "cherry", "avocado");

// toList() — mutable ArrayList
List<String> list = words.stream().collect(Collectors.toList());

// toUnmodifiableList() — immutable (Java 10+)
List<String> immutable = words.stream().collect(Collectors.toUnmodifiableList());

// toSet() — removes duplicates (order not guaranteed)
Set<String> set = words.stream().collect(Collectors.toSet());

// toCollection() — specify the exact collection type you want
LinkedList<String> linked = words.stream()
    .collect(Collectors.toCollection(LinkedList::new));
TreeSet<String> sorted = words.stream()
    .collect(Collectors.toCollection(TreeSet::new));
```

### `joining()` — String Concatenation

```java
List<String> parts = List.of("Java", "is", "great");

String plain    = parts.stream().collect(Collectors.joining());           // Javaisgreat
String spaced   = parts.stream().collect(Collectors.joining(" "));        // Java is great
String brackets = parts.stream().collect(Collectors.joining(", ", "[", "]")); // [Java, is, great]
```

### `toMap()` — Collect into a Map

```java
List<String> words2 = List.of("apple", "banana", "cherry");

// key = the word itself, value = its length
Map<String, Integer> wordLengths = words2.stream()
    .collect(Collectors.toMap(
        Function.identity(), // key: the word itself
        String::length       // value: the length
    ));
// {apple=5, banana=6, cherry=6}

// ⚠️ Duplicate key problem: if two elements produce the same key, an exception is thrown.
// Use the merge function to resolve conflicts:
List<String> withDupes = List.of("apple", "apricot", "banana", "blueberry");
Map<Character, String> firstByLetter = withDupes.stream()
    .collect(Collectors.toMap(
        s -> s.charAt(0),       // key: first letter
        Function.identity(),    // value: the word
        (existing, newValue) -> existing // on conflict: keep existing
    ));
// {a=apple, b=banana}
```

### `groupingBy()` — Group Elements Into a Map of Lists

`groupingBy()` is one of the most powerful collectors. It partitions a stream into
groups based on a classifier function, producing a `Map<K, List<V>>`.

```java
List<String> words3 = List.of("apple", "avocado", "banana", "blueberry", "cherry", "cranberry");

// Group by first letter
Map<Character, List<String>> byLetter = words3.stream()
    .collect(Collectors.groupingBy(s -> s.charAt(0)));
// {a=[apple, avocado], b=[banana, blueberry], c=[cherry, cranberry]}

// Group and count (downstream collector)
Map<Character, Long> countByLetter = words3.stream()
    .collect(Collectors.groupingBy(
        s -> s.charAt(0),
        Collectors.counting()  // downstream: count elements in each group
    ));
// {a=2, b=2, c=2}

// Group and collect to a set instead of a list
Map<Character, Set<String>> setsByLetter = words3.stream()
    .collect(Collectors.groupingBy(
        s -> s.charAt(0),
        Collectors.toSet()
    ));

// Multi-level grouping (group employees by department, then by seniority)
record Employee(String name, String dept, String level) {}
List<Employee> employees = List.of(
    new Employee("Alice",   "Engineering", "Senior"),
    new Employee("Bob",     "Engineering", "Junior"),
    new Employee("Charlie", "Marketing",   "Senior"),
    new Employee("Diana",   "Engineering", "Senior")
);
Map<String, Map<String, List<Employee>>> grouped = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::dept,
        Collectors.groupingBy(Employee::level)
    ));
// {Engineering={Senior=[Alice, Diana], Junior=[Bob]}, Marketing={Senior=[Charlie]}}
```

### `partitioningBy()` — Split Into Two Groups

When your classifier is boolean (true/false), `partitioningBy()` is cleaner than
`groupingBy()` — it always produces a map with exactly two keys: `true` and `false`.

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

Map<Boolean, List<Integer>> oddEven = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
// {true=[2, 4, 6, 8, 10], false=[1, 3, 5, 7, 9]}

List<Integer> evens = oddEven.get(true);
List<Integer> odds  = oddEven.get(false);
```

### `counting()`, `summingInt()`, `averagingInt()`

These are **downstream collectors** — designed to be used inside `groupingBy()` or
`partitioningBy()`, though they can also be used standalone.

```java
record Product(String category, String name, double price) {}
List<Product> products = List.of(
    new Product("Electronics", "Phone",  699.0),
    new Product("Electronics", "Laptop", 1299.0),
    new Product("Books",       "Java",   49.0),
    new Product("Books",       "Clean Code", 39.0)
);

// Count products per category
Map<String, Long> countByCategory = products.stream()
    .collect(Collectors.groupingBy(Product::category, Collectors.counting()));
// {Electronics=2, Books=2}

// Average price per category
Map<String, Double> avgPriceByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::category,
        Collectors.averagingDouble(Product::price)
    ));
// {Electronics=999.0, Books=44.0}

// Total price per category
Map<String, Double> totalByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::category,
        Collectors.summingDouble(Product::price)
    ));
// {Electronics=1998.0, Books=88.0}
```

### `collectingAndThen()` — Post-Process a Collection Result

Applies a finisher function to the result of another collector. A classic use case is
wrapping the result list in an unmodifiable view.

```java
List<String> immutableUpper = List.of("a", "b", "c").stream()
    .map(String::toUpperCase)
    .collect(Collectors.collectingAndThen(
        Collectors.toList(),            // collect to a list first
        Collections::unmodifiableList   // then make it unmodifiable
    ));
```

### `teeing()` — Two Collectors at Once (Java 12+)

`teeing()` feeds each element to two different collectors simultaneously and then
merges their results. It's the streams equivalent of computing two aggregates in one pass.

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

record MinMax(int min, int max) {}
MinMax minMax = nums.stream()
    .collect(Collectors.teeing(
        Collectors.minBy(Integer::compareTo),  // collector 1: find min
        Collectors.maxBy(Integer::compareTo),  // collector 2: find max
        (min, max) -> new MinMax(min.get(), max.get()) // merge function
    ));
System.out.println(minMax); // MinMax[min=1, max=10]
```

---

## ⚡ Parallel Streams

A parallel stream splits the data across multiple threads using the **Fork-Join
framework**, then merges the results. For the right workload, this can dramatically
improve performance with zero extra code.

```java
// Sequential
long sequentialSum = LongStream.rangeClosed(1, 1_000_000).sum();

// Parallel — splits work across CPU cores automatically
long parallelSum = LongStream.rangeClosed(1, 1_000_000).parallel().sum();

// Or start from a collection
List<Integer> bigList = ...; // millions of elements
bigList.parallelStream()
       .filter(n -> n % 2 == 0)
       .mapToLong(Integer::longValue)
       .sum();
```

### When Parallel Streams Help vs. Hurt

Parallel streams are not always faster — there is overhead to splitting and merging
the work. Use them when all of the following are true:

**Data volume is large enough** to justify the overhead (generally thousands+ of
elements, though the real threshold depends on the operation cost).

**The operations are CPU-bound and stateless.** If each element can be processed
independently — like a mathematical transformation — parallel is great. If elements
depend on each other (stateful), parallel will produce incorrect results.

**The data source splits well.** `ArrayList`, arrays, and `IntStream.range()` split
perfectly. `LinkedList` splits poorly. `HashSet` and `HashMap` split reasonably.

**There is no shared mutable state.** Modifying a shared collection from parallel
stream operations causes race conditions and data corruption.

```java
// ✅ Good candidate for parallel: CPU-bound, stateless, large data
List<Double> result = bigNumberList.parallelStream()
    .map(n -> Math.sqrt(Math.pow(n, 3) + Math.log(n)))  // expensive math
    .filter(d -> d > 100.0)
    .collect(Collectors.toList());

// ❌ Bad candidate: accumulating into shared mutable state
List<String> sharedList = new ArrayList<>();
words.parallelStream()
     .filter(s -> s.length() > 3)
     .forEach(sharedList::add); // RACE CONDITION! Don't do this.

// ✅ Correct alternative: use collect() instead
List<String> safeResult = words.parallelStream()
     .filter(s -> s.length() > 3)
     .collect(Collectors.toList()); // thread-safe reduction
```

### `Spliterator` — The Parallel Splitting Mechanism

Under the hood, parallel streams use a `Spliterator` — an interface that knows how
to split a data source into two halves recursively. For standard collections, Java
provides efficient built-in `Spliterator` implementations. For custom data sources,
you can implement your own.

---

## 🏗️ Complete Real-World Example

```java
import java.util.*;
import java.util.stream.*;
import java.util.function.*;

public class OrderAnalysis {

    record Order(String customerId, String productId,
                 String category, int quantity, double unitPrice) {
        double total() { return quantity * unitPrice; }
    }

    public static void main(String[] args) {
        List<Order> orders = List.of(
            new Order("C1", "P1", "Electronics", 1, 999.0),
            new Order("C2", "P2", "Books",       3, 29.99),
            new Order("C1", "P3", "Electronics", 2, 499.0),
            new Order("C3", "P4", "Clothing",    5, 59.99),
            new Order("C2", "P1", "Electronics", 1, 999.0),
            new Order("C3", "P5", "Books",       2, 19.99),
            new Order("C1", "P6", "Clothing",    1, 89.99)
        );

        // 1. Total revenue
        double totalRevenue = orders.stream()
            .mapToDouble(Order::total)
            .sum();
        System.out.printf("Total revenue: $%.2f%n", totalRevenue);

        // 2. Revenue per category
        Map<String, Double> revenueByCategory = orders.stream()
            .collect(Collectors.groupingBy(
                Order::category,
                Collectors.summingDouble(Order::total)
            ));
        revenueByCategory.forEach((cat, rev) ->
            System.out.printf("  %s: $%.2f%n", cat, rev));

        // 3. Top spending customer
        Map<String, Double> spendingByCustomer = orders.stream()
            .collect(Collectors.groupingBy(
                Order::customerId,
                Collectors.summingDouble(Order::total)
            ));
        spendingByCustomer.entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .ifPresent(e ->
                System.out.printf("Top customer: %s ($%.2f)%n", e.getKey(), e.getValue()));

        // 4. Customers who bought Electronics
        Set<String> electronicsBuyers = orders.stream()
            .filter(o -> o.category().equals("Electronics"))
            .map(Order::customerId)
            .collect(Collectors.toSet());
        System.out.println("Electronics buyers: " + electronicsBuyers);

        // 5. Average order value
        OptionalDouble avg = orders.stream()
            .mapToDouble(Order::total)
            .average();
        avg.ifPresent(a -> System.out.printf("Avg order value: $%.2f%n", a));

        // 6. Orders above average
        double avgValue = avg.orElse(0);
        List<String> aboveAvg = orders.stream()
            .filter(o -> o.total() > avgValue)
            .sorted(Comparator.comparingDouble(Order::total).reversed())
            .map(o -> String.format("%s → $%.2f", o.productId(), o.total()))
            .collect(Collectors.toList());
        System.out.println("Above-average orders: " + aboveAvg);
    }
}
```

---

## ⚠️ Best Practices & Common Pitfalls

**Streams are single-use.** Once a terminal operation is called, the stream is
exhausted and cannot be reused. Attempting to reuse a stream throws
`IllegalStateException`. If you need to run multiple operations on the same data,
create a new stream each time.

**Never mutate the source during streaming.** Modifying the list you're streaming
while the stream is running causes `ConcurrentModificationException` (for sequential)
or undefined behavior (for parallel). Always operate on the data, not on the source.

**Avoid stateful lambdas in streams.** Side effects in `filter()`, `map()`, and other
intermediate operations lead to bugs, especially with parallel streams. Use `peek()`
only for debugging, never for business logic.

**Don't over-stream simple operations.** If a traditional `for` loop is clearer and
the operation is trivial (e.g., print each element), use the loop. Streams improve
readability for multi-step transformations, not for simple single-step operations.

**Use primitive streams for numbers.** Always use `IntStream`, `LongStream`, or
`DoubleStream` when processing numeric data — they avoid boxing/unboxing and come
with helpful methods like `sum()`, `average()`, and `summaryStatistics()`.

**Prefer `collect()` over `reduce()` for mutable containers.** `reduce()` is designed
for immutable aggregation. Collecting into an `ArrayList` with `reduce()` is
inefficient and breaks with parallel streams. Use `collect()` for that purpose.

---

## 🔑 Key Takeaways

- A stream is a **pipeline**, not a data structure. Source → intermediate ops (lazy) → terminal op (triggers execution).
- Lazy evaluation means the pipeline only processes what's needed — critical for `findFirst()`, `anyMatch()`, and `limit()` efficiency.
- `flatMap()` is `map()` + flatten — essential for nested collections.
- `collect()` with `Collectors` handles virtually any aggregation: lists, maps, groupings, partitions, strings.
- `groupingBy()` is the stream equivalent of SQL's `GROUP BY` — master it.
- Use **primitive streams** for numbers; use **parallel streams** carefully, only for large, stateless, CPU-bound operations.
- A stream can only be consumed **once** — create a new stream from the source if you need to process again.
