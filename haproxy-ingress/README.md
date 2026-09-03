# haproxy-ingress — HAProxy Ingress Controller via ArgoCD

The cluster's only ingress controller (IngressClass `haproxy`, default class). Every public
host — rrobinson.me services, tidansrealm.com, cosmicv2.com, betterourself.com — comes in
through the LoadBalancer Service in the `haproxy-ingress` namespace (MetalLB, 147.135.8.100:80/443).

| | |
|---|---|
| Chart | `kubernetes-ingress` 1.42.0 from https://haproxytech.github.io/helm-charts (controller 3.0.4), declared as a dependency like `argocd/` and `cnpg-operator/` — ArgoCD runs `helm dependency build` at sync |
| ArgoCD app | `Both/Arcana-Argocd-Apps/prod/haproxy-ingress.yaml` → path `haproxy-ingress/`, **releaseName `haproxy-ingress`** (do not change: resource names and the `--configmap` / `--publish-service` arguments derive from it) |
| Config | `values.yaml` — all controller settings, including the global ConfigMap keys under `controller.config` |

## History

Installed by hand with Helm on 2026-05-01 (`helm install haproxy-ingress ...`, revision 2) and
adopted into GitOps on 2026-09-03. Before the takeover a render of the same chart version with
the release's own values was diffed against the live manifest and every persistent resource was
identical, so ArgoCD's first sync only added the `ssl-redirect-port` key to the ConfigMap. The
old Helm release records (`sh.helm.release.v1.haproxy-ingress.*` Secrets) were deleted the
same day; `helm list` no longer knows the release and ArgoCD is the only owner.

## Why `ssl-redirect-port: "443"`

The controller listens on 8443 inside its pod and, by default, builds HTTP→HTTPS redirects
with that port. The Service maps 443 → 8443, so every `http://<host>/` request used to be
redirected to `https://<host>:8443/`, which is not reachable. With the key set, redirects go to
`https://<host>/`.

## Changing controller configuration

Add keys under `kubernetes-ingress.controller.config` in `values.yaml` (see
https://www.haproxy.com/documentation/kubernetes-ingress/community/configuration-reference/configmap/
— e.g. `timeout-client`, `syslog-server`, `ssl-redirect`). The controller watches its ConfigMap
and reloads without a restart. Changes to the Deployment itself (replicas, image, args) roll
the pods; there are two replicas behind the Service, so a rollout is not an outage.

## Default certificate

Hosts without their own cert-manager certificate get the self-signed fallback in Secret
`haproxy-ingress-kubernetes-ingress-default-cert`, generated once by the original install.
`values.yaml` references it as an existing secret on purpose: the chart's own template is a
Helm pre-install hook that would otherwise re-generate (and ArgoCD rotate) it on every sync.

## Upgrading

Bump `version` in `Chart.yaml`, diff the new chart's `values.yaml` against ours, render and
compare with the live manifest as was done for the adoption:

```bash
cd haproxy-ingress && helm dependency build && \
helm template haproxy-ingress . -n haproxy-ingress -f values.yaml --kube-version v1.35.4
```

Do not commit the `charts/` directory or `Chart.lock` that `helm dependency build` creates —
the sibling charts don't, and ArgoCD resolves the dependency itself.
