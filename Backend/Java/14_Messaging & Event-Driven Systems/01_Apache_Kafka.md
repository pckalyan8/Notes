# 14.1 — Apache Kafka
### Phase 14: Messaging & Event-Driven Systems

---

## The Mental Model: Kafka Is a Distributed Log

Before diving into components and APIs, it's worth building the right mental model for Kafka because it's fundamentally different from traditional message brokers.

Think of a traditional queue like a to-do list. You write a task on it, someone picks it up and crosses it off. Once crossed off, it's gone. Most message systems work this way.

Kafka is not a to-do list. Kafka is an **append-only log** — like a ledger or a journal. Messages are written to the end of the log and stay there. Readers advance through the log at their own pace by tracking where they are (their "offset"). Multiple independent readers can read the same log simultaneously, each at their own position, without affecting each other. Messages are not deleted when read — they're retained for a configured period (often 7 days by default, sometimes forever).

This seemingly simple idea — "just keep an ordered log" — turns out to be enormously powerful. It means:

- You can replay historical events by resetting your position in the log
- Adding a new consumer never affects existing consumers
- Consumer failures don't lose messages — the consumer just re-reads from where it left off
- The log becomes a source of truth that multiple systems can derive state from

Kafka was built at LinkedIn in 2010 to handle their activity stream — billions of events per day including page views, searches, and interactions. It was open-sourced in 2011 and donated to the Apache Foundation. Today it processes trillions of events daily across companies like Netflix, Uber, Airbnb, and Twitter.

---

## Architecture: The Core Components

### Brokers

A **broker** is a single Kafka server — a process running on a machine that stores and serves messages. Kafka is designed to run as a cluster of brokers. A typical production deployment has three to dozens of brokers, providing fault tolerance and horizontal scalability. When one broker fails, the others continue serving traffic.

Each broker handles a subset of the data. No single broker holds all the data, which is what allows Kafka to scale far beyond what fits on one machine.

### Topics

A **topic** is a named category or feed to which messages are published. Think of it like a table in a database, or a folder in a filesystem. Producers write to topics; consumers read from topics. Topics are the fundamental unit of organization in Kafka.

