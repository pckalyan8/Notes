# 🏗️ Phase 1 — Setting Up & Developer Environment
### Getting Productive Before You Write a Single Line of Code

---

> **Why This Phase Matters**
>
> Your development environment is your workshop. A carpenter with dull tools and a disorganized bench works slower and makes more mistakes than one with sharp tools and a clean, well-arranged workspace. The same is true for software development. Investing time upfront to properly configure your JDK, IDE, build tool, and version control will save you hours every week and prevent countless frustrating errors. This phase teaches you not just *how* to install things, but *why* each tool exists and *how* they work together.

---

## 1.1 — JDK Installation & Management

### What Is the JDK?

Before installing anything, it's important to understand what you're installing. There are three related but distinct concepts:

**JDK (Java Development Kit)** is the full toolkit for *writing* Java programs. It includes the Java compiler (`javac`), the Java runtime, the Java debugger (`jdb`), tools like `jconsole`, `jmap`, `jstack`, and all the standard library source code. If you're writing Java, you need the JDK.

**JRE (Java Runtime Environment)** is a subset of the JDK — it contains only what's needed to *run* Java programs, not to compile them. The JRE used to be distributed separately, but since Java 11, Oracle stopped shipping a standalone JRE. Modern JDK distributions include everything.

**JVM (Java Virtual Machine)** is the engine that actually *executes* compiled Java bytecode. The JVM is part of the JRE, which is part of the JDK. When you say "Java runs on the JVM," you mean the JVM reads your `.class` files and runs them.

---

### Choosing a JDK Distribution

As discussed in Phase 0, multiple organizations distribute JDK builds. Here's how to decide:

**Eclipse Temurin (from Adoptium)** is the community recommendation for most developers. It's free, open source, based on OpenJDK, receives regular security updates, and is backed by major companies like Microsoft, Red Hat, and IBM. Available at `https://adoptium.net`.

**Amazon Corretto** is Amazon's free distribution, optimized for running on AWS but perfectly usable everywhere. It's a safe choice if you deploy to AWS infrastructure. Available at `https://aws.amazon.com/corretto/`.

**Oracle JDK** is Oracle's official distribution. Since Java 17, Oracle JDK is free under the Oracle No-Fee Terms and Conditions (NFTC) for all uses including production. However, Oracle JDK has historically had commercial licensing complexities, so many teams default to Temurin or Corretto.

**Microsoft Build of OpenJDK** is optimized for Azure. If you deploy to Azure, this is a sensible choice. Otherwise, Temurin or Corretto are equally good.

**The practical advice:** Download **Eclipse Temurin LTS (Java 21 or Java 17)** from `https://adoptium.net`. Select your OS, architecture (x64 for Intel/AMD, aarch64 for Apple Silicon or ARM), and choose the JDK package (not JRE).

---

### Installing the JDK on Different Operating Systems

**On macOS:** Download the `.pkg` file from Adoptium and run it. The installer places the JDK at `/Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home`. Alternatively, use **Homebrew**: `brew install --cask temurin@21`.

**On Windows:** Download the `.msi` installer and run it. The installer handles the PATH configuration for you. The JDK is typically installed at `C:\Program Files\Eclipse Adoptium\jdk-21.x.x-hotspot\`.

**On Linux (Debian/Ubuntu):** You can use the package manager: `sudo apt install temurin-21-jdk` after adding the Adoptium apt repository. Alternatively, download the `.tar.gz`, extract it to `/usr/lib/jvm/`, and configure PATH manually.

---

### Environment Variables: JAVA_HOME and PATH

Two environment variables are fundamental to Java development:

**JAVA_HOME** is a variable that points to the root directory of your JDK installation. Many tools — Maven, Gradle, Tomcat, and others — look for `JAVA_HOME` to find the Java installation. Without it, these tools may fail to start or may pick up the wrong Java version.

**PATH** is the list of directories the operating system searches when you type a command. Adding the JDK's `bin` directory to PATH means you can type `java`, `javac`, and `jdb` from any terminal window.

Here's what configuring these looks like on each platform:

**On macOS/Linux**, you add the following to your shell configuration file (`~/.zshrc` for Zsh on modern macOS, `~/.bashrc` for Bash on Linux):

```bash
# Set JAVA_HOME to the JDK installation directory
export JAVA_HOME=/Library/Java/JavaVirtualMachines/temurin-21.jdk/Contents/Home

