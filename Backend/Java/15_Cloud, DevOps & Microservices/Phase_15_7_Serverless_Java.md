# Phase 15.7 — Serverless Java

## What is Serverless?

**Serverless** doesn't mean there are no servers. It means *you* don't manage the servers. You provide a function (or a container), define what triggers it, and the cloud provider handles all the infrastructure: provisioning instances, scaling from zero to thousands of concurrent invocations, patching the OS, and billing you only for the exact time your code runs.

The mental model shift is profound. In traditional deployment you think: "I need 3 instances, each with 2 CPU cores and 4GB RAM, running 24/7." In serverless you think: "I have a function that takes 200ms to run. I only pay for those 200ms, and the platform scales it automatically."

This model is ideal for event-driven workloads: processing an S3 upload, responding to an API call, consuming a queue message, running a scheduled job. It is less suited for long-running, stateful workloads or applications that need persistent TCP connections (like traditional WebSocket servers).

---

## AWS Lambda with Java

**AWS Lambda** is the serverless compute service you will encounter most often. You deploy a JAR file (or a container image), configure a trigger (HTTP via API Gateway, S3 event, SQS message, EventBridge schedule, etc.), set the memory allocation, and Lambda does the rest.

### Writing Your First Lambda Function

A Java Lambda function implements one of two interfaces: `RequestHandler<Input, Output>` for synchronous invocations, or `RequestStreamHandler` when you need raw access to the input and output streams.

```java
// OrderHandler.java — handles an API Gateway HTTP request
public class OrderHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {

    // The Jackson ObjectMapper is expensive to create (reflection-heavy).
    // Declaring it as a static field means it is created once when the Lambda
    // container starts (during the cold start), not on every invocation.
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper()
        .registerModule(new JavaTimeModule());

    // Reuse the service layer across invocations — the container stays warm
    // between requests, so static/instance fields persist across calls
    private static final OrderService ORDER_SERVICE = new OrderService();

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent request, Context context) {

        // The context object gives you Lambda metadata: remaining time, request ID, logger
        LambdaLogger logger = context.getLogger();
        logger.log("Processing order request, requestId=" + context.getAwsRequestId());

        try {
            CreateOrderRequest orderRequest = OBJECT_MAPPER.readValue(
                request.getBody(), CreateOrderRequest.class);

            Order order = ORDER_SERVICE.createOrder(orderRequest);

            return APIGatewayProxyResponseEvent.builder()
                .statusCode(201)
                .header("Content-Type", "application/json")
                .body(OBJECT_MAPPER.writeValueAsString(order))
                .build();

        } catch (JsonProcessingException e) {
            logger.log("Invalid request body: " + e.getMessage());
            return APIGatewayProxyResponseEvent.builder()
                .statusCode(400)
                .body("{\"error\": \"Invalid request body\"}")
                .build();
        }
    }
}
```

```yaml
# template.yaml — AWS SAM (Serverless Application Model) deployment descriptor
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 30           # Maximum 30 seconds per invocation
    MemorySize: 512       # 512MB memory (also proportionally allocates CPU)
    Runtime: java21       # Use Java 21 managed runtime
    Environment:
      Variables:
        SPRING_PROFILES_ACTIVE: production

Resources:
  OrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: com.example.OrderHandler::handleRequest
      CodeUri: target/myapp.jar
      Policies:
        - AmazonDynamoDBFullAccess    # IAM permissions (use least privilege in real apps)
        - AmazonSQSFullAccess
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /orders
            Method: POST
        # This same function can also be triggered by SQS:
        # SqsEvent:
        #   Type: SQS
        #   Properties:
        #     Queue: !GetAtt OrderQueue.Arn
        #     BatchSize: 10
```

---

## The Cold Start Problem — The Biggest Challenge for Java in Lambda

This is the most important concept to understand about Java serverless. It is the reason many teams have historically avoided Java in Lambda and chosen Python or Node.js instead.

When Lambda needs to run your function for the first time (or after it has been idle for a while), it must:
1. Provision a new execution environment (container).
2. Download your JAR from S3.
3. Start a JVM process.
4. Load and initialize your classes.
5. Execute your function handler.

Steps 1–4 constitute the **cold start**. For a typical Spring Boot application, this can take anywhere from **5 to 15 seconds** — an eternity for a user waiting for an API response.

Python and Node.js functions cold-start in **50–200ms** because they have no JVM startup overhead.

The Java community has responded with three main strategies to solve this problem: **Minimizing initialization work**, **Provisioned Concurrency**, and **GraalVM Native Image**.

### Strategy 1: Minimize Initialization Work

