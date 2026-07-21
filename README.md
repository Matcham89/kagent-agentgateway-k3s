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

## Service Port Map

All ports below are `ClusterIP` (cluster-internal only) unless noted otherwise
— reach them from the mini PC's LAN via `kubectl port-forward`, not directly.
The node's LAN IP is whatever `kubectl get nodes -o wide` reports for
`INTERNAL-IP` (find it with that command; it depends on your network).

### Commonly accessed (port-forward from the host)

| Service | Namespace | Port | Purpose | Port-forward |
|---|---|---|---|---|
| `flux-operator` | `flux-system` | 9080 | Flux Status web UI | `kubectl port-forward -n flux-system svc/flux-operator 9080:9080` |
| `kube-prometheus-stack-grafana` | `monitoring` | 80 | Grafana | `kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80` |
| `kube-prometheus-stack-prometheus` | `monitoring` | 9090 | Prometheus UI | `kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090` |
| `kube-prometheus-stack-alertmanager` | `monitoring` | 9093 | Alertmanager UI | `kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093` |
| `loki-gateway` | `monitoring` | 80 | Loki query API (for `logcli`/Grafana Explore outside the cluster) | `kubectl port-forward -n monitoring svc/loki-gateway 3100:80` |
| `tempo` | `monitoring` | 3200 | Tempo query API | `kubectl port-forward -n monitoring svc/tempo 3200:3200` |
| `kagent-ui` | `kagent` | 8080 | kagent web UI | `kubectl port-forward -n kagent svc/kagent-ui 8080:8080` |
| `kagent-controller` | `kagent` | 8083 | kagent controller API — this is the endpoint agentgateway (host) connects to | `kubectl port-forward -n kagent svc/kagent-controller 8083:8083` |

### Already reachable on the host (no port-forward needed)

Traefik ships with k3s by default and binds directly to the node's host ports
via k3s `ServiceLB` — `http://<node-ip>:80` and `:443` reach it today. No
`IngressRoute` is currently configured (this cluster intentionally has no
ingress — see [Architecture](docs/architecture.md#agentgateway-host)), so
nothing routes through it yet, but the ports are open on the host.

| Service | Namespace | Host port |
|---|---|---|
| `traefik` | `kube-system` | 80, 443 |

### Everything else (cluster-internal only, not meant for direct host access)

| Namespace | Service | Ports |
|---|---|---|
| `1password` | `onepassword-connect` | 8080, 8081 |
| `ate-system` | `api` (ate-api-server) | 443 |
| `ate-system` | `ate-controller` | 8080 |
| `ate-system` | `atenet-router` | 80, 443 |
| `ate-system` | `dns` | 53 |
| `ate-system` | `rustfs` | 9000, 9001 |
| `ate-system` | `valkey-cluster` / `valkey-cluster-service` | 6379 (client), 16379 (cluster bus) |
| `kagent` | `kagent-postgresql` | 5432 |
| `kagent` | `kagent-tools` | 8084 |
| `kagent` | `kagent-querydoc`, `kagent-grafana-mcp`, and each pre-built agent (`k8s-agent`, `helm-agent`, `istio-agent`, `cilium-*`, `kgateway-agent`, `observability-agent`, `promql-agent`, `argo-rollouts-conversion-agent`) | 8080 (MCP endpoint) |
| `monitoring` | `loki`, `loki-headless` | 3100, 9095 |
| `monitoring` | `loki-canary` | 3500 |
| `monitoring` | `prometheus-operated`, `alertmanager-operated` | 9090 / 9093, 9094 |

Get the live, authoritative list at any time with:

```bash
kubectl get svc -A
```

## Secret Management

Secrets are managed by the 1Password Operator. After the operator is running, create `OnePasswordItem` resources in the target namespace and the operator materialises them as Kubernetes Secrets automatically.

Bootstrap secrets (`op-credentials`, `onepassword-token`) are created manually — see [bootstrap/README.md](bootstrap/README.md).

## agentgateway

agentgateway runs on the host machine as a standalone process, not inside the cluster. Point it at the kagent MCP endpoint:

```bash
kubectl port-forward -n kagent svc/kagent <local-port>:<kagent-port>
```
