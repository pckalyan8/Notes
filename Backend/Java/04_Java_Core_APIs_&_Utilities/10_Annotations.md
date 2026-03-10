# 4.10 Annotations in Java

## What & Why

An annotation is a form of **metadata** — information about code, attached directly to the code it describes. Annotations don't change how the code executes by themselves; they are markers that tools (compilers, IDEs, frameworks, annotation processors) can read and act upon. Think of annotations as structured comments that machines can understand.

Before annotations (introduced in Java 5), developers used XML configuration files to describe the same metadata — how a class should be persisted to a database, which methods were test cases, which classes Spring should manage. Annotations moved this metadata directly into the source code next to what it describes, making the configuration co-located, refactorable, and harder to get out of sync.

Every annotation-driven framework you'll encounter in Java — Spring, Hibernate, JUnit 5, Jakarta EE, Jackson — is built on the annotation system. Understanding how annotations are declared, applied, and read via reflection gives you insight into how all of those tools work under the hood.

---

## 1. Built-in Java Annotations

Java ships with a small set of annotations for compiler and runtime use.

### @Override

Tells the compiler that this method is intended to override a method in a superclass or implement a method from an interface. The compiler verifies this — if no such method exists in the parent, you get a compile error instead of silently creating a new method with a typo:

```java
public class Animal {
    public String speak() { return "..."; }
}

public class Dog extends Animal {
    @Override
    public String speak() { return "Woof"; }  // compiler verifies this overrides Animal.speak()

    // @Override
    // public String speek() { ... }  // compile error: no method speek() in Animal
}
```

Always use `@Override`. It's a zero-cost safety net.

### @Deprecated

Marks an element as obsolete. The compiler emits a warning at any call site that uses the deprecated element, signalling to developers that they should migrate to a newer API:

```java
public class StringUtils {

    @Deprecated(since = "2.0", forRemoval = true)  // Java 9+: richer metadata
    public static String trim(String s) {
        return s.trim();
    }

    public static String strip(String s) {         // the recommended replacement
        return s.strip();
    }
}
```

The `since` attribute tells developers when it was deprecated; `forRemoval = true` warns that the element will eventually be deleted, making the migration more urgent.

### @SuppressWarnings

Tells the compiler to suppress specific categories of warnings. Use sparingly — suppressing warnings hides legitimate problems:

```java
@SuppressWarnings("unchecked")       // suppress unchecked cast warnings (generics)
public <T> T castToType(Object obj) {
    return (T) obj;
}

@SuppressWarnings({"unused", "deprecation"})  // multiple warnings suppressed
public void legacyMethod() {
    int unusedVar = 5;
    StringUtils.trim("hello");  // calling a deprecated method
}
```

### @FunctionalInterface

Documents that an interface is intended to be a functional interface (exactly one abstract method). The compiler verifies this — if you accidentally add a second abstract method, it's a compile error rather than a silent API break:

```java
@FunctionalInterface
public interface Transformer<T, R> {
    R transform(T input);   // the one abstract method

    // default and static methods are allowed
    default Transformer<T, R> withLogging() {
        return input -> {
            System.out.println("Transforming: " + input);
            return this.transform(input);
        };
    }
}
```

### @SafeVarargs

