# 01 — Catalog Service Deep Dive

> **Location:** `src/catalog/`  
> **Language:** Go  
> **Framework:** Gin (HTTP router)  
> **Database:** MySQL 8.0 (in-cluster StatefulSet) **or** external MySQL/MariaDB **or** in-memory (no DB)  
> **Container Port:** `8080`  
> **Public Image:** `public.ecr.aws/aws-containers/retail-store-sample-catalog`

---

## What This Service Does

The Catalog service is a standalone REST API that serves product data to the rest of the retail application. It is the only service the UI calls when a user browses products, views a product detail page, or filters by tag. It has no dependency on any other microservice — it only needs a database (or none, in in-memory mode).

---

## Source Tree

```
src/catalog/
├── main.go                  # Entry point — wires everything together
├── Dockerfile               # Multi-stage: Amazon Linux 2023 build → slim runtime
├── docker-compose.yml       # Local dev stack: catalog + MariaDB 10.9
├── go.mod / go.sum          # Go module dependencies
├── openapi.yml              # OpenAPI 3.0 spec for the catalog API
├── project.json             # Nx project definition (build/test/push targets)
├── .dockerignore            # Excludes test/, docs, *.md from Docker context
├── .gitignore               # Ignores the compiled binary
├── ATTRIBUTION.md           # OSS license attribution (~600KB)
│
├── config/
│   └── config.go            # Reads all env vars → AppConfiguration struct
│
├── api/                     # CatalogAPI interface + implementation
├── controller/              # Gin route handlers (GetProducts, GetProduct, etc.)
├── repository/              # DB abstraction: in-memory vs MySQL drivers
├── model/                   # Product, Tag data structs
├── middleware/              # Chaos middleware (controllable failure injection)
├── httputil/                # Shared HTTP response helpers
├── scripts/                 # seed-data.sh (populates DB on first run)
├── test/                    # Integration test helpers
│
└── chart/                   # Helm chart — covered in full below
```

---

## Application Runtime

### Entry Point — `main.go`

`main.go` is the entire wiring layer. It:

1. Checks for `OTEL_SERVICE_NAME` — if present, initializes an OTLP trace exporter (HTTP) using AWS X-Ray ID generator and EC2 resource detector.
2. Reads config from env vars via `go-envconfig`.
3. Creates the repository (in-memory or MySQL depending on `RETAIL_CATALOG_PERSISTENCE_PROVIDER`).
4. Builds the CatalogAPI, Controller, and mounts Gin routes.
5. Registers Prometheus metrics middleware (`/metrics` endpoint).
6. Registers a chaos controller (can be toggled via `POST /chaos/...` routes to simulate failures).
7. Starts the HTTP server on the configured port and handles graceful shutdown on `SIGTERM`/`SIGINT`.

### API Routes

| Method | Path | Handler | Description |
|--------|------|---------|-------------|
| `GET` | `/catalog/products` | `GetProducts` | List all products (paginated, filterable by tag) |
| `GET` | `/catalog/products/:id` | `GetProduct` | Fetch a single product by ID |
| `GET` | `/catalog/tags` | `ListTags` | List all available product tags |
| `GET` | `/catalog/size` | `CatalogSize` | Returns total product count |
| `GET` | `/health` | inline | Returns `200 OK` (skipped in access logs) |
| `GET` | `/topology` | inline | Returns persistence provider + DB endpoint as JSON |
| `GET` | `/metrics` | Prometheus | Prometheus metrics scrape endpoint |
| `POST` | `/chaos/...` | ChaosController | Trigger simulated failures for resilience testing |

### Config — `config/config.go`

All configuration is injected via **environment variables**. There are no config files at runtime.

| Env Var | Default | Purpose |
|---------|---------|--------|
| `PORT` | `8080` | HTTP listen port |
| `RETAIL_CATALOG_PERSISTENCE_PROVIDER` | `in-memory` | `in-memory` or `mysql` |
| `RETAIL_CATALOG_PERSISTENCE_ENDPOINT` | _(empty)_ | `host:port` of MySQL server |
| `RETAIL_CATALOG_PERSISTENCE_DB_NAME` | `catalogdb` | Database name |
| `RETAIL_CATALOG_PERSISTENCE_USER` | `catalog_user` | DB username |
| `RETAIL_CATALOG_PERSISTENCE_PASSWORD` | _(empty)_ | DB password |
| `RETAIL_CATALOG_PERSISTENCE_CONNECT_TIMEOUT` | `5` | TCP connect timeout (seconds) |
| `OTEL_SERVICE_NAME` | _(unset)_ | If set, enables OpenTelemetry tracing |
| `GIN_MODE` | `debug` | Set to `release` in production |

