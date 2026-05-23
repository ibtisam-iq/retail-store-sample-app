# 02 — Cart Service Deep Dive

> **Location:** `src/cart/`
> **Language:** Java 21
> **Framework:** Spring Boot 3.5.5 (Spring MVC + Actuator)
> **Build Tool:** Maven (via `mvnw` wrapper)
> **Database:** DynamoDB Local (in-cluster) **or** AWS DynamoDB (external) **or** in-memory (no DB)
> **Container Port:** `8080`
> **Public Image:** `public.ecr.aws/aws-containers/retail-store-sample-cart`

---

## What This Service Does

The Cart service manages shopping cart state for individual users. Every time a user adds, removes, or updates an item in their cart, this service is called. It is the only service in the stack that uses DynamoDB as its persistence backend — a deliberate design choice that makes it easy to swap between a lightweight in-cluster DynamoDB Local container (for dev/test) and real AWS DynamoDB (for production) without changing any application code.

---

## Source Tree

```
src/cart/
├── CartApplication.java         # Spring Boot entry point
├── Dockerfile                   # Multi-stage: Amazon Linux 2023 build → slim runtime
├── docker-compose.yml           # Local dev stack: cart + DynamoDB Local
├── pom.xml                      # Maven dependencies and build plugins
├── mvnw / mvnw.cmd              # Maven wrapper scripts (no local Maven install needed)
├── openapi.yml                  # OpenAPI 3.0 spec for the cart API
├── project.json                 # Nx project definition (build/test/push targets)
├── .dockerignore                # Excludes test/, docs, *.md from Docker build context
├── .gitignore                   # Ignores compiled target/ directory
├── ATTRIBUTION.md               # OSS license attribution (~1.9MB)
│
├── src/main/
│   ├── java/com/amazon/sample/carts/
│   │   ├── CartApplication.java     # @SpringBootApplication entry point
│   │   ├── action/                  # Command-style action classes (add/remove/update cart items)
│   │   ├── chaos/                   # Chaos engineering: controllable failure injection
│   │   ├── config/                  # Spring @Configuration classes: DynamoDB client, persistence wiring
│   │   ├── diagnostics/             # Topology/health diagnostic endpoints
│   │   ├── repositories/            # Repository layer: in-memory impl + DynamoDB impl
│   │   ├── services/                # CartService interface + business logic
│   │   └── web/                     # Spring MVC REST controllers
│   └── resources/
│       ├── application.yml          # Base Spring config: server port, persistence defaults
│       └── application-prod.yml     # Production profile overrides (disables debug logging)
│
├── src/test/                        # Integration tests using Testcontainers (DynamoDB Local)
├── scripts/                         # Helper scripts (seed data, local setup)
│
└── chart/                           # Helm chart — covered in full below
```

---

## Application Runtime

### Entry Point — `CartApplication.java`

`CartApplication.java` is the standard `@SpringBootApplication` class. Spring Boot auto-configuration handles all wiring based on which beans are present on the classpath and which Spring profiles are active. The key bootstrapping flow is:

1. Spring reads `application.yml` and `application-prod.yml` (active in container via `SPRING_PROFILES_ACTIVE=prod` in Dockerfile).
2. `config/` package `@Configuration` classes detect `retail.cart.persistence.provider` and conditionally create either the in-memory repository bean or the DynamoDB client + enhanced client beans.
3. Actuator endpoints are exposed: `health`, `info`, `metrics`, `prometheus` (configured in `application.yml`).
4. The Spring MVC dispatcher registers all `web/` controllers.
5. The chaos controller registers endpoints for failure simulation.
6. Server starts on `${port:8080}` (env var `PORT` overrides the default).

### API Routes

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/carts/{customerId}` | Get cart for a customer (creates if absent) |
| `DELETE` | `/carts/{customerId}` | Delete entire cart |
| `GET` | `/carts/{customerId}/items` | List all items in a cart |
| `POST` | `/carts/{customerId}/items` | Add item to cart |
| `DELETE` | `/carts/{customerId}/items/{itemId}` | Remove specific item |
| `PATCH` | `/carts/{customerId}/items/{itemId}` | Update item quantity |
| `GET` | `/actuator/health` | Spring Boot health check (liveness) |
| `GET` | `/actuator/health/readiness` | Readiness probe (used by Kubernetes) |
| `GET` | `/actuator/prometheus` | Prometheus metrics scrape endpoint |
| `GET` | `/topology` | Returns persistence provider + endpoint as JSON |
| `POST` | `/chaos/...` | Trigger simulated failures |

> **Note:** The readiness probe path is `/actuator/health/readiness`, not `/health`. This matters when configuring Kubernetes probes or load balancer health checks — the generic `/actuator/health` returns liveness state only.

### Spring Configuration — `application.yml`

Unlike catalog (which reads env vars directly), the cart service uses Spring Boot's `application.yml` as the configuration layer. Environment variables map to Spring properties via Spring's relaxed binding rules.

```yaml
# src/cart/src/main/resources/application.yml
management:
  endpoints:
    web:
      exposure:
        include: info,health,metrics,prometheus

