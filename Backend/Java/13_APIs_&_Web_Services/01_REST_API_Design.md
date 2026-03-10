# 13.1 — REST API Design
### Phase 13: APIs & Web Services

---

## What Is REST?

REST stands for **Representational State Transfer**. It is an architectural *style* (not a protocol, not a standard) defined by Roy Fielding in his 2000 doctoral dissertation. When an API is described as "RESTful," it means it conforms to the principles Fielding laid out.

The core idea is elegant: the web already works beautifully at massive scale using HTTP. REST says — design your API the same way the web is designed. Use URLs to identify resources, use HTTP verbs to describe actions, and use HTTP status codes to communicate outcomes.

> **Key mental model:** Think of every piece of data in your system as a *resource* (a user, an order, a product). REST gives each resource a unique address (URL), and lets clients interact with it using standardized verbs (GET, POST, PUT, DELETE).

---

## The 6 REST Constraints

These are the architectural rules Fielding defined. A truly RESTful API satisfies all six.

### 1. Client-Server Separation
The client (who makes the request) and the server (who processes it) are separate concerns. The client doesn't need to know how the server stores data, and the server doesn't need to know what kind of client is calling it. This separation allows them to evolve independently.

### 2. Statelessness
Each request from the client must contain **all information** the server needs to process it. The server stores no session state between requests. Authentication tokens, pagination parameters — everything must come in with the request. This makes REST services easy to scale horizontally because any server instance can handle any request.

### 3. Cacheability
Responses must declare themselves as cacheable or non-cacheable. When responses are cached appropriately, clients can reuse them, reducing server load. HTTP headers like `Cache-Control` and `ETag` enable this.

### 4. Uniform Interface
This is the most important constraint and has four sub-rules:
- Resources are identified by URIs (`/users/42`)
- Resources are manipulated through their representations (the client sends back a JSON body to update a resource)
- Messages are self-descriptive (headers carry enough information to describe how to process the message)
- Hypermedia as the engine of application state (HATEOAS — covered below)

### 5. Layered System
The client should not be able to tell whether it's connected directly to the server or through an intermediary (a load balancer, API gateway, cache). Layers can be added without the client knowing.

### 6. Code on Demand (Optional)
Servers can extend client functionality by sending executable code (like JavaScript). This is the only optional constraint.

---

## HTTP Methods — The Verbs of REST

Each HTTP method has a specific semantic meaning. Using them correctly is one of the most important REST practices.

| Method | Meaning | Idempotent? | Safe? | Request Body? |
|--------|---------|-------------|-------|---------------|
| `GET` | Retrieve a resource | Yes | Yes | No |
| `POST` | Create a new resource | No | No | Yes |
| `PUT` | Replace an entire resource | Yes | No | Yes |
| `PATCH` | Partially update a resource | No | No | Yes |
| `DELETE` | Remove a resource | Yes | No | No |
| `HEAD` | Like GET but no body returned | Yes | Yes | No |
| `OPTIONS` | Describe communication options | Yes | Yes | No |

**Idempotent** means calling the method multiple times with the same input produces the same result. `DELETE /users/42` called three times is the same as calling it once — the user is gone.

**Safe** means the method does not modify server state. `GET` is safe; `POST` is not.

### Why Does This Matter in Java?

In Spring Boot, each annotation maps to an HTTP method:

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    // GET /api/users/42  → retrieve user 42
    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }

    // POST /api/users  → create a new user (request body contains the data)
    @PostMapping
    public ResponseEntity<UserDto> createUser(@RequestBody @Valid CreateUserRequest request) {
        UserDto created = userService.create(request);
        // Return 201 Created with the Location of the new resource
        URI location = URI.create("/api/users/" + created.getId());
        return ResponseEntity.created(location).body(created);
    }

    // PUT /api/users/42  → replace the entire user record
    @PutMapping("/{id}")
    public ResponseEntity<UserDto> replaceUser(
            @PathVariable Long id,
            @RequestBody @Valid ReplaceUserRequest request) {
        return ResponseEntity.ok(userService.replace(id, request));
    }

    // PATCH /api/users/42  → update only the provided fields
    @PatchMapping("/{id}")
    public ResponseEntity<UserDto> updateUser(
            @PathVariable Long id,
            @RequestBody @Valid PatchUserRequest request) {
        return ResponseEntity.ok(userService.patch(id, request));
    }

    // DELETE /api/users/42  → remove user 42
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build(); // 204 No Content
    }
}
```

### PUT vs PATCH — A Common Source of Confusion

`PUT` replaces the **entire** resource. If you PUT a user object without including the `email` field, the email gets erased. `PATCH` only updates the fields you provide.

```java
// PUT: you must send ALL fields or they get replaced with null/default
// { "name": "Alice", "email": "alice@example.com", "role": "ADMIN" }