> **Key insight:** The `RETAIL_CATALOG_PERSISTENCE_PROVIDER` env var is the single toggle that switches the entire persistence layer. When set to `in-memory`, the service needs zero external dependencies and boots instantly with seeded product data.

### Dockerfile — Build Strategy

The Dockerfile uses a **two-stage build** on Amazon Linux 2023:

| Stage | Base | What Happens |
|-------|------|-------------|
| `build-env` | `amazonlinux:2023` | Installs Go, downloads modules, compiles `main.go` → `/appsrc/main` binary |
| Runtime | `amazonlinux:2023` (fresh) | Creates `appuser` (UID 1000), copies binary only, sets `ENTRYPOINT` |

The runtime image contains no Go toolchain — just the compiled binary. This keeps the final image small and reduces attack surface. The `GIN_MODE=release` env var is baked into the image so Gin's debug logging is off by default.

### `docker-compose.yml` — Local Dev Stack

Runs two containers locally:

| Container | Image | Notes |
|-----------|-------|-------|
| `catalog` | Built from `./Dockerfile` | `RETAIL_CATALOG_PERSISTENCE_PROVIDER=mysql`, depends on `catalog-db` health |
| `catalog-db` | `mariadb:10.9` | DB = `catalogdb`, user = `catalog_user`, password from `$DB_PASSWORD` |

The compose file uses `depends_on.condition: service_healthy` — the catalog container will not start until MariaDB passes its `mysqladmin ping` health check. This mirrors Kubernetes readiness probe behaviour.

### `openapi.yml`

A full OpenAPI 3.0 spec documenting all catalog endpoints — request parameters, response schemas, and example objects for `Product` and `Tag`. Useful for understanding the exact API contract before writing integration tests or calling it from another service.

---

## Helm Chart

### Chart Files

```
src/catalog/chart/
├── Chart.yaml                       # apiVersion: v2, name: catalog, type: application
├── .helmignore                      # Excludes *.md, tests/ from packaged chart
├── values.yaml                      # Master defaults — every knob the chart exposes
├── values-in-memory.yaml            # Scenario: no DB
├── values-mysql-ephemeral.yaml      # Scenario: in-cluster MySQL, no PVC
├── values-mysql-pvc-eks.yaml        # Scenario: in-cluster MySQL + PVC on EKS (gp2)
├── values-mysql-pvc-baremetal.yaml  # Scenario: in-cluster MySQL + PVC on bare-metal (local-path)
├── values-external-mysql.yaml       # Scenario: point to external MySQL
├── values-hpa.yaml                  # Scenario: HPA enabled
├── values-pdb.yaml                  # Scenario: PDB enabled
├── catalog-chart-runbook.md         # Helm command reference for each scenario
│
└── templates/
    ├── _helpers.tpl                 # All named templates (labels, fullname, endpoint logic)
    ├── deployment.yaml              # Catalog app Deployment
    ├── service.yaml                 # ClusterIP service on port 80 → container 8080
    ├── serviceaccount.yaml          # Optional ServiceAccount
    ├── configmap.yml                # RETAIL_CATALOG_PERSISTENCE_PROVIDER + endpoint
    ├── secret.yaml                  # DB username + password (base64)
    ├── hpa.yaml                     # HorizontalPodAutoscaler (conditional)
    ├── pdb.yaml                     # PodDisruptionBudget (conditional)
    ├── mysql-statefulset.yaml       # MySQL StatefulSet (conditional)
    ├── mysql-service.yaml           # MySQL ClusterIP service on 3306 (conditional)
    ├── security-group.yaml          # AWS SecurityGroupPolicy CRD (conditional)
    └── tests/                       # helm test hook (connectivity check)
```

---

## `values.yaml` — Annotated Deep Dive

`values.yaml` is the **single source of truth** for everything the chart can do. Understanding it is the prerequisite for all the scenario files. The file is divided into logical sections.

### Section 1 — Pod & Image

