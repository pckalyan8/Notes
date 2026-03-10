# Phase 9.3 — Module Commands and Tooling

> **Why master the command-line tooling?**
> When you work with JPMS in a real project, your IDE and build tool handle compilation and
> packaging automatically. But when something goes wrong — a split package, a missing module, an
> illegal reflective access — you need to know exactly which flags to pass and which tools to
> reach for. The command-line tooling is also the lens through which you truly understand how the
> module system works. Watching `jdeps` draw a module graph or watching `jlink` assemble a
> custom JRE makes JPMS tangible in a way that reading documentation alone cannot.

---

## Overview of the Key Module-Aware Flags

The Java compiler (`javac`) and the Java launcher (`java`) were extended in Java 9 with a set of
module-aware flags. Here is a map of the most important ones before diving into each:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  FLAG                   │  TOOL     │  PURPOSE                               │
├──────────────────────────────────────────────────────────────────────────────┤
│  --module-path (-p)     │ javac/java │ Where to find named/automatic modules │
│  --module (-m)          │ javac/java │ Which module (and class) to run        │
│  --add-modules          │ javac/java │ Force extra modules into the graph     │
│  --add-opens            │ java       │ Open a package for deep reflection     │
│  --add-exports          │ java       │ Export a package to another module     │
│  --add-reads            │ java       │ Make a module read another module      │
│  --list-modules         │ java       │ Print all observable modules           │
│  --describe-module      │ java       │ Print one module's descriptor          │
│  --show-module-resolution│ java      │ Trace the module graph resolution      │
│  --patch-module         │ javac/java │ Add classes into an existing module    │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## `--module-path` (`-p`) — The Module-Aware Classpath

### What It Does

`--module-path` (short form: `-p`) tells the JVM and compiler where to look for modules. It is
the module-system counterpart to the old `-classpath` flag. You pass it a colon-separated (Linux/Mac)
or semicolon-separated (Windows) list of directories and JAR files that contain modules.

JARs placed on the module path must either have a `module-info.class` (named modules) or they
will be treated as automatic modules. Contrast this with the classpath, where the same JARs are
just anonymous code in the unnamed module.

### Syntax

```bash
# Single directory containing compiled modules
java --module-path mods --module com.myapp.app/com.myapp.app.Main

# Multiple directories and individual JARs
java --module-path mods:lib/jackson-databind.jar:lib/jackson-core.jar \
     --module com.myapp.app/com.myapp.app.Main

# Short form -p is equivalent
java -p mods:lib --module com.myapp.app/com.myapp.app.Main

# Windows: use semicolons
java --module-path "mods;lib" --module com.myapp.app/com.myapp.app.Main
```

### Full Compilation and Run Workflow

Let's work through a complete, real example with three modules. The goal is to see `--module-path`
in action at both compile time and runtime, and to understand the exact commands step by step.

The project structure is:

```
project/
├── src/
│   ├── com.myapp.utils/
│   │   ├── module-info.java
│   │   └── com/myapp/utils/FormatUtils.java
│   ├── com.myapp.service/
│   │   ├── module-info.java
│   │   └── com/myapp/service/GreetingService.java
│   └── com.myapp.app/
│       ├── module-info.java
│       └── com/myapp/app/Main.java
└── mods/   ← compiled output goes here
```

The source files:

```java
// ── src/com.myapp.utils/module-info.java ─────────────────────────────────────
module com.myapp.utils {
    exports com.myapp.utils;
}
```

```java
// ── src/com.myapp.utils/com/myapp/utils/FormatUtils.java ─────────────────────
package com.myapp.utils;

public final class FormatUtils {
    private FormatUtils() {}

    public static String shout(String text) {
        return text.toUpperCase() + "!!!";
    }

    public static String whisper(String text) {
        return text.toLowerCase() + "...";
    }
}
```

```java
// ── src/com.myapp.service/module-info.java ────────────────────────────────────
module com.myapp.service {
    requires com.myapp.utils;        // depends on utils
    exports com.myapp.service;       // exposes service to callers
}
```

