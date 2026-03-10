# Phase 16.4 — Secrets Management

## What Is a Secret and Why Does It Demand Special Handling?

A **secret** is any piece of information that, if exposed to an unauthorized party, grants them power they should not have. Database passwords, API keys, cryptographic keys, OAuth client secrets, TLS private keys, and service account tokens are all secrets. The thread running through all of them is that they are credentials — possession is proof of identity and implies permission.

Secrets are uniquely dangerous compared to other sensitive data for one reason: they are **operational**. A leaked credit card number is harmful to one customer. A leaked database password is harmful to every record in the database. A leaked cloud provider credential can result in infrastructure being destroyed or ransomed. A leaked signing key invalidates the integrity of every token and document ever signed with it.

The core discipline of secrets management addresses three challenges: how to prevent secrets from leaking in the first place (avoiding exposure), how to minimize the damage if they do leak (limiting blast radius), and how to recover quickly when they are compromised (rotation and revocation).

---

## The Cardinal Sin — Hardcoding Secrets

The most common secrets management failure, and the one with the most severe consequences, is hardcoding credentials directly in source code:

```java
// ❌ CATASTROPHICALLY BAD — seen in real production codebases every day
@Service
public class PaymentService {
    // Do you see what's wrong? These values are now:
    // 1. In every developer's local Git history, forever
    // 2. In your CI/CD pipeline logs
    // 3. In your Docker image layers
    // 4. In any code review system
    // 5. Exposed if your GitHub repo is accidentally made public
    private static final String DATABASE_URL = "jdbc:postgresql://prod-db.example.com:5432/payments";
    private static final String DATABASE_USER = "paymentapp";
    private static final String DATABASE_PASSWORD = "Tr0ub4dor&3";   // ← Compromised forever
    private static final String STRIPE_SECRET_KEY = "sk_live_abc123xyz789..."; // ← Burning money
}
```

This is not a hypothetical risk. Tools like **truffleHog**, **git-secrets**, and GitHub's built-in secret scanning actively crawl public and private repositories looking for patterns that match known secret formats (AWS access keys, Stripe keys, Slack tokens, etc.). When GitHub's secret scanning detects a committed credential, it can automatically notify the affected service provider, who may revoke it. The window between a secret being committed and being exploited is often measured in minutes, not hours.

Once a secret is committed to Git, the conventional wisdom is that it must be treated as **permanently compromised** — even if you delete the file in the next commit, the secret lives in the repository's history. Changing the file does not rewrite history. You must rotate (change) the credential immediately and audit what was accessed with it.

---

## Environment Variables — The Baseline Approach

The simplest and most universally supported technique for separating secrets from code is **environment variables**. The application reads its configuration from the process environment at runtime rather than from compiled-in values. This satisfies the 12-Factor App methodology's "Store config in the environment" principle.

```bash
# Set environment variables before starting the application
export DB_PASSWORD="Tr0ub4dor&3"
export STRIPE_SECRET_KEY="sk_live_abc123..."
export JWT_SECRET="base64encodedvalue..."

# Spring Boot reads these automatically
java -jar myapp.jar
```

Spring Boot's property binding maps environment variables directly to `application.yml` properties. The mapping converts `.` to `_` and uppercases, so `spring.datasource.password` maps to the environment variable `SPRING_DATASOURCE_PASSWORD`:

```yaml
# application.yml — reference environment variables with placeholders
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:mydb}
    username: ${DB_USER}
    password: ${DB_PASSWORD}   # Required — no default, will fail fast if not set

stripe:
  secret-key: ${STRIPE_SECRET_KEY}

jwt:
  secret: ${JWT_SECRET}
  expiry-minutes: ${JWT_EXPIRY_MINUTES:15}   # Optional with default

# CRITICAL: Never provide default values for actual secrets:
# password: ${DB_PASSWORD:changeme}   ← This is a trap — if the env var is missing,
#                                        the insecure default is silently used in production
```

```java
// Consuming secrets in your application
@Configuration
public class StripeConfig {

    // @Value reads from Spring's property system, which includes environment variables
    @Value("${stripe.secret-key}")
    private String secretKey;

    @Bean
    public Stripe stripeClient() {
        Stripe.apiKey = secretKey;
        return new Stripe();
    }
}
```

### Fail-Fast on Missing Secrets

