# Complete Java Backend Notes with Code Snippets

**Comprehensive Interview Preparation Guide**

---

## Table of Contents

1. [Object-Oriented Programming](#1-oop)
2. [Java Basics](#2-java-basics)
3. [Exception Handling](#3-exceptions)
4. [Collections Framework](#4-collections)
5. [Multithreading](#5-multithreading)
6. [Java 8+ Features](#6-java-8)
7. [Memory Management](#7-memory)
8. [Input/Output](#8-io)

---

## 1. OOP

### 1.1 Classes and Objects

```java
public class Person {
    private String name;
    private int age;
    
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public void introduce() {
        System.out.println("I'm " + name + ", " + age + " years old");
    }
}

// Creating objects
Person p1 = new Person("Alice", 25);
p1.introduce();
```

### 1.2 Encapsulation

```java
public class BankAccount {
    private double balance; // Encapsulated
    
    public void deposit(double amount) {
        if (amount > 0) balance += amount;
    }
    
    public void withdraw(double amount) {
        if (amount > 0 && amount <= balance) {
            balance -= amount;
        }
    }
    
    public double getBalance() {
        return balance;
    }
}
```

### 1.3 Inheritance

```java
// Parent class
class Animal {
    protected String name;
    
    public void eat() {
        System.out.println(name + " is eating");
    }
}

// Child class
class Dog extends Animal {
    private String breed;
    
    public Dog(String name, String breed) {
        this.name = name;
        this.breed = breed;
    }
    
    @Override
    public void eat() {
        System.out.println(name + " the " + breed + " is eating");
    }
    
    public void bark() {
        System.out.println("Woof!");
    }
}
```

### 1.4 Polymorphism

**Method Overloading (Compile-time):**

```java
class Calculator {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
    public int add(int a, int b, int c) { return a + b + c; }
}
```

**Method Overriding (Runtime):**

```java
class Shape {
    public double area() { return 0; }
}

class Circle extends Shape {
    private double radius;
    
    public Circle(double radius) {
        this.radius = radius;
    }
    
    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}

// Polymorphic behavior
Shape s = new Circle(5);
System.out.println(s.area()); // Calls Circle's area()
```

### 1.5 Abstraction

**Abstract Classes:**

```java
abstract class Vehicle {
    protected String brand;
    
    public Vehicle(String brand) {
        this.brand = brand;
    }
    
    // Abstract method
    public abstract void start();
    
    // Concrete method
    public void stop() {
        System.out.println("Vehicle stopped");
    }
}

class Car extends Vehicle {
    public Car(String brand) {
        super(brand);
    }
    
    @Override
    public void start() {
        System.out.println(brand + " car starting");
    }
}
```

**Interfaces:**

```java
interface Drawable {
    void draw(); // public abstract by default
    
    default void display() { // Java 8+
        System.out.println("Displaying");
    }
    
    static void info() { // Java 8+
        System.out.println("Drawable interface");
    }
}

class Circle implements Drawable {
    @Override
    public void draw() {
        System.out.println("Drawing circle");
    }
}
```

### 1.6 SOLID Principles

**S - Single Responsibility:**

```java
// Bad
class User {
    public void saveUser() { }
    public void sendEmail() { }
}

// Good
class User { }
class UserRepository {
    public void save(User user) { }
}
class EmailService {
    public void send(User user) { }
}
```

**O - Open/Closed:**

```java
// Use interfaces/abstract classes for extension
interface Shape {
    double area();
}

class Circle implements Shape {
    private double radius;
    public double area() { return Math.PI * radius * radius; }
}

class Rectangle implements Shape {
    private double width, height;
    public double area() { return width * height; }
}
```

### 1.7 Design Patterns

**Singleton:**

```java
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() { }
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**Factory:**

```java
interface Vehicle {
    void drive();
}

class Car implements Vehicle {
    public void drive() { System.out.println("Driving car"); }
}

class Bike implements Vehicle {
    public void drive() { System.out.println("Riding bike"); }
}

class VehicleFactory {
    public static Vehicle createVehicle(String type) {
        if (type.equals("car")) return new Car();
        if (type.equals("bike")) return new Bike();
        return null;
    }
}
```

**Builder:**

```java
public class User {
    private final String name;
    private final int age;
    private final String email;
    
    private User(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.email = builder.email;
    }
    
    public static class Builder {
        private String name;
        private int age;
        private String email;
        
        public Builder name(String name) {
            this.name = name;
            return this;
        }
        
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        
        public Builder email(String email) {
            this.email = email;
            return this;
        }
        
        public User build() {
            return new User(this);
        }
    }
}

// Usage
User user = new User.Builder()
    .name("John")
    .age(30)
    .email("john@example.com")
    .build();
```

---

## 2. Java Basics

### 2.1 Data Types

```java
// Primitives
byte b = 127;           // 8-bit
short s = 32767;        // 16-bit
int i = 2147483647;     // 32-bit
long l = 9223372036854775807L; // 64-bit

float f = 3.14f;        // 32-bit
double d = 3.14159;     // 64-bit

char c = 'A';           // 16-bit Unicode
boolean bool = true;    // true/false

// Reference types
String str = "Hello";
int[] arr = {1, 2, 3};
```

### 2.2 Wrapper Classes

```java
// Autoboxing
Integer num = 10;

// Unboxing
int primitive = num;

// Useful methods
Integer.parseInt("123");
Integer.MAX_VALUE;
Integer.compare(10, 20);
```

### 2.3 Strings

```java
String s1 = "Hello";                  // String pool
String s2 = new String("Hello");      // Heap

// String methods
s1.length();
s1.toUpperCase();
s1.toLowerCase();
s1.substring(0, 2);
s1.charAt(0);
s1.indexOf("l");
s1.contains("ell");
s1.replace("l", "L");
s1.split(" ");
s1.trim();

// String comparison
s1.equals(s2);        // Content
s1 == s2;             // Reference

// StringBuilder (mutable, not thread-safe)
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World");
sb.insert(5, ",");
sb.delete(5, 6);
sb.reverse();

// StringBuffer (mutable, thread-safe)
StringBuffer sbf = new StringBuffer("Hello");
sbf.append(" World");
```

### 2.4 Arrays

```java
// Declaration
int[] arr1 = new int[5];
int[] arr2 = {1, 2, 3, 4, 5};

// Access
int first = arr2[0];
int last = arr2[arr2.length - 1];

// Iteration
for (int i = 0; i < arr2.length; i++) {
    System.out.println(arr2[i]);
}

for (int num : arr2) {
    System.out.println(num);
}

// Multi-dimensional
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6}
};

// Array operations
Arrays.sort(arr2);
Arrays.binarySearch(arr2, 3);
Arrays.toString(arr2);
Arrays.copyOf(arr2, arr2.length);
```

### 2.5 Operators

```java
// Arithmetic
int a = 10, b = 3;
a + b;  // 13
a - b;  // 7
a * b;  // 30
a / b;  // 3
a % b;  // 1

// Relational
a == b; // false
a != b; // true
a > b;  // true
a < b;  // false

// Logical
boolean x = true, y = false;
x && y; // false (AND)
x || y; // true (OR)
!x;     // false (NOT)

// Bitwise
5 & 3;  // 1 (AND)
5 | 3;  // 7 (OR)
5 ^ 3;  // 6 (XOR)
~5;     // -6 (NOT)
5 << 1; // 10 (Left shift)
5 >> 1; // 2 (Right shift)

// Ternary
int max = (a > b) ? a : b;

// Increment/Decrement
a++;    // Post-increment
++a;    // Pre-increment
a--;    // Post-decrement
--a;    // Pre-decrement
```

### 2.6 Control Flow

```java
// if-else
if (age >= 18) {
    System.out.println("Adult");
} else {
    System.out.println("Minor");
}

// switch
switch (day) {
    case 1:
        System.out.println("Monday");
        break;
    case 2:
        System.out.println("Tuesday");
        break;
    default:
        System.out.println("Other day");
}

// Enhanced switch (Java 14+)
String dayName = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    default -> "Other day";
};

// for loop
for (int i = 0; i < 5; i++) {
    System.out.println(i);
}

// Enhanced for
int[] numbers = {1, 2, 3, 4, 5};
for (int num : numbers) {
    System.out.println(num);
}

// while
int i = 0;
while (i < 5) {
    System.out.println(i);
    i++;
}

// do-while
int j = 0;
do {
    System.out.println(j);
    j++;
} while (j < 5);

// break and continue
for (int k = 0; k < 10; k++) {
    if (k == 5) break;      // Exit loop
    if (k % 2 == 0) continue; // Skip iteration
    System.out.println(k);
}
```

---

## 3. Exceptions

### 3.1 Exception Hierarchy

```
Throwable
├── Error (Unchecked)
└── Exception
    ├── RuntimeException (Unchecked)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   └── ArithmeticException
    └── Checked Exceptions
        ├── IOException
        ├── SQLException
        └── ClassNotFoundException
```

### 3.2 Try-Catch-Finally

```java
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    System.out.println("Cannot divide by zero");
} finally {
    System.out.println("Always executes");
}

// Multiple catch
try {
    // code
} catch (IOException e) {
    // handle
} catch (SQLException e) {
    // handle
} catch (Exception e) {
    // catch all
}

// Multi-catch (Java 7+)
try {
    // code
} catch (IOException | SQLException e) {
    // handle both
}
```

### 3.3 Throw and Throws

```java
// throws - declares exceptions
public void readFile(String file) throws IOException {
    FileReader fr = new FileReader(file);
}

// throw - throws exception
public void checkAge(int age) {
    if (age < 18) {
        throw new IllegalArgumentException("Must be 18+");
    }
}
```

### 3.4 Custom Exceptions

```java
class InsufficientFundsException extends Exception {
    public InsufficientFundsException(String message) {
        super(message);
    }
}

class BankAccount {
    private double balance;
    
    public void withdraw(double amount) throws InsufficientFundsException {
        if (amount > balance) {
            throw new InsufficientFundsException("Insufficient funds");
        }
        balance -= amount;
    }
}
```

### 3.5 Try-with-Resources

```java
// Auto-closes resources
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line = br.readLine();
} catch (IOException e) {
    e.printStackTrace();
}
```

---

## 4. Collections

### 4.1 List - ArrayList

```java
List<String> list = new ArrayList<>();

// Add
list.add("Apple");
list.add(0, "Banana");

// Get
String first = list.get(0);

// Update
list.set(0, "Mango");

// Remove
list.remove(0);
list.remove("Apple");

// Size
int size = list.size();

// Contains
boolean exists = list.contains("Banana");

// Iterate
for (String item : list) {
    System.out.println(item);
}

list.forEach(item -> System.out.println(item));

// Sort
Collections.sort(list);

// Clear
list.clear();
```

### 4.2 List - LinkedList

```java
LinkedList<String> list = new LinkedList<>();

list.add("First");
list.addFirst("Start");
list.addLast("End");

String first = list.getFirst();
String last = list.getLast();

list.removeFirst();
list.removeLast();

// As Queue
list.offer("Task");  // Add to end
list.poll();         // Remove from front
list.peek();         // View front

// As Stack
list.push("A");      // Add to front
list.pop();          // Remove from front
```

### 4.3 Set - HashSet

```java
Set<String> set = new HashSet<>();

set.add("Apple");
set.add("Banana");
set.add("Apple");  // Duplicate ignored

set.contains("Apple");
set.remove("Banana");
set.size();

// Iterate
for (String item : set) {
    System.out.println(item);
}

// Set operations
Set<Integer> set1 = new HashSet<>(Arrays.asList(1, 2, 3));
Set<Integer> set2 = new HashSet<>(Arrays.asList(3, 4, 5));

Set<Integer> union = new HashSet<>(set1);
union.addAll(set2);  // {1, 2, 3, 4, 5}

Set<Integer> intersection = new HashSet<>(set1);
intersection.retainAll(set2);  // {3}

Set<Integer> difference = new HashSet<>(set1);
difference.removeAll(set2);  // {1, 2}
```

### 4.4 Set - TreeSet

```java
TreeSet<Integer> set = new TreeSet<>();

set.add(5);
set.add(2);
set.add(8);  // Automatically sorted

set.first();  // 2
set.last();   // 8
set.lower(5); // 2
set.higher(5); // 8

set.headSet(5);  // Elements < 5
set.tailSet(5);  // Elements >= 5
set.subSet(2, 8); // Elements in [2, 8)
```

### 4.5 Map - HashMap

```java
Map<String, Integer> map = new HashMap<>();

// Put
map.put("Alice", 25);
map.put("Bob", 30);

// Get
int age = map.get("Alice");
int defaultAge = map.getOrDefault("Charlie", 0);

// Contains
boolean hasKey = map.containsKey("Alice");
boolean hasValue = map.containsValue(25);

// Remove
map.remove("Bob");

// Size
int size = map.size();

// Iterate
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}

for (String key : map.keySet()) {
    System.out.println(key);
}

for (Integer value : map.values()) {
    System.out.println(value);
}

map.forEach((key, value) -> 
    System.out.println(key + ": " + value));

// Operations
map.putIfAbsent("Alice", 35);  // Won't update
map.compute("Alice", (k, v) -> v + 1);
map.merge("Bob", 5, (old, new_) -> old + new_);
```

### 4.6 Map - TreeMap

```java
TreeMap<String, Integer> map = new TreeMap<>();

map.put("Charlie", 35);
map.put("Alice", 25);
map.put("Bob", 30);  // Sorted by keys

map.firstKey();   // "Alice"
map.lastKey();    // "Charlie"
map.lowerKey("Bob");   // "Alice"
map.higherKey("Bob");  // "Charlie"

map.headMap("Bob");  // Keys < "Bob"
map.tailMap("Bob");  // Keys >= "Bob"
```

### 4.7 Queue - PriorityQueue

```java
// Min heap by default
PriorityQueue<Integer> pq = new PriorityQueue<>();

pq.offer(30);
pq.offer(10);
pq.offer(50);

pq.peek();  // 10 (min element)
pq.poll();  // Removes 10

// Max heap
PriorityQueue<Integer> maxHeap = 
    new PriorityQueue<>((a, b) -> b - a);
```

### 4.8 Comparable and Comparator

```java
// Comparable - natural ordering
class Student implements Comparable<Student> {
    String name;
    int age;
    
    @Override
    public int compareTo(Student other) {
        return this.name.compareTo(other.name);
    }
}

// Comparator - custom ordering
Comparator<Student> ageComparator = (s1, s2) -> s1.age - s2.age;
Collections.sort(students, ageComparator);

// Java 8+ Comparator
students.sort(Comparator.comparing(s -> s.name));
students.sort(Comparator.comparing(Student::getName)
                        .thenComparing(Student::getAge));
```

---

## 5. Multithreading

### 5.1 Creating Threads

```java
// Method 1: Extend Thread
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running");
    }
}

