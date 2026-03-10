# 4.9 Reflection API in Java

## What & Why

Reflection is Java's ability to inspect and manipulate its own structure at runtime. Normally, a Java program knows at compile time which classes it uses and what their methods are. Reflection breaks that constraint — it lets code ask "what methods does this class have?" and "can you call this method for me?" without knowing the answer at compile time.

This capability is what makes Java frameworks like Spring, Hibernate, JUnit, and Jackson possible. They read your class structure at runtime, find annotations, inject dependencies, and wire components together — all without being compiled with your code. Understanding reflection helps you understand how these tools work and equips you to build similar capabilities.

Reflection does come at a cost: it bypasses compile-time type checking, is slower than direct method calls (though this gap has narrowed significantly in modern JVMs), and can break encapsulation by accessing private members. Use it deliberately, for infrastructure and tooling code, not for ordinary business logic.

---

## 1. The Class Object — Your Reflection Entry Point

Every Java class, at runtime, is represented by exactly one `Class<T>` object loaded by the ClassLoader. This object is your gateway into everything reflection can tell you about a type.

There are three ways to obtain a `Class<?>` object:

```java
// Method 1: use .class on the type name — resolved at compile time, no exception
Class<String> strClass  = String.class;
Class<int[]>  arrClass  = int[].class;
Class<Void>   voidClass = void.class;

// Method 2: call .getClass() on an instance — returns the runtime type
Object obj = "Hello";
Class<?> runtimeClass = obj.getClass(); // returns String.class, not Object.class

// Method 3: Class.forName() — loads a class by its fully-qualified name at runtime
// Throws ClassNotFoundException if the class isn't on the classpath
// This is how plugin systems and dependency injection containers load arbitrary classes
try {
    Class<?> loaded = Class.forName("java.util.ArrayList");
    System.out.println(loaded.getName()); // java.util.ArrayList
} catch (ClassNotFoundException e) {
    System.err.println("Class not found: " + e.getMessage());
}
```

Once you have the `Class` object, you can ask it about its structure:

```java
Class<?> clazz = ArrayList.class;

// Identity
System.out.println(clazz.getName());           // java.util.ArrayList (fully qualified)
System.out.println(clazz.getSimpleName());     // ArrayList (short name)
System.out.println(clazz.getPackageName());    // java.util (Java 9+)

// Hierarchy
System.out.println(clazz.getSuperclass());     // class java.util.AbstractList
Class<?>[] interfaces = clazz.getInterfaces();
for (Class<?> iface : interfaces) {
    System.out.println(iface.getName());
    // java.util.List, java.util.RandomAccess, java.lang.Cloneable, java.io.Serializable
}

// Type checks
System.out.println(clazz.isInterface());  // false
System.out.println(clazz.isArray());      // false
System.out.println(clazz.isEnum());       // false
System.out.println(List.class.isAssignableFrom(ArrayList.class)); // true — ArrayList IS a List
```

---

## 2. Inspecting and Using Fields

```java
import java.lang.reflect.Field;

public class Person {
    public String name;
    private int age;
    protected String email;
}

Class<?> clazz = Person.class;

// getFields() — only PUBLIC fields (including inherited ones from superclasses)
Field[] publicFields = clazz.getFields();
for (Field f : publicFields) {
    System.out.println(f.getName() + " : " + f.getType().getSimpleName());
    // name : String
}

// getDeclaredFields() — ALL fields declared in THIS class (private, protected, public)
// Does NOT include inherited fields
Field[] allFields = clazz.getDeclaredFields();
for (Field f : allFields) {
    System.out.printf("%-20s %-10s%n", f.getName(), f.getType().getSimpleName());
    // name                 String
    // age                  int
    // email                String
}

// Reading and writing field values
Person person = new Person();
person.name = "Alice";

// For private fields, you must call setAccessible(true) first
// This bypasses the normal access control
Field ageField = clazz.getDeclaredField("age");
ageField.setAccessible(true); // "open the door" to private access

ageField.set(person, 30);                       // write: person.age = 30
int age = (int) ageField.get(person);           // read: returns 30
System.out.println("Age via reflection: " + age);

// Field metadata
System.out.println(ageField.getModifiers()); // int encoding the modifiers (private = 2)
System.out.println(java.lang.reflect.Modifier.isPrivate(ageField.getModifiers())); // true
```

---

