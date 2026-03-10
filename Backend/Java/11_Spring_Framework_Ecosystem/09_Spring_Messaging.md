# 📘 11.9 — Spring Messaging: Kafka, RabbitMQ, WebSocket & Spring Integration

Messaging is the backbone of any system that needs components to communicate without being tightly coupled in time or space. Instead of Service A calling Service B directly and waiting for a response, Service A publishes a message to a broker, and Service B consumes it whenever it's ready. The services are decoupled — they don't need to be available simultaneously, they can scale independently, and failures in one don't cascade to the other. This chapter covers the messaging technologies and Spring abstractions you'll use in real-world applications.

---

## 🧠 Why Messaging? The Core Value Proposition

Three architectural benefits drive the adoption of messaging in production systems.

**Temporal decoupling** means the sender and receiver don't need to be available at the same time. If the email service is down when an order is placed, the message waits in the queue. When the service recovers, it processes the backlog. Without messaging, the order service would fail (or need complex retry logic) whenever the email service was unavailable.

**Load levelling** means a burst of incoming events is absorbed by the message queue rather than hitting your service directly. If one thousand orders arrive in ten seconds, they queue up and your order-processing service handles them at its own pace — say, one hundred per second — without being overwhelmed.

**Service decoupling** means neither service needs to know about the other. The order service publishes a `OrderPlacedEvent` and has no idea whether zero, one, or ten other services consume it. You can add a new consumer (say, a fraud detection service) without touching the producer. This is the foundation of event-driven architecture.

---

## 📨 Part 1 — Apache Kafka

Apache Kafka is a distributed, fault-tolerant, high-throughput event streaming platform. Unlike traditional message queues where messages are deleted after consumption, Kafka retains messages for a configurable period (the **retention period**) — meaning messages can be replayed, and new consumers can read the entire history.

The key concepts are a **Topic** (a named channel — like a table in a database), a **Partition** (topics are split into partitions for parallelism; each partition is an ordered, immutable log), a **Producer** (writes messages to a topic), a **Consumer** (reads messages from a topic from a specific offset), and a **Consumer Group** (a set of consumers that collectively consume a topic — each partition is assigned to exactly one consumer in the group, enabling parallel processing).

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: school-app-group
      auto-offset-reset: earliest       # Start from the beginning for new consumer groups
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.events"
    producer:
      key-serializer:   org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all          # Wait for all replicas to acknowledge the write (strongest durability)
      retries: 3
```

### 1.1 Producing Messages with KafkaTemplate

```java
// The event class — sent as a JSON payload
public record StudentEnrolledEvent(
    Long   studentId,
    Long   courseId,
    String studentEmail,
    String courseName,
    LocalDateTime enrolledAt
) {}

// The producer service
@Service
@Slf4j
public class EnrollmentEventProducer {

    // KafkaTemplate is the primary Spring abstraction for sending messages
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public EnrollmentEventProducer(KafkaTemplate<String, Object> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publishEnrollmentEvent(StudentEnrolledEvent event) {
        // The message key determines which partition the message goes to.
        // Using studentId as the key guarantees all events for the same student
        // go to the same partition — preserving per-student ordering.
        String messageKey = "student-" + event.studentId();

        // send() returns a CompletableFuture<SendResult>
        // whenComplete lets you react to success or failure asynchronously
        kafkaTemplate.send("student-enrollments", messageKey, event)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    log.info("Published enrollment event for student {} to partition {} offset {}",
                        event.studentId(),
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                } else {
                    log.error("Failed to publish enrollment event for student {}: {}",
                        event.studentId(), ex.getMessage());
                    // In production: retry, dead-letter queue, or compensating transaction
                }
            });
    }

    // Sending with explicit headers (useful for routing, versioning, tracing)
    public void publishWithHeaders(StudentEnrolledEvent event) {
        ProducerRecord<String, Object> record = new ProducerRecord<>(
            "student-enrollments",
            null,   // partition — null means use key-based partitioning
            "student-" + event.studentId(),
            event
        );
        record.headers()
            .add("event-type",    "STUDENT_ENROLLED".getBytes())
            .add("event-version", "v1".getBytes())
            .add("source-service","enrollment-service".getBytes());

        kafkaTemplate.send(record);
    }
}
```

### 1.2 Consuming Messages with @KafkaListener

```java
@Service
@Slf4j
public class NotificationConsumer {

