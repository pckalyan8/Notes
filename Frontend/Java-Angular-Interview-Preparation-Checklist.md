# Java/Angular Full Stack Interview Preparation Checklist

## 1. Core Java Fundamentals

### 1.1 Object-Oriented Programming (OOP)
- [ ] Classes and Objects
- [ ] Inheritance (single, multilevel, hierarchical)
- [ ] Polymorphism (compile-time and runtime)
- [ ] Encapsulation
- [ ] Abstraction (abstract classes vs interfaces)
- [ ] SOLID principles
- [ ] Design patterns (Singleton, Factory, Builder, Observer, Strategy, etc.)

### 1.2 Java Basics
- [ ] Data types (primitive and reference)
- [ ] Operators and expressions
- [ ] Control flow statements (if-else, switch, loops)
- [ ] Arrays and Strings
- [ ] Type casting and conversion
- [ ] Wrapper classes and autoboxing/unboxing

### 1.3 Exception Handling
- [ ] Try-catch-finally blocks
- [ ] Checked vs unchecked exceptions
- [ ] Custom exceptions
- [ ] Exception hierarchy
- [ ] Try-with-resources

### 1.4 Collections Framework
- [ ] List (ArrayList, LinkedList, Vector)
- [ ] Set (HashSet, LinkedHashSet, TreeSet)
- [ ] Map (HashMap, LinkedHashMap, TreeMap, ConcurrentHashMap)
- [ ] Queue and Deque
- [ ] Iterator and ListIterator
- [ ] Comparable vs Comparator
- [ ] Collections utility class

### 1.5 Multithreading and Concurrency
- [ ] Thread creation (Thread class, Runnable interface)
- [ ] Thread lifecycle
- [ ] Synchronization (synchronized keyword, locks)
- [ ] Volatile keyword
- [ ] Executor framework
- [ ] Callable and Future
- [ ] Thread pools
- [ ] CountDownLatch, CyclicBarrier, Semaphore
- [ ] Deadlock and race conditions

### 1.6 Java 8+ Features
- [ ] Lambda expressions
- [ ] Functional interfaces
- [ ] Stream API (map, filter, reduce, collect)
- [ ] Optional class
- [ ] Method references
- [ ] Default and static methods in interfaces
- [ ] Date and Time API (LocalDate, LocalDateTime)
- [ ] Records (Java 14+)
- [ ] Text blocks (Java 15+)

### 1.7 Memory Management
- [ ] JVM architecture
- [ ] Heap vs stack memory
- [ ] Garbage collection (types and algorithms)
- [ ] Memory leaks
- [ ] String pool

### 1.8 Input/Output
- [ ] File handling
- [ ] Serialization and deserialization
- [ ] NIO (New I/O)
- [ ] Streams (byte and character)

## 2. Spring Framework

### 2.1 Spring Core
- [ ] Dependency Injection (DI)
- [ ] Inversion of Control (IoC)
- [ ] Bean lifecycle
- [ ] Bean scopes (singleton, prototype, request, session)
- [ ] ApplicationContext vs BeanFactory
- [ ] Autowiring
- [ ] Annotations (@Component, @Service, @Repository, @Controller)
- [ ] Java-based configuration vs XML configuration

### 2.2 Spring Boot
- [ ] Auto-configuration
- [ ] Starter dependencies
- [ ] Application properties/YAML configuration
- [ ] Spring Boot annotations (@SpringBootApplication, @Configuration)
- [ ] Embedded servers (Tomcat, Jetty)
- [ ] Actuator for monitoring
- [ ] Profiles (dev, test, prod)
- [ ] CommandLineRunner and ApplicationRunner

### 2.3 Spring MVC
- [ ] MVC architecture
- [ ] DispatcherServlet
- [ ] Controllers and RequestMapping
- [ ] Request handling (@GetMapping, @PostMapping, etc.)
- [ ] @PathVariable, @RequestParam, @RequestBody
- [ ] Model and View
- [ ] Exception handling (@ExceptionHandler, @ControllerAdvice)
- [ ] Interceptors
- [ ] CORS configuration

