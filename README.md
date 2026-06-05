# Retail Store Sample App — DevOps Engineering

> **Authorship Notice:** I did not write this application from scratch. The source code across all five services (`src/`) was originally developed by the [AWS Containers](https://github.com/aws-containers/retail-store-sample-app) team as an educational reference for container-based architectures on AWS. As a DevOps / Platform Engineer, my work begins where the application code ends — understanding the architecture, studying the Helm charts, authoring environment-specific `values-*.yaml` overrides and runbooks for each service, orchestrating multi-service deployments with Helmfile, and validating the full stack via Terraform-provisioned infrastructure.

---

## Table of Contents

- [Application Architecture](#application-architecture)
- [Step 0 — Architecture Comprehension](#step-0--architecture-comprehension)
- [Step 1 — Per-Service Helm Chart Deployment](#step-1--per-service-helm-chart-deployment)
- [Step 2 — Helmfile Unified Deployment](#step-2--helmfile-unified-deployment)
- [Step 3 — Terraform Infrastructure Deployment](#step-3--terraform-infrastructure-deployment)

---

## Application Architecture

This is a deliberately over-engineered microservices retail store — five independent services, each written in a different language, each with its own persistence backend. The complexity is intentional: it mirrors the kind of heterogeneous stack you encounter in real-world platform engineering.

![Architecture](./docs/images/architecture.png)

| Service | Language | Role | Database(s) | Depends On |
|---|---|---|---|---|
| [UI](./src/ui/) | Java | Store frontend — routes all user traffic | None | Catalog, Cart, Orders, Checkout |
| [Catalog](./src/catalog/) | Go | Product catalog REST API | MySQL / MariaDB | None |
| [Cart](./src/cart/) | Java | Shopping cart state management | DynamoDB (local or AWS) / In-memory | Catalog |
| [Orders](./src/orders/) | Java | Order processing and persistence | PostgreSQL + RabbitMQ or SQS | Cart, Checkout |
| [Checkout](./src/checkout/) | Node.js | Checkout orchestration | Redis (local, TLS, or ElastiCache) | Orders |

> Pre-built container images for both `x86-64` and `ARM64` are available on [Amazon ECR Public Gallery](https://gallery.ecr.aws/aws-containers).

---

## Step 0 — Architecture Comprehension

Before writing a single line of deployment configuration, I studied the codebase and its Helm charts thoroughly. This step is non-negotiable — deploying something you do not understand is not DevOps, it is guesswork.

**What I mapped out:**

- How many services exist and what each one does
- Which programming language each service is written in
- Which database(s) each service can connect to (some support multiple backends)
- Which services depend on other services and in what direction
- The structure of every `chart/` folder under `src/<service>/`
- The base `values.yaml` for each chart — what is exposed, what is hardcoded, what is configurable

This comprehension work directly informed every `values-*.yaml` file authored in Step 1.

---

## Step 1 — Per-Service Helm Chart Deployment

Each service ships with its own Helm chart under `src/<service>/chart/`, including a base `values.yaml`. I studied that base file for each service, then authored additional `values-*.yaml` override files on top of it — one per deployment scenario — so that the same chart could be deployed across three different target environments without modifying the chart itself.

This application was built by AWS primarily for EKS. Extending it to bare-metal Kubernetes is deliberate — it validates that the charts are environment-agnostic and exercises the full range of platform engineering decisions around storage, networking, and managed vs. self-hosted dependencies.

**Three deployment targets:**

| # | Target | Characteristics |
|---|---|---|
| 1 | **Local Cluster** | Kind / Minikube; ephemeral storage; in-cluster or in-memory dependencies |
| 2 | **Bare-metal Kubernetes** | Self-managed nodes (k3s / kubeadm); PVC-backed storage with `local-path` StorageClass; in-cluster databases |
| 3 | **AWS EKS** | Managed node groups; EBS PVC via `gp2` StorageClass; AWS-managed services (RDS, DynamoDB, ElastiCache, SQS, ALB) |

> Files marked with ✱ are the ones referenced by the Helmfile configurations in Step 2.

| Service | Values Override File | Purpose / Notes |
|---|---|---|
| **UI** | [`values-nodeport.yaml`](./src/ui/chart/values-nodeport.yaml) ✱ | Expose UI via NodePort — bare-metal and local clusters where no cloud LB exists |
| | [`values-clusterip.yaml`](./src/ui/chart/values-clusterip.yaml) | ClusterIP only — use when UI is fronted by a separate Ingress controller |
| | [`values-loadbalancer.yaml`](./src/ui/chart/values-loadbalancer.yaml) | Cloud LoadBalancer service type — EKS with NLB |
| | [`values-alb-ingress.yaml`](./src/ui/chart/values-alb-ingress.yaml) ✱ | AWS ALB Ingress Controller — EKS with AWS Load Balancer Controller installed |
| | [`values-endpoints.yaml`](./src/ui/chart/values-endpoints.yaml) ✱ | Kubernetes Endpoints object for cross-namespace service routing |
| | [`values-chat-bedrock.yaml`](./src/ui/chart/values-chat-bedrock.yaml) | Enable the generative AI chat feature via AWS Bedrock |
| | [`values-chat-openai.yaml`](./src/ui/chart/values-chat-openai.yaml) | Enable the generative AI chat feature via OpenAI API |
| | 📖 [UI Chart Runbook](./src/ui/chart/ui-chart-runbook.md) | |
| **Catalog** | [`values-in-memory.yaml`](./src/catalog/chart/values-in-memory.yaml) | No MySQL — catalog data served from memory; fastest local spin-up |
| | [`values-mysql-ephemeral.yaml`](./src/catalog/chart/values-mysql-ephemeral.yaml) ✱ | MySQL deployed in-cluster with no PVC — bare-metal ephemeral scenario |
| | [`values-mysql-pvc-baremetal.yaml`](./src/catalog/chart/values-mysql-pvc-baremetal.yaml) ✱ | MySQL with PVC using `local-path` StorageClass — bare-metal persistent |
| | [`values-mysql-pvc-eks.yaml`](./src/catalog/chart/values-mysql-pvc-eks.yaml) ✱ | MySQL with PVC using `gp2` EBS StorageClass — EKS |
| | [`values-external-mysql.yaml`](./src/catalog/chart/values-external-mysql.yaml) | Point catalog at an external MySQL endpoint (e.g. RDS) — no in-cluster DB |
| | 📖 [Catalog Chart Runbook](./src/catalog/chart/catalog-chart-runbook.md) | |
| **Cart** | [`values-in-memory.yaml`](./src/cart/chart/values-in-memory.yaml) | No DynamoDB — cart state held in memory; resets on pod restart |
| | [`values-dynamodb-local.yaml`](./src/cart/chart/values-dynamodb-local.yaml) ✱ | DynamoDB Local running as a sidecar/container in-cluster — local and bare-metal |
| | [`values-dynamodb-aws.yaml`](./src/cart/chart/values-dynamodb-aws.yaml) ✱ | AWS DynamoDB managed service — EKS with IAM role for service account (IRSA) |
| | 📖 [Cart Chart Runbook](./src/cart/chart/cart-chart-runbook.md) | |
| **Orders** | [`values-01-in-memory.yaml`](./src/orders/chart/values-01-in-memory.yaml) | Fully in-memory — no PostgreSQL, no message broker; zero external dependencies |
| | [`values-02-postgresql-ephemeral-msg-in-memory.yaml`](./src/orders/chart/values-02-postgresql-ephemeral-msg-in-memory.yaml) ✱ | PostgreSQL in-cluster (no PVC) + messaging in-memory — bare-metal ephemeral |
| | [`values-03-postgresql-rabbitmq-pvc-baremetal.yaml`](./src/orders/chart/values-03-postgresql-rabbitmq-pvc-baremetal.yaml) ✱ | PostgreSQL + RabbitMQ, both PVC-backed with `local-path` — bare-metal persistent |
| | [`values-04-postgresql-rabbitmq-pvc-eks.yaml`](./src/orders/chart/values-04-postgresql-rabbitmq-pvc-eks.yaml) | PostgreSQL + RabbitMQ, both PVC-backed with `gp2` — EKS |
| | [`values-05-postgresql-rabbitmq-external.yaml`](./src/orders/chart/values-05-postgresql-rabbitmq-external.yaml) | External PostgreSQL + external RabbitMQ endpoints — fully managed, no in-cluster DB |
| | [`values-06-postgresql-pvc-eks-sqs.yaml`](./src/orders/chart/values-06-postgresql-pvc-eks-sqs.yaml) ✱ | PostgreSQL with EBS PVC + AWS SQS as message broker — EKS native |
| | 📖 [Orders Chart Runbook](./src/orders/chart/orders-chart-runbook.md) | |
| **Checkout** | [`values-in-memory.yaml`](./src/checkout/chart/values-in-memory.yaml) | No Redis — session state held in memory |
| | [`values-redis-local.yaml`](./src/checkout/chart/values-redis-local.yaml) ✱ | Redis deployed in-cluster — local and bare-metal targets |
| | [`values-redis-tls.yaml`](./src/checkout/chart/values-redis-tls.yaml) | Redis with TLS — when cert-manager or a TLS-terminating proxy is in place |
| | [`values-redis-aws-elasticache.yaml`](./src/checkout/chart/values-redis-aws-elasticache.yaml) | AWS ElastiCache (managed Redis) — EKS |
| | 📖 [Checkout Chart Runbook](./src/checkout/chart/checkout-chart-runbook.md) | |

> The numbered prefix on Orders files (`01-`, `02-`, ...) is intentional — it communicates a recommended progression from zero dependencies to full production-grade persistence, making it straightforward to validate each layer before introducing the next.

---

## Step 2 — Helmfile Unified Deployment

Deploying five services independently with `helm install` per service is workable for exploration, but operationally fragile — release ordering, dependency sequencing, and teardown all become manual. In this step I introduced [Helmfile](https://helmfile.readthedocs.io/) to declare all five releases in a single manifest, enforce their deployment order via `needs:`, and operate the full stack with one command.

Three Helmfile configurations were authored, one per deployment target. Each one references the `values-*.yaml` files marked ✱ in Step 1 above.

| File | Target | Storage | Message Broker | UI Exposure |
|---|---|---|---|---|
| [`helmfile-baremetal-ephemeral.yaml`](./helmfile/helmfile-baremetal-ephemeral.yaml) | Bare-metal Kubernetes | Ephemeral (no PVC) | In-memory | NodePort |
| [`helmfile-baremetal-persistent.yaml`](./helmfile/helmfile-baremetal-persistent.yaml) | Bare-metal Kubernetes | `local-path` PVC | RabbitMQ | NodePort |
| [`helmfile-eks.yaml`](./helmfile/helmfile-eks.yaml) | AWS EKS | `gp2` EBS PVC | AWS SQS | ALB Ingress |

📖 [Helmfile README](./helmfile/README.md)

**Deploy the full stack:**

```bash
# Bare-metal — ephemeral
helmfile -f helmfile/helmfile-baremetal-ephemeral.yaml apply

# Bare-metal — persistent
helmfile -f helmfile/helmfile-baremetal-persistent.yaml apply

# AWS EKS
helmfile -f helmfile/helmfile-eks.yaml apply
```

**Teardown:**

```bash
helmfile -f helmfile/helmfile-eks.yaml destroy
```

---

## Step 3 — Terraform Infrastructure Deployment

The upstream repository ships Terraform modules for provisioning and deploying the full stack on AWS-managed compute. I used these scripts to validate end-to-end deployment from infrastructure provisioning through application availability.

| Module | Target | AWS Services Used |
|---|---|---|
| [`terraform/eks/`](./terraform/eks/) | Amazon EKS | EKS, RDS (MySQL/PostgreSQL), DynamoDB, ElastiCache, SQS, ALB |
| [`terraform/ecs/`](./terraform/ecs/) | Amazon ECS (Fargate) | ECS, RDS, DynamoDB, ElastiCache, SQS, ALB |
| [`terraform/apprunner/`](./terraform/apprunner/) | AWS App Runner | App Runner, RDS, DynamoDB, ElastiCache |

> These Terraform modules are authored by the AWS Containers team. My contribution in this step was execution, validation, and observing how the infrastructure choices map back to the Helm chart override files authored in Steps 1 and 2.

**Basic workflow (EKS example):**

```bash
cd terraform/eks/default

terraform init
terraform plan
terraform apply
```

**Destroy all resources when done:**

```bash
terraform destroy
```

---

## Repository Layout

```
retail-store-sample-app/
├── src/
│   ├── ui/chart/               # UI Helm chart + values overrides + runbook
│   ├── catalog/chart/          # Catalog Helm chart + values overrides + runbook
│   ├── cart/chart/             # Cart Helm chart + values overrides + runbook
│   ├── orders/chart/           # Orders Helm chart + values overrides + runbook
│   └── checkout/chart/         # Checkout Helm chart + values overrides + runbook
├── helmfile/                   # Helmfile configs for all 3 deployment targets
├── terraform/
│   ├── eks/                    # EKS deployment (default + minimal)
│   ├── ecs/                    # ECS deployment
│   └── apprunner/              # App Runner deployment
└── docs/                       # Architecture diagrams, feature documentation
```

---

## License

The original application and Terraform modules are licensed under the [MIT-0 License](https://github.com/aws-containers/retail-store-sample-app/blob/main/LICENSE) by Amazon Web Services. All DevOps additions in this fork (values overrides, runbooks, Helmfile configurations) are authored by [Muhammad Ibtisam Iqbal](https://github.com/ibtisam-iq).
