# tidangames.com data services Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Three self-managed services (Postgres + pgAdmin, Redis + RedisInsight, OpenBao) on the k3s cluster, each a Helm chart in its own private GitHub repo, each deployed by its own ArgoCD Application, reachable at pdb/rdb/sm.tidangames.com.

**Architecture:** Plain Helm charts at each repo root (OpenBao wraps the official chart as a vendored subchart). ArgoCD Applications live in `Arcana-Argocd-Apps/prod/` and are auto-registered by the app-of-apps. Credentials are out-of-band Kubernetes Secrets created with kubectl before the first sync.

**Tech Stack:** Helm 3.17 (scratchpad binary), CloudNativePG 1.25 (already on the cluster), Postgres 17.11, pgAdmin 9.18.0, Redis 8.8-alpine, RedisInsight 3.8.0, OpenBao 2.6.2 via openbao-helm 0.29.5, haproxy ingress 3.0, cert-manager letsencrypt-prod, ArgoCD 2.14.

**Spec:** `Both/Argocd/docs/superpowers/specs/2026-09-17-tidangames-data-services-design.md`

## Global Constraints

- Repos: `jojobobby/tidan-postgres`, `jojobobby/tidan-redis`, `jojobobby/tidan-secrets-manager`, private, default branch `main`, chart at repo root.
- Local clones: `Both/Postgres`, `Both/Redis`, `Both/SecretsManager`.
- Namespaces `postgres`, `redis`, `secrets-manager`. Hosts `pdb.tidangames.com`, `rdb.tidangames.com`, `sm.tidangames.com`.
- Ingress: `ingressClassName: haproxy`, annotation `cert-manager.io/cluster-issuer: letsencrypt-prod`, TLS secret `<host>-tls`.
- Images pinned: `ghcr.io/cloudnative-pg/postgresql:17.11`, `dpage/pgadmin4:9.18.0`, `redis:8.8-alpine`, `redis/redisinsight:3.8.0`, `quay.io/openbao/openbao:2.6.2`, `busybox:1.36`.
- No secret values in git. Out-of-band Secrets: `postgres/pgadmin-admin` (email, password), `redis/redis-auth` (password, insight-encryption-key), `redis/redisinsight-basic-auth` (`<user>: <crypt hash>`), `secrets-manager/openbao-unseal` (key).
- Render check for every chart: `helm template <name> <dir>` with the scratchpad Helm (`.../scratchpad/helm/windows-amd64/helm.exe`) must exit 0.
- Commit author: the global git identity (Jojobobby). Commit messages end with `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`.
- `Arcana-Argocd-Apps` has four pre-existing untracked files (better-accounts, slabbed); never add them.

---

### Task 1: Postgres chart (`Both/Postgres`)

**Files:**
- Create: `Both/Postgres/Chart.yaml`, `values.yaml`, `templates/cluster.yaml`, `templates/pgadmin.yaml`, `secret.example.yaml`, `README.md`

**Interfaces:**
- Produces: CNPG Cluster `postgres` -> Services `postgres-rw/-ro/-r`, Secrets `postgres-app` (uri, pgpass...) and `postgres-superuser` (pgpass). Deployment/Service `pgadmin`. Ingress `pgadmin` on pdb.tidangames.com.

- [ ] **Step 1: Chart.yaml**

```yaml
apiVersion: v2
name: postgres
description: >-
  Self-managed PostgreSQL for tidangames.com services: one CloudNativePG cluster (the
  cnpg-operator ArgoCD app must be present) plus pgAdmin 4 at pdb.tidangames.com,
  pre-wired to the cluster's superuser.
type: application
version: 0.1.0
appVersion: "17.11"
```

- [ ] **Step 2: values.yaml**

```yaml
postgres:
  name: postgres
  imageName: ghcr.io/cloudnative-pg/postgresql:17.11
  instances: 1
  storage: 10Gi
  database: tidan
  owner: tidan
  parameters:
    shared_buffers: 256MB
    max_connections: "100"
  resources:
    requests: {memory: 256Mi, cpu: 100m}
    limits: {memory: 2Gi, cpu: "2"}

pgadmin:
  image: {repository: dpage/pgadmin4, tag: "9.18.0", pullPolicy: IfNotPresent}
  existingSecret: pgadmin-admin
  listenPort: 5050
  storage: 1Gi
  serverName: Tidan Postgres
  resources:
    requests: {memory: 256Mi, cpu: 50m}
    limits: {memory: 1Gi, cpu: "1"}

ingress:
  className: haproxy
  clusterIssuer: letsencrypt-prod
  host: pdb.tidangames.com
  tlsSecret: pdb.tidangames.com-tls
```

