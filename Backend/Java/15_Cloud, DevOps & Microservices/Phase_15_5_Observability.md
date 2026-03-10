# Phase 15.5 — Observability

## The Three Pillars of Observability

Observability is the ability to understand what is happening inside your system by examining its outputs. The concept comes from control theory: a system is "observable" if you can determine its internal state from its external outputs alone. In software engineering, observability is built on three pillars: **Logging**, **Metrics**, and **Tracing**. Each pillar answers a different question.

**Logging** answers: "What happened?" Logs are time-stamped records of discrete events — "user logged in", "order #12345 created", "database connection failed". They are the narrative record of your application's activity.

**Metrics** answer: "How is the system performing right now and over time?" Metrics are numeric measurements aggregated over time — request rate, error rate, response latency percentiles, heap memory usage, active threads. They are ideal for alerting and capacity planning.

**Tracing** answers: "How did this specific request flow through the system?" A trace follows a single request as it moves through multiple services and components, recording the time each step took. It is invaluable for diagnosing performance bottlenecks in distributed systems.

---

## Logging with SLF4J and Logback

**SLF4J (Simple Logging Facade for Java)** is an abstraction layer — your code always writes to the SLF4J API, and the actual logging implementation (Logback, Log4j2, java.util.logging) is wired in at runtime. This means you can switch logging backends without touching your application code.

Spring Boot uses Logback as its default implementation. It is auto-configured, so you get sensible console logging out of the box.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class OrderService {

    // Always declare the logger as a private static final field.
    // Using the class itself as the logger name makes it easy to configure per-package.
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    // If you use Lombok: @Slf4j annotation auto-generates this field as "log"

    public Order createOrder(CreateOrderRequest request) {
        // Use parameterized logging — NEVER string concatenation.
        // With "log.info("Creating order for " + userId)", the string is built
        // even if INFO logging is disabled. With parameterized form, it is lazy.
        log.info("Creating order for customer={}, items={}", 
                 request.getCustomerId(), request.getItems().size());

        Order order = buildOrder(request);

        try {
            orderRepository.save(order);
            log.info("Order created successfully orderId={}, customerId={}", 
                     order.getId(), order.getCustomerId());
        } catch (DataAccessException e) {
            // Use log.error for exceptions — it records the full stack trace
            log.error("Failed to persist order for customerId={}", 
                      request.getCustomerId(), e);
            throw new OrderCreationException("Could not create order", e);
        }
        return order;
    }
}
```

### Log Levels — Use Them Deliberately

**ERROR** — The application cannot continue a meaningful operation. Requires immediate investigation. Examples: database connection lost, unexpected exception in critical path.

**WARN** — Something unexpected happened but the application recovered or degraded gracefully. Examples: a retry was needed, a deprecated API was called, a configuration value is missing (falling back to default).

**INFO** — Important business events. Examples: order created, user authenticated, scheduled job started/completed. INFO logs should tell the story of what your application is doing without being chatty.

**DEBUG** — Detailed diagnostic information useful during development or when investigating a specific problem. Should not be enabled in production normally because the volume is too high.

**TRACE** — Even more granular than DEBUG. Typically used to trace method entry/exit in troublesome code paths.

### Structured Logging — JSON Logs

In production, especially with multiple service instances, plain-text log lines are nearly impossible to search and aggregate efficiently. **Structured logging** outputs logs as JSON objects, making each field individually searchable in log aggregation platforms like Elasticsearch/Kibana (the ELK stack).

```xml
<!-- logback-spring.xml — custom Logback configuration for Spring Boot -->
<configuration>
    <springProfile name="production">
        <!-- Use Logstash encoder to output structured JSON -->
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <!-- Add the application name to every log entry -->
                <customFields>{"app":"order-service","env":"production"}</customFields>
            </encoder>
        </appender>
    </springProfile>

    <springProfile name="development">
        <!-- Human-readable format for local development -->
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </springProfile>
    
    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

A structured JSON log entry looks like this, which Elasticsearch can index field by field:

```json
{
  "timestamp": "2025-03-01T10:23:45.123Z",
  "level": "INFO",
  "logger": "com.example.OrderService",
  "message": "Order created successfully",
  "app": "order-service",
  "env": "production",
  "traceId": "3d2f1a4b8c6e7f2a",
  "spanId": "9e1f2d3a",
  "customerId": "C-98765",
  "orderId": "O-12345"
}
```

### MDC — Mapped Diagnostic Context

**MDC (Mapped Diagnostic Context)** is a thread-local map of key-value pairs that is automatically appended to every log line on that thread. It is the standard way to add contextual information (like a request ID or user ID) to all logs within a request, without passing that information explicitly to every method call.

```java
@Component
public class RequestLoggingFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // Extract the correlation ID from the incoming request header
        // (set by the API gateway) or generate a new one
        String correlationId = Optional
            .ofNullable(httpRequest.getHeader("X-Correlation-ID"))
            .orElse(UUID.randomUUID().toString());

        String userId = extractUserId(httpRequest);

        // Put values into MDC — they will appear in ALL logs on this thread
        MDC.put("correlationId", correlationId);
        MDC.put("userId", userId);
        MDC.put("requestPath", httpRequest.getRequestURI());

        try {
            chain.doFilter(request, response);
        } finally {
            // CRITICAL: Always clear MDC in a finally block.
            // HTTP server threads are pooled and reused. If you don't clear MDC,
            // the next request on this thread will inherit the previous request's context.
            MDC.clear();
        }
    }
}
```

Now every `log.info(...)` call within that request thread will automatically include `correlationId`, `userId`, and `requestPath` in the output.

---

## Metrics with Micrometer and Prometheus

**Micrometer** is to metrics what SLF4J is to logging — a vendor-neutral facade. Your code writes metrics using the Micrometer API, and the backend (Prometheus, Datadog, CloudWatch, InfluxDB) is plugged in as a dependency. Spring Boot auto-configures Micrometer and exposes metrics via the `/actuator/metrics` and `/actuator/prometheus` endpoints.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  metrics:
    tags:
      # These tags are added to EVERY metric, useful for filtering in Grafana
      application: order-service
      environment: production
```

### Creating Custom Metrics

Spring Boot auto-instruments many metrics (JVM heap, GC pauses, HTTP request rates, datasource connection pool). For business-level metrics, you create custom meters:

```java
@Service
public class OrderService {

    private final MeterRegistry registry;
    private final Counter ordersCreatedCounter;
    private final Counter orderFailuresCounter;
    private final Timer orderProcessingTimer;
    private final DistributionSummary orderValueSummary;

    public OrderService(MeterRegistry registry) {
        this.registry = registry;
        
        // Counter — only goes up, counts occurrences
        this.ordersCreatedCounter = Counter.builder("orders.created")
            .description("Total number of orders successfully created")
            .tag("type", "standard")     // Tags allow filtering/grouping in Grafana
            .register(registry);

        this.orderFailuresCounter = Counter.builder("orders.failed")
            .description("Total number of order creation failures")
            .register(registry);

        // Timer — measures duration and count of events
        this.orderProcessingTimer = Timer.builder("orders.processing.duration")
            .description("Time taken to process an order")
            .publishPercentiles(0.5, 0.95, 0.99)   // Track median and tail latency
            .register(registry);

        // DistributionSummary — measures distribution of values (e.g., order amounts)
        this.orderValueSummary = DistributionSummary.builder("orders.value")
            .description("Distribution of order values in USD cents")
            .baseUnit("cents")
            .register(registry);
    }

    public Order createOrder(CreateOrderRequest request) {
        // Record the time taken for the entire operation
        return orderProcessingTimer.record(() -> {
            try {
                Order order = processOrderInternal(request);
                ordersCreatedCounter.increment();
                orderValueSummary.record(order.getTotalCents());
                return order;
            } catch (Exception e) {
                orderFailuresCounter.increment();
                throw e;
            }
        });
    }
}
```

### Prometheus and Grafana

**Prometheus** is a time-series database that **scrapes** (polls) your application's `/actuator/prometheus` endpoint at a configured interval (e.g., every 15 seconds) and stores the metrics. It uses a powerful query language called **PromQL** to filter and aggregate metrics.

**Grafana** is a visualization platform that connects to Prometheus and displays metrics as dashboards with graphs, gauges, and heatmaps. Grafana also handles alerting — sending notifications to Slack, PagerDuty, or email when a metric crosses a threshold.

```yaml
# prometheus.yml — Prometheus scrape configuration
scrape_configs:
  - job_name: 'spring-boot-services'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets:
          - 'order-service:8080'
          - 'payment-service:8080'
          - 'inventory-service:8080'
    # In Kubernetes, use service discovery instead of static_configs:
    # kubernetes_sd_configs:
    #   - role: pod
