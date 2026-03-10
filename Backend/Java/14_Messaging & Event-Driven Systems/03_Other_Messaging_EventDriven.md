# 14.3 — Other Messaging Systems & Event-Driven Architecture Patterns
### Phase 14: Messaging & Event-Driven Systems

---

## Why You Need to Know More Than Just Kafka and RabbitMQ

Kafka and RabbitMQ are the dominant choices for Java backend systems, but they are not the only ones you will encounter. In many organizations you will find **Apache ActiveMQ** powering legacy enterprise systems — it predates both Kafka and modern RabbitMQ and there are enormous amounts of production code built on it. In cloud-native environments, particularly on AWS, you will encounter **Amazon SQS and SNS** as the default managed messaging services. And in systems that already use Redis for caching, **Redis Pub/Sub** often gets used for lightweight event broadcasting because it requires no new infrastructure.

Beyond the specific systems, this section covers the **architectural patterns** that tie messaging systems together into coherent event-driven applications. Understanding these patterns — the Event-Driven Architecture style, the Saga pattern for distributed transactions, the Outbox pattern for atomicity, CQRS, and Event Sourcing — is what elevates you from someone who knows how to use a messaging API to someone who can design distributed systems thoughtfully.

---

## Apache ActiveMQ

### What It Is and Where It Comes From

Apache ActiveMQ is one of the oldest and most widely deployed open-source message brokers, initially released in 2004. It implements the **JMS (Java Message Service)** specification — a Java standard API for messaging, analogous to JDBC for databases. JMS defines a common interface, so code written against the JMS API can theoretically switch between JMS-compliant brokers (ActiveMQ, IBM MQ, JBoss Messaging) without changing application code.

There are two current versions worth knowing. **ActiveMQ Classic** (formerly just "ActiveMQ 5.x") is the original, still widely deployed in enterprises. **ActiveMQ Artemis** is the modern, rewritten version with significantly better performance, and is what new deployments should use. Spring Boot's embedded broker support uses Artemis.

You are most likely to encounter ActiveMQ when working on existing enterprise applications, Spring Integration pipelines, or systems that need JMS compatibility with IBM or Oracle enterprise software.

### JMS Concepts

JMS defines two messaging models that map to familiar concepts. A **Queue** in JMS is a point-to-point channel — each message is consumed by exactly one receiver. A **Topic** in JMS is a publish-subscribe channel — each message is delivered to all active subscribers. Notice that JMS Topics are NOT the same as Kafka Topics — JMS Topics are broadcast mechanisms, while Kafka Topics are partitioned logs.

A critical JMS limitation is that **durable subscriptions must be active to receive messages**. If a JMS Topic subscriber is offline when a message is published, it misses the message unless it has a durable subscription (which requires configuration). Kafka has no such constraint — consumers can be offline for days and catch up when they reconnect.

### Spring Boot Integration with Artemis

```xml
<!-- pom.xml — Spring Boot auto-configures the embedded Artemis broker in test/dev -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-artemis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>artemis-jakarta-client</artifactId>
</dependency>
```

```yaml
# application.yml — connect to an external Artemis broker
spring:
  artemis:
    mode: native           # 'native' connects to external broker; 'embedded' runs in-process
    host: localhost
    port: 61616            # Default Artemis port (CORE protocol)
    user: admin
    password: admin
  jms:
    template:
      default-destination: orders
    listener:
      acknowledge-mode: client  # Manual acknowledgment (equivalent to MANUAL in Spring AMQP)
```