# Add JAVA_HOME/bin to the front of PATH so 'java' and 'javac' commands are found
export PATH=$JAVA_HOME/bin:$PATH
```

After adding these lines, run `source ~/.zshrc` (or open a new terminal) to apply the changes.

**On Windows**, set environment variables through System Properties → Advanced → Environment Variables. Create a new System Variable named `JAVA_HOME` with the value `C:\Program Files\Eclipse Adoptium\jdk-21.x.x-hotspot`. Then edit the `Path` system variable and add `%JAVA_HOME%\bin`.

---

### Verifying Your Installation

After installation, open a new terminal and run:

```bash
# This should print something like: java version "21.0.x" ...
java -version

# This should print something like: javac 21.0.x
javac -version

# This should print the JAVA_HOME path
echo $JAVA_HOME   # macOS/Linux
echo %JAVA_HOME%  # Windows
```

If these commands work, your JDK is correctly installed and configured. If `java: command not found` appears, the `bin` directory hasn't been added to PATH correctly.

---

### Managing Multiple JDK Versions: SDKMAN and jenv

Real-world developers often need multiple Java versions simultaneously. Perhaps a client's project requires Java 11, your new project uses Java 21, and you're learning a Java 17 feature. Switching `JAVA_HOME` manually every time is tedious and error-prone.

**SDKMAN! (Software Development Kit Manager)** is a command-line tool for managing multiple versions of JDKs and other Java-ecosystem tools (Maven, Gradle, Kotlin). It works on macOS and Linux (and via WSL on Windows).

```bash
# Install SDKMAN! (run in terminal)
curl -s "https://get.sdkman.io" | bash

# After installation, install a specific Java version
sdk install java 21.0.2-tem    # Temurin 21
sdk install java 17.0.9-tem    # Temurin 17
sdk install java 11.0.22-tem   # Temurin 11

# Switch the current terminal session to Java 17
sdk use java 17.0.9-tem

# Set the default Java version system-wide
sdk default java 21.0.2-tem