Topics have names like `order-events`, `user-signups`, or `payment-transactions`. You create a topic before producing to it (or configure Kafka to auto-create topics, though that's generally disabled in production).

### Partitions

Each topic is divided into one or more **partitions**. A partition is an ordered, immutable sequence of messages — the actual log. When you write a message to a topic, Kafka assigns it to one partition where it gets an offset number (0, 1, 2, 3...) that permanently identifies its position.

Partitions are the key to both scalability and ordering in Kafka:

- **Scalability:** Different partitions can live on different brokers. A topic with 12 partitions spread across 4 brokers means each broker handles only 3 partitions, sharing the load. You scale a topic by adding partitions.
- **Parallelism:** A consumer group with multiple consumers can assign different partitions to different consumers, processing in parallel.
- **Ordering:** Kafka guarantees that messages within a single partition are ordered by their offset. It makes no ordering guarantee across partitions.

```
Topic: "order-events" (3 partitions)

Partition 0: [msg@0] [msg@1] [msg@2] [msg@3] ──► (newest)
Partition 1: [msg@0] [msg@1] [msg@2]          ──► (newest)  
Partition 2: [msg@0] [msg@1] [msg@2] [msg@3] [msg@4] ──► (newest)
```

### Offsets

An **offset** is a unique, monotonically increasing integer assigned to each message within a partition. Offset 0 is the first message, offset 1 is the second, and so on. Offsets are specific to a partition — partition 0 has its own offset 0, and partition 1 has its own offset 0.

Offsets are how consumers track their progress. A consumer says "I've read up to offset 47 in partition 2 of topic `order-events`." This position is called the consumer's **committed offset**. If the consumer crashes and restarts, it reads its committed offset from Kafka and resumes from where it left off.

### Replication

Each partition can be **replicated** across multiple brokers. You configure a `replication factor` (typically 3 for production). One replica is the **leader** — all reads and writes go to the leader. The others are **followers** that replicate the leader asynchronously. If the leader broker fails, one follower is automatically promoted to leader, and the system continues with no data loss.

```
Partition 0, replication-factor=3:
  Broker 1: LEADER   [replicated copy of all messages]
  Broker 2: FOLLOWER [replicated copy — in sync]
  Broker 3: FOLLOWER [replicated copy — in sync]

If Broker 1 fails → Broker 2 is elected new LEADER automatically
```

### ZooKeeper and KRaft

Historically, Kafka required **Apache ZooKeeper** to manage cluster metadata, leader election, and configuration. ZooKeeper is a separate distributed coordination service, meaning you had to operate two systems. From Kafka 3.3 onwards, Kafka introduced **KRaft mode** (Kafka Raft Metadata mode), which removes the ZooKeeper dependency entirely — Kafka manages its own metadata using a built-in Raft consensus protocol. New deployments should use KRaft.

### Producers

A **producer** is any client application that publishes (writes) messages to Kafka topics. Producers choose which topic to write to. They can also choose which partition within that topic, or let Kafka decide:

- If a message has a **key**, Kafka hashes the key to determine the partition. All messages with the same key always go to the same partition, which guarantees ordering for that key.
- If there is no key, Kafka distributes messages across partitions in a round-robin or sticky manner.

### Consumers

A **consumer** is a client application that reads messages from Kafka topics. Consumers track their own position (offset) in each partition and can process messages at any speed.

### Consumer Groups

A **consumer group** is a set of consumers that cooperate to consume a topic. Kafka automatically assigns partitions to consumers in the group, ensuring each partition is read by exactly one consumer in the group at a time. This is how Kafka parallelizes consumption.

```
Topic: "order-events" (6 partitions)
Consumer Group: "order-processor"

Consumer A gets: Partition 0, Partition 1
Consumer B gets: Partition 2, Partition 3
Consumer C gets: Partition 4, Partition 5

Each message is processed by exactly ONE consumer in the group.
```

If you add a fourth consumer to the group when there are only 3 partitions, the fourth consumer sits idle — it has no partition assigned. You can never have more active consumers in a group than there are partitions.

The power of consumer groups is that different groups are completely independent. Two groups reading the same topic each get all the messages. This is how Kafka supports fan-out: a single stream of order events can be consumed by an order-processor, an analytics service, and a notification service simultaneously, each at their own pace.

---

## Setting Up Kafka Locally with Docker

The fastest way to get a local Kafka for development:

```yaml
# docker-compose.yml
version: '3.8'
services:
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    hostname: kafka
    ports:
      - "9092:9092"     # External port for your Java app
      - "9101:9101"     # JMX monitoring port
    environment:
      # KRaft mode — no ZooKeeper needed
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT'
      KAFKA_ADVERTISED_LISTENERS: 'PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092'
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:29093'
      KAFKA_LISTENERS: 'PLAINTEXT://kafka:29092,CONTROLLER://kafka:29093,PLAINTEXT_HOST://0.0.0.0:9092'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'PLAINTEXT'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_DEFAULT_REPLICATION_FACTOR: 1
      KAFKA_NUM_PARTITIONS: 3
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports:
      - "8090:8080"   # Browse at http://localhost:8090
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
    depends_on:
      - kafka
```

Start with `docker-compose up -d`. Kafka is available at `localhost:9092`, and the UI at `http://localhost:8090`.

---

## Spring Kafka — Setup

Spring Kafka provides a high-level abstraction over the official Kafka Java client that integrates with Spring Boot's auto-configuration.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <!-- Version managed by Spring Boot BOM -->
</dependency>
```

```yaml
# application.yml
spring:
  kafka:
    bootstrap-servers: localhost:9092  # Comma-separated list for production cluster

    # Producer defaults
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                 # Wait for all in-sync replicas to acknowledge
      retries: 3                # Retry on transient failures
      properties:
        enable.idempotence: true     # Prevent duplicate messages on retry
        max.in.flight.requests.per.connection: 5  # Required for idempotent producer

    # Consumer defaults
    consumer:
      group-id: my-application
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest   # Start from beginning if no committed offset exists
      enable-auto-commit: false     # We handle offset commits manually (safer)
      properties:
        spring.json.trusted.packages: "com.example.events"  # Security: only deserialize known types

    # Listener defaults
    listener:
      ack-mode: MANUAL_IMMEDIATE    # Acknowledge after successful processing
      concurrency: 3                # How many listener threads (should match partition count)
      missing-topics-fatal: false   # Don't fail on startup if topic doesn't exist yet
```

---

## Producing Messages

### Basic Producer with `KafkaTemplate`

`KafkaTemplate` is the central Spring Kafka class for sending messages. It wraps the native `KafkaProducer` with a friendlier API.

```java
@Service
public class OrderEventProducer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventProducer.class);
    private static final String TOPIC = "order-events";

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    /**
     * Sends an order event to Kafka.
     *
     * The message KEY is the order ID. This is important: messages with the
     * same key always go to the same partition, which guarantees that all events
     * for a given order are processed in the order they were produced.
     */
    public void sendOrderEvent(OrderEvent event) {
        // send() is asynchronous — it returns a CompletableFuture immediately
        // and sends in the background
        CompletableFuture<SendResult<String, OrderEvent>> future =
            kafkaTemplate.send(TOPIC, event.getOrderId(), event);

        // Add a callback to handle success and failure
        future.whenComplete((result, ex) -> {
            if (ex == null) {
                // Success: log the partition and offset where the message landed
                RecordMetadata metadata = result.getRecordMetadata();
                log.info("Sent order event for orderId={} to partition={} at offset={}",
                    event.getOrderId(),
                    metadata.partition(),
                    metadata.offset());
            } else {
                // Failure: this happens after all retries are exhausted
                // At this point you should decide: dead-letter queue? database fallback?
                log.error("Failed to send order event for orderId={}: {}",
                    event.getOrderId(), ex.getMessage(), ex);
                // In production, consider persisting failed events to a database
                // for later retry (outbox pattern — covered below)
            }
        });
    }

    /**
     * Sends an event to a specific partition.
     * Use this when you need deterministic routing beyond key-based hashing.
     */
    public void sendToSpecificPartition(OrderEvent event, int partition) {
        // ProducerRecord gives you full control: topic, partition, key, value, headers, timestamp
        ProducerRecord<String, OrderEvent> record = new ProducerRecord<>(
            TOPIC,
            partition,            // Explicitly specified partition
            event.getOrderId(),   // Message key
            event                 // Message value
        );

        // Add custom headers — useful for tracing, versioning, routing metadata
        record.headers()
            .add("event-version", "v2".getBytes())
            .add("source-service", "order-service".getBytes())
            .add("correlation-id", UUID.randomUUID().toString().getBytes());

        kafkaTemplate.send(record);
    }

    /**
     * Synchronous send — blocks until Kafka acknowledges.
     * Only use when you absolutely need to know the result before proceeding.
     * Dramatically reduces throughput compared to async sending.
     */
    public SendResult<String, OrderEvent> sendSync(OrderEvent event) {
        try {
            return kafkaTemplate.send(TOPIC, event.getOrderId(), event)
                .get(10, TimeUnit.SECONDS); // Block with a timeout
        } catch (TimeoutException e) {
            throw new RuntimeException("Kafka send timed out after 10 seconds", e);
        } catch (Exception e) {
            throw new RuntimeException("Failed to send to Kafka", e);
        }
    }
}

