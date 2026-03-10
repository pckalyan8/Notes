# Phase 9.2 — The Four Module Types in Java

> **Why does Java define four kinds of modules?**
> JPMS had to solve a genuine bootstrap problem: the entire Java ecosystem — the JDK itself,
> millions of existing libraries, and all future code — had to be able to work together, even
> though only a tiny fraction of it would ever have a `module-info.java`. The four module types
> are the solution. They form a spectrum from "fully modular and strongly encapsulated" to
> "legacy classpath code that just needs to work." Understanding each type, and the rules that
> govern how they interact, is essential for working with real projects.

---

## The Module Type Spectrum at a Glance

Think of the four types as a gradient of modularity:

```
Most modular                                            Least modular
     │                                                        │
     ▼                                                        ▼
Platform      Named           Automatic           Unnamed
Modules       Modules         Modules             Module
(JDK itself)  (your code,     (legacy JARs on     (legacy JARs on
              has module-     the module path)     the classpath)
              info.java)
```

Each has different rules about what it can see, what it exposes, and how it fits into the module
graph. Let us go through each type in thorough detail.

---

## Type 1 — Platform Modules

### Theory

Platform modules are the modules that make up the **JDK itself**. Before Java 9, the JDK was a
single monolithic `rt.jar` that weighed in at over 60 MB, containing everything from the core
language API to Swing, CORBA, XML parsers, and the Java compiler. There was no way to use Java
on a constrained device without carrying all of this.

Starting with Java 9, Oracle modularized the JDK into more than 70 distinct platform modules.
Each has its own `module-info.java`, its own clearly defined exports, and its own dependency
declarations. This is the foundation that makes `jlink` (custom minimal JRE creation) possible.

You can see every platform module by running:

```bash
java --list-modules
```

The output on Java 21 includes modules such as:

```
java.base@21
java.compiler@21
java.datatransfer@21
java.desktop@21
java.instrument@21
java.logging@21
java.management@21
java.naming@21
java.net.http@21
java.prefs@21
java.rmi@21
java.scripting@21
java.se@21
java.security.jgss@21
java.security.sasl@21
java.smartcardio@21
java.sql@21
java.sql.rowset@21
java.transaction.xa@21
java.xml@21
java.xml.crypto@21
jdk.accessibility@21
jdk.attach@21
jdk.compiler@21
jdk.crypto.ec@21
jdk.dynalink@21
jdk.editpad@21
jdk.httpserver@21
jdk.incubator.vector@21
jdk.jartool@21
jdk.javadoc@21
jdk.jconsole@21
jdk.jdeps@21
jdk.jdi@21
jdk.jdwp.agent@21
jdk.jfr@21
jdk.jlink@21
jdk.jpackage@21
jdk.jshell@21
jdk.jstatd@21
jdk.management@21
jdk.management.agent@21
jdk.naming.dns@21
jdk.naming.rmi@21
jdk.net@21
jdk.nio.mapmode@21
jdk.sctp@21
jdk.security.auth@21
jdk.security.jgss@21
jdk.unsupported@21
jdk.xml.dom@21
... and more
```

The `jdk.*` modules are JDK-specific tools and internal utilities. The `java.*` modules are the
standard Java SE API. The `java.se` module is a special "aggregator module" — it doesn't define
any packages of its own but simply `requires transitive` all the other `java.*` modules, making
the entire Java SE API available in one declaration.

### The Root of Everything — `java.base`

`java.base` is the most fundamental module. Every other module implicitly `requires java.base`
without needing to declare it. It provides the core Java API: `java.lang`, `java.util`,
`java.io`, `java.nio`, `java.math`, `java.net`, `java.security`, `java.time`, and more.

You can inspect any platform module's descriptor with the `--describe-module` flag:

```bash
java --describe-module java.base
```

This prints something like:

```
java.base@21
exports java.io
exports java.lang
exports java.lang.annotation
exports java.lang.constant
exports java.lang.invoke
exports java.lang.module
exports java.lang.ref
exports java.lang.reflect
exports java.math
exports java.net
exports java.net.spi
exports java.nio
exports java.nio.channels
exports java.nio.channels.spi
exports java.nio.charset
exports java.nio.charset.spi
exports java.nio.file
exports java.nio.file.attribute
exports java.nio.file.spi
exports java.security
exports java.security.cert
exports java.security.interfaces
exports java.security.spec
exports java.text
exports java.text.spi
exports java.time
...
```

### Commonly Used Platform Modules and What They Contain

