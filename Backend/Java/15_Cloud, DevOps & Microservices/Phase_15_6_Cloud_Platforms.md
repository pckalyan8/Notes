# Phase 15.6 — Cloud Platforms for Java

## Why Cloud Matters for Java Developers

The shift to cloud computing is not merely operational — it fundamentally changes how you architect, deploy, and scale Java applications. Instead of managing physical servers and manually installing JDKs, you work with managed services, infrastructure-as-code, and on-demand compute. As a Java developer, understanding the major cloud platforms lets you leverage services like managed databases, message queues, and object storage without reinventing the wheel.

There are three dominant cloud platforms: **AWS (Amazon Web Services)**, **GCP (Google Cloud Platform)**, and **Azure (Microsoft Azure)**. Each has its own service naming conventions and Java SDK, but the underlying concepts are identical. This chapter focuses primarily on AWS (the most widely used) with coverage of GCP and Azure equivalents.

---

## AWS for Java Developers

### Core Compute Services

**EC2 (Elastic Compute Cloud)** is a virtual machine in the cloud. You choose the OS, instance type (CPU/memory/network), and are billed per second. For Java, you would install the JDK, deploy your `.jar`, and manage it yourself. EC2 gives maximum control but maximum operational burden — you are responsible for patching, scaling, monitoring, and high availability.

**ECS (Elastic Container Service)** is AWS's managed container orchestration service. You provide a Docker image and a task definition (CPU, memory, environment variables), and ECS runs your containers on a cluster of EC2 instances. With the **Fargate** launch type, you do not even manage the EC2 instances — AWS handles all that. ECS is the pragmatic choice for teams that want container-based deployments without the full complexity of Kubernetes.

**EKS (Elastic Kubernetes Service)** is managed Kubernetes. AWS handles the Kubernetes control plane (the API server, etcd, scheduler), and you manage your worker nodes. It is the right choice when you need the full Kubernetes ecosystem, have teams already trained on Kubernetes, or need features that ECS doesn't provide (advanced networking, custom admission controllers, etc.).

**Lambda** is serverless compute, covered in depth in Phase 15.7.

### AWS SDK for Java v2

The **AWS SDK for Java v2** is a ground-up rewrite of the v1 SDK, with key improvements: it is non-blocking by default (uses Netty under the hood), supports both synchronous and asynchronous clients, and has much cleaner, builder-based APIs.

```xml
<!-- pom.xml — Using the Bill of Materials (BOM) to manage SDK version consistency -->
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>software.amazon.awssdk</groupId>
      <artifactId>bom</artifactId>
      <version>2.25.0</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <!-- Only include the services you need — avoids a 200MB dependency tree -->
  <dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
  </dependency>
  <dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>sqs</artifactId>
  </dependency>
  <dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>rds</artifactId>
  </dependency>
</dependencies>
```

### Working with Amazon S3

**S3 (Simple Storage Service)** is AWS's object storage. It is the de facto standard for storing binary files: user uploads, report PDFs, backup archives, static assets, and ML model artifacts. Objects are stored in **buckets** and accessed by a key (essentially a file path string). S3 is effectively infinitely scalable and durable (11 nines of durability).

```java
@Service
public class DocumentStorageService {

    private final S3Client s3Client;
    private final String bucketName;

    public DocumentStorageService(@Value("${aws.s3.bucket}") String bucketName) {
        this.bucketName = bucketName;
        // The SDK automatically finds credentials from:
        // 1. Environment variables (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
        // 2. ~/.aws/credentials file (local dev)
        // 3. EC2/ECS instance IAM role (production — PREFERRED, no hardcoded keys)
        this.s3Client = S3Client.builder()
            .region(Region.US_EAST_1)
            .build();
    }

    // Upload a file to S3
    public String uploadDocument(String fileName, byte[] content, String contentType) {
        String objectKey = "documents/" + UUID.randomUUID() + "-" + fileName;

        s3Client.putObject(
            PutObjectRequest.builder()
                .bucket(bucketName)
                .key(objectKey)
                .contentType(contentType)
                .serverSideEncryption(ServerSideEncryption.AES256)  // Encrypt at rest
                .build(),
            RequestBody.fromBytes(content)
        );

        return objectKey;
    }

    // Generate a pre-signed URL — lets a client download directly from S3
    // without going through your application server (saves bandwidth and CPU)
    public String generateDownloadUrl(String objectKey, Duration expiry) {
        S3Presigner presigner = S3Presigner.builder()
            .region(Region.US_EAST_1)
            .build();

        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
            .signatureDuration(expiry)
            .getObjectRequest(r -> r.bucket(bucketName).key(objectKey))
            .build();

        return presigner.presignGetObject(presignRequest).url().toString();
    }

    // Download from S3
    public byte[] downloadDocument(String objectKey) {
        ResponseBytes<GetObjectResponse> responseBytes = s3Client.getObjectAsBytes(
            GetObjectRequest.builder()
                .bucket(bucketName)
                .key(objectKey)
                .build()
        );
        return responseBytes.asByteArray();
    }
}
```