## 3. Inspecting and Invoking Methods

```java
import java.lang.reflect.Method;

public class Calculator {
    public int add(int a, int b)       { return a + b; }
    public int multiply(int a, int b)  { return a * b; }
    private int secret(String s)       { return s.length(); }
}

Class<?> clazz = Calculator.class;

// getMethods() — all PUBLIC methods including those inherited from Object
Method[] publicMethods = clazz.getMethods();

// getDeclaredMethods() — all methods declared in this class, any access level
Method[] allMethods = clazz.getDeclaredMethods();
for (Method m : allMethods) {
    System.out.printf("%-15s returns %-10s takes %d params%n",
        m.getName(),
        m.getReturnType().getSimpleName(),
        m.getParameterCount());
}

// Invoking a method by name
Calculator calc  = new Calculator();
Method addMethod = clazz.getDeclaredMethod("add", int.class, int.class);
                                            // ^ method name  ^ parameter types

Object result = addMethod.invoke(calc, 5, 3);  // same as: calc.add(5, 3)
System.out.println("5 + 3 = " + result);        // 5 + 3 = 8

// Invoking a private method
Method secretMethod = clazz.getDeclaredMethod("secret", String.class);
secretMethod.setAccessible(true);
int length = (int) secretMethod.invoke(calc, "reflection");
System.out.println("String length: " + length); // 10

// Inspecting method parameters
for (java.lang.reflect.Parameter param : addMethod.getParameters()) {
    System.out.println(param.getName() + " : " + param.getType().getSimpleName());
    // Note: parameter names are only preserved if compiled with -parameters flag
}
```

---

## 4. Inspecting and Using Constructors

```java
import java.lang.reflect.Constructor;

public class Point {
    private final int x, y;

    public Point()           { this(0, 0); }
    public Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public String toString() { return "Point(" + x + ", " + y + ")"; }
}

Class<?> clazz = Point.class;

// Inspect all declared constructors
Constructor<?>[] constructors = clazz.getDeclaredConstructors();
for (Constructor<?> c : constructors) {
    System.out.println("Constructor with " + c.getParameterCount() + " params");
}

// Create an instance using the no-arg constructor
Constructor<?> noArg = clazz.getDeclaredConstructor();
Point origin = (Point) noArg.newInstance();  // same as: new Point()
System.out.println(origin); // Point(0, 0)

// Create an instance using the two-arg constructor
Constructor<?> twoArg = clazz.getDeclaredConstructor(int.class, int.class);
Point p = (Point) twoArg.newInstance(3, 4);  // same as: new Point(3, 4)
System.out.println(p); // Point(3, 4)

// Shorter: getDeclaredConstructor + newInstance in one call
// But the above pattern is more explicit and easier to debug
```

---

## 5. Annotations via Reflection

This is one of the most practically important uses of reflection — it's how every annotation-driven framework (Spring, JUnit, Hibernate) discovers your intent. An annotation is only accessible via reflection if it is marked `@Retention(RetentionPolicy.RUNTIME)`.

```java
import java.lang.annotation.*;
import java.lang.reflect.*;

// Define a custom runtime annotation
@Retention(RetentionPolicy.RUNTIME)   // must be RUNTIME to be readable via reflection
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.TYPE})
public @interface Validate {
    int minLength() default 0;
    int maxLength() default Integer.MAX_VALUE;
    boolean required() default true;
}

// Apply the annotation
public class UserForm {
    @Validate(minLength = 2, maxLength = 50, required = true)
    private String username;

    @Validate(minLength = 8, required = true)
    private String password;

    private int age; // no @Validate annotation
}

// Read annotations via reflection
public class Validator {

    public static void validate(Object form) throws IllegalAccessException {
        Class<?> clazz = form.getClass();

        for (Field field : clazz.getDeclaredFields()) {
            // Check if this field has our annotation
            if (!field.isAnnotationPresent(Validate.class)) continue;

            Validate rule = field.getAnnotation(Validate.class);
            field.setAccessible(true);
            Object value = field.get(form);

            System.out.printf("Checking field '%s' (required=%b, min=%d, max=%d)%n",
                field.getName(), rule.required(), rule.minLength(), rule.maxLength());

            if (rule.required() && value == null) {
                throw new IllegalArgumentException(field.getName() + " is required");
            }

            if (value instanceof String str) {  // pattern matching instanceof (Java 16+)
                if (str.length() < rule.minLength()) {
                    throw new IllegalArgumentException(
                        field.getName() + " is too short (min: " + rule.minLength() + ")");
                }
                if (str.length() > rule.maxLength()) {
                    throw new IllegalArgumentException(
                        field.getName() + " is too long (max: " + rule.maxLength() + ")");
                }
            }
            System.out.println("  -> OK");
        }
    }

    public static void main(String[] args) throws Exception {
        UserForm form = new UserForm();
        form.username = "alice";
        form.password = "securePass";
        validate(form); // both fields pass validation
    }
}
```

