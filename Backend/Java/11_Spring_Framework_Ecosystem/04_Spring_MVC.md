# 📒 11.4 — Spring MVC / Spring Web: Controllers, REST APIs & Exception Handling

Spring MVC is the web layer of a Spring Boot application. It is responsible for receiving HTTP requests, routing them to the correct handler method, converting request/response bodies, validating input, and sending back responses. Understanding how the `DispatcherServlet` works is the foundation for everything in this chapter.

---

## 🧠 The DispatcherServlet — Spring's Front Controller

Every HTTP request that enters a Spring MVC application passes through a single entry point: the `DispatcherServlet`. This is the **Front Controller** pattern — rather than having many separate servlets handling different URLs, one servlet receives everything and delegates to the appropriate handler based on a set of mappings.

The DispatcherServlet's processing pipeline works as follows: the request arrives and the servlet asks the `HandlerMapping` which controller method handles this URL. It then passes the request through any configured `HandlerInterceptors` (pre-processing). Next it invokes the controller method using a `HandlerAdapter`, which handles the complexity of binding parameters, converting request bodies, and reading headers. The controller method returns either a view name, a model object, or directly writes to the response. For REST APIs (using `@RestController`), the return value is serialized to JSON/XML by a `MessageConverter` and written directly to the response body.

```
HTTP Request
    │
    ▼
DispatcherServlet
    │ asks HandlerMapping: "which method handles GET /api/students/42?"
    ▼
HandlerInterceptor.preHandle()     ← logging, auth, rate limiting
    │
    ▼
@GetMapping findById(@PathVariable Long id)   ← your controller method
    │
    ▼
HandlerInterceptor.postHandle()
    │
    ▼
HttpMessageConverter (Jackson)    ← serializes Student to JSON
    │
    ▼
HTTP Response (200 OK, JSON body)
```

---

## 🎮 Controllers — @Controller vs @RestController

```java
// @Controller returns view names (for server-side rendered HTML with Thymeleaf/JSP)
@Controller
public class StudentViewController {

    @GetMapping("/students")
    public String listStudents(Model model) {
        model.addAttribute("students", studentService.findAll());
        return "students/list"; // Returns the name of a Thymeleaf template to render
    }
}

// @RestController = @Controller + @ResponseBody on every method
// Every method's return value is serialized directly to the HTTP response body (as JSON by default)
@RestController
@RequestMapping("/api/students") // Base path — all methods inherit this prefix
public class StudentController {

    private final StudentService studentService;

    // Constructor injection — no @Autowired needed with a single constructor
    public StudentController(StudentService studentService) {
        this.studentService = studentService;
    }

    // GET /api/students → returns all students as a JSON array
    @GetMapping
    public List<StudentDTO> findAll() {
        return studentService.findAll();
    }

    // GET /api/students/42 → returns one student
    @GetMapping("/{id}")
    public ResponseEntity<StudentDTO> findById(@PathVariable Long id) {
        return studentService.findById(id)
            .map(student -> ResponseEntity.ok(student))              // 200 OK with body
            .orElse(ResponseEntity.notFound().build());              // 404 Not Found
    }

    // POST /api/students → creates a new student
    @PostMapping
    public ResponseEntity<StudentDTO> create(@Valid @RequestBody CreateStudentRequest request,
                                             UriComponentsBuilder ucb) {
        StudentDTO created = studentService.create(request);
        // Build the Location header pointing to the new resource
        URI location = ucb.path("/api/students/{id}").buildAndExpand(created.id()).toUri();
        return ResponseEntity.created(location).body(created); // 201 Created
    }

    // PUT /api/students/42 → full replacement update
    @PutMapping("/{id}")
    public ResponseEntity<StudentDTO> update(@PathVariable Long id,
                                              @Valid @RequestBody UpdateStudentRequest request) {
        return studentService.update(id, request)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // PATCH /api/students/42 → partial update
    @PatchMapping("/{id}/score")
    public ResponseEntity<Void> updateScore(@PathVariable Long id,
                                             @RequestBody @Valid UpdateScoreRequest request) {
        studentService.updateScore(id, request.score());
        return ResponseEntity.noContent().build(); // 204 No Content — update succeeded
    }

    // DELETE /api/students/42
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        studentService.delete(id);
        return ResponseEntity.noContent().build(); // 204 No Content
    }
}
```

