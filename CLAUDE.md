# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This repository builds and publishes a Docker image combining Pi-hole and Unbound DNS resolver into a single container. The base is the official Pi-hole container (`pihole/pihole`) with Unbound integrated following official Pi-hole guidance. Images are published to Docker Hub (`mpgirro/pihole-unbound` and `mpgirro/docker-pihole-unbound`) and GitHub Container Registry (`ghcr.io/mpgirro/docker-pihole-unbound`).

## Build and Test Commands

### Building the Docker Image

```bash
# Build locally (single platform)
docker build -t pihole-unbound:local docker/

# Build with Buildx (multi-platform)
docker buildx build --platform linux/amd64,linux/arm64 -t pihole-unbound:local docker/
```

### Testing the Image

```bash
# Run the container locally using example compose file
cd example
docker compose up -d

# View logs
docker logs pihole-unbound

# Check if Unbound is running
docker exec pihole-unbound ps aux | grep unbound

# Test DNS resolution through the container
docker exec pihole-unbound dig @127.0.0.1 -p 5335 example.com

# Stop and clean up
docker compose down
```

## Architecture

### Docker Build Structure

The build process is defined in `docker/Dockerfile`:
- Base: Official `pihole/pihole` image (version pinned in FROM statement)
- Installs Unbound via `apk add unbound`
- Copies configuration files into the container
- Uses a custom entrypoint script to start both services

### Configuration Files in `docker/`

- **Dockerfile**: Main build definition, version is extracted by CI/CD
- **custom-entrypoint.sh**: Custom entrypoint that starts Unbound before calling Pi-hole's original `start.sh`
- **unbound-pihole.conf**: Unbound DNS resolver configuration (listens on 127.0.0.1:5335)
- **99-edns.conf**: EDNS buffer size configuration for dnsmasq
- **lighttpd-external.conf**: Allows Pi-hole admin interface in iframes (e.g., Home Assistant)
- **unbound-run**: Legacy s6 service script (not used by custom-entrypoint.sh)

### Service Startup Flow

1. Container starts with `custom-entrypoint.sh`
2. Creates Unbound log directory at `/var/log/unbound`
3. Starts Unbound in background on port 5335
4. Verifies Unbound started successfully
5. Calls original Pi-hole `start.sh` to start Pi-hole services
6. Pi-hole forwards DNS queries to Unbound (127.0.0.1#5335)

### CI/CD Architecture

Two GitHub Actions workflows handle image publishing:

**docker-publish.yml** (main builds):
- Triggers on pushes to `main` branch when `docker/Dockerfile` changes
- Extracts Pi-hole version from Dockerfile using grep
- Builds multi-platform images (linux/amd64, linux/386, linux/arm/v6, linux/arm/v7, linux/arm64)
- Publishes to Docker Hub and GHCR
- Tags with extracted version and `latest`
- Signs images with Cosign
- Creates GitHub releases with version-specific tags

**pr-docker-image.yml** (PR builds):
- Triggers on pull requests
- Builds and pushes PR-specific test images to GHCR only
- Tags format: `pr{PR_NUMBER}-{RUN_NUMBER}-{SHA}`

### Renovate Integration

Renovate automatically updates the Pi-hole base image version in `docker/Dockerfile`:
- Monitors `pihole/pihole` Docker image for new releases
- Creates separate PRs for major, minor, and patch updates
- Configuration in `renovate.json` with custom rules for Pi-hole updates

## Important Implementation Details

### Version Management

The Pi-hole version is the single source of truth for this image's versioning:
- Version is specified in `docker/Dockerfile` as `FROM pihole/pihole:X.Y.Z`
- CI/CD extracts version with: `grep -oP '(?<=pihole/pihole:)\S+' docker/Dockerfile`
- Released images use same version tags as base Pi-hole image

When updating Pi-hole version:
1. Update only the version in `docker/Dockerfile` FROM statement
2. Push to `main` branch (or let Renovate do it)
3. CI/CD will automatically build, tag, and release with that version

### Unbound Configuration

The Unbound resolver configuration (`unbound-pihole.conf`) is based on official Pi-hole documentation:
- Listens only on localhost (127.0.0.1:5335)
- DNSSEC validation enabled
- IPv6 disabled by default
- Private address ranges protected
- EDNS buffer size set to 1232 (reduced fragmentation)
- Single-threaded (sufficient for most home networks)

### Pi-hole Integration Points

Pi-hole must be configured to use Unbound as upstream DNS:
- Set via environment variable: `FTLCONF_dns_upstreams=127.0.0.1#5335`
- DNSSEC should be enabled in Pi-hole: `FTLCONF_dns_dnssec="true"`
- Example configuration in `example/compose.yaml`

### Multi-Platform Build Support

The image supports five architectures:
- linux/amd64 (x86_64)
- linux/386 (x86)
- linux/arm/v6 (ARMv6, e.g., Raspberry Pi Zero)
- linux/arm/v7 (ARMv7, e.g., Raspberry Pi 2/3)
- linux/arm64 (ARM64, e.g., Raspberry Pi 4)

Builds use QEMU for cross-compilation and GitHub Actions cache for faster rebuilds.

## Development Workflow

### Making Changes to the Image

1. Edit files in `docker/` directory
2. Test locally using Docker build and example compose file
3. For version updates, only change the FROM line in `docker/Dockerfile`
4. PR builds will create test images tagged with PR number

### Testing Configuration Changes

When modifying Unbound or Pi-hole configuration:
1. Build image locally
2. Run with `example/compose.yaml`
3. Check both services started: `docker logs pihole-unbound`
4. Test DNS resolution: `docker exec pihole-unbound dig @127.0.0.1 -p 5335 example.com`
5. Test Pi-hole admin interface: `http://localhost:<PIHOLE_WEBPORT>/admin`

### Release Process

Releases are fully automated:
1. Renovate creates PR with updated Pi-hole version in Dockerfile
2. Review and merge PR to `main`
3. CI/CD automatically builds, publishes, signs, and creates GitHub release
4. No manual intervention needed

## Environment Variables

All Pi-hole environment variables are supported. Key variables for this image:

- `FTLCONF_dns_upstreams`: Must be set to `127.0.0.1#5335` to use Unbound
- `FTLCONF_dns_dnssec`: Should be `"true"` for DNSSEC validation
- `FTLCONF_webserver_api_password`: Admin interface password
- `TZ`: Timezone for proper log rotation
- `FTLCONF_dns_listeningMode`: Recommended `single` for Docker networking

See `example/compose.yaml` for a complete working configuration and README.md for full variable documentation.