```java
// ── src/com.myapp.service/com/myapp/service/GreetingService.java ─────────────
package com.myapp.service;

import com.myapp.utils.FormatUtils;  // fine: utils is required and exported

public class GreetingService {

    public String greet(String name, boolean enthusiastic) {
        String message = "Hello, " + name;
        return enthusiastic
            ? FormatUtils.shout(message)
            : FormatUtils.whisper(message);
    }
}
```

```java
// ── src/com.myapp.app/module-info.java ───────────────────────────────────────
module com.myapp.app {
    requires com.myapp.service;  // depends on service (which transitively needs utils)
    // Notice: we do NOT need 'requires com.myapp.utils' here because our code
    // does not directly use FormatUtils. Only service does.
}
```

```java
// ── src/com.myapp.app/com/myapp/app/Main.java ────────────────────────────────
package com.myapp.app;

import com.myapp.service.GreetingService;
// import com.myapp.utils.FormatUtils; ← This would be a COMPILER ERROR:
//   module com.myapp.app does not read module com.myapp.utils

public class Main {
    public static void main(String[] args) {
        GreetingService svc = new GreetingService();
        System.out.println(svc.greet("Alice", true));
        System.out.println(svc.greet("Bob",   false));
    }
}
```

Compiling each module in dependency order:

```bash
# 1. Compile com.myapp.utils (no dependencies)
javac -d mods/com.myapp.utils \
      src/com.myapp.utils/module-info.java \
      src/com.myapp.utils/com/myapp/utils/FormatUtils.java

# 2. Compile com.myapp.service (depends on utils — use --module-path)
javac --module-path mods \
      -d mods/com.myapp.service \
      src/com.myapp.service/module-info.java \
      src/com.myapp.service/com/myapp/service/GreetingService.java

# 3. Compile com.myapp.app (depends on service)
javac --module-path mods \
      -d mods/com.myapp.app \
      src/com.myapp.app/module-info.java \
      src/com.myapp.app/com/myapp/app/Main.java

# 4. Run — --module flag specifies module/fully.qualified.MainClass
java --module-path mods \
     --module com.myapp.app/com.myapp.app.Main
```

The output:

```
HELLO, ALICE!!!
hello, bob...
```

You can also compile all three modules in a single command using the `--module-source-path` flag,
which tells `javac` to find source files using a consistent directory layout:

```bash
# Compile all three modules at once (requires consistent src/<module-name>/ layout)
javac --module-source-path src \
      -d mods \
      --module com.myapp.utils,com.myapp.service,com.myapp.app
```

---

## `--module` (`-m`) — Specifying the Entry Point

The `--module` flag (short form: `-m`) has a different meaning depending on whether it is used
with `javac` or `java`.

With `java`, it specifies the module and main class to run. The format is
`module.name/fully.qualified.MainClass`. If the module's `module-info.java` declares a main
class via the `MainClass` attribute in the module descriptor (which Maven/Gradle can set), you
can omit the class name and use just the module name:

```bash
# Full form: module name + main class
java --module-path mods --module com.myapp.app/com.myapp.app.Main

# Short form if the module declares a main class in its descriptor
java --module-path mods --module com.myapp.app
```

With `javac`, the `--module` flag tells the compiler which modules to compile from the module
source path. This is used with `--module-source-path` for multi-module compilation:

```bash
javac --module-source-path src --module com.myapp.app -d mods
# javac resolves the entire dependency tree automatically
```

---

## `--add-modules` — Forcing Modules into the Graph

### What It Does

The module system builds a module graph starting from the **root modules** — modules that are
the entry point of your application. Only modules reachable from these roots (via `requires`
declarations) are included in the graph. `--add-modules` lets you add modules to the graph
even if they are not required by anything in the dependency chain.

This flag is most commonly used in three scenarios.