Understanding which platform module contains which API saves you time when writing `module-info.java`
files and understanding which modules a deployment needs.

`java.base` is always present and provides the entire core language API. You never need to declare
`requires java.base` — it is implicit.

`java.sql` provides the JDBC API — `java.sql.Connection`, `java.sql.PreparedStatement`,
`java.sql.ResultSet`, and so on. Any application using JDBC directly must declare:

```java
module com.myapp.data {
    requires java.sql;
}
```

`java.logging` provides `java.util.logging.Logger` and related classes. Most applications use
a third-party logging framework instead (SLF4J, Logback), but the JUL API is here.

`java.net.http` (introduced in Java 11) provides the modern `HttpClient`, `HttpRequest`, and
`HttpResponse` API for making HTTP calls. An application using it declares:

```java
module com.myapp.client {
    requires java.net.http;
}
```

`java.xml` provides the DOM, SAX, and StAX XML APIs, as well as XSLT. Almost every enterprise
application touches XML in some form.

`java.desktop` contains Swing (`javax.swing.*`), AWT (`java.awt.*`), and JavaBeans. It is a
large module that desktop GUI applications need but server applications do not. By not including
it in a server deployment, you save significant memory.

`jdk.compiler` contains the Java compiler API (`javax.tools.JavaCompiler`). Most applications
never need this — it is for tools that compile Java code programmatically.

### Practical Example — Checking Which Platform Modules Your App Needs

```bash
# List the module graph for a specific module — shows all transitive dependencies
java --describe-module java.sql

# Expected output includes:
# requires java.logging transitive
# requires java.transaction.xa transitive
# requires java.xml transitive
```

This tells you that if your module requires `java.sql`, you transitively get `java.logging`,
`java.transaction.xa`, and `java.xml` as readable modules.

---

## Type 2 — Application Modules (Named Modules)

### Theory

An application module — often called simply a "named module" — is any module that has a
`module-info.java` file. This is the module type you write for your own code. It fully participates
in JPMS: its dependencies are explicit, its exports are controlled, and its encapsulation is
enforced by both the compiler and the runtime.

Named modules are placed on the **module path** (`--module-path` flag) rather than the classpath.
The module system reads their `module-info.class` files to build the module graph.

Every named module must have a unique name within the system. By convention, module names follow
the reverse domain notation used for package names (`com.company.product.module`), but this is
a convention, not a hard requirement.

### A Layered Application Module Structure

A well-designed application typically defines modules that mirror its architectural layers.
Here is an example of a production-grade order management system split into modules:

```
com.mycompany.orders.api        ← Public API (interfaces and DTOs only)
com.mycompany.orders.domain     ← Domain model (entities, value objects)
com.mycompany.orders.service    ← Business logic (depends on domain)
com.mycompany.orders.persistence← Data access (depends on domain + java.sql)
com.mycompany.orders.web        ← REST controllers (depends on service)
com.mycompany.orders.app        ← Main entry point and wiring (depends on all)
```

Let's define the `module-info.java` for each:

```java
// ── API module: only interfaces and data transfer objects
// Has no dependencies — it is the foundation everything else builds on
module com.mycompany.orders.api {
    exports com.mycompany.orders.api;          // public interfaces
    exports com.mycompany.orders.api.dto;      // data transfer objects
}
```

```java
// ── Domain module: business entities and value objects
module com.mycompany.orders.domain {
    requires com.mycompany.orders.api;          // depends on the API contracts

    exports com.mycompany.orders.domain;        // entities like Order, Product
    exports com.mycompany.orders.domain.events; // domain events
    // com.mycompany.orders.domain.internal is NOT exported
}
```

```java
// ── Service module: business logic and orchestration
module com.mycompany.orders.service {
    requires com.mycompany.orders.api;           // uses API interfaces
    requires com.mycompany.orders.domain;        // uses domain entities
    requires java.logging;                       // logging

    exports com.mycompany.orders.service;        // service interfaces and implementations
    // com.mycompany.orders.service.impl is NOT exported
}
```

```java
// ── Persistence module: data access layer
module com.mycompany.orders.persistence {
    requires com.mycompany.orders.domain;        // maps domain entities
    requires java.sql;                           // JDBC API
    requires com.zaxxer.hikari;                  // connection pool (automatic module)

    // NOT exported — implementation detail; service layer calls through domain repos
    // exports nothing — it registers its repository implementations as services
    provides com.mycompany.orders.domain.OrderRepository
        with com.mycompany.orders.persistence.JdbcOrderRepository;
}
```