# List all available Java distributions and versions
sdk list java
```

SDKMAN also installs Maven, Gradle, and many other tools with a single command, making environment setup on a new machine trivially easy.

**jenv** is an alternative version manager focused specifically on Java. It's modeled after rbenv (Ruby) and pyenv (Python). jenv is popular on macOS.

---

## 1.2 — IDEs & Editors

An IDE (Integrated Development Environment) is far more than a text editor. A good Java IDE understands the structure of your code, provides intelligent code completion, catches errors as you type, lets you navigate to any class or method instantly, and integrates the debugger, build tools, and version control into a single interface. For Java in particular, a good IDE can multiply your productivity by a factor of five or more.

---

### IntelliJ IDEA — The Professional's Choice

**IntelliJ IDEA**, made by JetBrains, is widely considered the best Java IDE and the de facto standard for professional Java development. It has two editions:

**Community Edition (free and open source)** supports Java, Kotlin, Groovy, Maven, Gradle, Git, and basic web development. For most Java learning and development, the Community Edition is sufficient.

**Ultimate Edition (paid, free for students)** adds support for Spring Boot, Java EE/Jakarta EE, advanced database tools, HTTP client, Kubernetes, and many frameworks. If you're doing enterprise Java with Spring, the Ultimate Edition's Spring-specific support (detecting bean injection issues, navigating Spring Boot configuration, etc.) is genuinely valuable.

**Key IntelliJ concepts to learn early:**

*Project structure* — IntelliJ organizes code into Projects, Modules, and Packages. A Project corresponds to one application. Modules are sub-projects within a Project (common in multi-module Maven/Gradle builds). Packages map to directory structures.

*Code completion* — Press `Ctrl+Space` (Windows/Linux) or `Ctrl+Space` (macOS) to trigger code completion. IntelliJ analyzes the context and suggests methods, variables, classes, and keywords. It learns your coding patterns and improves over time.

*Navigation* — `Ctrl+N` (Navigate → Class) lets you jump to any class by name. `Ctrl+Shift+N` navigates to any file. `Ctrl+Alt+B` navigates to the implementation of a method. `Ctrl+B` goes to the declaration of a symbol. These shortcuts make navigating large codebases effortless.

*Refactoring* — IntelliJ's refactoring tools are best-in-class. `Shift+F6` renames a variable, method, or class and updates all usages across the entire project. `Ctrl+Alt+M` extracts selected code into a new method. `Ctrl+Alt+V` extracts an expression into a variable. These refactorings are safe — IntelliJ understands Java semantics, not just text.

*Live Templates* — Typing `sout` and pressing Tab generates `System.out.println()`. Typing `psvm` generates the main method. You can create your own templates for common code patterns.

*Inspections* — IntelliJ continuously analyzes your code and highlights issues: potential NullPointerExceptions, unused variables, resources not closed in try-with-resources, unnecessarily complex expressions, and hundreds of other patterns. The suggestions are almost always correct and educational.

---

### Eclipse — The Historical Standard

**Eclipse** was the dominant Java IDE for over a decade (roughly 2001–2015) and remains widely used in enterprises with legacy setups.

Eclipse organizes work into a **Workspace** — a directory on your filesystem that contains your projects and IDE preferences. This workspace concept differs from IntelliJ, where preferences are per-project. The workspace approach can cause confusion when you open the same project in a different workspace.

Eclipse uses a plugin architecture — the core is minimal, and you add functionality through plugins. The **Eclipse IDE for Java Developers** package bundles the most common plugins. The **Spring Tool Suite (STS)** is an Eclipse distribution optimized for Spring development.

If you encounter an Eclipse-based project, the key to getting started is understanding **Maven/Gradle integration**: import the project via File → Import → Existing Maven Projects (or Gradle Projects), and Eclipse will configure itself from the `pom.xml` or `build.gradle`.

---

### Visual Studio Code — The Lightweight Challenger

**VS Code** from Microsoft has become a capable Java IDE through extensions, particularly relevant for developers who already use VS Code for other languages.

The **Extension Pack for Java** (published by Microsoft) bundles six extensions that together provide:
- Language support (syntax highlighting, code completion, error detection) via the Language Server Protocol
- Debugging support
- Testing support (JUnit, TestNG)
- Maven and Gradle project support
- IntelliCode (AI-assisted completions)

VS Code is lighter than IntelliJ and Eclipse, starts faster, and is familiar to JavaScript/TypeScript developers. However, its Java support — while good — is less mature than IntelliJ's for complex refactoring, Spring Boot introspection, and Java EE features.

**The recommendation:** Start with IntelliJ IDEA Community Edition. It's the best tool for learning Java deeply, and it's what most professional Java teams use.

---

### Essential IDE Skills: The Debugger

The debugger is the most under-used power tool in a beginner's arsenal. Learning to debug properly is often the difference between spending 5 minutes or 5 hours finding a bug.

**Breakpoints** tell the debugger to pause execution at a specific line. Set one by clicking in the gutter (the margin to the left of line numbers) in IntelliJ. When execution reaches that line, the program pauses and you can inspect everything.

**Step Over (F8 in IntelliJ)** executes the current line and moves to the next one. If the current line calls a method, that method runs completely before the debugger pauses again.

**Step Into (F7)** enters the method being called on the current line, letting you follow execution inside it.

**Step Out (Shift+F8)** finishes the current method and pauses when execution returns to the calling method.

**Watches and Evaluate Expression** allow you to inspect any variable's value and even evaluate arbitrary expressions while execution is paused. You can call methods, modify values, and explore the program state interactively.

**The mental model:** The debugger lets you turn a running program into a "movie" you can pause, rewind (via restart), and step through frame by frame. Every time you encounter a bug you don't immediately understand, your first move should be to reproduce it under the debugger.

---

## 1.3 — Build Tools

### The Problem Build Tools Solve

Imagine you have a Java project with 50 source files. You also depend on 20 external libraries (Jackson for JSON, JUnit for testing, Spring Core, etc.). Without a build tool, you'd need to:

1. Download all 20 libraries as JAR files manually
2. Compile your 50 source files with `javac`, specifying every library JAR in the classpath
3. Run your tests by specifying all the classpaths again
4. Package everything into a JAR or WAR file manually
5. Repeat this entire process every time you change anything

Build tools automate all of this. They manage dependencies (downloading them from repositories automatically), define a lifecycle (compile → test → package → deploy), and provide a reproducible process that works identically on every developer's machine and on CI/CD servers.

---

### Maven: Convention Over Configuration

**Apache Maven** is the most widely used Java build tool, particularly in enterprise environments. Maven introduced the revolutionary idea of **Convention Over Configuration**: if you structure your project the Maven way, you get sensible defaults for everything without configuration.

The Maven conventional project structure looks like this:

```
my-project/
├── pom.xml                        ← Project Object Model (the heart of Maven)
└── src/
    ├── main/
    │   ├── java/                  ← Your application source code goes here
    │   │   └── com/example/
    │   │       └── App.java
    │   └── resources/             ← Config files, templates, properties
    │       └── application.properties
    └── test/
        ├── java/                  ← Your test source code goes here
        │   └── com/example/
        │       └── AppTest.java
        └── resources/             ← Test-specific config files
