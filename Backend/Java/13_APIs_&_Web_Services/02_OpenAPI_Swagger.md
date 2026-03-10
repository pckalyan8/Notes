# 13.2 — OpenAPI / Swagger
### Phase 13: APIs & Web Services

---

## The Problem OpenAPI Solves

Imagine you've built a beautiful REST API. It's well-designed, properly versioned, and your team is proud of it. Now a mobile developer needs to integrate with it. How do they know what endpoints exist? What parameters are required? What does the response look like? What does a 404 response body look like?

Without documentation, they read your source code, ask you questions, or write code that breaks when you change something. This is the problem OpenAPI solves.

**OpenAPI** (formerly known as Swagger) is a specification for describing REST APIs in a machine-readable format (JSON or YAML). From a single OpenAPI document, you can:

- Generate interactive documentation (Swagger UI) that developers can use to test your API in the browser
- Generate client libraries in almost any language automatically
- Generate server stub code
- Validate requests and responses automatically
- Drive contract testing between services

The relationship between OpenAPI and Swagger is a common point of confusion. Swagger was the original name of both the specification and the tooling. In 2016, the specification was donated to the OpenAPI Initiative (under the Linux Foundation) and renamed to "OpenAPI Specification." The tooling (Swagger UI, Swagger Editor, Swagger Codegen) kept the Swagger name. So today, "OpenAPI" is the specification, and "Swagger" refers to the ecosystem of tools built around it.

---

## What an OpenAPI Document Looks Like

Before learning how to generate it from Java, it helps to understand what you're generating. OpenAPI documents are written in YAML or JSON.

```yaml
openapi: 3.0.3
info:
  title: User Management API
  description: API for managing users in the system
  version: 1.0.0
  contact:
    name: API Team
    email: api@example.com

servers:
  - url: https://api.example.com/v1
    description: Production server
  - url: https://staging-api.example.com/v1
    description: Staging server

paths:
  /users:
    get:
      summary: List all users
      description: Returns a paginated list of users, optionally filtered by role
      operationId: listUsers
      tags:
        - Users
      parameters:
        - name: role
          in: query
          description: Filter users by role
          required: false
          schema:
            type: string
            enum: [ADMIN, USER, MODERATOR]
        - name: page
          in: query
          required: false
          schema:
            type: integer
            minimum: 0
            default: 0
      responses:
        '200':
          description: Successfully retrieved users
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PagedUserResponse'
        '401':
          description: Authentication required
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  schemas:
    UserDto:
      type: object
      properties:
        id:
          type: integer
          format: int64
          example: 42
        name:
          type: string
          example: Alice Smith
        email:
          type: string
          format: email
          example: alice@example.com
        role:
          type: string
          enum: [ADMIN, USER, MODERATOR]

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

This is a structured contract. Now let's see how to generate this automatically from Spring Boot code.

---

## Springdoc-OpenAPI — Auto-Generation in Spring Boot

The recommended library for integrating OpenAPI with Spring Boot is **springdoc-openapi**. It inspects your controllers and model classes at startup and generates the OpenAPI specification automatically, with no boilerplate XML configuration.

### Setup

Add the dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

That's genuinely all you need for basic functionality. Spring Boot auto-configuration does the rest. Once your application starts, visit:

- `http://localhost:8080/swagger-ui.html` → Interactive Swagger UI
- `http://localhost:8080/v3/api-docs` → Raw OpenAPI JSON document
- `http://localhost:8080/v3/api-docs.yaml` → Raw OpenAPI YAML document

Springdoc scans your `@RestController` classes and builds the spec from what it finds. However, the generated spec will only contain structural information (URLs, parameter names, response types). To make it truly useful for consumers, you annotate your code with OpenAPI annotations.

---

## Configuring Global API Info

