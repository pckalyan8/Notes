# 4.1 Java Collections Framework

## What & Why

Before the Collections Framework arrived in Java 1.2, developers managed groups of objects using plain arrays or ad-hoc classes like `Vector` and `Hashtable`. Arrays are fixed-size and type-unsafe (without generics); `Vector` and `Hashtable` were over-synchronized for every use case. The Java Collections Framework (JCF) solved all of this in one sweep by providing a **unified architecture** of interfaces, implementations, and algorithms that cover every common data-structure need.

The key design insight is the separation of **interface** (what a collection *promises*) from **implementation** (how it *delivers*). You program to `List`, `Set`, or `Map`; you choose `ArrayList`, `TreeSet`, or `HashMap` based on performance needs.

---

## The Interface Hierarchy

Everything starts with `Iterable<E>`, which promises only one thing: you can iterate over the elements using a for-each loop. From there the tree grows:

```
Iterable<E>
  └── Collection<E>
        ├── List<E>         — ordered, index-accessible, duplicates allowed
        ├── Set<E>          — no duplicates
        │     └── SortedSet<E>
        │           └── NavigableSet<E>
        ├── Queue<E>        — FIFO ordering
        │     └── Deque<E>  — double-ended queue (both Stack and Queue)
        
Map<K,V>                    — key-value pairs (NOT a Collection)
  └── SortedMap<K,V>
        └── NavigableMap<K,V>
```

`Map` is separate because it doesn't deal with single elements — it deals with key-value *pairs* — so inheriting from `Collection` would be philosophically wrong.

---

## 1. List Implementations

A `List` guarantees **insertion order** and allows **duplicate elements**. You access elements by a zero-based integer index.

### ArrayList

`ArrayList` is backed by a plain Object array. When the array fills up, Java allocates a new array 1.5× larger and copies everything over. This means:

- Random access by index (`get(5)`) is O(1) — pure array lookup.
- Adding to the **end** is amortized O(1) — usually fast, occasionally an O(n) resize.
- Inserting or deleting in the **middle** is O(n) — every element after the insertion point must shift.

```java
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

public class ArrayListDemo {
    public static void main(String[] args) {
        // Type parameter makes this a List of Strings only
        List<String> cities = new ArrayList<>();

        cities.add("Tokyo");
        cities.add("Paris");
        cities.add("London");
        cities.add("Paris");          // duplicates are allowed in a List

        System.out.println(cities);                // [Tokyo, Paris, London, Paris]
        System.out.println(cities.get(1));         // Paris — O(1) index access
        System.out.println(cities.size());         // 4
        System.out.println(cities.contains("London")); // true

        cities.remove("Paris");                    // removes FIRST occurrence
        System.out.println(cities);                // [Tokyo, London, Paris]

        Collections.sort(cities);                  // natural (alphabetical) order
        System.out.println(cities);                // [London, Paris, Tokyo]

        // Pre-size when you know the expected capacity to avoid resizes
        List<Integer> numbers = new ArrayList<>(1000);
    }
}
```

**When to use ArrayList:** This should be your default `List`. Most workloads involve far more reads than insertions, and array-backed O(1) random access wins nearly every time.

### LinkedList

`LinkedList` is a doubly-linked list where every element (node) holds a reference to the previous and next node. No array, no shifting:

- Inserting or deleting at the **head or tail** is O(1).
- Inserting or deleting **anywhere else** is O(1) if you already have the iterator/position, but finding that position takes O(n).
- Random access (`get(5)`) is O(n) — you must walk the chain.
- Each node carries two extra object references (prev/next) — memory overhead is real.

```java
import java.util.LinkedList;
import java.util.Deque;

public class LinkedListDemo {
    public static void main(String[] args) {
        // LinkedList also implements Deque, so it can act as a stack or queue
        Deque<String> deque = new LinkedList<>();

        deque.addFirst("Middle");
        deque.addFirst("First");    // push to front
        deque.addLast("Last");      // push to back

        System.out.println(deque);              // [First, Middle, Last]
        System.out.println(deque.peekFirst());  // First — look without removing
        System.out.println(deque.pollLast());   // Last  — remove from back
        System.out.println(deque);              // [First, Middle]
    }
}
```

**When to use LinkedList:** Almost never as a plain `List` — `ArrayList` beats it there. Use it as a `Deque` (double-ended queue) when you need frequent head/tail operations and don't need random access.

