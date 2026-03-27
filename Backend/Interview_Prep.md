## Interview preparation questions (Java)

### Introduction
- Introduction & project info
- Explain the architecture of your recent project
- What was your role in the project?
- How does frontend communicate with backend
- How do you connect React / Angular with backend API's

### Java Core
- Which java versions have you worked on?
- What are the important features of Java 8?
    - functional interfaces
    - lambda expressions
    - Stream API
    - default methods
    - Optional class
    - New Date & Time API
- What are the features of Java 17?
    - sealed classes
    - pattern matching
    - records
- What are the features of Java 21?
    - virtual threads
    - record patterns
- String vs String Builder vs String Buffer
- When to use interface and abstract class and class
- Exception Hierarchy in Java
- Checked vs UnChecked exceptions with examples
- What is volatile?
    - stores the variable in main memory instead of cache so that it always gets latest value
- What is synchonozied?
    - synchronized is a locking mechanism. It ensures that only one thread can execute a block of code or a method at a time.
- Java Memory Model (Stack vs Heap).
- Different layers of JVM
- What is garbage collection?
- How do we skip Garbage collection?
- How to skip finally block?
- Have you worked on collection API?
    - Difference between List, Set and Map.
    - ArrayList vs LinkedList
    - reason for these 2 data structures
    - internal data structure for linked list
    - List of employee data - fetch from 1st or nth position, which is better? which one will you go?
    - insert node in linked list vs arrayList
    - Hashmap vs Linked Hashmap
    - What is hashmap?
    - what is collision in hashmap
    - HashMap internal interface
        - How does it work internally
    
    ```java
    // insert the record
    emp(0, 0, X)
    emp(0, 0, Y)
    emp(0, 0, Z)

    // What is size, what is final value this map returns? 
    ```
- write a query to get total number of emp from each department (10) (using streams and also SQL)
- what is completablefuture in java 8



### Java / Functional Programming
- What is functional interface?
    - Why do we need functional interfaces?
- Lambda expressions - why and how?
- What is default method in interface?
- What is Stream API? why was it introduced?
    - terminal & intermediate operations in java 8
    - predicate vs function java
    - print each character at an even position (indices 0, 2, 4, 6, and 8) from the string "DummyString" using Java 8 Streams
    - stream api in java 8 display the names of all employees belonging to each project A = Raju, Ram, Rakesh, LaxmiB = Raju1, Ram1, RameshC = Raju2, Ram2
    ```java
    List<Employee> empList = Arrays.asList(
            new Employee("Raju", "A"), new Employee("Ram", "A"), 
            new Employee("Rakesh", "A"), new Employee("Laxmi", "A"),
            new Employee("Raju1", "B"), new Employee("Ram1", "B"), 
            new Employee("Ramesh", "B"),
            new Employee("Raju2", "C"), new Employee("Ram2", "C")
        );

        // Grouping by project and mapping to employee names
        Map<String, List<String>> projectMap = empList.stream()
            .collect(Collectors.groupingBy(
                Employee::getProject,
                Collectors.mapping(Employee::getName, Collectors.toList())
            ));

        // Display results
        projectMap.forEach((project, names) -> 
            System.out.println(project + " = " + String.join(", ", names)));
    ```
- Difference between map() and filter()
- What is Optional?
    - Why was Optional introduced?
    - How does it work internally?
- What is memory leak? what are the reasons for memory leak? different tools to identify memory leak?



### Coding / Problem Solving 
- Write java 8 stream code to replace odd numbers with "X"
- Given an integer n, count the total number of digit 1 appearing in all non-negative integers less tha or equal to n.
    ```bash
    n = 9
    [1, 2, 3, .... 9]
    output = 1

    n = 13
    [1, 2, 3, .... 13]
    output = 6

    ```
    - White pseudo code for counting digit 1's
