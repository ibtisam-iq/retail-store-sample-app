# Catalog Chart — Helm Practice Runbook

This repository was forked from the [AWS Retail Store Sample App](https://github.com/aws-containers/retail-store-sample-app). The `src/catalog/chart` directory contained a production-grade Helm chart for the catalog microservice. Multiple `values.yaml` variations were authored and deployed to explore different persistence strategies, scaling configurations, and availability patterns.

---

## Chart Reference

| Field | Value |
|---|---|
| Chart name | `retail-store-sample-catalog-chart` |
| Chart version | `1.5.0` |
| Chart path | `src/catalog/chart/` |
| Default image | `public.ecr.aws/aws-containers/retail-store-sample-catalog` |
| Default MySQL image | `public.ecr.aws/docker/library/mysql:8.0` |

---

## Values Files — Scenario Matrix

| File | Persistence | MySQL Pod | PVC | Key Toggle |
|---|---|---|---|---|
| `values.yaml` | `in-memory` | ✗ | ✗ | Upstream default — untouched |
| `values-in-memory.yaml` | `in-memory` | ✗ | ✗ | Clean baseline, confirmed chart renders |
| `values-mysql-ephemeral.yaml` | `mysql` | ✓ | ✗ | `mysql.create: true`, ephemeral storage |
| `values-mysql-pvc.yaml` | `mysql` | ✓ | ✓ | `mysql.persistentVolume.enabled: true` |
| `values-external-mysql.yaml` | `mysql` | ✗ | ✗ | `app.persistence.endpoint` set to external host |
| `values-hpa.yaml` | `in-memory` | ✗ | ✗ | `autoscaling.enabled: true`, min 2 / max 5 replicas |
| `values-pdb.yaml` | `in-memory` | ✗ | ✗ | `podDisruptionBudget.enabled: true`, minAvailable: 2 |

---

## Commands Executed Per Scenario

### Template Validation (all scenarios)

```bash
# Dry-run render to validate templating before any cluster deployment
helm template catalog src/catalog/chart/ -f src/catalog/chart/<values-file>.yaml
```

### Scenario 1 — In-Memory Baseline

```bash
helm install catalog src/catalog/chart/ \
  -f src/catalog/chart/values-in-memory.yaml \
  --namespace catalog --create-namespace

kubectl get pods -n catalog
kubectl logs -n catalog -l app.kubernetes.io/name=retail-store-sample-catalog-chart
```

### Scenario 2 — In-Cluster MySQL, Ephemeral Storage

```bash
helm install catalog src/catalog/chart/ \
  -f src/catalog/chart/values-mysql-ephemeral.yaml \
  --namespace catalog --create-namespace

kubectl get pods -n catalog
kubectl exec -n catalog -it <mysql-pod> -- mysql -u catalog -pcatalog123 catalog
```

### Scenario 3 — In-Cluster MySQL with PVC

```bash
helm install catalog src/catalog/chart/ \
  -f src/catalog/chart/values-mysql-pvc.yaml \
  --namespace catalog --create-namespace

kubectl get pvc -n catalog
kubectl get pods -n catalog
```

### Scenario 4 — External MySQL

```bash
# Replace <EXTERNAL_MYSQL_HOST> and <REPLACE_ME> in the values file before running
helm install catalog src/catalog/chart/ \
  -f src/catalog/chart/values-external-mysql.yaml \
  --namespace catalog --create-namespace

kubectl describe pod -n catalog -l app.kubernetes.io/name=retail-store-sample-catalog-chart
```

### Scenario 5 — HPA

```bash
helm install catalog src/catalog/chart/ \
  -f src/catalog/chart/values-hpa.yaml \
  --namespace catalog --create-namespace

kubectl get hpa -n catalog
kubectl describe hpa -n catalog
```

### Scenario 6 — PodDisruptionBudget

```bash
helm install catalog src/catalog/chart/ \
  -f src/catalog/chart/values-pdb.yaml \
  --namespace catalog --create-namespace

kubectl get pdb -n catalog
kubectl describe pdb -n catalog
```

### Upgrade Pattern

```bash
# Applied to any running release after editing a values file
helm upgrade catalog src/catalog/chart/ \
  -f src/catalog/chart/<values-file>.yaml \
  --namespace catalog

helm history catalog -n catalog
```

### Teardown

```bash
helm uninstall catalog -n catalog
kubectl delete namespace catalog
```

---

## Key Observations

| Scenario | Observed Behaviour |
|---|---|
| In-memory | Pod started immediately — no DB dependency, fastest startup |
| MySQL ephemeral | MySQL pod created alongside catalog; data lost on pod restart |
| MySQL + PVC | Data persisted across restarts; PVC bound to default StorageClass |
| External MySQL | No MySQL pod created; catalog connects via endpoint env var |
| HPA | HPA object created; scaled up when CPU threshold exceeded |
| PDB | PDB blocked node drain until minAvailable constraint was satisfied |
