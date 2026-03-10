# 13.3 — GraphQL with Java
### Phase 13: APIs & Web Services

---

## Why GraphQL Exists — The Problem It Solves

To understand GraphQL, you need to first understand the pain it was designed to eliminate. Facebook (now Meta) created GraphQL in 2012 to solve a concrete problem they encountered when building their mobile app: their REST APIs were ill-suited for the complex, interconnected data their news feed required.

Consider a typical scenario: you're building a screen that shows a user's profile alongside their three most recent posts and the number of likes on each. With REST, you might need to make three separate API calls:

1. `GET /users/42` — fetch the user's profile
2. `GET /users/42/posts?limit=3` — fetch their recent posts
3. `GET /posts/{id}/likes` (called three times) — fetch like counts

This is the **over-fetching and under-fetching problem**. The `/users/42` endpoint might return 30 fields when you only need 5 (over-fetching). The posts endpoint might not include like counts, forcing another round trip (under-fetching). Every REST endpoint returns a fixed shape of data, regardless of what the client actually needs.

GraphQL solves this by letting the **client declare exactly what data it needs** in a single request. Instead of the server deciding the response shape, the client asks a precise question and gets a precise answer.

---

## Core GraphQL Concepts

### Schema

A GraphQL API is defined by its **schema** — a strongly-typed description of all the data your API can provide and all the operations clients can perform. The schema is written in the Schema Definition Language (SDL).

```graphql
# This defines the data types your API exposes
type User {
  id: ID!           # The ! means this field is non-nullable (required)
  name: String!
  email: String!
  role: UserRole!
  posts: [Post!]!   # A list of Post objects (never null, list items never null)
  createdAt: String
}

type Post {
  id: ID!
  title: String!
  content: String!
  likeCount: Int!
  author: User!     # A Post always knows its author
}

enum UserRole {
  ADMIN
  USER
  MODERATOR
}

# Query = read operations (like GET in REST)
type Query {
  user(id: ID!): User           # Fetch a single user — might return null if not found
  users: [User!]!               # Fetch all users — always returns a list (possibly empty)
  post(id: ID!): Post
}

# Mutation = write operations (like POST/PUT/DELETE in REST)
type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

# Subscription = real-time updates over WebSocket
type Subscription {
  userCreated: User!
  postLiked(postId: ID!): Post!
}

input CreateUserInput {
  name: String!
  email: String!
  password: String!
}

input UpdateUserInput {
  name: String
  email: String
}
```

Everything in GraphQL is defined in this schema. The schema is the contract between client and server. Clients can only request fields and operations that exist in the schema.

### Queries — Asking for Data

A GraphQL query is a request from the client. The client describes the shape of the data it wants, and the server returns exactly that shape.

```graphql
# This is a GraphQL query — it looks like a skeleton of the response you want
query GetUserProfile {
  user(id: "42") {
    id
    name
    email
    posts {
      id
      title
      likeCount
    }
  }
}
```

The response will match this exact structure:

```json
{
  "data": {
    "user": {
      "id": "42",
      "name": "Alice Smith",
      "email": "alice@example.com",
      "posts": [
        { "id": "1", "title": "First Post", "likeCount": 15 },
        { "id": "2", "title": "Second Post", "likeCount": 8 }
      ]
    }
  }
}
```

Notice that `role` and `createdAt` were not requested, so they don't appear in the response. This is GraphQL's core value proposition: precise, predictable responses.

### Mutations — Changing Data

Mutations modify data. They have the same query syntax but are semantically different — they tell the server to make a change.

```graphql
mutation CreateNewUser {
  createUser(input: {
    name: "Bob Jones"
    email: "bob@example.com"
    password: "SecurePass123"
  }) {
    id        # Return only what we need from the created user
    name
    email
  }
}
```

### Subscriptions — Real-Time Updates

Subscriptions maintain a persistent connection (typically WebSocket) and push updates to the client when events occur.

```graphql
subscription WatchNewUsers {
  userCreated {
    id
    name
    email
  }
}
```

---

## Spring for GraphQL Setup