- Input: s = "applebananaorange" > Earliest repeating characters in string
- Leetcode 2 sum
- Longest substring without repeating characters for the string "ababcabb"
- Consider this price of a product from Day 1 to day n - `[23, 55, 3, 53, 14, 78, 43, 1]`. Find the day at which you buy and day you sell to acheive maximum profit. Note: you should buy the product before you sell it.
- `int[] arr = {10, 20, 30, 40, 50};` Sum of array using for loop and using streams
- merge 2 arrays and sort it. Using streams and using Data structures
- Reverse a linkedlist, using data structures logic and using java methods
- find the second largest number in the array



### Spring Core / Spring boot
- What is Spring Framework?
- Difference between Spring and Spring Boot?
- What configurations are needed in spring but not in spring boot?
- What annotations have you used in springboot?
- Explain `@RestController`, `@Service` and `@Repository`.
- `@RestController` vs `@Controller` in spring boot
- How do you develop REST API's in Spring boot?
- Explain Controller-Service-Repository flow?
- How do you handle exceptions in Spring Boot?
- What is @Transactional?
- What is serialization?
- What are HTTP methods? How many are there?
- Write an example of how did you handle exception handling
- `@Autowired` vs `@Component` difference
- what is dependency injection, what are the advantages
- What are different design patterns in springboot
- What are NFR's in springboot
- What are different key components (4) of springboot
- Pre construct & Post construct application
- How to avoid circular dependency?
- what is application.properties
- how to handle multiple exceptions in a single block?
- component scan vs entity scan?
- What are AOP Proxies?
- What is actuator in spring boot?
- what is filter and interceptor in spring boot?

### Spring MVC
- What is DispatchServlet?
- How does request flow work in Spring MVC?
- Explain spring mvc architecture

### Spring Security
- What is authentication and authorization?
- How do you secure REST API's?
- Explain JWT flow
- End to end flow of login and accessing protected API.
- Role based access control.
- Write a snippet to do jwt authentication, how do you set an expiry for JWT token, Any built-in class
- How to use springboot authentication
    - when you are in same session, how will you check authentication
    - handshake will happen in code? or how will you manage API call?


### Spring Data JPA
- What is Spring Data JPA?
- CRUD operations in Spring Data JPA
- Repository interfaces used in JPA.
- How to configure 2 database in springboot?

### Mongo DB
- What is MongoDB?
- Why Mongo DB over MySQL?
- Collection vs document
- How to connect MongoDB with Spring boot?
- What is Spring Data MongoDB?

### Microservices
- What is microservices architecture?
- Monolithic vs microservices
- what is latency and throughput
- What is API Gateway? what are the uses?
- What is API Gateway filters?
- What is circuit breaker?
- What is circuit breaker pattern, how do you implement the same?
- What is Saga pattern
- Multithreading
    - what is race condition
    - How to call parallel API's
    - what are deadlocks, how do we analize the deadlock situation? What are different tools used to analize deadlocks
    - how to avoid deadlocks
- What is retry mechanism?
- What is Kafka?
- What is redis?
- unicast vs multicast in context of gRPC and Kafka
- what is sidecar pattern

### Design & Principles
- What challenges did you face in projects?
- How did you resolve performance issues?
- How do you ensure quality and maintainability?

### Deployement / Devops / Cloud
- Have you deployed applications?
- What is Maven?
- Maven lifecycle phases.
- What are Azure / AWS services have you used?
- How do you manage enviroinment configurations?
- stateful vs stateless lambda

### DB / SQL
- Types of DB joins
- Business use cases of DB joins
- what is DB view? Advantages of writing view?
- What is indexing, procedures and partitions? When to use these?
- What are different kind of index?
- Types of partitions in database?
- range partition vs list partition
- Difference between delete, truncate and drop?
- `where` vs `having`?
- How to optimize slow running query?
- What is table partition?
- Diff between `CHAR` and `VARCHAR`?
- what are different constraints in SQR?
- What is primary key , secondary key?
- What is cross join? use cases?
- Difference between clustered and non clustered index?
- What is scalability?
- does MySQL support cluster index?
- `YOU` in MYSQL
- what is triggers?
- SQL vs NoSQL, why are NoSQL getting popular? How does each perform?