```java
@Configuration
@EnableJms  // Activates @JmsListener processing
public class JmsConfig {

    /**
     * JmsTemplate is the Spring JMS equivalent of RabbitTemplate or KafkaTemplate.
     * Override the default converter to use JSON instead of Java serialization.
     */
    @Bean
    public MessageConverter jacksonJmsMessageConverter() {
        MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
        converter.setTargetType(MessageType.TEXT);          // Serialize to TextMessage (JSON string)
        converter.setTypeIdPropertyName("_type");           // Header that tells receiver the Java type
        return converter;
    }

    // Configure the container factory for @JmsListener
    @Bean
    public DefaultJmsListenerContainerFactory jmsListenerContainerFactory(
            ConnectionFactory connectionFactory,
            MessageConverter messageConverter) {

        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(messageConverter);
        factory.setSessionAcknowledgeMode(Session.CLIENT_ACKNOWLEDGE); // Manual ack
        factory.setConcurrency("3-10");     // Min 3, max 10 concurrent consumers
        factory.setErrorHandler(t -> log.error("JMS error", t));
        return factory;
    }

    // Declare the queue (creates it if it doesn't exist)
    @Bean
    public Queue orderQueue() {
        return new ActiveMQQueue("order.processing.queue");
    }

    @Bean
    public Topic orderEventsTopic() {
        return new ActiveMQTopic("order.events.topic");
    }
}


// ── PRODUCER ─────────────────────────────────────────────────────────────────
@Service
public class JmsOrderProducer {

    private final JmsTemplate jmsTemplate;

    public JmsOrderProducer(JmsTemplate jmsTemplate) {
        this.jmsTemplate = jmsTemplate;
    }

    // Send to a queue (point-to-point — processed by exactly one consumer)
    public void sendToQueue(OrderEvent event) {
        jmsTemplate.convertAndSend("order.processing.queue", event);
    }

    // Publish to a topic (broadcast — all active subscribers receive it)
    public void publishToTopic(OrderEvent event) {
        jmsTemplate.convertAndSend("order.events.topic", event);
    }

    // Send with full control over message properties
    public void sendWithProperties(OrderEvent event) {
        jmsTemplate.send("order.processing.queue", session -> {
            TextMessage message = session.createTextMessage(
                objectMapper.writeValueAsString(event)
            );
            message.setStringProperty("orderId", event.orderId());
            message.setIntProperty("priority", 5);
            message.setJMSExpiration(System.currentTimeMillis() + 3_600_000); // Expire in 1 hour
            message.setJMSCorrelationID(UUID.randomUUID().toString());
            return message;
        });
    }
}


// ── CONSUMER ─────────────────────────────────────────────────────────────────
@Component
public class JmsOrderConsumer {

    @JmsListener(destination = "order.processing.queue", containerFactory = "jmsListenerContainerFactory")
    public void processOrderEvent(OrderEvent event, Session session, Message rawMessage) throws JMSException {
        log.info("Received JMS order event: orderId={}", event.orderId());

        try {
            orderService.process(event);
            rawMessage.acknowledge(); // Commit offset — message removed from queue

        } catch (Exception e) {
            log.error("Failed to process order: {}", event.orderId(), e);
            session.recover(); // Roll back — message will be redelivered
        }
    }

    // Subscribe to a JMS Topic — receives all published messages
    // Note: this subscription only receives messages published WHILE this consumer is active
    @JmsListener(destination = "order.events.topic")
    public void subscribeToOrderEvents(OrderEvent event) {
        log.info("Topic subscription received: orderId={}", event.orderId());
        analyticsService.record(event);
    }
}
```

---

## AWS SQS and SNS

### The Cloud Messaging Philosophy

Amazon's messaging services differ from ActiveMQ and RabbitMQ in one important way: you never manage the infrastructure. There are no brokers to configure, no servers to patch, no clusters to size. You pay per message and AWS handles everything else. For teams running on AWS, this is often the path of least resistance.

**Amazon SQS (Simple Queue Service)** is a managed message queue. It holds messages until a consumer processes them. It comes in two flavors: **Standard Queues** offer maximum throughput with best-effort ordering and at-least-once delivery, and **FIFO Queues** guarantee strict ordering and exactly-once processing within a message group (at a lower throughput limit of 3,000 messages/sec with batching).

**Amazon SNS (Simple Notification Service)** is a managed pub/sub service. It has **topics** (broadcast channels) to which you publish once, and SNS fans the message out to all subscribed endpoints — which can be SQS queues, Lambda functions, HTTP endpoints, email addresses, or mobile push notifications. The SNS → SQS fan-out pattern is the standard AWS architecture for distributing a single event to multiple consuming services.

```
                           ┌─────────────────────────────────────┐
                           │           SNS Topic                  │
                           │     "order-events-topic"             │
                           └─────────────┬───────────────────────┘
                                         │  Fan-out
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                           │
              ▼                          ▼                           ▼
    ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
    │  SQS Queue       │     │  SQS Queue       │     │  Lambda Function │
    │ "order-process"  │     │ "order-analytics"│     │ send-email       │
    │ (fulfillment     │     │ (analytics       │     │ (notification    │
    │  service polls)  │     │  service polls)  │     │  service)        │
    └──────────────────┘     └──────────────────┘     └──────────────────┘
```