### Immutable Lists (Java 9+)

```java
// List.of() returns a truly immutable list — no add/remove allowed
List<String> immutable = List.of("a", "b", "c");

// List.copyOf() creates an immutable copy of an existing list
List<String> copy = List.copyOf(immutable);

// Attempting to modify throws UnsupportedOperationException
// immutable.add("d");  // ← throws!
```

---

## 2. Set Implementations

A `Set` is a collection that contains **no duplicate elements**. The definition of "duplicate" comes from `equals()` and `hashCode()` on the stored objects.

### HashSet

Backed by a `HashMap` internally (values are stored as keys; a dummy `PRESENT` object fills the values). Element lookup, insertion, and removal are all O(1) on average.

**Order is unpredictable** — do not rely on iteration order.

```java
import java.util.HashSet;
import java.util.Set;

public class HashSetDemo {
    public static void main(String[] args) {
        Set<String> tags = new HashSet<>();
        tags.add("java");
        tags.add("backend");
        tags.add("java");     // duplicate — silently ignored, add() returns false

        System.out.println(tags.size());         // 2
        System.out.println(tags.contains("java")); // true

        // Iteration order is NOT guaranteed to match insertion order
        for (String tag : tags) {
            System.out.println(tag);
        }
    }
}
```

### LinkedHashSet

Like `HashSet` but maintains a **doubly-linked list** connecting elements in insertion order. Slightly more memory and a tiny overhead per insertion/removal — but iteration order is predictable.

```java
import java.util.LinkedHashSet;
import java.util.Set;

Set<String> ordered = new LinkedHashSet<>();
ordered.add("banana");
ordered.add("apple");
ordered.add("cherry");
// Iterates as: banana, apple, cherry — insertion order preserved
```

### TreeSet

Backed by a Red-Black tree. Elements are kept in **sorted order** (natural ordering via `Comparable`, or a custom `Comparator`). All operations are O(log n).

```java
import java.util.TreeSet;
import java.util.Set;

Set<Integer> sorted = new TreeSet<>();
sorted.add(5);
sorted.add(1);
sorted.add(3);
System.out.println(sorted); // [1, 3, 5] — always sorted

// NavigableSet methods available via TreeSet reference
TreeSet<Integer> nav = new TreeSet<>(sorted);
System.out.println(nav.floor(4));   // 3 — largest element ≤ 4
System.out.println(nav.ceiling(2)); // 3 — smallest element ≥ 2
System.out.println(nav.headSet(4)); // [1, 3] — elements strictly less than 4
```

**Important:** Objects stored in a `TreeSet` must implement `Comparable`, or you must provide a `Comparator` — otherwise you get a `ClassCastException` at runtime.

---

## 3. Queue and Deque Implementations

`Queue` models a **FIFO** (First-In-First-Out) structure. The critical API distinction is between methods that throw exceptions on failure vs. those that return a special value:

| Operation | Throws Exception | Returns Special Value |
|-----------|-----------------|----------------------|
| Insert    | `add(e)`        | `offer(e)` → false   |
| Remove    | `remove()`      | `poll()` → null      |
| Examine   | `element()`     | `peek()` → null      |

### PriorityQueue

Elements are retrieved in **priority order**, not insertion order. Backed by a min-heap. The smallest element (by natural order or `Comparator`) is always at the head.

```java
import java.util.PriorityQueue;
import java.util.Queue;

public class PriorityQueueDemo {
    public static void main(String[] args) {
        Queue<Integer> minHeap = new PriorityQueue<>();
        minHeap.offer(30);
        minHeap.offer(10);
        minHeap.offer(20);

        // poll() always returns the MINIMUM element
        System.out.println(minHeap.poll()); // 10
        System.out.println(minHeap.poll()); // 20
        System.out.println(minHeap.poll()); // 30

        // Max-heap using a reversed comparator
        Queue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        maxHeap.offer(30);
        maxHeap.offer(10);
        maxHeap.offer(20);
        System.out.println(maxHeap.poll()); // 30
    }
}
```

### ArrayDeque

The preferred implementation for both `Stack` (LIFO) and `Queue` (FIFO) behaviour. Backed by a resizable circular array. Faster than `LinkedList` for both roles because it avoids the per-node memory allocation.

