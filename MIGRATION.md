# Migration Guide: Upgrading to Environment-Based Configuration

**Version**: 2.0+
**Last Updated**: 2025-12-15

---

## Overview

Version 2.0 introduces environment variable support for:
- **Container image versions** - Pin specific releases
- **Data storage locations** - Use external volumes or SSDs
- **All OTS configuration parameters** - Complete environment control

---

## Do I Need to Migrate?

**Short answer: No.** Your existing deployment will continue working without any changes.

The new `.env` configuration is **optional** and backward compatible. All changes use `${VAR:-default}` syntax, which means:
- If `.env` doesn't exist, defaults match current hardcoded behavior
- If you don't set a variable, the current behavior is preserved
- Existing `compose.override.yaml` continues to work as before

---

## When Should I Migrate?

Consider upgrading to use `.env` if you want to:

1. **Pin production image versions** (prevent automatic updates)
   ```bash
   OTS_IMAGE_TAG=v1.7.0
   RABBITMQ_IMAGE_TAG=3.13
   ```

2. **Move data to dedicated storage** (faster SSDs, NAS, etc.)
   ```bash
   DB_DATA_PATH=/mnt/fast-ssd/ots-database
   OTS_DATA_PATH=/opt/opentakserver/data
   ```

3. **Manage configuration centrally** (easier than editing config.yml)
   ```bash
   DOCKER_OTS_CA_NAME=MyOrganization
   DOCKER_OTS_MEDIAMTX_ENABLE=False
   ```

4. **Standardize across multiple deployments** (dev, staging, prod)

---

## Migration Steps

### Step 1: Update Repository

```bash
cd ots-docker
git pull origin main
```

Verify you have new files:
- `.env.example` (new)
- `MIGRATION.md` (this file)
- Updated: `compose.yaml`, `compose.traefik.yaml`, Dockerfiles, `CLAUDE.md`

### Step 2: Review Changes (Optional)

```bash
# See git history
git log --oneline -10

# Review changes to compose.yaml
git diff HEAD~1 compose.yaml | head -50

# Review new .env.example
cat .env.example | head -100
```

### Step 3: Test Backward Compatibility (Recommended)

Verify existing setup still works:

```bash
# Without .env file, should work exactly as before
make down
rm -f .env
make up
make logs
# Check: All services start, Web UI accessible at https://localhost
```

### Step 4: Pin Current Versions (Optional but Recommended)

Discover current running versions:

```bash
docker-compose ps --format "table {{.Service}}\t{{.Image}}"
```

Create `.env` with current versions to prevent unexpected updates:

```bash
cat > .env << 'EOF'
# Pin current versions
OTS_IMAGE_TAG=latest
OTS_WEBUI_IMAGE_TAG=latest
RABBITMQ_IMAGE_TAG=latest
MEDIAMTX_IMAGE_TAG=1.13.0-ffmpeg
TRAEFIK_IMAGE_TAG=v3.5
NGINX_BASE_IMAGE_TAG=mainline-bookworm
POSTGRES_BASE_IMAGE_TAG=18-trixie

# Keep existing data locations
# (Only change if you want to move data elsewhere)
EOF
```

Test with `.env`:

```bash
# Validate configuration
docker-compose config > /dev/null && echo "✓ Valid configuration"

# Restart services
make down && make up
make logs
```

### Step 5: Customize Configuration (Optional)

Copy full `.env.example` for all available options:

```bash
cp .env.example .env
nano .env  # Edit with your preferences
```

Common customizations:

```bash
# Example 1: Move database to SSD
DB_DATA_PATH=/mnt/ssd/ots-database

# Example 2: Custom certificate authority
DOCKER_OTS_CA_NAME=YourOrganization-CA
DOCKER_OTS_CA_PASSWORD=secure-password

# Example 3: Enable email
DOCKER_OTS_ENABLE_EMAIL=True
DOCKER_MAIL_SERVER=smtp.gmail.com
DOCKER_MAIL_PORT=587
```

Restart services:

```bash
make down && make up
make logs
```

### Step 6: Use Local Overrides (Optional)

For environment-specific configurations without editing `.env`:

```bash
cp compose.override.yaml-example compose.override.yaml
nano compose.override.yaml  # Customize

# Restart
make down && make up
```

---

## Rollback