// The event class that gets serialized to JSON
public record OrderEvent(
    String orderId,
    String customerId,
    String status,       // CREATED, PAID, SHIPPED, DELIVERED, CANCELLED
    BigDecimal amount,
    String currency,
    Instant occurredAt
) {}
```

### Topic Configuration — Creating Topics Programmatically

```java
@Configuration
public class KafkaTopicConfig {

    // Spring Kafka's NewTopic beans are automatically created at startup
    // if they don't already exist
    @Bean
    public NewTopic orderEventsTopic() {
        return TopicBuilder.name("order-events")
            .partitions(6)          // 6 partitions for parallel consumption
            .replicas(3)            // 3 replicas for fault tolerance (need 3-node cluster)
            .config(TopicConfig.RETENTION_MS_CONFIG, String.valueOf(7 * 24 * 60 * 60 * 1000L)) // 7 days
            .config(TopicConfig.COMPRESSION_TYPE_CONFIG, "lz4")  // Compress stored messages
            .config(TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG, "2") // At least 2 replicas must be in sync
            .build();
    }

    @Bean
    public NewTopic orderEventsDeadLetterTopic() {
        // Dead-letter topic for messages that fail processing
        return TopicBuilder.name("order-events.DLT")
            .partitions(6)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, String.valueOf(30 * 24 * 60 * 60 * 1000L)) // 30 days
            .build();
    }
}
```

---

## Consuming Messages

### Basic Consumer with `@KafkaListener`

```java
@Component
public class OrderEventConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventConsumer.class);
    private final OrderService orderService;

    public OrderEventConsumer(OrderService orderService) {
        this.orderService = orderService;
    }

    /**
     * The simplest listener — processes one message at a time.
     *
     * @KafkaListener wires this method to the specified topic(s) and consumer group.
     * Spring Kafka handles partition assignment, offset management, and deserialization.
     *
     * The Acknowledgment parameter is required when ack-mode=MANUAL_IMMEDIATE.
     * You call ack.acknowledge() after successful processing, which commits the offset
     * back to Kafka. If processing fails and you don't acknowledge, the message will
     * be redelivered on the next poll.
     */
    @KafkaListener(
        topics = "order-events",
        groupId = "order-processor",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderEvent(OrderEvent event, Acknowledgment ack) {
        log.info("Received order event: orderId={}, status={}", event.orderId(), event.status());

        try {
            // Process the event — this is your business logic
            orderService.processOrderEvent(event);

            // Only acknowledge AFTER successful processing.
            // This is the critical pattern: at-least-once delivery.
            // If processing succeeds, we commit the offset.
            // If it throws before ack(), Kafka will redeliver the message.
            ack.acknowledge();

        } catch (RecoverableException e) {
            // For transient errors (DB temporarily down, external API timeout):
            // Don't acknowledge — let Kafka redeliver after a delay
            log.warn("Transient error processing orderId={}, will retry: {}", event.orderId(), e.getMessage());
            // Don't call ack.acknowledge() — message will be redelivered

        } catch (NonRecoverableException e) {
            // For permanent errors (bad data, business rule violation):
            // Acknowledge to prevent infinite redelivery, but route to dead-letter
            log.error("Permanent error for orderId={}, routing to DLT: {}", event.orderId(), e.getMessage());
            // Send to dead-letter topic for manual investigation
            // (See dead-letter handling section below)
            ack.acknowledge(); // Commit offset so we don't block other messages
        }
    }

    /**
     * Accessing full message metadata via ConsumerRecord.
     * Use this when you need the partition, offset, headers, or timestamp.
     */
    @KafkaListener(topics = "order-events", groupId = "order-auditor")
    public void auditOrderEvent(ConsumerRecord<String, OrderEvent> record, Acknowledgment ack) {
        log.info("Audit: topic={}, partition={}, offset={}, key={}, timestamp={}",
            record.topic(),
            record.partition(),
            record.offset(),
            record.key(),
            Instant.ofEpochMilli(record.timestamp()));

        // Read custom headers we set in the producer
        String correlationId = new String(record.headers().lastHeader("correlation-id").value());
        log.info("Correlation ID: {}", correlationId);

        orderAuditService.record(record.value(), correlationId);
        ack.acknowledge();
    }

    /**
     * Batch listener — processes a list of messages at once.
     * Much more efficient for high-throughput use cases because you can
     * batch database inserts and make fewer round trips.
     */
    @KafkaListener(
        topics = "order-events",
        groupId = "order-batch-processor",
        containerFactory = "batchKafkaListenerContainerFactory"  // Separate factory for batch mode
    )
    public void handleOrderEventBatch(
            List<OrderEvent> events,
            List<Acknowledgment> acks) {  // One ack per message in the batch

        log.info("Processing batch of {} order events", events.size());

        // Process all events as a batch — great for bulk database operations
        orderService.processOrderEventBatch(events);

        // Acknowledge all messages in the batch
        acks.forEach(Acknowledgment::acknowledge);
    }
}
```

### Listener Container Factory Configuration

The factory configures how listeners run — thread count, error handling, retry behavior:

```java
@Configuration
@EnableKafka  // Required to enable @KafkaListener processing
public class KafkaConsumerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    // ── Standard (single-message) listener factory ──────────────────────────
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
            kafkaListenerContainerFactory(
                ConsumerFactory<String, OrderEvent> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();

        factory.setConsumerFactory(consumerFactory);

        // How many threads to use per listener. Each thread handles one partition.
        // Should ideally match or be a divisor of the partition count.
        factory.setConcurrency(3);

        // Manual acknowledgment mode — we control when offsets are committed
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);

        // How long to wait between polls (heartbeat to keep the group alive)
        factory.getContainerProperties().setPollTimeout(3000);

        // ── Retry and Error Handling ──────────────────────────────────────────
        // DefaultErrorHandler replaces the old SeekToCurrentErrorHandler
        DefaultErrorHandler errorHandler = new DefaultErrorHandler(
            // Dead-letter publishing recoverer: after exhausting retries,
            // send the failed message to the .DLT topic automatically
            new DeadLetterPublishingRecoverer(kafkaTemplate),

            // Retry backoff: wait 1s, 2s, 4s, 8s, then give up (4 retries)
            new FixedBackOff(1000L, 4L)
        );

        // Don't retry on these exceptions — they're non-recoverable
        errorHandler.addNotRetryableExceptions(
            DeserializationException.class,     // Bad message format — can't deserialize
            IllegalArgumentException.class      // Business validation failure
        );

        factory.setCommonErrorHandler(errorHandler);

        return factory;
    }

    // ── Batch listener factory ───────────────────────────────────────────────
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
            batchKafkaListenerContainerFactory(
                ConsumerFactory<String, OrderEvent> consumerFactory) {

        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);   // This is the key difference
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}
```

---

## Consumer Groups — Deep Dive

Understanding consumer groups is critical for designing scalable Kafka consumers.

```java
@Component
public class MultiGroupExample {