- [ ] **Step 3: templates/cluster.yaml** (CNPG Cluster with `enableSuperuserAccess: true`, initdb database/owner from values, parameters and resources via `toYaml`).

- [ ] **Step 4: templates/pgadmin.yaml** (ConfigMap `pgadmin-servers` with servers.json Host `<name>-rw`, PassFile `/pgpass`; PVC `pgadmin-data`; Deployment Recreate, pod securityContext 5050/5050, init container busybox copying `/cnpg/pgpass` to `/var/lib/pgadmin/storage/<email @->_>/pgpass` chmod 600; env from Secret; probes `/misc/ping`; Service :80->5050; Ingress).

- [ ] **Step 5: secret.example.yaml + README.md** (how to create `pgadmin-admin`, how consumers connect, where the generated Secrets are, upgrade notes).

- [ ] **Step 6: Render** `helm template postgres Both/Postgres` -> exit 0, contains `kind: Cluster`, `kind: Deployment`, `kind: Ingress`.

- [ ] **Step 7: git init, commit** `feat: Postgres (CloudNativePG) + pgAdmin chart for pdb.tidangames.com`.

### Task 2: Redis chart (`Both/Redis`)

**Files:**
- Create: `Both/Redis/Chart.yaml`, `values.yaml`, `templates/redis.yaml`, `templates/redisinsight.yaml`, `secret.example.yaml`, `README.md`

**Interfaces:**
- Produces: StatefulSet `redis`, Services `redis` (ClusterIP 6379) and `redis-headless`; Deployment/Service `redisinsight`; Ingress `redisinsight` with haproxy basic auth on rdb.tidangames.com.

- [ ] **Step 1: Chart.yaml** (name `redis`, appVersion `8.8`).
- [ ] **Step 2: values.yaml** (redis image/tag, `existingSecret: redis-auth`, storage 5Gi, `maxmemory: 1gb`, resources; redisinsight image/tag, storage 1Gi, alias, `basicAuthSecret: redisinsight-basic-auth`, realm; ingress block for rdb).
- [ ] **Step 3: templates/redis.yaml**: ConfigMap `redis-config` (redis.conf with `include /etc/redis-secret/requirepass.conf`, appendonly, save points, maxmemory, noeviction, protected-mode); StatefulSet with init container that writes `requirepass <password>` from env into an emptyDir; container `redis-server /etc/redis/redis.conf`; probes `redis-cli --no-auth-warning -a "$REDIS_PASSWORD" ping`; securityContext 999; PVC template 5Gi; Services.
- [ ] **Step 4: templates/redisinsight.yaml**: PVC, Deployment (Recreate, uid 1000, env `RI_REDIS_HOST=redis`, `RI_REDIS_PORT=6379`, `RI_REDIS_ALIAS`, `RI_REDIS_PASSWORD`, `RI_ENCRYPTION_KEY` from Secret, probe `/api/health/` on 5540), Service :80->5540, Ingress with `haproxy.org/auth-type: basic-auth`, `haproxy.org/auth-secret`, `haproxy.org/auth-realm`, `haproxy.org/timeout-server: 60s`.
- [ ] **Step 5: secret.example.yaml + README.md** (both Secrets, how to generate the crypt hash with `openssl passwd -6`, consumer connection string).
- [ ] **Step 6: Render** `helm template redis Both/Redis` -> exit 0.
- [ ] **Step 7: git init, commit** `feat: Redis + RedisInsight chart for rdb.tidangames.com`.

### Task 3: Secrets manager chart (`Both/SecretsManager`)

**Files:**
- Create: `Both/SecretsManager/Chart.yaml` (dependency `openbao` 0.29.5, `repository: ""`), `charts/openbao-0.29.5.tgz` (from `helm pull openbao/openbao --version 0.29.5`), `values.yaml`, `secret.example.yaml`, `README.md`, `.helmignore`

**Interfaces:**
- Produces: StatefulSet `openbao`, Services `openbao`, `openbao-internal`, `openbao-ui`; Ingress `openbao` on sm.tidangames.com; PVC `data-openbao-0`.

