# 📙 11.3 — Spring Boot: Auto-Configuration, Starters, Actuator & DevTools

Spring Boot is not a replacement for Spring Framework — it is a set of conventions and tools built on top of it that eliminates the boilerplate of configuring a Spring application. Before Spring Boot, setting up a Spring web application required dozens of lines of XML or Java configuration just to get started. Spring Boot replaced all of that with sensible defaults that you can override when needed.

---

## 🧠 The Philosophy: Convention Over Configuration

Spring Boot's central idea is to look at the libraries on your classpath, make reasonable assumptions about what you want, and configure everything automatically. If you add `spring-boot-starter-data-jpa` to your project, Spring Boot assumes you want a `DataSource`, an `EntityManagerFactory`, and a `TransactionManager` — and creates all three automatically based on properties in your `application.properties`. You override any piece of this only when the default isn't right for your use case.

This is called **auto-configuration**, and it is the most important Spring Boot concept to understand.

---

## 🏗️ Project Structure & Entry Point

```
src/
├── main/
│   ├── java/
│   │   └── com/example/school/
│   │       ├── SchoolApplication.java    ← Entry point
│   │       ├── controller/               ← Web layer
│   │       ├── service/                  ← Business logic
│   │       ├── repository/               ← Data access
│   │       ├── model/                    ← Domain entities
│   │       └── config/                   ← Custom configurations
│   └── resources/
│       ├── application.properties        ← Default config
│       ├── application-dev.properties    ← Dev profile config
│       ├── application-prod.properties   ← Prod profile config
│       └── static/                       ← Static files (HTML, CSS, JS)
└── test/
    └── java/
        └── com/example/school/           ← Test classes
```

```java
// The entry point of every Spring Boot application
@SpringBootApplication // This single annotation does three things:
// 1. @SpringBootConfiguration → Marks this as a Spring configuration class
// 2. @EnableAutoConfiguration → Activates Spring Boot's auto-configuration mechanism
// 3. @ComponentScan          → Scans this class's package and sub-packages for beans
public class SchoolApplication {

    public static void main(String[] args) {
        // Creates the ApplicationContext, starts the embedded server, and runs auto-configuration
        SpringApplication.run(SchoolApplication.class, args);
    }
}
```

---

## 📦 Spring Boot Starters — Curated Dependency Sets

A **starter** is a single Maven/Gradle dependency that pulls in everything you need for a particular feature — the library itself, all its required dependencies, and any necessary auto-configuration. You never have to look up "what version of Hibernate works with this version of Spring Data?". Starters manage all of that for you.

```xml
<!-- Web application (Spring MVC + embedded Tomcat) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- JPA + Hibernate + HikariCP connection pool -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Spring Security (authentication + authorization) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Testing (JUnit 5, Mockito, MockMvc, AssertJ) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- Validation (Hibernate Validator / Jakarta Validation) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- Actuator (health, metrics, monitoring endpoints) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Reactive web (Spring WebFlux) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

---

## ⚙️ Auto-Configuration — How It Works

Auto-configuration classes live inside Spring Boot JAR files. Each is annotated with `@ConditionalOn*` annotations that control whether the configuration activates based on classpath contents, property values, or existing beans.

When you add `spring-boot-starter-data-jpa`, Spring Boot's `HibernateJpaAutoConfiguration` activates automatically because:
1. Hibernate is on the classpath → `@ConditionalOnClass(HibernateJpaVendorAdapter.class)` passes
2. A `DataSource` bean exists (auto-configured from your DB properties) → `@ConditionalOnBean(DataSource.class)` passes
3. You haven't defined your own `EntityManagerFactory` → `@ConditionalOnMissingBean(LocalContainerEntityManagerFactoryBean.class)` passes

```java
// A simplified look at what an auto-configuration class looks like under the hood
// (You don't write this — Spring Boot provides it — but understanding it demystifies magic)
@Configuration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean // Only create this bean if you haven't defined your own DataSource
    public DataSource dataSource(DataSourceProperties properties) {
        // Creates a HikariCP DataSource from application.properties values
        return properties.initializeDataSourceBuilder().build();
    }
}
```

### Overriding Auto-Configuration

You override auto-configuration by simply declaring your own bean of the same type. `@ConditionalOnMissingBean` ensures Spring Boot's default is skipped in favour of yours:

```java
@Configuration
public class CustomDataSourceConfig {

    // Because you've defined your own DataSource bean, Spring Boot's
    // DataSourceAutoConfiguration will skip its default DataSource creation
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setMaximumPoolSize(20); // Custom pool size
        config.setConnectionInitSql("SET search_path TO myschema"); // Custom init SQL
        return new HikariDataSource(config);
    }
}
```

You can also completely exclude an auto-configuration class if you need to:

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class MyApp { ... }
```

