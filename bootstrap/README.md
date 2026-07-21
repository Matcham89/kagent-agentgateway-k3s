# Bootstrap — k3s Single-Node Cluster

This cluster uses the **ControlPlane Flux Operator** (not `flux bootstrap` CLI).
Bootstrap is a two-step manual process. After that, GitOps takes over everything.

---

## Prerequisites

- k3s installed and running (`curl -sfL https://get.k3s.io | sh -`)
- **16GB+ RAM on the node.** 8GB is not enough once kube-prometheus-stack, Loki,
  Tempo, the 6-node valkey (Redis) cluster, and the full set of kagent
  pre-built agents are all running — pods will get OOM-killed or crash-loop
  under memory pressure. Check current usage with `kubectl top node`.
- `kubectl` configured against the cluster. `/etc/rancher/k3s/k3s.yaml` is
  root-only by default; either run as root, or copy it out for your own user:
  ```bash
  mkdir -p ~/.kube
  sudo cat /etc/rancher/k3s/k3s.yaml > ~/.kube/config
  chmod 600 ~/.kube/config
  export KUBECONFIG=~/.kube/config
  ```
- 1Password Connect credentials JSON downloaded from 1Password
- A 1Password item holding your model provider API key (see Step 5)

---

## Step 1 — Create the 1Password credentials secret

This secret must exist before Flux reconciles so the 1Password Connect pod can start.

```bash
# Replace with the path to your downloaded credentials file
kubectl create secret generic op-credentials \
  --namespace 1password \
  --from-file=1password-credentials.json=./1password-credentials.json \
  --create-namespace
```

> The 1password-credentials.json file is obtained from 1Password under
> "Integrations > 1Password Connect Server > Add a Connect Server".

---

## Step 2 — Install the Flux Operator

This is a **one-time bootstrap install only**. Once Flux reconciles, the
`flux-operator` HelmRelease in `flux/apps/base/flux-system/flux-operator/`
takes over management of this same Helm release (name `flux-operator`,
namespace `flux-system`) and keeps it updated to the latest chart version via
GitOps — Flux manages Flux. There is no need to keep the version below current
with the latest chart version; it will self-upgrade shortly after Step 3.

```bash
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --version=0.55.0 \
  --namespace flux-system \
  --create-namespace
```

> **Why the version matters at bootstrap time**: the FluxInstance (Step 3)
> asks for `distribution.version: 2.x`, which always resolves to the latest
> matching Flux release manifests. An old `flux-operator` build applying its
> internal CRD-migration patches against those newer manifests can fail with
> an error like:
> `build failed: add operation does not apply: doc is missing path: ...`
> Keep the bootstrap version reasonably current to avoid this on first apply
> — after that, the self-management HelmRelease keeps it current automatically.

---

## Step 3 — Apply the FluxInstance

```bash
kubectl apply -f flux/clusters/k3s/flux-instance.yaml
```

Flux Operator will now:
1. Install all Flux controllers into `flux-system`
2. Create a `GitRepository` pointing to this repo
3. Reconcile `cluster-apps` → which deploys all namespaces in order

Watch progress:

```bash
kubectl get fluxinstance -n flux-system -w
kubectl get kustomizations -A -w
```

---

## Step 4 — Create the 1Password Connect token secret

After the 1Password operator is running, create the token secret it uses to
authenticate with the Connect server:

```bash
kubectl create secret generic onepassword-token \
  --namespace 1password \
  --from-literal=token=<YOUR_CONNECT_TOKEN>
```

> The Connect token is generated alongside the credentials JSON in the
> "1Password Connect Server" setup flow.

---

## Step 5 — Configure the kagent model provider secret

kagent's pre-built agents (`k8s-agent`, `helm-agent`, `istio-agent`,
`cilium-*`, `kgateway-agent`, `observability-agent`, `promql-agent`,
`argo-rollouts-conversion-agent`) all read an `OPENAI_API_KEY` environment
variable from a Secret named `kagent-openai` in the `kagent` namespace.
Without it, those pods sit in `CreateContainerConfigError` and the `kagent`
HelmRelease never goes `Ready` (it waits on all its Deployments and retries
every 10 minutes).

