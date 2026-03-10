# 4.6 Math & Number APIs in Java

## What & Why

Java's number-handling story has two distinct chapters: the `Math` class (and primitive arithmetic) for fast, standard computations, and `BigInteger` / `BigDecimal` for cases where precision or magnitude exceeds what primitive types can represent. A surprising number of bugs in financial and scientific software trace back to using the wrong type — using `double` for currency, for instance, produces rounding errors that accumulate across calculations. This section explains when each tool applies and how to use it correctly.

---

## 1. The Math Class

`java.lang.Math` is a utility class (all static methods, no instances) that wraps the native math library. It covers basic arithmetic, trigonometry, exponentiation, logarithms, rounding, and more.

```java
// Basic arithmetic helpers
System.out.println(Math.abs(-42));       // 42 — absolute value
System.out.println(Math.abs(-3.14));     // 3.14

System.out.println(Math.min(10, 20));    // 10
System.out.println(Math.max(10, 20));    // 20

// Rounding
System.out.println(Math.floor(3.9));     // 3.0 — rounds DOWN toward negative infinity
System.out.println(Math.ceil(3.1));      // 4.0 — rounds UP toward positive infinity
System.out.println(Math.round(3.5));     // 4   — rounds to nearest (half-up)
System.out.println(Math.round(3.49));    // 3

// Powers and roots
System.out.println(Math.pow(2, 10));     // 1024.0 — 2 to the 10th power
System.out.println(Math.sqrt(144));      // 12.0  — square root
System.out.println(Math.cbrt(27));       // 3.0   — cube root

// Logarithms
System.out.println(Math.log(Math.E));    // 1.0 — natural log (base e)
System.out.println(Math.log10(1000));    // 3.0 — log base 10
System.out.println(Math.log(8) / Math.log(2)); // 3.0 — log base 2 via change of base

// Trigonometry (arguments in RADIANS, not degrees)
System.out.println(Math.sin(Math.PI / 2));   // 1.0
System.out.println(Math.cos(0));             // 1.0
System.out.println(Math.toRadians(180));     // π ≈ 3.14159
System.out.println(Math.toDegrees(Math.PI)); // 180.0

// Useful constants
System.out.println(Math.PI);  // 3.141592653589793
System.out.println(Math.E);   // 2.718281828459045
```

### Exact Arithmetic Methods (Java 8+)

Standard `int` and `long` arithmetic silently overflows — the result wraps around without any error. The `Math.addExact`, `multiplyExact` family throws `ArithmeticException` instead of silently overflowing:

```java
// Silent overflow — wrong answer, no error
int silent = Integer.MAX_VALUE + 1;
System.out.println(silent); // -2147483648 — wrapped around!

// Exact methods — throws ArithmeticException on overflow
try {
    int safe = Math.addExact(Integer.MAX_VALUE, 1);
} catch (ArithmeticException e) {
    System.out.println("Overflow detected: " + e.getMessage());
}

Math.subtractExact(0, Integer.MIN_VALUE); // also throws — same overflow edge case
Math.multiplyExact(100_000, 100_000);     // fine: 10_000_000_000 would overflow int
                                           // but this returns long if you used longs
```

Use the exact methods in any business logic where overflow would produce silent wrong results (counters, financial calculations on primitive types).

---

## 2. Random Number Generation

Java provides three main classes for random numbers, each with different trade-offs:

```java
import java.util.Random;
import java.util.concurrent.ThreadLocalRandom;
import java.security.SecureRandom;

// --- java.util.Random ---
// Not thread-safe if shared across threads. Fine for single-threaded use.
Random random = new Random();               // seeds from current time
Random seeded  = new Random(12345L);       // deterministic seed — same sequence every run

int anyInt       = random.nextInt();       // any int value
int bounded      = random.nextInt(100);    // 0 to 99 (exclusive upper bound)
int ranged       = random.nextInt(50, 100); // Java 17+: 50 to 99 inclusive lower, exclusive upper
double d         = random.nextDouble();    // 0.0 to 1.0
boolean flip     = random.nextBoolean();   // true or false

// Stream-based generation (Java 8+)
random.ints(5, 1, 7)                       // 5 random ints in [1, 6]
    .forEach(System.out::println);

// --- ThreadLocalRandom ---
// The recommended choice in multi-threaded code — each thread gets its own Random instance,
// avoiding contention without the cost of synchronization.
int threadSafe = ThreadLocalRandom.current().nextInt(1, 101); // 1 to 100

// --- SecureRandom ---
// Cryptographically strong — use for passwords, tokens, session IDs, keys.
// Significantly slower than Random; don't use it for general-purpose generation.
SecureRandom crypto = new SecureRandom();
byte[] token = new byte[32];
crypto.nextBytes(token);  // fills with cryptographically random bytes
```

