# 03 — Order Service Deep Dive

> **Location:** `src/orders/`
> **Language:** Java 21
> **Framework:** Spring Boot 3.5.5 (Spring MVC + Actuator)
> **Build Tool:** Maven (via `mvnw` wrapper)
> **Database:** PostgreSQL (in-cluster StatefulSet or external) **or** in-memory (H2)
> **Messaging:** RabbitMQ (in-cluster StatefulSet or external) **or** AWS SQS **or** in-memory
> **Container Port:** `8080`
> **Public Image:** `public.ecr.aws/aws-containers/retail-store-sample-orders`

---

## What This Service Does

The Orders service is the most complex service in the stack. It handles order creation and persistence, and it is the only service that integrates both a relational database **and** a message broker. When an order is placed, the service writes the order to PostgreSQL and publishes an order-created event to a messaging backend (RabbitMQ, AWS SQS, or in-memory). It exposes a full REST API documented via OpenAPI 3.0.

---

## Source Tree

```
src/orders/
├── OrdersApplication.java       # Spring Boot entry point
├── Dockerfile                   # Multi-stage: Amazon Linux 2023 build → slim runtime
├── docker-compose.yml           # Local dev stack: orders + PostgreSQL + RabbitMQ
├── pom.xml                      # Maven dependencies and build plugins
├── mvnw / mvnw.cmd              # Maven wrapper scripts
├── openapi.yml                  # OpenAPI 3.0 spec for the orders API
├── project.json                 # Nx project definition (build/test/push targets)
├── .dockerignore                # Excludes test/, docs, *.md from Docker build context
├── .gitignore                   # Ignores compiled target/ directory
├── ATTRIBUTION.md               # OSS license attribution
│
├── events/                      # Shared event schema definitions (order events)
│
├── src/main/
│   ├── java/com/amazon/sample/orders/
│   │   ├── OrdersApplication.java    # @SpringBootApplication entry point
│   │   ├── chaos/                    # Chaos engineering: controllable failure injection
│   │   ├── config/                   # Spring @Configuration: DB, messaging, Flyway wiring
│   │   ├── entities/                 # JPA-style JDBC entities: Order, OrderItem
│   │   ├── messaging/                # Messaging abstraction: in-memory, RabbitMQ, SQS impls
│   │   ├── metrics/                  # Micrometer custom metrics (order counters, timers)
│   │   ├── repositories/             # Spring Data JDBC repositories
│   │   ├── services/                 # OrderService interface + business logic
│   │   └── web/                      # Spring MVC REST controllers + DTOs
│   └── resources/
│       ├── application.yml           # Base Spring config: actuator, persistence, messaging defaults
│       ├── application-prod.yml      # Production profile overrides (disables debug logging)
│       └── db/                       # Flyway SQL migration scripts
│
├── src/test/                         # Integration tests using Testcontainers (PostgreSQL)
├── scripts/                          # Helper scripts
│
└── chart/                            # Helm chart — covered in full below
```

---

## Application Runtime

### Entry Point — `OrdersApplication.java`

Spring Boot auto-configuration wires all components based on env vars injected by the Helm ConfigMap. The bootstrapping flow is:

1. Spring reads `application.yml` and `application-prod.yml` (active via `SPRING_PROFILES_ACTIVE=prod` in Dockerfile).
2. `spring.autoconfigure.exclude` in `application.yml` disables both `RabbitAutoConfiguration` and `SqsAutoConfiguration` by default. The `config/` classes re-enable only the selected backend based on `retail.orders.messaging.provider`.
3. Flyway runs DB migrations from `resources/db/` against the configured PostgreSQL endpoint (or H2 in-memory for `provider=in-memory`).
4. `repositories/` Spring Data JDBC repositories are initialized.
5. Actuator endpoints are exposed: `health`, `info`, `metrics`, `prometheus`.
6. The Spring MVC dispatcher registers all `web/` controllers.
7. Server starts on `${port:8080}`.