**Scenario 1: Running code in the unnamed module that needs a specific platform module.** Suppose
you have a legacy application on the classpath that uses `javax.xml.bind` (JAXB), which was
removed from the default set of platform modules in Java 11. You need to add it explicitly:

```bash
# Legacy app using JAXB needs java.xml.bind added to the graph
java --add-modules java.xml.bind -jar my-legacy-app.jar
```

**Scenario 2: Using `ALL-MODULE-PATH`.** This special value adds ALL modules found on the module
path to the graph, regardless of whether they are required. Useful during testing or exploration:

```bash
java --module-path mods --add-modules ALL-MODULE-PATH -jar app.jar
```

**Scenario 3: Using `ALL-DEFAULT`.** Adds all modules that are in the default module set (the
standard `java.*` modules). This is useful for unnamed module code that relies on the full Java SE
API without any module declarations:

```bash
java --add-modules ALL-DEFAULT -cp legacy.jar com.example.LegacyMain
```

---

## `--add-opens` — Granting Reflective Access at Runtime

### The Problem It Solves

When a fully modular application uses a framework that relies on deep reflection (Spring,
Hibernate, Jackson, JUnit, etc.), the framework needs to access private fields, private
constructors, and non-exported packages. If your module's `module-info.java` does not include
the appropriate `opens` directive, the framework gets an `InaccessibleObjectException` at runtime.

