# tidangames.com data services: Postgres, Redis, secrets manager

Date: 2026-09-17. Status: approved in chat (OpenBao chosen over Infisical; private repos;
secrets manager on its own storage; build and deploy everything).

## Goal

Three self-managed, cluster-wide services on the `rrobinson-services` k3s node, each with a
web UI on its own `tidangames.com` host, each in its own GitHub repo that holds nothing but
a Helm chart, each watched by its own ArgoCD Application:

| Service | Repo (private) | Local clone | Namespace | Host | Runs |
|---|---|---|---|---|---|
| Postgres | `jojobobby/tidan-postgres` | `Both/Postgres` | `postgres` | pdb.tidangames.com | CloudNativePG `Cluster` + pgAdmin 4 |
| Redis | `jojobobby/tidan-redis` | `Both/Redis` | `redis` | rdb.tidangames.com | Redis 8 StatefulSet + RedisInsight |
| Secrets manager | `jojobobby/tidan-secrets-manager` | `Both/SecretsManager` | `secrets-manager` | sm.tidangames.com | OpenBao (official chart, vendored) |

Production-only, one instance each, like the other cluster-infra apps (monitoring, umami,
harbor is the exception with a dev copy). Nothing sensitive is ever committed.

## What already existed

- `cnpg-operator` ArgoCD app (CloudNativePG 1.25.0) from `jojobobby/Argocd`. Supports
  Postgres up to 17, so the cluster image is `ghcr.io/cloudnative-pg/postgresql:17.11`.
- `umami/` in the same repo is the template for "CNPG Cluster + web app + haproxy ingress".
- Redis only as embedded StatefulSets inside Cosmic/Harbor/BetterUs/ArgoCD; no standalone.
- No secrets manager anywhere; no `tidangames.com` references anywhere in `Both`.
- DNS (Route 53): `pdb` and `rdb` already point at 147.135.8.100. `sm` has no record yet.

## Shared conventions

- Chart at the **repo root** (Chart.yaml, values.yaml, templates/, README.md,
  secret.example.yaml). `path: .` in the Application.
- Ingress: `ingressClassName: haproxy`, `cert-manager.io/cluster-issuer: letsencrypt-prod`,
  TLS secret `<host>-tls`. Only the web UIs are exposed; the databases stay ClusterIP.
- Storage: k3s default `local-path` StorageClass.
- ArgoCD Application in `Arcana-Argocd-Apps/prod/<name>.yaml`: repoURL
  `git@github.com:jojobobby/tidan-<name>.git` (the existing `repo-github-jojobobby` SSH
  credential covers all jojobobby repos, public or private), `targetRevision: main`,
  automated sync with prune + selfHeal, `CreateNamespace`, `ApplyOutOfSyncOnly`,
  `ServerSideApply`, retry block, finalizer. App-of-apps recursion registers it on push.
- Out-of-band Secrets are created with `kubectl` before first sync, documented in each
  repo's `secret.example.yaml`; generated values are handed to the user in chat once.
- Images pinned to exact versions; no Bitnami (images paywalled).

## Postgres (`tidan-postgres`, chart `postgres`)

- `templates/cluster.yaml`: CNPG `Cluster` named `postgres`, 1 instance, 10Gi
  (`resizeInUseVolumes`), image 17.11, `enableSuperuserAccess: true` (so the operator
  writes `postgres-superuser` with a ready `pgpass`), `bootstrap.initdb` database `tidan`
  owner `tidan` (operator writes `postgres-app` with `uri`), `shared_buffers 256MB`,
  `max_connections 100`, requests 256Mi/100m, limits 2Gi/2 CPU.
- Consumers connect at `postgres-rw.postgres.svc:5432` using either generated Secret.
- `templates/pgadmin.yaml`: Deployment `pgadmin` (Recreate), `dpage/pgadmin4:9.18.0`,
  `PGADMIN_DEFAULT_EMAIL` / `PGADMIN_DEFAULT_PASSWORD` from Secret `pgadmin-admin`
  (keys `email`, `password`), `PGADMIN_LISTEN_PORT=5050`, 1Gi PVC at `/var/lib/pgadmin`,
  runs as uid/gid 5050. A ConfigMap `servers.json` pre-registers the cluster
  (Host `postgres-rw`, port 5432, user `postgres`, MaintenanceDB `postgres`,
  `PassFile: /pgpass`) and `PGADMIN_REPLACE_SERVERS_ON_STARTUP=True` keeps it declarative.
  An init container copies the `pgpass` key of `postgres-superuser` to
  `/var/lib/pgadmin/storage/<email with @ -> _>/pgpass` (mode 0600), which is where
  pgAdmin server mode resolves `PassFile`. Probe `GET /misc/ping`.
- Service `pgadmin` :80 -> 5050; Ingress `pdb.tidangames.com`.