Spring for GraphQL is the official Spring integration, released as stable in Spring Boot 2.7 and significantly enhanced in Spring Boot 3.x. It integrates deeply with the rest of the Spring ecosystem.

### Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
<!-- For HTTP transport (REST-over-HTTP + WebSocket) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- For WebFlux transport (optional, for reactive) -->
<!-- <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency> -->
```

### Schema File Location

Spring for GraphQL automatically discovers `.graphqls` and `.gqls` files from `src/main/resources/graphql/`. Create your schema there:

```
src/
└── main/
    └── resources/
        └── graphql/
            ├── schema.graphqls     ← Main schema
            ├── user.graphqls       ← User-related types (can split across files)
            └── post.graphqls       ← Post-related types
```

### application.properties Configuration

```properties
# Enable the GraphQL endpoint (defaults to /graphql)
spring.graphql.path=/graphql

# Enable GraphiQL — the browser IDE for testing GraphQL (like Swagger UI)
spring.graphql.graphiql.enabled=true
spring.graphql.graphiql.path=/graphiql

# Enable schema introspection (disable in production for security)
spring.graphql.schema.introspection.enabled=true

# Print the schema at startup (useful for debugging)
spring.graphql.schema.printer.enabled=true
```

---

## Writing Resolvers with `@QueryMapping`, `@MutationMapping`, and `@SchemaMapping`

In Spring for GraphQL, you write resolvers using controller-style annotations. Each resolver method is responsible for fetching the data for one field in the schema.

### Query Resolvers

```java
@Controller  // Note: @Controller, not @RestController (GraphQL handles serialization differently)
public class UserGraphQLController {

    private final UserService userService;
    private final PostService postService;

    public UserGraphQLController(UserService userService, PostService postService) {
        this.userService = userService;
        this.postService = postService;
    }

    // Resolves the 'user(id: ID!)' field in the Query type
    @QueryMapping
    public User user(@Argument String id) {
        // @Argument maps the GraphQL argument 'id' to the Java parameter
        return userService.findById(Long.parseLong(id))
            .orElse(null); // Return null if not found — GraphQL schema says User is nullable
    }

    // Resolves the 'users' field in the Query type
    @QueryMapping
    public List<User> users() {
        return userService.findAll();
    }

    // Resolves the 'post(id: ID!)' field in the Query type
    @QueryMapping
    public Post post(@Argument String id) {
        return postService.findById(Long.parseLong(id)).orElse(null);
    }
}
```

### Field Resolvers with `@SchemaMapping`

`@SchemaMapping` resolves a field on a specific type — not a root query, but a nested field. This is where GraphQL's data fetching model becomes clear: each field in the graph can have its own resolver.

```java
@Controller
public class UserGraphQLController {

    // Resolves the 'posts' field on the User type
    // Called once for each User object that was requested
    @SchemaMapping(typeName = "User", field = "posts")
    public List<Post> posts(User user) {
        // 'user' is the parent object — the User whose posts we need
        return postService.findByAuthorId(user.getId());
    }
}

@Controller
public class PostGraphQLController {

    // Resolves the 'author' field on the Post type
    @SchemaMapping(typeName = "Post", field = "author")
    public User author(Post post) {
        return userService.findById(post.getAuthorId()).orElseThrow();
    }
}
```

The annotation can be simplified when the method name matches the field name:

```java
// typeName can be inferred from the class name pattern (e.g., UserController → User type)
// Or set explicitly:
@Controller
public class UserController {

    @SchemaMapping(typeName = "User")
    // field defaults to "posts" because the method is named "posts"
    public List<Post> posts(User user) {
        return postService.findByAuthorId(user.getId());
    }
}
```

### Mutation Resolvers

```java
@Controller
public class UserMutationController {

    private final UserService userService;

    // Resolves the 'createUser(input: CreateUserInput!)' mutation
    @MutationMapping
    public User createUser(@Argument CreateUserInput input) {
        // @Argument maps the entire 'input' object to a Java record/class
        return userService.create(input);
    }

