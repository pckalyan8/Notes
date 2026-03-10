# 📙 12.3 — Test Pyramid, Strategy, AAA, BDD & Naming Conventions

This chapter is the most important in the phase, even though it contains the least code. The tools in JUnit 5 and Mockito are only valuable when guided by a coherent testing strategy. Without that strategy, developers write too many of the wrong kind of tests, too few of the right kind, and end up with a test suite that is simultaneously slow, fragile, and low-confidence.

---

## 🧠 The Test Pyramid — Your Architectural Blueprint

The **Test Pyramid** was popularized by Mike Cohn in "Succeeding with Agile" and later refined by Martin Fowler. It describes not just *what* kinds of tests to write, but *how many* of each and *why* the proportions matter.

```
                    ▲
                   / \
                  /   \          ← E2E Tests (few, slow, high-confidence for critical paths)
                 /     \
                /───────\
               /         \      ← Integration Tests (moderate, test component interactions)
              /           \
             /─────────────\
            /               \   ← Unit Tests (many, fast, test individual logic)
           /─────────────────\
```

The pyramid shape is deliberate. The base is wide because unit tests are cheap to write, cheap to run (milliseconds each), and precise in their feedback when they fail. The top is narrow because end-to-end tests are expensive to write, slow to run (seconds to minutes each), and imprecise in their feedback — a broken E2E test tells you "something is wrong" rather than "this specific function has a bug".

Building an **inverted pyramid** (many E2E tests, few unit tests) is one of the most damaging things a team can do to its development velocity. The test suite becomes a bottleneck: it takes 30 minutes to run, it breaks on every infrastructure hiccup (Docker not available, external API rate limited), and when it does fail, the failure could be anywhere in the stack.

---

## 🧱 The Three Layers — What Each Layer Tests

### Unit Tests — The Foundation

A unit test verifies the behaviour of a single class in complete isolation. Every external collaborator is replaced with a mock (Mockito). The test runs in a single JVM thread, touches no I/O of any kind (no DB, no network, no filesystem), and completes in milliseconds.

```java
// This is a unit test. It tests OrderService's logic.
// PaymentService is a mock — it doesn't make real charges.
// OrderRepository is a mock — it doesn't touch a database.
@ExtendWith(MockitoExtension.class)
class OrderServiceUnitTest {

    @Mock private PaymentService   paymentService;
    @Mock private OrderRepository  orderRepository;
    @InjectMocks private OrderService orderService;

    @Test
    @DisplayName("placeOrder: saves the order and returns it when payment succeeds")
    void placeOrder_whenPaymentSucceeds_savesAndReturnsOrder() {
        // Arrange
        when(paymentService.charge(any())).thenReturn(new ChargeResult("ch_123", true));
        when(orderRepository.save(any())).thenAnswer(inv -> {
            Order o = inv.getArgument(0);
            o.setId(99L);
            return o;
        });

        // Act
        Order result = orderService.placeOrder(new PlaceOrderRequest(1L, "COURSE-42"));

        // Assert
        assertNotNull(result.getId(), "Saved order should have a DB-assigned ID");
        verify(paymentService, times(1)).charge(any());
    }
}
```

Unit tests should make up roughly 70% of your test suite. Because they are isolated, a failing unit test tells you precisely which class and method is broken.

### Integration Tests — The Middle Layer

An integration test verifies that two or more real components work correctly together. It might test that `StudentRepository` produces the correct SQL for Spring Data's derived query methods, or that `OrderService` correctly coordinates with `InventoryRepository` and `PaymentService` when they are both real objects pointed at a real test database.

```java
// This is an integration test. It uses a real PostgreSQL database
// (spun up by Testcontainers) and real Spring Data JPA.
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class OrderRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired private OrderRepository orderRepository;
    @Autowired private TestEntityManager entityManager;

    @Test
    @DisplayName("findByCustomerId: returns only orders belonging to the given customer")
    void findByCustomerId_returnsOnlyMatchingOrders() {
        entityManager.persistAndFlush(new Order(1L, "COURSE-10", OrderStatus.PLACED));
        entityManager.persistAndFlush(new Order(2L, "COURSE-20", OrderStatus.PLACED));

        List<Order> result = orderRepository.findByCustomerId(1L);

        assertEquals(1, result.size());
        assertEquals("COURSE-10", result.get(0).getCourseId());
    }
}
```

Integration tests should make up roughly 20% of your suite. They are slower and need infrastructure (a database, a message broker), but they catch a critical class of bugs that unit tests miss: the gap between what your mock returns and what the real component actually does.

### End-to-End (E2E) Tests — The Top

An E2E test exercises the entire application from the outside: an HTTP request enters, travels through the controller, service, repository, and database, and a response comes back. Nothing is mocked.

