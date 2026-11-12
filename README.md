# floci-gitops

GitOps configuration for a Kubernetes cluster running on [Floci](https://github.com/floci-io/floci)'s emulated
Amazon EKS: `dev-eu-central-1`, a single k3s node on Kubernetes 1.33. Argo CD deploys every Application in `apps/`.

## Layout

| Path | What it holds |
|---|---|
| `bootstrap/root.yaml` | The root Application (app of apps): deploys every Application in `apps/`. |
| `apps/` | One Argo CD Application per component. |
| `argocd/argocd-values.yaml` | Values for Argo CD itself, installed once by Helm. |
| `monitoring/prometheus-values.yaml` | Values for Prometheus, deployed by `apps/prometheus.yaml`. |

## Bootstrap

```sh
helm upgrade --install argocd oci://ghcr.io/argoproj/argo-helm/argo-cd --version 10.11.7 \
  -n argocd --create-namespace -f argocd/argocd-values.yaml
kubectl apply -f bootstrap/root.yaml
```

From then on, a change merged to `main` is deployed by Argo CD; a new component is a new Application in `apps/`.

## Components

- **Argo CD 3.5.7**, the newest line tested on Kubernetes 1.33: one instance of each component, no Dex, no
  notifications, no ingress. The UI is reached by port-forward:
  `kubectl -n argocd port-forward svc/argocd-server 8080:443`, then https://localhost:8080 as `admin`, with the
  password in the `argocd-initial-admin-secret` secret.
- **Prometheus** (prometheus-community chart 29.47.2), only for Freelens's charts: the server, node-exporter and
  kube-state-metrics, with no Alertmanager, Pushgateway or ingress. The cluster has no storage class, so the data
  lives in an emptyDir, kept 7 days or up to 1 GB.