```yaml
replicaCount: 1

image:
  repository: public.ecr.aws/aws-containers/retail-store-sample-catalog
  pullPolicy: IfNotPresent
  tag:          # Empty — falls back to .Chart.Version in deployment.yaml
```

`replicaCount` is only used when HPA is **disabled**. The deployment template has:
```yaml
{{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
{{- end }}
```
So setting `replicaCount: 3` when HPA is on has no effect — HPA owns the replica count.

The `image.tag` field is intentionally blank so the chart auto-uses the chart's own version as the image tag when nothing is specified.

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

The security context matches the Dockerfile runtime user (`appuser`, UID 1000). `readOnlyRootFilesystem: true` is enforced — the container cannot write to any filesystem path except the `/tmp` volume mounted as `emptyDir` (medium: Memory). This is a real production hardening decision, not a placeholder.

`podSecurityContext.fsGroup: 1000` ensures mounted volumes are owned by the app user group, which matters for PVC mounts.

---

### Section 3 — Service & Resources

```yaml
service:
  type: ClusterIP
  port: 80        # External port — maps to containerPort 8080

resources:
  limits:
    memory: 256Mi
  requests:
    cpu: 256m
    memory: 256Mi
```

The service exposes port 80, but the container listens on 8080. The mapping happens in `service.yaml`. There is **no CPU limit** — only a CPU request. This is intentional: setting a CPU limit on a Go service can cause throttling even when the node has free capacity.

---

### Section 4 — Autoscaling & Scheduling

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
```

When `autoscaling.enabled: true`, Helm renders `hpa.yaml` and the Deployment omits the `replicas:` field entirely (leaving HPA in full control). `topologySpreadConstraints` is an advanced field for spreading pods across zones or nodes — it maps directly to the Pod spec's `topologySpreadConstraints` array.

---

### Section 5 — Metrics

```yaml
metrics:
  enabled: true
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

When `metrics.enabled: true`, these annotations are merged into the pod's annotation block (via the `catalog.podAnnotations` helper in `_helpers.tpl`). Prometheus will automatically discover and scrape this pod. The scrape port (`8080`) is the container port, not the service port (`80`).

---

### Section 6 — ConfigMap

```yaml
configMap:
  create: true
  name:           # Empty = auto-generated from release name
```

The ConfigMap carries the non-secret env vars: `RETAIL_CATALOG_PERSISTENCE_PROVIDER` and (if MySQL) `RETAIL_CATALOG_PERSISTENCE_ENDPOINT` and `RETAIL_CATALOG_PERSISTENCE_DB_NAME`. The deployment template mounts this ConfigMap via `envFrom.configMapRef`. If you set `configMap.create: false` and provide a `name`, the chart uses a pre-existing ConfigMap (useful for GitOps where secrets/configs are managed externally).

---

### Section 7 — `app.persistence` (The Core Toggle)

```yaml
app:
  persistence:
    provider: in-memory     # "in-memory" | "mysql"
    endpoint: ""            # Only used when mysql.create: false (external DB)
    database: "catalog"     # DB name passed to MySQL and the app

    secret:
      create: true
      name: catalog-db
      username: catalog
      password: ""          # Auto-generated if blank (see _helpers.tpl)
```

This section is the heart of the chart. `app.persistence.provider` controls two things simultaneously:

1. **ConfigMap value** — sets `RETAIL_CATALOG_PERSISTENCE_PROVIDER` env var in the app container.
2. **Secret mounting** — the deployment template only mounts the DB secret via `envFrom.secretRef` when `provider == "mysql"`:
   ```yaml
   {{- if (eq "mysql" .Values.app.persistence.provider) }}
   - secretRef:
       name: {{ .Values.app.persistence.secret.name }}
   {{- end }}
   ```

The `app.persistence.endpoint` field is only used when `mysql.create: false` (i.e., you want to point to an external DB). When `mysql.create: true`, the endpoint is automatically computed by `_helpers.tpl`:
```
{{- define "catalog.mysql.endpoint" -}}
{{- if .Values.mysql.create -}}
{{ include "catalog.mysql.fullname" . }}:{{ .Values.mysql.service.port }}
{{- else }}
{{- .Values.app.persistence.endpoint -}}
{{- end -}}
{{- end -}}
```
So `app.persistence.endpoint` is ignored entirely if an in-cluster MySQL is created. You must set `mysql.create: false` AND populate `app.persistence.endpoint` for external DB mode to work.