MyThread t = new MyThread();
t.start();

// Method 2: Implement Runnable
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable running");
    }
}

Thread t = new Thread(new MyRunnable());
t.start();

// Method 3: Lambda
Thread t = new Thread(() -> {
    System.out.println("Lambda thread");
});
t.start();
```

### 5.2 Thread Methods

```java
Thread t = new Thread(() -> {
    // Thread logic
});

t.start();           // Start thread
t.join();            // Wait for completion
t.sleep(1000);       // Sleep 1 second
t.setName("MyThread");
t.setPriority(Thread.MAX_PRIORITY);
t.setDaemon(true);   // Daemon thread

// Thread states
t.getState();  // NEW, RUNNABLE, BLOCKED, WAITING, TERMINATED
t.isAlive();
```

### 5.3 Synchronization

```java
class Counter {
    private int count = 0;
    
    // Synchronized method
    public synchronized void increment() {
        count++;
    }
    
    // Synchronized block
    public void incrementBlock() {
        synchronized (this) {
            count++;
        }
    }
}

// Static synchronization
class StaticCounter {
    private static int count = 0;
    
    public static synchronized void increment() {
        count++;
    }
}
```

### 5.4 Volatile Keyword

```java
class SharedResource {
    private volatile boolean flag = false;
    
    public void setFlag() {
        flag = true;  // Visible to all threads
    }
    
