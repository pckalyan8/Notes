# 📘 Phase 2 — Java Language Fundamentals
### The Absolute Bedrock — Master Every Detail Before Moving On

---

> **Why This Phase Is the Most Important**
>
> Every building has a foundation, and in Java, Phase 2 is yours. Every concept you'll ever encounter — Spring Boot, concurrency, design patterns, microservices — is built on top of the fundamentals covered here. Developers who rush through this phase and jump to frameworks without mastering data types, control flow, methods, and strings spend the rest of their careers making avoidable mistakes and writing code they don't fully understand. Give this phase the time it deserves. Read the examples carefully. Type them yourself, not copy-paste. Modify them and observe what changes. A week spent here properly will save you a month of confusion later.

---

## 2.1 — Java Program Structure

### The Big Picture: What Java Looks Like

Before any concept can make sense, you need to see the complete skeleton of a Java program and understand every part of it. Here is a fully annotated example:

```java
// This is the package declaration.
// It must be the FIRST statement in the file (comments don't count).
// It declares that this class belongs to the 'com.example.basics' namespace.
// The directory structure must mirror this: src/main/java/com/example/basics/
package com.example.basics;

// Import statements bring other classes into scope so you can use
// their short names. Without this import, you'd have to write
// java.util.ArrayList every single time you use it.
import java.util.ArrayList;
import java.util.List;

// This is a Javadoc comment. It generates API documentation.
// Convention: document every public class and every public method.
/**
 * Demonstrates the structure of a complete Java source file.
 * The filename MUST be ProgramStructure.java — it must match the public class name.
 */
public class ProgramStructure {

    // This is a class-level field (also called an instance variable or member variable).
    // Every object created from this class will have its own copy of 'instanceCount'.
    // 'private' means it's only accessible within this class.
    private int instanceCount = 0;

    // A static field belongs to the CLASS itself, not to any specific object.
    // There is only ONE copy of this variable, shared across all instances.
    private static int totalInstances = 0;

    // This is a constructor — a special method called when you create a new object.
    // It has no return type (not even void) and must share the class name exactly.
    public ProgramStructure() {
        this.instanceCount = ++totalInstances; // 'this' refers to the current object
    }

    // The main method — the entry point. The JVM calls this first.
    // It must be: public, static, void, named 'main', and accept String[]
    public static void main(String[] args) {

        // A 'block' is code wrapped in curly braces { }.
        // The method body is a block. If-statements, loops — all use blocks.

        // This is a statement — a complete unit of execution, ending with semicolon.
        System.out.println("Program started.");

        // Creating objects (instances) using the 'new' keyword
        ProgramStructure first = new ProgramStructure();
        ProgramStructure second = new ProgramStructure();

        // Calling a method on an object
        first.printInfo();
        second.printInfo();

    } // End of main method block

    // An instance method — can access 'this', instance fields, and static fields
    public void printInfo() {
        System.out.println("This is instance number: " + this.instanceCount);
        System.out.println("Total instances created: " + totalInstances);
    }

} // End of class block
```

Run this in your mind. When `main` executes, it creates two `ProgramStructure` objects. Each call to `new ProgramStructure()` increments `totalInstances` (shared) and assigns that value to `instanceCount` (per-object). So `first.instanceCount` = 1, `second.instanceCount` = 2, and `totalInstances` = 2 for both.

**Output:**
```
Program started.
This is instance number: 1
Total instances created: 2
This is instance number: 2
Total instances created: 2
```

---

### Comments: Communicating Intent

Java has three comment styles, and each has a specific purpose. Comments don't affect program behavior — the compiler ignores them — but they are crucial for communicating *why* code does what it does.

**Single-line comments** (`//`) are for brief inline explanations. Use them to explain something non-obvious about the very next line.

**Multi-line comments** (`/* ... */`) span multiple lines. Use them to temporarily disable blocks of code during debugging, or for longer explanations.

**Javadoc comments** (`/** ... */`) are processed by the `javadoc` tool to generate HTML API documentation — the same kind of documentation you see at `docs.oracle.com`. You'll see `@param` (describes a parameter), `@return` (describes what's returned), and `@throws` (describes exceptions that can be thrown) tags inside Javadoc comments.

```java
/**
 * Calculates the area of a rectangle.
 * This method validates inputs — passing a negative dimension throws an exception.
 *
 * @param width  The width of the rectangle in centimeters. Must be positive.
 * @param height The height of the rectangle in centimeters. Must be positive.
 * @return The area in square centimeters.
 * @throws IllegalArgumentException if width or height is not positive.
 */
public static double calculateArea(double width, double height) {
    if (width <= 0 || height <= 0) {
        // Fail fast: catch bad inputs immediately rather than returning garbage data
        throw new IllegalArgumentException(
            "Dimensions must be positive. Got: width=" + width + ", height=" + height
        );
    }
    return width * height;  // Straightforward multiplication
}
```

---

## 2.2 — Data Types & Variables

### The Two Worlds: Primitives and References

Java's type system is divided into two fundamentally different categories, and confusing them is one of the most common sources of bugs for beginners. Understanding them deeply will spare you enormous frustration.

**Primitive types** hold actual values directly in memory. When you write `int age = 25`, the variable `age` literally contains the number 25 stored in memory. Primitives are small, fast, and stored on the stack (for local variables).

**Reference types** hold memory addresses — pointers to objects living on the heap. When you write `String name = "Alice"`, the variable `name` doesn't contain the characters 'A', 'l', 'i', 'c', 'e'. It contains a memory address that points to a `String` object elsewhere in memory that contains those characters. This distinction explains a huge amount about how Java behaves.

---

### Primitive Types: The Eight Building Blocks

Java has exactly eight primitive types. Memorize them — they appear everywhere.

```java
public class PrimitiveTypes {
    public static void main(String[] args) {

        // --- INTEGER TYPES ---
        // The difference is storage size, which determines the range of values they can hold.

        byte b = 127;         // 8 bits: range -128 to 127. Used for raw binary data, files.
        short s = 32767;      // 16 bits: range -32,768 to 32,767. Rarely used.
        int i = 2_147_483_647; // 32 bits: range ~-2.1 billion to ~2.1 billion. DEFAULT choice.
        long l = 9_223_372_036_854_775_807L; // 64 bits: HUGE range. Note the 'L' suffix!

        // The underscore (_) in numeric literals is purely cosmetic — the compiler ignores it.
        // It's like a thousands separator: 1_000_000 is clearer than 1000000.
        int million = 1_000_000;
        long creditCardNumber = 1234_5678_9012_3456L;

        // --- FLOATING-POINT TYPES ---
        float f = 3.14f;       // 32 bits, ~7 decimal digits of precision. Note 'f' suffix.
        double d = 3.141592653589793; // 64 bits, ~15 decimal digits. DEFAULT choice for decimals.

        // CRITICAL WARNING: Never use float or double for money/financial calculations!
        // They cannot represent most decimal fractions exactly.
        // This is because they're stored in binary, and 0.1 in binary is a repeating fraction.
        double moneyBug = 0.1 + 0.2;
        System.out.println(moneyBug);  // Prints: 0.30000000000000004  ← NOT 0.3!
        // Use BigDecimal for financial calculations (covered in Phase 4).

        // --- CHARACTER TYPE ---
        char letter = 'A';      // 16 bits: stores a SINGLE Unicode character (not a String!)
        char unicode = '\u0041'; // Unicode escape: also 'A' (U+0041 is capital A)
        char newline = '\n';     // Escape sequences: newline, tab (\t), backslash (\\), etc.

        // A char is actually a number under the hood — the Unicode code point.
        // You can do arithmetic with chars (careful with this):
        char next = (char)(letter + 1); // 'A' + 1 = 'B'
        System.out.println(next);       // Prints: B

        // --- BOOLEAN TYPE ---
        boolean isActive = true;
        boolean hasPermission = false;
        // Booleans are NOT integers in Java (unlike C/C++)!
        // You CANNOT write: if (1) { } — it must be: if (true) { }
        // You CANNOT write: int x = isActive; — that's a type error.

        // Default values when declared as FIELDS (not local variables):
        // byte, short, int, long → 0
        // float, double         → 0.0
        // char                  → '\u0000' (null character)
        // boolean               → false
        // Reference types       → null
        // NOTE: Local variables have NO default value — using an uninitialized local
        //       variable is a COMPILE ERROR in Java. This is a safety feature.

    }
}
```