### Setting Up with Spring Cloud AWS and Spring Boot

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sqs</artifactId>
    <version>3.1.0</version>
</dependency>
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sns</artifactId>
    <version>3.1.0</version>
</dependency>
```

```yaml
# application.yml
spring:
  cloud:
    aws:
      region:
        static: us-east-1
      credentials:
        # In production on EC2/ECS/Lambda, use IAM roles (no explicit credentials needed)
        # For local dev, set these environment variables:
        # AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY
      sqs:
        listener:
          acknowledgement-mode: ON_SUCCESS   # Auto-ack if method completes without exception
          max-concurrent-messages: 10        # Up to 10 messages processed concurrently
          max-messages-per-poll: 10          # SQS max per receive call
          poll-timeout: 20s                  # Long-polling timeout (reduces empty receives)
```

```java
// ── SNS PRODUCER — Publish to a topic, fanout to all subscribers ──────────────
@Service
public class OrderEventSnsProducer {

    private final SnsTemplate snsTemplate;

    // SNS topic ARN (Amazon Resource Name) — from environment variable in production
    @Value("${aws.sns.order-events-topic-arn}")
    private String orderEventsTopicArn;

    public OrderEventSnsProducer(SnsTemplate snsTemplate) {
        this.snsTemplate = snsTemplate;
    }

    public void publishOrderEvent(OrderEvent event) {
        // snsTemplate.sendNotification wraps the payload in an SNS envelope
        snsTemplate.sendNotification(
            orderEventsTopicArn,
            event,          // Serialized to JSON by Jackson
            "Order Event"   // Optional subject (used for email subscriptions)
        );
    }

    // SNS supports message attributes for server-side filtering —
    // SQS subscriptions can filter messages based on these attributes,
    // avoiding unnecessary message delivery to uninterested consumers
    public void publishWithAttributes(OrderEvent event) {
        snsTemplate.send(orderEventsTopicArn,
            SnsNotification.builder()
                .payload(event)
                .messageAttribute("eventType", MessageAttribute.builder()
                    .dataType(MessageAttributeDataTypes.STRING)
                    .stringValue(event.status())
                    .build())
                .messageAttribute("region", MessageAttribute.builder()
                    .dataType(MessageAttributeDataTypes.STRING)
                    .stringValue(event.region())
                    .build())
                .build()
        );
    }
}


// ── SQS PRODUCER — Send directly to a queue ──────────────────────────────────
@Service
public class OrderEventSqsProducer {

    private final SqsTemplate sqsTemplate;

    @Value("${aws.sqs.order-processing-queue-url}")
    private String orderProcessingQueueUrl;

    public OrderEventSqsProducer(SqsTemplate sqsTemplate) {
        this.sqsTemplate = sqsTemplate;
    }

    // Send to a Standard Queue (best-effort ordering)
    public void sendToQueue(OrderEvent event) {
        sqsTemplate.send(orderProcessingQueueUrl, event);
    }

    // Send to a FIFO Queue (strict ordering within a MessageGroupId)
    // FIFO queue URLs end with ".fifo"
    public void sendToFifoQueue(OrderEvent event) {
        sqsTemplate.send(send -> send
            .queue(orderProcessingQueueUrl + ".fifo")
            .payload(event)
            // MessageGroupId: messages with the same group ID are processed in order
            // All events for the same order should use orderId as the group
            .messageGroupId(event.orderId())
            // MessageDeduplicationId: prevents duplicate processing within 5 minutes
            // Using a hash of the event content ensures true deduplication
            .messageDeduplicationId(DigestUtils.sha256Hex(objectMapper.writeValueAsString(event)))
        );
    }

    // Batch send — up to 10 messages per API call (much more efficient)
    public void sendBatch(List<OrderEvent> events) {
        sqsTemplate.sendMany(orderProcessingQueueUrl, events);
    }
}


// ── SQS CONSUMER ─────────────────────────────────────────────────────────────
@Component
public class OrderEventSqsConsumer {

