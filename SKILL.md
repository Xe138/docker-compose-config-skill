---
name: docker-compose-config
description: Docker Compose configuration management for multi-host server deployments. Use when creating, modifying, or managing docker-compose.yml files with environment-specific configurations, external volume mounts, and reverse proxy networks. Triggers on tasks involving Docker Compose files, environment overrides, multi-host deployments, or service configuration for self-hosted applications.
---

# Docker Compose Configuration Management

This skill provides guidance for managing Docker Compose configurations across multiple server environments with per-host overrides.

## Directory Structure

Each project follows this structure:

```
/docker/config/
├── <project>/
│   ├── docker-compose.yml              # Main service definitions
│   ├── docker-compose.override.yml     # Current host overrides (gitignored)
│   ├── .env                            # Environment variables (gitignored)
│   ├── .gitignore                      # Excludes /.env and /docker-compose.override.yml
│   ├── env.example                     # Template for .env files
│   ├── README.md                       # Setup and usage instructions
│   └── environments/
│       └── <hostname>/                 # Per-host configs (committed, secrets scrubbed)
│           ├── .env
│           └── docker-compose.override.<hostname>.yml
```

## Appdata Storage: One ZFS Dataset Per Stack (REQUIRED)

On ZFS-backed hosts (**inkling**, **shellington** — their `/docker/appdata` is a ZFS dataset), **every stack gets its own ZFS dataset** at `docker/appdata/<stack>`. Create it **before first boot**, then bind-mount subpaths of it into containers.

**Why:** each dataset is an independent snapshot/rollback/replication boundary. You can `zfs rollback`, restore, clone, or replicate **one stack without touching any other**. Plain subdirectories of the parent `docker/appdata` share a single snapshot timeline — you cannot roll back or restore one stack's data without rolling back **every** service's appdata at once. A per-stack dataset is what makes "restore just this app to last night" possible.

```bash
# REQUIRED first storage step for a new stack on a ZFS appdata host:
sudo zfs create docker/appdata/<stack>

# then create whatever subdirs the containers bind-mount, INSIDE the dataset:
sudo mkdir -p /docker/appdata/<stack>/{db,uploads}
```

The new dataset is backed up automatically: sanoid on `[docker/appdata]` is `recursive=true` (snapshots every child), and the `syncoid -r` replication to dipper picks up child datasets — **no backup-config change needed**.

**Verify** the data path is backed by the child dataset (not silently shadowing the parent):

```bash
df -T /docker/appdata/<stack>
# FSTYPE must be `zfs` and Filesystem must read `docker/appdata/<stack>` (not `docker/appdata`)
```

### Do NOT plain-`mkdir` an appdata dir on a ZFS host

A bare `mkdir /docker/appdata/<stack>` silently forfeits per-stack restore. These are **not** valid reasons to skip the dataset:

| Rationalization | Reality |
|-----------------|---------|
| "It matches the other ~70 containers / follows convention" | Several existing stacks are plain dirs by accident — they're migration candidates, not a pattern to copy. This rule *is* the convention now. |
| "The parent `docker/appdata` is already snapshotted" | Parent snapshots can't be rolled back per-stack — restoring one app would revert all of them. That defeats the entire purpose. |
| "A dataset adds one to the sanoid policy / more complexity" | Zero policy change: `recursive=true` already covers it. One `zfs create` is the whole cost. |
| "The examples use `/mnt/user/appdata` as plain dirs" | Those are **Unraid** (non-ZFS) examples — see the fallback below. They do not apply to ZFS hosts. |

If you catch yourself about to `mkdir` a top-level appdata dir on inkling/shellington, **stop and `zfs create` instead.**

### Non-ZFS appdata hosts (fallback)

If the host's appdata filesystem is **not** ZFS (e.g. Unraid `/mnt/user/appdata`), there is no dataset to create — `mkdir -p` the appdata dir as the examples show. Detect which case you're in:

```bash
df -T <appdata-root>   # e.g. df -T /docker/appdata  →  zfs ⇒ dataset required;  anything else ⇒ plain dir
```

### Optional: tune the database subtree

For a stack with a real database, a dedicated child dataset with `recordsize=16k` matches Postgres/MySQL page I/O (a real win vs. the 128k default):

```bash
sudo zfs create -o recordsize=16k docker/appdata/<stack>/db
```

## Compose File Conventions

### Service Definition Pattern

```yaml
name: <project-name>

services:
  <service-name>:
    container_name: <service_container>
    image: <registry>/<image>:<version>
    environment:
      PUID: ${UID}
      PGID: ${GID}
      # Service-specific vars use env substitution
      VAR_NAME: ${VAR_NAME}
    volumes:
      - ${DATA_PATH}:/app/data
      - /etc/localtime:/etc/localtime:ro
    ports:
      - ${SERVICE_PORT:-default}:internal_port
    restart: unless-stopped
    networks:
      - default
      - reverse_proxy

networks:
  reverse_proxy:
    name: reverse_proxy
    attachable: true
```

