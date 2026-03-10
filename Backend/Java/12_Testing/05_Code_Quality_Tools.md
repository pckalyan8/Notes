# 🔬 12.5 — Code Quality Tools
## JaCoCo · SonarQube · Checkstyle · PMD · SpotBugs · Pitest

Writing tests is only half the discipline. The other half is continuously measuring and enforcing quality — automatically, as part of every build and every pull request — so that standards don't erode quietly over time as deadlines increase and team sizes grow. This chapter covers the six tools that make quality enforcement automatic rather than human-dependent.

---

## 🧠 The Big Picture — Why Automated Quality Gates Exist

Imagine a team of ten developers each committing code three times a day. No human reviewer can inspect every line for style violations, duplicate code, null pointer risks, dead code, and test coverage simultaneously without introducing mistakes or becoming a bottleneck. Automated quality tools do this inspection consistently, on every commit, in seconds, and never get tired or skip a file.

Each tool in this chapter guards a different dimension of quality: JaCoCo measures how much of your code is touched by tests, SonarQube analyses the code itself for bugs and design problems, Checkstyle enforces formatting and style rules, PMD catches common programming mistakes, SpotBugs finds patterns that reliably lead to runtime crashes, and Pitest tests whether your tests are actually meaningful rather than just high-coverage.

---

## 1. JaCoCo — Code Coverage Reports

JaCoCo (Java Code Coverage) instruments your bytecode at class-load time and records which lines and branches were executed during the test run. It then generates human-readable HTML reports and machine-readable XML files that CI tools and SonarQube can consume.

