# 📗 12.1 — JUnit 5 (Jupiter): The Testing Framework

JUnit 5 is the foundation of the entire Java testing ecosystem. Every other tool in this phase — Mockito, AssertJ, Testcontainers — plugs into it. Before writing a single assertion, you need to understand what JUnit 5 is, how it is structured, and why it was redesigned from JUnit 4 the way it was.

---

## 🧠 What JUnit 5 Actually Is

JUnit 5 is not a single library. It is an architecture composed of three distinct sub-projects, each with a separate responsibility.

**JUnit Platform** is the foundation. It defines the API for launching test frameworks on the JVM. Crucially, it is what your IDE (IntelliJ, Eclipse) and build tool (Maven, Gradle) speak to when they discover and run tests. You never interact with it directly — it works behind the scenes.

**JUnit Jupiter** is the part you write against. It provides the `@Test` annotation, lifecycle hooks, assertions, extensions, and everything else in this chapter. When people say "JUnit 5", they really mean Jupiter.

**JUnit Vintage** is a compatibility bridge that lets you run JUnit 4 tests (and JUnit 3 tests) on the JUnit 5 platform. It exists so teams can migrate incrementally rather than rewriting everything at once.

This three-module design was the key architectural decision in JUnit 5. Previously, test frameworks, IDEs, and build tools were all coupled to JUnit's internal APIs. JUnit 5 separated the stable public API (Platform) from the authoring API (Jupiter), allowing both to evolve independently.

---

## 📦 Maven Setup

```xml
<!-- spring-boot-starter-test already includes JUnit 5. For non-Spring projects: -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.2</version>
    <scope>test</scope>
</dependency>

<!-- Maven Surefire plugin must be at 2.22+ to run JUnit 5 tests -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.5</version>
</plugin>
```

---

## ✍️ Writing Your First Test — Anatomy of a Test Class

```java
import org.junit.jupiter.api.*;
import static org.junit.jupiter.api.Assertions.*;

// Test classes do NOT need to be public in JUnit 5 — package-private is preferred
// JUnit 5 uses reflection to discover tests, so the access modifier doesn't matter
class CalculatorTest {

    // The subject under test — the object we are testing
    private Calculator calculator;

    // @BeforeEach: runs before EVERY individual test method
    // Use it to reset state so tests don't bleed into each other
    @BeforeEach
    void setUp() {
        calculator = new Calculator(); // Fresh instance for every test
    }

    // @AfterEach: runs after EVERY individual test method
    // Use it for cleanup (closing resources, resetting static state)
    @AfterEach
    void tearDown() {
        // For a simple calculator we don't need cleanup,
        // but the hook is here when we do
    }

    // @BeforeAll: runs ONCE before all tests in this class
    // Must be static (unless using @TestInstance(Lifecycle.PER_CLASS))
    @BeforeAll
    static void setUpClass() {
        System.out.println("Setting up expensive shared resources (e.g., a test database)");
    }

    // @AfterAll: runs ONCE after all tests in this class
    @AfterAll
    static void tearDownClass() {
        System.out.println("Releasing shared resources");
    }

    // @Test marks a method as a test case
    // @DisplayName gives the test a human-readable name shown in IDE and reports
    @Test
    @DisplayName("Adding two positive numbers returns their sum")
    void add_twoPositiveNumbers_returnsSum() {
        int result = calculator.add(3, 4);
        assertEquals(7, result); // (expected, actual)
    }

    // @Disabled skips this test — always add a reason explaining WHY
    @Test
    @Disabled("Bug #4521: divide returns wrong result for large numbers — fix pending")
    void divide_largeNumbers_returnsCorrectQuotient() {
        assertEquals(1_000_000, calculator.divide(2_000_000, 2));
    }
}
```

---

## 💡 Assertions — The Core of Every Test

JUnit 5 ships with a rich built-in assertion library in `org.junit.jupiter.api.Assertions`. The key difference from JUnit 4 is that argument order is consistent: the expected value always comes first, the actual value second, and an optional failure message comes last.

