# Deploying PostHog to Coolify (behind existing Traefik)

This guide covers deploying PostHog on a Coolify host that already runs Traefik
(Coolify's bundled proxy), using [`docker-compose.coolify.yml`](docker-compose.coolify.yml).
Target host: **https://e.avertxai.com**.

This stack is **derived from** PostHog's `docker-compose.hobby.yml` (+
`docker-compose.base.yml`) with the bundled Caddy reverse proxy removed and
ingress moved to Traefik. See the header of `docker-compose.coolify.yml` for the
full change list.

> ⚠️ **Review before you deploy.** This file has not been run end-to-end. Treat
> the "Known issues / things to verify" section as a pre-flight checklist.

---

## 1. Architecture in one paragraph

The PostHog web app (Django) listens on container port **8000**. Several
endpoints are *not* served by the web app — they go to dedicated services that
the old Caddy proxy used to split traffic to. We reproduce that split with
multiple Traefik routers (one per service), all on the same host
`e.avertxai.com`, with the bare-host router as a low-priority catch-all to
`web:8000`:

| Path(s) | Service | Container port |
|---|---|---|
| `/e`, `/i/v0`, `/batch`, `/capture` (+ subpaths) | `capture` | 3000 |
| `/s` (+ subpaths) | `replay-capture` | 3000 |
| `/flags` (+ subpaths) | `feature-flags` | 3001 |
| `/surveys`, `/api/surveys`, `/array/*` | `hypercache-server` | 3002 |
| `/livestream/*` (prefix stripped) | `livestream` | 8080 |
| `/public/webhooks`, `/public/m/*` | `plugins` | 6738 |
| everything else (catch-all) | `web` | 8000 |

> Two Caddy routes are intentionally **not** reproduced: `/i/v0/ai` (capture-ai)
> and `/i/v1/logs|traces|metrics` (capture-logs). Those services are not part of
> the hobby stack, so neither AI-event ingestion over HTTP nor the OTLP log/trace
> intake is available here. The kafka-consuming `ingestion-logs` /
> `ingestion-traces` services *are* present.

---

## 2. Prerequisites

1. **Coolify** with its Traefik proxy enabled and reachable on 80/443.
2. **DNS**: an `A`/`AAAA` record for `e.avertxai.com` → the Coolify host's public IP.
3. **Git access**: Coolify must be able to clone this repository/branch. Deploy
   as a **Docker Compose** resource backed by this git repo — *not* by pasting
   the file. The compose reads repo files via bind mounts
   (`./docker/...`, `./posthog/...`, `./compose`, `./share`), so the working
   copy must be present on the host.
4. **GeoIP database**: place `GeoLite2-City.mmdb` at `./share/GeoLite2-City.mmdb`
   on the host before first boot. It is **gitignored** (licensed binary) so it is
   not in the repo. `feature-flags` and `cymbal` mount it. Fetch it the way
   `bin/deploy-hobby` does, or from MaxMind. Without it those two services log
   errors and geo-enrichment is disabled.
5. **Memory**: PostHog's full stack wants **8 GB+** RAM. Below that, ClickHouse /
   Kafka / Elasticsearch will thrash or OOM.

---

## 3. Deploy steps

1. In Coolify, create a new **Docker Compose** resource pointing at this repo,
   branch `phase0/coolify-deploy-prep`, compose file `docker-compose.coolify.yml`.
2. Set **environment variables** (next section). At minimum the two REQUIRED
   secrets must be set or the stack will refuse to start.
