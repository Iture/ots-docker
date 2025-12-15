# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Docker Compose infrastructure setup** for [OpenTAKServer](https://github.com/brian7704/OpenTAKServer) (OTS), an open-source TAK (Team Awareness Kit) server for tactical applications.

**Upstream Project**: OpenTAKServer is designed to be a lightweight, easy-to-install TAK server alternative compatible with ATAK, iTAK, WinTAK, TAKAware, TAKX, and CloudTAK clients. Written primarily in Python (GPL-3.0 licensed).

**This Repository**: Provides containerized deployment of OTS with all supporting services (database, messaging, web UI, video streaming).

**Status**: Not ready for production (per README).

### What is TAK?

**TAK = Team Awareness Kit** - a tactical data sharing framework. OpenTAKServer enables:
- Real-time location sharing among clients
- Message, point, route, image exchange
- Data package management and alerts
- CasEvac functionality
- Video streaming support
- ATAK plugin update server
- Client certificate enrollment with SSL encryption

## Core Stack

| Service | Role | Technology |
|---------|------|-----------|
| OTS Server | Main application | Python-based TAK server |
| PostgreSQL | Persistent data | PostGIS extension for geospatial data |
| RabbitMQ | Message broker | Internal messaging + MQTT bridge |
| Nginx | Web proxy | HTTP(S) reverse proxy (default) |
| OTS WebUI | Management interface | Web UI for OTS administration |
| MediaMTX | Video streaming | Optional video streaming service |

## Architecture

### Service Organization

**Main Server** (`ots`):
- Core TAK server running on internal port 8081
- Depends on: RabbitMQ, PostgreSQL
- Configuration: environment variables with `DOCKER_` prefix or `persistent/ots/config.yml`
- Volumes: `./persistent/ots:/app/ots`
- Pre-built image: `ghcr.io/milsimdk/ots-docker-image:latest`

**Helper Services** (same base image as OTS):
- `ots-cot_parser` - processes CoT (Cursor on Target) messages
- `ots-eud_handler` - TCP CoT streaming (port 8088, unencrypted)
- `ots-eud_handler-ssl` - SSL/TLS CoT streaming (port 8089, encrypted)

**Web Services**:
- `nginx-proxy` - custom-built reverse proxy from `persistent/nginx/Dockerfile`
  - Proxies HTTP(S) traffic to OTS port 8081
  - Handles certificate management
- `ots-webui` - web UI management interface (pre-built image)

**Data Services**:
- `ots-db` - PostgreSQL with PostGIS, built from `persistent/db/Dockerfile`
  - Stores all persistent data
  - PostGIS initialization scripts in `persistent/db/`
- `rabbitmq` - MQTT-enabled message broker
  - Internal OTS messaging
  - MQTT bridge for external clients (Meshtastic, etc.)

### Network Ports & Access

| Port | Protocol | Purpose | Endpoint |
|------|----------|---------|----------|
| 80, 443 | HTTP(S) | WebUI & REST API | https://localhost |
| 8080, 8443 | HTTP(S) | TAK API (proxied) | Internal proxy to :8081 |
| 8446 | HTTPS | Certificate Enrollment | TAK client cert enrollment |
| 8088 | TCP | CoT Streaming (plain) | Real-time CoT messages (unencrypted) |
| 8089 | TCP+TLS | CoT Streaming (SSL) | Real-time CoT messages (encrypted) |
| 8883 | MQTT/TLS | MQTT messaging | Message broker for external clients |

### Volume Persistence

All data persists in `./persistent/` directory (survives `make down`):

```
persistent/
├── ots/                    # OTS application data
│   ├── config.yml          # Configuration file
│   ├── ca/                 # SSL certificates & CA
│   ├── mediamtx/           # MediaMTX config
│   └── logs/               # Application logs
├── db/                     # PostgreSQL data
├── rabbitmq/               # RabbitMQ config & queues
├── nginx/                  # Nginx configs & Dockerfile
└── traefik/                # Traefik config (if using compose.traefik.yaml)
```