```

This structure is universal across Maven projects. Any Maven developer can open any Maven project and immediately know where to find source code, tests, and resources — no explanatory documentation needed.

---

### The POM (Project Object Model)

The `pom.xml` file is the heart of a Maven project. It's an XML file that declares everything about your project: its identity, its dependencies, how it should be built, and what plugins to use.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <!-- Maven POM version — always 4.0.0 for Maven 2/3 -->
    <modelVersion>4.0.0</modelVersion>

    <!-- GAV coordinates — uniquely identify this project -->
    <groupId>com.example</groupId>      <!-- Your organization/domain, reversed -->
    <artifactId>my-app</artifactId>     <!-- The name of this specific project -->
    <version>1.0.0-SNAPSHOT</version>   <!-- SNAPSHOT = still in development -->
    <packaging>jar</packaging>          <!-- Output format: jar, war, pom, etc. -->

    <!-- Metadata -->
    <name>My Application</name>
    <description>A sample Java application</description>

    <!-- Global properties — referenced as ${java.version} -->
    <properties>
        <java.version>21</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!-- Centralize dependency versions here -->
        <junit.version>5.10.2</junit.version>
        <jackson.version>2.17.0</jackson.version>
    </properties>

    <!-- Dependencies section — declare what libraries you need -->
    <dependencies>

        <!-- A compile-time dependency: available everywhere -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
            <!-- scope defaults to 'compile' if not specified -->
        </dependency>

        <!-- A test-scope dependency: only available in tests -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>   <!-- Not included in final JAR -->
        </dependency>

        <!-- A provided dependency: available at compile time, 
             but the runtime environment supplies it (e.g., servlet container) -->
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <!-- Maven Compiler Plugin: controls Java version for compilation -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.12.1</version>
                <configuration>
                    <release>${java.version}</release>
                </configuration>
            </plugin>

            <!-- Maven Surefire Plugin: controls how unit tests are run -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.2.5</version>
            </plugin>
        </plugins>
    </build>

</project>
```

The **GAV coordinates** (GroupId, ArtifactId, Version) are how Maven uniquely identifies every artifact in the universe of Java libraries. When you declare a dependency, Maven downloads the corresponding JAR from **Maven Central** (the central public repository at `https://repo1.maven.org/maven2`) and caches it in your **local repository** (at `~/.m2/repository` on macOS/Linux or `C:\Users\<user>\.m2\repository` on Windows). This cache means a library is only downloaded once, regardless of how many projects use it.

---

### The Maven Build Lifecycle

Maven defines a **build lifecycle** — an ordered sequence of phases. When you run a phase, Maven executes all phases that come before it in order.

The three built-in lifecycles are `default` (for project build), `clean` (for cleaning), and `site` (for documentation generation). The `default` lifecycle's phases in order are:

`validate` → `compile` → `test` → `package` → `verify` → `install` → `deploy`

Running `mvn package` will execute validate, compile, test, and then package. Running `mvn test` will execute validate, compile, and then test. You don't need to specify earlier phases; Maven figures it out.

The most commonly used Maven commands are:

```bash
# Compile source code and run all tests, then create a JAR file
mvn package

# Like package, but also copies the JAR to your local .m2 repository
# (so other local projects can depend on it)
mvn install

# Delete the target/ directory (compiled classes and artifacts)
mvn clean

# The most common combination: clean everything first, then build
mvn clean install

# Run only the tests (compile source if needed)
mvn test

# Skip running tests (useful when you need a JAR quickly for deployment)
mvn package -DskipTests

# Download all project dependencies (useful for offline work setup)
mvn dependency:resolve

# Show the entire dependency tree, including transitive dependencies
mvn dependency:tree
```

---

### Gradle: Power and Flexibility

**Gradle** is the newer, more powerful build tool. It was created to address Maven's verbosity and inflexibility. Gradle replaces XML configuration with a **Groovy DSL** or **Kotlin DSL** — essentially a programming language for your build script. This makes complex builds far more expressible.

Gradle is the default build tool for Android development and is increasingly preferred for large, complex Java projects. Spring Boot's own build uses Gradle.

A Gradle build script (`build.gradle`) for a Java project looks like this:

```groovy
// Apply the Java plugin — this provides tasks like compileJava, test, jar
plugins {
    id 'java'
    id 'application'  // Adds 'run' task to execute the application
}

// Project identity — equivalent to Maven's GAV
group = 'com.example'
version = '1.0.0'

// Tells Gradle where to find dependencies
repositories {
    mavenCentral()  // Use Maven Central repository
}

// Java compiler configuration
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

// Dependencies section — concise compared to Maven
dependencies {
    // 'implementation' = compile + runtime, not exposed to dependents
    implementation 'com.fasterxml.jackson.core:jackson-databind:2.17.0'

    // 'testImplementation' = only available in tests
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

// Configure the main class for the 'run' task
application {
    mainClass = 'com.example.App'
}

// Configure test framework
test {
    useJUnitPlatform()  // Tell Gradle to use JUnit 5
}
```

The equivalent in **Kotlin DSL** (`build.gradle.kts`), which is increasingly preferred because IDEs provide better code completion for it:

```kotlin
plugins {
    java
    application
}

group = "com.example"
version = "1.0.0"

repositories {
    mavenCentral()
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}

dependencies {
    implementation("com.fasterxml.jackson.core:jackson-databind:2.17.0")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.2")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
}

application {
    mainClass.set("com.example.App")
}

tasks.test {
    useJUnitPlatform()
}
```

**Common Gradle commands:**

```bash
# List all available tasks
./gradlew tasks

# Compile and run all tests
./gradlew test

# Build the project (compile + test + create JAR)
./gradlew build

# Run the application (requires 'application' plugin)
./gradlew run

# Clean build outputs
./gradlew clean

# Clean then build
./gradlew clean build

# Build without running tests
./gradlew build -x test
```

Note the `./gradlew` command (the **Gradle Wrapper**). Every Gradle project includes a `gradlew` script that downloads and uses the exact Gradle version specified in `gradle/wrapper/gradle-wrapper.properties`. This guarantees that everyone who builds the project uses the same Gradle version, eliminating "works on my machine" build inconsistencies. Always use `./gradlew` rather than a globally installed `gradle` command.

---

### Maven vs. Gradle: Which Should You Learn?

Both are used in production. For learning purposes, start with **Maven** because:
- Its lifecycle is simpler and easier to reason about
- The vast majority of tutorials, Stack Overflow answers, and documentation use Maven
- Spring Initializr generates Maven projects by default
- XML is verbose but unambiguous — there's no "how does this code run" mystery

Once you're comfortable with Maven, learn Gradle fundamentals because:
- It's faster (incremental builds only recompile changed code; Gradle's caching is smarter)
- It's required for Android development
- Large multi-module projects are more manageable in Gradle
- Modern Spring projects increasingly use Gradle

---

## 1.4 — Version Control with Git

### Why Every Java Developer Must Know Git

Git is not optional. Every professional software team uses version control, and the overwhelming majority use Git. Git allows multiple developers to work on the same codebase simultaneously without overwriting each other's changes. It creates a complete, reviewable history of every change ever made, allowing you to understand *why* the code is the way it is and to safely revert mistakes.

---

### Core Git Concepts