### 2.4 Spring Data JPA
- [ ] JPA basics and entity mapping
- [ ] @Entity, @Table, @Id, @GeneratedValue
- [ ] Relationships (@OneToOne, @OneToMany, @ManyToOne, @ManyToMany)
- [ ] JpaRepository and CrudRepository
- [ ] Query methods and custom queries
- [ ] @Query annotation (JPQL and native SQL)
- [ ] Pagination and sorting
- [ ] Transactions (@Transactional)
- [ ] Lazy vs eager loading
- [ ] Criteria API

### 2.5 Spring Security
- [ ] Authentication vs authorization
- [ ] Security configuration
- [ ] UserDetailsService
- [ ] Password encoding (BCrypt)
- [ ] JWT (JSON Web Tokens)
- [ ] OAuth 2.0
- [ ] Method-level security (@PreAuthorize, @Secured)
- [ ] CSRF protection
- [ ] Session management

### 2.6 Spring REST APIs
- [ ] RESTful principles
- [ ] HTTP methods (GET, POST, PUT, DELETE, PATCH)
- [ ] Status codes
- [ ] ResponseEntity
- [ ] Content negotiation
- [ ] API versioning
- [ ] HATEOAS
- [ ] API documentation (Swagger/OpenAPI)

## 3. Database & SQL

### 3.1 SQL Fundamentals
- [ ] SELECT, INSERT, UPDATE, DELETE
- [ ] WHERE, ORDER BY, GROUP BY, HAVING
- [ ] Joins (INNER, LEFT, RIGHT, FULL OUTER, CROSS)
- [ ] Subqueries and nested queries
- [ ] Aggregate functions (COUNT, SUM, AVG, MIN, MAX)
- [ ] UNION, INTERSECT, EXCEPT
- [ ] DISTINCT, LIMIT/OFFSET

### 3.2 Database Design
- [ ] Normalization (1NF, 2NF, 3NF, BCNF)
- [ ] Primary keys and foreign keys
- [ ] Indexes (clustered, non-clustered)
- [ ] Constraints (NOT NULL, UNIQUE, CHECK, DEFAULT)
- [ ] Views
- [ ] Stored procedures and functions
- [ ] Triggers

### 3.3 Advanced SQL
- [ ] Window functions (ROW_NUMBER, RANK, DENSE_RANK)
- [ ] CTEs (Common Table Expressions)
- [ ] Transactions (ACID properties)
- [ ] Isolation levels
- [ ] Deadlocks
- [ ] Query optimization and execution plans

### 3.4 NoSQL Databases
- [ ] MongoDB basics
- [ ] Document-based storage
- [ ] CRUD operations in MongoDB
- [ ] Aggregation pipeline
- [ ] Indexing in NoSQL

## 4. Angular Framework

### 4.1 TypeScript Fundamentals
- [ ] Types and interfaces
- [ ] Classes and inheritance
- [ ] Generics
- [ ] Decorators
- [ ] Modules and namespaces
- [ ] Type assertions
- [ ] Enums
- [ ] Union and intersection types

### 4.2 Angular Basics
- [ ] Angular architecture (modules, components, services)
- [ ] Angular CLI commands
- [ ] Project structure
- [ ] Bootstrap process
- [ ] Data binding (interpolation, property, event, two-way)
- [ ] Directives (structural and attribute)
- [ ] Pipes (built-in and custom)
- [ ] Lifecycle hooks (ngOnInit, ngOnDestroy, etc.)

### 4.3 Components
- [ ] Component creation and decoration
- [ ] Component communication (@Input, @Output, EventEmitter)
- [ ] View encapsulation
- [ ] Content projection (ng-content)
- [ ] Dynamic components
- [ ] Component inheritance

### 4.4 Modules
- [ ] NgModule decorator
- [ ] Root module vs feature modules
- [ ] Shared modules
- [ ] Core modules
- [ ] Lazy loading modules
- [ ] Preloading strategies

### 4.5 Services and Dependency Injection
- [ ] Service creation (@Injectable)
- [ ] Dependency injection hierarchy
- [ ] Providers and providedIn
- [ ] Singleton services
- [ ] Multiple instances of services

