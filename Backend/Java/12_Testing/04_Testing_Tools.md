# 🛠️ 12.4 — Specialised Testing Tools
## AssertJ · Hamcrest · WireMock · Testcontainers · H2 · Awaitility · RestAssured

Each tool in this chapter fills a gap that JUnit 5's built-ins cannot fill elegantly. JUnit's built-in `assertEquals` works, but AssertJ's fluent chains read more like English. JUnit's `assertTrue` can technically verify a list contains a value, but Hamcrest matchers express that intent in a self-describing way. And JUnit has no answer at all for testing a slow async event, mocking an external HTTP server, or running your tests against a real PostgreSQL instance. That's where these tools come in.

---

## 1. AssertJ — Fluent, Readable Assertions

AssertJ is a Java assertion library built around a single idea: every assertion should read like a sentence in English, and every failure message should tell you exactly what went wrong without requiring you to mentally decode the output. The `spring-boot-starter-test` dependency includes AssertJ automatically.

### Why AssertJ Over JUnit's Built-ins?

Consider checking whether a list of students contains someone named "Alice". With JUnit you write `assertTrue(students.stream().anyMatch(s -> s.getName().equals("Alice")))`. When this fails, the message is simply `expected: <true> but was: <false>` — no indication of what was in the list. With AssertJ you write `assertThat(students).extracting(Student::getName).contains("Alice")`. When it fails, AssertJ prints the full content of the list and tells you exactly what it was looking for. The difference in debugging time is significant.

```java
import static org.assertj.core.api.Assertions.*;

class AssertJDemoTest {

    // ── Basic Value Assertions ──
    @Test
    void basicAssertions() {
        int score = 87;

        // Chaining reads as a sentence: "assert that score is greater than 80 and less than 100"
        assertThat(score)
            .isGreaterThan(80)
            .isLessThan(100)
            .isBetween(0, 100);

        String name = "Alice";
        assertThat(name)
            .isNotNull()
            .isNotBlank()
            .startsWith("Al")
            .endsWith("ce")
            .hasSize(5)
            .contains("lic")
            .doesNotContain("Bob")
            .isEqualToIgnoringCase("ALICE");
    }

    // ── Exception Assertions ──
    @Test
    void exceptionAssertions() {
        // assertThatThrownBy is more expressive than JUnit's assertThrows
        // because you chain assertions on the exception itself
        assertThatThrownBy(() -> studentService.findById(-1L))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("must be positive")
            .hasMessageStartingWith("ID")
            .hasNoCause();

        // assertThatExceptionOfType is the strongly-typed variant
        assertThatExceptionOfType(ResourceNotFoundException.class)
            .isThrownBy(() -> studentService.findById(999L))
            .withMessage("Student not found: 999")
            .withNoCause();

        // Verifying no exception is thrown
        assertThatCode(() -> studentService.findById(1L))
            .doesNotThrowAnyException();
    }

    // ── Collection Assertions ──
    @Test
    void collectionAssertions() {
        List<Student> students = studentService.findAll();

        // Size checks
        assertThat(students).hasSize(3).isNotEmpty();

        // Containment — does the collection contain these exact objects?
        assertThat(students).contains(alice, bob);

        // Containment using a specific field — "extracting" projects a field for assertion
        // This is one of AssertJ's most powerful features
        assertThat(students)
            .extracting(Student::getName)
            .containsExactlyInAnyOrder("Alice", "Bob", "Carol");

        // Multi-field extracting — checks multiple fields simultaneously per element
        assertThat(students)
            .extracting(Student::getName, Student::getEmail)
            .containsExactlyInAnyOrder(
                tuple("Alice", "alice@example.com"),
                tuple("Bob",   "bob@example.com"),
                tuple("Carol", "carol@example.com")
            );

        // Filtering before asserting
        assertThat(students)
            .filteredOn(s -> s.getScore() > 85.0)
            .extracting(Student::getName)
            .containsExactlyInAnyOrder("Alice", "Carol");

        // Condition-based filtering using a named Condition object
        Condition<Student> enrolled = new Condition<>(
            s -> s.getStatus() == Status.ENROLLED, "enrolled");
        assertThat(students).areAtLeast(2, enrolled);

        // Verifying every element satisfies a condition
        assertThat(students)
            .allMatch(s -> s.getScore() >= 0 && s.getScore() <= 100,
                "all scores must be in valid range");

        // Verifying at least one element satisfies a condition
        assertThat(students)
            .anyMatch(s -> s.getScore() > 90, "at least one student should be high achiever");

        // None satisfy
        assertThat(students)
            .noneMatch(s -> s.getScore() < 0, "no student should have negative score");
    }

    // ── Optional Assertions ──
    @Test
    void optionalAssertions() {
        Optional<Student> found = studentService.findById(1L);
        Optional<Student> missing = studentService.findById(999L);

        assertThat(found)
            .isPresent()
            .hasValueSatisfying(s -> {
                assertThat(s.getName()).isEqualTo("Alice");
                assertThat(s.getScore()).isGreaterThan(90.0);
            });

        assertThat(missing).isEmpty();
    }

    // ── Map Assertions ──
    @Test
    void mapAssertions() {
        Map<String, Integer> gradeMap = gradeService.buildGradeMap(students);

        assertThat(gradeMap)
            .hasSize(3)
            .containsKey("alice@example.com")
            .containsEntry("alice@example.com", 95)
            .doesNotContainKey("nobody@example.com");
    }

    // ── Custom Failure Message with as() ──
    @Test
    void customFailureMessages() {
        Student student = studentService.findById(1L).orElseThrow();

        // as() adds a description that appears at the START of the failure message
        // making it immediately clear what business rule was being checked
        assertThat(student.getScore())
            .as("Score for student '%s' (id=%d)", student.getName(), student.getId())
            .isGreaterThanOrEqualTo(0.0)
            .isLessThanOrEqualTo(100.0);
    }

    // ── Recursive Comparison — comparing deep object graphs ──
    @Test
    void recursiveComparison() {
        Student expected = new Student(1L, "Alice", "alice@example.com", 95.5);
        Student actual   = studentService.findById(1L).orElseThrow();

        // Compares all fields recursively, ignoring specified fields
        // Perfect for DTOs where you don't want to override equals()
        assertThat(actual)
            .usingRecursiveComparison()
            .ignoringFields("createdAt", "updatedAt") // Skip audit timestamps
            .isEqualTo(expected);
    }
}
```

