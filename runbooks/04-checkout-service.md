# 04 — Checkout Service Deep Dive

> **Location:** `src/checkout/`
> **Language:** TypeScript (Node.js 20)
> **Framework:** NestJS 11 (Express platform)
> **Build Tool:** Yarn 4 (Berry) + `nest build` (TypeScript → `dist/`)
> **Database:** Redis (in-cluster Deployment or external/ElastiCache) **or** in-memory
> **Downstream Dependency:** Orders service (HTTP) — wired via `RETAIL_CHECKOUT_ENDPOINTS_ORDERS`
> **Container Port:** `8080`
> **Public Image:** `public.ecr.aws/aws-containers/retail-store-sample-checkout`

---

## What This Service Does

Checkout is the session management layer of the stack. It stores the in-progress cart/checkout state (items, shipping address, delivery option) while the user is working through the checkout flow, then submits the final order to the Orders service via HTTP. It is the only Node.js / TypeScript service in the stack and the only service with a **downstream HTTP dependency** (Orders) that must be configured explicitly.

---

## Source Tree

```
src/checkout/
├── Dockerfile                   # Two-stage: node:20-alpine build → amazonlinux:2023 runtime
├── docker-compose.yml           # Local dev: checkout + Redis
├── package.json                 # NestJS 11, ioredis, nestjs-otel, prom-client
├── nest-cli.json                # NestJS CLI config: compilerOptions, entryFile
├── tsconfig.json / tsconfig.build.json  # TypeScript compiler config
├── yarn.lock / .yarnrc.yml      # Yarn 4 (Berry) lockfile + config
├── openapi.yml                  # OpenAPI 3.0 spec
├── eslint.config.mjs            # ESLint flat config (TypeScript)
├── .env                         # Local dev defaults (not used in Kubernetes)
├── .prettierrc                  # Prettier formatting rules
│
├── src/
│   ├── main.ts                  # NestJS bootstrap: OTEL SDK start, Swagger setup, listen on PORT
│   ├── app.module.ts            # Root NestJS module: imports, middleware wiring
│   ├── app.controller.ts        # GET /health (Terminus), GET /topology (persistence info)
│   ├── tracing.ts               # OTEL SDK setup: OTLP exporter, AWS X-Ray ID generator
│   ├── config/
│   │   └── configuration.ts     # NestJS ConfigModule factory: reads env vars → typed config object
│   ├── checkout/                # Core checkout domain module
│   │   ├── checkout.module.ts
│   │   ├── checkout.controller.ts  # REST handlers for checkout CRUD
│   │   ├── checkout.service.ts     # Business logic: session state, order submission
│   │   ├── models/              # TypeScript DTOs and interfaces
│   │   ├── orders/              # HTTP client for the Orders service
│   │   ├── repositories/        # Persistence abstraction: in-memory vs Redis impls
│   │   └── shipping/            # Shipping options logic
│   ├── chaos/                   # Chaos engineering: injectable failure middleware
│   └── middleware/
│       └── logger.middleware.ts # HTTP request logging (excludes /health)
│
├── test/                        # e2e tests (Jest + Supertest)
├── scripts/
└── chart/                       # Helm chart — covered below
```

---

## Application Runtime

### Bootstrap — `main.ts`

1. `tracing.ts` OTEL SDK starts **before** `NestFactory.create()` — required to instrument all NestJS module initializations.
2. `NestFactory.create(AppModule)` wires all modules, middleware, and DI providers.
3. `ValidationPipe` is registered globally — all incoming request bodies are validated via `class-validator` decorators.
4. Swagger UI is mounted at `/api` (scope: `CheckoutModule` routes only).
5. `app.enableShutdownHooks()` registers SIGTERM/SIGINT handlers for graceful shutdown.
6. Server starts on `process.env.PORT || 8080`.

### Module Wiring — `app.module.ts`

| Module | Purpose |
|--------|---------|
| `ConfigModule` | Loads `configuration.ts` factory; makes typed config available via `ConfigService` |
| `TerminusModule` | Powers `GET /health` endpoint with pluggable health indicators |
| `PrometheusModule` | Registers Prometheus metrics at `GET /metrics` |
| `CheckoutModule` | Core domain: checkout controller, service, repositories, orders client |
| `OpenTelemetryModule` | OTEL auto-instrumentation for NestJS (HTTP, gRPC, DB calls) |

**Middleware:**
- `LoggerMiddleware` — applied to all routes except `/health`
- `ChaosMiddleware` — applied to `checkout/*` routes; injects controllable failures for resilience testing

### Config Factory — `src/config/configuration.ts`