```java
class AssertionDemoTest {

    @Test
    void demonstrateBuiltInAssertions() {
        // ── Equality ──
        assertEquals(42, calculator.compute(6, 7), "6 × 7 should equal 42");
        assertNotEquals(0, calculator.compute(6, 7));

        // ── Boolean ──
        assertTrue(student.isEnrolled(), "Student should be enrolled after registration");
        assertFalse(course.isFull(), "New course should not be full");

        // ── Null checks ──
        assertNotNull(studentService.findById(1L), "Service should return a non-null result");
        assertNull(studentService.findById(999L),  "Missing student should return null");

        // ── Same instance ──
        Student a = studentService.findById(1L);
        Student b = studentService.findById(1L); // Same bean from cache
        assertSame(a, b, "Cached lookups should return the identical object");

        // ── Array equality (deep comparison) ──
        int[] expected = {1, 2, 3, 4};
        int[] actual   = calculator.range(1, 4);
        assertArrayEquals(expected, actual, "Range should produce sequential integers");

        // ── Exception assertion — the modern, clean way ──
        // assertThrows returns the exception so you can inspect its message
        IllegalArgumentException ex = assertThrows(
            IllegalArgumentException.class,
            () -> calculator.divide(10, 0),
            "Dividing by zero should throw IllegalArgumentException"
        );
        assertTrue(ex.getMessage().contains("zero"),
            "Exception message should mention 'zero'");

        // ── No exception thrown ──
        assertDoesNotThrow(() -> calculator.add(Integer.MAX_VALUE, 0),
            "Adding zero should never throw");
    }

    // assertAll: groups multiple assertions — ALL run even if one fails
    // Without assertAll, the first failing assertion stops the test immediately,
    // hiding all subsequent failures. assertAll gives you the complete picture.
    @Test
    void studentDTO_hasCorrectFields() {
        StudentDTO dto = studentService.buildDTO(1L);

        assertAll("StudentDTO field validation",
            () -> assertNotNull(dto,                             "DTO must not be null"),
            () -> assertEquals(1L,          dto.id(),           "ID should match"),
            () -> assertEquals("Alice",     dto.name(),         "Name should match"),
            () -> assertEquals("alice@ex.com", dto.email(),     "Email should match"),
            () -> assertTrue(dto.score() > 0,                   "Score must be positive"),
            () -> assertNotNull(dto.enrolledAt(),               "Enrollment date required")
        );
        // If ID is wrong AND email is wrong, assertAll reports BOTH failures at once
        // Without assertAll you would fix ID, re-run, discover email is wrong, fix it, re-run...
    }

    // assertTimeout: the test must complete within a time limit
    @Test
    void findAll_shouldCompleteQuickly() {
        assertTimeout(Duration.ofMillis(500), () -> {
            List<Student> results = studentService.findAll();
            assertFalse(results.isEmpty());
        });
    }
}
```

---

## 🔄 Parameterized Tests — One Test, Many Inputs

Parameterized tests are one of JUnit 5's most powerful features. They let you write the test logic once and run it against a table of inputs and expected outputs. This is far superior to writing nearly-identical test methods or using loops inside a single test (which only reports one failure).

### @ValueSource — Simple Single-Value Parameters

```java
@ParameterizedTest(name = "isValidEmail(\"{0}\") should return true")
@ValueSource(strings = {
    "user@example.com",
    "user.name+tag@sub.domain.co.uk",
    "very.common@example.org"
})
void isValidEmail_withValidAddresses_returnsTrue(String email) {
    // This test method runs 3 times — once for each string in @ValueSource
    assertTrue(emailValidator.isValid(email));
}

@ParameterizedTest
@ValueSource(ints = {-100, -1, 0})
void calculateGrade_withNonPositiveScore_throwsException(int score) {
    assertThrows(IllegalArgumentException.class, () -> gradeService.calculate(score));
}
```

### @CsvSource — Multiple Parameters Per Row

