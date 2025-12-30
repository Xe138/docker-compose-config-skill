# Docker Compose Configuration Examples

## Multi-Service Application (Immich-style)

Complete example with app server, machine learning, Redis, and PostgreSQL:

```yaml
name: immich

services:
  immich-server:
    container_name: immich_server
    image: ghcr.io/immich-app/immich-server:v2.3.1
    environment:
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      DB_DATABASE_NAME: ${DB_DATABASE_NAME}
      REDIS_HOSTNAME: immich_redis
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    ports:
      - ${APP_PORT:-2283}:2283
    depends_on:
      - redis
      - database
    restart: always
    networks:
      - default
      - reverse_proxy

  immich-machine-learning:
    container_name: immich_machine_learning
    image: ghcr.io/immich-app/immich-machine-learning:v2.3.1
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    restart: always
    networks:
      - default

  redis:
    container_name: immich_redis
    image: registry.redict.io/redict:7.3.6@sha256:2a99f322eed7...
    restart: always
    volumes:
      - redis-data:/data
    networks:
      - default

  database:
    container_name: immich_postgres
    image: ghcr.io/immich-app/postgres:16-vectorchord0.3.0@sha256:5b434f184ec...
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - ${DB_DATA_LOCATION}/data:/var/lib/postgresql/data
    restart: always
    networks:
      - default

volumes:
  model-cache:
  redis-data:

networks:
  reverse_proxy:
    name: reverse_proxy
    attachable: true
```

## Reverse Proxy Service (SWAG-style)

```yaml
services:
  swag:
    image: lscr.io/linuxserver/swag:5.2.2@sha256:c8afbd137c2f...
    container_name: swag
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=America/New_York
      - URL=${SWAG_URL}
      - VALIDATION=dns
      - SUBDOMAINS=wildcard
      - DNSPLUGIN=cloudflare
      - PROPAGATION=20
      - EMAIL=user@domain.com
      - ONLY_SUBDOMAINS=${ONLY_SUBDOMAINS:-false}
      - EXTRA_DOMAINS=${EXTRA_DOMAINS:-}
      - STAGING=false
    volumes:
      - ${SWAG_DATA_PATH}:/config
    ports:
      - 1443:443
      - 81:80
    restart: unless-stopped
    networks:
      - default
      - reverse_proxy

networks:
  reverse_proxy:
    name: reverse_proxy
    attachable: true
```

## Media Server with GPU (Jellyfin-style)

```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:10.11.4@sha256:234ea8d508b2...
    container_name: jellyfin
    runtime: nvidia
    environment:
      PUID: ${UID}
      PGID: ${GID}
      UMASK: "022"
      JELLYFIN_PublishedServerUrl: ${HOST_IP}
      NVIDIA_VISIBLE_DEVICES: all
    ports:
      - ${JELLYFIN_PORT:-8096}:8096
      - ${JELLYFIN_LOCAL_PORT:-7359}:7359
      - ${JELLYFIN_DNLA_PORT:-1900}:1900
    volumes:
      - ${MEDIA_PATH}:/data/media
      - ${BASE_SERVER_PATH}/jellyfin:/config
      - ${TRANSCODE_PATH}jellyfin:/config/cache/transcodes
    healthcheck:
      test: ["CMD", "curl", "-f", "http://127.0.0.1:8096/"]
      start_period: 5m
      interval: 30s
      timeout: 20s
      retries: 10
```

## Service with VPN (Deluge-style)

```yaml
services:
  deluge:
    image: binhex/arch-delugevpn:latest@sha256:2ff474cba3af...
    container_name: deluge
    cap_add:
      - NET_ADMIN
    environment:
      PUID: ${UID}
      PGID: ${GID}
      UMASK: "000"
      NAME_SERVERS: ${VPN_NAME_SERVERS}
      DEBUG: false
    env_file:
      - .env
    volumes:
      - ${SEEDBOX_PATH}:/data
      - ${BASE_SERVER_PATH}/deluge:/config
    ports:
      - ${DELUGE_PORT_1:-8112}:8112
      - ${DELUGE_PORT_2:-58846}:58846
```

## Database with Healthcheck (PostgreSQL-style)

```yaml
services:
  database:
    image: postgres:18.1@sha256:5ec39c188013...
    container_name: app-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:-appuser}
      POSTGRES_PASSWORD: ${DB_PASS}
    ports:
      - ${DB_PORT:-5432}:5432
    volumes:
      - ${BASE_SERVER_PATH}/app/data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d postgres -U $${DB_USER:-appuser}"]
      start_period: 20s
      interval: 30s
      retries: 5
      timeout: 5s
```

## Environment File Examples

### Main .env

```bash
# Permissions
UID=1000
GID=1000

# Paths
UPLOAD_LOCATION=/mnt/main/pictures
DB_DATA_LOCATION=/docker/appdata/immich

# Ports
APP_PORT=2283

# Secrets
DB_PASSWORD=GeneratedSecurePassword123

# Database
DB_HOSTNAME=database
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

### Host-specific .env (environments/mabel/.env)

```bash
# Permissions (same across hosts)
UID=1000
GID=1000

# Paths (host-specific mount points)
UPLOAD_LOCATION=/mnt/user/pictures
DB_DATA_LOCATION=/mnt/user/appdata/immich

# Ports (may vary by host)
APP_PORT=2283

# Secrets (same across hosts)
DB_PASSWORD=GeneratedSecurePassword123

# Database
DB_HOSTNAME=database
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```
