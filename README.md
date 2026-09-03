# Argocd
Argocd manifests to apply — cluster-level infrastructure for the `rrobinson-services` k3s
cluster, deployed by the app-of-apps in `jojobobby/Arcana-Argocd-Apps` (`prod/`).

| Directory | ArgoCD app | What |
|---|---|---|
| `argocd/` | `argocd` | ArgoCD self-management (argo-cd Helm chart + values) |
| `cnpg-operator/` | `cnpg-operator` | CloudNativePG operator |
| `default/` | `default` | Cluster certificates (ClusterIssuers) in the `default` namespace |
| `monitoring/` | `monitoring` | Prometheus + Alertmanager + Grafana + Loki + Alloy — see [monitoring/README.md](monitoring/README.md) |

Design notes live under `docs/superpowers/specs/`.