---

## 🔢 Parameter Binding — Getting Data Into Your Methods

Spring MVC can bind HTTP data to method parameters from several sources simultaneously:

```java
@RestController
@RequestMapping("/api")
public class ParameterDemoController {

    // @PathVariable — extracts a variable segment from the URL path
    // GET /api/departments/5/students/42
    @GetMapping("/departments/{deptId}/students/{studentId}")
    public StudentDTO findStudent(@PathVariable Long deptId,
                                   @PathVariable Long studentId) {
        return studentService.findByDepartmentAndId(deptId, studentId);
    }

    // @RequestParam — reads a query string or form parameter
    // GET /api/students?page=0&size=20&sort=name&grade=A
    @GetMapping("/students")
    public Page<StudentDTO> search(
            @RequestParam(defaultValue = "0")  int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id") String sort,
            @RequestParam(required = false)    String grade) { // Optional parameter
        return studentService.search(page, size, sort, grade);
    }

    // @RequestBody — deserializes the JSON request body into a Java object
    @PostMapping("/students")
    public ResponseEntity<StudentDTO> create(@RequestBody @Valid CreateStudentRequest request) {
        return ResponseEntity.ok(studentService.create(request));
    }

    // @RequestHeader — reads an HTTP header value
    @GetMapping("/secure-data")
    public DataDTO getSecureData(
            @RequestHeader("X-API-Key") String apiKey,
            @RequestHeader(value = "X-Request-Id", required = false) String requestId) {
        return dataService.fetchSecure(apiKey, requestId);
    }

    // @CookieValue — reads a cookie
    @GetMapping("/preferences")
    public PreferencesDTO getPreferences(@CookieValue(value = "theme", defaultValue = "light") String theme) {
        return preferencesService.get(theme);
    }

    // @ModelAttribute — binds multiple form fields or query parameters to an object
    @GetMapping("/search")
    public List<StudentDTO> search(@ModelAttribute StudentSearchCriteria criteria) {
        // Spring binds ?name=Alice&minScore=85&status=ENROLLED to the criteria object
        return studentService.search(criteria);
    }

    // Multiple sources at once — Spring handles them all independently
    @GetMapping("/reports/{type}")
    public ReportDTO generateReport(@PathVariable String type,
                                     @RequestParam(defaultValue = "2024") int year,
                                     @RequestHeader("Accept-Language") String lang,
                                     @CookieValue(defaultValue = "USD") String currency) {
        return reportService.generate(type, year, lang, currency);
    }
}
```

---

## ✅ Request Validation with @Valid

Validation should always happen at the web layer — before data reaches your service or database. Spring MVC integrates with Jakarta Bean Validation (Hibernate Validator) seamlessly.

```java
// Define validation constraints on your DTO (Data Transfer Object)
public record CreateStudentRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 150, message = "Name must be between 2 and 150 characters")
    String name,

    @NotBlank(message = "Email is required")
    @Email(message = "Must be a valid email address")
    String email,

    @NotNull(message = "Date of birth is required")
    @Past(message = "Date of birth must be in the past")
    LocalDate dateOfBirth,

    @DecimalMin(value = "0.0", message = "Score must be non-negative")
    @DecimalMax(value = "100.0", message = "Score cannot exceed 100")
    @Digits(integer = 3, fraction = 2)
    BigDecimal score,

    @NotNull
    @Pattern(regexp = "^[A-Z]{2}-\\d{4}$", message = "Department code must be like CS-2024")
    String departmentCode
) {}

// DTO validation for nested objects
public record UpdateStudentRequest(
    @Size(max = 150) String name,
    @Valid AddressDTO address // @Valid triggers cascaded validation on nested objects
) {}

public record AddressDTO(
    @NotBlank String street,
    @NotBlank String city,
    @Pattern(regexp = "\\d{5}(-\\d{4})?") String zipCode
) {}
```

