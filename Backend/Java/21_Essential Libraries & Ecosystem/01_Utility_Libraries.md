# 21.1 — Utility Libraries

> **Libraries covered:** Apache Commons · Google Guava · Lombok · MapStruct · ModelMapper
>
> Every Java project eventually reaches for utility code that the standard library doesn't provide — null-safe string operations, immutable collections with richer APIs, object mapping between layers, and elimination of boilerplate. These five libraries are the workhorses of the Java ecosystem. Understanding them deeply means you stop reinventing wheels and start building on a proven foundation.

---

## 🏛️ Apache Commons

Apache Commons is a suite of reusable Java components maintained by the Apache Software Foundation. It predates many modern Java features and fills in what the early JDK was missing. Even today, with Java having improved substantially, Apache Commons libraries remain valuable for their battle-tested implementations and breadth of utility.

The suite is divided into independent modules — you add only what you need. The four most important for daily Java work are Commons Lang, Commons IO, Commons Collections, and Commons Math.

---

### 21.1.1 Commons Lang (`commons-lang3`)

Commons Lang enriches `java.lang` with utilities for strings, numbers, dates, reflection, and concurrent helpers. The most-used class is `StringUtils`, which provides null-safe string operations — a feature so fundamental it's hard to believe the JDK didn't include it.

```java
import org.apache.commons.lang3.*;
import org.apache.commons.lang3.builder.*;

// ── StringUtils ─────────────────────────────────────────────────────────────
// Every method in StringUtils handles null gracefully — no NullPointerException

StringUtils.isEmpty(null);          // true  (null is considered empty)
StringUtils.isEmpty("");            // true
StringUtils.isEmpty("  ");         // false (non-empty: has whitespace chars)

StringUtils.isBlank(null);          // true  (null, empty, AND whitespace are blank)
StringUtils.isBlank("  ");         // true  ← this is the key difference from isEmpty
StringUtils.isBlank("hi");         // false

// Null-safe comparison — no NPE if either argument is null
StringUtils.equals(null, null);     // true
StringUtils.equals(null, "hello");  // false (no NPE!)
StringUtils.equalsIgnoreCase("Java", "JAVA"); // true

// Padding, truncating, centering — great for log output and reports
StringUtils.leftPad("42", 6, '0');   // "000042"  — zero-padding numbers
StringUtils.rightPad("Hello", 10);   // "Hello     " — right-pad with spaces
StringUtils.center("Title", 20, '-');// "-------Title--------"
StringUtils.truncate("A very long string", 10); // "A very lon"

// Reversing, wrapping, abbreviating
StringUtils.reverse("hello");                    // "olleh"
StringUtils.abbreviate("Hello World", 8);        // "Hello..."  — smart truncation with ellipsis
StringUtils.wrap("hello", "'");                  // "'hello'"

// Checking content type
StringUtils.isNumeric("12345");    // true
StringUtils.isNumeric("123.45");   // false  (decimal point fails isNumeric)
StringUtils.isAlpha("Hello");      // true
StringUtils.isAlphanumeric("abc123"); // true

// Joining and splitting
StringUtils.join(new String[]{"a", "b", "c"}, ", "); // "a, b, c"
StringUtils.join(List.of(1, 2, 3), " | ");           // "1 | 2 | 3"

// ── NumberUtils ─────────────────────────────────────────────────────────────
// Safe parsing — returns a default instead of throwing on invalid input
int parsed   = NumberUtils.toInt("42");        // 42
int safe     = NumberUtils.toInt("oops", -1); // -1  (default on parse failure)
double d     = NumberUtils.toDouble("3.14");   // 3.14
long max     = NumberUtils.max(10L, 25L, 7L);  // 25  — varargs max/min

// ── ArrayUtils ──────────────────────────────────────────────────────────────
// Null-safe operations on primitive and object arrays
int[] nums  = {5, 3, 8, 1, 9, 2};
int[]  copy = ArrayUtils.clone(nums);
ArrayUtils.reverse(copy);                    // Reverses in place: [2, 9, 1, 8, 3, 5]
boolean has = ArrayUtils.contains(nums, 8);  // true
int idx     = ArrayUtils.indexOf(nums, 9);   // 4
int[] added = ArrayUtils.add(nums, 10);      // Returns new array with 10 appended
int[] removed = ArrayUtils.removeElement(nums, 3); // New array with first 3 removed

// ── ObjectUtils ─────────────────────────────────────────────────────────────
// Null-safe utilities for any object
String result = ObjectUtils.defaultIfNull(maybeNull, "default"); // "default" if maybeNull is null
String first  = ObjectUtils.firstNonNull(null, null, "found", "other"); // "found"
boolean empty = ObjectUtils.isEmpty(new ArrayList<>()); // true — works on collections, arrays, strings, maps

// ── Builder utilities — ToStringBuilder, EqualsBuilder, HashCodeBuilder ─────
// Useful when you CAN'T use Lombok (e.g., in a legacy codebase or on an entity)
public class Order {
    private long id;
    private String customerId;
    private BigDecimal total;

    @Override
    public String toString() {
        return new ToStringBuilder(this, ToStringStyle.JSON_STYLE)
            .append("id", id)
            .append("customerId", customerId)
            .append("total", total)
            .toString();
        // Output: {"id":42,"customerId":"cust-001","total":99.99}
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order other)) return false;
        return new EqualsBuilder()
            .append(id, other.id)
            .append(customerId, other.customerId)
            .isEquals();
    }

    @Override
    public int hashCode() {
        return new HashCodeBuilder(17, 37)
            .append(id)
            .append(customerId)
            .toHashCode();
    }
}

// ── Validate — precondition checking ────────────────────────────────────────
import org.apache.commons.lang3.Validate;

public void processOrder(Order order, int quantity) {
    Validate.notNull(order, "order must not be null");
    Validate.isTrue(quantity > 0, "quantity must be positive, got %d", quantity);
    Validate.notBlank(order.getCustomerId(), "customer ID must not be blank");
    Validate.inclusiveBetween(1, 1000, quantity,
        "quantity must be between 1 and 1000, got %d", quantity);
    // Continues only if all validations pass — otherwise throws IllegalArgumentException
}
```