server:
  port: ${port:8080}

retail:
  cart:
    persistence:
      provider: "in-memory"
      dynamodb:
        endpoint:
        create-table: false
        table-name: Items

otel.sdk.disabled: true
```

**Relaxed binding:** Spring maps env var `RETAIL_CART_PERSISTENCE_PROVIDER` → `retail.cart.persistence.provider`. This is how the Helm ConfigMap's env vars (`RETAIL_CART_PERSISTENCE_PROVIDER`, `RETAIL_CART_PERSISTENCE_DYNAMODB_TABLE_NAME`, etc.) override the `application.yml` defaults at runtime without modifying the file.

The `otel.sdk.disabled: true` default disables the OpenTelemetry SDK unless overridden. Tracing is only activated when the OTEL Operator injects the SDK (via `opentelemetry.instrumentation` in Helm values).

### Key Dependencies — `pom.xml`

| Dependency | Version | Purpose |
|------------|---------|---------|
| `spring-boot-starter-parent` | 3.5.5 | BOM for all Spring dependencies |
| `spring-boot-starter-web` | (managed) | Spring MVC HTTP server (Tomcat embedded) |
| `spring-boot-starter-actuator` | (managed) | Health, metrics, info endpoints |
| `spring-boot-starter-log4j2` | (managed) | Log4j2 logging (replaces default Logback) |
| `micrometer-registry-prometheus` | (managed) | Prometheus metrics export via `/actuator/prometheus` |
| `software.amazon.awssdk:dynamodb` | 2.32.13 | AWS DynamoDB low-level client |
| `software.amazon.awssdk:dynamodb-enhanced` | 2.32.13 | High-level DynamoDB mapper (table → Java object) |
| `software.amazon.awssdk:sts` | 2.32.13 | AWS STS for IAM role assumption (IRSA on EKS) |
| `lombok` | 1.18.40 | Code generation: `@Data`, `@Builder`, `@Slf4j` |
| `springdoc-openapi-starter-webmvc-ui` | 2.8.9 | Auto-generates OpenAPI docs + Swagger UI |
| `opentelemetry-spring-boot-starter` | 2.17.0 | OTEL auto-instrumentation for Spring |
| `opentelemetry-aws-sdk-2.2-autoconfigure` | 2.17.1-alpha | OTEL instrumentation for AWS SDK calls |
| `testcontainers` + `junit-jupiter` | 1.21.3 | Integration tests spin up real DynamoDB Local container |

**Important:** The AWS SDK (`dynamodb`, `sts`) is always on the classpath. When `provider=in-memory`, the DynamoDB beans are simply not instantiated by Spring. When `provider=dynamodb`, the `config/` classes create a `DynamoDbClient` pointed at either the local endpoint or real AWS. The `sts` dependency enables IRSA (IAM Roles for Service Accounts) on EKS — the SDK automatically exchanges the service account token for AWS credentials via STS, requiring no hardcoded AWS keys.

### Dockerfile — Build Strategy

Two-stage build on Amazon Linux 2023:

| Stage | Base | What Happens |
|-------|------|-------------|
| `build-env` | `amazonlinux:2023` | Installs Maven + Java 21 Corretto (headless), downloads dependencies offline first (`mvn dependency:go-offline`), then builds `target/carts-0.0.1-SNAPSHOT.jar` → renames to `/app.jar` |
| Runtime | `amazonlinux:2023` (fresh) | Installs Java 21 Corretto runtime only (no Maven), creates `appuser` (UID 1000), sets `SPRING_PROFILES_ACTIVE=prod`, copies `/app.jar` |

The `mvn dependency:go-offline` step is a build optimization — it downloads all Maven dependencies into the Docker layer cache before copying source code. This means source changes don't invalidate the dependency layer, making rebuilds significantly faster.

The `ENTRYPOINT` is `sh -c "java $JAVA_OPTS -jar /app/app.jar"` — the shell wrapper is required so that the `$JAVA_OPTS` env var (injected by the Helm Deployment template as `-XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom`) is evaluated at container start time.

### `docker-compose.yml` — Local Dev Stack

Runs two containers:

| Container | Image | Notes |
|-----------|-------|-------|
| `cart` | Built from `./Dockerfile` | `RETAIL_CART_PERSISTENCE_PROVIDER=dynamodb`, `RETAIL_CART_PERSISTENCE_DYNAMODB_ENDPOINT=http://carts-db:8000`, `RETAIL_CART_PERSISTENCE_DYNAMODB_CREATE_TABLE=true`, dummy AWS credentials, port `8082:8080` |
| `carts-db` | `amazon/dynamodb-local:1.20.0` | No auth, in-memory only, port 8000 |