    private final EmailService emailService;

    public NotificationConsumer(EmailService emailService) {
        this.emailService = emailService;
    }

    // @KafkaListener: Spring manages the consumer lifecycle — thread creation, polling, etc.
    // groupId: this consumer is part of the "notification-group" consumer group
    // If you run two instances of this service, Kafka will give each instance half the partitions
    @KafkaListener(
        topics   = "student-enrollments",
        groupId  = "notification-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleEnrollmentEvent(StudentEnrolledEvent event,
                                       @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
                                       @Header(KafkaHeaders.OFFSET) long offset,
                                       @Header("event-type") String eventType) {
        log.info("Consumed event from partition {} at offset {}: student {} enrolled in {}",
            partition, offset, event.studentId(), event.courseName());

        // Process the event — send a welcome email
        emailService.sendEnrollmentConfirmation(event.studentEmail(), event.courseName());
        // If this method returns normally, Spring auto-commits the offset
        // If this method throws, Spring will retry based on your retry config
    }

    // Consuming multiple topics with the same listener
    @KafkaListener(topics = {"student-enrollments", "student-withdrawals"}, groupId = "audit-group")
    public void auditAllEnrollmentChanges(ConsumerRecord<String, Object> record) {
        // ConsumerRecord gives you full access to key, value, headers, partition, offset
        log.info("[AUDIT] Topic: {}, Key: {}, Partition: {}, Offset: {}",
            record.topic(), record.key(), record.partition(), record.offset());
        auditService.record(record.topic(), record.value().toString());
    }

    // Batch consuming — receive multiple records at once for higher throughput
    @KafkaListener(topics = "score-updates", groupId = "batch-group",
                   containerFactory = "batchKafkaListenerContainerFactory")
    public void processScoreUpdatesBatch(List<ConsumerRecord<String, ScoreUpdateEvent>> records) {
        log.info("Processing batch of {} score updates", records.size());
        List<ScoreUpdateEvent> events = records.stream()
            .map(ConsumerRecord::value)
            .collect(Collectors.toList());
        scoreService.updateBatch(events); // Process as a batch — much more efficient for DB writes
    }
}
```

### 1.3 Dead Letter Topics — Handling Poison Messages

A **poison message** is one that consistently fails processing. Without special handling, one bad message can block an entire partition forever. The dead letter topic (DLT) pattern moves failed messages to a separate topic for later inspection and reprocessing.

```java
@Configuration
public class KafkaConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object>
           kafkaListenerContainerFactory(ConsumerFactory<String, Object> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);

        // Configure retry with backoff before sending to DLT
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate,
                (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition())
            ),
            new FixedBackOff(1000L, 3) // Retry 3 times with 1-second delay between attempts
        );

        // Don't retry deserialization errors — bad messages go straight to DLT
        errorHandler.addNotRetryableExceptions(DeserializationException.class);
        factory.setCommonErrorHandler(errorHandler);

        return factory;
    }
}
```

---

## 🐇 Part 2 — RabbitMQ (AMQP)

RabbitMQ is a traditional message broker implementing the **AMQP** (Advanced Message Queuing Protocol). Its model is fundamentally different from Kafka's. In RabbitMQ, messages are routed through **exchanges** to **queues** based on rules. Messages are deleted from the queue after successful consumption (not retained like Kafka).

The routing mechanism is what makes RabbitMQ powerful and flexible. A **Direct Exchange** routes messages to the queue whose binding key exactly matches the message's routing key — like a direct address. A **Topic Exchange** routes messages based on wildcard patterns in the routing key (e.g., `student.*` matches `student.enrolled` and `student.withdrawn`). A **Fanout Exchange** broadcasts every message to all bound queues, ignoring the routing key entirely — ideal for event broadcasting. A **Headers Exchange** routes based on message headers rather than routing keys.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    listener:
      simple:
        acknowledge-mode: auto      # auto = Spring acks on success, nacks on exception
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000
```

