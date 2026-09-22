# Company Tools (Wiki, Tasks, Support) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy Docmost (wiki), Plane CE (tasks) and Zammad (support desk) on the rrobinson-services k3s cluster, GitOps-managed, on the shared Postgres and Redis.

**Architecture:** Three private `jojobobby/tidan-*` Helm-chart repos (Docmost hand-written; Plane and Zammad as umbrella charts vendoring the official charts), three ArgoCD Applications in `Arcana-Argocd-Apps/prod/`, per-app role + database added to the CNPG cluster through `tidan-postgres`, two new mailboxes in the `inbox` mailserver.

**Tech Stack:** k3s 1.35, ArgoCD 2.14, Helm 3.17 (scratchpad `helm/windows-amd64/helm.exe`), CloudNativePG 1.25 (PG 17.11), Redis 8.8, haproxytech ingress 3.0.4, cert-manager `letsencrypt-prod`.

**Spec:** `Argocd/docs/superpowers/specs/2026-09-22-company-tools-design.md`

## Global Constraints

- kube context `rrobinson-services`; every `kubectl` below implies `--context rrobinson-services`.
- Secrets are never committed. Generate with `openssl rand -hex <n>` (URL-safe, needed inside connection URLs). Plaintext lives only in the scratchpad file `company-tools.env`; never print it to the terminal.
- New pods request ≤ 50m CPU (node is 87% CPU-requested).
- Pin every image: Docmost `docmost/docmost:0.96.0`, Plane `v1.4.1` (chart 1.8.1), Zammad chart 19.0.1 (app 7.1.3-0017).
- Hosts: `wiki.company.tidangames.com`, `tasks.company.tidangames.com`, `support.tidangames.com`; TLS secret `<host>-tls`; annotations `cert-manager.io/cluster-issuer: letsencrypt-prod` + `haproxy.org/ssl-redirect: "true"`.
- Commit messages end with the session's `Co-Authored-By:` line. In `Arcana-Argocd-Apps` add files by path only (pre-existing untracked files).
- Shared endpoints: Postgres `postgres-rw.postgres.svc:5432`; Redis `redis.redis.svc:6379` (password in Secret `redis/redis-auth` key `password`), DB indexes Docmost 1, Plane 2, Zammad 3. App mail goes to `mail.tidangames.com:465` (implicit TLS, valid cert) authenticated as `noreply@tidangames.com`.
- No agent creates app user accounts: the owner claims each app's first admin in the browser.

---

### Task 1: Credentials file + shared Postgres roles/databases

**Files:**
- Modify: `Both/Postgres/values.yaml` (add `appRoles`, raise `max_connections` to 200)
- Modify: `Both/Postgres/templates/cluster.yaml` (managed roles)
- Create: `Both/Postgres/templates/databases.yaml`
- Modify: `Both/Postgres/README.md`, `Both/Postgres/secret.example.yaml`

- [ ] **Step 1: Generate every credential once** into `$SCRATCH/company-tools.env` (mode 600):

```bash
f="$SCRATCH/company-tools.env"; umask 077; : > "$f"
for k in PG_DOCMOST PG_PLANE PG_ZAMMAD MINIO_PLANE RMQ_PLANE; do echo "$k=$(openssl rand -hex 20)" >> "$f"; done
for k in DOCMOST_APP_SECRET PLANE_SECRET_KEY PLANE_LIVE_KEY; do echo "$k=$(openssl rand -hex 32)" >> "$f"; done
for k in MAIL_SUPPORT MAIL_NOREPLY; do echo "$k=$(openssl rand -base64 18 | tr -d '/+=')" >> "$f"; done
```

- [ ] **Step 2: Role Secrets in `postgres`** (CNPG reads `basic-auth` Secrets; label enables reload):

```bash
. "$SCRATCH/company-tools.env"
for app in docmost plane zammad; do
  var="PG_$(echo $app | tr a-z A-Z)"
  kubectl -n postgres create secret generic pg-role-$app --type=kubernetes.io/basic-auth \
    --from-literal=username=$app --from-literal=password="${!var}"
  kubectl -n postgres label secret pg-role-$app cnpg.io/reload=true
done
```