---

## 6. Dynamic Proxies

Java's `Proxy` class lets you create objects at runtime that implement a given interface without writing the implementing class. Every method call on the proxy is intercepted by an `InvocationHandler` you provide. This is the mechanism behind Spring AOP, mock frameworks (Mockito), and logging wrappers.

```java
import java.lang.reflect.*;

// The interface we want to proxy
public interface UserService {
    User findById(long id);
    void save(User user);
}

// A real implementation
public class UserServiceImpl implements UserService {
    public User findById(long id) { return new User(id, "Alice"); }
    public void save(User user)   { System.out.println("Saved: " + user); }
}

// Create a proxy that logs every method call
UserService real = new UserServiceImpl();

UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class<?>[]{ UserService.class },
    (proxyObj, method, args) -> {
        // This InvocationHandler is called for EVERY method on the proxy
        System.out.println("Calling " + method.getName() + " with args: " + Arrays.toString(args));
        long start = System.nanoTime();

        Object result = method.invoke(real, args);  // delegate to the real implementation

        long elapsed = System.nanoTime() - start;
        System.out.printf("  %s completed in %.3f ms%n", method.getName(), elapsed / 1e6);
        return result;
    }
);

User user = proxy.findById(42L);
// Output:
// Calling findById with args: [42]
//   findById completed in 0.123 ms
```

---

## 7. Practical Use Cases Summary

Reflection is the engine behind many Java ecosystem tools, even though application developers rarely write reflection code directly:

**Dependency Injection (Spring IoC):** Spring scans your classpath for classes annotated with `@Component`, `@Service`, etc. It uses reflection to inspect their constructors and fields marked `@Autowired`, then instantiates them and wires dependencies together.

**Object-Relational Mapping (Hibernate/JPA):** Hibernate reads `@Entity`, `@Column`, `@OneToMany` annotations on your classes via reflection to map Java objects to database tables without requiring any XML configuration.

**JSON Serialization (Jackson):** Jackson's `ObjectMapper` inspects field names and types via reflection to convert Java objects to JSON and back, respecting annotations like `@JsonProperty` and `@JsonIgnore`.

**Testing Frameworks (JUnit 5):** JUnit uses reflection to find methods annotated `@Test`, `@BeforeEach`, etc., and calls them without needing you to extend any base class.

**Mock Frameworks (Mockito):** Mockito creates proxy objects at runtime (using reflection + byte-code generation) that record method calls and return configured results.

---

## ⚡ Best Practices & Important Points

**Use reflection only for infrastructure code** — frameworks, utilities, and tools — not for everyday business logic. If you find yourself using reflection to call a method that you could just call directly, something has gone wrong in your design.

**Cache `Method`, `Field`, and `Constructor` objects if you'll use them repeatedly.** Reflective lookups are relatively expensive. Storing the `Method` object from `getDeclaredMethod()` and reusing it is significantly faster than looking it up every time.

**Always call `setAccessible(true)` before accessing private members**, and be aware that in Java 9+ the module system can block this — you may need `--add-opens` flags or the module must `opens` the package.

**Check for annotations correctly.** Use `isAnnotationPresent()` before calling `getAnnotation()` to avoid `NullPointerException`, and ensure the annotation is declared `@Retention(RetentionPolicy.RUNTIME)` — annotations with `SOURCE` or `CLASS` retention are not available at runtime.

**`method.invoke()` wraps exceptions in `InvocationTargetException`.** If the invoked method throws, you get an `InvocationTargetException`; call `e.getCause()` to get the original exception.

**Prefer the Java 11+ `MethodHandles` API** for performance-critical reflective access. `MethodHandle` has much lower invocation overhead than `Method.invoke()` because the JVM can optimise it more aggressively.
