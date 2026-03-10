# 13.4 — gRPC with Java
### Phase 13: APIs & Web Services

---

## What Is gRPC and Why Does It Exist?

gRPC is a high-performance, open-source Remote Procedure Call (RPC) framework developed by Google and released publicly in 2015. The name stands for "Google Remote Procedure Call." The goal of gRPC is to make calling a function on a remote service feel as natural and efficient as calling a local function.

To understand why gRPC matters, consider what's happening when two microservices communicate over REST. You have a Java object — say, a `Payment` with a dozen fields. To send it over REST, you serialize it to JSON (a text format), transmit it over the network, the receiving service deserializes the text back into an object, does some work, serializes a response to JSON, sends it back, and the calling service deserializes again. Each step takes time and memory.

gRPC takes a fundamentally different approach. It uses **Protocol Buffers (protobuf)** — a binary serialization format — instead of JSON. Binary data is smaller and faster to serialize/deserialize than text. It uses **HTTP/2** as its transport layer instead of HTTP/1.1, gaining built-in multiplexing (multiple requests on a single connection), header compression, and flow control. The combination of binary serialization and HTTP/2 makes gRPC typically 5–10x faster than REST+JSON for internal service communication.

gRPC is also **strongly contract-driven**. You define your API in a `.proto` file, run a code generator, and get type-safe client and server code in multiple languages. There is no ambiguity about field names or types — the `.proto` file is the authoritative contract.

---

## Protocol Buffers (Protobuf) — The Foundation

Protocol Buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data. Before writing any gRPC code, you must understand protobuf because gRPC is built on top of it.

### Defining Messages

A `.proto` file defines your data structures (messages) and services:

```protobuf
// user.proto

// Always specify the syntax version
syntax = "proto3";

// The Java package where generated classes will be placed
option java_package = "com.example.grpc.user";

// Generate separate class files instead of nested classes (recommended)
option java_multiple_files = true;

// The outer class name (only relevant if java_multiple_files = false)
option java_outer_classname = "UserProto";

// Proto package (separate from Java package)
package com.example.user;

// A "message" is like a Java class — it defines a data structure
message User {
  // Field format: type fieldName = fieldNumber;
  // Field numbers are critical — they identify fields in the binary format.
  // Never change a field number once it's in use — that would break compatibility!
  int64 id = 1;
  string name = 2;
  string email = 3;
  UserRole role = 4;
  repeated Post posts = 5;  // 'repeated' means a list
  google.protobuf.Timestamp created_at = 6;  // Using a well-known type
}

enum UserRole {
  // First value in an enum MUST be 0 in proto3
  USER_ROLE_UNSPECIFIED = 0;
  USER_ROLE_ADMIN = 1;
  USER_ROLE_USER = 2;
  USER_ROLE_MODERATOR = 3;
}

message Post {
  int64 id = 1;
  string title = 2;
  string content = 3;
  int32 like_count = 4;
}

// Request messages — gRPC methods typically take one message as input
message GetUserRequest {
  int64 id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  string password = 3;
}

message DeleteUserRequest {
  int64 id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  string role_filter = 3;  // Optional filter
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total_count = 2;
  bool has_next_page = 3;
}

// A "service" defines the RPC methods your service exposes
// This is the equivalent of your REST controller or GraphQL schema
service UserService {
  // Unary RPC: one request, one response (like a normal REST call)
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);

  // Server streaming: one request, many responses (server pushes a stream)
  rpc ListUsers(ListUsersRequest) returns (stream ListUsersResponse);

  // Client streaming: many requests, one response (client sends a stream)
  rpc BatchCreateUsers(stream CreateUserRequest) returns (ListUsersResponse);

  // Bidirectional streaming: both sides send streams simultaneously
  rpc SyncUsers(stream User) returns (stream User);
}
```

The **field numbers** (the `= 1`, `= 2` etc.) are crucial to understand. In the binary format, protobuf uses these numbers (not field names) to identify fields. This means you can safely rename a field in your `.proto` file as long as you don't change the number. But if you change a field number, or reuse a number for a different field, you'll break binary compatibility with any deployed clients.

---

## gRPC Service Types

gRPC supports four patterns of communication, which gives it much more expressive power than REST.

### 1. Unary RPC — One Request, One Response

This is the simplest pattern and works just like a REST API call. The client sends one request and waits for one response.

```
Client ──────── Request ──────────► Server
Client ◄──────── Response ──────── Server
```

Use this for: typical CRUD operations, queries, any operation where you need one answer to one question.

### 2. Server Streaming RPC — One Request, Stream of Responses

The client sends a single request and the server responds with a stream of messages. The client reads from the stream until the server says it's done.

