# 🧪 11.7 — Spring Testing: JUnit 5, MockMvc, Test Slices & Testcontainers

Testing is not an afterthought in professional Java development — it is the activity that gives you the confidence to change code without fear. Spring Boot provides one of the richest testing ecosystems of any framework: sliced contexts that load only the layers you need, auto-configured mock objects, built-in `MockMvc` for testing HTTP without a running server, and seamless Testcontainers integration for spinning up real databases in Docker.

---

## 🧠 The Testing Pyramid — Know What You're Building

Before writing a single test, you need a mental model of what you're testing and why. The testing pyramid describes three levels:

**Unit tests** sit at the base and are the most numerous. They test a single class in complete isolation — all collaborators are replaced with mock objects. They run in milliseconds, require no Spring context, and give precise feedback when something breaks. A service class with ten methods should have tens of unit tests.

**Integration tests** sit in the middle. They test the interaction between two or more real components: your service with your real repository, or your repository with a real (test) database. They require a Spring context or a database container to run. They are slower but catch problems that unit tests miss — like incorrect SQL queries or misconfigured Spring beans.

**End-to-end tests** sit at the top and are the fewest. They test the entire system from HTTP request to database response. They are the slowest and most fragile — a change anywhere in the stack can break them. Reserve these for critical user flows.

A healthy test suite has many fast unit tests, a moderate number of integration tests, and a small number of end-to-end tests. The spring-boot-starter-test brings all the tools you need for all three levels.

---

## 📦 What's in spring-boot-starter-test

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

This one dependency bundles JUnit 5 (Jupiter) for test structure and assertions, Mockito for creating mock objects, AssertJ for fluent, readable assertions, MockMvc for testing Spring MVC controllers without a running server, `@SpringBootTest` for full integration context loading, test slices (`@WebMvcTest`, `@DataJpaTest`, etc.) for partial context loading, and Hamcrest matchers. Everything you need is in one place.

---

## 🧩 Part 1 — Unit Testing with JUnit 5 and Mockito

Unit tests should not start a Spring context. They instantiate the class under test directly with mock collaborators. This makes them extremely fast — typically running in under 10ms.

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.*;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

// @ExtendWith(MockitoExtension.class) activates Mockito annotations without Spring
@ExtendWith(MockitoExtension.class)
class StudentServiceTest {

    // @Mock creates a Mockito mock — a fake implementation that records all calls
    @Mock
    private StudentRepository studentRepository;

    @Mock
    private EmailService emailService;

    // @InjectMocks creates the real class and injects the mocks into it
    // This simulates Spring's DI without starting Spring
    @InjectMocks
    private StudentService studentService;

    // @Captor captures the argument passed to a mock method, so you can assert on it
    @Captor
    private ArgumentCaptor<Student> studentCaptor;

    @BeforeEach
    void setUp() {
        // Runs before each test; useful for setting up shared state
        // With @ExtendWith(MockitoExtension.class) you rarely need this for mock setup
    }

    // ── Testing happy path ──
    @Test
    @DisplayName("findById should return StudentDTO when student exists")
    void findById_whenStudentExists_returnsStudentDTO() {
        // ── Arrange ──
        // Stub the repository: when findById(1L) is called, return this student
        Student alice = new Student(1L, "Alice", "alice@example.com", 95.5);
        when(studentRepository.findById(1L)).thenReturn(Optional.of(alice));

        // ── Act ──
        Optional<StudentDTO> result = studentService.findById(1L);

        // ── Assert (using AssertJ's fluent API) ──
        assertThat(result).isPresent();
        assertThat(result.get().id()).isEqualTo(1L);
        assertThat(result.get().name()).isEqualTo("Alice");
        assertThat(result.get().email()).isEqualTo("alice@example.com");

        // Verify the repository was called exactly once with the correct argument
        verify(studentRepository, times(1)).findById(1L);
        // Verify emailService was NEVER called (it shouldn't be for a simple read)
        verifyNoInteractions(emailService);
    }

    // ── Testing the not-found case ──
    @Test
    @DisplayName("findById should return empty Optional when student does not exist")
    void findById_whenStudentNotFound_returnsEmpty() {
        when(studentRepository.findById(anyLong())).thenReturn(Optional.empty());

        Optional<StudentDTO> result = studentService.findById(999L);

        assertThat(result).isEmpty();
    }

