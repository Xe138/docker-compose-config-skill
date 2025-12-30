# Docker Compose Network Configuration

## Network Topology

```
┌─────────────────────────────────────────────────────────────┐
│                     reverse_proxy network                    │
│  (attachable, shared across all projects)                   │
│                                                             │
│  ┌─────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │  SWAG   │  │ immich-srv  │  │  grist-srv  │  ...       │
│  │ :443/80 │  │   :2283     │  │   :8484     │            │
│  └─────────┘  └─────────────┘  └─────────────┘            │
└─────────────────────────────────────────────────────────────┘
         │              │                │
         │              │                │
    External        Internal         Internal
    Traffic         Services         Services

┌───────────────────┐    ┌───────────────────┐
│   immich_default  │    │   grist_default   │
│                   │    │                   │
│  ┌─────────────┐  │    │  ┌─────────────┐  │
│  │ immich-ml   │  │    │  │   grist     │  │
│  └─────────────┘  │    │  └─────────────┘  │
│  ┌─────────────┐  │    │  ┌─────────────┐  │
│  │   redis     │  │    │  │  postgres   │  │
│  └─────────────┘  │    │  └─────────────┘  │
│  ┌─────────────┐  │    └───────────────────┘
│  │  postgres   │  │
│  └─────────────┘  │
└───────────────────┘
```

## Network Types

### 1. Project Default Network

Automatically created by Compose. Services within same project communicate here.

```yaml
services:
  app:
    networks:
      - default  # Implicit, connects to <project>_default

  database:
    networks:
      - default  # Same network, can reach 'app' by service name
```

### 2. Reverse Proxy Network

Shared external network for services needing reverse proxy access.

```yaml
services:
  web-app:
    networks:
      - default        # Internal communication
      - reverse_proxy  # Accessible by SWAG

networks:
  reverse_proxy:
    name: reverse_proxy
    attachable: true
```

### 3. Host Network Mode

For services requiring direct host network access (e.g., Plex discovery):

```yaml
services:
  plex:
    network_mode: host
    # No 'ports' mapping needed - uses host ports directly
```

## Network Configuration Patterns

### Standard Web Service

```yaml
services:
  myapp:
    networks:
      - default
      - reverse_proxy

networks:
  reverse_proxy:
    name: reverse_proxy
    attachable: true
```

### Internal-Only Service (Database)

```yaml
services:
  postgres:
    networks:
      - default
    # No reverse_proxy - not externally accessible
```

### Service with Multiple Networks

```yaml
services:
  api-gateway:
    networks:
      - default
      - reverse_proxy
      - monitoring

networks:
  reverse_proxy:
    name: reverse_proxy
    attachable: true
  monitoring:
    name: monitoring
    attachable: true
```

## Service Discovery

Services on the same network can reach each other by container name:

```yaml
services:
  app:
    container_name: myapp_server
    environment:
      REDIS_HOST: myapp_redis  # Uses container_name
      DB_HOST: myapp_postgres

  redis:
    container_name: myapp_redis

  database:
    container_name: myapp_postgres
```

## External Network Declaration

When connecting to pre-existing external networks:

```yaml
networks:
  reverse_proxy:
    external: true
    name: reverse_proxy
```

vs creating if not exists:

```yaml
networks:
  reverse_proxy:
    name: reverse_proxy
    attachable: true
```

## SWAG Proxy Configuration

Services on `reverse_proxy` network are accessible to SWAG at their container name and internal port:

| Service | Container Name | Internal URL |
|---------|---------------|--------------|
| Immich | immich_server | http://immich_server:2283 |
| Grist | grist | http://grist:8484 |

SWAG nginx config example:
```nginx
upstream_app immich_server;
upstream_port 2283;
```
