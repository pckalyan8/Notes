# 14.2 — RabbitMQ
### Phase 14: Messaging & Event-Driven Systems

---

## The Mental Model: RabbitMQ Is a Smart Postal Sorting Office

Before examining any code, it helps enormously to carry the right mental picture of RabbitMQ in your head. Kafka is a log you read from — a ledger that retains everything. RabbitMQ is fundamentally different: it is a **broker that routes messages to their destinations and then removes them once delivered**.

Think of it as a sophisticated postal sorting office. A sender drops a letter (message) into the office and writes an address on the envelope (routing key). The sorting office has multiple sorting tables (exchanges) that look at the envelope and decide which mailboxes (queues) to put it into. Each recipient empties their mailbox periodically. Once a letter is picked up and signed for (acknowledged), it's gone — the office doesn't keep copies.

This mental model immediately reveals what RabbitMQ is great at: intelligent routing, task distribution, and request/reply patterns. It also reveals what it's not designed for: replaying old events or retaining a historical log. If you delete a consumer, its messages are gone. If you need to re-process last month's events, RabbitMQ is the wrong tool.

---

## The AMQP Protocol

RabbitMQ is built on **AMQP** — the Advanced Message Queuing Protocol. AMQP 0-9-1 is the wire-level protocol that defines exactly how clients and brokers communicate: how to open channels, declare queues, publish messages, consume, and acknowledge.

Understanding AMQP is important because many of RabbitMQ's concepts (exchanges, bindings, virtual hosts) come directly from the protocol specification, not from RabbitMQ's own design choices. This also means AMQP 0-9-1 clients in other languages (Python's `pika`, Node's `amqplib`) work identically against RabbitMQ because they all speak the same protocol.

---

## Core Architecture Components

### Virtual Hosts (vhosts)

A **virtual host** is a logical partition within a single RabbitMQ instance. Think of it like a schema in a database — it provides a namespace for exchanges, queues, and bindings. Users are granted permissions to specific vhosts. Different applications sharing one RabbitMQ server use different vhosts to avoid namespace collisions.

### Connections and Channels

Opening a TCP connection to RabbitMQ is expensive. A **connection** is a long-lived TCP socket. Within a single connection you can open multiple **channels** — lightweight virtual connections that are cheap to create and destroy. The pattern is: one `Connection` per application, one `Channel` per thread.

### Exchanges

An **exchange** is the component that receives messages from producers and decides which queues to route them to. The exchange never stores messages — it only routes them. The routing decision is made by matching the message's **routing key** against the exchange's **bindings**.

### Bindings

A **binding** is a link between an exchange and a queue, optionally with a **binding key** or pattern. When a message arrives at an exchange, the exchange compares the message's routing key against all its bindings to decide which queues should receive a copy.

### Queues

A **queue** stores messages until a consumer retrieves them. Queues have properties: durable (survive broker restart), exclusive (only one connection), and auto-delete (deleted when last consumer disconnects). Messages within a queue are ordered FIFO.

---

## The Four Exchange Types — This Is Where RabbitMQ's Power Lives

Understanding exchange types is the key to using RabbitMQ effectively. Each type implements a different routing algorithm.

### 1. Direct Exchange — Exact Routing Key Match

A direct exchange routes a message to a queue whose binding key exactly matches the message's routing key. It's the simplest exchange type and is useful for routing messages to specific workers.

```
Producer sends message with routing key: "order.created"
                          │
                    [Direct Exchange]
                          │
        ┌─────────────────┼─────────────────┐
   binding key:      binding key:       binding key:
  "order.created"  "order.shipped"    "payment.done"
        │                                   
   [Queue: order-creator-service]      (no match — not routed here)
```

### 2. Topic Exchange — Pattern Matching on Routing Key

A topic exchange routes based on patterns using wildcards:
- `*` matches exactly one word (a dot-delimited segment)
- `#` matches zero or more words

```
Routing key: "order.europe.created"
              │
        [Topic Exchange]
              │
   ┌──────────┼────────────────┐
"order.#"  "*.europe.*"  "payment.#"
   │            │
[Queue A]    [Queue B]
   ↑            ↑
matches!      matches!  (both receive the message)
```

This is the most flexible and widely used exchange type. It lets you subscribe to exactly the events you care about with fine-grained patterns.