## Common Commands

All commands use Docker Compose via the `Makefile`:

```bash
make                       # Show all available commands
make up                    # Start all services (detached mode)
make stop                  # Stop running services (containers remain)
make down                  # Remove all containers (data persists)
make pull                  # Pull latest container images
make restart               # Restart OTS container only
make logs                  # Follow logs from all services (last 100 lines)
```

**Development/Testing**:
```bash
make dev-clean             # DELETE all OTS and database files (for fresh start)
make dev-test-ssl          # Test SSL endpoint: curl -k https://localhost:8089/Marti/
make push                  # Git add/commit with "Work-in-progress" message
```

## Configuration

### Environment Variables

Set in `compose.override.yaml` (create from `compose.override.yaml-example`).

All variables passed to OTS **must** have `DOCKER_` prefix to be recognized. Examples:

```yaml
DOCKER_SQLALCHEMY_DATABASE_URI: "postgresql+psycopg://ots:password@ots-db/ots"
POSTGRESQL_PASSWORD: "your-secure-password"
```

### Configuration Files

| File | Purpose |
|------|---------|
| `compose.yaml` | Primary Docker Compose (Nginx proxy) |
| `compose.traefik.yaml` | Alternative setup using Traefik reverse proxy |
| `persistent/ots/config.yml` | OTS runtime configuration (auto-generated) |
| `persistent/nginx/templates/` | Nginx configuration templates |
| `persistent/nginx/Dockerfile` | Custom Nginx build with certificates |
| `persistent/db/Dockerfile` | PostgreSQL + PostGIS setup |
| `persistent/db/initdb-postgis.sh` | PostGIS extension initialization |
| `persistent/rabbitmq/99-opentakserver.conf` | RabbitMQ broker config |

### User ID/Group ID

If you get permission denied errors on `persistent/` directory:

```bash
id                                    # Get your UID:GID
cp compose.override.yaml-example compose.override.yaml
# Edit compose.override.yaml line 4 to set user: "YOUR_UID:YOUR_GID"
```

The OTS container runs as this user to ensure file permissions align with your system.

### Environment-Based Configuration (New in v2.0)

**All configuration is now manageable via environment variables** - no need to manually edit config files!

#### Quick Start

```bash
# Copy the environment template
cp .env.example .env

# Edit with your preferences
nano .env

# Start with custom configuration
make up
```

#### Configuration Priority

Docker Compose reads configuration in this order (later overrides earlier):

1. **Defaults in compose.yaml** (e.g., `${VAR:-default}`)
2. **.env file** (automatically loaded)
3. **compose.override.yaml** (service-specific overrides)
4. **Shell environment** (highest priority)

#### Image Version Management

Pin specific container versions to prevent unexpected updates:

```bash
# .env
OTS_IMAGE_TAG=v1.7.0
OTS_WEBUI_IMAGE_TAG=v1.5.0
RABBITMQ_IMAGE_TAG=3.13-management
MEDIAMTX_IMAGE_TAG=1.13.0-ffmpeg
```

Verify pinned versions:
```bash
docker-compose ps --format "table {{.Service}}\t{{.Image}}"
```

#### Data Location Configuration

Move persistent data to dedicated storage (SSD, NAS, etc.):

```bash
# .env
DB_DATA_PATH=/mnt/fast-ssd/ots-database
OTS_DATA_PATH=/opt/opentakserver/data
RABBITMQ_CONFIG_PATH=/opt/rabbitmq-config
```

Create directories before first start:
```bash
mkdir -p /mnt/fast-ssd/ots-database /opt/opentakserver/data
```

#### OTS Configuration Parameters

All 100+ OpenTAKServer parameters are available via environment variables with the `DOCKER_` prefix:

```bash
# .env
DOCKER_OTS_CA_NAME=MyOrganization-CA
DOCKER_OTS_CA_PASSWORD=secure-password
DOCKER_OTS_LISTENER_PORT=8081
DOCKER_OTS_MEDIAMTX_ENABLE=True
DOCKER_OTS_ENABLE_PLUGINS=True
DOCKER_OTS_ENABLE_EMAIL=False
DOCKER_OTS_ENABLE_LDAP=False
```

**Available parameters** (organized by category):
- **Network & Ports**: `OTS_LISTENER_*`, `OTS_MARTI_*`, `OTS_TCP_STREAMING_*`, `OTS_SSL_STREAMING_*`
- **Security**: `OTS_SSL_*`, `OTS_CA_*`, `SECRET_KEY`, `DEBUG`
- **RabbitMQ**: `OTS_RABBITMQ_*`
- **MediaMTX**: `OTS_MEDIAMTX_*`
- **Email**: `OTS_ENABLE_EMAIL`, `MAIL_*`, `OTS_EMAIL_*`
- **LDAP**: `OTS_ENABLE_LDAP`, `LDAP_*`, `OTS_LDAP_*`
- **ADS-B**: `OTS_AIRPLANES_*`, `OTS_ADSB_*`
- **AIS**: `OTS_AISHUB_*`, `OTS_AIS_*`
- **Meshtastic**: `OTS_ENABLE_MESHTASTIC`, `OTS_MESHTASTIC_*`
- **Plugins**: `OTS_ENABLE_PLUGINS`, `OTS_PLUGIN_*`

See `.env.example` for complete list with descriptions.

#### Environment Variable Examples

```bash
# Production: Pinned versions, external database, security hardening
OTS_IMAGE_TAG=v1.7.0
DB_DATA_PATH=/mnt/ssd/ots-db
POSTGRESQL_PASSWORD=secure-password-here
DOCKER_OTS_CA_NAME=YourOrganization-CA
DOCKER_DEBUG=False

# Development: Latest versions, debug enabled
OTS_IMAGE_TAG=latest
DOCKER_DEBUG=True
DOCKER_OTS_MEDIAMTX_ENABLE=False

# With LDAP: Directory-based authentication
DOCKER_OTS_ENABLE_LDAP=True
DOCKER_LDAP_HOST=ldap.example.com
DOCKER_LDAP_BASE_DN=dc=example,dc=com
```

#### Migration from Previous Versions

Upgrading from v1.x? See [MIGRATION.md](./MIGRATION.md) for detailed migration steps.

**Short version**: Your existing deployment continues to work unchanged. `.env` is optional.

## Network Security

### Network Segmentation (Optional)

Version 3.0 introduces optional two-tier network isolation to enhance security by separating internet-facing services from internal database and message broker.

#### Network Modes

**Default Mode** (current behavior - backward compatible):
- All services on Docker's default bridge network
- No network isolation
- Simpler configuration, suitable for development/testing
- Enable: `NETWORK_SECURITY_MODE=default` (or omit the variable)

**Segmented Mode** (enhanced security for production):
- Two isolated networks: `external` and `internal`
- External network: nginx proxy, CoT handlers, MediaMTX video
- Internal network: OTS server, PostgreSQL, RabbitMQ, WebUI (protected)
- Database and broker inaccessible from internet directly
- Enable: `NETWORK_SECURITY_MODE=segmented`

#### Network Topology

```
INTERNET
   │
   ├─── Port 80,443 ──→ nginx-proxy ──→ WebUI (internal)
   ├─── Port 8080,8443 ─→ nginx-proxy ──→ OTS API (internal)
   ├─── Port 8088 ──→ ots-eud_handler ──→ RabbitMQ (internal)
   ├─── Port 8089 ──→ ots-eud_handler-ssl ──→ RabbitMQ (internal)
   └─── Port 8883 ──→ nginx-proxy (MQTT proxy) ──→ RabbitMQ (internal)

EXTERNAL NETWORK (172.20.0.0/24):
   - nginx-proxy (both external + internal)
   - ots-eud_handler (both external + internal)
   - ots-eud_handler-ssl (both external + internal)
   - mediamtx (both external + internal - video)

INTERNAL NETWORK (172.21.0.0/24):
   - ots (main server - both external + internal)
   - ots-cot_parser (message parser)
   - ots-webui (WebUI - proxied via nginx)
   - ots-db (PostgreSQL database) ← PROTECTED
   - rabbitmq (message broker) ← PROTECTED

DEBUG PORTS (optional, development only):
   - 5432 → PostgreSQL (pgAdmin, DBeaver, psql)
   - 5672 → RabbitMQ AMQP (testing)
   - 1883 → RabbitMQ MQTT (mosquitto)
   - 15672 → RabbitMQ Management UI (monitoring)
```