    public boolean getFlag() {
        return flag;
    }
}
```

### 5.5 Executor Framework

```java
// Fixed thread pool
ExecutorService executor = Executors.newFixedThreadPool(3);

for (int i = 0; i < 10; i++) {
    executor.execute(() -> {
        System.out.println("Task executed");
    });
}

executor.shutdown();

// Single thread executor
ExecutorService single = Executors.newSingleThreadExecutor();

// Cached thread pool
ExecutorService cached = Executors.newCachedThreadPool();

// Scheduled executor
ScheduledExecutorService scheduler = 
    Executors.newScheduledThreadPool(2);

scheduler.schedule(() -> {
    System.out.println("Delayed task");
}, 2, TimeUnit.SECONDS);

scheduler.scheduleAtFixedRate(() -> {
    System.out.println("Repeating task");
}, 0, 1, TimeUnit.SECONDS);
```

### 5.6 Callable and Future

```java
Callable<Integer> task = () -> {
    Thread.sleep(1000);
    return 42;
};

ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(task);

// Check if done
if (!future.isDone()) {
    System.out.println("Task running...");
}

// Get result (blocks)
Integer result = future.get();

// Get with timeout
try {
    result = future.get(2, TimeUnit.SECONDS);
} catch (TimeoutException e) {
    future.cancel(true);
}

