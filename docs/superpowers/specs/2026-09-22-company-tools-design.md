# Company tools: wiki, tasks, support desk — design

Date: 2026-09-22 · Status: approved in chat

## Goal

Self-host free, open-source replacements for Atlassian on the rrobinson-services k3s
cluster. Atlassian is not an option: Data Center stopped selling to new customers on
2026-03-30 and goes read-only 2029-03-28; the free tier is cloud-only.

| Replaces | App | Host | Repo / clone | Namespace |
|---|---|---|---|---|
| Confluence | Docmost 0.96.0 | `wiki.company.tidangames.com` | `jojobobby/tidan-wiki` → `Both/Wiki` | `wiki` |
| Jira | Plane CE v1.4.1 (chart `plane-ce` 1.8.1) | `tasks.company.tidangames.com` | `jojobobby/tidan-tasks` → `Both/Tasks` | `tasks` |
| Jira Service Management | Zammad 7.1 (chart `zammad` 19.0.1) | `support.tidangames.com` | `jojobobby/tidan-support` → `Both/Support` | `support` |

Each follows the `deploying-tidan-cluster-service` pattern: private repo, chart at the
root, ArgoCD Application in `Arcana-Argocd-Apps/prod/`, haproxy ingress, cert-manager
`letsencrypt-prod` certificate, secrets created out of band and never committed.

## Shared data services

- **Postgres:** the existing CNPG cluster `postgres` (namespace `postgres`) gains three
  managed roles and three `Database` objects — `docmost`, `plane`, `zammad`, each owned by
  its role — committed to `tidan-postgres`. Role passwords come from `basic-auth` Secrets
  in `postgres` (`pg-role-<app>`); the same password is copied into the app namespace.
  The monthly `pg_dumpall` backup therefore covers all three from day one. Docmost's
  migration creates `unaccent` and `pg_trgm`; both are trusted extensions, so the database
  owner can create them without superuser.
- **Redis:** the existing `redis.redis.svc:6379` (password `redis-auth`). Separate
  logical databases: Docmost `/1`, Plane `/2`, Zammad `/3`. `maxmemory 1gb noeviction`
  stays; watch usage after launch.

## Per-app shape

- **Docmost:** own small chart — one Deployment (port 3000), a 5Gi PVC for attachments
  at `/app/data/storage` (`STORAGE_DRIVER=local`), Service, Ingress. Env from Secret
  `docmost-env` (`APP_SECRET`, `DATABASE_URL`, `REDIS_URL`, `SMTP_PASSWORD`).
- **Plane:** vendored `plane-ce` subchart. `postgres.local_setup=false`,
  `redis.local_setup=false`; bundled RabbitMQ and MinIO kept (one pod each, 5Gi MinIO).
  All credentials via the chart's `external_secrets.*` hooks: `plane-app-env`,
  `plane-live-env`, `plane-doc-store`, `plane-rabbitmq`. `requireExplicitSecrets: true`
  so the public example keys can never be used. Image pull policy `IfNotPresent`.
- **Zammad:** vendored `zammad` subchart. Bundled Postgres/Redis off (pointed at shared);
  **Elasticsearch off** (needs the ECK operator; Zammad falls back to database search,
  fine at startup volume — can be added later); memcached kept; `volumePermissions` init
  container off (it is privileged and unnecessary: tmp is an emptyDir and `fsGroup` is
  set). Attachments stored in the database (Zammad default), so backups cover them.

## CPU budget

Node has 8 cores, 87% already requested. Every new pod requests ≤ 50m CPU (Postgres/
Redis already exist). About 17 new pods ≈ 0.8 core requested. Limits stay generous; memory
is ample (25% requested).

## Email

- New mailboxes in the `inbox` (docker-mailserver): `support@tidangames.com` (Zammad
  fetches it over IMAPS and replies through it) and `noreply@tidangames.com` (Docmost and
  Plane invites and notifications). Apps submit authenticated to `inbox.inbox.svc:587`,
  so mail leaves DKIM-signed via the relay and passes SPF.
- Zammad mail channel and Plane SMTP are configured in their admin UIs (both store it in
  the database), not in the charts.

## Access and security

- No public signup: Docmost is invite-only by design; Plane's god-mode disables signup;
  Zammad customer self-registration off (players reach support by email).
- First admin of each app is claimed **by the owner in the browser**, right after the
  site comes up (the first visitor becomes admin; an AI must not create accounts).
- Known upstream trait: Plane's MinIO init marks the `uploads` bucket anonymous-download
  (served under `/uploads`, random object keys). Accepted as upstream default.

## Out of scope (follow-ups)

- Backing up Docmost's attachment PVC and Plane's MinIO (add to the `backups` job via the
  exec-into-pod pattern once the apps are live).
- Zammad Elasticsearch, SSO across the three apps, NetworkPolicies.

## Verification

For each app: ArgoCD `Synced/Healthy`; all pods Ready; certificate `Ready=True` with a
Let's Encrypt issuer; the site answers over HTTPS with its setup/login page; a test email
from each app lands in the target mailbox.
