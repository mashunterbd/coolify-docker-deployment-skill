---
name: coolify-docker-compose
description: Create, review, or fix Dockerfile, docker-compose.yaml, .dockerignore and .env.example files so a project deploys on Coolify (self-hosted PaaS) using the "Docker Compose" build pack from a Git repository, without errors. Use this skill whenever the user mentions Coolify, Coolify deployment, "deploy with Coolify", docker compose for Coolify, Dockerfile for a Coolify deploy, "No Available Server", 502/503 behind Coolify, magic variables (SERVICE_URL / SERVICE_FQDN / SERVICE_PASSWORD), or asks to make a repo "deployable" or "Coolify-ready" - even if they do not explicitly say "skill" or "compose".
---

# Coolify Docker Compose Deployment Skill

Goal: produce a `docker-compose.yaml` (plus Dockerfile(s), `.dockerignore`, `.env.example`) that deploys on **Coolify v4** with the **Docker Compose build pack** on the first try: correct domain routing, SSL, environment variables, healthchecks, persistence, and startup order.

Coolify's behavior changes between releases. If the user's instance is old or something below does not behave as described, consult the official docs (`https://coolify.io/docs`, LLM-friendly index at `https://coolify.io/docs/llms.txt`, and append `.md` to a docs page URL for Markdown) and prefer the docs over this file. Items marked **(verify)** are version-sensitive.

---

## 1. Operating procedure (follow in order)

1. **Inspect the repository first.** Identify: languages/frameworks, package manager and lockfile, build command, start command, listening port, health/ready endpoint (if any), required env vars (search for `process.env`, `os.environ`, `.env.example`, config files), database/cache/queue/storage needs, migrations, background workers, file-upload paths, monorepo layout.
2. **Decide the service topology.** One compose service per process (web, api, worker, db, redis, migrate job). Do not put two long-running processes in one container.
3. **Decide the exposure model.** Which services get a public domain? Everything else stays internal (no domain, no `ports:`).
4. **Write the files** using the rules and templates below. Keep it minimal; do not add features the project does not need.
5. **Validate** (section 12). Run `docker compose config` if Docker is available; otherwise self-review against the checklist.
6. **Report** using the output format in section 14, including the exact Coolify UI settings the user must set (domains, env vars, build pack, base directory).

Never invent env vars, ports, or endpoints. If the app has no health endpoint, add a minimal one (or use a TCP/HTTP check on `/`) and say so. If something essential is unknown (port, start command), ask one concise question or state the assumption explicitly.

---

## 2. Mental model: what Coolify does vs. what you must write

### Coolify handles automatically - DO NOT write these
| Concern | What Coolify does | Rule |
|---|---|---|
| Reverse proxy (Traefik/Caddy) | Generates routing labels from the domain set in the UI (or from `SERVICE_URL_*`) | No Traefik/Caddy `labels:` for routing |
| SSL / HTTPS | Issues and renews Let's Encrypt certs for `https://` domains | No cert files, no certbot, no SSL config in the app container |
| Network | Creates an isolated bridge network per resource (named after the resource UUID) | Remove custom `networks:` |
| Container names | Generates unique names | No `container_name:` |
| Volume name prefixing | Prefixes named volumes with the resource UUID | Use short plain names |
| `.env` file | Builds it from the Environment Variables tab and feeds compose interpolation | Reference with `${VAR}` |
| Secrets generation | `SERVICE_PASSWORD_*`, `SERVICE_USER_*`, `SERVICE_BASE64_*` | Use instead of hardcoding secrets |
| Domain variables | `SERVICE_URL_*`, `SERVICE_FQDN_*`, `SERVICE_NAME_*` | Use instead of hardcoding hostnames |
| Build-arg injection | Injects env vars as build args (default on) | Declare `ARG` in the Dockerfile for any build-time variable |
| Webhooks / auto-deploy | Deploy on git push when enabled | Nothing in compose |
| Build cache | `SOURCE_COMMIT` is excluded from the build by default so layers stay cached | Do not rely on `SOURCE_COMMIT` inside the image |