When you annotate a method parameter with `@Valid`, Spring automatically validates the object before the method body runs. If validation fails, Spring throws a `MethodArgumentNotValidException`, which you catch in your exception handler:

```java
// Global exception handler — a single class handles errors for all controllers
@RestControllerAdvice // @ControllerAdvice + @ResponseBody on every method
public class GlobalExceptionHandler {

    // Handles validation failures from @Valid on @RequestBody
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST) // 400 Bad Request
    public ErrorResponse handleValidationErrors(MethodArgumentNotValidException ex) {
        // Collect all field-level validation errors
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(fieldError ->
            errors.put(fieldError.getField(), fieldError.getDefaultMessage())
        );
        return new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation failed",
            errors
        );
    }

    // Handles validation failures from @Valid on @PathVariable or @RequestParam
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleConstraintViolations(ConstraintViolationException ex) {
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getConstraintViolations().forEach(violation ->
            errors.put(
                violation.getPropertyPath().toString(),
                violation.getMessage()
            )
        );
        return new ErrorResponse(400, "Constraint violation", errors);
    }

    // Handles custom business exceptions
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse(404, ex.getMessage(), null);
    }

    @ExceptionHandler(DuplicateEmailException.class)
    @ResponseStatus(HttpStatus.CONFLICT) // 409 Conflict
    public ErrorResponse handleDuplicateEmail(DuplicateEmailException ex) {
        return new ErrorResponse(409, ex.getMessage(), null);
    }

    // Catch-all for unexpected exceptions — prevents internal details leaking to clients
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGenericException(Exception ex) {
        log.error("Unexpected error", ex);
        return new ErrorResponse(500, "An internal error occurred", null);
    }
}

// A consistent error response format
public record ErrorResponse(
    int status,
    String message,
    Map<String, String> validationErrors
) {}
```

---

## 📐 ResponseEntity — Full HTTP Response Control

`ResponseEntity<T>` gives you complete control over the HTTP response: status code, headers, and body. Use it when you need to set a specific status code or headers, or when the response may have no body (like 204 No Content).

```java
@GetMapping("/students/{id}/avatar")
public ResponseEntity<byte[]> getAvatar(@PathVariable Long id) {
    byte[] imageData = studentService.getAvatarBytes(id);
    if (imageData == null) {
        return ResponseEntity.notFound().build();
    }

    // Set Content-Type and Content-Disposition headers for file download
    HttpHeaders headers = new HttpHeaders();
    headers.setContentType(MediaType.IMAGE_PNG);
    headers.setContentDispositionFormData("inline", id + "-avatar.png");
    headers.setCacheControl("max-age=3600"); // Cache for 1 hour

    return ResponseEntity.ok()
        .headers(headers)
        .body(imageData);
}

// Returning a resource for streaming (efficient for large files)
@GetMapping("/students/{id}/report")
public ResponseEntity<Resource> downloadReport(@PathVariable Long id) {
    Resource file = reportService.generatePdf(id);
    return ResponseEntity.ok()
        .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"student-report-" + id + ".pdf\"")
        .contentType(MediaType.APPLICATION_PDF)
        .body(file);
}
```

---

## 🌐 Content Negotiation & Consuming/Producing

You can restrict what media types a controller method accepts and produces:

```java
@RestController
@RequestMapping("/api/students")
public class StudentController {

    // Only accepts JSON request bodies; only produces JSON responses
    @PostMapping(consumes = MediaType.APPLICATION_JSON_VALUE,
                 produces = MediaType.APPLICATION_JSON_VALUE)
    public ResponseEntity<StudentDTO> create(@RequestBody @Valid CreateStudentRequest req) {
        return ResponseEntity.ok(studentService.create(req));
    }

    // One method returns JSON; another returns XML — Spring picks based on Accept header
    @GetMapping(value = "/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
    public StudentDTO findByIdJson(@PathVariable Long id) {
        return studentService.findById(id).orElseThrow();
    }

    @GetMapping(value = "/{id}", produces = MediaType.APPLICATION_XML_VALUE)
    public StudentDTO findByIdXml(@PathVariable Long id) {
        return studentService.findById(id).orElseThrow();
    }
}
```

---

## 🔀 HandlerInterceptor — Pre/Post Request Processing

Interceptors run around controller methods and are useful for cross-cutting web concerns like request logging, authentication checks, locale setting, and request timing.

```java
@Component
public class RequestLoggingInterceptor implements HandlerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(RequestLoggingInterceptor.class);

    // preHandle: runs BEFORE the controller method
    // Return true to continue; return false to abort processing and send the response yourself
    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String requestId = UUID.randomUUID().toString().substring(0, 8);
        request.setAttribute("requestId", requestId);
        request.setAttribute("startTime", System.currentTimeMillis());
        log.info("[{}] → {} {}", requestId, request.getMethod(), request.getRequestURI());
        return true; // Continue processing
    }

    // postHandle: runs AFTER the controller method but BEFORE response is written
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) throws Exception {
        response.setHeader("X-Request-Id", (String) request.getAttribute("requestId"));
    }

    // afterCompletion: runs AFTER the response is fully written (even on exception)
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) throws Exception {
        long elapsed = System.currentTimeMillis() - (long) request.getAttribute("startTime");
        String requestId = (String) request.getAttribute("requestId");
        log.info("[{}] ← {} {} {}ms", requestId,
            response.getStatus(), request.getRequestURI(), elapsed);
    }
}

// Register the interceptor and define which paths it applies to
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    private final RequestLoggingInterceptor loggingInterceptor;

    public WebConfig(RequestLoggingInterceptor loggingInterceptor) {
        this.loggingInterceptor = loggingInterceptor;
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor)
            .addPathPatterns("/api/**")        // Apply to all API paths
            .excludePathPatterns("/api/health"); // But not the health check
    }
}
```

---

## 🔒 CORS Configuration

Cross-Origin Resource Sharing (CORS) is a browser security mechanism that blocks JavaScript from making requests to a different domain than the one that served the page. For REST APIs consumed by a frontend app on a different domain, you must configure CORS.

```java
// Method-level CORS — fine-grained control
@RestController
@RequestMapping("/api/public")
public class PublicController {

    @GetMapping("/courses")
    @CrossOrigin(origins = {"https://app.example.com", "http://localhost:3000"}) // Allow specific origins
    public List<CourseDTO> getPublicCourses() {
        return courseService.findPublic();
    }
}

// Global CORS — applies to all controllers
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com", "http://localhost:3000")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .exposedHeaders("X-Request-Id", "Location")
            .allowCredentials(true)  // Allow cookies/auth headers to be sent
            .maxAge(3600);           // Cache preflight response for 1 hour
    }
}
```

---

## 📤 File Upload & Download

```java
@RestController
@RequestMapping("/api/students/{id}")
public class StudentFileController {

    private final StorageService storageService;

    // MultipartFile receives the uploaded file from a multipart/form-data request
    @PostMapping("/avatar")
    public ResponseEntity<String> uploadAvatar(
            @PathVariable Long id,
            @RequestParam("file") MultipartFile file) {

        if (file.isEmpty()) {
            return ResponseEntity.badRequest().body("File cannot be empty");
        }

        // Validate file type
        String contentType = file.getContentType();
        if (!Set.of("image/jpeg", "image/png", "image/webp").contains(contentType)) {
            return ResponseEntity.badRequest().body("Only JPEG, PNG, WebP images allowed");
        }

        // Validate file size (also configurable in application.properties)
        if (file.getSize() > 5 * 1024 * 1024) { // 5MB limit
            return ResponseEntity.badRequest().body("File too large (max 5MB)");
        }

        String fileUrl = storageService.store(id, file);
        return ResponseEntity.ok(fileUrl);
    }
}
```