**Repository (repo):** A directory that Git tracks. It contains your project files and a hidden `.git` directory that stores the entire history of changes.

**Working Directory:** The actual files on your disk that you see and edit.

**Staging Area (Index):** A preparation zone where you assemble changes you intend to commit. You choose which modified files to stage; only staged files are included in the next commit. This lets you make multiple changes but commit them in logical groups.

**Commit:** A snapshot of the staged changes, saved permanently to Git's history. Each commit has a unique SHA-1 hash identifier, a message describing the change, and a pointer to the previous commit. Together, commits form a chain of snapshots representing the entire history of the project.

**Branch:** A movable pointer to a specific commit. When you create a new branch, you create a new pointer. When you commit on that branch, the pointer moves forward. Branches let you work on new features or bug fixes in isolation, without disturbing the main codebase.

**Remote:** A copy of the repository hosted on a server (like GitHub, GitLab, or Bitbucket). You push your commits to the remote and pull others' commits from it.

---

### Essential Git Commands Every Java Developer Uses Daily

```bash
# Initialize a new Git repository in the current directory
git init

# Clone (download) a repository from GitHub to your local machine
git clone https://github.com/user/repository.git

# Show the current state: which files are modified, staged, or untracked
git status

# Stage a specific file for the next commit
git add src/main/java/com/example/App.java

# Stage ALL modified and new files
git add .

# Commit the staged changes with a descriptive message
# (present tense imperative: "Add feature" not "Added feature")
git commit -m "Add user authentication endpoint"

# See the history of commits
git log

# More readable log: one line per commit
git log --oneline --graph --all

# Show what changed in a specific commit
git show abc1234

# Push your local commits to the remote repository
git push origin main

# Pull the latest commits from the remote
git pull origin main

# Create a new branch and switch to it
git checkout -b feature/user-authentication
# Or the modern syntax:
git switch -c feature/user-authentication

# Switch to an existing branch
git checkout main
git switch main

# Merge a branch into the current branch
git merge feature/user-authentication

# Discard all uncommitted changes in a file (DESTRUCTIVE — can't be undone)
git checkout -- src/main/java/com/example/App.java
git restore src/main/java/com/example/App.java  # Modern syntax

# Unstage a staged file (without discarding changes)
git restore --staged src/main/java/com/example/App.java

# See what changed in working directory (unstaged changes)
git diff

# See what's staged (about to be committed)
git diff --staged
```

---

### Branching Strategies

**Git Flow** is a branching model with strict rules. It uses two main branches (`main`/`master` for production code, `develop` for integration), plus supporting branches (`feature/*`, `release/*`, `hotfix/*`). Git Flow is common in teams that maintain multiple versions simultaneously or have formalized release cycles.

**Trunk-Based Development** is simpler: everyone commits to a single main branch (the "trunk"), using short-lived feature branches that are merged back within hours or days. This approach is favored for continuous delivery, where code is deployed frequently. It requires good automated testing to catch regressions early.

**For most Java projects** (especially learning projects), the workflow is simple: work on `main` directly for solo projects, or use `feature/*` branches that you merge to `main` when complete for team projects.

---

### The `.gitignore` File for Java Projects

A `.gitignore` file tells Git which files and directories to ignore. For Java projects, you should never commit compiled class files, build outputs, IDE-specific configuration, or OS metadata files.

A solid `.gitignore` for a Java/Maven/IntelliJ project:

```gitignore
# ===== COMPILED OUTPUT =====
target/          # Maven build output directory
build/           # Gradle build output directory
*.class          # Compiled Java bytecode
*.jar            # Packaged JARs (project's own output)
*.war            # Web archives
*.ear            # Enterprise archives

# ===== IDE FILES =====
.idea/           # IntelliJ IDEA project files
*.iml            # IntelliJ module files
*.iws            # IntelliJ workspace files
.project         # Eclipse project file
.classpath       # Eclipse classpath file
.settings/       # Eclipse project settings
.vscode/         # VS Code settings

# ===== OS FILES =====
.DS_Store        # macOS Finder metadata
Thumbs.db        # Windows thumbnail cache
desktop.ini      # Windows folder configuration

# ===== GRADLE =====
.gradle/         # Gradle cache
gradle-app.setting

# ===== LOGS =====
*.log
logs/

# ===== ENVIRONMENT / SECRETS =====
.env             # Environment variable files (NEVER commit secrets!)
*.properties.local
application-local.yml
```