### SoftAssertions — Non-Failing Multiple Assertions

SoftAssertions collect all failures and report them together at the end, rather than stopping at the first failure. This serves the same purpose as JUnit's `assertAll` but works with AssertJ's fluent API.

```java
@Test
void studentDTO_allFieldsCorrect_usingsSoftAssertions() {
    StudentDTO dto = studentService.buildDTO(1L);

    // All assertions run; ALL failures are reported at once
    SoftAssertions softly = new SoftAssertions();
    softly.assertThat(dto.id()).isEqualTo(1L);
    softly.assertThat(dto.name()).isEqualTo("Alice");
    softly.assertThat(dto.email()).isEqualTo("alice@example.com");
    softly.assertThat(dto.score()).isBetween(0.0, 100.0);
    softly.assertAll(); // Throws an AssertionError listing ALL failures, not just the first
}

// Or the lambda-based version which calls assertAll automatically
@Test
void studentDTO_allFieldsCorrect_usingAssertSoftly() {
    StudentDTO dto = studentService.buildDTO(1L);

    assertSoftly(softly -> {
        softly.assertThat(dto.id()).isEqualTo(1L);
        softly.assertThat(dto.name()).isEqualTo("Alice");
        softly.assertThat(dto.email()).isEqualTo("alice@example.com");
    });
}
```

---

## 2. Hamcrest — Matcher-Based Assertions

Hamcrest provides a library of composable "matchers" — objects that describe a condition and produce readable failure messages. It is older than AssertJ and less expressive for complex scenarios, but you will encounter it frequently in existing codebases and in Spring's `MockMvc` integration where its matchers are used in `andExpect()` chains.