executor.shutdown();
```

### 5.7 CountDownLatch

```java
CountDownLatch latch = new CountDownLatch(3);

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        System.out.println("Working...");
        latch.countDown();
    }).start();
}

latch.await();  // Wait for all threads
System.out.println("All done!");
```

### 5.8 CyclicBarrier

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All reached barrier!");
});

for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        try {
            System.out.println("Working...");
            barrier.await();  // Wait at barrier
            System.out.println("Proceeding...");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }).start();
}
```

### 5.9 Semaphore

```java
Semaphore semaphore = new Semaphore(2);  // 2 permits

for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        try {
            semaphore.acquire();
            System.out.println("Permit acquired");
            Thread.sleep(1000);
            semaphore.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }).start();
}
```

---

## 6. Java 8+

### 6.1 Lambda Expressions

```java
// Traditional
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Run");
    }
};

// Lambda
Runnable r2 = () -> System.out.println("Run");

// With parameters
Comparator<Integer> comp = (a, b) -> a.compareTo(b);

// Multiple statements
Calculator add = (a, b) -> {
    int result = a + b;
    return result;
};
```

### 6.2 Functional Interfaces

```java
// Predicate<T> - returns boolean
Predicate<Integer> isEven = n -> n % 2 == 0;
isEven.test(4);  // true

// Function<T, R> - transforms T to R
Function<String, Integer> length = s -> s.length();
length.apply("Hello");  // 5

// Consumer<T> - accepts T, returns nothing
Consumer<String> print = s -> System.out.println(s);
print.accept("Hello");

// Supplier<T> - returns T, no input
Supplier<Double> random = () -> Math.random();
random.get();

// BiFunction<T, U, R>
BiFunction<Integer, Integer, Integer> multiply = (a, b) -> a * b;
multiply.apply(5, 3);  // 15
```