Think of primitive types like physical containers of different sizes: a `byte` is a tiny shot glass (holds a little), an `int` is a standard mug (holds enough for most things), and a `long` is a large pitcher (holds a lot). You choose based on what you need to hold.

---

### Numeric Literals in Different Bases

Java lets you write integer literals in decimal, hexadecimal, binary, and octal:

```java
int decimal     = 255;      // Base 10 — normal counting
int hexadecimal = 0xFF;     // Base 16 — prefix 0x. Digits: 0-9, A-F. 0xFF = 255
int binary      = 0b11111111; // Base 2 — prefix 0b. Digits: 0, 1. 0b11111111 = 255
int octal       = 0377;     // Base 8 — prefix 0 (leading zero). Digits: 0-7. 0377 = 255

// All four are the exact same value: 255.
System.out.println(decimal == hexadecimal); // true
System.out.println(decimal == binary);      // true

// Hexadecimal is most useful for bit manipulation, colors, memory addresses.
int red   = 0xFF0000;  // RGB red component
int green = 0x00FF00;  // RGB green component
int blue  = 0x0000FF;  // RGB blue component

// Binary literals are useful for bit flags and masks:
int readPermission    = 0b001; // binary 001 = decimal 1
int writePermission   = 0b010; // binary 010 = decimal 2
int executePermission = 0b100; // binary 100 = decimal 4
int allPermissions    = 0b111; // binary 111 = decimal 7
```

---

### Reference Types: Objects and Memory

Every type that isn't a primitive is a reference type — that means all classes, arrays, interfaces, and enums. The reference is the variable; the object is what it points to.

```java
public class ReferenceTypes {
    public static void main(String[] args) {

        // 'name1' is a REFERENCE VARIABLE stored on the stack.
        // The String object "Alice" lives on the HEAP.
        // name1 contains a memory address pointing to that object.
        String name1 = "Alice";

        // name2 is assigned the same reference as name1.
        // Both variables point to THE SAME object in memory.
        String name2 = name1;

        // Does this prove they're the same object? Let's check with ==
        // For reference types, == compares MEMORY ADDRESSES (identity), not content.
        System.out.println(name1 == name2);        // true — same object reference

        // What about two Strings with identical content but different objects?
        String a = new String("Hello"); // Forces creation of a new String object
        String b = new String("Hello"); // Creates ANOTHER new String object
        System.out.println(a == b);     // false — different objects in memory!
        System.out.println(a.equals(b));// true — .equals() compares CONTENT

        // This is a fundamental Java rule:
        // For reference types, use .equals() to compare VALUES.
        // Use == only when you specifically need to check if two variables
        // point to the exact same object.

        // The null reference: a reference that points to nothing.
        String notAssigned = null;

        // Calling ANY method on null causes NullPointerException (NPE),
        // the most famous Java error.
        // System.out.println(notAssigned.length()); // ← NPE! Don't do this.

        // Always check for null before using a reference:
        if (notAssigned != null) {
            System.out.println(notAssigned.length()); // Safe
        }

    }
}
```

The memory picture is crucial. Imagine two separate rooms: **Stack** and **Heap**. Local variables and their primitive values live in the Stack. All objects (created with `new` or string literals) live in the Heap. Reference variables in the Stack contain addresses pointing to objects in the Heap. When a method finishes, its Stack frame is removed. Objects in the Heap are cleaned up by the Garbage Collector when no more references point to them.

---

### Type Conversion: Widening and Narrowing

Java enforces that you cannot lose data accidentally. The rules are intuitive once you understand the concept.

**Widening conversion** (also called implicit conversion or promotion) happens automatically when you assign a smaller type to a larger type. No data can be lost — it's like pouring water from a small cup into a large pitcher. Java does this for you without asking.

**Narrowing conversion** (explicit casting) is when you go from a larger type to a smaller one. You might lose data — like trying to pour a full pitcher into a small cup. Java forces you to explicitly cast to acknowledge that you understand the risk.

```java
public class TypeConversion {
    public static void main(String[] args) {

        // --- WIDENING (Automatic) ---
        // Chain of widening: byte → short → int → long → float → double
        byte  byteVal  = 42;
        short shortVal = byteVal;   // byte → short: automatic, no data loss
        int   intVal   = shortVal;  // short → int: automatic
        long  longVal  = intVal;    // int → long: automatic
        float floatVal = longVal;   // long → float: automatic (minor precision loss possible)
        double doubleVal = floatVal;// float → double: automatic

        System.out.println(doubleVal); // 42.0 — widened all the way to double

        // int → long is always safe (no data loss possible)
        int population = 1_400_000_000; // China's population
        long worldPopulation = population; // Safe: just widens the storage

        // --- NARROWING (Requires explicit cast) ---
        // The cast operator is (targetType) before the value.
        // You're telling the compiler: "I know what I'm doing."

        double pi = 3.14159265;
        int truncated = (int) pi;       // Fractional part is DROPPED (not rounded!)
        System.out.println(truncated);  // 3 — not 3.14, not rounded to 3.14

        long bigNumber = 10_000_000_000L; // 10 billion — too big for int
        int overflow = (int) bigNumber;   // Data LOSS occurs!
        System.out.println(overflow);     // -1539607552 — meaningless garbage

        // This is exactly why Java requires explicit casting for narrowing:
        // it forces you to acknowledge that data loss CAN happen.

        // --- INTEGER PROMOTION ---
        // When you do arithmetic with byte or short, Java automatically promotes
        // them to int first. This is called integer promotion.
        byte x = 10;
        byte y = 20;
        // byte result = x + y;  // COMPILE ERROR! x+y produces an int, not a byte
        int result = x + y;      // This works: int = int + int
        byte result2 = (byte)(x + y); // This also works with explicit cast

        // --- PATTERN MATCHING WITH instanceof (Java 16+) ---
        // Old style (tedious and error-prone):
        Object obj = "Hello, World!";
        if (obj instanceof String) {
            String s = (String) obj; // Must cast manually after checking
            System.out.println(s.toUpperCase());
        }

        // New style (Java 16+ pattern matching — concise and safe):
        if (obj instanceof String s) { // Check AND bind to variable 's' in one step
            System.out.println(s.toUpperCase()); // 's' is already the right type
        }

    }
}
```

---

## 2.3 — Operators

### Arithmetic Operators

These work almost exactly as in mathematics, with one important exception: integer division.