---

### 21.1.2 Commons IO (`commons-io`)

Commons IO provides utilities for file and stream operations that the JDK's `java.io` and `java.nio` APIs handle verbosely. Its most valuable classes are `IOUtils`, `FileUtils`, and the `FileFilter` hierarchy.

```java
import org.apache.commons.io.*;
import org.apache.commons.io.filefilter.*;
import java.io.*;
import java.nio.charset.StandardCharsets;

// ── IOUtils — stream and reader/writer operations ────────────────────────────

// Read an entire InputStream as a String — one line instead of try/buffer/loop
InputStream input = new FileInputStream("data.txt");
String content = IOUtils.toString(input, StandardCharsets.UTF_8);

// Copy one stream to another — handles buffering internally
InputStream  from = new FileInputStream("source.bin");
OutputStream to   = new FileOutputStream("target.bin");
long bytesCopied  = IOUtils.copy(from, to);  // Returns number of bytes copied

// Close quietly — no checked exception to handle, perfect for finally blocks
// (Before try-with-resources became idiomatic, this was essential)
IOUtils.closeQuietly(input);
IOUtils.closeQuietly(from, to);  // Varargs: close multiple at once

// Read all lines from a stream into a List
List<String> lines = IOUtils.readLines(input, StandardCharsets.UTF_8);

// Convert content to a byte array
byte[] bytes = IOUtils.toByteArray(inputStream);

// ── FileUtils — file system operations ──────────────────────────────────────

File source = new File("original.txt");
File dest   = new File("backup.txt");

// Copy file, copy directory — recursive copy without writing a walk
FileUtils.copyFile(source, dest);
FileUtils.copyDirectory(new File("src/main"), new File("backup/main"));

// Move file/directory
FileUtils.moveFile(source, new File("archive/original.txt"));
FileUtils.moveDirectory(new File("old-dir"), new File("new-dir"));

// Delete recursively — safer than File.delete() which only works on empty dirs
FileUtils.deleteDirectory(new File("temp-dir"));  // Deletes dir and all contents
FileUtils.cleanDirectory(new File("cache-dir"));  // Empties dir but keeps the dir itself

// Read/write files with charset awareness
String text = FileUtils.readFileToString(new File("notes.txt"), StandardCharsets.UTF_8);
FileUtils.writeStringToFile(new File("output.txt"), "Hello!\n", StandardCharsets.UTF_8);
FileUtils.writeStringToFile(new File("output.txt"), "More!\n",  StandardCharsets.UTF_8, true); // append=true

// Read/write byte arrays
byte[] data = FileUtils.readFileToByteArray(new File("image.png"));
FileUtils.writeByteArrayToFile(new File("copy.png"), data);

// List files with a filter — find all .java files recursively
Collection<File> javaFiles = FileUtils.listFiles(
    new File("src"),
    new SuffixFileFilter(".java"),   // File filter: only .java files
    TrueFileFilter.INSTANCE          // Directory filter: recurse into all subdirectories
);

// File size and comparison
long sizeBytes = FileUtils.sizeOf(new File("bigfile.zip"));
long sizeDirMB = FileUtils.sizeOfDirectory(new File("project")) / (1024 * 1024);
boolean same   = FileUtils.contentEquals(source, dest); // true if file contents are identical

// Touch a file (create if not exists, update modification time if it does)
FileUtils.touch(new File("timestamp.lock"));
```

