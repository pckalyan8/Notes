# 📘 12.2 — Mockito: Mocking, Stubbing & Verification

Mockito is the most widely used mocking framework in the Java ecosystem. Its purpose is deceptively simple: it lets you replace real objects with fakes that you control completely, so that your unit tests remain isolated from the complexity of the real world. Understanding Mockito deeply means understanding not just the API, but *why* you need mocks at all and when using them causes more harm than good.

---

## 🧠 Why Mocks Exist — The Isolation Problem

A unit test has one goal: to verify that a single class behaves correctly in isolation. The moment your test reaches outside that class — to a database, a network call, the filesystem, or even just another class whose own tests might be broken — you have introduced a dependency that can cause your test to fail for a reason unrelated to the class you are testing.

Consider testing `OrderService.placeOrder()`. This method calls `paymentService.charge()`, `emailService.sendConfirmation()`, `inventoryService.deductStock()`, and `orderRepository.save()`. In a real test that uses the actual implementations of all these collaborators, an `OrderService` test would fail if the email server is down, if the database schema changed, or if `InventoryService` has a bug. None of those failures tell you anything about whether `OrderService.placeOrder()` is correct.

Mockito solves this by letting you replace `PaymentService`, `EmailService`, `InventoryService`, and `OrderRepository` with mocks — objects that look and feel like the real thing (they implement the same interface or extend the same class) but whose behaviour you control completely. A mock's `charge()` method does exactly what you tell it to do in the test and nothing more.

---

## 📦 Maven Setup

```xml
<!-- Already included in spring-boot-starter-test. For standalone use: -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>
</dependency>

<!-- For mocking static methods and constructors (Mockito 3.4+) -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-inline</artifactId>
    <version>5.2.0</version>
    <scope>test</scope>
</dependency>
```

---

## 🏗️ Creating Mocks — Two Approaches

Mockito offers two ways to create mocks. The programmatic approach gives you full control inside the test method. The annotation approach is more concise and is the preferred style in practice.

```java
// ── Approach 1: Programmatic (no annotations required) ──
class ProgrammaticMockingTest {

    @Test
    void placeOrder_chargesCustomerAndSendsEmail() {
        // Create mock objects — these are fake implementations generated at runtime
        PaymentService paymentService   = Mockito.mock(PaymentService.class);
        EmailService   emailService     = Mockito.mock(EmailService.class);
        OrderRepository orderRepository = Mockito.mock(OrderRepository.class);

        // Build the class under test, injecting the mocks manually
        OrderService orderService = new OrderService(paymentService, emailService, orderRepository);

        // ... the rest of the test
    }
}

// ── Approach 2: Annotations (preferred — less noise, clearer intent) ──
@ExtendWith(MockitoExtension.class) // Activates @Mock, @InjectMocks, @Spy, @Captor processing
class AnnotationMockingTest {

    @Mock
    private PaymentService paymentService;

    @Mock
    private EmailService emailService;

    @Mock
    private OrderRepository orderRepository;

    // @InjectMocks creates the real OrderService and injects the three mocks above into it
    // Injection preference order: constructor > setter > field
    @InjectMocks
    private OrderService orderService;

    @Test
    void placeOrder_chargesCustomerAndSendsEmail() {
        // No setup boilerplate needed — mocks are ready to use
    }
}
```

When you use `@ExtendWith(MockitoExtension.class)`, Mockito resets all mocks before each test automatically, so no `@BeforeEach` cleanup is needed. This is the main practical advantage over creating mocks programmatically inside each test method.

---

## 🎭 Stubs — Controlling What Mocks Return

A stub tells a mock what to return (or throw) when a specific method is called with specific arguments. Without a stub, calling any method on a mock returns the default "zero value" for its return type: `null` for objects, `0` for numeric types, `false` for booleans, and an empty collection for collection types. Stubbing is what makes mocks useful.

