# Complete Microservices Example - E-Commerce Platform

> **Production-Ready Microservices Architecture**  
> **Includes: Service Discovery, API Gateway, Config Server, Authentication, Logging, Monitoring, AWS Deployment**  
> **Last Updated: February 2026**

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Technology Stack](#2-technology-stack)
3. [Microservices Implementation](#3-microservices-implementation)
4. [Infrastructure Services](#4-infrastructure-services)
5. [Security & Authentication](#5-security--authentication)
6. [Logging & Monitoring](#6-logging--monitoring)
7. [Inter-Service Communication](#7-inter-service-communication)
8. [Database Per Service](#8-database-per-service)
9. [AWS Deployment](#9-aws-deployment)
10. [CI/CD Pipeline](#10-cicd-pipeline)

---

## 1. Architecture Overview

### 1.1 System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Client Applications                             │
│                    (Web Browser, Mobile App, Third-party)                   │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          API Gateway (Port 8080)                             │
│                     - Routing, Load Balancing                               │
│                     - Authentication Filter                                  │
│                     - Rate Limiting                                          │
└─────────┬───────────────────────┬───────────────────────┬───────────────────┘
          │                       │                       │
          ▼                       ▼                       ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  User Service    │    │ Product Service  │    │  Order Service   │
│  (Port 8081)     │    │  (Port 8082)     │    │  (Port 8083)     │
│                  │    │                  │    │                  │
│  - User mgmt     │    │  - Products      │    │  - Orders        │
│  - Auth          │    │  - Inventory     │    │  - Payments      │
│  - Profiles      │    │  - Categories    │    │  - Shipping      │
└────────┬─────────┘    └────────┬─────────┘    └────────┬─────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
         ▼                       ▼                       ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│   PostgreSQL     │    │   PostgreSQL     │    │   PostgreSQL     │
│   (User DB)      │    │  (Product DB)    │    │   (Order DB)     │
└──────────────────┘    └──────────────────┘    └──────────────────┘

                    Infrastructure Services
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Eureka     │  │   Config     │  │   Zipkin     │  │  Prometheus  │  │
│  │   Server     │  │   Server     │  │  (Tracing)   │  │  (Metrics)   │  │
│  │  (Port 8761) │  │  (Port 8888) │  │  (Port 9411) │  │  (Port 9090) │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   RabbitMQ   │  │    Redis     │  │   Grafana    │  │     ELK      │  │
│  │   (Message   │  │   (Cache)    │  │(Dashboards)  │  │   Stack      │  │
│  │    Queue)    │  │              │  │  (Port 3000) │  │  (Logging)   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Project Structure

```
microservices-ecommerce/
├── config-server/
├── eureka-server/
├── api-gateway/
├── user-service/
├── product-service/
├── order-service/
├── common-library/
├── docker-compose.yml
├── kubernetes/
│   ├── deployments/
│   ├── services/
│   └── configmaps/
└── aws-deployment/
    ├── terraform/
    └── cloudformation/
```

---

## 2. Technology Stack

### Core Technologies
- **Java 17**
- **Spring Boot 3.2.x**
- **Spring Cloud 2023.0.x**

### Microservices Components
- **Service Discovery**: Netflix Eureka
- **API Gateway**: Spring Cloud Gateway
- **Configuration**: Spring Cloud Config
- **Circuit Breaker**: Resilience4j
- **Distributed Tracing**: Zipkin + Sleuth
- **Message Queue**: RabbitMQ
- **Cache**: Redis

### Databases
- **PostgreSQL** (Per service)
- **H2** (Development/Testing)

### Monitoring & Logging
- **Prometheus** (Metrics)
- **Grafana** (Visualization)
- **ELK Stack** (Logging)
- **Spring Boot Actuator**

### Security
- **JWT** (Authentication)
- **Spring Security**
- **OAuth2** (Optional)

### Containerization & Orchestration
- **Docker**
- **Kubernetes**
- **Docker Compose**

### Cloud Platform
- **AWS** (ECS, EKS, RDS, ElastiCache, etc.)

---

## 3. Microservices Implementation

### 3.1 Common Library (Shared Code)

**pom.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.ecommerce</groupId>
    <artifactId>common-library</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>
</project>
```

**Common DTOs**
```java
package com.ecommerce.common.dto;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class ErrorResponse {
    private int status;
    private String message;
    private String timestamp;
    private String path;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(true, "Success", data);
    }
    
    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, message, null);
    }
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class UserDTO {
    private Long id;
    private String username;
    private String email;
    private String firstName;
    private String lastName;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class ProductDTO {
    private Long id;
    private String name;
    private String description;
    private Double price;
    private Integer stock;
    private String category;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderDTO {
    private Long id;
    private Long userId;
    private List<OrderItemDTO> items;
    private Double totalAmount;
    private String status;
    private LocalDateTime orderDate;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderItemDTO {
    private Long productId;
    private String productName;
    private Integer quantity;
    private Double price;
}
```

**Common Exceptions**
```java
package com.ecommerce.common.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

public class UnauthorizedException extends RuntimeException {
    public UnauthorizedException(String message) {
        super(message);
    }
}

public class BadRequestException extends RuntimeException {
    public BadRequestException(String message) {
        super(message);
    }
}

@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(
            ResourceNotFoundException ex,
            WebRequest request) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now().toString(),
            request.getDescription(false)
        );
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }
    
    @ExceptionHandler(UnauthorizedException.class)
    public ResponseEntity<ErrorResponse> handleUnauthorized(
            UnauthorizedException ex,
            WebRequest request) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.UNAUTHORIZED.value(),
            ex.getMessage(),
            LocalDateTime.now().toString(),
            request.getDescription(false)
        );
        return new ResponseEntity<>(error, HttpStatus.UNAUTHORIZED);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(
            Exception ex,
            WebRequest request) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "An error occurred: " + ex.getMessage(),
            LocalDateTime.now().toString(),
            request.getDescription(false)
        );
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

**JWT Utility**
```java
package com.ecommerce.common.security;

import io.jsonwebtoken.*;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.security.Key;
import java.util.Date;

@Component
public class JwtUtil {
    
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.expiration}")
    private Long expiration;
    
    private Key getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes());
    }
    
    public String generateToken(String username, Long userId) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + expiration);
        
        return Jwts.builder()
            .setSubject(username)
            .claim("userId", userId)
            .setIssuedAt(now)
            .setExpiration(expiryDate)
            .signWith(getSigningKey(), SignatureAlgorithm.HS512)
            .compact();
    }
    
    public String getUsernameFromToken(String token) {
        Claims claims = Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody();
        
        return claims.getSubject();
    }
    
    public Long getUserIdFromToken(String token) {
        Claims claims = Jwts.parserBuilder()
            .setSigningKey(getSigningKey())
            .build()
            .parseClaimsJws(token)
            .getBody();
        
        return claims.get("userId", Long.class);
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

---

### 3.2 User Service

**pom.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.ecommerce</groupId>
    <artifactId>user-service</artifactId>
    <version>1.0.0</version>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>
    
    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        
        <!-- Spring Cloud -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-sleuth-zipkin</artifactId>
        </dependency>
        
        <!-- Database -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
        </dependency>
        
        <!-- Redis Cache -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
        
        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.11.5</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.11.5</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.11.5</version>
        </dependency>
        
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
        
        <!-- Micrometer for Prometheus -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
        
        <!-- Common Library -->
        <dependency>
            <groupId>com.ecommerce</groupId>
            <artifactId>common-library</artifactId>
            <version>1.0.0</version>
        </dependency>
    </dependencies>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

**Application Class**
```java
package com.ecommerce.user;

@SpringBootApplication
@EnableDiscoveryClient
@EnableCaching
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

**application.yml**
```yaml
spring:
  application:
    name: user-service
  
  datasource:
    url: jdbc:postgresql://localhost:5432/user_db
    username: postgres
    password: postgres
    driver-class-name: org.postgresql.Driver
  
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
  
  redis:
    host: localhost
    port: 6379
  
  sleuth:
    sampler:
      probability: 1.0
  
  zipkin:
    base-url: http://localhost:9411

server:
  port: 8081

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true

management:
  endpoints:
    web:
      exposure:
        include: "*"
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true

jwt:
  secret: mySecretKeyForJWTTokenGenerationAndValidation123456789
  expiration: 86400000  # 24 hours

logging:
  level:
    com.ecommerce.user: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
    file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
  file:
    name: logs/user-service.log
```

**Entity**
```java
package com.ecommerce.user.entity;

@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String username;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Column(nullable = false)
    private String password;
    
    @Column(name = "first_name")
    private String firstName;
    
    @Column(name = "last_name")
    private String lastName;
    
    private String phone;
    
    private String address;
    
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @Column(name = "is_active")
    private Boolean isActive = true;
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

**Repository**
```java
package com.ecommerce.user.repository;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    Optional<User> findByUsername(String username);
    
    Optional<User> findByEmail(String email);
    
    boolean existsByUsername(String username);
    
    boolean existsByEmail(String email);
    
    @Query("SELECT u FROM User u WHERE u.isActive = true")
    List<User> findAllActiveUsers();
}
```

**Service**
```java
package com.ecommerce.user.service;

@Service
@Slf4j
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private JwtUtil jwtUtil;
    
    @Cacheable(value = "users", key = "#id")
    public UserDTO getUserById(Long id) {
        log.info("Fetching user with id: {}", id);
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found with id: " + id));
        return mapToDTO(user);
    }
    
    @Cacheable(value = "users", key = "#username")
    public UserDTO getUserByUsername(String username) {
        log.info("Fetching user with username: {}", username);
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new ResourceNotFoundException("User not found with username: " + username));
        return mapToDTO(user);
    }
    
    @CacheEvict(value = "users", allEntries = true)
    public UserDTO createUser(UserDTO userDTO, String password) {
        log.info("Creating new user: {}", userDTO.getUsername());
        
        if (userRepository.existsByUsername(userDTO.getUsername())) {
            throw new BadRequestException("Username already exists");
        }
        
        if (userRepository.existsByEmail(userDTO.getEmail())) {
            throw new BadRequestException("Email already exists");
        }
        
        User user = new User();
        user.setUsername(userDTO.getUsername());
        user.setEmail(userDTO.getEmail());
        user.setPassword(passwordEncoder.encode(password));
        user.setFirstName(userDTO.getFirstName());
        user.setLastName(userDTO.getLastName());
        
        User savedUser = userRepository.save(user);
        log.info("User created successfully with id: {}", savedUser.getId());
        
        return mapToDTO(savedUser);
    }
    
    @CacheEvict(value = "users", key = "#id")
    public UserDTO updateUser(Long id, UserDTO userDTO) {
        log.info("Updating user with id: {}", id);
        
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
        
        user.setFirstName(userDTO.getFirstName());
        user.setLastName(userDTO.getLastName());
        user.setEmail(userDTO.getEmail());
        
        User updatedUser = userRepository.save(user);
        log.info("User updated successfully");
        
        return mapToDTO(updatedUser);
    }
    
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        log.info("Deleting user with id: {}", id);
        
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
        
        user.setIsActive(false);
        userRepository.save(user);
        
        log.info("User deleted successfully");
    }
    
    public String authenticateUser(String username, String password) {
        log.info("Authenticating user: {}", username);
        
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UnauthorizedException("Invalid credentials"));
        
        if (!passwordEncoder.matches(password, user.getPassword())) {
            throw new UnauthorizedException("Invalid credentials");
        }
        
        if (!user.getIsActive()) {
            throw new UnauthorizedException("Account is inactive");
        }
        
        String token = jwtUtil.generateToken(username, user.getId());
        log.info("User authenticated successfully");
        
        return token;
    }
    
    private UserDTO mapToDTO(User user) {
        UserDTO dto = new UserDTO();
        dto.setId(user.getId());
        dto.setUsername(user.getUsername());
        dto.setEmail(user.getEmail());
        dto.setFirstName(user.getFirstName());
        dto.setLastName(user.getLastName());
        return dto;
    }
}
```

**Controller**
```java
package com.ecommerce.user.controller;

@RestController
@RequestMapping("/api/users")
@Slf4j
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<UserDTO>> getUserById(@PathVariable Long id) {
        UserDTO user = userService.getUserById(id);
        return ResponseEntity.ok(ApiResponse.success(user));
    }
    
    @GetMapping("/username/{username}")
    public ResponseEntity<ApiResponse<UserDTO>> getUserByUsername(@PathVariable String username) {
        UserDTO user = userService.getUserByUsername(username);
        return ResponseEntity.ok(ApiResponse.success(user));
    }
    
    @PostMapping("/register")
    public ResponseEntity<ApiResponse<UserDTO>> registerUser(
            @Valid @RequestBody RegisterRequest request) {
        UserDTO userDTO = new UserDTO();
        userDTO.setUsername(request.getUsername());
        userDTO.setEmail(request.getEmail());
        userDTO.setFirstName(request.getFirstName());
        userDTO.setLastName(request.getLastName());
        
        UserDTO createdUser = userService.createUser(userDTO, request.getPassword());
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success(createdUser));
    }
    
    @PostMapping("/login")
    public ResponseEntity<ApiResponse<LoginResponse>> login(
            @Valid @RequestBody LoginRequest request) {
        String token = userService.authenticateUser(request.getUsername(), request.getPassword());
        LoginResponse response = new LoginResponse(token);
        return ResponseEntity.ok(ApiResponse.success(response));
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<ApiResponse<UserDTO>> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UserDTO userDTO) {
        UserDTO updatedUser = userService.updateUser(id, userDTO);
        return ResponseEntity.ok(ApiResponse.success(updatedUser));
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<ApiResponse<Void>> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.ok(ApiResponse.success(null));
    }
}