### 4.6 Routing and Navigation
- [ ] RouterModule configuration
- [ ] Route parameters and query parameters
- [ ] Child routes
- [ ] Route guards (CanActivate, CanDeactivate, Resolve)
- [ ] Lazy loading routes
- [ ] Router events
- [ ] RouterLink and programmatic navigation

### 4.7 Forms
- [ ] Template-driven forms
- [ ] Reactive forms (FormControl, FormGroup, FormBuilder)
- [ ] Form validation (built-in and custom validators)
- [ ] Async validators
- [ ] Dynamic forms
- [ ] Form arrays

### 4.8 HTTP Client
- [ ] HttpClient module
- [ ] GET, POST, PUT, DELETE requests
- [ ] Observables and RxJS
- [ ] HTTP interceptors
- [ ] Error handling
- [ ] Request and response transformation

### 4.9 RxJS and Observables
- [ ] Observable vs Promise
- [ ] Operators (map, filter, mergeMap, switchMap, catchError)
- [ ] Subject, BehaviorSubject, ReplaySubject
- [ ] Subscription management
- [ ] Pipe and composition
- [ ] Error handling operators

### 4.10 State Management
- [ ] Component state
- [ ] Service-based state management
- [ ] NgRx (Store, Actions, Reducers, Effects, Selectors)
- [ ] Redux pattern
- [ ] Immutability

### 4.11 Angular Advanced Topics
- [ ] Change detection strategy
- [ ] OnPush strategy
- [ ] Zones and NgZone
- [ ] Angular Universal (SSR)
- [ ] PWA with Angular
- [ ] Angular animations
- [ ] Standalone components (Angular 14+)

## 5. Frontend Technologies

### 5.1 HTML5
- [ ] Semantic HTML elements
- [ ] Forms and input types
- [ ] Canvas and SVG
- [ ] Web storage (localStorage, sessionStorage)
- [ ] Accessibility (ARIA)

### 5.2 CSS3
- [ ] Selectors and specificity
- [ ] Box model
- [ ] Flexbox
- [ ] CSS Grid
- [ ] Responsive design and media queries
- [ ] CSS animations and transitions
- [ ] Preprocessors (SASS/SCSS, LESS)

### 5.3 JavaScript/ES6+
- [ ] Variables (var, let, const)
- [ ] Arrow functions
- [ ] Promises and async/await
- [ ] Destructuring
- [ ] Spread and rest operators
- [ ] Template literals
- [ ] Modules (import/export)
- [ ] Classes
- [ ] Array methods (map, filter, reduce, forEach)
- [ ] Closures and scope
- [ ] Event loop and asynchronous programming

### 5.4 UI Frameworks
- [ ] Bootstrap
- [ ] Angular Material
- [ ] PrimeNG
- [ ] Responsive design principles

## 6. Microservices Architecture

### 6.1 Microservices Concepts
- [ ] Microservices vs monolithic architecture
- [ ] Service decomposition
- [ ] API Gateway pattern
- [ ] Service discovery (Eureka)
- [ ] Load balancing
- [ ] Circuit breaker pattern (Resilience4j, Hystrix)
- [ ] Distributed tracing

### 6.2 Spring Cloud
- [ ] Config Server
- [ ] Eureka Server
- [ ] API Gateway (Spring Cloud Gateway)
- [ ] Feign Client
- [ ] Load balancer (Ribbon)
- [ ] Circuit breaker

### 6.3 Communication
- [ ] REST vs gRPC
- [ ] Message queues (RabbitMQ, Kafka)
- [ ] Event-driven architecture
- [ ] Synchronous vs asynchronous communication

## 7. DevOps & Tools

### 7.1 Version Control
- [ ] Git basics (init, clone, add, commit, push, pull)
- [ ] Branching and merging
- [ ] Git flow
- [ ] Pull requests and code reviews
- [ ] Resolving conflicts

### 7.2 Build Tools
- [ ] Maven (pom.xml, dependencies, plugins, lifecycle)
- [ ] Gradle
- [ ] npm/yarn for Angular

### 7.3 Testing
- [ ] JUnit 5
- [ ] Mockito
- [ ] Integration testing
- [ ] Test-driven development (TDD)
- [ ] Jasmine and Karma (Angular testing)
- [ ] Protractor/Cypress for E2E testing
- [ ] Code coverage