## Redis (`tidan-redis`, chart `redis`)

- `templates/redis.yaml`: ConfigMap `redis.conf` (`appendonly yes`, `appendfsync
  everysec`, RDB `save` points, `maxmemory` from values, `maxmemory-policy noeviction`,
  `protected-mode yes`), StatefulSet `redis` (1 replica, `redis:8.8-alpine`, PVC 5Gi at
  `/data`, uid/gid 999, `--requirepass` taken from Secret `redis-auth` key `password` via
  `$(REDIS_PASSWORD)` expansion), probes via `redis-cli --no-auth-warning ping`,
  ClusterIP Service `redis` :6379 plus headless `redis-headless`.
- Consumers connect at `redis.redis.svc:6379` with the password.
- `templates/redisinsight.yaml`: Deployment `redisinsight` (`redis/redisinsight:3.8.0`,
  port 5540, 1Gi PVC at `/data`, uid/gid 1000). Pre-wired with `RI_REDIS_HOST=redis`,
  `RI_REDIS_PORT=6379`, `RI_REDIS_ALIAS`, `RI_REDIS_PASSWORD` (Secret) and
  `RI_ENCRYPTION_KEY` (Secret `redis-auth` key `insight-encryption-key`) so stored
  credentials are encrypted at rest. Probe `GET /api/health/`.
- RedisInsight has no login of its own, so the Ingress `rdb.tidangames.com` adds haproxy
  basic auth: `haproxy.org/auth-type: basic-auth`, `haproxy.org/auth-secret:
  redisinsight-basic-auth` (Secret mapping username -> crypt(3) hash),
  `haproxy.org/auth-realm`, and `haproxy.org/timeout-server: 60s` (RedisInsight warns
  that some requests exceed 30s).

## Secrets manager (`tidan-secrets-manager`, chart `secrets-manager`)

- Umbrella chart vendoring the official `openbao` chart 0.29.5 (OpenBao v2.6.2) as
  `charts/openbao-0.29.5.tgz`, declared under `dependencies:` with `repository: ""` (the
  monitoring-stack pattern, so condition flags are honoured and ArgoCD renders offline).
  All settings under the `openbao:` key in values.yaml.
- Server: standalone mode, 1 replica, integrated Raft storage on a 10Gi PVC at
  `/openbao/data` (raft gives `bao operator raft snapshot` backups later), `ui = true`,
  TCP listener with `tls_disable = 1` (TLS terminates at haproxy, same as every other
  service), `updateStrategyType: RollingUpdate`, `ui.enabled: true`, injector and CSI
  disabled (add later if pods should consume secrets directly).
- **Auto-unseal** with OpenBao's `seal "static"`: `current_key = "env://BAO_SEAL_KEY"`,
  the env var injected from Secret `openbao-unseal` (key `key`, 32 random bytes,
  base64) via `server.extraSecretEnvironmentVars`. Without this, every pod restart on
  the single node would leave the vault sealed until someone typed unseal keys. The
  trade-off (documented in the README): whoever can read that Secret can unseal the
  data; that is the same trust boundary as the node itself. The key id is dated so it
  can be rotated with `previous_key`.
- Ingress via the chart's `server.ingress` for `sm.tidangames.com`.
- After the first sync the store is initialised once by hand
  (`bao operator init` in the pod). With an auto-unseal seal this yields a root token
  and recovery keys, which are handed to the user in chat and never stored in the
  cluster. The README carries the exact command.

## Out-of-band Secrets (created by kubectl before the first sync)

| Namespace | Secret | Keys |
|---|---|---|
| postgres | `pgadmin-admin` | `email`, `password` |
| redis | `redis-auth` | `password`, `insight-encryption-key` |
| redis | `redisinsight-basic-auth` | `<username>: <crypt(3) sha512 hash>` |
| secrets-manager | `openbao-unseal` | `key` |

## Verification

1. `helm template` of every chart renders cleanly with Helm 3.17 (ArgoCD 2.14's line).
2. After pushing: each Application appears, syncs, reports Healthy; pods Ready; PVCs Bound.
3. `curl -I https://pdb.tidangames.com`, `https://rdb.tidangames.com` (expect 401 without
   basic auth, 200 with) and `https://sm.tidangames.com` answer with Let's Encrypt
   certificates. `sm` waits on the user's DNS record; cert-manager retries on its own.
4. Postgres: pgAdmin shows the pre-registered server and can open it. Redis: RedisInsight
   shows the pre-wired database as connected. OpenBao: initialised, unsealed, UI login with
   the root token works.

## Explicitly out of scope (follow-ups)

Off-node backups (CNPG `ScheduledBackup`, Raft snapshot agent, Redis RDB copies),
NetworkPolicies restricting which namespaces may reach the databases, Prometheus
ServiceMonitors, the OpenBao Kubernetes auth method / injector, a dev copy of any service.