// PATCH: send only what you want to change
// { "role": "USER" }  ← only the role is updated
```

---

## HTTP Status Codes

Status codes communicate the outcome of a request. Using the right code is as important as using the right HTTP method.

### 2xx — Success

```
200 OK           → General success for GET, PUT, PATCH
201 Created      → Resource was successfully created (POST)
                   Should include a Location header pointing to the new resource
202 Accepted     → Request accepted but processing is async (will be done later)
204 No Content   → Success but nothing to return (DELETE, some PATCHes)
```

### 3xx — Redirection

```
301 Moved Permanently  → Resource has a new permanent URL
302 Found              → Temporary redirect
304 Not Modified       → Cached version is still valid (used with ETag/Last-Modified)
```

### 4xx — Client Errors

```
400 Bad Request         → Malformed request, validation failures
401 Unauthorized        → Not authenticated (no token or bad token)
403 Forbidden           → Authenticated but not authorized (token valid, but no permission)
404 Not Found           → Resource does not exist
405 Method Not Allowed  → HTTP method not supported on this endpoint
409 Conflict            → Request conflicts with current state (e.g., duplicate email)
410 Gone                → Resource existed but was permanently deleted
422 Unprocessable Entity → Validation error (semantically wrong even if syntactically valid)
429 Too Many Requests   → Rate limit exceeded
```

### 5xx — Server Errors

```
500 Internal Server Error → Unexpected server failure
502 Bad Gateway           → Upstream server returned invalid response
503 Service Unavailable   → Server temporarily down or overloaded
504 Gateway Timeout       → Upstream server timed out
```

### Practical Example in Spring Boot

```java
@RestControllerAdvice // This handles exceptions across ALL controllers
public class GlobalExceptionHandler {

    // 400: Validation failure
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.toList());

        return ResponseEntity
            .badRequest()
            .body(new ErrorResponse(400, "Validation failed", errors));
    }

    // 404: Resource not found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(404, ex.getMessage(), null));
    }

    // 409: Conflict (e.g., unique constraint violation)
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleConflict(DuplicateResourceException ex) {
        return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(new ErrorResponse(409, ex.getMessage(), null));
    }

    // 500: Unexpected error (catch-all)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        // Log the full exception here, but don't expose internals to client
        log.error("Unexpected error", ex);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(500, "An unexpected error occurred", null));
    }
}

// Consistent error response structure
public record ErrorResponse(int status, String message, List<String> details) {}
```

> **Best Practice:** Never expose stack traces or internal error messages to API consumers. Log them server-side, but return a clean, predictable error object to the client.

---

## URI Design Best Practices

URIs are the addresses of your resources. Good URI design makes your API intuitive and discoverable.

### Rule 1: Use Nouns, Not Verbs
URIs identify *things* (resources), not *actions*. The HTTP method is the action.

```
❌ Bad (verb in URL)
GET  /getUsers
POST /createUser
DELETE /deleteUser/42

✅ Good (noun in URL, verb in HTTP method)
GET    /users
POST   /users
DELETE /users/42
```

### Rule 2: Use Plural Nouns for Collections

```
✅ /users          (collection of users)
✅ /users/42       (specific user)
✅ /users/42/posts (posts belonging to user 42)
✅ /orders/99/items
```

### Rule 3: Use Lowercase and Hyphens

```
✅ /user-profiles
✅ /shipping-addresses
❌ /userProfiles  (camelCase in URL is unconventional)
❌ /User_Profiles (underscores and mixed case)
```

### Rule 4: Express Hierarchy Through Path Segments

```
GET /departments/5/employees        → All employees in department 5
GET /departments/5/employees/101    → Employee 101 in department 5
GET /orders/88/line-items           → All line items on order 88
```

Only nest when the nested resource is truly *owned by* the parent. If an employee can exist independently of a department (e.g., they can be in multiple departments), a flat `/employees/101` might be better.

### Rule 5: Use Query Parameters for Filtering, Sorting, and Searching

Path parameters (`/users/42`) identify a specific resource. Query parameters (`/users?role=ADMIN`) filter or modify how you retrieve a collection.

```
GET /users?role=ADMIN                    → Filter by role
GET /users?sort=lastName&order=asc       → Sort
GET /products?minPrice=10&maxPrice=50    → Range filter
GET /users?search=alice                  → Full-text search
GET /orders?status=PENDING&page=2&size=20 → Multiple filters + pagination
```

---

## Request and Response Headers

Headers are metadata attached to requests and responses. Understanding common headers is essential.

### Common Request Headers

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...   ← Authentication token
Content-Type: application/json                   ← Format of request body
Accept: application/json                         ← Formats client can handle
Accept-Language: en-US,en;q=0.9                 ← Preferred language
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000 ← Correlation ID for tracing
If-None-Match: "abc123"                          ← ETag for conditional requests
```