    // @SqsListener is Spring Cloud AWS's equivalent of @KafkaListener or @RabbitListener
    @SqsListener(value = "${aws.sqs.order-processing-queue-url}")
    public void processOrderEvent(
            OrderEvent event,
            @Header("MessageId") String messageId,      // SQS assigns each message a unique ID
            @Header(value = "ApproximateReceiveCount", required = false) Integer receiveCount) {

        log.info("Processing SQS message: messageId={}, receiveCount={}", messageId, receiveCount);

        // SQS delivers messages at least once — be idempotent!
        // Check if we've already processed this message before doing work.
        if (processedMessageRepository.exists(messageId)) {
            log.info("Message {} already processed — skipping", messageId);
            return; // Return without throwing — SQS will delete the message
        }

        orderService.processOrderEvent(event);
        processedMessageRepository.markProcessed(messageId);

        // When the method returns normally, Spring Cloud AWS automatically
        // deletes the message from SQS (acknowledges it).
        // If the method throws an exception, the message becomes visible again
        // after the visibility timeout and gets redelivered.
    }

    // Batch listener — process up to 10 messages at once
    @SqsListener(value = "${aws.sqs.order-analytics-queue-url}")
    public void processOrderEventBatch(List<Message<OrderEvent>> messages) {
        log.info("Processing batch of {} SQS messages", messages.size());

        List<OrderEvent> events = messages.stream()
            .map(Message::getPayload)
            .collect(Collectors.toList());

        analyticsService.recordBatch(events);
        // All messages in the batch are deleted upon successful return
    }
}
```

### SQS Dead-Letter Queues

SQS has built-in DLQ support configured at the queue level. After a message is received (but not deleted) `maxReceiveCount` times, SQS automatically moves it to the configured DLQ.

```java
@Configuration
public class SqsQueueConfig {

    @Autowired
    private SqsAsyncClient sqsClient;

    // Create queues programmatically on startup (or use Terraform/CloudFormation in production)
    @PostConstruct
    public void createQueues() {
        // Create the dead-letter queue first
        String dlqUrl = sqsClient.createQueue(CreateQueueRequest.builder()
            .queueName("order-processing-dlq")
            .build())
            .join().queueUrl();

        // Get the DLQ ARN
        String dlqArn = sqsClient.getQueueAttributes(GetQueueAttributesRequest.builder()
            .queueUrl(dlqUrl)
            .attributeNames(QueueAttributeName.QUEUE_ARN)
            .build())
            .join().attributes().get(QueueAttributeName.QUEUE_ARN);

        // Create the main queue with DLQ configured
        sqsClient.createQueue(CreateQueueRequest.builder()
            .queueName("order-processing-queue")
            .attributes(Map.of(
                QueueAttributeName.VISIBILITY_TIMEOUT, "30",   // Seconds to hide after receive
                QueueAttributeName.MESSAGE_RETENTION_PERIOD, "86400", // Keep messages 1 day
                QueueAttributeName.REDRIVE_POLICY, objectMapper.writeValueAsString(Map.of(
                    "deadLetterTargetArn", dlqArn,
                    "maxReceiveCount", 3   // Move to DLQ after 3 failed attempts
                ))
            ))
            .build())
            .join();
    }
}
```

---

## Redis Pub/Sub

### What Redis Pub/Sub Is — and What It Isn't

Redis is primarily an in-memory data store and cache, but it includes a **Publish/Subscribe** messaging feature that many teams use for lightweight event broadcasting. It's attractive because it requires no additional infrastructure if you're already using Redis for caching.

However, Redis Pub/Sub has a critical limitation you must understand before choosing it: **messages are not persisted and not buffered**. A message published to a Redis channel is delivered only to subscribers that are connected and listening at that exact moment. If a subscriber is offline, it simply misses the message — there is no queue, no replay, no retry. Redis Pub/Sub is pure fire-and-forget broadcast.

This makes Redis Pub/Sub appropriate for scenarios where losing messages is acceptable: invalidating caches across multiple application instances, broadcasting real-time dashboard updates to connected WebSocket clients, or refreshing configuration in a cluster. It is completely inappropriate for scenarios where message delivery must be guaranteed: order processing, payment events, or any business-critical workflow.

For persistent pub/sub in Redis, use **Redis Streams** (a different data structure) — it's essentially Redis's answer to Kafka, with consumer groups, persistent storage, and message replay.

### Spring Data Redis Pub/Sub

```java
@Configuration
public class RedisPubSubConfig {