```java
// Each string is one test case; commas separate the arguments
@ParameterizedTest(name = "score {0} → grade ''{1}''")
@CsvSource({
    "95,  A",
    "85,  B",
    "75,  C",
    "65,  D",
    "55,  F",
    "0,   F",
    "100, A"
})
void calculateGrade_returnsCorrectLetterGrade(double score, String expectedGrade) {
    assertEquals(expectedGrade, gradeService.calculate(score));
}

// @CsvFileSource loads test data from a CSV file in src/test/resources
@ParameterizedTest
@CsvFileSource(resources = "/test-data/grade-scenarios.csv", numLinesToSkip = 1)
void calculateGrade_fromCsvFile(double score, String expectedGrade, String description) {
    assertEquals(expectedGrade, gradeService.calculate(score), description);
}
```

### @MethodSource — Complex Objects as Parameters

```java
// @MethodSource points to a static factory method that provides test arguments
@ParameterizedTest(name = "{index}: {2}")
@MethodSource("provideStudentValidationCases")
void validateStudent_withVariousInputs(CreateStudentRequest request,
                                        boolean shouldPass,
                                        String description) {
    if (shouldPass) {
        assertDoesNotThrow(() -> studentValidator.validate(request), description);
    } else {
        assertThrows(ValidationException.class,
            () -> studentValidator.validate(request), description);
    }
}

// The factory method must be static, return Stream<Arguments>
static Stream<Arguments> provideStudentValidationCases() {
    return Stream.of(
        Arguments.of(
            new CreateStudentRequest("Alice", "alice@example.com", 20),
            true,
            "Valid student — all fields present and well-formed"
        ),
        Arguments.of(
            new CreateStudentRequest("", "alice@example.com", 20),
            false,
            "Empty name should fail validation"
        ),
        Arguments.of(
            new CreateStudentRequest("Alice", "not-an-email", 20),
            false,
            "Malformed email should fail validation"
        ),
        Arguments.of(
            new CreateStudentRequest("Alice", "alice@example.com", -1),
            false,
            "Negative age should fail validation"
        )
    );
}
```

### @EnumSource — Testing Against Enum Values

```java
// Runs once for each value in the enum — great for exhaustive enum testing
@ParameterizedTest
@EnumSource(CourseStatus.class)
void getStatusLabel_neverReturnsNull(CourseStatus status) {
    assertNotNull(courseService.getStatusLabel(status),
        "Every CourseStatus must have a non-null label");
}

// Include only specific enum values
@ParameterizedTest
@EnumSource(value = CourseStatus.class, names = {"OPEN", "WAITLISTED"})
void enroll_withEnrollableStatus_succeeds(CourseStatus status) {
    course.setStatus(status);
    assertDoesNotThrow(() -> enrollmentService.enroll(student, course));
}

// Exclude specific enum values
@ParameterizedTest
@EnumSource(value = CourseStatus.class,
            mode = EnumSource.Mode.EXCLUDE,
            names = {"OPEN", "WAITLISTED"})
void enroll_withNonEnrollableStatus_throws(CourseStatus status) {
    course.setStatus(status);
    assertThrows(CourseNotOpenException.class,
        () -> enrollmentService.enroll(student, course));
}
```

---

## 🔁 @RepeatedTest — Stability and Load Tests

```java
// Runs the test exactly 10 times — useful for tests involving randomness,
// concurrency, or any non-deterministic behaviour
@RepeatedTest(value = 10, name = "Run {currentRepetition} of {totalRepetitions}")
void shuffle_shouldProduceDifferentOrdersOverTime(RepetitionInfo info) {
    List<Integer> list = new ArrayList<>(List.of(1, 2, 3, 4, 5));
    Collections.shuffle(list);
    // Not every shuffle will differ, but over 10 runs at least one should
    System.out.printf("Run %d: %s%n", info.getCurrentRepetition(), list);
}

// RepetitionInfo lets you change behaviour based on which repetition you're on
@RepeatedTest(5)
void retryableOperation_succeedsWithinMaxAttempts(RepetitionInfo info) {
    // Simulate a flaky external service that might fail a few times
    if (info.getCurrentRepetition() < 3) {
        // First two runs: verify it handles failure gracefully
        when(externalService.call()).thenThrow(new TimeoutException());
        assertThrows(RetryExhaustedException.class, () -> retryService.call());
    } else {
        // Remaining runs: verify it succeeds when service recovers
        when(externalService.call()).thenReturn("OK");
        assertEquals("OK", retryService.call());
    }
}
```