### API Routes

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/orders` | List all orders |
| `POST` | `/orders` | Create a new order |
| `GET` | `/orders/{orderId}` | Get a specific order |
| `GET` | `/actuator/health` | Spring Boot health check (liveness) |
| `GET` | `/actuator/health/readiness` | Readiness probe (used by Kubernetes) |
| `GET` | `/actuator/prometheus` | Prometheus metrics scrape endpoint |

> **Note:** The Kubernetes readiness probe hits `/actuator/health/readiness`, not `/actuator/health`. The cart and orders services share this pattern — the generic `/actuator/health` path returns liveness state only.

### Spring Configuration — `application.yml`

```yaml
# src/orders/src/main/resources/application.yml
management:
  endpoints:
    web:
      exposure:
        include: info,health,metrics,prometheus

spring.autoconfigure.exclude:
  - org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration
  - io.awspring.cloud.autoconfigure.sqs.SqsAutoConfiguration

server:
  port: ${port:8080}

spring.flyway.baseline-on-migrate: true

retail:
  orders:
    persistence:
      provider: "in-memory"
      postgres:
        endpoint:
        dbname:
        username:
        password:
    messaging:
      provider: "in-memory"
      rabbitmq:
        addresses: ""
        username: ""
        password: ""
      sqs:
        topic: ""

otel.sdk.disabled: true
```

**Relaxed binding:** `RETAIL_ORDERS_PERSISTENCE_PROVIDER` env var maps to `retail.orders.persistence.provider`. `RETAIL_ORDERS_MESSAGING_PROVIDER` maps to `retail.orders.messaging.provider`. This is the same Spring Boot relaxed binding mechanism used in the cart service — uppercase underscore-separated env vars override dot-separated YAML keys at runtime.

`spring.autoconfigure.exclude` disables RabbitMQ and SQS auto-configuration globally. This prevents Spring from trying to connect to a broker that isn't configured. The `config/` classes then conditionally enable only the selected provider. `otel.sdk.disabled: true` disables OpenTelemetry tracing unless the OTEL Operator injects the SDK.

### Key Dependencies — `pom.xml`

| Dependency | Version | Purpose |
|------------|---------|--------|
| `spring-boot-starter-parent` | 3.5.5 | BOM for all Spring dependencies |
| `spring-boot-starter-web` | (managed) | Spring MVC HTTP server (Tomcat embedded) |
| `spring-boot-starter-actuator` | (managed) | Health, metrics, info endpoints |
| `spring-boot-starter-data-jdbc` | (managed) | Spring Data JDBC (lighter than JPA — no Hibernate) |
| `spring-boot-starter-amqp` | (managed) | RabbitMQ/AMQP messaging support |
| `spring-boot-starter-validation` | (managed) | Bean Validation (`@NotNull`, `@Valid` on request bodies) |
| `spring-boot-starter-log4j2` | (managed) | Log4j2 (replaces default Logback) |
| `micrometer-registry-prometheus` | (managed) | Prometheus metrics at `/actuator/prometheus` |
| `flyway-core` + `flyway-database-postgresql` | (managed) | Database schema migrations on startup |
| `postgresql` | (managed) | PostgreSQL JDBC driver |
| `h2` | (managed, runtime) | In-memory H2 DB for `provider=in-memory` |
| `spring-cloud-aws-starter-sqs` | 3.3.0 | AWS SQS messaging integration |
| `spring-cloud-aws-starter` | 3.3.0 | AWS Spring Cloud core (credential provider chain) |
| `mapstruct` | 1.6.3 | Compile-time DTO ↔ entity mapper (replaces manual conversion) |
| `lombok` | (managed) | Code generation: `@Data`, `@Builder`, `@Slf4j` |
| `springdoc-openapi-starter-webmvc-ui` | 2.8.9 | OpenAPI docs + Swagger UI |
| `opentelemetry-spring-boot-starter` | 2.17.0 | OTEL auto-instrumentation |
| `opentelemetry-aws-sdk-2.2-autoconfigure` | 2.17.1-alpha | OTEL instrumentation for AWS SDK calls |
| `testcontainers` + `postgresql` | (managed) | Integration tests with real PostgreSQL container |

**Key design choices:**
- **Spring Data JDBC over JPA:** No Hibernate, no lazy loading, no ORM magic. Explicit SQL via repositories. This keeps the service lightweight and the SQL predictable.
- **Flyway:** Schema migrations run automatically on startup from `resources/db/`. `spring.flyway.baseline-on-migrate: true` prevents failures when Flyway encounters a non-empty database.
- **H2:** The in-memory mode uses H2 (embedded relational DB) rather than a NoSQL store — unlike the cart service which uses DynamoDB Local. This means the in-memory mode fully exercises the SQL code path, just without persistence across restarts.
- **AWS SQS:** `spring-cloud-aws-starter-sqs` adds SQS as a third messaging option alongside RabbitMQ and in-memory. On EKS, SQS is the recommended production messaging backend — no broker to manage, natively serverless.

### Dockerfile — Build Strategy

Two-stage build on Amazon Linux 2023:

| Stage | Base | What Happens |
|-------|------|-------------|
| `build-env` | `amazonlinux:2023` | Installs Maven + Java 21 Corretto, downloads dependencies offline (`mvn dependency:go-offline`), builds `target/orders-0.0.1-SNAPSHOT.jar` → renames to `/app.jar` |
| Runtime | `amazonlinux:2023` (fresh) | Installs Java 21 Corretto + `curl-full` (for AMQP telnet checks), creates `appuser` (UID 1000), sets `SPRING_PROFILES_ACTIVE=prod`, copies `/app.jar` |

**`curl-full` swap:** The runtime stage swaps `libcurl-minimal` + `curl-minimal` for `libcurl-full` + `curl-full`. This is needed because RabbitMQ health checks use the `telnet://` scheme (`curl telnet://host:5672`), which is only available in the full curl build. Amazon Linux 2023 ships the minimal curl variant by default.

