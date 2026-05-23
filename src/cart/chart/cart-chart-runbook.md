# Cart Chart — Helm Deployment Runbook

> **Pattern:** `helm upgrade --install` always passes `values.yaml` first (base) then the scenario overlay second. Helm deep-merges; the overlay overrides only declared keys.

## Values Files — Scenario Matrix

All values files are scoped to **database configuration only**. Three platform scenarios are covered:

| File | Persistence | DynamoDB Pod | AWS DynamoDB | Table Auto-Created |
|------|-------------|:------------:|:------------:|:------------------:|
| `values.yaml` | `in-memory` | ✗ | ✗ | — |
| `values-in-memory.yaml` | `in-memory` | ✗ | ✗ | — |
| `values-dynamodb-local.yaml` | `dynamodb` | ✓ in-cluster | ✗ | ✓ (`createTable: true`) |
| `values-dynamodb-aws.yaml` | `dynamodb` | ✗ | ✓ real AWS | ✗ (must pre-exist) |

> When `dynamodb.create: true`, the chart auto-sets `RETAIL_CART_PERSISTENCE_DYNAMODB_ENDPOINT` to the in-cluster service and injects dummy `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` (required by DynamoDB Local SDK). No manual endpoint needed.

## Commands Per Scenario

### Scenario 1 — In-Memory (Baseline)
```bash
helm upgrade --install cart src/cart/chart/ \
  -f src/cart/chart/values.yaml \
  -f src/cart/chart/values-in-memory.yaml \
  --namespace cart --create-namespace

kubectl get pods -n cart
```

### Scenario 2 — DynamoDB Local (In-Cluster)
```bash
helm upgrade --install cart src/cart/chart/ \
  -f src/cart/chart/values.yaml \
  -f src/cart/chart/values-dynamodb-local.yaml \
  --namespace cart --create-namespace

kubectl get pods,svc -n cart
```

### Scenario 3 — AWS DynamoDB (EKS + IAM)
```bash
# Pre-requisite: DynamoDB table "Items" must exist in the target region.
# Pod IAM access via IRSA (annotate serviceAccount) or node instance profile.
helm upgrade --install cart src/cart/chart/ \
  -f src/cart/chart/values.yaml \
  -f src/cart/chart/values-dynamodb-aws.yaml \
  --namespace cart --create-namespace

kubectl get pod -n cart
kubectl logs -n cart -l app.kubernetes.io/name=carts
```

### Teardown
```bash
helm uninstall cart -n cart
kubectl delete namespace cart
```

## Key Observations

| Scenario | Observed Behaviour |
|----------|-------------------|
| In-memory | No DB dependency; data lost on pod restart |
| DynamoDB Local | Chart injects endpoint + dummy AWS keys automatically |
| AWS DynamoDB | Table must pre-exist; IRSA annotation required on ServiceAccount |
