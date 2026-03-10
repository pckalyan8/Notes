# API & Web Services Notes

This document covers key concepts related to APIs and web services, with a focus on REST, and provides examples using Java and the Spring Framework.

## 8. API & Web Services

### 8.1 REST API Design

#### RESTful Principles and Best Practices

REST (Representational State Transfer) is an architectural style for designing networked applications.

-   **Client-Server**: The client and server are separate. The server provides the API, and the client consumes it.
-   **Stateless**: Every request from a client to a server must contain all the information needed to understand and complete the request. The server does not store any client context between requests.
-   **Cacheable**: Responses should explicitly state whether they are cacheable.
-   **Layered System**: The client does not need to know whether it is connected directly to the end server or to an intermediary.
-   **Uniform Interface**: This is the core principle of REST. It simplifies the architecture and includes:
    -   **Resource Identification**: Resources are identified by URIs.
    -   **Manipulation of Resources Through Representations**: The client modifies resources using representations (e.g., JSON).
    -   **Self-descriptive Messages**: Each message includes enough information to describe how to process it (e.g., using `Content-Type` header).
    -   **HATEOAS (Hypermedia as the Engine of Application State)**: Responses include links to related resources.

#### Resource Naming Conventions

-   Use nouns to represent resources (e.g., `/products`).
-   Be consistent in naming.
-   Use plural nouns for collections.
-   For nested resources, reflect the relationship in the URI (e.g., `/customers/123/orders`).

**Spring Example:**

```java
@RestController
@RequestMapping("/customers")
public class CustomerController {

    @GetMapping("/{customerId}/orders")
    public List<Order> getOrdersForCustomer(@PathVariable Long customerId) {
        // ...
    }
}
```

#### HTTP Status Codes

-   **2xx (Successful)**:
    -   `200 OK`
    -   `201 Created`
    -   `204 No Content`
-   **4xx (Client Error)**:
    -   `400 Bad Request`
    -   `401 Unauthorized`
    -   `403 Forbidden`
    -   `404 Not Found`
-   **5xx (Server Error)**:
    -   `500 Internal Server Error`

**Spring Example:**

```java
@PostMapping("/products")
public ResponseEntity<Product> addProduct(@RequestBody Product product) {
    Product newProduct = productService.save(product);
    return new ResponseEntity<>(newProduct, HttpStatus.CREATED);
}
```

#### Pagination

For APIs that return a large number of items, pagination is essential.

**Spring Example (with Spring Data JPA):**

```java
@GetMapping("/products")
public Page<Product> getProducts(Pageable pageable) {
    return productService.findAll(pageable);
}
```

A client can then request a page like this: `/products?page=0&size=10&sort=name,asc`.

#### Filtering and Sorting

-   **Filtering**: Can be implemented using query parameters.
-   **Sorting**: Spring Data JPA's `Pageable` also supports sorting.

**Spring Example:**

```java
// Using query parameters for filtering
@GetMapping("/products")
public List<Product> getProducts(@RequestParam(required = false) String category) {
    if (category != null) {
        return productService.findByCategory(category);
    }
    return productService.findAll();
}
```

#### API Versioning Strategies

-   **URI Versioning**: `/v1/products`, `/v2/products`. This is the most common method.
-   **Header Versioning**: The version is specified in a custom header (e.g., `X-API-Version: 1`).
-   **Media Type Versioning**: The version is included in the `Accept` header (e.g., `Accept: application/vnd.company.app-v1+json`).

**Spring Example (URI Versioning):**

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductControllerV1 {
    // ...
}

@RestController
@RequestMapping("/api/v2/products")
public class ProductControllerV2 {
    // ...
}
```

#### Rate Limiting

Rate limiting is used to control the amount of incoming traffic to an API. This can be implemented using libraries like Resilience4j or by using an API Gateway.

### 8.2 API Documentation

#### Swagger/OpenAPI Specification

Swagger (now OpenAPI) is a standard for documenting REST APIs. Spring Boot can be integrated with `springdoc-openapi` to automatically generate API documentation.

**Spring Example:**

1.  Add the dependency:
    ```xml
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-ui</artifactId>
        <version>1.6.9</version>
    </dependency>
    ```
2.  Your Spring Boot application will now expose the API documentation at `/swagger-ui.html`.
3.  Use annotations to provide more details:

```java
@RestController
@RequestMapping("/api/products")
@Tag(name = "Product API")
public class ProductController {

    @Operation(summary = "Get a product by its id")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "200", description = "Found the product",
                    content = { @Content(mediaType = "application/json",
                            schema = @Schema(implementation = Product.class)) }),
            @ApiResponse(responseCode = "404", description = "Product not found")
    })
    @GetMapping("/{id}")
    public Product getProductById(@PathVariable Long id) {
        // ...
    }
}
```

#### Postman for API Testing

Postman is a popular tool for testing APIs. You can create and save requests, organize them into collections, and write test scripts.

### 8.3 Other Protocols

#### SOAP Web Services

SOAP (Simple Object Access Protocol) is a protocol for exchanging structured information. It relies on XML and has a more rigid structure than REST.

**Spring-WS Example:**

Spring-WS can be used to create SOAP web services. It involves creating an XSD schema, generating domain objects, and creating an endpoint.

```java
@Endpoint
public class ProductEndpoint {
    private static final String NAMESPACE_URI = "http://www.example.com/products";

    @PayloadRoot(namespace = NAMESPACE_URI, localPart = "getProductRequest")
    @ResponsePayload
    public GetProductResponse getProduct(@RequestPayload GetProductRequest request) {
        // ...
    }
}
```

#### GraphQL Basics

GraphQL is a query language for APIs. It allows clients to request exactly the data they need.

**Spring for GraphQL Example:**

With `spring-boot-starter-graphql`, you can create a GraphQL API.

1.  Define a GraphQL schema (`schema.graphqls`):
    ```graphql
    type Query {
        productById(id: ID): Product
    }

    type Product {
        id: ID
        name: String
    }
    ```
2.  Create a `Controller` to handle the queries:
    ```java
    @Controller
    public class ProductGraphqlController {

        @QueryMapping
        public Product productById(@Argument String id) {
            return productService.getProductById(id);
        }
    }
    ```

#### WebSockets

WebSockets provide full-duplex communication over a single TCP connection.

**Spring WebSocket Example:**

1.  Enable WebSocket support:
    ```java
    @Configuration
    @EnableWebSocket
    public class WebSocketConfig implements WebSocketConfigurer {
        public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
            registry.addHandler(new MyWebSocketHandler(), "/my-handler");
        }
    }
    ```
2.  Create a handler:
    ```java
    public class MyWebSocketHandler extends TextWebSocketHandler {
        @Override
        public void handleTextMessage(WebSocketSession session, TextMessage message) {
            // ...
        }
    }
    ```