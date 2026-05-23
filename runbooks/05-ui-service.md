# 05 — UI Service Deep Dive

> **Location:** `src/ui/`
> **Language:** Java 21
> **Framework:** Spring Boot 3.5.5 (Spring WebFlux + Thymeleaf)
> **Build Tool:** Maven (via `mvnw` wrapper)
> **Downstream Dependencies:** Catalog, Cart, Checkout, Orders (all HTTP via reactive `WebClient`)
> **AI Chat:** Optional — AWS Bedrock Converse API **or** OpenAI-compatible endpoint
> **Container Port:** `8080`
> **Public Image:** `public.ecr.aws/aws-containers/retail-store-sample-ui`

---

## What This Service Does

UI is the frontend and API gateway of the stack. It is the only service directly exposed to end-users. It renders server-side HTML via Thymeleaf templates, acts as a reverse proxy to the downstream microservices, and optionally serves an AI-powered chat assistant. Unlike the other Spring Boot services (which use Spring MVC), UI uses **Spring WebFlux** — the reactive, non-blocking HTTP stack. All downstream calls use a reactive `WebClient`.

---

## Source Tree

```
src/ui/
├── Dockerfile                   # Same pattern as orders: amazonlinux:2023 → curl-full swap
├── docker-compose.yml           # Local dev: ui + all downstream services
├── pom.xml                      # Spring WebFlux, Thymeleaf, Spring AI, WebClient
├── mvnw / mvnw.cmd              # Maven wrapper
├── project.json                 # Nx project definition
│
└── src/main/
    ├── java/com/amazon/sample/ui/
    │   ├── UiApplication.java       # @SpringBootApplication entry point
    │   ├── chat/                    # Spring AI chat integration: providers, service
    │   ├── client/                  # Reactive WebClient-based HTTP clients
    │   │   ├── cart/            # Cart service client
    │   │   ├── catalog/         # Catalog service client
    │   │   ├── checkout/        # Checkout service client
    │   │   └── orders/          # Orders service client
    │   ├── config/                  # Spring @Configuration: WebClient beans, AI config
    │   ├── services/                # Business logic layer (session, cart aggregation)
    │   ├── util/                    # Utility classes
    │   └── web/                     # Spring WebFlux controllers
    │       ├── CartController.java
    │       ├── CatalogController.java
    │       ├── ChatController.java
    │       ├── CheckoutController.java
    │       ├── HomeController.java
    │       ├── MetadataController.java
    │       ├── ProxyController.java    # Reverse proxy for static assets
    │       ├── TopologyController.java  # Returns wired endpoint topology (debug)
    │       ├── UtilityController.java
    │       └── dialect/                 # Custom Thymeleaf dialect extensions
    └── resources/
        ├── application.yml              # Base config: endpoints, chat, actuator
        ├── application-prod.yml         # Production overrides
        ├── templates/                   # Thymeleaf HTML templates
        ├── static/                      # CSS, JS, images
        └── lang/                        # i18n message bundles
```

---

## Application Runtime

### Architecture — WebFlux vs MVC

UI uses **Spring WebFlux** (Netty reactor) rather than Spring MVC (Tomcat). Key implications:

| Aspect | Spring MVC (Cart, Orders) | Spring WebFlux (UI) |
|--------|--------------------------|---------------------|
| Thread model | Thread-per-request (Tomcat) | Event-loop (Netty) |
| HTTP client | `RestTemplate` / `RestClient` | Reactive `WebClient` |
| Response type | Synchronous | `Mono<T>` / `Flux<T>` |
| Thymeleaf | Synchronous rendering | Reactive Thymeleaf |
| Suitability | CRUD services, DB-heavy | API gateway, fan-out calls |

For the UI service — which fans out to 4 downstream services on every page render — the non-blocking model prevents thread starvation under load. A traditional MVC controller waiting on 4 concurrent HTTP calls would hold 4 threads; WebFlux holds none.

### `application.yml` — Annotated

```yaml
server:
  port: ${port:8080}
  forward-headers-strategy: NATIVE  # Trusts X-Forwarded-* headers from ALB/Nginx ingress

spring:
  messages:
    basename: lang/messages           # i18n message bundles
  http:
    codecs:
      max-in-memory-size: 10MB        # Max response buffer for downstream calls

retail:
  ui:
    theme: default
    endpoints:
      catalog:   # http://localhost:8081
      carts:     # http://localhost:8082
      checkout:  # http://localhost:8085
      orders:    # http://localhost:8083
    chat:
      enabled: false
      provider:           # "mock" | "bedrock" | "openai"
      model:
      temperature: 0.6
      max-tokens: 300
      prompt: |           # Full system prompt (A.G.E.N.T. persona — spy-themed assistant)
      bedrock:
        region:
      openai:
        base-url: http://localhost:8000
        api-key:

management:
  endpoints:
    web:
      exposure:
        include: info,health,metrics,prometheus

otel.sdk.disabled: true
```