```java
public class ArithmeticOperators {
    public static void main(String[] args) {

        int a = 17, b = 5;

        System.out.println(a + b);  // 22 — addition
        System.out.println(a - b);  // 12 — subtraction
        System.out.println(a * b);  // 85 — multiplication
        System.out.println(a / b);  // 3  — INTEGER division: fractional part is DROPPED!
        System.out.println(a % b);  // 2  — modulo: the remainder after integer division

        // CRITICAL: Integer division truncates toward zero.
        // 17 / 5 = 3 remainder 2. So 17 / 5 = 3, and 17 % 5 = 2.
        // Verify: 3 * 5 + 2 = 17. ✓

        // To get decimal division, at least ONE operand must be a double/float:
        System.out.println(17.0 / 5);    // 3.4 — one operand is double, so result is double
        System.out.println((double)a / b); // 3.4 — cast one operand

        // Modulo is incredibly useful. Common uses:
        // 1. Check if a number is even or odd
        int n = 14;
        System.out.println(n % 2 == 0 ? "even" : "odd"); // "even"

        // 2. Wrap around within a range (e.g., circular arrays, clocks)
        int hour = 25;
        System.out.println(hour % 24); // 1 — wraps around: hour 25 is hour 1

        // 3. Get the last N digits of a number
        int bigNum = 12345;
        System.out.println(bigNum % 100); // 45 — last two digits

        // Increment and Decrement operators
        int counter = 5;
        System.out.println(counter++); // prints 5, THEN increments → counter is now 6
        System.out.println(++counter); // increments first → counter is now 7, prints 7
        System.out.println(counter--); // prints 7, THEN decrements → counter is now 6
        System.out.println(--counter); // decrements first → counter is now 5, prints 5

        // The difference: post-increment (x++) returns the OLD value, then increments.
        // Pre-increment (++x) increments first, then returns the NEW value.
        // In isolation (counter++ on its own line), they're identical.
        // The difference only matters when you USE the expression's value.

    }
}
```

---

### Comparison and Logical Operators

```java
public class LogicalOperators {
    public static void main(String[] args) {

        int x = 10, y = 20;

        // Comparison operators — always return boolean
        System.out.println(x == y);  // false — equal to
        System.out.println(x != y);  // true  — not equal to
        System.out.println(x < y);   // true  — less than
        System.out.println(x > y);   // false — greater than
        System.out.println(x <= 10); // true  — less than or equal to
        System.out.println(y >= 20); // true  — greater than or equal to

        // Logical operators — combine boolean expressions
        boolean isAdult = true;
        boolean hasID = false;

        // && (AND): true ONLY if BOTH sides are true
        System.out.println(isAdult && hasID);  // false
        System.out.println(isAdult && true);   // true

        // || (OR): true if AT LEAST ONE side is true
        System.out.println(isAdult || hasID);  // true
        System.out.println(false || hasID);    // false

        // ! (NOT): flips the boolean value
        System.out.println(!isAdult);  // false
        System.out.println(!hasID);    // true

        // SHORT-CIRCUIT EVALUATION — a critical Java behavior!
        // With &&: if the LEFT side is false, the RIGHT side is NEVER evaluated.
        // With ||: if the LEFT side is true, the RIGHT side is NEVER evaluated.
        // This is not just an optimization — it prevents errors:

        String name = null;
        // This would throw NullPointerException: name.length() > 0 when name is null
        // But thanks to short-circuit evaluation, when name == null is true,
        // Java NEVER evaluates name.length() > 0, so no NPE occurs:
        if (name != null && name.length() > 0) {
            System.out.println("Name: " + name);
        } else {
            System.out.println("Name is null or empty");
        }

        // Similarly with ||: if the first condition is true, the second is skipped:
        int[] arr = null;
        if (arr == null || arr.length == 0) { // arr.length would NPE if arr were null,
            System.out.println("Array is null or empty"); // but arr==null short-circuits it
        }

        // & and | (non-short-circuit): BOTH sides are always evaluated.
        // Use these only when the right side HAS to run regardless (rare).
        System.out.println(isAdult & hasID);  // false — both sides evaluated
        System.out.println(isAdult | hasID);  // true  — both sides evaluated

        // Ternary operator: condition ? valueIfTrue : valueIfFalse
        int temperature = 22;
        String weather = temperature > 30 ? "hot" : "comfortable";
        System.out.println(weather); // "comfortable"

        // Ternary can be nested, but avoid deep nesting — it kills readability:
        String grade = (temperature >= 35) ? "scorching"
                     : (temperature >= 25) ? "warm"
                     : (temperature >= 15) ? "mild"
                     : "cold";
        System.out.println(grade); // "mild"

    }
}
```

---

### Bitwise Operators

Bitwise operators work directly on the individual bits of integers. They're used in low-level programming, cryptography, flags, and performance-critical code.

```java
public class BitwiseOperators {
    public static void main(String[] args) {

        // Think of these as working on the binary representation:
        // a = 12 = 0000 1100 in binary
        // b =  5 = 0000 0101 in binary

        int a = 12, b = 5;

        // & (AND): 1 only where BOTH bits are 1
        // 0000 1100
        // 0000 0101
        // --------- &
        // 0000 0100 = 4
        System.out.println(a & b); // 4

        // | (OR): 1 where EITHER bit is 1
        // 0000 1100
        // 0000 0101
        // --------- |
        // 0000 1101 = 13
        System.out.println(a | b); // 13

        // ^ (XOR — exclusive OR): 1 where bits are DIFFERENT
        // 0000 1100
        // 0000 0101
        // --------- ^
        // 0000 1001 = 9
        System.out.println(a ^ b); // 9

        // ~ (NOT — bitwise complement): flips every bit
        // ~12 = -(12 + 1) = -13 (due to two's complement representation)
        System.out.println(~a); // -13

        // << (left shift): shifts bits left, fills with 0s on the right
        // Equivalent to multiplying by 2^n (much faster than multiplication):
        System.out.println(1 << 3);  // 1 * 2^3 = 8
        System.out.println(5 << 2);  // 5 * 2^2 = 20

        // >> (right shift, signed): shifts bits right, preserves sign bit
        // Equivalent to dividing by 2^n (integer division):
        System.out.println(20 >> 2);  // 20 / 4 = 5
        System.out.println(-20 >> 2); // -5 (sign preserved: negative stays negative)

        // >>> (unsigned right shift): shifts right, fills with 0s (ignores sign)
        System.out.println(-1 >>> 1); // 2147483647 (Integer.MAX_VALUE): fills with 0

        // Practical use: permission/flag system using bit fields
        final int READ    = 0b001; // bit 0
        final int WRITE   = 0b010; // bit 1
        final int EXECUTE = 0b100; // bit 2

        int myPermissions = READ | WRITE; // = 0b011 = 3: has read and write

        // Check if a specific permission is set using AND:
        boolean canRead    = (myPermissions & READ)    != 0; // true
        boolean canWrite   = (myPermissions & WRITE)   != 0; // true
        boolean canExecute = (myPermissions & EXECUTE) != 0; // false

        System.out.println("Can read: " + canRead);
        System.out.println("Can write: " + canWrite);
        System.out.println("Can execute: " + canExecute);

    }
}
```

---

## 2.4 — Control Flow

Control flow determines the *order* in which statements execute. Without control flow, every program would just run from the first line to the last in sequence — it could never make decisions or repeat work.

### Conditionals: Making Decisions

