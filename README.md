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

## Related chart repos (tidangames.com data services)

Three cluster-wide services live in their own private repos (each repo is just a Helm
chart at its root) and are registered next to the apps above in
`Arcana-Argocd-Apps/prod/`. Design: `docs/superpowers/specs/2026-09-17-tidangames-data-services-design.md`.

| Repo | ArgoCD app / namespace | What |
|---|---|---|
| `jojobobby/tidan-postgres` (`Both/Postgres`) | `postgres` | CloudNativePG cluster + pgAdmin 4 at pdb.tidangames.com (needs `cnpg-operator/`) |
| `jojobobby/tidan-redis` (`Both/Redis`) | `redis` | Redis 8 + RedisInsight at rdb.tidangames.com (haproxy basic auth) |
| `jojobobby/tidan-secrets-manager` (`Both/SecretsManager`) | `secrets-manager` | OpenBao (vendored official chart, Raft, static-key auto-unseal) at sm.tidangames.com |

Design notes live under `docs/superpowers/specs/`.