    @MutationMapping
    public User updateUser(@Argument String id, @Argument UpdateUserInput input) {
        return userService.update(Long.parseLong(id), input);
    }

    @MutationMapping
    public Boolean deleteUser(@Argument String id) {
        userService.delete(Long.parseLong(id));
        return true;
    }
}

// Input types map to Java records or classes
public record CreateUserInput(String name, String email, String password) {}
public record UpdateUserInput(String name, String email) {}
```

---

## The N+1 Problem — The Most Critical GraphQL Performance Issue

The N+1 problem is the most important concept in GraphQL performance, and it arises directly from how field resolvers work.

Imagine this query:

```graphql
query {
  users {        # Returns 100 users
    id
    name
    posts {      # For each user, we want their posts
      title
    }
  }
}
```

Without any optimization, this would result in:
- 1 query to load 100 users
- 100 queries to load posts for each individual user

That's 101 database queries for a single GraphQL request. This is the N+1 problem (1 query + N queries for N parent objects).

### The Solution: DataLoader

DataLoader batches multiple individual data-fetching requests into a single bulk request. Instead of fetching one user's posts immediately, it collects all user IDs from across the request, then makes a single bulk call.

```java
@Configuration
public class DataLoaderConfig {

    @Bean
    public BatchLoaderRegistry batchLoaderRegistry(PostService postService) {
        BatchLoaderRegistry registry = new DefaultBatchLoaderRegistry();

        // Register a DataLoader named "posts" that loads posts in batches by user ID
        registry.forTypePair(Long.class, List.class)
            .withName("postsByUser")
            .registerBatchLoader((userIds, environment) -> {
                // Called ONCE with ALL userIds from the request, not one at a time
                Map<Long, List<Post>> postsByUserId = postService.findByUserIds(new HashSet<>(userIds));
                // Return results in the same order as the input IDs
                return Flux.fromIterable(userIds).map(id -> postsByUserId.getOrDefault(id, List.of()));
            });

        return registry;
    }
}
```

Then in your field resolver, instead of directly calling the service, you use the DataLoader:

```java
@SchemaMapping(typeName = "User", field = "posts")
public CompletableFuture<List<Post>> posts(User user, DataLoader<Long, List<Post>> postsByUserLoader) {
    // This doesn't execute immediately — it schedules a load for user.getId()
    // DataLoader collects all such calls across the request, then executes them as one batch
    return postsByUserLoader.load(user.getId());
}
```

The result: instead of 101 queries, we now have 2 — one for users, one batch query for all posts.

---

## Exception Handling in GraphQL

GraphQL doesn't use HTTP status codes for errors the way REST does. Instead, it always returns `200 OK` at the HTTP level (for valid GraphQL requests), but the response body may contain an `errors` array alongside the `data`.

```json
{
  "data": {
    "user": null
  },
  "errors": [
    {
      "message": "User not found: 999",
      "locations": [{ "line": 2, "column": 3 }],
      "path": ["user"],
      "extensions": {
        "classification": "NOT_FOUND",
        "code": "USER_NOT_FOUND"
      }
    }
  ]
}
```

Spring for GraphQL lets you customize error handling with `DataFetcherExceptionResolverAdapter`:

```java
@Component
public class GraphQLExceptionHandler extends DataFetcherExceptionResolverAdapter {

    @Override
    protected GraphQLError resolveToSingleError(Throwable ex, DataFetchingEnvironment env) {

        if (ex instanceof ResourceNotFoundException) {
            return GraphqlErrorBuilder.newError(env)
                .message(ex.getMessage())
                .errorType(ErrorType.NOT_FOUND)
                .extensions(Map.of("code", "NOT_FOUND"))
                .build();
        }

        if (ex instanceof AccessDeniedException) {
            return GraphqlErrorBuilder.newError(env)
                .message("You don't have permission to access this resource")
                .errorType(ErrorType.FORBIDDEN)
                .build();
        }

        if (ex instanceof ConstraintViolationException cve) {
            List<String> violations = cve.getConstraintViolations().stream()
                .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                .collect(Collectors.toList());

            return GraphqlErrorBuilder.newError(env)
                .message("Validation failed")
                .errorType(ErrorType.BAD_REQUEST)
                .extensions(Map.of("violations", violations))
                .build();
        }

        // For unexpected errors, don't leak internals
        return GraphqlErrorBuilder.newError(env)
            .message("Internal server error")
            .errorType(ErrorType.INTERNAL_ERROR)
            .build();
    }
}
```

---

## Testing GraphQL APIs

Spring for GraphQL provides a `GraphQlTester` specifically for testing GraphQL operations.

```java
@SpringBootTest
@AutoConfigureHttpGraphQlTester  // Provides GraphQlTester over HTTP
class UserGraphQLControllerTest {