---

### 21.1.3 Commons Collections (`commons-collections4`)

Commons Collections extends `java.util` with data structures the standard library lacks — multi-value maps, bidirectional maps, bags, and rich transformations.

```java
import org.apache.commons.collections4.*;
import org.apache.commons.collections4.bag.*;
import org.apache.commons.collections4.bidimap.*;

// ── MultiValuedMap — one key, many values ───────────────────────────────────
// Standard Map<K, List<V>> is awkward; MultiValuedMap wraps it cleanly
MultiValuedMap<String, String> tagMap = new ArrayListValuedHashMap<>();
tagMap.put("java",   "language");
tagMap.put("java",   "platform");
tagMap.put("java",   "coffee");       // Three values for "java"
tagMap.put("python", "language");

Collection<String> javaTags = tagMap.get("java");  // ["language", "platform", "coffee"]
System.out.println(tagMap.containsMapping("java", "coffee")); // true
System.out.println(tagMap.size());  // 4 (total mapping count, not key count)
System.out.println(tagMap.keySet()); // [java, python]

// ── BidiMap — bidirectional map: look up by key OR by value ─────────────────
BidiMap<String, Integer> dayNumbers = new DualHashBidiMap<>();
dayNumbers.put("Monday",    1);
dayNumbers.put("Tuesday",   2);
dayNumbers.put("Wednesday", 3);

String  day  = dayNumbers.getKey(2);     // "Tuesday"  — reverse lookup by value
Integer num  = dayNumbers.get("Monday"); // 1           — forward lookup by key

// Invert the entire map (keys become values, values become keys)
BidiMap<Integer, String> inverted = dayNumbers.inverseBidiMap();
System.out.println(inverted.get(3));  // "Wednesday"

// ── Bag — counts occurrences, like a multiset ────────────────────────────────
Bag<String> wordBag = new HashBag<>();
wordBag.add("the");  wordBag.add("the");  wordBag.add("the");
wordBag.add("cat");  wordBag.add("cat");
wordBag.add("sat");

System.out.println(wordBag.getCount("the")); // 3
System.out.println(wordBag.getCount("cat")); // 2
System.out.println(wordBag.uniqueSet());     // [the, cat, sat]

// Sorted bag — iterates in sorted order of elements
SortedBag<String> sorted = new TreeBag<>(wordBag);

// ── CollectionUtils — safe, null-tolerant operations on collections ──────────
List<String> listA = List.of("a", "b", "c", "d");
List<String> listB = List.of("c", "d", "e", "f");

// Set operations (return new collections, do not modify inputs)
Collection<String> union        = CollectionUtils.union(listA, listB);        // [a,b,c,d,e,f]
Collection<String> intersection = CollectionUtils.intersection(listA, listB); // [c,d]
Collection<String> subtract     = CollectionUtils.subtract(listA, listB);     // [a,b]

// Null-safe checks
boolean notEmpty = CollectionUtils.isNotEmpty(myList);   // safe even if myList is null
boolean empty    = CollectionUtils.isEmpty(null);         // true — no NPE

// Filtering and transforming
CollectionUtils.filter(names, s -> s.startsWith("A"));    // Removes non-matching in place
Collection<Integer> lengths = CollectionUtils.collect(names, String::length); // Maps to new collection
```

---

### 21.1.4 Commons Math (`commons-math3`)

Commons Math provides mathematical tools that go beyond `java.lang.Math` — statistics, distributions, linear algebra, root-finding, and more.

