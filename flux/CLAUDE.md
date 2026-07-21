# CLAUDE.md

This file provides guidance when working with code in this repository.

## What This Is

A Flux CD v2 GitOps repository for a single-node k3s cluster running:
- **Flux** (via Flux Operator)
- **1Password Operator** (Connect + Operator)
- **Grafana LGTM stack** (Prometheus, Grafana, Loki, Tempo, Promtail)
- **kagent** (AI agent platform)

agentgateway runs **outside** the cluster as a standalone process.

Git is the source of truth. No build system — all changes are applied by pushing to `main`.

---

## Common Operational Commands

```bash
# Check Flux reconciliation status
flux get kustomizations -A
flux get helmreleases -A

# Force immediate reconciliation
flux reconcile kustomization cluster-apps --with-source
flux reconcile helmrelease <name> -n <namespace>

# Validate manifests before pushing
kubectl kustomize flux/apps/base/<app>     # Preview rendered output
kubectl kustomize flux/apps/dev/<namespace> # Preview with components applied

# Check why something isn't reconciling
flux logs --follow --level=error
kubectl describe kustomization <name> -n flux-system
```

---

## Architecture: Three-Layer GitOps

```
clusters/k3s/           <- Flux entrypoint (FluxInstance + root Kustomization)
apps/
  base/                 <- Cluster-agnostic Kubernetes manifests (HelmReleases, CRDs)
  dev/                  <- Flux Kustomization resources (ks.yaml) referencing base paths
infra/                  <- Reusable Kustomize Components for namespace + PSS config
```

**How it works:**
1. `clusters/k3s/flux-instance.yaml` — FluxInstance installs Flux and syncs this repo
2. `clusters/k3s/cluster-apps.yaml` — root Flux Kustomization, sources `./flux/apps/dev`
3. `apps/dev/kustomization.yaml` — aggregates all namespace kustomizations
4. `apps/dev/**/ks.yaml` — each is a Flux `Kustomization` pointing to a path in `apps/base/`
5. `apps/base/**/` — contains the actual Kubernetes manifests (HelmReleases, etc.)

**File naming conventions:**
- `ks.yaml` — Flux Kustomization resource (lives in `apps/dev/`)
- `helmrelease.yaml` — HelmRelease resource (lives in `apps/base/`)
- `kustomization.yaml` — Kustomize resource aggregation (both layers)

---

## Application Directory Pattern

```
apps/base/<namespace>/<app>/
├── helmrelease.yaml    # HelmRepository + HelmRelease (or OCIRepository + HelmRelease)
└── kustomization.yaml

apps/dev/<namespace>/
├── kustomization.yaml  # references infra/ component for namespace + PSS config
└── <app>/ks.yaml       # Flux Kustomization -> apps/base/<namespace>/<app>/
```

**ks.yaml template:**
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: &app <name>
spec:
  commonMetadata:
    labels:
      app.kubernetes.io/name: *app
  interval: 1h
  path: ./flux/apps/base/<path>
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  targetNamespace: <namespace>
  timeout: 5m
  prune: true
  wait: true
  dependsOn:
    - name: <dependency>
```

---

## Namespace Infra Components

Two reusable Kustomize Components in `infra/` — include via `components:` in a namespace's `kustomization.yaml`:

| Component | PSS | Use For |
|---|---|---|
| `namespace` | restricted | Regular app namespaces (1password, kagent) |
| `namespace-privileged` | privileged | System namespaces (monitoring — node-exporter, promtail need host access) |

---

## Deployed Applications

| Namespace | App | Notes |
|---|---|---|
| `1password` | 1Password Connect + Operator | Secret source for ExternalSecrets (if added later) |
| `ate-system` | substrate-crds + substrate-operator | gVisor sandbox runtime for kagent worker pool |
| `monitoring` | kube-prometheus-stack, Loki, Tempo, Promtail | Full LGTM stack, local-path storage |
| `kagent` | kagent CRDs + kagent operator | AI agent platform, OTLP traces -> Tempo |

## Storage

k3s ships with `local-path` provisioner as the default StorageClass.
All PVCs in this cluster use `storageClassName: local-path`.
Data lives under `/var/lib/rancher/k3s/storage/` on the host.

---

## Key Dependencies (install order via `dependsOn`)

```
substrate-crds   ->  substrate-operator (ate-system)
kagent-crds      ->  kagent-operator
substrate-operator (ate-system)  ->  kagent-operator
kube-prometheus-stack  ->  grafana (Loki + Tempo + Promtail)
kube-prometheus-stack  ->  kagent-operator (for ServiceMonitor CRDs)
```

## Substrate Notes

Substrate runs in the `ate-system` namespace and provides the gVisor-based
worker pool that kagent uses to sandbox tool execution.

**Critical one-time configuration**: before deploying, set the JWT issuer in
`flux/apps/base/substrate/substrate-operator/helmrelease.yaml` to match your
k3s node's actual API server address:

```bash
kubectl get --raw /.well-known/openid-configuration | jq .issuer
```

On amd64 nodes, pin `runscAMD64URL` in the kagent helmrelease to a known-good
gVisor nightly if the chart default fails checkpoint/restore. See the commented
values in `flux/apps/base/kagent/operator/helmrelease.yaml`.

---

## Secret Management

Secrets are managed by the **1Password Connect Operator**.
Place `OnePasswordItem` CRs in the target namespace and the operator will
materialise them as Kubernetes Secrets.

For bootstrap secrets (op-credentials, onepassword-token) — created manually before Flux runs.
See `bootstrap/README.md`.

---

## Adding a New Application

1. Create `apps/base/<namespace>/<app>/helmrelease.yaml` + `kustomization.yaml`
2. Create `apps/dev/<namespace>/<app>/ks.yaml`
3. If new namespace: create `apps/dev/<namespace>/kustomization.yaml` with infra component
4. Add the new namespace dir to `apps/dev/kustomization.yaml` resources list
5. Set `dependsOn` as needed in the ks.yaml
