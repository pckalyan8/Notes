# Phase 15.2 — Kubernetes for Java

## What is Kubernetes and Why Do You Need It?

Once you have a Dockerized Java application, you face a new set of problems: How do you run multiple copies of it for high availability? How do you roll out a new version without downtime? What happens when a container crashes — who restarts it? How do containers in different pods find each other?

**Kubernetes (K8s)** is a container orchestration platform that answers all of these questions. It provides automatic scheduling, self-healing, scaling, service discovery, and rolling deployments for containerized workloads. Think of Docker as the "shipping container standard" and Kubernetes as "the port that manages thousands of those containers."

---

## Core Kubernetes Objects

### Pod

A **Pod** is the smallest deployable unit in Kubernetes. It wraps one or more containers that share a network namespace and storage. For Java applications, a Pod almost always contains exactly one container — your Spring Boot app. However, a Pod might also contain a "sidecar" container (e.g., a log forwarder or Envoy proxy).

Pods are **ephemeral** — they are not meant to be long-lived. If a Pod dies, it is gone. This is why you never create Pods directly in production; you always manage them through higher-level objects like Deployments.

### Deployment

A **Deployment** is a controller that ensures a specified number of Pod replicas are always running. If a Pod crashes, the Deployment controller immediately creates a replacement. It also manages rolling updates.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
spec:
  replicas: 3              # Run 3 identical copies of the app
  selector:
    matchLabels:
      app: myapp           # The Deployment manages Pods with this label
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: registry.example.com/myorg/myapp:1.2.0
          ports:
            - containerPort: 8080
          
          # Resource requests and limits (discussed in detail below)
          resources:
            requests:
              memory: "256Mi"    # Minimum memory guaranteed
              cpu: "250m"        # 250 millicores = 0.25 CPU cores
            limits:
              memory: "512Mi"    # Maximum memory allowed
              cpu: "500m"        # Maximum CPU allowed
          
          # Environment variables (injected into the container)
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret    # Reference a Kubernetes Secret
                  key: password
          
          # Liveness and readiness probes (explained below)
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60   # Give the JVM time to start
            periodSeconds: 15
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
```

### Service

Pods get a new IP address every time they restart. A **Service** provides a stable virtual IP (ClusterIP) and DNS name that routes traffic to the currently-running Pods matching its label selector. This is how other components find your app inside the cluster.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp             # Routes to all Pods with this label
  ports:
    - protocol: TCP
      port: 80             # Port the service listens on
      targetPort: 8080     # Port on the Pod (your Spring Boot port)
  type: ClusterIP          # Only accessible inside the cluster
  # Use LoadBalancer to expose externally (cloud provider creates an LB)
  # Use NodePort for bare-metal clusters
```

### ConfigMap and Secret

**ConfigMaps** store non-sensitive configuration (application properties, feature flags). **Secrets** store sensitive data (passwords, API keys, certificates) in base64-encoded form (note: Secrets are NOT encrypted by default — use an external secrets manager for true security).

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  application.properties: |
    server.port=8080
    spring.application.name=myapp
    logging.level.root=INFO

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Values must be base64 encoded: echo -n "mypassword" | base64
  password: bXlwYXNzd29yZA==
```

To mount a ConfigMap as a file in the container:

```yaml
# In the Pod spec, under volumes:
volumes:
  - name: config-volume
    configMap:
      name: myapp-config

# Under containers[].volumeMounts:
volumeMounts:
  - name: config-volume
    mountPath: /app/config     # Spring Boot auto-loads from this path
```

---

## Liveness and Readiness Probes — Critical for Java Apps

This is one of the most important concepts for Java developers deploying to Kubernetes, because getting it wrong causes production incidents.

**Liveness Probe** — Kubernetes asks "is this container still alive?" If it fails, Kubernetes **kills and restarts** the container. Use this to detect infinite loops or deadlocks. For Spring Boot, map this to `/actuator/health/liveness`.

**Readiness Probe** — Kubernetes asks "is this container ready to receive traffic?" If it fails, Kubernetes **removes the Pod from the Service endpoints** (no new requests are sent to it) but does NOT restart it. Use this during startup (JVM warmup, DB connection establishment) and for temporary unavailability. Map this to `/actuator/health/readiness`.

The distinction is crucial: if your Java app is doing a long startup (warming JIT, migrating the database via Flyway, etc.), the liveness probe should NOT fire during this time. Use `initialDelaySeconds` or, better, use **Spring Boot Actuator's built-in probe groups**.

```yaml
# application.properties — Spring Boot Actuator probe configuration
management.health.livenessState.enabled=true
management.health.readinessState.enabled=true
management.endpoint.health.probes.enabled=true
# Expose health, info, metrics endpoints
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```

Spring Boot automatically manages its own liveness and readiness state. During startup, `readinessState` is `REFUSING_TRAFFIC`, which means Kubernetes won't send requests until the app signals it's ready.

**Startup Probe** — A third probe type for slow-starting containers. While it is active, liveness and readiness probes are paused. Once the startup probe succeeds, normal liveness/readiness probing begins. This is ideal for Spring Boot apps with large dependency graphs.

```yaml
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  failureThreshold: 30   # Allow 30 * periodSeconds = 5 minutes for startup
  periodSeconds: 10
