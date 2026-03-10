# Microservices Architecture Notes

This document provides a comprehensive guide to microservices architecture, covering fundamental concepts, Spring Cloud implementations, and communication patterns.

## 6. Microservices Architecture

### 6.1 Microservices Concepts

#### Microservices vs Monolithic Architecture

| Feature                  | Monolithic Architecture                                   | Microservices Architecture                                |
| ------------------------ | --------------------------------------------------------- | --------------------------------------------------------- |
| **Structure**            | A single, unified unit.                                   | A collection of small, independent services.              |
| **Development**          | Simpler to start, but becomes complex as it grows.        | More complex to set up, but easier to manage per service. |
| **Deployment**           | The entire application must be redeployed for any change. | Services can be deployed independently.                   |
| **Scalability**          | Can only scale the entire application.                    | Can scale individual services as needed.                  |
| **Technology Stack**     | Limited to a single technology stack.                     | Polyglot (different services can use different stacks).   |
| **Fault Isolation**      | A failure in one part can bring down the whole system.    | A failure in one service only affects that service.       |

**Diagram:**

```
Monolithic:
+----------------------------------+
|   +--------------------------+   |
|   |                          |   |
|   |   Application Code       |   |
|   |   (UI, Business Logic,   |   |
|   |    Data Access)          |   |
|   |                          |   |
|   +--------------------------+   |
+----------------------------------+

Microservices:
+-----------+   +-----------+   +-----------+
| Service A |   | Service B |   | Service C |
+-----------+   +-----------+   +-----------+
```

#### Service Decomposition

Breaking down a large application into smaller services. Common strategies include:

-   **Decomposition by Business Capability**: Each service is responsible for a specific business function (e.g., `Ordering Service`, `Payment Service`).
-   **Decomposition by Subdomain (DDD)**: Aligns services with the subdomains defined in Domain-Driven Design.

#### API Gateway Pattern

An API Gateway is a single entry point for all client requests, routing them to the appropriate microservice.

**Responsibilities:**

-   **Routing**: Directs client requests to the correct service.
-   **Authentication & Authorization**: Centralizes user authentication.
-   **Rate Limiting**: Protects services from being overloaded.
-   **Load Balancing**: Distributes requests among service instances.

**Diagram:**

```
           +--------------+
Client ->  | API Gateway  | -> Service A
           +--------------+ -> Service B
                            -> Service C
```

#### Service Discovery

In a microservices environment, service instances have dynamic addresses. Service Discovery allows services to find and communicate with each other.

-   **Server-side Discovery (e.g., Eureka)**: A central server (the discovery server) keeps a registry of all service instances.
-   **Client-side Discovery**: The client is responsible for querying the discovery server to find the address of a service.

**Spring Example (Eureka Client):**

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ProductServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProductServiceApplication.class, args);
    }
}
```

#### Load Balancing

Distributes incoming network traffic across multiple service instances to ensure no single instance is overwhelmed.

-   **Client-side Load Balancing**: The client chooses which service instance to connect to.
-   **Server-side Load Balancing**: A hardware or software load balancer distributes traffic.

#### Circuit Breaker Pattern

Prevents an application from repeatedly trying to execute an operation that is likely to fail.

**States:**

-   **Closed**: Requests are allowed to pass through.
-   **Open**: Requests are blocked, and a fallback is returned immediately.
-   **Half-Open**: After a timeout, a limited number of requests are allowed through to test if the service has recovered.

**Libraries**: Resilience4j, Hystrix (now in maintenance mode).

**Spring Example (Resilience4j):**

```java
@RestController
public class MyController {

    @Autowired
    private MyService myService;

    @GetMapping("/data")
    @CircuitBreaker(name = "myService", fallbackMethod = "fallback")
    public String getData() {
        return myService.fetchData();
    }

    public String fallback(Throwable t) {
        return "Fallback response";
    }
}
```

#### Distributed Tracing

Allows you to trace a request as it travels through multiple microservices. Each service adds a unique trace ID to the request, which can be used to correlate logs and identify performance bottlenecks.

**Tools**: Zipkin, Jaeger. Spring Cloud Sleuth can be used to integrate these tools.

### 6.2 Spring Cloud

A framework for building microservices with Spring Boot.

#### Config Server

Provides centralized configuration management for all microservices.

**Example Client `application.yml`:**

```yaml
spring:
  application:
    name: product-service
  cloud:
    config:
      uri: http://config-server:8888
```

#### Eureka Server

A service discovery server.

**Example (Eureka Server setup):**

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

#### API Gateway (Spring Cloud Gateway)

A modern API Gateway built on Spring 5, Project Reactor, and Spring Boot 2.

**Example Route Configuration:**

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/products/**
```

#### Feign Client

A declarative REST client that makes it easy to call other microservices.

**Spring Example:**

```java
@FeignClient(name = "product-service")
public interface ProductServiceClient {
    @GetMapping("/products/{id}")
    Product getProductById(@PathVariable("id") Long id);
}
```

#### Load Balancer

Spring Cloud provides its own client-side load balancer, `Spring Cloud LoadBalancer`, which has replaced Netflix Ribbon.

### 6.3 Communication

#### REST vs gRPC

| Feature      | REST                             | gRPC                              |
| ------------ | -------------------------------- | --------------------------------- |
| **Protocol** | HTTP/1.1                         | HTTP/2                            |
| **Payload**  | JSON                             | Protocol Buffers (binary)         |
| **API Style**| Request-Response                 | Streaming, Request-Response       |
| **Schema**   | OpenAPI                          | `.proto` file                     |

#### Message Queues (RabbitMQ, Kafka)

Used for asynchronous communication between services.

-   **RabbitMQ**: A traditional message broker. Good for complex routing.
-   **Kafka**: A distributed streaming platform. Good for high-throughput, real-time data feeds.

#### Event-Driven Architecture

An architecture where services communicate through events. A service publishes an event when something happens, and other services can subscribe to those events.

**Diagram:**

```
Service A -> [Event Bus] -> Service B
                        -> Service C
```

#### Synchronous vs Asynchronous Communication

-   **Synchronous**: The client sends a request and waits for a response (e.g., REST). This can lead to tight coupling.
-   **Asynchronous**: The client sends a request and does not wait for an immediate response (e.g., using a message queue). This improves decoupling and resilience.