```typescript
export default () => ({
  persistence: {
    provider: process.env.RETAIL_CHECKOUT_PERSISTENCE_PROVIDER || 'in-memory',
    redis: {
      url:    process.env.RETAIL_CHECKOUT_PERSISTENCE_REDIS_URL    || '',
      reader: { url: process.env.RETAIL_CHECKOUT_PERSISTENCE_REDIS_READER_URL || '' },
    },
  },
  endpoints: {
    orders: process.env.RETAIL_CHECKOUT_ENDPOINTS_ORDERS || '',
  },
  shipping: {
    prefix: process.env.RETAIL_CHECKOUT_SHIPPING_NAME_PREFIX || '',
  },
});
```

The factory pattern (vs. `process.env` inline reads) enables typed injection via `ConfigService.get('persistence.provider')`. It also enables mocking in unit tests without touching `process.env`.

`RETAIL_CHECKOUT_PERSISTENCE_REDIS_READER_URL` is used for **read replicas** — specifically AWS ElastiCache cluster mode where reads can be directed to a separate reader endpoint for lower latency.

### API Routes

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Terminus health check (liveness + chaos indicator) |
| `GET` | `/topology` | Returns `persistenceProvider` + `databaseEndpoint` (debug) |
| `POST` | `/checkout` | Create a new checkout session |
| `GET` | `/checkout/{checkoutId}` | Get checkout session by ID |
| `PUT` | `/checkout/{checkoutId}` | Update checkout session (items, address) |
| `POST` | `/checkout/{checkoutId}/confirm` | Submit order to Orders service |
| `GET` | `/metrics` | Prometheus metrics scrape endpoint |
| `GET` | `/api` | Swagger UI (CheckoutModule routes) |

> **Note:** Checkout uses a single `/health` endpoint for both liveness and readiness probes (unlike the Java services which split liveness vs. readiness via `/actuator/health` and `/actuator/health/readiness`). The Kubernetes readiness probe in `deployment.yaml` hits `GET /health`.

### Persistence — In-Memory vs Redis

| Provider | Behaviour | State on Restart |
|----------|-----------|------------------|
| `in-memory` | Checkout sessions stored in Node.js process memory (plain Map) | **Lost** — every pod restart clears all sessions |
| `redis` | Sessions stored in Redis via `ioredis` | **Persistent** across restarts; shared across replicas |

`in-memory` is the default and is only suitable for single-replica deployments or demos. For multi-replica deployments (`replicaCount > 1` or HPA enabled), Redis is **required** — without it, a user's session will be lost if the request is routed to a different replica.

### Key Dependencies — `package.json`

| Package | Version | Purpose |
|---------|---------|--------|
| `@nestjs/common` / `core` / `platform-express` | ^11.1.3 | NestJS framework + Express HTTP adapter |
| `@nestjs/config` | ^4.0.2 | ConfigModule + ConfigService |
| `@nestjs/terminus` | ^11.0.0 | Health check endpoints (`/health`) |
| `@nestjs/swagger` | ^11.2.0 | OpenAPI docs + Swagger UI at `/api` |
| `ioredis` | ^5.9.2 | Redis client (supports cluster, sentinel, TLS) |
| `@willsoto/nestjs-prometheus` | ^6.0.2 | Prometheus metrics at `/metrics` |
| `prom-client` | ^15.1.3 | Prometheus client library (peer dep) |
| `class-validator` / `class-transformer` | ^0.14.2 / ^0.5.1 | DTO validation via decorators + `ValidationPipe` |
| `uuid` | ^9.0.0 | Session ID generation |
| `@opentelemetry/sdk-node` | ^0.202.0 | OTEL Node.js SDK |
| `@opentelemetry/auto-instrumentations-node` | ^0.60.1 | Auto-instrumentation for HTTP, Express, Redis, etc. |
| `@opentelemetry/exporter-trace-otlp-http` | ^0.202.0 | OTLP HTTP trace exporter (to OTEL Collector) |
| `@opentelemetry/id-generator-aws-xray` | ^2.0.0 | AWS X-Ray compatible trace ID format |
| `nestjs-otel` | ^7.0.0 | NestJS OTEL module integration |

---

## Dockerfile — Build Strategy

Two-stage build:

| Stage | Base | What Happens |
|-------|------|--------------|
| `build` | `node:20-alpine` | Copies `package*.json`, runs `yarn install --frozen-lockfile` (uses `yarn.lock` for exact versions), runs `nest build` → compiles TypeScript to `dist/` |
| Runtime | `public.ecr.aws/amazonlinux/amazonlinux:2023` | Installs `nodejs20` via `dnf` (weak deps disabled), creates `appuser` (UID 1000), copies `node_modules/` and `dist/` from build stage |