### Key Patterns

1. **Image versioning**: Always use pinned version tags (never `latest`)
   ```yaml
   image: ghcr.io/org/image:v1.0.0
   image: lscr.io/linuxserver/app:2.5.1
   ```
   Note: SHA256 digests are managed automatically by Renovate bot and should not be added manually.

2. **Port defaults**: Always provide defaults for ports
   ```yaml
   ports:
     - ${APP_PORT:-8080}:8080
   ```

3. **Volume mounts**: Use environment variables for paths
   ```yaml
   volumes:
     - ${UPLOAD_LOCATION}:/usr/src/app/upload
     - ${DB_DATA_LOCATION}/data:/var/lib/postgresql/data
   ```

4. **Proxy network**: Services needing reverse proxy access join `reverse_proxy` network
   ```yaml
   networks:
     - default
     - reverse_proxy

   networks:
     reverse_proxy:
       name: reverse_proxy
       attachable: true
   ```

## Environment Variable Isolation

**Never use `env_file` directives in service definitions.** Each service should only receive the specific environment variables it needs.

### Why
- **Security**: Prevents secrets meant for one service from leaking to others
- **Clarity**: Makes explicit which variables each service requires
- **Debugging**: Easier to trace environment-related issues

### Correct Pattern
```yaml
services:
  app:
    environment:
      - DATABASE_URL=postgres://${DB_USER}:${DB_PASS}@db:5432/${DB_NAME}
      - APP_SECRET=${APP_SECRET}
    # NO env_file directive

  database:
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
      POSTGRES_DB: ${DB_NAME}
    # NO env_file directive
```

### Incorrect Pattern
```yaml
services:
  app:
    env_file:
      - .env  # BAD: passes ALL variables to container
    environment:
      - EXTRA_VAR=value

  database:
    env_file:
      - .env  # BAD: database receives app secrets, TURN secrets, etc.
```

### How .env Files Work
The `.env` file serves **compose-time interpolation only**:
- Variables like `${DATA_PATH}` in volumes are substituted when `docker compose` parses the file
- This happens at compose parse time, NOT at container runtime
- The container only receives variables explicitly listed in `environment:`

## Environment File Conventions

### Structure (.env)

```bash
# Permissions
UID=1000
GID=1000

# Paths
UPLOAD_LOCATION=/mnt/user/pictures
DB_DATA_LOCATION=/mnt/user/appdata/<project>

# Ports
APP_PORT=8080

# Secrets
DB_PASSWORD=<generated>

# Database
DB_HOSTNAME=database
DB_USERNAME=postgres
DB_DATABASE_NAME=<project>
```

### Per-Host Variations

- `environments/<hostname>/.env` overrides paths for specific hosts
- Common overrides: mount paths (`/mnt/user/` vs `/mnt/main/`), ports

## Secret Management with SOPS

Projects use SOPS with age encryption to securely commit secrets to git. This replaces the manual "secret scrubbing" approach.

### Directory Structure (with SOPS)

```
/docker/config/<project>/
├── docker-compose.yml
├── docker-compose.override.yml     # (gitignored)
├── .env                            # (gitignored, generated by decrypt)
├── .gitignore
├── .sops.yaml                      # SOPS encryption config
├── Makefile                        # encrypt/decrypt targets
├── env.sops.yaml                   # Encrypted env vars (committed)
└── environments/
    └── <hostname>/
        ├── env.sops.yaml           # Encrypted, host-specific (committed)
        └── .env                    # (gitignored, generated)
```

### Key Repository

Public keys are managed centrally at `~/repos/sops-age-keys/`:
- `recipients/*.pub` - Public keys for each host
- `scripts/init-host.sh` - Generate key for new host
- `scripts/add-recipient.sh` - Add host's public key
- `templates/Makefile` - Makefile template for projects

Private keys stay at `~/.config/sops/age/keys.txt` (never committed).

### .sops.yaml Configuration

```yaml
creation_rules:
  - path_regex: .*\.env$|.*env\.sops\.yaml$
    encrypted_regex: "^(.*_PASSWORD.*|.*_SECRET.*|.*_TOKEN.*|.*_KEY.*|SECRET_.*)$"
    age: age1abc...
```

**Configuration notes:**