### when().thenReturn() — The Standard Stub

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock private PaymentService   paymentService;
    @Mock private EmailService     emailService;
    @Mock private CourseRepository courseRepository;
    @InjectMocks private OrderService orderService;

    @Test
    void placeOrder_whenPaymentSucceeds_savesOrderAndSendsEmail() {
        // ── Arrange: define what the mocks return ──

        // Simple stub: when findById is called with 101L, return this course
        Course javaCourse = new Course(101L, "Java Mastery", 299.99, 10);
        when(courseRepository.findById(101L)).thenReturn(Optional.of(javaCourse));

        // Stub a void method — paymentService.charge() returns void
        // doNothing() is the explicit no-op; it is actually the default for void methods,
        // but writing it makes the intention clear in the test
        doNothing().when(paymentService).charge(any(Order.class));

        // Stub a method to return different values on successive calls
        when(courseRepository.getAvailableSeats(101L))
            .thenReturn(10) // First call returns 10
            .thenReturn(9)  // Second call returns 9
            .thenReturn(8); // Third and subsequent calls return 8

        // ── Act ──
        Order result = orderService.placeOrder(new PlaceOrderRequest(1L, 101L));

        // ── Assert ──
        assertNotNull(result);
        assertEquals(101L, result.getCourseId());
    }

    @Test
    void placeOrder_whenPaymentFails_throwsPaymentException() {
        Course javaCourse = new Course(101L, "Java Mastery", 299.99, 10);
        when(courseRepository.findById(101L)).thenReturn(Optional.of(javaCourse));

        // Stub to throw an exception — simulates the payment gateway being down
        when(paymentService.charge(any(Order.class)))
            .thenThrow(new PaymentGatewayException("Gateway timeout"));

        assertThrows(PaymentException.class,
            () -> orderService.placeOrder(new PlaceOrderRequest(1L, 101L)));
    }
}
```

### when() vs doReturn() — A Critical Distinction

There are two syntaxes for stubbing: `when(mock.method()).thenReturn(value)` and `doReturn(value).when(mock).method()`. For regular mocks they are equivalent, but the `do*` style is **required** for `void` methods and for `@Spy` objects (because `when(spy.method())` actually calls the real method before the stub is applied).

```java
@Test
void demonstrateDoReturnUsage() {
    // ── With a regular mock: both syntaxes work ──
    when(orderRepository.save(any())).thenReturn(savedOrder); // Fine
    doReturn(savedOrder).when(orderRepository).save(any());   // Also fine

    // ── With a spy: MUST use doReturn to avoid calling the real method ──
    OrderService realService = new OrderService();
    OrderService spyService  = Mockito.spy(realService);

    // ❌ WRONG: when(spyService.expensiveCalculation()) calls the REAL method
    // before the stub is registered — may throw or have side effects
    // when(spyService.expensiveCalculation()).thenReturn(42);

    // ✅ CORRECT: doReturn registers the stub WITHOUT calling the real method
    doReturn(42).when(spyService).expensiveCalculation();

    // ── For void methods: cannot use when() at all ──
    // when(emailService.send(any())).thenDoNothing();  // Won't compile
    doNothing().when(emailService).send(any());          // Correct
    doThrow(new EmailException()).when(emailService).send(any()); // Or throw
}
```

### thenAnswer() — Dynamic Return Values

Sometimes the return value needs to depend on the arguments that were passed in. `thenAnswer()` gives you access to the actual arguments at call time.

```java
@Test
void save_assignsDatabaseId_toSavedEntity() {
    // When save() is called, simulate what a real database does:
    // assign a generated ID to the entity and return it
    when(studentRepository.save(any(Student.class))).thenAnswer(invocation -> {
        Student student = invocation.getArgument(0); // Get the first argument
        student.setId(42L); // Simulate DB-generated ID
        return student;
    });

    Student result = studentService.create(new CreateStudentRequest("Alice", "alice@ex.com"));

    assertNotNull(result.getId(), "Service should return the DB-assigned ID");
    assertEquals(42L, result.getId());
    assertEquals("Alice", result.getName());
}

@Test
void findAll_returnsFilteredList_basedOnArgument() {
    // Return different results based on which status was passed in
    when(studentRepository.findByStatus(any(Status.class))).thenAnswer(invocation -> {
        Status status = invocation.getArgument(0);
        return switch (status) {
            case ENROLLED  -> List.of(alice, bob);
            case GRADUATED -> List.of(carol);
            case WITHDRAWN -> Collections.emptyList();
        };
    });

    List<Student> enrolled = studentService.findByStatus(Status.ENROLLED);
    assertEquals(2, enrolled.size());
}
```

---

## 🔍 Argument Matchers — Flexible Stubbing and Verification

By default, Mockito matches method calls using `equals()` on the arguments. Argument matchers let you match calls more flexibly — any value, any string, any object of a given type, or a value satisfying a custom predicate.

```java
import static org.mockito.ArgumentMatchers.*;