```java
import org.apache.commons.math3.stat.descriptive.*;
import org.apache.commons.math3.distribution.*;
import org.apache.commons.math3.stat.inference.*;

// ── DescriptiveStatistics — window of statistical history ───────────────────
DescriptiveStatistics stats = new DescriptiveStatistics();
// Optionally, specify a sliding window size: new DescriptiveStatistics(100)
// means it only retains the last 100 values

double[] measurements = {4.2, 3.8, 5.1, 4.9, 4.0, 5.5, 3.6, 4.7};
for (double m : measurements) stats.addValue(m);

System.out.println("Mean:   " + stats.getMean());        // 4.475
System.out.println("StdDev: " + stats.getStandardDeviation()); // ~0.63
System.out.println("Min:    " + stats.getMin());          // 3.6
System.out.println("Max:    " + stats.getMax());          // 5.5
System.out.println("P95:    " + stats.getPercentile(95)); // 95th percentile

// ── SummaryStatistics — running stats without storing all values ─────────────
SummaryStatistics running = new SummaryStatistics();
// Efficiently computes statistics as values stream in — constant memory usage
for (double v : largeDataset) running.addValue(v);
System.out.println("N:    " + running.getN());
System.out.println("Mean: " + running.getMean());
System.out.println("Var:  " + running.getVariance());

// ── Statistical distributions ────────────────────────────────────────────────
// Useful for simulation, testing, and understanding probability
NormalDistribution normal = new NormalDistribution(0, 1); // mean=0, stddev=1
double prob = normal.cumulativeProbability(1.96);  // ~0.975 (97.5th percentile)
double z    = normal.inverseCumulativeProbability(0.975); // ~1.96

PoissonDistribution poisson = new PoissonDistribution(3.5); // mean arrival rate = 3.5
double pExactly5 = poisson.probability(5);     // P(X = 5)
double pAtMost5  = poisson.cumulativeProbability(5); // P(X <= 5)
```

---

## 🛡️ Google Guava

Guava is Google's core Java library, open-sourced in 2007 and maintained ever since. It predates Java 8 by many years and introduced concepts (immutable collections, `Optional`, `Function`, `Predicate`) that were later adopted into the JDK. Today, with Java having absorbed many of these ideas, Guava's most valuable parts are its data structures that still don't exist in the standard library and its high-performance utilities.

---

### 21.1.5 Guava — Immutable Collections

Guava's immutable collections are more rigorous than Java's `Collections.unmodifiableList()` (which is only a view — changes to the underlying collection still show through) and have better performance characteristics because they can be internally optimised knowing they'll never change.

```java
import com.google.common.collect.*;

// ImmutableList — an ordered, immutable list
ImmutableList<String> countries = ImmutableList.of("France", "Germany", "Japan");
ImmutableList<String> built = ImmutableList.<String>builder()
    .add("France")
    .add("Germany")
    .addAll(otherCountries)   // Append all elements from another iterable
    .build();

// ImmutableSet — no duplicates, preserves insertion order (unlike HashSet)
ImmutableSet<String> tags = ImmutableSet.of("java", "spring", "microservices");

// ImmutableMap — key-value pairs, preserves insertion order
ImmutableMap<String, Integer> httpCodes = ImmutableMap.of(
    "OK",           200,
    "NOT_FOUND",    404,
    "SERVER_ERROR", 500
);
// For more than 5 entries, use the builder:
ImmutableMap<String, Integer> codes = ImmutableMap.<String, Integer>builder()
    .put("OK",           200)
    .put("CREATED",      201)
    .put("BAD_REQUEST",  400)
    .put("UNAUTHORIZED", 401)
    .put("NOT_FOUND",    404)
    .buildOrThrow(); // Throws if any key is duplicated (unlike put which silently overwrites)
```

### 21.1.6 Guava — Multimap and Table