```

Useful PromQL queries you should know:

```promql
# Request rate per second over the last 5 minutes (by HTTP status code)
rate(http_server_requests_seconds_count{application="order-service"}[5m])

# 99th percentile response latency
histogram_quantile(0.99, 
  rate(http_server_requests_seconds_bucket{application="order-service"}[5m]))

# Error rate (5xx responses) as a percentage of total requests
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count[5m])) * 100

# JVM heap used vs. committed
jvm_memory_used_bytes{area="heap"}
jvm_memory_committed_bytes{area="heap"}
```

---

## Distributed Tracing with OpenTelemetry

**OpenTelemetry (OTel)** is the CNCF standard for distributed tracing and is increasingly used for metrics and logging too. It provides a vendor-neutral SDK and a collector that can export to any backend (Jaeger, Zipkin, Datadog, Honeycomb, etc.). Spring Boot 3+ uses Micrometer Tracing, which can bridge to the OpenTelemetry SDK.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 0.1       # Sample 10% in production to reduce overhead
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
```

Once configured, Spring Boot automatically instruments HTTP requests, RestTemplate/WebClient calls, and database queries — propagating trace IDs through the `traceparent` HTTP header across service boundaries.

---

## Centralized Logging — ELK Stack

The **ELK Stack** (Elasticsearch, Logstash, Kibana) is the most widely deployed centralized logging solution for Java applications.

**Logstash** is a data processing pipeline that ingests log data from multiple sources (files, Beats agents, syslog), parses and transforms it, and ships it to Elasticsearch.

**Elasticsearch** is a distributed search and analytics engine that stores and indexes the log data. It allows extremely fast full-text search across billions of log lines.

**Kibana** is the visualization layer — you use it to search logs, build dashboards, and set up alerts.

A lightweight alternative to Logstash is **Filebeat** (a Go-based agent with low resource usage). In Kubernetes, a **DaemonSet** runs a Filebeat pod on every node, collecting logs from all container stdout streams and shipping them to Elasticsearch.

```yaml
# filebeat-daemonset.yml (simplified)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    spec:
      containers:
        - name: filebeat
          image: docker.elastic.co/beats/filebeat:8.12.0
          volumeMounts:
            # Mount the node's container log directory
            - name: varlog
              mountPath: /var/log/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log/containers
```

---

## Best Practices Summary

**Log events, not states.** "User logged in at 10:23" is useful. "User is logged in" is not a good log entry — that belongs in a database. Think of your logs as an event stream.

**Correlate logs with traces.** When using distributed tracing, configure your logging to include the `traceId` in every log line. Spring Boot does this automatically when Micrometer Tracing is on the classpath. This allows you to go from a Grafana alert → find the trace in Jaeger → find the exact log lines from all services involved in that trace in Kibana.

**Do not log sensitive data.** Never log passwords, credit card numbers, full Social Security numbers, or PII. Use masking in your logging configuration if the data flows through frameworks you do not fully control (e.g., incoming JSON bodies).

**Use the RED method for service metrics.** For any service, the three most important metrics are: **R**ate (requests per second), **E**rrors (error rate percentage), and **D**uration (latency distribution, especially p95 and p99). Set alerts on all three.

**Alert on symptoms, not causes.** Alert on "error rate > 1%" (a user-impacting symptom) rather than "CPU > 80%" (an internal cause that may or may not matter). False alerts cause alert fatigue, where engineers start ignoring alerts — the most dangerous operational state.

**Make metrics actionable.** Every metric you track should have a clear owner, a documented threshold, and a runbook describing what to do when an alert fires. A metric nobody acts on is just noise.
