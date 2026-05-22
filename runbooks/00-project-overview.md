# Retail Store Sample App — Project Overview

> **Fork of:** [aws-containers/retail-store-sample-app](https://github.com/aws-containers/retail-store-sample-app)  
> **Purpose:** Personal DevOps portfolio lab — applying real-world Helm, Kubernetes, CI/CD, and cloud infrastructure practices on a production-grade microservices codebase.

---

## Why This Repo Was Forked

AWS Containers built this project specifically so that students, engineers, and DevOps practitioners could deploy a realistic multi-service application without needing to build one from scratch. The original repo is designed for learning container orchestration and deployment patterns on AWS.

This fork uses the same codebase as the foundation to practice and document:
- Helm chart customization and multi-scenario values files
- Kubernetes deployment strategies (HPA, PDB, persistent volumes)
- Understanding how each microservice is structured, configured, and connected

Everything added in this fork lives in the `runbooks/` folder and inside `src/<service>/chart/` — no upstream application code has been modified.

---

## Repository Structure

| Path | Owner | Purpose |
|------|-------|---------|
| `.github/` | Upstream | GitHub Actions workflows: `artifacts.yaml`, `e2e-test.yml`, `pr.yaml`, `release-please.yml`, `oss.yml`, `publish-build.yml` |
| `.github/actions/` | Upstream | Reusable composite actions used by the workflows above |
| `docs/` | Upstream | Architecture diagrams (`diagrams.drawio.xml`), feature documentation (`features.md`), and screenshot images |
| `src/` | Upstream + **This fork** | All five microservice source trees; Helm charts live inside each service folder |
| `terraform/` | Upstream | IaC for deploying to EKS, ECS, and App Runner (`eks/`, `ecs/`, `apprunner/`, `lib/`) |
| `samples/` | Upstream | Sample data and images used by the application |
| `scripts/` | Upstream | Build/release automation: `compose-dist.sh`, `kubernetes-dist.sh`, `e2e*.sh`, `create-manifest.sh`, `add-licenses.sh`, `generate-security-report.sh` |
| `oss/` | Upstream | Open-source license attribution (`attribution/`, `ort/`) and `run.sh` for OSS scanning |
| `runbooks/` | **This fork** | DevOps runbooks written while studying this codebase — one per service plus this overview |
| `CHANGELOG.md` | Upstream | Auto-generated release history via `release-please` |
| `CONTRIBUTING.md` | Upstream | Contribution and security reporting guidelines |
| `DEVELOPER_GUIDE.md` | Upstream | Local development setup instructions |
| `nx.json` | Upstream | Nx monorepo configuration — orchestrates builds across all services |
| `package.json` | Upstream | Yarn workspace root, defines monorepo-level scripts and tooling |
| `.mise.toml` | Upstream | Tool version pinning (Go, Node, Java, Helm, etc.) via `mise` |
| `lefthook.yml` | Upstream | Git hook configuration for pre-commit checks (Prettier formatting) |
| `renovate.json` | Upstream | Automated dependency update bot configuration |
| `.mergify.yml` | Upstream | Auto-merge rules for Renovate PRs that pass CI |
| `.release-please-manifest.json` | Upstream | Tracks the current release version (`"." = "1.x.x"`) |
| `release-please-config.json` | Upstream | Release-please strategy configuration |
| `.prettierrc` / `.prettierignore` | Upstream | Code formatting rules enforced via lefthook |
| `.yarn/` / `.yarnrc.yml` | Upstream | Yarn Berry (v4) configuration |

---

## Microservices — Quick Reference

| Service | Language | DB / Backend | Port | Helm Chart |
|---------|----------|-------------|------|-----------|
| [UI](./src/ui/) | Java (Spring Boot) | None (calls other services) | 8080 | `src/ui/chart/` |
| [Catalog](./src/catalog/) | Go | MySQL / MariaDB | 8080 | `src/catalog/chart/` |
| [Cart](./src/cart/) | Java (Spring Boot) | DynamoDB or Redis | 8080 | `src/cart/chart/` |
| [Orders](./src/orders/) | Java (Spring Boot) | MySQL / PostgreSQL + RabbitMQ | 8080 | `src/orders/chart/` |
| [Checkout](./src/checkout/) | Node.js | Redis (sessions) | 8080 | `src/checkout/chart/` |

All services expose Prometheus metrics and support OpenTelemetry OTLP tracing. Each service also ships a `docker-compose.yml` for local standalone testing.

---

## Deployment Modes Supported (Upstream)

| Mode | How |
|------|-----|
| Single container (demo) | `docker run public.ecr.aws/aws-containers/retail-store-sample-ui:1.0.0` |
| Docker Compose (local full stack) | `docker-compose.yaml` released with each version |
| Kubernetes (plain manifests) | `kubectl apply -f kubernetes.yaml` from releases |
| EKS via Terraform | `terraform/eks/default/` — uses RDS, DynamoDB, etc. |
| ECS via Terraform | `terraform/ecs/default/` |
| App Runner via Terraform | `terraform/apprunner/` |

---

## GitHub Actions Workflows (Upstream)

| Workflow | Trigger | What It Does |
|----------|---------|-------------|
| `artifacts.yaml` | Push to main / tags | Builds and pushes container images + Helm charts to ECR |
| `e2e-test.yml` | PR / push | Runs end-to-end tests on Kind cluster |
| `pr.yaml` | Pull request | Runs lint and build validation |
| `release-please.yml` | Push to main | Auto-creates release PRs and changelogs |
| `publish-build.yml` | Manual / release | Triggers artifact publishing |
| `oss.yml` | Scheduled | Runs OSS license scanning via ORT |

---

## What This Fork Adds

### `src/catalog/chart/` — Completed ✅

| File | Scenario |
|------|----------|
| `values.yaml` | Upstream default — untouched |
| `values-in-memory.yaml` | No DB; confirms chart renders without persistence |
| `values-mysql-ephemeral.yaml` | In-cluster MySQL, no PVC (dev/test) |
| `values-mysql-pvc.yaml` | In-cluster MySQL with persistent volume |
| `values-external-mysql.yaml` | External MySQL endpoint; no in-cluster DB pod |
| `values-hpa.yaml` | Horizontal Pod Autoscaler enabled (min 2 / max 5, 70% CPU) |
| `values-pdb.yaml` | Pod Disruption Budget enabled (3 replicas, minAvailable 2) |
| `catalog-chart-runbook.md` | Scenario matrix, helm commands, key observations |

### `runbooks/` — In Progress 🔄

| File | Status |
|------|--------|
| `00-project-overview.md` | ✅ This file |
| `01-catalog-service.md` | 🔜 Next |
| `02-cart-service.md` | 🔜 Planned |
| `03-orders-service.md` | 🔜 Planned |
| `04-checkout-service.md` | 🔜 Planned |
| `05-ui-service.md` | 🔜 Planned |

---

## How to Study Each Service

1. Read `src/<service>/README.md` — configuration env vars and dependencies
2. Read `src/<service>/chart/values.yaml` — understand all tunable parameters
3. Read the matching `runbooks/<N>-<service>-service.md` — explains values, scenarios, and helm commands
4. Apply a custom values file to a real cluster using `-f values-<scenario>.yaml`
