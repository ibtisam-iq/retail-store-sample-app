# Orders Chart — Helm Deployment Runbook

> **Pattern:** `helm upgrade --install` passes `values.yaml` (base) first, overlay second. Two independent axes: **persistence** (DB) and **messaging**.
>
> **Two axes:**
> - **DB:** `in-memory` (H2) | `postgresql` in-cluster ephemeral | `postgresql` in-cluster + PVC | `postgresql` external (RDS)
> - **Messaging:** `in-memory` | `rabbitmq` in-cluster ephemeral | `rabbitmq` in-cluster + PVC | `sqs` (AWS)

---

## Values Files — Full Scenario Matrix

| # | File | DB | DB PVC | StorageClass | Messaging | MQ PVC | SQS/IRSA |
|---|------|----|:------:|:------------:|-----------|:------:|:--------:|
| 01 | `values-01-in-memory.yaml` | H2 in-memory | ✗ | — | in-memory | ✗ | ✗ |
| 02 | `values-02-postgresql-ephemeral-inmsg.yaml` | PostgreSQL in-cluster | ✗ | — | in-memory | ✗ | ✗ |
| 03 | `values-03-postgresql-pvc-baremetal-inmsg.yaml` | PostgreSQL in-cluster | ✓ | `local-path` | in-memory | ✗ | ✗ |
| 04 | `values-04-postgresql-pvc-eks-inmsg.yaml` | PostgreSQL in-cluster | ✓ | `gp2` | in-memory | ✗ | ✗ |
| 05 | `values-05-external-postgresql-inmsg.yaml` | PostgreSQL external (RDS) | ✗ | — | in-memory | ✗ | ✗ |
| 06 | `values-06-postgresql-ephemeral-rabbitmq-ephemeral.yaml` | PostgreSQL in-cluster | ✗ | — | RabbitMQ in-cluster | ✗ | ✗ |
| 07 | `values-07-postgresql-pvc-baremetal-rabbitmq-ephemeral.yaml` | PostgreSQL in-cluster | ✓ | `local-path` | RabbitMQ in-cluster | ✗ | ✗ |
| 08 | `values-08-postgresql-pvc-baremetal-rabbitmq-pvc-baremetal.yaml` | PostgreSQL in-cluster | ✓ | `local-path` | RabbitMQ in-cluster | ✓ | `local-path` |
| 09 | `values-09-postgresql-pvc-eks-rabbitmq-ephemeral.yaml` | PostgreSQL in-cluster | ✓ | `gp2` | RabbitMQ in-cluster | ✗ | ✗ |
| 10 | `values-10-postgresql-pvc-eks-rabbitmq-pvc-eks.yaml` | PostgreSQL in-cluster | ✓ | `gp2` | RabbitMQ in-cluster | ✓ | `gp2` |
| 11 | `values-11-postgresql-ephemeral-sqs.yaml` | PostgreSQL in-cluster | ✗ | — | AWS SQS | ✗ | ✓ IRSA |
| 12 | `values-12-postgresql-pvc-eks-sqs.yaml` | PostgreSQL in-cluster | ✓ | `gp2` | AWS SQS | ✗ | ✓ IRSA |
| 13 | `values-13-external-postgresql-sqs.yaml` | PostgreSQL external (RDS) | ✗ | — | AWS SQS | ✗ | ✓ IRSA |
| 14 | `values-14-external-postgresql-sqs-securitygroup.yaml` | PostgreSQL external (RDS) | ✗ | — | AWS SQS + SecurityGroupPolicy | ✗ | ✓ IRSA |
| 15 | `values-15-hpa.yaml` | any | — | — | any | — | — |
| 16 | `values-16-pdb.yaml` | any | — | — | any | — | — |

> **Endpoint rule:** `app.persistence.endpoint` is only set when `postgresql.create: false` (external). When `postgresql.create: true`, `_helpers.tpl` auto-constructs the endpoint as `<release>-postgresql:5432`. **SQS note:** `messaging.provider: sqs` injects no RabbitMQ addresses — credentials come entirely from IRSA via the AWS credential provider chain. No Secret is created.

---

## Commands Per Scenario

### Template Validation (all scenarios)
```bash
helm template orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/<values-file>.yaml
```

### Scenario 01 — In-Memory (Baseline)
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-01-in-memory.yaml \
  --namespace orders --create-namespace

kubectl get pods -n orders
```

### Scenario 02 — PostgreSQL Ephemeral + In-Memory Messaging
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-02-postgresql-ephemeral-inmsg.yaml \
  --namespace orders --create-namespace

kubectl get pods,svc -n orders
```

### Scenario 03 — PostgreSQL PVC (Bare-Metal) + In-Memory Messaging
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-03-postgresql-pvc-baremetal-inmsg.yaml \
  --namespace orders --create-namespace

kubectl get sc
kubectl get pvc,pods -n orders
```

### Scenario 04 — PostgreSQL PVC (EKS gp2) + In-Memory Messaging
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-04-postgresql-pvc-eks-inmsg.yaml \
  --namespace orders --create-namespace

kubectl get sc
kubectl get pvc,pods -n orders
```

### Scenario 05 — External PostgreSQL (RDS) + In-Memory Messaging
```bash
# Replace <EXTERNAL_POSTGRESQL_HOST> and <REPLACE_ME> before running.
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-05-external-postgresql-inmsg.yaml \
  --namespace orders --create-namespace

kubectl get pod -n orders
kubectl logs -n orders -l app.kubernetes.io/name=orders
```

