# Monitoring stack (Prometheus + Loki + Grafana) for the rrobinson-services k3s cluster

Date: 2026-09-03. Status: approved for implementation (built the same day).

## Goal

Give the shared k3s cluster (`machine-01`, 147.135.8.100) a GitOps-managed observability
stack so the Cosmic-LTS game servers (`jojobobby/Cosmic-LTS`, namespaces
`cosmic-development` / `cosmic-production`) have metrics, logs, dashboards and alerts,
deployed and reconciled by ArgoCD exactly like every other service in the cluster.

## Context that shaped the design

- Single node k3s v1.35.4, 8 CPU / 64 GiB / 425 GB. CPU *requests* already at 67 %,
  memory requests at 17 %. Only `local-path` storage (no volume expansion).
- ArgoCD v2.14 app-of-apps in `jojobobby/Arcana-Argocd-Apps` (`prod/` recursed). Cluster
  infrastructure charts (ArgoCD itself, cnpg-operator) live in `jojobobby/Argocd`, one
  directory per component. Service repos (Harbor, Jenkins) vendor upstream charts under
  `k8s/charts/` so ArgoCD renders offline.
- Prometheus-operator CRDs (v0.87.0) are already in the cluster, orphaned from an earlier
  attempt (May 2026) - no operator, Prometheus, Grafana or Loki pods exist.
- Both Cosmic gameservers run with `hostNetwork: true` and hard-code their Prometheus
  listener to port 9090: the dev one holds it, the prod one logs
  "Address already in use". That port is reachable from the public internet (no host
  firewall), which also rules out running node-exporter on the host network.
- The Cosmic chart already publishes metrics Services (`cosmic-metrics-<env>` :9091 for
  the appserver, `redis-metrics-<env>` :9121 for the redis exporter) but nothing scrapes
  them.
- DNS for rrobinson.me is on Cloudflare (DNS-only records pointing at 147.135.8.100).
  Certificates come from cert-manager `letsencrypt-prod` via HTTP-01 through the haproxy
  ingress.

## Decision summary