The `ENTRYPOINT` is `["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]` — the shell wrapper is required so that `$JAVA_OPTS` (injected by the Helm Deployment as `-XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom`) is evaluated at container start time rather than being treated as a literal string.

---

## Helm Chart

### Chart Files

```
src/orders/chart/
├── Chart.yaml                       # apiVersion: v2, name: orders, type: application
├── .helmignore                      # Excludes *.md, tests/ from packaged chart
├── values.yaml                      # Master defaults — every knob the chart exposes
│
└── templates/
    ├── _helpers.tpl                 # Named templates: fullname, labels, endpoint helpers
    ├── deployment.yaml              # Orders app Deployment
    ├── service.yaml                 # ClusterIP service on port 80 → container 8080
    ├── serviceaccount.yaml          # Optional ServiceAccount
    ├── configmap.yml                # Persistence + messaging provider env vars
    ├── secret-db.yaml               # DB credentials Secret (conditional)
    ├── rabbitmq-secret.yaml         # RabbitMQ credentials Secret (conditional)
    ├── postgresql-statefulset.yaml  # PostgreSQL StatefulSet (conditional)
    ├── postgresql-service.yaml      # PostgreSQL ClusterIP service (conditional)
    ├── rabbitmq-statefulset.yaml    # RabbitMQ StatefulSet (conditional)
    ├── rabbitmq-service.yaml        # RabbitMQ ClusterIP + management services (conditional)
    ├── security-group.yaml          # AWS SecurityGroupPolicy CRD (conditional, EKS only)
    ├── hpa.yaml                     # HorizontalPodAutoscaler (conditional)
    ├── pdb.yaml                     # PodDisruptionBudget (conditional)
    └── tests/                       # helm test hook
```