### Scenario 06 — PostgreSQL Ephemeral + RabbitMQ Ephemeral
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-06-postgresql-ephemeral-rabbitmq-ephemeral.yaml \
  --namespace orders --create-namespace

kubectl get pods,svc -n orders
# RabbitMQ Management UI: kubectl port-forward svc/orders-rabbitmq 15672:15672 -n orders
```

### Scenario 07 — PostgreSQL PVC (Bare-Metal) + RabbitMQ Ephemeral
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-07-postgresql-pvc-baremetal-rabbitmq-ephemeral.yaml \
  --namespace orders --create-namespace

kubectl get pvc,pods,svc -n orders
```

### Scenario 08 — PostgreSQL PVC (Bare-Metal) + RabbitMQ PVC (Bare-Metal)
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-08-postgresql-pvc-baremetal-rabbitmq-pvc-baremetal.yaml \
  --namespace orders --create-namespace

kubectl get pvc,pods,svc -n orders
```

### Scenario 09 — PostgreSQL PVC (EKS gp2) + RabbitMQ Ephemeral
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-09-postgresql-pvc-eks-rabbitmq-ephemeral.yaml \
  --namespace orders --create-namespace

kubectl get pvc,pods,svc -n orders
```

### Scenario 10 — PostgreSQL PVC (EKS gp2) + RabbitMQ PVC (EKS gp2)
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-10-postgresql-pvc-eks-rabbitmq-pvc-eks.yaml \
  --namespace orders --create-namespace

kubectl get pvc,pods,svc -n orders
```

### Scenario 11 — PostgreSQL Ephemeral + AWS SQS
```bash
# Replace <ACCOUNT_ID> and <SQS_IRSA_ROLE_NAME> before running.
# SQS queue must exist in the region.
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-11-postgresql-ephemeral-sqs.yaml \
  --namespace orders --create-namespace

kubectl get pods,svc -n orders
kubectl logs -n orders -l app.kubernetes.io/name=orders | grep -i sqs
```

### Scenario 12 — PostgreSQL PVC (EKS gp2) + AWS SQS
```bash
# Replace <ACCOUNT_ID> and <SQS_IRSA_ROLE_NAME> before running.
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-12-postgresql-pvc-eks-sqs.yaml \
  --namespace orders --create-namespace

kubectl get pvc,pods -n orders
```

### Scenario 13 — External PostgreSQL (RDS) + AWS SQS
```bash
# Fully managed AWS backend — no in-cluster stateful pods.
# Replace <RDS_ENDPOINT>, <REPLACE_ME>, <ACCOUNT_ID>, <SQS_IRSA_ROLE_NAME> before running.
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-13-external-postgresql-sqs.yaml \
  --namespace orders --create-namespace

kubectl get pod -n orders
kubectl describe pod -n orders -l app.kubernetes.io/name=orders
```

### Scenario 14 — External PostgreSQL (RDS) + AWS SQS + SecurityGroupPolicy
```bash
# VPC CNI security groups for pods must be enabled on the EKS cluster.
# Replace all <PLACEHOLDER> values before running.
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-14-external-postgresql-sqs-securitygroup.yaml \
  --namespace orders --create-namespace

kubectl get SecurityGroupPolicy -n orders
kubectl get pod -n orders -o jsonpath='{.items[0].metadata.annotations}'
```

### Scenario 15 — HPA
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-15-hpa.yaml \
  --namespace orders --create-namespace

kubectl get hpa -n orders
```

### Scenario 16 — PDB
```bash
helm upgrade --install orders src/orders/chart/ \
  -f src/orders/chart/values.yaml \
  -f src/orders/chart/values-16-pdb.yaml \
  --namespace orders --create-namespace

kubectl get pdb -n orders
kubectl get pods -n orders
```

### Teardown
```bash
helm uninstall orders -n orders
kubectl delete namespace orders
```

---

## Key Observations

| Scenario | Observed Behaviour |
|----------|-------------------|
| In-memory (01) | H2 DB + in-memory queue; Flyway still runs migrations against H2; zero external deps |
| PostgreSQL ephemeral (02, 06, 11) | DB pod starts; Flyway migrates schema on first boot; data lost on pod restart |
| PostgreSQL + PVC bare-metal (03, 07, 08) | `local-path` provisioner required (Rancher); data persists across restarts |
| PostgreSQL + PVC EKS (04, 09, 10, 12) | `gp2` EBS StorageClass; EBS volume created automatically; data persists |
| External PostgreSQL / RDS (05, 13, 14) | No DB pod; Flyway runs migrations against RDS on first start; ~60s cold-start |
| RabbitMQ ephemeral (06, 07, 09) | MQ pod created; unprocessed messages lost on restart; management UI at port 15672 |
| RabbitMQ + PVC (08, 10) | Messages persist across restarts; `storageClass` must exist on cluster |
| SQS (11, 12, 13, 14) | No broker pod; no RabbitMQ Secret; credentials via IRSA only; SQS queue must pre-exist |
| SecurityGroupPolicy (14) | VPC CNI assigns pod-level SG; allows fine-grained RDS access without broad VPC rules |
| HPA (15) | Only scales the orders app pod — PostgreSQL/RabbitMQ StatefulSets are unaffected |
| PDB (16) | Only effective when `replicaCount ≥ 2`; single-replica PDB is a no-op |