### Common Response Headers

```http
Content-Type: application/json; charset=UTF-8
Location: /api/users/123                         ← Where the new resource lives (after 201)
ETag: "abc123"                                   ← Version hash for caching
Cache-Control: max-age=3600, must-revalidate     ← Caching instructions
X-Rate-Limit-Remaining: 95                       ← How many requests left
```

### Reading Headers in Spring Boot

```java
@GetMapping("/profile")
public ResponseEntity<UserDto> getProfile(
        @RequestHeader("Authorization") String authHeader,
        @RequestHeader(value = "Accept-Language", defaultValue = "en") String language) {

    // Extract token from "Bearer <token>"
    String token = authHeader.startsWith("Bearer ") 
        ? authHeader.substring(7) 
        : authHeader;

    return ResponseEntity.ok(userService.getProfile(token, language));
}
```

---

## Content Negotiation

Content negotiation is the mechanism by which client and server agree on the format of the data being exchanged. The client declares what it can handle with the `Accept` header, and the server responds with what it can produce.

```java
@GetMapping(
    value = "/report",
    produces = {MediaType.APPLICATION_JSON_VALUE, "text/csv"}
)
public ResponseEntity<?> getReport(
        @RequestHeader(value = "Accept", defaultValue = "application/json") String accept) {

    if (accept.contains("text/csv")) {
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("text/csv"))
            .body(reportService.generateCsv());
    }
    return ResponseEntity.ok(reportService.generateJson());
}
```

Spring MVC handles content negotiation automatically when you configure `produces` correctly — the framework matches the `Accept` header to the appropriate method or serializer.

---

## HATEOAS — Hypermedia as the Engine of Application State

HATEOAS is the most advanced and least commonly implemented REST constraint. The idea is that API responses include links describing what the client can do next — just like how a web page contains links to other pages.

A HATEOAS response for a user resource might look like this:

```json
{
  "id": 42,
  "name": "Alice",
  "email": "alice@example.com",
  "_links": {
    "self": { "href": "/api/users/42" },
    "update": { "href": "/api/users/42", "method": "PUT" },
    "delete": { "href": "/api/users/42", "method": "DELETE" },
    "orders": { "href": "/api/users/42/orders" }
  }
}
```

Spring HATEOAS provides support for this:

```java
// Add Spring HATEOAS dependency: spring-boot-starter-hateoas

import org.springframework.hateoas.*;
import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

@GetMapping("/{id}")
public EntityModel<UserDto> getUser(@PathVariable Long id) {
    UserDto user = userService.findById(id);

    // Build links programmatically
    return EntityModel.of(user,
        linkTo(methodOn(UserController.class).getUser(id)).withSelfRel(),
        linkTo(methodOn(UserController.class).deleteUser(id)).withRel("delete"),
        linkTo(methodOn(OrderController.class).getOrdersByUser(id)).withRel("orders")
    );
}
```

> **Honest perspective:** Full HATEOAS is rarely implemented in practice outside highly formalized APIs. Understanding it is important, but most teams stop at well-designed URIs and proper HTTP methods/status codes. Start there before reaching for HATEOAS.

---

## API Versioning Strategies

As your API evolves, you'll need to make breaking changes. API versioning lets you introduce new behavior without breaking existing clients.

### Strategy 1: URL Path Versioning (Most Common)

```
GET /api/v1/users
GET /api/v2/users
```

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 { ... }
```

**Pros:** Explicit, easy to test in a browser, easy to route at the API gateway level.
**Cons:** Pollutes the URI (URIs should identify resources, not versions).

### Strategy 2: Request Header Versioning

```
GET /api/users
Accept-Version: v2
```

```java
@GetMapping(value = "/users", headers = "Accept-Version=v1")
public List<UserDtoV1> getUsersV1() { ... }