```java
@Configuration
public class RabbitMQConfig {

    // Constants — use these instead of magic strings throughout your code
    public static final String STUDENT_EXCHANGE   = "student.exchange";
    public static final String ENROLLMENT_QUEUE   = "student.enrollment.queue";
    public static final String NOTIFICATION_QUEUE = "student.notification.queue";
    public static final String ENROLLMENT_KEY     = "student.enrolled";
    public static final String DEAD_LETTER_QUEUE  = "student.dlq";

    // Declare the exchange
    @Bean
    public TopicExchange studentExchange() {
        return new TopicExchange(STUDENT_EXCHANGE, true, false); // durable, not auto-deleted
    }

    // Declare the queues
    @Bean
    public Queue enrollmentQueue() {
        // Create queue with dead-letter configuration
        return QueueBuilder.durable(ENROLLMENT_QUEUE)
            .withArgument("x-dead-letter-exchange", "")       // Route to default exchange
            .withArgument("x-dead-letter-routing-key", DEAD_LETTER_QUEUE) // Then to DLQ
            .withArgument("x-message-ttl", 300_000)           // Messages expire after 5 minutes
            .build();
    }

    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable(DEAD_LETTER_QUEUE).build();
    }

    @Bean
    public Queue notificationQueue() {
        return new Queue(NOTIFICATION_QUEUE, true); // durable
    }

    // Bind the enrollment queue to the exchange with a routing key
    @Bean
    public Binding enrollmentBinding(Queue enrollmentQueue, TopicExchange studentExchange) {
        return BindingBuilder.bind(enrollmentQueue)
            .to(studentExchange)
            .with(ENROLLMENT_KEY); // Only "student.enrolled" messages route here
    }

    // Topic wildcard: "student.*" matches student.enrolled, student.withdrawn, etc.
    @Bean
    public Binding notificationBinding(Queue notificationQueue, TopicExchange studentExchange) {
        return BindingBuilder.bind(notificationQueue)
            .to(studentExchange)
            .with("student.*"); // Wildcard: all student events go to notification queue
    }

    // Configure message conversion — use JSON instead of default Java serialization
    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) log.error("Message not confirmed by broker: {}", cause);
        });
        return template;
    }
}
```

```java
// Producer
@Service
@Slf4j
public class EnrollmentEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public EnrollmentEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void publishEnrollment(StudentEnrolledEvent event) {
        // convertAndSend: serializes to JSON and sends to the exchange with the routing key
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.STUDENT_EXCHANGE,  // Exchange name
            RabbitMQConfig.ENROLLMENT_KEY,    // Routing key
            event,                            // Message body (Jackson serializes to JSON)
            message -> {
                // Customize message properties
                message.getMessageProperties().setExpiration("300000"); // TTL: 5 min
                message.getMessageProperties().setHeader("event-version", "1.0");
                return message;
            }
        );
        log.info("Published enrollment event for student {}", event.studentId());
    }
}

// Consumer
@Service
@Slf4j
public class EnrollmentProcessor {

    @RabbitListener(queues = RabbitMQConfig.ENROLLMENT_QUEUE)
    public void processEnrollment(StudentEnrolledEvent event,
                                   @Header(AmqpHeaders.RECEIVED_ROUTING_KEY) String routingKey) {
        log.info("Processing enrollment via routing key {}: student {} → course {}",
            routingKey, event.studentId(), event.courseId());

        try {
            enrollmentService.processEnrollment(event);
            // If method returns normally, Spring auto-acknowledges the message
        } catch (BusinessException e) {
            // Throwing an AmqpRejectAndDontRequeueException sends to DLQ without requeueing
            throw new AmqpRejectAndDontRequeueException("Invalid enrollment data: " + e.getMessage(), e);
        }
        // Other exceptions: Spring requeues the message and retries per the retry config
    }
}
```

---

## 🌐 Part 3 — WebSocket with STOMP

WebSocket provides a full-duplex, persistent connection between a browser and server. Unlike HTTP, where the client always initiates the request, WebSocket allows the server to push data to clients at any time. **STOMP** (Simple Text Oriented Messaging Protocol) is a lightweight messaging protocol layered on top of WebSocket, and Spring provides first-class support for it.

Classic use cases for WebSocket + STOMP are real-time notifications, live dashboards, collaborative editing, online multiplayer games, and live chat.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

```java
@Configuration
@EnableWebSocketMessageBroker // Enables Spring's STOMP message broker support
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // /topic → broadcast to all subscribers (many consumers)
        // /queue → send to a specific user (one consumer)
        // In production, replace with a real broker: registry.enableStompBrokerRelay(...)
        registry.enableSimpleBroker("/topic", "/queue");

        // Messages from the client to the server must start with /app
        // They are routed to @MessageMapping methods in your controllers
        registry.setApplicationDestinationPrefixes("/app");

        // Prefix for user-specific destinations
        registry.setUserDestinationPrefix("/user");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // The WebSocket endpoint — clients connect here to establish the STOMP session
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("*") // In production: restrict to your frontend domain
            .withSockJS(); // SockJS fallback: if WebSocket is blocked, falls back to HTTP long-polling
    }
}
```

