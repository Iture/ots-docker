# OpenTAKServer Docker Configuration Enhancement Plan

**Status**: Approved ✓
**Last Updated**: 2025-12-15
**Reference**: `/Users/kapturm/.claude/plans/bright-sparking-kettle.md`

## Objective

Expand Docker Compose configuration to support:
1. **Configurable data locations** - All volume paths via environment variables
2. **Image version management** - Pin container versions via environment variables
3. **Comprehensive OTS configuration** - Complete .env.example with 100+ parameters

---

## Implementation Overview

### 1. Environment Variable Naming Conventions

**Image Tags:**
```bash
OTS_IMAGE_TAG=latest
OTS_WEBUI_IMAGE_TAG=latest
RABBITMQ_IMAGE_TAG=latest
MEDIAMTX_IMAGE_TAG=1.13.0-ffmpeg
TRAEFIK_IMAGE_TAG=v3.5
NGINX_BASE_IMAGE_TAG=mainline-bookworm
POSTGRES_BASE_IMAGE_TAG=18-trixie
```

**Volume Paths:**
```bash
OTS_DATA_PATH=./persistent/ots
OTS_CA_PATH=./persistent/ots/ca
DB_DATA_PATH=./persistent/db
RABBITMQ_CONFIG_PATH=./persistent/rabbitmq
RABBITMQ_DATA_PATH=./persistent/rabbitmq/data
NGINX_TEMPLATES_PATH=./persistent/nginx/templates
MEDIAMTX_CONFIG_PATH=./persistent/ots/mediamtx
TRAEFIK_CONFIG_PATH=./persistent/traefik
```

**OTS Configuration:**
- All parameters use `DOCKER_` prefix (existing convention)
- Example: `DOCKER_OTS_CA_NAME`, `DOCKER_OTS_LISTENER_PORT`

---

## Files to Modify

### A. compose.yaml (~162 lines)

**Changes:**
- Replace image tags with `${VAR:-default}` syntax:
  - Line 8: `ghcr.io/milsimdk/ots-docker-image:${OTS_IMAGE_TAG:-latest}`
  - Line 31: Same (cot_parser uses YAML anchor)
  - Line 90: `ghcr.io/milsimdk/ots-ui-docker-image:${OTS_WEBUI_IMAGE_TAG:-latest}`
  - Line 120: `rabbitmq:${RABBITMQ_IMAGE_TAG:-latest}`
  - Line 139: `bluenviron/mediamtx:${MEDIAMTX_IMAGE_TAG:-1.13.0-ffmpeg}`

- Replace volume paths with variables:
  - Line 18: `"${OTS_DATA_PATH:-./persistent/ots}:/app/ots"`
  - Line 41: Same (cot_parser)
  - Line 84: `"${OTS_CA_PATH:-./persistent/ots/ca}:/app/ots/ca:ro"`
  - Line 85: `"${NGINX_TEMPLATES_PATH:-./persistent/nginx/templates}:/etc/nginx/templates:ro"`
  - Line 112: `"${DB_DATA_PATH:-./persistent/db}:/var/lib/postgresql"`
  - Lines 125-126: RabbitMQ config files using `${RABBITMQ_CONFIG_PATH:-...}`
  - Lines 157-158: MediaMTX using `${OTS_DATA_PATH:-...}` and `${MEDIAMTX_CONFIG_PATH:-...}`

- Add build args for local images:
  ```yaml
  nginx-proxy:
    build:
      context: .
      dockerfile: persistent/nginx/Dockerfile
      args:
        NGINX_BASE_IMAGE_TAG: ${NGINX_BASE_IMAGE_TAG:-mainline-bookworm}

  ots-db:
    build:
      context: .
      dockerfile: persistent/db/Dockerfile
      args:
        POSTGRES_BASE_IMAGE_TAG: ${POSTGRES_BASE_IMAGE_TAG:-18-trixie}
  ```

### B. compose.traefik.yaml (~183 lines)