```java
import static org.hamcrest.MatcherAssert.assertThat;
import static org.hamcrest.Matchers.*;

class HamcrestDemoTest {

    @Test
    void basicHamcrestMatchers() {
        // Core matchers
        assertThat(42,    is(equalTo(42)));         // is() is just syntactic sugar
        assertThat(42,    not(equalTo(0)));          // not() negates any matcher
        assertThat(score, greaterThan(80));
        assertThat(score, lessThanOrEqualTo(100));
        assertThat(score, allOf(greaterThan(0), lessThan(100))); // AND composition
        assertThat(score, anyOf(equalTo(90), equalTo(95)));       // OR composition

        // String matchers
        assertThat(name, containsString("Ali"));
        assertThat(name, startsWith("Al"));
        assertThat(name, endsWith("ce"));
        assertThat(name, matchesPattern("[A-Za-z]+"));

        // Null matchers
        assertThat(result,  notNullValue());
        assertThat(missing, nullValue());
    }

    @Test
    void hamcrestCollectionMatchers() {
        List<String> names = List.of("Alice", "Bob", "Carol");

        assertThat(names, hasSize(3));
        assertThat(names, hasItem("Alice"));
        assertThat(names, hasItems("Alice", "Bob")); // Both must be present
        assertThat(names, not(hasItem("Dave")));

        // containsInAnyOrder: EVERY element must match, in any order, no extras
        assertThat(names, containsInAnyOrder("Carol", "Alice", "Bob"));

        // Map matchers
        Map<String, Integer> grades = Map.of("Alice", 95, "Bob", 82);
        assertThat(grades, hasEntry("Alice", 95));
        assertThat(grades, hasKey("Bob"));
        assertThat(grades, hasValue(82));
    }

    // ── Hamcrest in MockMvc — the place you'll use it most ──
    @WebMvcTest(StudentController.class)
    class StudentControllerHamcrestTest {
        @Autowired MockMvc mockMvc;
        @MockBean StudentService studentService;

        @Test
        void getStudents_returnsJsonArrayWithMatchingFields() throws Exception {
            when(studentService.findAll()).thenReturn(
                List.of(new StudentDTO(1L, "Alice", "alice@example.com", 95.5))
            );

            mockMvc.perform(get("/api/students"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$",          hasSize(1)))
                .andExpect(jsonPath("$[0].id",    is(1)))
                .andExpect(jsonPath("$[0].name",  is("Alice")))
                .andExpect(jsonPath("$[0].score", greaterThan(90.0)));
        }
    }
}
```

---

## 3. WireMock — Mock HTTP Servers for External API Testing

WireMock spins up a real HTTP server on a local port during your test and lets you define what it returns for specific requests. This solves one of the hardest integration testing problems: your code calls an external REST API (a payment gateway, a weather service, an identity provider), and you need to test how your code behaves for different responses — success, 404, 500, timeout, malformed JSON — without actually calling the real service.

```xml
<dependency>
    <groupId>org.wiremock</groupId>
    <artifactId>wiremock-standalone</artifactId>
    <version>3.5.4</version>
    <scope>test</scope>
</dependency>
```