### Miscellaneous
- When to use abstract classes and methods?
- Merge 2 sorted linkedlists
- New features giving performance issues after migration, How to debug?
- What are different cache which you can use?
- Elaborate Event Handler service which you developed
- Exception propagation
- Explain Spring bean lifecycle.
- `@RestController` vs `@Controller`
- What design patterns you are aware and you have used in your application
- Explain factory and Singleton pattern, which pattern you used in your application?
- When do we use Abstract class and Interface in real life
- What kind of AI tools have you used?
- did you implement AI Chatbot?
- one db per microservice or single DB?
- Concurrent Hashmap vs Hashmap?
    - what is syncronized map?
- fail fast and fail safe iterators in java, why is it called fail safe?
- I have 3 transactions, 1 failed, other 2 passed, how do you handle this?
- We have 20 schedulers, I am restarting my app, will all 20 schedulers start at the same time? 
- How to handle global exception, 3 different controllers are there
- Will string be created in heap memory?
- Have you created custom exception?
- Equals and Hashcode method
- When adding 2 employee, how will you ensure they are unique?
- What is hash collision?
- What is hashing principle? What Data structure they use internally?
- ACID property
- In Spring, how many annotations have you used so far
- What is `@Aspect` annotation
- Explain AOP, I need to do logging, How do aspects work?
- We have to select from 10 tables, how to approach here, my SLA is 5/10 secs
- `where` clause consists of multiple columns, which kind of index we can create so that we can get faster results, How many index will you create? 1 or 1 per each column
- salary greater than 1000 and dept should have more than 10 employees
- How many types of partitions in SQL databases?
- Even in MongoDB, have you used?
- In MongoDB, how to return aggregate query
- Have you inserted data or files in MongoDB?
- common use cases of production grade multithreading code in java



### Angular
- Angular version 17 and latest features
- Explain angular lifecycle with some examples
- Observable vs Promise. Any example you have used in your application
- How many types of data-binding do we have in angular
- What is event and property binding?
- What is use of `@Input` and `@Output`
- Did you use Oauth2 implementation.
- Redux and state management in Angular
- Tell me about router and different type of routers, when do you use lazy loading router?
- How do you check if user is unautorized and what steps did you take
- What is activate route observables?
- What is activated route snapshot?
- Redirect to routes, when do you use it and what are disadvantages?
- Explain Route Guards?
- Write a small snippet of lazy loading routing
- should you not improt any modules?
- Observables vs promises vs signals. And also what is push based and pull based?
- What are ngZones in angular?
- How to handle change detections?
- When do you use effect?




### Tech lead
- Introduce
- Explain project and technology stack
- where is application hosted?
- How will you move the code to production? Which you are aware and what you follow?
- For pipelines what are you using?
- Are you aware, how to deploy, I need to pass an variable such as service account and pwd(env specific), how to pass those kind of variables?
- For service account, I need to send an encrypted one, how do you deal with that?
- What is the difference between Angular / React
- What are profiles in springboot?
- How do you implement caching in Springboot? 
    - Any annotation do you remember? 
    - If I need to do third party caching, how do you do it?
    - Are you aware of redis cache
- In springboot, How can you do asynchronous programming?
    - Which API gateway have you used in your application?
    - Explain how to do you build a API gateway in your application?
- Any performance issues when you worked on springboot?
- What is thread pooling?
- In performance wise, what all you can do in caching?
- High level azure active directories, registries and subscriptions
- Dev AI bot - what all different API you used while developing the AI lib
- What's the life of JWT token
- From where are you getting the user credentials?
- Different java memory layers, how do you clean up these areas?
- Different types of garbage collections
- Difference between streaming functions and collection functions
    - what is the internal working mechanism
    - Any advantages over each other?
    - what do you mean by lambda functions
    - Any existing ones, which is functional interfaces