### 3. Fanout Exchange — Broadcast to All Queues

A fanout exchange ignores routing keys entirely and delivers a copy of every message to every queue bound to it. It's a broadcast mechanism.

```
Producer sends any message
              │
       [Fanout Exchange]
              │
    ┌─────────┼──────────┐
[Queue A]  [Queue B]  [Queue C]
    ↑          ↑          ↑
all receive a copy of every message
```

Fanout is ideal for publish-subscribe scenarios where you want to notify multiple independent consumers of every event — for example, broadcasting a "system maintenance" announcement to all connected services.

### 4. Headers Exchange — Route by Message Headers

A headers exchange ignores the routing key and instead routes based on message header attributes. You specify header key-value pairs in the binding, and the exchange matches messages whose headers satisfy the binding's conditions.

```java
// Producer sets headers
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
    .headers(Map.of("region", "europe", "priority", "high"))
    .build();

// Queue bound with x-match=all: message must match ALL headers
// "region=europe" AND "priority=high" → routes to this queue
```

Headers exchange is rarely used compared to topic exchange, but it's useful when you need to route on structured metadata rather than a string key.

---

## Setting Up RabbitMQ with Docker

```yaml
# docker-compose.yml
version: '3.8'
services:
  rabbitmq:
    image: rabbitmq:3.13-management  # Includes the Management UI plugin
    hostname: rabbitmq
    ports:
      - "5672:5672"    # AMQP protocol port
      - "15672:15672"  # Management UI at http://localhost:15672
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin123
      RABBITMQ_DEFAULT_VHOST: /
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
volumes:
  rabbitmq-data:
```

Browse to `http://localhost:15672` (admin/admin123) for the management dashboard — it lets you inspect exchanges, queues, bindings, and message rates in real time. This UI is invaluable during development.

---

## Spring AMQP Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: admin
    password: admin123
    virtual-host: /

    # Connection pool — reuse connections rather than opening one per request
    connection-timeout: 5000

    # Publisher confirms — get callback when broker accepts a message
    publisher-confirm-type: correlated

    # Publisher returns — get callback if a message cannot be routed to any queue
    publisher-returns: true

    listener:
      simple:
        acknowledge-mode: manual         # We commit acknowledgments manually
        concurrency: 3                   # Min consumer threads
        max-concurrency: 10              # Max consumer threads (scales up under load)
        prefetch: 20                     # How many unacknowledged messages per consumer
        retry:
          enabled: true
          initial-interval: 1000ms      # Wait 1s before first retry
          max-attempts: 3               # Retry up to 3 times
          multiplier: 2.0               # Exponential backoff (1s, 2s, 4s)
```

---

## Declaring Infrastructure: Exchanges, Queues, and Bindings

Spring AMQP recommends declaring your messaging topology in a `@Configuration` class. This code is idempotent — it declares the components if they don't exist, or verifies them if they do.

```java
@Configuration
public class RabbitMQConfig {

    // ── Constants — centralize names to avoid typos ──────────────────────────

    // Exchange names
    public static final String ORDER_EXCHANGE = "order.events";         // Topic exchange
    public static final String NOTIFICATION_EXCHANGE = "notifications"; // Fanout exchange
    public static final String PAYMENT_EXCHANGE = "payment.direct";     // Direct exchange
    public static final String DEAD_LETTER_EXCHANGE = "order.events.dlx";

    // Queue names
    public static final String ORDER_CREATED_QUEUE = "order.created.queue";
    public static final String ORDER_SHIPPED_QUEUE = "order.shipped.queue";
    public static final String ORDER_ALL_QUEUE = "order.all.queue";
    public static final String NOTIFICATION_QUEUE = "notification.queue";
    public static final String DEAD_LETTER_QUEUE = "order.dead-letter.queue";
    public static final String PAYMENT_QUEUE = "payment.processing.queue";


    // ── EXCHANGES ─────────────────────────────────────────────────────────────

    @Bean
    public TopicExchange orderExchange() {
        return ExchangeBuilder.topicExchange(ORDER_EXCHANGE)
            .durable(true)  // Survives broker restart
            .build();
    }

    @Bean
    public FanoutExchange notificationExchange() {
        return ExchangeBuilder.fanoutExchange(NOTIFICATION_EXCHANGE)
            .durable(true)
            .build();
    }