The simplest approach is to not use a heavy framework in your Lambda. Instead of loading the entire Spring ApplicationContext (which wires hundreds of beans), keep your Lambda handler lean.

```java
// ❌ Don't do this in Lambda — it loads the entire Spring ApplicationContext on every cold start
@SpringBootApplication
public class OrderHandler implements RequestHandler<...> {
    // SpringApplication.run() triggers full DI container initialization
    // This adds 5-10 seconds to your cold start
}

// ✅ Lean Lambda — use dependency injection manually for Lambda-specific code
public class OrderHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    
    // Initialize only what you need, as static fields (created once)
    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final DynamoDbClient DYNAMO = DynamoDbClient.create();
    private static final OrderRepository REPO = new OrderRepository(DYNAMO);
    private static final OrderService SERVICE = new OrderService(REPO);
    
    // Handler is fast — no framework startup
    public APIGatewayProxyResponseEvent handleRequest(...) { ... }
}
```

### Strategy 2: Provisioned Concurrency

AWS Lambda's **Provisioned Concurrency** feature pre-warms a specified number of Lambda execution environments. They are always running (and warmed up), so the first invocation does not hit a cold start. You pay a fixed cost for the provisioned environments even when they receive no traffic — essentially giving you the warm-start speed of traditional servers at a slightly higher cost for those N warm instances, while retaining the auto-scaling behavior above that baseline.

```yaml
# SAM template with provisioned concurrency
OrderFunction:
  Type: AWS::Serverless::Function
  Properties:
    AutoPublishAlias: live   # Required for provisioned concurrency
    ProvisionedConcurrencyConfig:
      ProvisionedConcurrentExecutions: 5   # Keep 5 warm instances ready
```

### Strategy 3: GraalVM Native Image — The Modern Solution

**GraalVM Native Image** compiles your Java application ahead-of-time (AOT) into a native binary executable — no JVM, no bytecode interpretation. The resulting binary starts in **10–50 milliseconds** because all the class loading, reflection resolution, and JIT compilation work that normally happens at JVM startup has already been done at build time.

The tradeoff is a much longer build time (minutes instead of seconds) and restrictions on dynamic Java features like reflection, which must be explicitly declared so GraalVM can include them in the binary.

**Quarkus** and **Micronaut** are Java frameworks designed specifically for GraalVM Native Image from the ground up. **Spring Boot 3+** also supports native compilation via Spring AOT.

---

## Quarkus — Cloud-Native Java from the Ground Up

**Quarkus** is a full-stack Kubernetes-native Java framework by Red Hat, built on the Vert.x reactive engine and MicroProfile standards. It supports both JVM mode (for development speed) and Native mode (for production serverless and containers). The developer experience is excellent — Quarkus's dev mode provides live reload in under a second.

```xml
<!-- pom.xml -->
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-resteasy-reactive-jackson</artifactId>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-hibernate-orm-panache</artifactId>
</dependency>
<dependency>
  <groupId>io.quarkus</groupId>
  <artifactId>quarkus-amazon-lambda-http</artifactId>
</dependency>
```

```java
// OrderResource.java — A Quarkus REST resource
// Quarkus uses JAX-RS annotations (standard Jakarta EE)
@Path("/orders")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class OrderResource {

    // CDI injection — Quarkus's DI container
    @Inject
    OrderService orderService;

    @POST
    public Response createOrder(CreateOrderRequest request) {
        Order order = orderService.createOrder(request);
        return Response.status(Response.Status.CREATED)
                       .entity(order)
                       .build();
    }

    @GET
    @Path("/{id}")
    public Order getOrder(@PathParam("id") Long id) {
        return orderService.findById(id)
            .orElseThrow(() -> new NotFoundException("Order not found: " + id));
    }
}
```

```yaml
# application.properties — Quarkus configuration
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=${DB_USER}
quarkus.datasource.password=${DB_PASSWORD}
quarkus.datasource.jdbc.url=${DB_URL}
quarkus.hibernate-orm.database.generation=validate
```

Building and running as a native executable:

```bash
# Build native image (requires GraalVM or Docker with the native tools)
./mvnw package -Pnative -Dquarkus.native.container-build=true

# The resulting binary is in target/
./target/myapp-runner   # Starts in ~20ms!

# Or build a native Docker image
./mvnw package -Pnative -Dquarkus.container-image.build=true
```

The cold start improvement is dramatic. A Quarkus native executable serving an HTTP endpoint with a database connection starts in about 20–30ms compared to 8–12 seconds for the same application as a Spring Boot fat JAR.

---

## Micronaut — AOT Compilation for the JVM