```properties
# application.properties — configure max upload size
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=10MB
```

---

## 📡 API Versioning Strategies

As your API evolves, you will need to introduce breaking changes without disrupting existing clients. There are three common approaches:

```java
// Strategy 1: URL Path versioning — most visible and cacheable
@RestController
@RequestMapping("/api/v1/students")
public class StudentControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/students")
public class StudentControllerV2 { ... }

// Strategy 2: Request parameter versioning
@RestController
@RequestMapping("/api/students")
public class StudentController {
    @GetMapping(params = "version=1")
    public StudentDTOv1 findAllV1() { ... }

    @GetMapping(params = "version=2")
    public StudentDTOv2 findAllV2() { ... }
}

// Strategy 3: Header versioning — cleanest for REST purists
@RestController
@RequestMapping("/api/students")
public class StudentController {
    @GetMapping(headers = "API-Version=1")
    public StudentDTOv1 findAllV1() { ... }

    @GetMapping(headers = "API-Version=2")
    public StudentDTOv2 findAllV2() { ... }
}
```

---

## ✅ Best Practices & Important Points

**Use `@RestControllerAdvice` for all exception handling, not per-controller `@ExceptionHandler`.** A global handler ensures every exception across every controller is handled consistently. Define a standard `ErrorResponse` structure so that clients always see the same JSON format regardless of which error occurs.

**Always validate input at the controller boundary with `@Valid`.** Do not pass raw, unvalidated data to your service layer. The service should be able to assume data is structurally correct because the controller already checked it. Business rule validation (e.g., "a student cannot enroll in a course they already completed") belongs in the service layer.

**Return `ResponseEntity` with the correct HTTP status code.** Many developers return `200 OK` for everything — even errors. Clients (browsers, mobile apps, other services) use status codes to understand what happened. Return `201 Created` for successful creation, `204 No Content` for successful deletion, `400 Bad Request` for invalid input, `404 Not Found` when a resource doesn't exist, `409 Conflict` for uniqueness violations, and `500 Internal Server Error` only for truly unexpected failures.

**Keep controllers thin.** A controller method should extract parameters, delegate to the service, and format the response. Business logic — validating business rules, coordinating multiple operations, managing transactions — belongs in the service layer. If your controller method is more than 10–15 lines, consider moving logic to the service.

**Disable `spring.jpa.open-in-view`.** When this setting is `true` (the default), the Hibernate session remains open for the entire HTTP request lifecycle, allowing lazy loading anywhere in your controller or view. This silently generates extra database queries you may not be aware of. Always disable it and load all necessary data explicitly in your service layer.

---

## 🔑 Key Summary

```
DispatcherServlet     → Single front controller that routes all HTTP requests
@RestController       → @Controller + @ResponseBody; methods return JSON/XML directly
@RequestMapping       → Base URL prefix for all methods in the controller
@GetMapping/@PostMapping/@PutMapping/@DeleteMapping → HTTP method-specific shortcuts
@PathVariable         → Extract {variable} from URL path
@RequestParam         → Extract ?key=value query parameters
@RequestBody          → Deserialize JSON request body into a Java object
@Valid                → Trigger Bean Validation on the parameter before method runs
ResponseEntity<T>     → Full control over status code, headers, and body
@RestControllerAdvice → Global exception handler across all controllers
@ExceptionHandler     → Maps a specific exception class to an HTTP error response
HandlerInterceptor    → Pre/post request logic (logging, timing, auth checks)
CORS                  → @CrossOrigin or WebMvcConfigurer.addCorsMappings()
Content negotiation   → produces/consumes restrict accepted/returned media types
```