GitHub provides auto-generated `.gitignore` templates at `https://github.com/github/gitignore`. When creating a repository, select "Java" and GitHub generates an appropriate file.

---

## 1.5 — Writing and Running Your First Java Program

### Anatomy of a Java File

Every Java source file contains one public class, and the filename must exactly match the public class name. Here is the minimal Java program:

```java
// File: src/main/java/com/example/HelloWorld.java

// Package declaration: mirrors the directory structure
// com/example/HelloWorld.java → package com.example;
package com.example;

/**
 * This is a Javadoc comment. It documents the class for API documentation generation.
 * The HelloWorld class demonstrates the minimum structure of a Java program.
 */
public class HelloWorld {

    /**
     * The main method is the entry point of any standalone Java application.
     * When you run 'java com.example.HelloWorld', the JVM calls this method first.
     *
     * @param args Command-line arguments passed when starting the program.
     *             For example: java HelloWorld arg1 arg2
     *             args[0] would be "arg1", args[1] would be "arg2"
     */
    public static void main(String[] args) {
        // This is a single-line comment
        
        /*
         * This is a multi-line comment.
         * It can span multiple lines.
         */
        
        System.out.println("Hello, World!");  // Print with a newline at the end
        System.out.print("No newline here");  // Print without newline
        System.out.printf("Hello, %s! You are %d years old.%n", "Alice", 30); // Formatted
    }
}
```

Let's dissect every keyword in the method signature `public static void main(String[] args)`:

**`public`** — This is an access modifier. `public` means this method is accessible from anywhere. The JVM needs to call `main` from outside the class, so it must be public.

**`static`** — This method belongs to the class itself, not to any instance of the class. When the JVM starts your program, no objects of `HelloWorld` exist yet — the JVM calls `main` directly on the class without creating an instance first. Therefore, `main` must be static.

**`void`** — The return type. `void` means this method returns nothing. The JVM doesn't expect a return value from `main` (unlike C, which returns an exit code from `main`).

**`main`** — The special name the JVM looks for. This exact name, with this exact signature, is the program's entry point.

**`String[] args`** — The parameter. `String[]` is an array of Strings. When you run `java HelloWorld Alice 30`, the JVM collects command-line arguments into this array: `args[0]` = "Alice", `args[1]` = "30".

*(Note: Java 21 introduced "Unnamed Classes and Instance Main Methods" as a preview feature, which relaxes these requirements for simple programs. But understanding the full signature is essential.)*

---

### The Compilation and Execution Pipeline

Understanding what happens when you compile and run Java is foundational knowledge that will serve you for your entire career.

**Step 1 — Source Code (`.java` files):** You write human-readable Java source code in `.java` files. This is text that you can read and edit.

**Step 2 — Compilation (`javac`):** The Java compiler `javac` reads your `.java` source files and performs several tasks:
- It checks your code for syntax errors and type errors
- It resolves all the names you use (verifying that `String` class exists, that `println` is a valid method on `System.out`, etc.)
- It generates **bytecode** — a compact, platform-neutral instruction set — stored in `.class` files

Compiling from the command line:

```bash
# Single file compilation
javac HelloWorld.java

# With packages: must compile from project root and specify source path
javac -d out src/main/java/com/example/HelloWorld.java

# Compile all Java files in a directory recursively
find src -name "*.java" -print | xargs javac -d out

# Compile with specific classpath (dependencies):
javac -cp "libs/jackson-databind.jar:libs/jackson-core.jar" \
      -d out \
      src/main/java/com/example/App.java
```

The result is a `.class` file for each class. The bytecode inside is not readable by humans, but it's compact and precisely defined.

**Step 3 — Bytecode (`.class` files):** The bytecode is a set of instructions for an idealized, abstract stack-based machine — the JVM. Bytecode instructions include things like "push this integer onto the stack," "call this method," "create a new object," etc. The key insight: bytecode doesn't target any specific CPU. It targets the abstract JVM machine.