### 6.3 Stream API

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// filter
List<Integer> even = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());

// map
List<Integer> squared = numbers.stream()
    .map(n -> n * n)
    .collect(Collectors.toList());

// flatMap
List<List<Integer>> nested = Arrays.asList(
    Arrays.asList(1, 2),
    Arrays.asList(3, 4)
);
List<Integer> flat = nested.stream()
    .flatMap(list -> list.stream())
    .collect(Collectors.toList());

// distinct
List<Integer> unique = Arrays.asList(1, 2, 2, 3, 3)
    .stream()
    .distinct()
    .collect(Collectors.toList());

// sorted
List<Integer> sorted = numbers.stream()
    .sorted()
    .collect(Collectors.toList());

// limit, skip
List<Integer> limited = numbers.stream()
    .limit(5)
    .collect(Collectors.toList());

List<Integer> skipped = numbers.stream()
    .skip(5)
    .collect(Collectors.toList());

// forEach
numbers.stream()
    .filter(n -> n % 2 == 0)
    .forEach(System.out::println);

// count
long count = numbers.stream()
    .filter(n -> n > 5)
    .count();

// reduce
int sum = numbers.stream()
    .reduce(0, (a, b) -> a + b);

Optional<Integer> max = numbers.stream()
    .reduce((a, b) -> a > b ? a : b);

