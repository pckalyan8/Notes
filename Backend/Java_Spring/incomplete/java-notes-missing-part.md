### 4.11 Iterator and ListIterator

```java
import java.util.*;

public class IteratorExample {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C", "D"));

        // Using Iterator
        System.out.println("Using Iterator:");
        Iterator<String> iterator = list.iterator();
        while (iterator.hasNext()) {
            String element = iterator.next();
            System.out.println(element);
            if (element.equals("B")) {
                iterator.remove(); // Safely remove element during iteration
            }
        }
        System.out.println("After removing 'B': " + list);

        // Using ListIterator
        System.out.println("\nUsing ListIterator (forward):");
        ListIterator<String> listIterator = list.listIterator();
        while (listIterator.hasNext()) {
            String element = listIterator.next();
            System.out.println("Index: " + listIterator.previousIndex() + ", Element: " + element);
            if (element.equals("C")) {
                listIterator.set("C++"); // Modify element
            }
            if (element.equals("A")) {
                listIterator.add("A+"); // Add element
            }
        }
        System.out.println("After forward iteration: " + list);

        System.out.println("\nUsing ListIterator (backward):");
        while (listIterator.hasPrevious()) {
            String element = listIterator.previous();
            System.out.println("Index: " + listIterator.nextIndex() + ", Element: " + element);
        }
    }
}
```

---

## 6. Java 8+ Features (Continued)

### 6.2 Method References

Method references are a shorthand for lambda expressions that call a specific method.

```java
import java.util.Arrays;
import java.util.List;
import java.util.function.Function;

public class MethodReferencesExample {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("alice", "bob", "charlie");

        // Lambda expression
        names.forEach(s -> System.out.println(s));

        // Method reference to an instance method of an arbitrary object of a particular type
        names.forEach(System.out::println);

        // Reference to a static method
        names.replaceAll(String::toUpperCase);
        System.out.println("Uppercase: " + names);

        // Reference to an instance method of a particular object
        String prefix = "User: ";
        names.forEach(prefix::concat); // This doesn't modify the list, just an example

        // Reference to a constructor
        Function<String, User> userFactory = User::new;
        User user = userFactory.apply("Alice");
        System.out.println("Created user: " + user.getName());
    }
}

class User {
    private String name;
    public User(String name) { this.name = name; }
    public String getName() { return name; }
}
```

### 6.3 Stream API

The Stream API is used for processing collections of objects in a functional style.

```java
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class StreamApiExample {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David", "Anna");

        // Filtering: Find names starting with 'A'
        List<String> aNames = names.stream()
                .filter(name -> name.startsWith("A"))
                .collect(Collectors.toList());
        System.out.println("Names starting with A: " + aNames);

        // Mapping: Get lengths of all names
        List<Integer> nameLengths = names.stream()
                .map(String::length)
                .collect(Collectors.toList());
        System.out.println("Name lengths: " + nameLengths);

        // Sorting
        List<String> sortedNames = names.stream()
                .sorted()
                .collect(Collectors.toList());
        System.out.println("Sorted names: " + sortedNames);

        // Reducing: Concatenate all names
        String concatenated = names.stream()
                .reduce("", (s1, s2) -> s1 + s2 + ", ");
        System.out.println("Concatenated: " + concatenated);

        // Chaining operations: find names longer than 3 chars, make them uppercase, and collect
        List<String> result = names.stream()
                .filter(name -> name.length() > 3)
                .map(String::toUpperCase)
                .sorted()
                .collect(Collectors.toList());
        System.out.println("Processed names: " + result);
    }
}
```

### 6.4 Optional Class

`Optional` is a container object which may or may not contain a non-null value.

