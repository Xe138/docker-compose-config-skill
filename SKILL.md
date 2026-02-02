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
  - path_regex: ".*"
    encrypted_regex: "^(.*PASSWORD.*|.*SECRET.*|.*KEY.*|.*PASS.*)$"
    age: age1abc...
```

- `path_regex: ".*"` matches any file to avoid "no matching creation rules" errors
- `encrypted_regex` specifies which keys to encrypt (only secrets)
- Get recipient keys from `~/repos/sops-age-keys/recipients/*.pub`
- Comments and non-matching keys remain unencrypted

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
decrypt:
	@sops decrypt --input-type dotenv --output-type dotenv --output .env env.sops.yaml
	@echo "Decrypted env.sops.yaml -> .env"

encrypt:
	@sops encrypt --input-type dotenv --output env.sops.yaml .env
	@echo "Encrypted .env -> env.sops.yaml"

edit:
	@sops --input-type dotenv --output-type dotenv env.sops.yaml

clean:
	@rm -f .env
	@echo "Removed .env"
```

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

## Creating a New Project

When creating a new Docker Compose project:

1. Create the project directory structure
2. Create `docker-compose.yml` with service definitions
3. Create `env.example` as a template
4. Create environment config **only for the current host** (`environments/<current-hostname>/.env`)
5. Optionally create a `README.md` with first-run instructions

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