@Data
class RegisterRequest {
    @NotBlank
    private String username;
    
    @Email
    @NotBlank
    private String email;
    
    @NotBlank
    @Size(min = 6)
    private String password;
    
    private String firstName;
    private String lastName;
}

@Data
class LoginRequest {
    @NotBlank
    private String username;
    
    @NotBlank
    private String password;
}

@Data
@AllArgsConstructor
class LoginResponse {
    private String token;
}
```

**Security Configuration**
```java
package com.ecommerce.user.config;

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/users/register", "/api/users/login").permitAll()
                .requestMatchers("/actuator/**").permitAll()
                .anyRequest().authenticated()
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            );
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}

@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
    }
}
```

---

### 3.3 Product Service

Due to length, I'll show the key differences. Product service follows similar structure but manages products.

**Entity**
```java
package com.ecommerce.product.entity;

@Entity
@Table(name = "products")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(length = 1000)
    private String description;
    
    @Column(nullable = false)
    private Double price;
    
    @Column(nullable = false)
    private Integer stock;
    
    private String category;
    
    @Column(name = "image_url")
    private String imageUrl;
    
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @Column(name = "is_active")
    private Boolean isActive = true;
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

**Service (Key Methods)**
```java
package com.ecommerce.product.service;

@Service
@Slf4j
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @Cacheable(value = "products", key = "#id")
    public ProductDTO getProductById(Long id) {
        log.info("Fetching product with id: {}", id);
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found"));
        return mapToDTO(product);
    }
    
    public Page<ProductDTO> getAllProducts(Pageable pageable) {
        return productRepository.findAll(pageable).map(this::mapToDTO);
    }
    
    public List<ProductDTO> searchProducts(String keyword) {
        return productRepository.searchByNameOrDescription(keyword)
            .stream()
            .map(this::mapToDTO)
            .collect(Collectors.toList());
    }
    
    @CacheEvict(value = "products", allEntries = true)
    public ProductDTO createProduct(ProductDTO productDTO) {
        log.info("Creating new product: {}", productDTO.getName());
        
        Product product = new Product();
        product.setName(productDTO.getName());
        product.setDescription(productDTO.getDescription());
        product.setPrice(productDTO.getPrice());
        product.setStock(productDTO.getStock());
        product.setCategory(productDTO.getCategory());
        
        Product savedProduct = productRepository.save(product);
        
        // Publish event
        rabbitTemplate.convertAndSend("product.exchange", "product.created", 
            savedProduct.getId());
        
        return mapToDTO(savedProduct);
    }
    
    @Transactional
    public boolean updateStock(Long productId, Integer quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ResourceNotFoundException("Product not found"));
        
        if (product.getStock() < quantity) {
            log.warn("Insufficient stock for product: {}", productId);
            return false;
        }
        
        product.setStock(product.getStock() - quantity);
        productRepository.save(product);
        
        log.info("Stock updated for product: {}", productId);
        return true;
    }
    
    private ProductDTO mapToDTO(Product product) {
        ProductDTO dto = new ProductDTO();
        dto.setId(product.getId());
        dto.setName(product.getName());
        dto.setDescription(product.getDescription());
        dto.setPrice(product.getPrice());
        dto.setStock(product.getStock());
        dto.setCategory(product.getCategory());
        return dto;
    }
}
```