    @Bean
    public DirectExchange paymentExchange() {
        return ExchangeBuilder.directExchange(PAYMENT_EXCHANGE)
            .durable(true)
            .build();
    }

    // Dead-letter exchange — receives messages that fail processing or expire
    @Bean
    public DirectExchange deadLetterExchange() {
        return ExchangeBuilder.directExchange(DEAD_LETTER_EXCHANGE)
            .durable(true)
            .build();
    }


    // ── QUEUES ────────────────────────────────────────────────────────────────

    // Queue for "order.created" events
    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder.durable(ORDER_CREATED_QUEUE)
            // If processing fails, route to the dead-letter exchange
            .withArgument("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE)
            .withArgument("x-dead-letter-routing-key", "order.dead-letter")
            // Messages expire after 24 hours if not consumed
            .withArgument("x-message-ttl", 86_400_000)
            .build();
    }

    @Bean
    public Queue orderShippedQueue() {
        return QueueBuilder.durable(ORDER_SHIPPED_QUEUE)
            .withArgument("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE)
            .build();
    }

    // Queue that receives ALL order events (will bind with a wildcard pattern)
    @Bean
    public Queue orderAllQueue() {
        return QueueBuilder.durable(ORDER_ALL_QUEUE)
            .withArgument("x-dead-letter-exchange", DEAD_LETTER_EXCHANGE)
            .build();
    }

    @Bean
    public Queue notificationQueue() {
        return QueueBuilder.durable(NOTIFICATION_QUEUE).build();
    }

    // Dead-letter queue — holds failed messages for investigation
    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable(DEAD_LETTER_QUEUE).build();
    }

    // Payment queue with maximum length — reject new messages when full
    @Bean
    public Queue paymentQueue() {
        return QueueBuilder.durable(PAYMENT_QUEUE)
            .withArgument("x-max-length", 10_000)          // Max 10k messages
            .withArgument("x-overflow", "reject-publish")  // Reject new when full
            .build();
    }


    // ── BINDINGS — wiring exchanges to queues ─────────────────────────────────