@Test
void demonstrateArgumentMatchers() {
    // any() — matches any argument including null
    when(repository.findById(any())).thenReturn(Optional.of(student));

    // any(Class) — matches any non-null argument of that type
    when(repository.save(any(Student.class))).thenReturn(savedStudent);

    // anyString() / anyInt() / anyLong() / anyDouble() — typed matchers
    when(service.findByName(anyString())).thenReturn(List.of(student));

    // eq() — matches a specific value (required when mixing matchers)
    when(service.findScoreForStudent(eq(1L), anyString())).thenReturn(95.5);
    // ⚠️ CRITICAL RULE: if you use ANY argument matcher, ALL arguments in the same
    // method call must use matchers. Mixing eq() with a literal value won't compile:
    // when(service.findScoreForStudent(1L, anyString())) // WRONG — must wrap 1L in eq()

    // Matchers for specific conditions
    when(repository.findByScoreGreaterThan(doubleThat(score -> score > 0)))
        .thenReturn(List.of(student));

    when(service.process(argThat(order ->
        order.getTotalPrice() > 100.0 && order.getCustomerId() != null)))
        .thenReturn(new ProcessResult("SUCCESS"));

    // isNull() / isNotNull() — null-specific matchers
    when(repository.findByDepartment(isNull())).thenReturn(Collections.emptyList());
    when(repository.findByDepartment(isNotNull())).thenReturn(List.of(student));

    // startsWith / endsWith / contains — string-specific
    when(service.findByEmail(startsWith("alice"))).thenReturn(Optional.of(alice));
}
```

---

## ✅ Verification — Proving Interactions Happened

Stubbing tells mocks what to return. Verification tells Mockito to check that specific methods were actually called (or not called) during the test. Both have their place — but overusing verification is a testing anti-pattern called "over-specification" (see Best Practices below).

```java
@Test
void placeOrder_shouldChargePaymentAndSendEmail() {
    when(courseRepository.findById(101L)).thenReturn(Optional.of(javaCourse));

    orderService.placeOrder(new PlaceOrderRequest(customerId, 101L));

    // Verify paymentService.charge() was called exactly once
    verify(paymentService, times(1)).charge(any(Order.class));

    // Verify emailService.sendConfirmation() was called exactly once
    verify(emailService, times(1)).sendConfirmation(any(Order.class));

    // Verify that orderRepository.delete() was NEVER called
    verify(orderRepository, never()).delete(any());

    // Other verification modes:
    verify(auditService, atLeastOnce()).log(anyString());     // At least 1 call
    verify(cacheService, atMost(3)).put(anyString(), any());  // At most 3 calls
    verify(service,      atLeast(2)).process(any());          // At least 2 calls

    // Verify no MORE interactions happened after the ones already verified
    // Use sparingly — it makes tests brittle when internal implementation changes
    verifyNoMoreInteractions(emailService);
}

@Test
void createStudent_shouldNeverSendEmailForAdminUsers() {
    // Verify that the email service was never touched at all during this operation
    adminService.createAdminUser(new CreateAdminRequest("sysadmin", "admin@company.com"));
    verifyNoInteractions(emailService);
}

// Verifying call ORDER using InOrder
@Test
void placeOrder_shouldChargeBeforeSendingEmail() {
    orderService.placeOrder(new PlaceOrderRequest(customerId, courseId));

    // InOrder verifies that methods were called in a specific sequence
    InOrder inOrder = Mockito.inOrder(paymentService, emailService);
    inOrder.verify(paymentService).charge(any(Order.class));     // Must happen first
    inOrder.verify(emailService).sendConfirmation(any(Order.class)); // Must happen after
}
```

---

## 📸 ArgumentCaptor — Inspecting What Was Passed

`ArgumentCaptor` captures the actual argument that was passed to a mock method so you can run detailed assertions on it. This is the bridge between verification (did the method get called?) and assertion (was it called with the right data?).

```java
@ExtendWith(MockitoExtension.class)
class NotificationServiceTest {