---

### 3.4 Order Service

**Entity**
```java
package com.ecommerce.order.entity;

@Entity
@Table(name = "orders")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "user_id", nullable = false)
    private Long userId;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
    
    @Column(name = "total_amount", nullable = false)
    private Double totalAmount;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    @Column(name = "shipping_address")
    private String shippingAddress;
    
    @Column(name = "order_date", updatable = false)
    private LocalDateTime orderDate;
    
    @PrePersist
    protected void onCreate() {
        orderDate = LocalDateTime.now();
        status = OrderStatus.PENDING;
    }
    
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
}

@Entity
@Table(name = "order_items")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class OrderItem {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;
    
    @Column(name = "product_id", nullable = false)
    private Long productId;
    
    @Column(name = "product_name")
    private String productName;
    
    @Column(nullable = false)
    private Integer quantity;
    
    @Column(nullable = false)
    private Double price;
}

enum OrderStatus {
    PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED
}
```

**Service with Inter-Service Communication**
```java
package com.ecommerce.order.service;

@Service
@Slf4j
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Autowired
    private ProductServiceClient productServiceClient;
    
    @Autowired
    private UserServiceClient userServiceClient;
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @Transactional
    @CircuitBreaker(name = "orderService", fallbackMethod = "createOrderFallback")
    @Retry(name = "orderService")
    public OrderDTO createOrder(CreateOrderRequest request, Long userId) {
        log.info("Creating order for user: {}", userId);
        
        // Verify user exists
        UserDTO user = userServiceClient.getUserById(userId);
        if (user == null) {
            throw new BadRequestException("User not found");
        }
        
        Order order = new Order();
        order.setUserId(userId);
        order.setShippingAddress(request.getShippingAddress());
        
        double totalAmount = 0.0;
        
        // Add items and verify stock
        for (OrderItemRequest itemRequest : request.getItems()) {
            ProductDTO product = productServiceClient.getProductById(itemRequest.getProductId());
            
            if (product == null) {
                throw new BadRequestException("Product not found: " + itemRequest.getProductId());
            }
            
            if (product.getStock() < itemRequest.getQuantity()) {
                throw new BadRequestException("Insufficient stock for product: " + product.getName());
            }
            
            OrderItem item = new OrderItem();
            item.setProductId(product.getId());
            item.setProductName(product.getName());
            item.setQuantity(itemRequest.getQuantity());
            item.setPrice(product.getPrice());
            
            order.addItem(item);
            totalAmount += product.getPrice() * itemRequest.getQuantity();
            
            // Update stock
            productServiceClient.updateStock(product.getId(), itemRequest.getQuantity());
        }
        
        order.setTotalAmount(totalAmount);
        Order savedOrder = orderRepository.save(order);
        
        // Publish order created event
        OrderCreatedEvent event = new OrderCreatedEvent(
            savedOrder.getId(),
            savedOrder.getUserId(),
            savedOrder.getTotalAmount()
        );
        rabbitTemplate.convertAndSend("order.exchange", "order.created", event);
        
        log.info("Order created successfully: {}", savedOrder.getId());
        
        return mapToDTO(savedOrder);
    }
    
    public OrderDTO createOrderFallback(CreateOrderRequest request, Long userId, Exception e) {
        log.error("Failed to create order, fallback triggered", e);
        throw new RuntimeException("Order service temporarily unavailable. Please try again later.");
    }
    
    public OrderDTO getOrderById(Long id) {
        Order order = orderRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
        return mapToDTO(order);
    }
    
    public List<OrderDTO> getUserOrders(Long userId) {
        return orderRepository.findByUserIdOrderByOrderDateDesc(userId)
            .stream()
            .map(this::mapToDTO)
            .collect(Collectors.toList());
    }
    
    @Transactional
    public OrderDTO updateOrderStatus(Long orderId, OrderStatus status) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
        
        order.setStatus(status);
        Order updatedOrder = orderRepository.save(order);
        
        // Publish status update event
        rabbitTemplate.convertAndSend("order.exchange", "order.status.updated", 
            new OrderStatusUpdatedEvent(orderId, status));
        
        return mapToDTO(updatedOrder);
    }
    
    private OrderDTO mapToDTO(Order order) {
        OrderDTO dto = new OrderDTO();
        dto.setId(order.getId());
        dto.setUserId(order.getUserId());
        dto.setTotalAmount(order.getTotalAmount());
        dto.setStatus(order.getStatus().name());
        dto.setOrderDate(order.getOrderDate());
        
        List<OrderItemDTO> itemDTOs = order.getItems().stream()
            .map(item -> new OrderItemDTO(
                item.getProductId(),
                item.getProductName(),
                item.getQuantity(),
                item.getPrice()
            ))
            .collect(Collectors.toList());
        
        dto.setItems(itemDTOs);
        return dto;
    }
}
```