**Key differences from the Java services:**
- No `curl-full` needed — NestJS health probes are HTTP-native (Terminus uses `@nestjs/axios` internally).
- `ENTRYPOINT` is `["node", "dist/main.js"]` — no shell wrapper needed since there are no JVM flags to expand.
- `node_modules/` is copied from the build stage, not reinstalled in the runtime stage — keeps the runtime layer lean.
- Build stage uses `node:20-alpine` (smaller, faster) while runtime uses `amazonlinux:2023` for consistency with the rest of the stack.

---

## Helm Chart

### Chart Files

```
src/checkout/chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── configmap.yaml
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── redis-deployment.yaml    # Redis as plain Deployment (not StatefulSet)
    ├── redis-service.yaml
    ├── security-group.yaml      # AWS SecurityGroupPolicy CRD (EKS only)
    ├── hpa.yaml
    ├── pdb.yaml
    └── tests/
```

> **Redis as Deployment (not StatefulSet):** Unlike PostgreSQL and RabbitMQ in the Orders chart (which use StatefulSets for stable identity), the in-cluster Redis here is a plain **Deployment**. Redis does not require stable pod identity for single-node mode — and for the demo use case, a Deployment is sufficient. Production Redis should use ElastiCache or a proper Redis operator.

---

## `values.yaml` — Annotated Sections

### Section 1 — Pod & Image

```yaml
replicaCount: 1
image:
  repository: public.ecr.aws/aws-containers/retail-store-sample-checkout
  pullPolicy: IfNotPresent
  tag:
```

`image.tag` defaults to `.Chart.Version` when blank. `replicaCount` is ignored when `autoscaling.enabled: true`.

> **Multi-replica warning:** If `replicaCount > 1` or HPA is enabled, you **must** set `app.persistence.provider: redis` and configure a Redis endpoint. In-memory sessions are not shared across replicas — users will get session-not-found errors on requests routed to a different pod.

---

### Section 2 — ServiceAccount & Security

```yaml
serviceAccount:
  create: true
  annotations: {}
  name: ''

podSecurityContext:
  fsGroup: 1000

securityContext:
  capabilities:
    drop: [ALL]
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
```

`serviceAccount.annotations` is available for IRSA but checkout has no AWS service calls — it does not need an IAM role under normal operation.

---

### Section 3 — Resources & Metrics

```yaml
resources:
  limits:
    memory: 256Mi
  requests:
    cpu: 128m
    memory: 256Mi

metrics:
  enabled: true
  podAnnotations:
    prometheus.io/scrape: 'true'
    prometheus.io/port: '8080'
    prometheus.io/path: '/metrics'
```

**Metrics path is `/metrics`** (not `/actuator/prometheus` as in the Java services). NestJS + `@willsoto/nestjs-prometheus` registers the scrape endpoint at `/metrics` by default.

`256Mi` memory limit is appropriate for a Node.js service — the V8 heap is much smaller than the JVM heap required by the Spring Boot services (which use `512Mi`).

---

### Section 4 — `app.persistence` (Redis Toggle)

```yaml
app:
  persistence:
    provider: 'in-memory'    # "in-memory" | "redis"
    redis:
      endpoint: ''
      tls: false
```

| Value | What Renders |
|-------|-------------|
| `in-memory` | ConfigMap contains only `RETAIL_CHECKOUT_PERSISTENCE_PROVIDER=in-memory`; no Redis URL set |
| `redis` + `redis.create: true` | ConfigMap sets `RETAIL_CHECKOUT_PERSISTENCE_REDIS_URL=redis://<release>-checkout-redis:6379` (auto-generated) |
| `redis` + `redis.create: false` | ConfigMap sets `RETAIL_CHECKOUT_PERSISTENCE_REDIS_URL=redis[s]://<endpoint>` using `app.persistence.redis.endpoint` and `tls` flag |

`app.persistence.redis.tls: true` changes the URL scheme from `redis://` to `rediss://` (double-s = TLS). Required for AWS ElastiCache with in-transit encryption enabled.

---

### Section 5 — `app.endpoints.orders`

```yaml
app:
  endpoints:
    orders: ''
```

This is the **only cross-service wiring** the checkout chart needs. Set it to the Orders service ClusterIP DNS name:

```yaml
app:
  endpoints:
    orders: 'http://orders'
```

When left blank, `RETAIL_CHECKOUT_ENDPOINTS_ORDERS` is not injected into the ConfigMap (the template conditionally omits it — see ConfigMap Logic below). In that case, the checkout service can accept and persist sessions but order submission will fail at runtime.