Define the top-level information for your API using a configuration bean:

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("User Management API")
                .description("Complete API for managing users, roles, and permissions")
                .version("v1.0.0")
                .contact(new Contact()
                    .name("API Support Team")
                    .email("api-support@example.com")
                    .url("https://example.com/support"))
                .license(new License()
                    .name("Apache 2.0")
                    .url("https://www.apache.org/licenses/LICENSE-2.0")))
            .addServersItem(new Server()
                .url("https://api.example.com/v1")
                .description("Production"))
            .addServersItem(new Server()
                .url("http://localhost:8080")
                .description("Local Development"))
            // Describe the security scheme so Swagger UI shows an "Authorize" button
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"));
    }
}
```

---

## Annotating Controllers and Operations

The `@Operation`, `@ApiResponse`, and `@Parameter` annotations let you describe each endpoint in detail.

```java
@RestController
@RequestMapping("/api/v1/users")
@Tag(name = "Users", description = "Operations for managing users") // Groups endpoints in Swagger UI
public class UserController {

    @Operation(
        summary = "Get a user by ID",
        description = "Retrieves a single user by their unique identifier. " +
                      "Returns 404 if no user exists with the given ID."
    )
    @ApiResponses({
        @ApiResponse(
            responseCode = "200",
            description = "User found",
            content = @Content(schema = @Schema(implementation = UserDto.class))
        ),
        @ApiResponse(
            responseCode = "404",
            description = "User not found",
            content = @Content(schema = @Schema(implementation = ErrorResponse.class))
        ),
        @ApiResponse(
            responseCode = "401",
            description = "Authentication required — include a Bearer token"
        )
    })
    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(
            @Parameter(description = "The unique ID of the user", example = "42")
            @PathVariable Long id) {

        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElseThrow(() -> new ResourceNotFoundException("User " + id + " not found"));
    }


    @Operation(
        summary = "Create a new user",
        description = "Creates a new user. The email address must be unique."
    )
    @ApiResponses({
        @ApiResponse(
            responseCode = "201",
            description = "User created successfully",
            content = @Content(schema = @Schema(implementation = UserDto.class)),
            headers = @Header(name = "Location", description = "URL of the created user")
        ),
        @ApiResponse(
            responseCode = "400",
            description = "Invalid input — check the request body for validation errors"
        ),
        @ApiResponse(
            responseCode = "409",
            description = "A user with this email already exists"
        )
    })
    @PostMapping
    public ResponseEntity<UserDto> createUser(
            @io.swagger.v3.oas.annotations.parameters.RequestBody(
                description = "Details of the user to create",
                required = true,
                content = @Content(schema = @Schema(implementation = CreateUserRequest.class))
            )
            @RequestBody @Valid CreateUserRequest request,
            UriComponentsBuilder uriBuilder) {

        UserDto created = userService.create(request);
        URI location = uriBuilder.path("/api/v1/users/{id}").buildAndExpand(created.getId()).toUri();
        return ResponseEntity.created(location).body(created);
    }


    @Operation(
        summary = "List users",
        description = "Returns a paginated list of users. Supports filtering by role and searching by name/email."
    )
    @GetMapping
    public ResponseEntity<PagedResponse<UserDto>> listUsers(
            @Parameter(description = "Filter by role", schema = @Schema(allowableValues = {"ADMIN", "USER"}))
            @RequestParam(required = false) String role,

            @Parameter(description = "Page number (0-based)", example = "0")
            @RequestParam(defaultValue = "0") int page,

            @Parameter(description = "Number of results per page (max 100)", example = "20")
            @RequestParam(defaultValue = "20") int size) {

        return ResponseEntity.ok(userService.findAll(role, page, size));
    }
}
```

---

## Annotating Models / DTOs with `@Schema`

The `@Schema` annotation enriches the OpenAPI document with descriptions, examples, validation constraints, and more — making the generated documentation far more useful to API consumers.

```java
@Schema(description = "Data Transfer Object representing a user")
public class UserDto {

    @Schema(description = "Unique identifier", example = "42", accessMode = Schema.AccessMode.READ_ONLY)
    private Long id;