    // Topic Exchange bindings
    // "order.created.queue" receives messages with routing key "order.created.*"
    // Example keys that match: "order.created.europe", "order.created.us"
    @Bean
    public Binding orderCreatedBinding(Queue orderCreatedQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderCreatedQueue)
            .to(orderExchange)
            .with("order.created.*");
    }

    // "order.shipped.queue" receives only "order.shipped.*" messages
    @Bean
    public Binding orderShippedBinding(Queue orderShippedQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderShippedQueue)
            .to(orderExchange)
            .with("order.shipped.*");
    }

    // "order.all.queue" receives ALL order events (order.#)
    @Bean
    public Binding orderAllBinding(Queue orderAllQueue, TopicExchange orderExchange) {
        return BindingBuilder.bind(orderAllQueue)
            .to(orderExchange)
            .with("order.#");   // # matches zero or more words
    }

    // Fanout Exchange binding — no routing key needed, receives everything
    @Bean
    public Binding notificationBinding(Queue notificationQueue, FanoutExchange notificationExchange) {
        return BindingBuilder.bind(notificationQueue)
            .to(notificationExchange);
    }

    // Direct Exchange binding — exact routing key match
    @Bean
    public Binding paymentBinding(Queue paymentQueue, DirectExchange paymentExchange) {
        return BindingBuilder.bind(paymentQueue)
            .to(paymentExchange)
            .with("payment.process");
    }

    // Dead-letter binding
    @Bean
    public Binding deadLetterBinding(Queue deadLetterQueue, DirectExchange deadLetterExchange) {
        return BindingBuilder.bind(deadLetterQueue)
            .to(deadLetterExchange)
            .with("order.dead-letter");
    }


    // ── MESSAGE CONVERTER ─────────────────────────────────────────────────────

    // By default Spring AMQP uses Java serialization — always override this with JSON!
    // Java serialization is brittle, slow, and a security risk.
    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    // The RabbitTemplate is the central class for sending messages
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());

        // Callback when broker confirms receipt (publisher confirms)
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (!ack) {
                log.error("Message rejected by broker: {}", cause);
                // Handle rejected message — retry, alert, store for retry
            }
        });

        // Callback when a message cannot be routed to any queue (mandatory=true)
        template.setReturnsCallback(returned -> {
            log.error("Message returned — no queue for routing key {}: {}",
                returned.getRoutingKey(), returned.getMessage());
        });

        template.setMandatory(true); // Trigger returns callback for unroutable messages
        return template;
    }
}
```

---

## Publishing Messages

```java
@Service
public class OrderEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public OrderEventPublisher(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    /**
     * Publish an order created event.
     *
     * The routing key "order.created.europe" will be matched against all bindings
     * on the ORDER_EXCHANGE. Our configuration has queues bound to:
     *   "order.created.*"  → matches (1 word after "order.created")
     *   "order.#"          → matches (any words after "order")
     * Both queues will receive this message.
     */
    public void publishOrderCreated(Order order) {
        String routingKey = "order.created." + order.getRegion(); // e.g., "order.created.europe"

        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(), order.getCustomerId(), order.getAmount(), Instant.now()
        );

        // Exchange + routing key + payload
        rabbitTemplate.convertAndSend(RabbitMQConfig.ORDER_EXCHANGE, routingKey, event);
    }

    /**
     * Publish with message properties — set priority, expiry, headers, correlation ID.
     */
    public void publishWithProperties(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(order.getId(), order.getCustomerId(),
            order.getAmount(), Instant.now());

        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE,
            "order.created." + order.getRegion(),
            event,
            // MessagePostProcessor lets you set AMQP message properties
            message -> {
                message.getMessageProperties().setContentType("application/json");
                message.getMessageProperties().setPriority(8);           // High priority
                message.getMessageProperties().setExpiration("3600000"); // Expire in 1 hour
                message.getMessageProperties().setCorrelationId(UUID.randomUUID().toString());
                message.getMessageProperties().setHeader("source-service", "order-service");
                message.getMessageProperties().setHeader("schema-version", "v2");
                return message;
            }
        );
    }

    /**
     * Publish a broadcast — fanout exchange ignores routing key.
     * ALL queues bound to this exchange receive the message.
     */
    public void broadcastSystemNotification(SystemNotification notification) {
        // Third parameter is the routing key — fanout ignores it, but
        // convention is to pass an empty string
        rabbitTemplate.convertAndSend(RabbitMQConfig.NOTIFICATION_EXCHANGE, "", notification);
    }

    /**
     * Publisher Confirms — get a callback confirming the broker stored the message.
     * Use this when you need guaranteed delivery from producer to broker.
     */
    public void publishWithConfirm(OrderCreatedEvent event) {
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());

        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE, "order.created.us", event, correlationData
        );

        try {
            // Block and wait for the broker's confirmation (with timeout)
            CorrelationData.Confirm confirm = correlationData.getFuture().get(5, TimeUnit.SECONDS);
            if (confirm.isAck()) {
                log.info("Message confirmed by broker: {}", correlationData.getId());
            } else {
                log.error("Message NACKed by broker: {}", confirm.getReason());
                // Retry or fall back to outbox pattern
            }
        } catch (TimeoutException e) {
            log.error("Broker confirmation timed out");
        }
    }
}
```

---

## Consuming Messages

### `@RabbitListener` — The Standard Approach

```java
@Component
public class OrderEventConsumer {

    private static final Logger log = LoggerFactory.getLogger(OrderEventConsumer.class);

    private final OrderService orderService;
    private final RabbitTemplate rabbitTemplate;

    public OrderEventConsumer(OrderService orderService, RabbitTemplate rabbitTemplate) {
        this.orderService = orderService;
        this.rabbitTemplate = rabbitTemplate;
    }