```java
// Multimap — one key maps to multiple values
// Cleaner than Map<K, List<V>> and correctly handles size(), isEmpty(), etc.
ListMultimap<String, String> booksByGenre = ArrayListMultimap.create();
booksByGenre.put("fiction",   "Dune");
booksByGenre.put("fiction",   "Neuromancer");
booksByGenre.put("non-fiction", "Clean Code");
booksByGenre.put("non-fiction", "SICP");

List<String> fiction = booksByGenre.get("fiction"); // ["Dune", "Neuromancer"]
System.out.println(booksByGenre.size());             // 4 — total values, not key count
System.out.println(booksByGenre.keySet());           // [fiction, non-fiction]
booksByGenre.remove("fiction", "Dune");              // Remove one specific mapping

// SetMultimap — values are sets (no duplicates per key)
SetMultimap<String, String> userRoles = HashMultimap.create();
userRoles.put("alice", "ADMIN");
userRoles.put("alice", "USER");
userRoles.put("alice", "ADMIN"); // Duplicate — silently ignored
System.out.println(userRoles.get("alice")); // [ADMIN, USER] (only 2, not 3)

// Multimaps.index — a powerful utility for grouping a collection by a key function
// This is like Stream's groupingBy but produces an ImmutableListMultimap
List<String> words = List.of("cat", "can", "car", "bat", "bar", "bee");
ImmutableListMultimap<Character, String> byFirstLetter =
    Multimaps.index(words, word -> word.charAt(0));
// Result: {c=[cat, can, car], b=[bat, bar, bee]}

// ── Table — two-key map: think spreadsheet with row + column keys ─────────────
// Replaces the ugly Map<RowKey, Map<ColKey, Value>> pattern
Table<String, String, Integer> scores = HashBasedTable.create();
scores.put("Alice", "Math",    95);
scores.put("Alice", "Science", 88);
scores.put("Bob",   "Math",    72);
scores.put("Bob",   "Science", 91);

System.out.println(scores.get("Alice", "Math"));    // 95
System.out.println(scores.row("Alice"));             // {Math=95, Science=88}
System.out.println(scores.column("Math"));           // {Alice=95, Bob=72}
System.out.println(scores.rowKeySet());              // [Alice, Bob]
System.out.println(scores.columnKeySet());           // [Math, Science]
System.out.println(scores.cellSet());                // All three-tuples
```

### 21.1.7 Guava — Cache (`LoadingCache`)

Guava's cache is a local in-memory cache with eviction, expiry, and statistics. This was the gold standard before Caffeine arrived (Caffeine is now preferred for new projects, but Guava Cache is still widely found in existing codebases).

```java
import com.google.common.cache.*;

// LoadingCache — automatically loads missing values when first requested
LoadingCache<Long, User> userCache = CacheBuilder.newBuilder()
    .maximumSize(1_000)                            // Evict when > 1000 entries (LRU-like)
    .expireAfterWrite(10, TimeUnit.MINUTES)        // Expire 10 min after write
    .expireAfterAccess(5, TimeUnit.MINUTES)        // Also expire if not accessed for 5 min
    .recordStats()                                 // Enable hit/miss statistics
    .build(new CacheLoader<Long, User>() {
        @Override
        public User load(Long userId) {            // Called automatically on cache miss
            return userRepository.findById(userId)
                .orElseThrow(() -> new UserNotFoundException(userId));
        }
    });

// Usage: get() transparently loads if absent
User user = userCache.get(42L);      // Hits DB on first call, returns cached on subsequent

// Batch load (calls loadAll() which you can override for batch-efficient loading)
ImmutableMap<Long, User> users = userCache.getAll(List.of(1L, 2L, 3L));

// Explicitly invalidate an entry (e.g., after an update)
userCache.invalidate(42L);
userCache.invalidateAll();  // Clear entire cache

// View statistics
CacheStats stats = userCache.stats();
System.out.println("Hit rate:      " + stats.hitRate());           // 0.0 to 1.0
System.out.println("Miss count:    " + stats.missCount());
System.out.println("Load time avg: " + stats.averageLoadPenalty()); // nanoseconds
```

### 21.1.8 Guava — Preconditions, Strings, EventBus