**The guiding rule:** Use `ThreadLocalRandom` for simulation, games, and general-purpose generation in concurrent code. Use `SecureRandom` for anything security-sensitive. Use `Random` with a fixed seed only when you need reproducible results in tests.

---

## 3. BigInteger — Arbitrary Precision Integers

`int` holds up to ~2.1 billion and `long` up to ~9.2 quintillion. When you need larger integers — cryptographic keys, factorial of large numbers, combinatorics — `BigInteger` handles any size, limited only by available memory.

`BigInteger` is **immutable**. Every arithmetic operation returns a new `BigInteger` object.

```java
import java.math.BigInteger;

// Creating BigIntegers
BigInteger fromLong   = BigInteger.valueOf(Long.MAX_VALUE);
BigInteger fromString = new BigInteger("123456789012345678901234567890");
BigInteger two        = BigInteger.TWO;   // constant (Java 9+)
BigInteger zero       = BigInteger.ZERO;
BigInteger one        = BigInteger.ONE;
BigInteger ten        = BigInteger.TEN;

// Arithmetic — note that all operations return NEW objects
BigInteger a = new BigInteger("99999999999999999999");
BigInteger b = new BigInteger("11111111111111111111");

BigInteger sum         = a.add(b);
BigInteger difference  = a.subtract(b);
BigInteger product     = a.multiply(b);
BigInteger quotient    = a.divide(b);
BigInteger remainder   = a.remainder(b);
BigInteger[] divAndRem = a.divideAndRemainder(b);  // both at once

BigInteger squared     = a.pow(2);    // a²
BigInteger gcd         = a.gcd(b);    // greatest common divisor
BigInteger absolute    = a.abs();
BigInteger negated     = a.negate();

// Bitwise operations
BigInteger shifted = a.shiftLeft(3);   // multiply by 8
BigInteger and     = a.and(b);

// Comparison — BigInteger implements Comparable
int cmp = a.compareTo(b); // negative, zero, or positive
boolean eq = a.equals(b); // value equality

// Practical example: factorial of 50 (too large for long)
BigInteger factorial = BigInteger.ONE;
for (int i = 2; i <= 50; i++) {
    factorial = factorial.multiply(BigInteger.valueOf(i));
}
System.out.println("50! = " + factorial);
// 30414093201713378043612608166979581188299763898377856000000000000

// Primality testing — useful in cryptography
BigInteger prime = BigInteger.probablePrime(512, new Random()); // 512-bit probable prime
System.out.println(prime.isProbablePrime(100)); // certainty 1 - 1/2^100
```

---

## 4. BigDecimal — Exact Decimal Arithmetic

This is the most practically important number type for any financial, accounting, or tax calculation. The fundamental issue with `double` is that most decimal fractions cannot be represented exactly in binary floating-point:

```java
// The classic floating-point problem
System.out.println(0.1 + 0.2);            // 0.30000000000000004 — NOT 0.3!
System.out.println(1.0 - 0.9);            // 0.09999999999999998 — NOT 0.1!

// BigDecimal is exact for decimal values
import java.math.BigDecimal;
import java.math.RoundingMode;

BigDecimal price    = new BigDecimal("0.10"); // ALWAYS use String constructor for literals
BigDecimal discount = new BigDecimal("0.20");
System.out.println(price.add(discount));       // 0.30 — exact!
```

**Critical rule:** Never use `new BigDecimal(0.1)` — the `double` 0.1 is already imprecise by the time it reaches the `BigDecimal` constructor. Always use `new BigDecimal("0.1")` or `BigDecimal.valueOf(0.1)` (which converts via `Double.toString` and is safer).

```java
// Wrong — the double 0.1 is already imprecise
BigDecimal wrong = new BigDecimal(0.1);
System.out.println(wrong); // 0.1000000000000000055511151231257827021181583404541015625 !!

// Correct — use String or valueOf
BigDecimal correct = new BigDecimal("0.1");   // exact
BigDecimal also    = BigDecimal.valueOf(0.1); // safe via Double.toString
```

### Scale and Precision

`BigDecimal` has two key properties: **precision** (total significant digits) and **scale** (digits to the right of the decimal point). These determine how arithmetic results are shaped.

```java
BigDecimal price    = new BigDecimal("19.99");
BigDecimal taxRate  = new BigDecimal("0.08");

// multiply() — scale is sum of both scales
BigDecimal tax      = price.multiply(taxRate);
System.out.println(tax);      // 1.5992 — 4 decimal places (2 + 2)

// setScale() — round to a specific number of decimal places
BigDecimal taxRounded = tax.setScale(2, RoundingMode.HALF_UP);
System.out.println(taxRounded); // 1.60

BigDecimal total  = price.add(taxRounded);
System.out.println(total); // 21.59
```

### Rounding Modes

`RoundingMode` is an enum that specifies how to round when the result requires fewer digits than the exact value:

```java
BigDecimal val = new BigDecimal("2.5");

System.out.println(val.setScale(0, RoundingMode.UP));         // 3 — away from zero
System.out.println(val.setScale(0, RoundingMode.DOWN));       // 2 — toward zero (truncate)
System.out.println(val.setScale(0, RoundingMode.CEILING));    // 3 — toward +infinity
System.out.println(val.setScale(0, RoundingMode.FLOOR));      // 2 — toward -infinity
System.out.println(val.setScale(0, RoundingMode.HALF_UP));    // 3 — school rounding
System.out.println(val.setScale(0, RoundingMode.HALF_DOWN));  // 2 — rounds 0.5 toward zero
System.out.println(val.setScale(0, RoundingMode.HALF_EVEN));  // 2 — banker's rounding
```

`HALF_EVEN` (banker's rounding) rounds 0.5 to the nearest *even* digit, which eliminates statistical bias in large datasets. It's the standard in financial systems.

### Division

Division is special — `BigDecimal` will throw `ArithmeticException` if the result is a non-terminating decimal (like 1/3) and you haven't specified a scale:

```java
BigDecimal one   = new BigDecimal("1");
BigDecimal three = new BigDecimal("3");

// This throws ArithmeticException: Non-terminating decimal expansion
// BigDecimal result = one.divide(three);

// Always specify scale and rounding mode for division
BigDecimal result = one.divide(three, 10, RoundingMode.HALF_UP);
System.out.println(result); // 0.3333333333
```

### Comparison

`equals()` on `BigDecimal` considers scale — `2.0` and `2.00` are **not** equal by `equals()`. Use `compareTo()` for value equality:

```java
BigDecimal a = new BigDecimal("2.0");
BigDecimal b = new BigDecimal("2.00");

System.out.println(a.equals(b));      // false — different scales
System.out.println(a.compareTo(b));   // 0 — same numeric value

// In collections, if you need BigDecimal keys in a TreeMap/TreeSet,
// compareTo is the sorting/deduplication mechanism anyway.
// For HashSet/HashMap keys, the scale inconsistency can cause bugs.
```

---

## 5. A Complete Financial Calculation Example

```java
import java.math.BigDecimal;
import java.math.RoundingMode;

public class Invoice {

    private static final RoundingMode ROUNDING = RoundingMode.HALF_EVEN;
    private static final int SCALE = 2;

    public static BigDecimal calculateTotal(
            BigDecimal unitPrice,
            int quantity,
            BigDecimal discountPct,
            BigDecimal taxRatePct) {

        // Step 1: subtotal = unitPrice × quantity
        BigDecimal subtotal = unitPrice.multiply(BigDecimal.valueOf(quantity));

        // Step 2: discount amount
        BigDecimal discount = subtotal
            .multiply(discountPct)
            .divide(BigDecimal.valueOf(100), SCALE, ROUNDING);

        // Step 3: discounted price
        BigDecimal afterDiscount = subtotal.subtract(discount);

        // Step 4: tax amount
        BigDecimal tax = afterDiscount
            .multiply(taxRatePct)
            .divide(BigDecimal.valueOf(100), SCALE, ROUNDING);

        // Step 5: total
        return afterDiscount.add(tax).setScale(SCALE, ROUNDING);
    }

    public static void main(String[] args) {
        BigDecimal total = calculateTotal(
            new BigDecimal("49.99"),  // unit price
            3,                        // quantity
            new BigDecimal("10"),     // 10% discount
            new BigDecimal("8")       // 8% tax
        );
        System.out.println("Total: $" + total); // Total: $145.77
    }
}
```

---

## ⚡ Best Practices & Important Points

**Use `BigDecimal` for all monetary values without exception.** The accumulated error from `double` arithmetic in financial calculations is not theoretical — it causes real discrepancies in cent amounts that compound over large transaction volumes.

**Use the `String` constructor or `BigDecimal.valueOf()` for decimal literals.** `new BigDecimal(0.1)` silently introduces the imprecision of binary floating-point before `BigDecimal` can do anything about it.

**Always specify scale and `RoundingMode` when calling `divide()`.** Failing to do so causes `ArithmeticException` for any non-terminating result.

**Use `compareTo()` not `equals()` to compare `BigDecimal` values.** The scale difference makes `equals()` unreliable for value comparison. If you store `BigDecimal` in a `HashSet` or as a `HashMap` key, normalise the scale first with `stripTrailingZeros()`.

**Prefer `ThreadLocalRandom` over `new Random()` in concurrent code.** Sharing a single `Random` instance across threads causes contention on the internal state, which degrades performance. `ThreadLocalRandom` is purpose-built for this.

**Use `Math.addExact()` and friends whenever integer overflow is a genuine risk** — for instance in counters that could theoretically exceed `Integer.MAX_VALUE`, or in algorithms that multiply user-supplied values.