`forward-headers-strategy: NATIVE` is essential when UI is behind an ALB or Nginx ingress controller — it propagates the real client IP and protocol from `X-Forwarded-For` / `X-Forwarded-Proto` headers into the request context (used for redirect URL generation).

`max-in-memory-size: 10MB` raises the WebFlux codec buffer limit. The default (256KB) can be exceeded when the catalog returns large product image datasets.

### API Routes

| Controller | Routes | Description |
|------------|--------|-------------|
| `HomeController` | `GET /` | Homepage, product listing |
| `CatalogController` | `GET /catalog/**` | Product detail pages |
| `CartController` | `GET /cart`, `POST /cart/**` | Cart view and mutations |
| `CheckoutController` | `GET /checkout/**`, `POST /checkout/**` | Checkout flow |
| `ChatController` | `POST /chat` | AI chat endpoint (when enabled) |
| `ProxyController` | `GET /assets/**` | Reverse-proxies static asset requests to catalog or S3 |
| `TopologyController` | `GET /topology` | Returns configured downstream endpoint URLs (debug) |
| `AppController` | `GET /actuator/health` | Spring Boot health (liveness) |
| | `GET /actuator/health/readiness` | Readiness probe |
| | `GET /actuator/prometheus` | Prometheus metrics |

### Key Dependencies — `pom.xml`

| Dependency | Purpose |
|------------|---------|
| `spring-boot-starter-webflux` | Reactive HTTP server (Netty) + `WebClient` |
| `spring-boot-starter-thymeleaf` | Server-side HTML rendering |
| `spring-boot-starter-actuator` | Health, metrics, prometheus endpoints |
| `spring-boot-starter-validation` | Bean validation (`@Valid` on form DTOs) |
| `spring-boot-starter-log4j2` | Log4j2 (replaces default Logback) |
| `micrometer-registry-prometheus` | Prometheus metrics scrape |
| `spring-ai-starter-model-bedrock-converse` | AWS Bedrock Converse API chat provider |
| `spring-ai-openai` | OpenAI-compatible chat provider (also works with Ollama) |
| `spring-boot-properties-migrator` | Warns about deprecated property keys on startup |
| `opentelemetry-spring-boot-starter` | OTEL auto-instrumentation |

**Spring AI** is the most distinctive dependency in the stack — not present in any other service. The BOM version is `1.0.0`. Two providers are bundled: Bedrock Converse (for AWS-native deployments) and OpenAI-compatible (for self-hosted models like Ollama or direct OpenAI). The `mock` provider requires no configuration and returns canned responses — useful for demos without a real LLM.

### AI Chat Architecture

```
ChatController
    └→ ChatService
           ├→ BedrockChatProvider   (provider=bedrock)  → AWS Bedrock Converse API via IRSA
           ├→ OpenAIChatProvider    (provider=openai)   → OpenAI API or Ollama
           └→ MockChatProvider      (provider=mock)     → Static canned response
```

`RETAIL_UI_CHAT_ENABLED=true` alone is not enough — `RETAIL_UI_CHAT_PROVIDER` must also be set. For `bedrock`, the pod's ServiceAccount must have an IRSA annotation with an IAM role that has `bedrock:InvokeModel` permission. For `openai`, `RETAIL_UI_CHAT_OPENAI_BASE_URL` must point to a running OpenAI-compatible server.

---

## Dockerfile — Build Strategy

Identical two-stage build pattern to the Orders service:

| Stage | Base | What Happens |
|-------|------|--------------|
| `build-env` | `amazonlinux:2023` | Installs Maven + Java 21 Corretto, `mvnw dependency:go-offline`, builds `target/ui-0.0.1-SNAPSHOT.jar` → `/app.jar` |
| Runtime | `amazonlinux:2023` (fresh) | Installs Java 21 Corretto, **swaps `curl-minimal` → `curl-full`** (same reason as Orders — health checks may use `telnet://` scheme), creates `appuser` (UID 1000), `SPRING_PROFILES_ACTIVE=prod` |

`ENTRYPOINT` is `sh -c java $JAVA_OPTS -jar /app/app.jar` — shell wrapper for `$JAVA_OPTS` expansion.

---

## Helm Chart

### Chart Files

```
src/ui/chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── _helpers.tpl
    ├── configmap.yml
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── ingress.yaml            # Single ingress OR multi-ingress (ingresses[])
    ├── istio-gateway.yml       # Istio Gateway CRD
    ├── istio-virtualservice.yml # Istio VirtualService CRD
    ├── hpa.yaml
    ├── pdb.yaml
    └── tests/
```