    // ── Testing that an exception is thrown ──
    @Test
    @DisplayName("enrollStudent should throw NoSeatsAvailableException when course is full")
    void enrollStudent_whenCourseIsFull_throwsException() {
        Course fullCourse = new Course(1L, "Java 101", 0); // 0 available seats
        Student student   = new Student(1L, "Alice", "alice@example.com", 95.5);
        when(studentRepository.findById(1L)).thenReturn(Optional.of(student));
        when(courseRepository.findById(1L)).thenReturn(Optional.of(fullCourse));

        // assertThatThrownBy: verifies the exact exception and message
        assertThatThrownBy(() -> studentService.enrollStudent(1L, 1L))
            .isInstanceOf(NoSeatsAvailableException.class)
            .hasMessageContaining("Java 101");
    }

    // ── Verifying what was passed to a mock using ArgumentCaptor ──
    @Test
    @DisplayName("createStudent should send welcome email with correct recipient")
    void createStudent_shouldSendWelcomeEmailToNewStudent() {
        CreateStudentRequest request = new CreateStudentRequest("Bob", "bob@example.com");
        when(studentRepository.save(any(Student.class))).thenAnswer(invocation -> {
            Student s = invocation.getArgument(0);
            s.setId(42L); // Simulate ID assignment by DB
            return s;
        });

        studentService.createStudent(request);

        // Capture what was passed to emailService.sendWelcome()
        verify(emailService).sendWelcome(studentCaptor.capture());
        Student capturedStudent = studentCaptor.getValue();
        assertThat(capturedStudent.getEmail()).isEqualTo("bob@example.com");
        assertThat(capturedStudent.getId()).isEqualTo(42L); // Verify ID was set before emailing
    }

    // ── Parameterized test — same test logic, multiple inputs ──
    @ParameterizedTest(name = "score {0} should have grade {1}")
    @CsvSource({
        "95.0, A",
        "85.0, B",
        "75.0, C",
        "65.0, D",
        "55.0, F"
    })
    void calculateGrade_shouldReturnCorrectGrade(double score, String expectedGrade) {
        String grade = studentService.calculateGrade(score);
        assertThat(grade).isEqualTo(expectedGrade);
    }

    // ── Testing exceptions with specific message ──
    @Test
    void deleteStudent_whenNotFound_throwsResourceNotFoundException() {
        when(studentRepository.findById(999L)).thenReturn(Optional.empty());

        assertThatExceptionOfType(ResourceNotFoundException.class)
            .isThrownBy(() -> studentService.deleteStudent(999L))
            .withMessage("Student not found: 999");
    }
}
```

---

## 🌐 Part 2 — Controller Tests with @WebMvcTest

`@WebMvcTest` loads only the web layer of your Spring context — controllers, `@ControllerAdvice`, filters, and `WebMvcConfigurer` beans. It does NOT load services, repositories, or the database. Service collaborators are `@MockBean`s (Mockito mocks registered as Spring beans). This is the right tool for testing your HTTP API: routing, request/response serialization, validation, and error handling.

```java
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.hamcrest.Matchers.*;

@WebMvcTest(StudentController.class) // Loads ONLY StudentController and the web layer
class StudentControllerTest {

    // MockMvc is auto-configured by @WebMvcTest — lets you simulate HTTP requests
    @Autowired
    private MockMvc mockMvc;

    // ObjectMapper is available for converting objects to/from JSON
    @Autowired
    private ObjectMapper objectMapper;

    // @MockBean registers a mock as a Spring bean — replaces the real service in the context
    @MockBean
    private StudentService studentService;

    // ── Test a GET endpoint ──
    @Test
    @DisplayName("GET /api/students/{id} should return 200 and student JSON when found")
    void getStudent_whenFound_returns200WithJson() throws Exception {
        StudentDTO alice = new StudentDTO(1L, "Alice", "alice@example.com", 95.5);
        when(studentService.findById(1L)).thenReturn(Optional.of(alice));

        mockMvc.perform(get("/api/students/1")
                    .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())                          // HTTP 200
            .andExpect(jsonPath("$.id").value(1L))               // JSON field assertions
            .andExpect(jsonPath("$.name").value("Alice"))
            .andExpect(jsonPath("$.email").value("alice@example.com"))
            .andExpect(jsonPath("$.score").value(95.5))
            .andDo(print()); // Prints request and response to console — helpful for debugging
    }