`--add-opens` is the command-line equivalent of writing an `opens` directive in `module-info.java`.
It is intended as a **migration escape hatch** — if you can't or don't want to modify
`module-info.java` (perhaps because you don't own the module), you can open the package at
launch time.

### Syntax

```bash
--add-opens <module>/<package>=<target-module>
```

The `<target-module>` can be `ALL-UNNAMED` to grant access to all code in the unnamed module
(classpath code), which is the most common use case when running framework code.

### Practical Examples

```bash
# Spring needs to reflect on your service beans
java --add-opens com.myapp.service/com.myapp.service=spring.core \
     --module-path mods \
     --module com.myapp.app/com.myapp.app.Main

# Hibernate needs to reflect on your entity fields
java --add-opens com.myapp.domain/com.myapp.domain.model=ALL-UNNAMED \
     --module-path mods \
     --module com.myapp.app/com.myapp.app.Main

# JUnit 5 needs reflective access to run your test methods
java --add-opens com.myapp.tests/com.myapp.tests=org.junit.platform.commons \
     --module-path mods \
     --module com.myapp.tests/com.myapp.tests.TestSuite
```

### The `--add-opens` Flag in Maven and Gradle

In real Maven or Gradle builds, these flags are typically set in the build tool's surefire plugin
configuration (for tests) or in the application's startup scripts. A typical Maven `pom.xml`
configuration for a Spring Boot application's tests might look like:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <argLine>
            --add-opens com.myapp.service/com.myapp.service=ALL-UNNAMED
            --add-opens com.myapp.domain/com.myapp.domain=ALL-UNNAMED
        </argLine>
    </configuration>
</plugin>
```

### ⚠️ Important: Prefer `opens` in `module-info.java` Over `--add-opens`

Using `--add-opens` on the command line is a workaround. The declarative, maintainable approach
is to add the `opens` directive to your `module-info.java` with appropriate `to` qualification.
Reserve `--add-opens` for cases where you cannot modify the module descriptor — such as when
you're working around issues in third-party code or the JDK itself.

---

## `--add-exports` — Exporting Packages at Runtime

`--add-exports` is the command-line equivalent of adding an `exports` directive to a
`module-info.java`. It makes a package that is not exported accessible to another module or to
all unnamed module code. This is used primarily when tools or frameworks need access to JDK
internal packages.

```bash
# Syntax: --add-exports <source-module>/<package>=<target-module-or-ALL-UNNAMED>

# Allow classpath code to use a non-exported JDK internal package
java --add-exports java.base/sun.nio.ch=ALL-UNNAMED \
     -cp myapp.jar com.example.Main

# Export a non-exported package from your own module to another specific module
java --add-exports com.myapp.service/com.myapp.service.internal=com.myapp.plugin \
     --module-path mods \
     --module com.myapp.app/com.myapp.app.Main
```

This flag is most commonly needed when running older tools (like some JVM agents, profilers,
or bytecode manipulation libraries) that directly use internal JDK APIs like `sun.misc.Unsafe`.
The preferred long-term solution is always to update the library to use public APIs.

---

## `--add-reads` — Granting Read Access Between Modules

`--add-reads` makes one module readable to another, even without a `requires` declaration in
`module-info.java`. This is the escape hatch for situations where you cannot modify a module's
descriptor but need two modules to communicate.

```bash
# Syntax: --add-reads <module>=<other-module-or-ALL-UNNAMED>

# Make com.myapp.service able to read from the unnamed module (classpath code)
java --add-reads com.myapp.service=ALL-UNNAMED \
     --module-path mods \
     --module com.myapp.app/com.myapp.app.Main
```

In practice, `--add-reads` is rarely needed in application code. It appears mostly in test
harnesses or when using very old tooling that puts code on the classpath that needs to be visible
to your named modules.

---

## `jar --describe-module` — Inspecting Module Descriptors

The `jar` tool was extended in Java 9 to work with modular JARs. The `--describe-module` flag
prints the module descriptor of a JAR file — invaluable for understanding what a library exports,
what it requires, and what its auto-generated module name would be.

```bash
# Describe a modular JAR you built yourself
jar --describe-module --file=mods/com.myapp.service.jar
```

Output:

```
com.myapp.service
  requires com.myapp.utils
  requires java.base mandated
  exports com.myapp.service
```

```bash
# Describe a third-party JAR — check if it has a module-info or is automatic
jar --describe-module --file=lib/jackson-databind-2.15.3.jar
```

Output when a full module descriptor exists:

```
com.fasterxml.jackson.databind
  requires com.fasterxml.jackson.annotations transitive
  requires com.fasterxml.jackson.core transitive
  requires java.base mandated
  requires java.desktop
  requires java.logging
  exports com.fasterxml.jackson.databind
  exports com.fasterxml.jackson.databind.annotation
  ...
```

Output when only an `Automatic-Module-Name` is present in the manifest:

```
No module descriptor found. Derived automatic module.

com.fasterxml.jackson.databind automatic
  requires java.base mandated
  requires ALL-UNNAMED
  contains ... (all packages listed)
```

This tells you the module would be an automatic module with name `com.fasterxml.jackson.databind`,
and that name came from the manifest's `Automatic-Module-Name` attribute — stable and safe to use.

```bash
# You can also describe a platform module using the java launcher
java --describe-module java.sql

# Output:
# java.sql@21
# requires java.logging transitive
# requires java.transaction.xa transitive
# requires java.xml transitive
# requires java.base mandated
# exports java.sql
# exports javax.sql
# uses java.sql.Driver
```

---

## `jdeps` — Module Dependency Analyser

`jdeps` is one of the most useful tools in the JPMS toolkit. It statically analyses the bytecode
of your JARs and reports what they depend on — which packages, which modules, and whether any
dependencies are on JDK internal APIs. It is your first tool to reach for when planning a
migration to JPMS.

### Basic Usage — Analysing a Single JAR

```bash
# Analyse a JAR file and show its module dependencies
jdeps --module-path mods myapp.jar
```

Sample output:

```
myapp.jar -> java.base
myapp.jar -> com.myapp.service
   com.myapp.app   -> com.myapp.service.GreetingService   com.myapp.service
   com.myapp.app   -> java.io.PrintStream                 java.base
   com.myapp.app   -> java.lang.Object                    java.base
   com.myapp.app   -> java.lang.String                    java.base
   com.myapp.app   -> java.lang.System                    java.base
```

This tells you that `myapp.jar` depends on `java.base` and `com.myapp.service`, and shows the
exact classes that create those dependencies.

### Checking for JDK Internal API Usage

One of the most important uses of `jdeps` is finding code that uses JDK internal APIs (those
not exported from any module), which are a common source of compatibility issues when upgrading
Java versions:

```bash
# The --jdk-internals flag specifically flags usage of non-public JDK APIs
jdeps --jdk-internals --multi-release 11 \
      --module-path $JAVA_HOME/jmods \
      myapp.jar
```

Sample output when internal APIs are found:

```
myapp.jar -> JDK removed internal API
   com.example.MyClass    -> sun.misc.BASE64Encoder   JDK internal API (java.base)

Warning: JDK internal APIs are unsupported and private to JDK implementation that are
subject to be removed or changed incompatibly and could break your application.
Please modify your code to eliminate dependence on any JDK internal APIs.
For the most recent updates on JDK internal API replacements, please check:
https://wiki.openjdk.org/display/JDK8/Java+Dependency+Analysis+Tool
```

This is how you find code that was using `sun.misc.BASE64Encoder` (replaced by
`java.util.Base64` in Java 8) or `sun.reflect.Reflection` (now in `java.lang.reflect`).

### Generating Module Dependency Graphs

`jdeps` can generate DOT format dependency graphs that you can visualize with Graphviz:

```bash
# Generate a dependency graph for the entire module path
jdeps --module-path mods \
      --dot-output graphs \
      --module com.myapp.app

# This creates a .dot file in the graphs/ directory
# Visualize with Graphviz:
dot -Tpng graphs/com.myapp.app.dot -o module-graph.png
```

### Generating a `module-info.java` Suggestion

One of `jdeps`' most powerful features for migration is its ability to suggest the content of
`module-info.java` for a JAR that doesn't have one yet:

```bash
# jdeps analyses legacy-app.jar and suggests what its module-info.java should contain
jdeps --generate-module-info ./suggested-descriptors \
      --module-path mods \
      legacy-app.jar
```

This creates a `./suggested-descriptors/com.example.legacyapp/module-info.java` with suggested
`requires` and `exports` based on actual bytecode analysis. You should treat this as a starting
point that needs human review, not a perfect final product, but it can save hours of work on
large codebases.

### Checking Module Summary Information

```bash
# --summary: one line per module dependency (less verbose than default)
jdeps --module-path mods --summary myapp.jar

# Output:
# myapp.jar -> com.myapp.service
# myapp.jar -> java.base
```

```bash
# Analyse all JARs in a directory
jdeps --module-path mods mods/

# List which JDK modules are needed by a JAR (useful before jlink)
jdeps --print-module-deps myapp.jar
# Output: java.base,java.sql,java.logging
# This tells you exactly which --add-modules to pass to jlink
```

---

## `jlink` — Building a Custom Minimal JRE

### What It Does and Why It Matters

`jlink` is the tool that makes one of JPMS's most exciting promises real: **you can build a
custom, minimal Java runtime image** that contains only the JDK modules your application
actually needs, plus your application itself. The result is a self-contained, runnable package
that does not require a JDK or JRE to be installed on the target machine.

Before `jlink`, deploying a Java application to a Docker container or embedded device meant
including the full JDK (hundreds of megabytes). With `jlink`, a minimal Spring Boot service
might need only `java.base`, `java.logging`, `java.net.http`, and `java.sql` — shrinking
from 400MB+ to perhaps 40-60MB, with a significantly reduced attack surface.

`jlink` **only works with fully modular code**. All modules in the image — including your
application modules and all dependencies — must be named modules. Automatic modules and the
unnamed module cannot be included in a `jlink` image. This is the main motivator for libraries
to complete their JPMS migration.

### Basic `jlink` Workflow

```bash
# Step 1: Find out which modules your app actually needs
#         jdeps --print-module-deps gives you the exact list
jdeps --module-path mods --print-module-deps mods/com.myapp.app.jar

# Output: java.base,java.logging
# (your app only needs these two platform modules plus your own)

# Step 2: Build the custom runtime image
jlink \
  --module-path $JAVA_HOME/jmods:mods \
  --add-modules com.myapp.app,java.base,java.logging \
  --output custom-runtime \
  --launcher myapp=com.myapp.app/com.myapp.app.Main \
  --strip-debug \
  --no-header-files \
  --no-man-pages \
  --compress=2
```

The flags used here deserve explanation. `--module-path` tells jlink where to find both the
JDK's own platform modules (in `$JAVA_HOME/jmods`) and your application's compiled modules.
`--add-modules` specifies the root modules to include — jlink transitively includes all required
modules automatically. `--output` is the directory where the custom runtime is written.
`--launcher` creates a convenient shell script that runs your application. `--strip-debug`
removes debug information to reduce size. `--compress=2` applies zip-level compression to reduce
size further. `--no-header-files` and `--no-man-pages` omit development-only content.

After running this, the `custom-runtime/` directory contains:

```
custom-runtime/
├── bin/
│   ├── java          ← The JVM
│   └── myapp         ← The generated launcher script
├── conf/
├── legal/
├── lib/
│   └── ...           ← Only the modules you asked for
└── release
```

Running the application is simply:

```bash
./custom-runtime/bin/myapp
# or
./custom-runtime/bin/java --module com.myapp.app/com.myapp.app.Main
```

### Using `jlink` With Maven

In real projects, Maven's `jlink` plugin automates this:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jlink-plugin</artifactId>
    <version>3.2.0</version>
    <extensions>true</extensions>
    <configuration>
        <launcher>myapp=com.myapp.app/com.myapp.app.Main</launcher>
        <modulePaths>
            <modulePath>${project.build.directory}/modules</modulePath>
        </modulePaths>
        <addModules>
            <addModule>com.myapp.app</addModule>
        </addModules>
        <stripDebug>true</stripDebug>
        <compress>2</compress>
        <noHeaderFiles>true</noHeaderFiles>
        <noManPages>true</noManPages>
    </configuration>
</plugin>
```

### `jlink` with Docker — Minimal Container Images

The real power of `jlink` becomes clear in Docker. A standard OpenJDK Docker image is 400-500MB.
A custom jlink image can be an order of magnitude smaller:

```dockerfile
# ──── Stage 1: Build the custom runtime ────
FROM eclipse-temurin:21-jdk AS jlink-stage

# Copy your application modules
COPY target/modules/ /app/modules/

# Build the custom runtime — only the modules the app actually needs
RUN jlink \
    --module-path $JAVA_HOME/jmods:/app/modules \
    --add-modules com.myapp.app,java.base,java.logging,java.net.http \
    --output /custom-runtime \
    --launcher myapp=com.myapp.app/com.myapp.app.Main \
    --strip-debug \
    --no-header-files \
    --no-man-pages \
    --compress=2

# ──── Stage 2: Final minimal image ────
# Use a minimal base — no JDK, no JRE needed
FROM debian:bookworm-slim

# Only copy the custom runtime from stage 1
COPY --from=jlink-stage /custom-runtime /opt/myapp-runtime

# No JVM installation needed — the runtime is self-contained
CMD ["/opt/myapp-runtime/bin/myapp"]
```

The resulting image can be as small as 40-80MB, compared to 400MB+ for a standard JDK image.

### `--show-module-resolution` — Debugging the Module Graph

When the module system fails to start because of missing or conflicting modules, this flag
traces the entire module resolution process step by step:

```bash
java --module-path mods \
     --module com.myapp.app/com.myapp.app.Main \
     --show-module-resolution
```

Sample output:

```
root com.myapp.app file:///project/mods/com.myapp.app/
  com.myapp.app requires com.myapp.service file:///project/mods/com.myapp.service/
    com.myapp.service requires com.myapp.utils file:///project/mods/com.myapp.utils/
      com.myapp.utils requires java.base (java.base)
    com.myapp.service requires java.base (java.base)
  com.myapp.app requires java.base (java.base)
```

This lets you trace exactly which modules were found, from where, and in what order dependencies
were resolved — essential for debugging "module not found" errors.

---

## Common Error Messages and How to Fix Them

Working with JPMS means learning to read its error messages. Here are the most common ones,
what they mean, and how to resolve them.

**`module X not found`**

```
Error occurred during initialization of boot layer
java.lang.module.FindException: Module com.myapp.service not found
```

This means the module system could not find `com.myapp.service` on the module path. Check
that the compiled module (`mods/com.myapp.service/` or a JAR) is present in the directory
you're passing to `--module-path`.

**`package X in module Y does not export package Z to module A`**

```
error: package com.myapp.service.internal is not visible
  (package com.myapp.service.internal is declared in module com.myapp.service,
   which does not export it)
```

You're trying to import a package that the module has not exported. Either add
`exports com.myapp.service.internal;` to the provider's `module-info.java`, or (better)
redesign so that you don't need to access the internal package.

**`module X does not read module Y`**

```
error: package com.myapp.utils is not visible
  (package com.myapp.utils is declared in module com.myapp.utils,
   which is not in the module graph)
```

You're trying to use a class from a module that isn't in your `requires` clause. Add
`requires com.myapp.utils;` to your `module-info.java`.

**`InaccessibleObjectException` at Runtime**

```
java.lang.reflect.InaccessibleObjectException:
  Unable to make field private final java.lang.String com.myapp.domain.Order.id
  accessible: module com.myapp.domain does not "opens com.myapp.domain" to module org.hibernate.orm.core
```

A framework is trying to reflect on your code and cannot because the package is not open.
Add `opens com.myapp.domain to org.hibernate.orm.core;` to your `module-info.java`, or use
`--add-opens com.myapp.domain/com.myapp.domain=ALL-UNNAMED` on the command line.

**`split package` Error**

```
Error occurred during initialization of boot layer
java.lang.LayerInstantiationException: Package com.example.util in both module alpha and module beta
```

The package `com.example.util` appears in two different modules on the module path. This is
forbidden by JPMS. Resolve by removing the duplicate or moving the split package into a single
shared module.

---

## ⚠️ Important Points and Best Practices

**Use `jdeps` before attempting any JPMS migration.** Running `jdeps --jdk-internals` on your
existing JARs tells you exactly what you're in for before you write a single `module-info.java`.
This intelligence shapes your migration plan significantly.

**Always use `jdeps --print-module-deps` before running `jlink`.** This gives you the minimum
set of platform modules required. Adding modules you don't need to a `jlink` image defeats its
purpose of size reduction.

**Commit `--add-opens` and `--add-exports` flags to version control, but plan to eliminate them.**
When you first migrate and need many of these flags for framework compatibility, add them to
your build tool configuration and commit them. Then create a backlog ticket to eliminate each
one by proper `opens` declarations or by upgrading the framework that requires them.

**Prefer `--module-path` over `--classpath` for all new code.** Every JAR you put on the module
path gets at least automatic module treatment with a name that other modules can depend on.
JARs on the classpath are in the unnamed module, which named modules cannot `requires`. As you
migrate, progressively move JARs from the classpath to the module path.

**Set `Automatic-Module-Name` in any library JAR you publish.** If your project produces a
library JAR that others depend on, always add the `Automatic-Module-Name` manifest attribute
with a stable, well-chosen name before anyone starts putting it on their module path. Changing
this name later is a breaking change for all your consumers.

**The `--module-source-path` flag is your friend for multi-module projects.** When you have
many modules, compiling each one individually with separate `javac` commands quickly becomes
unmanageable. The `--module-source-path` flag lets `javac` compile an entire multi-module
project tree in a single command, resolving cross-module dependencies automatically.

**`jlink` is the endgame for production deployments.** If your entire dependency stack is
fully modular, using `jlink` to produce a custom runtime has compelling advantages: smaller
Docker images, faster container startup, smaller attack surface, and the ability to deploy on
targets that don't have a JVM installed. This is the compelling payoff for the migration effort.