### You must write / ensure (Coolify will not do it for you)
- Correct `build` context and Dockerfile path (relative to the compose file / base directory).
- **Healthchecks** for every long-running service (Coolify's proxy will not route to an unhealthy container).
- `depends_on` with `condition: service_healthy` (or `service_completed_successfully` for one-shot jobs).
- Named **volumes** for every piece of state (DB data, uploads, caches that must survive).
- The app listening on **`0.0.0.0`** and on the port you declare to Coolify.
- `restart: unless-stopped` on long-running services.
- Database migration strategy.
- Proxy-awareness in the app (trust `X-Forwarded-*`, secure cookies, CORS origins, correct public URL).
- `.dockerignore`, lockfile usage, non-root user, reasonable image size.

---

## 3. Hard rules

**MUST**
- Use the file name `docker-compose.yaml` (or `.yml`) at the repo root unless the user states another location. Tell the user the "Docker Compose Location" and "Base Directory" to set in Coolify (the location is combined with the Base Directory).
- Give every long-running service a `healthcheck` and `restart: unless-stopped`.
- Reference every configurable or secret value as `${VAR}` / `${VAR:-default}` / `${VAR:?error message}` so it appears in Coolify's Environment Variables UI. Use `:?` for values the app cannot start without (JWT secrets, API keys).
- Use `SERVICE_PASSWORD_<NAME>` (or `_64`) for generated DB/internal secrets.
- Use `expose:` (or nothing) for internal ports.
- Connect services by **compose service name** (`db`, `redis`), never `localhost`.
- Bind servers to `0.0.0.0`.
- Escape `$` as `$$` inside healthcheck commands and any compose value that must reach the container shell unexpanded.
- Pin image tags (`postgres:16-alpine`, not `latest`).
- Put persistent data in named volumes.

**MUST NOT**
- No `container_name:`.
- No `networks:` block (custom networks conflict with Coolify's managed network).
- No `ports:` for HTTP services behind the Coolify proxy (host port conflicts, proxy bypass). Use `ports:` only for deliberate non-HTTP TCP/UDP exposure (e.g., a game server, SSH, public DB) with a unique host port.
- No Traefik/Caddy routing `labels:`.
- No hardcoded secrets, passwords, API keys, or domains in compose or Dockerfile.
- No `version:` key (obsolete).
- No `env_file: .env` pointing at a file that is not committed (Coolify supplies variables through its UI-managed `.env`).
- No `network_mode: host` unless the user explicitly requires it.
- No bind mounts to absolute host paths (`/data/...`) unless the user explicitly requires it; use named volumes or repo-relative paths.
- No `build.args` containing secrets (build args can leak into image history).
- No variables whose names start with `SERVICE_` unless they are Coolify magic variables.
- Never `COPY .env` into an image.

---

## 4. Environment variables

- `${VAR}` in compose -> appears in the Coolify UI; the user fills the value. A hardcoded `VAR=value` in `environment:` is shown separately and cannot be edited from the UI (edit the compose file instead). Use hardcoding only for non-secret constants (`NODE_ENV=production`).
- `${VAR:-default}` -> default when unset. `${VAR:?message}` -> deployment fails with `message` when unset (use for required secrets).
- **Runtime vs build-time**: Coolify has per-variable "Build Variable" / "Runtime" toggles. Frameworks that inline variables at build time (`NEXT_PUBLIC_*`, `VITE_*`, `PUBLIC_*`, CRA `REACT_APP_*`, Nuxt `NUXT_PUBLIC_*` in some setups) need the value at **build** time: pass through `build.args` AND declare `ARG` + `ENV` in the Dockerfile. Changing such a value requires a rebuild; use "Force deploy (without cache)" if stale.
- Provide a committed `.env.example` listing every variable with safe placeholder values and comments (which are required, which are build-time).
- Predefined variables Coolify may provide **(verify)**: `COOLIFY_URL`, `COOLIFY_FQDN`, `COOLIFY_BRANCH`, `COOLIFY_RESOURCE_UUID`, `COOLIFY_CONTAINER_NAME`, `SOURCE_COMMIT` (runtime only; excluded from build by default). Do not depend on them for core logic.
- Shared variables (team/project/environment scope) can be referenced in the UI as `{{environment.VAR}}`; this is a UI-side feature, not compose syntax.

### Magic variables (Coolify v4, recent betas and later) **(verify minimum version if magic variables do not generate)**
| Variable | Meaning |
|---|---|
| `SERVICE_URL_<SERVICE>` / `SERVICE_URL_<SERVICE>_<PORT>` | Full public URL (with scheme) of that compose service; with `_<PORT>` it also routes the domain to that internal port |
| `SERVICE_FQDN_<SERVICE>` / `SERVICE_FQDN_<SERVICE>_<PORT>` | Hostname only (no scheme) |
| `SERVICE_NAME_<SERVICE>` | Compose service name (auto-created) |
| `SERVICE_PASSWORD_<ID>` / `SERVICE_PASSWORD_64_<ID>` | Generated random password (default length / 64 chars), stable across redeploys |
| `SERVICE_USER_<ID>` | Generated random username |
| `SERVICE_BASE64_<ID>` / `SERVICE_BASE64_64_<ID>` | Generated random base64 value |

Rules: `<SERVICE>` is the compose service name uppercased (hyphens may be used; keep names simple: `app`, `api`, `web`). Declaring a magic variable in a service's `environment:` list (e.g. `- SERVICE_URL_APP_3000`) is what makes Coolify treat the service as domain-enabled. Generated values persist and are visible in the Environment Variables tab. Do not rename a service that is referenced by magic variables without updating them.

---

## 5. Domains, ports, proxy

Choose **one** approach per project and tell the user which:

**Approach A - magic variable (preferred for reproducibility; lives in the repo):**
```yaml
environment:
  - SERVICE_URL_APP_3000        # domain-enabled service, routed to internal port 3000
```
The user then sets/edits the actual domain in the Coolify UI (Domains field of the service).

**Approach B - UI only:** no magic variable; the user enters `https://app.example.com:3000` in the service's Domains field, where `:3000` is the **internal container port** (not a public port). Multiple domains: comma-separated.

Rules:
- Only services that must be public get a domain. DB, Redis, workers: none.
- The internal port must match what the app actually listens on.
- Use `https://` domains so Coolify issues SSL and redirects HTTP.
- Behind the proxy the app sees HTTP and `X-Forwarded-Proto/-For/-Host`. Configure the framework accordingly (Express `app.set('trust proxy', 1)`, Django `SECURE_PROXY_SSL_HEADER` + `CSRF_TRUSTED_ORIGINS`, Rails `config.force_ssl` + proxy headers, Laravel `TrustProxies`, Next.js `X-Forwarded-*` handled by default, etc.).
- Public URL needed by the app (callbacks, OAuth, emails, CORS): take it from a variable (`APP_URL`, `CORS_ORIGIN`) the user sets, or from `${SERVICE_URL_<SERVICE>}` **(verify)**; never hardcode.
- WebSockets generally work through the proxy; make sure the app binds `0.0.0.0` and uses the same port.
- Do not add `ports:` to "make it work". If it only works with `ports:`, the real problem is the bind address, port, or healthcheck.

---

## 6. Healthchecks (mandatory)

Coolify routes traffic only to healthy containers. A failing or missing check is the #1 cause of "No Available Server" / 503.

```yaml
healthcheck:
  test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:3000/health >/dev/null 2>&1 || exit 1"]
  interval: 15s
  timeout: 5s
  retries: 5
  start_period: 30s
```
- The tool used (`wget` / `curl`) must exist **in the image**. `alpine` has `wget` (busybox) but not `curl`; `-slim` Debian images have neither by default; install what you use in the Dockerfile, or use a language-native check (`node -e`, `python -c`).
- Check `127.0.0.1` (inside the container), not the public domain.
- `/health` should return 200 only when the app can serve (optionally verify DB connectivity). Keep it cheap and unauthenticated.
- `start_period` must cover cold start and migrations.
- Database checks: Postgres `pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}`; MySQL/MariaDB `healthcheck.sh --connect --innodb_initialized` or `mysqladmin ping`; Redis `redis-cli ping`; MongoDB `mongosh --eval "db.adminCommand('ping')"`.
- One-shot jobs (migrations) are not healthchecked; see section 7.

---

## 7. Startup order and migrations

```yaml
depends_on:
  db:
    condition: service_healthy
```
Migrations - pick the simplest option that fits:
1. **In the app start command** (`npm run migrate && npm start`, `alembic upgrade head && uvicorn ...`) - simple, fine for a single replica.
2. **One-shot `migrate` service** (preferred for clarity):
```yaml
  migrate:
    build: { context: ., dockerfile: Dockerfile }
    command: ["npm", "run", "migrate"]
    restart: "no"
    exclude_from_hc: true          # Coolify extension: ignore this exited container in health aggregation (verify)
    environment:
      - DATABASE_URL=${DATABASE_URL:?}
    depends_on:
      db:
        condition: service_healthy
  app:
    depends_on:
      migrate:
        condition: service_completed_successfully
```
Never run destructive operations (`migrate reset`, `db push --force-reset`, seed that wipes data) on every deploy.

---

## 8. Volumes and storage

- Declare named volumes at top level and mount them: `pg-data:/var/lib/postgresql/data`.
- Anything users upload or the app writes (images, SQLite file, caches that must persist) needs a volume or external object storage (S3-compatible). Container filesystem is wiped on every redeploy.
- Repo-relative bind mounts (`./config/app.conf:/etc/app.conf:ro`) are acceptable for read-only config shipped in the repo.
- Respect file ownership: if the container runs as a non-root user, ensure the volume path is writable by it (create and `chown` the directory in the Dockerfile before `USER`).
- Postgres/MySQL: the password is written to the data directory at **first init**. Changing the env var later does not change the password; either change it inside the DB or recreate the volume (data loss).
- Back up databases (Coolify scheduled backups work for Coolify-managed DB resources; for DBs inside compose, add a backup strategy or use a managed DB resource).

---

## 9. Networking and databases

- Inside the stack, reach services by name: `postgresql://user:pass@db:5432/app`, `redis://redis:6379`.
- Services in one compose stack share Coolify's managed network automatically.
- Using a separate Coolify-managed Database resource instead of a compose DB: it lives on a different network. Enable "Connect to Predefined Network" on the application and use that resource's internal hostname/UUID and internal URL from its page. Mention this to the user if they choose that path.
- Public DB access: avoid; if required, use `ports:` with a unique host port, strong credentials, and firewall rules, and tell the user it is a deliberate exception.
- Never expose Redis, Postgres, Mongo, Elasticsearch to the internet without authentication and TLS.

---

## 10. Build and Dockerfile rules

- Multi-stage builds; final stage contains only runtime artifacts.
- Copy dependency manifests first, install with the lockfile (`npm ci`, `pnpm install --frozen-lockfile`, `yarn --frozen-lockfile`, `pip install -r requirements.txt` / `uv sync --frozen`, `composer install --no-dev --optimize-autoloader`, `bundle install --deployment`), then copy source. This preserves Docker layer cache.
- `.dockerignore` must exclude: `.git`, `node_modules`, `.env*` (except `.env.example` is fine to include or exclude), `dist`/`build` outputs when rebuilt in image, logs, local DB files, IDE folders, `docker-compose*.yaml`.
- Run as non-root (`USER node`, or create a user in other images).
- `EXPOSE` the real port (documentation only; Coolify routing uses the domain port / `SERVICE_URL_*_<PORT>`).
- Use `ENV NODE_ENV=production` (or equivalent) in the runtime stage.
- Use `exec`-form `CMD ["..."]` so signals reach the process (clean shutdown on redeploy). Use an init (`tini`) or `init: true` in compose if the app spawns children.
- Declare `ARG` for every build-time variable; only `ARG`s that are declared receive injected values.
- Small, pinned base images (`node:22-alpine`, `python:3.12-slim`). Beware: native modules on alpine (musl) may need build tools or a glibc image.
- Small servers can run out of memory during builds (symptom: `Killed`, exit 137): add swap, raise limits, or use a separate build server (a build server cannot run deployed resources).
- Use "Force deploy (without cache)" after changing build-time variables or when stale layers are suspected.
- Coolify's "Include Source Commit in Build" invalidates cache on every commit; leave off unless the app needs it.
- "Raw Compose Deployment" skips many Coolify modifications; do not enable it unless the user explicitly asks and understands that magic variables/proxy automation may not apply.

---

## 11. Templates

### 11.1 Node.js API/web + Postgres (single public service)
```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      - SERVICE_URL_APP_3000
      - NODE_ENV=production
      - PORT=3000
      - HOST=0.0.0.0
      - DATABASE_URL=postgresql://${POSTGRES_USER:-app}:${SERVICE_PASSWORD_POSTGRES}@db:5432/${POSTGRES_DB:-app}
      - JWT_SECRET=${JWT_SECRET:?JWT_SECRET is required}
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:3000/health >/dev/null 2>&1 || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER:-app}
      - POSTGRES_PASSWORD=${SERVICE_PASSWORD_POSTGRES}
      - POSTGRES_DB=${POSTGRES_DB:-app}
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  pg-data:
```

### 11.2 Frontend + backend + Postgres + Redis (two public services)
```yaml
services:
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        VITE_API_URL: ${VITE_API_URL:?Set the public API URL (build-time)}
    restart: unless-stopped
    environment:
      - SERVICE_URL_FRONTEND_80
    depends_on:
      backend:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:80/ >/dev/null 2>&1 || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 10s

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      - SERVICE_URL_BACKEND_4000
      - NODE_ENV=production
      - PORT=4000
      - HOST=0.0.0.0
      - DATABASE_URL=postgresql://${POSTGRES_USER:-app}:${SERVICE_PASSWORD_POSTGRES}@db:5432/${POSTGRES_DB:-app}
      - REDIS_URL=redis://redis:6379
      - CORS_ORIGIN=${CORS_ORIGIN:?Set the frontend public URL}
      - JWT_SECRET=${JWT_SECRET:?JWT_SECRET is required}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:4000/health >/dev/null 2>&1 || exit 1"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 40s

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER:-app}
      - POSTGRES_PASSWORD=${SERVICE_PASSWORD_POSTGRES}
      - POSTGRES_DB=${POSTGRES_DB:-app}
    volumes:
      - pg-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  pg-data:
  redis-data:
```
`VITE_API_URL` and `CORS_ORIGIN` are set in the Coolify UI (the first as a Build Variable). Document both in `.env.example`.

### 11.3 Dockerfile - Node.js (multi-stage)
```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL
RUN npm run build && npm prune --omit=dev

FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build --chown=node:node /app ./
USER node
EXPOSE 3000
CMD ["npm", "start"]
```
(Remove the `ARG/ENV` pair if the project has no build-time public variables. For Next.js prefer `output: "standalone"` and copy `.next/standalone`, `.next/static`, `public`; run `node server.js` with `HOSTNAME=0.0.0.0`.)

### 11.4 Dockerfile - static SPA (Vite/React/Vue) served by nginx
```dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```
`nginx.conf` must include SPA fallback: `location / { try_files $uri $uri/ /index.html; }`.

### 11.5 Dockerfile - Python (FastAPI/Django style)
```dockerfile
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
RUN useradd -m appuser && chown -R appuser /app
USER appuser
EXPOSE 8000
CMD ["gunicorn", "app.main:app", "-k", "uvicorn.workers.UvicornWorker", "-b", "0.0.0.0:8000"]
```
If the healthcheck uses `curl`, add `RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*` before `USER`, or use `python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')"`.

### 11.6 `.dockerignore` (baseline)
```
.git
.gitignore
node_modules
npm-debug.log
.env
.env.*
!.env.example
dist
build
coverage
*.log
.vscode
.idea
docker-compose*.y*ml
Dockerfile*
README.md
```
(Adjust: do not ignore `dist`/`build` if the Dockerfile copies prebuilt output.)

### 11.7 `.env.example` (pattern)
```
# --- Required (set in Coolify > Environment Variables) ---
JWT_SECRET=change-me
# --- Build-time (tick "Build Variable") ---
VITE_API_URL=https://api.example.com
# --- Optional ---
POSTGRES_USER=app
POSTGRES_DB=app
# Auto-generated by Coolify (do not set): SERVICE_PASSWORD_POSTGRES
```

---

## 12. Validation checklist (run before reporting)

- [ ] `docker compose config` parses without errors/warnings (supply dummy values for `${VAR:?}`).
- [ ] No `container_name`, `networks`, `version`, routing `labels`, or HTTP `ports:`.
- [ ] Every long-running service has `restart` and `healthcheck`; the health tool exists in the image; `$` escaped as `$$` where needed.
- [ ] Every public service has either a `SERVICE_URL_<SERVICE>_<PORT>` entry or documented UI domain with `:port`; port matches the app's listen port.
- [ ] App binds `0.0.0.0`; proxy headers/trusted proxy configured.
- [ ] All secrets/config are `${VAR}` references; required ones use `:?`; generated secrets use `SERVICE_PASSWORD_*`.
- [ ] Build-time variables flow: UI Build Variable -> `build.args` -> Dockerfile `ARG`/`ENV`.
- [ ] DB/Redis/state use named volumes; upload paths persisted.
- [ ] `depends_on` uses health conditions; migrations handled safely.
- [ ] Dockerfile: lockfile install, multi-stage, non-root, exec-form CMD, `.dockerignore` present.
- [ ] `.env.example` lists every variable the compose file or app needs.
- [ ] No secrets committed, no `.env` copied into images.

---

## 13. Troubleshooting map

| Symptom | Likely cause | Fix |
|---|---|---|
| "No Available Server" / 503 | Container unhealthy, crashing, or no healthcheck-compatible tool | Check logs and `docker ps` health; fix healthcheck command, start period, or crash |
| 502 Bad Gateway | Wrong internal port in domain / `SERVICE_URL_*_<PORT>`, or app bound to `127.0.0.1` | Match port; bind `0.0.0.0` |
| Gateway timeout on first deploy | Slow start / migrations longer than `start_period` | Increase `start_period`, `retries` |
| Env var not shown in UI | Hardcoded `VAR=value` in compose | Use `${VAR}` |
| Magic variable not generated | Typo, wrong service name casing, old Coolify | Fix name (`SERVICE_` prefix, service name uppercased); update Coolify |
| `port is already allocated` | `ports:` used | Remove; use domain routing |
| `container name already in use` | `container_name:` set | Remove |
| App cannot reach DB | `localhost` used, or DB on another network | Use service name; or enable "Connect to Predefined Network" |
| Data lost after redeploy | No volume | Add named volume |
| Postgres auth fails after changing password | Password fixed at first init | Change inside DB or recreate volume (data loss) |
| Build killed / exit 137 | Out of memory | Add swap / bigger server / build server |
| Frontend calls old API URL | Build-time var not passed or cached | Add `build.args` + `ARG`; Force deploy without cache |
| Mixed content / redirect loop / CSRF failure | App unaware of HTTPS proxy | Configure trusted proxy, secure cookies, trusted origins |
| Permission denied on volume | Non-root user cannot write | `chown` path in Dockerfile before `USER` |
| Compose file not found | Wrong Base Directory / Compose Location | Set both correctly (monorepo) |
| Healthcheck passes locally, fails in Coolify | Tool missing in image, `$` not escaped, or checks public domain | Fix command; use `127.0.0.1` |

---

## 14. Output format when using this skill

After creating or modifying files, reply with:
1. **Files created/changed** (paths).
2. **Coolify UI settings**: Build Pack = Docker Compose; Base Directory; Docker Compose Location; domain per public service (with `:port` if using approach B); auto-deploy webhook.
3. **Environment variables to set in Coolify**: table of name, required?, build-time?, example/placeholder (never real secrets). Note which are auto-generated.
4. **Assumptions and open questions** (port, health endpoint, migration approach, anything unverifiable).
5. **Post-deploy verification**: open the domain, check the health endpoint, confirm the DB persists across a redeploy.

Keep explanations short; the files are the deliverable. Do not leave TODO placeholders in compose unless a value is truly unknown, and flag each one.

---

## 15. Maintenance

This file is a reusable convention for this repository set. When Coolify releases change behavior (magic variables, `exclude_from_hc`, network handling, build-arg handling), update the **(verify)** items and the version notes, and re-validate the templates by deploying them once on a test resource.