```java
// This is an E2E test. The full Spring Boot context is loaded.
// A real embedded Tomcat serves on a random port.
// A real PostgreSQL container (Testcontainers) is the database.
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderApiE2ETest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry reg) {
        reg.add("spring.datasource.url",      postgres::getJdbcUrl);
        reg.add("spring.datasource.username", postgres::getUsername);
        reg.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired private TestRestTemplate restTemplate;

    @Test
    @DisplayName("POST /api/orders → 201 Created with order body")
    void placeOrder_returnsCreatedOrder() {
        PlaceOrderRequest request = new PlaceOrderRequest(1L, "COURSE-42");

        ResponseEntity<OrderDTO> response = restTemplate.postForEntity(
            "/api/orders", request, OrderDTO.class
        );

        assertEquals(HttpStatus.CREATED, response.getStatusCode());
        assertNotNull(response.getBody().getId());
        assertEquals("COURSE-42", response.getBody().getCourseId());
    }
}
```

E2E tests should make up only about 10% of your suite. They give high confidence for the critical paths through your system (login, checkout, core CRUD flows) but are not meant to exhaustively test every edge case — that is unit tests' job.

---

## 🗂️ The AAA Pattern — Arrange, Act, Assert

Every good test, regardless of its level in the pyramid, follows the **AAA pattern**. Each test has exactly three sections: one that sets up the world, one that performs the operation being tested, and one that checks the outcome.

```java
@Test
@DisplayName("calculateDiscount: 20% off for orders over $500")
void calculateDiscount_forLargeOrder_appliesTwentyPercent() {

    // ── Arrange ──
    // Build the preconditions. Set up all data, mocks, and state
    // that the operation needs to be exercised correctly.
    Order order = new Order();
    order.setSubtotal(new BigDecimal("600.00"));
    order.setCustomerTier(CustomerTier.STANDARD);

    // ── Act ──
    // Perform exactly ONE operation — the thing being tested.
    // There should be one line here, or a small cluster of tightly related lines.
    BigDecimal discount = discountService.calculate(order);

    // ── Assert ──
    // Verify the outcome. Check return values, state changes, or interactions.
    assertEquals(new BigDecimal("120.00"), discount,
        "20% of $600 should equal $120 discount");
}
```

The value of the AAA structure is that it makes tests scannable. Any developer can open an unfamiliar test and immediately locate the setup, the operation, and the verification by looking at the blank lines. Tests without this structure tend to intermix assertions with more actions, making them confusing to read and maintain.

A related variant is the **GWT (Given-When-Then)** structure from Behaviour-Driven Development (BDD). The concepts are identical — Given = Arrange, When = Act, Then = Assert — but the terminology is more business-readable and is used in tools like Cucumber or when writing tests that non-technical stakeholders will read.

```java
@Test
@DisplayName("Given a premium customer with order over $1000, when discount is calculated, then 30% discount is applied")
void given_premiumCustomer_when_largeOrder_then_thirtyPercentDiscount() {

    // Given — the world state
    Order order = new Order();
    order.setSubtotal(new BigDecimal("1200.00"));
    order.setCustomerTier(CustomerTier.PREMIUM);

    // When — the action
    BigDecimal discount = discountService.calculate(order);

    // Then — the expected outcome
    assertEquals(new BigDecimal("360.00"), discount,
        "Premium customers get 30% on orders over $1000");
}
```

---

## 📛 Test Naming Conventions

Good test names are the most underrated form of documentation. When a test fails in CI at 2am, the name is the first thing the on-call engineer sees. A good name tells you what was being tested, under what conditions, and what the expected outcome was — without opening the file.

The most widely used convention in Java is `methodName_condition_expectedResult`, sometimes written `should_expectedResult_when_condition`:

```java
// Convention: methodName_condition_expectedResult
void placeOrder_whenCourseFull_throwsNoSeatsAvailableException()
void findById_whenStudentExists_returnsStudentDTO()
void calculateGrade_withScoreAbove90_returnsA()
void sendEmail_whenSmtpDown_retriesThreeTimes()
void enroll_withDuplicateStudent_returnsFalseWithoutModifyingCourse()

// Convention: should_expectedResult_when_condition (BDD flavour)
void should_returnEmptyOptional_when_studentIdNotFound()
void should_throwValidationException_when_emailIsBlank()
void should_sendWelcomeEmail_when_newStudentRegisters()

// Bad names — useless when they appear in a test report
void test1()
void testCreate()
void testPlaceOrder()
void myTest()
```

Whichever convention you choose, what matters most is consistency across the codebase. Pick one style per project and enforce it in your code review checklist and static analysis rules.

---

## 📊 Test Coverage — Line, Branch & Mutation

Code coverage measures how much of your source code is exercised by your tests. Three metrics matter in practice:

**Line coverage** counts the percentage of executable lines that were executed by at least one test. It is the weakest metric because a line can be "covered" without its logic being truly tested — a single call through a method counts every line as covered even if only one branch was taken.

**Branch coverage** (also called decision coverage) counts whether both the `true` and `false` branches of every `if`, `switch`, and ternary expression have been exercised. This is a much stronger signal. A branch-covered test suite means your conditional logic has been tested for both outcomes.

