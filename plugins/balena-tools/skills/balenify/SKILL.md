---
name: balenify
description: "Guide for converting Docker projects to run on balenaCloud. Use this skill when balenifying or converting an existing Docker/docker-compose project for balena, writing or modifying Dockerfile.template files, setting up docker-compose.yml for balena fleets, choosing between Dockerfile.template and plain Dockerfile for balena services, debugging balena build failures, converting volume mounts for balena, handling environment variables on balena, configuring hardware access (GPIO, USB, serial) for containers, or understanding balena networking. Also use when the user mentions balena device types, %%BALENA_ARCH%%, %%BALENA_MACHINE_NAME%%, balenalib base images, bh.cr registry references, or asks how to run something on balena."
---

# Balenify — Convert Docker Projects to balenaCloud

## What This Skill Covers

This skill covers the full workflow for converting a standard Docker or docker-compose project to run on balenaCloud. That includes Dockerfile templates for multi-architecture builds, volume mount conversion, networking differences, environment variable handling, hardware access, docker-compose.yml adaptations, and balena.yml fleet metadata.

Use this as a reference whenever you're porting an existing project to balena or building a new balena fleet from scratch.

## Balenify Checklist

Quick reference for the conversion steps:

- [ ] Replace host bind mounts (`./data:/app`) with named volumes
- [ ] Remove `.env` files — use dashboard/CLI variables instead
- [ ] Add `balena.yml` with fleet name and supported device types
- [ ] Convert Dockerfiles to `.template` if arch-specific base images are needed
- [ ] Add balena-specific labels for hardware features (D-Bus, kernel modules, supervisor API)
- [ ] Set `privileged: true` or selective `devices:`/`cap_add:` for hardware access
- [ ] Set `network_mode: host` if the service needs host networking (multicontainer defaults to bridge)
- [ ] Remove `links:` — bridge network provides service discovery by container name
- [ ] Pin `version: "2.1"` in docker-compose.yml (balena uses Compose v2/2.1)

## Dockerfile Templates

Balena's build system supports Dockerfile templates — files named `Dockerfile.template` that undergo variable substitution before Docker processes them. The template system exists because Docker images are architecture-specific, and balena fleets often span multiple device types (e.g., Raspberry Pi 3, Pi 4, Intel NUC) with different CPU architectures.

### Core Concept: Template Variable Substitution

Template variables use `%%VARIABLE%%` syntax. The balena builder replaces them with fleet/device-specific values *before* the Docker build begins — Docker never sees the `%%` tokens.

#### Available Variables

| Variable | Description | Example Value |
|----------|-------------|---------------|
| `%%BALENA_ARCH%%` | Target CPU architecture | `aarch64`, `amd64`, `armv7hf` |
| `%%BALENA_MACHINE_NAME%%` | Yocto machine name for the device type | `raspberrypi4-64`, `genericx86-64-ext` |
| `%%BALENA_APP_NAME%%` | Fleet name | `my-fleet` |
| `%%BALENA_RELEASE_HASH%%` | Release hash | `abc123...` |
| `%%BALENA_SERVICE_NAME%%` | Service name from docker-compose.yml | `pihole` |

**Use `%%BALENA_ARCH%%` by default.** Only use `%%BALENA_MACHINE_NAME%%` when you genuinely need device-type-specific behavior (rare). Machine name won't evaluate correctly if a fleet has mixed device types.

### Dockerfile Resolution Order

The balena builder picks the *first match* from this priority list:

1. `Dockerfile.<device-type>` — exact device type match (e.g., `Dockerfile.raspberrypi4-64`)
2. `Dockerfile.<arch>` — architecture match (e.g., `Dockerfile.aarch64`)
3. `Dockerfile.template` — universal fallback with variable substitution
4. `Dockerfile` — plain Dockerfile, no substitution

This applies during `balena push`, `balena build`, `balena deploy`, and `git push`.

**When to use templates vs plain Dockerfiles:**

- If your base image is multi-platform (published with Docker manifest lists) or you only target one architecture → use a plain `Dockerfile`
- If you need to select between architecture-specific images that lack multi-platform manifests → use `Dockerfile.template`
- Most services in a fleet can use plain Dockerfiles. Only the ones pulling arch-specific images need templates.

