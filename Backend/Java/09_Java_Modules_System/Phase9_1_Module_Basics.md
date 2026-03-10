# Phase 9.1 — Module Basics: The Java Platform Module System (JPMS)

> **What is JPMS?**
> The Java Platform Module System (Project Jigsaw) was introduced in Java 9 as the most significant
> structural change to the Java platform since its inception. It adds a new layer of encapsulation
> *above* packages — the **module** — that lets you explicitly declare what your code depends on and
> what parts of it are accessible to the outside world. Understanding *why* it was needed is just as
> important as understanding *how* it works.

---

## The Problem JPMS Solves — JAR Hell

Before Java 9, the Java runtime loaded all classes from a flat **classpath** — a long list of JARs
and directories. This worked fine for small applications but caused serious problems at scale.

### Problem 1 — No True Encapsulation at the Package Level

Suppose you write a library and mark your internal helper package `com.mylib.internal` with the
intention that nobody outside your library should use it. Before JPMS, this was just a naming
convention — nothing actually enforced it. Any code on the classpath could call
`new com.mylib.internal.HelperClass()` freely. This made it impossible to refactor internals
without risking breaking someone who was depending on them, and it's exactly why changing the
`sun.misc.Unsafe` class in older JDK versions was so difficult.

### Problem 2 — No Versioning or Duplicate Class Resolution

If two JARs on the classpath contain the same fully-qualified class name, the JVM silently picks
the first one it finds. The second one is simply invisible. This is the core of the problem known
as the "JAR hell" or "classpath hell" — in large enterprise applications with hundreds of
transitive dependencies, conflicting versions of the same library were a constant source of
mysterious `NoClassDefFoundError` and `ClassCastException` failures.

### Problem 3 — Monolithic JDK

The entire JDK was a single monolith. Even a tiny microservice or embedded device had to carry
the full JDK — including Swing, CORBA, XML APIs, and other components it would never use. There
was no way to say "I only need the core language and networking."

### The JPMS Solution

JPMS addresses all three by introducing **modules**, which are named, self-describing units that:

1. **Explicitly declare what they require** from other modules (no silent class stealing)
2. **Explicitly declare what they export** to other modules (true encapsulation — packages not
   exported are completely invisible to the outside, even via reflection)
3. **Allow the JDK itself to be split into modules**, making custom, minimal JRE images possible

---

## What Is a Module?

A module is a **named grouping of packages**, accompanied by a module descriptor file called
`module-info.java`, placed at the root of the module's source tree. Think of it as a package's
big brother — it describes not only what packages exist, but also which ones are visible to others,
and which other modules it depends on.

A module has:
- A **unique name** (typically following reverse domain notation, e.g., `com.mycompany.orders`)
- A **module descriptor** (`module-info.java`) that declares its dependencies and exports
- A set of **packages** it contains

The **module path** is the JPMS-aware replacement for the classpath. Where the classpath is a
flat bag of JARs, the module path is a structured list of modular JARs or directories, each
containing a `module-info.class`.

---

## The `module-info.java` File

This is the heart of JPMS. Every named module must have exactly one `module-info.java` file at
the root of the module source. It is a special Java compilation unit — it does not declare a
package, and its syntax uses a small set of dedicated keywords.

Here is the full syntax with every directive:

```java
// file: src/com.myapp.orders/module-info.java
// (placed at the ROOT of the module source tree, NOT inside a package folder)

module com.myapp.orders {

    // ─── DEPENDENCIES ────────────────────────────────────────────────────────

    // requires: this module needs com.myapp.domain to compile and run
    requires com.myapp.domain;

    // requires transitive: anyone who requires com.myapp.orders
    // automatically also gets com.myapp.utils — they can use its types in our API
    requires transitive com.myapp.utils;

    // requires static: needed at COMPILE time only, optional at RUNTIME
    // useful for annotation processors, optional features
    requires static com.myapp.annotations;

    // ─── EXPORTS ─────────────────────────────────────────────────────────────

    // exports: makes this package's public types accessible to ALL other modules
    exports com.myapp.orders.api;

    // exports ... to: qualified export — only visible to the listed module(s)
    // Great for internal frameworks where you control all consumers
    exports com.myapp.orders.internal to com.myapp.orderstest, com.myapp.admin;

    // ─── OPENS (for reflection) ───────────────────────────────────────────────

    // opens: allows DEEP reflection (including private members) on this package
    // at runtime — but NOT normal compile-time access
    // Required by frameworks like Spring, Hibernate, Jackson that use reflection
    opens com.myapp.orders.model;

    // opens ... to: qualified open — reflection allowed only by named modules
    opens com.myapp.orders.config to spring.core, com.fasterxml.jackson.databind;

    // ─── SERVICES (ServiceLoader SPI) ────────────────────────────────────────

    // uses: this module is a SERVICE CONSUMER — it will look up implementations
    // of the given interface via java.util.ServiceLoader at runtime
    uses com.myapp.orders.spi.ShippingProvider;

    // provides ... with: this module is a SERVICE PROVIDER — it registers
    // this implementation for the given interface
    provides com.myapp.orders.spi.ShippingProvider
        with com.myapp.orders.shipping.StandardShippingProvider,
             com.myapp.orders.shipping.ExpressShippingProvider;
}
```

Each directive has a precise meaning. Let's walk through each one with real context.

---

## Directive Deep Dive

### `requires` — Declaring Dependencies

`requires` is how a module declares that it cannot run without another module being present.
The compiler and runtime both enforce this — if the required module is missing from the module
path, you get a clear, early error rather than a mysterious `NoClassDefFoundError` at runtime.

```java
// A "greeter" module that depends on the logging module
module com.myapp.greeter {
    requires com.myapp.logging;   // must be on the module path at compile and runtime
}
```

`requires transitive` is a special form. When your module's **public API exposes types from
another module**, you must declare `requires transitive` so your callers can also use those types
without having to declare their own `requires`.

```java
// If OrderService's public method returns Money from the "finance" module:
// public Money calculateTotal(Order order);
// Then any caller of our module also needs to be able to use Money.
// We declare transitive so our API "re-exports" the finance module's visibility.
module com.myapp.orders {
    requires transitive com.myapp.finance;   // callers inherit this dependency
}
```

The rule of thumb: if a type from module X appears in your public API (method signatures,
field types, supertype), then `requires transitive X`. Otherwise, plain `requires X`.

`requires static` means the module is needed at compile time but is treated as absent at runtime.
This is useful for annotation processors (like Lombok), optional integrations, or APIs that exist
in the compile environment but may not be deployed with the application.

```java
module com.myapp.service {
    requires static lombok;   // Lombok only needed during compilation, not at runtime
}
```

Every module implicitly `requires java.base` — you never need to declare it.

---

### `exports` — Controlling What Is Visible

This is JPMS's most powerful feature. A package inside a module is **completely inaccessible** to
the outside world unless it is explicitly exported. Not just the public types — the entire package.
This is a stronger guarantee than any access modifier ever provided.

```java
module com.myapp.payments {
    // Only this package is public API — everything else is invisible
    exports com.myapp.payments.api;

    // com.myapp.payments.internal — NOT exported — completely sealed
    // com.myapp.payments.db       — NOT exported — completely sealed
    // com.myapp.payments.crypto   — NOT exported — completely sealed
}
```

If code in another module tries to import from `com.myapp.payments.internal`, the **compiler
itself refuses**, even if every class in that package is declared `public`. This is a revolution
compared to the old "please don't use this" naming convention.

**Qualified exports** restrict visibility to a specific set of trusted modules:

```java
module com.myapp.framework.core {
    // Only our own test module can see internal test utilities
    exports com.myapp.framework.testutils to com.myapp.framework.core.test;
}
```

This pattern is used heavily inside the JDK itself — some packages are exported only to specific
internal modules.

---

### `opens` — Granting Reflective Access

`exports` controls compile-time access. But frameworks like Spring, Hibernate, and Jackson heavily
rely on **reflection** to inspect private fields, call private constructors, and access annotations
at runtime. JPMS distinguishes these two kinds of access:

- `exports` — grants compile-time access (and limited reflection on public members)
- `opens` — grants deep reflective access (including private members) but NOT compile-time access

```java
module com.myapp.domain {
    // Hibernate needs to reflect on entity fields (including private) to map them
    // Spring needs to inject beans into private fields
    // Jackson needs to read/write private properties for serialization
    opens com.myapp.domain.model;

    // You can be more restrictive: only allow Spring's reflection, nobody else
    opens com.myapp.domain.config to spring.context;
}
```

The distinction matters for security: you can allow a trusted framework to reflect on your
internals without exposing them to all modules on the classpath.

An `open module` (note: the `open` keyword before `module`) is a shorthand that opens all packages
in the module for deep reflection, while still controlling which are exported for compile-time use:

```java
// Opens EVERYTHING for reflection — useful for framework/test modules
// where you want convenience without listing every package
open module com.myapp.tests {
    requires com.myapp.domain;
    // All packages are open for reflection — no need to list them individually
}
```

---

### `uses` and `provides` — The Service Loader SPI

JPMS has first-class support for the **Service Provider Interface (SPI)** pattern via
`java.util.ServiceLoader`. This allows a module to declare that it needs implementations of
a service interface, and other modules can provide those implementations — all without any direct
dependency between provider and consumer.

This decoupling is powerful: you can add new implementations (new JARs) to the module path without
changing any existing code, and `ServiceLoader` will discover them automatically.

```java
// The SERVICE INTERFACE — defined in a shared API module
// file: com.myapp.shipping.api / module-info.java
module com.myapp.shipping.api {
    exports com.myapp.shipping.api;   // ShippingProvider interface is public
}

// The CONSUMER module — knows about the interface, not the implementations
// file: com.myapp.orders / module-info.java
module com.myapp.orders {
    requires com.myapp.shipping.api;
    uses com.myapp.shipping.api.ShippingProvider;  // "I will look for implementations"
}

// A PROVIDER module — knows about the interface, not the consumer
// file: com.myapp.shipping.fedex / module-info.java
module com.myapp.shipping.fedex {
    requires com.myapp.shipping.api;
    provides com.myapp.shipping.api.ShippingProvider
        with com.myapp.shipping.fedex.FedExShippingProvider;  // "Here's my implementation"
}
```

And in the consumer's Java code:

```java
import java.util.ServiceLoader;
import com.myapp.shipping.api.ShippingProvider;

public class ShippingService {

    public void shipOrder(Order order) {
        // ServiceLoader discovers ALL providers registered via 'provides' in any module
        ServiceLoader<ShippingProvider> loader =
            ServiceLoader.load(ShippingProvider.class);

        // Iterate over all discovered implementations
        for (ShippingProvider provider : loader) {
            if (provider.supports(order.getDestination())) {
                provider.ship(order);
                return;
            }
        }
        throw new RuntimeException("No shipping provider available for: "
            + order.getDestination());
    }
}
```

This is how JDBC drivers work: your code calls `DriverManager.getConnection(url)`, and the JDBC
driver JAR on the module path registers itself via `provides java.sql.Driver with ...`. Your code
never imports the driver class directly.

---

## A Complete Working Example — A Modular Application

Let's build a small but real modular application with three modules to see everything together.

### Project Directory Structure

```
myapp/
│
├── com.myapp.utils/                         ← Module 1: shared utilities
│   └── src/
│       ├── module-info.java
│       └── com/myapp/utils/
│           ├── StringUtils.java
│           └── internal/
│               └── StringHelper.java        ← internal — NOT exported
│
├── com.myapp.domain/                        ← Module 2: business domain
│   └── src/
│       ├── module-info.java
│       └── com/myapp/domain/
│           ├── Product.java
│           └── Order.java
│
└── com.myapp.app/                           ← Module 3: main application
    └── src/
        ├── module-info.java
        └── com/myapp/app/
            └── Main.java
```

### Module 1 — `com.myapp.utils`