**Key differences from cart:** Orders has two Secrets (`secret-db.yaml` for PostgreSQL credentials, `rabbitmq-secret.yaml` for RabbitMQ), two StatefulSets (PostgreSQL, RabbitMQ — both using StatefulSet because they need stable identity for cluster formation and optional PVC attachment), and a `security-group.yaml` for AWS VPC security group assignment on EKS.

---

## `values.yaml` — Annotated Deep Dive

### Section 1 — Pod & Image

```yaml
replicaCount: 1

image:
  repository: public.ecr.aws/aws-containers/retail-store-sample-orders
  pullPolicy: IfNotPresent
  tag:
```

`replicaCount` is ignored when `autoscaling.enabled: true`. `image.tag` falls back to `.Chart.Version` when blank.

---

### Section 2 — ServiceAccount & Pod Settings

```yaml
serviceAccount:
  create: true
  annotations: {}
  name: ""

podSecurityContext:
  fsGroup: 1000

securityContext:
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
```

`serviceAccount.annotations` is the **IRSA hook** for production EKS when using AWS SQS:

```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
```

The `spring-cloud-aws-starter` dependency on the classpath uses the AWS credential provider chain. On EKS with this annotation, the EKS Pod Identity Webhook injects a projected token that is automatically exchanged for IAM credentials via STS — no hardcoded keys required.

---

### Section 3 — Metrics

```yaml
metrics:
  enabled: true
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"
```

Scrape path is `/actuator/prometheus` — same as cart, via Micrometer's Prometheus registry over Spring Actuator.

---

### Section 4 — `app.persistence` (DB Toggle)

```yaml
app:
  persistence:
    provider: 'in-memory'      # "in-memory" | "postgres"
    endpoint: ''
    database: 'orders'

    secret:
      create: true
      name: orders-db
      username: orders
      password: ""
```

`app.persistence.provider` is the master persistence switch:

| Value | What Happens |
|-------|-------------|
| `in-memory` | H2 embedded DB; Flyway still runs migrations against H2; no PostgreSQL pod or Secret created |
| `postgres` | ConfigMap sets `RETAIL_ORDERS_PERSISTENCE_PROVIDER=postgres`, `ENDPOINT`, `NAME`; `secret-db.yaml` creates a Secret with username/password; Deployment mounts the Secret via `secretRef` |

`app.persistence.endpoint` accepts either the auto-generated in-cluster PostgreSQL service name (left blank when `postgresql.create: true` — the `_helpers.tpl` resolves it) or an explicit external endpoint (e.g., `mydb.cluster.us-east-1.rds.amazonaws.com:5432`).

`app.persistence.secret.create: true` creates the `orders-db` Kubernetes Secret from `username` and `password`. Set `create: false` and pre-create the Secret when using an external DB with credentials managed outside Helm (e.g., via AWS Secrets Manager sync).

---

### Section 5 — `app.messaging` (Messaging Toggle)

```yaml
app:
  messaging:
    provider: 'in-memory'     # "in-memory" | "rabbitmq" | "sqs"

    rabbitmq:
      addresses: []

      secret:
        create: true
        name: orders-rabbitmq
        username: ""
        password: ""
```

`app.messaging.provider` is the master messaging switch:

| Value | What Happens |
|-------|-------------|
| `in-memory` | Events are published to an in-memory queue (lost on restart); no broker pod or Secret created |
| `rabbitmq` | ConfigMap sets `RETAIL_ORDERS_MESSAGING_PROVIDER=rabbitmq` + `ADDRESSES`; `rabbitmq-secret.yaml` creates a Secret with username/password; Deployment mounts the Secret via `secretRef` |
| `sqs` | ConfigMap sets `RETAIL_ORDERS_MESSAGING_PROVIDER=sqs`; credentials come from IRSA (no Secret needed) |