Suppresses "heap pollution" warnings on varargs methods that use generic types. Use only when you can manually verify that the method does not pollute the heap (i.e., it doesn't mix elements from different parameterised types):

```java
@SafeVarargs
@SuppressWarnings("varargs")
public static <T> List<T> listOf(T... elements) {
    return Arrays.asList(elements);
}
```

---

## 2. Meta-Annotations — Annotations About Annotations

Meta-annotations control how your custom annotations behave.

### @Retention

Controls how long the annotation's information survives after compilation. This is the most critical meta-annotation because it determines whether the annotation is readable via reflection at runtime:

```java
@Retention(RetentionPolicy.SOURCE)   // exists only in source code — stripped by compiler
                                     // Use for: @Override, @SuppressWarnings
@Retention(RetentionPolicy.CLASS)    // embedded in .class file but NOT available at runtime
                                     // This is the DEFAULT if @Retention is omitted
@Retention(RetentionPolicy.RUNTIME)  // available at runtime via reflection
                                     // Use for: Spring, Hibernate, JUnit, and any annotation
                                     // you want to read at runtime
```

If you forget `@Retention(RetentionPolicy.RUNTIME)` on an annotation you intend to read via reflection, `getAnnotation()` will always return null — a confusing bug to diagnose.

### @Target

Restricts where the annotation can be placed. Without `@Target`, it can be placed anywhere:

```java
import java.lang.annotation.ElementType;

@Target(ElementType.TYPE)            // only on classes, interfaces, enums
@Target(ElementType.METHOD)          // only on methods
@Target(ElementType.FIELD)           // only on fields
@Target(ElementType.PARAMETER)       // only on method parameters
@Target(ElementType.CONSTRUCTOR)     // only on constructors
@Target(ElementType.LOCAL_VARIABLE)  // only on local variables
@Target(ElementType.ANNOTATION_TYPE) // only on other annotations (meta-annotation)
@Target(ElementType.PACKAGE)         // only on package declarations

// Combine multiple targets using an array:
@Target({ElementType.METHOD, ElementType.FIELD})
```

### @Documented

Includes the annotation in the Javadoc output for annotated elements. Without it, annotations are invisible in generated documentation.

### @Inherited

If you annotate a class with an `@Inherited` annotation, its subclasses automatically inherit that annotation. This only applies to class-level annotations, not methods or fields.

```java
@Inherited
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Category {
    String value();
}

@Category("finance")
public class FinancialReport { }

// FinancialSummary inherits @Category("finance") automatically
public class FinancialSummary extends FinancialReport { }

// Verify inheritance
Category cat = FinancialSummary.class.getAnnotation(Category.class);
System.out.println(cat.value()); // "finance" — inherited from parent
```

### @Repeatable (Java 8+)

Allows the same annotation to be applied multiple times to the same element. Without `@Repeatable`, you could only use an annotation once per element:

```java
// Step 1: define the container annotation
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Schedules {
    Schedule[] value();  // container holds an array
}

// Step 2: make the repeatable annotation point to its container
@Repeatable(Schedules.class)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Schedule {
    String cron();
    String timezone() default "UTC";
}

// Step 3: use it multiple times on the same element
public class ReportJob {
    @Schedule(cron = "0 6 * * MON-FRI")          // weekdays at 6am
    @Schedule(cron = "0 8 * * SAT-SUN")          // weekends at 8am
    public void generateDailyReport() { ... }
}

// Step 4: read back via reflection
Method method = ReportJob.class.getDeclaredMethod("generateDailyReport");
Schedule[] schedules = method.getAnnotationsByType(Schedule.class);
for (Schedule s : schedules) {
    System.out.println("Cron: " + s.cron() + " Timezone: " + s.timezone());
}
```

---

## 3. Creating Custom Annotations

An annotation is declared like an interface but with `@interface`. Its elements are declared like methods (no parameters, no throws) with optional default values:

```java
import java.lang.annotation.*;

/**
 * Marks a field that must be validated before the containing object is persisted.
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)      // must be RUNTIME to read via reflection
@Target({ElementType.FIELD, ElementType.METHOD})
public @interface NotBlank {
    // The "message" element is the validation failure description
    String message() default "Field must not be blank";

    // Optional: group-based validation (used by Bean Validation frameworks)
    Class<?>[] groups() default {};
}
```

Annotation elements can only be of these types: primitives (`int`, `long`, `boolean`, etc.), `String`, `Class`, `enum`, another annotation type, or a one-dimensional array of any of the above. You cannot use `List`, `Optional`, or arbitrary objects.

```java
// Apply the annotation
public class User {
    @NotBlank(message = "Username is required")
    private String username;

    @NotBlank  // uses default message
    private String email;
}
```

---

## 4. A Complete Working Example — A Validation Framework

This ties together annotation declaration, application, and runtime reading via reflection — the exact pattern that Bean Validation (Jakarta Validation) and Spring use internally:

```java
// --- Step 1: Define the annotations ---

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Required {
    String message() default "Field is required";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Length {
    int min() default 0;
    int max() default Integer.MAX_VALUE;
    String message() default "Field length is invalid";
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface Range {
    int min() default Integer.MIN_VALUE;
    int max() default Integer.MAX_VALUE;
    String message() default "Field value is out of range";
}

// --- Step 2: Annotate a domain class ---

public class RegistrationForm {
    @Required(message = "Username cannot be empty")
    @Length(min = 3, max = 20, message = "Username must be 3-20 chars")
    private String username;

    @Required
    @Length(min = 8, message = "Password must be at least 8 characters")
    private String password;

    @Range(min = 13, max = 120, message = "Age must be between 13 and 120")
    private int age;

    // constructor, getters...
}

// --- Step 3: Write the validator that reads annotations at runtime ---

import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.List;

public class BeanValidator {

    public static List<String> validate(Object bean) throws IllegalAccessException {
        List<String> errors = new ArrayList<>();
        Class<?> clazz = bean.getClass();

        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);  // allow reading private fields
            Object value = field.get(bean);

            // Check @Required
            if (field.isAnnotationPresent(Required.class)) {
                Required req = field.getAnnotation(Required.class);
                if (value == null || (value instanceof String s && s.isBlank())) {
                    errors.add(field.getName() + ": " + req.message());
                }
            }

            // Check @Length (applicable only to Strings)
            if (field.isAnnotationPresent(Length.class) && value instanceof String str) {
                Length len = field.getAnnotation(Length.class);
                if (str.length() < len.min() || str.length() > len.max()) {
                    errors.add(field.getName() + ": " + len.message());
                }
            }

            // Check @Range (applicable only to Numbers)
            if (field.isAnnotationPresent(Range.class) && value instanceof Number num) {
                Range range = field.getAnnotation(Range.class);
                int intVal = num.intValue();
                if (intVal < range.min() || intVal > range.max()) {
                    errors.add(field.getName() + ": " + range.message());
                }
            }
        }

        return errors;
    }

    public static void main(String[] args) throws Exception {
        RegistrationForm form = new RegistrationForm("al", "short", 10); // bad values

        List<String> errors = validate(form);
        if (errors.isEmpty()) {
            System.out.println("Validation passed!");
        } else {
            System.out.println("Validation failed:");
            errors.forEach(e -> System.out.println("  - " + e));
        }
        // Validation failed:
        //   - username: Username must be 3-20 chars
        //   - password: Password must be at least 8 characters
        //   - age: Age must be between 13 and 120
    }
}
```

---

## 5. Annotation Processors — Compile-Time Processing

Annotation processors run during compilation (not at runtime) and can generate new source files, validate code, or produce build artifacts. They implement `javax.annotation.processing.AbstractProcessor` and are declared in `META-INF/services/javax.annotation.processing.Processor`.

This is how **Lombok** works — it processes your `@Data`, `@Builder`, `@Getter` annotations at compile time and *generates* the boilerplate source code that Java then compiles. By the time the `.class` file exists, all the generated getters, setters, and constructors are already present.

```java
// Lombok in action — @Data generates equals(), hashCode(), toString(),
// all getters, all setters, and a required-args constructor
@Data
@Builder
public class Product {
    private final Long id;
    private String name;
    private BigDecimal price;
    private int stock;
}

// You can now do:
Product p = Product.builder()
    .id(1L)
    .name("Widget")
    .price(new BigDecimal("9.99"))
    .stock(100)
    .build();

System.out.println(p.getName()); // Widget — getter generated by @Data
System.out.println(p);           // Product(id=1, name=Widget, ...) — toString generated
```

MapStruct is another annotation-processor-based tool — it processes `@Mapper` annotations and generates type-safe mapping code between DTOs and domain objects, again at compile time.

---

## 6. Common Framework Annotations at a Glance

Understanding where framework annotations fit in the annotation lifecycle helps you debug and reason about them:

Spring's `@Autowired`, `@Component`, `@Service`, `@Transactional` — all `RUNTIME` retention. Spring reads them via reflection to build its application context.

JUnit 5's `@Test`, `@BeforeEach`, `@ParameterizedTest` — `RUNTIME` retention. JUnit scans for them via reflection to discover and run tests.

Hibernate's `@Entity`, `@Table`, `@Column`, `@OneToMany` — `RUNTIME` retention. Hibernate reads the ORM mapping at startup.

Jackson's `@JsonProperty`, `@JsonIgnore` — `RUNTIME` retention. Jackson reads them during serialization and deserialization.

Lombok's `@Data`, `@Builder`, `@Slf4j` — `SOURCE` retention. Lombok processes them at compile time and then they're gone — there's nothing to read at runtime because the generated code is already compiled into the `.class` file.

---

## ⚡ Best Practices & Important Points

**Always specify `@Retention` explicitly.** The default (`CLASS`) means your annotation is invisible at runtime — a common source of mysterious "annotation not found" bugs. For any annotation you intend to read via reflection, use `@Retention(RetentionPolicy.RUNTIME)`.

**Always specify `@Target`.** Restricting where an annotation can be placed prevents mistakes like accidentally putting a field-level annotation on a class or method. The compiler will catch the error immediately.

**Prefer `getAnnotationsByType()` over `getAnnotation()`** for repeatable annotations. `getAnnotation()` on a repeated annotation will return null (because what actually exists is the container annotation); `getAnnotationsByType()` handles the unwrapping correctly.

**Add `@Documented` to annotations intended for library/API use.** This ensures your users can see which annotations apply to which elements in the generated Javadoc, making the API self-describing.

**Annotation elements cannot have `null` default values.** If an element has no meaningful default, omit the default and require the annotation user to supply it explicitly — this makes missing values a compile error rather than a runtime surprise.

**Think carefully before using `@Inherited`.** It only applies to class-level annotations, not methods. If a subclass overrides a method from the parent, the method-level annotations from the parent are *not* inherited — the subclass method must re-declare them.
