# Helmfile Scenarios

Three ready-to-use Helmfile definitions that deploy the full retail store stack in one command.
Each file encodes a specific infrastructure scenario. Pick the one that matches your cluster.

## Prerequisites

```bash
# Install helmfile (if not already installed)
brew install helmfile        # macOS

# or

helmfile_version=1.5.2
linux_arch=amd64
curl -L https://github.com/helmfile/helmfile/releases/download/v${helmfile_version}/helmfile_${helmfile_version}_linux_${linux_arch}.tar.gz -o /tmp/helmfile.tar.gz
tar -xzf /tmp/helmfile.tar.gz -C /tmp
sudo mv /tmp/helmfile /usr/local/bin/
sudo chmod +x /usr/local/bin/helmfile
rm /tmp/helmfile.tar.gz

# Verify
helmfile --version

# Install helm-diff plugin
helm plugin install https://github.com/databus23/helm-diff
```

## Scenarios

| File | Cluster | Storage | UI Access | Use Case |
|------|---------|---------|-----------|----------|
| `helmfile-baremetal-persistent.yaml` | k3s / kubeadm | `local-path` PVC | NodePort `:30080` | Durable local testing |
| `helmfile-baremetal-ephemeral.yaml` | k3s / kubeadm | None (in-memory) | NodePort `:30080` | Fast spin-up, CI smoke tests |
| `helmfile-eks.yaml` | AWS EKS | `gp3` EBS PVC | ClusterIP + port-forward | EKS validation |

## Commands

```bash
# Bare-metal — persistent (what was run and verified on 2026-05-31)
helmfile -f helmfile/helmfile-baremetal-persistent.yaml apply

# Bare-metal — ephemeral (fastest, no PVC needed)
helmfile -f helmfile/helmfile-baremetal-ephemeral.yaml apply

# EKS — gp3 StorageClass
helmfile -f helmfile/helmfile-eks.yaml apply
```

## Tear Down

```bash
# Destroy all releases (reverse order handled automatically)
helmfile -f helmfile/helmfile-baremetal-persistent.yaml destroy

# Or uninstall namespaces manually
kubectl delete ns catalog cart orders checkout ui
```

## Design Notes

### Why a top-level `helmfile/` folder (not `src/app/`)

`src/app/helmfile.yaml` is the upstream project's file. It deploys everything into `default` namespace
and uses `gotmpl` templates tied to their CI/CD tooling. This folder is independent: it uses
per-app namespaces, the `values-*.yaml` overlays added in each chart, and the same commands
that were manually verified.

### Why `values-endpoints.yaml` appears in every UI release

The UI is a front-door aggregator — it makes HTTP calls to catalog, cart, orders, and checkout
at startup. Without the four `app.endpoints.*` values set, the UI pod starts but every page
returns empty or errors. The file `src/ui/chart/values-endpoints.yaml` holds these four
in-cluster DNS URLs and is always passed as the second `-f` overlay for the UI release.

### Release ordering

```
catalog ─┬─────────────────────────────────────────► ui
cart     ┘                                           ↑
orders ──► checkout ─────────────────────────────────┘
```

`needs:` is used instead of `helmfile.yaml` top-level `helmDefaults.wait` so that:
- `catalog` and `orders` start in parallel (they don't depend on each other)
- `cart` waits only for `catalog` (needs DNS resolvable before UI can reach it)
- `checkout` waits for `orders` (needs the orders endpoint)
- `ui` waits for all four services

### The `set:` block on checkout

The orders endpoint URL (`http://orders.orders.svc.cluster.local`) is passed via `--set` rather
than a dedicated values file because it is a single scalar value that is identical across all
scenarios. Adding a `values-orders-endpoint.yaml` for one key would be over-engineering.