// min, max
Optional<Integer> min = numbers.stream().min(Integer::compareTo);
Optional<Integer> max = numbers.stream().max(Integer::compareTo);

// anyMatch, allMatch, noneMatch
boolean anyEven = numbers.stream().anyMatch(n -> n % 2 == 0);
boolean allPositive = numbers.stream().allMatch(n -> n > 0);
boolean noneNegative = numbers.stream().noneMatch(n -> n < 0);

// findFirst, findAny
Optional<Integer> first = numbers.stream()
    .filter(n -> n > 5)
    .findFirst();

// Collectors
String joined = Arrays.asList("A", "B", "C").stream()
    .collect(Collectors.joining(", "));

Map<Boolean, List<Integer>> partitioned = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));

List<Person> people = /*...*/;
Map<Integer, List<Person>> grouped = people.stream()
    .collect(Collectors.groupingBy(Person::getAge));

int totalAge = people.stream()
    .collect(Collectors.summingInt(Person::getAge));

double avgAge = people.stream()
    .collect(Collectors.averagingInt(Person::getAge));
```

### 6.4 Method References

```java
// Static method reference
Function<String, Integer> parser = Integer::parseInt;

// Instance method reference
String prefix = "Hello, ";
Function<String, String> greeter = prefix::concat;

// Arbitrary object instance method
List<String> names = Arrays.asList("Alice", "Bob");
names.stream()
    .map(String::toUpperCase)
    .forEach(System.out::println);

// Constructor reference
Supplier<List<String>> listSupplier = ArrayList::new;
Function<String, Person> personCreator = Person::new;
```

### 6.5 Optional

```java
// Creating Optional
Optional<String> empty = Optional.empty();
Optional<String> nonEmpty = Optional.of("Hello");
Optional<String> nullable = Optional.ofNullable(null);

// Checking
if (nonEmpty.isPresent()) {
    System.out.println(nonEmpty.get());
}

if (empty.isEmpty()) {
    System.out.println("Empty");
}

// orElse
String value = empty.orElse("Default");

// orElseGet
String value2 = empty.orElseGet(() -> "Generated");

// orElseThrow
String value3 = empty.orElseThrow(() -> 
    new IllegalArgumentException("No value"));

// ifPresent
nonEmpty.ifPresent(val -> System.out.println(val));

// ifPresentOrElse
empty.ifPresentOrElse(
    val -> System.out.println(val),
    () -> System.out.println("No value")
);

// map
Optional<Integer> length = nonEmpty.map(String::length);

// flatMap
Optional<String> upper = nonEmpty.flatMap(s -> 
    Optional.of(s.toUpperCase()));

// filter
Optional<String> filtered = nonEmpty.filter(s -> s.length() > 3);

// or
Optional<String> alternative = empty.or(() -> 
    Optional.of("Alternative"));
```

### 6.6 Date and Time API

```java
// LocalDate
LocalDate today = LocalDate.now();
LocalDate specific = LocalDate.of(2024, 1, 15);
LocalDate parsed = LocalDate.parse("2024-01-15");

today.plusDays(1);
today.minusMonths(1);
today.getYear();
today.getMonth();
today.getDayOfWeek();

// LocalTime
LocalTime now = LocalTime.now();
LocalTime specific = LocalTime.of(14, 30, 45);

now.plusHours(2);
now.getHour();
now.getMinute();

// LocalDateTime
LocalDateTime dateTime = LocalDateTime.now();
LocalDateTime specific = LocalDateTime.of(2024, 1, 15, 14, 30);

