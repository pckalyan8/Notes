# 4.3 Exception Handling in Java

## What & Why

Every program eventually encounters situations it didn't plan for: a file doesn't exist, a network connection drops, someone passes a null where a value was expected. Java's exception handling gives you a principled way to detect, report, and recover from these situations without littering your main logic with defensive `if`-checks everywhere.

The core idea is **separation of concerns** — your happy-path code stays clean, and the error-handling code sits in designated `catch` blocks. When something goes wrong deep inside a call stack, the exception *propagates upward* automatically until something handles it or the program prints a stack trace and exits.

---

## 1. The Exception Hierarchy

Every throwable thing in Java extends `Throwable`. From there, the tree splits into two branches:

```
Throwable
├── Error                         — serious JVM problems; never catch these
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── AssertionError
└── Exception                     — recoverable problems
    ├── IOException               — checked: must be declared or caught
    ├── SQLException              — checked
    ├── ClassNotFoundException    — checked
    └── RuntimeException          — unchecked: optional to declare/catch
        ├── NullPointerException
        ├── ArrayIndexOutOfBoundsException
        ├── ClassCastException
        ├── IllegalArgumentException
        ├── IllegalStateException
        └── UnsupportedOperationException
```

The pivotal distinction is **checked vs. unchecked**:

A **checked exception** is one the compiler forces you to acknowledge. Any method that might throw a checked exception must either catch it with `try/catch` or declare it with `throws`. This is Java's way of saying "this failure mode is realistic; caller, you must plan for it."

An **unchecked exception** (any `RuntimeException` or `Error`) represents a programming error or a condition so unrecoverable that forcing the caller to handle it would create noise rather than safety. The compiler doesn't require you to acknowledge them.

```java
// Checked — compiler REQUIRES you to handle or declare this
public void readFile(String path) throws IOException {   // declared with throws
    Files.readAllLines(Path.of(path));                  // throws IOException
}

// Unchecked — no compile-time requirement; just good to be aware of it
public int divide(int a, int b) {
    return a / b;  // throws ArithmeticException if b == 0; no need to declare
}
```

---

## 2. try / catch / finally

The `try` block wraps code that might throw. The `catch` block handles a specific type (or a group of types). The `finally` block runs unconditionally — whether or not an exception occurred — making it perfect for cleanup.

```java
public class ExceptionDemo {

    public static String readFirstLine(String path) {
        BufferedReader reader = null;
        try {
            reader = new BufferedReader(new FileReader(path));
            return reader.readLine(); // may throw IOException

        } catch (FileNotFoundException e) {
            // More specific exception — handle it differently
            System.err.println("File not found: " + path);
            return null;

        } catch (IOException e) {
            // Broader IOException — covers other I/O problems
            System.err.println("Error reading file: " + e.getMessage());
            return null;

        } finally {
            // This ALWAYS runs — perfect for releasing resources
            if (reader != null) {
                try {
                    reader.close(); // close() itself can throw IOException
                } catch (IOException ignored) { }
            }
        }
    }
}
```

**Catch order matters.** Always list more specific exceptions before more general ones. If you catch `IOException` before `FileNotFoundException`, the compiler will complain that the `FileNotFoundException` catch block is unreachable because `FileNotFoundException` is a subtype of `IOException`.

### Multi-Catch (Java 7+)

When two unrelated exceptions need identical handling, multi-catch avoids duplication:

```java
try {
    // code that might throw either of these
} catch (IOException | SQLException e) {
    // handles both the same way
    log.error("Data access error", e);
    throw new DataAccessException("Failed to read data", e);
}
```

The caught variable `e` is implicitly `final` in a multi-catch — you cannot reassign it.

---

## 3. try-with-resources (Java 7+)

The pattern in the example above — opening a resource, using it, and closing it in `finally` — is so common that Java 7 gave it first-class syntax. Any class that implements `AutoCloseable` (which `Closeable` extends) can be opened in a `try(...)` declaration and will be **automatically closed** when the block exits, even if an exception is thrown.

```java
// Clean, safe, and handles close() exceptions correctly
public static String readFirstLine(String path) throws IOException {
    try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
        return reader.readLine();
        // reader.close() is called automatically here
    }
    // No finally needed!
}

// Multiple resources — closed in REVERSE order of declaration
public static void copyFile(String src, String dst) throws IOException {
    try (InputStream  in  = new FileInputStream(src);
         OutputStream out = new FileOutputStream(dst)) {
        in.transferTo(out);
        // out.close() is called first, then in.close()
    }
}
```

**Suppressed exceptions:** If the `try` block throws an exception and `close()` also throws, Java attaches the close exception as a *suppressed exception* on the primary one. You can inspect them with `e.getSuppressed()`. This is far better than the old `finally` approach, which would silently discard the original exception.