    /**
     * RedisMessageListenerContainer is the engine that maintains the subscription
     * to Redis channels and dispatches incoming messages to listener beans.
     * It runs a dedicated background thread that blocks on Redis's SUBSCRIBE command.
     */
    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory,
            CacheInvalidationListener cacheInvalidationListener,
            LiveScoreListener liveScoreListener) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);

        // Register listeners with their channel patterns
        // ChannelTopic: exact channel name match
        container.addMessageListener(cacheInvalidationListener,
            new ChannelTopic("cache.invalidation"));

        // PatternTopic: wildcard pattern — subscribe to all "scores.*" channels
        container.addMessageListener(liveScoreListener,
            new PatternTopic("scores.*"));

        return container;
    }

    // Jackson-based message converter for JSON serialization
    @Bean
    public RedisSerializer<Object> redisSerializer() {
        return RedisSerializer.json();
    }
}


// ── PUBLISHER ────────────────────────────────────────────────────────────────
@Service
public class CacheInvalidationPublisher {

    private final RedisTemplate<String, Object> redisTemplate;

    public CacheInvalidationPublisher(RedisTemplate<String, Object> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    /**
     * Publish a cache invalidation event to all instances of the application.
     * Each instance that has this product cached will evict it from its local cache.
     * This is the primary use case for Redis Pub/Sub — lightweight cluster coordination.
     *
     * IMPORTANT: This is fire-and-forget. Any instance that is not connected
     * at the moment of publication will NOT receive this message and will keep
     * its stale cache until the TTL expires or another invalidation is published.
     * This is an acceptable tradeoff for cache invalidation (brief inconsistency).
     */
    public void invalidateProductCache(String productId) {
        CacheInvalidationEvent event = new CacheInvalidationEvent(
            "product", productId, Instant.now()
        );
        // convertAndSend channel, message
        redisTemplate.convertAndSend("cache.invalidation", event);
    }

    public void publishLiveScore(String gameId, ScoreUpdate score) {
        // Publish to a channel named after the game: "scores.game-123"
        redisTemplate.convertAndSend("scores." + gameId, score);
    }
}


// ── SUBSCRIBER ───────────────────────────────────────────────────────────────
@Component
public class CacheInvalidationListener implements MessageListener {

    private final CacheManager cacheManager;

    // onMessage is called by the container whenever a message arrives on the subscribed channel
    @Override
    public void onMessage(Message message, byte[] pattern) {
        // Deserialize the message body from JSON bytes
        CacheInvalidationEvent event = deserialize(message.getBody(), CacheInvalidationEvent.class);

        log.info("Cache invalidation received: entityType={}, entityId={}",
            event.entityType(), event.entityId());

        // Evict the specific entry from the local cache
        Cache cache = cacheManager.getCache(event.entityType());
        if (cache != null) {
            cache.evict(event.entityId());
        }
    }
}

// Pattern subscriber receives messages from multiple channels matching the pattern
@Component
public class LiveScoreListener implements MessageListener {

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel()); // e.g., "scores.game-123"
        String gameId = channel.substring("scores.".length());

        ScoreUpdate score = deserialize(message.getBody(), ScoreUpdate.class);
        log.info("Score update for game {}: {}", gameId, score);

        // Push to WebSocket clients watching this game
        webSocketTemplate.convertAndSend("/topic/scores/" + gameId, score);
    }
}
```

### Redis Streams — When You Need Persistence in Redis

If you're using Redis and need guaranteed delivery (no message loss), use Redis Streams instead of Pub/Sub. Redis Streams behave much more like Kafka topics — messages are persisted, consumers track their position with IDs, and consumer groups allow parallel processing.

```java
@Service
public class RedisStreamService {

    private final RedisTemplate<String, Object> redisTemplate;

    // Produce: add a message to a stream (like Kafka's producer.send())
    public void publishEvent(OrderEvent event) {
        // XADD order-events * orderId <id> status <status> amount <amount>
        // Returns a RecordId (auto-generated timestamp-based ID)
        RecordId recordId = redisTemplate.opsForStream().add(
            StreamRecords.newRecord()
                .ofObject(event)
                .withStreamKey("order-events")
        );
        log.info("Published to Redis Stream: recordId={}", recordId);
    }