```java
import com.google.common.base.*;
import com.google.common.eventbus.*;

// ── Preconditions — argument validation with descriptive messages ─────────────
// Very similar to Objects.requireNonNull but with a richer set of checks
public void transfer(Account from, Account to, BigDecimal amount) {
    Preconditions.checkNotNull(from,   "source account must not be null");
    Preconditions.checkNotNull(to,     "destination account must not be null");
    Preconditions.checkNotNull(amount, "amount must not be null");
    Preconditions.checkArgument(amount.compareTo(BigDecimal.ZERO) > 0,
        "amount must be positive, got %s", amount);
    Preconditions.checkState(!from.isFrozen(),
        "source account %s is frozen", from.getId());
    // Throws: checkNotNull → NPE, checkArgument → IllegalArgumentException,
    //         checkState   → IllegalStateException
}

// ── Strings — null-safe string helpers ───────────────────────────────────────
Strings.isNullOrEmpty(null);       // true
Strings.isNullOrEmpty("");         // true
Strings.isNullOrEmpty("hello");    // false

String safe = Strings.nullToEmpty(null);  // ""   — converts null to empty string
String orig = Strings.emptyToNull("");    // null — converts empty to null (useful for normalisation)

Strings.padStart("42",  6, '0');   // "000042"  — left-pad
Strings.padEnd("hello", 10, '-');  // "hello-----" — right-pad
Strings.repeat("ab", 3);           // "ababab"

// ── Joiner and Splitter — richer than String.join / String.split ──────────────
// Joiner handles nulls gracefully
String joined = Joiner.on(", ")
    .skipNulls()               // Omit null elements
    .join("alpha", null, "gamma");  // "alpha, gamma"

String withDefault = Joiner.on(", ")
    .useForNull("N/A")         // Replace null with a placeholder
    .join("alpha", null, "gamma");  // "alpha, N/A, gamma"

// Splitter is more predictable than String.split() (no trailing empty strings weirdness)
List<String> parts = Splitter.on(',')
    .trimResults()             // Trim whitespace from each result
    .omitEmptyStrings()        // Skip empty tokens (e.g. from "a,,b")
    .splitToList("alpha, beta, , gamma, ");  // ["alpha", "beta", "gamma"]

// Split into a Map from "key=value" strings
Map<String, String> map = Splitter.on('&')
    .withKeyValueSeparator('=')
    .split("name=Alice&role=admin&active=true");
// {name=Alice, role=admin, active=true}

// ── EventBus — synchronous publish-subscribe within a single JVM ─────────────
// Useful for decoupling components without a full message broker
EventBus eventBus = new EventBus("order-events");

// Subscriber: any object with @Subscribe methods
class OrderAuditLogger {
    @Subscribe
    public void onOrderCreated(OrderCreatedEvent event) {
        System.out.println("AUDIT: Order " + event.orderId() + " created");
    }
}

class InventoryUpdater {
    @Subscribe
    public void onOrderCreated(OrderCreatedEvent event) {
        inventoryService.reserveItems(event.orderId());
    }
}

// Register subscribers
eventBus.register(new OrderAuditLogger());
eventBus.register(new InventoryUpdater());

// Publish an event — all subscribers receive it synchronously in the same thread
eventBus.post(new OrderCreatedEvent(orderId));
// Both OrderAuditLogger and InventoryUpdater will be called
```

---

## ✂️ Lombok

Lombok is a compile-time annotation processor that generates boilerplate Java code — constructors, getters/setters, `equals`/`hashCode`/`toString`, builders, and logging fields — directly into the `.class` file without you having to write it. The source file stays clean; the generated code never appears in your editor.

Lombok requires the annotation processor to be configured in your IDE (IntelliJ has a Lombok plugin) and build tool. With Maven, add it as a `provided` scope dependency — it's only needed at compile time, not at runtime.

### 21.1.9 Lombok — Core Annotations

```java
import lombok.*;
import lombok.extern.slf4j.Slf4j;

// ── @Data — the all-in-one annotation for mutable POJOs ──────────────────────
// Generates: getters for all fields, setters for non-final fields,
//            equals(), hashCode(), toString(), and an all-args constructor
@Data
public class UserDto {
    private Long id;
    private String username;
    private String email;
    private boolean active;
}
// Equivalent to ~60 lines of hand-written Java

// ── @Value — immutable version of @Data (all fields are private final) ────────
// Generates: all-args constructor (required since fields are final),
//            getters (no setters — the class is immutable), equals, hashCode, toString
@Value
public class Money {
    BigDecimal amount;
    String currency;
    // No setters generated; fields are final; class is implicitly final
}

// ── @Builder — implements the Builder design pattern automatically ─────────────
// Generates: a static inner Builder class with a fluent API
@Builder
@Getter
public class Order {
    private Long id;
    private String customerId;
    private List<OrderItem> items;
    @Builder.Default  // Provides a default value for the builder
    private OrderStatus status = OrderStatus.PENDING;
    private LocalDateTime createdAt;
}

// Usage — clean, readable, handles optional fields gracefully
Order order = Order.builder()
    .customerId("cust-001")
    .items(List.of(item1, item2))
    .createdAt(LocalDateTime.now())
    .build();   // status defaults to PENDING

// toBuilder() — create a modified copy without touching the original
// Requires @Builder(toBuilder = true)
Order updated = order.toBuilder()
    .status(OrderStatus.CONFIRMED)
    .build();

// ── @NoArgsConstructor, @AllArgsConstructor, @RequiredArgsConstructor ─────────
@NoArgsConstructor    // Generates public Order() {}
@AllArgsConstructor   // Generates public Order(Long id, String customerId, ...) {}
// @RequiredArgsConstructor: generates a constructor only for final and @NonNull fields
@RequiredArgsConstructor
public class PaymentService {
    private final PaymentGateway gateway;    // Injected via constructor (Spring DI friendly)
    private final OrderRepository repository;
    private int retryCount = 3;              // Not final — excluded from required constructor
}

// ── @Slf4j — injects a Logger field named 'log' ──────────────────────────────
@Slf4j                        // Equivalent to: private static final Logger log = LoggerFactory.getLogger(OrderService.class);
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository repository;

    public Order findById(Long id) {
        log.debug("Looking up order {}", id);  // 'log' is injected by Lombok
        return repository.findById(id)
            .orElseThrow(() -> {
                log.warn("Order {} not found", id);
                return new OrderNotFoundException(id);
            });
    }
}

// ── @NonNull — generates a null check at the start of a method/constructor ────
public class CustomerService {
    public Customer create(@NonNull String name, @NonNull String email) {
        // Lombok generates: if (name == null) throw new NullPointerException("name is marked non-null but is null");
        return new Customer(name, email);
    }
}

// ── @SneakyThrows — rethrows checked exceptions as unchecked ─────────────────
// Useful when working with APIs that force checked exceptions in lambda contexts
public class FileProcessor {
    @SneakyThrows(IOException.class)  // IOException is rethrown as-is but unchecked
    public String readFile(Path path) {
        return Files.readString(path);  // Normally requires try/catch for IOException
    }
}

// ── @Cleanup — automatic resource management (before try-with-resources) ──────
public void processStream() throws IOException {
    @Cleanup InputStream in = new FileInputStream("data.bin");
    @Cleanup OutputStream out = new FileOutputStream("output.bin");
    // in.close() and out.close() are called at end of scope — like try-with-resources
}
```