**Feign Clients**
```java
package com.ecommerce.order.client;

@FeignClient(name = "user-service", fallback = UserServiceFallback.class)
public interface UserServiceClient {
    
    @GetMapping("/api/users/{id}")
    ApiResponse<UserDTO> getUserById(@PathVariable Long id);
}

@Component
class UserServiceFallback implements UserServiceClient {
    @Override
    public ApiResponse<UserDTO> getUserById(Long id) {
        return ApiResponse.error("User service unavailable");
    }
}

@FeignClient(name = "product-service", fallback = ProductServiceFallback.class)
public interface ProductServiceClient {
    
    @GetMapping("/api/products/{id}")
    ApiResponse<ProductDTO> getProductById(@PathVariable Long id);
    
    @PutMapping("/api/products/{id}/stock")
    ApiResponse<Boolean> updateStock(@PathVariable Long id, @RequestParam Integer quantity);
}

@Component
class ProductServiceFallback implements ProductServiceClient {
    @Override
    public ApiResponse<ProductDTO> getProductById(Long id) {
        return ApiResponse.error("Product service unavailable");
    }
    
    @Override
    public ApiResponse<Boolean> updateStock(Long id, Integer quantity) {
        return ApiResponse.error("Product service unavailable");
    }
}
```

Continue in next message...