```
Client ──────── Request ──────────► Server
Client ◄──────── Response 1 ─────── Server
Client ◄──────── Response 2 ─────── Server
Client ◄──────── Response 3 ─────── Server
Client ◄──────── (stream ends) ──── Server
```

Use this for: real-time updates, large dataset pagination (server pushes pages), progress reporting, live monitoring feeds.

### 3. Client Streaming RPC — Stream of Requests, One Response

The client sends a stream of messages and the server responds once after the stream ends.

```
Client ──── Request 1 ──────────► Server
Client ──── Request 2 ──────────► Server
Client ──── Request 3 ──────────► Server
Client ──── (stream ends) ───────► Server
Client ◄──────── Response ───────── Server
```

Use this for: bulk data upload (uploading a large file in chunks), batch inserts, sensor data aggregation.

### 4. Bidirectional Streaming RPC — Both Sides Stream

Both client and server send streams of messages independently and simultaneously. Either side can finish whenever it wants.

```
Client ──── Request 1 ──────────► Server
Client ◄──────── Response A ─────── Server
Client ──── Request 2 ──────────► Server
Client ──── Request 3 ──────────► Server
Client ◄──────── Response B ─────── Server
(either side can close the stream at any time)
```

Use this for: chat applications, collaborative editing, real-time gaming, live dashboards where both sides continuously send data.

---

## Setting Up gRPC in a Spring Boot Project

### Dependencies

```xml
<dependencies>
    <!-- Spring Boot starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <!-- gRPC Spring Boot Starter by LogNet (community starter) -->
    <!-- This provides @GrpcService annotation and auto-configuration -->
    <dependency>
        <groupId>net.devh</groupId>
        <artifactId>grpc-spring-boot-starter</artifactId>
        <version>3.1.0.RELEASE</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- The protobuf-maven-plugin generates Java classes from .proto files -->
        <plugin>
            <groupId>com.github.os72</groupId>
            <artifactId>protoc-jar-maven-plugin</artifactId>
            <version>3.11.4</version>
            <executions>
                <execution>
                    <phase>generate-sources</phase>
                    <goals>
                        <goal>run</goal>
                    </goals>
                    <configuration>
                        <!-- Where to find your .proto files -->
                        <inputDirectories>
                            <include>src/main/proto</include>
                        </inputDirectories>
                        <outputTargets>
                            <!-- Generate Java message classes -->
                            <outputTarget>
                                <type>java</type>
                                <outputDirectory>target/generated-sources/protobuf</outputDirectory>
                            </outputTarget>
                            <!-- Generate gRPC service stubs -->
                            <outputTarget>
                                <type>grpc-java</type>
                                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.61.0</pluginArtifact>
                                <outputDirectory>target/generated-sources/grpc</outputDirectory>
                            </outputTarget>
                        </outputTargets>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

Place your `.proto` files in `src/main/proto/`. When you run `mvn generate-sources` or just `mvn compile`, the plugin generates Java classes in `target/generated-sources/`. These are not files you write or modify — they are generated output.

### What Gets Generated

From the `user.proto` file above, the generator creates:

- `User.java` — a Java class with all the fields, builders, and serialization logic
- `UserRole.java` — a Java enum
- `GetUserRequest.java`, `CreateUserRequest.java`, etc. — message classes for each request/response
- `UserServiceGrpc.java` — contains the service base class (`UserServiceGrpc.UserServiceImplBase`) you extend to implement your server, and the `UserServiceBlockingStub` you use to make client calls

---

## Implementing a gRPC Server

```java
// Extend the generated base class — the class name comes from the service name in the .proto
// @GrpcService registers this as a gRPC service with Spring
@GrpcService
public class UserGrpcServiceImpl extends UserServiceGrpc.UserServiceImplBase {

    private final UserService userService; // Your business logic service

    public UserGrpcServiceImpl(UserService userService) {
        this.userService = userService;
    }

    // =====================================================
    // 1. UNARY RPC: GetUser
    // =====================================================
    @Override
    public void getUser(GetUserRequest request, StreamObserver<User> responseObserver) {
        // StreamObserver is how gRPC communicates responses back to the client

        try {
            User user = userService.findById(request.getId())
                .map(this::toProto) // Convert your domain object to a protobuf message
                .orElseThrow(() -> new NotFoundException("User not found: " + request.getId()));

            // onNext(): send the response
            responseObserver.onNext(user);

            // onCompleted(): signal that we're done (for unary, called once after onNext)
            responseObserver.onCompleted();

        } catch (NotFoundException e) {
            // onError(): signal that an error occurred — use gRPC Status codes (like HTTP status codes)
            responseObserver.onError(
                Status.NOT_FOUND
                    .withDescription(e.getMessage())
                    .asRuntimeException()
            );
        }
    }