### 7.4 CI/CD
- [ ] Jenkins
- [ ] GitLab CI/CD
- [ ] GitHub Actions
- [ ] Continuous integration concepts
- [ ] Deployment pipelines

### 7.5 Containerization
- [ ] Docker basics (images, containers, Dockerfile)
- [ ] Docker Compose
- [ ] Kubernetes fundamentals

### 7.6 Cloud Platforms
- [ ] AWS basics (EC2, S3, RDS)
- [ ] Azure fundamentals
- [ ] Google Cloud Platform

## 8. API & Web Services

### 8.1 REST API Design
- [ ] RESTful principles and best practices
- [ ] Resource naming conventions
- [ ] HTTP status codes
- [ ] Pagination
- [ ] Filtering and sorting
- [ ] API versioning strategies
- [ ] Rate limiting

### 8.2 API Documentation
- [ ] Swagger/OpenAPI specification
- [ ] Postman for API testing

### 8.3 Other Protocols
- [ ] SOAP web services
- [ ] GraphQL basics
- [ ] WebSockets

## 9. System Design & Architecture

### 9.1 Design Patterns
- [ ] Creational (Singleton, Factory, Builder, Prototype)
- [ ] Structural (Adapter, Decorator, Proxy, Facade)
- [ ] Behavioral (Observer, Strategy, Command, Template Method)

### 9.2 Architectural Patterns
- [ ] MVC, MVP, MVVM
- [ ] Layered architecture
- [ ] Event-driven architecture
- [ ] CQRS
- [ ] Hexagonal architecture

### 9.3 System Design Concepts
- [ ] Scalability (horizontal vs vertical)
- [ ] Load balancing
- [ ] Caching strategies (Redis, Memcached)
- [ ] Database sharding and replication
- [ ] CAP theorem
- [ ] Consistency patterns

## 10. Security

### 10.1 Web Security
- [ ] OWASP Top 10 vulnerabilities
- [ ] XSS (Cross-Site Scripting)
- [ ] CSRF (Cross-Site Request Forgery)
- [ ] SQL Injection
- [ ] Authentication vs Authorization
- [ ] Password hashing and salting
- [ ] HTTPS and SSL/TLS

### 10.2 Application Security
- [ ] Input validation
- [ ] Security headers
- [ ] Secure coding practices
- [ ] Token-based authentication (JWT)
- [ ] OAuth 2.0 and OpenID Connect

## 11. Performance Optimization

### 11.1 Backend Optimization
- [ ] Database query optimization
- [ ] Connection pooling
- [ ] Caching strategies
- [ ] Asynchronous processing
- [ ] Lazy loading

### 11.2 Frontend Optimization
- [ ] Lazy loading modules and components
- [ ] OnPush change detection
- [ ] Tree shaking
- [ ] Code splitting
- [ ] Bundle size optimization
- [ ] Image optimization
- [ ] Browser caching

## 12. Soft Skills & Best Practices

### 12.1 Coding Best Practices
- [ ] Clean code principles
- [ ] Code reviews
- [ ] Documentation
- [ ] Naming conventions
- [ ] DRY (Don't Repeat Yourself)
- [ ] KISS (Keep It Simple, Stupid)

### 12.2 Problem-Solving
- [ ] Data structures (arrays, linked lists, stacks, queues, trees, graphs)
- [ ] Algorithms (sorting, searching, recursion)
- [ ] Time and space complexity
- [ ] Common coding patterns

### 12.3 Communication
- [ ] Explaining technical concepts
- [ ] Discussing trade-offs
- [ ] Asking clarifying questions
- [ ] Behavioral interview preparation

---

## Tips for Interview Preparation

- Practice coding problems on LeetCode, HackerRank, or CodeSignal
- Build full-stack projects to demonstrate your skills
- Review your past projects and be ready to discuss them
- Prepare questions to ask the interviewer
- Mock interviews with peers or mentors
- Keep up with latest framework versions and features

---

## Progress Tracking

You can track your progress by checking off items as you learn them. Consider using this file with a markdown editor that supports interactive checkboxes, or simply mark items with [x] when completed.

**Example:**
- [x] Completed topic
- [ ] Not yet studied

---

**Good luck with your interviews!** 🚀