Configure your application to fail immediately at startup if required secrets are absent, rather than starting with a broken configuration that fails on the first actual use:

```java
@Configuration
@Validated  // Triggers Bean Validation on the config properties
@ConfigurationProperties(prefix = "app.security")
public class SecurityProperties {

    @NotBlank(message = "JWT secret must be configured (app.security.jwt-secret)")
    private String jwtSecret;

    @NotBlank(message = "Database password must be configured")
    private String dbPassword;

    @Min(value = 32, message = "JWT secret must be at least 32 characters (256 bits)")
    // This validates the length, not just presence
    private int jwtSecretMinLength;
}
```

### `.env` Files for Local Development

For local development, teams often use a `.env` file to avoid setting environment variables manually on each machine. Tools like **dotenv-java** load these at development time:

```bash
# .env — NEVER commit this file to Git
DB_PASSWORD=localdevpassword
STRIPE_SECRET_KEY=sk_test_localdevelopmentkey
JWT_SECRET=localdevsecret_at_least_32_chars_long
```

```
# .gitignore — make absolutely sure .env is ignored
.env
.env.local
*.env
```

The `.env` file is for local development convenience only. In production, environment variables are set by the deployment system (Kubernetes Secrets, ECS task definitions, Helm chart values injected from a vault).

---

## HashiCorp Vault — Enterprise Secrets Management

While environment variables work for simple deployments, they have significant limitations at scale. They cannot enforce rotation schedules, they don't provide an audit trail of who accessed what secret and when, they can't issue dynamic short-lived credentials, and managing environment variables across dozens of microservices and multiple environments becomes operationally chaotic.

**HashiCorp Vault** is the industry-standard solution for enterprise secrets management. It is a dedicated secrets store that: encrypts all secrets at rest and in transit, provides a fine-grained ACL system controlling exactly which services can access which secrets, logs every access for auditing, supports automatic rotation of credentials, and can generate **dynamic secrets** — credentials that exist only for a specific purpose and expire automatically.

### Core Vault Concepts

**Secrets Engines** are Vault's data plugins. The **KV (Key-Value) engine** stores static secrets (like API keys). The **Database engine** is the most powerful for Java applications — it connects to your database and generates temporary, short-lived credentials on demand. Instead of your application knowing the database password, Vault creates a temporary username and password with a 1-hour TTL specifically for that application's deployment. When the TTL expires, those credentials are automatically revoked.

**Auth Methods** control how clients authenticate to Vault. In Kubernetes, the **Kubernetes auth method** is standard — your Pod's service account token is sent to Vault, which validates it with the Kubernetes API server and returns a Vault token with the appropriate policies. No static credentials needed to access Vault itself.

**Policies** are HCL documents that define what a given entity is allowed to do — which secret paths they can read, write, or delete.

### Spring Cloud Vault Integration

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-vault-config-databases</artifactId>
</dependency>
```

```yaml
# bootstrap.yml (Spring Cloud loads this before application.yml)
spring:
  cloud:
    vault:
      host: vault.internal.example.com
      port: 8200
      scheme: https
      # In Kubernetes, use the Kubernetes auth method
      authentication: KUBERNETES
      kubernetes:
        role: order-service            # Vault role mapped to your Kubernetes ServiceAccount
        kubernetes-path: kubernetes    # The path where the K8s auth method is mounted
      # Paths to read from Vault's KV engine:
      kv:
        enabled: true
        backend: secret                # The KV engine mount point
        application-name: order-service
        profiles:
          - production
      # Database credential rotation:
      database:
        enabled: true
        role: order-service-db-role    # Vault generates dynamic credentials for this role
        backend: database              # The database secrets engine mount
```

With this configuration, Spring Cloud Vault contacts Vault at startup, authenticates using the Kubernetes service account token, reads secrets from `secret/order-service/production`, generates short-lived database credentials from the `database` engine, and injects everything into Spring's property system. Your application code sees `${spring.datasource.password}` without ever knowing where it came from.

### Dynamic Database Credentials — The Gold Standard

The most powerful pattern Vault enables is **dynamic database credentials**. Instead of your application knowing a static database password:

```hcl
# Vault configuration (run by the ops team, not the application)
# Tell Vault how to connect to the database with a highly privileged account
vault write database/config/myapp-postgres \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@postgres-host:5432/mydb" \
    allowed_roles="order-service-role" \
    username="vault-admin" \
    password="vault-admin-password"

