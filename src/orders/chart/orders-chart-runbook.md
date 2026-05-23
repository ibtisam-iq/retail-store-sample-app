# Orders Chart — Helm Deployment Runbook

> **Pattern:** `helm upgrade --install` passes `values.yaml` (base) first, scenario overlay second. Two independent axes: **persistence** (PostgreSQL) and **messaging** (RabbitMQ).

## Values Files — Scenario Matrix

| File | DB Provider | PostgreSQL Pod | PVC | StorageClass | Messaging | RabbitMQ Pod |
|------|-------------|:--------------:|:---:|:------------:|-----------|:------------:|
| `values.yaml` | `in-memory` | ✗ | ✗ | — | `in-memory` | ✗ |
| `values-in-memory.yaml` | `in-memory` | ✗ | ✗ | — | `in-memory` | ✗ |
| `values-postgresql-ephemeral.yaml` | `postgresql` | ✓ | ✗ | — | `in-memory` | ✗ |
| `values-postgresql-pvc-baremetal.yaml` | `postgresql` | ✓ | ✓ | `local-path` | `in-memory` | ✗ |
| `values-postgresql-pvc-eks.yaml` | `postgresql` | ✓ | ✓ | `gp3` | `in-memory` | ✗ |
| `values-external-postgresql.yaml` | `postgresql` | ✗ | ✗ | — | `in-memory` | ✗ |
| `values-rabbitmq.yaml` | `postgresql` | ✓ (ephemeral) | ✗ | — | `rabbitmq` | ✓ (ephemeral) |
| `values-full-stack-eks.yaml` | `postgresql` | ✓ | ✓ | `gp3` | `rabbitmq` | ✓ + `gp3` |
| `values-hpa.yaml` | any | — | — | — | — | — |
| `values-pdb.yaml` | any | — | — | — | — | — |

## Commands Per Scenario

### Scenario 1 — In-Memory (Baseline)
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-in-memory.yaml \
  --namespace orders --create-namespace

kubectl get pods -n orders
```

### Scenario 2 — PostgreSQL Ephemeral
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-postgresql-ephemeral.yaml \
  --namespace orders --create-namespace

kubectl get pods,svc -n orders
```

### Scenario 3 — PostgreSQL + PVC (Bare-metal)
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-postgresql-pvc-baremetal.yaml \
  --namespace orders --create-namespace

kubectl get pvc,pods -n orders
```

### Scenario 4 — PostgreSQL + PVC (EKS)
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-postgresql-pvc-eks.yaml \
  --namespace orders --create-namespace

kubectl get pvc,pods -n orders
```

### Scenario 5 — External PostgreSQL (RDS)
```bash
# Update endpoint and password in values-external-postgresql.yaml before running.
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-external-postgresql.yaml \
  --namespace orders --create-namespace

kubectl get pod -n orders
kubectl logs -n orders -l app.kubernetes.io/name=orders
```

### Scenario 6 — RabbitMQ (In-Cluster)
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-rabbitmq.yaml \
  --namespace orders --create-namespace

kubectl get pods,svc -n orders
```

### Scenario 7 — Full Stack EKS (PostgreSQL + RabbitMQ + PVC)
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-full-stack-eks.yaml \
  --namespace orders --create-namespace

kubectl get pvc,pods,svc -n orders
```

### Teardown
```bash
helm uninstall orders -n orders
kubectl delete namespace orders
```

## Key Observations

| Scenario | Observed Behaviour |
|----------|-------------------|
| In-memory | No pods besides orders itself; data lost on restart |
| PostgreSQL ephemeral | DB pod created; schema migrated on first start |
| PostgreSQL + PVC | Data survives pod restarts; StorageClass must exist on cluster |
| External PostgreSQL | No DB pod; `endpoint` placeholder must be replaced before deploy |
| RabbitMQ | Async messaging enabled; orders service publishes events to `orders` exchange |
| Full stack EKS | Both StatefulSets use `gp3` EBS PVCs; most durable production pattern |
| HPA | Metrics server required; only scales orders app pod, not DB/MQ |
| PDB | Only meaningful when `replicaCount ≥ 2` |