`app.messaging.rabbitmq.addresses` is a list that is joined and set as `RETAIL_ORDERS_MESSAGING_RABBITMQ_ADDRESSES`. When `rabbitmq.create: true`, the `_helpers.tpl` auto-generates the in-cluster address (`<release>-orders-rabbitmq:5672`). For external RabbitMQ, supply explicit addresses.

---

### Section 6 — `postgresql` (In-Cluster PostgreSQL)

```yaml
postgresql:
  create: false

  image:
    repository: public.ecr.aws/docker/library/postgres
    tag: "16.1"

  service:
    type: ClusterIP
    port: 5432

  persistentVolume:
    enabled: false
    size: 10Gi
    # storageClass: gp2
```

`postgresql.create` gates the StatefulSet and Service for in-cluster PostgreSQL. Unlike the cart's DynamoDB Local (a plain Deployment), PostgreSQL runs as a **StatefulSet** — this gives it a stable pod identity (`orders-postgresql-0`) and is required for PVC attachment when `persistentVolume.enabled: true`.

`persistentVolume.enabled: false` means all PostgreSQL data is **ephemeral** — lost on pod restart. For non-production scenarios where data persistence between restarts is needed, set `persistentVolume.enabled: true` and provide a `storageClass`. For production, use an external managed PostgreSQL (e.g., AWS RDS) instead of the in-cluster StatefulSet.

---

### Section 7 — `rabbitmq` (In-Cluster RabbitMQ)

```yaml
rabbitmq:
  create: false

  image:
    repository: public.ecr.aws/docker/library/rabbitmq
    tag: "3-management"

  service:
    type: ClusterIP
    amqp:
      port: 5672
    http:
      port: 15672

  persistentVolume:
    enabled: false
    size: 10Gi
    # storageClass: gp2
```

`rabbitmq.create` gates the StatefulSet and both Services (AMQP on 5672, HTTP management UI on 15672). RabbitMQ also runs as a **StatefulSet** for the same reasons as PostgreSQL. The `3-management` image tag includes the management UI at port 15672 — accessible for debugging via `kubectl port-forward`.

`persistentVolume.enabled: false` means the RabbitMQ message queue is **ephemeral**. Any unprocessed messages are lost when the pod restarts. For production, use external managed RabbitMQ or switch to AWS SQS.

---

### Section 8 — `securityGroups` (EKS only)

```yaml
securityGroups:
  create: false
  securityGroupIds: []
```

When `create: true`, renders a `SecurityGroupPolicy` CRD resource (from the AWS VPC CNI plugin). This assigns specific VPC security groups to the orders pod — typically used to grant access to an RDS PostgreSQL instance or RabbitMQ that is protected by a VPC security group rule. This is an EKS-only feature and requires the VPC CNI security groups for pods feature to be enabled on the cluster.

---

### Section 9 — Autoscaling, Observability & Disruption Budget

```yaml
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

opentelemetry:
  enabled: false
  instrumentation: ""

podDisruptionBudget:
  enabled: false
  minAvailable: 2
  maxUnavailable: 1
```

Same patterns as cart and catalog. When `autoscaling.enabled: true`, `replicas` is dropped from the Deployment spec and HPA takes control. `opentelemetry.instrumentation` sets the `instrumentation.opentelemetry.io/inject-sdk` pod annotation for the OTEL Operator.

---

## ConfigMap Template — Full Logic

The `configmap.yml` template controls which env vars reach the Spring application:

```yaml
data:
  RETAIL_ORDERS_MESSAGING_PROVIDER: {{ .Values.app.messaging.provider }}
  {{- if (eq "rabbitmq" .Values.app.messaging.provider) }}
  RETAIL_ORDERS_MESSAGING_RABBITMQ_ADDRESSES: {{ include "orders.rabbitmq.addresses" . }}
  {{- end }}
  RETAIL_ORDERS_PERSISTENCE_PROVIDER: {{ .Values.app.persistence.provider }}
  {{- if (eq "postgres" .Values.app.persistence.provider) }}
  RETAIL_ORDERS_PERSISTENCE_ENDPOINT: {{ include "orders.postgresql.endpoint" . }}
  RETAIL_ORDERS_PERSISTENCE_NAME: {{ .Values.app.persistence.database }}
  {{- end }}
```