@GetMapping(value = "/users", headers = "Accept-Version=v2")
public List<UserDtoV2> getUsersV2() { ... }
```

**Pros:** Clean URIs.
**Cons:** Harder to test in a browser.

### Strategy 3: Accept Header / Media Type Versioning

```
GET /api/users
Accept: application/vnd.myapp.users.v2+json
```

**Pros:** Follows HTTP specification most precisely.
**Cons:** Verbose, awkward to use in practice.

### Strategy 4: Query Parameter Versioning

```
GET /api/users?version=2
```

**Pros:** Simple.
**Cons:** Query parameters should be for filtering/sorting, not versioning.

> **Best Practice:** URL path versioning (`/v1/`, `/v2/`) is the most widely adopted strategy because it's visible, tooling-friendly, and easy to cache. Use it unless you have a specific reason not to.

---

## Pagination

Returning all records in a single response doesn't scale. Pagination breaks large result sets into manageable pages.

### Strategy 1: Offset-Based Pagination (Most Common)

The client specifies how many records to skip (`offset` or `page`) and how many to return (`size` or `limit`).

```
GET /api/users?page=2&size=20
GET /api/products?offset=40&limit=20
```

Response structure:

```json
{
  "data": [ ... ],
  "pagination": {
    "page": 2,
    "size": 20,
    "totalElements": 157,
    "totalPages": 8,
    "hasNext": true,
    "hasPrevious": true
  }
}
```

Spring Data + Spring MVC implementation:

```java
@GetMapping
public ResponseEntity<Page<UserDto>> listUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "id") String sortBy,
        @RequestParam(defaultValue = "asc") String direction) {

    // Validate page size to prevent abuse (don't let clients ask for 10,000 records)
    int safeSize = Math.min(size, 100);

    Sort sort = direction.equalsIgnoreCase("desc") 
        ? Sort.by(sortBy).descending() 
        : Sort.by(sortBy).ascending();

    Pageable pageable = PageRequest.of(page, safeSize, sort);
    Page<UserDto> result = userService.findAll(pageable);

    return ResponseEntity.ok(result);
}
```

**Weakness of offset-based pagination:** If records are inserted or deleted between page requests, items can appear twice or be skipped (the "phantom row" problem).

### Strategy 2: Cursor-Based Pagination

Instead of a page number, the client receives an opaque cursor (typically an encoded ID or timestamp). On the next request, they send this cursor to get the next page.

```
GET /api/users?limit=20                          → First page
GET /api/users?after=eyJpZCI6MjB9&limit=20       → Second page (cursor is base64 encoded)
```

```java
@GetMapping
public ResponseEntity<CursorPage<UserDto>> listUsers(
        @RequestParam(required = false) String after,
        @RequestParam(defaultValue = "20") int limit) {

    Long afterId = after != null ? decodeCursor(after) : null;
    List<UserDto> users = userService.findAfter(afterId, limit + 1); // +1 to detect hasNext

    boolean hasNext = users.size() > limit;
    if (hasNext) users = users.subList(0, limit);

    String nextCursor = hasNext ? encodeCursor(users.get(users.size() - 1).getId()) : null;

    return ResponseEntity.ok(new CursorPage<>(users, nextCursor));
}