### Maven Setup

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>

        <!-- 1. Prepare the JaCoCo agent before tests run -->
        <execution>
            <id>prepare-agent</id>
            <goals><goal>prepare-agent</goal></goals>
        </execution>

        <!-- 2. Generate the HTML and XML report after tests finish -->
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>

        <!-- 3. Enforce minimum coverage thresholds — fail the build if not met -->
        <execution>
            <id>check</id>
            <phase>verify</phase>
            <goals><goal>check</goal></goals>
            <configuration>
                <rules>
                    <rule>
                        <!-- BUNDLE = entire project; CLASS = per class; PACKAGE = per package -->
                        <element>BUNDLE</element>
                        <limits>
                            <!-- Line coverage: at least 80% of all executable lines must be covered -->
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.80</minimum>
                            </limit>
                            <!-- Branch coverage: at least 70% of all branches must be covered -->
                            <limit>
                                <counter>BRANCH</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.70</minimum>
                            </limit>
                        </limits>
                    </rule>

                    <!-- Stricter threshold for the service layer -->
                    <rule>
                        <element>PACKAGE</element>
                        <includes>
                            <include>com/example/service/*</include>
                        </includes>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>0.90</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

Running `mvn verify` now does three things in sequence: runs your tests, generates the coverage report at `target/site/jacoco/index.html`, and fails the build if coverage drops below the configured thresholds. The HTML report shows each class file with green (covered), red (not covered), and yellow (partially covered — one branch taken but not the other) highlighting.

### Excluding Classes from Coverage

Not every class benefits from a coverage requirement. Configuration classes, generated code, and main application entry points should typically be excluded so they don't dilute the meaningful coverage metrics.

```xml
<configuration>
    <excludes>
        <!-- Exclude Spring Boot main class — untestable in unit tests -->
        <exclude>com/example/SchoolApplication.class</exclude>
        <!-- Exclude all @Configuration classes -->
        <exclude>com/example/config/**</exclude>
        <!-- Exclude Lombok-generated classes (they show up as uncovered boilerplate) -->
        <exclude>com/example/model/**DTO.class</exclude>
        <!-- Exclude generated code -->
        <exclude>com/example/generated/**</exclude>
    </excludes>
</configuration>
```

You can also exclude a specific class using the `@ExcludeFromJacocoGeneratedReport` annotation (JaCoCo 0.8.2+) or by annotating generated/boilerplate methods with `@Generated`:

```java
// Marks a method as generated code — JaCoCo will exclude it from coverage metrics
@Generated("lombok")
public String toString() { ... }

// Or exclude the entire class
@ExcludeFromJacocoGeneratedReport
public class SchoolApplication { ... }
```

### Understanding the Coverage Report

The JaCoCo HTML report opens at the package level. Clicking into a package shows its classes. Clicking a class shows its source with colour-coded highlights. The key columns to read are the "Missed Branches" column (the most informative metric — tells you which decisions in your code were never tested for both outcomes) and the "Missed Lines" column (simpler but less meaningful).

A common mistake is treating 80% line coverage as the goal rather than the floor. Coverage metrics are a proxy for quality, not quality itself. A file can have 100% line coverage and still be undertested if its assertions are trivial. The mutation testing section below (Pitest) addresses this.

---

## 2. SonarQube / SonarCloud — Static Code Analysis at Scale

SonarQube analyses your code for bugs, security vulnerabilities, code smells, and design problems without running it. It is far more sophisticated than a style checker — it performs data flow analysis, understands how values propagate through your code, and detects patterns that reliably cause problems at runtime.

SonarCloud is the hosted, zero-infrastructure version of SonarQube that integrates directly with GitHub, GitLab, and Bitbucket. For most teams, SonarCloud is the practical choice because it requires no server setup.

### Core Concepts

SonarQube classifies issues into three types. A **Bug** is code that will cause incorrect behaviour at runtime — a confirmed defect. A **Vulnerability** is a security weakness that could be exploited — an SQL injection vector, a missing input sanitisation, or a weak cryptographic algorithm. A **Code Smell** is a maintainability issue — overly complex methods, duplicated blocks, unused variables, methods that are too long — that increases the cost of future changes without causing an immediate runtime failure.

The **Quality Gate** is a set of conditions that must all pass for SonarQube to mark a build as passing. You define your own Quality Gate (or use the default "Sonar Way") to require, for instance, that new code has no bugs, no critical vulnerabilities, and less than 3% code duplication. If any condition fails, SonarQube marks the analysis as failed and you can configure your CI pipeline to block the pull request.

### Maven Setup (SonarCloud)

```xml
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.11.0.3922</version>
</plugin>
```

```bash
# Run from CI — typically after mvn verify (so JaCoCo reports are already generated)
mvn sonar:sonar \
  -Dsonar.projectKey=my-org_school-app \
  -Dsonar.organization=my-org \
  -Dsonar.host.url=https://sonarcloud.io \
  -Dsonar.token=$SONAR_TOKEN \
  -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
```

### GitHub Actions Integration

```yaml
# .github/workflows/quality.yml
name: Build and Quality Check

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Sonar needs full git history for blame information

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build, Test, and Coverage
        run: mvn verify   # Runs tests + JaCoCo coverage

      - name: SonarCloud Analysis
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn sonar:sonar \
            -Dsonar.projectKey=my-org_school-app \
            -Dsonar.organization=my-org \
            -Dsonar.host.url=https://sonarcloud.io \
            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
```

### sonar-project.properties

```properties
# sonar-project.properties (placed at project root)
sonar.projectKey=com.example:school-app
sonar.projectName=School Application
sonar.projectVersion=1.0.0

# Source and test directories
sonar.sources=src/main/java
sonar.tests=src/test/java

# Point to JaCoCo XML for coverage data
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

# Exclude classes that shouldn't count against coverage or quality metrics
sonar.exclusions=**/config/**,**/generated/**,**/*Application.java