- [ ] **Step 3: Chart changes.** `values.yaml` under `postgres:` add `appRoles: [docmost, plane, zammad]` and set `max_connections: "200"`. In `cluster.yaml` after `enableSuperuserAccess: true` add:

```yaml
  {{- with .Values.postgres.appRoles }}
  # One login role per consuming app; password from Secret pg-role-<app> (basic-auth,
  # created out of band - see secret.example.yaml).
  managed:
    roles:
      {{- range . }}
      - name: {{ . }}
        ensure: present
        login: true
        passwordSecret:
          name: pg-role-{{ . }}
      {{- end }}
  {{- end }}
```

`templates/databases.yaml`:

```yaml
# One database per app role, owned by that role (CloudNativePG `Database` objects).
{{- range .Values.postgres.appRoles }}
---
apiVersion: postgresql.cnpg.io/v1
kind: Database
metadata:
  name: {{ $.Values.postgres.name }}-{{ . }}
spec:
  name: {{ . }}
  owner: {{ . }}
  cluster:
    name: {{ $.Values.postgres.name }}
{{- end }}
```

- [ ] **Step 4: Render** `helm template postgres .` → exit 0; output contains `managed:` with 3 roles and 3 `kind: Database`.
- [ ] **Step 5: Commit + push** (`feat: app roles + databases for docmost, plane, zammad`), refresh ArgoCD app `postgres`.
- [ ] **Step 6: Verify:** `kubectl -n postgres get database` → 3 rows `APPLIED=true`; `kubectl -n postgres get cluster postgres -o jsonpath='{.status.managedRolesStatus.byStatus}'` lists the 3 roles under `reconciled`.

### Task 2: Mailboxes support@ and noreply@

**Files:** none in git (Secret `inbox/mailbox-accounts` is out of band); update `Both/Inbox/README.md` mailbox list.

- [ ] **Step 1: Append two accounts** to the existing Secret and restart:

```bash
. "$SCRATCH/company-tools.env"; a="$SCRATCH/accounts.cf"
kubectl -n inbox get secret mailbox-accounts -o jsonpath='{.data.postfix-accounts\.cf}' | base64 -d > "$a"
printf 'support@tidangames.com|{SHA512-CRYPT}%s\n' "$(openssl passwd -6 "$MAIL_SUPPORT")" >> "$a"
printf 'noreply@tidangames.com|{SHA512-CRYPT}%s\n' "$(openssl passwd -6 "$MAIL_NOREPLY")" >> "$a"
kubectl -n inbox create secret generic mailbox-accounts --from-file=postfix-accounts.cf="$a" --dry-run=client -o yaml | kubectl apply -f -
rm -f "$a"; kubectl -n inbox rollout restart deploy/inbox && kubectl -n inbox rollout status deploy/inbox
```

- [ ] **Step 2: Verify** `kubectl -n inbox exec deploy/inbox -- doveadm user '*'` lists backups@, support@, noreply@.
- [ ] **Step 3: Verify pods can reach the public mail host** (hairpin):
`kubectl run mailcheck --rm -i --restart=Never --image=alpine:3.21 -- sh -c 'apk add -q openssl && echo | openssl s_client -connect mail.tidangames.com:465 -servername mail.tidangames.com 2>/dev/null | grep -E "Verify return code"'` → `0 (ok)`. If it fails, apps use `inbox.inbox.svc:587` instead and the spec's mail row is revised.

### Task 3: Docmost (Both/Wiki → jojobobby/tidan-wiki)

**Files:** Create `Both/Wiki/{Chart.yaml,values.yaml,templates/docmost.yaml,README.md,secret.example.yaml}`; `Both/Arcana-Argocd-Apps/prod/wiki.yaml`.

- [ ] **Step 1: Write the chart.** `values.yaml`:

```yaml
image: { repository: docmost/docmost, tag: "0.96.0", pullPolicy: IfNotPresent }
host: wiki.company.tidangames.com
tlsSecret: wiki.company.tidangames.com-tls
clusterIssuer: letsencrypt-prod
# Out-of-band Secret: APP_SECRET, DATABASE_URL, REDIS_URL, SMTP_PASSWORD (secret.example.yaml)
existingSecret: docmost-env
storage: 5Gi
uploadLimit: 50mb
mail: { host: mail.tidangames.com, port: 465, secure: "true", user: noreply@tidangames.com, from: noreply@tidangames.com, fromName: Tidan Wiki }
resources: { requests: { cpu: 50m, memory: 256Mi }, limits: { cpu: "1", memory: 1Gi } }
```

`templates/docmost.yaml`: PVC `docmost-data` (RWO, `.Values.storage`); Deployment `docmost` (strategy `Recreate`, `fsGroup: 1000`, container port 3000 named `http`, env `APP_URL=https://{{host}}`, `PORT=3000`, `STORAGE_DRIVER=local`, `FILE_UPLOAD_SIZE_LIMIT`, `MAIL_DRIVER=smtp`, `SMTP_HOST/PORT/SECURE/USERNAME`, `MAIL_FROM_ADDRESS/NAME`, `DISABLE_TELEMETRY=true`, `envFrom` secret `existingSecret`, volume at `/app/data/storage`, readiness+liveness `tcpSocket: http`); Service `docmost` 80→http; Ingress as in Global Constraints.

- [ ] **Step 2: Render** `helm template wiki .` → exit 0, kinds PVC, Deployment, Service, Ingress.
- [ ] **Step 3: Secret** in namespace `wiki` (build URLs without printing):

```bash
. "$SCRATCH/company-tools.env"; kubectl create namespace wiki
RP=$(kubectl -n redis get secret redis-auth -o jsonpath='{.data.password}' | base64 -d | python -c "import sys,urllib.parse;print(urllib.parse.quote(sys.stdin.read(),safe=''))")
kubectl -n wiki create secret generic docmost-env \
  --from-literal=APP_SECRET="$DOCMOST_APP_SECRET" \
  --from-literal=DATABASE_URL="postgresql://docmost:${PG_DOCMOST}@postgres-rw.postgres.svc:5432/docmost?schema=public" \
  --from-literal=REDIS_URL="redis://:${RP}@redis.redis.svc:6379/1" \
  --from-literal=SMTP_PASSWORD="$MAIL_NOREPLY"
```

- [ ] **Step 4: Repo + Application.** `gh repo create jojobobby/tidan-wiki --private --source . --push`; write `prod/wiki.yaml` (copy of `webmail.yaml` with name/namespace `wiki`, repo `tidan-wiki`, releaseName `wiki`); commit by path; refresh `app-of-apps`.
- [ ] **Step 5: Verify:** pod Ready; logs show migrations complete and `listening on 3000`; `kubectl -n wiki port-forward svc/docmost 8081:80` + `curl -s -o /dev/null -w '%{http_code}' localhost:8081/` → 200.

### Task 4: Plane (Both/Tasks → jojobobby/tidan-tasks)

**Files:** Create `Both/Tasks/{Chart.yaml,values.yaml,charts/plane-ce-1.8.1.tgz,README.md,secret.example.yaml}`; `prod/tasks.yaml` (releaseName **`plane`**, namespace `tasks`).

- [ ] **Step 1: Umbrella chart.** `Chart.yaml` dependency `plane-ce` 1.8.1, `repository: ""` (vendored, like SecretsManager). `values.yaml`:

```yaml
plane-ce:
  planeVersion: v1.4.1
  ingress:
    enabled: true
    appHost: tasks.company.tidangames.com
    ingressClass: haproxy
    ingress_annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      haproxy.org/ssl-redirect: "true"
  ssl: { tls_secret_name: tasks.company.tidangames.com-tls }
  redis: { local_setup: false }
  postgres: { local_setup: false }
  rabbitmq: { local_setup: true, volumeSize: 1Gi }
  minio: { local_setup: true, volumeSize: 5Gi }
  web:        { pullPolicy: IfNotPresent, cpuRequest: 25m, memoryRequest: 128Mi }
  space:      { pullPolicy: IfNotPresent, cpuRequest: 25m, memoryRequest: 128Mi }
  admin:      { pullPolicy: IfNotPresent, cpuRequest: 25m, memoryRequest: 128Mi }
  live:       { pullPolicy: IfNotPresent, cpuRequest: 25m, memoryRequest: 128Mi }
  api:        { pullPolicy: IfNotPresent, cpuRequest: 50m, memoryRequest: 256Mi }
  worker:     { pullPolicy: IfNotPresent, cpuRequest: 25m, memoryRequest: 256Mi }
  beatworker: { pullPolicy: IfNotPresent, cpuRequest: 25m, memoryRequest: 128Mi }
  external_secrets:
    app_env_existingSecret: plane-app-env
    live_env_existingSecret: plane-live-env
    doc_store_existingSecret: plane-doc-store
    rabbitmq_existingSecret: plane-rabbitmq
  env:
    requireExplicitSecrets: true
    docstore_bucket: uploads
    doc_upload_size_limit: "52428800"
```

- [ ] **Step 2: Render** → exit 0; no `kind: StatefulSet` named `plane-pgdb` or `plane-redis`; Ingress host is tasks.company…; every Deployment `envFrom` names the four Secrets above.
- [ ] **Step 3: Secrets** in namespace `tasks`:

```bash
. "$SCRATCH/company-tools.env"; kubectl create namespace tasks; RP=<url-encoded redis pw as Task 3>
R="redis://:${RP}@redis.redis.svc:6379/2"
kubectl -n tasks create secret generic plane-app-env \
  --from-literal=SECRET_KEY="$PLANE_SECRET_KEY" --from-literal=LIVE_SERVER_SECRET_KEY="$PLANE_LIVE_KEY" \
  --from-literal=REDIS_URL="$R" \
  --from-literal=DATABASE_URL="postgresql://plane:${PG_PLANE}@postgres-rw.postgres.svc:5432/plane" \
  --from-literal=AMQP_URL="amqp://plane:${RMQ_PLANE}@plane-rabbitmq.tasks.svc.cluster.local/" \
  --from-literal=EMAIL_HOST=mail.tidangames.com --from-literal=EMAIL_PORT=465 --from-literal=EMAIL_USE_SSL=1 --from-literal=EMAIL_USE_TLS=0 \
  --from-literal=EMAIL_HOST_USER=noreply@tidangames.com --from-literal=EMAIL_HOST_PASSWORD="$MAIL_NOREPLY" \
  --from-literal=EMAIL_FROM="Tidan Tasks <noreply@tidangames.com>" --from-literal=ENABLE_SIGNUP=0
kubectl -n tasks create secret generic plane-live-env --from-literal=LIVE_SERVER_SECRET_KEY="$PLANE_LIVE_KEY" --from-literal=REDIS_URL="$R"
kubectl -n tasks create secret generic plane-rabbitmq --from-literal=RABBITMQ_DEFAULT_USER=plane --from-literal=RABBITMQ_DEFAULT_PASS="$RMQ_PLANE"
kubectl -n tasks create secret generic plane-doc-store --from-literal=FILE_SIZE_LIMIT=52428800 --from-literal=AWS_S3_BUCKET_NAME=uploads \
  --from-literal=USE_MINIO=1 --from-literal=MINIO_ROOT_USER=plane --from-literal=MINIO_ROOT_PASSWORD="$MINIO_PLANE" \
  --from-literal=AWS_ACCESS_KEY_ID=plane --from-literal=AWS_SECRET_ACCESS_KEY="$MINIO_PLANE" \
  --from-literal=AWS_S3_ENDPOINT_URL=http://plane-minio:9000
```

- [ ] **Step 4: Repo + Application** (`tidan-tasks`, `prod/tasks.yaml`, `helm.releaseName: plane`).
- [ ] **Step 5: Verify:** migrator Job `Complete`; all pods Ready; api logs free of DB/Redis/AMQP errors; port-forward `svc/plane-web` → 200.