private String encodeCursor(Long id) {
    return Base64.getEncoder().encodeToString(("{\"id\":" + id + "}").getBytes());
}
```

**Advantage:** Consistent results even when data is inserted/deleted between pages. Ideal for infinite-scroll UIs and large datasets.

---

## Filtering, Sorting, and Searching

These are expressed through query parameters on collection endpoints.

```java
@GetMapping
public ResponseEntity<List<ProductDto>> listProducts(
        @RequestParam(required = false) String category,
        @RequestParam(required = false) BigDecimal minPrice,
        @RequestParam(required = false) BigDecimal maxPrice,
        @RequestParam(required = false) String search,
        @RequestParam(defaultValue = "name") String sortBy,
        @RequestParam(defaultValue = "asc") String order) {

    ProductFilter filter = ProductFilter.builder()
        .category(category)
        .minPrice(minPrice)
        .maxPrice(maxPrice)
        .search(search)
        .sortBy(sortBy)
        .order(order)
        .build();

    return ResponseEntity.ok(productService.findAll(filter));
}
```

---

## Idempotency

An operation is idempotent if performing it multiple times has the same effect as performing it once.

`GET`, `PUT`, `DELETE`, and `HEAD` are idempotent. `POST` is not idempotent by default — submitting a payment form twice can result in two charges.

For critical `POST` operations, you can add idempotency using an **idempotency key** in the request header:

```
POST /api/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{ "amount": 100.00, "currency": "USD" }
```

The server stores the idempotency key with the result. If the same key comes in again, it returns the stored result without processing again.

```java
@PostMapping("/payments")
public ResponseEntity<PaymentDto> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @RequestBody @Valid CreatePaymentRequest request) {

    // Check if we've already processed this key
    Optional<PaymentDto> cached = idempotencyService.getResult(idempotencyKey);
    if (cached.isPresent()) {
        return ResponseEntity.ok(cached.get()); // Return stored result
    }

    // Process new payment
    PaymentDto result = paymentService.process(request);

    // Store result with the idempotency key
    idempotencyService.storeResult(idempotencyKey, result);

    return ResponseEntity.status(HttpStatus.CREATED).body(result);
}
```

---

## Complete Example: A Well-Designed User API

Putting it all together, here is what a thoughtfully designed REST API controller looks like:

```java
@RestController
@RequestMapping("/api/v1/users")
@Validated  // Enables constraint validation on method parameters
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService; // Constructor injection (preferred)
    }

    /**
     * List users with filtering, sorting, and pagination.
     * GET /api/v1/users?role=ADMIN&page=0&size=20&sortBy=email&order=asc
     */
    @GetMapping
    public ResponseEntity<PagedResponse<UserDto>> listUsers(
            @RequestParam(required = false) String role,
            @RequestParam(required = false) String search,
            @RequestParam(defaultValue = "0") @Min(0) int page,
            @RequestParam(defaultValue = "20") @Min(1) @Max(100) int size,
            @RequestParam(defaultValue = "id") String sortBy,
            @RequestParam(defaultValue = "asc") String order) {

        PagedResponse<UserDto> response = userService.findAll(role, search, page, size, sortBy, order);
        return ResponseEntity.ok(response);
    }

    /**
     * Get a single user by ID.
     * GET /api/v1/users/42
     */
    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable @Positive Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElseThrow(() -> new ResourceNotFoundException("User not found: " + id));
    }

    /**
     * Create a new user.
     * POST /api/v1/users
     */
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<UserDto> createUser(
            @RequestBody @Valid CreateUserRequest request,
            UriComponentsBuilder uriBuilder) {

        UserDto created = userService.create(request);

        // Tell the client where the new resource lives
        URI location = uriBuilder
            .path("/api/v1/users/{id}")
            .buildAndExpand(created.getId())
            .toUri();

        return ResponseEntity.created(location).body(created);
    }

    /**
     * Fully replace a user. All fields must be provided.
     * PUT /api/v1/users/42
     */
    @PutMapping("/{id}")
    public ResponseEntity<UserDto> replaceUser(
            @PathVariable @Positive Long id,
            @RequestBody @Valid ReplaceUserRequest request) {

        return ResponseEntity.ok(userService.replace(id, request));
    }

    /**
     * Partially update a user. Only provided fields are updated.
     * PATCH /api/v1/users/42
     */
    @PatchMapping("/{id}")
    public ResponseEntity<UserDto> patchUser(
            @PathVariable @Positive Long id,
            @RequestBody @Valid PatchUserRequest request) {

        return ResponseEntity.ok(userService.patch(id, request));
    }

    /**
     * Delete a user.
     * DELETE /api/v1/users/42
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable @Positive Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build(); // 204 No Content
    }
}
```

---

## ⚠️ Important Points & Best Practices

**1. Return consistent error responses.** Every error, from 400 to 500, should return the same JSON structure. Clients should not have to guess the shape of an error response.

**2. Never use GET requests with a body.** Although technically possible, most proxies and caches strip GET request bodies. Use query parameters for GET filtering.

**3. Don't expose your database schema through your API.** Your API DTOs (Data Transfer Objects) should be a deliberate contract, not a direct mirror of your database tables. Field names, structures, and IDs in your database are internal details.

**4. Version from day one.** Even if you only have v1, build versioning into your URI structure from the start. Retrofitting versioning into an unversioned API is painful.

**5. Always cap pagination size.** Allow clients to specify page size, but enforce a maximum. Without a cap, a malicious or careless client can bring down your server with a request for all records.

**6. Use 422 for semantic validation failures and 400 for syntactic ones.** A request with malformed JSON is `400 Bad Request`. A request with valid JSON but a negative price is `422 Unprocessable Entity`.

**7. Keep responses small and consistent.** Don't return every field of every nested object. Use sparse fieldsets or separate endpoints for detailed views.

**8. Treat your API as a product.** Write documentation, version changes, communicate breaking changes to consumers, and maintain a changelog.