```java
// File: com.myapp.utils/src/module-info.java
module com.myapp.utils {
    // Only export the public API package
    exports com.myapp.utils;
    // com.myapp.utils.internal is NOT listed — completely sealed
}
```

```java
// File: com.myapp.utils/src/com/myapp/utils/StringUtils.java
package com.myapp.utils;

public final class StringUtils {

    private StringUtils() {} // utility class — no instances

    public static String capitalize(String input) {
        if (input == null || input.isBlank()) return input;
        return Character.toUpperCase(input.charAt(0))
             + input.substring(1).toLowerCase();
    }

    public static boolean isNullOrEmpty(String input) {
        return input == null || input.isEmpty();
    }
}
```

```java
// File: com.myapp.utils/src/com/myapp/utils/internal/StringHelper.java
package com.myapp.utils.internal;

// This class exists but is in a non-exported package.
// No code outside this module can ever import or use it — guaranteed by JPMS.
public class StringHelper {
    public static String reverseWords(String sentence) {
        String[] words = sentence.split(" ");
        StringBuilder sb = new StringBuilder();
        for (int i = words.length - 1; i >= 0; i--) {
            sb.append(words[i]);
            if (i > 0) sb.append(" ");
        }
        return sb.toString();
    }
}
```

### Module 2 — `com.myapp.domain`

```java
// File: com.myapp.domain/src/module-info.java
module com.myapp.domain {
    // We use StringUtils in our public API return types? No.
    // We only USE it internally — so plain requires (not transitive)
    requires com.myapp.utils;

    // Export both domain classes for other modules to use
    exports com.myapp.domain;
}
```

```java
// File: com.myapp.domain/src/com/myapp/domain/Product.java
package com.myapp.domain;

import com.myapp.utils.StringUtils; // fine — com.myapp.utils is required and exported

public class Product {
    private final String id;
    private final String name;
    private final double price;

    public Product(String id, String name, double price) {
        if (StringUtils.isNullOrEmpty(id)) {
            throw new IllegalArgumentException("Product ID cannot be empty");
        }
        this.id    = id;
        this.name  = StringUtils.capitalize(name); // using the utility module
        this.price = price;
    }

    public String getId()    { return id; }
    public String getName()  { return name; }
    public double getPrice() { return price; }

    @Override
    public String toString() {
        return "Product[" + id + ", " + name + ", $" + price + "]";
    }
}
```

### Module 3 — `com.myapp.app`

```java
// File: com.myapp.app/src/module-info.java
module com.myapp.app {
    // Direct dependency on domain
    requires com.myapp.domain;

    // We don't need to declare 'requires com.myapp.utils' here
    // even though domain depends on it, because we don't use utils types directly.
    // (If com.myapp.domain had declared 'requires transitive com.myapp.utils',
    //  we would get it for free — but it didn't, because StringUtils is internal.)
}
```

```java
// File: com.myapp.app/src/com/myapp/app/Main.java
package com.myapp.app;

import com.myapp.domain.Product;

// This would be a COMPILE ERROR — utils is not accessible from this module:
// import com.myapp.utils.StringUtils;   // ERROR: module com.myapp.app does not read com.myapp.utils
// import com.myapp.utils.internal.*;    // ERROR: even if we required utils, internal is not exported

public class Main {
    public static void main(String[] args) {
        Product laptop = new Product("P001", "gaming laptop", 1299.99);
        Product mouse  = new Product("P002", "wireless mouse", 29.99);

        System.out.println("Products loaded:");
        System.out.println("  " + laptop);
        System.out.println("  " + mouse);
    }
}
```

### Compiling and Running (Manual — for understanding)

```bash
# Step 1: Compile module 1 (no dependencies)
javac -d mods/com.myapp.utils \
      com.myapp.utils/src/module-info.java \
      com.myapp.utils/src/com/myapp/utils/StringUtils.java \
      com.myapp.utils/src/com/myapp/utils/internal/StringHelper.java

# Step 2: Compile module 2 (depends on module 1 — use --module-path)
javac --module-path mods \
      -d mods/com.myapp.domain \
      com.myapp.domain/src/module-info.java \
      com.myapp.domain/src/com/myapp/domain/Product.java

# Step 3: Compile module 3 (depends on module 2)
javac --module-path mods \
      -d mods/com.myapp.app \
      com.myapp.app/src/module-info.java \
      com.myapp.app/src/com/myapp/app/Main.java

# Step 4: Run — specify module path and entry point as module/class
java --module-path mods \
     --module com.myapp.app/com.myapp.app.Main
```