> **Best Practice:** Use `@Value` or records for immutable data carriers. Use `@Builder` on classes that have more than 3–4 constructor parameters — it makes call sites readable and avoids the telescoping constructor problem. Prefer `@RequiredArgsConstructor` with `final` fields for Spring services — it works perfectly with Spring's constructor injection and communicates clearly which dependencies are mandatory. Avoid `@Data` on JPA entities — its `equals`/`hashCode` implementation can cause subtle problems with Hibernate's entity identity semantics; implement `equals`/`hashCode` manually for entities.

---

## 🗺️ MapStruct

MapStruct is a compile-time code generator for object mapping — converting between different Java bean types. The classic use case is mapping JPA entities to DTOs and back. MapStruct generates fast, readable, plain-Java mapping code at compile time (unlike reflection-based mappers), making it both type-safe and performant.

### 21.1.10 MapStruct — Core Concepts

```java
// Add to pom.xml:
// <dependency> groupId: org.mapstruct, artifactId: mapstruct, version: 1.5.5.Final </dependency>
// <annotationProcessorPaths> must include mapstruct-processor AND lombok-processor

// ── Source and target beans ───────────────────────────────────────────────────
// Entity (domain model — what the database returns)
@Entity
public class Customer {
    @Id private Long id;
    private String firstName;
    private String lastName;
    private String emailAddress;
    @OneToMany private List<Order> orders;
}

// DTO (data transfer object — what the REST API returns/accepts)
public record CustomerDto(
    Long id,
    String fullName,     // firstName + " " + lastName — needs combining
    String email,        // emailAddress → email — needs renaming
    int orderCount       // orders.size() — needs derivation
) {}

// ── Defining the Mapper ───────────────────────────────────────────────────────
// MapStruct reads this interface and generates a complete implementation at compile time
@Mapper(componentModel = "spring") // "spring" means the generated impl is a @Component
public interface CustomerMapper {

    // Simple mapping: same-named fields are mapped automatically
    // @Mapping handles everything else
    @Mapping(source = "emailAddress", target = "email")      // Rename field
    @Mapping(expression = "java(customer.getFirstName() + \" \" + customer.getLastName())",
             target = "fullName")                             // Combine fields
    @Mapping(expression = "java(customer.getOrders().size())",
             target = "orderCount")                          // Derive from collection
    CustomerDto toDto(Customer customer);

    // Reverse mapping: DTO back to entity
    // 'ignore = true' for fields that shouldn't be mapped from the DTO
    @Mapping(target = "orders", ignore = true)               // Don't map orderCount back
    @Mapping(source = "email", target = "emailAddress")      // Reverse the rename
    @Mapping(target = "firstName", ignore = true)            // Will be derived — see @AfterMapping
    @Mapping(target = "lastName",  ignore = true)
    Customer toEntity(CustomerDto dto);

    // @AfterMapping — called after the generated mapping to handle complex derivation
    @AfterMapping
    default void splitFullName(CustomerDto dto, @MappingTarget Customer customer) {
        if (dto.fullName() != null) {
            String[] parts = dto.fullName().split(" ", 2);
            customer.setFirstName(parts[0]);
            customer.setLastName(parts.length > 1 ? parts[1] : "");
        }
    }

    // Bulk mapping — maps a list of entities to a list of DTOs
    // MapStruct generates this automatically using the single-item mapping above
    List<CustomerDto> toDtoList(List<Customer> customers);
}

// ── Usage in a Spring service ─────────────────────────────────────────────────
@Service
@RequiredArgsConstructor
public class CustomerService {
    private final CustomerRepository repository;
    private final CustomerMapper mapper;     // Spring injects the generated implementation

    public CustomerDto getCustomer(Long id) {
        return repository.findById(id)
            .map(mapper::toDto)
            .orElseThrow(() -> new CustomerNotFoundException(id));
    }

    public List<CustomerDto> getAllCustomers() {
        return mapper.toDtoList(repository.findAll());
    }

    public Customer updateCustomer(Long id, CustomerDto dto) {
        Customer existing = repository.findById(id)
            .orElseThrow(() -> new CustomerNotFoundException(id));
        mapper.toEntity(dto);  // Could also use updateEntityFromDto pattern
        return repository.save(existing);
    }
}
```

