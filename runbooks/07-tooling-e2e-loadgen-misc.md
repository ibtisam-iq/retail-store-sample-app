# 07 — Tooling: `e2e`, `load-generator`, `misc`

Three support directories inside `src/` that contain no application services. Each provides a distinct testing or code-quality function.

---

## `src/e2e` — End-to-End Tests

> **Runtime:** Node.js ≥ 20 / Yarn 4
> **Framework:** Cypress 13.17.0 (headless Electron browser)
> **Scope:** Black-box — treats the entire stack as a single HTTP endpoint

### Directory Layout

```
src/e2e/
├── cypress.config.js           # Base URL default: http://localhost:8080
├── package.json                # devDependency: cypress ^13.17.0
├── Dockerfile.run              # FROM cypress/included:13.17.0 (zero-install runner)
├── scripts/
│   ├── run-docker.sh           # Builds runner image + runs container against endpoint
│   └── assert-traces.sh        # Queries Jaeger API and asserts distributed trace spans
└── cypress/
    ├── e2e/                    # Test specs (one per feature area)
    │   ├── home.cy.js
    │   ├── catalog.cy.js
    │   ├── cart.cy.js
    └─── checkout.cy.js
    ├── pages/                  # Page Object Model — one class per UI page/step
    │   ├── Page.js               # Base page
    │   ├── Home.js
    │   ├── Catalog.js
    │   ├── Product.js
    │   ├── Cart.js
    │   ├── Checkout.js
    │   ├── CheckoutAddress.js
    │   ├── CheckoutShipping.js
    │   ├── CheckoutPayment.js
    │   └── CheckoutOrder.js
    ├── fixtures/
    └── support/
```

### Test Specs

| Spec | Feature area covered |
|------|---------------------|
| `home.cy.js` | Homepage load, navigation elements |
| `catalog.cy.js` | Product listing, product detail page |
| `cart.cy.js` | Add to cart, cart view, quantity/removal |
| `checkout.cy.js` | Full checkout flow: address → shipping → payment → order confirmation |

All specs follow the **Page Object Model** (POM) pattern — test logic lives in spec files, DOM selectors and interactions live in `cypress/pages/`. The checkout flow is the most comprehensive: it exercises UI → Catalog → Cart → Checkout → Orders in a single browser session.

### Running

**Docker (recommended — no local Cypress install needed):**

```bash
# Against any running endpoint
bash src/e2e/scripts/run-docker.sh 'http://<ui-endpoint>:8080'

# Against local Docker Compose stack (ui service on internal network)
bash src/e2e/scripts/run-docker.sh --network docker-compose_default 'http://ui:8080'
```

`run-docker.sh` builds `retail-store-sample-e2e:run` from `Dockerfile.run` (`FROM cypress/included:13.17.0`), mounts `$PWD` into `/e2e`, and passes `CYPRESS_BASE_URL` as an env var.

| Flag | Default | Description |
|------|---------|-------------|
| `-n, --network` | `bridge` | Docker network to join |
| `-v, --verbose` | off | `set -x` debug output |
| `-h, --help` | — | Usage |

**Yarn (local, requires Node ≥ 20):**

```bash
cd src/e2e
yarn
CYPRESS_BASE_URL='http://<endpoint>:8080' yarn cypress run
```

### `assert-traces.sh` — Jaeger Span Assertions

Validates that distributed tracing is working correctly by querying the Jaeger HTTP API and asserting the presence of specific spans.

```bash
JAEGER_API=http://localhost:16686/api LOOKBACK=10m bash src/e2e/scripts/assert-traces.sh
```

| Assertion | Service | What it verifies |
|-----------|---------|------------------|
| `GET /catalog` span | `ui` | UI instrumented the catalog page request |
| `/catalog/products` span | `catalog` | Trace propagated from UI to Catalog service |
| `db.statement` SQL span | `catalog` | MySQL query traced via OTEL |
| `GET /carts/{customerId}` span | `cart` | Trace propagated to Cart service |
| `aws.table.name = Items` span | `cart` | DynamoDB operation traced |
| `GET /checkout` span | `ui` | UI instrumented the checkout page request |
| `POST /checkout/:customerId/update` span | `checkout` | Trace propagated to Checkout service |
| `set` + `db.system = redis` span | `checkout` | Redis session write traced |

Returns exit code `0` if all assertions pass, `1` if any fail.

---

## `src/load-generator` — Synthetic Load Generator