**Password auto-generation:** If `app.persistence.secret.password` is left blank, `_helpers.tpl` checks if a secret named `catalog-db` already exists in the namespace. If it does, it reuses the existing password (idempotent upgrades). If not, it generates a 16-character random alphanumeric string and base64-encodes it. This means `helm upgrade` will not rotate your DB password unexpectedly.

---

### Section 8 — `mysql` (In-Cluster MySQL)

```yaml
mysql:
  create: false           # Master switch — set true to deploy MySQL StatefulSet

  image:
    repository: public.ecr.aws/docker/library/mysql
    tag: "8.0"
    pullPolicy: IfNotPresent

  service:
    type: ClusterIP
    port: 3306

  podAnnotations: {}
  nodeSelector: {}
  tolerations: []
  affinity: {}

  persistentVolume:
    enabled: false          # false = emptyDir (data lost on pod restart)
    annotations: {}
    labels: {}
    accessModes:
      - ReadWriteOnce
    size: 10Gi
    storageClass: ""        # Empty = use cluster default StorageClass
```

`mysql.create` is the gate for both `mysql-statefulset.yaml` and `mysql-service.yaml`. Both templates begin with `{{- if .Values.mysql.create }}`. Setting this to `true` without also setting `app.persistence.provider: mysql` would create a MySQL pod that the app never connects to.

The `persistentVolume` sub-section controls whether the StatefulSet uses a `volumeClaimTemplate` (persistent data) or an `emptyDir` volume (ephemeral). The template logic:

```yaml
{{- if .Values.mysql.persistentVolume.enabled }}
  volumeClaimTemplates:
    ...
{{- else }}
      volumes:
      - name: data
        emptyDir: {}
{{- end }}
```

`storageClass: ""` (empty string) tells Kubernetes to use the cluster's default StorageClass. Setting it to `"-"` explicitly sets `storageClassName: ""` which disables dynamic provisioning and requires a manually pre-provisioned PV. Any other string (e.g., `gp2`, `local-path`) names a specific StorageClass.

---

### Section 9 — AWS & Observability

```yaml
securityGroups:
  create: false
  securityGroupIds: []

opentelemetry:
  enabled: false
  instrumentation: ""

podDisruptionBudget:
  enabled: false
  minAvailable: 2
  maxUnavailable: 1
```

`securityGroups` creates an AWS `SecurityGroupPolicy` CRD resource — this is EKS-specific and requires the VPC CNI plugin. It is irrelevant for bare-metal or non-AWS clusters.

`opentelemetry.instrumentation` holds the name of an `Instrumentation` CRD resource (from the OpenTelemetry Operator). When set, the annotation `instrumentation.opentelemetry.io/inject-sdk` is added to the pod, triggering automatic SDK injection.

`podDisruptionBudget.minAvailable` and `maxUnavailable` are both defined in values but the `pdb.yaml` template only uses `minAvailable`. The `maxUnavailable` field is a placeholder — check `pdb.yaml` before using it.

---

## How the Scenario Values Files Override `values.yaml`

Each `values-*.yaml` file is a **partial override** — it only specifies the keys that differ from `values.yaml`. Helm performs a deep merge at install time: keys not mentioned in the override file keep their `values.yaml` defaults.

### Values Files at a Glance

| File | `app.persistence.provider` | `mysql.create` | `mysql.persistentVolume.enabled` | Use Case |
|------|-----------------------------|----------------|----------------------------------|----------|
| `values.yaml` (default) | `in-memory` | `false` | `false` | Baseline |
| `values-in-memory.yaml` | `in-memory` | `false` | `false` | Explicit no-DB test |
| `values-mysql-ephemeral.yaml` | `mysql` | `true` | `false` | Dev/test — data lost on pod restart |
| `values-mysql-pvc-eks.yaml` | `mysql` | `true` | `true` (gp2) | EKS with EBS persistent storage |
| `values-mysql-pvc-baremetal.yaml` | `mysql` | `true` | `true` (local-path) | Bare-metal Kubernetes with local storage |
| `values-external-mysql.yaml` | `mysql` | `false` | N/A | Point to RDS or any external MySQL |
| `values-hpa.yaml` | `in-memory` | `false` | `false` | HPA on, min 2 / max 5 replicas |
| `values-pdb.yaml` | `in-memory` | `false` | `false` | PDB on, 3 replicas, minAvailable 2 |

