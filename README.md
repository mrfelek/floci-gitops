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
| `storage/csi-driver-nfs-values.yaml` | Values for the NFS CSI driver and the `nfs` StorageClass, deployed by `apps/csi-driver-nfs.yaml`. |
| `storage/aws-ebs-csi-driver-values.yaml` | Values for the EBS CSI driver and the `gp3` StorageClass, deployed by `apps/aws-ebs-csi-driver.yaml`. |

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
  kube-state-metrics, with no Alertmanager, Pushgateway or ingress. Its data lives on a 4 GiB `nfs` volume, kept 7
  days or up to 3 GB. Prometheus doesn't support NFS for its database, which these metrics can afford.
- **csi-driver-nfs 4.13.6** and the `nfs` StorageClass: volumes on the NAS, NFS 4.1, in
  `nas.home.lan:/mnt/nas/Backup/k8s-storage/dev-eu-central-1` (a dataset per cluster), one directory per volume,
  deleted with its claim. The share admits the cluster's VM alone and maps root to root, so the driver can make each
  volume's directory and a pod's `fsGroup` owns its files. It isn't the default class: a claim names
  `storageClassName: nfs`.
- **aws-ebs-csi-driver 1.68.0** (chart 2.68.0) and the `gp3` StorageClass: EBS volumes, which Floci backs with a
  file and a loop device on the cluster's host and attaches into the node, where the driver formats and mounts
  them. The controller's AWS identity comes from IRSA: the cluster's Terraform registers its OIDC issuer and gives
  the role `<cluster>-ebs-csi-driver`, with `AmazonEBSCSIDriverPolicy`, to `kube-system/ebs-csi-controller-sa`.
  The role's ARN and Floci's endpoint come from a ConfigMap kept out of this repository, created once per cluster:
  `kubectl -n kube-system create configmap ebs-csi-floci --from-literal=AWS_ROLE_ARN=<role ARN>
  --from-literal=AWS_ENDPOINT_URL=<Floci endpoint>`.
