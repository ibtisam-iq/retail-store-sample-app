# Retail Store Sample App — DevOps Engineering

> I forked this repository from the [AWS Retail Store Sample App](https://github.com/aws-containers/retail-store-sample-app). As a DevOps / Platform Engineer, my work begins with understanding the architecture, studying the Helm charts, authoring environment-specific `values-*.yaml` overrides and runbooks for each service, orchestrating multi-service deployments with Helmfile, and validating the full stack via Terraform-provisioned infrastructure.

---

## Table of Contents

- [Application Architecture](#application-architecture)
- [Step 0 — Architecture Comprehension](#step-0--architecture-comprehension)
- [Lab Environment](#lab-environment)
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

---

## Step 0 — Architecture Comprehension

Before writing a single line of deployment configuration, I studied the codebase and its Helm charts thoroughly. This step is non-negotiable — deploying something that I do not understand is not DevOps, it is guesswork.

**What I mapped out:**

- How many services exist and what each one does
- Which programming language each service is written in
- Which database(s) each service can connect to (some support multiple backends)
- Which services depend on other services and in what direction
- The structure of every `chart/` folder under `src/<service>/`
- The base `values.yaml` for each chart — what is exposed, what is hardcoded, what is configurable

This comprehension work directly informed every `values-*.yaml` file authored in Step 1.

---

## Lab Environment

All work in this repository was done on infrastructure I provisioned myself — no local VM, no personal AWS account, no pre-configured cloud instance.

### Bare-metal Kubernetes

I hold a lifetime premium membership on [iximiuz Labs](https://labs.iximiuz.com/a/ibtisam-iq), a platform that lets you author and publish custom micro-VM playgrounds. I have built and published a family of these under the **SilverStack** name — each a purpose-built Ubuntu 24.04 micro-VM with a specific tool stack pre-installed and pre-wired. For all bare-metal work, I used the [SilverStack Dev Machine](https://labs.iximiuz.com/playgrounds/SilverStack-dev-machine-e672bcf7) — a fully provisioned DevOps workstation with every required tool already available, zero setup needed on my end.

On top of that machine, I provisioned a multi-node kubeadm cluster using my own bootstrap script:

```bash
curl -fsSL https://raw.githubusercontent.com/ibtisam-iq/silver-stack/main/scripts/kubernetes/entrypoints/init-controlplane.sh | sudo bash
```

Every step this script automates — container runtime installation, CNI setup, kubeadm init, and post-install configuration — is documented in the [Cluster Bootstrap Runbook](https://runbook.ibtisam-iq.com/bootstrap/kubernetes/cluster-kubeadm/).

### AWS EKS

For EKS validation, I used the [KodeKloud AWS Playground](https://learn.kodekloud.com/user/playgrounds/playground-aws) (annual membership) rather than a personal AWS account. This is a sandboxed environment with restricted IAM permissions and limited service quotas — you cannot simply run `terraform apply` and expect a clean EKS cluster. I hit real provisioning constraints and worked through each one. Everything I encountered and how I resolved it is documented in the [EKS on KodeKloud Playground Runbook](https://runbook.ibtisam-iq.com/iac/terraform/provisioning/eks-on-kodekloud-aws-playground/).

| | Bare-metal | EKS |
|---|---|---|
| **Platform** | [iximiuz Labs](https://labs.iximiuz.com/a/ibtisam-iq) — lifetime premium | [KodeKloud AWS Playground](https://learn.kodekloud.com/user/playgrounds/playground-aws) — annual membership |
| **Machine** | [SilverStack Dev Machine](https://labs.iximiuz.com/playgrounds/SilverStack-dev-machine-e672bcf7) — custom micro-VM, all tools pre-installed | Sandboxed AWS account — restricted IAM, limited quotas |
| **Cluster** | kubeadm — provisioned via my own bootstrap script | EKS — provisioned via Terraform |
| **Runbook** | [Cluster Bootstrap Runbook](https://runbook.ibtisam-iq.com/bootstrap/kubernetes/cluster-kubeadm/) | [EKS on KodeKloud Runbook](https://runbook.ibtisam-iq.com/iac/terraform/provisioning/eks-on-kodekloud-aws-playground/) |

---

## Step 1 — Per-Service Helm Chart Deployment

Each service ships with its own Helm chart under `src/<service>/chart/`, including a base `values.yaml`. I studied that base file for each service, then authored additional `values-*.yaml` overrides on top of it — one per deployment scenario — so that the same chart could be deployed across three different target environments without modifying the chart itself.

> The table below lists the key override files per service. Each service runbook (last row) documents the complete values file set, per-scenario `helm` commands, validation steps, and teardown.

| [Catalog](./src/catalog/chart/) | [Cart](./src/cart/chart/) | [Orders](./src/orders/chart/) | [Checkout](./src/checkout/chart/) | [UI](./src/ui/chart/) |
|---|---|---|---|---|
| [`values-mysql-ephemeral.yaml`](./src/catalog/chart/values-mysql-ephemeral.yaml) | [`values-dynamodb-local.yaml`](./src/cart/chart/values-dynamodb-local.yaml) | [`values-postgresql-ephemeral-msg-in-memory.yaml`](./src/orders/chart/values-02-postgresql-ephemeral-msg-in-memory.yaml) | [`values-redis-local.yaml`](./src/checkout/chart/values-redis-local.yaml) | [`values-clusterip.yaml`](./src/ui/chart/values-clusterip.yaml) |
| [`values-mysql-pvc-baremetal.yaml`](./src/catalog/chart/values-mysql-pvc-baremetal.yaml) | [`values-dynamodb-aws.yaml`](./src/cart/chart/values-dynamodb-aws.yaml) | [`values-postgresql-rabbitmq-pvc-baremetal.yaml`](./src/orders/chart/values-03-postgresql-rabbitmq-pvc-baremetal.yaml) | [`values-redis-tls.yaml`](./src/checkout/chart/values-redis-tls.yaml) | [`values-nodeport.yaml`](./src/ui/chart/values-nodeport.yaml) |
| [`values-mysql-pvc-eks.yaml`](./src/catalog/chart/values-mysql-pvc-eks.yaml) | | [`values-postgresql-rabbitmq-pvc-eks.yaml`](./src/orders/chart/values-04-postgresql-rabbitmq-pvc-eks.yaml) | [`values-redis-aws-elasticache.yaml`](./src/checkout/chart/values-redis-aws-elasticache.yaml) | [`values-loadbalancer.yaml`](./src/ui/chart/values-loadbalancer.yaml) |
| [`values-external-mysql.yaml`](./src/catalog/chart/values-external-mysql.yaml) | | [`values-postgresql-rabbitmq-external.yaml`](./src/orders/chart/values-05-postgresql-rabbitmq-external.yaml) | | [`values-alb-ingress.yaml`](./src/ui/chart/values-alb-ingress.yaml) |
| | | [`values-postgresql-pvc-eks-sqs.yaml`](./src/orders/chart/values-06-postgresql-pvc-eks-sqs.yaml) | | |
| 📖 [Catalog Runbook](./src/catalog/chart/catalog-chart-runbook.md) | 📖 [Cart Runbook](./src/cart/chart/cart-chart-runbook.md) | 📖 [Orders Runbook](./src/orders/chart/orders-chart-runbook.md) | 📖 [Checkout Runbook](./src/checkout/chart/checkout-chart-runbook.md) | 📖 [UI Runbook](./src/ui/chart/ui-chart-runbook.md) |

---

## Step 2 — Helmfile Unified Deployment

Deploying five services independently with `helm install` per service is workable for exploration, but operationally fragile — release ordering, dependency sequencing, and teardown all become manual. In this step I introduced [Helmfile](https://helmfile.readthedocs.io/) to declare all five releases in a single manifest, enforce their deployment order via `needs:`, and operate the full stack with one command.

Three Helmfile configurations were authored, one per deployment target. Each one references the `values-*.yaml` files marked ✱ in Step 1 above.

| File | Target | Storage | Message Broker | UI Exposure |
|---|---|---|---|---|
| [`helmfile-baremetal-ephemeral.yaml`](./helmfile/helmfile-baremetal-ephemeral.yaml) | Bare-metal Kubernetes | Ephemeral (no PVC) | In-memory | NodePort |
| [`helmfile-baremetal-persistent.yaml`](./helmfile/helmfile-baremetal-persistent.yaml) | Bare-metal Kubernetes | `local-path` PVC | RabbitMQ | NodePort |
| [`helmfile-eks.yaml`](./helmfile/helmfile-eks.yaml) | AWS EKS | `gp3` EBS PVC | AWS SQS | ALB Ingress |

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
