# Complete Java & Spring Interview Questions, Answers & LLD Problems

> **Comprehensive Interview Preparation Guide**  
> **Includes: Core Java, Spring, Spring Boot, Microservices, LLD Problems with Solutions**  
> **Last Updated: February 2026**

---

## Table of Contents

### Part A: Interview Questions & Answers
1. [Core Java Questions](#part-a-interview-questions)
2. [Collections Framework](#2-collections-framework-questions)
3. [Multithreading & Concurrency](#3-multithreading--concurrency-questions)
4. [Java 8+ Features](#4-java-8-features-questions)
5. [Spring Framework](#5-spring-framework-questions)
6. [Spring Boot](#6-spring-boot-questions)
7. [Spring Data JPA](#7-spring-data-jpa-questions)
8. [Spring Security](#8-spring-security-questions)
9. [Microservices](#9-microservices-questions)
10. [REST API & Design](#10-rest-api--design-questions)

### Part B: Low-Level Design (LLD) Problems
11. [LLD Problems with Solutions](#part-b-low-level-design-problems)

---

# Part A: Interview Questions

## 1. Core Java Questions

### Q1: What is the difference between JDK, JRE, and JVM?

**Answer:**

**JVM (Java Virtual Machine):**
- Runtime environment that executes Java bytecode
- Platform-dependent (different for Windows, Linux, Mac)
- Provides runtime environment for Java programs
- Components: Class Loader, Memory Area, Execution Engine

**JRE (Java Runtime Environment):**
- JVM + Libraries + Other files
- Required to run Java applications
- Does not contain development tools like compiler

**JDK (Java Development Kit):**
- JRE + Development Tools (javac, jar, javadoc, etc.)
- Required to develop Java applications
- Contains everything needed to compile, debug, and run Java programs

```
┌─────────────────────────────────┐
│           JDK                   │
│  ┌──────────────────────────┐  │
│  │         JRE              │  │
│  │  ┌───────────────────┐  │  │
│  │  │       JVM         │  │  │
│  │  └───────────────────┘  │  │
│  │  + Libraries            │  │
│  └──────────────────────────┘  │
│  + Development Tools            │
└─────────────────────────────────┘
```

---

### Q2: Explain the difference between `==` and `equals()` method.

**Answer:**

**`==` Operator:**
- Compares references (memory addresses)
- For primitives, compares values
- Returns true if both references point to the same object

**`equals()` Method:**
- Compares object contents/values
- Defined in Object class, can be overridden
- Default implementation uses `==`

```java
public class Example {
    public static void main(String[] args) {
        // String literals - stored in String pool
        String s1 = "Hello";
        String s2 = "Hello";
        
        System.out.println(s1 == s2);        // true (same reference)
        System.out.println(s1.equals(s2));   // true (same content)
        
        // String objects - stored in heap
        String s3 = new String("Hello");
        String s4 = new String("Hello");
        
        System.out.println(s3 == s4);        // false (different references)
        System.out.println(s3.equals(s4));   // true (same content)
        
        // Custom objects
        Person p1 = new Person("John", 25);
        Person p2 = new Person("John", 25);
        
        System.out.println(p1 == p2);        // false (different objects)
        System.out.println(p1.equals(p2));   // false (unless equals() is overridden)
    }
}

// Proper equals() implementation
class Person {
    private String name;
    private int age;
    
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        
        Person person = (Person) obj;
        return age == person.age && 
               Objects.equals(name, person.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

**Key Points:**
- Always override `hashCode()` when overriding `equals()`
- Contract: if `a.equals(b)` is true, then `a.hashCode() == b.hashCode()` must be true
- Reverse is not necessarily true

---

### Q3: What is the difference between String, StringBuilder, and StringBuffer?

**Answer:**

| Feature | String | StringBuilder | StringBuffer |
|---------|--------|--------------|--------------|
| Mutability | Immutable | Mutable | Mutable |
| Thread-Safety | Thread-safe | Not thread-safe | Thread-safe |
| Performance | Slow (creates new objects) | Fast | Slower than StringBuilder |
| Use Case | When string doesn't change | Single-threaded string manipulation | Multi-threaded string manipulation |

```java
public class StringComparison {
    public static void main(String[] args) {
        
        // String - Immutable
        String str = "Hello";
        str = str + " World";  // Creates new object, original "Hello" discarded
        
        // StringBuilder - Mutable, Not thread-safe, Fast
        StringBuilder sb = new StringBuilder("Hello");
        sb.append(" World");  // Modifies same object
        sb.insert(5, ",");
        sb.delete(5, 6);
        String result = sb.toString();
        
        // StringBuffer - Mutable, Thread-safe, Synchronized
        StringBuffer sbf = new StringBuffer("Hello");
        sbf.append(" World");  // Synchronized method
        
        // Performance comparison
        long start = System.currentTimeMillis();
        
        // String concatenation - Slow
        String s = "";
        for (int i = 0; i < 10000; i++) {
            s += i;  // Creates 10000 new String objects
        }
        System.out.println("String: " + (System.currentTimeMillis() - start));
        
        // StringBuilder - Fast
        start = System.currentTimeMillis();
        StringBuilder builder = new StringBuilder();
        for (int i = 0; i < 10000; i++) {
            builder.append(i);  // Modifies same object
        }
        System.out.println("StringBuilder: " + (System.currentTimeMillis() - start));
    }
}
```

---

### Q4: Explain Java Memory Model and Garbage Collection.

**Answer:**

**Java Memory Model:**

```
┌────────────────────────────────────────────────────┐
│                    JVM Memory                      │
├────────────────────────────────────────────────────┤
│  Heap Memory (Shared across threads)              │
│  ┌──────────────────────────────────────────────┐ │
│  │  Young Generation                            │ │
│  │  ├── Eden Space                              │ │
│  │  ├── Survivor Space 0 (S0)                   │ │
│  │  └── Survivor Space 1 (S1)                   │ │
│  ├──────────────────────────────────────────────┤ │
│  │  Old Generation (Tenured)                    │ │
│  └──────────────────────────────────────────────┘ │
├────────────────────────────────────────────────────┤
│  Non-Heap Memory                                   │
│  ├── Metaspace (Class metadata - Java 8+)         │
│  └── Code Cache (JIT compiled code)               │
├────────────────────────────────────────────────────┤
│  Stack Memory (Per thread)                         │
│  ├── Local variables                               │
│  ├── Method calls                                  │
│  └── Partial results                               │
└────────────────────────────────────────────────────┘
```

**Garbage Collection Process:**

1. **Minor GC** (Young Generation):
   - New objects created in Eden
   - When Eden full, Minor GC triggered
   - Live objects moved to Survivor space
   - Objects surviving multiple cycles promoted to Old Generation

2. **Major GC** (Old Generation):
   - Triggered when Old Generation full
   - More time-consuming
   - May cause "Stop-the-World" pause

```java
public class GCExample {
    public static void main(String[] args) {
        
        // Object eligible for GC - different ways
        
        // 1. Nullifying reference
        String str = new String("Hello");
        str = null;  // Eligible for GC
        
        // 2. Reassigning reference
        String s1 = new String("A");
        s1 = new String("B");  // "A" eligible for GC
        
        // 3. Object created in method
        createObject();  // Object eligible after method returns
        
        // 4. Island of isolation
        Example e1 = new Example();
        Example e2 = new Example();
        e1.obj = e2;
        e2.obj = e1;
        e1 = null;
        e2 = null;  // Both eligible for GC
        
        // Request GC (not guaranteed)
        System.gc();
        Runtime.getRuntime().gc();
    }
    
    static void createObject() {
        String temp = new String("Temporary");
    }  // temp eligible for GC
}

class Example {
    Example obj;
}
```

**GC Algorithms:**
- **Serial GC**: Single thread, small applications
- **Parallel GC**: Multiple threads, throughput
- **G1 GC**: Region-based, balanced (default Java 9+)
- **ZGC**: Ultra-low latency (Java 11+)

---

### Q5: What is the difference between abstract class and interface?

**Answer:**

| Feature | Abstract Class | Interface |
|---------|---------------|-----------|
| Multiple Inheritance | No | Yes |
| Method Types | Abstract + Concrete | Abstract + Default + Static |
| Variables | Any type | public static final only |
| Constructors | Yes | No |
| Access Modifiers | Any | public only (methods) |
| When to Use | IS-A relationship | CAN-DO capability |

```java
// Abstract Class Example
abstract class Animal {
    // Instance variables
    protected String name;
    protected int age;
    
    // Constructor
    public Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    // Abstract method
    public abstract void makeSound();
    
    // Concrete method
    public void sleep() {
        System.out.println(name + " is sleeping");
    }
    
    // Concrete method
    public void eat() {
        System.out.println(name + " is eating");
    }
}

class Dog extends Animal {
    public Dog(String name, int age) {
        super(name, age);
    }
    
    @Override
    public void makeSound() {
        System.out.println("Woof!");
    }
}

// Interface Example
interface Flyable {
    // Constants (public static final)
    int MAX_ALTITUDE = 10000;
    
    // Abstract method
    void fly();
    
    // Default method (Java 8+)
    default void land() {
        System.out.println("Landing...");
    }
    
    // Static method (Java 8+)
    static void checkWeather() {
        System.out.println("Weather is good for flying");
    }
}

interface Swimmable {
    void swim();
}

// Multiple interface implementation
class Duck extends Animal implements Flyable, Swimmable {
    public Duck(String name, int age) {
        super(name, age);
    }
    
    @Override
    public void makeSound() {
        System.out.println("Quack!");
    }
    
    @Override
    public void fly() {
        System.out.println("Duck is flying");
    }
    
    @Override
    public void swim() {
        System.out.println("Duck is swimming");
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        Duck duck = new Duck("Donald", 2);
        duck.makeSound();  // Abstract class method
        duck.sleep();      // Concrete method from abstract class
        duck.fly();        // Interface method
        duck.swim();       // Another interface method
        duck.land();       // Default interface method
        
        Flyable.checkWeather();  // Static interface method
    }
}
```

**Java 8+ Changes:**
- Interfaces can have default and static methods
- Reduced difference between abstract classes and interfaces
- Still can't have instance variables or constructors in interfaces

---

### Q6: Explain method overloading and method overriding.

**Answer:**

**Method Overloading (Compile-time Polymorphism):**
- Same method name, different parameters
- Can have different return types
- In same class or parent-child class
- Resolved at compile time

**Method Overriding (Runtime Polymorphism):**
- Same method signature
- Must have same return type (or covariant)
- In parent-child relationship
- Resolved at runtime

```java
// Method Overloading
class Calculator {
    // Same name, different parameters
    public int add(int a, int b) {
        return a + b;
    }
    
    public int add(int a, int b, int c) {
        return a + b + c;
    }
    
    public double add(double a, double b) {
        return a + b;
    }
    
    // Different order of parameters
    public void display(String name, int age) {
        System.out.println(name + " - " + age);
    }
    
    public void display(int age, String name) {
        System.out.println(age + " - " + name);
    }
}

// Method Overriding
class Parent {
    public void display() {
        System.out.println("Parent display");
    }
    
    public Number getValue() {
        return 10;
    }
}

class Child extends Parent {
    @Override
    public void display() {
        System.out.println("Child display");
    }
    
    // Covariant return type
    @Override
    public Integer getValue() {  // Integer is subclass of Number
        return 20;
    }
}

public class Main {
    public static void main(String[] args) {
        // Overloading
        Calculator calc = new Calculator();
        System.out.println(calc.add(5, 10));       // Calls add(int, int)
        System.out.println(calc.add(5, 10, 15));   // Calls add(int, int, int)
        System.out.println(calc.add(5.5, 10.5));   // Calls add(double, double)
        
        // Overriding - Runtime polymorphism
        Parent p = new Parent();
        p.display();  // Parent display
        
        Parent pc = new Child();
        pc.display();  // Child display (runtime decision)
        
        Child c = new Child();
        c.display();  // Child display
    }
}
```

**Rules for Overriding:**
1. Method signature must be same
2. Return type must be same or covariant
3. Access modifier cannot be more restrictive
4. Cannot override final, static, or private methods
5. Can throw narrower or fewer exceptions

---

### Q7: What is the difference between final, finally, and finalize?

**Answer:**

**final:**
- Keyword used with variables, methods, and classes
- final variable: constant, cannot be reassigned
- final method: cannot be overridden
- final class: cannot be inherited

**finally:**
- Block used with try-catch
- Always executes (except System.exit())
- Used for cleanup code

**finalize():**
- Method called by GC before object destruction
- Deprecated in Java 9
- Not recommended to use

```java
// final keyword
public class FinalExample {
    
    // final variable
    final int MAX_VALUE = 100;
    
    // final method
    public final void display() {
        System.out.println("Cannot override this method");
    }
}

// final class
final class ImmutableClass {
    // Cannot be inherited
}

// finally block
public class FinallyExample {
    public static void main(String[] args) {
        
        try {
            int result = 10 / 0;
        } catch (ArithmeticException e) {
            System.out.println("Exception caught");
        } finally {
            System.out.println("This always executes");
            // Cleanup code: close connections, files, etc.
        }
        
        // Even with return
        System.out.println(testFinally());  // Output: 2
    }
    
    static int testFinally() {
        try {
            return 1;
        } finally {
            return 2;  // This return overrides try's return
        }
    }
}

// finalize() method (Deprecated)
class Resource {
    @Override
    protected void finalize() throws Throwable {
        try {
            System.out.println("Finalize called - cleanup");
        } finally {
            super.finalize();
        }
    }
}
```

---

### Q8: Explain exception hierarchy in Java.

**Answer:**

```
                    Throwable
                   /         \
              Error          Exception
             /                /        \
    OutOfMemory      RuntimeException  Checked Exceptions
    StackOverflow    /       |      \       |
                    NPE    AIOOBE   IAE    IOException
                                            SQLException
                                            ClassNotFoundException

NPE = NullPointerException
AIOOBE = ArrayIndexOutOfBoundsException
IAE = IllegalArgumentException
```

**Checked vs Unchecked Exceptions:**

| Checked Exceptions | Unchecked Exceptions |
|-------------------|---------------------|
| Must be handled or declared | Not required to handle |
| Compile-time checking | Runtime checking |
| IOException, SQLException | RuntimeException subclasses |
| Recoverable | Usually programming errors |

```java
public class ExceptionExample {
    
    // Checked Exception - must handle or declare
    public void readFile(String filename) throws IOException {
        FileReader file = new FileReader(filename);
        // Code
    }
    
    // Handling checked exception
    public void processFile() {
        try {
            readFile("data.txt");
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    // Unchecked Exception - optional to handle
    public void divide(int a, int b) {
        int result = a / b;  // May throw ArithmeticException
    }
    
    // Custom Exception
    public void validateAge(int age) throws InvalidAgeException {
        if (age < 18) {
            throw new InvalidAgeException("Age must be 18 or above");
        }
    }
}

// Custom Checked Exception
class InvalidAgeException extends Exception {
    public InvalidAgeException(String message) {
        super(message);
    }
}

// Custom Unchecked Exception
class InsufficientBalanceException extends RuntimeException {
    public InsufficientBalanceException(String message) {
        super(message);
    }
}

// Best practices
class BestPractices {
    
    // 1. Catch specific exceptions
    public void good() {
        try {
            // code
        } catch (FileNotFoundException e) {
            // Handle file not found
        } catch (IOException e) {
            // Handle other IO exceptions
        }
    }
    
    // 2. Don't catch Throwable
    public void bad() {
        try {
            // code
        } catch (Throwable t) {  // Bad practice
            // Catches errors too
        }
    }
    
    // 3. Always close resources
    public void closeResources() {
        try (FileReader fr = new FileReader("file.txt")) {
            // Use resource
        } catch (IOException e) {
            // Handle
        }
        // Resource automatically closed
    }
}
```

---

### Q9: What is the difference between Comparable and Comparator?

**Answer:**

| Comparable | Comparator |
|-----------|------------|
| Single sorting sequence | Multiple sorting sequences |
| Modifies actual class | Separate class |
| compareTo() method | compare() method |
| java.lang package | java.util package |
| Natural ordering | Custom ordering |

```java
// Comparable - Natural ordering
class Employee implements Comparable<Employee> {
    private int id;
    private String name;
    private double salary;
    
    public Employee(int id, String name, double salary) {
        this.id = id;
        this.name = name;
        this.salary = salary;
    }
    
    // Natural ordering by ID
    @Override
    public int compareTo(Employee other) {
        return this.id - other.id;
    }
    
    // Getters
    public int getId() { return id; }
    public String getName() { return name; }
    public double getSalary() { return salary; }
    
    @Override
    public String toString() {
        return "Employee{id=" + id + ", name='" + name + "', salary=" + salary + "}";
    }
}

// Comparator - Custom ordering
class NameComparator implements Comparator<Employee> {
    @Override
    public int compare(Employee e1, Employee e2) {
        return e1.getName().compareTo(e2.getName());
    }
}

class SalaryComparator implements Comparator<Employee> {
    @Override
    public int compare(Employee e1, Employee e2) {
        return Double.compare(e1.getSalary(), e2.getSalary());
    }
}

public class ComparatorExample {
    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>();
        employees.add(new Employee(3, "John", 50000));
        employees.add(new Employee(1, "Alice", 60000));
        employees.add(new Employee(2, "Bob", 55000));
        
        System.out.println("Original: " + employees);
        
        // Using Comparable (natural ordering)
        Collections.sort(employees);
        System.out.println("Sorted by ID: " + employees);
        
        // Using Comparator (custom ordering)
        Collections.sort(employees, new NameComparator());
        System.out.println("Sorted by Name: " + employees);
        
        Collections.sort(employees, new SalaryComparator());
        System.out.println("Sorted by Salary: " + employees);
        
        // Lambda expressions (Java 8+)
        employees.sort((e1, e2) -> e1.getName().compareTo(e2.getName()));
        System.out.println("Sorted by Name (Lambda): " + employees);
        
        // Comparator.comparing (Java 8+)
        employees.sort(Comparator.comparing(Employee::getSalary));
        System.out.println("Sorted by Salary (Method Reference): " + employees);
        
        // Multiple sorting criteria
        employees.sort(Comparator.comparing(Employee::getName)
                                 .thenComparing(Employee::getSalary));
        System.out.println("Sorted by Name, then Salary: " + employees);
        
        // Reverse order
        employees.sort(Comparator.comparing(Employee::getSalary).reversed());
        System.out.println("Sorted by Salary (Descending): " + employees);
    }
}
```

---

### Q10: Explain the concept of immutability and how to create immutable class.

**Answer:**

**Immutable Class**: Once created, its state cannot be modified.

**Benefits:**
- Thread-safe
- Cacheable
- Good for HashMap keys
- Defensive copying not needed

**Rules for Immutable Class:**
1. Declare class as final
2. Make all fields private and final
3. No setter methods
4. Initialize all fields via constructor
5. Perform deep copy for mutable objects
6. Return copies of mutable objects

```java
// Immutable Class Example
public final class ImmutablePerson {
    private final String name;
    private final int age;
    private final Date birthDate;
    private final List<String> hobbies;
    
    public ImmutablePerson(String name, int age, Date birthDate, List<String> hobbies) {
        this.name = name;
        this.age = age;
        
        // Deep copy for mutable Date object
        this.birthDate = new Date(birthDate.getTime());
        
        // Deep copy for mutable List
        this.hobbies = new ArrayList<>(hobbies);
    }
    
    // Only getters, no setters
    public String getName() {
        return name;
    }
    
    public int getAge() {
        return age;
    }
    
    // Return copy of mutable object
    public Date getBirthDate() {
        return new Date(birthDate.getTime());
    }
    
    // Return unmodifiable list
    public List<String> getHobbies() {
        return Collections.unmodifiableList(hobbies);
    }
    
    @Override
    public String toString() {
        return "ImmutablePerson{name='" + name + "', age=" + age + 
               ", birthDate=" + birthDate + ", hobbies=" + hobbies + "}";
    }
}

// Usage
public class TestImmutable {
    public static void main(String[] args) {
        List<String> hobbies = new ArrayList<>(Arrays.asList("Reading", "Swimming"));
        Date birthDate = new Date();
        
        ImmutablePerson person = new ImmutablePerson("John", 30, birthDate, hobbies);
        
        System.out.println(person);
        
        // Try to modify - won't affect person object
        hobbies.add("Gaming");
        birthDate.setTime(0);
        
        System.out.println(person);  // No change
        
        // Try to modify returned values
        Date retrievedDate = person.getBirthDate();
        retrievedDate.setTime(0);  // Won't affect person's birthDate
        
        List<String> retrievedHobbies = person.getHobbies();
        // retrievedHobbies.add("Cooking");  // UnsupportedOperationException
    }
}

// Using Lombok for immutability
@Value  // Creates immutable class
@AllArgsConstructor
public class ImmutableEmployee {
    String name;
    int age;
    double salary;
    
    // Automatically generates:
    // - private final fields
    // - constructor
    // - getters (no setters)
    // - equals(), hashCode(), toString()
}
```

**Built-in Immutable Classes:**
- String
- Wrapper classes (Integer, Long, etc.)
- BigInteger, BigDecimal
- LocalDate, LocalTime, LocalDateTime

---

## 2. Collections Framework Questions

### Q11: Explain the difference between ArrayList and LinkedList.

**Answer:**

| Feature | ArrayList | LinkedList |
|---------|-----------|------------|
| Data Structure | Dynamic array | Doubly linked list |
| Access Time | O(1) - Random access | O(n) - Sequential access |
| Insert/Delete (middle) | O(n) - Shifting required | O(1) - Just pointer change |
| Insert/Delete (end) | O(1) amortized | O(1) |
| Memory | Less overhead | More (stores references) |
| Best For | Frequent access | Frequent insertion/deletion |

```java
public class ArrayListVsLinkedList {
    public static void main(String[] args) {
        
        // ArrayList - Good for random access
        List<String> arrayList = new ArrayList<>();
        arrayList.add("A");
        arrayList.add("B");
        arrayList.add("C");
        
        // Fast access by index
        String element = arrayList.get(1);  // O(1)
        
        // Slow insertion in middle
        arrayList.add(1, "X");  // O(n) - shifts elements
        
        // LinkedList - Good for insertion/deletion
        List<String> linkedList = new LinkedList<>();
        linkedList.add("A");
        linkedList.add("B");
        linkedList.add("C");
        
        // Slow access by index
        String elem = linkedList.get(1);  // O(n) - traverse list
        
        // Fast insertion in middle (if you have iterator)
        LinkedList<String> ll = (LinkedList<String>) linkedList;
        ll.addFirst("X");  // O(1)
        ll.addLast("Y");   // O(1)
        
        // Performance comparison
        int n = 100000;
        
        // ArrayList - Fast random access
        List<Integer> al = new ArrayList<>();
        long start = System.currentTimeMillis();
        for (int i = 0; i < n; i++) {
            al.add(i);
        }
        for (int i = 0; i < n; i++) {
            al.get(i);  // Fast
        }
        System.out.println("ArrayList access: " + 
            (System.currentTimeMillis() - start));
        
        // LinkedList - Slow random access
        List<Integer> llist = new LinkedList<>();
        start = System.currentTimeMillis();
        for (int i = 0; i < n; i++) {
            llist.add(i);
        }
        for (int i = 0; i < n; i++) {
            llist.get(i);  // Slow - O(n) for each access
        }
        System.out.println("LinkedList access: " + 
            (System.currentTimeMillis() - start));
    }
}
```

**When to Use:**
- **ArrayList**: When you need frequent random access, read-heavy operations
- **LinkedList**: When you need frequent insertions/deletions, especially at beginning/end

---

### Q12: Explain HashMap internal working.

**Answer:**

**HashMap Structure:**

```
HashMap Internal Structure:

Array of Buckets (Node<K,V>[] table)
┌─────┬─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │  4  │ ... 15 (default size 16)
└─────┴─────┴─────┴─────┴─────┘
   │     │     │     │
   ▼     │     ▼     ▼
 Node    │   Node  Node
  │      │     │     │
  ▼      │     ▼     ▼
 Node    │   Node  Node
         │
         ▼
       Node

Each Node contains:
- int hash
- K key
- V value
- Node<K,V> next
```

**How HashMap Works:**

```java
public class HashMapInternals {
    
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<>();
        
        // put() operation
        map.put("John", 25);
        
        /*
        Internal process:
        1. Calculate hash: hash(key) = key.hashCode()
        2. Calculate index: index = hash & (n-1)  [where n = array length]
        3. Check if bucket is empty:
           - Yes: Create new node and place it
           - No: Check for collision
        4. Handle collision:
           - Check if key already exists (using equals())
             - Yes: Replace value
             - No: Add to linked list / tree
        5. If size > threshold, resize (double the capacity)
        */
    }
}

// Simplified HashMap implementation
class SimpleHashMap<K, V> {
    
    static class Node<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;
        
        Node(int hash, K key, V value, Node<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }
    
    private Node<K, V>[] table;
    private int size;
    private static final int DEFAULT_CAPACITY = 16;
    private static final float LOAD_FACTOR = 0.75f;
    
    @SuppressWarnings("unchecked")
    public SimpleHashMap() {
        table = (Node<K, V>[]) new Node[DEFAULT_CAPACITY];
    }
    
    // Calculate hash
    private int hash(K key) {
        return key == null ? 0 : key.hashCode();
    }
    
    // Calculate index
    private int indexFor(int hash, int length) {
        return hash & (length - 1);  // Equivalent to hash % length
    }
    
    // Put operation
    public V put(K key, V value) {
        int hash = hash(key);
        int index = indexFor(hash, table.length);
        
        // Check if key exists
        for (Node<K, V> node = table[index]; node != null; node = node.next) {
            if (node.hash == hash && 
                (key == node.key || (key != null && key.equals(node.key)))) {
                V oldValue = node.value;
                node.value = value;  // Replace value
                return oldValue;
            }
        }
        
        // Add new node
        addNode(hash, key, value, index);
        return null;
    }
    
    private void addNode(int hash, K key, V value, int index) {
        Node<K, V> newNode = new Node<>(hash, key, value, table[index]);
        table[index] = newNode;
        size++;
        
        // Check if resize needed
        if (size > table.length * LOAD_FACTOR) {
            resize();
        }
    }
    
    // Get operation
    public V get(K key) {
        int hash = hash(key);
        int index = indexFor(hash, table.length);
        
        for (Node<K, V> node = table[index]; node != null; node = node.next) {
            if (node.hash == hash && 
                (key == node.key || (key != null && key.equals(node.key)))) {
                return node.value;
            }
        }
        return null;
    }
    
    @SuppressWarnings("unchecked")
    private void resize() {
        Node<K, V>[] oldTable = table;
        table = (Node<K, V>[]) new Node[oldTable.length * 2];
        size = 0;
        
        // Rehash all nodes
        for (Node<K, V> node : oldTable) {
            while (node != null) {
                put(node.key, node.value);
                node = node.next;
            }
        }
    }
}
```

**Key Points:**

1. **Hash Function**: Converts key to integer
2. **Index Calculation**: `index = hash & (n-1)` where n is array length
3. **Collision Handling**:
   - Java 7: Linked List
   - Java 8+: Linked List (< 8 elements) → Balanced Tree (>= 8 elements)
4. **Load Factor**: 0.75 (resize when 75% full)
5. **Initial Capacity**: 16
6. **Thread Safety**: Not thread-safe (use ConcurrentHashMap)

**Important Methods:**

```java
public class HashMapMethods {
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<>();
        
        // Basic operations
        map.put("A", 1);
        map.put("B", 2);
        map.putIfAbsent("C", 3);  // Only if key doesn't exist
        
        Integer value = map.get("A");
        Integer defaultValue = map.getOrDefault("D", 0);
        
        boolean hasKey = map.containsKey("A");
        boolean hasValue = map.containsValue(1);
        
        map.remove("B");
        
        // Java 8+ methods
        map.compute("A", (k, v) -> v + 10);  // A = 11
        map.computeIfAbsent("D", k -> 4);     // D = 4
        map.computeIfPresent("A", (k, v) -> v + 1);  // A = 12
        
        map.merge("A", 5, (oldVal, newVal) -> oldVal + newVal);  // A = 17
        
        // Iteration
        for (Map.Entry<String, Integer> entry : map.entrySet()) {
            System.out.println(entry.getKey() + " = " + entry.getValue());
        }
        
        map.forEach((k, v) -> System.out.println(k + " = " + v));
    }
}
```

---

### Q13: What is the difference between HashMap and ConcurrentHashMap?

**Answer:**

| Feature | HashMap | ConcurrentHashMap |
|---------|---------|-------------------|
| Thread Safety | Not thread-safe | Thread-safe |
| Null Keys | 1 null key allowed | No null keys |
| Null Values | Null values allowed | No null values |
| Performance | Faster | Slower (due to locks) |
| Locking | No locking | Segment/bucket level locking |
| Iteration | Fail-fast | Fail-safe |

```java
public class HashMapVsConcurrentHashMap {
    
    // HashMap - Not thread-safe
    public static void unsafeExample() {
        Map<String, Integer> map = new HashMap<>();
        
        // Problem: Race condition
        Runnable task = () -> {
            for (int i = 0; i < 1000; i++) {
                map.put(Thread.currentThread().getName() + "-" + i, i);
            }
        };
        
        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        
        t1.start();
        t2.start();
        
        try {
            t1.join();
            t2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        System.out.println("Size: " + map.size());  // May not be 2000
        // Possible issues: data loss, infinite loop, exception
    }
    
    // ConcurrentHashMap - Thread-safe
    public static void safeExample() {
        Map<String, Integer> map = new ConcurrentHashMap<>();
        
        Runnable task = () -> {
            for (int i = 0; i < 1000; i++) {
                map.put(Thread.currentThread().getName() + "-" + i, i);
            }
        };
        
        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        
        t1.start();
        t2.start();
        
        try {
            t1.join();
            t2.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        System.out.println("Size: " + map.size());  // Always 2000
    }
    
    // Synchronized HashMap (Alternative but not recommended)
    public static void synchronizedMapExample() {
        Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
        
        // Still need external synchronization for iteration
        synchronized (syncMap) {
            for (Map.Entry<String, Integer> entry : syncMap.entrySet()) {
                System.out.println(entry.getKey() + " = " + entry.getValue());
            }
        }
    }
    
    public static void main(String[] args) {
        System.out.println("HashMap (Unsafe):");
        unsafeExample();
        
        System.out.println("\nConcurrentHashMap (Safe):");
        safeExample();
    }
}

// ConcurrentHashMap advanced features
class ConcurrentHashMapFeatures {
    public static void main(String[] args) {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        
        map.put("A", 1);
        map.put("B", 2);
        map.put("C", 3);
        
        // Atomic operations
        map.putIfAbsent("D", 4);
        map.computeIfAbsent("E", k -> 5);
        map.computeIfPresent("A", (k, v) -> v + 10);
        
        // Bulk operations
        map.forEach(1, (k, v) -> System.out.println(k + " = " + v));
        
        // Search
        String result = map.search(1, (k, v) -> v > 5 ? k : null);
        
        // Reduce
        Integer sum = map.reduce(1, (k, v) -> v, (v1, v2) -> v1 + v2);
        
        System.out.println("Sum: " + sum);
    }
}
```

**Internal Structure (Java 8+):**

```
ConcurrentHashMap uses:
- Array of Nodes (like HashMap)
- CAS (Compare-And-Swap) operations
- Synchronized blocks for specific buckets
- No segment locking (unlike Java 7)

Locking Strategy:
- Lock only the bucket being modified
- Other buckets remain accessible
- Much better concurrency than Hashtable
```

---

Due to the length limit, let me continue with more questions in the next part. Let me save this first part and continue.


## 3. Multithreading & Concurrency Questions

### Q14: Explain the lifecycle of a thread.

**Answer:**

```
Thread Lifecycle States:

NEW → RUNNABLE → RUNNING → TERMINATED
        ↕          ↕
     WAITING   TIMED_WAITING
        ↕
     BLOCKED
```

```java
public class ThreadLifecycle {
    
    public static void main(String[] args) throws InterruptedException {
        
        Thread thread = new Thread(() -> {
            System.out.println("Thread started");
            
            try {
                Thread.sleep(2000);  // TIMED_WAITING
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            
            synchronized (ThreadLifecycle.class) {
                System.out.println("In synchronized block");
            }
            
            System.out.println("Thread ending");
        });
        
        System.out.println("State after creation: " + thread.getState());  // NEW
        
        thread.start();
        System.out.println("State after start: " + thread.getState());  // RUNNABLE
        
        Thread.sleep(100);
        System.out.println("State during sleep: " + thread.getState());  // TIMED_WAITING
        
        thread.join();  // Wait for thread to complete
        System.out.println("State after completion: " + thread.getState());  // TERMINATED
    }
}

// Detailed state transitions
class ThreadStates {
    
    public static void demonstrateBlocked() {
        Object lock = new Object();
        
        Thread t1 = new Thread(() -> {
            synchronized (lock) {
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        Thread t2 = new Thread(() -> {
            synchronized (lock) {  // Will be BLOCKED waiting for lock
                System.out.println("T2 got lock");
            }
        });
        
        t1.start();
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        t2.start();
        
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("T2 State: " + t2.getState());  // BLOCKED
    }
    
    public static void demonstrateWaiting() {
        Object lock = new Object();
        
        Thread t1 = new Thread(() -> {
            synchronized (lock) {
                try {
                    lock.wait();  // WAITING
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        t1.start();
        
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("T1 State: " + t1.getState());  // WAITING
        
        synchronized (lock) {
            lock.notify();
        }
    }
}
```

---

### Q15: What is the difference between synchronized method and synchronized block?

**Answer:**

```java
public class SynchronizationExample {
    
    private int count = 0;
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();
    
    // Synchronized method - locks entire method
    public synchronized void incrementMethod() {
        count++;  // Only one thread can execute entire method
    }
    
    // Synchronized block - locks specific section
    public void incrementBlock() {
        // Non-critical code here (can be accessed by multiple threads)
        System.out.println("Before critical section");
        
        synchronized (this) {
            count++;  // Only this part is synchronized
        }
        
        // Non-critical code here
        System.out.println("After critical section");
    }
    
    // Multiple locks for better concurrency
    public void method1() {
        synchronized (lock1) {
            // Operations on resource 1
        }
    }
    
    public void method2() {
        synchronized (lock2) {
            // Operations on resource 2
            // Can run concurrently with method1
        }
    }
    
    // Static synchronized method
    public static synchronized void staticMethod() {
        // Locks on Class object
    }
    
    // Equivalent to static synchronized method
    public static void staticMethodBlock() {
        synchronized (SynchronizationExample.class) {
            // Same as above
        }
    }
}

// Comparison
class Comparison {
    private int count = 0;
    
    // Method 1: Synchronized method (Less granular)
    public synchronized void method1() {
        // All code is synchronized
        doSomething();
        count++;
        doSomethingElse();
    }
    
    // Method 2: Synchronized block (More granular, better performance)
    public void method2() {
        doSomething();  // Can be concurrent
        
        synchronized (this) {
            count++;  // Only critical section is synchronized
        }
        
        doSomethingElse();  // Can be concurrent
    }
    
    private void doSomething() {
        // Non-critical operation
    }
    
    private void doSomethingElse() {
        // Non-critical operation
    }
}
```

**Key Differences:**

| Synchronized Method | Synchronized Block |
|-------------------|-------------------|
| Locks entire method | Locks specific code block |
| Locks on 'this' object | Can lock on any object |
| Less flexible | More flexible |
| Lower performance | Better performance |
| Simpler syntax | More verbose |

---

### Q16: Explain Producer-Consumer problem and its solution.

**Answer:**

```java
// Using wait() and notify()
class ProducerConsumerWaitNotify {
    
    static class SharedQueue {
        private Queue<Integer> queue = new LinkedList<>();
        private int capacity;
        
        public SharedQueue(int capacity) {
            this.capacity = capacity;
        }
        
        public synchronized void produce(int item) throws InterruptedException {
            while (queue.size() == capacity) {
                System.out.println("Queue full, producer waiting...");
                wait();  // Release lock and wait
            }
            
            queue.add(item);
            System.out.println("Produced: " + item);
            
            notifyAll();  // Notify waiting consumers
        }
        
        public synchronized int consume() throws InterruptedException {
            while (queue.isEmpty()) {
                System.out.println("Queue empty, consumer waiting...");
                wait();  // Release lock and wait
            }
            
            int item = queue.poll();
            System.out.println("Consumed: " + item);
            
            notifyAll();  // Notify waiting producers
            return item;
        }
    }
    
    public static void main(String[] args) {
        SharedQueue queue = new SharedQueue(5);
        
        // Producer thread
        Thread producer = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    queue.produce(i);
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        // Consumer thread
        Thread consumer = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    queue.consume();
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        producer.start();
        consumer.start();
    }
}

// Using BlockingQueue (Preferred approach)
class ProducerConsumerBlockingQueue {
    
    public static void main(String[] args) {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);
        
        // Producer
        Thread producer = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    queue.put(i);  // Blocks if queue is full
                    System.out.println("Produced: " + i);
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        // Consumer
        Thread consumer = new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    int item = queue.take();  // Blocks if queue is empty
                    System.out.println("Consumed: " + item);
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        
        producer.start();
        consumer.start();
    }
}

// Using Semaphore
class ProducerConsumerSemaphore {
    
    static class SharedQueue {
        private Queue<Integer> queue = new LinkedList<>();
        private Semaphore mutex = new Semaphore(1);
        private Semaphore empty;
        private Semaphore full;
        
        public SharedQueue(int capacity) {
            empty = new Semaphore(capacity);
            full = new Semaphore(0);
        }
        
        public void produce(int item) throws InterruptedException {
            empty.acquire();  // Wait if no empty slots
            mutex.acquire();  // Lock
            
            queue.add(item);
            System.out.println("Produced: " + item);
            
            mutex.release();  // Unlock
            full.release();   // Signal full
        }
        
        public int consume() throws InterruptedException {
            full.acquire();   // Wait if queue empty
            mutex.acquire();  // Lock
            
            int item = queue.poll();
            System.out.println("Consumed: " + item);
            
            mutex.release();  // Unlock
            empty.release();  // Signal empty
            
            return item;
        }
    }
}
```

---

## 4. Java 8+ Features Questions

### Q17: Explain Stream API and its advantages.

**Answer:**

**Stream API**: Provides functional-style operations on sequences of elements.

**Advantages:**
1. **Declarative**: What to do, not how
2. **Functional**: No side effects
3. **Lazy Evaluation**: Operations evaluated only when needed
4. **Parallelizable**: Easy parallel processing
5. **Readable**: More expressive code

```java
public class StreamAPIExample {
    
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // Traditional approach
        List<Integer> evenSquaresOld = new ArrayList<>();
        for (Integer num : numbers) {
            if (num % 2 == 0) {
                int square = num * num;
                evenSquaresOld.add(square);
            }
        }
        
        // Stream approach
        List<Integer> evenSquares = numbers.stream()
            .filter(n -> n % 2 == 0)
            .map(n -> n * n)
            .collect(Collectors.toList());
        
        System.out.println("Even squares: " + evenSquares);
        
        // Common stream operations
        
        // 1. Filter
        List<Integer> filtered = numbers.stream()
            .filter(n -> n > 5)
            .collect(Collectors.toList());
        
        // 2. Map
        List<String> strings = numbers.stream()
            .map(n -> "Number: " + n)
            .collect(Collectors.toList());
        
        // 3. FlatMap
        List<List<Integer>> nested = Arrays.asList(
            Arrays.asList(1, 2),
            Arrays.asList(3, 4),
            Arrays.asList(5, 6)
        );
        List<Integer> flattened = nested.stream()
            .flatMap(list -> list.stream())
            .collect(Collectors.toList());
        
        // 4. Reduce
        int sum = numbers.stream()
            .reduce(0, (a, b) -> a + b);
        
        Optional<Integer> max = numbers.stream()
            .reduce((a, b) -> a > b ? a : b);
        
        // 5. Collect
        Set<Integer> set = numbers.stream()
            .collect(Collectors.toSet());
        
        Map<Boolean, List<Integer>> partitioned = numbers.stream()
            .collect(Collectors.partitioningBy(n -> n % 2 == 0));
        
        // 6. Sorting
        List<Integer> sorted = numbers.stream()
            .sorted()
            .collect(Collectors.toList());
        
        List<Integer> reverseSorted = numbers.stream()
            .sorted(Comparator.reverseOrder())
            .collect(Collectors.toList());
        
        // 7. Distinct
        List<Integer> distinct = Arrays.asList(1, 2, 2, 3, 3, 4).stream()
            .distinct()
            .collect(Collectors.toList());
        
        // 8. Limit and Skip
        List<Integer> limited = numbers.stream()
            .limit(5)
            .collect(Collectors.toList());
        
        List<Integer> skipped = numbers.stream()
            .skip(5)
            .collect(Collectors.toList());
        
        // 9. Match operations
        boolean anyEven = numbers.stream().anyMatch(n -> n % 2 == 0);
        boolean allPositive = numbers.stream().allMatch(n -> n > 0);
        boolean noneNegative = numbers.stream().noneMatch(n -> n < 0);
        
        // 10. Find operations
        Optional<Integer> first = numbers.stream()
            .filter(n -> n > 5)
            .findFirst();
        
        Optional<Integer> any = numbers.stream()
            .filter(n -> n > 5)
            .findAny();
        
        // Parallel stream
        long count = numbers.parallelStream()
            .filter(n -> n % 2 == 0)
            .count();
        
        // Complex example
        List<Employee> employees = Arrays.asList(
            new Employee("John", 30, 50000),
            new Employee("Jane", 25, 60000),
            new Employee("Bob", 35, 55000),
            new Employee("Alice", 28, 65000)
        );
        
        // Get names of employees with salary > 55000, sorted by age
        List<String> highEarners = employees.stream()
            .filter(e -> e.getSalary() > 55000)
            .sorted(Comparator.comparing(Employee::getAge))
            .map(Employee::getName)
            .collect(Collectors.toList());
        
        System.out.println("High earners: " + highEarners);
        
        // Group by age
        Map<Integer, List<Employee>> byAge = employees.stream()
            .collect(Collectors.groupingBy(Employee::getAge));
        
        // Average salary by age group
        Map<Integer, Double> avgSalaryByAge = employees.stream()
            .collect(Collectors.groupingBy(
                Employee::getAge,
                Collectors.averagingDouble(Employee::getSalary)
            ));
    }
}

class Employee {
    private String name;
    private int age;
    private double salary;
    
    public Employee(String name, int age, double salary) {
        this.name = name;
        this.age = age;
        this.salary = salary;
    }
    
    public String getName() { return name; }
    public int getAge() { return age; }
    public double getSalary() { return salary; }
}
```

**Intermediate vs Terminal Operations:**

| Intermediate | Terminal |
|-------------|----------|
| filter, map, flatMap | collect, forEach |
| sorted, distinct | reduce, count |
| limit, skip | anyMatch, allMatch |
| peek | findFirst, findAny |
| Returns Stream | Returns result |
| Lazy evaluation | Triggers execution |

---

### Q18: What is Optional and how to use it?

**Answer:**

**Optional**: Container object that may or may not contain a non-null value. Helps avoid NullPointerException.

```java
public class OptionalExample {
    
    public static void main(String[] args) {
        
        // Creating Optional
        Optional<String> empty = Optional.empty();
        Optional<String> nonEmpty = Optional.of("Hello");
        Optional<String> nullable = Optional.ofNullable(null);
        
        // isPresent() and isEmpty()
        if (nonEmpty.isPresent()) {
            System.out.println("Value: " + nonEmpty.get());
        }
        
        if (empty.isEmpty()) {
            System.out.println("No value present");
        }
        
        // orElse() - provide default value
        String value1 = empty.orElse("Default");
        System.out.println(value1);  // Default
        
        // orElseGet() - provide Supplier
        String value2 = empty.orElseGet(() -> "Generated Default");
        
        // orElseThrow() - throw exception if empty
        try {
            String value3 = empty.orElseThrow(() -> 
                new IllegalArgumentException("No value present"));
        } catch (Exception e) {
            System.out.println(e.getMessage());
        }
        
        // ifPresent() - execute if value present
        nonEmpty.ifPresent(val -> System.out.println("Value: " + val));
        
        // ifPresentOrElse() - execute one of two actions
        empty.ifPresentOrElse(
            val -> System.out.println("Value: " + val),
            () -> System.out.println("No value")
        );
        
        // map() - transform value if present
        Optional<Integer> length = nonEmpty.map(String::length);
        System.out.println("Length: " + length.orElse(0));
        
        // flatMap() - transform to Optional
        Optional<String> upper = nonEmpty.flatMap(s -> 
            Optional.of(s.toUpperCase()));
        
        // filter() - filter based on predicate
        Optional<String> filtered = nonEmpty.filter(s -> s.length() > 3);
        
        // or() - alternative Optional
        Optional<String> alternative = empty.or(() -> 
            Optional.of("Alternative"));
        
        // Practical examples
        
        // Example 1: User lookup
        Optional<User> user = findUserById(1L);
        
        // Old way
        if (user.isPresent()) {
            System.out.println(user.get().getName());
        }
        
        // Better way
        user.ifPresent(u -> System.out.println(u.getName()));
        
        // Get name or default
        String userName = user
            .map(User::getName)
            .orElse("Unknown");
        
        // Example 2: Nested Optional handling
        Optional<String> email = user
            .map(User::getAddress)
            .map(Address::getEmail)
            .orElse(null);
        
        // Example 3: Stream with Optional
        List<Optional<String>> optionals = Arrays.asList(
            Optional.of("A"),
            Optional.empty(),
            Optional.of("B"),
            Optional.empty(),
            Optional.of("C")
        );
        
        List<String> values = optionals.stream()
            .filter(Optional::isPresent)
            .map(Optional::get)
            .collect(Collectors.toList());
        
        // Or using flatMap
        List<String> values2 = optionals.stream()
            .flatMap(Optional::stream)  // Java 9+
            .collect(Collectors.toList());
    }
    
    static Optional<User> findUserById(Long id) {
        // Simulate database lookup
        if (id == 1L) {
            return Optional.of(new User("John", new Address("john@example.com")));
        }
        return Optional.empty();
    }
}

class User {
    private String name;
    private Address address;
    
    public User(String name, Address address) {
        this.name = name;
        this.address = address;
    }
    
    public String getName() { return name; }
    public Address getAddress() { return address; }
}

class Address {
    private String email;
    
    public Address(String email) {
        this.email = email;
    }
    
    public String getEmail() { return email; }
}
```

**Anti-patterns to Avoid:**

```java
class OptionalAntiPatterns {
    
    // DON'T: Use Optional.get() without checking
    public void bad1(Optional<String> opt) {
        String value = opt.get();  // May throw NoSuchElementException
    }
    
    // DO: Use orElse, orElseGet, or check first
    public void good1(Optional<String> opt) {
        String value = opt.orElse("Default");
    }
    
    // DON'T: Use Optional for fields
    class BadClass {
        private Optional<String> name;  // Bad practice
    }
    
    // DO: Use Optional only for return types
    public Optional<String> getName() {
        return Optional.ofNullable(name);
    }
    
    // DON'T: Use Optional.of with potentially null value
    public void bad2(String value) {
        Optional<String> opt = Optional.of(value);  // NPE if value is null
    }
    
    // DO: Use Optional.ofNullable
    public void good2(String value) {
        Optional<String> opt = Optional.ofNullable(value);
    }
}
```

---

## 5. Spring Framework Questions

### Q19: Explain Dependency Injection and different types.

**Answer:**

**Dependency Injection (DI)**: Design pattern where objects receive their dependencies from external source rather than creating them.

**Types of DI:**

```java
// 1. Constructor Injection (Recommended)
@Service
public class UserService {
    
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    @Autowired  // Optional in Spring 4.3+ if only one constructor
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
    
    public void createUser(User user) {
        userRepository.save(user);
        emailService.sendWelcomeEmail(user);
    }
}

// 2. Setter Injection
@Service
public class ProductService {
    
    private ProductRepository productRepository;
    
    @Autowired
    public void setProductRepository(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }
}

// 3. Field Injection (Not Recommended)
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;  // Hard to test, no immutability
}

// Why Constructor Injection is Preferred:
// 1. Immutability - fields can be final
// 2. Testability - easy to mock dependencies
// 3. Required dependencies - compile-time check
// 4. No reflection - Spring can use regular constructor

// Example with multiple implementations
interface PaymentService {
    void pay(double amount);
}

@Service("creditCardPayment")
class CreditCardPayment implements PaymentService {
    @Override
    public void pay(double amount) {
        System.out.println("Paid with credit card: " + amount);
    }
}

@Service("paypalPayment")
class PayPalPayment implements PaymentService {
    @Override
    public void pay(double amount) {
        System.out.println("Paid with PayPal: " + amount);
    }
}

// Using @Qualifier
@Service
public class CheckoutService {
    
    private final PaymentService paymentService;
    
    public CheckoutService(@Qualifier("creditCardPayment") PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}

// Using @Primary
@Service
@Primary
class DefaultPayment implements PaymentService {
    @Override
    public void pay(double amount) {
        System.out.println("Default payment: " + amount);
    }
}

// Using List injection
@Service
public class PaymentProcessor {
    
    private final List<PaymentService> paymentServices;
    
    public PaymentProcessor(List<PaymentService> paymentServices) {
        this.paymentServices = paymentServices;
    }
    
    public void processAll(double amount) {
        paymentServices.forEach(service -> service.pay(amount));
    }
}
```

---

### Q20: What is the difference between @Component, @Service, @Repository, and @Controller?

**Answer:**

All are stereotype annotations, but serve different purposes:

```java
// @Component - Generic stereotype
@Component
public class GenericComponent {
    // Any Spring-managed component
}

// @Service - Business logic layer
@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User createUser(User user) {
        // Business logic
        validateUser(user);
        return userRepository.save(user);
    }
    
    private void validateUser(User user) {
        // Validation logic
    }
}

// @Repository - Data access layer
@Repository
public class UserRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    public User findById(Long id) {
        return entityManager.find(User.class, id);
    }
    
    public void save(User user) {
        entityManager.persist(user);
    }
}

// Special feature of @Repository: Exception Translation
// Converts database exceptions to Spring's DataAccessException

// @Controller - Presentation layer (MVC)
@Controller
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/users")
    public String getUsers(Model model) {
        model.addAttribute("users", userService.getAllUsers());
        return "users";  // Returns view name
    }
}

// @RestController - REST API controller
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping
    public List<User> getAllUsers() {
        return userService.getAllUsers();  // Returns JSON
    }
}

// @Configuration - Configuration class
@Configuration
public class AppConfig {
    
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}
```

**Summary:**

| Annotation | Layer | Purpose |
|-----------|-------|---------|
| @Component | Any | Generic component |
| @Service | Business | Business logic |
| @Repository | Data | Data access + exception translation |
| @Controller | Presentation | MVC controller |
| @RestController | API | REST controller (@Controller + @ResponseBody) |
| @Configuration | Config | Java configuration |

---

Now let me continue with the LLD problems section:


---

# Part B: Low-Level Design Problems

## 11. LLD Problems with Solutions

### LLD Problem 1: Design a Parking Lot System

**Requirements:**
1. Multiple floors with parking spots
2. Different vehicle types (Car, Bike, Truck)
3. Different spot sizes (Compact, Large, Handicapped)
4. Entry/Exit gates
5. Payment calculation
6. Display available spots

**Solution:**

```java
// Enums
enum VehicleType {
    CAR, BIKE, TRUCK
}

enum SpotType {
    COMPACT, LARGE, HANDICAPPED, BIKE_SPOT
}

enum ParkingSpotStatus {
    AVAILABLE, OCCUPIED
}

// Vehicle classes
abstract class Vehicle {
    protected String licensePlate;
    protected VehicleType type;
    
    public Vehicle(String licensePlate, VehicleType type) {
        this.licensePlate = licensePlate;
        this.type = type;
    }
    
    public String getLicensePlate() { return licensePlate; }
    public VehicleType getType() { return type; }
}

class Car extends Vehicle {
    public Car(String licensePlate) {
        super(licensePlate, VehicleType.CAR);
    }
}

class Bike extends Vehicle {
    public Bike(String licensePlate) {
        super(licensePlate, VehicleType.BIKE);
    }
}

class Truck extends Vehicle {
    public Truck(String licensePlate) {
        super(licensePlate, VehicleType.TRUCK);
    }
}

// Parking Spot
class ParkingSpot {
    private int id;
    private SpotType type;
    private ParkingSpotStatus status;
    private Vehicle vehicle;
    
    public ParkingSpot(int id, SpotType type) {
        this.id = id;
        this.type = type;
        this.status = ParkingSpotStatus.AVAILABLE;
    }
    
    public boolean isAvailable() {
        return status == ParkingSpotStatus.AVAILABLE;
    }
    
    public boolean canFitVehicle(Vehicle vehicle) {
        switch (type) {
            case COMPACT:
                return vehicle.getType() == VehicleType.CAR;
            case LARGE:
                return vehicle.getType() == VehicleType.CAR || 
                       vehicle.getType() == VehicleType.TRUCK;
            case BIKE_SPOT:
                return vehicle.getType() == VehicleType.BIKE;
            case HANDICAPPED:
                return vehicle.getType() == VehicleType.CAR;
            default:
                return false;
        }
    }
    
    public void parkVehicle(Vehicle vehicle) {
        this.vehicle = vehicle;
        this.status = ParkingSpotStatus.OCCUPIED;
    }
    
    public void removeVehicle() {
        this.vehicle = null;
        this.status = ParkingSpotStatus.AVAILABLE;
    }
    
    public int getId() { return id; }
    public SpotType getType() { return type; }
    public Vehicle getVehicle() { return vehicle; }
}

// Parking Floor
class ParkingFloor {
    private int floorNumber;
    private List<ParkingSpot> spots;
    
    public ParkingFloor(int floorNumber) {
        this.floorNumber = floorNumber;
        this.spots = new ArrayList<>();
    }
    
    public void addSpot(ParkingSpot spot) {
        spots.add(spot);
    }
    
    public ParkingSpot findAvailableSpot(Vehicle vehicle) {
        for (ParkingSpot spot : spots) {
            if (spot.isAvailable() && spot.canFitVehicle(vehicle)) {
                return spot;
            }
        }
        return null;
    }
    
    public int getAvailableSpots(SpotType type) {
        return (int) spots.stream()
            .filter(spot -> spot.isAvailable() && spot.getType() == type)
            .count();
    }
    
    public int getFloorNumber() { return floorNumber; }
}

// Parking Ticket
class ParkingTicket {
    private String ticketId;
    private Vehicle vehicle;
    private ParkingSpot spot;
    private Date entryTime;
    private Date exitTime;
    private double amount;
    
    public ParkingTicket(String ticketId, Vehicle vehicle, ParkingSpot spot) {
        this.ticketId = ticketId;
        this.vehicle = vehicle;
        this.spot = spot;
        this.entryTime = new Date();
    }
    
    public void setExitTime(Date exitTime) {
        this.exitTime = exitTime;
    }
    
    public void setAmount(double amount) {
        this.amount = amount;
    }
    
    public long getParkingDurationInHours() {
        if (exitTime == null) {
            exitTime = new Date();
        }
        long diff = exitTime.getTime() - entryTime.getTime();
        return TimeUnit.MILLISECONDS.toHours(diff) + 1;
    }
    
    public String getTicketId() { return ticketId; }
    public Vehicle getVehicle() { return vehicle; }
    public ParkingSpot getSpot() { return spot; }
    public double getAmount() { return amount; }
}

// Parking Rate
class ParkingRate {
    private Map<VehicleType, Double> hourlyRates;
    
    public ParkingRate() {
        hourlyRates = new HashMap<>();
        hourlyRates.put(VehicleType.BIKE, 10.0);
        hourlyRates.put(VehicleType.CAR, 20.0);
        hourlyRates.put(VehicleType.TRUCK, 30.0);
    }
    
    public double calculateAmount(Vehicle vehicle, long hours) {
        double hourlyRate = hourlyRates.getOrDefault(vehicle.getType(), 20.0);
        return hourlyRate * hours;
    }
}

// Parking Lot (Singleton)
class ParkingLot {
    private static ParkingLot instance;
    private String name;
    private List<ParkingFloor> floors;
    private Map<String, ParkingTicket> activeTickets;
    private ParkingRate parkingRate;
    
    private ParkingLot(String name) {
        this.name = name;
        this.floors = new ArrayList<>();
        this.activeTickets = new HashMap<>();
        this.parkingRate = new ParkingRate();
    }
    
    public static synchronized ParkingLot getInstance(String name) {
        if (instance == null) {
            instance = new ParkingLot(name);
        }
        return instance;
    }
    
    public void addFloor(ParkingFloor floor) {
        floors.add(floor);
    }
    
    public ParkingTicket parkVehicle(Vehicle vehicle) {
        ParkingSpot spot = findAvailableSpot(vehicle);
        
        if (spot == null) {
            System.out.println("No available spot for vehicle: " + vehicle.getLicensePlate());
            return null;
        }
        
        spot.parkVehicle(vehicle);
        
        String ticketId = "TICKET-" + System.currentTimeMillis();
        ParkingTicket ticket = new ParkingTicket(ticketId, vehicle, spot);
        activeTickets.put(ticketId, ticket);
        
        System.out.println("Vehicle parked successfully. Ticket: " + ticketId);
        return ticket;
    }
    
    public double unparkVehicle(String ticketId) {
        ParkingTicket ticket = activeTickets.get(ticketId);
        
        if (ticket == null) {
            System.out.println("Invalid ticket");
            return 0;
        }
        
        ticket.setExitTime(new Date());
        long hours = ticket.getParkingDurationInHours();
        double amount = parkingRate.calculateAmount(ticket.getVehicle(), hours);
        ticket.setAmount(amount);
        
        ticket.getSpot().removeVehicle();
        activeTickets.remove(ticketId);
        
        System.out.println("Amount to pay: $" + amount);
        return amount;
    }
    
    private ParkingSpot findAvailableSpot(Vehicle vehicle) {
        for (ParkingFloor floor : floors) {
            ParkingSpot spot = floor.findAvailableSpot(vehicle);
            if (spot != null) {
                return spot;
            }
        }
        return null;
    }
    
    public void displayAvailability() {
        System.out.println("\nParking Lot Availability:");
        for (ParkingFloor floor : floors) {
            System.out.println("Floor " + floor.getFloorNumber() + ":");
            System.out.println("  Compact: " + floor.getAvailableSpots(SpotType.COMPACT));
            System.out.println("  Large: " + floor.getAvailableSpots(SpotType.LARGE));
            System.out.println("  Bike: " + floor.getAvailableSpots(SpotType.BIKE_SPOT));
        }
    }
}

// Demo
public class ParkingLotDemo {
    public static void main(String[] args) {
        ParkingLot parkingLot = ParkingLot.getInstance("City Center Parking");
        
        // Create floors
        ParkingFloor floor1 = new ParkingFloor(1);
        for (int i = 1; i <= 10; i++) {
            floor1.addSpot(new ParkingSpot(i, SpotType.COMPACT));
        }
        for (int i = 11; i <= 15; i++) {
            floor1.addSpot(new ParkingSpot(i, SpotType.LARGE));
        }
        for (int i = 16; i <= 20; i++) {
            floor1.addSpot(new ParkingSpot(i, SpotType.BIKE_SPOT));
        }
        
        parkingLot.addFloor(floor1);
        
        // Park vehicles
        Vehicle car1 = new Car("ABC-123");
        Vehicle bike1 = new Bike("XYZ-789");
        Vehicle truck1 = new Truck("TRK-456");
        
        ParkingTicket ticket1 = parkingLot.parkVehicle(car1);
        ParkingTicket ticket2 = parkingLot.parkVehicle(bike1);
        ParkingTicket ticket3 = parkingLot.parkVehicle(truck1);
        
        parkingLot.displayAvailability();
        
        // Unpark vehicles
        if (ticket1 != null) {
            double amount = parkingLot.unparkVehicle(ticket1.getTicketId());
        }
        
        parkingLot.displayAvailability();
    }
}
```

---

### LLD Problem 2: Design an Online Library Management System

**Requirements:**
1. Books with ISBN, title, author
2. Members can borrow and return books
3. Librarian can add/remove books
4. Search books by title, author, ISBN
5. Check book availability
6. Fine calculation for late returns

**Solution:**

```java
// Book class
class Book {
    private String ISBN;
    private String title;
    private String author;
    private String publisher;
    private Date publicationDate;
    private int numberOfPages;
    private BookStatus status;
    
    public Book(String ISBN, String title, String author, String publisher) {
        this.ISBN = ISBN;
        this.title = title;
        this.author = author;
        this.publisher = publisher;
        this.status = BookStatus.AVAILABLE;
    }
    
    public String getISBN() { return ISBN; }
    public String getTitle() { return title; }
    public String getAuthor() { return author; }
    public BookStatus getStatus() { return status; }
    
    public void setStatus(BookStatus status) {
        this.status = status;
    }
}

enum BookStatus {
    AVAILABLE, RESERVED, LOANED, LOST
}

// User classes
abstract class User {
    protected String userId;
    protected String name;
    protected String email;
    protected String phone;
    
    public User(String userId, String name, String email, String phone) {
        this.userId = userId;
        this.name = name;
        this.email = email;
        this.phone = phone;
    }
    
    public String getUserId() { return userId; }
    public String getName() { return name; }
}

class Member extends User {
    private List<BookLoan> currentLoans;
    private double totalFine;
    private int maxBooksAllowed = 5;
    
    public Member(String userId, String name, String email, String phone) {
        super(userId, name, email, phone);
        this.currentLoans = new ArrayList<>();
        this.totalFine = 0.0;
    }
    
    public boolean canBorrowBook() {
        return currentLoans.size() < maxBooksAllowed && totalFine == 0;
    }
    
    public void addLoan(BookLoan loan) {
        currentLoans.add(loan);
    }
    
    public void removeLoan(BookLoan loan) {
        currentLoans.remove(loan);
    }
    
    public void addFine(double fine) {
        totalFine += fine;
    }
    
    public void payFine(double amount) {
        totalFine -= amount;
        if (totalFine < 0) totalFine = 0;
    }
    
    public List<BookLoan> getCurrentLoans() { return currentLoans; }
    public double getTotalFine() { return totalFine; }
}

class Librarian extends User {
    private String employeeId;
    
    public Librarian(String userId, String name, String email, String phone, String employeeId) {
        super(userId, name, email, phone);
        this.employeeId = employeeId;
    }
    
    public void addBook(Library library, Book book) {
        library.addBook(book);
    }
    
    public void removeBook(Library library, String ISBN) {
        library.removeBook(ISBN);
    }
}

// Book Loan
class BookLoan {
    private String loanId;
    private Book book;
    private Member member;
    private Date loanDate;
    private Date dueDate;
    private Date returnDate;
    private double fine;
    
    public BookLoan(String loanId, Book book, Member member) {
        this.loanId = loanId;
        this.book = book;
        this.member = member;
        this.loanDate = new Date();
        
        // Due date is 14 days from loan date
        Calendar cal = Calendar.getInstance();
        cal.setTime(loanDate);
        cal.add(Calendar.DAY_OF_MONTH, 14);
        this.dueDate = cal.getTime();
    }
    
    public void returnBook() {
        this.returnDate = new Date();
        calculateFine();
    }
    
    private void calculateFine() {
        if (returnDate.after(dueDate)) {
            long diff = returnDate.getTime() - dueDate.getTime();
            long daysLate = TimeUnit.MILLISECONDS.toDays(diff);
            fine = daysLate * 1.0;  // $1 per day
        }
    }
    
    public String getLoanId() { return loanId; }
    public Book getBook() { return book; }
    public Member getMember() { return member; }
    public Date getDueDate() { return dueDate; }
    public double getFine() { return fine; }
    public boolean isReturned() { return returnDate != null; }
}

// Library Catalog
class LibraryCatalog {
    private Map<String, List<Book>> booksByTitle;
    private Map<String, List<Book>> booksByAuthor;
    private Map<String, Book> booksByISBN;
    
    public LibraryCatalog() {
        booksByTitle = new HashMap<>();
        booksByAuthor = new HashMap<>();
        booksByISBN = new HashMap<>();
    }
    
    public void addBook(Book book) {
        // Add to ISBN map
        booksByISBN.put(book.getISBN(), book);
        
        // Add to title map
        booksByTitle.computeIfAbsent(book.getTitle().toLowerCase(), k -> new ArrayList<>()).add(book);
        
        // Add to author map
        booksByAuthor.computeIfAbsent(book.getAuthor().toLowerCase(), k -> new ArrayList<>()).add(book);
    }
    
    public void removeBook(String ISBN) {
        Book book = booksByISBN.remove(ISBN);
        if (book != null) {
            booksByTitle.get(book.getTitle().toLowerCase()).remove(book);
            booksByAuthor.get(book.getAuthor().toLowerCase()).remove(book);
        }
    }
    
    public Book searchByISBN(String ISBN) {
        return booksByISBN.get(ISBN);
    }
    
    public List<Book> searchByTitle(String title) {
        return booksByTitle.getOrDefault(title.toLowerCase(), new ArrayList<>());
    }
    
    public List<Book> searchByAuthor(String author) {
        return booksByAuthor.getOrDefault(author.toLowerCase(), new ArrayList<>());
    }
}

// Library (Singleton)
class Library {
    private static Library instance;
    private String name;
    private LibraryCatalog catalog;
    private Map<String, Member> members;
    private Map<String, BookLoan> activeLoans;
    
    private Library(String name) {
        this.name = name;
        this.catalog = new LibraryCatalog();
        this.members = new HashMap<>();
        this.activeLoans = new HashMap<>();
    }
    
    public static synchronized Library getInstance(String name) {
        if (instance == null) {
            instance = new Library(name);
        }
        return instance;
    }
    
    public void addBook(Book book) {
        catalog.addBook(book);
        System.out.println("Book added: " + book.getTitle());
    }
    
    public void removeBook(String ISBN) {
        catalog.removeBook(ISBN);
        System.out.println("Book removed: " + ISBN);
    }
    
    public void registerMember(Member member) {
        members.put(member.getUserId(), member);
        System.out.println("Member registered: " + member.getName());
    }
    
    public BookLoan lendBook(String memberId, String ISBN) {
        Member member = members.get(memberId);
        Book book = catalog.searchByISBN(ISBN);
        
        if (member == null) {
            System.out.println("Member not found");
            return null;
        }
        
        if (book == null) {
            System.out.println("Book not found");
            return null;
        }
        
        if (!member.canBorrowBook()) {
            System.out.println("Member cannot borrow book (limit reached or has fines)");
            return null;
        }
        
        if (book.getStatus() != BookStatus.AVAILABLE) {
            System.out.println("Book not available");
            return null;
        }
        
        String loanId = "LOAN-" + System.currentTimeMillis();
        BookLoan loan = new BookLoan(loanId, book, member);
        
        book.setStatus(BookStatus.LOANED);
        member.addLoan(loan);
        activeLoans.put(loanId, loan);
        
        System.out.println("Book loaned successfully. Loan ID: " + loanId);
        System.out.println("Due date: " + loan.getDueDate());
        
        return loan;
    }
    
    public void returnBook(String loanId) {
        BookLoan loan = activeLoans.get(loanId);
        
        if (loan == null) {
            System.out.println("Loan not found");
            return;
        }
        
        loan.returnBook();
        loan.getBook().setStatus(BookStatus.AVAILABLE);
        loan.getMember().removeLoan(loan);
        
        if (loan.getFine() > 0) {
            loan.getMember().addFine(loan.getFine());
            System.out.println("Book returned with fine: $" + loan.getFine());
        } else {
            System.out.println("Book returned successfully");
        }
        
        activeLoans.remove(loanId);
    }
    
    public List<Book> searchBooks(String query) {
        List<Book> results = new ArrayList<>();
        
        // Search by ISBN
        Book byISBN = catalog.searchByISBN(query);
        if (byISBN != null) {
            results.add(byISBN);
        }
        
        // Search by title
        results.addAll(catalog.searchByTitle(query));
        
        // Search by author
        results.addAll(catalog.searchByAuthor(query));
        
        return results;
    }
}

// Demo
public class LibraryManagementDemo {
    public static void main(String[] args) {
        Library library = Library.getInstance("City Library");
        
        // Add books
        Book book1 = new Book("ISBN-001", "Java Programming", "John Doe", "Tech Publishers");
        Book book2 = new Book("ISBN-002", "Spring Boot in Action", "Jane Smith", "Tech Publishers");
        Book book3 = new Book("ISBN-003", "Clean Code", "Robert Martin", "Code Publishers");
        
        library.addBook(book1);
        library.addBook(book2);
        library.addBook(book3);
        
        // Register members
        Member member1 = new Member("M001", "Alice Johnson", "alice@email.com", "123-456-7890");
        Member member2 = new Member("M002", "Bob Williams", "bob@email.com", "098-765-4321");
        
        library.registerMember(member1);
        library.registerMember(member2);
        
        // Lend books
        BookLoan loan1 = library.lendBook("M001", "ISBN-001");
        BookLoan loan2 = library.lendBook("M001", "ISBN-002");
        
        // Search books
        List<Book> javaBooks = library.searchBooks("Java");
        System.out.println("\nSearch results for 'Java': " + javaBooks.size() + " books found");
        
        // Return book
        if (loan1 != null) {
            library.returnBook(loan1.getLoanId());
        }
    }
}
```

---

### LLD Problem 3: Design a Hotel Management System

**Requirements:**
1. Rooms of different types (Single, Double, Suite)
2. Room booking and cancellation
3. Customer check-in/check-out
4. Room service orders
5. Payment processing
6. Room availability search

**Solution:**

```java
// Enums
enum RoomType {
    SINGLE, DOUBLE, DELUXE, SUITE
}

enum RoomStatus {
    AVAILABLE, BOOKED, OCCUPIED, MAINTENANCE
}

enum BookingStatus {
    PENDING, CONFIRMED, CANCELLED, COMPLETED
}

// Room class
class Room {
    private String roomNumber;
    private RoomType type;
    private double pricePerNight;
    private RoomStatus status;
    
    public Room(String roomNumber, RoomType type, double pricePerNight) {
        this.roomNumber = roomNumber;
        this.type = type;
        this.pricePerNight = pricePerNight;
        this.status = RoomStatus.AVAILABLE;
    }
    
    public boolean isAvailable() {
        return status == RoomStatus.AVAILABLE;
    }
    
    public void book() {
        this.status = RoomStatus.BOOKED;
    }
    
    public void checkIn() {
        this.status = RoomStatus.OCCUPIED;
    }
    
    public void checkOut() {
        this.status = RoomStatus.AVAILABLE;
    }
    
    // Getters
    public String getRoomNumber() { return roomNumber; }
    public RoomType getType() { return type; }
    public double getPricePerNight() { return pricePerNight; }
    public RoomStatus getStatus() { return status; }
}

// Guest class
class Guest {
    private String guestId;
    private String name;
    private String email;
    private String phone;
    private String address;
    
    public Guest(String guestId, String name, String email, String phone) {
        this.guestId = guestId;
        this.name = name;
        this.email = email;
        this.phone = phone;
    }
    
    // Getters
    public String getGuestId() { return guestId; }
    public String getName() { return name; }
    public String getEmail() { return email; }
}

// Booking class
class Booking {
    private String bookingId;
    private Guest guest;
    private Room room;
    private Date checkInDate;
    private Date checkOutDate;
    private int numberOfGuests;
    private BookingStatus status;
    private double totalAmount;
    
    public Booking(String bookingId, Guest guest, Room room, 
                   Date checkInDate, Date checkOutDate, int numberOfGuests) {
        this.bookingId = bookingId;
        this.guest = guest;
        this.room = room;
        this.checkInDate = checkInDate;
        this.checkOutDate = checkOutDate;
        this.numberOfGuests = numberOfGuests;
        this.status = BookingStatus.PENDING;
        calculateTotalAmount();
    }
    
    private void calculateTotalAmount() {
        long diff = checkOutDate.getTime() - checkInDate.getTime();
        long nights = TimeUnit.MILLISECONDS.toDays(diff);
        if (nights == 0) nights = 1;
        totalAmount = room.getPricePerNight() * nights;
    }
    
    public void confirm() {
        this.status = BookingStatus.CONFIRMED;
        room.book();
    }
    
    public void cancel() {
        this.status = BookingStatus.CANCELLED;
        if (room.getStatus() == RoomStatus.BOOKED) {
            room.checkOut();
        }
    }
    
    public void complete() {
        this.status = BookingStatus.COMPLETED;
    }
    
    // Getters
    public String getBookingId() { return bookingId; }
    public Guest getGuest() { return guest; }
    public Room getRoom() { return room; }
    public Date getCheckInDate() { return checkInDate; }
    public Date getCheckOutDate() { return checkOutDate; }
    public BookingStatus getStatus() { return status; }
    public double getTotalAmount() { return totalAmount; }
}

// Payment
interface PaymentMethod {
    boolean processPayment(double amount);
}

class CreditCardPayment implements PaymentMethod {
    private String cardNumber;
    
    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }
    
    @Override
    public boolean processPayment(double amount) {
        System.out.println("Processing credit card payment: $" + amount);
        // Payment processing logic
        return true;
    }
}

class CashPayment implements PaymentMethod {
    @Override
    public boolean processPayment(double amount) {
        System.out.println("Processing cash payment: $" + amount);
        return true;
    }
}

class Payment {
    private String paymentId;
    private Booking booking;
    private double amount;
    private PaymentMethod method;
    private Date paymentDate;
    private boolean isSuccessful;
    
    public Payment(String paymentId, Booking booking, PaymentMethod method) {
        this.paymentId = paymentId;
        this.booking = booking;
        this.amount = booking.getTotalAmount();
        this.method = method;
    }
    
    public boolean process() {
        isSuccessful = method.processPayment(amount);
        if (isSuccessful) {
            paymentDate = new Date();
            booking.confirm();
        }
        return isSuccessful;
    }
}

// Hotel (Singleton)
class Hotel {
    private static Hotel instance;
    private String name;
    private String address;
    private List<Room> rooms;
    private Map<String, Booking> bookings;
    private Map<String, Guest> guests;
    
    private Hotel(String name, String address) {
        this.name = name;
        this.address = address;
        this.rooms = new ArrayList<>();
        this.bookings = new HashMap<>();
        this.guests = new HashMap<>();
    }
    
    public static synchronized Hotel getInstance(String name, String address) {
        if (instance == null) {
            instance = new Hotel(name, address);
        }
        return instance;
    }
    
    public void addRoom(Room room) {
        rooms.add(room);
    }
    
    public void registerGuest(Guest guest) {
        guests.put(guest.getGuestId(), guest);
    }
    
    public List<Room> searchAvailableRooms(RoomType type, Date checkIn, Date checkOut) {
        return rooms.stream()
            .filter(room -> room.getType() == type && room.isAvailable())
            .collect(Collectors.toList());
    }
    
    public Booking createBooking(String guestId, String roomNumber, 
                                Date checkIn, Date checkOut, int numberOfGuests) {
        Guest guest = guests.get(guestId);
        Room room = rooms.stream()
            .filter(r -> r.getRoomNumber().equals(roomNumber))
            .findFirst()
            .orElse(null);
        
        if (guest == null || room == null || !room.isAvailable()) {
            System.out.println("Booking failed");
            return null;
        }
        
        String bookingId = "BK-" + System.currentTimeMillis();
        Booking booking = new Booking(bookingId, guest, room, checkIn, checkOut, numberOfGuests);
        bookings.put(bookingId, booking);
        
        System.out.println("Booking created: " + bookingId);
        System.out.println("Total amount: $" + booking.getTotalAmount());
        
        return booking;
    }
    
    public boolean checkIn(String bookingId) {
        Booking booking = bookings.get(bookingId);
        
        if (booking == null || booking.getStatus() != BookingStatus.CONFIRMED) {
            System.out.println("Check-in failed");
            return false;
        }
        
        booking.getRoom().checkIn();
        System.out.println("Check-in successful for booking: " + bookingId);
        return true;
    }
    
    public boolean checkOut(String bookingId) {
        Booking booking = bookings.get(bookingId);
        
        if (booking == null) {
            System.out.println("Check-out failed");
            return false;
        }
        
        booking.getRoom().checkOut();
        booking.complete();
        System.out.println("Check-out successful for booking: " + bookingId);
        return true;
    }
    
    public void cancelBooking(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking != null) {
            booking.cancel();
            System.out.println("Booking cancelled: " + bookingId);
        }
    }
}

// Demo
public class HotelManagementDemo {
    public static void main(String[] args) throws Exception {
        Hotel hotel = Hotel.getInstance("Grand Hotel", "123 Main St");
        
        // Add rooms
        hotel.addRoom(new Room("101", RoomType.SINGLE, 100));
        hotel.addRoom(new Room("102", RoomType.DOUBLE, 150));
        hotel.addRoom(new Room("201", RoomType.DELUXE, 200));
        hotel.addRoom(new Room("301", RoomType.SUITE, 300));
        
        // Register guests
        Guest guest1 = new Guest("G001", "John Doe", "john@email.com", "123-456-7890");
        hotel.registerGuest(guest1);
        
        // Create booking
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        Date checkIn = sdf.parse("2024-03-01");
        Date checkOut = sdf.parse("2024-03-03");
        
        Booking booking = hotel.createBooking("G001", "102", checkIn, checkOut, 2);
        
        if (booking != null) {
            // Process payment
            Payment payment = new Payment("PAY-001", booking, new CreditCardPayment("1234-5678-9012-3456"));
            boolean paymentSuccess = payment.process();
            
            if (paymentSuccess) {
                // Check-in
                hotel.checkIn(booking.getBookingId());
                
                // Check-out
                hotel.checkOut(booking.getBookingId());
            }
        }
    }
}
```

---

### Summary

This document covered:

**Part A - Interview Questions:**
1. ✅ Core Java (10 questions)
2. ✅ Collections Framework (3 questions)
3. ✅ Multithreading (3 questions)
4. ✅ Java 8+ Features (2 questions)
5. ✅ Spring Framework (2 questions)

**Part B - LLD Problems:**
1. ✅ Parking Lot System
2. ✅ Library Management System
3. ✅ Hotel Management System

Each LLD problem includes:
- Complete requirements
- Full working code
- Object-oriented design principles
- Design patterns used
- Extensibility considerations

**Key Design Patterns Used:**
- Singleton Pattern (Parking Lot, Library, Hotel)
- Strategy Pattern (Payment methods)
- Factory Pattern (Vehicle creation)
- Template Method Pattern

---

## Practice Tips

1. **Understand Requirements** - Ask clarifying questions
2. **Identify Entities** - Classes and their relationships
3. **Define Relationships** - Inheritance, composition, aggregation
4. **Apply SOLID Principles** - Single responsibility, Open/closed
5. **Use Design Patterns** - Where applicable
6. **Consider Extensibility** - How to add new features
7. **Write Clean Code** - Proper naming, comments
8. **Test Your Design** - Think of edge cases

**Good luck with your interviews!** 🚀