UI is the **only service in the stack** with Ingress, Istio Gateway, and Istio VirtualService templates — because it is the only externally-exposed service.

---

## `values.yaml` — Annotated Sections

### Section 1 — Pod & Image

```yaml
replicaCount: 1
image:
  repository: public.ecr.aws/aws-containers/retail-store-sample-ui
  pullPolicy: IfNotPresent
  tag:
```

`image.tag` defaults to `.Chart.Version`. UI is stateless — no persistence, no messaging. Multiple replicas are safe without any additional configuration.

---

### Section 2 — Security Context

```yaml
securityContext:
  capabilities:
    drop: [ALL]
    add: [NET_BIND_SERVICE]   # <— unique to UI
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
```

**`NET_BIND_SERVICE` capability** is added only in the UI chart (not present in any other service chart). It allows the process to bind to ports below 1024. In practice the container runs on 8080 (which does not need this), but the Helm chart includes it for deployments where a service mesh sidecar or init container may need to intercept traffic on privileged ports.

---

### Section 3 — Resources & Metrics

```yaml
resources:
  limits:
    memory: 512Mi
  requests:
    cpu: 128m
    memory: 512Mi

metrics:
  enabled: true
  podAnnotations:
    prometheus.io/path: '/actuator/prometheus'   # Java service path
```

Same `512Mi` as the Spring Boot services. HPA `targetCPUUtilizationPercentage` defaults to `50` (vs. `80` in other charts) — UI scales out more aggressively because it is the user-facing entry point.

---

### Section 4 — `app.endpoints` (Downstream Service Wiring)

```yaml
app:
  endpoints: {}
    # catalog:  http://catalog:80
    # carts:    http://carts:80
    # orders:   http://orders:80
    # checkout: http://checkout:80
```

All four endpoints are **optional** in the values. Each is only injected into the ConfigMap if set. A missing endpoint means that feature area will fail at runtime with a connection error (not at deploy time). The typical full-stack wiring:

| Env Var | Value | Serves |
|---------|-------|--------|
| `RETAIL_UI_ENDPOINTS_CATALOG` | `http://catalog` | Product listing, detail pages |
| `RETAIL_UI_ENDPOINTS_CARTS` | `http://carts` | Cart add/remove/view |
| `RETAIL_UI_ENDPOINTS_CHECKOUT` | `http://checkout` | Checkout session management |
| `RETAIL_UI_ENDPOINTS_ORDERS` | `http://orders` | Order history |

`RETAIL_UI_ENDPOINTS_ASSETS` is also supported (not in default values) — routes `/assets/**` proxy requests to a CDN or S3 bucket instead of the catalog service.

---

### Section 5 — `app.chat` (AI Chat Toggle)

```yaml
app:
  chat:
    enabled: false
    provider: ""          # "mock" | "bedrock" | "openai"
    model: ""
    # temperature: 0.7
    # maxTokens: 300
    # prompt: |            # Overrides the built-in A.G.E.N.T. system prompt
    bedrock:
      region: ""
    openai:
      baseUrl: ""
      # apiKey: ""
```

| Provider | Required Config | Auth |
|----------|----------------|------|
| `mock` | Nothing | None |
| `bedrock` | `app.chat.bedrock.region`, `app.chat.model` | IRSA — `serviceAccount.annotations` with IAM role |
| `openai` | `app.chat.openai.baseUrl`, `app.chat.model` | Optional `apiKey` (Secret-mounted separately) |

When `chat.enabled: true` and `provider: bedrock`, the ConfigMap sets `RETAIL_UI_CHAT_BEDROCK_REGION` and the pod uses IRSA to call Bedrock — no API key Secret needed. When `provider: openai`, `RETAIL_UI_CHAT_OPENAI_BASE_URL` points to any OpenAI-compatible endpoint (e.g., `http://ollama:11434/v1` for a self-hosted Ollama deployment).

---

### Section 6 — Traffic Exposure (Three Options)

UI is the only chart that exposes three distinct traffic ingress mechanisms:

| Mechanism | When to Use |
|-----------|------------|
| `ingress.enabled: true` | Single Kubernetes Ingress (e.g., Nginx, AWS ALB via annotations) |
| `ingresses: [...]` | Multiple Ingresses with different names/classes (e.g., internal + external ALBs) |
| `istio.enabled: true` | Istio Gateway + VirtualService (service mesh deployments) |

> **Mutual exclusion:** `ingress.enabled` and `ingresses` cannot both be set — the template has a `{{- fail }}` guard that aborts rendering if both are configured simultaneously.