```java
public class Conditionals {
    public static void main(String[] args) {

        // --- if / else if / else ---
        int score = 75;

        if (score >= 90) {
            System.out.println("Grade: A");
        } else if (score >= 80) {
            System.out.println("Grade: B");
        } else if (score >= 70) {
            System.out.println("Grade: C");
        } else if (score >= 60) {
            System.out.println("Grade: D");
        } else {
            System.out.println("Grade: F");
        }
        // Output: Grade: C

        // Java evaluates top-to-bottom and stops at the FIRST true condition.
        // Ordering matters — if you put score >= 60 first, a score of 95 would
        // match that condition and print "D" before ever reaching the 90 check.

        // Single-statement bodies can omit braces (but DON'T — it's a bad practice):
        // if (score > 50) System.out.println("Pass"); // Legal but risky
        // Always use braces, even for single-statement bodies. It prevents bugs
        // like Apple's famous "goto fail" SSL vulnerability.

        // --- Traditional switch statement ---
        // Good for equality checks against a single variable.
        // Works with: int, byte, short, char, String, enum (NOT long, float, double, boolean)
        String day = "WEDNESDAY";
        switch (day) {
            case "MONDAY":
            case "TUESDAY":
            case "WEDNESDAY":
            case "THURSDAY":
            case "FRIDAY":
                System.out.println("Weekday");
                break; // CRUCIAL: without 'break', execution FALLS THROUGH to next case!
            case "SATURDAY":
            case "SUNDAY":
                System.out.println("Weekend");
                break;
            default:
                System.out.println("Unknown day");
        }

        // The 'break' is one of Java's historical warts. Forgetting it causes
        // fall-through bugs that are notoriously hard to spot. The new switch
        // expression solves this problem.

        // --- Switch Expression (Java 14+) — the modern, preferred style ---
        // Uses arrow (->) syntax: NO fall-through, NO break needed, and it's an EXPRESSION
        // (meaning it returns a value).
        int month = 4; // April
        int daysInMonth = switch (month) {
            case 1, 3, 5, 7, 8, 10, 12 -> 31;  // Multiple cases in one line
            case 4, 6, 9, 11           -> 30;
            case 2                     -> 28;   // (ignoring leap year for simplicity)
            default -> throw new IllegalArgumentException("Invalid month: " + month);
        };
        System.out.println("Days in month " + month + ": " + daysInMonth); // 30

        // Notice: the switch expression is COMPLETE — every possible value is handled.
        // The compiler enforces this when using switch expressions.

        // When a case needs multiple statements, use a block with 'yield':
        String season = "SPRING";
        String description = switch (season) {
            case "SPRING" -> "Warm and rainy";
            case "SUMMER" -> "Hot and sunny";
            case "FALL"   -> {
                String base = "Cool and windy";
                yield base + " — leaves falling"; // 'yield' returns the value from a block
            }
            case "WINTER" -> "Cold and snowy";
            default       -> "Unknown season";
        };
        System.out.println(description);

        // Pattern matching in switch (Java 21) — can match on TYPES:
        Object obj = 42;
        String result = switch (obj) {
            case Integer i when i > 0 -> "Positive integer: " + i;
            case Integer i            -> "Non-positive integer: " + i;
            case String s             -> "String of length " + s.length();
            case null                 -> "null value";
            default                   -> "Something else: " + obj;
        };
        System.out.println(result); // "Positive integer: 42"

    }
}
```

---

### Loops: Repeating Work

```java
public class Loops {
    public static void main(String[] args) {

        // --- for loop: when you know exactly how many iterations you need ---
        // Structure: for (initialization; condition; update)
        // The three parts are: set up counter, check if we should continue, update counter
        for (int i = 0; i < 5; i++) {
            System.out.print(i + " "); // Prints: 0 1 2 3 4
        }
        System.out.println();

        // Counting backward:
        for (int i = 10; i > 0; i--) {
            System.out.print(i + " "); // 10 9 8 7 6 5 4 3 2 1
        }
        System.out.println();

        // Any or all parts of the for loop can be omitted:
        // for (;;) { } is an infinite loop (use with break to exit)

        // --- Enhanced for-each loop: iterating over arrays and collections ---
        // Simpler syntax when you don't need the index.
        int[] numbers = {10, 20, 30, 40, 50};
        for (int num : numbers) {
            System.out.print(num + " "); // 10 20 30 40 50
        }
        System.out.println();

        // --- while loop: when you don't know the number of iterations upfront ---
        // Checks the condition BEFORE executing the body.
        // If the condition is false from the start, the body never runs.
        int count = 1;
        while (count <= 5) {
            System.out.print(count + " "); // 1 2 3 4 5
            count++;
        }
        System.out.println();

        // Classic while use case: reading until a sentinel value
        // (Example with a fixed list for clarity)
        java.util.Scanner scanner = new java.util.Scanner(System.in);
        // while (scanner.hasNextInt()) {
        //     int value = scanner.nextInt();
        //     process(value);
        // }

        // --- do-while loop: when you need to execute the body AT LEAST ONCE ---
        // Checks the condition AFTER executing the body.
        int attempt = 0;
        do {
            System.out.println("Attempt #" + (++attempt));
            // In a real program, you'd prompt the user here and check their input
        } while (attempt < 3);
        // Output: Attempt #1, Attempt #2, Attempt #3

        // The do-while guarantees at least one execution, which is useful for
        // menu-driven programs: show the menu, get input, then check if user wants to quit.

        // --- break: exit the loop entirely ---
        for (int i = 0; i < 100; i++) {
            if (i == 5) break; // Stop the loop when i reaches 5
            System.out.print(i + " "); // 0 1 2 3 4
        }
        System.out.println();

        // --- continue: skip the rest of this iteration, go to next ---
        for (int i = 0; i < 10; i++) {
            if (i % 2 == 0) continue; // Skip even numbers
            System.out.print(i + " "); // 1 3 5 7 9 — only odd numbers
        }
        System.out.println();

        // --- Labeled break/continue: for nested loops ---
        // When you need to break out of multiple levels of nesting at once.
        // Labels are rarely needed but important to know.
        outer:  // This label names the outer loop
        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 3; j++) {
                if (i == 1 && j == 1) {
                    break outer; // Breaks out of BOTH loops, not just the inner one
                }
                System.out.println("i=" + i + ", j=" + j);
            }
        }
        // Output: i=0,j=0  i=0,j=1  i=0,j=2  i=1,j=0
        // (stops when i==1, j==1 because break outer exits the outer loop)

    }
}
```

---

## 2.5 — Methods

Methods are the fundamental unit of code organization in Java. A method is a named, reusable block of code that performs a specific task. Good methods do one thing, have a clear name that describes what that one thing is, and have well-defined inputs and outputs.