# Define what credentials Vault should generate for order-service
vault write database/roles/order-service-role \
    db_name=myapp-postgres \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

When `order-service` starts up, it asks Vault for database credentials. Vault creates a temporary PostgreSQL user (e.g., `v-order-service-abc123`) with a 1-hour TTL and returns the credentials. The application uses them. After 1 hour, Vault drops the user automatically. If the application is compromised and the credentials are stolen, they expire in at most 1 hour with no manual intervention needed. This is a fundamentally better security posture than a long-lived static password.

---

## AWS Secrets Manager

**AWS Secrets Manager** is AWS's managed secrets service, similar in purpose to HashiCorp Vault but fully managed — you do not need to run and maintain Vault infrastructure yourself. It integrates natively with AWS IAM for access control and AWS KMS for encryption at rest.

Key capabilities include automatic rotation (you define a rotation Lambda and Secrets Manager calls it on a schedule), versioning (you can access the previous version of a secret during rotation windows), and first-class integration with RDS — Secrets Manager can rotate your RDS database password and update the secret value automatically without any downtime.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>secretsmanager</artifactId>
</dependency>
<!-- For Spring Boot property integration: -->
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-secrets-manager</artifactId>
</dependency>
```

```java
// Direct SDK usage — fetching a secret programmatically
@Service
public class SecretsService {

    private final SecretsManagerClient secretsManagerClient;

    public SecretsService() {
        this.secretsManagerClient = SecretsManagerClient.builder()
            .region(Region.US_EAST_1)
            // Authentication uses the EC2/ECS/Lambda IAM role automatically —
            // no access key IDs or secret keys needed in the code
            .build();
    }

    // Retrieve a JSON secret and deserialize it
    public DatabaseCredentials getDatabaseCredentials() {
        GetSecretValueRequest request = GetSecretValueRequest.builder()
            .secretId("myapp/production/database")   // The secret's name/ARN
            // .versionStage("AWSPREVIOUS")          // Uncomment to get previous version
            .build();

        GetSecretValueResponse response = secretsManagerClient.getSecretValue(request);
        String secretJson = response.secretString();

        // The secret is typically stored as a JSON string:
        // {"username":"myapp","password":"Tr0ub4dor&3","host":"prod-db.example.com","port":5432}
        try {
            return objectMapper.readValue(secretJson, DatabaseCredentials.class);
        } catch (JsonProcessingException e) {
            throw new RuntimeException("Failed to parse database credentials from Secrets Manager", e);
        }
    }

    // For high-traffic services, cache the secret locally to avoid API rate limits
    // Refresh the cache periodically or on authentication failure
    @Cacheable(value = "secrets", key = "#secretName", unless = "#result == null")
    public String getSecret(String secretName) {
        return secretsManagerClient.getSecretValue(
            GetSecretValueRequest.builder().secretId(secretName).build()
        ).secretString();
    }
}
```

### Spring Cloud AWS Integration

With Spring Cloud AWS, secrets from Secrets Manager are injected directly into Spring Boot's property system, so your code needs zero changes:

```yaml
# application.yml
spring:
  config:
    # Import secrets at startup — Spring resolves ${db.password} from Secrets Manager
    import: "aws-secretsmanager:/myapp/production/db-credentials;/myapp/production/api-keys"

# The imported secrets are accessible as regular Spring properties:
# /myapp/production/db-credentials (JSON: {"db.password":"...", "db.username":"..."})
spring:
  datasource:
    username: ${db.username}
    password: ${db.password}
```

---

## AWS SSM Parameter Store — For Non-Secret Configuration Too

For values that are configuration (not truly secret), **AWS SSM Parameter Store** is a lighter-weight alternative to Secrets Manager. It stores both plain String parameters (no encryption, for non-sensitive config like environment names, feature flags, endpoint URLs) and **SecureString** parameters (encrypted with KMS, for sensitive values like passwords). It is significantly cheaper than Secrets Manager for high-volume parameter reads.

```yaml
# application.yml — mixing SSM Parameter Store and Secrets Manager
spring:
  config:
    import:
      - "aws-parameterstore:/myapp/production/"     # Non-secret config from SSM
      - "aws-secretsmanager:/myapp/production/creds"  # Actual secrets from Secrets Manager
