# Liquibase Toolkit

A Docker-based toolkit for three common Liquibase operations against **PostgreSQL** or **MySQL**:

| Operation | What it does |
|---|---|
| `generate` | Reverse-engineer a live database into a YAML changelog |
| `update` | Apply a changelog to a database, executing any pending changesets |
| `checksum` | Compute the checksum of a specific changeset — **no SQL executed** |

Everything runs inside Docker; no local Liquibase or JDK installation is required.

---

## How it works

1. A Docker image is built from `liquibase/liquibase:latest` with the MySQL JDBC connector added (PostgreSQL is bundled in the base image).
2. The container connects to your database and runs the requested Liquibase command.
3. Changelogs are read from and written to the `changelogs/` directory on your host.

---

## Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| [Docker](https://docs.docker.com/get-docker/) | 20+ | Must be running |
| [Task](https://taskfile.dev/installation/) | 3.28+ | Only for the Taskfile approach |

---

## Quick start

```bash
# 1. Build the image (once, or after any Dockerfile change)
task build

# 2. Extract schema from an existing database
task generate DB_USER=myuser DB_PASSWORD=mypass DB_NAME=mydb

# 3. Apply that changelog to another database
task update DB_USER=myuser DB_PASSWORD=mypass DB_NAME=mydb

# 4. Verify a changeset has not been altered
task checksum \
  CHANGESET_ID="changelog.yaml::1234567890-1::liquibase" \
  DB_USER=myuser DB_PASSWORD=mypass DB_NAME=mydb
```

---

## Option 1 — Taskfile

### Build the image

```bash
task build
```

### `task generate` — reverse-engineer a database

Connects to a live database and writes its full schema to `changelogs/changelog.yaml`.

```bash
task generate [VARIABLES]
```

| Variable | Required | Default | Description |
|---|---|---|---|
| `DB_USER` | Yes | — | Database username |
| `DB_PASSWORD` | Yes | — | Database password |
| `DB_NAME` | Yes | — | Database / catalog name |
| `DB_TYPE` | No | `postgres` | `postgres` or `mysql` |
| `DB_HOST` | No | `localhost` | Hostname or IP |
| `DB_PORT` | No | `5432` / `3306` | Port (defaults by `DB_TYPE`) |
| `DB_SCHEMA` | No | `public` | Schema name (PostgreSQL only) |
| `OUTPUT_FILE` | No | `changelog.yaml` | Output filename inside `changelogs/` |

**PostgreSQL — minimal**
```bash
task generate \
  DB_USER=admin \
  DB_PASSWORD=secret \
  DB_NAME=mydb
```

**PostgreSQL — custom host, port, and schema**
```bash
task generate \
  DB_TYPE=postgres \
  DB_HOST=10.0.0.5 \
  DB_PORT=5433 \
  DB_USER=app \
  DB_PASSWORD=secret \
  DB_NAME=production \
  DB_SCHEMA=app_schema
```

**MySQL**
```bash
task generate \
  DB_TYPE=mysql \
  DB_HOST=10.0.0.5 \
  DB_USER=root \
  DB_PASSWORD=secret \
  DB_NAME=mydb
```

**Timestamped output file**
```bash
task generate \
  DB_USER=admin \
  DB_PASSWORD=secret \
  DB_NAME=mydb \
  OUTPUT_FILE=baseline-$(date +%Y%m%d).yaml
```

---

### `task update` — apply a changelog to a database

Runs `liquibase update` against the target database. Liquibase tracks which changesets have already been applied in the `DATABASECHANGELOG` table, so only new changesets are executed.

```bash
task update [VARIABLES]
```

| Variable | Required | Default | Description |
|---|---|---|---|
| `DB_USER` | Yes | — | Database username |
| `DB_PASSWORD` | Yes | — | Database password |
| `DB_NAME` | Yes | — | Database / catalog name |
| `CHANGELOG_FILE` | No | `changelog.yaml` | Filename inside `changelogs/` to apply |
| `DB_TYPE` | No | `postgres` | `postgres` or `mysql` |
| `DB_HOST` | No | `localhost` | Hostname or IP |
| `DB_PORT` | No | `5432` / `3306` | Port (defaults by `DB_TYPE`) |
| `DB_SCHEMA` | No | `public` | Schema name (PostgreSQL only) |

```bash
# Apply the default changelog.yaml
task update \
  DB_USER=admin \
  DB_PASSWORD=secret \
  DB_NAME=mydb

# Apply a specific file to a remote MySQL database
task update \
  CHANGELOG_FILE=baseline-20240101.yaml \
  DB_TYPE=mysql \
  DB_HOST=10.0.0.5 \
  DB_USER=root \
  DB_PASSWORD=secret \
  DB_NAME=mydb
```

---

### `task checksum` — verify a changeset has not changed

Computes the checksum of a single changeset from a local changelog file and prints it. **No SQL is executed against the database.** This is useful for confirming that a changeset that has already been applied has not been modified after the fact.

```bash
task checksum [VARIABLES]
```

| Variable | Required | Default | Description |
|---|---|---|---|
| `CHANGESET_ID` | Yes | — | Identifier in `filepath::id::author` format |
| `DB_USER` | Yes | — | Database username |
| `DB_PASSWORD` | Yes | — | Database password |
| `DB_NAME` | Yes | — | Database / catalog name |
| `CHANGELOG_FILE` | No | `changelog.yaml` | Filename inside `changelogs/` |
| `DB_TYPE` | No | `postgres` | `postgres` or `mysql` |
| `DB_HOST` | No | `localhost` | Hostname or IP |
| `DB_PORT` | No | `5432` / `3306` | Port (defaults by `DB_TYPE`) |

#### Finding the `CHANGESET_ID`

Open your changelog file. The identifier is assembled from three fields in each `changeSet` block:

```yaml
databaseChangeLog:
- changeSet:
    id: 1234567890-1      # ← second segment
    author: liquibase      # ← third segment
    ...
```

The first segment is the path of the changelog file **as Liquibase sees it inside the container**, which is `/liquibase/changelog/<filename>` — but when passed to `--changeset-identifier` it is typically just the filename, e.g. `changelog.yaml`.

Full format: `changelog.yaml::1234567890-1::liquibase`

```bash
task checksum \
  CHANGESET_ID="changelog.yaml::1234567890-1::liquibase" \
  DB_USER=admin \
  DB_PASSWORD=secret \
  DB_NAME=mydb

# Against a custom file with a named changeset
task checksum \
  CHANGELOG_FILE=baseline.yaml \
  CHANGESET_ID="baseline.yaml::create-users-table::dev" \
  DB_USER=admin \
  DB_PASSWORD=secret \
  DB_NAME=mydb
```

---

### Connecting to a local database

Docker containers cannot reach `localhost` on the host directly. Use `host.docker.internal` as `DB_HOST` — `--add-host=host.docker.internal:host-gateway` is already wired into every `docker run` call, so this works on Linux and WSL2 without any extra setup.

```bash
task generate \
  DB_HOST=host.docker.internal \
  DB_USER=admin \
  DB_PASSWORD=secret \
  DB_NAME=mydb
```

---

## Option 2 — Docker Compose

Use this approach if you prefer not to install Task, or to integrate Liquibase into an existing Compose stack.

Three services are defined — one per operation:

| Service | Liquibase command |
|---|---|
| `liquibase-generate` | `generate-changelog` |
| `liquibase-update` | `update` |
| `liquibase-checksum` | `calculate-checksum` |

### 1. Create a `.env` file

All services share the same connection variables. Add operation-specific variables as needed.

**Shared (all operations)**
```env
DB_USER=myuser
DB_PASSWORD=mypassword
```

**PostgreSQL**
```env
JDBC_URL=jdbc:postgresql://host.docker.internal:5432/mydb
```

**MySQL**
```env
JDBC_URL=jdbc:mysql://host.docker.internal:3306/mydb
```

**`liquibase-generate` extras**
```env
DB_SCHEMA=public          # postgres only
OUTPUT_FILE=changelog.yaml
```

**`liquibase-update` extras**
```env
CHANGELOG_FILE=changelog.yaml
DB_SCHEMA=public          # postgres only
```

**`liquibase-checksum` extras**
```env
CHANGELOG_FILE=changelog.yaml
CHANGESET_ID=changelog.yaml::1234567890-1::liquibase
```

### 2. Build the image

```bash
docker compose build
```

### 3. Run the operation you need

```bash
# Reverse-engineer the database
docker compose run --rm liquibase-generate

# Apply a changelog
docker compose run --rm liquibase-update

# Check a changeset's integrity
docker compose run --rm liquibase-checksum
```

---

## Output

The `generate` operation produces a standard Liquibase YAML changelog:

```yaml
databaseChangeLog:
- changeSet:
    id: 1234567890-1
    author: liquibase
    changes:
    - createTable:
        tableName: users
        columns:
        - column:
            name: id
            type: BIGINT
            constraints:
              primaryKey: true
              nullable: false
        - column:
            name: email
            type: VARCHAR(255)
            constraints:
              nullable: false
```

The `checksum` operation prints the MD5 hash of the changeset as Liquibase computed it — compare this against the value stored in the `DATABASECHANGELOG` table (`MD5SUM` column) to confirm the changeset has not been modified.

---

## Project structure

```
liquibase/
├── Dockerfile          # Extends liquibase/liquibase with MySQL JDBC driver
├── Taskfile.yml        # Task runner — generate, update, checksum tasks
├── docker-compose.yml  # Compose-based alternative with three named services
├── changelogs/         # Changelogs are read from and written to here (git-ignored)
└── README.md
```

---

## Troubleshooting

**`Connection refused` or `Unable to connect`**

- Verify the database is running and accepting connections.
- If the database is on your local machine, use `DB_HOST=host.docker.internal` instead of `localhost`.
- Double-check the port and database name.

**`liquibase-reverse` image not found**

Run `task build` or `docker compose build` before any other command.

**Checksum mismatch warning from Liquibase**

Liquibase warns when the checksum in `DATABASECHANGELOG` differs from the one computed from the current file. This means the changeset was modified after it was applied. Use `task checksum` to compute the current value and compare it manually against the `MD5SUM` column in `DATABASECHANGELOG`.

**`Unknown database` / JDBC driver error**

Ensure `DB_TYPE` is exactly `postgres` or `mysql`. Any other value silently falls back to the PostgreSQL JDBC URL.

**Permission denied on `changelogs/`**

The directory is created automatically. If it has root ownership from a previous Docker run, fix it with:

```bash
sudo chown -R $(whoami) changelogs/
```