```java
public class Methods {

    // --- Method anatomy ---
    // access-modifier return-type methodName(parameterType paramName, ...) { body }

    // 'public': can be called from anywhere
    // 'static': belongs to the class, not an instance (so main() can call it without creating an object)
    // 'int': this method returns an integer value
    // 'add': the method name (verb or verb-phrase, camelCase)
    // 'int a, int b': parameters — variables local to this method, filled by caller
    public static int add(int a, int b) {
        return a + b; // 'return' statement sends a value back to the caller and exits the method
    }

    // void methods: do something but return no value
    public static void greet(String name) {
        System.out.println("Hello, " + name + "!");
        // No return statement needed (or you can write 'return;' to exit early)
    }

    // --- Method Overloading ---
    // Multiple methods CAN have the same name IF their parameter TYPES or COUNT differ.
    // The compiler picks the right one based on what arguments you pass.
    // This is called COMPILE-TIME polymorphism or static dispatch.

    public static double multiply(double a, double b) {
        return a * b;
    }

    public static int multiply(int a, int b) { // Same name, different parameter types
        return a * b;
    }

    public static int multiply(int a, int b, int c) { // Same name, different parameter count
        return a * b * c;
    }

    // --- Varargs (Variable-Length Arguments) ---
    // The '...' syntax means this parameter accepts ZERO OR MORE arguments of that type.
    // Inside the method, 'numbers' is treated as an int[].
    // There can only be ONE varargs parameter, and it must be the LAST parameter.
    public static int sum(int... numbers) {
        int total = 0;
        for (int n : numbers) {
            total += n;
        }
        return total;
    }

    // --- Pass by Value ---
    // Java ALWAYS passes by value. But for reference types, the "value" is the reference.
    // This is one of Java's most confusing concepts, so let's examine it carefully.

    public static void tryToChangeInt(int x) {
        x = 999; // This only modifies the LOCAL copy. The original is unaffected.
    }

    public static void modifyArray(int[] arr) {
        arr[0] = 999; // This modifies the OBJECT that 'arr' points to.
        // The caller's reference still points to the same array, so they SEE this change.
    }

    public static void tryToReplaceArray(int[] arr) {
        arr = new int[]{1, 2, 3}; // This makes the LOCAL 'arr' point to a new array.
        // The caller's reference is unchanged — they still point to the original array.
    }

    // --- Recursion ---
    // A method that calls ITSELF. Every recursive method needs:
    // 1. A BASE CASE: a condition that stops the recursion (prevents infinite loops)
    // 2. A RECURSIVE CASE: a call to itself with a SIMPLER/SMALLER input

    public static long factorial(int n) {
        // Base case: factorial(0) = 1, factorial(1) = 1
        if (n <= 1) return 1;
        // Recursive case: n! = n * (n-1)!
        return n * factorial(n - 1);
    }
    // factorial(5) = 5 * factorial(4)
    //              = 5 * 4 * factorial(3)
    //              = 5 * 4 * 3 * factorial(2)
    //              = 5 * 4 * 3 * 2 * factorial(1)
    //              = 5 * 4 * 3 * 2 * 1
    //              = 120

    public static void main(String[] args) {
        // Calling the methods
        System.out.println(add(3, 4));      // 7
        greet("Alice");                      // Hello, Alice!

        // Overloaded methods — compiler picks the right one:
        System.out.println(multiply(3, 4));       // 12 (calls int version)
        System.out.println(multiply(3.0, 4.0));   // 12.0 (calls double version)
        System.out.println(multiply(2, 3, 4));    // 24 (calls three-arg version)

        // Varargs — can pass any number of arguments:
        System.out.println(sum());            // 0 — zero arguments
        System.out.println(sum(1, 2, 3));     // 6
        System.out.println(sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)); // 55

        // Pass by value demonstration:
        int num = 10;
        tryToChangeInt(num);
        System.out.println(num); // Still 10 — unchanged

        int[] myArray = {1, 2, 3};
        modifyArray(myArray);
        System.out.println(myArray[0]); // 999 — the array's contents were changed

        tryToReplaceArray(myArray);
        System.out.println(myArray[0]); // Still 999 — the reference wasn't changed

        // Recursion:
        System.out.println(factorial(5));  // 120
        System.out.println(factorial(10)); // 3628800
        // Warning: factorial(20) exceeds long's range. Use BigInteger for big factorials.

    }
}
```

The "pass by value for references" concept trips up almost everyone initially. The clearest way to think about it: Java always copies the *variable*. If the variable is a primitive, you copy the value. If the variable is a reference, you copy the reference (the address). The object itself is not copied — both the original and the copy point to the same object. So you can modify the object's contents through either reference, but you can't make the caller's reference point somewhere else.

---

## 2.6 — Strings (Deep Dive)

Strings are perhaps the most-used type in all of Java programming. They deserve a thorough treatment.

### Immutability: The Most Important String Property

A `String` in Java is **immutable** — once created, its content *can never change*. Every method that seems to "modify" a String actually creates and returns a new String object, leaving the original untouched.

```java
public class StringImmutability {
    public static void main(String[] args) {

        String original = "Hello";

        // toUpperCase() does NOT modify 'original'. It returns a NEW String.
        String upper = original.toUpperCase();

        System.out.println(original); // "Hello" — unchanged!
        System.out.println(upper);    // "HELLO" — a new String object

        // This is the #1 beginner mistake with Strings:
        String s = "   trim me   ";
        s.trim();                        // This does NOTHING to s — the result is discarded!
        System.out.println(s);           // "   trim me   " — still has spaces!
        s = s.trim();                    // Reassign! Now s points to the trimmed String.
        System.out.println(s);           // "trim me"

        // Why is String immutable?
        // 1. THREAD SAFETY: Immutable objects can be shared between threads safely,
        //    with no risk of one thread changing a String while another reads it.
        // 2. STRING POOL EFFICIENCY: The JVM maintains a pool of String literals.
        //    Because Strings can't change, the same "Hello" literal can be safely
        //    shared across thousands of references without copying.
        // 3. SECURITY: Password strings, file paths, etc. can't be changed after creation,
        //    preventing a class of security vulnerabilities.
        // 4. HASH CODE CACHING: String's hashCode is cached on first computation.
        //    This makes Strings very efficient as HashMap keys.

    }
}
```

### The String Pool

```java
public class StringPool {
    public static void main(String[] args) {

        // String literals are stored in a special memory area called the String Pool (interned).
        // The JVM checks the pool before creating a new object.
        // If "Hello" is already in the pool, the existing object is reused.

        String a = "Hello"; // JVM creates "Hello" in the string pool
        String b = "Hello"; // JVM finds "Hello" already in the pool, returns the SAME object

        System.out.println(a == b);       // true — they're the SAME object!
        System.out.println(a.equals(b));  // true — same content

        // 'new String()' FORCES creation of a new String object on the heap,
        // BYPASSING the string pool. Almost never what you want.
        String c = new String("Hello"); // New object on heap, NOT in pool
        String d = new String("Hello"); // Another new object, different from c

        System.out.println(a == c);       // false — different objects!
        System.out.println(c == d);       // false — different objects!
        System.out.println(a.equals(c));  // true — same content
        System.out.println(c.equals(d));  // true — same content

        // The intern() method: manually add a String to the pool (or get the existing one)
        String e = c.intern(); // Returns the pool instance of "Hello"
        System.out.println(a == e); // true — e is now the pool instance

        // THE GOLDEN RULE: ALWAYS use .equals() to compare String content.
        // Use == only when you intentionally want to check object identity (rare).

        // Concatenation and the pool:
        String s1 = "Hello" + " World"; // Compile-time constant: stored in pool!
        String s2 = "Hello World";
        System.out.println(s1 == s2); // true — both are the same pool entry

        String part = "Hello";
        String s3 = part + " World"; // Runtime concatenation: NOT in pool (creates new object)
        System.out.println(s2 == s3); // false — s3 is a new heap object

    }
}
```

### Essential String Methods