#### Enabling Network Segmentation

**Option 1: Via `.env` file**
```bash
# .env
NETWORK_SECURITY_MODE=segmented
```

**Option 2: Inline**
```bash
NETWORK_SECURITY_MODE=segmented docker compose up -d
```

**Restart services**:
```bash
make down && make up
```

#### Database and Broker Access

**Production (segmented mode)**:
- PostgreSQL: Only accessible from `ots` service (internal network)
- RabbitMQ: Only accessible from OTS and CoT handlers (internal network)
- MQTT: Exposed on port 8883 through nginx proxy (only to authenticated clients)

**Development (debug ports via compose.dev.yaml)**:
```bash
# Start with debug ports exposed on 0.0.0.0
docker compose -f compose.yaml -f compose.dev.yaml up -d

# Now accessible from anywhere on your network:
psql -h 192.168.1.100 -U ots -d ots        # PostgreSQL
mosquitto_pub -h 192.168.1.100 -p 1883 -t test -m "hello"  # MQTT
curl http://192.168.1.100:15672            # RabbitMQ UI
```

**Warning**: Debug ports expose services on all interfaces (0.0.0.0). Use only in development, not in production!

#### Verifying Network Isolation

```bash
# Check if networks are created (segmented mode)
docker network ls | grep ots

# Verify database is ONLY in internal network
docker network inspect ots-docker_external | grep ots-db
# Should return: (empty) - db is NOT in external network

# Verify nginx can reach internal services
docker exec nginx-proxy nc -zv ots 8081
# Should return: Connection succeeded
```

#### Security Benefits

| Scenario | Default Mode | Segmented Mode |
|----------|--------------|----------------|
| Internet exploit targets PostgreSQL | ✗ Accessible | ✓ Blocked (internal only) |
| Compromised CoT handler | ✗ Can access DB | ✓ Can access DB (intended) |
| Compromised WebUI | ✗ Can access anything | ✓ Can access anything (frontend) |
| Lateral movement | ✗ No boundaries | ✓ Network boundaries exist |
| Debug port exposure | ✗ Not available | ✓ Development only (opt-in) |

#### MediaMTX Port Mode

**Minimal mode** (recommended - production):
```bash
MEDIAMTX_PORT_MODE=minimal
# Exposes only: 8554 (RTSP), 8888 (HLS)
```

**Full mode** (all protocols):
```bash
MEDIAMTX_PORT_MODE=full
# Exposes: 1935 (RTMP), 1936 (RTMPS), 8000 (RTP), 8001 (RTCP),
#          8322 (RTSPS), 8554 (RTSP), 8888 (HLS), 8890 (SRT)
```

## Development Workflow

### Getting Started

```bash
git clone https://github.com/milsimdk/ots-docker.git && cd ots-docker

# Set up permissions and config
id                                       # Note your UID:GID
cp compose.override.yaml-example compose.override.yaml
# Edit UID:GID if needed

# Start all services
make up

# Check that services are healthy
make logs

# Access WebUI at https://localhost
```

### Viewing Logs

```bash
make logs                              # All services, 100 lines, follow mode
docker compose logs ots -f             # OTS service only, follow
docker compose logs ots --tail=50      # Last 50 lines
docker compose logs ots-db             # Database service
docker compose logs rabbitmq           # Message broker
```