```java
// ── Web module: HTTP controllers
module com.mycompany.orders.web {
    requires com.mycompany.orders.service;       // calls the service layer
    requires com.mycompany.orders.api;           // uses DTOs
    requires io.undertow.core;                  // web server (automatic module)

    // Open for Jackson deserialization of request bodies
    opens com.mycompany.orders.web.dto to com.fasterxml.jackson.databind;
}
```

```java
// ── App module: entry point and dependency wiring
module com.mycompany.orders.app {
    requires com.mycompany.orders.web;
    requires com.mycompany.orders.service;
    requires com.mycompany.orders.persistence;   // brings in the ServiceLoader provider
}
```

### Why This Layering Matters

Notice that the `persistence` module does not export anything — it is entirely internal. The
`service` module never imports from `persistence`. Instead, it discovers the
`OrderRepository` implementation at runtime via `ServiceLoader`. This means you could swap out
the entire database layer without touching a single line in the service module — just put a
different persistence JAR on the module path.

This is the architectural ideal that JPMS makes mechanically enforceable rather than just
aspirational.

---

## Type 3 — Automatic Modules

### Theory

Automatic modules solve a critical practical problem: at the time Java 9 was released, virtually
no third-party libraries had `module-info.java`. If JPMS required every dependency to be a
named module, nobody could have adopted it. The automatic module mechanism allows you to put
a regular (non-modular) JAR on the **module path** and the system automatically creates a named
module from it with a generated name.

The rules that govern automatic modules make them a "trusted guest" in the module system:

**Rule 1 — Auto-Generated Name.** The module name is derived either from the `Automatic-Module-Name`
attribute in the JAR's `META-INF/MANIFEST.MF` (if present — the library author can set this as a
migration hint) or from the JAR file name, with version numbers and hyphens cleaned up:
`jackson-databind-2.15.3.jar` becomes `jackson.databind`.

**Rule 2 — Reads Everything.** An automatic module can read all named modules, all other automatic
modules, and the unnamed module. This means any code inside an automatic module can import any
class from any other module on the module path without restriction.

**Rule 3 — Exports Everything.** All packages in an automatic module are exported to all other
modules. No encapsulation at all. This is necessary because the JDK has no way to know which
packages the library author intended to be public.

**Rule 4 — Opens Everything.** All packages in an automatic module are also open for deep
reflection.

**Rule 5 — Named Modules Can Require It.** This is the key: a named module's `module-info.java`
can declare `requires jackson.databind`, allowing it to use the library even though the library
hasn't been modularized.

### Example — Using Jackson (an Automatic Module) from a Named Module

Jackson is one of the most popular Java libraries. Suppose you're writing a named module that
serializes domain objects to JSON using Jackson:

```java
// Your named module depends on Jackson as an automatic module
// The name 'com.fasterxml.jackson.databind' comes from Jackson's MANIFEST.MF:
// Automatic-Module-Name: com.fasterxml.jackson.databind
module com.myapp.json {
    requires com.myapp.domain;

    // Jackson is an automatic module — its name from MANIFEST or JAR filename
    requires com.fasterxml.jackson.databind;
    requires com.fasterxml.jackson.core;

    // Jackson uses reflection to inspect your classes — you must open them
    opens com.myapp.domain to com.fasterxml.jackson.databind;

    exports com.myapp.json;  // expose our JSON utilities
}
```

And the Java code inside this module:

```java
package com.myapp.json;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.myapp.domain.Product;

public class JsonSerializer {

    private static final ObjectMapper mapper = new ObjectMapper();

    public static String toJson(Object obj) throws Exception {
        return mapper.writeValueAsString(obj);
    }

    public static <T> T fromJson(String json, Class<T> type) throws Exception {
        return mapper.readValue(json, type);
    }
}
```

### The Migration Path — How Libraries Adopt JPMS

Good library authors go through a deliberate migration path:

**Step 1:** Add `Automatic-Module-Name` to their JAR's MANIFEST, giving consumers a stable module
name to put in their `requires` clauses. This is the minimal, backwards-compatible first step.

**Step 2:** Add a full `module-info.java` with proper `exports`, `opens`, and `requires` — becoming
a named module. At this point, consumers' `module-info.java` files don't need to change because
the name matches what was declared in the manifest.

Most major libraries — Spring, Hibernate, Jackson, Guava — now ship with either an
`Automatic-Module-Name` in their manifest or a full `module-info.java`.