> **Runtime:** Node.js / Yarn
> **Engine:** [Artillery](https://github.com/artilleryio/artillery) 2.0.22
> **Purpose:** Generates realistic traffic for autoscaling, observability, and resiliency testing

### Directory Layout

```
src/load-generator/
├── scenario.yml                # Artillery scenario: full user journey
├── helpers.js                  # Custom Artillery functions: product ID pool
├── Dockerfile                  # amazonlinux:2023-minimal, copies scenario + helpers
├── Dockerfile.run              # Artillery runner image
└── scripts/
    ├── run-docker.sh           # Local Docker runner with full flag set
    └── run-test.sh             # Smoke test: 30s run, asserts zero failures + HTTP 200s
```

### `scenario.yml` — Traffic Pattern

Simulates a complete user journey across all five services:

```
beforeScenario: getAllProducts   → fetches product ID list from helpers.js

GET  /home
GET  /catalog
GET  /catalog/{productId}        → loops over all products
POST /cart                       → adds a random product (setRandomProductId)
GET  /cart
GET  /checkout
POST /checkout                   → submits address form
POST /checkout/delivery          → selects shipping method
POST /checkout/payment           → submits payment form
```

This exercises UI → Catalog → Cart → Checkout → Orders in a single virtual user session. Nine hardcoded product UUIDs are defined in `helpers.js` — they map to the seeded product data in the catalog service.

### `run-docker.sh` — Local Runner Flags

| Flag | Default | Description |
|------|---------|-------------|
| `-t, --target` | `http://localhost:8888` | UI endpoint URL |
| `--vu` | `1` | Number of virtual users (arrival rate) |
| `-d, --duration` | `0` (forever) | Test duration in seconds |
| `-n, --network` | `bridge` | Docker network to join |
| `-q, --quiet` | off | Reduce Artillery log output |
| `-o, --output` | none | Path to save JSON output file |

```bash
# Basic run against local Compose stack
bash src/load-generator/scripts/run-docker.sh -t http://localhost:8888

# 5 virtual users, 60 seconds, against Docker Compose
bash src/load-generator/scripts/run-docker.sh \
  --network docker-compose_default \
  -t http://ui:8080 \
  --vu 5 -d 60

# Save results to file
bash src/load-generator/scripts/run-docker.sh \
  -t http://localhost:8888 -d 30 -o ./results.json
```

`run-docker.sh` builds `retail-store-sample-loadgen:run` from `Dockerfile.run`, then injects `--overrides` to Artillery to dynamically set `duration` and `arrivalRate` from the CLI flags — no manual edits to `scenario.yml` required.

### `run-test.sh` — Smoke Test

Runs a 30-second load test against the Docker Compose stack and asserts three conditions via `jq`:

| Assertion | Pass condition |
|-----------|---------------|
| `vusers.failed == 0` | No virtual user journeys failed |
| `http.codes.200 > 0` | At least one HTTP 200 response received |
| `http.codes.404 == 0` | No 404 responses |

```bash
bash src/load-generator/scripts/run-test.sh
```

Exits `0` if all pass, `1` on first failed assertion.

### Kubernetes Pod Pattern

The recommended production pattern uses an `initContainer` to copy the scenario files into a shared `emptyDir` volume, then runs Artillery in the main container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: load-generator
spec:
  containers:
  - name: artillery
    image: artilleryio/artillery:2.0.22
    args: ["run", "-t", "http://ui.ui.svc", "/scripts/scenario.yml"]
    volumeMounts:
    - name: scripts
      mountPath: /scripts
  initContainers:
  - name: setup
    image: public.ecr.aws/aws-containers/retail-store-sample-utils:load-gen.1.2.1
    command: ["bash", "-c", "cp /artillery/* /scripts"]
    volumeMounts:
    - name: scripts
      mountPath: /scripts
  volumes:
  - name: scripts
    emptyDir: {}
```

The `initContainer` image (`retail-store-sample-utils:load-gen.*`) is built from `Dockerfile` — it contains only `scenario.yml` and `helpers.js` under `/artillery/`. The version tag must match the application version being tested.

---

## `src/misc` — Shared Code-Style Configuration

```
src/misc/
└── style/
    └── java/
        └── checkstyle.xml      # Google Java Style Guide — Checkstyle configuration
```

`misc/` contains only one file: `checkstyle.xml`. It defines the **Google Java Style Guide** rules enforced by Checkstyle across all Java services (Catalog, Cart, Orders, UI). Each Java service's `pom.xml` references this file via the Maven Checkstyle plugin using a relative path (`../../misc/style/java/checkstyle.xml`).

| Aspect | Detail |
|--------|--------|
| Standard | Google Java Style Guide |
| Tool | Checkstyle (Maven plugin: `maven-checkstyle-plugin`) |
| Enforcement | Build-time — `mvn validate` or `mvn checkstyle:check` fails the build on violations |
| Scope | All Java services: catalog, cart, orders, ui |
| File | `checkstyle.xml` (5,978 bytes — full ruleset) |

No other files or configuration exist in `src/misc/`. It serves as a single shared location for cross-service code-style rules to avoid duplication across four separate `pom.xml` files.

---

## Tooling Summary

| Directory | Tool | Primary Use | Trigger |
|-----------|------|-------------|--------|
| `src/e2e` | Cypress 13 | Black-box browser tests across full stack | Post-deploy validation |
| `src/e2e/scripts/assert-traces.sh` | Bash + `jq` + Jaeger API | Distributed trace span assertions | After OTEL-enabled deploy |
| `src/load-generator` | Artillery 2 | Synthetic load for autoscaling/observability | Performance / chaos testing |
| `src/load-generator/scripts/run-test.sh` | Bash + `jq` | Smoke test: zero failures + HTTP 200 | CI pipeline gate |
| `src/misc/style/java/checkstyle.xml` | Checkstyle | Google Java Style enforcement | `mvn validate` at build time |