The compose file sets `RETAIL_CART_PERSISTENCE_DYNAMODB_CREATE_TABLE=true` and dummy `AWS_ACCESS_KEY_ID=key` / `AWS_SECRET_ACCESS_KEY=dummy`. DynamoDB Local does not validate AWS credentials — any non-empty string works. This mirrors exactly what the Helm chart does when `dynamodb.create: true` in the ConfigMap template.

The cart container uses `read_only: true` with a `tmpfs` at `/tmp` — this is the same security posture as the Kubernetes deployment (`readOnlyRootFilesystem: true` + `emptyDir medium: Memory`).

### `openapi.yml`

A full OpenAPI 3.0 spec documenting all cart endpoints — `Cart` and `Item` schemas, request/response bodies, HTTP status codes. This is also what the `springdoc-openapi-maven-plugin` in `pom.xml` uses to auto-generate the spec during the Maven build lifecycle.

---

## Helm Chart

### Chart Files

```
src/cart/chart/
├── Chart.yaml                   # apiVersion: v2, name: carts, type: application
├── .helmignore                  # Excludes *.md, tests/ from packaged chart
├── values.yaml                  # Master defaults — every knob the chart exposes
│
└── templates/
    ├── _helpers.tpl             # Named templates: fullname, labels, dynamodb helpers
    ├── deployment.yaml          # Cart app Deployment
    ├── service.yaml             # ClusterIP service on port 80 → container 8080
    ├── serviceaccount.yaml      # Optional ServiceAccount
    ├── configmap.yaml           # Persistence provider + DynamoDB config env vars
    ├── dynamodb-deployment.yaml # DynamoDB Local Deployment (conditional)
    ├── dynamodb-service.yaml    # DynamoDB Local ClusterIP on port 8000 (conditional)
    ├── hpa.yaml                 # HorizontalPodAutoscaler (conditional)
    ├── pdb.yaml                 # PodDisruptionBudget (conditional)
    └── tests/                   # helm test hook (connectivity check)
```

**Key difference from catalog:** The cart chart has no `secret.yaml`. There are no database credentials to manage — DynamoDB Local uses dummy keys, and real AWS DynamoDB uses IRSA (IAM role attached to the Kubernetes ServiceAccount). Credentials never touch a Kubernetes Secret.

---

## `values.yaml` — Annotated Deep Dive

### Section 1 — Pod & Image

```yaml
replicaCount: 1

image:
  repository: public.ecr.aws/aws-containers/retail-store-sample-cart
  pullPolicy: IfNotPresent
  tag:
```

Same pattern as catalog: `replicaCount` is ignored when HPA is enabled. `image.tag` falls back to `.Chart.Version` when blank.

---

### Section 2 — ServiceAccount & Pod Settings

```yaml
serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}

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

The `serviceAccount.annotations` field is the **IRSA hook**. For production EKS deployments with real AWS DynamoDB, you set:

```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
```

This annotation causes the EKS Pod Identity Webhook to inject the AWS credentials as a projected volume. The AWS SDK (`sts` dependency in `pom.xml`) then exchanges the pod's service account token for temporary IAM credentials automatically. No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` env vars are needed.

The memory limit is `512Mi` — double the catalog service's `256Mi` — reflecting the heavier JVM footprint compared to the Go binary.

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

The scrape path is `/actuator/prometheus` — different from catalog's `/metrics`. This is because the cart uses Micrometer's Prometheus registry exposed via Spring Actuator, whereas catalog (Go) uses the native Prometheus client library. Both expose the same Prometheus text format, just at different paths.

---

### Section 4 — `app.persistence` (The Core Toggle)

```yaml
app:
  persistence:
    provider: in-memory        # "in-memory" | "dynamodb"
    dynamodb:
      tableName: Items
      createTable: false
```

`app.persistence.provider` is the master switch. It controls:

1. **ConfigMap value:** Sets `RETAIL_CART_PERSISTENCE_PROVIDER` env var → Spring reads it as `retail.cart.persistence.provider` → Spring's `config/` class activates either the in-memory or DynamoDB repository bean.
2. **Conditional DynamoDB resources:** Both `dynamodb-deployment.yaml` and `dynamodb-service.yaml` use a double condition: `{{- if and (eq "dynamodb" .Values.app.persistence.provider) .Values.dynamodb.create }}`. Both conditions must be true for those resources to be rendered.

`app.persistence.dynamodb.tableName` sets the DynamoDB table name the app will read/write. It defaults to `Items` — matching both the `application.yml` default and the table the app creates on startup when `createTable: true`.

`app.persistence.dynamodb.createTable` controls `RETAIL_CART_PERSISTENCE_DYNAMODB_CREATE_TABLE`. When `true`, the app bootstraps the DynamoDB table on first startup. This is only safe to use once — if the table already exists and `createTable: true`, the app startup will fail with a `ResourceInUseException`. In production with real AWS DynamoDB, the table should be pre-created via Terraform/CDK and `createTable` left `false`.

---

### Section 5 — `dynamodb` (In-Cluster DynamoDB Local)

```yaml
dynamodb:
  create: false

  image:
    repository: public.ecr.aws/aws-dynamodb-local/aws-dynamodb-local
    pullPolicy: IfNotPresent
    tag: "1.25.1"

  service:
    type: ClusterIP
    port: 8000

  podAnnotations: {}
  nodeSelector: {}
  tolerations: []
  affinity: {}
```

`dynamodb.create` is the gate for both `dynamodb-deployment.yaml` and `dynamodb-service.yaml`. Setting `dynamodb.create: true` alone is not sufficient — `app.persistence.provider` must also be `dynamodb` for the ConfigMap to write the correct endpoint env var.

When `dynamodb.create: true`, the ConfigMap template automatically injects:
```
RETAIL_CART_PERSISTENCE_DYNAMODB_ENDPOINT: http://<release>-carts-dynamodb:8000
AWS_ACCESS_KEY_ID: key
AWS_SECRET_ACCESS_KEY: secret
RETAIL_CART_PERSISTENCE_DYNAMODB_CREATE_TABLE: "true"
```

The dummy credentials (`key` / `secret`) are hardcoded strings in the ConfigMap template — they are not Secrets. DynamoDB Local does not authenticate; it accepts any credentials. This is intentional and safe for local use.

The DynamoDB Local pod runs as a plain `Deployment` (not StatefulSet) with `replicas: 1` hardcoded in the template — it does not respect `replicaCount`. It also has no `readOnlyRootFilesystem` or security context constraints, since DynamoDB Local needs to write its in-memory database files.

**Critical:** DynamoDB Local stores all data **in memory** by default. All cart data is lost when the DynamoDB Local pod restarts. There is no persistent volume option in this chart for DynamoDB Local. For data durability in non-production environments, you must use the external DynamoDB path (real AWS DynamoDB or a separately managed DynamoDB Local with a mounted volume).

---

### Section 6 — Autoscaling, Scheduling & Observability

```yaml
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}
topologySpreadConstraints: []

opentelemetry:
  enabled: false
  instrumentation: ""

podDisruptionBudget:
  enabled: false
  minAvailable: 2
  maxUnavailable: 1
```

Same patterns as catalog: HPA drops `replicas` from Deployment, `topologySpreadConstraints` maps directly to Pod spec, `opentelemetry.instrumentation` sets the OTEL Operator annotation. One difference: `autoscaling` for the cart service is more practically useful than for catalog, since cart is a stateless service (session state is in DynamoDB, not in the pod) and can safely scale horizontally.

---

## ConfigMap Template — Full Logic

The `configmap.yaml` template is the most complex in the chart. Understanding it fully explains all the scenario behaviours:

```yaml
data:
  RETAIL_CART_PERSISTENCE_PROVIDER: {{ .Values.app.persistence.provider }}

  {{- if (eq "dynamodb" .Values.app.persistence.provider) }}
  RETAIL_CART_PERSISTENCE_DYNAMODB_TABLE_NAME: {{ .Values.app.persistence.dynamodb.tableName }}

  {{- if .Values.dynamodb.create }}
  # In-cluster DynamoDB Local path
  RETAIL_CART_PERSISTENCE_DYNAMODB_CREATE_TABLE: "true"
  AWS_ACCESS_KEY_ID: key
  AWS_SECRET_ACCESS_KEY: secret
  RETAIL_CART_PERSISTENCE_DYNAMODB_ENDPOINT: http://{{ include "carts.dynamodb.fullname" . }}:{{ .Values.dynamodb.service.port }}

  {{- else }}
  # External DynamoDB path (real AWS or self-managed)
  RETAIL_CART_PERSISTENCE_DYNAMODB_CREATE_TABLE: "{{ .Values.app.persistence.dynamodb.createTable }}"
  # No endpoint set → SDK uses default AWS regional endpoint
  {{- end }}
  {{- end }}
```

