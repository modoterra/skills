---
name: compose
description: >
  Generate or normalize Docker Compose stacks with collision-resistant high host ports,
  project-named networks, and project-prefixed volumes (host-global unique), then wire
  those ports into the current project's config (e.g. Laravel .env). Also assign high
  free ports for host-run app HTTP and Vite/dev servers so multi-project local stacks
  do not collide. Use when the user wants compose services, docker-compose, randomized
  ports, local infra (Postgres, Redis, Mailpit, MinIO, etc.), app/vite port allocation,
  or runs /compose.
compatibility: Designed for Agent Skills-compatible coding agents. Requires Docker Compose and shell access to inspect listening ports.
metadata:
  author: Modoterra
  version: "1.2.0"
---

# Compose

Build local Docker Compose infrastructure that is safe to run alongside other projects on the same host. Also assign **high free ports for host-run app processes** (HTTP app server, Vite/dev bundler) so multiple projects can run concurrently without fighting over 8000/5173.

## Goals

1. Map requested services to **randomized high host ports** so collisions with other stacks are rare.
2. **Normalize names**: network Docker name = project name (not `project_default`); volume Docker names = `{project}_{service}` because volume names are **unique across the entire Docker host**.
3. **Wire host ports** into the current application config (`.env`, etc.) so the app talks to those ports on `127.0.0.1`.
4. **Assign high free ports for the host-run app HTTP server and Vite (or equivalent dev server)** and wire them into framework config (`APP_URL`, `SERVER_PORT`, `VITE_PORT`, etc.) so multi-project local dev does not collide on 8000/5173.

## When to run

- User asks for Compose services (e.g. "add postgres and redis", "set up local mail", `/compose`).
- User wants an existing `compose.yaml` / `docker-compose.yml` normalized (ports, network, volumes).
- User wants project config updated to match Compose-published ports.
- User wants app HTTP / Vite ports randomized for multi-project hosts (even if Compose services are already present).

## Inputs

Collect if not already clear:

1. **Services** — which containers (postgres, redis, mailpit, mysql, minio, meilisearch, …).
2. **Project name** — default to the repository / directory name, lowercased and safe for Compose (`[a-z0-9][a-z0-9_-]*`). Confirm if ambiguous.
3. **Compose file path** — prefer existing `compose.yaml`, `compose.yml`, `docker-compose.yml`, or `docker-compose.yaml`; otherwise create `compose.yaml`.
4. **App config targets** — discover from the project (see [Wire ports into the project](#wire-ports-into-the-project)).
5. **Host-run app ports** — whether the app and frontend dev server run on the host (default for this skill). Always allocate them for detected frameworks that need them (e.g. Laravel + Vite).

Do not invent **container** services the user did not request. Prefer official or well-known images and documented defaults.

Do allocate **app HTTP + Vite/dev-server** high ports whenever the project is a host-run web app that uses them (Laravel with Vite is the primary case). These are not Compose services unless the user wants Sail/full containerization.

## Port assignment

### Range

Assign **host** ports from the **high ephemeral band**:

- Inclusive range: **49152–65535**
- Prefer mid-high draws so you avoid the very top edge used by some OS ephemeral allocators when possible, but any free port in the range is valid.

For **Compose services**, container-internal ports stay at the service's standard port (5432, 6379, 1025, …). Only the **left-hand host port** is randomized.

For **host-run app processes** (HTTP app server, Vite), there is no container side: allocate a free high port and configure the process to listen on `127.0.0.1:<port>`.

### What gets a high port

Allocate unique free high ports for **all** of the following that apply:

| Kind | Examples | Notes |
|------|----------|--------|
| Compose published ports | postgres, redis, mailpit SMTP/UI, minio, … | Host port only |
| App HTTP server | Laravel `artisan serve`, Symfony CLI, generic PHP/Node HTTP | Always for host-run web apps |
| Frontend / Vite dev server | Vite (Laravel, plain Vite), Webpack dev server if used | When the project has Vite or equivalent |
| Other local UIs | only if the stack already documents a fixed local UI port you are replacing | Do not invent extra UIs |

Never leave the app on default **8000** or Vite on default **5173** when this skill runs for a multi-project host setup—always reassign both into the high band (unless the user explicitly asks to keep a specific app/vite port).

### Randomization and uniqueness

1. For each port the stack needs (Compose host ports **and** app/vite ports), pick a cryptographically random integer in the range (e.g. `python3 -c 'import secrets; print(secrets.randbelow(65535-49152+1)+49152)'` or equivalent).
2. Ensure uniqueness **across this project's full port set** (Compose host ports + app HTTP + Vite + any other allocated local ports). No two allocations may share a host port.
3. Before committing a port, verify it is free on the host:

```bash
# Linux: port is free if ss prints nothing
ss -H -ltn "sport = :<PORT>" 2>/dev/null
ss -H -lun "sport = :<PORT>" 2>/dev/null
```

If busy, draw again. Cap retries (e.g. 32) per port; if exhausted, stop and report.

4. Never reuse well-known low ports (80, 443, 3000, 3306, 5173, 5432, 6379, 8000, 8080, …) as allocated host or app ports for this skill's output.

### Port mapping form (Compose)

Use explicit host→container mappings:

```yaml
ports:
  - "52341:5432"
```

Prefer binding to localhost when the service is only for local app use and Compose version supports it:

```yaml
ports:
  - "127.0.0.1:52341:5432"
```

Default to `127.0.0.1` bind unless the user explicitly needs LAN access.

### Host-run app listen form

Prefer loopback for local-only dev:

- App: `http://127.0.0.1:<APP_PORT>`
- Vite: `http://127.0.0.1:<VITE_PORT>`

Document the exact start commands in the summary (framework-specific flags/env). Do **not** add app or Vite as Compose services by default; only allocate ports and wire config.

## Naming normalization

### Project / Compose project name

Set the Compose top-level `name` to the project name:

```yaml
name: myapp
```

This becomes the Compose project name instead of the directory-derived default.

### Network

Define a single default network whose **Docker network name** is exactly the project name (not `project_default` or `project_network`):

```yaml
networks:
  default:
    name: myapp
```

Attach services to `default` (implicit) unless multi-network is required.

### Volumes

Docker volume names are **global on the host** (not scoped per Compose project). Two stacks that both declare `name: postgres` will collide or share data unintentionally.

For each named volume, set the Docker volume **name** to `{project}_{service}`:

```yaml
services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
    name: myapp_postgres
```

Rules:

- One primary data volume per stateful service → `name: {project}_{service}` (e.g. `myapp_postgres`, `myapp_redis`, `myapp_minio`).
- If a service needs multiple volumes, primary uses `{project}_{service}`; secondary uses `{project}_{service}_{purpose}` (e.g. `myapp_minio_config`).
- Compose volume **keys** (left of `volumes:` under the service) may stay short (`postgres_data`); only the explicit `name:` field must be host-unique.
- Prefer named volumes over bind mounts for database data unless the user or repo already uses binds.
- Never use bare service names (`postgres`) as the Docker volume `name:` on a multi-project host.

### Service names

Use short, conventional service keys: `postgres`, `redis`, `mailpit`, `mysql`, `minio`, `meilisearch`, etc. Do not prefix with the project name.

## Compose file structure

Produce a modern Compose file (Compose Specification). Minimal shape:

```yaml
name: <project>

services:
  <service>:
    image: <image>:<tag>
    restart: unless-stopped
    ports:
      - "127.0.0.1:<host_port>:<container_port>"
    environment: { ... }
    volumes:
      - <volume_key>:/path/in/container
    networks:
      - default
    healthcheck: { ... }  # when standard and useful

networks:
  default:
    name: <project>

volumes:
  <volume_key>:
    name: <project>_<service>
```

### Service defaults (reference)

Use current official images and documented env vars. Typical local-dev defaults:

| Service     | Image (example)              | Container port | Typical env / notes                                      |
|-------------|------------------------------|----------------|----------------------------------------------------------|
| postgres    | `postgres:16`                | 5432           | `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`      |
| mysql       | `mysql:8`                    | 3306           | `MYSQL_ROOT_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_USER`    |
| redis       | `redis:7-alpine`             | 6379           | Optional AOF volume                                      |
| mailpit     | `axllent/mailpit`            | 1025 SMTP, 8025 UI | Publish both if useful                               |
| minio       | `minio/minio`                | 9000 API, 9001 console | `minio server /data --console-address ":9001"`   |
| meilisearch | `getmeili/meilisearch`       | 7700           | `MEILI_ENV=development`, master key if needed            |

Credentials for local stacks:

- Prefer generating strong random passwords when creating new services.
- Reuse existing `.env` credentials when updating an existing stack so data volumes remain accessible.
- Never commit real secrets; write them to `.env` (gitignored) and reference via Compose `env_file` or variable substitution when appropriate.

If the repo already has Compose conventions (Sail, custom images, profiles), extend them rather than fighting them. Still apply port randomization and name normalization where they do not break the existing workflow.

## Wire ports into the project

After host ports are chosen, update the **application** so it connects via `127.0.0.1` and the new host ports—including **app HTTP** and **Vite** when allocated.

### Discovery order

1. Inspect the repo for framework and config files.
2. Map each Compose-published port to the config keys that framework expects.
3. Map app HTTP + Vite ports to framework/env keys (see framework sections below).
4. Update existing keys when present; add missing keys only when the app needs them for the new services or local dev servers.
5. Prefer gitignored local env files over committed examples. Update `.env.example` / `.env.sample` with **placeholder** host ports or comments only when the project already documents local ports there — do not put random per-machine ports into committed templates as if they were universal.

### Laravel

Look for `.env` (and create from `.env.example` if `.env` is missing).

#### Compose-backed services

| Service    | Keys (typical) |
|------------|----------------|
| postgres   | `DB_CONNECTION=pgsql`, `DB_HOST=127.0.0.1`, `DB_PORT=<host>`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` |
| mysql      | `DB_CONNECTION=mysql`, `DB_HOST=127.0.0.1`, `DB_PORT=<host>`, … |
| redis      | `REDIS_HOST=127.0.0.1`, `REDIS_PORT=<host>`, `REDIS_PASSWORD` if set |
| mailpit    | `MAIL_MAILER=smtp`, `MAIL_HOST=127.0.0.1`, `MAIL_PORT=<smtp host port>`, `MAIL_USERNAME=null`, `MAIL_PASSWORD=null`, `MAIL_ENCRYPTION=null`; optional UI URL in comments or `MAILPIT_UI_PORT` |
| meilisearch| `MEILISEARCH_HOST=http://127.0.0.1:<host>`, key vars if used |
| minio/S3   | `AWS_ENDPOINT=http://127.0.0.1:<api host>`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`, `AWS_BUCKET`, path-style flags as required by the app |

#### Host-run app HTTP + Vite (required for Laravel when this skill runs)

Always allocate two free high ports for Laravel projects that run on the host:

| Process | Port role | Config keys / files |
|---------|-----------|---------------------|
| App HTTP (`artisan serve` / Octane local / similar) | `APP_PORT` | `APP_URL=http://127.0.0.1:<APP_PORT>`, `SERVER_PORT=<APP_PORT>` (`artisan serve` reads `SERVER_PORT` as default `--port`) |
| Vite | `VITE_PORT` | `VITE_PORT=<VITE_PORT>`; ensure Vite listens on that port |

Wiring details:

1. **`.env`**
   - `APP_URL=http://127.0.0.1:<APP_PORT>`
   - `SERVER_PORT=<APP_PORT>`
   - `VITE_PORT=<VITE_PORT>`
   - Optional clarity comments:
     - `# App: php artisan serve --host=127.0.0.1 --port=${SERVER_PORT}`
     - `# Vite uses VITE_PORT via laravel-vite-plugin`

2. **`vite.config.ts` / `vite.config.js`**
   - Prefer env-driven port so `.env` stays the source of truth. Laravel Vite plugin already defaults to `env.VITE_PORT` when set; still set explicit server options when the project hardcodes `server.port`:

```ts
server: {
  host: "127.0.0.1",
  port: Number(process.env.VITE_PORT) || 5173,
  strictPort: true,
},
```

   - If the file already has `server: { ... }`, merge `host` / `port` / `strictPort` rather than duplicating the block.
   - Use `strictPort: true` so Vite fails fast instead of silently moving to the next port (which would desync `.env`).

3. **Do not** put app/vite processes into Compose unless the user asked for Sail or a full containerized app.

4. **Summary commands** (after wiring):

```bash
docker compose -f compose.yaml up -d   # if services were defined
php artisan serve --host=127.0.0.1 --port="$SERVER_PORT"
npm run dev                            # Vite reads VITE_PORT
# or: composer run dev / php artisan dev when the project uses a concurrent dev script
```

If the project uses `composer run dev` / `artisan dev` (concurrently serve + vite + queue), ensure env ports are enough for child processes; only change those scripts if they hardcode `--port=8000` or Vite ports.

Use `127.0.0.1` (or `localhost`) for host-side app processes. Do **not** use Docker service DNS names in `.env` unless the **PHP/app process itself runs inside the Compose network**.

### Other stacks

Apply the same idea:

- **Node / Nest / Next**: `.env`, `.env.local` — allocate high `PORT` / `NEXT_PORT` and Vite port when applicable; set `strictPort` when using Vite.
- **Django**: `.env`, `settings` env module — `runserver 127.0.0.1:<APP_PORT>`; frontend Vite port if present.
- **Rails**: `.env`, `config/database.yml` only if the project already uses non-ENV port config; `bin/rails server -p <APP_PORT>`; Vite/jsbundling ports if present.
- **Generic**: any `*.env*`, `config/local.*`, or documented docker port files

Search the codebase for existing `DB_PORT`, `REDIS_PORT`, `APP_URL`, `SERVER_PORT`, `VITE_PORT`, `localhost:8000`, `localhost:5173`, `localhost:5432`, etc., and update consistently. Prefer a single source of truth (env vars) over scattering literals.

### What not to change

- Production / CI deploy configs unless the user asks.
- Committed secrets or vault material.
- Ports that are intentionally fixed for a documented public interface.
- Default low ports left in place “for convenience” on multi-project hosts—this skill exists to avoid that.

## Existing Compose files

When a compose file already exists:

1. Read it fully (includes, profiles, overrides).
2. Preserve images, command, healthchecks, and env that still apply.
3. Reassign **host** ports into the high random range (unless the user asks to keep a specific host port).
4. Normalize `name`, `networks.default.name`, and volume `name:` fields.
5. Re-wire project config to the new host ports.
6. Still allocate/reassign **app HTTP + Vite** high ports and wire them even when only normalizing an existing Compose file.
7. Warn if renaming volumes will detach existing data (`docker volume ls`). Offer to keep old volume names when data preservation matters, or document a one-time migration.

## Workflow checklist

1. Confirm services + project name.
2. Detect framework and config files (including Vite / frontend tooling).
3. Read existing Compose and `.env` if present.
4. Allocate unique free high host ports for **Compose services + app HTTP + Vite** (as applicable).
5. Write / update Compose with normalized network and volume names.
6. Write / update app config with `127.0.0.1` + new dependency ports, credentials, **`APP_URL` / `SERVER_PORT`, and `VITE_PORT`** (and vite config if needed).
7. Show a concise summary table: service/process → binding → config keys touched.
8. Suggest next commands only; do not start containers or long-running dev servers unless asked:

```bash
docker compose -f compose.yaml up -d
php artisan serve --host=127.0.0.1 --port="$SERVER_PORT"   # Laravel example
npm run dev
```

9. If credentials or volume renames can lose data, call that out explicitly before `up`.

## Output summary format

Always finish with a short table and file list, for example:

| Service / process | Host binding              | Config |
|-------------------|---------------------------|--------|
| postgres          | `127.0.0.1:52341` → 5432  | `DB_PORT=52341` in `.env` |
| redis             | `127.0.0.1:60102` → 6379  | `REDIS_PORT=60102` in `.env` |
| app (HTTP)        | `127.0.0.1:51447`         | `APP_URL`, `SERVER_PORT=51447` in `.env` |
| vite              | `127.0.0.1:53291`         | `VITE_PORT=53291` in `.env` (+ vite `server.port` if needed) |

Network: `myapp`  
Volumes: `myapp_postgres`, `myapp_redis`  
Files: `compose.yaml`, `.env`, optionally `vite.config.ts`

## Constraints

- Do not expose Compose services on `0.0.0.0` by default.
- Do not leave host-run app/Vite on well-known defaults (8000/5173/3000) when this skill runs—always assign high free ports unless the user opts out.
- Do not use low/fixed host ports for multi-project hosts.
- Do not use bare service names as Docker volume `name:` values — always prefix with `{project}_` so volumes stay unique on the host.
- Network Docker name stays the bare project name (not `project_default`); that is intentional and separate from volume naming.
- Do not put the app on the Compose network by default unless the user wants a full containerized app (Sail-style); this skill targets **host-run apps + containerized dependencies**.
- Prefer editing existing files over adding parallel compose variants (`docker-compose.dev.yml`, etc.) unless the repo already uses that pattern.