    /**
     * GROUP 1: Order Processing Service
     * Processes order events for fulfillment.
     * Uses 3 threads to consume 6 partitions (2 partitions per thread).
     */
    @KafkaListener(
        topics = "order-events",
        groupId = "order-fulfillment-service",
        concurrency = "3"   // Overrides the factory default for this listener
    )
    public void processForFulfillment(OrderEvent event, Acknowledgment ack) {
        orderFulfillmentService.fulfill(event);
        ack.acknowledge();
    }

    /**
     * GROUP 2: Analytics Service — COMPLETELY INDEPENDENT
     * Receives all the same messages as GROUP 1.
     * Each consumer group maintains its own offset — they don't interfere.
     * If this service is down for an hour, it simply catches up when it restarts.
     */
    @KafkaListener(
        topics = "order-events",
        groupId = "order-analytics-service"
    )
    public void processForAnalytics(OrderEvent event, Acknowledgment ack) {
        analyticsService.record(event);
        ack.acknowledge();
    }

    /**
     * GROUP 3: Notification Service
     * Same events, third independent consumer group.
     * Sends email/push notifications for order status changes.
     */
    @KafkaListener(
        topics = "order-events",
        groupId = "notification-service"
    )
    public void processForNotifications(OrderEvent event, Acknowledgment ack) {
        if ("SHIPPED".equals(event.status()) || "DELIVERED".equals(event.status())) {
            notificationService.sendOrderUpdateNotification(event);
        }
        ack.acknowledge();
    }
}
```

### Manual Partition Assignment (Advanced)

Sometimes you need complete control over which partitions a consumer reads:

```java
@KafkaListener(
    topicPartitions = @TopicPartition(
        topic = "order-events",
        partitionOffsets = {
            @PartitionOffset(partition = "0", initialOffset = "0"),   // Start from beginning
            @PartitionOffset(partition = "1", initialOffset = "-1"),  // Start from latest
            @PartitionOffset(partition = "2", initialOffset = "100")  // Start from specific offset
        }
    ),
    groupId = "manual-partition-reader"
)
public void readSpecificPartitions(ConsumerRecord<String, OrderEvent> record, Acknowledgment ack) {
    log.info("Reading from partition {} at offset {}", record.partition(), record.offset());
    processRecord(record.value());
    ack.acknowledge();
}
```

---

## Kafka Streams — Stream Processing Inside Kafka

Kafka Streams is a client library (not a separate cluster) for building real-time stream processing applications. It reads from Kafka topics, processes the data using a pipeline of operations, and writes results back to Kafka (or other stores). It runs inside your application process — no separate cluster needed.

### Key Concepts

A **KStream** is an unbounded, continuously updating stream of records — each record is an independent event. Think of it as a stream of rows in a database.

A **KTable** is a stream where the latest value for each key is kept — it represents the current state of something. Think of it as a database table that's continuously updated by a stream.

```java
@Configuration
@EnableKafkaStreams  // Activates Kafka Streams auto-configuration
public class KafkaStreamsConfig {