```java
public class StringMethods {
    public static void main(String[] args) {

        String str = "  Hello, World!  ";

        // --- Length and Character Access ---
        System.out.println(str.length());       // 18 — includes spaces
        System.out.println(str.charAt(7));      // 'H' (0-indexed: spaces at 0,1; 'H' at 2... wait)
        // Let's use a cleaner string:
        String clean = "Hello, World!";
        System.out.println(clean.charAt(0));    // 'H'
        System.out.println(clean.charAt(7));    // 'W'

        // --- Searching ---
        System.out.println(clean.indexOf('o'));          // 4 — first 'o'
        System.out.println(clean.lastIndexOf('o'));      // 8 — last 'o'
        System.out.println(clean.indexOf("World"));      // 7 — index where "World" starts
        System.out.println(clean.contains("World"));     // true
        System.out.println(clean.startsWith("Hello"));   // true
        System.out.println(clean.endsWith("!"));         // true
        System.out.println(clean.indexOf("xyz"));        // -1 — not found, returns -1

        // --- Extracting Substrings ---
        // substring(beginIndex) — from beginIndex to end
        System.out.println(clean.substring(7));        // "World!"
        // substring(beginIndex, endIndex) — from beginIndex to endIndex (EXCLUSIVE)
        System.out.println(clean.substring(7, 12));    // "World" (index 7 to 11 inclusive)
        // Memory aid: endIndex is where the selection STOPS, not the last character taken.
        // "Hello, World!" — indices 0-12. substring(7,12) gives indices 7,8,9,10,11 = "World"

        // --- Transformation (all return NEW strings) ---
        System.out.println(clean.toUpperCase());        // "HELLO, WORLD!"
        System.out.println(clean.toLowerCase());        // "hello, world!"
        System.out.println(clean.replace('l', 'L'));    // "HeLLo, WorLd!" — char replacement
        System.out.println(clean.replace("World", "Java")); // "Hello, Java!" — substring replacement

        // --- Whitespace Handling ---
        String padded = "   hello   ";
        System.out.println(padded.trim());   // "hello" — removes ASCII whitespace (\t, \n, space)
        System.out.println(padded.strip());  // "hello" — Unicode-aware (preferred, Java 11+)
        System.out.println(padded.stripLeading());  // "hello   "
        System.out.println(padded.stripTrailing()); // "   hello"

        // --- Checking Content ---
        System.out.println("".isEmpty());         // true — zero length
        System.out.println("  ".isEmpty());       // false — has spaces, length is not 0
        System.out.println("  ".isBlank());       // true — contains only whitespace (Java 11+)
        System.out.println("  ".strip().isEmpty()); // true — the old way pre-Java 11

        // --- Splitting and Joining ---
        String csv = "apple,banana,cherry,date";
        String[] fruits = csv.split(","); // Split on comma — returns array
        for (String fruit : fruits) {
            System.out.print(fruit + " "); // apple banana cherry date
        }
        System.out.println();

        // Limit parameter: split into at most N parts
        String[] twoparts = csv.split(",", 2); // ["apple", "banana,cherry,date"]
        System.out.println(twoparts[0]); // "apple"
        System.out.println(twoparts[1]); // "banana,cherry,date"

        String joined = String.join(" | ", "apple", "banana", "cherry");
        System.out.println(joined); // "apple | banana | cherry"

        String joinedArray = String.join(", ", fruits);
        System.out.println(joinedArray); // "apple, banana, cherry, date"

        // --- Formatting ---
        String formatted = String.format("Name: %s, Age: %d, Score: %.2f", "Alice", 25, 98.567);
        System.out.println(formatted); // "Name: Alice, Age: 25, Score: 98.57"
        // Common format specifiers:
        // %s = String, %d = integer, %f = float/double, %.2f = float with 2 decimal places
        // %n = newline (platform-independent), %% = literal '%'
        // %5d = integer in at least 5 chars wide (right-aligned), %-5d = left-aligned

        // String.formatted() — instance method equivalent (Java 15+):
        String msg = "Hello, %s!".formatted("World");
        System.out.println(msg); // "Hello, World!"

        // --- Repeat and Lines (Java 11+) ---
        System.out.println("ha".repeat(3));   // "hahaha"
        "line1\nline2\nline3"
            .lines()                           // Splits into a Stream<String> of lines
            .forEach(System.out::println);     // prints each line separately

        // --- Conversions ---
        int number = 42;
        String fromInt = String.valueOf(number);   // "42" — converts anything to String
        String alsoFromInt = Integer.toString(number); // "42" — equivalent
        String concat = "" + number;               // "42" — concatenation trick (works but verbose)

        int backToInt = Integer.parseInt("42");    // "42" → 42
        double backToDouble = Double.parseDouble("3.14"); // "3.14" → 3.14
        // parseInt throws NumberFormatException if the String isn't a valid integer

        // --- Comparison ---
        String s1 = "apple", s2 = "Apple";
        System.out.println(s1.equals(s2));           // false — case-sensitive
        System.out.println(s1.equalsIgnoreCase(s2)); // true — case-insensitive
        System.out.println(s1.compareTo(s2));         // positive: 'a'(97) > 'A'(65)
        // compareTo returns negative if s1 < s2, 0 if equal, positive if s1 > s2
        // Used for sorting: Collections.sort() on Strings uses this internally

    }
}
```

### Text Blocks (Java 15+)

Text blocks solve the age-old problem of writing multi-line strings without cluttering them with `\n` and `+` operators:

```java
public class TextBlocks {
    public static void main(String[] args) {

        // Old way: JSON string was painful to write
        String jsonOld = "{\n" +
                         "  \"name\": \"Alice\",\n" +
                         "  \"age\": 30,\n" +
                         "  \"active\": true\n" +
                         "}";

        // Text Block (Java 15+): Three double-quotes to open and close.
        // The content's indentation is stripped relative to the closing """.
        // Newlines are preserved. Escape sequences work normally.
        String jsonNew = """
                {
                  "name": "Alice",
                  "age": 30,
                  "active": true
                }
                """;

        System.out.println(jsonNew);
        // Prints exactly:
        // {
        //   "name": "Alice",
        //   "age": 30,
        //   "active": true
        // }

        // Text blocks work great for SQL:
        String sql = """
                SELECT u.name, u.email, o.total
                FROM users u
                JOIN orders o ON u.id = o.user_id
                WHERE u.active = true
                  AND o.total > 100
                ORDER BY o.total DESC
                """;

        // And for HTML:
        String html = """
                <html>
                    <body>
                        <h1>Hello, World!</h1>
                    </body>
                </html>
                """;

        // .formatted() works with text blocks too:
        String name = "Bob";
        String greeting = """
                Dear %s,
                Thank you for your order.
                """.formatted(name);
        System.out.println(greeting);

    }
}
```

### StringBuilder: When You Need Mutable Strings

Because `String` is immutable, repeated concatenation in a loop creates a tsunami of throwaway objects:

```java
public class StringBuilderDemo {
    public static void main(String[] args) {

        // WRONG way — performance disaster for large N
        // Each '+' creates a NEW String object. For 10,000 iterations, this
        // creates ~10,000 intermediate String objects, most immediately garbage.
        String result = "";
        for (int i = 0; i < 10; i++) {
            result = result + i; // Creates new String each time!
        }
        System.out.println(result); // "0123456789"

        // CORRECT way — StringBuilder is mutable and efficient
        // It maintains an internal char array and only creates a String at the end.
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 10; i++) {
            sb.append(i); // Modifies the internal buffer — no new objects
        }
        System.out.println(sb.toString()); // "0123456789"

        // StringBuilder methods:
        StringBuilder builder = new StringBuilder("Hello");

        builder.append(", World");          // "Hello, World"
        builder.append("!");                // "Hello, World!"
        builder.insert(5, " Beautiful");    // "Hello Beautiful, World!" (insert at index 5)
        builder.delete(5, 15);             // "Hello, World!" (delete from 5 to 14)
        builder.replace(7, 12, "Java");    // "Hello, Java!" (replace chars 7-11)
        builder.reverse();                 // "!avaJ ,olleH"

        System.out.println(builder.length()); // Current length
        System.out.println(builder.charAt(0)); // '!'
        System.out.println(builder.toString()); // Convert to String when done

        // StringBuilder vs StringBuffer:
        // StringBuilder is NOT thread-safe (faster, use in single-threaded code)
        // StringBuffer IS thread-safe (slower, use when shared between threads — rare)
        // In practice, you almost always want StringBuilder.

        // Building a comma-separated list cleanly:
        String[] items = {"apple", "banana", "cherry"};
        StringBuilder csv = new StringBuilder();
        for (int i = 0; i < items.length; i++) {
            csv.append(items[i]);
            if (i < items.length - 1) {
                csv.append(", ");
            }
        }
        System.out.println(csv.toString()); // "apple, banana, cherry"

        // Actually, String.join() is cleaner for this specific case:
        System.out.println(String.join(", ", items)); // Same result, less code

    }
}
```

---

## 2.7 — Arrays

An array is a fixed-size, ordered collection of elements of the same type. "Fixed-size" is the key difference from `ArrayList` — once you create an array of size 10, it stays size 10 forever.

```java
import java.util.Arrays;

public class ArraysDemo {
    public static void main(String[] args) {

        // --- Declaration and Initialization ---

        // Declare only (no initial values — all elements get default value: 0 for int)
        int[] scores = new int[5]; // Creates array with 5 slots: [0, 0, 0, 0, 0]

        // Initialize with literal values (size is inferred from the values)
        int[] primes = {2, 3, 5, 7, 11}; // Size is 5

        // Declare and initialize separately:
        String[] names;
        names = new String[]{"Alice", "Bob", "Carol"}; // The new String[] part is needed here

        // Array access: 0-indexed. First element is [0], last is [length-1]
        System.out.println(primes[0]); // 2 — first element
        System.out.println(primes[4]); // 11 — last element
        // primes[5] would throw ArrayIndexOutOfBoundsException — there's no index 5!

        // Modifying an element:
        primes[2] = 99;
        System.out.println(Arrays.toString(primes)); // [2, 3, 99, 7, 11]

        // Length property (note: it's a property, not a method — no parentheses):
        System.out.println(primes.length); // 5

        // Iterating:
        for (int i = 0; i < primes.length; i++) {
            System.out.print(primes[i] + " ");
        }
        System.out.println();

        // Enhanced for-each (cleaner when you don't need the index):
        for (int prime : primes) {
            System.out.print(prime + " ");
        }
        System.out.println();

        // --- Multidimensional Arrays ---
        // A 2D array is an array of arrays — like a table/grid.
        int[][] matrix = new int[3][3]; // 3 rows, 3 columns, all zeros

        // Initialize with values:
        int[][] grid = {
            {1, 2, 3},  // row 0
            {4, 5, 6},  // row 1
            {7, 8, 9}   // row 2
        };

        System.out.println(grid[1][2]); // 6 — row 1, column 2 (0-indexed)

        // Nested loop to print 2D array:
        for (int row = 0; row < grid.length; row++) {         // grid.length = 3 (rows)
            for (int col = 0; col < grid[row].length; col++) { // grid[row].length = 3 (cols)
                System.out.printf("%3d", grid[row][col]);
            }
            System.out.println();
        }
        // Output:
        //   1  2  3
        //   4  5  6
        //   7  8  9

        // Jagged arrays: rows can have different lengths
        int[][] jagged = new int[3][];
        jagged[0] = new int[]{1};
        jagged[1] = new int[]{1, 2};
        jagged[2] = new int[]{1, 2, 3};
        // Useful for triangular tables, adjacency lists in graphs, etc.

        // --- Arrays Utility Class ---
        int[] data = {5, 3, 1, 4, 2};

        // Sort in ascending order (modifies the array IN PLACE):
        Arrays.sort(data);
        System.out.println(Arrays.toString(data)); // [1, 2, 3, 4, 5]

        // Binary search (array MUST be sorted first!):
        int idx = Arrays.binarySearch(data, 3);
        System.out.println(idx); // 2 — index where 3 was found

        // Fill: set all elements to a value
        int[] zeros = new int[5];
        Arrays.fill(zeros, 7);
        System.out.println(Arrays.toString(zeros)); // [7, 7, 7, 7, 7]

        // Copy: create a new array (copyOf pads with 0s or truncates as needed)
        int[] original = {1, 2, 3, 4, 5};
        int[] copy = Arrays.copyOf(original, 3);        // First 3 elements: [1, 2, 3]
        int[] extended = Arrays.copyOf(original, 7);    // Extends with 0s: [1, 2, 3, 4, 5, 0, 0]
        int[] range = Arrays.copyOfRange(original, 1, 4); // Index 1 to 3: [2, 3, 4]

        // Equality check (element-by-element):
        System.out.println(Arrays.equals(copy, new int[]{1, 2, 3})); // true

        // Readable string representation:
        System.out.println(Arrays.toString(original)); // [1, 2, 3, 4, 5]
        System.out.println(Arrays.deepToString(grid)); // [[1, 2, 3], [4, 5, 6], [7, 8, 9]]

        // --- Passing Arrays to Methods ---
        // Arrays are reference types! Passing to a method passes the reference.
        // The method can modify the array's content and the caller will see the changes.
        double[] temperatures = {36.6, 37.1, 38.0, 36.8};
        double avg = average(temperatures);
        System.out.println("Average temperature: " + avg); // 37.125

    }

    public static double average(double[] numbers) {
        double sum = 0;
        for (double n : numbers) sum += n;
        return sum / numbers.length;
    }
}
```

---

## 2.8 — Input & Output (Basic)

### Printing to the Console

```java
public class ConsoleOutput {
    public static void main(String[] args) {

        // Three standard output streams:
        System.out.println("With newline");     // prints and moves to next line
        System.out.print("No newline");         // prints and STAYS on same line
        System.out.println();                   // just a newline
        System.err.println("Error message");    // standard error stream (often appears in red)

        // printf / format: formatted output (same as String.format but prints directly)
        System.out.printf("%-15s %5d %8.2f%n", "Alice", 25, 98567.50);
        System.out.printf("%-15s %5d %8.2f%n", "Bob", 30, 75000.00);
        // Output (aligned):
        // Alice               25  98567.50
        // Bob                 30  75000.00

        // Format specifier anatomy: %[flags][width][.precision]type
        // Flags: - (left-align), + (show sign), 0 (zero-pad)
        // Width: minimum field width
        // Precision: decimal places for %f
        // Type: d (int), f (float), s (String), c (char), b (boolean), n (newline)

        System.out.printf("%+d%n", 42);    // "+42" — always show sign
        System.out.printf("%08d%n", 42);   // "00000042" — zero-padded to width 8
        System.out.printf("%-10s|%n", "hi"); // "hi        |" — left-aligned in 10 chars

    }
}
```