**Single Ingress (AWS ALB example):**
```yaml
ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /actuator/health/liveness
  hosts:
    - my-store.example.com
```

**Multi-Ingress (internal + external):**
```yaml
ingresses:
  - name: external
    className: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
    hosts: [my-store.example.com]
  - name: internal
    className: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internal
    hosts: [my-store.internal]
```

**Istio:**
```yaml
istio:
  enabled: true
  hosts:
    - my-store.example.com
```
Renders a `Gateway` (port 80, HTTP) and a `VirtualService` routing all traffic to the UI service.

---

## ConfigMap Logic

```yaml
data:
  # Optional: only rendered when set
  RETAIL_UI_THEME:                     # if app.theme set
  RETAIL_UI_ENDPOINTS_CATALOG:         # if app.endpoints.catalog set
  RETAIL_UI_ENDPOINTS_CARTS:           # if app.endpoints.carts set
  RETAIL_UI_ENDPOINTS_CHECKOUT:        # if app.endpoints.checkout set
  RETAIL_UI_ENDPOINTS_ORDERS:          # if app.endpoints.orders set
  RETAIL_UI_ENDPOINTS_ASSETS:          # if app.endpoints.assets set

  # Chat block: only rendered when app.chat.enabled=true
  RETAIL_UI_CHAT_ENABLED:              # "true"
  RETAIL_UI_CHAT_PROVIDER:             # "mock" | "bedrock" | "openai"
  RETAIL_UI_CHAT_MODEL:
  RETAIL_UI_CHAT_TEMPERATURE:          # if set
  RETAIL_UI_CHAT_MAX_TOKENS:           # if set
  RETAIL_UI_CHAT_PROMPT:               # if set (overrides built-in A.G.E.N.T. prompt)

  # Provider-specific (within chat block)
  RETAIL_UI_CHAT_OPENAI_BASE_URL:      # if provider=openai
  RETAIL_UI_CHAT_OPENAI_API_KEY:       # if provider=openai AND apiKey set
  RETAIL_UI_CHAT_BEDROCK_REGION:       # if provider=bedrock
```

| Scenario | ConfigMap env vars |
|----------|-------------------|
| Minimal — no endpoints, no chat | Empty ConfigMap (only header) |
| Full-stack, no chat | 4 endpoint vars |
| Full-stack + mock chat | 4 endpoints + `CHAT_ENABLED`, `CHAT_PROVIDER=mock`, `CHAT_MODEL` |
| Full-stack + Bedrock chat | + `CHAT_BEDROCK_REGION` |
| Full-stack + OpenAI chat | + `CHAT_OPENAI_BASE_URL` (+ `CHAT_OPENAI_API_KEY` if set) |

---

## Deployment Template — Key Behaviours

- **`NET_BIND_SERVICE` capability:** Only UI has `capabilities.add: [NET_BIND_SERVICE]` — all other services only `drop: [ALL]`.
- **No `secretRef`:** All config comes from the ConfigMap. If `openai.apiKey` is set, it lands in the ConfigMap as plaintext — for production, pre-create a Secret and inject it via a custom `secretRef` block or use an external secrets operator.
- **`JAVA_OPTS`** injected as env var — same shell-wrapper ENTRYPOINT pattern as Orders.
- **Readiness probe:** `GET /actuator/health/readiness` (port 8080) — same split liveness/readiness pattern as the other Java services.
- **`/tmp` volume:** `emptyDir medium: Memory` — required for `readOnlyRootFilesystem: true`.
- **`topologySpreadConstraints`:** Available at top-level values (same as Checkout).
- **HPA CPU target:** `50%` (more aggressive than the `80%` in other services) — appropriate for the user-facing entry point.

---

## End-to-End: What Happens on `helm install`

1. Helm merges `values.yaml` + any `-f` override files.
2. Templates rendered:
   - `configmap.yml` → always (if `configMap.create: true`)
   - `ingress.yaml` (single) → only if `ingress.enabled: true`
   - `ingress.yaml` (multi) → only if `ingresses[]` is non-empty
   - `istio-gateway.yml` + `istio-virtualservice.yml` → only if `istio.enabled: true`
   - `hpa.yaml` → only if `autoscaling.enabled: true`
   - `pdb.yaml` → only if `podDisruptionBudget.enabled: true`
3. UI Deployment starts; Spring WebFlux reads endpoint + chat env vars from ConfigMap.
4. `WebClient` beans are initialized for each configured downstream endpoint.
5. Spring AI `ChatClient` is conditionally initialized based on `provider`.
6. `GET /actuator/health/readiness` returns `UP` → pod marked ready.
7. Traffic reaches the pod via the configured Ingress, `ingresses[]`, or Istio Gateway.