```

---

## Kubernetes Secrets — A Pragmatic Baseline with Caveats

**Kubernetes Secrets** provide a basic mechanism for storing and injecting credentials into Pods. They are base64-encoded (not encrypted by default in etcd) and are mounted as environment variables or volume files in containers.

```yaml
# Create a Kubernetes Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: production
type: Opaque
stringData:                     # stringData auto-base64-encodes the values
  DB_PASSWORD: "Tr0ub4dor&3"
  DB_USERNAME: "myapp"
```

```yaml
# Inject the Secret into your Pod as environment variables
spec:
  containers:
    - name: myapp
      image: myapp:1.2.0
      env:
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: DB_PASSWORD
```

The critical caveat is that Kubernetes Secrets are **not encrypted at rest by default** — they are stored as base64-encoded data in etcd. Anyone with read access to etcd (or the right RBAC permissions in Kubernetes) can retrieve them. For production systems, you should either:

Enable **etcd encryption at rest** in your Kubernetes cluster (this encrypts all Secrets in etcd using AES-GCM). Use **Vault Agent** or **External Secrets Operator** to pull secrets from a proper vault and inject them as Kubernetes Secrets automatically. Use **Sealed Secrets** (Bitnami) — encrypted secrets that can be safely committed to Git and are decrypted only inside the cluster by the Sealed Secrets controller.

```yaml
# External Secrets Operator — pull from AWS Secrets Manager into a K8s Secret automatically
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h              # Re-sync from Secrets Manager every hour
  secretStoreRef:
    name: aws-secretsmanager       # Reference to the ClusterSecretStore
    kind: ClusterSecretStore
  target:
    name: db-credentials           # The Kubernetes Secret to create/update
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD       # Key in the Kubernetes Secret
      remoteRef:
        key: myapp/production/db   # Path in AWS Secrets Manager
        property: password         # JSON property within the secret
```

---

## Secret Rotation — Keeping a Moving Target

A static secret that never changes is a permanent liability — if it is ever compromised, the attacker has it indefinitely. **Secret rotation** is the practice of periodically changing secrets, limiting the window of exploitation if a secret is ever exposed.

The rotation workflow must be designed to avoid downtime: create the new credential, update the secret store with the new value, update the application to use the new credential (without restarting if possible), verify everything works, then revoke the old credential.

For database passwords with Vault dynamic secrets, rotation is automatic and continuous — every instance gets fresh credentials that expire hourly. For API keys, implement a **dual-key strategy**: during rotation, both the old and new API key are valid simultaneously for a short overlap window, allowing all clients to transition smoothly.

Spring Boot supports **live secret refresh** using Spring Cloud's `@RefreshScope` annotation and Actuator's `/actuator/refresh` endpoint. When a secret is rotated in Vault or Secrets Manager, you can trigger a refresh without restarting:

```java
@RefreshScope        // This bean is re-created when /actuator/refresh is called
@Service
public class PaymentService {

    @Value("${stripe.secret-key}")
    private String stripeSecretKey;   // Picks up the new value after refresh
}
```

---

## Best Practices Summary

The hierarchy of secrets management approaches, from worst to best, is: hardcoded in source code (never acceptable), committed to a config file (almost as bad), plaintext environment variables at rest (acceptable only for non-production), encrypted secrets in a managed service (Secrets Manager, Vault), and dynamic short-lived credentials generated on-demand (the gold standard).

Implement a **secrets scanning pre-commit hook** in your development workflow. Tools like `detect-secrets` or `gitleaks` can scan every commit for secret patterns before they enter the repository, stopping leaks at the source rather than detecting them after the fact.

Adopt the **principle of least privilege** for secrets too — each application and each deployment should only be able to access the secrets it genuinely needs. An `order-service` should not have access to `payment-service` secrets. Use IAM roles, Vault policies, and Kubernetes RBAC to enforce this granularity.

Maintain a **secrets inventory** — a documented list of every secret your application uses, where it is stored, when it was last rotated, and who is responsible for it. Without this, you cannot efficiently respond to a breach (which credentials do you rotate? which systems were potentially affected?).

Treat secret rotation as a normal operational activity, not an emergency measure. Automate it wherever possible. The applications that handle security breaches best are those that have practiced rotation so frequently that it is completely routine. A team that has never rotated their database password will fumble it under pressure; a team that rotates automatically every 24 hours will be calm.
