# 06 — App Aggregator (`src/app/`)

> **Location:** `src/app/`
> **Role:** Top-level orchestration layer — no application code, no Dockerfile, no runtime service
> **Contains:** Umbrella Helm chart, Helmfile releases, Docker Compose aggregator, Tilt dev loop, Go templates, OTEL collector config

`src/app/` is the single entry point for deploying the entire retail-store-sample stack. It references each service chart as a dependency and provides tooling for three deployment contexts: **Helm** (umbrella chart), **Helmfile** (release orchestration), and **Docker Compose** (local dev).

---

## Directory Layout

```
src/app/
├── chart/                      # Umbrella Helm chart (wraps all 5 service charts)
│   ├── Chart.yaml              # Lists 5 sub-chart dependencies (file:// repos)
│   ├── values.yaml             # Base wiring: fullnameOverride + UI endpoint URLs
│   └── values-stateful.yaml    # Stateful overlay: enables in-cluster DB/broker per service
├── templates/                  # Go templates (.gotmpl) used by helmfile.yaml
│   ├── components.yaml.gotmpl  # IMAGE_TAG + IMAGE_REPOSITORY injection
│   ├── ingress.yaml.gotmpl     # INGRESS / NODE_PORT / LOAD_BALANCER env-driven exposure
│   ├── catalog.yaml.gotmpl     # Tilt mode + RANDOM_PASSWORD for MySQL
│   ├── carts.yaml.gotmpl       # Tilt mode image override
│   ├── orders.yaml.gotmpl      # Tilt mode + RANDOM_PASSWORD for PostgreSQL
│   ├── checkout.yaml.gotmpl    # Tilt mode image override
│   └── ui.yaml.gotmpl          # Tilt mode image override
├── scripts/
│   └── generate_password.sh    # Generates/caches a 20-char random password in .random_password
├── helmfile.yaml               # Full-stack: all 5 services + DB/broker per service
├── helmfile.slim.yaml          # Minimal: catalog + ui only (stateless demo)
├── docker-compose.yml          # Aggregator: includes all 5 service compose files
├── compose.override.yaml       # Wires UI endpoints + collapses service port bindings
├── docker-compose.tracing.yml  # Adds OTEL collector + Jaeger for distributed tracing
├── otel-collector-config.yml   # OTEL collector: OTLP HTTP receiver → Jaeger exporter
├── Tiltfile                    # Tilt: helmfile template + docker_build for all 5 services
└── project.json                # Nx project definition
```

---

## Umbrella Helm Chart

### `chart/Chart.yaml` — Sub-chart Dependencies

```yaml
apiVersion: v2
name: retail-store-sample-chart
version: 1.5.0
dependencies:
  - name: retail-store-sample-cart-chart
    alias: cart
    version: 1.5.0
    repository: file://../../cart/chart
  - name: retail-store-sample-catalog-chart
    alias: catalog
    version: 1.5.0
    repository: file://../../catalog/chart
  - name: retail-store-sample-checkout-chart
    alias: checkout
    version: 1.5.0
    repository: file://../../checkout/chart
  - name: retail-store-sample-orders-chart
    alias: orders
    version: 1.5.0
    repository: file://../../orders/chart
  - name: retail-store-sample-ui-chart
    alias: ui
    version: 1.5.0
    repository: file://../../ui/chart
```

All five sub-charts are referenced as local `file://` repositories. The `alias` value is the key used to scope values in `values.yaml` (e.g., `cart:`, `ui:`, `orders:`). `helm dependency build` must be run before `helm install`.

### `chart/values.yaml` — Base Wiring

```yaml
cart:
  fullnameOverride: carts
catalog:
  fullnameOverride: catalog
checkout:
  fullnameOverride: checkout
  retail:
    checkout:
      endpoints:
        orders: http://orders:80
orders:
  fullnameOverride: orders
ui:
  fullnameOverride: ui
  app:
    endpoints:
      catalog: http://catalog:80
      carts:   http://carts:80
      checkout: http://checkout:80
      orders:  http://orders:80
```

`fullnameOverride` pins the Kubernetes resource name for each service to the short alias (e.g., `carts`, not `retail-store-sample-chart-carts`). Without this, in-cluster DNS names would not match the hardcoded endpoint URLs.

### `chart/values-stateful.yaml` — Stateful Overlay

Enables in-cluster DB/broker for all four backend services:

| Service | Provider | Helm key | Component created |
|---------|----------|----------|-------------------|
| `catalog` | `mysql` | `catalog.mysql.create: true` | MySQL StatefulSet |
| `cart` | `dynamodb` | `cart.dynamodb.create: true` | DynamoDB local pod |
| `checkout` | `redis` | `checkout.redis.create: true` | Redis StatefulSet |
| `orders` | `postgres` + `rabbitmq` | `orders.postgresql.create: true` + `orders.rabbitmq.create: true` | PostgreSQL + RabbitMQ StatefulSets |

Applied as a second `-f` file on top of `values.yaml`:

```bash
helm dependency build src/app/chart
helm install retail src/app/chart -f src/app/chart/values-stateful.yaml
```

Without `-f values-stateful.yaml`, all services start in stateless/in-memory mode — no persistent components are created.

---

## Helmfile Orchestration

### `helmfile.yaml` — Full Stack (5 services)

Deploys all five services in dependency order using `wait: true` per release:

| Release | Chart | Notable values |
|---------|-------|----------------|
| `catalog` | `../catalog/chart` | `app.persistence.provider: mysql`, `mysql.create: true` |
| `carts` | `../cart/chart` | `app.persistence.provider: dynamodb`, `dynamodb.create: true` |
| `orders` | `../orders/chart` | `app.persistence.provider: postgres`, `app.messaging.provider: rabbitmq`, `postgresql.create: true`, `rabbitmq.create: true` |
| `checkout` | `../checkout/chart` | `app.persistence.provider: redis`, `redis.create: true`, `app.endpoints.orders: http://orders:80` |
| `ui` | `../ui/chart` | All 4 endpoint URLs; `timeout: 600` |

Each release merges two or three value sources in order:
1. `templates/components.yaml.gotmpl` — optional image tag/repo override
2. `templates/<service>.yaml.gotmpl` — Tilt mode and secret injection
3. Inline `app:` block — persistence/messaging provider + `create: true` flags

### `helmfile.slim.yaml` — Minimal (catalog + ui)

Deploys only catalog and ui — no DB, no broker, no cart, no orders, no checkout. Used for quick demos or smoke tests:

```yaml
releases:
  - name: catalog
    chart: ../catalog/chart
    values:
      - templates/components.yaml.gotmpl
      - templates/catalog.yaml.gotmpl

  - name: ui
    chart: ../ui/chart
    timeout: 600
    values:
      - templates/components.yaml.gotmpl
      - templates/ui.yaml.gotmpl
      - templates/ingress.yaml.gotmpl
      - retail:
          ui:
            endpoints:
              catalog: http://catalog
```

### Go Template System (`templates/*.gotmpl`)

All `.gotmpl` files are evaluated by Helmfile at deploy time using `{{ env "VAR" }}` syntax. Each resolves to a YAML fragment merged into the release values.

| Template | Env var(s) read | Effect |
|----------|----------------|--------|
| `components.yaml.gotmpl` | `IMAGE_TAG`, `IMAGE_REPOSITORY` | Overrides `image.tag` and `image.repository` for all releases |
| `ingress.yaml.gotmpl` | `INGRESS`, `NODE_PORT`, `LOAD_BALANCER` | Enables ingress, NodePort, or LoadBalancer on UI |
| `catalog.yaml.gotmpl` | `TILT_MODE`, `RANDOM_PASSWORD` | Sets `image.tag: dev` + MySQL password in Tilt |
| `orders.yaml.gotmpl` | `TILT_MODE`, `RANDOM_PASSWORD` | Sets `image.tag: dev` + PostgreSQL password in Tilt |
| `carts.yaml.gotmpl` | `TILT_MODE` | Sets `image.tag: dev` in Tilt |
| `checkout.yaml.gotmpl` | `TILT_MODE` | Sets `image.tag: dev` in Tilt |
| `ui.yaml.gotmpl` | `TILT_MODE` | Sets `image.tag: dev` in Tilt |

**Common deploy commands:**

```bash
# Full stack — stateful in-cluster components
helmfile -f src/app/helmfile.yaml sync

# Custom image tag
IMAGE_TAG=v1.5.0 helmfile -f src/app/helmfile.yaml sync

# Custom image registry
IMAGE_REPOSITORY=123456789012.dkr.ecr.us-east-1.amazonaws.com/retail-ui \
  helmfile -f src/app/helmfile.yaml sync

# NodePort exposure on UI
NODE_PORT=30080 helmfile -f src/app/helmfile.yaml sync

# Minimal demo — catalog + ui only
helmfile -f src/app/helmfile.slim.yaml sync

# Teardown
helmfile -f src/app/helmfile.yaml destroy
```