    @Autowired
    private HttpGraphQlTester graphQlTester;

    @MockBean
    private UserService userService;

    @Test
    void testGetUser() {
        User mockUser = new User(42L, "Alice", "alice@example.com", UserRole.USER);
        when(userService.findById(42L)).thenReturn(Optional.of(mockUser));

        graphQlTester.document("""
            query {
              user(id: "42") {
                id
                name
                email
              }
            }
            """)
            .execute()
            .path("user.id").entity(String.class).isEqualTo("42")
            .path("user.name").entity(String.class).isEqualTo("Alice")
            .path("user.email").entity(String.class).isEqualTo("alice@example.com");
    }

    @Test
    void testGetUserNotFound() {
        when(userService.findById(999L)).thenReturn(Optional.empty());

        graphQlTester.document("""
            query {
              user(id: "999") { id name }
            }
            """)
            .execute()
            .path("user").valueIsNull()  // Field returns null
            .errors().satisfy(errors ->
                assertThat(errors).isNotEmpty()  // Or check for errors array
            );
    }
}
```

---

## GraphQL vs REST — When to Choose Each

This is a decision every team building an API will face. Neither is universally better.

**Choose REST when:**

- You're building a simple CRUD API with well-defined, stable resource shapes
- You want maximum caching (HTTP caching is more natural with REST's predictable URLs)
- Your clients are simple and always need the same data
- You need broad tooling support without expertise in GraphQL
- You're building a public API consumed by unknown third parties

**Choose GraphQL when:**

- Your clients have genuinely different data needs (mobile app vs. web app vs. partner integration)
- Your data is highly interconnected and relational (social networks, e-commerce catalogs)
- You want to reduce the number of round trips, especially on mobile over slow connections
- You have multiple client teams who can move independently without waiting for the API team to add new endpoints
- You're willing to invest in DataLoader patterns and schema design discipline

> **The key insight:** GraphQL shifts power from the server to the client. In REST, the server decides what data each endpoint returns. In GraphQL, the client decides. This is liberating for client teams but requires more discipline on the server side.

---

## ⚠️ Important Points & Best Practices

**1. Always implement DataLoader for any field that fetches from a parent object.** If a type has a field resolver that calls a database (e.g., `User.posts`, `Post.author`), it MUST use DataLoader or you will have N+1 queries in production. This is the most common GraphQL mistake.

**2. Disable introspection in production.** Introspection lets clients discover the entire schema — every type, every field, every operation. This is invaluable for development but is an information-disclosure vulnerability in production. Disable it with `spring.graphql.schema.introspection.enabled=false` in your production profile.

**3. Implement query depth and complexity limits.** A malicious client could send an infinitely nested query like `user { friends { friends { friends { ... } } } }` to perform a denial-of-service attack. Use `graphql-java`'s `MaxQueryDepthInstrumentation` and `MaxQueryComplexityInstrumentation` to protect against this.

**4. Don't use GraphQL for everything on your API surface.** Some things — file uploads, binary data, streaming downloads, webhook receivers — are better handled as REST endpoints even if the rest of your API is GraphQL.

**5. Think carefully about mutations that return rich objects.** When a mutation returns the created or updated object, clients can request exactly the fields they need from the mutated result, which avoids a follow-up query. This is a much better pattern than returning just a success boolean.