| Scenario | `persistence.provider` | `messaging.provider` | ConfigMap contains |
|----------|----------------------|---------------------|-------------------|
| Fully in-memory | `in-memory` | `in-memory` | `PERSISTENCE_PROVIDER`, `MESSAGING_PROVIDER` only |
| In-cluster PostgreSQL + in-memory messaging | `postgres` | `in-memory` | + `PERSISTENCE_ENDPOINT`, `PERSISTENCE_NAME` |
| In-cluster PostgreSQL + RabbitMQ | `postgres` | `rabbitmq` | + all of above + `MESSAGING_RABBITMQ_ADDRESSES` |
| External PostgreSQL + SQS | `postgres` | `sqs` | + `PERSISTENCE_ENDPOINT`, `PERSISTENCE_NAME` (no RabbitMQ addr) |

**Secrets are mounted separately** via `envFrom.secretRef` in the Deployment template — not in the ConfigMap. The `secret-db.yaml` Secret contains `RETAIL_ORDERS_PERSISTENCE_USERNAME` and `RETAIL_ORDERS_PERSISTENCE_PASSWORD`; the `rabbitmq-secret.yaml` Secret contains `RETAIL_ORDERS_MESSAGING_RABBITMQ_USERNAME` and `RETAIL_ORDERS_MESSAGING_RABBITMQ_PASSWORD`. Both Secrets are only mounted when their respective `provider` is `postgres` or `rabbitmq` respectively.

---

## Deployment Template — Key Behaviours

- **`JAVA_OPTS`** is hardcoded as a literal env var: `-XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom`. `MaxRAMPercentage=75.0` caps the JVM heap at 75% of the `512Mi` memory limit → `~384Mi` heap, `~128Mi` for JVM overhead and OS.
- **`envFrom.secretRef`** mounts are conditional: `secret-db.yaml` is only mounted when `persistence.provider == "postgres"`, and `rabbitmq-secret.yaml` only when `messaging.provider == "rabbitmq"`. In `in-memory` or `sqs` mode, no Secret is mounted.
- **Readiness probe:** `/actuator/health/readiness` with `initialDelaySeconds: 10` — accounts for Spring Boot startup + Flyway migration time.
- **`/tmp` volume:** `emptyDir medium: Memory` — required because `readOnlyRootFilesystem: true` prevents JVM from writing temp files to the container filesystem.
- **Rolling update strategy:** `maxUnavailable: 1` — always keeps at least `replicaCount - 1` pods running during deploys.

---

## End-to-End: What Happens on `helm install`

1. Helm merges `values.yaml` + any `-f` override files.
2. Templates are rendered:
   - `configmap.yml` → always rendered
   - `secret-db.yaml` → only if `app.persistence.secret.create: true`
   - `rabbitmq-secret.yaml` → only if `app.messaging.rabbitmq.secret.create: true`
   - `postgresql-statefulset.yaml` + `postgresql-service.yaml` → only if `postgresql.create: true`
   - `rabbitmq-statefulset.yaml` + `rabbitmq-service.yaml` → only if `rabbitmq.create: true`
   - `security-group.yaml` → only if `securityGroups.create: true`
   - `hpa.yaml` → only if `autoscaling.enabled: true`
   - `pdb.yaml` → only if `podDisruptionBudget.enabled: true`
3. PostgreSQL StatefulSet starts (if applicable); Flyway runs migrations on first connect.
4. RabbitMQ StatefulSet starts (if applicable) and becomes ready.
5. Orders Deployment starts; Spring Boot reads provider env vars from ConfigMap + credentials from Secrets.
6. `config/` classes activate the selected persistence and messaging beans.
7. `/actuator/health/readiness` returns `UP` → Deployment marks pod as ready.
