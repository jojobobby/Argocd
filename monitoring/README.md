# monitoring-stack — Prometheus + Loki + Grafana on k3s via ArgoCD

This directory is a Tidan **umbrella Helm chart** that deploys the cluster observability
stack onto the shared k3s cluster (`rrobinson-services`, node `machine-01`). It follows the
same GitOps convention as everything else: an ArgoCD `Application`
(`Both/Arcana-Argocd-Apps/prod/monitoring.yaml`) points at this repo's `monitoring/` path
and renders the chart with `values.yaml`.

One cluster-wide instance lives in the `monitoring` namespace and scrapes **every**
namespace. Services declare their own scrape targets, alert rules and dashboards in their
own repos (see "How a service plugs in" below) — Cosmic-LTS does exactly that from its
`k8s/templates/metrics.yaml`.

| Component | Chart (vendored) | App version | What it does |
|---|---|---|---|
| Prometheus Operator, Prometheus, Alertmanager, node-exporter, kube-state-metrics, Grafana, default rules + dashboards | `kube-prometheus-stack` 88.6.4 | operator v0.93.1, Grafana 13.2.0 | Metrics, alerting, dashboards |
| Loki (single binary, filesystem) | `loki` 7.3.0 | Loki 3.6.12 | Log storage, 30-day retention |
| Alloy (DaemonSet) | `alloy` 1.12.1 | Alloy v1.19.2 | Ships every pod's logs + Kubernetes events to Loki |

## Layout

```
monitoring/
├── Chart.yaml                 # umbrella chart (name: monitoring-stack) — declares the 3 deps
├── charts/
│   ├── kube-prometheus-stack-88.6.4.tgz   # VENDORED upstream charts
│   ├── loki-7.3.0.tgz
│   └── alloy-1.12.1.tgz
├── values.yaml                # ALL configuration, keyed by subchart name
├── secret.example.yaml        # how to create the required Grafana admin Secret
├── templates/NOTES.txt
└── README.md
```

### Why the subcharts are vendored AND declared (this differs from Harbor/Jenkins)

The upstream charts are committed as packaged tgz files in `charts/`, so `helm template`
(and therefore ArgoCD's repo-server) renders with **no network access** at sync time.

Unlike the Harbor and Jenkins stacks they are also listed under `dependencies:` in
`Chart.yaml`, each with an **empty `repository`**. Helm only evaluates `condition:` flags when
the parent declares its dependencies; without the declaration every *nested* subchart renders
unconditionally — Loki's bundled MinIO, rollout-operator and grafana-agent-operator, and
kube-prometheus-stack's windows-exporter would all be deployed (verified: that is exactly what
the first render did). With the declaration, `helm template` loads the tgz files straight from
`charts/`, honours the conditions, and `helm lint` is clean.