```java
import com.github.tomakehurst.wiremock.junit5.WireMockExtension;
import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static com.github.tomakehurst.wiremock.core.WireMockConfiguration.wireMockConfig;

@ExtendWith(WireMockExtension.class) // Starts WireMock server; stops it after all tests
class PaymentGatewayClientTest {

    // @RegisterExtension gives you a reference to the WireMock server
    // so you can read its dynamic port and point your HTTP client at it
    @RegisterExtension
    static WireMockExtension wireMock = WireMockExtension.newInstance()
        .options(wireMockConfig().dynamicPort()) // Use a random available port
        .build();

    private PaymentGatewayClient client;

    @BeforeEach
    void setUp() {
        // Point your service at WireMock's URL instead of the real gateway
        String wireMockUrl = wireMock.baseUrl(); // e.g. "http://localhost:49821"
        client = new PaymentGatewayClient(wireMockUrl, new RestTemplate());
    }

    // ── Testing a successful charge ──
    @Test
    @DisplayName("charge: returns ChargeResult when gateway returns 200 with charge ID")
    void charge_whenGatewaySucceeds_returnsChargeResult() {
        // Stub: when POST /v1/charges is called with a JSON body containing the amount,
        // return a 200 OK with this JSON response
        wireMock.stubFor(post(urlEqualTo("/v1/charges"))
            .withHeader("Content-Type", containing("application/json"))
            .withRequestBody(matchingJsonPath("$.amount"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                      "id": "ch_abc123",
                      "status": "succeeded",
                      "amount": 29999
                    }
                    """)
            )
        );

        ChargeResult result = client.charge(new ChargeRequest("tok_visa", 29999, "USD"));

        assertThat(result.getId()).isEqualTo("ch_abc123");
        assertThat(result.getStatus()).isEqualTo("succeeded");

        // Verify WireMock received exactly one matching request
        wireMock.verify(1, postRequestedFor(urlEqualTo("/v1/charges")));
    }

    // ── Testing a declined payment ──
    @Test
    @DisplayName("charge: throws PaymentDeclinedException when gateway returns 402")
    void charge_whenGatewayDeclines_throwsPaymentDeclinedException() {
        wireMock.stubFor(post(urlEqualTo("/v1/charges"))
            .willReturn(aResponse()
                .withStatus(402)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                      "error": {
                        "code": "card_declined",
                        "message": "Your card was declined."
                      }
                    }
                    """)
            )
        );

        assertThatThrownBy(() -> client.charge(new ChargeRequest("tok_declined", 9999, "USD")))
            .isInstanceOf(PaymentDeclinedException.class)
            .hasMessageContaining("card_declined");
    }

    // ── Testing timeout handling ──
    @Test
    @DisplayName("charge: throws GatewayTimeoutException when gateway takes too long")
    void charge_whenGatewayTimesOut_throwsGatewayTimeoutException() {
        wireMock.stubFor(post(urlEqualTo("/v1/charges"))
            .willReturn(aResponse()
                .withFixedDelay(5000)  // Delay response by 5 seconds
                .withStatus(200)
            )
        );
        // Your client should have a 2-second timeout configured
        assertThatThrownBy(() -> client.charge(new ChargeRequest("tok_visa", 100, "USD")))
            .isInstanceOf(GatewayTimeoutException.class);
    }

    // ── Testing retry behaviour ──
    @Test
    @DisplayName("charge: retries on 503 and succeeds when service recovers")
    void charge_when503ThenSuccess_retriesAndSucceeds() {
        // First call → 503 Service Unavailable; second call → 200 OK
        wireMock.stubFor(post(urlEqualTo("/v1/charges"))
            .inScenario("service-recovery")
            .whenScenarioStateIs(STARTED)
            .willReturn(aResponse().withStatus(503))
            .willSetStateTo("recovered")
        );
        wireMock.stubFor(post(urlEqualTo("/v1/charges"))
            .inScenario("service-recovery")
            .whenScenarioStateIs("recovered")
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""{"id":"ch_retry_ok","status":"succeeded","amount":100}""")
            )
        );

        ChargeResult result = client.charge(new ChargeRequest("tok_visa", 100, "USD"));

        assertThat(result.getStatus()).isEqualTo("succeeded");
        // Verify the client made exactly 2 requests (1 failed + 1 retry)
        wireMock.verify(2, postRequestedFor(urlEqualTo("/v1/charges")));
    }
}
```

---

## 4. Testcontainers — Real Infrastructure in Tests

Testcontainers starts a Docker container during your test run, giving your integration tests a real database, message broker, or any other service rather than an imperfect in-memory substitute. The container is managed automatically: it starts before your tests and is garbage-collected when the test class finishes.

The key insight behind Testcontainers is that H2 and Redis mock clients are approximations. PostgreSQL behaves differently from H2 in dozens of subtle ways: different SQL syntax, different handling of case sensitivity in column names, different transaction isolation semantics, different index behaviour. Testing against the real database eliminates an entire class of bugs that only appear in production.

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>kafka</artifactId>
    <version>1.19.7</version>
    <scope>test</scope>
