# external-secrets

External Secrets Operator (ESO) for the cluster, plus the cluster-wide
`openbao` `ClusterSecretStore` that lets any namespace pull secrets out of the
self-managed OpenBao (`secrets-manager` namespace).

Deployed by ArgoCD Application `external-secrets` (`prod/external-secrets.yaml`
in `Arcana-Argocd-Apps`) into namespace `external-secrets`. Same shape as
`cnpg-operator/`: an umbrella chart that vendors the upstream chart plus a few
local templates.

## What it creates

| Resource | Purpose |
|---|---|
| ESO controller / webhook / cert-controller | reconciles `ExternalSecret` → Kubernetes `Secret` |
| CRDs (`external-secrets.io/v1`) | `ExternalSecret`, `ClusterSecretStore`, … |
| ServiceAccount `eso-openbao` | login identity ESO presents to OpenBao's Kubernetes auth (no privileges of its own) |
| `ClusterSecretStore/openbao` | vault provider → `http://openbao.secrets-manager.svc:8200`, KV v2 mount `tidan`, Kubernetes auth role `arcana-eso` |

OpenBao reviews the login JWT with its own pod ServiceAccount, which already
holds `system:auth-delegator` (`openbao-server-binding`) — so ESO's SA needs no
cluster RBAC.

## Prerequisite: OpenBao must be configured (one-time, needs the root token)

The store stays `Ready=False` until OpenBao has the Kubernetes auth method,
policy and role that back it. That config lives in OpenBao's Raft storage, not
in git. The Arcana repo carries the script that provisions it
(`scratchpad`/runbook in `Arcana/k8s/README.md` → "OpenBao + ESO"):

- `bao secrets enable -path=tidan kv-v2`
- `bao auth enable kubernetes` + `bao write auth/kubernetes/config kubernetes_host=https://kubernetes.default.svc`
- policy `arcana-eso`: `read` on `tidan/data/arcana/*`
- role `arcana-eso`: bound SA `eso-openbao` / ns `external-secrets`, policy `arcana-eso`

## Verify

```bash
kubectl --context rrobinson-services -n external-secrets get pods
kubectl --context rrobinson-services get clustersecretstore openbao -o jsonpath='{.status.conditions}'
# want: type Ready, status "True", reason Valid
```

If `Ready=False` with a 403, the OpenBao role/policy is missing or the SA name
in the role doesn't match `eso-openbao`. If it's a token-review error, the
OpenBao pod SA lost `system:auth-delegator` (check `openbao-server-binding`).