    @Mock private EmailClient emailClient;
    @InjectMocks private NotificationService notificationService;

    @Captor private ArgumentCaptor<EmailMessage> emailCaptor;

    @Test
    void sendWelcomeEmail_buildsCorrectEmailMessage() {
        Student newStudent = new Student(42L, "Alice", "alice@example.com");

        notificationService.sendWelcomeEmail(newStudent);

        // Capture the EmailMessage that was passed to emailClient.send()
        verify(emailClient).send(emailCaptor.capture());
        EmailMessage sentEmail = emailCaptor.getValue();

        // Now we can assert on every field of the captured argument
        assertEquals("alice@example.com", sentEmail.getTo());
        assertEquals("Welcome to the platform, Alice!", sentEmail.getSubject());
        assertTrue(sentEmail.getBody().contains("Hello, Alice"));
        assertTrue(sentEmail.getBody().contains("Get started"));
        assertEquals("noreply@platform.com", sentEmail.getFrom());
    }

    @Test
    void sendBulkEmails_capturesAllSentMessages() {
        List<Student> students = List.of(
            new Student(1L, "Alice", "alice@example.com"),
            new Student(2L, "Bob",   "bob@example.com")
        );

        notificationService.sendBulkWelcome(students);

        // Capture ALL calls to send() (getAllValues() returns a list)
        verify(emailClient, times(2)).send(emailCaptor.capture());
        List<EmailMessage> allEmails = emailCaptor.getAllValues();

        assertEquals(2, allEmails.size());
        assertThat(allEmails).extracting(EmailMessage::getTo)
            .containsExactlyInAnyOrder("alice@example.com", "bob@example.com");
    }
}
```

---

## 🕵️ @Spy — Partial Mocking

A spy wraps a real object. By default, all method calls go through to the real implementation. You can selectively stub individual methods while leaving the rest real. This is useful when you want to test a class that has one expensive or side-effecting method you want to stub out while keeping everything else real.

```java
@ExtendWith(MockitoExtension.class)
class ReportServiceTest {

    // @Spy creates a real ReportService AND wraps it in a spy proxy
    // The real object MUST be instantiable (needs a no-arg constructor or
    // the @Spy annotation field must be initialized inline)
    @Spy
    private ReportService reportService = new ReportService(new DataFormatter());

    @Mock
    private ExternalDataSource externalDataSource;

    @Test
    void generateReport_usesRealFormattingButStubbedDataFetch() {
        // Stub ONLY the method that calls the external service
        // All other methods on reportService use their REAL implementation
        doReturn(List.of(new DataRow("Alice", 95.5))).when(externalDataSource).fetchData();

        // But the report formatting logic is real — we're testing that too
        String report = reportService.generate(externalDataSource);

        assertTrue(report.contains("Alice"),   "Real formatting should include the name");
        assertTrue(report.contains("95.5"),    "Real formatting should include the score");
        assertTrue(report.startsWith("REPORT"),"Real formatting should add the header");
    }
}
```

Use spies sparingly. If you frequently need to spy on a class and stub half its methods, that is usually a design smell: the class has too many responsibilities and should be broken apart.

---

## 🔒 Mocking Static Methods (Mockito 5+)

Static method mocking is available via `mockStatic()`. It is deliberately verbose to discourage overuse — if you are mocking static methods frequently, it usually means your design has untestable global dependencies that should be refactored.

```java
@Test
void generateOrderId_usesUUID() {
    // mockStatic() must be used in a try-with-resources block
    // The mock is automatically closed when the block exits
    try (MockedStatic<UUID> mockedUuid = Mockito.mockStatic(UUID.class)) {
        UUID fixedUuid = UUID.fromString("00000000-0000-0000-0000-000000000042");
        mockedUuid.when(UUID::randomUUID).thenReturn(fixedUuid);

        String orderId = orderService.generateOrderId();

        assertEquals("ORD-00000000-0000-0000-0000-000000000042", orderId);
        mockedUuid.verify(UUID::randomUUID, times(1));
    }
    // Outside the try block, UUID.randomUUID() works normally again
}