```java
@Controller
@Slf4j
public class NotificationController {

    // SimpMessagingTemplate: the Spring abstraction for sending STOMP messages
    private final SimpMessagingTemplate messagingTemplate;

    public NotificationController(SimpMessagingTemplate messagingTemplate) {
        this.messagingTemplate = messagingTemplate;
    }

    // @MessageMapping: handles messages from clients sent to /app/chat
    // @SendTo: broadcasts the return value to all subscribers of /topic/announcements
    @MessageMapping("/chat")
    @SendTo("/topic/announcements")
    public ChatMessage broadcastMessage(ChatMessage incomingMessage,
                                         SimpMessageHeaderAccessor headerAccessor) {
        String sender = headerAccessor.getUser() != null ?
            headerAccessor.getUser().getName() : "Anonymous";
        log.info("Broadcast message from {}: {}", sender, incomingMessage.content());
        return new ChatMessage(sender, incomingMessage.content(), LocalDateTime.now());
    }

    // @MessageMapping without @SendTo — handle the message without an automatic reply
    @MessageMapping("/acknowledge")
    public void handleAcknowledgement(AckMessage ack, Principal principal) {
        log.info("User {} acknowledged message {}", principal.getName(), ack.messageId());
        // Could send a targeted reply using messagingTemplate.convertAndSendToUser()
    }

    // Server-initiated push: send a message to all subscribers of a topic
    // Call this from anywhere in your application (triggered by Kafka, scheduled task, etc.)
    public void broadcastScoreUpdate(ScoreUpdateEvent event) {
        messagingTemplate.convertAndSend("/topic/score-updates", event);
    }

    // Send a message to a SPECIFIC connected user (user-specific queue)
    // The user receives it at /user/queue/notifications in their STOMP session
    public void sendPersonalNotification(String username, Notification notification) {
        messagingTemplate.convertAndSendToUser(username, "/queue/notifications", notification);
    }
}
```

```javascript
// JavaScript client code (using STOMP.js)
const stompClient = new Client({
    webSocketFactory: () => new SockJS('/ws'),

    onConnect: () => {
        // Subscribe to the broadcast topic
        stompClient.subscribe('/topic/score-updates', (message) => {
            const update = JSON.parse(message.body);
            console.log('Score update:', update);
            updateScoreDisplay(update);
        });

        // Subscribe to personal notifications
        stompClient.subscribe('/user/queue/notifications', (message) => {
            const notification = JSON.parse(message.body);
            showNotification(notification);
        });

        // Send a message to the server
        stompClient.publish({
            destination: '/app/chat',
            body: JSON.stringify({ content: 'Hello from the browser!' })
        });
    }
});

stompClient.activate();
```

---

## 🔗 Part 4 — Spring Integration (Overview)

Spring Integration implements the **Enterprise Integration Patterns** (from the seminal book by Gregor Hohpe and Bobby Woolf). It provides a framework for building message-driven pipelines that connect heterogeneous systems — reading from a file, transforming the content, filtering some records, enriching with database data, and sending the result to an external API or queue.