### Balena Arch vs Docker/Go Arch

Balena has its own architecture naming convention, distinct from Docker and Go:

| Balena (`%%BALENA_ARCH%%`) | Docker Platform | Go GOARCH |
|---|---|---|
| `aarch64` | `linux/arm64` | `arm64` |
| `amd64` | `linux/amd64` | `amd64` |
| `armv7hf` | `linux/arm/v7` | `arm` (GOARM=7) |
| `rpi` | `linux/arm/v6` | `arm` (GOARM=6) |
| `i386` | `linux/386` | `386` |

This matters because balenaCloud doesn't support multi-arch fleets yet — each fleet targets a single device type and architecture. Fleets and blocks on balena hub (bh.cr) are typically published as separate images per balena arch (e.g., `my-block-aarch64`, `my-block-amd64`), not as multi-platform manifests. Templates with `%%BALENA_ARCH%%` map directly to these naming conventions.

### The Simple Pattern (Use This by Default)

For most cases, a single FROM line with `%%BALENA_ARCH%%` interpolated into the image reference is all you need:

```dockerfile
FROM bh.cr/some-org/my-block-%%BALENA_ARCH%%/1.0.0
```

Or with a standard registry:

```dockerfile
FROM myregistry/myimage-%%BALENA_ARCH%%:1.0.0

COPY . /app
CMD ["/app/start.sh"]
```

This pattern works especially well with bh.cr, where blocks and fleets are published per-architecture using balena's naming convention. The builder substitutes `%%BALENA_ARCH%%` → `aarch64` (or `amd64`, `armv7hf`, etc.) and Docker pulls the right image.

### Multi-Stage Pattern

Use this when the simple pattern won't work. The two main reasons:

1. **Arch naming mismatch**: The upstream image uses Docker/Go arch names (`arm64`, `arm64v8`) instead of balena names (`aarch64`, `armv7hf`). A simple `FROM image-%%BALENA_ARCH%%:tag` would try to pull `image-aarch64:tag` which doesn't exist.
2. **Renovate/Dependabot pinning**: You want automated dependency tooling to independently update per-arch image references. The simple `%%BALENA_ARCH%%` pattern embeds a variable inside the image reference, which Renovate can't parse.

```dockerfile
ARG BALENA_ARCH=%%BALENA_ARCH%%

# Stage names use balena arch convention — map to whatever the upstream publishes
FROM linuxserver/wireguard:amd64-latest AS wireguard-amd64
FROM linuxserver/wireguard:arm64v8-latest AS wireguard-aarch64

# hadolint ignore=DL3006
FROM wireguard-${BALENA_ARCH}
```

The key trick: name each stage with the **balena** arch suffix (e.g., `wireguard-aarch64`) regardless of what the upstream tag calls it (e.g., `arm64v8-latest`). Then `FROM wireguard-${BALENA_ARCH}` resolves correctly.

**How it works:**
1. `%%BALENA_ARCH%%` is substituted by the balena builder before Docker runs
2. Docker ARG captures the resolved value
3. Named FROM stages alias each upstream image to balena arch names
4. `FROM wireguard-${BALENA_ARCH}` resolves to the correct stage
5. Docker only pulls the selected stage — the others are skipped

The `# hadolint ignore=DL3006` comment suppresses the "always tag the version of an image explicitly" lint warning on the dynamic FROM line.

#### Supported Architecture Values

| `%%BALENA_ARCH%%` | Devices |
|---|---|
| `aarch64` | Raspberry Pi 3/4/5 (64-bit), Jetson, most modern ARM boards |
| `armv7hf` | Raspberry Pi 2/3 (32-bit), BeagleBone |
| `amd64` | Intel NUC, generic x86_64 |
| `rpi` | Raspberry Pi 1 / Zero (armv6) |

## Volume Mounts

Host bind mounts (`./data:/app/data`) do **not** work on balena. The host filesystem layout is managed by balenaOS and is not directly accessible to containers.

### Use Named Volumes

Declare volumes in the top-level `volumes:` section and reference them in services:

```yaml
version: "2.1"

volumes:
  app-data:
  config:

services:
  my-service:
    build: my-service
    volumes:
      - app-data:/app/data
      - config:/etc/myapp
```