</dependency>
```

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class StudentRepositoryContainerTest {

    // static = one container shared by all test methods in this class
    // This is important for performance: starting a container takes 2–5 seconds;
    // you don't want to pay that cost for every test method
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test")
        // Optionally run a SQL script to pre-populate schema/data
        .withInitScript("db/test-schema.sql");

    // @DynamicPropertySource wires the container's runtime URL into Spring's
    // environment before the context is created — overriding application.properties
    @DynamicPropertySource
    static void overrideDataSourceProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "create-drop");
    }

    @Autowired private StudentRepository studentRepository;
    @Autowired private TestEntityManager  entityManager;

    @BeforeEach
    void setUp() {
        // Each test starts with clean data
        studentRepository.deleteAll();
        entityManager.flush();
    }

    @Test
    @DisplayName("findByEmail: returns student when email matches (PostgreSQL case-sensitive)")
    void findByEmail_isCaseSensitive() {
        entityManager.persistAndFlush(
            new Student("Alice", "alice@example.com", 95.0, Status.ENROLLED));

        Optional<Student> found   = studentRepository.findByEmail("alice@example.com");
        Optional<Student> notFound = studentRepository.findByEmail("ALICE@EXAMPLE.COM");

        assertThat(found).isPresent();
        // This test would PASS incorrectly against H2 which is case-insensitive by default
        // Against real PostgreSQL it correctly reveals the case-sensitivity behaviour
        assertThat(notFound).isEmpty();
    }

    @Test
    @DisplayName("findTopByOrderByScoreDesc: returns the highest-scoring student")
    void findTopByOrderByScoreDesc_returnsHighestScorer() {
        entityManager.persistAndFlush(new Student("Alice", "a@e.com", 95.0, Status.ENROLLED));
        entityManager.persistAndFlush(new Student("Bob",   "b@e.com", 72.0, Status.ENROLLED));
        entityManager.persistAndFlush(new Student("Carol", "c@e.com", 88.0, Status.ENROLLED));

        Optional<Student> topStudent = studentRepository.findTopByOrderByScoreDesc();

        assertThat(topStudent).isPresent();
        assertThat(topStudent.get().getName()).isEqualTo("Alice");
    }
}

// ── Kafka integration test with Testcontainers ──
@SpringBootTest
@Testcontainers
class OrderEventProducerIntegrationTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

    @DynamicPropertySource
    static void kafkaProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired private OrderEventProducer producer;

    @Test
    @DisplayName("publishOrderPlaced: message arrives in the topic with correct content")
    void publishOrderPlaced_messageArrivesInTopic() throws Exception {
        OrderPlacedEvent event = new OrderPlacedEvent(1L, "COURSE-42", 299.99);

        producer.publish(event);

        // Use a real Kafka consumer to verify the message was actually published
        Properties consumerProps = new Properties();
        consumerProps.put("bootstrap.servers", kafka.getBootstrapServers());
        consumerProps.put("group.id", "test-verifier");
        consumerProps.put("auto.offset.reset", "earliest");
        consumerProps.put("key.deserializer",   StringDeserializer.class.getName());
        consumerProps.put("value.deserializer", StringDeserializer.class.getName());

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(consumerProps)) {
            consumer.subscribe(List.of("order-events"));
            ConsumerRecords<String, String> records =
                consumer.poll(Duration.ofSeconds(10));

            assertThat(records.count()).isGreaterThanOrEqualTo(1);
            ConsumerRecord<String, String> record = records.iterator().next();
            assertThat(record.value()).contains("COURSE-42");
        }
    }
}
```

### Reusing Containers Across Test Classes

Starting a new container for every test class that needs a database is wasteful when all those classes can share one container. The standard pattern uses a base class with a static container:

```java
// Base class — all DB integration tests extend this
public abstract class PostgresIntegrationTestBase {

    // static on a base class means ONE container for the entire test suite
    @Container
    protected static final PostgreSQLContainer<?> POSTGRES =
        new PostgreSQLContainer<>("postgres:16-alpine")
            .withReuse(true); // Testcontainers reuse: container survives between Maven runs

    static {
        POSTGRES.start();
    }

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      POSTGRES::getJdbcUrl);
        registry.add("spring.datasource.username", POSTGRES::getUsername);
        registry.add("spring.datasource.password", POSTGRES::getPassword);
    }
}

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class StudentRepositoryTest extends PostgresIntegrationTestBase { /* tests */ }

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class CourseRepositoryTest extends PostgresIntegrationTestBase { /* tests */ }
// Both classes share the same Postgres container — started once, reused
```

---

## 5. H2 — In-Memory Database for Fast Tests

H2 is a Java SQL database that runs entirely in-memory. Spring Boot configures it automatically when `@DataJpaTest` is present and no other database driver is on the classpath. It is the fastest option for repository tests because it starts and stops in milliseconds, no Docker required.

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

```properties
# src/test/resources/application-test.properties
# H2 in-memory: fresh database for every test run
spring.datasource.url=jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;MODE=PostgreSQL
# MODE=PostgreSQL makes H2 behave more like PostgreSQL (but not perfectly)
spring.datasource.driver-class-name=org.h2.Driver
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
```

