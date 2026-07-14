---
name: compose
description: >
  Generate or normalize Docker Compose stacks with collision-resistant high host ports,
  project-named networks, and service-named volumes, then wire those ports into the
  current project's config (e.g. Laravel .env). Use when the user wants compose services,
  docker-compose, randomized ports, local infra (Postgres, Redis, Mailpit, MinIO, etc.),
  or runs /compose.
compatibility: Designed for Agent Skills-compatible coding agents. Requires Docker Compose and shell access to inspect listening ports.
metadata:
  author: Modoterra
  version: "1.0.0"
---

# Compose

Build local Docker Compose infrastructure that is safe to run alongside other projects on the same host.

## Goals

1. Map requested services to **randomized high host ports** so collisions with other stacks are rare.
2. **Normalize names**: network = project name; each volume = the service it belongs to (not `project_service`).
3. **Wire host ports** into the current application config (`.env`, etc.) so the app talks to those ports on `127.0.0.1`.

## When to run

- User asks for Compose services (e.g. "add postgres and redis", "set up local mail", `/compose`).
- User wants an existing `compose.yaml` / `docker-compose.yml` normalized (ports, network, volumes).
- User wants project config updated to match Compose-published ports.

## Inputs

Collect if not already clear:

1. **Services** — which containers (postgres, redis, mailpit, mysql, minio, meilisearch, …).
2. **Project name** — default to the repository / directory name, lowercased and safe for Compose (`[a-z0-9][a-z0-9_-]*`). Confirm if ambiguous.
3. **Compose file path** — prefer existing `compose.yaml`, `compose.yml`, `docker-compose.yml`, or `docker-compose.yaml`; otherwise create `compose.yaml`.
4. **App config targets** — discover from the project (see [Wire ports into the project](#wire-ports-into-the-project)).

Do not invent services the user did not request. Prefer official or well-known images and documented defaults.

## Port assignment

### Range

Assign **host** ports from the **high ephemeral band**:

- Inclusive range: **49152–65535**
- Prefer mid-high draws so you avoid the very top edge used by some OS ephemeral allocators when possible, but any free port in the range is valid.

Container-internal ports stay at the service's standard port (5432, 6379, 1025, …). Only the **left-hand host port** is randomized.

### Randomization and uniqueness

1. For each published port the stack needs, pick a cryptographically random integer in the range (e.g. `python3 -c 'import secrets; print(secrets.randbelow(65535-49152+1)+49152)'` or equivalent).
2. Ensure uniqueness **within this Compose file** (no two host ports collide).
3. Before committing a port, verify it is free on the host:

```bash
# Linux: port is free if ss prints nothing
ss -H -ltn "sport = :<PORT>" 2>/dev/null
ss -H -lun "sport = :<PORT>" 2>/dev/null
```

If busy, draw again. Cap retries (e.g. 32) per port; if exhausted, stop and report.

4. Never reuse well-known low ports (80, 443, 3306, 5432, 6379, 8080, …) as host ports for this skill's output.

### Port mapping form

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

For each named volume, set the Docker volume **name** to the **service name** that owns it (not `project_postgres`):

```yaml
services:
  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
    name: postgres
```

Rules:

- One primary data volume per stateful service → external name = service name (`postgres`, `redis`, `minio`).
- If a service needs multiple volumes, name the primary after the service; secondary volumes use `service-purpose` (e.g. `minio-config`) and set `name:` accordingly.
- Prefer named volumes over bind mounts for database data unless the user or repo already uses binds.

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
    name: <service>
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

After host ports are chosen, update the **application** so it connects via `127.0.0.1` and the new host ports.

### Discovery order

1. Inspect the repo for framework and config files.
2. Map each Compose-published port to the config keys that framework expects.
3. Update existing keys when present; add missing keys only when the app needs them for the new services.
4. Prefer gitignored local env files over committed examples. Update `.env.example` / `.env.sample` with **placeholder** host ports or comments only when the project already documents local ports there — do not put random per-machine ports into committed templates as if they were universal.

### Laravel

Look for `.env` (and create from `.env.example` if `.env` is missing).

| Service    | Keys (typical) |
|------------|----------------|
| postgres   | `DB_CONNECTION=pgsql`, `DB_HOST=127.0.0.1`, `DB_PORT=<host>`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` |
| mysql      | `DB_CONNECTION=mysql`, `DB_HOST=127.0.0.1`, `DB_PORT=<host>`, … |
| redis      | `REDIS_HOST=127.0.0.1`, `REDIS_PORT=<host>`, `REDIS_PASSWORD` if set |
| mailpit    | `MAIL_MAILER=smtp`, `MAIL_HOST=127.0.0.1`, `MAIL_PORT=<smtp host port>`, `MAIL_USERNAME=null`, `MAIL_PASSWORD=null`, `MAIL_ENCRYPTION=null`; optional UI URL in comments or `MAILPIT_UI_PORT` |
| meilisearch| `MEILISEARCH_HOST=http://127.0.0.1:<host>`, key vars if used |
| minio/S3   | `AWS_ENDPOINT=http://127.0.0.1:<api host>`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`, `AWS_BUCKET`, path-style flags as required by the app |

Use `127.0.0.1` (or `localhost`) for host-side app processes. Do **not** use Docker service DNS names in `.env` unless the **PHP/app process itself runs inside the Compose network**.

### Other stacks

Apply the same idea:

- **Node / Nest / Next**: `.env`, `.env.local`
- **Django**: `.env`, `settings` env module
- **Rails**: `.env`, `config/database.yml` only if the project already uses non-ENV port config
- **Generic**: any `*.env*`, `config/local.*`, or documented docker port files

Search the codebase for existing `DB_PORT`, `REDIS_PORT`, `localhost:5432`, etc., and update consistently. Prefer a single source of truth (env vars) over scattering literals.

### What not to change

- Production / CI deploy configs unless the user asks.
- Committed secrets or vault material.
- Ports that are intentionally fixed for a documented public interface.

## Existing Compose files

When a compose file already exists:

1. Read it fully (includes, profiles, overrides).
2. Preserve images, command, healthchecks, and env that still apply.
3. Reassign **host** ports into the high random range (unless the user asks to keep a specific host port).
4. Normalize `name`, `networks.default.name`, and volume `name:` fields.
5. Re-wire project config to the new host ports.
6. Warn if renaming volumes will detach existing data (`docker volume ls`). Offer to keep old volume names when data preservation matters, or document a one-time migration.

## Workflow checklist

1. Confirm services + project name.
2. Detect framework and config files.
3. Read existing Compose and `.env` if present.
4. Allocate unique free high host ports.
5. Write / update Compose with normalized network and volume names.
6. Write / update app config with `127.0.0.1` + new ports and credentials.
7. Show a concise summary table: service → image → host port(s) → config keys touched.
8. Suggest next commands only; do not start containers unless asked:

```bash
docker compose -f compose.yaml up -d
```

9. If credentials or volume renames can lose data, call that out explicitly before `up`.

## Output summary format

Always finish with a short table and file list, for example:

| Service  | Host binding              | Config |
|----------|---------------------------|--------|
| postgres | `127.0.0.1:52341` → 5432  | `DB_PORT=52341` in `.env` |
| redis    | `127.0.0.1:60102` → 6379  | `REDIS_PORT=60102` in `.env` |

Network: `myapp`  
Volumes: `postgres`, `redis`  
Files: `compose.yaml`, `.env`

## Constraints

- Do not expose Compose services on `0.0.0.0` by default.
- Do not use low/fixed host ports for multi-project hosts.
- Do not name networks or volumes with a redundant `project_` prefix.
- Do not put the app on the Compose network by default unless the user wants a full containerized app (Sail-style); this skill targets **host-run apps + containerized dependencies**.
- Prefer editing existing files over adding parallel compose variants (`docker-compose.dev.yml`, etc.) unless the repo already uses that pattern.