    // =====================================================
    // 2. UNARY RPC: CreateUser
    // =====================================================
    @Override
    public void createUser(CreateUserRequest request, StreamObserver<User> responseObserver) {
        try {
            // Validate
            if (request.getName().isBlank() || request.getEmail().isBlank()) {
                responseObserver.onError(
                    Status.INVALID_ARGUMENT
                        .withDescription("Name and email are required")
                        .asRuntimeException()
                );
                return;
            }

            com.example.domain.User created = userService.create(
                request.getName(), request.getEmail(), request.getPassword()
            );

            responseObserver.onNext(toProto(created));
            responseObserver.onCompleted();

        } catch (DuplicateEmailException e) {
            responseObserver.onError(
                Status.ALREADY_EXISTS
                    .withDescription("A user with this email already exists")
                    .asRuntimeException()
            );
        }
    }

    // =====================================================
    // 3. SERVER STREAMING RPC: ListUsers
    // Server sends multiple User messages, one at a time
    // =====================================================
    @Override
    public void listUsers(ListUsersRequest request, StreamObserver<ListUsersResponse> responseObserver) {
        try {
            int pageSize = request.getPageSize() > 0 ? request.getPageSize() : 20;
            int totalUsers = userService.count();
            int page = request.getPage();

            // For large datasets, we could send pages one at a time
            List<com.example.domain.User> users = userService.findPage(page, pageSize);

            ListUsersResponse response = ListUsersResponse.newBuilder()
                .addAllUsers(users.stream().map(this::toProto).collect(Collectors.toList()))
                .setTotalCount(totalUsers)
                .setHasNextPage((page + 1) * pageSize < totalUsers)
                .build();

            responseObserver.onNext(response); // Could call this multiple times for streaming
            responseObserver.onCompleted();

        } catch (Exception e) {
            responseObserver.onError(Status.INTERNAL.withDescription("Server error").asRuntimeException());
        }
    }

    // =====================================================
    // 4. CLIENT STREAMING RPC: BatchCreateUsers
    // Client sends a stream of users, server replies once with results
    // =====================================================
    @Override
    public StreamObserver<CreateUserRequest> batchCreateUsers(StreamObserver<ListUsersResponse> responseObserver) {
        // We return a StreamObserver that RECEIVES the client's stream
        return new StreamObserver<>() {
            private final List<com.example.domain.User> created = new ArrayList<>();

            @Override
            public void onNext(CreateUserRequest request) {
                // Called for EACH message the client sends
                com.example.domain.User user = userService.create(
                    request.getName(), request.getEmail(), request.getPassword()
                );
                created.add(user);
            }

            @Override
            public void onError(Throwable t) {
                // The client stream had an error — handle cleanup
                log.error("Client streaming error during batch create", t);
            }

            @Override
            public void onCompleted() {
                // Client has finished sending all users — send our single response
                ListUsersResponse response = ListUsersResponse.newBuilder()
                    .addAllUsers(created.stream().map(UserGrpcServiceImpl.this::toProto).collect(Collectors.toList()))
                    .setTotalCount(created.size())
                    .build();

                responseObserver.onNext(response);
                responseObserver.onCompleted();
            }
        };
    }

    // Helper: convert domain object to protobuf message
    private User toProto(com.example.domain.User domainUser) {
        return User.newBuilder()
            .setId(domainUser.getId())
            .setName(domainUser.getName())
            .setEmail(domainUser.getEmail())
            .setRole(UserRole.forNumber(domainUser.getRole().ordinal()))
            .build();
    }
}
```

### application.properties for the Server

```properties
# gRPC server port (default is 9090, separate from the HTTP port)
grpc.server.port=9090

# Or set to 0 for a random available port (useful in tests)
# grpc.server.port=0

# For production: use TLS
# grpc.server.security.enabled=true
# grpc.server.security.certificateChain=classpath:certs/server.crt
# grpc.server.security.privateKey=classpath:certs/server.key
```

---

## Implementing a gRPC Client

```java
@Service
public class UserServiceClient {

    // @GrpcClient injects a configured gRPC channel/stub
    // "user-service" refers to the configuration key in application.properties
    @GrpcClient("user-service")
    private UserServiceGrpc.UserServiceBlockingStub blockingStub;
    // Use BlockingStub for synchronous calls (simpler, request blocks until response arrives)

    // For async calls, use:
    // @GrpcClient("user-service")
    // private UserServiceGrpc.UserServiceFutureStub futureStub;

