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
| [Cart](./src/cart/) | Java | Shopping cart state management | DynamoDB / In-memory | Catalog |
| [Orders](./src/orders/) | Java | Order processing and persistence | PostgreSQL + RabbitMQ or SQS | Cart, Checkout |
| [Checkout](./src/checkout/) | Node.js | Checkout orchestration | Redis / ElastiCache | Orders |

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

Each service ships with its own Helm chart under `src/<service>/chart/`, including a base `values.yaml`. I studied that base file for each service, then authored additional `values-*.yaml` overrides on top of it — one per deployment scenario — so that the same chart could be deployed across three different target environments without modifying the chart itself.

This application was built by AWS primarily for EKS. Extending it to bare-metal Kubernetes is deliberate — it validates environment-agnostic chart design and exercises the full range of platform engineering decisions around storage classes, service exposure, and managed vs. self-hosted dependencies.

> Files marked ✱ are referenced by the Helmfile configurations in Step 2. Each service has a dedicated runbook (last row of each block) with per-scenario `helm` commands, validation steps, and observed behaviour.

| | Values Override File | Target |
|---|---|---|
| **UI** | | |
| | [`values-endpoints.yaml`](./src/ui/chart/values-endpoints.yaml) ✱ | All — mandatory companion |
| | [`values-nodeport.yaml`](./src/ui/chart/values-nodeport.yaml) ✱ | Bare-metal / Local |
| | [`values-clusterip.yaml`](./src/ui/chart/values-clusterip.yaml) | All (port-forward) |
| | [`values-loadbalancer.yaml`](./src/ui/chart/values-loadbalancer.yaml) | EKS — NLB |
| | [`values-alb-ingress.yaml`](./src/ui/chart/values-alb-ingress.yaml) ✱ | EKS — ALB Ingress |
| | [`values-chat-bedrock.yaml`](./src/ui/chart/values-chat-bedrock.yaml) | EKS — AWS Bedrock |
| | [`values-chat-openai.yaml`](./src/ui/chart/values-chat-openai.yaml) | Any — OpenAI-compatible |
| | 📖 [UI Chart Runbook](./src/ui/chart/ui-chart-runbook.md) | scenario matrix · per-scenario commands · exposure + chat axes · teardown |
| **Catalog** | | |
| | [`values-in-memory.yaml`](./src/catalog/chart/values-in-memory.yaml) | Local |
| | [`values-mysql-ephemeral.yaml`](./src/catalog/chart/values-mysql-ephemeral.yaml) ✱ | Bare-metal — ephemeral |
| | [`values-mysql-pvc-baremetal.yaml`](./src/catalog/chart/values-mysql-pvc-baremetal.yaml) ✱ | Bare-metal — persistent |
| | [`values-mysql-pvc-eks.yaml`](./src/catalog/chart/values-mysql-pvc-eks.yaml) ✱ | EKS — EBS PVC |
| | [`values-external-mysql.yaml`](./src/catalog/chart/values-external-mysql.yaml) | EKS — RDS endpoint |
| | 📖 [Catalog Chart Runbook](./src/catalog/chart/catalog-chart-runbook.md) | per-scenario commands · MySQL connectivity validation · teardown |
| **Cart** | | |
| | [`values-in-memory.yaml`](./src/cart/chart/values-in-memory.yaml) | Local |
| | [`values-dynamodb-local.yaml`](./src/cart/chart/values-dynamodb-local.yaml) ✱ | Bare-metal / Local |
| | [`values-dynamodb-aws.yaml`](./src/cart/chart/values-dynamodb-aws.yaml) ✱ | EKS — AWS DynamoDB (IRSA) |
| | 📖 [Cart Chart Runbook](./src/cart/chart/cart-chart-runbook.md) | per-scenario commands · DynamoDB table validation · teardown |
| **Orders** | | |
| | [`values-01-in-memory.yaml`](./src/orders/chart/values-01-in-memory.yaml) | Local — zero dependencies |
| | [`values-02-postgresql-ephemeral-msg-in-memory.yaml`](./src/orders/chart/values-02-postgresql-ephemeral-msg-in-memory.yaml) ✱ | Bare-metal — ephemeral |
| | [`values-03-postgresql-rabbitmq-pvc-baremetal.yaml`](./src/orders/chart/values-03-postgresql-rabbitmq-pvc-baremetal.yaml) ✱ | Bare-metal — persistent |
| | [`values-04-postgresql-rabbitmq-pvc-eks.yaml`](./src/orders/chart/values-04-postgresql-rabbitmq-pvc-eks.yaml) | EKS — RabbitMQ + EBS PVC |
| | [`values-05-postgresql-rabbitmq-external.yaml`](./src/orders/chart/values-05-postgresql-rabbitmq-external.yaml) | EKS — fully external endpoints |
| | [`values-06-postgresql-pvc-eks-sqs.yaml`](./src/orders/chart/values-06-postgresql-pvc-eks-sqs.yaml) ✱ | EKS — AWS SQS + EBS PVC |
| | 📖 [Orders Chart Runbook](./src/orders/chart/orders-chart-runbook.md) | per-scenario commands · numbered progression rationale · broker validation · teardown |
| **Checkout** | | |
| | [`values-in-memory.yaml`](./src/checkout/chart/values-in-memory.yaml) | Local |
| | [`values-redis-local.yaml`](./src/checkout/chart/values-redis-local.yaml) ✱ | Bare-metal / Local |
| | [`values-redis-tls.yaml`](./src/checkout/chart/values-redis-tls.yaml) | Any — Redis with TLS |
| | [`values-redis-aws-elasticache.yaml`](./src/checkout/chart/values-redis-aws-elasticache.yaml) | EKS — ElastiCache |
| | 📖 [Checkout Chart Runbook](./src/checkout/chart/checkout-chart-runbook.md) | per-scenario commands · Redis connectivity · TLS configuration · teardown |

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
