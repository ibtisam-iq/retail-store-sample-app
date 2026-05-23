# UI Chart — Helm Deployment Runbook

> **Pattern:** `helm upgrade --install` passes `values.yaml` (base) first, scenario overlay second. Two independent axes: **service exposure** (ClusterIP / NodePort / LoadBalancer / Ingress) and **AI chat** (disabled / Bedrock / OpenAI).

## Values Files — Scenario Matrix

| File | Service Type | External Access | Ingress | AI Chat |
|------|-------------|:---------------:|:-------:|:-------:|
| `values.yaml` | `ClusterIP` | ✗ | ✗ | ✗ |
| `values-clusterip.yaml` | `ClusterIP` | ✗ (port-forward only) | ✗ | ✗ |
| `values-nodeport.yaml` | `NodePort :30080` | ✓ via node IP | ✗ | ✗ |
| `values-loadbalancer.yaml` | `LoadBalancer` | ✓ cloud NLB | ✗ | ✗ |
| `values-alb-ingress.yaml` | `ClusterIP` | ✓ AWS ALB | ✓ `alb` class | ✗ |
| `values-hpa.yaml` | — (any) | — | — | — |
| `values-pdb.yaml` | — (any) | — | — | — |
| `values-chat-bedrock.yaml` | — (any) | — | — | ✓ AWS Bedrock |
| `values-chat-openai.yaml` | — (any) | — | — | ✓ OpenAI-compatible |

## Commands Per Scenario

### Scenario 1 — ClusterIP + Port-Forward (Baseline)
```bash
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-clusterip.yaml \
  --namespace ui --create-namespace

kubectl port-forward svc/ui 8080:80 -n ui
# Access: http://localhost:8080
```

### Scenario 2 — NodePort (Bare-Metal)
```bash
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-nodeport.yaml \
  --namespace ui --create-namespace

kubectl get svc -n ui
# Access: http://<node-ip>:30080
```

### Scenario 3 — LoadBalancer (EKS NLB)
```bash
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-loadbalancer.yaml \
  --namespace ui --create-namespace

kubectl get svc -n ui -w   # Wait for EXTERNAL-IP
# Access: http://<EXTERNAL-IP>
```

### Scenario 4 — ALB Ingress (EKS)
```bash
# Pre-requisite: AWS Load Balancer Controller installed in the cluster.
# Replace <YOUR_DOMAIN_OR_ALB_DNS> in the values file before running.
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-alb-ingress.yaml \
  --namespace ui --create-namespace

kubectl get ingress -n ui -w   # Wait for ADDRESS
```

### Scenario 5 — HPA
```bash
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-hpa.yaml \
  --namespace ui --create-namespace

kubectl get hpa -n ui
```

### Scenario 6 — PDB
```bash
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-pdb.yaml \
  --namespace ui --create-namespace

kubectl get pdb -n ui
```

### Scenario 7 — AI Chat via AWS Bedrock
```bash
# Pre-requisite: IRSA or node profile with bedrock:InvokeModel permission.
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-chat-bedrock.yaml \
  --namespace ui --create-namespace

kubectl get pod -n ui
kubectl logs -n ui -l app.kubernetes.io/name=ui | grep -i chat
```

### Scenario 8 — AI Chat via OpenAI-Compatible API
```bash
# Works with OpenAI, Ollama, LM Studio, or any OpenAI-compatible endpoint.
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-chat-openai.yaml \
  --namespace ui --create-namespace

kubectl get pod -n ui
```

### Teardown
```bash
helm uninstall ui -n ui
kubectl delete namespace ui
```

## Key Observations

| Scenario | Observed Behaviour |
|----------|-------------------|
| ClusterIP | No external access; `port-forward` required for testing |
| NodePort | Static port 30080 on every node; no cloud dependency |
| LoadBalancer | EKS provisions AWS NLB; EXTERNAL-IP available after ~30s |
| ALB Ingress | Requires AWS LB Controller; ALB DNS available after ~60s |
| Bedrock chat | IAM permission required; chat icon appears in UI header |
| OpenAI chat | `baseUrl` must be reachable from pod; works with self-hosted models |
| HPA | UI scales at 50% CPU (lower threshold than backend services) |
| PDB | Only effective when `replicaCount ≥ 2` |