```java
import java.util.ArrayDeque;
import java.util.Deque;

Deque<String> stack = new ArrayDeque<>();
stack.push("first");   // addFirst
stack.push("second");  // addFirst
stack.push("third");   // addFirst

System.out.println(stack.pop()); // third — LIFO
System.out.println(stack.pop()); // second

Deque<String> queue = new ArrayDeque<>();
queue.offer("a");  // addLast
queue.offer("b");
queue.offer("c");

System.out.println(queue.poll()); // a — FIFO
```

**Best Practice:** Prefer `ArrayDeque` over the legacy `Stack` class and over `LinkedList` for queue/stack use cases.

---

## 4. Map Implementations

`Map<K, V>` maps keys to values. Each key is unique; each key maps to exactly one value.

### HashMap

The most common map. Keys are stored in a hash table. Average O(1) for `get`, `put`, and `remove`. Does **not** preserve insertion order. Allows one `null` key and multiple `null` values.

**How it works internally:** `HashMap` computes `key.hashCode()`, reduces it to a bucket index, and stores the entry in a linked list (or red-black tree in Java 8+ when a bucket exceeds 8 entries) at that bucket.

```java
import java.util.HashMap;
import java.util.Map;

public class HashMapDemo {
    public static void main(String[] args) {
        Map<String, Integer> scores = new HashMap<>();
        scores.put("Alice", 95);
        scores.put("Bob", 87);
        scores.put("Charlie", 92);
        scores.put("Alice", 98);  // overwrites existing value for "Alice"

        System.out.println(scores.get("Alice"));     // 98
        System.out.println(scores.get("Dave"));      // null — key not present
        System.out.println(scores.getOrDefault("Dave", 0)); // 0 — safe fallback

        // putIfAbsent — only inserts if key is not already present
        scores.putIfAbsent("Dave", 75);

        // computeIfAbsent — compute and insert in one step (great for grouping)
        Map<String, List<String>> groups = new HashMap<>();
        groups.computeIfAbsent("fruits", k -> new ArrayList<>()).add("apple");
        groups.computeIfAbsent("fruits", k -> new ArrayList<>()).add("banana");
        System.out.println(groups); // {fruits=[apple, banana]}

        // Iterate over entries
        for (Map.Entry<String, Integer> entry : scores.entrySet()) {
            System.out.println(entry.getKey() + " -> " + entry.getValue());
        }

        // Java 8+ forEach
        scores.forEach((name, score) ->
            System.out.println(name + ": " + score));
    }
}
```

### LinkedHashMap

Preserves **insertion order** (or optionally access order, useful for LRU caches). Slightly more memory than `HashMap`.

```java
Map<String, Integer> ordered = new LinkedHashMap<>();
ordered.put("c", 3);
ordered.put("a", 1);
ordered.put("b", 2);
// Iterates as: c=3, a=1, b=2 — insertion order
```

### TreeMap

Sorted by key order. O(log n) for all operations. Keys must implement `Comparable` or you must provide a `Comparator`.

```java
import java.util.TreeMap;

TreeMap<String, Integer> sorted = new TreeMap<>();
sorted.put("banana", 2);
sorted.put("apple", 5);
sorted.put("cherry", 1);

System.out.println(sorted);               // {apple=5, banana=2, cherry=1}
System.out.println(sorted.firstKey());    // apple
System.out.println(sorted.lastKey());     // cherry
System.out.println(sorted.headMap("c")); // {apple=5, banana=2}
```

### Immutable Maps (Java 9+)

```java
// Up to 10 entries with Map.of()
Map<String, Integer> immutable = Map.of("a", 1, "b", 2, "c", 3);

// More entries with Map.ofEntries()
Map<String, Integer> larger = Map.ofEntries(
    Map.entry("alpha", 1),
    Map.entry("beta",  2),
    Map.entry("gamma", 3)
);
```

---

## 5. The Collections Utility Class

`java.util.Collections` is a companion class (all static methods) for operating on collection instances.