    @Schema(description = "Full name of the user", example = "Alice Smith", minLength = 2, maxLength = 100)
    private String name;

    @Schema(description = "Email address (unique across all users)", example = "alice@example.com")
    private String email;

    @Schema(description = "User's role in the system", allowableValues = {"ADMIN", "USER", "MODERATOR"})
    private String role;

    @Schema(description = "Account creation timestamp", accessMode = Schema.AccessMode.READ_ONLY)
    private LocalDateTime createdAt;
}


@Schema(description = "Request body for creating a new user")
public class CreateUserRequest {

    @NotBlank
    @Schema(description = "Full name", example = "Alice Smith", requiredMode = Schema.RequiredMode.REQUIRED)
    private String name;

    @NotBlank
    @Email
    @Schema(description = "Email address (must be unique)", example = "alice@example.com", requiredMode = Schema.RequiredMode.REQUIRED)
    private String email;

    @NotBlank
    @Size(min = 8)
    @Schema(description = "Password (min 8 characters)", example = "SecurePass123!", requiredMode = Schema.RequiredMode.REQUIRED)
    private String password;
}
```

---

## Grouping Endpoints with Tags

For larger APIs, you can organize endpoints into groups. Tags control how operations are grouped in Swagger UI.

```java
// Apply tag at class level — applies to all methods in this controller
@Tag(name = "Users", description = "User management operations")
@RestController
@RequestMapping("/api/v1/users")
public class UserController { ... }

// Different controller, different tag
@Tag(name = "Orders", description = "Order processing operations")
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController { ... }
```

You can also define tags globally with descriptions in your `OpenApiConfig`:

```java
.info(...)
.tags(List.of(
    new Tag().name("Users").description("Everything about users"),
    new Tag().name("Orders").description("Everything about orders")
))
```

---

## Hiding Endpoints from Documentation

Sometimes you have internal or deprecated endpoints you don't want appearing in the public documentation.

```java
// Hide this endpoint entirely from OpenAPI docs
@Hidden
@GetMapping("/internal/debug-info")
public ResponseEntity<Map<String, Object>> getDebugInfo() { ... }

// Mark as deprecated in Swagger UI (shows a warning to consumers)
@Operation(summary = "Get user (deprecated, use v2)", deprecated = true)
@GetMapping("/{id}")
public ResponseEntity<UserDto> getUserV1(@PathVariable Long id) { ... }
```

---

## Multiple API Docs with GroupedOpenApi

If your application has distinct sections (e.g., public API vs. admin API), you can create separate Swagger UI pages for each.

```java
@Configuration
public class OpenApiConfig {

    // Accessible at /swagger-ui.html?urls.primaryName=public-api
    @Bean
    public GroupedOpenApi publicApi() {
        return GroupedOpenApi.builder()
            .group("public-api")
            .displayName("Public API")
            .pathsToMatch("/api/v1/users/**", "/api/v1/products/**")
            .build();
    }

    // Accessible at /swagger-ui.html?urls.primaryName=admin-api
    @Bean
    public GroupedOpenApi adminApi() {
        return GroupedOpenApi.builder()
            .group("admin-api")
            .displayName("Admin API")
            .pathsToMatch("/api/v1/admin/**")
            .build();
    }
}
```

---

## application.properties Configuration

```properties
# Change the path of the Swagger UI
springdoc.swagger-ui.path=/docs

# Change the path of the raw API docs
springdoc.api-docs.path=/openapi

# Disable swagger in production
springdoc.swagger-ui.enabled=false   # (set to false in prod, true in dev)

# Sort operations alphabetically in Swagger UI
springdoc.swagger-ui.operationsSorter=alpha

# Sort tags alphabetically
springdoc.swagger-ui.tagsSorter=alpha

# Show the "Try it out" button expanded by default
springdoc.swagger-ui.defaultModelsExpandDepth=3