    // Read from a stream using a consumer group (like Kafka's consumer group)
    @Scheduled(fixedDelay = 100)
    public void consumeFromStream() {
        // XREADGROUP GROUP order-processor consumer-1 COUNT 10 STREAMS order-events >
        // The ">" means: give me new messages not yet delivered to this consumer group
        List<MapRecord<String, Object, Object>> messages = redisTemplate.opsForStream().read(
            Consumer.from("order-processor", "consumer-1"),
            StreamReadOptions.empty().count(10),
            StreamOffset.create("order-events", ReadOffset.lastConsumed())
        );

        for (MapRecord<String, Object, Object> message : messages) {
            try {
                OrderEvent event = mapToOrderEvent(message.getValue());
                orderService.process(event);
                // XACK: acknowledge after successful processing
                redisTemplate.opsForStream().acknowledge("order-events", "order-processor", message.getId());
            } catch (Exception e) {
                log.error("Failed to process stream message {}", message.getId(), e);
            }
        }
    }
}
```

---

## Event-Driven Architecture Patterns

Understanding the specific messaging APIs is only half the picture. The other half is knowing how to structure your application using event-driven patterns. These patterns emerge directly from the constraints and capabilities of messaging systems.

### The Core Philosophy: Events as First-Class Citizens

In a synchronous, request-response system, services call each other directly. Service A says "please create a payment" and waits for B to say "done." In an event-driven system, services instead announce what happened. Service A says "payment was created" and moves on. Any service that cares about that fact can subscribe to those events and react asynchronously.

This shift — from commands to events, from synchronous to asynchronous — has profound implications. Services become loosely coupled because a publisher doesn't know or care who is listening. Services become more resilient because a subscriber being down doesn't block the publisher. And the system becomes more scalable because each service can process at its own pace.

### The Event Sourcing Pattern

Traditional applications store the *current state* of entities. Your orders table has one row per order showing the latest status. When an order is updated, the row is overwritten. History is lost.

**Event Sourcing** stores the *history of events* that led to the current state instead of the current state itself. Every change to an entity is recorded as an immutable event. The current state is derived by replaying all events.

```java
// Traditional approach: store current state only
// orders table: id=123, status="SHIPPED", updatedAt="..."
// → You know it's shipped but not what happened before

// Event Sourcing approach: store the event history
// order_events table:
// (123, "OrderCreated",  { amount: 99.99 }, 2024-01-01T10:00)
// (123, "PaymentTaken",  { txId: "abc" },   2024-01-01T10:01)
// (123, "OrderShipped",  { trackingId: "X"},2024-01-01T10:05)
// → You know everything that happened, in order

@Service
public class EventSourcedOrderService {

    private final EventStore eventStore;  // Append-only storage for events
    private final KafkaTemplate<String, DomainEvent> kafkaTemplate;

    /**
     * To create an order: generate events, save to event store, publish to Kafka.
     * The event store is the source of truth. Kafka is for notifying other services.
     */
    @Transactional
    public void createOrder(CreateOrderCommand command) {
        String orderId = UUID.randomUUID().toString();

        // Generate the domain event
        OrderCreatedEvent event = new OrderCreatedEvent(
            orderId, command.customerId(), command.items(), command.amount(),
            Instant.now()
        );

        // Persist to event store (append-only)
        eventStore.append(orderId, event);

        // Publish to Kafka so other services know about it
        kafkaTemplate.send("order-events", orderId, event);
    }

    /**
     * To reconstruct an order's current state: replay all its events.
     * This is called "event replay" or "state reconstitution."
     */
    public OrderState getOrderState(String orderId) {
        List<DomainEvent> events = eventStore.getEvents(orderId);
        OrderState state = OrderState.empty();

        // Apply each event to the state in sequence — this is the "aggregate"
        for (DomainEvent event : events) {
            state = state.apply(event);
        }
        return state;
    }
}

// The OrderState knows how to apply each type of event to itself
public class OrderState {

    private String id;
    private String status;
    private BigDecimal amount;
    private List<String> items;

    public static OrderState empty() {
        return new OrderState();
    }

    // Each apply() method handles one event type — pure function, no side effects
    public OrderState apply(DomainEvent event) {
        return switch (event) {
            case OrderCreatedEvent e -> this.withId(e.orderId())
                                           .withStatus("CREATED")
                                           .withAmount(e.amount())
                                           .withItems(e.items());
            case PaymentTakenEvent e -> this.withStatus("PAID");
            case OrderShippedEvent e -> this.withStatus("SHIPPED")
                                           .withTrackingId(e.trackingId());
            case OrderDeliveredEvent e -> this.withStatus("DELIVERED");
            default -> throw new UnknownEventException(event.getClass().getSimpleName());
        };
    }
}
```

Event Sourcing is powerful but complex. Use it when you need a full audit trail, when you need to rebuild state from history, or when your domain is naturally event-oriented (finance, healthcare). Don't use it as a default — it adds significant complexity.

### CQRS — Command Query Responsibility Segregation

**CQRS** separates your application into two distinct models: a **Command side** that handles writes (creates, updates, deletes) and a **Query side** that handles reads. They use different data models, different services, and often different data stores.

Why? Because the requirements for writes and reads are usually different. Writes need transactional integrity and business rule validation. Reads need to be fast, denormalized, and shaped exactly for the UI. Trying to do both with the same model often leads to compromises that serve neither well.

CQRS pairs naturally with event sourcing and messaging: the command side produces events, which are consumed and used to update the read-side projections.

```java
// ── COMMAND SIDE: handles writes with strong consistency ─────────────────────