// ZonedDateTime
ZonedDateTime zoned = ZonedDateTime.now();
ZonedDateTime ny = ZonedDateTime.now(ZoneId.of("America/New_York"));

// Duration - time-based
Duration duration = Duration.between(start, end);
duration.toHours();
duration.toMinutes();

// Period - date-based
Period period = Period.between(birthDate, today);
period.getYears();
period.getMonths();
period.getDays();

// Formatting
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
String formatted = today.format(formatter);
LocalDate parsed = LocalDate.parse("15/01/2024", formatter);

// Temporal Adjusters
LocalDate firstDay = today.with(TemporalAdjusters.firstDayOfMonth());
LocalDate lastDay = today.with(TemporalAdjusters.lastDayOfMonth());
LocalDate nextMonday = today.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
```

### 6.7 Default and Static Methods in Interfaces

```java
interface Vehicle {
    void start();  // Abstract
    
    default void stop() {  // Default
        System.out.println("Stopped");
    }
    
    static void service() {  // Static
        System.out.println("Servicing");
    }
}

class Car implements Vehicle {
    @Override
    public void start() {
        System.out.println("Car started");
    }
    
    // Can override default method
    @Override
    public void stop() {
        System.out.println("Car stopped");
    }
}

// Usage
Car car = new Car();
car.start();
car.stop();          // Instance method
Vehicle.service();   // Static method
```

---

## 7. Memory

### 7.1 JVM Architecture

```
1. Class Loader Subsystem
   - Loading, Linking, Initialization

2. Runtime Data Areas
   - Method Area (Metaspace)
   - Heap (Objects, Young Gen, Old Gen)
   - Stack (per thread, local variables)
   - PC Register
   - Native Method Stack

3. Execution Engine
   - Interpreter
   - JIT Compiler
   - Garbage Collector
```

### 7.2 Heap vs Stack

```java
public class Memory {
    public static void main(String[] args) {
        // Stack: primitives and references
        int x = 10;              // Stack
        String str = "Hello";    // Reference in stack
        
        // Heap: objects
        Person p = new Person(); // Object in heap
    }
}

/*
Stack:
- Thread-specific
- LIFO
- Faster
- Smaller
- Auto-managed
- StackOverflowError

Heap:
- Shared across threads
- Larger
- Slower
- Garbage collected
- OutOfMemoryError
*/
```

### 7.3 Garbage Collection

```java
// Ways to make object eligible for GC

// 1. Nullifying reference
Object obj = new Object();
obj = null;

// 2. Reassigning reference
obj = new Object();
obj = new Object();  // First object eligible

// 3. Object created in method
public void method() {
    Object obj = new Object();
}  // obj eligible after method returns

// 4. Island of isolation
Object obj1 = new Object();
Object obj2 = new Object();
obj1 = obj2;
obj2 = obj1;
obj1 = null;
obj2 = null;  // Both eligible

// Request GC (not guaranteed)
System.gc();
Runtime.getRuntime().gc();

// finalize() - called before GC (deprecated)
@Override
protected void finalize() throws Throwable {
    System.out.println("Object being garbage collected");
}

/*
GC Algorithms:
1. Serial GC - Single thread
2. Parallel GC - Multiple threads (default in Java 8)
3. G1 GC - Region-based (default in Java 9+)
4. ZGC - Low latency (Java 11+)
5. Shenandoah GC - Low pause times

JVM Options:
-Xms - Initial heap size
-Xmx - Maximum heap size
-XX:+UseG1GC
-XX:+UseZGC
*/
```

### 7.4 String Pool

```java
// String literals - in string pool
String s1 = "Hello";
String s2 = "Hello";
System.out.println(s1 == s2);  // true (same reference)

// new keyword - in heap
String s3 = new String("Hello");
String s4 = new String("Hello");
System.out.println(s3 == s4);  // false (different objects)

// equals - content comparison
System.out.println(s1.equals(s3));  // true

// intern() - adds to pool
String s5 = new String("Hello").intern();
System.out.println(s1 == s5);  // true

// String immutability
String str = "Java";
str.concat(" Programming");  // Creates new string
System.out.println(str);  // Still "Java"