```java
// @DataJpaTest uses H2 by default — nothing special required
@DataJpaTest
class CourseRepositoryH2Test {

    @Autowired private CourseRepository courseRepository;
    @Autowired private TestEntityManager entityManager;

    @Test
    @DisplayName("findByInstructor: returns all courses taught by a given instructor")
    void findByInstructor_returnsMatchingCourses() {
        Instructor alice = entityManager.persistAndFlush(
            new Instructor("Alice", "alice@uni.edu"));
        entityManager.persistAndFlush(new Course("Java Basics",    alice, 200.0));
        entityManager.persistAndFlush(new Course("Advanced Java",  alice, 350.0));
        entityManager.persistAndFlush(new Course("Spring Boot",    alice, 299.0));

        Instructor bob = entityManager.persistAndFlush(
            new Instructor("Bob", "bob@uni.edu"));
        entityManager.persistAndFlush(new Course("Python 101", bob, 150.0));

        List<Course> aliceCourses = courseRepository.findByInstructor(alice);

        assertThat(aliceCourses).hasSize(3);
        assertThat(aliceCourses)
            .extracting(Course::getTitle)
            .containsExactlyInAnyOrder("Java Basics", "Advanced Java", "Spring Boot");
    }
}
```

The trade-off between H2 and Testcontainers is a recurring architectural decision. Use H2 when you want maximum test speed and your queries are simple and standard SQL. Switch to Testcontainers when your queries use database-specific features (PostgreSQL JSON operators, window functions, full-text search, custom types), when you have hit an H2 compatibility bug, or when you want absolute confidence that your integration tests match production behaviour.

---

## 6. Awaitility — Testing Asynchronous Code

Asynchronous operations don't finish by the time your test's `Act` line returns control. If you call `emailService.sendAsync(message)` and immediately assert `verify(smtpClient).send(any())`, the assertion races against the background thread — it will pass or fail non-deterministically, making the test unreliable. The naive fix is `Thread.sleep(500)` — but that is both slow (it always sleeps the full duration even when the operation finishes in 10ms) and brittle (a slow CI machine may need 2 seconds instead of 500ms).

Awaitility is the correct solution. It polls your condition repeatedly until it passes or a timeout expires, sleeping briefly between polls. Your test waits exactly as long as needed — no more, no less.

```xml
<dependency>
    <groupId>org.awaitility</groupId>
    <artifactId>awaitility</artifactId>
    <version>4.2.1</version>
    <scope>test</scope>
</dependency>
```

```java
import static org.awaitility.Awaitility.*;
import java.time.Duration;

class AsyncEmailServiceTest {

    private AsyncEmailService emailService;
    private FakeSmtpServer     fakeSmtp;

    @BeforeEach
    void setUp() {
        fakeSmtp     = new FakeSmtpServer();
        emailService = new AsyncEmailService(fakeSmtp);
    }

    // ── Basic usage: wait until a condition is true ──
    @Test
    @DisplayName("sendAsync: email is eventually delivered to the SMTP server")
    void sendAsync_eventuallyDelivered() {
        emailService.sendAsync(new EmailMessage("alice@example.com", "Welcome!"));

        // "within 5 seconds, poll every 100ms until fakeSmtp has received a message"
        await()
            .atMost(Duration.ofSeconds(5))
            .pollInterval(Duration.ofMillis(100))
            .untilAsserted(() ->
                assertThat(fakeSmtp.getReceivedMessages()).hasSize(1)
            );
    }

    // ── Waiting for a specific field value ──
    @Test
    @DisplayName("processOrder: status changes to COMPLETED within 3 seconds")
    void processOrderAsync_statusBecomesCompleted() {
        Order order = orderService.create(new CreateOrderRequest(1L, "COURSE-42"));

        await()
            .atMost(3, TimeUnit.SECONDS)
            .pollInterval(200, TimeUnit.MILLISECONDS)
            .until(() -> orderRepository.findById(order.getId())
                             .map(o -> o.getStatus() == OrderStatus.COMPLETED)
                             .orElse(false));

        // Now that we know the status has changed, we can assert on the full order
        Order completed = orderRepository.findById(order.getId()).orElseThrow();
        assertThat(completed.getCompletedAt()).isNotNull();
    }

    // ── untilAsserted vs until ──
    @Test
    void awaitilityTwoStyles() {
        sendBackgroundNotification();

        // Style 1: until(Callable<Boolean>) — polls until the callable returns true
        await().atMost(3, SECONDS).until(() -> notificationStore.count() > 0);

        // Style 2: untilAsserted(ThrowableRunnable) — polls until the block doesn't throw
        // This is more expressive because you can use any assertion inside
        await()
            .atMost(3, SECONDS)
            .untilAsserted(() -> {
                assertThat(notificationStore.getLatest().getRecipient())
                    .isEqualTo("alice@example.com");
                assertThat(notificationStore.getLatest().getStatus())
                    .isEqualTo(NotificationStatus.SENT);
            });
    }

    // ── Configuring Awaitility globally for your test suite ──
    // Put this in a base test class or a @BeforeAll
    @BeforeAll
    static void configureAwaitility() {
        Awaitility.setDefaultTimeout(Duration.ofSeconds(10));
        Awaitility.setDefaultPollInterval(Duration.ofMillis(50));
        Awaitility.setDefaultPollDelay(Duration.ZERO); // Start polling immediately
    }
}
```

