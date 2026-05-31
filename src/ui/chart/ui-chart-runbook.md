# UI Chart — Helm Deployment Runbook

I forked this repository from the [AWS Retail Store Sample App](https://github.com/aws-containers/retail-store-sample-app) and extended the `src/ui/chart` directory by authoring platform-specific overlay values files on top of the base `values.yaml`. The overlay files practice two independent axes: **service exposure** (how the UI is accessed) and **AI chat** (which LLM backend powers the in-app assistant). I validated each scenario end-to-end across bare-metal and EKS targets.

> **Deployment pattern:** Every `helm` command passes `values.yaml` (base) first, `values-endpoints.yaml` second (mandatory companion), and the scenario overlay third. Helm deep-merges all three; each file overrides only the keys it declares.

---

## Why `values-endpoints.yaml` Exists

The UI service acts as the front-door aggregator — it calls all four backend services at runtime. Without valid in-cluster endpoints the UI pods start but every page request fails. Rather than hardcoding these endpoints into each scenario file or modifying the base `values.yaml`, a single shared file captures them once and is passed to every `helm` command:

```
values-endpoints.yaml
  app.endpoints.catalog   → http://catalog.catalog.svc.cluster.local:80
  app.endpoints.orders    → http://orders.orders.svc.cluster.local:80
  app.endpoints.carts     → http://cart-carts.cart.svc.cluster.local:80
  app.endpoints.checkout  → http://checkout.checkout.svc.cluster.local:80
```

---

## Values Files — Scenario Matrix

All scenario overlay files are scoped to **service exposure** or **AI chat configuration only**. Database and messaging are not applicable to the UI service.

| File | Axis | Service Type | External Access | Ingress | AI Chat |
|------|------|-------------|:---------------:|:-------:|:-------:|
| `values.yaml` | base | `ClusterIP` | ✗ | ✗ | ✗ |
| `values-endpoints.yaml` | companion | — | — | — | — |
| `values-clusterip.yaml` | exposure | `ClusterIP` | ✗ (port-forward) | ✗ | ✗ |
| `values-nodeport.yaml` | exposure | `NodePort :30080` | ✓ via node IP | ✗ | ✗ |
| `values-loadbalancer.yaml` | exposure | `LoadBalancer` | ✓ cloud NLB | ✗ | ✗ |
| `values-alb-ingress.yaml` | exposure | `ClusterIP` | ✓ AWS ALB | ✓ `alb` | ✗ |
| `values-chat-bedrock.yaml` | chat | — (any) | — | — | ✓ AWS Bedrock |
| `values-chat-openai.yaml` | chat | — (any) | — | — | ✓ OpenAI-compatible |

---

## Commands Per Scenario

### Template Validation (all scenarios)

```bash
helm template ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-endpoints.yaml \
  -f src/ui/chart/<values-file>.yaml
```

### Scenario 1 — ClusterIP + Port-Forward (Baseline)

```bash
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-endpoints.yaml \
  -f src/ui/chart/values-clusterip.yaml \
  --namespace ui --create-namespace

kubectl get pods -n ui
kubectl port-forward svc/ui 8080:80 -n ui
# Access: http://localhost:8080
```

### Scenario 2 — NodePort (Bare-Metal)

```bash
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-endpoints.yaml \
  -f src/ui/chart/values-nodeport.yaml \
  --namespace ui --create-namespace

kubectl get svc -n ui
# Access: http://<node-ip>:30080
```

### Scenario 3 — LoadBalancer (EKS NLB)

```bash
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-endpoints.yaml \
  -f src/ui/chart/values-loadbalancer.yaml \
  --namespace ui --create-namespace

kubectl get svc -n ui -w   # Wait for EXTERNAL-IP
# Access: http://<EXTERNAL-IP>
```

### Scenario 4 — ALB Ingress (EKS)

```bash
# Pre-requisite: AWS Load Balancer Controller installed in the cluster.
# Replace <YOUR_DOMAIN_OR_ALB_DNS> in values-alb-ingress.yaml before running.
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-endpoints.yaml \
  -f src/ui/chart/values-alb-ingress.yaml \
  --namespace ui --create-namespace

kubectl get ingress -n ui -w   # Wait for ADDRESS
```

### Scenario 5 — AI Chat via AWS Bedrock

```bash
# Pre-requisite: IRSA or node instance profile with bedrock:InvokeModel permission.
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-endpoints.yaml \
  -f src/ui/chart/values-chat-bedrock.yaml \
  --namespace ui --create-namespace

kubectl get pod -n ui
kubectl logs -n ui -l app.kubernetes.io/name=ui | grep -i chat
```

### Scenario 6 — AI Chat via OpenAI-Compatible API

```bash
# Works with OpenAI, Ollama, LM Studio, or any OpenAI-compatible endpoint.
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-endpoints.yaml \
  -f src/ui/chart/values-chat-openai.yaml \
  --namespace ui --create-namespace

kubectl get pod -n ui
kubectl logs -n ui -l app.kubernetes.io/name=ui | grep -i chat
```

### Combining Exposure + Chat

```bash
# Example: NodePort access + Bedrock chat enabled simultaneously
helm upgrade --install ui src/ui/chart/ \
  -f src/ui/chart/values.yaml \
  -f src/ui/chart/values-endpoints.yaml \
  -f src/ui/chart/values-nodeport.yaml \
  -f src/ui/chart/values-chat-bedrock.yaml \
  --namespace ui --create-namespace
```

### Teardown

```bash
helm uninstall ui -n ui
kubectl delete namespace ui
```

---

## Key Observations

| Scenario | Observed Behaviour |
|----------|-------------------|
| ClusterIP | No external access; `port-forward` required for local testing |
| NodePort | Static port 30080 on every node; no cloud dependency |
| LoadBalancer | EKS provisions AWS NLB; EXTERNAL-IP available after ~30s |
| ALB Ingress | Requires AWS LB Controller; ALB DNS available after ~60s |
| Bedrock chat | IAM permission required; chat icon appears in UI header |
| OpenAI chat | `baseUrl` must be reachable from the pod; works with self-hosted models too |
| `values-endpoints.yaml` | Mandatory in every deploy; UI pods start but all pages fail without backend endpoints |
| Exposure + chat axes | The two axes are fully independent — any exposure scenario can be combined with any chat scenario |