    // ── Test 404 response ──
    @Test
    @DisplayName("GET /api/students/{id} should return 404 when not found")
    void getStudent_whenNotFound_returns404() throws Exception {
        when(studentService.findById(999L)).thenReturn(Optional.empty());

        mockMvc.perform(get("/api/students/999"))
            .andExpect(status().isNotFound());
    }

    // ── Test a POST endpoint with a JSON body ──
    @Test
    @DisplayName("POST /api/students should return 201 with Location header when created")
    void createStudent_withValidBody_returns201() throws Exception {
        CreateStudentRequest request = new CreateStudentRequest("Bob", "bob@example.com");
        StudentDTO created = new StudentDTO(42L, "Bob", "bob@example.com", null);
        when(studentService.create(any(CreateStudentRequest.class))).thenReturn(created);

        mockMvc.perform(post("/api/students")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(request))) // Serialize to JSON
            .andExpect(status().isCreated())                              // HTTP 201
            .andExpect(header().string("Location", containsString("/api/students/42")))
            .andExpect(jsonPath("$.id").value(42L));
    }

    // ── Test request validation ──
    @Test
    @DisplayName("POST /api/students should return 400 when name is blank")
    void createStudent_withBlankName_returns400() throws Exception {
        CreateStudentRequest invalidRequest = new CreateStudentRequest("", "bob@example.com");

        mockMvc.perform(post("/api/students")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(invalidRequest)))
            .andExpect(status().isBadRequest())                              // HTTP 400
            .andExpect(jsonPath("$.validationErrors.name").exists())        // Validation error map
            .andExpect(jsonPath("$.validationErrors.name").value("Name is required"));
    }

    // ── Test a list endpoint with pagination parameters ──
    @Test
    @DisplayName("GET /api/students should support pagination")
    void findAll_withPaginationParams_returnsPagedResults() throws Exception {
        List<StudentDTO> students = List.of(
            new StudentDTO(1L, "Alice", "alice@example.com", 95.5),
            new StudentDTO(2L, "Bob",   "bob@example.com",   82.0)
        );
        Page<StudentDTO> page = new PageImpl<>(students, PageRequest.of(0, 10), 2);
        when(studentService.findAll(any(Pageable.class))).thenReturn(page);

        mockMvc.perform(get("/api/students").param("page", "0").param("size", "10"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.content", hasSize(2)))
            .andExpect(jsonPath("$.totalElements").value(2))
            .andExpect(jsonPath("$.content[0].name").value("Alice"));
    }

    // ── Test with security (when using @WithMockUser) ──
    @Test
    @WithMockUser(username = "admin@example.com", roles = {"ADMIN"})
    @DisplayName("DELETE /api/students/{id} should return 204 for ADMIN")
    void deleteStudent_asAdmin_returns204() throws Exception {
        doNothing().when(studentService).delete(1L);

        mockMvc.perform(delete("/api/students/1"))
            .andExpect(status().isNoContent());
    }

    @Test
    @WithMockUser(username = "student@example.com", roles = {"STUDENT"})
    @DisplayName("DELETE /api/students/{id} should return 403 for non-ADMIN")
    void deleteStudent_asStudent_returns403() throws Exception {
        mockMvc.perform(delete("/api/students/1"))
            .andExpect(status().isForbidden());
    }
}
```

---

## 💾 Part 3 — Repository Tests with @DataJpaTest

`@DataJpaTest` loads only the JPA layer — entities, repositories, and the DataSource. It uses an in-memory H2 database by default (or you can point it at a real database with Testcontainers). Transactions roll back automatically after each test, keeping the database clean between tests.

```java
@DataJpaTest
class StudentRepositoryTest {

    @Autowired
    private StudentRepository studentRepository;

    @Autowired
    private TestEntityManager entityManager; // Helper for JPA setup/teardown in tests

    @BeforeEach
    void setUp() {
        // Persist test data using TestEntityManager (not the repository under test)
        entityManager.persistAndFlush(new Student("Alice", "alice@example.com", 95.5, Status.ENROLLED));
        entityManager.persistAndFlush(new Student("Bob",   "bob@example.com",   72.0, Status.ENROLLED));
        entityManager.persistAndFlush(new Student("Carol", "carol@example.com", 88.0, Status.GRADUATED));
    }

    @Test
    @DisplayName("findByStatus should return only enrolled students")
    void findByStatus_shouldReturnMatchingStudents() {
        List<Student> enrolled = studentRepository.findByStatus(Status.ENROLLED);

        assertThat(enrolled).hasSize(2);
        assertThat(enrolled).extracting(Student::getName)
            .containsExactlyInAnyOrder("Alice", "Bob");
    }

    @Test
    @DisplayName("findByScoreGreaterThan should return students above the threshold")
    void findByScoreGreaterThan_shouldFilterCorrectly() {
        List<Student> highScorers = studentRepository.findByScoreGreaterThan(85.0);

        assertThat(highScorers).hasSize(2);
        assertThat(highScorers).extracting(Student::getName)
            .containsExactlyInAnyOrder("Alice", "Carol");
    }

    @Test
    @DisplayName("existsByEmail should return true for existing email")
    void existsByEmail_shouldReturnTrueForExistingEmail() {
        boolean exists = studentRepository.existsByEmail("alice@example.com");
        assertThat(exists).isTrue();
    }

    @Test
    @DisplayName("existsByEmail should return false for non-existent email")
    void existsByEmail_shouldReturnFalseForMissingEmail() {
        boolean exists = studentRepository.existsByEmail("nobody@example.com");
        assertThat(exists).isFalse();
    }

    @Test
    @DisplayName("custom @Query method should compute average score correctly")
    void findAverageScore_shouldReturnCorrectAverage() {
        Double average = studentRepository.findAverageScore(Status.ENROLLED);
        // Average of 95.5 and 72.0 = 83.75
        assertThat(average).isEqualByComparingTo(83.75);
    }
}
```

---

## 🐳 Part 4 — Real Databases with Testcontainers

H2 is convenient but it has subtle differences from PostgreSQL or MySQL — different SQL syntax, different behavior for certain operations. Testcontainers spins up the actual database in a Docker container during your test and tears it down when done. Your tests run against the real thing.

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
```

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // Don't use H2
@Testcontainers // Activates @Container field management
class StudentRepositoryIntegrationTest {

    // Testcontainers starts this PostgreSQL container before any tests run
    // and stops it when all tests in this class finish
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    // Tell Spring to use the Testcontainer's dynamic URL, not the application.properties one
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private StudentRepository studentRepository;

    @Test
    @DisplayName("save and findById should work against real PostgreSQL")
    void saveAndFind_withRealPostgres_shouldWork() {
        Student student = new Student("Alice", "alice@example.com", 95.5, Status.ENROLLED);
        Student saved   = studentRepository.save(student);

        assertThat(saved.getId()).isNotNull(); // PostgreSQL generated the ID

        Optional<Student> found = studentRepository.findById(saved.getId());
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Alice");
    }
}
```

---

## 🏗️ Part 5 — Full Integration Tests with @SpringBootTest

`@SpringBootTest` loads the complete Spring application context — all beans, the real database connection, security, etc. This is the closest to running the actual application. Use it sparingly because loading the full context takes seconds. Combine with `TestRestTemplate` or `WebTestClient` for actual HTTP calls.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
// RANDOM_PORT starts the real embedded Tomcat on a random port — avoids port conflicts
@Testcontainers
class StudentApiIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void setDatasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate; // Makes real HTTP calls to the running server

    @Autowired
    private StudentRepository studentRepository;

    @BeforeEach
    void setUp() {
        studentRepository.deleteAll(); // Clean database before each test
    }

    @Test
    @DisplayName("Full flow: create student via POST, then retrieve via GET")
    void createAndRetrieveStudent_fullStack() {
        // Create
        CreateStudentRequest request = new CreateStudentRequest("Alice", "alice@example.com");
        ResponseEntity<StudentDTO> createResponse = restTemplate.postForEntity(
            "/api/students", request, StudentDTO.class
        );
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(createResponse.getBody()).isNotNull();
        Long id = createResponse.getBody().id();

        // Retrieve
        ResponseEntity<StudentDTO> getResponse = restTemplate.getForEntity(
            "/api/students/" + id, StudentDTO.class
        );
        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(getResponse.getBody().name()).isEqualTo("Alice");
    }
}
```

---

## 🔑 Test Slices Reference

Spring Boot provides purpose-built test slices that load only the beans needed for a specific layer. Using slices instead of `@SpringBootTest` makes your tests faster and more focused.

```
@WebMvcTest         → Web layer only (controllers, filters, MVC config); mocks everything else
@DataJpaTest        → JPA layer (entities, repositories, DataSource); uses H2 by default
@DataMongoTest      → MongoDB layer only
@DataRedisTest      → Redis layer only
@JsonTest           → JSON serialization/deserialization only (Jackson config)
@RestClientTest     → RestTemplate/WebClient HTTP client testing
@WebFluxTest        → Reactive WebFlux layer only
@SpringBootTest     → Full context; use only for full end-to-end integration tests
```

---

## ✅ Best Practices & Important Points

**Name your tests using the "method_condition_expectedResult" or BDD Given-When-Then style.** `findById_whenStudentExists_returnsStudentDTO` is clear, specific, and self-documenting. A failing test with this name immediately tells you exactly what broke without reading the test body.

**Prefer `@MockBean` in `@WebMvcTest` over manual mock setup.** `@MockBean` integrates Mockito mocks with the Spring context. The mock is reset automatically between tests and is available for injection into the application context, which is necessary when the controller being tested gets its dependencies injected by Spring.

**Use `@Transactional` on `@DataJpaTest` tests to auto-rollback.** `@DataJpaTest` annotates each test with `@Transactional` by default — meaning each test runs in a transaction that is automatically rolled back at the end. This keeps each test isolated without you needing to manually clean up data. If you need to test that data persisted after a commit, annotate the specific test method with `@Commit`.

**Testcontainers `@Container` field should be `static` and use `@Testcontainers`.** A `static` container is started once for the entire test class, rather than being created and destroyed for each test method — which would be extremely slow. The `@Testcontainers` annotation on the class manages the container lifecycle automatically.

**Avoid loading the full `@SpringBootTest` context for most tests.** The full context is convenient but slow. A large application may take 30–60 seconds to load its full context. With test slices, you pay only for what you need. Use `@SpringBootTest` only for true end-to-end integration tests that need the full stack.

**AssertJ is significantly more readable than JUnit's built-in assertions.** Compare `assertEquals(expected, actual)` with `assertThat(actual).isEqualTo(expected)`. AssertJ reads as English, provides better failure messages, supports fluent chaining, and has rich collection assertion methods like `containsExactlyInAnyOrder`, `extracting`, `filteredOn`, and dozens of others.

---

## 🔑 Key Summary

```
@ExtendWith(MockitoExtension.class) → Pure unit tests without Spring; fast; no context loading
@Mock                               → Creates a Mockito mock object for a collaborator
@InjectMocks                        → Creates the class under test and injects @Mocks into it
@MockBean                           → Registers a Mockito mock as a Spring bean (for @WebMvcTest)
@Captor                             → Captures arguments passed to mock methods for assertions
@WebMvcTest                         → Loads web layer only; use MockMvc to simulate HTTP requests
MockMvc                             → Simulates HTTP requests in tests without a real server
@DataJpaTest                        → Loads JPA layer only; auto-rollback; H2 by default
TestEntityManager                   → Test helper for persisting setup data in @DataJpaTest
Testcontainers                      → Spins up real Docker containers (PostgreSQL, etc.) in tests
@SpringBootTest(RANDOM_PORT)        → Full context + real embedded server; use for end-to-end tests
@DynamicPropertySource              → Overrides Spring properties dynamically (e.g., Testcontainers URL)
@WithMockUser                       → Provides a mock authenticated user in @WebMvcTest security tests
```