---

## 7. RestAssured — Fluent REST API Testing

RestAssured is a DSL for testing REST APIs using real HTTP. Where MockMvc simulates HTTP calls inside the JVM without a running server, RestAssured makes genuine HTTP requests to a running server. This makes it perfect for integration and E2E tests where you want the full HTTP stack — real serialization, real content negotiation, real headers.

```xml
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>rest-assured</artifactId>
    <version>5.4.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.rest-assured</groupId>
    <artifactId>spring-mock-mvc</artifactId>
    <version>5.4.0</version>
    <scope>test</scope>
</dependency>
```

```java
import static io.restassured.RestAssured.*;
import static io.restassured.matcher.RestAssuredMatchers.*;
import static org.hamcrest.Matchers.*;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class StudentApiRestAssuredTest {

    @LocalServerPort
    private int port;

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url",      postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired private StudentRepository studentRepository;

    @BeforeEach
    void setUp() {
        // Point RestAssured at our random port
        RestAssured.baseURI  = "http://localhost";
        RestAssured.port     = port;
        RestAssured.basePath = "/api";
        studentRepository.deleteAll();
    }

    // ── GET all students ──
    @Test
    @DisplayName("GET /students returns 200 with an array of students")
    void getAllStudents_returns200AndArray() {
        studentRepository.saveAll(List.of(
            new Student("Alice", "alice@example.com", 95.0, Status.ENROLLED),
            new Student("Bob",   "bob@example.com",   82.0, Status.ENROLLED)
        ));

        given()
            .header("Accept", "application/json")
        .when()
            .get("/students")
        .then()
            .statusCode(200)
            .contentType(ContentType.JSON)
            .body("",         hasSize(2))
            .body("name",     hasItems("Alice", "Bob"))
            .body("[0].email", notNullValue());
    }

    // ── POST — create a new student ──
    @Test
    @DisplayName("POST /students creates and returns 201 with Location header")
    void createStudent_returns201WithLocationHeader() {
        String requestBody = """
            {
              "name":  "Carol",
              "email": "carol@example.com",
              "score": 88.5
            }
            """;

        Integer newId =
        given()
            .contentType(ContentType.JSON)
            .body(requestBody)
        .when()
            .post("/students")
        .then()
            .statusCode(201)
            .header("Location", containsString("/api/students/"))
            .body("name",  equalTo("Carol"))
            .body("email", equalTo("carol@example.com"))
            .body("score", equalTo(88.5f))
            .body("id",    notNullValue())
        .extract()
            .path("id"); // Extract the returned ID for use in subsequent assertions

        // Verify we can retrieve the created student
        given()
        .when()
            .get("/students/" + newId)
        .then()
            .statusCode(200)
            .body("name", equalTo("Carol"));
    }

    // ── Validation error response ──
    @Test
    @DisplayName("POST /students with blank name returns 400 with validation errors")
    void createStudent_withBlankName_returns400() {
        String invalidBody = """
            {"name": "", "email": "carol@example.com", "score": 88.5}
            """;

        given()
            .contentType(ContentType.JSON)
            .body(invalidBody)
        .when()
            .post("/students")
        .then()
            .statusCode(400)
            .body("validationErrors.name", notNullValue())
            .body("validationErrors.name", containsString("required"));
    }

    // ── Authenticated endpoint using JWT ──
    @Test
    @DisplayName("DELETE /students/{id} returns 403 without a valid JWT")
    void deleteStudent_withoutAuth_returns403() {
        given()
        .when()
            .delete("/students/1")
        .then()
            .statusCode(403); // Spring Security blocks the unauthenticated request
    }

    @Test
    @DisplayName("DELETE /students/{id} returns 204 with valid ADMIN JWT")
    void deleteStudent_withAdminJwt_returns204() {
        Student saved = studentRepository.save(
            new Student("Dave", "dave@example.com", 70.0, Status.ENROLLED));

        String adminToken = obtainAdminJwt(); // Helper that logs in and retrieves a JWT

        given()
            .header("Authorization", "Bearer " + adminToken)
        .when()
            .delete("/students/" + saved.getId())
        .then()
            .statusCode(204);
    }

    // ── Response body extraction and further assertion ──
    @Test
    @DisplayName("GET /students/{id} — extract and assert on response body as Java object")
    void getStudentById_extractsAndAsserts() {
        Student saved = studentRepository.save(
            new Student("Eve", "eve@example.com", 91.5, Status.ENROLLED));

        StudentDTO dto =
        given()
        .when()
            .get("/students/" + saved.getId())
        .then()
            .statusCode(200)
        .extract()
            .as(StudentDTO.class); // Deserialise response body into a Java object

        assertThat(dto.name()).isEqualTo("Eve");
        assertThat(dto.email()).isEqualTo("eve@example.com");
        assertThat(dto.score()).isEqualTo(91.5);
    }

    private String obtainAdminJwt() {
        return given()
            .contentType(ContentType.JSON)
            .body("""{"email":"admin@example.com","password":"adminpass"}""")
        .when()
            .post("/auth/login")
        .then()
            .statusCode(200)
        .extract()
            .path("accessToken");
    }
}
```