### ⚠️ Caution: Automatic Module Naming Instability

If you rely on a JAR file name for the automatic module name (rather than the manifest attribute),
and the library changes its JAR filename (e.g., by changing its artifact ID or version format),
your `requires` clause breaks at compile time. This is why you should **always check if a library
provides an `Automatic-Module-Name` manifest attribute** before relying on the file-name-derived
name. You can check with:

```bash
jar --describe-module --file=jackson-databind-2.15.3.jar
```

If the output shows `No module descriptor found. Derived automatic module`, the name is
file-name-derived and potentially unstable. If it shows `Automatic-Module-Name:` followed by
a name, that is the stable, author-declared name to use.

---

## Type 4 — The Unnamed Module

### Theory

The unnamed module is the backward-compatibility cornerstone of the entire module system. It
contains **all classes loaded from the classpath**. There is exactly one unnamed module per
class loader, and it has a special set of rules:

**The unnamed module reads all named modules.** Code on the classpath can import and use classes
from any named module's exported packages without any declaration. This is why existing code that
uses `java.util.List` or `java.sql.Connection` didn't break when Java 9 was released — those
classes are in named modules (`java.base` and `java.sql`), and the unnamed module can read them.

**Named modules generally cannot read the unnamed module.** This is the key asymmetry that
enforces encapsulation. A named module that tries to import a class from a JAR on the classpath
(not the module path) gets a compiler error, because named modules must have explicit
`requires` declarations, and you cannot write `requires <unnamed>`. This creates a natural
incentive to migrate dependencies from the classpath to the module path.

**The unnamed module exports all its packages.** Like automatic modules, the unnamed module has
no encapsulation — everything in it is accessible to everything else in the unnamed module.

**Named modules can access the unnamed module with `--add-reads`.** For migration scenarios where
you temporarily need a named module to read from the classpath, you can use:

```bash
java --add-reads com.myapp.service=ALL-UNNAMED ...
```

This is a command-line escape hatch, not a permanent solution.

### Visualizing the Relationship

```
┌──────────────────────────────────────────────────────────────────────┐
│  MODULE PATH                                                         │
│  ┌─────────────┐    ┌──────────────────┐    ┌────────────────────┐  │
│  │  java.base  │    │  com.myapp.svc   │    │  jackson.databind  │  │
│  │  (Platform) │◄───│  (Named Module)  │───►│  (Auto Module)     │  │
│  └─────────────┘    └──────────────────┘    └────────────────────┘  │
│                             ▲                        ▲              │
└─────────────────────────────┼────────────────────────┼──────────────┘
                              │ reads named modules      │ reads everyone
┌─────────────────────────────┼────────────────────────┼──────────────┐
│  CLASSPATH                  │                         │              │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │  Unnamed Module (all classpath JARs + legacy code)            │   │
│  │  Reads: all named modules ✓    Is read by named modules: ✗   │   │
│  └───────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Practical Implications for Existing Projects

When you migrate an existing application to Java 9+, your code typically starts entirely in the
unnamed module — you haven't added any `module-info.java` files yet, and everything is on the
classpath. This means your code has full access to all JDK platform modules (their exported
packages are readable by the unnamed module), but you lose the strong encapsulation guarantees
of JPMS.

The unnamed module is intentionally permissive to make this migration non-breaking. You can run
an unmodified Java 8 application on Java 17 or 21 and it will generally work (with some caveats
around removed APIs and reflection restrictions on JDK internals).

### How Unnamed Module Access Works in Practice

```java
// This code is on the CLASSPATH (in the unnamed module)
// It can use any exported package from any platform module:

import java.util.ArrayList;          // from java.base — fine
import java.sql.Connection;          // from java.sql — fine
import java.net.http.HttpClient;     // from java.net.http — fine (Java 11+)
import java.util.logging.Logger;     // from java.logging — fine