    public com.example.domain.User getUser(Long id) {
        try {
            GetUserRequest request = GetUserRequest.newBuilder()
                .setId(id)
                .build();

            // This call blocks until the response arrives
            User protoUser = blockingStub.getUser(request);
            return fromProto(protoUser);

        } catch (StatusRuntimeException e) {
            // Translate gRPC status codes to your application's exceptions
            if (e.getStatus().getCode() == Status.Code.NOT_FOUND) {
                throw new ResourceNotFoundException("User not found: " + id);
            }
            throw new RuntimeException("gRPC call failed", e);
        }
    }

    public com.example.domain.User createUser(String name, String email, String password) {
        CreateUserRequest request = CreateUserRequest.newBuilder()
            .setName(name)
            .setEmail(email)
            .setPassword(password)
            .build();

        User created = blockingStub.createUser(request);
        return fromProto(created);
    }

    private com.example.domain.User fromProto(User proto) {
        com.example.domain.User user = new com.example.domain.User();
        user.setId(proto.getId());
        user.setName(proto.getName());
        user.setEmail(proto.getEmail());
        return user;
    }
}
```

### application.properties for the Client

```properties
# Configure the address of the remote gRPC server (using the name "user-service")
grpc.client.user-service.address=static://user-service-host:9090
grpc.client.user-service.negotiationType=plaintext  # For development (no TLS)

# For production with TLS:
# grpc.client.user-service.negotiationType=tls

# Or use service discovery (e.g., with Eureka)
# grpc.client.user-service.address=discovery:///user-service
```

---

## gRPC Status Codes

gRPC has its own set of status codes (analogous to HTTP status codes) that represent the outcome of an RPC call.

| Status Code | HTTP Equivalent | When to Use |
|-------------|-----------------|-------------|
| `OK` | 200 | Success |
| `INVALID_ARGUMENT` | 400 | Client provided invalid data |
| `NOT_FOUND` | 404 | Requested resource does not exist |
| `ALREADY_EXISTS` | 409 | Resource already exists |
| `PERMISSION_DENIED` | 403 | Authenticated but not authorized |
| `UNAUTHENTICATED` | 401 | No authentication provided |
| `RESOURCE_EXHAUSTED` | 429 | Rate limit exceeded |
| `UNIMPLEMENTED` | 501 | Method not implemented |
| `INTERNAL` | 500 | Unexpected server error |
| `UNAVAILABLE` | 503 | Server temporarily down |
| `DEADLINE_EXCEEDED` | 504 | Request timed out |

```java
// Setting a status on an error response
responseObserver.onError(
    Status.INVALID_ARGUMENT
        .withDescription("Email address is invalid")
        .augmentDescription(" Provided: " + email)
        .asRuntimeException()
);
```

---

## gRPC vs REST — When to Use Each

**Choose gRPC when:**

- You're doing **internal service-to-service communication** (microservices talking to each other)
- You need **high throughput and low latency** — gRPC's binary format and HTTP/2 transport can be significantly faster than REST+JSON
- You need **streaming** capabilities built into the protocol (not layered on top with WebSockets)
- You want **strong contracts** with compile-time type safety across services
- Your services span **multiple programming languages** (gRPC generates idiomatic clients in Go, Python, Rust, etc.)

**Don't use gRPC when:**

- You're building a **public-facing API** consumed by web browsers directly (browser gRPC support exists but requires a proxy like gRPC-Web)
- Your team or ecosystem has limited gRPC tooling or expertise
- The API is **simple CRUD** and performance isn't a concern
- You need **easy HTTP caching** at the infrastructure level

---

## ⚠️ Important Points & Best Practices

**1. Treat .proto files as your API contract.** Check them into version control. Do code reviews on them. When you change them, you're changing the contract between services. Never change a field number once it has been published — you can add new fields with new numbers, but reusing or changing existing numbers corrupts binary data.

**2. Always use deadline/timeout on client calls.** Without a deadline, a slow or hung gRPC server will block the client thread indefinitely. Set a deadline on every outbound call:

```java
User user = blockingStub
    .withDeadlineAfter(5, TimeUnit.SECONDS)  // This call must complete within 5 seconds
    .getUser(request);
```

**3. Handle `StatusRuntimeException` on the client.** Every gRPC client call can throw `StatusRuntimeException`. Always catch it and translate it into your application's exception hierarchy, rather than letting protobuf-generated exception types propagate through your business logic.

**4. Use TLS in production without exception.** gRPC over plaintext is fine for local development but must never go to production. Configure server and client certificates properly.

**5. Keep your proto messages backward-compatible.** Adding new fields is safe because older clients that don't know about the new field will simply ignore it. Removing fields requires care — mark removed field numbers as `reserved` in your `.proto` to prevent accidental reuse.

```protobuf
message User {
  reserved 7, 8;              // These field numbers are reserved — never reuse them
  reserved "phone_number";    // This field name is reserved too
  int64 id = 1;
  string name = 2;
  // ... other active fields
}
```