```java
import java.util.Optional;

public class OptionalExample {
    public static void main(String[] args) {
        Optional<String> optionalWithValue = Optional.of("Hello");
        Optional<String> emptyOptional = Optional.empty();

        // isPresent() and get()
        if (optionalWithValue.isPresent()) {
            System.out.println("Value: " + optionalWithValue.get());
        }

        // ifPresent()
        emptyOptional.ifPresent(value -> System.out.println("This won't be printed"));

        // orElse()
        String value1 = emptyOptional.orElse("Default Value");
        System.out.println("orElse(): " + value1);

        // orElseGet()
        String value2 = emptyOptional.orElseGet(() -> "Default from Supplier");
        System.out.println("orElseGet(): " + value2);

        // orElseThrow()
        try {
            emptyOptional.orElseThrow(() -> new IllegalStateException("Value not present"));
        } catch (IllegalStateException e) {
            System.out.println("orElseThrow(): " + e.getMessage());
        }
    }
}
```

### 6.5 Default and Static Methods in Interfaces

```java
public interface MyInterface {
    void regularMethod();

    default void defaultMethod() {
        System.out.println("This is a default method.");
    }

    static void staticMethod() {
        System.out.println("This is a static method in an interface.");
    }
}

class MyClass implements MyInterface {
    @Override
    public void regularMethod() {
        System.out.println("Implementing the regular method.");
    }
}

public class InterfaceMethodsExample {
    public static void main(String[] args) {
        MyClass myClass = new MyClass();
        myClass.regularMethod();
        myClass.defaultMethod(); // Calling default method
        MyInterface.staticMethod(); // Calling static method
    }
}
```

### 6.6 Date and Time API

```java
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;

public class DateTimeApiExample {
    public static void main(String[] args) {
        // Current date and time
        LocalDate today = LocalDate.now();
        LocalTime now = LocalTime.now();
        LocalDateTime currentDateTime = LocalDateTime.now();

        System.out.println("Today: " + today);
        System.out.println("Now: " + now);
        System.out.println("Current DateTime: " + currentDateTime);

        // Creating specific dates
        LocalDate specificDate = LocalDate.of(2024, 8, 20);
        System.out.println("Specific Date: " + specificDate);

        // Formatting
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss");
        String formattedDateTime = currentDateTime.format(formatter);
        System.out.println("Formatted DateTime: " + formattedDateTime);
    }
}
```

### 6.7 Records (Java 14+)

Records are immutable data classes that require only the type and name of fields.

```java
// Record definition
public record Person(String name, int age) {}

public class RecordsExample {
    public static void main(String[] args) {
        Person person1 = new Person("John Doe", 30);
        System.out.println(person1.name()); // Accessor
        System.out.println(person1.age());
        System.out.println(person1); // toString() is auto-generated
    }
}
```

### 6.8 Text Blocks (Java 15+)

Text blocks are multi-line string literals.

```java
public class TextBlocksExample {
    public static void main(String[] args) {
        String json = """
        {
            "name": "John",
            "age": 30,
            "city": "New York"
        }
        """;
        System.out.println(json);
    }
}
```

---

## 7. Memory Management

### 7.1 JVM Architecture
- **Classloader**: Responsible for loading, linking, and initializing class files.
- **Runtime Data Areas**:
    - **Method Area**: Stores class structures, static variables, and the constant pool.
    - **Heap**: Stores all objects and their instance variables.
    - **Stack**: Stores local variables and partial results for each thread. Each thread has its own JVM stack.
    - **PC Registers**: Contains the address of the current JVM instruction for each thread.
    - **Native Method Stacks**: For native methods.
- **Execution Engine**: Executes the bytecode. Contains the JIT (Just-In-Time) compiler.

### 7.2 Heap vs Stack Memory
- **Stack**:
    - Last-In-First-Out (LIFO).
    - Used for static memory allocation and thread execution.
    - Contains primitive variables and references to objects in the heap.
    - Memory is allocated and deallocated automatically.
    - Fast access.
- **Heap**:
    - Used for dynamic memory allocation for objects and JRE classes.
    - Memory is managed by the Garbage Collector.
    - Slower access compared to stack.
    - Can lead to `OutOfMemoryError` if not managed properly.

