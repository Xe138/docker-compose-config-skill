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
│   └── environments/
│       └── <hostname>/                 # Per-host configs
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