    /**
     * Listen to the ORDER_CREATED_QUEUE.
     *
     * Spring AMQP deserializes the JSON payload into an OrderCreatedEvent
     * using the Jackson2JsonMessageConverter we configured.
     *
     * The Channel + deliveryTag parameters allow us to manually acknowledge.
     */
    @RabbitListener(queues = RabbitMQConfig.ORDER_CREATED_QUEUE)
    public void handleOrderCreated(
            OrderCreatedEvent event,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        log.info("Received OrderCreated event: orderId={}", event.orderId());

        try {
            orderService.processOrderCreated(event);

            // basicAck: acknowledge this message (deliveryTag, multiple=false)
            // multiple=false: ack only this message, not all previous unacked messages
            channel.basicAck(deliveryTag, false);

        } catch (TransientException e) {
            // Transient failure — requeue the message for redelivery
            log.warn("Transient failure for orderId={}, requeuing", event.orderId(), e);
            // basicNack(deliveryTag, multiple, requeue)
            // requeue=true: put the message back on the queue for retry
            channel.basicNack(deliveryTag, false, true);

        } catch (PermanentException e) {
            // Permanent failure — reject without requeue.
            // Because we configured x-dead-letter-exchange on this queue,
            // RabbitMQ will automatically route the message to the DLX/DLQ.
            log.error("Permanent failure for orderId={}, routing to DLQ", event.orderId(), e);
            // basicNack(deliveryTag, multiple, requeue=false) → goes to DLQ
            channel.basicNack(deliveryTag, false, false);
        }
    }

    /**
     * Accessing full AMQP message metadata with MessageHeaders.
     */
    @RabbitListener(queues = RabbitMQConfig.ORDER_ALL_QUEUE)
    public void handleAnyOrderEvent(
            Message message,    // Raw AMQP message — gives you headers + body
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag,
            @Header(value = "source-service", required = false) String sourceService) throws IOException {

        String routingKey = message.getMessageProperties().getReceivedRoutingKey();
        log.info("Received message from routing key: {}, source: {}", routingKey, sourceService);

        // You can inspect all headers:
        Map<String, Object> headers = message.getMessageProperties().getHeaders();
        String schemaVersion = (String) headers.get("schema-version");

        try {
            // Route to different handlers based on the routing key
            if (routingKey.startsWith("order.created")) {
                OrderCreatedEvent event = deserialize(message.getBody(), OrderCreatedEvent.class);
                orderService.processOrderCreated(event);
            } else if (routingKey.startsWith("order.shipped")) {
                OrderShippedEvent event = deserialize(message.getBody(), OrderShippedEvent.class);
                orderService.processOrderShipped(event);
            }
            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            channel.basicNack(deliveryTag, false, false);
        }
    }

    /**
     * Concurrent consumers — Spring AMQP can run multiple threads on the same queue.
     * concurrency="3-10" means: start 3 threads, scale up to 10 under load.
     */
    @RabbitListener(
        queues = RabbitMQConfig.PAYMENT_QUEUE,
        concurrency = "3-10"  // Scales automatically based on queue depth
    )
    public void handlePayment(
            PaymentEvent event,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        log.info("Processing payment: paymentId={}", event.paymentId());
        paymentService.process(event);
        channel.basicAck(deliveryTag, false);
    }

    /**
     * Declare queue, exchange, and binding inline in the annotation.
     * Useful for quick setups — declare everything in one place.
     * For production, prefer the explicit @Configuration approach (more control).
     */
    @RabbitListener(
        bindings = @QueueBinding(
            value = @Queue(
                name = "email.notification.queue",
                durable = "true",
                arguments = @Argument(name = "x-dead-letter-exchange", value = "notifications.dlx")
            ),
            exchange = @Exchange(
                name = "notifications",
                type = ExchangeTypes.FANOUT,
                durable = "true"
            )
        )
    )
    public void handleEmailNotification(
            SystemNotification notification,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        emailService.send(notification);
        channel.basicAck(deliveryTag, false);
    }
}
```

---

## Dead Letter Queues — Handling Failed Messages

A **Dead Letter Queue (DLQ)** is where messages go when they can't be processed. RabbitMQ automatically routes messages to the Dead Letter Exchange (and then to the DLQ) in three situations:

1. A consumer rejects the message with `requeue=false` (`basicNack` or `basicReject` with requeue=false)
2. The message's per-message or per-queue TTL expires
3. The queue reaches its maximum length limit

```java
@Component
public class DeadLetterProcessor {