In a real project, Maven or Gradle handle all of this compilation automatically. But knowing the
manual commands deeply reveals what the build tools are doing behind the scenes.

---

## Module Path vs Classpath

These two mechanisms for locating code are fundamentally different, and understanding the
distinction is critical for migrating existing projects to JPMS.

The **classpath** is a flat, unordered bag of JARs and directories. There is no concept of
identity, versioning, or declared dependencies. The JVM scans it for class files when needed,
picking the first match it finds. This simplicity is also its weakness.

The **module path** is a structured collection of modular artifacts. Each artifact on the module
path is either a named module (has a `module-info.class`) or gets treated as an **automatic
module** (described in Phase 9.2). The module system uses the declared `requires` relationships
to build a precise **module graph** — a directed graph of dependencies — before any class loading
begins. If a required module is missing, you get a clear error immediately on startup rather than
a mysterious failure at runtime when a specific class is first needed.

You can actually mix both mechanisms when migrating — put legacy JARs on the classpath, new
modular code on the module path. JPMS handles this gracefully via the "unnamed module" concept
(covered in Phase 9.2).

---

## Named, Unnamed, and Automatic Modules — Quick Introduction

To make JPMS practical for migration, it defines four kinds of modules. Two of them are introduced
fully in Phase 9.2, but it helps to know the vocabulary now:

A **named module** has an explicit `module-info.java` — it fully participates in the module system
with declared dependencies, exports, and encapsulation. This is what you write for new code.

An **unnamed module** is everything on the **classpath** — treated as one giant unnamed module.
It can read all named modules, and named modules with the right configuration can read it back.
This is the backward-compatibility bridge for legacy JARs.

An **automatic module** is a regular (non-modular) JAR placed on the **module path** instead of
the classpath. JPMS auto-generates a name for it (from its JAR file name or manifest attribute)
and implicitly makes it read all other modules. This is the migration bridge for third-party
libraries that haven't yet added `module-info.java`.

A **platform module** is a module that is part of the JDK itself, like `java.base`, `java.sql`,
`java.desktop`. These are fully modular and are what made it possible to split the JDK.

---

## ⚠️ Important Points and Best Practices

**The golden rule of module design** is to export only what is genuinely part of your public API.
Every package you don't export is an implementation detail you can freely change, refactor, or
remove without breaking anyone. Think hard before exporting any package — once exported and used
by others, removing it is a breaking change.

**Avoid `opens` where possible.** Opening a package for deep reflection is a powerful but
dangerous capability. Prefer opening only to the specific frameworks that need it
(`opens com.myapp.model to spring.core, hibernate.core`) rather than opening to everyone.
This keeps your encapsulation as strong as possible while still being compatible with frameworks.

**Prefer `requires transitive` only when your public API truly exposes types from the dependency.**
Overusing `requires transitive` creates implicit dependencies for your callers that they didn't
ask for, which can cause version conflicts and bloated module graphs.

**The `module-info.java` file is compiled code**, not a configuration file. This means changes to
it participate in normal incremental compilation. It also means naming is type-checked — you can't
misspell a package name in an `exports` clause without getting a compiler error.

**Circular module dependencies are not allowed.** If module A requires module B and module B
requires module A, the compiler will refuse to compile. This forces clean layering of your
architecture — a constraint that is actually very healthy in large systems.

**JPMS and reflection.** Many Java frameworks were designed before JPMS and use reflection
extensively. When you run such frameworks with fully modular code, you will often need to add
`opens` directives or `--add-opens` command-line flags. This is a common pain point when
migrating existing Spring or Hibernate applications to the module system — the frameworks
will tell you exactly what to open with clear error messages.