```java
import java.util.Collections;
import java.util.List;
import java.util.ArrayList;

List<Integer> nums = new ArrayList<>(List.of(3, 1, 4, 1, 5, 9, 2, 6));

Collections.sort(nums);                   // [1, 1, 2, 3, 4, 5, 6, 9]
Collections.reverse(nums);               // [9, 6, 5, 4, 3, 2, 1, 1]
Collections.shuffle(nums);               // random order
System.out.println(Collections.min(nums));
System.out.println(Collections.max(nums));
System.out.println(Collections.frequency(nums, 1)); // count of 1s

// Create thread-safe wrappers (prefer concurrent collections instead)
List<String> syncList = Collections.synchronizedList(new ArrayList<>());

// Create unmodifiable views
List<String> fixed = Collections.unmodifiableList(new ArrayList<>(List.of("x","y")));
// fixed.add("z"); // throws UnsupportedOperationException

// Singleton and nCopies
List<String> single = Collections.singletonList("only");
List<String> repeated = Collections.nCopies(5, "hello"); // [hello, hello, hello, hello, hello]
```

---

## 6. Sequenced Collections (Java 21)

Java 21 introduced three new interfaces that formalise the concept of "a collection with a defined encounter order and accessible first/last elements":

```
SequencedCollection<E>  — List, LinkedHashSet, Deque
SequencedSet<E>         — LinkedHashSet, TreeSet
SequencedMap<K,V>       — LinkedHashMap, TreeMap
```

These interfaces add `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `removeFirst()`, `removeLast()`, and `reversed()` — methods that previously existed on `Deque` and `LinkedList` but not uniformly across all ordered collections.

```java
// Java 21+
SequencedCollection<String> list = new ArrayList<>(List.of("a", "b", "c"));
System.out.println(list.getFirst()); // a
System.out.println(list.getLast());  // c
System.out.println(list.reversed()); // [c, b, a]
```

---

## 7. Choosing the Right Collection

This decision tree covers 90% of real-world scenarios:

**Need a list (ordered, duplicates OK)?**
— Default to `ArrayList`. Use `LinkedList` only as a `Deque`.

**Need a set (no duplicates)?**
— Need fast lookup, don't care about order → `HashSet`.
— Need insertion order → `LinkedHashSet`.
— Need sorted order → `TreeSet`.

**Need a map (key → value)?**
— Default to `HashMap`. Need insertion order → `LinkedHashMap`. Need sorted keys → `TreeMap`.

**Need a queue?**
— FIFO queue → `ArrayDeque`. Priority ordering → `PriorityQueue`. Blocking/concurrent → `LinkedBlockingQueue`.

---

## 8. Fail-Fast vs. Fail-Safe Iterators

**Fail-fast** iterators (used by `ArrayList`, `HashMap`, etc.) throw `ConcurrentModificationException` if the collection is structurally modified during iteration (outside of the iterator's own `remove()` method). They detect this via an internal `modCount` counter.

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) {
    if (s.equals("b")) {
        list.remove(s); // throws ConcurrentModificationException!
    }
}

// Correct approach: use iterator's own remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) {
        it.remove(); // safe
    }
}

// Or use removeIf() (Java 8+) — the cleanest approach
list.removeIf(s -> s.equals("b"));
```

**Fail-safe** iterators (used by `CopyOnWriteArrayList`, `ConcurrentHashMap`) operate on a **snapshot** of the collection, so they never throw `ConcurrentModificationException` but may not reflect the latest state.

---

## ⚡ Best Practices & Important Points

**Always program to the interface.** Declare variables as `List<String>`, not `ArrayList<String>`. This lets you swap implementations without changing calling code.

**Override `equals()` and `hashCode()` together.** If you store custom objects in a `HashSet` or as `HashMap` keys, both methods must be consistent: objects that are `equals()` must return the same `hashCode()`. Violating this contract causes objects to "disappear" from hash-based collections.

**Never modify a collection while iterating it** with a for-each loop. Use `Iterator.remove()`, `removeIf()`, or collect to a new list.

**Prefer `Map.getOrDefault()` over null checks.** `map.get(key) == null` doesn't distinguish between "key not present" and "key mapped to null". `getOrDefault` or `containsKey` are clearer.

**Size your `ArrayList` and `HashMap` when you know the capacity.** `new ArrayList<>(expectedSize)` and `new HashMap<>(expectedSize / 0.75 + 1)` avoid costly resizing operations.

**Immutable collections (`List.of()`, `Set.of()`, `Map.of()`) are not the same as unmodifiable collections.** `Collections.unmodifiableList()` returns a view — the underlying list can still be modified through the original reference. `List.of()` is truly immutable.

**`HashMap` is not thread-safe.** In concurrent code, use `ConcurrentHashMap`. Wrapping with `Collections.synchronizedMap()` works but serialises all access, killing performance under contention.