# Minimum duplication threshold before raising an issue
sonar.cpd.exclusions=**/dto/**,**/entity/**
```

### What SonarQube Catches That You'd Otherwise Miss

SonarQube's value is in the bugs and vulnerabilities it finds that are hard to spot in code review. A few concrete examples follow.

It detects **null dereference** patterns where a method returns null under certain conditions and the caller never checks for it — a `NullPointerException` waiting to be triggered in production. It flags **resource leaks** where a `Connection`, `InputStream`, or `Stream` is opened but not guaranteed to be closed (particularly in exception paths). It finds **SQL injection vulnerabilities** where a query is constructed by string concatenation using user input. It identifies **weak cryptography** — use of `MD5` or `SHA-1` for passwords, use of `Random` instead of `SecureRandom` for security-sensitive operations. It reports **duplicated blocks** that should be extracted into shared methods. It catches **empty catch blocks** that silently swallow exceptions. And it finds **methods with cyclomatic complexity above a threshold** (too many branches, too hard to test exhaustively).

---

## 3. Checkstyle — Enforcing Coding Conventions

Checkstyle validates your source code against a set of style rules — formatting, naming conventions, Javadoc requirements, import organisation, and structural rules. Unlike SonarQube, which analyses semantics, Checkstyle only looks at the surface form of the code. Its value is ensuring every developer on a team writes code that looks consistent — reducing cognitive friction during code review because reviewers can focus on logic rather than style.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.3.1</version>
    <configuration>
        <!-- Points to your Checkstyle rules file -->
        <configLocation>checkstyle.xml</configLocation>
        <!-- Fail the build if any violation is found -->
        <failsOnError>true</failsOnError>
        <violationSeverity>warning</violationSeverity>
        <!-- Also check test sources -->
        <includeTestSourceDirectory>true</includeTestSourceDirectory>
    </configuration>
    <executions>
        <execution>
            <id>checkstyle</id>
            <phase>validate</phase>  <!-- Run before compilation -->
            <goals><goal>check</goal></goals>
        </execution>
    </executions>
</plugin>
```

The `checkstyle.xml` file defines your rules. You can start from a well-known standard and customise it:

```xml
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC
    "-//Checkstyle//DTD Checkstyle Configuration 1.3//EN"
    "https://checkstyle.org/dtds/configuration_1_3.dtd">

<!-- Based on Google Java Style Guide — the most commonly used standard -->
<module name="Checker">

    <!-- File-level checks -->
    <module name="FileTabCharacter">
        <!-- No tabs — use spaces only -->
        <property name="eachLine" value="true"/>
    </module>
    <module name="NewlineAtEndOfFile"/>

    <module name="TreeWalker">

        <!-- ── Naming Conventions ── -->
        <module name="TypeName">
            <!-- Classes and interfaces: PascalCase -->
            <property name="format" value="^[A-Z][a-zA-Z0-9]*$"/>
        </module>
        <module name="MethodName">
            <!-- Methods: camelCase -->
            <property name="format" value="^[a-z][a-zA-Z0-9]*$"/>
        </module>
        <module name="ConstantName">
            <!-- Constants: UPPER_SNAKE_CASE -->
            <property name="format" value="^[A-Z][A-Z0-9]*(_[A-Z0-9]+)*$"/>
        </module>
        <module name="LocalVariableName">
            <property name="format" value="^[a-z][a-zA-Z0-9]*$"/>
        </module>

        <!-- ── Imports ── -->
        <module name="AvoidStarImport"/>           <!-- No import com.example.* -->
        <module name="UnusedImports"/>             <!-- No unused imports -->
        <module name="RedundantImport"/>           <!-- No duplicate imports -->

        <!-- ── Whitespace and Length ── -->
        <module name="LineLength">
            <property name="max" value="120"/>     <!-- Max 120 chars per line -->
        </module>
        <module name="WhitespaceAround">
            <!-- Spaces around operators: a = b, not a=b -->
        </module>
        <module name="EmptyLineSeparator">
            <!-- Blank line between methods, fields, constructors -->
            <property name="allowNoEmptyLineBetweenFields" value="true"/>
        </module>

        <!-- ── Code Structure ── -->
        <module name="MethodLength">
            <!-- Methods longer than 50 lines are a maintainability smell -->
            <property name="max" value="50"/>
        </module>
        <module name="ParameterNumber">
            <!-- More than 7 parameters is a design problem -->
            <property name="max" value="7"/>
        </module>
        <module name="CyclomaticComplexity">
            <!-- More than 10 branches makes a method hard to test and understand -->
            <property name="max" value="10"/>
        </module>
        <module name="NeedBraces"/>               <!-- Always use braces, even for single-line if -->

        <!-- ── Javadoc ── -->
        <module name="JavadocMethod">
            <!-- Public methods must have Javadoc -->
            <property name="scope" value="public"/>
            <property name="allowMissingParamTags"  value="false"/>
            <property name="allowMissingReturnTag"  value="false"/>
        </module>

        <!-- ── Correctness ── -->
        <module name="MissingSwitchDefault"/>     <!-- switch must have default case -->
        <module name="FallThrough"/>              <!-- No unintentional fall-through in switch -->
        <module name="EmptyCatchBlock">           <!-- Empty catch blocks must have a comment -->
            <property name="exceptionVariableName" value="expected|ignored"/>
        </module>
        <module name="StringLiteralEquality"/>    <!-- Use .equals(), not == for Strings -->
        <module name="EqualsHashCode"/>           <!-- If you override equals(), also override hashCode() -->
    </module>

</module>
```

Teams commonly use well-established pre-built configurations rather than writing their own from scratch. The two most popular are the **Google Java Style** ruleset (`google_checks.xml`, bundled with the Checkstyle plugin) and the **Sun/Oracle style** ruleset (`sun_checks.xml`). Both can be referenced directly without a local file:

```xml
<configLocation>google_checks.xml</configLocation>
```

---

## 4. PMD — Detecting Common Programming Mistakes

PMD performs static analysis focused on common programming mistakes: overly complex code, dead code (variables assigned but never read), empty blocks, unnecessary object creation, suboptimal collection usage, and many other patterns. Unlike SpotBugs, which analyses compiled bytecode, PMD analyses source code, which means it catches a different and somewhat complementary set of issues.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-pmd-plugin</artifactId>
    <version>3.21.2</version>
    <configuration>
        <rulesets>
            <!-- PMD ships with categorised rule sets you mix and match -->
            <ruleset>/rulesets/java/quickstart.xml</ruleset> <!-- Sensible defaults -->
        </rulesets>
        <failOnViolation>true</failOnViolation>
        <printFailingErrors>true</printFailingErrors>
        <minimumTokens>100</minimumTokens> <!-- Copy-paste detection: flag 100+ identical tokens -->
        <targetJdk>21</targetJdk>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
                <goal>cpd-check</goal> <!-- Copy-Paste Detector -->
            </goals>
        </execution>
    </executions>
</plugin>
```

PMD's rule categories are organised into seven areas. The **Best Practices** category catches issues like using `System.out.println` instead of a logger, using `==` to compare strings, or closing resources in the wrong order. The **Code Style** category flags things like unnecessary parentheses, confusing ternary expressions, and loose coupling violations. The **Design** category catches deep coupling, god classes, and methods that are too long or too complex. The **Error Prone** category is the most important for catching bugs — it flags things like assigning a field in a constructor whose parameter shadows the field name (a very common copy-paste mistake), calling `equals()` on objects of unrelated types (which always returns false), and comparing the result of `String.length()` with `<= 0` instead of `== 0`. The **Multithreading** category catches unsafe published fields in non-thread-safe classes. The **Performance** category flags inefficient string concatenation inside loops (should use `StringBuilder`), unnecessary use of boxing operations, and similar micro-optimisation patterns. The **Security** category flags hardcoded IVs, predictable seed values, and similar security-sensitive mistakes.

A custom PMD rule file lets you include exactly the rules you want:

```xml
<!-- pmd-rules.xml -->
<?xml version="1.0"?>
<ruleset name="School App Rules"
         xmlns="http://pmd.sourceforge.net/ruleset/2.0.0">

    <description>Custom PMD rules for the school application</description>

    <!-- Include entire categories selectively -->
    <rule ref="category/java/errorprone.xml">
        <!-- Exclude rules that don't fit your project -->
        <exclude name="BeanMembersShouldSerialize"/>
        <exclude name="DataflowAnomalyAnalysis"/> <!-- Too many false positives -->
    </rule>

    <rule ref="category/java/bestpractices.xml">
        <exclude name="GuardLogStatement"/> <!-- Handled by SLF4J parameterisation -->
    </rule>

    <rule ref="category/java/design.xml/CyclomaticComplexity">
        <!-- Customise the threshold for cyclomatic complexity -->
        <properties>
            <property name="methodReportLevel" value="15"/>
        </properties>
    </rule>

    <rule ref="category/java/performance.xml/InefficientStringBuffering"/>
    <rule ref="category/java/multithreading.xml/DoubleCheckedLocking"/>
    <rule ref="category/java/security.xml"/>

</ruleset>
```

```xml
<!-- Reference the custom file in the plugin config -->
<configuration>
    <rulesets>
        <ruleset>pmd-rules.xml</ruleset>
    </rulesets>
</configuration>
```

### PMD's Copy-Paste Detector (CPD)

CPD scans your source files for blocks of code that are structurally similar (based on token count, not text matching). Duplicated blocks of more than ~10–15 lines are a strong signal that a method or class should be extracted. CPD runs as the `cpd-check` goal and fails the build if duplicated blocks above the configured token threshold are found.

---

## 5. SpotBugs — Bytecode-Level Bug Pattern Detection

SpotBugs (the successor to FindBugs) analyses your compiled `.class` files using hundreds of pre-defined bug patterns. Because it works at the bytecode level rather than the source level, it can detect issues that source-level tools miss — particularly those that involve how the JVM handles certain operations differently than a developer might expect.

```xml
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.3.1</version>
    <configuration>
        <effort>Max</effort>                  <!-- Max effort: most thorough analysis -->
        <threshold>Low</threshold>            <!-- Report even low-priority issues -->
        <failOnError>true</failOnError>
        <excludeFilterFile>spotbugs-exclude.xml</excludeFilterFile>
    </configuration>
    <executions>
        <execution>
            <goals><goal>check</goal></goals>
        </execution>
    </executions>
</plugin>
```

SpotBugs groups its detectors into named categories, each corresponding to a family of bugs. The most impactful ones to know are described below.

The **Null Pointer Dereference (NP)** category catches places where SpotBugs's data-flow analysis determines a value could be null at the point it is dereferenced — for example, calling a method on the return value of another method that is annotated with `@Nullable` without a null check. The **Dodgy Code (DM)** category flags calls to deprecated APIs, use of `Random` where `SecureRandom` is appropriate, comparison of `Integer` objects with `==`, and other subtle correctness hazards. The **Malicious Code Vulnerability (MS)** category finds public static mutable fields, which can be modified by any class in the same package — a common accidental API exposure. The **Multithreading (MT)** category detects synchronisation mistakes: calling `wait()` without holding the object's lock, double-checked locking done incorrectly, and inconsistent synchronisation on a field. The **Performance (DM)** category finds unnecessary boxing/unboxing, inefficient use of `String.valueOf`, and calls to `new Integer(n)` instead of `Integer.valueOf(n)`. The **Bad Practice (BP)** category catches things like overriding `equals()` without overriding `hashCode()`, `catch(Exception e)` with an empty body, and `finalize()` methods that don't call `super.finalize()`.

```xml
<!-- spotbugs-exclude.xml — suppress specific warnings for legitimate reasons -->
<FindBugsFilter>

    <!-- Suppress false positives in test code -->
    <Match>
        <Class name="~.*Test"/>
        <Bug category="STYLE"/>
    </Match>

    <!-- Suppress a specific bug type in generated code -->
    <Match>
        <Package name="com.example.generated"/>
        <Bug pattern="DM_NUMBER_CTOR"/>
    </Match>

    <!-- Suppress a known false positive on a specific line -->
    <Match>
        <Class name="com.example.config.SecurityConfig"/>
        <Bug pattern="HARD_CODE_PASSWORD"/>
        <!-- This is actually a test configuration property, not a real password -->
    </Match>

</FindBugsFilter>
```

You can also suppress individual SpotBugs warnings in source code using the `@SuppressFBWarnings` annotation — but use this sparingly. Every suppression is a conscious decision to accept a risk, and should be accompanied by a comment explaining why.

```java
import edu.umd.cs.findbugs.annotations.SuppressFBWarnings;

public class CourseCache {

    // SpotBugs flags this as a mutable static field (MS_MUTABLE_COLLECTION)
    // Suppressing because this map is intentionally replaced atomically, not mutated
    @SuppressFBWarnings(value = "MS_MUTABLE_COLLECTION",
                        justification = "Replaced atomically via CAS; not incrementally mutated")
    private static volatile Map<Long, Course> cache = new HashMap<>();
}
```

---

## 6. Pitest — Mutation Testing

Pitest is the most sophisticated tool in this chapter because it measures not just whether your code is executed by tests, but whether your tests are actually capable of detecting errors. It does this by **mutation testing**: systematically introducing small artificial bugs ("mutations") into your production code, running your test suite against each mutated version, and checking whether any test fails.

A mutation that no test catches is called a **survived mutant** — it is concrete evidence that your test suite has a blind spot. If Pitest swaps `>` for `>=` in a comparison and all your tests still pass, you do not have a test that checks the exact boundary condition.

Understanding mutation testing is the key to appreciating why high line coverage is not the same as high quality tests. A test that calls a method without asserting on its return value will give you 100% line coverage and 0% mutation score.

### Maven Setup

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.15.3</version>
    <dependencies>
        <!-- JUnit 5 support (requires this additional dependency) -->
        <dependency>
            <groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId>
            <version>1.2.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <!-- Classes to mutate — should match your production code, not tests or config -->
        <targetClasses>
            <param>com.example.service.*</param>
            <param>com.example.domain.*</param>
        </targetClasses>
        <!-- Tests that Pitest will run against each mutant -->
        <targetTests>
            <param>com.example.*Test</param>
        </targetTests>
        <!-- Minimum mutation score to pass (0–100) — build fails below this -->
        <mutationThreshold>75</mutationThreshold>
        <!-- Which mutation operators to apply -->
        <mutators>
            <mutator>STRONGER</mutator> <!-- Pre-defined strong set: conditionals, arithmetic, returns -->
        </mutators>
        <!-- Number of threads — Pitest runs many JVM instances in parallel -->
        <threads>4</threads>
        <!-- Verbose HTML report — shows each mutant, its location, and whether it survived -->
        <outputFormats>
            <outputFormat>HTML</outputFormat>
            <outputFormat>XML</outputFormat>
        </outputFormats>
        <!-- Avoid mutating generated code -->
        <excludedClasses>
            <param>com.example.generated.*</param>
            <param>*.*Application</param>
        </excludedClasses>
    </configuration>
</plugin>
```

```bash
# Run mutation testing (separate from normal test run — it's slow)
mvn org.pitest:pitest-maven:mutationCoverage

# Report lands at: target/pit-reports/index.html
```

### Understanding Pitest's Mutation Operators

Pitest uses different mutation operators, each modelling a different category of developer mistake. **Conditional Boundary** mutations change `>` to `>=`, `<` to `<=`, and so on — testing whether your tests verify exact boundary conditions. **Negate Conditionals** flips `if (x > 0)` to `if (!(x > 0))` — verifying that the decision matters. **Return Values** mutations change `return true` to `return false`, `return n` to `return n + 1`, `return collection` to `return emptyList()` — testing whether you assert on what the method returns. **Void Method Calls** mutations delete calls to void methods — verifying that those calls have observable side effects that your tests check. **Constructor Calls** mutations replace `new SomeClass(args)` with the result of calling the constructor with different or no arguments.

```java
// Example class to illustrate mutation testing
public class DiscountService {

    public BigDecimal calculateDiscount(Order order) {
        // Pitest will mutate this boundary condition:
        // ORIGINAL: order.getTotal().compareTo(THRESHOLD) >= 0
        // MUTANT 1: order.getTotal().compareTo(THRESHOLD) > 0   (changes >= to >)
        // MUTANT 2: order.getTotal().compareTo(THRESHOLD) <= 0  (negates)
        if (order.getTotal().compareTo(new BigDecimal("500.00")) >= 0) {
            // Pitest will mutate this return value:
            // MUTANT 3: return order.getTotal().multiply(new BigDecimal("0.10"))
            // MUTANT 4: return BigDecimal.ZERO
            return order.getTotal().multiply(new BigDecimal("0.20"));
        }
        return BigDecimal.ZERO;
    }
}

// This test suite KILLS all four mutants above:
class DiscountServiceMutationTest {

    private DiscountService service = new DiscountService();

    @Test
    void exactBoundary_getsDiscount() {
        // This test kills MUTANT 1 — exactly $500 should get the discount
        // If Pitest changes >= to >, this test fails
        Order order = orderWith(new BigDecimal("500.00"));
        assertThat(service.calculateDiscount(order))
            .isEqualByComparingTo("100.00");
    }

    @Test
    void justBelowBoundary_getsNoDiscount() {
        // This test kills MUTANT 2 — $499.99 should NOT get the discount
        Order order = orderWith(new BigDecimal("499.99"));
        assertThat(service.calculateDiscount(order))
            .isEqualByComparingTo("0.00");
    }

    @Test
    void largeOrder_getsTwentyPercentDiscount() {
        // This test kills MUTANTS 3 and 4 — asserts the exact discount amount
        Order order = orderWith(new BigDecimal("1000.00"));
        assertThat(service.calculateDiscount(order))
            .isEqualByComparingTo("200.00"); // 20% of $1000
    }
}
```

### Reading the Pitest HTML Report

The Pitest report shows every class with a **mutation score** (killed mutants / total mutants as a percentage). Clicking into a class shows the source code with coloured markers: green means the mutant was killed (a test caught it), red means the mutant survived (no test caught it — a gap in your coverage). The report tells you exactly which line was mutated, what the mutation was, and which test killed it (or that none did). Use the red markers to identify exactly where to add more targeted assertions.

---

## 🔗 Integrating All Tools Into the CI Pipeline

The most effective setup runs all tools in a single `mvn verify` command and publishes results to SonarCloud:

```yaml
# .github/workflows/ci.yml
name: CI Quality Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: Run Full Quality Pipeline
        # This single command triggers (in order):
        # 1. Checkstyle (phase: validate)       — style enforcement
        # 2. Compile                             — source compilation
        # 3. PMD (phase: verify, before tests)  — static analysis
        # 4. Test + JaCoCo (phase: test)         — unit & integration tests with coverage
        # 5. SpotBugs (phase: verify)            — bytecode analysis
        # 6. JaCoCo check (phase: verify)        — coverage threshold enforcement
        run: mvn verify

      - name: SonarCloud Analysis
        if: always()   # Run even if tests failed — to see coverage on SonarCloud
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: |
          mvn sonar:sonar \
            -Dsonar.projectKey=${{ vars.SONAR_PROJECT_KEY }} \
            -Dsonar.organization=${{ vars.SONAR_ORG }} \
            -Dsonar.host.url=https://sonarcloud.io \
            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml

      - name: Mutation Testing (weekly)
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: mvn org.pitest:pitest-maven:mutationCoverage

      - name: Upload Reports
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: quality-reports
          path: |
            target/site/jacoco/
            target/pit-reports/
            target/checkstyle-result.xml
            target/pmd.xml
            target/spotbugsXml.xml
```

Running Pitest on every commit is impractical — it can take minutes to hours on large codebases because it runs your full test suite once per mutation. The standard approach is to run Pitest on every merge to `main` (as a quality measurement) and use the threshold to prevent regressions, rather than blocking every pull request with it.

---

## ✅ Best Practices & Important Points

**Introduce quality gates early and tighten them incrementally.** If you add code coverage requirements to an existing codebase, setting the threshold to 80% on day one will immediately fail hundreds of builds. Start with the current coverage level as the floor, establish a culture of not decreasing it, and raise the threshold by 5% every quarter. Greenfield projects should start at 80% from the first commit.

**Configure all tools to fail the build in CI.** A quality tool that reports findings without failing the build is just a suggestion. Developers ignore suggestions, especially under deadline pressure. The only reliable enforcement mechanism is making the build fail. Block merging a PR whose quality gates don't pass.

**Tune your rules over the first few sprints.** Out-of-the-box configurations from Checkstyle, PMD, and SpotBugs will generate false positives specific to your project's idioms. The initial weeks should include a process for the team to evaluate each category of finding, decide whether it is a genuine concern for the codebase, and either fix it or add a justified exclusion. After that calibration period, findings from new code should be treated as regressions.

**Treat SpotBugs high-priority findings with extreme seriousness.** Unlike Checkstyle (which catches style issues) or PMD (which catches code smells), SpotBugs high-priority findings are patterns that reliably cause production bugs. A SpotBugs NP (null pointer) finding is not a style suggestion — it is a bug report backed by static analysis. Fix it, don't suppress it, unless you can explain concretely why it is a false positive.

**Run Pitest on your most business-critical code first.** Mutation testing the entire codebase from day one is expensive. Identify the 20% of your code that handles the most important business rules — pricing logic, authentication, financial calculations, data validation — and configure Pitest's `targetClasses` to cover only that. You will get the highest return on investment from improving mutation scores in the code where a bug costs the most.

---

## 🔑 Key Summary

```
JaCoCo              → Instruments bytecode to measure line, branch, and instruction coverage
Line coverage       → % of executable lines run; weakest metric — use as a floor, not a goal
Branch coverage     → Both true/false of every decision covered; significantly more meaningful
<jacoco:check>      → Fails the build if coverage drops below configured thresholds
SonarQube           → Static analysis; finds bugs, vulnerabilities, code smells in source code
Quality Gate        → Pass/fail criteria configured in SonarQube; block PRs that don't meet them
sonar:sonar         → Maven goal that sends analysis results to SonarQube/SonarCloud
Checkstyle          → Validates source code formatting and naming against a rule XML file
google_checks.xml   → Ready-made Google Java Style rules; good default starting point
PMD                 → Source-level static analysis; catches programming mistakes and dead code
CPD (PMD)           → Copy-Paste Detector; flags duplicated blocks above a token threshold
SpotBugs            → Bytecode-level bug pattern detection; finds NullPointerException risks,
                      synchronisation mistakes, resource leaks, and security vulnerabilities
@SuppressFBWarnings → Suppress a specific SpotBugs warning with a mandatory justification comment
Pitest              → Mutation testing: introduces artificial bugs and checks if tests catch them
Mutant              → An artificially introduced change to production code
Killed mutant       → A mutant that caused at least one test to fail — good; your tests are effective
Survived mutant     → A mutant no test caught — bad; reveals a gap in your test assertions
Mutation score      → Killed / Total × 100; target 75%+ for business-critical code
mutationThreshold   → Pitest build threshold; fails the build if mutation score drops below it
```
