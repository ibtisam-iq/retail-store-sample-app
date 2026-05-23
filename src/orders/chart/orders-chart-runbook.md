# Orders Chart — Helm Deployment Runbook

I forked this repository from the [AWS Retail Store Sample App](https://github.com/aws-containers/retail-store-sample-app) and extended the `src/orders/chart` directory by authoring platform-specific overlay values files on top of the base `values.yaml`. I validated each scenario end-to-end — bare-metal, EKS, and external managed services — to demonstrate two-axis (DB × Messaging) Helm configuration across real deployment targets.

> **Deployment pattern:** Every `helm` command passes `values.yaml` first (base) and the scenario file second (patch). Helm deep-merges both; the patch file overrides only the keys it declares.

---

## Values Files — Scenario Matrix

All values files are scoped to **database and messaging configuration**. Two independent axes are covered: **DB** (persistence provider) and **Messaging** (event provider).

| File | DB Provider | DB Pod | DB PVC | Messaging Provider | Messaging Pod | StorageClass |
|---|---|:---:|:---:|---|:---:|---|
| `values.yaml` | `in-memory` | ✗ | ✗ | `in-memory` | ✗ | — |
| `values-01-in-memory.yaml` | `in-memory` | ✗ | ✗ | `in-memory` | ✗ | — |
| `values-02-postgresql-ephemeral-msg-in-memory.yaml` | `postgres` | ✓ | ✗ | `in-memory` | ✗ | — |
| `values-03-postgresql-rabbitmq-pvc-baremetal.yaml` | `postgres` | ✓ | ✓ | `rabbitmq` | ✓ | `local-path` |
| `values-04-postgresql-rabbitmq-pvc-eks.yaml` | `postgres` | ✓ | ✓ | `rabbitmq` | ✓ | `gp2` |
| `values-05-postgresql-rabbitmq-external.yaml` | `postgres` | ✗ | ✗ | `rabbitmq` | ✗ | — |
| `values-06-postgresql-pvc-eks-sqs.yaml` | `postgres` | ✓ | ✓ | `sqs` | ✗ | `gp2` |

> **DB endpoint auto-construction:** `app.persistence.endpoint` is only required when `postgresql.create: false` (external RDS). When `postgresql.create: true`, `_helpers.tpl` auto-constructs the endpoint as `<release>-orders-postgresql:<port>` — no manual value needed. `RETAIL_ORDERS_PERSISTENCE_ENDPOINT` is only injected into the ConfigMap when `provider: postgres`.

> **RabbitMQ addresses auto-construction:** `app.messaging.rabbitmq.addresses` is only required when `rabbitmq.create: false` (external broker). When `rabbitmq.create: true`, `_helpers.tpl` auto-constructs the address as `<release>-orders-rabbitmq:<amqp-port>` — no manual value needed. `RETAIL_ORDERS_MESSAGING_RABBITMQ_ADDRESSES` is only injected into the ConfigMap when `provider: rabbitmq`.

---

## Commands Per Scenario

### Template Validation (all scenarios)

```bash
helm template orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/<values-file>.yaml
```

### Scenario 1 — In-Memory (Baseline)

```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-01-in-memory.yaml \
  --namespace orders --create-namespace

kubectl get pods -n orders
```

### Scenario 2 — In-Cluster PostgreSQL (Ephemeral) + In-Memory Messaging

```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-02-postgresql-ephemeral-msg-in-memory.yaml \
  --namespace orders --create-namespace

kubectl get pods,svc -n orders
kubectl exec -n orders -it orders-postgresql-0 -- psql -U orders -d orders
```

### Scenario 3 — PostgreSQL + RabbitMQ (Bare-metal PVC)

```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-03-postgresql-rabbitmq-pvc-baremetal.yaml \
  --namespace orders --create-namespace

kubectl get sc
kubectl get pvc,pods,svc -n orders
kubectl exec -n orders -it orders-postgresql-0 -- psql -U orders -d orders
```

### Scenario 4 — PostgreSQL + RabbitMQ (EKS gp2 PVC)

```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-04-postgresql-rabbitmq-pvc-eks.yaml \
  --namespace orders --create-namespace

kubectl get sc
kubectl get pvc,pods,svc -n orders
kubectl exec -n orders -it orders-postgresql-0 -- psql -U orders -d orders
```

### Scenario 5 — External RDS + External RabbitMQ

```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-05-postgresql-rabbitmq-external.yaml \
  --namespace orders --create-namespace

kubectl get pod -n orders -l app.kubernetes.io/name=orders
kubectl describe pod -n orders -l app.kubernetes.io/name=orders
```

### Scenario 6 — PostgreSQL PVC (EKS gp2) + AWS SQS (IRSA)

```bash
# Pre-requisite: SQS queue must exist; IRSA role must have sqs:SendMessage permission.
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-06-postgresql-pvc-eks-sqs.yaml \
  --namespace orders --create-namespace

kubectl get pod -n orders
kubectl logs -n orders -l app.kubernetes.io/name=orders
```

### Teardown

```bash
helm uninstall orders -n orders
kubectl delete namespace orders
```

---

## Key Observations

| Scenario | Observed Behaviour |
|---|---|
| In-memory | No DB or messaging dependency; all data lost on pod restart |
| PostgreSQL ephemeral | PostgreSQL pod created alongside orders; data lost on pod restart |
| PostgreSQL + RabbitMQ (bare-metal) | Both PVCs bound to `local-path`; data and messages persisted across restarts |
| PostgreSQL + RabbitMQ (EKS) | Both PVCs bound to `gp2` EBS volumes; data and messages persisted across restarts |
| External RDS + External RabbitMQ | No StatefulSet created; `endpoint` and `addresses` must be set explicitly in the overlay |
| PostgreSQL + SQS (EKS) | No RabbitMQ pod; IRSA injects AWS credentials — no Secret needed for SQS auth |
| DB endpoint auto-build | When `postgresql.create: true`, chart builds `RETAIL_ORDERS_PERSISTENCE_ENDPOINT` as `<release>-orders-postgresql:5432` automatically via `_helpers.tpl` |
| RabbitMQ address auto-build | When `rabbitmq.create: true`, chart builds `RETAIL_ORDERS_MESSAGING_RABBITMQ_ADDRESSES` as `<release>-orders-rabbitmq:5672` automatically via `_helpers.tpl` |