---

## Docker Compose Aggregator

### `docker-compose.yml` — Multi-file Include

```yaml
include:
  - ../ui/docker-compose.yml
  - ../catalog/docker-compose.yml
  - ../cart/docker-compose.yml
  - ../checkout/docker-compose.yml
  - ../orders/docker-compose.yml
```

Each service's own `docker-compose.yml` defines its containers and dependencies. The aggregator composes them into a single stack. Run from `src/app/`:

```bash
docker compose up
```

### `compose.override.yaml` — Endpoint Wiring

Automatically applied alongside `docker-compose.yml` when `docker compose` is run from `src/app/`. Key behaviours:

- Injects all four `RETAIL_UI_ENDPOINTS_*` env vars into the `ui` container pointing at the compose service hostnames (`http://catalog:8080`, etc.).
- Adds `depends_on` with `condition: service_healthy` + `restart: true` for all four backend services — UI only starts after all backends pass health checks.
- Overrides `ports` on all backend services to `[]` — only UI is port-bound to the host. All inter-service traffic flows over the internal Docker network.
- Wires `RETAIL_CHECKOUT_ENDPOINTS_ORDERS=http://orders:8080` into the checkout container.

### `docker-compose.tracing.yml` — Distributed Tracing

Adds two containers to the stack:
- **OTEL Collector** (`otelcol`): receives traces from all services on `:4318` (OTLP HTTP), forwards to Jaeger.
- **Jaeger** (`jaeger`): trace storage and UI on `:16686`.

```bash
docker compose -f docker-compose.yml -f docker-compose.tracing.yml up
```

### `otel-collector-config.yml`

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
exporters:
  otlphttp/jaeger:
    endpoint: http://jaeger:4318
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlphttp/jaeger]
```

All application services have `otel.sdk.disabled: true` in their `application.yml` by default — tracing is off unless the OTEL collector is present and the SDK is re-enabled via environment variable.

---

## Tilt Local Dev Loop

`Tiltfile` wires Helmfile templating with local Docker builds for all five services:

```python
random_password = local("bash scripts/generate_password.sh")

def helmfile(file):
  update_env = {'TILT_MODE': '1', 'RANDOM_PASSWORD': random_password}
  return local("helmfile -f %s template --skip-tests" % file, env=update_env)

k8s_yaml(helmfile("./helmfile.yaml"))      # Renders helmfile → k8s YAML

docker_build('retail-store-sample-ui',       '../ui')
docker_build('retail-store-sample-cart',     '../cart')
docker_build('retail-store-sample-orders',   '../orders')
docker_build('retail-store-sample-checkout', '../checkout')
docker_build('retail-store-sample-catalog',  '../catalog')

k8s_resource('ui', port_forwards='8888:8080')  # UI accessible at localhost:8888
```

**`scripts/generate_password.sh`** generates a 20-char random password on first run and caches it in `.random_password`. Subsequent runs reuse the cached value — MySQL and PostgreSQL passwords remain stable across Tilt restarts.

**Flow:**
1. `TILT_MODE=1` causes all `*.yaml.gotmpl` files to set `image.tag: dev` and `image.repository: retail-store-sample-<service>`.
2. `docker_build` watches each service's source directory and rebuilds the local image on file change.
3. Tilt applies the updated deployment to the local cluster via the rendered helmfile YAML.
4. UI is port-forwarded to `localhost:8888`.

---

## Deployment Mode Reference

| Mode | Tool | Command | DB/Broker | Image source |
|------|------|---------|-----------|-------------|
| Local dev (hot-reload) | Tilt | `tilt up` | In-cluster (via `helmfile.yaml`) | Local `docker_build` |
| Full-stack Kubernetes | Helmfile | `helmfile -f helmfile.yaml sync` | In-cluster | ECR public image |
| Minimal smoke test | Helmfile | `helmfile -f helmfile.slim.yaml sync` | None (stateless) | ECR public image |
| Umbrella Helm install | Helm | `helm install retail src/app/chart` | None (base values) | ECR public image |
| Umbrella Helm stateful | Helm | `helm install retail src/app/chart -f src/app/chart/values-stateful.yaml` | In-cluster | ECR public image |
| Local Docker | Compose | `docker compose up` | Each service's own containers | ECR public image |
| Local Docker + tracing | Compose | `docker compose -f docker-compose.yml -f docker-compose.tracing.yml up` | + OTEL + Jaeger | ECR public image |