### Working with Amazon SQS and SNS

**SQS (Simple Queue Service)** is a managed message queue. Producers put messages on the queue; consumers poll the queue and process them. SQS provides at-least-once delivery, meaning a message might be delivered more than once — your consumer must be **idempotent** (processing the same message twice has no unintended side effects).

**SNS (Simple Notification Service)** is a pub-sub service. You publish a message to a **topic**, and SNS fans it out to all subscribers — which can be SQS queues, Lambda functions, HTTP endpoints, or email addresses. The combination of SNS + SQS (fan-out pattern) is extremely common: one SNS topic sends to multiple SQS queues so multiple independent services can each process the same event.

```java
@Service
public class OrderEventPublisher {

    private final SqsClient sqsClient;
    private final ObjectMapper objectMapper;

    @Value("${aws.sqs.order-events.url}")
    private String queueUrl;

    public void publishOrderCreated(OrderCreatedEvent event) {
        try {
            String messageBody = objectMapper.writeValueAsString(event);

            SendMessageResponse response = sqsClient.sendMessage(
                SendMessageRequest.builder()
                    .queueUrl(queueUrl)
                    .messageBody(messageBody)
                    // MessageGroupId is required for FIFO queues (preserve order per group)
                    // .messageGroupId(event.getCustomerId())
                    // MessageDeduplicationId prevents duplicate processing in FIFO queues
                    // .messageDeduplicationId(event.getOrderId())
                    .build()
            );

            log.info("Published OrderCreated event, messageId={}", response.messageId());

        } catch (JsonProcessingException e) {
            throw new EventPublishingException("Failed to serialize event", e);
        }
    }
}

// Consumer side — polling SQS for messages
@Component
public class OrderEventConsumer {

    private final SqsClient sqsClient;
    private final ObjectMapper objectMapper;

    @Value("${aws.sqs.order-events.url}")
    private String queueUrl;

    @Scheduled(fixedDelay = 1000)   // Poll every second
    public void pollMessages() {
        ReceiveMessageResponse response = sqsClient.receiveMessage(
            ReceiveMessageRequest.builder()
                .queueUrl(queueUrl)
                .maxNumberOfMessages(10)       // Process up to 10 messages per poll
                .waitTimeSeconds(20)           // Long polling — waits up to 20s for messages
                                               // Long polling is cheaper than short polling
                .visibilityTimeout(60)         // Message hidden from other consumers for 60s
                .build()                       // while we process it
        );

        for (Message message : response.messages()) {
            try {
                OrderCreatedEvent event = objectMapper.readValue(
                    message.body(), OrderCreatedEvent.class);
                
                processOrder(event);

                // Delete the message ONLY after successful processing
                sqsClient.deleteMessage(DeleteMessageRequest.builder()
                    .queueUrl(queueUrl)
                    .receiptHandle(message.receiptHandle())
                    .build());

            } catch (Exception e) {
                // Don't delete the message — it will become visible again after
                // the visibilityTimeout and be retried. After maxReceiveCount
                // retries, SQS moves it to the Dead Letter Queue (DLQ).
                log.error("Failed to process message id={}", message.messageId(), e);
            }
        }
    }
}
```

### Spring Cloud AWS

If you are using Spring Boot, **Spring Cloud AWS** provides first-class integration that feels native to the Spring way of doing things — auto-configured beans, annotation-driven messaging, and `@Value` injection for SSM Parameters.

```xml
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-aws-starter-sqs</artifactId>
</dependency>
<dependency>
  <groupId>io.awspring.cloud</groupId>
  <artifactId>spring-cloud-aws-starter-s3</artifactId>
</dependency>
```

```java
// Spring Cloud AWS SQS listener — much simpler than the raw SDK polling loop
@Component
public class OrderEventListener {

    // This annotation registers a listener that continuously polls the SQS queue
    // It handles message deserialization, acknowledgement, and error handling automatically
    @SqsListener("${aws.sqs.order-events.name}")
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Received OrderCreated event for orderId={}", event.getOrderId());
        // If this method throws an exception, the message is NOT deleted (will retry)
        // If it completes normally, the message is deleted automatically
        inventoryService.reserveItems(event.getItems());
    }
}
```

---

## GCP for Java Developers

**GKE (Google Kubernetes Engine)** is considered the most mature managed Kubernetes offering, largely because Google invented Kubernetes. It supports Autopilot mode (GCP manages nodes) and Standard mode (you manage nodes). GKE is the gold standard for Kubernetes in production.

**Cloud Run** is GCP's serverless container platform. You push a Docker image, and Cloud Run runs it on demand — scaling from zero to hundreds of instances automatically. Unlike AWS Lambda, Cloud Run has no language-specific runtime limitations; any HTTP server in a container can run on it. It is the sweet spot between serverless simplicity and container flexibility.