This is managed via `flux/apps/base/kagent/model-secret/onepassworditem.yaml`
— a `OnePasswordItem` CR (`itemPath: vaults/<vault-id>/items/<item-name>`)
that the 1Password Operator materialises into the `kagent-openai` Secret.

1. In 1Password, create (or reuse) an item with a field **labeled exactly
   `OPENAI_API_KEY`** holding your key. The 1Password Operator names Secret
   keys after the item's field labels — a field labeled `password` or
   `credential` will *not* produce the right key, even if its value looks
   correct.
2. Update `spec.itemPath` in `onepassworditem.yaml` to point at that vault and
   item. Vault names with spaces can be unreliable to resolve by name — if so,
   look up the vault ID instead:
   ```bash
   TOKEN=$(kubectl get secret onepassword-token -n 1password -o jsonpath='{.data.token}' | base64 -d)
   kubectl port-forward -n 1password svc/onepassword-connect 8080:8080 &
   curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8080/v1/vaults
   ```
3. Commit and push — Flux reconciles the `OnePasswordItem`, and the Secret
   appears automatically once the operator processes it.

---

## Verify

```bash
# All Flux resources healthy
flux get kustomizations -A
flux get helmreleases -A

# flux-operator has taken over its own HelmRelease (self-management)
helm history flux-operator -n flux-system

# 1Password
kubectl get pods -n 1password

# Monitoring
kubectl get pods -n monitoring

# kagent
kubectl get pods -n kagent
```

---

## Accessing the Flux Web UI

The Flux Status web UI is enabled by default on port 9080. Port-forward to access it:

```bash
kubectl port-forward -n flux-system svc/flux-operator 9080:9080
# Open http://localhost:9080
```

---

## Accessing Grafana

Grafana is deployed as a `ClusterIP` service. Port-forward to access it:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Open http://localhost:3000
# Default credentials: admin / (set via values or ExternalSecret)
```

---

## Disaster Recovery

### Full rebuild

Follow Steps 1–3. Wait 5–10 minutes for all HelmReleases to reconcile.

### FluxInstance deleted

```bash
kubectl apply -f flux/clusters/k3s/flux-instance.yaml
```

### Force reconcile

```bash
flux reconcile kustomization cluster-apps --with-source
flux reconcile helmrelease <name> -n <namespace>
```

### valkey (Redis) cluster stuck `CLUSTERDOWN` after a node reboot

On a single-node k3s cluster, a host reboot (e.g. resizing RAM) restarts every
pod at once, and each gets a **new pod IP**. The 6-pod valkey `StatefulSet`
persists its cluster node table (`nodes.conf`) on its PVCs, so after reboot
each node still has the *old* IPs for its peers and the gossip protocol can't
reconverge on its own — symptoms are `ate-api-server` crash-looping with
`CLUSTERDOWN The cluster is down`, which cascades into `kagent-controller`
also crash-looping (`dial ate-api: context deadline exceeded`).

Check for it:

```bash
kubectl exec -n ate-system valkey-cluster-0 -- valkey-cli cluster info | head -5
# cluster_state:fail means this is happening
```

Fix by re-introducing each node to the others at their current IPs (no data
loss, no PVC changes needed):

```bash
for i in 1 2 3 4 5; do
  ip=$(kubectl get pod valkey-cluster-$i -n ate-system -o jsonpath='{.status.podIP}')
  kubectl exec -n ate-system valkey-cluster-0 -- valkey-cli cluster meet "$ip" 6379
done
# wait a few seconds, then confirm:
kubectl exec -n ate-system valkey-cluster-0 -- valkey-cli cluster info | head -3
# cluster_state:ok
```

Then restart the pods that were crash-looping on the stale connection so they
pick up the now-healthy cluster:

```bash
kubectl delete pod -n ate-system -l app=ate-api-server
kubectl delete pod -n kagent -l app.kubernetes.io/name=kagent,app.kubernetes.io/component=controller
```

---

## Notes on agentgateway

agentgateway is intentionally **not** deployed inside this cluster.
Run it as a standalone process on the host machine, pointing it at the
kagent controller's MCP endpoint:

```
kagent controller service: kagent.kagent.svc.cluster.local:<port>
```

Expose the kagent service with `kubectl port-forward` or via a k3s
`LoadBalancer` service if you need stable access from the host.