---

## 🏛️ @Nested — Organising Tests with Context

`@Nested` classes let you group related tests that share context, dramatically improving readability in large test classes. Think of the outer class as the "subject" and each nested class as a "scenario" for that subject.

```java
@DisplayName("StudentService")
class StudentServiceTest {

    private StudentService studentService;
    private StudentRepository studentRepository;

    @BeforeEach
    void setUp() {
        studentRepository = mock(StudentRepository.class);
        studentService    = new StudentService(studentRepository);
    }

    @Nested
    @DisplayName("when finding a student by ID")
    class FindById {

        @Test
        @DisplayName("returns the student when they exist")
        void returnsStudentWhenFound() {
            when(studentRepository.findById(1L))
                .thenReturn(Optional.of(new Student(1L, "Alice")));

            Optional<Student> result = studentService.findById(1L);
            assertTrue(result.isPresent());
            assertEquals("Alice", result.get().getName());
        }

        @Test
        @DisplayName("returns empty Optional when student does not exist")
        void returnsEmptyWhenNotFound() {
            when(studentRepository.findById(999L)).thenReturn(Optional.empty());

            Optional<Student> result = studentService.findById(999L);
            assertTrue(result.isEmpty());
        }
    }

    @Nested
    @DisplayName("when creating a student")
    class CreateStudent {

        @BeforeEach
        void setUpCreateScenario() {
            // This @BeforeEach runs IN ADDITION to the outer @BeforeEach
            // Outer setUp() runs first, then this one
            when(studentRepository.existsByEmail(anyString())).thenReturn(false);
        }

        @Test
        @DisplayName("saves and returns the new student when data is valid")
        void savesStudentWhenValid() {
            CreateStudentRequest req = new CreateStudentRequest("Bob", "bob@example.com");
            when(studentRepository.save(any())).thenAnswer(inv -> {
                Student s = inv.getArgument(0);
                s.setId(42L);
                return s;
            });

            Student result = studentService.create(req);
            assertNotNull(result.getId());
            assertEquals("Bob", result.getName());
        }

        @Test
        @DisplayName("throws DuplicateEmailException when email already exists")
        void throwsWhenEmailTaken() {
            when(studentRepository.existsByEmail("alice@example.com")).thenReturn(true);
            CreateStudentRequest req = new CreateStudentRequest("Alice", "alice@example.com");

            assertThrows(DuplicateEmailException.class, () -> studentService.create(req));
        }
    }
}
```

The output in your IDE reads like a document: "StudentService > when finding a student by ID > returns the student when they exist". This is vastly more navigable than a flat list of disconnected method names.

---

## 🏷️ @Tag — Grouping and Filtering Tests

Tags let you categorize tests by purpose or speed so you can run subsets from the command line or CI pipeline.

```java
@Tag("unit")         // Fast, no I/O, no Spring context
@Tag("student")      // Belongs to the student feature area
class StudentServiceUnitTest { ... }

@Tag("integration")  // Slower, requires DB/Spring context
@Tag("student")
class StudentRepositoryIntegrationTest { ... }

@Tag("slow")
@Tag("integration")
class FullSystemTest { ... }
```

```xml
<!-- Maven Surefire: run only unit tests (excludes integration tests) -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <!-- Run tests tagged "unit" but NOT "slow" -->
        <groups>unit</groups>
        <excludedGroups>slow</excludedGroups>
    </configuration>
</plugin>
```

This lets your fast CI pipeline on every push run only `unit` tests (milliseconds), while a nightly or pre-merge pipeline runs `unit` and `integration` tags together.

