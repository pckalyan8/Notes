# Java Collections Framework - The Ultimate Cheat Sheet

## Table of Contents
1. [Core Interfaces & Hierarchy](#1-core-interfaces--hierarchy)
2. [List Interface & Implementations](#2-list-interface--implementations)
    - [ArrayList](#21-arraylist)
    - [LinkedList](#22-linkedlist)
    - [Vector & Stack (Legacy)](#23-vector--stack-legacy)
3. [Set Interface & Implementations](#3-set-interface--implementations)
    - [HashSet](#31-hashset)
    - [LinkedHashSet](#32-linkedhashset)
    - [TreeSet](#33-treeset)
4. [Queue & Deque Interfaces & Implementations](#4-queue--deque-interfaces--implementations)
    - [PriorityQueue](#41-priorityqueue)
    - [ArrayDeque](#42-arraydeque)
5. [Map Interface & Implementations](#5-map-interface--implementations)
    - [HashMap](#51-hashmap)
    - [LinkedHashMap](#52-linkedhashmap)
    - [TreeMap](#53-treemap)
    - [Hashtable (Legacy)](#54-hashtable-legacy)
6. [Concurrent Collections](#6-concurrent-collections)
7. [Comparable vs. Comparator](#7-comparable-vs-comparator)
8. [Collections Utility Class](#8-collections-utility-class)
9. [Performance (Big-O) Summary](#9-performance-big-o-summary)
10. [Common Patterns & Best Practices](#10-common-patterns--best-practices)
11. [When to Use What: A Decision Guide](#11-when-to-use-what-a-decision-guide)

---

## 1. Core Interfaces & Hierarchy

- **`Iterable<T>`**: The root of the hierarchy. Guarantees that the object can be iterated over. Provides the `iterator()` method.
- **`Collection<E>`**: The foundation for most collections. Defines basic methods like `add()`, `remove()`, `size()`, `contains()`.
- **`List<E>`**: An **ordered** collection (sequence) that allows **duplicate** elements.
- **`Set<E>`**: A collection that allows **no duplicate** elements.
- **`Queue<E>`**: A collection for holding elements prior to processing, typically in **FIFO** (First-In-First-Out) order.
- **`Deque<E>`**: A "double-ended queue" that supports element insertion and removal at both ends.
- **`Map<K, V>`**: An object that maps keys to values. Contains **no duplicate keys**. (Does not extend `Collection`).

---

## 2. List Interface & Implementations

**Characteristics**: Ordered, allows duplicates, zero-based index.

### 2.1 ArrayList

- **Underlying Structure**: Dynamic array (`Object[]`).
- **Best for**: Read-heavy operations and fast, random access. The default choice for a `List`.
- **Performance**:
    - `get(index)`: **O(1)** - very fast.
    - `add(element)` (at end): **O(1)** amortized time.
    - `add(index, element)`: **O(n)** - requires shifting elements.
    - `remove(index)`: **O(n)** - requires shifting elements.
    - `contains(element)`: **O(n)**.
- **Synchronization**: Not synchronized.
- **Nulls**: Allows `null` elements.
- **Key Feature**: Grows automatically. Initial capacity is 10; grows by 50% when full.

```java
// Common Usage
List<String> fruits = new ArrayList<>();
fruits.add("Apple");
fruits.add("Banana");
String firstFruit = fruits.get(0); // Fast random access

// Pre-size for better performance if you know the number of elements
List<String> items = new ArrayList<>(100);
```

### 2.2 LinkedList

- **Underlying Structure**: Doubly-linked list (each element has a reference to the next and previous element).
- **Best for**: Write-heavy operations where you frequently add or remove elements from the **beginning or end**. Ideal for implementing Queues and Stacks.
- **Performance**:
    - `addFirst()` / `addLast()`: **O(1)**.
    - `removeFirst()` / `removeLast()`: **O(1)**.
    - `get(index)`: **O(n)** - very slow, must traverse the list from the beginning or end.
    - `add(index, element)` / `remove(index)`: **O(n)**.
- **Synchronization**: Not synchronized.
- **Nulls**: Allows `null` elements.

```java
// Use as a Queue (FIFO)
Queue<Task> taskQueue = new LinkedList<>();
taskQueue.offer(new Task("First Task")); // Add to end
Task nextTask = taskQueue.poll(); // Remove from start

// Use as a Stack (LIFO) - But ArrayDeque is preferred
Deque<Page> browserHistory = new LinkedList<>();
browserHistory.push(new Page("Homepage")); // Add to front
Page lastPage = browserHistory.pop(); // Remove from front
```

### 2.3 Vector & Stack (Legacy)

- **`Vector`**: A synchronized version of `ArrayList`. Slower due to locking overhead. **Avoid in new code**. Use `ArrayList` and manage synchronization externally if needed, or use `CopyOnWriteArrayList`.
- **`Stack`**: Extends `Vector`. A LIFO stack. **Avoid in new code**. Use a `Deque` implementation like `ArrayDeque` which is cleaner and faster.

---

## 3. Set Interface & Implementations

**Characteristics**: No duplicates allowed.

### 3.1 HashSet

- **Underlying Structure**: `HashMap`. The element is stored as the key.
- **Best for**: High-performance (O(1)) `add`, `remove`, and `contains` checks where order is **not important**.
- **Performance**: **O(1)** amortized time for `add`, `remove`, `contains`.
- **Ordering**: No guaranteed order.
- **Nulls**: Allows one `null` element.
- **Key Feature**: Relies on a good `hashCode()` and `equals()` implementation for performance and correctness.

```java
// Remove duplicates from a list
List<Integer> numbers = List.of(1, 2, 3, 2, 4, 1);
Set<Integer> uniqueNumbers = new HashSet<>(numbers); // uniqueNumbers is {1, 2, 3, 4}
```

### 3.2 LinkedHashSet

- **Underlying Structure**: `HashSet` + `LinkedList`.
- **Best for**: Situations where you need the performance of a `HashSet` but want to maintain **insertion order**.
- **Performance**: **O(1)** for `add`, `remove`, `contains`. Slightly more memory and overhead than `HashSet`.
- **Ordering**: Maintains the order in which elements were inserted.
- **Nulls**: Allows one `null` element.

```java
Set<String> steps = new LinkedHashSet<>();
steps.add("Step 1");
steps.add("Step 3");
steps.add("Step 2");
// Iteration will be "Step 1", "Step 3", "Step 2"
```

### 3.3 TreeSet

- **Underlying Structure**: Red-Black Tree (`TreeMap`).
- **Best for**: Storing unique elements in a **sorted order**.
- **Performance**: **O(log n)** for `add`, `remove`, `contains` due to the tree structure.
- **Ordering**: Sorted according to the elements' natural order (if they implement `Comparable`) or by a `Comparator` provided at creation.
- **Nulls**: **Does not allow `null` elements** (throws `NullPointerException`).

```java
// Elements are automatically sorted
NavigableSet<Integer> sortedScores = new TreeSet<>();
sortedScores.add(88);
sortedScores.add(95);
sortedScores.add(72);
// Iteration will be 72, 88, 95

// Get all scores above 90
NavigableSet<Integer> highScores = sortedScores.tailSet(90, true);
```

---

## 4. Queue & Deque Interfaces & Implementations

### 4.1 PriorityQueue

- **Underlying Structure**: Binary heap.
- **Best for**: Processing elements based on **priority** rather than FIFO.
- **Performance**:
    - `offer()` (add): **O(log n)**.
    - `poll()` (remove head): **O(log n)**.
    - `peek()` (view head): **O(1)**.
- **Ordering**: The "head" of the queue is the element with the lowest value (min-heap) by default, or as defined by a `Comparator`.
- **Nulls**: **Does not allow `null` elements**.

```java
// Min-heap by default
Queue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5);
minHeap.offer(1);
minHeap.offer(3);
System.out.println(minHeap.poll()); // Prints 1

// Max-heap using a Comparator
Queue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(5);
maxHeap.offer(1);
maxHeap.offer(3);
System.out.println(maxHeap.poll()); // Prints 5
```

### 4.2 ArrayDeque

- **Underlying Structure**: Resizable circular array.
- **Best for**: A general-purpose **Stack (LIFO)** or **Queue (FIFO)**. Faster than `LinkedList` for these purposes.
- **Performance**: **O(1)** amortized time for `addFirst`, `addLast`, `removeFirst`, `removeLast`.
- **Nulls**: **Does not allow `null` elements**.

```java
// Use as a Stack (preferred over Stack class)
Deque<String> stack = new ArrayDeque<>();
stack.push("A");
stack.push("B");
System.out.println(stack.pop()); // Prints "B"

// Use as a Queue
Queue<String> queue = new ArrayDeque<>();
queue.offer("A");
queue.offer("B");
System.out.println(queue.poll()); // Prints "A"
```

---

## 5. Map Interface & Implementations

**Characteristics**: Key-value pairs, keys must be unique.

### 5.1 HashMap

- **Underlying Structure**: Hash table (array of nodes).
- **Best for**: High-performance (O(1)) key-value lookups where **order is not important**. The default choice for a `Map`.
- **Performance**: **O(1)** amortized time for `put`, `get`, `remove`.
- **Ordering**: No guaranteed order.
- **Nulls**: Allows one `null` key and multiple `null` values.
- **Key Feature**: Performance degrades if many keys have the same `hashCode()`. Since Java 8, buckets with too many collisions are converted from linked lists to red-black trees, improving worst-case performance from O(n) to O(log n).

```java
Map<String, User> userCache = new HashMap<>();
userCache.put("john.doe", new User("John"));
User john = userCache.get("john.doe");
```

### 5.2 LinkedHashMap

- **Underlying Structure**: `HashMap` + Doubly-linked list running through all its entries.
- **Best for**: Situations where you need the performance of a `HashMap` but want to maintain **insertion order**. Also excellent for building **LRU (Least Recently Used) caches**.
- **Performance**: **O(1)** for `put`, `get`, `remove`.
- **Ordering**: Maintains the order in which keys were inserted.
- **Nulls**: Allows one `null` key and multiple `null` values.

```java
// LRU Cache Example
int capacity = 100;
Map<Integer, String> lruCache = new LinkedHashMap<Integer, String>(capacity, 0.75f, true) {
    protected boolean removeEldestEntry(Map.Entry<Integer, String> eldest) {
        return size() > capacity;
    }
};
```

### 5.3 TreeMap

- **Underlying Structure**: Red-Black Tree.
- **Best for**: Storing key-value pairs in **sorted key order**.
- **Performance**: **O(log n)** for `put`, `get`, `remove`.
- **Ordering**: Keys are sorted according to their natural order or by a `Comparator`.
- **Nulls**: **Does not allow a `null` key**.

```java
NavigableMap<LocalDate, String> eventCalendar = new TreeMap<>();
eventCalendar.put(LocalDate.of(2023, 12, 25), "Christmas");
eventCalendar.put(LocalDate.of(2023, 1, 1), "New Year");
// Iterating over keySet() will be in sorted date order.

// Get all events in January 2023
NavigableMap<LocalDate, String> janEvents = eventCalendar.subMap(
    LocalDate.of(2023, 1, 1), true,
    LocalDate.of(2023, 1, 31), true
);
```

### 5.4 Hashtable (Legacy)

- A synchronized version of `HashMap`. Slower due to locking. **Avoid in new code**. Use `ConcurrentHashMap` for thread-safe map operations.
- Does **not** allow `null` keys or values.

---

## 6. Concurrent Collections

These are thread-safe collections designed for multi-threaded environments. They are part of the `java.util.concurrent` package.

| Standard Collection | Thread-Safe Alternative | Key Feature |
| :--- | :--- | :--- |
| `HashMap` | `ConcurrentHashMap` | High-performance, lock-free reads. |
| `TreeMap` | `ConcurrentSkipListMap` | A thread-safe, sorted map. |
| `ArrayList` | `CopyOnWriteArrayList` | Thread-safe reads are very fast; writes are expensive as they create a copy of the array. Best for read-mostly scenarios. |
| `Queue` | `BlockingQueue` interface (`ArrayBlockingQueue`, `LinkedBlockingQueue`) | The queue will block a thread trying to `put` into a full queue or `take` from an empty queue. Essential for producer-consumer patterns. |

```java
// Example: ConcurrentHashMap
Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();
// Multiple threads can read and write to this map safely without explicit locking.

// Example: Producer-Consumer with BlockingQueue
BlockingQueue<Message> queue = new LinkedBlockingQueue<>(10);

// Producer Thread
new Thread(() -> {
    try {
        queue.put(new Message("Hello")); // Blocks if queue is full
    } catch (InterruptedException e) { /* handle */ }
}).start();

// Consumer Thread
new Thread(() -> {
    try {
        Message msg = queue.take(); // Blocks if queue is empty
        System.out.println("Consumed: " + msg.getContent());
    } catch (InterruptedException e) { /* handle */ }
}).start();
```

---

## 7. Comparable vs. Comparator

- **`Comparable`**: For providing a single, **natural ordering** for a class. The class itself must implement `Comparable<T>` and the `compareTo()` method.
- **`Comparator`**: For providing **multiple, different ways of ordering** or for sorting a class that you cannot modify. It's a separate class that implements `Comparator<T>` and the `compare()` method.

```java
// Using Comparable for natural ordering by ID
class Employee implements Comparable<Employee> {
    private Integer id;
    private String name;
    
    @Override
    public int compareTo(Employee other) {
        return this.id.compareTo(other.id);
    }
}

// Using Comparator for a different ordering (by name)
class EmployeeNameComparator implements Comparator<Employee> {
    @Override
    public int compare(Employee e1, Employee e2) {
        return e1.getName().compareTo(e2.getName());
    }
}

// In Java 8+, use static helper methods
List<Employee> employees = ...;
employees.sort(Comparator.comparing(Employee::getName).thenComparing(Employee::getSalary).reversed());
```

---

## 8. Collections Utility Class

Provides static helper methods for collections.

- **Sorting**: `Collections.sort(list)`, `Collections.reverse(list)`
- **Searching**: `Collections.binarySearch(list, key)` (list must be sorted)
- **Wrappers**:
    - `Collections.unmodifiableList(list)`: Creates a read-only view.
    - `Collections.synchronizedList(list)`: Creates a thread-safe (but slow) wrapper.
- **Empty/Singleton**: `Collections.emptyList()`, `Collections.singleton("element")`

---

## 9. Performance (Big-O) Summary

| Collection | `add` | `remove` | `get`/`contains` | Key Feature |
| :--- | :---: | :---: | :---: | :--- |
| **ArrayList** | O(1)* | O(n) | O(1) | Fast random access |
| **LinkedList** | O(1) | O(1) | O(n) | Fast add/remove at ends |
| **HashSet** | O(1)* | O(1)* | O(1)* | No duplicates, unordered |
| **LinkedHashSet**| O(1)* | O(1)* | O(1)* | No duplicates, insertion order |
| **TreeSet** | O(log n)| O(log n)| O(log n)| No duplicates, sorted |
| **HashMap** | O(1)* | O(1)* | O(1)* | Fast key-value access |
| **TreeMap** | O(log n)| O(log n)| O(log n)| Sorted key-value access |
| **PriorityQueue**| O(log n)| O(log n)| O(1) | Min/max heap |

`*` = Amortized constant time.

---

## 10. Common Patterns & Best Practices

1.  **Program to the Interface**: Declare variables using the interface type, not the implementation type. This makes your code more flexible.
    ```java
    // GOOD
    List<String> names = new ArrayList<>();
    // BAD
    ArrayList<String> names = new ArrayList<>();
    ```
2.  **Use Diamond Operator**: Let the compiler infer the type for cleaner code. `new ArrayList<>()` instead of `new ArrayList<String>()`.
3.  **Use Factory Methods (Java 9+)**: For creating small, immutable collections.
    ```java
    List<String> immutableList = List.of("A", "B", "C");
    ```
4.  **Pre-size Collections**: If you know the approximate number of elements, initialize the collection with a capacity to avoid costly resizing.
    ```java
    List<Item> items = new ArrayList<>(1000);
    Map<String, Data> dataMap = new HashMap<>((int) (1000 / 0.75f) + 1); // For maps
    ```

---

## 11. When to Use What: A Decision Guide

1.  **Do you need to store key-value pairs?**
    - **YES**: Use a `Map`.
        - **Need keys sorted?** -> `TreeMap`.
        - **Need to maintain insertion order?** -> `LinkedHashMap`.
        - **Need thread-safety?** -> `ConcurrentHashMap`.
        - **None of the above?** -> `HashMap` (default choice).
2.  **Do you need to prevent duplicate elements?**
    - **YES**: Use a `Set`.
        - **Need elements sorted?** -> `TreeSet`.
        - **Need to maintain insertion order?** -> `LinkedHashSet`.
        - **None of the above?** -> `HashSet` (default choice).
3.  **If you need an ordered sequence and duplicates are okay:**
    - Use a `List`.
        - **Need fast random access by index (e.g., `get(i)`)?** -> `ArrayList` (default choice).
        - **Need to frequently add/remove from the beginning or end?** -> `LinkedList`.
        - **Need thread-safety for a read-heavy list?** -> `CopyOnWriteArrayList`.
4.  **Do you need a Stack (LIFO) or Queue (FIFO)?**
    - **YES**: Use `ArrayDeque` (it's faster than `LinkedList` for this).
5.  **Do you need to process elements based on priority?**
    - **YES**: Use `PriorityQueue`.