@Service
public class OrderCommandService {

    // The command model is normalized and transactional
    private final OrderRepository orderRepository; // JPA/RDBMS
    private final KafkaTemplate<String, DomainEvent> kafkaTemplate;

    @Transactional
    public void createOrder(CreateOrderCommand cmd) {
        // Validate business rules
        if (!inventoryService.isAvailable(cmd.items())) {
            throw new InsufficientInventoryException();
        }

        Order order = new Order(cmd.customerId(), cmd.items(), cmd.amount());
        orderRepository.save(order);

        // Publish event for the read side to consume and update its projections
        kafkaTemplate.send("order-events", order.getId(),
            new OrderCreatedEvent(order.getId(), cmd.customerId(), cmd.amount()));
    }
}


// ── QUERY SIDE: projection builder (updates the read model) ──────────────────

@Component
public class OrderProjectionUpdater {

    private final OrderReadRepository orderReadRepository; // Read model (could be ElasticSearch, Redis, etc.)

    @KafkaListener(topics = "order-events", groupId = "order-projection-updater")
    public void updateProjection(DomainEvent event, Acknowledgment ack) {
        switch (event) {
            case OrderCreatedEvent e -> orderReadRepository.save(
                new OrderView(e.orderId(), e.customerId(), e.amount(), "CREATED")
            );
            case OrderShippedEvent e -> orderReadRepository.updateStatus(e.orderId(), "SHIPPED");
            case OrderDeliveredEvent e -> orderReadRepository.updateStatus(e.orderId(), "DELIVERED");
        }
        ack.acknowledge();
    }
}


// ── QUERY SIDE: query service uses the pre-built read model ──────────────────

@Service
public class OrderQueryService {

    // The read model is denormalized and optimized for the queries we need
    private final OrderReadRepository orderReadRepository;

    // Fast, no joins — the read model was already shaped for this query
    public List<OrderView> getOrdersByCustomer(String customerId) {
        return orderReadRepository.findByCustomerId(customerId);
    }

    public OrderDashboardView getDashboard(String customerId) {
        return orderReadRepository.findDashboardByCustomerId(customerId);
    }
}
```

### The Saga Pattern — Distributed Transactions Without Two-Phase Commit

One of the hardest problems in microservices is maintaining consistency across multiple services without using a distributed transaction (which is slow, fragile, and often not supported). The **Saga pattern** solves this by replacing a distributed transaction with a sequence of local transactions, each publishing events that trigger the next step.

If any step fails, compensating transactions are executed in reverse to undo the previous steps.

There are two styles of saga: **orchestration** (one central coordinator directs each step) and **choreography** (each service reacts to events and knows what to do next independently).

```java
// CHOREOGRAPHY-BASED SAGA — no central coordinator
// Each service listens for events and reacts autonomously.
// This is more decoupled but harder to trace when something goes wrong.

// Step 1: Order Service creates order and publishes event
@Service
public class OrderSagaStep {

    @Transactional
    public void createOrder(CreateOrderCommand cmd) {
        Order order = new Order(cmd.customerId(), cmd.items(), cmd.totalAmount());
        order.setStatus("PENDING_PAYMENT");
        orderRepository.save(order);

        // This event triggers Step 2 (Payment Service)
        kafkaTemplate.send("order-events", order.getId(),
            new OrderCreatedEvent(order.getId(), cmd.totalAmount()));
    }