    /**
     * This bean builds the stream processing topology.
     * The topology describes the DAG (directed acyclic graph) of operations
     * that transform input streams into output streams.
     */
    @Bean
    public KStream<String, OrderEvent> orderProcessingTopology(StreamsBuilder builder) {

        // ── SOURCE: Read from the input topic ────────────────────────────────
        KStream<String, OrderEvent> orderStream = builder.stream(
            "order-events",
            Consumed.with(Serdes.String(), orderEventSerde())
        );

        // ── FILTER: Keep only completed orders ───────────────────────────────
        KStream<String, OrderEvent> completedOrders = orderStream
            .filter((orderId, event) -> "DELIVERED".equals(event.status()));

        // ── MAP: Transform — extract just the revenue figure ─────────────────
        KStream<String, Double> revenueStream = completedOrders
            .mapValues(event -> event.amount().doubleValue());

        // ── AGGREGATE: Count and sum orders per customer ─────────────────────
        // First, re-key the stream by customerId (not orderId)
        KStream<String, OrderEvent> byCustomer = orderStream
            .selectKey((orderId, event) -> event.customerId());

        // Create a KTable by aggregating the stream
        KTable<String, CustomerOrderStats> customerStats = byCustomer
            .groupByKey(Grouped.with(Serdes.String(), orderEventSerde()))
            .aggregate(
                // Initializer: what does an empty aggregate look like?
                () -> new CustomerOrderStats(0, BigDecimal.ZERO),
                // Aggregator: how do we incorporate a new event into existing state?
                (customerId, event, currentStats) -> new CustomerOrderStats(
                    currentStats.orderCount() + 1,
                    currentStats.totalSpend().add(event.amount())
                ),
                Materialized.<String, CustomerOrderStats, KeyValueStore<Bytes, byte[]>>
                    as("customer-stats-store")  // Named store — lets you query it
                    .withValueSerde(customerStatsSerde())
            );

        // ── WINDOWING: Count orders per 5-minute window ──────────────────────
        KTable<Windowed<String>, Long> ordersPerWindow = orderStream
            .groupByKey()
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
            .count(Materialized.as("order-window-store"));

        // ── SINK: Write processed results to output topics ───────────────────
        // Write completed order revenue to a topic
        revenueStream.to(
            "order-revenue",
            Produced.with(Serdes.String(), Serdes.Double())
        );

        // Convert KTable to stream and write customer stats
        customerStats.toStream()
            .to("customer-stats", Produced.with(Serdes.String(), customerStatsSerde()));

        return orderStream; // Return the source stream (optional, for reference)
    }