Apply same changes as compose.yaml:
- Image tag variables (lines 4, 36, 82, 97, 128, 138)
- Volume path variables (lines 24-25, 76, 92, 122)
- Build args if using local builds

### C. persistent/nginx/Dockerfile (~11 lines)

```dockerfile
ARG NGINX_BASE_IMAGE_TAG=mainline-bookworm
FROM nginx:${NGINX_BASE_IMAGE_TAG}
# ... rest unchanged
```

### D. persistent/db/Dockerfile (~25 lines)

```dockerfile
ARG POSTGRES_BASE_IMAGE_TAG=18-trixie
FROM postgres:${POSTGRES_BASE_IMAGE_TAG}
# ... rest unchanged
```

### E. compose.override.yaml-example (~17 lines)

Replace content with modern examples:
```yaml
---
services:
  ots:  # Fix: was "opentakserver" (inconsistent with compose.yaml)
    user: "${UID:-1000}:${GID:-1000}"
    environment:
      # Disable MediaMTX if not using video
      - DOCKER_OTS_MEDIAMTX_ENABLE=False

      # Enable debug mode for development
      # - DOCKER_DEBUG=True

      # Custom CA name
      # - DOCKER_OTS_CA_NAME=MyOrganization-CA

  # Example: Use external SSD for database
  # ots-db:
  #   volumes:
  #     - "${DB_DATA_PATH:-/mnt/ssd/ots-db}:/var/lib/postgresql"

  # Example: Enable RabbitMQ data persistence
  # rabbitmq:
  #   volumes:
  #     - "${RABBITMQ_DATA_PATH:-./persistent/rabbitmq/data}:/var/lib/rabbitmq/:rw"
```

---

## Files to Create

### F. .env.example (NEW, ~600 lines)

**Structure (7 sections):**

```bash
# =============================================================================
# OpenTAKServer Docker Compose Configuration
# =============================================================================
# Copy this file to .env and customize as needed
# Documentation: See CLAUDE.md for detailed usage
# Last Updated: 2025-12-15

# -----------------------------------------------------------------------------
# 1. CONTAINER IMAGE VERSIONS (~30 lines)
# -----------------------------------------------------------------------------
# Control which container versions to deploy
# Pin versions in production to prevent unexpected updates

OTS_IMAGE_TAG=latest
OTS_WEBUI_IMAGE_TAG=latest
RABBITMQ_IMAGE_TAG=latest
MEDIAMTX_IMAGE_TAG=1.13.0-ffmpeg
TRAEFIK_IMAGE_TAG=v3.5
NGINX_BASE_IMAGE_TAG=mainline-bookworm
POSTGRES_BASE_IMAGE_TAG=18-trixie

# ... (see .env.example for full content)
```

**Key Features:**
- All 100+ OTS parameters documented
- Clear section organization
- Sensible defaults matching current behavior
- Comments explain purpose and usage
- Security warnings for sensitive values

### G. MIGRATION.md (NEW, ~100 lines)

Brief migration guide covering:
- What changed in v2.0
- Do existing deployments need to migrate? (No, optional)
- Steps to pin image versions
- Steps to customize data locations
- Rollback instructions

---

## Documentation Updates

### H. CLAUDE.md (~380 lines total, add ~100 lines)

Insert new section after "## Configuration":

```markdown
### Environment-Based Configuration

**New**: All configuration is manageable via environment variables.

#### Configuration Priority

Docker Compose reads config in this order (later overrides earlier):
1. Defaults in compose.yaml (e.g., `${VAR:-default}`)
2. .env file (automatically loaded)
3. compose.override.yaml (service-specific overrides)
4. Shell environment variables (highest priority)

#### Image Version Management

Pin production versions to prevent unexpected updates:
```bash
# .env
OTS_IMAGE_TAG=v1.7.0
OTS_WEBUI_IMAGE_TAG=v1.5.0
RABBITMQ_IMAGE_TAG=3.13
```

#### Data Location Configuration

Move data to dedicated storage:
```bash
# .env
DB_DATA_PATH=/mnt/ssd/ots-database
OTS_DATA_PATH=/opt/opentakserver/data
```

#### OTS Configuration Parameters

All 100+ parameters available with `DOCKER_` prefix:
```bash
DOCKER_OTS_CA_NAME=MyOrganization-CA
DOCKER_OTS_LISTENER_PORT=8081
DOCKER_OTS_MEDIAMTX_ENABLE=True
```

See .env.example for complete list.
```