### Key behaviors

- Named volumes **persist across container updates** as long as the volume name stays the same
- Shared volumes between services: mount the same named volume in multiple services for inter-service data exchange
- Host OS path: `/var/lib/docker/volumes/<APP_ID>_<volume_name>/_data`
- **Single-container apps** get a default `resin-data` volume mounted at `/data` — no explicit volume config needed
- **Warning**: volumes are purged if a device is moved to a different fleet

### Converting bind mounts

| Docker (original) | Balena (converted) |
|---|---|
| `./data:/app/data` | `app-data:/app/data` (+ declare `app-data:` in `volumes:`) |
| `./config.json:/etc/app/config.json` | Bake into image with `COPY`, or use an env var |
| `/var/run/docker.sock:/var/run/docker.sock` | Use `io.balena.features.balena-socket: 1` label instead |

## Networking

### Single-container apps

Host networking by default — the container shares the host's network stack. No port mapping needed.

### Multicontainer apps

Bridge network by default — each service gets its own network namespace. Services are routable by container name.

```yaml
services:
  frontend:
    build: frontend
    ports:
      - "80:3000"   # expose to host/external network

  backend:
    build: backend
    # no ports needed — frontend reaches backend at http://backend:8080
```

### Key points

- `ports:` only needed to expose a service to the host network or external clients — **not** for inter-service communication
- No need for `links:` — the bridge network provides automatic service discovery by container name
- Set `network_mode: host` on a service if it needs direct access to the host network stack (e.g., mDNS, DHCP, network scanning)
- Custom networks (`networks:`) are not supported — all multicontainer services share a single bridge network

## Environment Variables

### No `.env` files

Balena does not read `.env` files. Instead, set variables through:

- **balenaCloud dashboard** → fleet or device variables (with optional per-service scope)
- **CLI**: `balena env set MY_VAR my_value --fleet my-fleet` or `--device <uuid>`

### Variable priority (highest to lowest)

1. Device + service variable
2. Device variable
3. Fleet + service variable
4. Fleet variable

### Balena-injected variables

These are automatically available in every container:

| Variable | Value |
|----------|-------|
| `BALENA` | `1` (use to detect running on balena) |
| `BALENA_DEVICE_UUID` | Device UUID |
| `BALENA_APP_ID` | Fleet/application numeric ID |
| `BALENA_APP_NAME` | Fleet name |
| `BALENA_SERVICE_NAME` | Service name from docker-compose.yml |
| `BALENA_SUPERVISOR_ADDRESS` | Supervisor API URL (usually `http://127.0.0.1:48484`) |
| `BALENA_SUPERVISOR_API_KEY` | API key for supervisor requests |

Values can be up to 1MB each.

### Converting from `.env` files

```bash
# Read each KEY=VALUE from .env and set as fleet variables
while IFS='=' read -r key value; do
  [[ "$key" =~ ^#.*$ || -z "$key" ]] && continue
  balena env set "$key" "$value" --fleet my-fleet
done < .env
```

## Hardware Access

### Single-container apps

Privileged by default — full access to `/dev` and all host devices. No extra config needed.

### Multicontainer apps

**Not** privileged by default. You must explicitly grant access:

**Option 1 — Full privileged mode:**
```yaml
services:
  my-service:
    build: my-service
    privileged: true
```

**Option 2 — Selective access (preferred when possible):**
```yaml
services:
  my-service:
    build: my-service
    devices:
      - "/dev/i2c-1:/dev/i2c-1"
      - "/dev/ttyUSB0:/dev/ttyUSB0"
    cap_add:
      - SYS_RAWIO
```

### Balena feature labels for hardware

Use labels instead of manual `/dev` paths where possible — they're more stable across device types:

| Label | Grants access to |
|-------|-----------------|
| `io.balena.features.kernel-modules: 1` | Kernel module loading |
| `io.balena.features.dbus: 1` | Host D-Bus socket |
| `io.balena.features.supervisor-api: 1` | Supervisor REST API |
| `io.balena.features.balena-api: 1` | balenaCloud API |
| `io.balena.features.balena-socket: 1` | balenaEngine socket (like Docker socket) |
| `io.balena.features.sysfs: 1` | Host `/sys` filesystem |
| `io.balena.features.procfs: 1` | Host `/proc` filesystem |
| `io.balena.features.gpu: 1` | GPU device access |