    /**
     * Kafka Streams also lets you JOIN streams.
     * Here we enrich orders with customer profile data.
     */
    @Bean
    public KStream<String, EnrichedOrder> enrichedOrderTopology(StreamsBuilder builder) {

        // Stream of raw orders (key = customerId)
        KStream<String, OrderEvent> orders = builder
            .stream("order-events", Consumed.with(Serdes.String(), orderEventSerde()))
            .selectKey((orderId, event) -> event.customerId());

        // Table of customer profiles (key = customerId) — represents current state
        KTable<String, CustomerProfile> customers = builder
            .table("customer-profiles", Consumed.with(Serdes.String(), customerProfileSerde()));

        // JOIN: for each order event, look up the customer profile
        // This is a stream-table join — each stream event is enriched with table data
        KStream<String, EnrichedOrder> enriched = orders.join(
            customers,
            (order, customer) -> new EnrichedOrder(order, customer),
            Joined.with(Serdes.String(), orderEventSerde(), customerProfileSerde())
        );

        enriched.to("enriched-orders", Produced.with(Serdes.String(), enrichedOrderSerde()));

        return enriched;
    }
}
```

### Querying Kafka Streams State Stores

One of Kafka Streams' most powerful features is **interactive queries** — you can query the local state stores that Streams builds and maintains:

```java
@Service
public class CustomerStatsQueryService {

    private final KafkaStreamsRegistry streamsRegistry;

    // Query customer stats from the local state store (no Kafka read needed)
    public CustomerOrderStats getCustomerStats(String customerId) {
        // Get the store that Kafka Streams maintains locally
        ReadOnlyKeyValueStore<String, CustomerOrderStats> store =
            streamsRegistry.getStore("customer-stats-store", QueryableStoreTypes.keyValueStore());

        CustomerOrderStats stats = store.get(customerId);
        return stats != null ? stats : CustomerOrderStats.empty();
    }

