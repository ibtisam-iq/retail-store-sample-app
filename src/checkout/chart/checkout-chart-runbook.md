# Checkout Chart — Helm Deployment Runbook

> **Pattern:** `helm upgrade --install` passes `values.yaml` (base) first, scenario overlay second. The single configuration axis is **session persistence** (in-memory vs Redis).

## Values Files — Scenario Matrix

| File | Persistence | Redis Pod | Redis Endpoint | TLS |
|------|-------------|:---------:|----------------|:---:|
| `values.yaml` | `in-memory` | ✗ | — | — |
| `values-in-memory.yaml` | `in-memory` | ✗ | — | — |
| `values-redis-local.yaml` | `redis` | ✓ in-cluster | auto (`redis://checkout-redis:6379`) | ✗ |
| `values-redis-aws-elasticache.yaml` | `redis` | ✗ | `<ELASTICACHE_ENDPOINT>:6379` | ✗ |
| `values-redis-tls.yaml` | `redis` | ✗ | `<ELASTICACHE_TLS_ENDPOINT>:6380` | ✓ (`rediss://`) |

> When `redis.create: true`, the chart auto-constructs `RETAIL_CHECKOUT_PERSISTENCE_REDIS_URL` as `redis://<release>-redis:<port>`. When `redis.create: false`, it uses `app.persistence.redis.tls` to choose `redis://` vs `rediss://` scheme.

## Commands Per Scenario

### Scenario 1 — In-Memory (Baseline)
```bash
helm upgrade --install checkout src/checkout/chart/ \
  -f src/checkout/chart/values.yaml \
  -f src/checkout/chart/values-in-memory.yaml \
  --set app.endpoints.orders="http://orders.orders.svc.cluster.local" \
  --namespace checkout --create-namespace

sleep 15

kubectl get pods -n checkout
```

### Scenario 2 — Redis In-Cluster
```bash
helm upgrade --install checkout src/checkout/chart/ \
  -f src/checkout/chart/values.yaml \
  -f src/checkout/chart/values-redis-local.yaml \
  --set app.endpoints.orders="http://orders.orders.svc.cluster.local" \
  --namespace checkout --create-namespace

sleep 15

kubectl get pods,svc -n checkout
kubectl exec -n checkout deploy/checkout -- env | grep -i order
kubectl get configmap -n checkout -o yaml
```

### Scenario 3 — AWS ElastiCache (no TLS)
```bash
helm upgrade --install checkout src/checkout/chart/ \
  -f src/checkout/chart/values.yaml \
  -f src/checkout/chart/values-redis-aws-elasticache.yaml \
  --set app.endpoints.orders="http://orders.orders.svc.cluster.local" \
  --namespace checkout --create-namespace

sleep 15

kubectl get pod -n checkout
kubectl logs -n checkout -l app.kubernetes.io/name=checkout
```

### Scenario 4 — ElastiCache with TLS
```bash
helm upgrade --install checkout src/checkout/chart/ \
  -f src/checkout/chart/values.yaml \
  -f src/checkout/chart/values-redis-tls.yaml \
  --set app.endpoints.orders="http://orders.orders.svc.cluster.local" \
  --namespace checkout --create-namespace

sleep 15

kubectl get pod -n checkout
```

### Teardown
```bash
helm uninstall checkout -n checkout
```

## Key Observations

| Scenario | Observed Behaviour |
|----------|-------------------|
| In-memory | No Redis dependency; sessions lost on pod restart |
| Redis in-cluster | Chart injects endpoint automatically; `redis://` scheme |
| ElastiCache no TLS | Endpoint must be set manually; uses `redis://` scheme |
| ElastiCache TLS | Uses `rediss://` scheme (double-s); port typically 6380 |