**Mutation testing** (covered in the JaCoCo/Pitest section) is the gold standard. It artificially introduces bugs ("mutations") into your code — flipping `>` to `>=`, removing a `return` statement, inverting a boolean — and checks whether any test fails. If your tests don't catch a mutation, they aren't actually verifying that logic. This reveals whether your test assertions are meaningful, not just whether your code is executed.

```java
// This class has 100% line coverage if divide() is called once
// But branch coverage requires both the positive and zero divisor paths to be tested
public class SafeDivider {
    public double divide(double a, double b) {
        if (b == 0) {                    // Branch 1: b is zero
            throw new ArithmeticException("Cannot divide by zero");
        }
        return a / b;                    // Branch 2: b is non-zero
    }
}

// Line coverage: ONE test covering either branch gives 100% line coverage
// Branch coverage: requires TWO tests — one for each branch
@Test
void divide_withNonZeroDivisor_returnsResult() {
    assertEquals(5.0, divider.divide(10, 2)); // Covers the non-zero branch
}

@Test
void divide_withZeroDivisor_throwsException() {
    assertThrows(ArithmeticException.class, () -> divider.divide(10, 0)); // Covers the zero branch
}
// Now you have 100% branch coverage — both decisions were exercised
```

Coverage targets vary by team, but a reasonable general standard is 80% line coverage as a minimum, aiming for 70%+ branch coverage for business-critical code. The danger of coverage thresholds is "coverage theatre" — tests that run code but don't assert anything meaningful, written purely to hit a number. Mutation testing is the antidote because it reveals whether your assertions are substantive.

---

## 🚫 Common Anti-Patterns to Avoid

Understanding what makes a test *bad* is as important as understanding what makes it good, because bad tests are often worse than no tests — they give false confidence, slow the suite, and break constantly.

A **test that never fails** is a test with no assertions, or assertions that can never be false. `assertTrue(true)`, `assertNotNull(new Object())`, or a test body with no assertion at all are the most common forms. They bloat your coverage numbers without giving any protection.

A **test that tests too much** verifies many unrelated things in sequence. When it fails, you don't know which of the five things it was checking is broken. Split it into focused tests.

A **test that depends on another test** means the tests are not independent. If `testA` must run before `testB` for `testB` to pass, your test suite has an order dependency that will cause mysterious failures. Every test must set up its own complete preconditions in `@BeforeEach` and clean up after itself.

A **test that mocks the system under test** tests the mock rather than the real class. `@Mock private OrderService orderService` in `OrderServiceTest` is always a mistake. Mock the collaborators, not the subject.

A **test with no assertion** executes code but checks nothing. It confirms the code doesn't throw, but not that it produces the correct output. If a refactoring changes the return value from `Order` to `null`, a test with no assertion on the return value will still pass.

---

## ✅ Best Practices & Important Points

Aim for tests that are **FIRST**: Fast (unit tests in < 10ms, integration tests in < 1s), Independent (no shared mutable state between tests), Repeatable (same result every run, no dependence on time, random numbers, or external services), Self-validating (pass or fail automatically — no human inspection required), and Timely (written before or alongside the code, not months later).

Treat your test code with the same quality standards as production code. Poorly factored tests are just as hard to maintain as poorly factored production code. Extract common setup into `@BeforeEach`. Use helper methods for building test data. Apply the same naming discipline you apply to production methods.

When a test fails, fix it immediately. A practice of disabling or skipping failing tests to "deal with later" creates a culture where the test suite is not trusted, which means developers stop running it, which means the suite stops providing value. Every `@Disabled` annotation should have a corresponding ticket number and a deadline.

Write tests before fixing bugs. When you find a bug, write a test that reproduces it (a "regression test") before you fix it. This serves two purposes: it confirms your understanding of the bug, and it permanently prevents the bug from returning undetected.

---

## 🔑 Key Summary

```
Test Pyramid         → Many unit tests, moderate integration tests, few E2E tests
Unit test            → One class, all collaborators mocked, no I/O, milliseconds to run
Integration test     → Multiple real components, real DB/infra (Testcontainers), slower
E2E test             → Full stack, HTTP in → DB out, highest confidence, slowest
AAA Pattern          → Arrange (setup) → Act (one operation) → Assert (verify outcome)
BDD / GWT            → Given (state) → When (action) → Then (outcome) — same idea, business language
Line coverage        → % of executable lines run by tests; weakest metric
Branch coverage      → Both true/false branches covered; stronger metric
Mutation testing     → Artificially introduce bugs; if tests don't fail, assertions are too weak
Good test names      → methodName_condition_expectedResult or should_result_when_condition
Test independence    → Each test sets up its own state; no test depends on another test running first
FIRST principles     → Fast, Independent, Repeatable, Self-validating, Timely
Coverage theatre     → Tests with no meaningful assertions that only exist to inflate the coverage number
```