**Micronaut** takes a different approach to the cold start problem: instead of compiling to native code, it performs **Ahead-of-Time (AOT) dependency injection at compile time**. Traditional Spring Boot discovers beans at runtime through reflection and classpath scanning — this is why it is slow to start. Micronaut generates all the DI wiring code at compile time as regular Java bytecode, eliminating the runtime reflection overhead entirely.

The result is a framework that starts in **500ms–2s** even on the JVM — fast enough for serverless without requiring native compilation, while retaining full JVM capabilities (profiling, debugging, mature GC algorithms).

```java
// OrderController.java — A Micronaut REST controller
@Controller("/orders")
public class OrderController {

    private final OrderService orderService;

    // Constructor injection — Micronaut generates the injection code at compile time
    // No reflection needed at runtime
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @Post
    @Status(HttpStatus.CREATED)
    public Order createOrder(@Body CreateOrderRequest request) {
        return orderService.createOrder(request);
    }

    @Get("/{id}")
    public Optional<Order> getOrder(Long id) {
        return orderService.findById(id);
    }
}

@Singleton   // Micronaut DI annotation
public class OrderService {
    
    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public Order createOrder(CreateOrderRequest request) {
        // Business logic here
        return orderRepository.save(Order.from(request));
    }
}
```

---

## Spring Boot on Lambda — AWS Lambda Web Adapter

If you are heavily invested in Spring Boot, AWS provides the **AWS Lambda Web Adapter** — a thin shim that makes any HTTP server (Spring Boot, Quarkus, or any other) run inside Lambda without code changes. You containerize your Spring Boot app as usual and add the adapter as a Lambda extension. Lambda invocations are translated into HTTP requests to your app's embedded Tomcat/Netty server.

```dockerfile
# Dockerfile for Spring Boot on Lambda using Lambda Web Adapter
FROM public.ecr.aws/lambda/java:21

# Copy the Lambda Web Adapter extension
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.8.1 /lambda-adapter /opt/extensions/lambda-adapter

WORKDIR ${LAMBDA_TASK_ROOT}

# Copy the Spring Boot fat JAR
COPY target/myapp.jar app.jar

# Tell Lambda Web Adapter where your Spring Boot server listens
ENV PORT=8080
ENV AWS_LWA_READINESS_CHECK_PATH=/actuator/health

CMD ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

This approach is the fastest migration path: your Spring Boot app runs on Lambda with near-zero code changes, though cold starts remain a concern for APIs that need sub-second response times.

---

## When to Use Serverless Java (and When Not To)

Serverless Java is an excellent fit for these scenarios: processing events asynchronously (SQS messages, S3 events, EventBridge schedules) where a few seconds of cold start is acceptable; APIs with spiky, unpredictable traffic where the auto-scaling and pay-per-use model saves money compared to always-on instances; lightweight microservices or single-purpose functions where the code is small enough to start quickly even on the JVM; and background jobs like nightly report generation, data pipeline steps, or cache warming tasks.

It is a poor fit for: APIs that require consistently fast response times (under 100ms) for all requests, including cold starts; applications that maintain persistent connections (WebSockets, long polling); computationally intensive workloads that run longer than Lambda's 15-minute maximum; and applications with significant warm-state (large in-memory caches, pre-warmed ML models) that would be destroyed on scale-down.

---

## Best Practices Summary

**Profile your cold start before optimizing.** Measure the actual cold start time of your Lambda function. If it is 300ms and your SLA is 2 seconds, you do not have a problem. Only reach for GraalVM or Quarkus if the measurements justify the added build complexity.

**Size your Lambda memory appropriately.** Lambda pricing is `memory × duration`. Counterintuitively, giving your Lambda more memory often makes it faster (because CPU scales proportionally with memory), which can reduce total cost despite the higher per-millisecond rate. Benchmark with AWS Lambda Power Tuning.

**Minimize your JAR/deployment package size.** Exclude unused Spring Boot starters, use the `spring-boot-thin-launcher` for Lambda, and ensure you are not bundling unused dependencies. A smaller JAR downloads faster from S3 during cold start.

**Externalize configuration via environment variables or AWS SSM Parameter Store/Secrets Manager.** Never bake environment-specific config into your Lambda package. This is both a security principle and a practical one — you should be able to promote the exact same artifact from staging to production by changing only the configuration, not the code.

**Design your functions to be stateless and idempotent.** Lambda containers can be terminated and recreated at any time. Any state must live in an external system (DynamoDB, RDS, ElastiCache). Idempotency — producing the same result if called multiple times with the same input — protects you from at-least-once delivery guarantees in messaging systems and Lambda's own retry behavior on failures.