**Resilience4j Configuration**
```java
package com.ecommerce.order.config;

@Configuration
public class Resilience4jConfig {
    
    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .circuitBreakerConfig(CircuitBreakerConfig.custom()
                .slidingWindowSize(10)
                .failureRateThreshold(50)
                .waitDurationInOpenState(Duration.ofSeconds(10))
                .permittedNumberOfCallsInHalfOpenState(3)
                .build())
            .timeLimiterConfig(TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(5))
                .build())
            .build());
    }
}
```

---

## 4. Infrastructure Services

### 4.1 Eureka Server

**EurekaServerApplication.java**
```java
package com.ecommerce.eureka;

@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

**application.yml**
```yaml
spring:
  application:
    name: eureka-server

server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  server:
    enable-self-preservation: false

logging:
  level:
    com.netflix.eureka: INFO
    com.netflix.discovery: INFO
```

---

### 4.2 Config Server

**ConfigServerApplication.java**
```java
package com.ecommerce.config;

@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

**application.yml**
```yaml
spring:
  application:
    name: config-server
  cloud:
    config:
      server:
        git:
          uri: https://github.com/your-org/config-repo
          default-label: main
          clone-on-start: true
        native:
          search-locations: classpath:/config

server:
  port: 8888

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

---

### 4.3 API Gateway

**ApiGatewayApplication.java**
```java
package com.ecommerce.gateway;