# Persist authorization token in Swagger UI across page refreshes
springdoc.swagger-ui.persistAuthorization=true
```

---

## Code Generation from OpenAPI Spec

One of the most powerful uses of the OpenAPI spec is automatic code generation. The **OpenAPI Generator** can produce client libraries, server stubs, and documentation in dozens of languages from your spec.

```xml
<!-- In pom.xml -->
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <version>7.3.0</version>
    <executions>
        <execution>
            <goals>
                <goal>generate</goal>
            </goals>
            <configuration>
                <!-- Point to your OpenAPI spec (can be a URL or file) -->
                <inputSpec>${project.basedir}/src/main/resources/openapi.yaml</inputSpec>
                
                <!-- Generate a Spring server stub -->
                <generatorName>spring</generatorName>
                
                <!-- Or generate a Java client library:
                <generatorName>java</generatorName>
                -->
                
                <apiPackage>com.example.api.generated</apiPackage>
                <modelPackage>com.example.model.generated</modelPackage>
                
                <configOptions>
                    <!-- Generate interface only, you implement it -->
                    <interfaceOnly>true</interfaceOnly>
                    <useTags>true</useTags>
                    <useSpringBoot3>true</useSpringBoot3>
                </configOptions>
            </configuration>
        </execution>
    </executions>
</plugin>
```

When `interfaceOnly=true`, the generator creates Java interfaces that your controllers implement. This is the "contract-first" approach: you define the OpenAPI spec first, generate the interface, then implement it.

```java
// Generated interface (do not modify — it is regenerated on every build)
public interface UsersApi {
    ResponseEntity<UserDto> getUser(@PathVariable Long id);
    ResponseEntity<UserDto> createUser(@RequestBody CreateUserRequest request);
}

// Your implementation
@RestController
public class UserController implements UsersApi {
    
    @Override
    public ResponseEntity<UserDto> getUser(Long id) {
        // Your implementation here
    }
}
```

---

## The "Code-First" vs "Contract-First" Debate

There are two philosophies for API development:

**Code-First (most common in Spring Boot):** You write your controllers and models, then use Springdoc to generate the OpenAPI spec. It's fast and keeps your code as the source of truth. The risk is that the spec reflects what you've accidentally built, not what you intended to build.

**Contract-First:** You write the OpenAPI spec first, generate the server interface from it, then implement the interface. The spec is the source of truth. This is more disciplined and is particularly valuable when multiple teams need to agree on an API before implementation starts.

> **Recommendation:** For internal APIs or rapid development, code-first works well. For public APIs, APIs consumed by multiple teams, or APIs governed by SLAs, contract-first produces more deliberate and stable designs.

---

## ⚠️ Important Points & Best Practices

**1. Enable Swagger UI in development, disable it in production.** Your internal API structure is information you don't necessarily want publicly accessible. Use Spring profiles to control this:

```java
@Profile("!production")  // Only active when NOT in production profile
@Bean
public GroupedOpenApi allEndpoints() {
    return GroupedOpenApi.builder().group("all").pathsToMatch("/**").build();
}
```

**2. Write real descriptions, not just type information.** A description like "The user's name (String, max 100 chars)" is redundant — the schema already says that. Write descriptions that explain *business rules*: "The user's display name. Must be unique within an organization. Cannot contain special characters."

**3. Include concrete examples everywhere.** The `example` field in `@Schema` is enormously helpful for consumers trying to understand the expected format. For complex request bodies, use `@ExampleObject` in your `@Content` annotation to provide complete worked examples.

**4. Document every error response, not just the happy path.** Most developers document `200 OK` and forget everything else. Documenting your `400`, `401`, `403`, `404`, `409`, and `500` responses is often the most valuable part of the documentation.

**5. Keep your OpenAPI spec in version control.** Generate the spec file as part of your build process and commit it. This allows you to do diff-based reviews of API changes and catch breaking changes before they go to production.