---

## 4. Exception Chaining

When you catch an exception deep in the stack and re-throw it as a higher-level exception, you should **always preserve the original cause** so that the full chain is visible in the stack trace:

```java
public User loadUser(long id) {
    try {
        return userRepository.findById(id);
    } catch (SQLException e) {
        // Wrap in a domain-level exception, preserving the original cause
        throw new UserNotFoundException("User not found: " + id, e);
        //                                                        ^ original cause
    }
}
```

The caller sees a clean `UserNotFoundException`, but if they call `e.getCause()`, they get the original `SQLException` — full context, no information lost.

---

## 5. Custom Exceptions

Creating your own exception types lets you build a rich, descriptive error vocabulary for your application.

```java
// Checked custom exception — caller must handle or declare it
public class InsufficientFundsException extends Exception {

    private final double amount;  // carry domain-specific context

    public InsufficientFundsException(double amount) {
        super(String.format("Insufficient funds: attempted to withdraw %.2f", amount));
        this.amount = amount;
    }

    public InsufficientFundsException(double amount, Throwable cause) {
        super(String.format("Insufficient funds: %.2f", amount), cause);
        this.amount = amount;
    }

    public double getAmount() { return amount; }
}

// Unchecked custom exception — no declaration needed by callers
public class InvalidOrderStateException extends RuntimeException {

    public InvalidOrderStateException(String message) {
        super(message);
    }

    public InvalidOrderStateException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

Using them:

```java
public class BankAccount {

    private double balance;

    public void withdraw(double amount) throws InsufficientFundsException {
        if (amount > balance) {
            throw new InsufficientFundsException(amount);
        }
        balance -= amount;
    }
}

// Calling code
BankAccount account = new BankAccount();
try {
    account.withdraw(500.0);
} catch (InsufficientFundsException e) {
    System.err.printf("Could not withdraw: %.2f%n", e.getAmount());
}
```

---

## 6. Re-throwing Exceptions

Sometimes a `catch` block needs to log or augment the exception but not fully handle it. Re-throwing is appropriate:

```java
public void processOrder(Order order) throws OrderProcessingException {
    try {
        paymentService.charge(order);
        inventoryService.reserve(order);

    } catch (PaymentException e) {
        auditLog.log("Payment failed for order: " + order.getId());
        throw new OrderProcessingException("Payment failed", e);  // re-throw wrapped

    } catch (RuntimeException e) {
        // You can also re-throw the same exception after logging
        log.error("Unexpected error processing order", e);
        throw e;  // rethrow as-is
    }
}
```

---

## 7. Helpful NullPointerExceptions (Java 14+)

Since Java 14, the JVM produces far more informative `NullPointerException` messages that tell you exactly *which* variable was null in a chain of dereferences:

```
Cannot invoke "String.length()" because "order.getCustomer().getName()" is null
```

This makes debugging much faster without any changes to your code.

---

## ⚡ Best Practices & Important Points

**Never swallow exceptions silently.** An empty `catch` block is among the worst things you can do — the error disappears, and you're left debugging a mysteriously broken state with no information. At minimum, log the exception.

```java
// Terrible — hides all errors
try { riskyOperation(); } catch (Exception e) { }

// Acceptable minimum — at least you know it happened
try { riskyOperation(); } catch (Exception e) { log.error("Operation failed", e); }
```

**Don't catch `Exception` or `Throwable` broadly** unless you are at the very top of your application (a framework's entry point or a thread's `run()` method) and need to ensure the process stays alive. Broad catches hide programming errors that should surface and be fixed.

**Prefer unchecked exceptions for programming errors**, and checked exceptions for expected, recoverable situations the caller should meaningfully handle. If you're not sure the caller *can* do anything useful with the exception, make it unchecked. Many modern frameworks (Spring, Hibernate) follow this philosophy.

**Always provide both a message-only and a cause-accepting constructor** in your custom exceptions. This ensures callers can do proper exception chaining.

**Use try-with-resources for every `Closeable` resource.** Forgetting to close streams, database connections, and network sockets causes resource leaks that are devilishly hard to diagnose in production.

**The `finally` block runs even after a `return` statement.** This surprises newcomers. The method returns the value it computed in `try`, but `finally` still executes before control actually leaves the method.

```java
int example() {
    try {
        return 1;        // prepared to return 1
    } finally {
        System.out.println("finally runs!"); // still executes
        // if you return here, it OVERRIDES the try's return
    }
}
// Prints "finally runs!" then returns 1
```

**Do not use exceptions for flow control.** Using `try/catch` as an `if/else` replacement (catching an exception you could have prevented with a simple check) is slow — creating an exception captures the entire stack trace, which is expensive.