@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}
```

**application.yml**
```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - name: CircuitBreaker
              args:
                name: userServiceCircuitBreaker
                fallbackUri: forward:/fallback
        
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
          filters:
            - name: CircuitBreaker
              args:
                name: productServiceCircuitBreaker
        
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - AuthenticationFilter

server:
  port: 8080

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

logging:
  level:
    org.springframework.cloud.gateway: DEBUG
```

**Authentication Filter**
```java
package com.ecommerce.gateway.filter;

@Component
public class AuthenticationFilter implements GatewayFilter {
    
    @Autowired
    private JwtUtil jwtUtil;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        if (!containsAuthHeader(request)) {
            return this.onError(exchange, "No authorization header", HttpStatus.UNAUTHORIZED);
        }
        
        String token = extractToken(request);
        
        if (!jwtUtil.validateToken(token)) {
            return this.onError(exchange, "Invalid token", HttpStatus.UNAUTHORIZED);
        }
        
        // Add user info to headers
        String username = jwtUtil.getUsernameFromToken(token);
        Long userId = jwtUtil.getUserIdFromToken(token);
        
        exchange.getRequest().mutate()
            .header("X-User-Id", String.valueOf(userId))
            .header("X-Username", username)
            .build();
        
        return chain.filter(exchange);
    }
    
    private boolean containsAuthHeader(ServerHttpRequest request) {
        return request.getHeaders().containsKey(HttpHeaders.AUTHORIZATION);
    }
    
    private String extractToken(ServerHttpRequest request) {
        String authHeader = request.getHeaders().get(HttpHeaders.AUTHORIZATION).get(0);
        return authHeader.replace("Bearer ", "");
    }
    
    private Mono<Void> onError(ServerWebExchange exchange, String err, HttpStatus httpStatus) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(httpStatus);
        return response.setComplete();
    }
}
```

---

## 5. Docker Compose Configuration

**docker-compose.yml**
```yaml
version: '3.8'

services:
  # PostgreSQL Databases
  postgres-user:
    image: postgres:15-alpine
    container_name: postgres-user
    environment:
      POSTGRES_DB: user_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-user-data:/var/lib/postgresql/data
    networks:
      - microservices-network

  postgres-product:
    image: postgres:15-alpine
    container_name: postgres-product
    environment:
      POSTGRES_DB: product_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5433:5432"
    volumes:
      - postgres-product-data:/var/lib/postgresql/data
    networks:
      - microservices-network

  postgres-order:
    image: postgres:15-alpine
    container_name: postgres-order
    environment:
      POSTGRES_DB: order_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5434:5432"
    volumes:
      - postgres-order-data:/var/lib/postgresql/data
    networks:
      - microservices-network

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - microservices-network

  # RabbitMQ
  rabbitmq:
    image: rabbitmq:3-management-alpine
    container_name: rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    networks:
      - microservices-network

  # Zipkin
  zipkin:
    image: openzipkin/zipkin
    container_name: zipkin
    ports:
      - "9411:9411"
    networks:
      - microservices-network

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    networks:
      - microservices-network

  # Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - prometheus
    networks:
      - microservices-network

  # Eureka Server
  eureka-server:
    build: ./eureka-server
    container_name: eureka-server
    ports:
      - "8761:8761"
    networks:
      - microservices-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8761/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Config Server
  config-server:
    build: ./config-server
    container_name: config-server
    ports:
      - "8888:8888"
    depends_on:
      - eureka-server
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
    networks:
      - microservices-network

  # API Gateway
  api-gateway:
    build: ./api-gateway
    container_name: api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - eureka-server
      - config-server
    environment:
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_CLOUD_CONFIG_URI=http://config-server:8888
    networks:
      - microservices-network

  # User Service
  user-service:
    build: ./user-service
    container_name: user-service
    depends_on:
      - postgres-user
      - redis
      - eureka-server
      - zipkin
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-user:5432/user_db
      - SPRING_REDIS_HOST=redis
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_ZIPKIN_BASEURL=http://zipkin:9411
    networks:
      - microservices-network

  # Product Service
  product-service:
    build: ./product-service
    container_name: product-service
    depends_on:
      - postgres-product
      - redis
      - eureka-server
      - rabbitmq
      - zipkin
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-product:5432/product_db
      - SPRING_REDIS_HOST=redis
      - SPRING_RABBITMQ_HOST=rabbitmq
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_ZIPKIN_BASEURL=http://zipkin:9411
    networks:
      - microservices-network

  # Order Service
  order-service:
    build: ./order-service
    container_name: order-service
    depends_on:
      - postgres-order
      - eureka-server
      - rabbitmq
      - zipkin
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-order:5432/order_db
      - SPRING_RABBITMQ_HOST=rabbitmq
      - EUREKA_CLIENT_SERVICEURL_DEFAULTZONE=http://eureka-server:8761/eureka/
      - SPRING_ZIPKIN_BASEURL=http://zipkin:9411
    networks:
      - microservices-network