```

---

## Resource Limits and Requests

Every Java container in Kubernetes should have both **requests** and **limits** defined.

**Requests** are what Kubernetes uses for scheduling. If you request 256Mi of memory, Kubernetes will only schedule your Pod on a node that has at least 256Mi available. Requests should reflect the app's **normal** operating footprint.

**Limits** are the hard ceiling. If your container tries to exceed its memory limit, the Linux OOM Killer terminates it with a `137` exit code (OOMKilled). If it exceeds the CPU limit, it is throttled (slowed down, not killed).

The key insight for Java: the JVM's heap (`-Xmx` or `MaxRAMPercentage`) must fit within the container's memory **limit**, not just the request. Set the heap to about 50–60% of the limit to leave room for Metaspace, thread stacks, off-heap buffers, and GC overhead.

```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"       # JVM MaxRAMPercentage=75 => ~768MB heap
    cpu: "1000m"        # 1 full CPU core
```

---

## Rolling Updates — Zero-Downtime Deployment

By default, Kubernetes uses a **RollingUpdate** strategy. When you update the image tag in your Deployment, Kubernetes gradually replaces old Pods with new ones, ensuring some Pods are always serving traffic.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # Never have fewer Pods than desired (zero downtime)
      maxSurge: 1          # Allow 1 extra Pod during the rollout
```

For zero-downtime rolling updates to work properly with Java/Spring Boot, your app must:
1. Handle requests in-flight gracefully during shutdown (graceful shutdown).
2. Signal readiness accurately so Kubernetes knows when new Pods are ready.

```yaml
# application.properties
# Enable graceful shutdown — wait up to 30s for in-flight requests to complete
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

Kubernetes also needs a `preStop` hook to delay sending `SIGTERM` until the Service endpoints are updated:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "10"]   # Give the load balancer time to drain connections
```

---

## Horizontal Pod Autoscaler (HPA)

The HPA automatically scales the number of Pod replicas based on CPU utilization, memory utilization, or custom metrics (e.g., messages in a Kafka queue).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale up when CPU > 70%
```

For HPA to work, your Pods must have CPU **requests** defined (without them, Kubernetes cannot calculate utilization percentage).

---

## Helm Charts for Java Applications

**Helm** is the package manager for Kubernetes. Instead of managing a directory of raw YAML files, you define a **chart** — a templated, parameterized Kubernetes application package. The same chart can be deployed to dev, staging, and production with different `values.yaml` files.

```
myapp-chart/
├── Chart.yaml           # Chart metadata
├── values.yaml          # Default values
└── templates/
    ├── deployment.yaml  # Templated deployment
    ├── service.yaml
    ├── configmap.yaml
    └── hpa.yaml
```

```yaml
# values.yaml — Override per environment
image:
  repository: registry.example.com/myorg/myapp
  tag: "1.2.0"

replicaCount: 3

resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

env:
  SPRING_PROFILES_ACTIVE: "production"
```

```yaml
# templates/deployment.yaml — Helm template syntax uses {{ }} for interpolation
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-deployment
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

```bash
# Install the chart
helm install myapp ./myapp-chart -f values-production.yaml

# Upgrade (rolling update)
helm upgrade myapp ./myapp-chart --set image.tag=1.3.0

# Rollback
helm rollback myapp 1
```

---

## Best Practices Summary

Always define both resource requests and limits; never let a Java Pod run without memory limits. Use Spring Boot Actuator's dedicated liveness and readiness endpoints — map them correctly in your probe configuration and do not reuse the same `/actuator/health` endpoint for both. Enable graceful shutdown in Spring Boot and add a `preStop` sleep hook to prevent dropped connections during rolling updates. Use Secrets for sensitive data but consider integrating with HashiCorp Vault or AWS Secrets Manager for encrypted secret storage rather than relying on base64-only Kubernetes Secrets. Use Helm to parameterize your manifests so the same chart deploys consistently across environments. Set `initialDelaySeconds` generously for Java — the JVM takes time to start, especially with larger Spring Boot apps.
