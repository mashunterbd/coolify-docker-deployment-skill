---
name: coolify-cli
description: Deploy, manage, debug and monitor applications, databases, services and servers on a self-hosted (or Cloud) Coolify instance using the official `coolify` CLI. Use when the user asks to deploy a project to their server, check deployment status or build logs, read or set environment variables and secrets, restart or roll back an app, manage Docker Compose apps, databases and backups, validate servers, or troubleshoot anything running on Coolify.
---

# Coolify CLI Skill

Operate a Coolify instance from the terminal with the official `coolify` CLI (https://github.com/coollabsio/coolify-cli). This skill tells an AI agent how to install and configure the CLI, deploy a project, verify the result, debug failures, and do all of it safely.

> Source of truth: official docs at https://coolify.io/docs/cli and the CLI's own `--help`. Flags vary by CLI version. **When unsure, run `coolify <command> --help` before guessing.**

---

## 1. Operating Principles (read first)

1. **Verify before acting.** Always run the preflight (section 2) before any change.
2. **Read before write.** Inspect (`list`, `get`, `logs`) before you create, update, restart or delete.
3. **Never run destructive commands without explicit user confirmation in this conversation.** Destructive = `app delete`, `database delete`, `service delete`, `server remove`, `app env delete`, `database backup delete`, `deploy cancel`, anything with `--force`, `--delete-volumes`, `--delete-from-provider`. State exactly what will be removed and wait for a clear "yes".
4. **Protect secrets.**
   - Do not use `--show-sensitive` / `-s` unless the user needs the values. Never paste secret values into summaries, commit messages, files in the repo, or logs.
   - Never commit tokens or `.env` files. Prefer `app env sync --file` over typing secrets as command arguments (arguments end up in shell history and CI logs).
   - Treat the API token as a password. Do not echo it.
5. **Use structured output when parsing.** Use `--format json` for anything you parse. Never parse the default table output.
6. **Prefer UUIDs over names** for mutating commands. Names can be ambiguous; UUIDs are not.
7. **Verify after acting.** A deploy is not done when the command returns. It is done when the deployment finished, the container is running, logs are clean, and (if it has a domain) the URL responds.
8. **Report honestly.** If something failed or is uncertain, say so with evidence (log lines, status codes).

---

## 2. Preflight Checklist

Run these in order. Stop and fix at the first failure.

```bash
coolify version                  # 1. CLI installed?
coolify context list             # 2. Any contexts configured?
coolify context verify           # 3. URL + token valid? (prints Coolify version)
```

### 2.1 CLI not installed

Tell the user, then (with their approval) install:

| OS | Command |
|---|---|
| Linux / macOS | `curl -fsSL https://raw.githubusercontent.com/coollabsio/coolify-cli/main/scripts/install.sh \| bash` |
| Linux / macOS (no root) | `curl -fsSL https://raw.githubusercontent.com/coollabsio/coolify-cli/main/scripts/install.sh \| bash -s -- --user` (installs to `~/.local/bin`; add it to `PATH`) |
| macOS / Linux (Homebrew) | `brew install coollabsio/coolify-cli/coolify-cli` |
| Windows (PowerShell) | `irm https://raw.githubusercontent.com/coollabsio/coolify-cli/main/scripts/install.ps1 \| iex` |
| Windows (no admin) | `$env:COOLIFY_USER_INSTALL=1; irm https://raw.githubusercontent.com/coollabsio/coolify-cli/main/scripts/install.ps1 \| iex` |
| Go | `go install github.com/coollabsio/coolify-cli/coolify@latest` (ensure `$GOPATH/bin` is on `PATH`) |

After installing, open a **new terminal** so `PATH` refreshes, then run `coolify version`.
Update later with `coolify update`.

### 2.2 No context / not authenticated

The user must create an API token. Ask them to do this; do not invent a token.

1. Open the Coolify dashboard at `/security/api-tokens`.
2. Create a token with only the permissions the job needs.
3. For self-hosted Coolify, an admin must enable **Settings, Advanced, API Access**, and the caller IP must be allowed if an IP allowlist is configured.

Then add the instance as a context (use the full URL, **without** `/api/v1`):

```bash
coolify context add -d <context_name> <https://coolify.example.com> <api-token>
coolify context verify
```

Flags for `context add`: `-d, --default` (make default), `-f, --force` (overwrite an existing context).

Coolify Cloud: a `cloud` context for `https://app.coolify.io` already exists:

```bash
coolify context set-token cloud <api-token>
coolify context use cloud
coolify context verify
```

Ask the user to run the token command themselves (or paste the token only if they choose to) so the token does not leak into transcripts unnecessarily.

### 2.3 Context management

```bash
coolify context list
coolify context get <name>
coolify context use <name>                         # set default
coolify context set-default <name>
coolify context set-token <name> <token>
coolify context update <name> --url <new-url>      # also --name, --token
coolify context delete <name>                      # local only; does not revoke the token
coolify context version                            # Coolify API version
coolify --context staging app list                 # one-off context override
coolify config                                     # shows config file location
```

Config file location varies by OS and version, so run `coolify config` rather than assuming a path. The file contains tokens in plain text; never print or commit it.

### 2.4 Global flags (work on every command)

| Flag | Purpose |
|---|---|
| `--context <name>` | Use a specific context for this command |
| `--token <token>` | Override the context token for this command |
| `--format table\|json\|pretty` | Output format (`table` default, `json` for scripts, `pretty` for readable JSON) |
| `-s, --show-sensitive` | Reveal sensitive values (tokens, secrets, IPs). Use sparingly |
| `--debug` | Debug output (diagnosis only; adds operational detail) |

---

## 3. Discover What Exists

```bash
coolify server list
coolify project list                  # (some versions: coolify projects list)
coolify project get <project-uuid>    # environments in a project
coolify resource list                 # every app, database, service (some versions: resources list)
coolify app list --format json
coolify database list
coolify service list
coolify deploy list                   # deployments
```

Identify the target resource by name, then **record its UUID**. Example: a resource listed as `portal:main-<uuid>` has the UUID after the final dash segment; confirm with `coolify app get <uuid>`.

Name groups interchangeably: `server`/`servers`; `private-key`/`private-keys`/`key`/`keys`. Command nouns are singular (`app list`, not `apps list`). `deploy` is a top-level command (`coolify deploy uuid <uuid>`, **not** `coolify app deploy`).

---

## 4. Deploy a New Application

### 4.1 Gather requirements

You need: server UUID, project UUID, environment name (e.g. `production`) or UUID, repository URL and branch, build pack, exposed port. Ask the user for anything missing. Do not guess the server or project.

```bash
coolify server list
coolify project list
coolify server validate <server-uuid>     # confirm Coolify can reach the server
```

### 4.2 Create the application

Choose the creator that matches the source:

| Source | Command |
|---|---|
| Public Git repo | `coolify app create public` |
| Private repo via GitHub App | `coolify app create github` |
| Private repo via deploy key | `coolify app create deploy-key` |
| Dockerfile content | `coolify app create dockerfile` |
| Prebuilt Docker image | `coolify app create dockerimage` |

For anything except `public`, run `coolify app create <type> --help` first and use exactly the flags it lists.

Public repository example:

```bash
coolify app create public \
  --server-uuid <server-uuid> \
  --project-uuid <project-uuid> \
  --environment-name production \
  --git-repository https://github.com/<owner>/<repo> \
  --git-branch main \
  --build-pack nixpacks \
  --ports-exposes 3000 \
  --name <app-name> \
  --domains https://app.example.com
```

Required for `create public`: `--server-uuid`, `--project-uuid`, `--git-repository`, `--git-branch`, `--build-pack` (`nixpacks`, `static`, `dockerfile`, `dockercompose`), `--ports-exposes`. Also provide `--environment-name` or `--environment-uuid`.

Useful optional flags: `--build-command`, `--install-command`, `--start-command`, `--base-directory`, `--publish-directory` (static sites), `--health-check-enabled` with `--health-check-path`, `--limits-memory`, `--limits-cpus`, `--instant-deploy`, `--is-preview-deployments-enabled`, `--tag`.

Build-pack guidance:
- **Next.js / Node / Python / Go app, no Dockerfile**: `nixpacks`, with the framework's real port in `--ports-exposes`.
- **Static site (Vite/React/Astro build output)**: `static`, with `--publish-directory dist` (or `build`/`out`) and a `--build-command`.
- **Repo has a Dockerfile**: `dockerfile`.
- **Repo has docker-compose.yml**: `dockercompose`; map domains per service with repeatable `--compose-domain <service>=<url>`.

The response contains the new **application UUID**. Save it and use it for all following commands.

### 4.3 Set environment variables and secrets

```bash
# Preferred: bulk load from a file (updates existing keys, creates missing, does NOT delete extras)
coolify app env sync <app-uuid> --file .env.production
coolify app env sync <app-uuid> --file .env.production --build-time --preview

# Single variable
coolify app env create <app-uuid> --key API_URL --value https://api.example.com
```

Flags for `env create` / `env update`: `--key`, `--value`, `--preview`, `--build-time`, `--runtime` (update), `--is-literal` (no variable interpolation), `--is-multiline`.

Important:
- Variables needed while building (e.g. `NEXT_PUBLIC_*`, `VITE_*`) must be **build-time**: use `--build-time`. Changing them requires a **new deploy** to take effect.
- `env sync` never deletes variables missing from the file; remove stale ones with `env delete`.
- Check the result without leaking values: `coolify app env list <app-uuid>`. Add `-s` only if the user asked to see values. Use `--all` to include preview variables and `--preview` for preview-only.
- Read one variable: `coolify app env get <app-uuid> <env-uuid-or-key>`.

### 4.4 Deploy

```bash
coolify deploy uuid <app-uuid>
# or
coolify deploy name <app-name>
# several at once
coolify deploy batch api,worker,frontend
```

Options: `-f, --force` (deploy even with no detected change; use only when needed), `--pull-request-id <id>` (preview deployment), `--docker-tag <tag>` (override image tag; needs Coolify `4.0.0-beta.471+`).

Cancel: `coolify deploy cancel <deployment-uuid>` (destructive; confirm first).

### 4.5 Verify (mandatory)

```bash
coolify deploy list                                    # find newest deployment uuid + status
coolify deploy get <deployment-uuid>                   # status details
coolify app deployments list <app-uuid>                # history for this app
coolify app deployments logs <app-uuid>                # latest deployment (build) logs
coolify app deployments logs <app-uuid> <deployment-uuid> --follow   # stream a specific build
coolify app get <app-uuid>                             # status, FQDN, config
coolify app logs <app-uuid> --lines 100 --show-timestamps            # runtime logs
```

Then, if the app has a domain, confirm it answers (use the agent's own shell tools, not Coolify):

```bash
curl -sS -o /dev/null -w "%{http_code}\n" https://app.example.com
```

**Definition of done:** deployment status is finished/successful, `app get` shows the app running (healthy if a health check is set), runtime logs show no repeated errors or crash loop, and the URL returns the expected HTTP status. Build and runtime can both be "green" while DNS or proxy is still wrong, so check the URL.

Deployment is asynchronous. Poll `coolify deploy get <deployment-uuid>` every 10-20 seconds, with a sensible timeout (for example 10 minutes). Do not report success until it reaches a terminal state.

---

## 5. Redeploy, Update and Control

```bash
coolify app get <uuid>
coolify app start   <uuid>
coolify app stop    <uuid>        # confirm with user if it serves production traffic
coolify app restart <uuid>        # restart without rebuild; env changes at runtime
coolify deploy uuid <uuid>        # rebuild + redeploy (needed for build-time env / code changes)
```

Update configuration:

```bash
coolify app update <uuid> \
  --git-branch main \
  --domains https://app.example.com \
  --build-command "npm run build" \
  --start-command "npm start" \
  --ports-exposes 3000 \
  --health-check-enabled --health-check-path /health
```

Other `app update` flags: `--name`, `--description`, `--git-repository`, `--install-command`, `--base-directory`, `--publish-directory`, `--dockerfile`, `--docker-image`, `--docker-tag`, `--ports-mappings`. After a config change, redeploy and verify (4.4 and 4.5).

Rule of thumb: **code or build-time env changed means deploy; runtime-only env changed means restart or deploy.**

### Rollback

```bash
coolify app rollback images <uuid>              # list available rollback images
coolify app rollback run <uuid> --commit <sha>  # roll back to a commit
```

Verify afterwards exactly as in 4.5.

### Preview deployments (pull requests)

```bash
coolify deploy uuid <app-uuid> --pull-request-id <id>
coolify app previews list <app-uuid>
```

Other `app previews` subcommands: `create`, `get`, `update`, `delete`, `logs` (run `--help` for flags).

---

## 6. Debugging Playbook

Work top-down. Collect evidence before changing anything.

```
Deployment failed or app not working
|
+-- 1. coolify deploy list / deploy get <deployment-uuid>      -> which deployment, what status?
+-- 2. coolify app deployments logs <app-uuid> [deployment-uuid] -> BUILD problem?
|        - install/build command failed, missing dependency, wrong base directory
|        - missing build-time env var (NEXT_PUBLIC_*, VITE_*), wrong build pack
|        - Dockerfile error, out of memory/disk during build
+-- 3. coolify app logs <app-uuid> --lines 200 --show-timestamps -> RUNTIME problem?
|        - crash on start, missing runtime env var, DB connection refused
|        - app listens on a different port than --ports-exposes, or on 127.0.0.1 only
|        - (Compose) add --service <name> to pick one container
+-- 4. coolify app get <uuid>                                   -> status, domains, config as expected?
+-- 5. coolify app env list <uuid>                              -> required vars present, right scope?
+-- 6. curl the URL                                             -> 502/503/404 = proxy/port/domain problem
+-- 7. coolify server validate <server-uuid>                    -> server reachable, Docker OK?
+-- 8. coolify server get <server-uuid> --resources             -> resource status on the server
+-- 9. coolify server proxy get <server-uuid>                   -> proxy status (restart: server proxy restart)
```

Common causes:

| Symptom | Likely cause | Check or fix |
|---|---|---|
| Build fails on a missing variable | Env var not set as build-time | `app env update <uuid> <key> --value ... --build-time`, then redeploy |
| 502 / Bad Gateway | App not listening on the exposed port or on `0.0.0.0` | Compare `--ports-exposes` with the app's real port; fix and redeploy |
| 404 from the proxy | Domain not set or mismatched | `app get`; `app update --domains https://...`; check DNS |
| Container restarts repeatedly | Crash at startup | `app logs --lines 200`; check env and DB connectivity |
| Change has no effect | Deploy needed, not restart | `coolify deploy uuid <uuid>` |
| "No changes" and nothing deployed | Nothing new to build | `coolify deploy uuid <uuid> --force` if a rebuild is really intended |
| Old env value still in use | Not redeployed | Redeploy or restart as appropriate |

### CLI and auth errors

| Error | Meaning | Action |
|---|---|---|
| `401` | Token missing, expired, revoked or invalid | Ask the user for a new token; `context set-token <name> <token>`; `context verify` |
| `403` | Token lacks permission, **API Access disabled**, or IP not in allowlist | Check token abilities, Settings, Advanced, API Access, and the IP allowlist |
| Resources missing | Token belongs to a different team | Confirm the token's team |
| `command not found: coolify` | Not installed or `PATH` stale | Section 2.1; open a new terminal |
| Connection refused / timeout | Wrong URL, firewall, instance down | `coolify context get <name>`; check URL, port, HTTPS; `curl` the URL |
| Unknown flag | CLI version differs | `coolify <command> --help`; consider `coolify update` |

Use `--debug` only when you need more detail, and never paste debug output containing tokens.

---

## 7. Databases and Backups

```bash
coolify database list
coolify database get <db-uuid>
coolify database start|stop|restart <db-uuid>
```

Create (types: `postgresql`, `mysql`, `mariadb`, `mongodb`, `redis`, `keydb`, `clickhouse`, `dragonfly`):

```bash
coolify database create postgresql \
  --server-uuid <server-uuid> \
  --project-uuid <project-uuid> \
  --environment-name production \
  --name mydb \
  --instant-deploy
```

Optional: `--destination-uuid`, `--image`, `--is-public`, `--public-port`, `--limits-memory`, `--limits-cpus`, plus engine-specific flags (see `--help`).

**Databases hold data.** `database delete` defaults to deleting configurations, volumes, networks and running cleanup. Always confirm with the user, and make sure a recent backup exists first.

Backups:

```bash
coolify database backup list <db-uuid>
coolify database backup create <db-uuid> \
  --frequency "0 2 * * *" --enabled \
  --retention-amount-local 7
coolify database backup trigger <db-uuid> <backup-uuid>       # run one now
coolify database backup executions <db-uuid> <backup-uuid>    # verify it succeeded
coolify database backup update <db-uuid> <backup-uuid> ...
```

S3 options: `--save-s3`, `--s3-storage-uuid`, `--retention-amount-s3`, `--retention-days-s3`, `--retention-storage-s3`. Also: `--databases-to-backup`, `--dump-all`, `--timeout`, `--disable-local`.
A backup is only real once an execution shows success: always check `backup executions`.

Connect an app to a database using the database's **internal** connection URL from `database get`, and store it as an app env var (section 4.3).

---

## 8. Services (one-click / Compose services)

```bash
coolify service list
coolify service get <uuid>
coolify service start|stop|restart <uuid>
coolify service env list <uuid>
coolify service env sync <uuid> --file .env
coolify service env create|update|delete ...
```

`service delete` is destructive: confirm first.

---

## 9. Servers

```bash
coolify server list
coolify server get <uuid> --resources        # what runs on it
coolify server validate <uuid>               # check connectivity and Docker
coolify server domains <uuid>
coolify server destinations list <uuid>      # run `coolify server --help` to confirm subcommand names in your version
```

Add a server (needs an existing private key in Coolify):

```bash
coolify private-key list
coolify private-key add <name> @~/.ssh/id_ed25519      # '@' reads the key from a file
coolify server add <name> <ip> <private-key-uuid> --user root --port 22 --validate
```

Other server operations available in current CLI versions (check `--help`): `server proxy get|restart|update|set-config`, `server docker-cleanup get|run|update|executions`, `server log-drains`, `server sentinel`, `server registry login|logout|list|check`, Cloudflare tunnel, and provider helpers for Hetzner, DigitalOcean and Vultr.

### Removing a server (DANGEROUS)

```bash
coolify server remove <uuid>
```

- `--force` also deletes **all resources on the server**.
- `--delete-from-provider` also deletes the cloud VM at Hetzner/Vultr/DigitalOcean. **Irreversible.**

Procedure: (1) `server get <uuid> --resources` and show the user what is on it, (2) state exactly which flags you intend to use, (3) require explicit confirmation, (4) only then run it. Never add `--force` or `--delete-from-provider` on your own.

---

## 10. Other Resources

```bash
coolify github list                         # GitHub App integrations
coolify github repos <github-app-uuid>
coolify github branches <github-app-uuid> <owner/repo>
coolify team list | team current | team members list
coolify private-key list
coolify app tag list <uuid>                 # tags: add / list / remove
coolify app storage list <uuid>             # persistent storage and storage backups
coolify app task list <uuid>                # scheduled tasks (create/update/delete/execute/executions)
coolify completion powershell|bash|zsh|fish # shell completion
```

---

## 11. Automation and CI

- Failed commands return a **non-zero exit code**; use `set -e` (bash) or check `$LASTEXITCODE` (PowerShell).
- Use `--format json` and a JSON tool (`jq`, or PowerShell `ConvertFrom-Json`) to parse.
- Inject tokens from the secret store: `coolify --context production --token "$COOLIFY_TOKEN" deploy name api`.
- Do not use `--show-sensitive` or `--debug` in shared logs.
- Give CI tokens least privilege, one token per workflow.

```bash
set -e
coolify --context production context verify
coolify --context production deploy name api
coolify --context production --format json deploy list
```

---

## 12. End-to-End Agent Workflow

When the user says "deploy this project to my server":

1. **Understand the project.** Read the repo: framework, package manager, build and start commands, port, env vars (`.env.example`), Dockerfile or compose file.
2. **Preflight** (section 2). Fix install/auth with the user if needed.
3. **Discover** server, project and environment (section 3). Ask the user to pick if ambiguous.
4. **Check for an existing app** (`coolify app list --format json`). If it exists, update and redeploy instead of creating a duplicate.
5. **Create** the app (section 4.2) with the right build pack and port.
6. **Configure env** (section 4.3). Mark build-time variables correctly. Never print secret values.
7. **Deploy** (section 4.4).
8. **Verify** (section 4.5): deployment status, build logs, runtime logs, URL check.
9. **Debug loop** if anything is wrong (section 6): read the evidence, fix one thing at a time, redeploy, re-verify. Stop and ask the user if you hit auth errors, destructive decisions, or the same failure after 3 attempts.
10. **Report**, using the template below.

### Final report template

```
Deployment result: SUCCESS | FAILED | PARTIAL
App: <name> (<app-uuid>) on server <server-name>
Deployment: <deployment-uuid> - <status>
URL: <https://...> -> HTTP <code>
Build: <ok / key findings>
Runtime logs: <clean / issues found, with 1-3 relevant lines>
Env vars: <N set; build-time: ...> (values not shown)
Changes I made: <created app, set env, ran deploy, ...>
Open items / risks: <anything the user should decide or fix>
```

---

## 13. Quick Reference

```bash
# Setup
coolify version | coolify context add -d <name> <url> <token> | coolify context verify
# Discover
coolify server list | project list | resource list | app list | database list | service list
# Create + configure
coolify app create public --server-uuid .. --project-uuid .. --environment-name production \
  --git-repository .. --git-branch main --build-pack nixpacks --ports-exposes 3000
coolify app env sync <uuid> --file .env.production
# Deploy + verify
coolify deploy uuid <uuid>
coolify deploy get <deployment-uuid>
coolify app deployments logs <uuid> [deployment-uuid] [--follow]
coolify app logs <uuid> --lines 100 --show-timestamps [--service <name>] [--follow]
# Control
coolify app start|stop|restart <uuid>
coolify app rollback images <uuid> | coolify app rollback run <uuid> --commit <sha>
# Always available
coolify --help | coolify <group> --help | coolify <group> <command> --help
```

## References

- CLI docs: https://coolify.io/docs/cli
- Authentication: https://coolify.io/docs/cli/authentication
- Contexts: https://coolify.io/docs/cli/contexts
- Deploy applications workflow: https://coolify.io/docs/cli/deploy-applications
- Automation and CI: https://coolify.io/docs/cli/automate-with-cli
- CLI repository: https://github.com/coollabsio/coolify-cli