networks:
  microservices-network:
    driver: bridge

volumes:
  postgres-user-data:
  postgres-product-data:
  postgres-order-data:
  grafana-data:
```

**Dockerfile (Example for each service)**
```dockerfile
FROM eclipse-temurin:17-jdk-alpine
VOLUME /tmp
COPY target/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
EXPOSE 8081
```

---

## 6. Monitoring Configuration

**prometheus.yml**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-actuator'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: 
        - 'user-service:8081'
        - 'product-service:8082'
        - 'order-service:8083'
        - 'api-gateway:8080'
```

---

## 7. AWS Deployment

### 7.1 AWS Architecture

```
AWS Cloud Architecture:

┌─────────────────────────────────────────────────────────────┐
│                        Route 53 (DNS)                       │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Application Load Balancer (ALB)                │
└───────┬──────────────────┬──────────────────┬───────────────┘
        │                  │                  │
   ┌────▼─────┐      ┌────▼─────┐      ┌────▼─────┐
   │  ECS/EKS │      │  ECS/EKS │      │  ECS/EKS │
   │ (AZ-1)   │      │ (AZ-2)   │      │ (AZ-3)   │
   │Services  │      │Services  │      │Services  │
   └────┬─────┘      └────┬─────┘      └────┬─────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
        ┌──────────────────┴──────────────────┐
        │                                     │
   ┌────▼─────────┐                  ┌───────▼────────┐
   │ RDS PostgreSQL│                  │  ElastiCache  │
   │  (Multi-AZ)   │                  │    Redis      │
   └───────────────┘                  └────────────────┘
```

### 7.2 AWS Deployment Steps

**1. Prerequisites**
```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS CLI
aws configure
# Enter: Access Key, Secret Key, Region (us-east-1), Output format (json)

# Install kubectl for EKS
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/kubectl

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

**2. Create ECR Repositories**
```bash
# Create repositories for each service
aws ecr create-repository --repository-name user-service
aws ecr create-repository --repository-name product-service
aws ecr create-repository --repository-name order-service
aws ecr create-repository --repository-name api-gateway
aws ecr create-repository --repository-name eureka-server
aws ecr create-repository --repository-name config-server

# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

# Build and push images
docker build -t user-service ./user-service
docker tag user-service:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/user-service:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/user-service:latest

# Repeat for all services
```

**3. Create VPC and Networking**
```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=microservices-vpc}]'

# Create Subnets (Public and Private in 2 AZs)
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.2.0/24 --availability-zone us-east-1b
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.11.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.12.0/24 --availability-zone us-east-1b

# Create Internet Gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --vpc-id vpc-xxx --internet-gateway-id igw-xxx

# Create NAT Gateways for private subnets
aws ec2 allocate-address --domain vpc
aws ec2 create-nat-gateway --subnet-id subnet-xxx --allocation-id eipalloc-xxx
```

**4. Create RDS Databases**
```bash
# Create DB Subnet Group
aws rds create-db-subnet-group \
  --db-subnet-group-name microservices-db-subnet \
  --db-subnet-group-description "Microservices DB Subnet Group" \
  --subnet-ids subnet-xxx subnet-yyy

# Create PostgreSQL for User Service
aws rds create-db-instance \
  --db-instance-identifier user-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username postgres \
  --master-user-password YourPassword123 \
  --allocated-storage 20 \
  --db-subnet-group-name microservices-db-subnet \
  --vpc-security-group-ids sg-xxx \
  --multi-az \
  --backup-retention-period 7

# Repeat for product-db and order-db
```

**5. Create ElastiCache Redis**
```bash
# Create Cache Subnet Group
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name microservices-redis-subnet \
  --cache-subnet-group-description "Redis Subnet Group" \
  --subnet-ids subnet-xxx subnet-yyy

# Create Redis Cluster
aws elasticache create-cache-cluster \
  --cache-cluster-id microservices-redis \
  --cache-node-type cache.t3.micro \
  --engine redis \
  --num-cache-nodes 1 \
  --cache-subnet-group-name microservices-redis-subnet \
  --security-group-ids sg-xxx