// What it CANNOT do without --add-opens:
// Use reflection on JDK internal packages (like sun.misc.Unsafe)
// because those packages are not exported — even to the unnamed module
```

### The Reflection Warning You've Seen

You've almost certainly seen this warning when running older tools or frameworks on modern Java:

```
WARNING: An illegal reflective access operation has occurred
WARNING: Use --illegal-access=warn to enable warnings of illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
```

This is the unnamed module trying to deeply reflect on JDK internals that are not open. In Java 9
through 15, this was a warning. In Java 16, it became an error by default. The fix for framework
code is the `--add-opens` command-line flag, or adding an `opens` directive in `module-info.java`.

---

## How the Four Types Interact — Access Rules Table

Understanding the access rules between module types is crucial for debugging classpath and
module path issues in real projects.

| | Platform Module | Named Module | Automatic Module | Unnamed Module |
|---|---|---|---|---|
| **Can be read by Platform Module** | Yes (if required) | Yes (if required) | No | No |
| **Can be read by Named Module** | Yes (if required) | Yes (if required) | Yes (if required) | No (use --add-reads) |
| **Can be read by Automatic Module** | Yes (reads all) | Yes (reads all) | Yes (reads all) | Yes (reads all) |
| **Can be read by Unnamed Module** | Yes (reads all named) | Yes (reads all named) | Yes (reads all named) | Yes (same classloader) |
| **Exports** | Declared in module-info | Declared in module-info | All packages | All packages |
| **Encapsulation** | Strong | Strong | None | None |

---

## The Migration Journey in Practice

Most real applications fall into one of these migration scenarios, and knowing which module types
you're dealing with at each stage keeps the work manageable.

**Scenario A — Pure Legacy (Everything on Classpath):** All code is in the unnamed module.
No `module-info.java` anywhere. This works on Java 21, but you get none of the JPMS benefits.
You may see `--add-opens` warnings if frameworks try to reflect on JDK internals.

**Scenario B — Mixed Migration (Module Path + Classpath):** Your application code is being
progressively modularized. New modules have `module-info.java` on the module path. Legacy
dependencies that aren't yet modular stay on the classpath (unnamed module) or are placed on
the module path as automatic modules. This is the most common real-world state for applications
actively migrating to JPMS.

**Scenario C — Fully Modular:** Every JAR — your code and all dependencies — is a named module.
You get the full benefits of JPMS: strong encapsulation, clear dependency declarations, and the
ability to use `jlink` to produce a minimal custom JRE.

### Example: Spring Boot Application Migration Path

A typical Spring Boot project migrating to JPMS goes through these steps:

```java
// Step 1: Your main application module (your code only)
module com.mycompany.app {
    requires spring.context;          // Spring is an automatic module
    requires spring.web;              // (name from Spring's MANIFEST.MF)
    requires spring.data.jpa;
    requires com.fasterxml.jackson.databind;

    // Spring needs reflection access to your beans
    opens com.mycompany.app.controllers to spring.web;
    opens com.mycompany.app.services    to spring.context;
    opens com.mycompany.app.entities    to org.hibernate.orm.core;

    exports com.mycompany.app.api;    // if exposing an API to other modules
}
```

Spring Boot itself started shipping with `Automatic-Module-Name` in its manifests from
version 2.x onwards, and Spring 6 / Spring Boot 3 added full JPMS `module-info.java` support,
making Spring applications capable of being fully modular.

---

## ⚠️ Important Points and Best Practices

**Prefer named modules for all new code.** There is no good reason to put new code you write on
the classpath. Named modules give you better encapsulation, clearer documentation of dependencies,
and the ability to use `jlink`. The only exception is very small throwaway scripts.

**Use `Automatic-Module-Name` in your library's MANIFEST before publishing.** If you write a
library that others consume, add a stable `Automatic-Module-Name` to your JAR's manifest even
before you write a full `module-info.java`. This gives your consumers a stable name to put in
their `requires` clauses and prevents breakage when you eventually add the full descriptor.
In Maven, add this to your `pom.xml`:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <configuration>
        <archive>
            <manifestEntries>
                <Automatic-Module-Name>com.mycompany.mylib</Automatic-Module-Name>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

**Don't split packages across modules.** A package cannot be split between a named module and
any other module (automatic or named). If `com.example.utils` is defined in two different JARs
on the module path, the module system will refuse to load either — this is called a "split
package" and is explicitly forbidden. This is actually a feature: it prevents the ambiguity that
caused classpath hell. But it means you may need to consolidate or rename packages when migrating.

**Understand the difference between unnamed and automatic.** The most common confusion in JPMS
is mixing up these two. A regular JAR on the **classpath** is in the unnamed module — it has
no name and named modules cannot require it. The same JAR on the **module path** becomes an
automatic module — it gets a name and named modules can require it. The physical JAR is identical;
only its location (classpath vs module path) determines its module type.

**Watch out for split packages across classpath and module path.** If `com.example.util` exists
in a JAR on the classpath and also in a named module on the module path, you have a split package.
The module system will reject it at startup with a clear error message. This is a common surprise
when migrating incrementally.