    // Step 4a: Happy path — payment succeeded, mark order as confirmed
    @KafkaListener(topics = "payment-events", groupId = "order-saga")
    public void handlePaymentResult(PaymentResultEvent event, Acknowledgment ack) {
        if (event.success()) {
            // Reserve inventory — triggers Step 3 (Inventory Service)
            orderRepository.updateStatus(event.orderId(), "PAYMENT_CONFIRMED");
            kafkaTemplate.send("inventory-events", event.orderId(),
                new ReserveInventoryCommand(event.orderId(), getOrderItems(event.orderId())));
        } else {
            // Step 4b: Compensate — cancel the order
            orderRepository.updateStatus(event.orderId(), "CANCELLED");
            kafkaTemplate.send("order-events", event.orderId(),
                new OrderCancelledEvent(event.orderId(), "Payment failed"));
        }
        ack.acknowledge();
    }
}

// Step 2: Payment Service listens for OrderCreated and attempts payment
@Service
public class PaymentSagaStep {

    @KafkaListener(topics = "order-events", groupId = "payment-saga")
    public void handleOrderCreated(OrderCreatedEvent event, Acknowledgment ack) {
        try {
            Payment payment = paymentService.charge(event.customerId(), event.amount());
            kafkaTemplate.send("payment-events", event.orderId(),
                new PaymentResultEvent(event.orderId(), true, payment.getTransactionId()));
        } catch (PaymentFailedException e) {
            kafkaTemplate.send("payment-events", event.orderId(),
                new PaymentResultEvent(event.orderId(), false, null));
        }
        ack.acknowledge();
    }
}
```

The orchestration style uses a separate **Saga Orchestrator** class that explicitly manages the state machine, making the workflow visible and traceable — but it introduces a central coordinator that becomes a coupling point.

---

## Choosing the Right Messaging Technology — A Decision Framework

Rather than memorizing rules, internalize the questions you should ask when choosing:

The first question is **do you need message replay?** If you need to re-process historical events — for rebuilding a new service's data, auditing, or recovering from a bug — only Kafka gives you this. RabbitMQ and SQS delete messages after consumption.

The second question is **what is your throughput?** If you need to process millions of events per second reliably, Kafka is the right tool. RabbitMQ handles hundreds of thousands per second, which is more than enough for most applications.

The third question is **do you need sophisticated routing?** If different consumers need to receive different subsets of messages based on flexible patterns, RabbitMQ's exchange model handles this elegantly. Kafka's routing is based purely on topic and partition, which is simpler but less flexible.

The fourth question is **are you already on AWS?** If yes, SQS/SNS gives you managed messaging with no infrastructure to operate. For most cloud-native applications where you don't need Kafka's log semantics, SQS is the pragmatic choice.

The fifth question is **is this truly ephemeral?** If losing a message is acceptable (cache invalidation, real-time UI updates), and you're already running Redis, Redis Pub/Sub is the simplest possible solution.

---

## ⚠️ Important Points & Best Practices

**Design for at-least-once delivery and idempotent consumers everywhere.** Regardless of which messaging system you use, assume any message can be delivered more than once. Network failures, consumer restarts, and rebalances all cause redelivery. Every consumer should check "have I already processed this?" before doing work, using a processed-message table in the database keyed on the message ID.

**Use correlation IDs across all messages.** Every event should carry a correlation ID (sometimes called a trace ID) that links it to the original request that caused it. When an HTTP request triggers a chain of five events processed by three services, having the same correlation ID in all logs makes distributed tracing possible. Without it, debugging cross-service failures is extremely difficult.

**Publish domain events, not data change notifications.** An event like `OrderShipped` (a meaningful business occurrence) is far more useful than `OrderUpdated` (a data change). The former tells consumers *what happened*, the latter tells them *something changed* and they have to figure out what. Name events in the past tense, from the domain's perspective.

**Separate your event contract from your internal domain model.** The events you publish are part of your API — other services depend on them. If you publish your JPA entities directly as events, any change to your database schema is a breaking change to your API. Define explicit event classes that you control separately from your persistence model, and use mappers to translate between them.

**Think carefully about event granularity.** Too fine-grained events (`UserFirstNameUpdated`, `UserLastNameUpdated`, `UserEmailUpdated`) flood the system and make consumers reconstruct state from many tiny pieces. Too coarse-grained events (`UserChanged` with the entire user object) transmit unnecessary data and make consumers process changes they don't care about. Events like `UserProfileUpdated` with the changed fields tend to strike the right balance.

**Plan for schema evolution from day one.** Messages are stored and consumed asynchronously — when you change an event's schema, old messages may still be in flight, and multiple consumer versions may be running simultaneously. Use additive-only changes (adding optional fields), use schema versions in event headers, and test your consumers against both old and new schema versions before deploying.