    /**
     * A separate consumer for the dead-letter queue.
     * This is your "last resort" handler — it receives messages that
     * failed processing even after retries.
     *
     * Operational responsibilities:
     * - Log full details for investigation
     * - Alert the on-call team
     * - Store in a persistent store for manual replaying later
     */
    @RabbitListener(queues = RabbitMQConfig.DEAD_LETTER_QUEUE)
    public void handleDeadLetter(
            Message message,
            Channel channel,
            @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        // RabbitMQ adds x-death headers explaining why the message was dead-lettered
        List<Map<String, Object>> xDeath =
            (List<Map<String, Object>>) message.getMessageProperties().getHeaders().get("x-death");

        if (xDeath != null && !xDeath.isEmpty()) {
            Map<String, Object> deathInfo = xDeath.get(0);
            String originalQueue = (String) deathInfo.get("queue");
            String deathReason = (String) deathInfo.get("reason"); // "rejected", "expired", "maxlen"
            Long deathCount = (Long) deathInfo.get("count");

            log.error("Dead letter received: originalQueue={}, reason={}, count={}, body={}",
                originalQueue, deathReason, deathCount,
                new String(message.getBody(), StandardCharsets.UTF_8));

            // Store for manual investigation and replay
            deadLetterRepository.save(new DeadLetterRecord(
                originalQueue, deathReason, message.getBody(), Instant.now()
            ));

            // Send alert to monitoring system
            alertingService.sendAlert(
                "Dead letter received from " + originalQueue + " — reason: " + deathReason
            );
        }

        // Always ack dead-letter messages — if you nack them, they'll loop
        channel.basicAck(deliveryTag, false);
    }

    /**
     * Admin endpoint to replay a dead-letter message back to its original queue.
     * This would be called manually after investigating and fixing the root cause.
     */
    public void replayDeadLetterMessage(String deadLetterId) {
        DeadLetterRecord record = deadLetterRepository.findById(deadLetterId)
            .orElseThrow(() -> new NotFoundException("Dead letter not found: " + deadLetterId));

        // Re-publish to the original queue directly
        rabbitTemplate.send(record.getOriginalQueue(), new Message(record.getPayload()));
        deadLetterRepository.delete(record);
        log.info("Replayed dead letter {} back to {}", deadLetterId, record.getOriginalQueue());
    }
}
```

---

## Request-Reply Pattern — RPC over RabbitMQ

RabbitMQ natively supports a **request-reply** (RPC) pattern where a producer sends a message and waits for a reply. This is similar to a synchronous HTTP call but uses Kafka as the transport — useful when you need response data but want the routing flexibility of messaging.

```java
// ── RPC Server (handles the request and sends back a reply) ──────────────────
@Component
public class PricingRpcServer {

    /**
     * A reply queue is typically exclusive to the requester.
     * The requester creates a temporary, anonymous reply queue, sets the
     * "reply-to" property to its name, and the server sends the response there.
     */
    @RabbitListener(queues = "pricing.request.queue")
    public PriceResponse calculatePrice(PriceRequest request) {
        // Spring AMQP's @RabbitListener handles the reply automatically when
        // you return a value — it sends the return value to the replyTo queue.
        log.info("Calculating price for productId={}", request.getProductId());
        BigDecimal price = pricingService.calculatePrice(request);
        return new PriceResponse(request.getProductId(), price, "USD");
    }
}


// ── RPC Client (sends a request and waits for the reply) ─────────────────────
@Service
public class PricingRpcClient {

    private final RabbitTemplate rabbitTemplate;