### Task 5: Zammad (Both/Support → jojobobby/tidan-support)

**Files:** Create `Both/Support/{Chart.yaml,values.yaml,charts/zammad-19.0.1.tgz,README.md,secret.example.yaml}`; `prod/support.yaml` (releaseName `zammad`, namespace `support`).

- [ ] **Step 1: Umbrella chart**, dependency `zammad` 19.0.1 vendored. `values.yaml`:

```yaml
zammad:
  ingress:
    enabled: true
    className: haproxy
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      haproxy.org/ssl-redirect: "true"
    hosts:
      - host: support.tidangames.com
        paths: [{ path: /, pathType: Prefix }]
    tls:
      - secretName: support.tidangames.com-tls
        hosts: [support.tidangames.com]
  secrets:
    postgresql: { useExisting: true, secretName: zammad-db, secretKey: postgresql-pass }
    redis: { useExisting: true, secretName: zammad-redis, secretKey: redis-password }
  zammadConfig:
    elasticsearch: { enabled: false }
    postgresql: { enabled: false, host: postgres-rw.postgres.svc, port: 5432, db: zammad, user: zammad, options: "pool=10" }
    # port carries the DB index: REDIS_URL renders as redis://:<pw>@host:<port>
    redis: { enabled: false, host: redis.redis.svc, port: "6379/3" }
    memcached: { enabled: true }
    nginx:       { resources: { requests: { cpu: 25m, memory: 64Mi },  limits: { memory: 256Mi } } }
    railsserver: { resources: { requests: { cpu: 50m, memory: 512Mi }, limits: { memory: 1536Mi } } }
    scheduler:   { resources: { requests: { cpu: 25m, memory: 384Mi }, limits: { memory: 1Gi } } }
    websocket:   { resources: { requests: { cpu: 25m, memory: 256Mi }, limits: { memory: 768Mi } } }
    initContainers:
      volumePermissions: { enabled: false }
  memcached:
    resources: { requests: { cpu: 10m, memory: 64Mi }, limits: { memory: 128Mi } }
```

- [ ] **Step 2: Render** → exit 0; no Elasticsearch/postgres/redis/rustfs objects; no container with `privileged: true`; `REDIS_URL` ends `@redis.redis.svc:6379/3`.
- [ ] **Step 3: Secrets** in `support`: `zammad-db` (`postgresql-pass=$PG_ZAMMAD`), `zammad-redis` (`redis-password=<raw redis pw>`, raw because the chart substitutes it into the URL — if it contains URL-special characters, use the encoded form).
- [ ] **Step 4: Repo + Application** (`tidan-support`, `prod/support.yaml`).
- [ ] **Step 5: Verify:** init Job `Complete`; railsserver, scheduler, websocket, nginx, memcached Ready; port-forward `svc/zammad-nginx` → 200.

### Task 6: DNS, certificates, owner handoff, docs

- [ ] **Step 1: Owner adds DNS** (Route 53, `tidangames.com`): A `wiki.company`, `tasks.company`, `support` → `147.135.8.100`.
- [ ] **Step 2: Certificates** `Ready=True` in `wiki`, `tasks`, `support` (delete stale orders if they predate DNS); `openssl s_client` issuer is Let's Encrypt.
- [ ] **Step 3: Owner claims admins**, in order: `https://wiki.company.tidangames.com` (setup page), `https://tasks.company.tidangames.com/god-mode` (instance admin; confirm Sign-up off, SMTP test), `https://support.tidangames.com` (setup wizard: system URL, email notification via SMTP `mail.tidangames.com:465` as noreply@, channel IMAP `mail.tidangames.com:993` as support@).
- [ ] **Step 4: Test mail** from each app reaches its mailbox (`doveadm mailbox status`).
- [ ] **Step 5: Docs:** Obsidian `Tidan Games LLC` (Web Logins + Access Index rows for the three apps + mailbox passwords), `administering-tidan-infra/reference.md` login table, Inbox README mailbox list, memory `tidangames-data-services.md`.