### 7.3 Garbage Collection
- The process of automatically freeing up memory by destroying unused objects.
- **Major GC vs Minor GC**: Minor GC cleans the Young Generation, Major GC cleans the Old Generation.
- **Garbage Collectors**:
    - **Serial GC**: Single-threaded, simple.
    - **Parallel GC**: Default in JDK 8. Uses multiple threads.
    - **G1 GC**: Garbage-First. Divides the heap into regions. Default from JDK 9.
    - **ZGC**: Low-latency garbage collector.

### 7.4 Memory Leaks
A memory leak occurs when an object is no longer needed but is not garbage collected because it's still being referenced. Common causes:
- Static field references.
- Unclosed resources.
- Improper `equals()` and `hashCode()` implementations in HashMaps.

### 7.5 String Pool
- A special storage area in the Java heap for string literals.
- When a string is created as a literal (e.g., `String s = "hello";`), the JVM checks the string pool first. If the string already exists, a reference to the pooled instance is returned.
- `new String("hello")` creates a new object in the heap, outside the pool.

---

## 8. Input/Output (I/O)

### 8.1 File Handling
```java
import java.io.File;
import java.io.FileWriter;
import java.io.IOException;
import java.util.Scanner;

public class FileHandlingExample {
    public static void main(String[] args) {
        try {
            // Writing to a file
            FileWriter writer = new FileWriter("example.txt");
            writer.write("Hello, World!\nThis is a test.");
            writer.close();

            // Reading from a file
            File file = new File("example.txt");
            Scanner scanner = new Scanner(file);
            while (scanner.hasNextLine()) {
                String data = scanner.nextLine();
                System.out.println(data);
            }
            scanner.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 8.2 Serialization and Deserialization
Serialization is the process of converting an object into a byte stream. Deserialization is the reverse.

```java
import java.io.*;

class Employee implements Serializable {
    private static final long serialVersionUID = 1L;
    public String name;
    public String address;
}

public class SerializationExample {
    public static void main(String[] args) {
        Employee emp = new Employee();
        emp.name = "John Doe";
        emp.address = "123 Main St";

        try {
            // Serialization
            FileOutputStream fileOut = new FileOutputStream("employee.ser");
            ObjectOutputStream out = new ObjectOutputStream(fileOut);
            out.writeObject(emp);
            out.close();
            fileOut.close();

            // Deserialization
            FileInputStream fileIn = new FileInputStream("employee.ser");
            ObjectInputStream in = new ObjectInputStream(fileIn);
            Employee deserializedEmp = (Employee) in.readObject();
            in.close();
            fileIn.close();

            System.out.println("Deserialized Employee: " + deserializedEmp.name);
        } catch (IOException | ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

### 8.3 NIO (New I/O)
NIO provides a different I/O model based on channels and buffers.

```java
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.file.Paths;
import java.nio.file.StandardOpenOption;

public class NioExample {
    public static void main(String[] args) {
        try (FileChannel channel = FileChannel.open(Paths.get("nio-example.txt"), StandardOpenOption.CREATE, StandardOpenOption.WRITE)) {
            ByteBuffer buffer = ByteBuffer.wrap("Hello NIO".getBytes());
            channel.write(buffer);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 8.4 Streams (Byte and Character)
- **Byte Streams**: Used to read and write binary data (8-bit bytes). `InputStream` and `OutputStream` are the abstract base classes.
- **Character Streams**: Used to read and write character data (16-bit Unicode). `Reader` and `Writer` are the abstract base classes. They are often wrappers around byte streams.
```java
// Example of using a character stream (FileReader) that wraps a byte stream (FileInputStream)
try {
    FileReader reader = new FileReader("file.txt"); // Character stream
    int character;
    while((character = reader.read()) != -1) {
        System.out.print((char) character);
    }
    reader.close();
} catch (IOException e) {
    e.printStackTrace();
}
```