**Step 4 — Execution (`java`):** When you run your program, the JVM's class loader reads the `.class` file. The JVM's verifier checks that the bytecode is valid and safe. The JVM's interpreter begins executing the bytecode instructions. As it runs, the JIT compiler identifies hot code paths and compiles them to native machine code for the specific CPU you're running on.

```bash
# Run a compiled class (specify the fully-qualified class name with package)
java com.example.HelloWorld

# Run with classpath
java -cp out:libs/jackson-databind.jar com.example.HelloWorld

# Pass command-line arguments
java com.example.HelloWorld Alice 30

# Set JVM options (heap size, GC choice, etc.)
java -Xms256m -Xmx1g -XX:+UseG1GC com.example.HelloWorld

# Run a JAR file (requires Main-Class in JAR's manifest)
java -jar my-application.jar
```

**Step 5 — JIT Compilation:** As the JVM runs your program, it monitors which code runs most frequently. After a method has been called roughly 10,000 times (the exact threshold varies), the JIT compiler kicks in, compiles that method to optimized native machine code, and replaces the interpreted execution path. From that point on, the native code runs at full CPU speed. This is why Java programs often get *faster* over time as they warm up.

---

### Package Structure and Directory Layout

**Packages** are Java's namespace mechanism. They prevent naming conflicts (two different companies can both have a class called `User` without conflict if they're in different packages) and provide a logical organization for classes.

The universal convention is to use your domain name reversed as the package prefix:
- Google uses `com.google.*`
- Spring Framework uses `org.springframework.*`
- Apache Commons uses `org.apache.commons.*`

For a company `example.com` building an app called `myapp`:
- `com.example.myapp` — root package
- `com.example.myapp.model` — domain model classes (User, Order, Product)
- `com.example.myapp.service` — business logic
- `com.example.myapp.repository` — data access
- `com.example.myapp.controller` — HTTP request handling
- `com.example.myapp.config` — configuration classes
- `com.example.myapp.exception` — custom exception classes
- `com.example.myapp.util` — utility classes

The package structure must mirror the directory structure. The class `com.example.myapp.service.UserService` must be in a file at `src/main/java/com/example/myapp/service/UserService.java`. This mapping is mandatory, not conventional.

---

### Understanding `.class` Files and Bytecode

You'll rarely look at bytecode directly, but knowing it exists is important for understanding how Java works under the hood.

The `javap` tool (included in the JDK) can disassemble `.class` files and show you the bytecode:

```bash
# Show bytecode instructions for a compiled class
javap -c HelloWorld.class

# Show verbose output including the constant pool
javap -verbose HelloWorld.class
```

The output looks like this for `System.out.println("Hello, World!")`:

```
0: getstatic     #7  // Field java/lang/System.out:Ljava/io/PrintStream;
3: ldc           #13 // String Hello, World!
5: invokevirtual #15 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
8: return
```

Each line is a JVM instruction: get the `System.out` static field, load the string constant, call `println` on it, return. This byte-by-byte precision is what makes Java portable — any JVM on any platform executes these exact instructions the same way.

---

## Summary: What to Take From Phase 1

You now have a complete picture of the Java development environment:

The **JDK** is your compiler and runtime. Managing multiple versions with SDKMAN lets you work across different projects without manual configuration changes.

Your **IDE** (preferably IntelliJ IDEA) is far more than a text editor — it understands Java's type system, guides you with completions and warnings, and lets you explore and refactor code at the semantic level. The debugger is your most powerful diagnostic tool.

**Maven and Gradle** are your build systems. They manage dependencies automatically, define a reproducible build process, and integrate with your IDE and CI/CD pipeline. Maven is the more common choice for beginners and enterprises; Gradle for Android and complex multi-module projects.

**Git** is non-negotiable. Every change you make from this point forward should be committed to a Git repository. Practicing Git commands daily is the only way to internalize them.

The **compilation pipeline** — from `.java` source to bytecode to JIT-compiled native code — is the mechanism that makes Java both platform-independent and high-performance. Understanding this pipeline means you'll never be mystified by the relationship between source files, class files, JARs, and running programs.

With this foundation, you are ready for Phase 2: the Java language itself.

---

*Next: Phase 2 — Java Language Fundamentals*