The three possible ConfigMap outputs depending on values:

| Scenario | `provider` | `dynamodb.create` | What the ConfigMap contains |
|----------|------------|-------------------|----------------------------|
| In-memory | `in-memory` | `false` | `PROVIDER=in-memory` only |
| In-cluster DynamoDB Local | `dynamodb` | `true` | `PROVIDER`, table name, `CREATE_TABLE=true`, dummy keys, local endpoint |
| External / AWS DynamoDB | `dynamodb` | `false` | `PROVIDER`, table name, `CREATE_TABLE=<value>` — no endpoint (SDK auto-discovers AWS endpoint) |

---

## `_helpers.tpl` — Named Templates

| Template | Purpose |
|----------|---------|
| `carts.fullname` | Release-name-aware resource name (63 char limit) |
| `carts.name` | Chart name (default: `carts`) |
| `carts.labels` | Standard Helm labels: `helm.sh/chart`, `app.kubernetes.io/*` |
| `carts.selectorLabels` | Pod selector: name, instance, `component=service`, owner |
| `carts.serviceAccountName` | Uses `serviceAccount.name` or auto-generates from fullname |
| `carts.configMapName` | Uses `configMap.name` or auto-generates from fullname |
| `carts.podAnnotations` | Merges `podAnnotations` + `metrics.podAnnotations` |
| `carts.dynamodb.fullname` | `<fullname>-dynamodb` — used for DynamoDB pod + service names |
| `carts.dynamodb.labels` | Same as `carts.labels` but with `component=dynamodb` selector |
| `carts.dynamodb.selectorLabels` | Pod selector for DynamoDB: `component=dynamodb` |

The `carts.dynamodb.fullname` helper is the link between the ConfigMap and the DynamoDB service. The endpoint injected into the ConfigMap (`http://<carts.dynamodb.fullname>:8000`) matches the name of the Service created by `dynamodb-service.yaml` — guaranteeing the cart app can always resolve the DynamoDB Local DNS name.

---

## Deployment Template — Key Behaviours

- **`JAVA_OPTS`** is hardcoded as a literal env var (not from ConfigMap): `-XX:MaxRAMPercentage=75.0 -Djava.security.egd=file:/dev/urandom`. `MaxRAMPercentage=75.0` caps the JVM heap at 75% of the container memory limit (`512Mi`), giving the JVM `~384Mi` for the heap and leaving `~128Mi` for metaspace, stack, and OS overhead.
- **`/dev/urandom`**: `java.security.egd=file:/dev/urandom` replaces the default `/dev/random` entropy source. `/dev/random` can block on low-entropy systems (common in containers), causing slow startup. `/dev/urandom` is non-blocking and equally secure for this purpose.
- **No `envFrom.secretRef`**: Unlike catalog, there is no secret mounted. All config comes from the ConfigMap only.
- **Readiness probe**: `/actuator/health/readiness` with `initialDelaySeconds: 10` — gives Spring Boot time to fully initialize before traffic is routed.
- **`/tmp` volume**: `emptyDir medium: Memory` — required because `readOnlyRootFilesystem: true` prevents the JVM and Spring Boot from writing temp files to the container filesystem. The JVM uses `/tmp` for class data sharing and other runtime writes.

---

## End-to-End: What Happens on `helm install`

1. Helm merges `values.yaml` + any `-f` override files.
2. Templates are rendered:
   - `configmap.yaml` → always rendered (sets `RETAIL_CART_PERSISTENCE_PROVIDER` and optional DynamoDB vars)
   - `dynamodb-deployment.yaml` + `dynamodb-service.yaml` → only if `provider == "dynamodb"` AND `dynamodb.create == true`
   - `hpa.yaml` → only if `autoscaling.enabled: true`
   - `pdb.yaml` → only if `podDisruptionBudget.enabled: true`
3. DynamoDB Local pod starts (if applicable) and becomes ready.
4. Cart Deployment starts; Spring Boot reads `RETAIL_CART_PERSISTENCE_PROVIDER` from the ConfigMap.
5. Spring's `@Configuration` activates the correct repository bean (in-memory or DynamoDB).
6. If DynamoDB: app connects to the endpoint from `RETAIL_CART_PERSISTENCE_DYNAMODB_ENDPOINT` (or AWS regional endpoint) and optionally creates the table.
7. `/actuator/health/readiness` returns `UP` → Deployment marks pod as ready.