---

## ✅ Best Practices & Key Decision Guide

**Prefer AssertJ for all new tests.** JUnit's built-in assertions are adequate but produce weak failure messages. AssertJ's fluent API is more readable in code reviews and produces detailed failure messages that dramatically reduce time spent debugging failing tests. Once you use `extracting().containsExactlyInAnyOrder()` on a list, you will never go back to `assertTrue(list.stream().anyMatch(...))`.

**Use Hamcrest only in MockMvc chains.** The `jsonPath()` assertion in `MockMvcResultMatchers` uses Hamcrest matchers natively — that is the one place Hamcrest is idiomatic and unavoidable. For standalone assertions, prefer AssertJ.

**Choose WireMock when your code makes outbound HTTP calls.** If your service calls a payment gateway, an identity provider, a weather API, or any external service, WireMock gives you a real HTTP server to test against — including timeout simulation, error responses, and request verification. A mock created with Mockito cannot simulate the network-level behaviour that WireMock can.

**Choose Testcontainers over H2 for any query that uses database-specific features.** H2's PostgreSQL compatibility mode handles common cases but fails on JSON operators, `RETURNING` clauses, certain index types, and many other PostgreSQL-specific features. Testcontainers costs 2–5 seconds of startup time but eliminates an entire class of production-only bugs.

**Use Awaitility for every async test — never `Thread.sleep()`.** A fixed sleep makes every test run as slow as the worst-case scenario. Awaitility makes tests run as fast as the operation finishes and only times out when something is genuinely wrong.

**Use RestAssured for E2E/contract tests; use MockMvc for controller unit/slice tests.** MockMvc is faster and doesn't require a running server, making it ideal for `@WebMvcTest` slices that test one controller. RestAssured makes real HTTP calls, making it the right tool when you need to verify the entire stack — serialization, content negotiation, security filters, and actual HTTP behaviour.

---

## 🔑 Key Summary

```
AssertJ                     → Fluent assertion library; rich failure messages; assertThat(x).isEqualTo(y)
extracting()                → Project a field from a collection for assertion — most powerful AssertJ feature
SoftAssertions              → Collect all assertion failures and report them together
usingRecursiveComparison()  → Deep object graph comparison without overriding equals()
Hamcrest                    → Matcher-based assertions; use mainly in MockMvc jsonPath() chains
WireMock                    → Starts a real HTTP server; stubs external REST API calls in tests
wireMock.stubFor()          → Define what the mock server returns for a given request pattern
wireMock.verify()           → Assert that WireMock received specific requests
Testcontainers              → Starts real Docker containers (PostgreSQL, Kafka, Redis) for tests
@Container (static)         → One container shared across all test methods — important for speed
@DynamicPropertySource      → Override Spring properties at runtime with container-generated values
H2                          → In-memory SQL database; fast, zero Docker; limited PostgreSQL compatibility
Awaitility                  → Polls until an async condition is true; never use Thread.sleep() in tests
await().atMost().untilAsserted() → Core Awaitility pattern for async assertions
RestAssured                 → Fluent HTTP client for testing REST APIs against a real running server
given().when().then()       → RestAssured's BDD-style DSL: setup → action → assertion
.extract().as(Class)        → Deserialise the response body into a Java object for further assertion
```