- **`path_regex`**: Must match the INPUT file when encrypting. The pattern `.*\.env$|.*env\.sops\.yaml$` matches both `.env` files (input) and `env.sops.yaml` files (output/edit).
- **`encrypted_regex`**: Specifies which keys to encrypt. Uses `.*_TOKEN.*` (not `.*_TOKEN$`) to match fields CONTAINING `_TOKEN` like `PVE_TOKEN_VALUE`.
- Get recipient keys from `~/repos/sops-age-keys/recipients/*.pub`
- Comments and non-matching keys remain unencrypted

**Common field patterns matched by encrypted_regex:**

| Pattern | Matches |
|---------|---------|
| `.*_PASSWORD.*` | `DB_PASSWORD`, `ADMIN_PASSWORD_HASH` |
| `.*_SECRET.*` | `APP_SECRET`, `JWT_SECRET_KEY`, `SECRET_VALUE` |
| `.*_TOKEN.*` | `API_TOKEN`, `PVE_TOKEN_VALUE`, `AUTH_TOKEN_SECRET` |
| `.*_KEY.*` | `API_KEY`, `ENCRYPTION_KEY`, `SSH_KEY_PATH` |
| `SECRET_.*` | `SECRET_KEY`, `SECRET_TOKEN` |

### env.sops.yaml Format

Use normal variable names. Only keys matching `encrypted_regex` are encrypted:

```yaml
# Permissions
UID: "1000"
GID: "1000"

# Paths
UPLOAD_LOCATION: /mnt/user/pictures
DB_DATA_LOCATION: /mnt/user/appdata/immich

# Ports
APP_PORT: "2283"

# Database
DB_HOSTNAME: database
DB_USERNAME: postgres
DB_DATABASE_NAME: immich
DB_PASSWORD: secret_value_here
```

After encryption, only `DB_PASSWORD` is encrypted; comments and other values remain readable.

### Makefile Targets

```makefile
# Default paths (can be overridden: make encrypt SOURCE=env.sops.yaml TARGET=.env)
SOURCE ?= env.sops.yaml
TARGET ?= .env

decrypt:
	@sops decrypt --input-type yaml --output-type dotenv --output $(TARGET) $(SOURCE)
	@echo "Decrypted $(SOURCE) -> $(TARGET)"

encrypt:
	@sops encrypt --input-type dotenv --output-type yaml --output $(SOURCE) $(TARGET)
	@echo "Encrypted $(TARGET) -> $(SOURCE)"

edit:
	@sops $(SOURCE)

clean:
	@rm -f .env
	@echo "Removed .env"
```

**Important**: Use `--output-type yaml` for encryption to produce readable YAML files. Use `--output-type dotenv` for decryption to produce shell-compatible `.env` files.

### Common SOPS Commands

```bash
# Copy encrypted file from environment to project root
cp environments/<host>/env.sops.yaml env.sops.yaml

# Decrypt to .env for editing
make decrypt

# Edit the .env file
nano .env

# Re-encrypt after changes
make encrypt

# Copy back to environment
cp env.sops.yaml environments/<host>/env.sops.yaml

# Clean up
make clean

# Edit secrets directly (decrypts in editor, re-encrypts on save)
make edit

# Add new host to recipients (after adding to .sops.yaml)
sops updatekeys env.sops.yaml
```

### Starting Services with SOPS

```bash
cd /docker/config/<project>

# Copy and decrypt secrets to project root
cp environments/<host>/env.sops.yaml env.sops.yaml
make decrypt

# Start services
docker compose -f docker-compose.yml \
  -f environments/<host>/docker-compose.override.<host>.yml up -d

# Clean up (optional)
make clean && rm env.sops.yaml
```

### SOPS Troubleshooting

**"no matching creation rules found"**

This error occurs when `path_regex` doesn't match the INPUT file path. SOPS matches against the file being read, not the output file.

```bash
# This fails if path_regex only matches *.sops.yaml
sops encrypt --input-type dotenv --output env.sops.yaml .env
# SOPS tries to match ".env" against path_regex, not "env.sops.yaml"
```

**Solution**: Use a path_regex that matches both input and output files:
```yaml
path_regex: .*\.env$|.*env\.sops\.yaml$
```

**Fields not being encrypted**

If secrets aren't being encrypted, check that `encrypted_regex` matches the field names. Common issues:

| Field | Pattern needed | Why |
|-------|---------------|-----|
| `PVE_TOKEN_VALUE` | `.*_TOKEN.*` | Contains `_TOKEN`, doesn't end with it |
| `API_KEY_ID` | `.*_KEY.*` | Contains `_KEY` in middle |
| `PASSWORD_HASH` | `.*PASSWORD.*` | `PASSWORD` in middle of name |

**Wrong output format**

If `env.sops.yaml` contains dotenv format instead of YAML:
```bash
# Wrong - produces dotenv format
sops encrypt --input-type dotenv --output env.sops.yaml .env

# Correct - produces YAML format
sops encrypt --input-type dotenv --output-type yaml --output env.sops.yaml .env
```

