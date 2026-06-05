# Retail Store Sample App — DevOps Engineering

> **Authorship Notice:** I did not write this application from scratch. The source code across all five services (`src/`) was originally developed by the [AWS Containers](https://github.com/aws-containers/retail-store-sample-app) team as an educational reference for container-based architectures on AWS. As a DevOps / Platform Engineer, my work begins where the application code ends — understanding the architecture, studying the Helm charts, authoring environment-specific `values-*.yaml` overrides and runbooks for each service, orchestrating multi-service deployments with Helmfile, and validating the full stack via Terraform-provisioned infrastructure.

---

## Table of Contents

- [Application Architecture](#application-architecture)
- [Step 0 — Architecture Comprehension](#step-0--architecture-comprehension)
- [Step 1 — Per-Service Helm Chart Deployment](#step-1--per-service-helm-chart-deployment)
  - [UI](#-ui-service)
  - [Catalog](#-catalog-service)
  - [Cart](#-cart-service)
  - [Orders](#-orders-service)
  - [Checkout](#-checkout-service)
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

This comprehension work directly informed the environment-specific `values-*.yaml` files authored in Step 1.

---

## Step 1 — Per-Service Helm Chart Deployment

Each service ships with its own Helm chart under `src/<service>/chart/`. The upstream chart provides a base `values.yaml`, but deploying across three different target environments — **Local Cluster**, **Bare-metal Kubernetes**, and **AWS EKS** — requires environment-specific overrides. I authored those override files and a runbook for each service.

**Three deployment targets:**

| # | Target | Characteristics |
|---|---|---|
| 1 | **Local Cluster** | Kind / Minikube; no persistent storage; in-memory or containerized dependencies |
| 2 | **Bare-metal Kubernetes** | Self-managed nodes; PVC-backed storage; in-cluster databases |
| 3 | **AWS EKS** | Managed node groups; AWS-managed services (RDS, DynamoDB, ElastiCache, SQS, ALB) |

---

### 🖥️ UI Service

The UI is the only internet-facing service. Its Helm chart controls how it is exposed — the `values-*.yaml` files here are primarily about **service type and ingress strategy**, not databases.

| File | Purpose | Target |
|---|---|---|
| [`values-nodeport.yaml`](./src/ui/chart/values-nodeport.yaml) | Expose via NodePort | Local Cluster / Bare-metal |
| [`values-clusterip.yaml`](./src/ui/chart/values-clusterip.yaml) | ClusterIP only (internal) | All (when fronted by Ingress) |
| [`values-loadbalancer.yaml`](./src/ui/chart/values-loadbalancer.yaml) | Cloud LoadBalancer service | AWS EKS (NLB) |
| [`values-alb-ingress.yaml`](./src/ui/chart/values-alb-ingress.yaml) | AWS ALB Ingress Controller | AWS EKS |
| [`values-endpoints.yaml`](./src/ui/chart/values-endpoints.yaml) | Kubernetes Endpoints object for external routing | Mixed / Advanced |
| [`values-chat-bedrock.yaml`](./src/ui/chart/values-chat-bedrock.yaml) | Enable AI chat via AWS Bedrock | AWS EKS |
| [`values-chat-openai.yaml`](./src/ui/chart/values-chat-openai.yaml) | Enable AI chat via OpenAI API | Any |

📖 [UI Chart Runbook](./src/ui/chart/ui-chart-runbook.md)

---

### 📦 Catalog Service

Catalog is a Go REST API backed by MySQL. The override files cover the full spectrum from no database at all (in-memory) to a fully external managed RDS instance.

| File | Database Mode | Target |
|---|---|---|
| [`values-in-memory.yaml`](./src/catalog/chart/values-in-memory.yaml) | No MySQL; in-memory data | Local Cluster |
| [`values-mysql-ephemeral.yaml`](./src/catalog/chart/values-mysql-ephemeral.yaml) | MySQL deployed in-cluster, no PVC | Local Cluster |
| [`values-mysql-pvc-baremetal.yaml`](./src/catalog/chart/values-mysql-pvc-baremetal.yaml) | MySQL with PVC (bare-metal StorageClass) | Bare-metal Kubernetes |
| [`values-mysql-pvc-eks.yaml`](./src/catalog/chart/values-mysql-pvc-eks.yaml) | MySQL with PVC (EBS StorageClass) | AWS EKS |
| [`values-external-mysql.yaml`](./src/catalog/chart/values-external-mysql.yaml) | External MySQL / RDS endpoint | AWS EKS / Any |

📖 [Catalog Chart Runbook](./src/catalog/chart/catalog-chart-runbook.md)

---

### 🛒 Cart Service

Cart is a Java service that can run with zero external dependencies (in-memory mode) or connect to DynamoDB — either the AWS-managed service or a containerized local replica.

| File | Database Mode | Target |
|---|---|---|
| [`values-in-memory.yaml`](./src/cart/chart/values-in-memory.yaml) | No DynamoDB; in-memory cart state | Local Cluster |
| [`values-dynamodb-local.yaml`](./src/cart/chart/values-dynamodb-local.yaml) | DynamoDB Local (containerized) | Local Cluster / Bare-metal |
| [`values-dynamodb-aws.yaml`](./src/cart/chart/values-dynamodb-aws.yaml) | AWS DynamoDB (managed service) | AWS EKS |

📖 [Cart Chart Runbook](./src/cart/chart/cart-chart-runbook.md)

---

### 📋 Orders Service

Orders is the most complex service — it depends on both a relational database (PostgreSQL) and a message broker (RabbitMQ or AWS SQS). The override files here reflect the highest number of deployment permutations in the stack.

| File | Database | Message Broker | Storage | Target |
|---|---|---|---|---|
| [`values-01-in-memory.yaml`](./src/orders/chart/values-01-in-memory.yaml) | In-memory | In-memory | None | Local Cluster |
| [`values-02-postgresql-ephemeral-msg-in-memory.yaml`](./src/orders/chart/values-02-postgresql-ephemeral-msg-in-memory.yaml) | PostgreSQL (ephemeral) | In-memory | None | Local Cluster |
| [`values-03-postgresql-rabbitmq-pvc-baremetal.yaml`](./src/orders/chart/values-03-postgresql-rabbitmq-pvc-baremetal.yaml) | PostgreSQL + PVC | RabbitMQ | Bare-metal PVC | Bare-metal Kubernetes |
| [`values-04-postgresql-rabbitmq-pvc-eks.yaml`](./src/orders/chart/values-04-postgresql-rabbitmq-pvc-eks.yaml) | PostgreSQL + PVC | RabbitMQ | EBS PVC | AWS EKS |
| [`values-05-postgresql-rabbitmq-external.yaml`](./src/orders/chart/values-05-postgresql-rabbitmq-external.yaml) | External PostgreSQL | External RabbitMQ | None | AWS EKS / Any |
| [`values-06-postgresql-pvc-eks-sqs.yaml`](./src/orders/chart/values-06-postgresql-pvc-eks-sqs.yaml) | PostgreSQL + PVC | AWS SQS | EBS PVC | AWS EKS |

> The numbered prefix (`01-`, `02-`, ...) on Orders files is intentional — it communicates a recommended progression from simplest to most production-like, making it easy to validate each layer before adding the next dependency.

📖 [Orders Chart Runbook](./src/orders/chart/orders-chart-runbook.md)

---

### 💳 Checkout Service

Checkout is a Node.js service that uses Redis as a session/state store. The overrides cover the full range from no Redis at all to TLS-encrypted connections and AWS ElastiCache.

| File | Redis Mode | Target |
|---|---|---|
| [`values-in-memory.yaml`](./src/checkout/chart/values-in-memory.yaml) | No Redis; in-memory state | Local Cluster |
| [`values-redis-local.yaml`](./src/checkout/chart/values-redis-local.yaml) | Redis deployed in-cluster | Local Cluster / Bare-metal |
| [`values-redis-tls.yaml`](./src/checkout/chart/values-redis-tls.yaml) | Redis with TLS enabled | Bare-metal / EKS |
| [`values-redis-aws-elasticache.yaml`](./src/checkout/chart/values-redis-aws-elasticache.yaml) | AWS ElastiCache (managed Redis) | AWS EKS |

📖 [Checkout Chart Runbook](./src/checkout/chart/checkout-chart-runbook.md)

---

## Step 2 — Helmfile Unified Deployment

Deploying five services independently with `helm install` per service is workable for learning, but operationally fragile. In this step I introduced [Helmfile](https://helmfile.readthedocs.io/) to declare all five releases in a single file, resolve their ordering, and deploy or teardown the entire stack with one command.

Three Helmfile configurations were authored, one per deployment target:

| File | Target | Storage | Message Broker |
|---|---|---|---|
| [`helmfile-baremetal-ephemeral.yaml`](./helmfile/helmfile-baremetal-ephemeral.yaml) | Bare-metal Kubernetes | Ephemeral (no PVC) | In-memory |
| [`helmfile-baremetal-persistent.yaml`](./helmfile/helmfile-baremetal-persistent.yaml) | Bare-metal Kubernetes | PVC-backed | RabbitMQ |
| [`helmfile-eks.yaml`](./helmfile/helmfile-eks.yaml) | AWS EKS | EBS PVC + AWS services | RabbitMQ / SQS |

Each Helmfile references the per-service `values-*.yaml` files authored in Step 1 — it is not a standalone configuration, it is an orchestration layer on top of the existing override files.

📖 [Helmfile README](./helmfile/README.md)

**Deploy the full stack (example — bare-metal persistent):**

```bash
helmfile -f helmfile/helmfile-baremetal-persistent.yaml apply
```

**Teardown:**

```bash
helmfile -f helmfile/helmfile-baremetal-persistent.yaml destroy
```

---

## Step 3 — Terraform Infrastructure Deployment

The upstream repository ships Terraform modules for provisioning and deploying the full stack on AWS-managed compute. I used these scripts to validate end-to-end deployment from infrastructure provisioning through application availability.

| Module | Target | AWS Services Used |
|---|---|---|
| [`terraform/eks/`](./terraform/eks/) | Amazon EKS | EKS, RDS (MySQL/PostgreSQL), DynamoDB, ElastiCache, SQS, ALB |
| [`terraform/ecs/`](./terraform/ecs/) | Amazon ECS (Fargate) | ECS, RDS, DynamoDB, ElastiCache, SQS, ALB |
| [`terraform/apprunner/`](./terraform/apprunner/) | AWS App Runner | App Runner, RDS, DynamoDB, ElastiCache |

> These Terraform modules are authored by the AWS Containers team. My contribution in this step was execution, validation, and observing how the infrastructure choices map back to the Helm chart override files I authored in Steps 1 and 2.

**Basic workflow (EKS example):**

```bash
cd terraform/eks/default

terraform init
terraform plan
terraform apply
```

Destroy all resources when done:

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