---

## 📂 @TempDir — Safe Temporary File Testing

```java
class FileProcessorTest {

    // @TempDir injects a temporary directory that JUnit creates before the test
    // and automatically deletes (with all contents) after the test
    @TempDir
    Path tempDir;

    @Test
    void processFile_shouldCreateOutputFile() throws IOException {
        // Create an input file in the temp directory
        Path input = tempDir.resolve("students.csv");
        Files.writeString(input,
            "Alice,alice@example.com,95\n" +
            "Bob,bob@example.com,82\n"
        );

        Path outputDir = tempDir.resolve("output");
        Files.createDirectory(outputDir);

        // Run the processor
        fileProcessor.process(input, outputDir);

        // Verify the output was created
        Path outputFile = outputDir.resolve("students-processed.json");
        assertTrue(Files.exists(outputFile), "Output file should be created");

        String content = Files.readString(outputFile);
        assertTrue(content.contains("Alice"));
        assertTrue(content.contains("alice@example.com"));
    }
    // JUnit deletes tempDir and everything inside it after this test completes
    // No manual cleanup needed, no leftover files between test runs
}
```

---

## 🔌 @ExtendWith — The Extension Model

JUnit 5 replaced JUnit 4's `@RunWith` with a far more flexible `@ExtendWith` mechanism. Extensions are composable (you can stack multiple) and can hook into every phase of the test lifecycle. Every major integration — Mockito, Spring, Testcontainers — ships as a JUnit 5 extension.

```java
// Using the Mockito extension (covered in full in 02_Mockito.md)
@ExtendWith(MockitoExtension.class)
class OrderServiceTest { ... }

// Stacking multiple extensions
@ExtendWith(MockitoExtension.class)
@ExtendWith(SpringExtension.class)      // Or just use @SpringBootTest which includes this
class OrderIntegrationTest { ... }

// Writing a custom extension — hooks into any lifecycle phase
public class TimingExtension implements BeforeTestExecutionCallback,
                                         AfterTestExecutionCallback {

    private static final String START_TIME_KEY = "startTime";

    @Override
    public void beforeTestExecution(ExtensionContext context) {
        // Store the start time in JUnit's extension store
        context.getStore(ExtensionContext.Namespace.create(getClass(), context.getRequiredTestMethod()))
               .put(START_TIME_KEY, System.currentTimeMillis());
    }

    @Override
    public void afterTestExecution(ExtensionContext context) {
        long startTime = context.getStore(
            ExtensionContext.Namespace.create(getClass(), context.getRequiredTestMethod()))
            .get(START_TIME_KEY, long.class);

        long elapsedMs = System.currentTimeMillis() - startTime;
        System.out.printf("[TIMING] %s: %dms%n",
            context.getRequiredTestMethod().getName(), elapsedMs);
    }
}

// Use your custom extension like any other
@ExtendWith(TimingExtension.class)
class PerformanceSensitiveTest {
    @Test
    void complexQuery_shouldRunInUnder100ms() { ... }
}
```

---

## 🔄 JUnit 5 vs JUnit 4 — Key Differences

Understanding the differences is important for reading older codebases and migrating them. The changes are not trivial tweaks — they represent a fundamentally different design philosophy.