ArgoCD only runs `helm dependency build` when `helm template` fails with a *missing dependency*
error, which cannot happen while the tgz files are present. (`helm dependency build` itself does
not work with tgz-vendored, repository-less deps — it wants an unpacked `charts/<name>/`
directory. Don't run it; it is not needed.)

Verify a change with a render, mirroring what ArgoCD does:

```bash
cd monitoring
helm template monitoring . -f values.yaml --namespace monitoring --include-crds \
  --kube-version v1.35.4 $(kubectl api-versions | sed 's/^/--api-versions /' | tr '\n' ' ')
```

## One-time prerequisites

1. **Grafana admin Secret** (never in git). The chart reads it via
   `grafana.admin.existingSecret`; the Grafana pod cannot start without it:

   ```bash
   kubectl create namespace monitoring
   kubectl -n monitoring create secret generic grafana-admin-secret \
     --from-literal=admin-user="admin" \
     --from-literal=admin-password="$(openssl rand -base64 24)"
   ```

   Read it back: `kubectl -n monitoring get secret grafana-admin-secret -o jsonpath='{.data.admin-password}' | base64 -d`

2. **DNS**: `metrics.rrobinson.me` → `147.135.8.100` (Cloudflare, DNS-only like the other
   rrobinson.me hosts). cert-manager (`letsencrypt-prod`, HTTP-01 through the haproxy ingress)
   issues the certificate as soon as the name resolves; until then the Ingress simply waits.

## How it's wired into ArgoCD

| File | App | Namespace | Source |
|---|---|---|---|
| `Both/Arcana-Argocd-Apps/prod/monitoring.yaml` | `monitoring` | `monitoring` | `git@github.com:jojobobby/Argocd.git`, path `monitoring/`, release `monitoring` |

Sync options that matter:

- `ServerSideApply=true` — **mandatory**: the prometheus-operator CRDs are far bigger than the
  client-side-apply annotation limit. SSA also lets ArgoCD adopt the v0.87 CRDs that were left in
  the cluster by an earlier (May 2026) attempt and upgrade them to v0.93.
- `ignoreDifferences` on the admission webhooks' `caBundle` — cert-manager's cainjector fills it
  in at runtime.
- `prune` + `selfHeal` like every other app. PVCs are never pruned (Prometheus/Alertmanager use
  volumeClaimTemplates; Loki's StatefulSet keeps its PVC on delete/scale; Grafana's PVC is a
  plain chart resource that ArgoCD only removes if the whole app is deleted).

## What gets deployed (and how to reach it)

| Resource | Name | Exposure |
|---|---|---|
| Grafana | Deployment `monitoring-grafana`, Service `monitoring-grafana:80` | **https://metrics.rrobinson.me** (haproxy + Let's Encrypt). User `admin`, password from the Secret. |
| Prometheus | CR `kps` → pod `prometheus-kps-0`, Service `prometheus-operated:9090` / `kps-prometheus` | not exposed: `kubectl -n monitoring port-forward svc/prometheus-operated 9090` |
| Alertmanager | CR `kps` → pod `alertmanager-kps-0`, Service `alertmanager-operated:9093` | not exposed: `kubectl -n monitoring port-forward svc/alertmanager-operated 9093` |
| Prometheus Operator | Deployment `kps-operator` | — |
| node-exporter | DaemonSet `monitoring-prometheus-node-exporter` (pod network, see below) | — |
| kube-state-metrics | Deployment `monitoring-kube-state-metrics` | — |
| Loki | StatefulSet `loki`, Service `loki:3100` | in-cluster only (`http://loki.monitoring.svc.cluster.local:3100`) |
| Alloy | DaemonSet `alloy`, Service `alloy:12345` (UI) | `kubectl -n monitoring port-forward svc/alloy 12345` |

Storage (all on the default `local-path` StorageClass, which cannot be resized — pick sizes
deliberately): Prometheus 30 Gi (15 d retention, hard cap 25 GiB), Loki 30 Gi (30 d retention),
Grafana 5 Gi, Alertmanager 2 Gi.

Resource requests total roughly 0.7 CPU / 2.3 GiB; memory limits are set on every component.

## Decisions tuned to this cluster

- **node-exporter runs with `hostNetwork: false`.** `machine-01` has a public IP and no host
  firewall (the Cosmic gameserver's host-network metrics port answers from the internet). The
  upstream default would publish node-exporter on `<public IP>:9100`. On the pod network the
  host CPU / memory / disk / filesystem / load metrics are unchanged (they come from the hostPath
  `/proc`, `/sys`, `/` mounts with `hostPID`); only `node_network_*` reflects the pod's own
  network namespace. Consider a host firewall regardless — it is the real fix.
- **k3s control plane targets are disabled** (`kubeControllerManager`, `kubeScheduler`,
  `kubeEtcd`, `kubeProxy`). k3s runs them inside its own process listening on 127.0.0.1 only,
  and uses sqlite/kine instead of etcd, so nothing can scrape them from a pod. Disabling the
  targets also disables their `...Down` alerts. To scrape controller-manager / scheduler later,
  start k3s with `--kube-controller-manager-arg=bind-address=0.0.0.0` (and the scheduler
  equivalent), then flip the flags back on.
- **Admission webhook certificate from cert-manager** (`prometheusOperator.admissionWebhooks.certManager.enabled`)
  instead of the chart's `kube-webhook-certgen` Jobs, which are Helm hooks ArgoCD handles poorly.
- **Prometheus selects everything** (`serviceMonitorSelectorNilUsesHelmValues: false` and the
  four siblings): ServiceMonitors, PodMonitors, PrometheusRules, Probes and ScrapeConfigs from any
  namespace, no `release` label required.
- **Loki single-binary on the filesystem**: right size for one node; the simple-scalable
  targets, gateway, memcached caches (1 + 8 GiB by default!), canary and helm-test are off.
- **Only Grafana is exposed.** Prometheus / Alertmanager / Alloy have no Ingress.

## How a service plugs in

Nothing in this chart needs to change. From the service's own chart:

1. A **Service** exposing the metrics port (named port), and a **ServiceMonitor** selecting it:

   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata: { name: my-svc, labels: { app: my-svc } }
   spec:
     jobLabel: app                 # job = the Service's `app` label
     selector: { matchLabels: { app: my-svc-metrics } }
     endpoints:
       - port: metrics
         interval: 30s
   ```

2. Alerts as a **PrometheusRule** (any namespace, any labels).
3. A dashboard as a **ConfigMap** labelled `grafana_dashboard: "1"` with the JSON under any
   `*.json` key; the annotation `grafana_folder: <Folder>` files it in that Grafana folder.

Guard them with `{{ if .Capabilities.APIVersions.Has "monitoring.coreos.com/v1" }}` so the chart
still renders on clusters without the operator. Cosmic-LTS (`k8s/templates/metrics.yaml`,
`k8s/dashboards/cosmic.json`) is the reference implementation: appserver / gameserver / redis
ServiceMonitors with `env` + `component` labels, a `cosmic-<env>` PrometheusRule and the
"Cosmic / Overview" dashboard.

Logs need nothing at all: Alloy tails every container on the node and labels lines with
`namespace`, `pod`, `container`, `node`, `app` (the pod's `app` label, else
`app.kubernetes.io/name`) and `job=<namespace>/<container>`. Kubernetes events land under
`job="kubernetes-events"`. In Grafana → Explore → Loki: `{namespace="cosmic-production"}`.

## Users and access

Grafana OSS has no per-datasource permissions, so **folders are the access boundary**:

| Folder | Content | Who can see it |
|---|---|---|
| `Cosmic` | Cosmic / Overview (from the Cosmic-LTS chart) | everyone, including the Viewer role and the `Cosmic Viewers` team |
| `Cosmic Wiki` | Cosmic / Wiki (from the Cosmic-LTS chart, wiki-enabled releases only) | everyone, same as `Cosmic` |
| `Kubernetes` | the 25 kube-prometheus-stack dashboards (annotated via `grafana.sidecar.dashboards.annotations` in values.yaml) | Editors and Admins only — the Viewer role's permission was removed from the folder |
| `Loki` | Loki's operational dashboards | Editors and Admins only — same |

Accounts (all created through the admin API, none in git):

- `admin` — the chart's built-in admin, password in the `grafana-admin-secret` Secret.
- `raphealsmall@gmail.com` — Grafana **Server Admin** + Main Org **Admin** (the owner's account).
- `dangergun`, `germoele`, `loxis`, `polity`, `xdelik` — Main Org **Viewer**, members of team `Cosmic Viewers`.
  A Viewer can open the Cosmic dashboards and their panels query Prometheus/Loki through
  them, but has **no Explore access** (the Viewer role lacks `datasources:explore`) and
  cannot see the Kubernetes or Loki folders.

To add another Cosmic viewer: Administration → Users → New user, role **Viewer**, then add
them to the `Cosmic Viewers` team (or just leave them as Viewer — the Cosmic folder is
visible to the whole Viewer role). To hide a new folder from Viewers: folder → Manage →
Permissions → remove the `Viewer` role entry. Dashboards a service publishes without a
`grafana_folder` annotation land in the root and are visible to Viewers.

## Privacy: player IP addresses

The Cosmic game servers log every connecting client's address and the wiki's Apache access
log carries visitor addresses. Because Cosmic dashboards are shared with viewer accounts and
Grafana OSS cannot hide fields per user, IPv4 addresses are masked **before storage**: the
Alloy pipeline (`loki.process "privacy"` in values.yaml) rewrites every IPv4 in the
`cosmic-*` namespaces to `[ip]`. Nothing in Loki, and therefore nothing in Grafana, contains
a player address; the unmasked originals exist only in the servers' own log files on the
node. Lines stored before the rule went live (2026-09-03 22:30 UTC) were purged with a Loki
delete request. The viewer-facing log panels additionally apply a `line_format` mask at
query time. IPv6 is not masked: players reach the servers over IPv4 only (the LoadBalancer
and the game hostPorts are IPv4).

Ingress traffic metrics come from the HAProxy ingress controller (`haproxy_backend_*`,
`haproxy_frontend_*`) via its ServiceMonitor in the `haproxy-ingress` namespace; they carry
backend names, never client addresses.

## Alert notifications

Alertmanager runs with the chart's default routing: everything goes to a `null` receiver, so
nothing pages anyone yet. To notify a Discord channel, add under `kube-prometheus-stack.alertmanager`
in `values.yaml`:

```yaml
    config:
      route:
        receiver: discord
        group_by: ['namespace', 'alertname']
        routes:
          - receiver: 'null'
            matchers: [ 'alertname = "Watchdog"' ]
      receivers:
        - name: 'null'
        - name: discord
          discord_configs:
            - webhook_url_file: /etc/alertmanager/secrets/alertmanager-discord/webhook
    alertmanagerSpec:
      secrets: [ alertmanager-discord ]     # kubectl -n monitoring create secret generic alertmanager-discord --from-file=webhook=...
```

Keep the webhook URL in a Secret (mounted via `alertmanagerSpec.secrets`), never in git.
`slack_configs` / `webhook_configs` work the same way.

## Verifying a deployment

```bash
kubectl -n argocd get app monitoring                       # Synced / Healthy
kubectl -n monitoring get pods                             # everything Running
kubectl -n monitoring port-forward svc/prometheus-operated 9090 &
curl -s localhost:9090/api/v1/targets | python -c "import json,sys; [print(t['labels'].get('job'), t['health']) for t in json.load(sys.stdin)['data']['activeTargets']]"
kubectl -n monitoring port-forward svc/loki 3100 &
curl -s localhost:3100/loki/api/v1/labels                  # should list namespace, pod, container, ...
kubectl -n monitoring get ingress,certificate               # cert READY once DNS exists
```

## Upgrading a component

1. `helm pull <chart> --repo <repo-url> --version <new>` into `charts/`, delete the old tgz
   (`helm pull kube-prometheus-stack --repo https://prometheus-community.github.io/helm-charts`,
   `helm pull loki --repo https://grafana.github.io/helm-charts`,
   `helm pull alloy --repo https://grafana.github.io/helm-charts`).
2. Update the `version:` of that dependency in `Chart.yaml` (must match the file name) and
   `appVersion`.
3. Diff the new upstream `values.yaml` against ours and reconcile renamed/removed keys.
4. Re-render with `helm template` (command above), commit, push — ArgoCD rolls it out. CRD
   changes are applied in the same sync (ServerSideApply).

## Troubleshooting

- **App stuck OutOfSync on a CRD / "metadata.annotations: Too long"** — ServerSideApply is
  missing from the Application; it is required.
- **Grafana pod Pending / CreateContainerConfigError** — `grafana-admin-secret` is missing in
  `monitoring` (see prerequisites).
- **Certificate not Ready** — DNS for `metrics.rrobinson.me` does not resolve yet, or the
  haproxy HTTP-01 solver cannot be reached on port 80.
- **A target shows `down` for a hostNetwork pod (Cosmic gameserver)** — the process failed to
  bind its metrics port at startup (dev and prod share the node's port space; they must use
  different `gameserver.metricsPort` values), or the pod needs a restart to pick up a new port.
- **Webhook caBundle keeps showing as a diff** — the Application's `ignoreDifferences` block was
  removed; restore it.