```java
@Configuration
@EnableIntegration
public class FileProcessingIntegrationConfig {

    // A Message Channel is the pipe that connects integration components
    @Bean
    public MessageChannel studentFileInputChannel() {
        return new DirectChannel(); // Synchronous, point-to-point delivery
    }

    @Bean
    public MessageChannel processedStudentChannel() {
        return new QueueChannel(100); // Buffered channel — handles 100 messages before blocking
    }

    // Inbound adapter: reads files from a directory and sends them as messages to the channel
    @Bean
    public IntegrationFlow fileReadingFlow() {
        return IntegrationFlow
            .from(Files.inboundAdapter(new File("/import/students"))
                    .patternFilter("*.csv"),          // Only process CSV files
                  poller -> poller.fixedDelay(5000))  // Poll every 5 seconds
            .transform(Files.toStringTransformer())   // File → String content
            .split(new FileSplitter())                // Split CSV into individual lines
            .filter((String line) -> !line.startsWith("#")) // Ignore comment lines
            .transform(this::parseCsvLine)            // CSV line → Student object
            .channel(studentFileInputChannel())       // Send to next channel
            .get();
    }

    // Processing flow: enrich, filter, and route
    @Bean
    public IntegrationFlow studentProcessingFlow() {
        return IntegrationFlow
            .from(studentFileInputChannel())
            .filter((Student s) -> s.getEmail() != null && s.getEmail().contains("@")) // Validate
            .<Student, Student>transform(s -> {       // Enrich with department
                s.setDepartment(departmentService.findByCode(s.getDepartmentCode()));
                return s;
            })
            .<Student, String>route(                  // Route based on status
                Student::getStatus,
                mapping -> mapping
                    .subFlowMapping("ACTIVE",   sf -> sf.channel("activeStudentChannel"))
                    .subFlowMapping("INACTIVE", sf -> sf.channel("inactiveStudentChannel"))
                    .defaultOutputChannel("unknownStatusChannel")
            )
            .get();
    }

    private Student parseCsvLine(String line) {
        String[] parts = line.split(",");
        return new Student(parts[0].trim(), parts[1].trim(), parts[2].trim());
    }
}
```

---

## ✅ Best Practices & Important Points

**Choose Kafka for event streaming and audit logs; choose RabbitMQ for task queuing and complex routing.** Kafka's message retention makes it ideal when you need message replay, event sourcing, or feeding multiple independent consumer groups from the same event stream. RabbitMQ's flexible exchange/routing model makes it excellent for request-reply patterns, priority queues, and scenarios where messages should be deleted after consumption.

**Use message keys in Kafka to ensure ordering.** Kafka only guarantees message ordering within a single partition. If you need all events for the same entity (e.g., all events for student ID 42) to be processed in order, use the entity ID as the message key — Kafka will route all messages with the same key to the same partition.

**Always configure Dead Letter Queues (DLQ) or Dead Letter Topics (DLT) in production.** Without them, a single malformed message ("poison pill") can block a consumer forever or get lost during retries. A DLQ gives you a safe place to park failed messages so you can inspect, fix, and reprocess them without data loss.

**Use `convertAndSend` over manual serialization in RabbitMQ.** Configure `Jackson2JsonMessageConverter` and let Spring handle serialization. Manual serialization with Java's default `ObjectOutputStream` is fragile (tied to class structure), insecure (can deserialize arbitrary objects), and not human-readable.

**Treat message consumers as idempotent.** Because of network partitions, retries, or at-least-once delivery guarantees, your consumer might receive the same message twice. Design processing logic so that processing a message twice has the same effect as processing it once — for example, use `INSERT ... ON CONFLICT DO NOTHING` in SQL or check for duplicates before processing.

**For WebSocket, use SockJS fallback.** Some corporate firewalls and proxies block the WebSocket protocol upgrade. Configuring SockJS allows the connection to fall back gracefully to HTTP long-polling, ensuring your real-time features work in all network environments.

---

## 🔑 Key Summary

```
Kafka Topic           → Named channel; messages are retained and replayable
Kafka Partition       → Subset of a topic; unit of parallelism; ordering guaranteed within a partition
Message Key           → Determines partition routing; use entity ID to preserve per-entity ordering
KafkaTemplate         → Spring abstraction for producing Kafka messages
@KafkaListener        → Declares a consumer method; Spring manages threads and polling
Dead Letter Topic     → Where failed messages go after exhausting retries; prevents poison pills
RabbitMQ Exchange     → Routes messages to queues based on routing keys or headers
Topic Exchange        → Routes using wildcard patterns (student.*)
Fanout Exchange       → Broadcasts to all bound queues — ignores routing keys
@RabbitListener       → Declares a RabbitMQ consumer method; Spring manages AMQP connection
RabbitTemplate        → Spring abstraction for sending RabbitMQ messages
WebSocket             → Full-duplex persistent connection between browser and server
STOMP                 → Messaging protocol over WebSocket; subscribe to topics, send to app prefixes
@MessageMapping       → Handles messages from WebSocket clients
SimpMessagingTemplate → Server-side: push messages to topics or specific users
/topic/*              → Broadcast destinations; all subscribers receive the message
/user/queue/*         → User-specific destinations; only that user's session receives it
Spring Integration    → Enterprise Integration Patterns framework; file/DB/API pipeline processing
```