---

## 🗂️ application.properties / application.yml

The primary way you configure Spring Boot is through `application.properties` or `application.yml`. Every Spring Boot feature exposes hundreds of configurable properties here — you don't need XML.

```properties
# ── Server ──
server.port=8080
server.servlet.context-path=/api
server.error.include-message=always

# ── DataSource ──
spring.datasource.url=jdbc:postgresql://localhost:5432/school
spring.datasource.username=postgres
spring.datasource.password=secret
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=2

# ── JPA / Hibernate ──
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.open-in-view=false   # IMPORTANT: always disable this in REST APIs

# ── Logging ──
logging.level.root=INFO
logging.level.com.example=DEBUG
logging.level.org.hibernate.SQL=DEBUG
logging.file.name=logs/school-app.log
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n

# ── Profiles ──
spring.profiles.active=dev

# ── Custom application properties ──
app.feature.registration-open=true
app.max-file-upload-mb=10
```

The same configuration expressed in YAML (which many teams prefer for readability with nested properties):

```yaml
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/school
    username: postgres
    password: secret
    hikari:
      maximum-pool-size: 10

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    open-in-view: false

  profiles:
    active: dev

app:
  feature:
    registration-open: true
  max-file-upload-mb: 10
```

---

## 🔧 Conditional Annotations — Fine-Grained Bean Creation

`@ConditionalOn*` annotations control whether a `@Bean` or `@Configuration` class is registered based on runtime conditions. They are the mechanism behind auto-configuration but are also useful in your own application configuration.

```java
@Configuration
public class PaymentConfig {

    // Activate this bean only when the property "app.payment.provider=stripe"
    @Bean
    @ConditionalOnProperty(name = "app.payment.provider", havingValue = "stripe")
    public PaymentService stripePaymentService(@Value("${app.stripe.api-key}") String key) {
        return new StripePaymentService(key);
    }

    // Activate this bean only when the property "app.payment.provider=paypal"
    @Bean
    @ConditionalOnProperty(name = "app.payment.provider", havingValue = "paypal")
    public PaymentService paypalPaymentService() {
        return new PaypalPaymentService();
    }

    // Activate only when a specific class is on the classpath
    @Bean
    @ConditionalOnClass(name = "com.stripe.Stripe")
    public StripeWebhookHandler stripeWebhookHandler() {
        return new StripeWebhookHandler();
    }

    // Activate only when a specific bean does NOT already exist
    @Bean
    @ConditionalOnMissingBean(PaymentService.class)
    public PaymentService defaultPaymentService() {
        return new MockPaymentService(); // Fallback for testing
    }

    // Activate based on a custom condition
    @Bean
    @ConditionalOnExpression("${app.feature.premium-features:false} && '${app.tier}' == 'enterprise'")
    public PremiumAnalyticsService premiumAnalyticsService() {
        return new PremiumAnalyticsService();
    }
}
```

---

## 📊 Spring Boot Actuator — Production Monitoring

Actuator adds production-ready monitoring and management endpoints to your application. These are HTTP endpoints (and optionally JMX MBeans) that expose information about your running application.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```properties
# By default, only /health and /info are exposed over HTTP
# This exposes all built-in endpoints (careful in production — secure these!)
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always

# Change the base path (default is /actuator)
management.endpoints.web.base-path=/management

# Run actuator on a separate port (great for internal monitoring tools)
management.server.port=9090
```

The built-in endpoints are: `/health` (app health status, shows DB/cache/queue connectivity), `/info` (application metadata like version), `/metrics` (JVM memory, CPU, thread counts, custom counters), `/env` (all configuration properties and their sources), `/beans` (all registered beans and their types), `/mappings` (all HTTP routes and their handlers), `/loggers` (view and change log levels at runtime without restart), `/heapdump` (download a heap dump for memory analysis), `/threaddump` (current thread state — useful for diagnosing deadlocks), and `/actuator/shutdown` (gracefully stop the application — disabled by default, never expose publicly).

```java
// Exposing custom health indicators — Spring aggregates all into the /health endpoint
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final RestTemplate restTemplate;
    private final String externalApiUrl;

    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                externalApiUrl + "/health", String.class
            );
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("externalApi", "Reachable")
                    .withDetail("statusCode", response.getStatusCode().value())
                    .build();
            }
            return Health.down().withDetail("statusCode", response.getStatusCode()).build();
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}

// Custom metrics — appear under /actuator/metrics/app.orders.placed
@Service
public class OrderService {

    private final Counter ordersPlacedCounter;

    public OrderService(MeterRegistry meterRegistry) {
        // Register a counter metric — appears in Prometheus/Grafana dashboards
        this.ordersPlacedCounter = Counter.builder("app.orders.placed")
            .description("Total number of orders placed")
            .tag("environment", "production")
            .register(meterRegistry);
    }

    public void placeOrder(Order order) {
        // ... business logic ...
        ordersPlacedCounter.increment(); // Increment the counter
    }
}

// Custom /info endpoint contribution
@Component
public class AppInfoContributor implements InfoContributor {
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("app-version", "2.3.1")
               .withDetail("build-date", "2024-03-15")
               .withDetail("team", "Platform Engineering");
    }
}
```