3. Ensure `./share/GeoLite2-City.mmdb` is present on the host (prerequisite 4).
4. Confirm the Traefik entrypoint + cert-resolver names match your Coolify Traefik
   (see Known issues — this is the #1 cause of "deployed but 404/redirect loop").
5. Deploy. First boot takes **5–10 minutes** (image pulls + DB migrations +
   Kafka/ClickHouse/Elasticsearch settling). Don't panic if `web` restarts a few
   times while datastores come up.
6. Verify health (section 6).

> This stack uses **prebuilt images** only (the Rust `build:` contexts from the
> hobby file were removed). Do not enable "build" in Coolify for this resource.

---

## 4. Required environment variables

Full annotated list with examples: [`.env.coolify.example`](.env.coolify.example).
Summary:

| Variable | Required? | Default | Purpose |
|---|---|---|---|
| `POSTHOG_SECRET_KEY` | **yes** | — | Django `SECRET_KEY`. Set once; never rotate live. |
| `POSTHOG_ENCRYPTION_SALT_KEYS` | **yes** | — | 16-byte hex. Encrypts stored secrets. Never rotate live. |
| `DOMAIN` | no | `e.avertxai.com` | Host in every Traefik rule. |
| `SITE_URL` | no | `https://e.avertxai.com` | Public origin (app, object-storage URLs, livestream, Temporal UI CORS). |
| `TRAEFIK_ENTRYPOINT` | no | `websecure` | **Verify** — Coolify Traefik often uses `https`. |
| `TRAEFIK_CERTRESOLVER` | no | `letsencrypt` | Must match your Traefik ACME resolver name. |
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | no | `posthog` / `posthog` / `posthog` | Interpolated into all `postgres://` URLs. |
| `OBJECT_STORAGE_ACCESS_KEY_ID` / `OBJECT_STORAGE_SECRET_ACCESS_KEY` | no | `object_storage_root_user` / `object_storage_root_password` | MinIO root creds (app + MinIO must match). |
| `SESSION_RECORDING_S3_ACCESS_KEY_ID` / `_SECRET_ACCESS_KEY` | no | `any` / `any` | SeaweedFS S3 placeholders (unauthenticated). |
| `OPT_OUT_CAPTURE` | no | `false` | Disable anonymous self-telemetry. |
| `POSTHOG_REGISTRY` | no | `posthog/posthog` | Registry/org for app + node images. |
| `POSTHOG_IMAGE_TAG` | no | `latest` | Tag for `posthog/posthog`. **Pin to a SHA for prod.** |
| `POSTHOG_NODE_TAG` | no | `latest` | Tag for `posthog/posthog-node`. **Pin for prod.** |
| `POSTHOG_RUST_TAG` | no | `master` | Tag for `ghcr.io/posthog/posthog/*`. **Pin for prod.** |
| `CLICKHOUSE_USER` / `CLICKHOUSE_PASSWORD` | n/a | `default` / *(empty)* | Listed but **not wired** — CH auth is in `docker/clickhouse/users.xml`. |

---

## 5. Services & startup order

33 services. Startup is governed by `depends_on` (with health conditions where
they existed in the hobby file):

```
db, redis7, zookeeper
   └─ kafka (needs zookeeper) ─ healthy
        └─ kafka-init (creates topics)
   └─ clickhouse (needs kafka, zookeeper)
objectstorage, seaweedfs
elasticsearch ─ healthy
   └─ temporal (needs db healthy + elasticsearch healthy)
personhog-replica (needs db healthy)
   └─ personhog-router
web (needs db, redis7, clickhouse, kafka, objectstorage, seaweedfs, personhog-router)
   │   runs ./bin/migrate via /compose/start  ← migrations happen here (slow on first boot)
   └─ worker (needs web started + db/redis/kafka healthy)
plugins, ingestion-general, ingestion-sessionreplay, recording-api,
ingestion-error-tracking, ingestion-logs, ingestion-traces,
capture, replay-capture, property-defs-rs, feature-flags, hypercache-server,
cyclotron-janitor, livestream, cymbal, temporal-django-worker,
asyncmigrationscheck, temporal-admin-tools, temporal-ui
```

**Full service list:** db, redis7, clickhouse, zookeeper, kafka, kafka-init,
objectstorage, seaweedfs, web, worker, asyncmigrationscheck,
temporal-django-worker, plugins, ingestion-general, ingestion-sessionreplay,
recording-api, ingestion-error-tracking, ingestion-logs, ingestion-traces,
capture, replay-capture, property-defs-rs, feature-flags, hypercache-server,
cyclotron-janitor, personhog-replica, personhog-router, livestream, cymbal,
temporal, elasticsearch, temporal-admin-tools, temporal-ui.

**Timing:** expect 5–10 minutes to first healthy `web`. Image pulls dominate the
first deploy; migrations add a few minutes.

---

## 6. Verifying the deployment is healthy

1. **Container health** — in Coolify (or `docker compose ps`), `db`, `redis7`,
   `kafka`, `clickhouse`, `feature-flags`, `hypercache-server`, `seaweedfs`,
   `temporal`, `elasticsearch`, and `web` should reach `healthy`.
2. **App health endpoint** — `curl -fsS https://e.avertxai.com/_health` returns
   `200`. (The `web` container also self-checks this every 15s.)
3. **Web UI** — `https://e.avertxai.com` loads the PostHog login/signup page over
   HTTPS with a valid Let's Encrypt cert.
4. **Event capture path** — `curl -i https://e.avertxai.com/e/?ip=0` should hit
   the `capture` service (a 200/empty-ish capture response), *not* a Django 404.
   If it returns a Django page, your Traefik path routing isn't taking effect.
5. **Flags** — `curl -i https://e.avertxai.com/flags` should reach
   `feature-flags`, not `web`.
6. Create the first user/org in the UI and send a test event from a project.

---

## 7. Known issues / things to verify

- **Traefik entrypoint name (most common breakage).** The spec'd labels use
  `websecure`/`tls.certresolver=letsencrypt`. Coolify's bundled Traefik commonly
  names its HTTPS entrypoint `https` and may use a differently-named resolver.
  If the site 404s or never gets a cert, set `TRAEFIK_ENTRYPOINT=https` (and the
  correct `TRAEFIK_CERTRESOLVER`) and redeploy. The labels read these env vars.
- **Traefik network.** `web` (and the other labelled services) must be attached
  to the Docker network Traefik watches. Coolify normally wires this up when it
  sees `traefik.enable=true`; if routing never works, confirm the containers are
  on Coolify's proxy network.
- **Moving image tags.** `latest`/`master` are not reproducible. Pin
  `POSTHOG_IMAGE_TAG` / `POSTHOG_NODE_TAG` / `POSTHOG_RUST_TAG` to specific
  commit SHAs before production. PostHog does not publish semver tags for these.
- **GeoLite2 missing.** If `./share/GeoLite2-City.mmdb` is absent, `feature-flags`
  and `cymbal` error on startup / disable geo features.
- **Config bind mounts require a git deploy.** `clickhouse` configs, `idl`,
  `compose/` scripts, livestream/temporal config, and `share/` are bind-mounted
  from the repo. Pasting the compose body alone (no repo) will fail.
- **`compose/` scripts.** `web` runs `/compose/start` and
  `temporal-django-worker` runs `/compose/temporal-django-worker`, bind-mounted
  from `./compose`. These are committed here and must stay executable on the host
  (the git index marks them `+x`).
- **Host ports removed.** No service publishes host ports anymore (MinIO console
  19001, Temporal UI 8081, Temporal 7233, MinIO 19000/19001 are internal-only).
  To reach the Temporal UI or MinIO console, add a dedicated Traefik router
  later, or temporarily re-add a port mapping.
- **ClickHouse credentials.** Auth is governed by `docker/clickhouse/users.xml`
  (`CLICKHOUSE_SKIP_USER_SETUP=1`); the `CLICKHOUSE_USER`/`CLICKHOUSE_PASSWORD`
  env vars are documented but not yet wired. Changing CH auth means editing
  `users.xml` and the consumers.
- **Elasticsearch is started but unused.** Temporal runs with `ENABLE_ES=false`
  (SQL visibility) yet still `depends_on` Elasticsearch (inherited from upstream).
  ES can be dropped in a later cleanup to save ~512 MB RAM; left in for fidelity.
- **First-boot only init.** The Postgres init scripts (auxiliary DB creation) run
  only against an empty data volume. If you change `POSTGRES_USER`/`DB` after the
  first boot, wipe the `postgres-data` volume or create the DBs manually.

### Debug tips

- App not coming up: `docker compose logs web` — look for migration errors or
  "waiting for ClickHouse/Postgres".
- 502/404 at the domain: check Traefik dashboard/logs for the `posthog*` routers;
  re-check `TRAEFIK_ENTRYPOINT`.
- DB connection refused: `docker compose logs db clickhouse`.
- Out of memory: `free -h`, `docker stats` — Kafka/ClickHouse/ES are the hogs.