    public PricingRpcClient(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    /**
     * convertSendAndReceive is Spring AMQP's built-in RPC method.
     * It:
     * 1. Creates a temporary exclusive reply queue
     * 2. Sets the replyTo and correlationId message properties
     * 3. Sends the message to the request queue
     * 4. Blocks and waits for a reply (up to the configured timeout)
     * 5. Returns the deserialized reply object
     * 6. Cleans up the temporary reply queue
     */
    public PriceResponse getPrice(String productId) {
        PriceRequest request = new PriceRequest(productId, 1);

        PriceResponse response = (PriceResponse) rabbitTemplate.convertSendAndReceive(
            RabbitMQConfig.PAYMENT_EXCHANGE,
            "pricing.request",
            request
        );

        if (response == null) {
            throw new ServiceUnavailableException("Pricing service did not respond within timeout");
        }
        return response;
    }
}
```

---

## Message Acknowledgment — The Prefetch and Flow Control Story

Prefetch (also called QoS — Quality of Service) is one of the most important settings to get right. It controls how many unacknowledged messages RabbitMQ will send to a consumer at once.

If prefetch is unlimited (the default when `basicQos` is not set), RabbitMQ dumps all queued messages onto the first available consumer, filling its memory. Other consumers starve. If prefetch is 1, each consumer can only hold one message at a time — perfectly fair but lower throughput. A value of 10–50 is a common starting point for most applications.

```java
// Set prefetch count via Spring configuration (in application.yml):
// spring.rabbitmq.listener.simple.prefetch: 10

// Or set programmatically in the container factory:
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
        ConnectionFactory connectionFactory) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
    factory.setPrefetchCount(10);  // Consumer holds at most 10 unacked messages at a time
    factory.setConcurrentConsumers(3);
    factory.setMaxConcurrentConsumers(10);
    factory.setMessageConverter(new Jackson2JsonMessageConverter());
    return factory;
}
```

The relationship between prefetch and consumer threads is worth understanding clearly. With `concurrency=3` and `prefetch=10`, each thread can hold 10 unacknowledged messages, meaning 30 messages can be in-flight across the 3 threads simultaneously. Setting prefetch too high wastes memory and creates unfair distribution; setting it too low throttles throughput.

---

## Testing RabbitMQ with Spring Boot

```java
@SpringBootTest
@Testcontainers  // Starts a real RabbitMQ container for integration tests
class OrderEventConsumerIntegrationTest {

    @Container
    static RabbitMQContainer rabbitMQ = new RabbitMQContainer("rabbitmq:3.13-management");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.rabbitmq.host", rabbitMQ::getHost);
        registry.add("spring.rabbitmq.port", rabbitMQ::getAmqpPort);
    }

    @Autowired
    private RabbitTemplate rabbitTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void whenOrderCreatedEventReceived_thenOrderIsProcessed() throws InterruptedException {
        OrderCreatedEvent event = new OrderCreatedEvent("order-123", "customer-456",
            BigDecimal.valueOf(99.99), Instant.now());

        rabbitTemplate.convertAndSend(
            RabbitMQConfig.ORDER_EXCHANGE, "order.created.us", event
        );

        // Wait up to 5 seconds for the listener to process the message
        await().atMost(5, TimeUnit.SECONDS)
            .untilAsserted(() ->
                assertThat(orderRepository.findById("order-123")).isPresent()
            );
    }
}
```

---

## ⚠️ Important Points & Best Practices

**Always declare queues as durable and mark your messages as persistent.** A non-durable queue is deleted when RabbitMQ restarts. A non-persistent message (delivery mode 1) is held only in memory and lost on restart. For any business-critical data, use `QueueBuilder.durable()` and set `MessageProperties.setDeliveryMode(MessageDeliveryMode.PERSISTENT)`. There is a minor performance cost, but it's always worth paying for correctness.

**Never use `requeue=true` in a tight retry loop.** If message processing fails and you immediately requeue (basicNack with requeue=true), the message goes straight back to the front of the queue and your consumer immediately processes it again. In a broken state, this creates a requeue storm that can overwhelm the broker and starve other messages. Instead, use Spring AMQP's retry interceptor (configured in `application.yml`) which adds exponential backoff delays, or route to a DLQ after N failures.

**Configure a dead-letter exchange on every business-critical queue.** Without a DLQ, messages that fail after retries are simply discarded — you lose data silently. With a DLQ, failed messages accumulate in a separate queue where they can be inspected, fixed, and replayed. This is non-negotiable for any production system.

**Use topic exchanges by default.** Direct exchanges are simple but inflexible — adding a new consumer often means changing binding configuration and redeploying. Topic exchanges with wildcard patterns let you add new consumers with new routing patterns without touching existing consumers. Start with topic exchange and only use direct or fanout when you have a clear reason.

**Set a reasonable message TTL on queues.** Without TTL, a slow or down consumer causes messages to pile up indefinitely, eventually exhausting broker memory or disk. Setting a TTL (via `x-message-ttl`) and configuring a dead-letter exchange means expired messages move to the DLQ rather than being silently dropped — you still have visibility and can replay if needed.

**Monitor queue depth as your primary health metric.** A queue that's filling up but not draining is the most important early warning sign. Set up alerts on queue depth using RabbitMQ's management API or Prometheus integration, and respond when queues consistently grow rather than drain.