| Topic | Decision |
|---|---|
| Home of the platform stack | `jojobobby/Argocd` -> `monitoring/` umbrella chart (`monitoring-stack`). |
| Chart packaging | Vendor `kube-prometheus-stack-88.6.4`, `loki-7.3.0`, `alloy-1.12.1` as tgz under `charts/` AND declare them in `dependencies:` with an empty `repository` (offline render like Harbor/Jenkins, but the declaration is required so Helm honours the nested `condition:` flags - without it Loki's MinIO/rollout-operator and the windows-exporter render too). |
| ArgoCD registration | One Application `prod/monitoring.yaml` (cluster singleton, like cnpg-operator), namespace `monitoring`, `ServerSideApply=true` (the CRDs exceed the client-side-apply annotation limit). |
| Metrics | kube-prometheus-stack: operator 0.93.1, Prometheus (15 d / 25 GiB cap, 30 GiB PVC), Alertmanager (2 GiB PVC), node-exporter, kube-state-metrics, default rules + dashboards. Prometheus selects ServiceMonitors / PodMonitors / PrometheusRules / Probes / ScrapeConfigs from **all** namespaces (`*NilUsesHelmValues: false`). |
| k3s control plane | controller-manager, scheduler, etcd, kube-proxy targets disabled (only reachable on 127.0.0.1 inside the k3s process); disabling the targets also disables their `...Down` rules. |
| Logs | Loki 3.6 single-binary, filesystem storage on a 30 GiB PVC, TSDB v13 schema, compactor retention 30 d, memcached caches / gateway / canary / helm-test disabled. |
| Log shipping | Alloy 1.19 DaemonSet (`discovery.kubernetes` + `loki.source.kubernetes` filtered to its own node, plus `loki.source.kubernetes_events`), labels `namespace`, `pod`, `container`, `node`, `app`, `job=<ns>/<container>`. |
| Grafana | 13.2 behind haproxy + cert-manager at `metrics.rrobinson.me`; admin credentials from the out-of-band Secret `grafana-admin-secret` (keys `admin-user`, `admin-password`); 5 GiB PVC (Recreate strategy); sidecar loads dashboards from ConfigMaps labelled `grafana_dashboard=1` in any namespace, folder from annotation `grafana_folder`; Loki added as a second datasource (uid `loki`). |
| Exposure | Only Grafana is exposed. Prometheus and Alertmanager stay ClusterIP (port-forward). |
| Security | node-exporter runs with `hostNetwork: false` (host-network would publish :9100 on the public IP). Operator admission webhook certificate issued by cert-manager instead of the chart's certgen Jobs. |
| Alerting | Alertmanager keeps the chart's default `null` receiver; adding a Discord/Slack receiver is a values change documented in the README. |
| Cosmic integration | In the Cosmic chart (`k8s/templates/metrics.yaml`, both branches): gameserver metrics Service, three ServiceMonitors (labels `env`, `component`), a PrometheusRule of game alerts, and a `Cosmic / Overview` dashboard ConfigMap. Rendered only when the `monitoring.coreos.com/v1` API exists. |
| Gameserver port clash | New `ServerSettings.metricsPort` (0 = app default) read by wServer and server; chart passes `gameserver.metricsPort` (dev 9092, prod 9090) into `wServer.json`. Code lands on `development` only; `master` receives manifest-only changes so no production pod restarts. |
| Resource budget | Requests ~0.7 CPU / ~2.3 GiB total; PVCs 30 + 30 + 5 + 2 GiB. |

## Components

### `monitoring/` umbrella chart (jojobobby/Argocd)

```
monitoring/
  Chart.yaml                 name monitoring-stack; dependencies declared with repository ""
  charts/                    kube-prometheus-stack-88.6.4.tgz, loki-7.3.0.tgz, alloy-1.12.1.tgz
  values.yaml                all config, keyed by subchart name
  templates/NOTES.txt
  secret.example.yaml        shape of grafana-admin-secret
  README.md                  operations doc (prereqs, verification, upgrades, alerts)
```

Names: `fullnameOverride: kps` + `cleanPrometheusOperatorObjectNames: true` gives
`kps` Prometheus / Alertmanager CRs (`prometheus-kps-0`, `alertmanager-kps-0` pods),
`kps-operator`, `monitoring-grafana`, `monitoring-kube-state-metrics`,
`monitoring-prometheus-node-exporter`, `loki`, `alloy`.

### Cosmic chart changes (jojobobby/Cosmic-LTS `k8s/`)

- `values.yaml`: `gameserver.metricsPort` (base 9090), `metrics.serviceMonitor.{enabled,interval}`,
  `metrics.alerts.enabled`, `metrics.dashboard.enabled`.
- `values.development.yaml`: `gameserver.metricsPort: 9092`; `values.production.yaml`: `9090`.
- `templates/configmap.yaml`: `"metricsPort": <port>` in `wServer.json`.
- `templates/gameserver.yaml` (development only): declares the metrics containerPort.
- `templates/metrics.yaml`: new Service `cosmic-gameserver-metrics-<env>`, ServiceMonitors,
  PrometheusRule, dashboard ConfigMap (`.Files.Get "dashboards/cosmic.json"`).
- `dashboards/cosmic.json`: generated dashboard, uid `cosmic-overview`, variables `env`, `namespace`.

### Server code (development branch)

- `Server-src/common/ConfigModels.cs`: `public int metricsPort { get; set; } = 0;`
- `Server-src/wServer/Program.cs` / `Server-src/server/Program.cs`: use `metricsPort` when > 0,
  otherwise the historical defaults (9090 / 9091). Log lines print the effective port.
- `Server-src/wServer.Tests/Config/ServerSettingsMetricsPortTests.cs`: default, numeric
  override, string override (the chart writes most settings as strings).

## Data flow

1. Prometheus (pod network) scrapes: kubelet/cAdvisor, apiserver, coredns, kube-state-metrics,
   node-exporter (pod network), operator, Grafana, Loki, Alloy, and every ServiceMonitor in
   the cluster - including Cosmic's appserver (pod IP :9091), Redis exporter (:9121) and
   gameserver (host IP :9090/9092, reachable because the pod IP of a hostNetwork pod is the
   node IP).
2. Alloy tails container logs through the kubelet API (no hostPath) and forwards them plus
   Kubernetes events to `http://loki.monitoring.svc:3100/loki/api/v1/push`.
3. Grafana reads Prometheus (`prometheus-operated:9090`) and Loki; dashboards arrive from
   ConfigMaps (kube-prometheus-stack defaults, Loki's own, Cosmic's).
4. Alertmanager receives rule evaluations from Prometheus; nothing is routed anywhere yet.

## Error handling / failure modes

- Grafana pod stays Pending until `grafana-admin-secret` exists (created out of band first).
- The Ingress exists before DNS; cert-manager retries HTTP-01 until `metrics.rrobinson.me`
  resolves. Grafana is reachable through `kubectl port-forward` meanwhile.
- If a ServiceMonitor target is down (e.g. the prod appserver that has been crash-looping
  since June), the Cosmic `...Down` alert fires - that is the intended signal.
- Prometheus is capped by `retentionSize` so the PVC cannot fill; Loki by compactor retention.
- ArgoCD `ignoreDifferences` on the webhook `caBundle` (cert-manager injects it) prevents a
  permanent OutOfSync.

## Testing / verification

- `helm template` of the umbrella chart and of the Cosmic chart (dev + prod values) with the
  cluster's Kubernetes version and the monitoring API version.
- `kubectl apply --server-side --dry-run=server` of the rendered umbrella chart.
- Alloy config checked with `alloy fmt` / `alloy validate` (container), Loki config with
  `loki -verify-config`, PrometheusRule with `promtool check rules`, dashboard JSON parsed.
- `dotnet test` for the new ServerSettings test.
- After sync: Application Healthy/Synced, all pods Running, Prometheus `/api/v1/targets`
  shows the Cosmic targets, Loki `/loki/api/v1/labels` lists `namespace`, Grafana `/api/health`
  and `/api/search?query=Cosmic` succeed, ingress + certificate state reported.

## Out of scope (reported, not changed)

- The prod appserver crash loop (enum `IncrementStatMoon` missing in image `tidan/cosmic:eab6ce8`)
  and the RollingUpdate/hostPort deadlock that keeps its replacement Pending.
- Host firewall rules for the public IP (9090-9092 and anything else host-network).
- Cloudflare DNS record for `metrics.rrobinson.me`.
- Alert notification receivers (Discord etc.).