- [ ] **Step 1: Chart.yaml + vendored tgz.**
- [ ] **Step 2: values.yaml** under `openbao:`: `global.tlsDisable: true`, `injector.enabled: false`, `csi.enabled: false`, `ui.enabled: true`, `server.image.tag: 2.6.2`, `server.updateStrategyType: RollingUpdate`, `server.dataStorage.size: 10Gi`, `server.extraSecretEnvironmentVars: [{envName: BAO_SEAL_KEY, secretName: openbao-unseal, secretKey: key}]`, `server.standalone.enabled: true` with HCL (`ui = true`, tcp listener `tls_disable = 1`, `storage "raft" { path = "/openbao/data" }`, `seal "static" { current_key_id = "unseal-2026-09-17"  current_key = "env://BAO_SEAL_KEY" }`), `server.ingress` for sm.tidangames.com with cert-manager annotation and TLS block, resources.
- [ ] **Step 3: secret.example.yaml + README.md** (unseal Secret, first `bao operator init`, where the root token/recovery keys go, raft snapshot command, key rotation with previous_key, upgrade = new tgz + bump).
- [ ] **Step 4: Render** `helm template secrets-manager Both/SecretsManager` -> exit 0, exactly one StatefulSet, no injector Deployment, Ingress host sm.tidangames.com, ConfigMap contains `seal "static"`.
- [ ] **Step 5: git init, commit** `feat: OpenBao secrets manager chart for sm.tidangames.com`.

### Task 4: ArgoCD Applications

**Files:**
- Create: `Both/Arcana-Argocd-Apps/prod/postgres.yaml`, `prod/redis.yaml`, `prod/secrets-manager.yaml`

- [ ] **Step 1:** Each file: `Application` in namespace `argocd`, finalizer, `project: default`, destination namespace per service, `source.repoURL: git@github.com:jojobobby/tidan-<x>.git`, `targetRevision: main`, `path: .`, `helm.releaseName` + `valueFiles: [values.yaml]`, syncPolicy automated prune+selfHeal, syncOptions `CreateNamespace=true`, `ApplyOutOfSyncOnly=true`, `ServerSideApply=true`, retry (5, 10s, x2, 5m), `revisionHistoryLimit: 5`, a comment naming the required out-of-band Secret(s).
- [ ] **Step 2:** `kubectl apply --dry-run=server -f` each file -> valid.
- [ ] **Step 3:** Commit only these three files (`git add prod/postgres.yaml prod/redis.yaml prod/secrets-manager.yaml`), message `Register postgres, redis and secrets-manager Applications (tidangames.com data services)`. Do NOT push until Task 5 has pushed the chart repos.

### Task 5: GitHub repos, Secrets, deploy, verify

- [ ] **Step 1:** `gh repo create jojobobby/tidan-postgres --private --source Both/Postgres --remote origin --push` (same for redis, secrets-manager). Confirm `gh repo view` shows `main`.
- [ ] **Step 2:** Create namespaces and Secrets with kubectl (context `rrobinson-services`): generate with `openssl rand`; record plaintext values for the final message only.
- [ ] **Step 3:** Push `Arcana-Argocd-Apps` main. Wait for Applications `postgres`, `redis`, `secrets-manager` to appear and reach Synced.
- [ ] **Step 4:** Watch pods: `kubectl -n postgres get pods`, `-n redis`, `-n secrets-manager`. Expected: `postgres-1` Running, `pgadmin-*` Running, `redis-0` Running, `redisinsight-*` Running, `openbao-0` Running (0/1 until init).
- [ ] **Step 5:** `kubectl -n secrets-manager exec openbao-0 -- bao operator init -format=json` once; capture root token + recovery keys for the user. Confirm `bao status` shows `Sealed: false`, `Initialized: true`.
- [ ] **Step 6:** HTTPS checks: `curl -sI https://pdb.tidangames.com` -> 200/302 with a valid cert; `https://rdb.tidangames.com` -> 401, with `-u user:pass` -> 200; `https://sm.tidangames.com` -> pending until the user adds DNS (report).
- [ ] **Step 7:** Functional: `redis-cli` ping through a port-forward with the password; `psql` via the CNPG `postgres-app` uri from a throwaway pod; pgAdmin login page loads.

### Task 6: Docs and memory

- [ ] **Step 1:** Add the three services to `Both/Argocd/README.md`'s table (they are external repos; add a short "Related repos" section) and commit the spec + plan in `Both/Argocd` (`docs: tidangames.com data services design + plan`), push.
- [ ] **Step 2:** Write memory `tidangames-data-services.md` (repos, hosts, secrets, init state, follow-ups) and index it in MEMORY.md.

## Self-review

- Spec coverage: Postgres (T1), Redis (T2), OpenBao incl. static seal (T3), Applications (T4), Secrets/deploy/verify (T5), docs (T6). DNS for `sm` is a user action, reported in T5.
- No placeholders; every file has concrete content in the task or in the spec.
- Names consistent: Secrets and hosts match the Global Constraints table.