Prefer feature labels over raw device paths — they abstract hardware differences across device types.

## docker-compose.yml for Balena

Balena uses Docker Compose v2/2.1. Each service with a `build:` directive points to a directory containing a Dockerfile (or Dockerfile.template).

```yaml
version: "2.1"

services:
  my-service:
    build: my-service
    network_mode: host
    cap_add:
      - NET_ADMIN
    labels:
      io.balena.features.kernel-modules: 1
    tmpfs:
      - /tmp
      - /var/run

  # Services using pre-built images don't need templates
  helper:
    image: bh.cr/some-org/some-block/1.0.0
    restart: no
```

### Balena-Specific Labels

| Label | Purpose |
|-------|---------|
| `io.balena.features.kernel-modules: 1` | Access kernel modules |
| `io.balena.features.dbus: 1` | Access host D-Bus |
| `io.balena.features.supervisor-api: 1` | Access supervisor API |
| `io.balena.features.balena-api: 1` | Access balena API |
| `io.balena.features.balena-socket: 1` | Access balenaEngine socket |
| `io.balena.features.sysfs: 1` | Access host `/sys` |
| `io.balena.features.procfs: 1` | Access host `/proc` |
| `io.balena.features.gpu: 1` | GPU device access |

### Update Strategy Labels

Control how balena updates containers:

| Label | Values | Default |
|-------|--------|---------|
| `io.balena.update.strategy` | `download-then-kill`, `kill-then-download`, `delete-then-download`, `hand-over` | `download-then-kill` |
| `io.balena.update.handover-timeout` | Seconds (for `hand-over` strategy) | `60` |

- `download-then-kill` — download new image, then stop old container (minimal downtime, needs 2x storage)
- `kill-then-download` — stop old container first, then download (saves storage, longer downtime)
- `delete-then-download` — delete old image and container, then download (maximum storage savings)
- `hand-over` — old and new containers run simultaneously during handover period (zero-downtime deploys)

## balena.yml

Every fleet project has a `balena.yml` at root:

```yaml
name: "My Fleet"
type: "sw.application"
version: 1.0.0
description: "What this fleet does"
data:
  defaultDeviceType: "raspberrypi4-64"
  supportedDeviceTypes:
    - "raspberrypi3-64"
    - "raspberrypi4-64"
    - "genericx86-64-ext"
```

The `supportedDeviceTypes` list determines which architectures your Dockerfile.template must handle.

## Debugging Build Failures

| Symptom | Likely Cause |
|---------|-------------|
| `invalid reference format` | Template variable not substituted — check file is named `.template` |
| `manifest unknown` | Architecture not published for that image tag |
| `no match for platform` | Using a plain Dockerfile where a template is needed |
| Wrong arch binary runs | `%%BALENA_MACHINE_NAME%%` resolved to fleet default, not target device |

## Common Patterns Reference

Read `references/patterns.md` for:
- Legacy `%%RESIN_MACHINE_NAME%%` syntax (pre-2019 projects)
- Balenalib base images (`FROM balenalib/%%BALENA_MACHINE_NAME%%-alpine`)
- `install_packages` helper in balenalib images
- Cross-compilation with `TARGETARCH` fallback
- The `INITSYSTEM on` ENV (legacy balena supervisor integration)
- bh.cr (balena hub container registry) image references
- Renovate configuration patterns for auto-updating image tags

## Workflow

When converting a Docker project to balena:

1. Check `balena.yml` → what device types does this fleet target?
2. Map device types to architectures (see table above)
3. For each service: does its base image support multi-platform? If yes → plain Dockerfile. If no → Dockerfile.template with the simple `%%BALENA_ARCH%%` pattern.
4. Replace bind mounts with named volumes.
5. Remove `.env` files — migrate variables to balenaCloud dashboard or CLI.
6. Add balena-specific labels for hardware features the service needs.
7. Set `privileged: true` or selective `devices:`/`cap_add:` for hardware access.
8. Test with `balena build --deviceType <type>` for each target.