---

## 🔥 Spring Boot DevTools — Faster Development

DevTools adds development-time conveniences that significantly speed up your development cycle.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional> <!-- Not included in the final production JAR -->
</dependency>
```

DevTools provides **automatic restart** — when files in the classpath change (you recompile), it restarts the application context using a custom classloader. The restart is much faster than a full JVM restart (typically under 1 second vs 5–15 seconds) because the base classloader (which holds library classes) is not recreated. It also provides **LiveReload** integration — a browser plugin that automatically reloads the browser when static resources or templates change. Additionally, it sets development-friendly property defaults: disables template caching, enables debug logging for web requests, and always shows the full error stack trace.

---

## 🏗️ Fat JAR — Deploying Spring Boot Applications

Spring Boot packages your application into a **self-contained executable JAR** that includes the embedded Tomcat server and all dependencies. You can run it anywhere Java is installed — no external server required.

```bash
# Build the fat JAR (all dependencies bundled inside)
mvn clean package

# Run it — no Tomcat installation needed
java -jar target/school-app-1.0.0.jar

# Override properties at startup — useful for different environments
java -jar school-app-1.0.0.jar \
  --spring.profiles.active=prod \
  --server.port=9080 \
  --spring.datasource.url=jdbc:postgresql://prod-db:5432/school
```

The Spring Boot Maven Plugin handles building the fat JAR. It repackages the standard JAR into a special format with an embedded server, a custom classloader, and all dependencies nested inside.

---

## 🐳 Spring Boot with Docker

```dockerfile
# Multi-stage build: Stage 1 compiles; Stage 2 creates the minimal runtime image
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml mvnw ./
COPY .mvn .mvn
# Download dependencies first (cached layer — only re-runs if pom.xml changes)
RUN ./mvnw dependency:go-offline -q
COPY src ./src
# Build the layered JAR (Spring Boot's layered format for better Docker caching)
RUN ./mvnw package -DskipTests

# Stage 2: Runtime image — much smaller than the build image
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
# JVM tuning for containers: respect container memory limits instead of host RAM
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## ✅ Best Practices & Important Points

**Always set `spring.jpa.open-in-view=false`.** The default value of `true` keeps the Hibernate session open for the entire duration of the HTTP request — including view rendering. This allows lazy loading in view templates but causes silent extra SQL queries and holds database connections much longer than necessary. Disable it and load all required data explicitly in your service layer.

**Never expose all Actuator endpoints publicly in production.** Endpoints like `/env`, `/heapdump`, and `/beans` expose sensitive internals of your application. Either run Actuator on a separate management port that is only accessible to internal tools, or use Spring Security to restrict access.

**Use `@ConfigurationProperties` and validate your properties at startup.** If your application has required properties (like a payment API key), annotate the `@ConfigurationProperties` class with `@Validated` and use `@NotBlank` on required fields. Spring Boot will refuse to start if mandatory configuration is missing — this is far better than discovering the problem when the first request fails in production.

**Understand the build lifecycle.** The `spring-boot-maven-plugin` must be in your `pom.xml` for `mvn package` to produce an executable fat JAR. Without it, `mvn package` produces a regular JAR without the embedded server.

**Keep `spring-boot-devtools` out of your production build.** Mark it `<optional>true</optional>` and `<scope>runtime</scope>` in Maven, or use Gradle's `developmentOnly` configuration. DevTools disables certain caches and adds overhead that hurts production performance.

---

## 🔑 Key Summary

```
@SpringBootApplication     → Combines @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
Auto-configuration         → Spring Boot configures beans based on classpath + properties; override with your own @Bean
Starters                   → Curated dependency sets; starter-web, starter-data-jpa, starter-security, etc.
@ConditionalOnProperty     → Create beans only when a specific property has a specific value
@ConditionalOnMissingBean  → Create a default bean only if no other bean of that type exists
application.properties     → Primary configuration file; supports profiles (application-dev.properties)
spring.profiles.active     → Selects which profile-specific properties file to load
Actuator                   → /health, /metrics, /env, /beans — production monitoring endpoints
DevTools                   → Auto-restart + LiveReload for faster development cycles
Fat JAR                    → Self-contained executable: java -jar app.jar (no external server needed)
spring.jpa.open-in-view    → Always set to false in REST APIs to avoid connection leaks
```