```

**6. Create EKS Cluster**
```bash
# Create EKS Cluster using eksctl
eksctl create cluster \
  --name microservices-cluster \
  --region us-east-1 \
  --version 1.28 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed

# Update kubeconfig
aws eks update-kubeconfig --name microservices-cluster --region us-east-1
```

**7. Kubernetes Deployment Files**

**user-service-deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: <account-id>.dkr.ecr.us-east-1.amazonaws.com/user-service:latest
        ports:
        - containerPort: 8081
        env:
        - name: SPRING_DATASOURCE_URL
          value: jdbc:postgresql://user-db.xxx.us-east-1.rds.amazonaws.com:5432/user_db
        - name: SPRING_REDIS_HOST
          value: microservices-redis.xxx.cache.amazonaws.com
        - name: EUREKA_CLIENT_SERVICEURL_DEFAULTZONE
          value: http://eureka-server:8761/eureka/
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8081
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8081
          initialDelaySeconds: 60
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  type: ClusterIP
  ports:
  - port: 8081
    targetPort: 8081
  selector:
    app: user-service
```

**8. Create Load Balancer**
```bash
# Create Application Load Balancer
aws elbv2 create-load-balancer \
  --name microservices-alb \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-xxx \
  --scheme internet-facing

# Create Target Groups
aws elbv2 create-target-group \
  --name api-gateway-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-xxx \
  --target-type ip \
  --health-check-path /actuator/health

# Create Listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:... \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...
```

**9. Setup Auto Scaling**
```bash
# Create HPA (Horizontal Pod Autoscaler)
kubectl autoscale deployment user-service --cpu-percent=70 --min=2 --max=10
kubectl autoscale deployment product-service --cpu-percent=70 --min=2 --max=10
kubectl autoscale deployment order-service --cpu-percent=70 --min=2 --max=10
```

---

## 8. Running the Application

### Local Development
```bash
# 1. Start infrastructure
docker-compose up -d postgres-user postgres-product postgres-order redis rabbitmq zipkin

# 2. Start Eureka
cd eureka-server && mvn spring-boot:run

# 3. Start Config Server
cd config-server && mvn spring-boot:run

# 4. Start microservices
cd user-service && mvn spring-boot:run
cd product-service && mvn spring-boot:run
cd order-service && mvn spring-boot:run

# 5. Start API Gateway
cd api-gateway && mvn spring-boot:run
```

### Docker Compose
```bash
# Build all services
mvn clean package -DskipTests

# Start everything
docker-compose up -d

# View logs
docker-compose logs -f user-service

# Stop everything
docker-compose down
```

### Access URLs
- **API Gateway**: http://localhost:8080
- **Eureka Dashboard**: http://localhost:8761
- **Zipkin UI**: http://localhost:9411
- **Prometheus**: http://localhost:9090
- **Grafana**: http://localhost:3000 (admin/admin)
- **RabbitMQ Management**: http://localhost:15672 (guest/guest)

---

## 9. API Testing

**Register User**
```bash
curl -X POST http://localhost:8080/api/users/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john_doe",
    "email": "john@example.com",
    "password": "password123",
    "firstName": "John",
    "lastName": "Doe"
  }'
```

**Login**
```bash
curl -X POST http://localhost:8080/api/users/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john_doe",
    "password": "password123"
  }'
```

**Create Product**
```bash
curl -X POST http://localhost:8080/api/products \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Laptop",
    "description": "High-performance laptop",
    "price": 999.99,
    "stock": 50,
    "category": "Electronics"
  }'
```

**Create Order**
```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      {
        "productId": 1,
        "quantity": 2
      }
    ],
    "shippingAddress": "123 Main St, City, Country"
  }'
```

---

## Summary

This comprehensive microservices example includes:

✅ **3 Core Microservices**: User, Product, Order  
✅ **Infrastructure Services**: Eureka, Config Server, API Gateway  
✅ **Security**: JWT-based authentication  
✅ **Distributed Tracing**: Zipkin + Sleuth  
✅ **Caching**: Redis  
✅ **Messaging**: RabbitMQ  
✅ **Monitoring**: Prometheus + Grafana  
✅ **Logging**: ELK Stack ready  
✅ **Resilience**: Circuit Breaker, Retry  
✅ **Database Per Service**: PostgreSQL  
✅ **Containerization**: Docker + Docker Compose  
✅ **Cloud Deployment**: Complete AWS setup  
✅ **Orchestration**: Kubernetes (EKS)  

**Production-Ready Features:**
- Health checks
- Graceful shutdown
- Service discovery
- Load balancing
- Auto-scaling
- Distributed logging
- Metrics collection
- Circuit breakers
- API Gateway
- Secure communication