    // Iterate all customers' stats
    public List<CustomerOrderStats> getAllStats() {
        ReadOnlyKeyValueStore<String, CustomerOrderStats> store =
            streamsRegistry.getStore("customer-stats-store", QueryableStoreTypes.keyValueStore());

        List<CustomerOrderStats> results = new ArrayList<>();
        try (KeyValueIterator<String, CustomerOrderStats> iter = store.all()) {
            iter.forEachRemaining(kv -> results.add(kv.value));
        }
        return results;
    }
}
```

---

## Kafka Connect — Data Pipelines Without Code

Kafka Connect is a framework for moving data between Kafka and external systems (databases, file systems, cloud services) using pre-built **connectors**. Instead of writing producer/consumer code, you configure a connector via a REST API or JSON file.

The two types of connectors are **Source Connectors** (pull data into Kafka from external systems) and **Sink Connectors** (push data from Kafka to external systems).

```json
// Source Connector: Debezium CDC — capture every database change as a Kafka event
// This is the "Outbox Pattern" at the infrastructure level
{
  "name": "orders-postgres-source",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "kafka_user",
    "database.password": "${file:/run/secrets/db-password:password}",
    "database.dbname": "orders",
    "table.include.list": "public.orders, public.order_items",
    "topic.prefix": "dbserver1",

    // Debezium reads the PostgreSQL WAL (Write-Ahead Log) — a database-native
    // change log — and converts each INSERT/UPDATE/DELETE into a Kafka event.
    // This is called Change Data Capture (CDC).
    "plugin.name": "pgoutput",
    "publication.name": "dbz_publication"
  }
}

// Sink Connector: Write Kafka events to Elasticsearch for search
{
  "name": "orders-elasticsearch-sink",
  "config": {
    "connector.class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector",
    "tasks.max": "3",
    "topics": "order-events",
    "connection.url": "http://elasticsearch:9200",
    "type.name": "order",
    "key.ignore": "false",
    "schema.ignore": "true"
  }
}
```

---

## Schema Registry — Governing Message Formats

As your system grows and teams evolve message schemas independently, you need governance to prevent breaking changes. **Confluent Schema Registry** solves this by storing a versioned history of all message schemas and enforcing compatibility rules.

When a producer sends a message, it registers its schema with the Registry and embeds a schema ID (a 4-byte integer) at the start of the message instead of repeating the full schema. Consumers look up the schema by ID to deserialize.

```yaml
# application.yml — Schema Registry configuration
spring:
  kafka:
    producer:
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
    consumer:
      value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
    properties:
      schema.registry.url: http://localhost:8081
      # Compatibility mode: BACKWARD means new schema can read old messages
      # Other options: FORWARD, FULL, NONE
      specific.avro.reader: true
```

The Schema Registry enforces compatibility. If you try to remove a required field from a schema that existing consumers depend on, the Registry rejects the registration — protecting consumers from breaking changes.

---

## The Outbox Pattern — Guaranteed Event Publishing

One of the most critical patterns in event-driven architecture is ensuring that your database write and your Kafka publish happen atomically. The naive approach fails:

```java
// ❌ WRONG — Non-atomic. If Kafka is down after DB commit, the event is lost!
@Transactional
public Order createOrder(CreateOrderRequest request) {
    Order order = orderRepository.save(new Order(request));  // DB write succeeds
    kafkaTemplate.send("order-events", new OrderEvent(order)); // Kafka might fail!
    return order;
}
```

The **Outbox Pattern** solves this by writing events to a database table (the "outbox") in the same transaction as your business data, then having a separate process read from the outbox and publish to Kafka:

```java
// ✅ CORRECT — Atomic. Both DB write and outbox write are in the same transaction.
@Transactional
public Order createOrder(CreateOrderRequest request) {
    Order order = orderRepository.save(new Order(request));

    // Write the event to the OUTBOX TABLE — same transaction, same DB connection
    OutboxEvent outboxEvent = OutboxEvent.builder()
        .id(UUID.randomUUID())
        .aggregateType("Order")
        .aggregateId(order.getId())
        .eventType("OrderCreated")
        .payload(objectMapper.writeValueAsString(new OrderCreatedEvent(order)))
        .createdAt(Instant.now())
        .status(OutboxEvent.Status.PENDING)
        .build();
    outboxRepository.save(outboxEvent);

    return order;
    // Either BOTH the order AND the outbox event are committed,
    // or NEITHER is — atomicity guaranteed.
}

