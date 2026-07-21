# Architecture

## Overview

This cluster runs on a single machine using k3s. All workloads are managed by Flux CD using a GitOps model — the Git repository is the source of truth, and Flux continuously reconciles the cluster state to match what is committed.

agentgateway runs outside the cluster as a standalone process on the host and communicates with kagent via port-forward or a host-accessible Service.

```
┌─────────────────────────────────────────────────────────────────┐
│  Host Machine                                                   │
│                                                                 │
│   agentgateway (standalone)                                     │
│        │                                                        │
│        │ MCP / HTTP                                             │
│        ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  k3s cluster                                            │   │
│  │                                                         │   │
│  │  ┌──────────┐   ┌───────────┐   ┌───────────────────┐  │   │
│  │  │ 1Password│   │ Substrate │   │     Monitoring    │  │   │
│  │  │ Connect  │   │ ate-system│   │  (monitoring ns)  │  │   │
│  │  │ Operator │   │           │   │                   │  │   │
│  │  │1password │   │  gVisor   │   │  Prometheus       │  │   │
│  │  │    ns    │   │  sandbox  │   │  Grafana          │  │   │
│  │  └──────────┘   │  runtime  │   │  Loki             │  │   │
│  │       │         └─────┬─────┘   │  Tempo            │  │   │
│  │  secrets         worker│pool    │  Promtail         │  │   │
│  │  injected        ▼     │        └───────────────────┘  │   │
│  │       │    ┌──────────────────┐         ▲              │   │
│  │       └───►│    kagent        │  OTLP traces/logs      │   │
│  │            │   (kagent ns)    │─────────┘              │   │
│  │            │                  │                         │   │
│  │            │  k8s-agent       │                         │   │
│  │            │  helm-agent      │                         │   │
│  │            │  promql-agent    │                         │   │
│  │            │  observability   │                         │   │
│  │            │  kgateway-agent  │                         │   │
│  │            └──────────────────┘                         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## GitOps Model

The repository uses a three-layer architecture inherited from the Flux CD community pattern.

```
flux/clusters/k3s/              Layer 1 — Flux entrypoint
flux/apps/dev/<namespace>/      Layer 2 — Flux Kustomization resources
flux/apps/base/<namespace>/     Layer 3 — Kubernetes manifests
```

### Layer 1 — Cluster entrypoint

`flux/clusters/k3s/` is the path Flux syncs first. It contains two resources:

| File | Purpose |
|---|---|
| `flux-instance.yaml` | `FluxInstance` CR — tells the Flux Operator which Flux components to install and where to sync from |
| `cluster-apps.yaml` | Root `Kustomization` — sources `flux/apps/dev` and kicks off everything else |

### Layer 2 — Flux Kustomizations (`apps/dev/`)

Each namespace has a directory under `flux/apps/dev/`. The namespace directory contains:

- `kustomization.yaml` — a Kustomize aggregation that applies the namespace's PSS label via an `infra/` component and lists all `ks.yaml` files as resources
- `<app>/ks.yaml` — a Flux `Kustomization` CR that points to the corresponding path in `apps/base/` and declares `dependsOn` relationships

### Layer 3 — Manifests (`apps/base/`)

Contains the actual Kubernetes manifests: `HelmRelease`, `OCIRepository`, `HelmRepository`, `ConfigMap`, etc. These are cluster-agnostic and have no Flux-specific resources.

### File naming conventions

| Filename | Layer | Purpose |
|---|---|---|
| `ks.yaml` | `apps/dev/` | Flux `Kustomization` resource |
| `helmrelease.yaml` | `apps/base/` | `HelmRelease` (+ source `OCIRepository` or `HelmRepository`) |
| `kustomization.yaml` | both | Kustomize aggregation |

---

## Namespaces

| Namespace | PSS | Contents |
|---|---|---|
| `flux-system` | — | Flux controllers (managed by Flux Operator) |
| `1password` | restricted | 1Password Connect server + Kubernetes Operator |
| `ate-system` | privileged | Substrate CRDs + Operator (gVisor sandbox runtime) |
| `monitoring` | privileged | Prometheus, Grafana, Loki, Tempo, Promtail |
| `kagent` | privileged | kagent CRDs + kagent controller + worker pool |

Pod Security Standards (PSS) are applied via reusable Kustomize Components in `flux/infra/`:

- `infra/namespace` — restricted PSS (default for regular apps)
- `infra/namespace-privileged` — privileged PSS (required for node-level access: node-exporter, promtail, gVisor workers)

---

## Dependency Chain

Flux `dependsOn` fields enforce this install order:

```
substrate-crds
    └── substrate-operator (ate-system)
            └── kagent-operator
                    ▲