---

### Section 6 — `redis` (In-Cluster Redis)

```yaml
redis:
  create: false
  image:
    repository: redis
    tag: '6.0-alpine'
  service:
    type: ClusterIP
    port: 6379
```

When `redis.create: true`, renders a plain `Deployment` + `ClusterIP Service` for Redis 6.0. No StatefulSet, no PVC — data is ephemeral. Suitable for non-production use only.

---

### Section 7 — `securityGroups`, Autoscaling, Observability, PDB

```yaml
securityGroups:
  create: false
  securityGroupIds: []

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

opentelemetry:
  enabled: false
  instrumentation: ''

podDisruptionBudget:
  enabled: false
  minAvailable: 2
  maxUnavailable: 1
```

Same patterns as the Orders chart. `securityGroups` assigns a VPC security group to the pod via the AWS VPC CNI `SecurityGroupPolicy` CRD — used when Redis (ElastiCache) is protected by a VPC security group. `opentelemetry.instrumentation` injects the OTEL Operator pod annotation.

---

## ConfigMap Logic

```yaml
data:
  RETAIL_CHECKOUT_PERSISTENCE_PROVIDER: {{ .Values.app.persistence.provider }}
  {{- if (eq "redis" .Values.app.persistence.provider) }}
    {{- if .Values.redis.create }}
  RETAIL_CHECKOUT_PERSISTENCE_REDIS_URL: redis://{{ include "checkout.redis.fullname" . }}:{{ .Values.redis.service.port }}
    {{- else }}
  RETAIL_CHECKOUT_PERSISTENCE_REDIS_URL: {{ ternary "rediss" "redis" .Values.app.persistence.redis.tls }}://{{ .Values.app.persistence.redis.endpoint }}
    {{- end }}
    {{- if .Values.app.endpoints.orders }}
  RETAIL_CHECKOUT_ENDPOINTS_ORDERS: {{ .Values.app.endpoints.orders }}
    {{- end }}
  {{- end }}
```

| Scenario | ConfigMap env vars |
|----------|-------------------|
| `provider=in-memory` | `PERSISTENCE_PROVIDER` only |
| `provider=redis`, `redis.create=true` | + `REDIS_URL` (auto-generated in-cluster address) |
| `provider=redis`, `redis.create=false`, `tls=false` | + `REDIS_URL=redis://<endpoint>` |
| `provider=redis`, `redis.create=false`, `tls=true` | + `REDIS_URL=rediss://<endpoint>` |
| Any + `app.endpoints.orders` set | + `ENDPOINTS_ORDERS` (only when provider=redis) |

> **Note:** `RETAIL_CHECKOUT_ENDPOINTS_ORDERS` is **only rendered when `provider=redis`**. This is intentional — the in-memory provider is used for standalone demos where there is no Orders service. For a full-stack deployment with order submission, `provider` must be `redis`.

---

## Deployment Template — Key Behaviours

- **No `secretRef`**: Checkout has no Secrets. All config comes from the ConfigMap via `envFrom.configMapRef`.
- **No `JAVA_OPTS`**: Node.js V8 memory is managed differently — no explicit heap flags needed at this scale.
- **Readiness probe:** `GET /health` (single endpoint, port 8080) — no split liveness/readiness like the Java services.
- **`/tmp` volume:** `emptyDir medium: Memory` — required because `readOnlyRootFilesystem: true` and Node.js needs a writable temp dir.
- **`topologySpreadConstraints`:** Exposed as a top-level values key (not present in the Java service charts at this level) — allows zone-spread scheduling for multi-AZ Redis-backed deployments.
- **Rolling update:** `maxUnavailable: 1` — same as all other services in the stack.

---

## End-to-End: What Happens on `helm install`

1. Helm merges `values.yaml` + any `-f` override files.
2. Templates rendered:
   - `configmap.yaml` → always (if `configMap.create: true`)
   - `redis-deployment.yaml` + `redis-service.yaml` → only if `redis.create: true`
   - `security-group.yaml` → only if `securityGroups.create: true`
   - `hpa.yaml` → only if `autoscaling.enabled: true`
   - `pdb.yaml` → only if `podDisruptionBudget.enabled: true`
3. Redis Deployment starts (if applicable) and becomes ready.
4. Checkout Deployment starts; Node.js reads `RETAIL_CHECKOUT_PERSISTENCE_PROVIDER` from ConfigMap.
5. `repositories/` layer initialises the selected persistence backend (in-memory Map or `ioredis` client).
6. `GET /health` returns `{status: "ok"}` → Kubernetes marks pod as ready.