// A separate @Scheduled job (or Debezium CDC) reads pending outbox events and publishes them
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishPendingOutboxEvents() {
    List<OutboxEvent> pending = outboxRepository.findByStatus(OutboxEvent.Status.PENDING);
    for (OutboxEvent event : pending) {
        try {
            kafkaTemplate.send("order-events", event.getAggregateId(), deserialize(event.getPayload()))
                .get(5, TimeUnit.SECONDS);
            event.setStatus(OutboxEvent.Status.PUBLISHED);
            outboxRepository.save(event);
        } catch (Exception e) {
            log.error("Failed to publish outbox event {}", event.getId(), e);
            event.incrementRetryCount();
            outboxRepository.save(event);
        }
    }
}
```

A more robust approach uses **Debezium** to CDC-capture the outbox table changes and publish them to Kafka automatically — no polling required, no missed events.

---

## Exactly-Once Semantics

Kafka's delivery guarantees deserve careful attention:

**At-most-once:** Messages may be lost but are never delivered more than once. Achieved by committing offsets before processing. Fast but risks data loss.

**At-least-once:** Messages are never lost but may be delivered more than once (duplicates). Achieved by committing offsets after successful processing. This is the most common practical choice.

**Exactly-once:** Messages are delivered exactly once, with no duplicates and no loss. Kafka supports this natively using idempotent producers and transactions, but it requires careful configuration and has performance cost.

```java
// Enabling exactly-once on the producer side
@Bean
public ProducerFactory<String, OrderEvent> exactlyOnceProducerFactory() {
    Map<String, Object> props = new HashMap<>();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);        // Prevents duplicates on retry
    props.put(ProducerConfig.ACKS_CONFIG, "all");                     // All replicas must acknowledge
    props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
    props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-producer-1"); // Unique per producer instance
    // ...serializers...
    return new DefaultKafkaProducerFactory<>(props);
}

// Using Kafka transactions for exactly-once (within Kafka itself)
@Service
public class ExactlyOnceOrderService {

    public void transferOrderToNewTopic(String oldTopic, String newTopic) {
        // executeInTransaction ensures messages are published atomically:
        // either all messages land in the new topic, or none do.
        kafkaTemplate.executeInTransaction(operations -> {
            operations.send(newTopic, "key1", event1);
            operations.send(newTopic, "key2", event2);
            operations.send(newTopic, "key3", event3);
            return true; // Transaction committed
            // If any send throws, the transaction is rolled back
        });
    }
}
```

> **Important note:** Exactly-once semantics only apply to Kafka-to-Kafka operations. If your consumer does a database write, Kafka's transaction doesn't cover the database. True end-to-end exactly-once across Kafka and a database requires idempotent consumer logic (e.g., checking if an event has already been applied before processing it) in addition to Kafka's transaction mechanism.

---

## ⚠️ Important Points & Best Practices

**Design partition keys thoughtfully.** The partition key determines message ordering and which partition stores the message. If you use orderId as the key, all events for an order go to the same partition in order. But if one customer has wildly more orders than others, that partition gets disproportionately large (the "hot partition" problem). For customer-keyed topics, customerId is usually a better key unless your customer distribution is highly skewed.

**Never shrink a topic's partition count.** Kafka only allows increasing partitions, never decreasing. When you add partitions, the key-to-partition mapping changes for new messages, which can break ordering for in-flight work. Plan your partition count carefully upfront. A common pattern is to start with a larger count than you currently need (24 or 48 are popular choices), giving room to add consumers as load grows.

**Set `enable.auto.commit=false` and use manual acknowledgment in production.** Auto-commit commits offsets on a timer, not after successful processing. This means a message can be committed before it's processed, causing silent data loss if your consumer crashes mid-processing. Always use manual acknowledgment and only acknowledge after you have durably processed or persisted the event.

**Implement idempotent consumers regardless of delivery semantics.** Even with at-least-once semantics and careful acknowledgment, network issues, consumer group rebalances, and retries can cause duplicate delivery. Design your message handling to be idempotent — applying the same message twice should produce the same result as applying it once. This usually means tracking which message IDs you've processed in a database.

**Monitor consumer lag.** Consumer lag is the difference between the latest offset in a partition and the consumer group's committed offset. High and growing lag means your consumers are falling behind. Monitor this metric in production — it's the leading indicator of a processing bottleneck.

**Use a dead-letter topic for all failures.** Messages that fail after all retries must go somewhere. Configure Spring Kafka's `DeadLetterPublishingRecoverer` to route them to a `.DLT` topic. Have an operational process to inspect, fix, and replay dead-letter messages. Never silently discard messages.

**Compress messages in production.** Kafka supports LZ4, Snappy, GZIP, and ZSTD compression. LZ4 and Snappy offer a good balance of speed and compression ratio. Compression is applied per batch on the producer and decompressed on the consumer — the broker stores compressed data as-is. This can reduce storage and network costs by 50–90% depending on your data.