str = str.concat(" Programming");  // Reassignment needed
```

---

## 8. I/O

### 8.1 File Handling

```java
import java.io.*;

// Creating file
File file = new File("example.txt");
file.createNewFile();

// File information
file.getName();
file.getPath();
file.getAbsolutePath();
file.exists();
file.isFile();
file.isDirectory();
file.length();

// Writing to file
try (FileWriter writer = new FileWriter("example.txt")) {
    writer.write("Hello, World!");
}

// Reading from file
try (FileReader reader = new FileReader("example.txt")) {
    int c;
    while ((c = reader.read()) != -1) {
        System.out.print((char) c);
    }
}

// BufferedWriter
try (BufferedWriter writer = new BufferedWriter(new FileWriter("example.txt"))) {
    writer.write("Line 1");
    writer.newLine();
    writer.write("Line 2");
}

// BufferedReader
try (BufferedReader reader = new BufferedReader(new FileReader("example.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
}

// Deleting file
file.delete();

// Directory operations
File dir = new File("testDir");
dir.mkdir();
dir.mkdirs();  // Create parent directories too

String[] files = dir.list();
File[] fileObjects = dir.listFiles();
```

### 8.2 Serialization

```java
import java.io.*;

// Class must implement Serializable
class Employee implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private String name;
    private int age;
    private transient String password;  // Won't be serialized
    private static String company = "ABC";  // Won't be serialized
    
    public Employee(String name, int age, String password) {
        this.name = name;
        this.age = age;
        this.password = password;
    }
}

// Serialization - object to bytes
Employee emp = new Employee("John", 30, "secret");

try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("employee.ser"))) {
    oos.writeObject(emp);
}

// Deserialization - bytes to object
try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("employee.ser"))) {
    Employee deserializedEmp = (Employee) ois.readObject();
}
```

### 8.3 NIO (New I/O)

```java
import java.nio.file.*;

// Path operations
Path path = Paths.get("example.txt");
Path path2 = Path.of("example.txt");  // Java 11+

path.getFileName();
path.getParent();
path.toAbsolutePath();

// File operations
Files.createFile(path);
Files.createDirectory(Paths.get("dir"));

// Write to file
String content = "Hello, NIO!";
Files.writeString(path, content);

// Read from file
String fileContent = Files.readString(path);
List<String> lines = Files.readAllLines(path);

// Append to file
Files.writeString(path, "Appended", StandardOpenOption.APPEND);

// Copy file
Files.copy(path, Paths.get("copy.txt"), 
           StandardCopyOption.REPLACE_EXISTING);

// Move file
Files.move(path, Paths.get("moved.txt"), 
           StandardCopyOption.REPLACE_EXISTING);

// File attributes
Files.size(path);
Files.isDirectory(path);
Files.isRegularFile(path);
Files.isReadable(path);
Files.isWritable(path);

// List directory
Files.list(Paths.get("."))
    .forEach(System.out::println);

// Walk directory tree
Files.walk(Paths.get("."))
    .filter(Files::isRegularFile)
    .forEach(System.out::println);

// Delete file
Files.delete(path);
```

---

## Summary

This comprehensive guide covered:

1. **OOP**: Classes, Inheritance, Polymorphism, Abstraction, SOLID, Design Patterns
2. **Java Basics**: Data types, Strings, Arrays, Operators, Control Flow
3. **Exceptions**: Try-catch, Custom exceptions, Try-with-resources
4. **Collections**: List, Set, Map, Queue implementations
5. **Multithreading**: Threads, Synchronization, Executor, Concurrent utilities
6. **Java 8+**: Lambda, Streams, Optional, Method references, Date/Time
7. **Memory**: JVM, Heap vs Stack, GC, String Pool
8. **I/O**: File handling, Serialization, NIO

---

## Interview Tips

1. **Practice coding** daily on LeetCode, HackerRank
2. **Build projects** using these concepts
3. **Explain concepts** clearly and concisely
4. **Understand** don't just memorize
5. **Review** this document regularly

---

**Good luck with your Java interviews!** 🚀