// Mocking static methods on your own utility classes
@Test
void process_usesCurrentTimestamp() {
    LocalDateTime fixedTime = LocalDateTime.of(2024, 6, 15, 10, 30, 0);

    try (MockedStatic<TimeUtil> mockedTime = Mockito.mockStatic(TimeUtil.class)) {
        mockedTime.when(TimeUtil::now).thenReturn(fixedTime);

        ProcessResult result = processor.process(new ProcessRequest("data"));

        assertEquals(fixedTime, result.getProcessedAt());
    }
}
```

---

## 🏗️ Mocking Constructors

Sometimes legacy code creates collaborators with `new` inside methods, making injection impossible. `MockedConstruction` intercepts constructor calls.

```java
@Test
void process_withLegacyCodeThatUsesNew() {
    // When new LegacyProcessor() is called anywhere in this try block,
    // return our controlled mock instead of the real object
    try (MockedConstruction<LegacyProcessor> mocked =
             Mockito.mockConstruction(LegacyProcessor.class,
                 (mock, context) -> {
                     // context contains the constructor arguments
                     when(mock.process(anyString())).thenReturn("MOCKED_RESULT");
                 })) {

        String result = serviceUnderTest.processData("input");

        assertEquals("MOCKED_RESULT", result);

        // Can also verify calls to the constructed mock
        LegacyProcessor constructedMock = mocked.constructed().get(0);
        verify(constructedMock).process("input");
    }
}
```

---

## ✅ Best Practices & Important Points

**Mock interfaces, not concrete classes.** Mocking an interface is clean and design-appropriate — it makes your production code depend on abstractions rather than implementations (the D in SOLID). Mocking concrete classes causes CGLIB subclassing, which fails on `final` classes and methods and is a sign that your design could be improved.

**Avoid over-verification — verify behaviour, not implementation.** A test that verifies every single method call on every mock is testing the implementation of `OrderService`, not its behaviour. If you later refactor the internals (e.g., combine two repository calls into one efficient query), all those over-specified tests break even though the behaviour is identical. Verify only the interactions that matter for correctness: that the customer was charged, that the email was sent.

**Do not mock the class under test.** If you are mocking `OrderService` in `OrderServiceTest`, you are testing the mock, not the service. The class under test should always be a real instance created with `@InjectMocks` or `new`.

**Prefer constructor injection to make `@InjectMocks` reliable.** Mockito's `@InjectMocks` tries constructor injection first, then setter injection, then field injection. Constructor injection is the most predictable and explicit — if a required dependency is missing, the constructor throws, giving you an immediate helpful error instead of a silent null.

**Reset mocks between tests by using `@ExtendWith(MockitoExtension.class)`.** Without the extension, mock state (stubbing, recorded interactions) carries over between tests in the same class, causing mysterious test interdependence. The extension resets everything automatically.

**Use `ArgumentCaptor` when the correctness of the arguments matters.** If you only need to verify that a method was called, `verify(mock).method(any())` is sufficient. Only reach for `ArgumentCaptor` when you genuinely need to assert on what was passed — for example, verifying the exact fields of an email that was sent.

---

## 🔑 Key Summary

```
@Mock                    → Creates a fake implementation; all methods return zero/null by default
@Spy                     → Wraps a real object; real methods are called unless stubbed
@InjectMocks             → Creates the real class under test and injects @Mocks into it
@Captor                  → Captures the argument passed to a mock for later assertion
@ExtendWith(MockitoExtension) → Activates annotations; resets mocks between tests
when(mock.method()).thenReturn(value) → Stub a method to return a specific value
when(mock.method()).thenThrow(ex)     → Stub a method to throw an exception
doReturn(value).when(spy).method()   → Required for spies and void methods
thenAnswer(invocation -> ...)        → Dynamic return value based on actual arguments
verify(mock).method(...)             → Assert the method was called
verify(mock, times(n))               → Assert exactly N calls
verify(mock, never())                → Assert the method was NEVER called
verifyNoInteractions(mock)           → Assert no methods were called on the mock
InOrder                              → Verify calls happened in a specific sequence
ArgumentCaptor.capture()             → Capture the argument for assertion
any(), eq(), anyString()             → Argument matchers for flexible matching
mockStatic(Class)                    → Mock static methods (use sparingly)
MockedConstruction                   → Intercept constructor calls (legacy code only)
```