```java
// ── JUnit 4 (old) ──
import org.junit.Test;
import org.junit.Before;
import org.junit.runner.RunWith;
import org.mockito.junit.MockitoJUnitRunner;

@RunWith(MockitoJUnitRunner.class)  // Only ONE runner allowed
public class OldStyleTest {          // Must be public

    @Before                          // Not @BeforeEach
    public void setUp() { ... }      // Must be public void

    @Test
    public void myTest() {           // Must be public void
        org.junit.Assert.assertEquals(expected, actual); // Different class
    }

    @Test(expected = IllegalArgumentException.class) // Clunky — can't inspect the exception
    public void throwsTest() { ... }

    @Test(timeout = 500) // Timeout in @Test annotation
    public void timesOut() { ... }
}

// ── JUnit 5 (new) ──
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.junit.jupiter.MockitoExtension;

@ExtendWith(MockitoExtension.class) // Composable — stack multiple
class NewStyleTest {                  // package-private is fine

    @BeforeEach                       // Renamed for clarity
    void setUp() { ... }              // No access modifier required

    @Test
    void myTest() {                   // No access modifier required
        assertEquals(expected, actual); // Same class: Assertions.*
    }

    @Test
    void throwsTest() {               // Returns the exception for inspection
        var ex = assertThrows(IllegalArgumentException.class, () -> ...);
        assertTrue(ex.getMessage().contains("expected text"));
    }

    @Test
    void timesOut() {                 // Dedicated assertion method
        assertTimeout(Duration.ofMillis(500), () -> ...);
    }
}
```

The three most impactful JUnit 5 changes are: test classes and methods no longer need to be public (reducing noise), `@RunWith` became `@ExtendWith` and supports multiple extensions (enabling composable tooling), and exception testing became `assertThrows()` which returns the exception (enabling inspection of the exception message and type together).

---

## ✅ Best Practices & Important Points

**Write tests that have exactly one reason to fail.** A test that verifies three unrelated things fails for the wrong reason and wastes debugging time. The rule of thumb is one logical assertion per test (though a single assertion can check multiple fields using `assertAll`). If you catch yourself writing "step 1, step 2, step 3" in a test comment, you have three tests masquerading as one.

**Use `@DisplayName` generously.** The string in `@DisplayName` appears in IDE test explorers, CI reports, and failure messages. A test named `test1` communicates nothing. A test named "findById returns empty Optional when ID does not exist" tells a developer exactly what broke without opening the file.

**Test class names should mirror the class they test.** If the class is `StudentService`, the unit test is `StudentServiceTest`. If it is an integration test involving the student persistence layer, name it `StudentRepositoryIntegrationTest`. Convention makes tests discoverable without IDE help.

**Lifecycle order in `@Nested` classes is: outer `@BeforeAll` → outer `@BeforeEach` → inner `@BeforeEach` → test → inner `@AfterEach` → outer `@AfterEach` → outer `@AfterAll`.** Understanding this lets you share expensive setup at the outer level while keeping scenario-specific setup in the nested class.

**Prefer `assertAll` whenever you are checking multiple properties of a single object.** Without it, a broken `id` field hides a broken `email` field until you fix `id` and rerun — a frustrating cycle on complex DTOs. `assertAll` gives you the complete failure picture in one run.

**Never put business logic in test code.** If your test has an `if` or a loop to decide what to assert, you are testing your test rather than your production code. Use `@ParameterizedTest` with `@MethodSource` instead.

---

## 🔑 Key Summary

```
JUnit Platform          → The launcher; what IDEs and build tools speak to
JUnit Jupiter           → The API you write against; everything in this chapter
JUnit Vintage           → Compatibility layer to run JUnit 4 tests on JUnit 5
@Test                   → Marks a method as a test case
@DisplayName            → Human-readable test name for reports and IDEs
@Disabled               → Skips a test; always include the reason
@BeforeEach / @AfterEach → Runs before/after every test method
@BeforeAll / @AfterAll  → Runs once before/after all tests in the class (static)
assertEquals            → (expected, actual) — order matters for failure messages
assertThrows            → Returns the exception so you can inspect its message
assertAll               → Groups assertions; all run even if one fails
@ParameterizedTest      → Run one test with many inputs
@ValueSource            → Single-value simple parameters
@CsvSource              → Multi-column table of parameters inline
@MethodSource           → Complex object parameters from a factory method
@EnumSource             → One run per enum value
@RepeatedTest           → Repeat a test N times; useful for randomness/concurrency
@Nested                 → Group related tests; outer lifecycle hooks still run
@Tag                    → Categorize tests for selective execution by build tools
@TempDir                → Auto-managed temporary directory; cleaned up after the test
@ExtendWith             → Composable hook system; replaces JUnit 4's @RunWith
```
