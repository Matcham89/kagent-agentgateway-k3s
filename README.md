# kagent-agentgateway-k3s

GitOps repository for a single-node k3s cluster running the kagent AI agent platform with full observability.

## Stack

| Component | Purpose |
|---|---|
| [Flux](https://fluxcd.io) (via [Flux Operator](https://github.com/controlplaneio-fluxcd/flux-operator)) | GitOps continuous delivery |
| [1Password Connect + Operator](https://github.com/1Password/connect-helm-charts) | Secret management |
| [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts) | Prometheus + Grafana + Alertmanager |
| [Loki](https://grafana.com/oss/loki/) | Log aggregation |
| [Tempo](https://grafana.com/oss/tempo/) | Distributed tracing |
| [Promtail](https://grafana.com/docs/loki/latest/send-data/promtail/) | Log shipping |
| [Substrate](https://github.com/kagent-dev/substrate) | gVisor-based sandbox runtime for agent execution |
| [kagent](https://github.com/kagent-dev/kagent) | AI agent platform |
| agentgateway | Runs standalone on the host (not in-cluster) |

The Flux Operator itself is also GitOps-managed post-bootstrap — see
[Flux managing Flux](#flux-managing-flux) below.

## Repository Layout

```
.
├── bootstrap/          # One-time manual steps to bring up Flux
├── docs/               # Architecture and operational documentation
└── flux/
    ├── clusters/k3s/   # Flux entrypoint — FluxInstance + root Kustomization
    ├── apps/
    │   ├── base/       # Helm chart definitions (HelmRelease, OCIRepository, etc.)
    │   └── dev/        # Flux Kustomizations that reference base paths
    └── infra/          # Reusable namespace + Pod Security Standards components
```

See [docs/architecture.md](docs/architecture.md) for a full breakdown of the GitOps model, dependency chain, and design decisions.

## Bootstrap

See [bootstrap/README.md](bootstrap/README.md) for the step-by-step instructions to provision a new cluster from scratch.

**Quick summary:**

```bash
# 1. Create 1Password credentials secret
kubectl create secret generic op-credentials \
  --namespace 1password --create-namespace \
  --from-file=1password-credentials.json=./1password-credentials.json

# 2. Install the Flux Operator (one-time bootstrap version — Flux takes over
#    upgrading its own operator via GitOps once Step 3 reconciles)
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --version=0.55.0 --namespace flux-system --create-namespace

# 3. Apply the FluxInstance — GitOps takes over from here
kubectl apply -f flux/clusters/k3s/flux-instance.yaml
```

## Flux managing Flux

After bootstrap, the `flux-operator` HelmRelease defined in
`flux/apps/base/flux-system/flux-operator/` takes over the same Helm release
that Step 2 installed manually (release name `flux-operator`, namespace
`flux-system`). It tracks the latest chart version (`semver: "*"`) and
upgrades itself via GitOps from then on, so the manually-installed bootstrap
version only needs to be current enough to get through the first reconcile.

The Flux Status web UI is enabled by default on port 9080:

```bash
kubectl port-forward -n flux-system svc/flux-operator 9080:9080
# Open http://localhost:9080
```

## Day-to-day Operations

```bash
# Check reconciliation status
flux get kustomizations -A
flux get helmreleases -A

# Force a resync
flux reconcile kustomization cluster-apps --with-source

# Tail Flux errors
flux logs --follow --level=error

# Access Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

## Secret Management

Secrets are managed by the 1Password Operator. After the operator is running, create `OnePasswordItem` resources in the target namespace and the operator materialises them as Kubernetes Secrets automatically.

Bootstrap secrets (`op-credentials`, `onepassword-token`) are created manually — see [bootstrap/README.md](bootstrap/README.md).

## agentgateway

agentgateway runs on the host machine as a standalone process, not inside the cluster. Point it at the kagent MCP endpoint:

```bash
kubectl port-forward -n kagent svc/kagent <local-port>:<kagent-port>
```