**Decryption produces wrong format**

If `.env` contains YAML instead of dotenv format:
```bash
# Correct - produces dotenv format for shell compatibility
sops decrypt --input-type yaml --output-type dotenv --output .env env.sops.yaml
```

### Migrating Existing Projects

See `~/repos/sops-age-keys/README.md` for full migration guide, or `~/repos/sops-age-keys/docs/design.md` for architecture details.

## Common Commands

```bash
# Start a project
docker compose -f <project>/docker-compose.yml up -d

# Start with host-specific override
docker compose -f <project>/docker-compose.yml \
  -f <project>/environments/<host>/docker-compose.override.<host>.yml up -d

# View logs
docker compose -f <project>/docker-compose.yml logs -f

# Stop services
docker compose -f <project>/docker-compose.yml down
```

## Deploying Updates from Merged PRs

After merging PRs that update image versions (e.g., Renovate bot PRs), deploy the changes to the target host. To review and merge Renovate PRs, use the gitea skill's PR management functions (`gitea_list_renovate_prs`, `gitea_merge_pr`).

### Pull and Redeploy a Stack

```bash
# On the target host, pull latest config and redeploy
ssh bill@<host> "cd /docker/config/<stack-name> && git pull origin main"

# Pull new images and restart
ssh bill@<host> "cd /docker/config/<stack-name> && \
  docker compose pull && \
  docker compose up -d"

# For stacks with host-specific overrides:
ssh bill@<host> "cd /docker/config/<stack-name> && \
  docker compose -f docker-compose.yml \
  -f environments/<host>/docker-compose.override.<host>.yml \
  pull && \
  docker compose -f docker-compose.yml \
  -f environments/<host>/docker-compose.override.<host>.yml \
  up -d"
```

### Verify Deployment

```bash
# Check container is running with new image
ssh bill@<host> "docker ps --filter name=<container> --format '{{.Image}} {{.Status}}'"

# Check logs for startup errors
ssh bill@<host> "docker logs --tail 20 <container>"
```

### Host Reference

| Host | SSH | Stack Path |
|------|-----|------------|
| inkling | `bill@inkling` | `/docker/config/<stack>/` |
| shellington | `bill@shellington` | `/docker/config/<stack>/` |

## Creating a New Project

When creating a new Docker Compose project:

1. **Provision storage:** on a ZFS appdata host, `sudo zfs create docker/appdata/<stack>` (see [Appdata Storage: One ZFS Dataset Per Stack](#appdata-storage-one-zfs-dataset-per-stack-required)), then `mkdir` the bind-mount subdirs inside it. On a non-ZFS host, `mkdir -p` the appdata dir.
2. Create the project directory structure under `/docker/config/<stack>/`
3. Create `docker-compose.yml` with service definitions (appdata bind mounts point at paths inside the dataset)
4. Create `env.example` as a template
5. Create environment config **only for the current host** (`environments/<current-hostname>/.env`)
6. Optionally create a `README.md` with first-run instructions

**Important**: Only create the environment for the current host unless explicitly asked to create configurations for other hosts.

## Adding a New Host Environment

1. Create `environments/<hostname>/` directory
2. Copy existing host's `.env` as template
3. Update paths and port mappings for the host
4. Create override compose file if device mappings differ

## Git Repository Management

Each project should be version controlled with its own git repository.

### .gitignore Configuration

```gitignore
# Root environment file (generated, may contain secrets)
/.env

# Docker compose override (host-specific, not committed)
/docker-compose.override.yml

# Per-host decrypted .env files (generated from env.sops.yaml)
/environments/*/.env
```

**Note**: All `.env` files are gitignored. Secrets are stored encrypted in `env.sops.yaml` files using SOPS (see "Secret Management with SOPS" section above).

### Creating a Git Repository

```bash
# Initialize
cd /docker/config/<project>
git init && git branch -m main

# Create remote on Gitea (using git-gitea skill)
source ~/.claude/skills/git-gitea/scripts/gitea-helper.sh
gitea_create_repo "docker-<project>" "Docker Compose configuration for <project>" true

# Add remote, commit, push
git remote add origin https://git.prettyhefty.com/Bill/docker-<project>.git
git add -A
git commit -m "Initial commit: <project> docker-compose configuration"
git push -u origin main
```

## Service Dependencies

Use `depends_on` with health checks for proper startup order:

```yaml
depends_on:
  database:
    condition: service_healthy
```

## Healthchecks

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://127.0.0.1:8080/"]
  start_period: 5m
  interval: 30s
  timeout: 20s
  retries: 10
```

## References

- See [examples.md](references/examples.md) for complete service configurations
- See [networks.md](references/networks.md) for network topology details