```bash
# Deploy a Spring Boot app to Cloud Run in one command
gcloud run deploy myapp \
  --image gcr.io/my-project/myapp:1.0.0 \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --memory 512Mi \
  --cpu 1 \
  --set-env-vars="SPRING_PROFILES_ACTIVE=production"
```

**Pub/Sub** is GCP's managed messaging service, analogous to AWS SNS+SQS combined. It is highly scalable and commonly used with Dataflow for stream processing.

**Cloud SQL** is managed MySQL, PostgreSQL, or SQL Server. It handles backups, patching, replication, and failover automatically.

---

## Azure for Java Developers

**AKS (Azure Kubernetes Service)** is Azure's managed Kubernetes offering, tightly integrated with Active Directory (for RBAC via Azure AD), Azure Monitor, and Azure Container Registry.

**Azure Functions** is Microsoft's serverless compute service. It supports Java (JDK 11 and 17) natively and integrates deeply with Azure Event Hubs, Service Bus, and Azure Storage.

**Azure Service Bus** is the enterprise-grade messaging service on Azure, equivalent to AWS SQS for queues and SNS for topics. It supports message ordering, sessions (for stateful message processing), and dead-lettering.

**Spring on Azure** — Microsoft is a major contributor to the Spring ecosystem. **Spring Cloud Azure** provides auto-configured beans for Azure Storage, Service Bus, Key Vault, and Active Directory that feel identical to how Spring Cloud AWS works.

```xml
<!-- Spring Cloud Azure starter -->
<dependency>
  <groupId>com.azure.spring</groupId>
  <artifactId>spring-cloud-azure-starter-servicebus</artifactId>
</dependency>
```

```java
// Consuming messages from Azure Service Bus
@Component
public class OrderProcessor {

    @ServiceBusListener(destination = "order-events-queue")
    public void processOrder(OrderCreatedEvent event) {
        log.info("Processing order from Service Bus: {}", event.getOrderId());
        // Exception → message goes to dead-letter queue after max retries
    }
}
```

---

## Important Concepts Across All Cloud Platforms

### IAM — Identity and Access Management

Every cloud action (reading from S3, writing to a queue, querying a database) is an API call, and every API call must be authorized. **IAM** (or its GCP/Azure equivalent) controls who (or what service) is allowed to perform which actions on which resources.

For Java applications running in the cloud, **never hardcode credentials**. Instead, attach an IAM role to your EC2 instance, ECS task, Kubernetes Pod (via IRSA — IAM Roles for Service Accounts), or Cloud Run service. The SDK automatically fetches temporary credentials from the instance metadata service. This is more secure (credentials rotate automatically) and simpler (no secrets to manage).

```yaml
# Kubernetes Pod — attach an IAM role via annotation (IRSA)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-account
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/order-service-role
```

### Managed Databases vs. Self-Managed

Cloud-managed databases (AWS RDS, GCP Cloud SQL, Azure Database for PostgreSQL) handle automated backups, point-in-time recovery, read replicas, minor version patching, and multi-AZ failover. For the vast majority of Java applications, this is the right choice — the operational overhead of managing your own database server on EC2 is rarely worth it.

The tradeoff is cost (managed services cost more per instance-hour) and slightly less control (you cannot always tune every database parameter). For most applications, this is the right tradeoff.

### Secrets Management

Use cloud-native secrets managers — **AWS Secrets Manager**, **GCP Secret Manager**, or **Azure Key Vault** — to store database passwords, API keys, and certificates. These services rotate secrets automatically, audit every access, and integrate with Spring Boot via the respective Spring Cloud starters.

```yaml
# application.yml — import secrets from AWS Secrets Manager at startup
spring:
  config:
    import: aws-secretsmanager:/myapp/production/db-credentials
```

---

## Best Practices Summary

Design your Java application to be **cloud-native from the start** — stateless (no local disk state that isn't ephemeral), externalize all configuration via environment variables or a config server, handle graceful shutdowns, and emit structured logs to stdout. These practices follow from the 12-Factor App methodology covered in Phase 18 and make your application easy to run on any cloud.

Use **managed services aggressively**. Your team's expertise is best spent on business logic, not on running Kafka clusters, managing PostgreSQL replicas, or patching Redis nodes. Managed services cost more in compute dollars but far less in engineering time.

**Design for multi-region** if your application requires high availability. Cloud providers offer regions (us-east-1, eu-west-1) and Availability Zones (data centers within a region). Deploying across multiple AZs gives you resilience to a data center failure; deploying across multiple regions gives you resilience to a regional outage (rare but catastrophic). Most applications only need multi-AZ; multi-region is a significant architectural commitment.

Never forget **cost management**. Cloud bills can surprise teams that do not monitor them. Enable cost alerts, review your bill weekly during early development, use Reserved Instances or Committed Use Discounts for stable workloads, and right-size your instances — an over-provisioned Java app on a `c5.4xlarge` when a `c5.large` would suffice is money burned.
