# Argocd
Argocd manifests to apply — cluster-level infrastructure for the `rrobinson-services` k3s
cluster, deployed by the app-of-apps in `jojobobby/Arcana-Argocd-Apps` (`prod/`).

| Directory | ArgoCD app | What |
|---|---|---|
| `argocd/` | `argocd` | ArgoCD self-management (argo-cd Helm chart + values) |
| `cnpg-operator/` | `cnpg-operator` | CloudNativePG operator |
| `default/` | `default` | Cluster certificates (ClusterIssuers) in the `default` namespace |
| `haproxy-ingress/` | `haproxy-ingress` | HAProxy ingress controller (the cluster's only IngressClass) — see [haproxy-ingress/README.md](haproxy-ingress/README.md) |
| `monitoring/` | `monitoring` | Prometheus + Alertmanager + Grafana + Loki + Alloy — see [monitoring/README.md](monitoring/README.md) |
| `umami/` | `umami` | Umami web analytics for the Cosmic shop (analytics.cosmicv2.com) + its CloudNativePG database — see [umami/README.md](umami/README.md) |

Design notes live under `docs/superpowers/specs/`.