### Restarting Services

```bash
make restart                           # Restart OTS service
docker compose restart ots-db          # Restart database
docker compose restart ots nginx-proxy # Restart multiple services
```

### Cleaning Up

```bash
make down                              # Stop and remove containers (data remains)
make dev-clean                         # Remove all OTS & database files for fresh start
docker compose down -v                 # Remove containers AND volumes (WARNING: data loss!)
make stop                              # Stop services without removing them
```

### Testing Connectivity

```bash
make dev-test-ssl                      # Test SSL connection to CoT endpoint (port 8089)
curl -k https://localhost/             # Test WebUI access (ignore SSL warnings locally)
curl -k https://localhost:8089/Marti/  # Direct test to CoT handler SSL endpoint
```

## Deployment Strategies

### Standard Setup (compose.yaml)

Uses Nginx as reverse proxy. Best for:
- Simple deployments
- Standard SSL certificate management
- Direct service access via well-known ports

Start with: `make up`

### Advanced Setup (compose.traefik.yaml)

Uses Traefik as reverse proxy. Best for:
- Multiple domain routing
- Advanced certificate management (Let's Encrypt)
- Docker-native configuration via labels
- Traefik dashboard monitoring (port 9000)

Start with: `docker compose -f compose.traefik.yaml up -d`

### Optional Services

**MediaMTX** (video streaming):
- Disabled by default in `compose.yaml`
- Profile-based: `docker compose --profile mediamtx up`
- Enabled in `compose.traefik.yaml`

**Mumble** (voice):
- Listed as "not supported yet" in README

## Relevant Files to Know

| File | Purpose |
|------|---------|
| `Makefile` | All development commands |
| `compose.yaml` | Primary Docker Compose (Nginx-based) |
| `compose.traefik.yaml` | Alternative with Traefik reverse proxy |
| `README.md` | User-facing documentation |
| `persistent/nginx/Dockerfile` | Custom Nginx reverse proxy build |
| `persistent/db/Dockerfile` | PostgreSQL + PostGIS customization |
| `persistent/db/initdb-postgis.sh` | PostGIS initialization hook |
| `.gitignore` | Files not tracked (env, db data, secrets) |

## Important Context

- **Upstream Repository**: [brian7704/OpenTAKServer](https://github.com/brian7704/OpenTAKServer) (main project)
- **OTS Docker Image**: [milsimdk/ots-docker-image](https://github.com/milsimdk/ots-docker-image) (pre-built)
- **WebUI Docker Image**: [milsimdk/ots-ui-docker-image](https://github.com/milsimdk/ots-ui-docker-image) (pre-built)
- **This Repository**: [milsimdk/ots-docker](https://github.com/milsimdk/ots-docker) (Docker Compose setup)

## Troubleshooting

| Problem | Cause | Solution |
|---------|-------|----------|
| Docker daemon permission denied | User not in docker group | `sudo usermod -aG docker $USER && newgrp docker` |
| Permission denied on `persistent/` | UID:GID mismatch | Adjust in `compose.override.yaml` |
| Services fail to start | Check image pulls, dependency order | `make logs` to see detailed errors |
| Port 80/443 already in use | Another service on those ports | Stop conflicting services or change ports in override |
| SSL certificate warnings | Local self-signed certs (expected) | Use `-k` flag with curl: `curl -k` |
| Database connection refused | PostGIS not initialized or DB unhealthy | Check `make logs` for ots-db; wait for health check |
| OTS not responding on :8081 | Service not healthy or dependencies not ready | Verify with `docker compose ps` and `make logs` |

## Related Resources

- [OpenTAKServer Documentation](https://github.com/brian7704/OpenTAKServer/wiki) - upstream project docs
- [TAK Protocol](https://www.civtak.org/) - TAK protocol specification (Civil TAK)
- [ATAK](https://atak.gov.mil/) - Android Tactical Assault Kit
- [WinTAK](https://www.civtak.org/) - Windows version of TAK client