### Reading from the Console: Scanner

```java
import java.util.Scanner;

public class ConsoleInput {
    public static void main(String[] args) {

        // Scanner reads tokens from an input stream (System.in = keyboard)
        // Always close Scanner when done to release resources
        Scanner scanner = new Scanner(System.in);

        System.out.print("Enter your name: ");
        String name = scanner.nextLine();  // Reads an entire line (including spaces)

        System.out.print("Enter your age: ");
        int age = scanner.nextInt();       // Reads next integer token
        scanner.nextLine();                // IMPORTANT: consume the leftover newline!
        // After nextInt(), there's a '\n' left in the buffer.
        // If you call nextLine() next without consuming it first,
        // you'll get an empty string instead of the user's input.

        System.out.print("Enter your height (meters): ");
        double height = scanner.nextDouble(); // Reads next double
        scanner.nextLine();                   // Consume leftover newline again

        System.out.printf("Hello %s! Age: %d, Height: %.1fm%n", name, age, height);

        // Reading multiple values on one line (space-separated):
        System.out.print("Enter two numbers: ");
        int a = scanner.nextInt();
        int b = scanner.nextInt();
        System.out.println("Sum: " + (a + b));

        // Safe input reading with hasNext* checks:
        System.out.println("Enter integers (non-integer to stop):");
        int sum = 0;
        while (scanner.hasNextInt()) {   // Returns true if next token is an int
            sum += scanner.nextInt();
        }
        System.out.println("Sum: " + sum);

        scanner.close(); // Always close when done

        // Scanner key methods:
        // nextLine()   — reads until newline, returns String (includes spaces)
        // next()       — reads until whitespace, returns String (no spaces)
        // nextInt()    — reads next integer
        // nextDouble() — reads next double
        // nextLong()   — reads next long
        // nextBoolean()— reads "true" or "false"
        // hasNext()    — true if there's more input
        // hasNextInt() — true if next token can be read as int
        // hasNextLine()— true if there's another line

    }
}
```

### BufferedReader: Faster Input

For competitive programming or reading large volumes of input, `BufferedReader` is significantly faster than `Scanner` because it reads large chunks from the input stream at once instead of parsing token by token:

```java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;

public class BufferedReaderDemo {
    // Note: main must declare 'throws IOException' when using BufferedReader
    public static void main(String[] args) throws IOException {

        // Wrapping System.in in InputStreamReader (converts bytes to chars),
        // then wrapping that in BufferedReader (buffers the chars for efficiency).
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));

        System.out.print("Enter your name: ");
        String name = reader.readLine(); // Reads a full line

        System.out.print("Enter your age: ");
        int age = Integer.parseInt(reader.readLine()); // readLine() returns String; parse it

        System.out.printf("Hello %s! You are %d years old.%n", name, age);

        reader.close();

        // When to use which:
        // Scanner: simpler, handles different types directly (nextInt, nextDouble),
        //          good for user interaction and small inputs. Slower for huge inputs.
        // BufferedReader: faster for large volumes of input (competitive programming),
        //                 reads lines (you parse manually), requires IOException handling.

    }
}
```

---

## Putting It All Together: A Complete Phase 2 Example

To cement everything from this phase, here is a complete program that uses data types, control flow, methods, strings, arrays, and console I/O together:

```java
import java.util.Arrays;
import java.util.Scanner;

/**
 * A complete example program that processes student grades.
 * Demonstrates: data types, operators, control flow, methods, strings, arrays, I/O.
 */
public class GradeProcessor {

    // Named constants: use 'static final' for values that never change.
    // Convention: ALL_CAPS_WITH_UNDERSCORES for constants.
    private static final int NUM_STUDENTS = 5;
    private static final double PASS_THRESHOLD = 60.0;

    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        String[] names  = new String[NUM_STUDENTS];
        double[] grades = new double[NUM_STUDENTS];

        // Collect input
        System.out.println("=== Grade Processor ===");
        for (int i = 0; i < NUM_STUDENTS; i++) {
            System.out.print("Enter name for student " + (i + 1) + ": ");
            names[i] = scanner.nextLine();

            System.out.print("Enter grade for " + names[i] + " (0-100): ");
            grades[i] = scanner.nextDouble();
            scanner.nextLine(); // consume newline

            // Validate input range
            while (grades[i] < 0 || grades[i] > 100) {
                System.out.print("Invalid grade. Enter a value between 0 and 100: ");
                grades[i] = scanner.nextDouble();
                scanner.nextLine();
            }
        }

        // Process and display results
        System.out.println("\n=== Results ===");
        System.out.printf("%-15s %-8s %-6s%n", "Name", "Grade", "Pass?");
        System.out.println("-".repeat(30));

        for (int i = 0; i < NUM_STUDENTS; i++) {
            boolean passed = grades[i] >= PASS_THRESHOLD;
            String letterGrade = getLetterGrade(grades[i]);
            System.out.printf("%-15s %-8.1f %-6s (%s)%n",
                names[i], grades[i], passed ? "PASS" : "FAIL", letterGrade);
        }

        System.out.println("-".repeat(30));
        System.out.printf("Class average: %.2f%n", average(grades));
        System.out.printf("Highest grade: %.1f%n", max(grades));
        System.out.printf("Lowest grade:  %.1f%n", min(grades));

        scanner.close();
    }

    // Returns the letter grade for a given numeric score
    public static String getLetterGrade(double score) {
        return switch ((int)(score / 10)) {
            case 10, 9 -> "A";
            case 8     -> "B";
            case 7     -> "C";
            case 6     -> "D";
            default    -> "F";
        };
    }

    // Computes the average of an array of doubles
    public static double average(double[] values) {
        double sum = 0;
        for (double v : values) sum += v;
        return sum / values.length;
    }

    // Finds the maximum value in an array
    public static double max(double[] values) {
        double max = values[0]; // Start with first element as "best so far"
        for (double v : values) {
            if (v > max) max = v;
        }
        return max;
    }

    // Finds the minimum value in an array
    public static double min(double[] values) {
        double min = values[0];
        for (double v : values) {
            if (v < min) min = v;
        }
        return min;
    }
}
```

---

## Summary: What to Take From Phase 2

Phase 2 has given you the complete vocabulary of the Java language. You now understand that Java's type system is divided into primitives (which hold values) and references (which hold addresses), and that this distinction explains the `==` vs `.equals()` rule for Strings and objects. You understand that the JVM never gives you uninitialized local variables, which eliminates an entire category of bugs.

Control flow gives you the ability to make decisions and repeat work. The modern switch expression with arrow syntax is the preferred style for new code. Short-circuit evaluation in `&&` and `||` is not just an optimization — it's a safety mechanism you'll use in null checks every day.

Methods are your abstraction tool: name a piece of logic, give it inputs and an output, and you can use it anywhere without thinking about its implementation. The subtlety of "pass by value for references" is one of Java's trickiest concepts but makes complete sense once you visualize the Stack and Heap.

Strings are immutable — every "modification" creates a new String and must be reassigned. Use `StringBuilder` for building strings in loops. Use `.equals()` to compare content. Use `String.format()` or text blocks for multi-line and formatted strings.

With this foundation, Phase 3 — Object-Oriented Programming — will make immediate sense, because OOP is simply an extension of classes and methods that you already understand.

---

*Next: Phase 3 — Object-Oriented Programming (OOP)*