### How the Merge Works — A Concrete Example

`values-mysql-ephemeral.yaml` contains only:
```yaml
app:
  persistence:
    provider: mysql
    secret:
      password: "catalog123"
mysql:
  create: true
```

After Helm merges this with `values.yaml`, the effective configuration is:
- `app.persistence.provider` → `mysql` ✅ (overridden)
- `mysql.create` → `true` ✅ (overridden)
- `mysql.persistentVolume.enabled` → `false` (kept from `values.yaml` default)
- `app.persistence.database` → `catalog` (kept from `values.yaml` default)
- `replicaCount` → `1` (kept from `values.yaml` default)
- Everything else → unchanged defaults

This is why you only need to specify what changes — the rest inherits automatically.

---

## Template Engine Internals

### `_helpers.tpl` — Named Templates

All reusable logic lives here. Key helpers:

| Template | Purpose |
|----------|---------|
| `catalog.fullname` | Release-name-aware resource name (truncated to 63 chars) |
| `catalog.labels` | Standard labels: `helm.sh/chart`, `app.kubernetes.io/*` |
| `catalog.selectorLabels` | Pod selector: name, instance, component=service, owner |
| `catalog.mysql.fullname` | `<fullname>-mysql` — used for StatefulSet + Service names |
| `catalog.mysql.endpoint` | Returns internal DNS (`<mysql-fullname>:3306`) or `app.persistence.endpoint` |
| `catalog.persistence.password` | Looks up existing secret OR generates random password |
| `catalog.podAnnotations` | Merges `podAnnotations` + `metrics.podAnnotations` |
| `getOrGeneratePass` | Generic helper: looks up existing k8s secret, generates random if absent |

The `catalog.mysql.endpoint` helper is the key link between `mysql.create` and `app.persistence.endpoint`:
- `mysql.create: true` → returns `<release>-catalog-mysql:3306` (internal DNS)
- `mysql.create: false` → returns `app.persistence.endpoint` (whatever you set)

### `deployment.yaml` — Key Behaviours

- **Replicas**: omitted entirely if `autoscaling.enabled: true`
- **Strategy**: always `RollingUpdate` with `maxUnavailable: 1`
- **envFrom**: always mounts the ConfigMap; only mounts the Secret when `provider == "mysql"`
- **Volume**: always mounts `/tmp` as `emptyDir (medium: Memory)` — required because `readOnlyRootFilesystem: true` prevents the app from writing to the container filesystem

### `secret.yaml`

Stores `RETAIL_CATALOG_PERSISTENCE_USER` and `RETAIL_CATALOG_PERSISTENCE_PASSWORD` as base64-encoded values. The password is handled by the `catalog.persistence.password` helper (auto-generate or use provided value). This secret is also read directly by the MySQL StatefulSet container for `MYSQL_USER` and `MYSQL_PASSWORD` — a single source of truth for credentials shared between the app and the DB pod.

---

## End-to-End: What Happens on `helm install`

1. Helm merges `values.yaml` + any `-f values-*.yaml` files
2. Templates are rendered:
   - `configmap.yml` → sets `RETAIL_CATALOG_PERSISTENCE_PROVIDER` (always rendered if `configMap.create: true`)
   - `secret.yaml` → creates DB credentials (always rendered if `app.persistence.secret.create: true`)
   - `mysql-statefulset.yaml` + `mysql-service.yaml` → only if `mysql.create: true`
   - `hpa.yaml` → only if `autoscaling.enabled: true`
   - `pdb.yaml` → only if `podDisruptionBudget.enabled: true`
   - `security-group.yaml` → only if `securityGroups.create: true`
3. Catalog Deployment starts; pod mounts ConfigMap (env vars) and optionally Secret
4. App reads `RETAIL_CATALOG_PERSISTENCE_PROVIDER` from env; picks in-memory or MySQL driver
5. If MySQL: connects to endpoint from `RETAIL_CATALOG_PERSISTENCE_ENDPOINT` (ConfigMap) using credentials from the Secret
6. `/health` returns `200 OK` once the DB connection is healthy → Deployment marks pod as ready