kagent-crds ────────┘

kube-prometheus-stack
    ├── grafana (Loki + Tempo + Promtail)
    └── kagent-operator
```

In plain English:
1. Substrate CRDs are installed first — the operator needs them to start
2. Substrate operator starts — kagent needs it for the worker pool
3. kagent CRDs are installed — the kagent operator needs them
4. kube-prometheus-stack is installed — ServiceMonitor CRDs must exist before kagent tries to create any
5. kagent operator starts — depends on both CRD sets and substrate
6. Grafana stack (Loki, Tempo, Promtail) starts after Prometheus is ready

---

## Observability

All components emit to a unified LGTM stack in the `monitoring` namespace.

```
Pods/nodes
    │ logs (stdout)
    ▼
Promtail (DaemonSet) ──► Loki ──► Grafana
                                       ▲
kagent / substrate                     │
    │ OTLP traces + logs               │
    ▼                                  │
Tempo ──► span metrics ──► Prometheus ─┘
              (remote write)
```

- **Prometheus** scrapes `ServiceMonitor` and `PodMonitor` resources cluster-wide (`selectorNilUsesHelmValues: false`)
- **Tempo** receives OTLP on ports 4317 (gRPC) and 4318 (HTTP); kagent is pre-configured to send traces to `tempo.monitoring.svc.cluster.local:4317`
- **Tempo metrics generator** remote-writes span metrics back to Prometheus, enabling service graph and RED metrics in Grafana without a separate OpenTelemetry collector
- **Grafana** is pre-configured with Prometheus, Loki, and Tempo datasources; dashboards are loaded from ConfigMaps labelled `grafana_dashboard: "1"`

### Storage

All persistent data uses the k3s built-in `local-path` StorageClass (`/var/lib/rancher/k3s/storage/` on the host).

| Component | PVC size |
|---|---|
| Prometheus | 20Gi |
| Loki | 10Gi |
| Tempo | 10Gi |

---

## Secret Management

```
1Password (cloud)
    │
    │ credentials JSON (pre-created secret: op-credentials)
    ▼
1Password Connect (in-cluster, 1password ns)
    │
    │ Connect API token (pre-created secret: onepassword-token)
    ▼
1Password Operator
    │
    │ watches OnePasswordItem CRs in all namespaces
    ▼
Kubernetes Secret (materialised in target namespace)
    │
    ▼
Pod (mounts secret as env var or volume)
```

Two secrets are created manually before Flux runs (see `bootstrap/README.md`). Everything else is managed declaratively via `OnePasswordItem` CRs committed to Git.

---

## Substrate and kagent Worker Pool

Substrate provides the gVisor-based sandbox runtime that kagent uses to execute agent tools safely.

```
kagent controller
    │  creates worker pods
    ▼
SubstrateWorkerPool (6 replicas)
    │  each pod runs ateom-gvisor
    ▼
gVisor (runsc) — sandboxed execution environment
```

The `ate-system` namespace is privileged because gVisor workers require elevated capabilities that the restricted and baseline PSS profiles reject.

**k3s-specific configuration required before first deploy:**

The Substrate operator validates Kubernetes ServiceAccount tokens using the cluster's OIDC issuer. On k3s this is the API server's advertised address — not the in-cluster default. Check it with:

```bash
kubectl get --raw /.well-known/openid-configuration | jq .issuer
```

Set the result in `flux/apps/base/substrate/substrate-operator/helmrelease.yaml` under `values.auth.jwt.issuer` before pushing.

---

## agentgateway (Host)

agentgateway is intentionally not deployed inside k3s. It runs as a process on the host and connects to kagent's MCP endpoint.

```
agentgateway (host process)
    │
    │ HTTP / MCP
    ▼
kagent controller Service (ClusterIP)
    exposed via: kubectl port-forward -n kagent svc/kagent <port>
```

This keeps the cluster simple — no ingress controller, no LoadBalancer, no TLS termination needed. agentgateway handles client-facing concerns on the host side.
