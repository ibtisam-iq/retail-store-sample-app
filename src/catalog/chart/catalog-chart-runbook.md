# Catalog Chart — Helm Deployment Runbook

I forked this repository from the [AWS Retail Store Sample App](https://github.com/aws-containers/retail-store-sample-app) and extended the `src/catalog/chart` directory by authoring platform-specific overlay values files on top of the base `values.yaml`. I validated each scenario end-to-end — bare-metal, EKS, and external RDS — to demonstrate database-driven Helm configuration across real deployment targets.

> **Deployment pattern:** Every `helm` command passes `values.yaml` first (base) and the scenario file second (patch). Helm deep-merges both; the patch file overrides only the keys it declares.

---

## Values Files — Scenario Matrix

All values files are scoped to **database configuration only**. Three platform scenarios are covered: bare-metal, EKS, and external/RDS.

| File | Platform | Persistence | MySQL Pod | PVC | StorageClass | Endpoint Required |
|---|---|---|---|---|---|---|
| `values.yaml` | Any | `in-memory` | ✗ | ✗ | — | — |
| `values-in-memory.yaml` | Any | `in-memory` | ✗ | ✗ | — | — |
| `values-mysql-ephemeral.yaml` | Any | `mysql` | ✓ | ✗ | — | ✗ auto-built |
| `values-mysql-pvc-baremetal.yaml` | Bare-metal | `mysql` | ✓ | ✓ | `local-path` | ✗ auto-built |
| `values-mysql-pvc-eks.yaml` | EKS | `mysql` | ✓ | ✓ | `gp3` | ✗ auto-built |
| `values-external-mysql.yaml` | Any (RDS) | `mysql` | ✗ | ✗ | — | ✓ must set |

> **Endpoint decision:** `app.persistence.endpoint` is only required when `mysql.create: false`. When `mysql.create: true`, the chart's `_helpers.tpl` auto-constructs the endpoint as `<release>-mysql:<port>` — no manual value needed. I deliberately omitted `endpoint` from all in-cluster scenarios based on this template logic.

---

## Commands Per Scenario

### Template Validation (all scenarios)

```bash
helm template catalog src/catalog/chart/ \
  -f src/catalog/chart/values.yaml \
  -f src/catalog/chart/<values-file>.yaml
```

### Scenario 1 — In-Memory (Baseline)

```bash
helm upgrade --install catalog src/catalog/chart/ \
  -f src/catalog/chart/values.yaml \
  -f src/catalog/chart/values-in-memory.yaml \
  --namespace catalog --create-namespace

kubectl get pods -n catalog
```

### Scenario 2 — In-Cluster MySQL, Ephemeral

```bash
helm upgrade --install catalog src/catalog/chart/ \
  -f src/catalog/chart/values.yaml \
  -f src/catalog/chart/values-mysql-ephemeral.yaml \
  --namespace catalog --create-namespace

kubectl get pods,svc -n catalog
kubectl exec -n catalog -it catalog-mysql-0 -- mysql -u catalog -pcatalog123 catalog
```

### Scenario 3 — In-Cluster MySQL + PVC (Bare-metal)

```bash
helm upgrade --install catalog src/catalog/chart/ \
  -f src/catalog/chart/values.yaml \
  -f src/catalog/chart/values-mysql-pvc-baremetal.yaml \
  --namespace catalog --create-namespace

kubectl get sc
kubectl get pvc,pods,svc -n catalog
kubectl exec -n catalog -it catalog-mysql-0 -- mysql -u catalog -pcatalog123 catalog
```

### Scenario 4 — In-Cluster MySQL + PVC (EKS)

```bash
helm upgrade --install catalog src/catalog/chart/ \
  -f src/catalog/chart/values.yaml \
  -f src/catalog/chart/values-mysql-pvc-eks.yaml \
  --namespace catalog --create-namespace

kubectl get sc
kubectl get pvc,pods,svc -n catalog
kubectl exec -n catalog -it catalog-mysql-0 -- mysql -u catalog -pcatalog123 catalog
```

### Scenario 5 — External MySQL / RDS

```bash
helm upgrade --install catalog src/catalog/chart/ \
  -f src/catalog/chart/values.yaml \
  -f src/catalog/chart/values-external-mysql.yaml \
  --namespace catalog --create-namespace

kubectl get pod -n catalog -l app.kubernetes.io/name=catalog
kubectl describe pod -n catalog -l app.kubernetes.io/name=catalog
kubectl run mysql-test -n catalog --rm -it --image=mysql:8.0 -- \
  mysql -h retail-store-sample-db.cri2is6gmf7z.us-east-1.rds.amazonaws.com \
  -u catalog -pcatalog123 catalog
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
| MySQL + PVC (bare-metal) | PVC bound to `local-path`; data persisted across pod restarts |
| MySQL + PVC (EKS) | PVC bound to `gp3` EBS volume; data persisted across pod restarts |
| External MySQL / RDS | No MySQL pod created; app ran DB migrations and seeded data into RDS on first start; ~60s cold-start expected while migrations complete |
