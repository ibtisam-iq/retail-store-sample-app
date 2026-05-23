# Orders Chart — Overlay Runbook

> Base chart: `src/orders/chart/` | Base values: `values.yaml` (untouched)
> Two independent axes: **DB** × **Messaging** — each overlay file overrides only what changes.

---

## Overlay Matrix

| # | File | DB | Messaging | Platform |
|---|------|----|-----------|----------|
| 01 | `values-01-in-memory.yaml` | H2 in-memory | in-memory | any |
| 02 | `values-02-postgresql-ephemeral-msg-in-memory.yaml` | PG ephemeral | in-memory | bare-metal/EKS |
| 03 | `values-03-postgresql-pvc-baremetal-msg-in-memory.yaml` | PG + `local-path` PVC | in-memory | bare-metal |
| 04 | `values-04-postgresql-pvc-eks-msg-in-memory.yaml` | PG + `gp2` EBS PVC | in-memory | EKS |
| 05 | `values-05-external-postgresql-msg-in-memory.yaml` | RDS (no pod) | in-memory | EKS |
| 06 | `values-06-postgresql-ephemeral-rabbitmq-ephemeral.yaml` | PG ephemeral | RabbitMQ ephemeral | any |
| 07 | `values-07-postgresql-pvc-baremetal-rabbitmq-ephemeral.yaml` | PG + `local-path` PVC | RabbitMQ ephemeral | bare-metal |
| 08 | `values-08-postgresql-pvc-baremetal-rabbitmq-pvc-baremetal.yaml` | PG + `local-path` PVC | RabbitMQ + `local-path` PVC | bare-metal |
| 09 | `values-09-postgresql-pvc-eks-rabbitmq-pvc-eks.yaml` | PG + `gp2` EBS PVC | RabbitMQ + `gp2` EBS PVC | EKS |
| 10 | `values-10-external-postgresql-rabbitmq-ephemeral.yaml` | RDS (no pod) | RabbitMQ ephemeral | EKS |
| 11 | `values-11-external-postgresql-rabbitmq-pvc-eks.yaml` | RDS (no pod) | RabbitMQ + `gp2` EBS PVC | EKS |
| 12 | `values-12-external-postgresql-external-rabbitmq.yaml` | RDS (no pod) | External RabbitMQ (no pod) | EKS |
| 13 | `values-13-postgresql-ephemeral-sqs.yaml` | PG ephemeral | AWS SQS (IRSA) | EKS |
| 14 | `values-14-external-postgresql-sqs.yaml` | RDS (no pod) | AWS SQS (IRSA) | EKS |
| 15 | `values-15-hpa.yaml` | — | — | any |
| 16 | `values-16-pdb.yaml` | — | — | any |

---

## Quick Commands

```bash
# 01 — fully in-memory (smoke test)
helm install orders ./src/orders/chart -f values-01-in-memory.yaml

# 02 — PostgreSQL ephemeral + in-memory messaging
helm install orders ./src/orders/chart -f values-02-postgresql-ephemeral-msg-in-memory.yaml

# 03 — PostgreSQL PVC (bare-metal) + in-memory messaging
helm install orders ./src/orders/chart -f values-03-postgresql-pvc-baremetal-msg-in-memory.yaml

# 04 — PostgreSQL PVC (EKS gp2) + in-memory messaging
helm install orders ./src/orders/chart -f values-04-postgresql-pvc-eks-msg-in-memory.yaml

# 05 — External RDS + in-memory messaging
helm install orders ./src/orders/chart -f values-05-external-postgresql-msg-in-memory.yaml

# 06 — PostgreSQL ephemeral + RabbitMQ ephemeral
helm install orders ./src/orders/chart -f values-06-postgresql-ephemeral-rabbitmq-ephemeral.yaml

# 07 — PostgreSQL PVC (bare-metal) + RabbitMQ ephemeral
helm install orders ./src/orders/chart -f values-07-postgresql-pvc-baremetal-rabbitmq-ephemeral.yaml

# 08 — Full bare-metal persistent stack
helm install orders ./src/orders/chart -f values-08-postgresql-pvc-baremetal-rabbitmq-pvc-baremetal.yaml

# 09 — Full EKS gp2 persistent stack (in-cluster)
helm install orders ./src/orders/chart -f values-09-postgresql-pvc-eks-rabbitmq-pvc-eks.yaml

# 10 — External RDS + RabbitMQ ephemeral
helm install orders ./src/orders/chart -f values-10-external-postgresql-rabbitmq-ephemeral.yaml

# 11 — External RDS + RabbitMQ PVC (EKS gp2)
helm install orders ./src/orders/chart -f values-11-external-postgresql-rabbitmq-pvc-eks.yaml

# 12 — Fully external (RDS + external RabbitMQ)
helm install orders ./src/orders/chart -f values-12-external-postgresql-external-rabbitmq.yaml

# 13 — PostgreSQL ephemeral + AWS SQS (IRSA)
helm install orders ./src/orders/chart -f values-13-postgresql-ephemeral-sqs.yaml

# 14 — External RDS + AWS SQS (IRSA) — fully managed AWS
helm install orders ./src/orders/chart -f values-14-external-postgresql-sqs.yaml

# Combine with HPA (stack on top of any scenario above)
helm install orders ./src/orders/chart \
  -f values-14-external-postgresql-sqs.yaml \
  -f values-15-hpa.yaml

# Combine with PDB (requires replicaCount >= 3)
helm install orders ./src/orders/chart \
  -f values-09-postgresql-pvc-eks-rabbitmq-pvc-eks.yaml \
  -f values-16-pdb.yaml
```

---

## Key Observations

| Topic | Note |
|-------|------|
| **Two-axis model** | `app.persistence.provider` and `app.messaging.provider` are independent — any DB can pair with any messaging backend |
| **`postgresql.create`** | `true` = StatefulSet created in-cluster; `false` = use `app.persistence.endpoint` for external RDS |
| **`rabbitmq.create`** | `true` = StatefulSet created in-cluster; `false` = use `app.messaging.rabbitmq.addresses` for external broker |
| **SQS credentials** | No Secret required; IRSA injects credentials via `serviceAccount.annotations` (`eks.amazonaws.com/role-arn`) |
| **`app.messaging.rabbitmq.secret`** | Only mounted by the Deployment when `messaging.provider: rabbitmq`; ignored for `in-memory` and `sqs` |
| **`securityGroups.create`** | EKS-only CRD (`SecurityGroupPolicy`) — enables VPC SG assignment to the orders pod for RDS/external broker access |
| **PVC + `storageClass`** | `local-path` for bare-metal; `gp2` for EKS — set under both `postgresql.persistentVolume` and `rabbitmq.persistentVolume` |
| **`values.yaml`** | Base file is never modified — all scenario-specific overrides live in numbered overlay files |