If you experience issues:

### Quick Rollback (Remove .env)

```bash
# Remove custom configuration
rm .env compose.override.yaml

# Restart with defaults
make down && make up

# Should work exactly as before
make logs
```

### Full Rollback (Revert to v1.x)

```bash
# Go back to previous version
git checkout v1.x -- compose.yaml compose.traefik.yaml
git checkout v1.x -- persistent/nginx/Dockerfile
git checkout v1.x -- persistent/db/Dockerfile

# Remove any .env files
rm -f .env

# Restart
make down && make up
```

### Data Preservation

**Important**: Data persists regardless of `.env` or version changes:
- `.env` only affects configuration, not data
- `docker-compose down` preserves data in `persistent/` directory
- `docker-compose down -v` removes volumes (WARNING: data loss!)

---

## Configuration Priority

Docker Compose reads configuration in this order (later overrides earlier):

1. **compose.yaml defaults** (e.g., `${VAR:-default}`)
2. **.env file** (automatically loaded)
3. **compose.override.yaml** (service-specific overrides)
4. **Shell environment** (highest priority)

### Examples

```bash
# Use default (compose.yaml)
make up

# Use version from .env
echo "OTS_IMAGE_TAG=v1.7.0" >> .env
make up  # Uses v1.7.0

# Override with shell variable (one-time)
OTS_IMAGE_TAG=v1.6.0 make up  # Uses v1.6.0, doesn't change .env
```

---

## Common Migration Scenarios

### Scenario 1: Production Deployment

**Goal**: Stable, reproducible, external database

```bash
# .env configuration
OTS_IMAGE_TAG=v1.7.0
OTS_WEBUI_IMAGE_TAG=v1.5.0
RABBITMQ_IMAGE_TAG=3.13-management
DB_DATA_PATH=/mnt/ssd/ots-database
POSTGRESQL_PASSWORD=secure-prod-password
DOCKER_OTS_CA_NAME=YourOrganization-CA
DOCKER_OTS_CA_PASSWORD=secure-ca-password
```

### Scenario 2: Development Environment

**Goal**: Latest features, quick iteration

```bash
# .env configuration
OTS_IMAGE_TAG=latest
OTS_WEBUI_IMAGE_TAG=latest
RABBITMQ_IMAGE_TAG=latest
DOCKER_DEBUG=True
DOCKER_OTS_MEDIAMTX_ENABLE=False
```

### Scenario 3: Multiple Environments

**Goal**: Same codebase, different configs

```bash
# Create environment-specific files
cp .env.example .env.dev
cp .env.example .env.prod

# Use with Docker Compose
docker-compose -f compose.yaml --env-file .env.dev up  # dev
docker-compose -f compose.yaml --env-file .env.prod up  # prod
```

---

## Verification Checklist

After migration, verify:

- [ ] All services start: `docker-compose ps`
- [ ] No errors in logs: `make logs`
- [ ] Web UI accessible: https://localhost
- [ ] SSL test passes: `make dev-test-ssl`
- [ ] Configuration applied: Check OTS logs for your custom settings
- [ ] Data intact: Verify data in `persistent/` directories

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| "image not found" | Wrong image tag | Check `*_IMAGE_TAG` in `.env` |
| "permission denied on volume" | Path doesn't exist | Create directory with `mkdir -p` |
| "connection refused" | Service not healthy | Wait for health checks, check logs |
| "configuration not applied" | Wrong variable name | OTS config needs `DOCKER_` prefix |
| Services don't start | Invalid .env syntax | Validate with `docker-compose config` |

---

## Support

- **Issues**: [GitHub Issues](https://github.com/milsimdk/ots-docker/issues)
- **Documentation**: See [CLAUDE.md](./CLAUDE.md) for detailed configuration guide
- **Upstream**: [OpenTAKServer Docs](https://docs.opentakserver.io/)

---

## What's New in v2.0

- ✨ Environment-based image version pinning
- ✨ Configurable data storage locations
- ✨ Complete OTS parameter documentation in `.env.example`
- ✨ Comprehensive migration guide (this file)
- ✨ Updated CLAUDE.md with configuration section
- 🔧 Build args in Dockerfiles for base image customization
- 🔧 Comments in compose files explaining mount purposes

All changes are **backward compatible** - existing deployments continue to work unchanged.