Add troubleshooting entries for new features.

---

## Implementation Steps (Sequential)

1. **Modify Dockerfiles** (lowest risk):
   - Add ARG to persistent/nginx/Dockerfile
   - Add ARG to persistent/db/Dockerfile
   - Test: `docker compose build nginx-proxy ots-db`

2. **Update compose.yaml** (core changes):
   - Replace all image tags with variables
   - Add build args for nginx-proxy and ots-db
   - Replace all volume paths with variables
   - Test: `docker compose config`

3. **Update compose.traefik.yaml** (parallel):
   - Apply same image tag changes
   - Apply same volume path changes
   - Test: `docker compose -f compose.traefik.yaml config`

4. **Create .env.example** (documentation):
   - Structure with 7 sections
   - Document all 100+ OTS parameters
   - Add clear comments and examples
   - Test: `cp .env.example .env && docker compose config`

5. **Update compose.override.yaml-example**:
   - Fix service name (opentakserver → ots)
   - Add modern examples
   - Show image tag and path overrides

6. **Update CLAUDE.md**:
   - Add Environment-Based Configuration section
   - Add troubleshooting entries
   - Update configuration reference table

7. **Create MIGRATION.md**:
   - Document what changed
   - Provide migration steps
   - Include rollback procedure

8. **Comprehensive testing**:
   - Default behavior (no .env) → works as before
   - With .env → custom images and paths
   - With override → local dev customization
   - Traefik variant → alternative proxy setup

---

## Validation Checklist

- [ ] Backward compatible (works without .env)
- [ ] All images parameterized
- [ ] All volume paths parameterized
- [ ] .env.example comprehensive and documented
- [ ] compose.override.yaml-example modernized
- [ ] CLAUDE.md updated with new section
- [ ] MIGRATION.md created
- [ ] `docker compose config` validates successfully
- [ ] `make up` starts all services
- [ ] Web UI accessible at https://localhost
- [ ] SSL test passes: `make dev-test-ssl`

---

## Critical Files for Implementation

1. `/Users/kapturm/Documents/Code/ots-docker/compose.yaml` - Core compose file
2. `/Users/kapturm/Documents/Code/ots-docker/.env.example` - New comprehensive template
3. `/Users/kapturm/Documents/Code/ots-docker/compose.traefik.yaml` - Alternative compose
4. `/Users/kapturm/Documents/Code/ots-docker/CLAUDE.md` - Documentation updates
5. `/Users/kapturm/Documents/Code/ots-docker/compose.override.yaml-example` - Modernize examples
6. `/Users/kapturm/Documents/Code/ots-docker/persistent/nginx/Dockerfile` - Add ARG
7. `/Users/kapturm/Documents/Code/ots-docker/persistent/db/Dockerfile` - Add ARG
8. `/Users/kapturm/Documents/Code/ots-docker/MIGRATION.md` - New migration guide

---

## Risk Mitigation

- **Backward Compatibility**: Use `${VAR:-default}` syntax ensures defaults match current behavior
- **Storage**: .env and compose.override.yaml already in .gitignore
- **Data Integrity**: No database schema changes
- **Rollback**: Easy via `rm .env` or git revert

---

## Success Criteria

✓ Existing deployments continue working without changes
✓ Users can pin any image version via .env
✓ Users can customize any data path via .env
✓ All 100+ OTS parameters documented and accessible
✓ Configuration system follows Docker Compose best practices
✓ Migration path clear for existing users