MapStruct also supports: nested object mapping, using existing mappers inside another mapper (`@Mapper(uses = AddressMapper.class)`), custom qualifier annotations for when there are multiple ways to map the same type, decorators for post-processing, and builder support for immutable targets (records, Lombok `@Value`).

> **MapStruct vs. ModelMapper:** MapStruct generates explicit Java code at compile time — you can read the generated code in `target/generated-sources/`. This means type errors are caught at compile time, the generated code is debuggable, and there is zero runtime reflection overhead. ModelMapper uses runtime reflection, which is more flexible but slower and moves errors to runtime. For most production applications, MapStruct is the better choice. Use ModelMapper when you need maximum flexibility with minimal configuration (e.g., quick prototypes, or mapping types you don't control).

---

## 🔄 ModelMapper

ModelMapper uses conventions and reflection to automatically map object graphs without requiring you to declare each field mapping explicitly. It works well for simple cases and prototyping but requires more care for complex or ambiguous mappings.

```java
import org.modelmapper.*;

// Configuration — created once and reused
ModelMapper mapper = new ModelMapper();
mapper.getConfiguration()
    .setMatchingStrategy(MatchingStrategies.STRICT)  // Require exact property name matches
    .setSkipNullEnabled(true);                        // Don't overwrite non-null targets with null sources

// Simple mapping — same-named properties are matched automatically
Customer customer = repository.findById(1L).orElseThrow();
CustomerDto dto = mapper.map(customer, CustomerDto.class);

// Custom TypeMap for controlling specific conversions
mapper.createTypeMap(Customer.class, CustomerDto.class)
    .addMapping(Customer::getEmailAddress, CustomerDto::setEmail)   // Rename
    .addMappings(m -> m.skip(CustomerDto::setOrderCount));          // Skip a field

// Mapping using a Converter for complex transformations
Converter<List<Order>, Integer> countConverter =
    ctx -> ctx.getSource() == null ? 0 : ctx.getSource().size();

mapper.createTypeMap(Customer.class, CustomerDto.class)
    .addMappings(m -> m.using(countConverter)
        .map(Customer::getOrders, CustomerDto::setOrderCount));

// Validate the mapper configuration at startup — throws if any property is ambiguous
mapper.validate();  // Best practice: call this in a @PostConstruct or test

// Map a list — ModelMapper handles collections automatically
List<CustomerDto> dtos = customers.stream()
    .map(c -> mapper.map(c, CustomerDto.class))
    .collect(Collectors.toList());
```

---

## 💡 Phase 21.1 — Key Takeaways

Apache Commons fills the gaps in `java.lang`, `java.io`, and `java.util` that accumulated over decades of conservative JDK evolution — use it for null-safe strings, file operations, and multi-value collections. Guava provides richer immutable collections, bidirectional maps, the powerful `Multimap` and `Table`, and a lightweight in-process event bus. Lombok is one of the highest-leverage tools in the Java ecosystem — a single annotation eliminates dozens of lines of boilerplate, making your domain model clean and expressive; use it thoughtfully and avoid it on JPA entities. MapStruct is the correct choice for type-safe, fast object mapping in production systems, while ModelMapper suits rapid prototyping. Together, these libraries represent accumulated Java community wisdom about what was missing from the JDK and how to fill those gaps cleanly.
