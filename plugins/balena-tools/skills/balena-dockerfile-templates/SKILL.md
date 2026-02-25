---
name: balena-dockerfile-templates
description: "Guide for writing Dockerfile.template files for balena multi-architecture fleet deployments. Use this skill whenever working on balena projects that need to support multiple device types or architectures, writing or modifying Dockerfile.template files, setting up docker-compose.yml for balena fleets, choosing between Dockerfile.template and plain Dockerfile for balena services, or debugging balena build failures related to template variable substitution. Also use when the user mentions balena device types, %%BALENA_ARCH%%, %%BALENA_MACHINE_NAME%%, balenalib base images, or bh.cr registry references."
---

# Balena Dockerfile Templates

## What This Skill Covers

Balena's build system supports Dockerfile templates — files named `Dockerfile.template` that undergo variable substitution before Docker processes them. The template system exists because Docker images are architecture-specific, and balena fleets often span multiple device types (e.g., Raspberry Pi 3, Pi 4, Intel NUC) with different CPU architectures.

## Core Concept: Template Variable Substitution

Template variables use `%%VARIABLE%%` syntax. The balena builder replaces them with fleet/device-specific values *before* the Docker build begins — Docker never sees the `%%` tokens.

### Available Variables

| Variable | Description | Example Value |
|----------|-------------|---------------|
| `%%BALENA_ARCH%%` | Target CPU architecture | `aarch64`, `amd64`, `armv7hf` |
| `%%BALENA_MACHINE_NAME%%` | Yocto machine name for the device type | `raspberrypi4-64`, `genericx86-64-ext` |
| `%%BALENA_APP_NAME%%` | Fleet name | `my-fleet` |
| `%%BALENA_RELEASE_HASH%%` | Release hash | `abc123...` |
| `%%BALENA_SERVICE_NAME%%` | Service name from docker-compose.yml | `pihole` |

**Use `%%BALENA_ARCH%%` by default.** Only use `%%BALENA_MACHINE_NAME%%` when you genuinely need device-type-specific behavior (rare). Machine name won't evaluate correctly if a fleet has mixed device types.

## Dockerfile Resolution Order

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

## Balena Arch vs Docker/Go Arch

Balena has its own architecture naming convention, distinct from Docker and Go:

| Balena (`%%BALENA_ARCH%%`) | Docker Platform | Go GOARCH |
|---|---|---|
| `aarch64` | `linux/arm64` | `arm64` |
| `amd64` | `linux/amd64` | `amd64` |
| `armv7hf` | `linux/arm/v7` | `arm` (GOARM=7) |
| `rpi` | `linux/arm/v6` | `arm` (GOARM=6) |
| `i386` | `linux/386` | `386` |

This matters because balenaCloud doesn't support multi-arch fleets yet — each fleet targets a single device type and architecture. Fleets and blocks on balena hub (bh.cr) are typically published as separate images per balena arch (e.g., `my-block-aarch64`, `my-block-amd64`), not as multi-platform manifests. Templates with `%%BALENA_ARCH%%` map directly to these naming conventions.

## The Simple Pattern (Use This by Default)

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

## Multi-Stage Pattern

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

### Supported Architecture Values

| `%%BALENA_ARCH%%` | Devices |
|---|---|
| `aarch64` | Raspberry Pi 3/4/5 (64-bit), Jetson, most modern ARM boards |
| `armv7hf` | Raspberry Pi 2/3 (32-bit), BeagleBone |
| `amd64` | Intel NUC, generic x86_64 |
| `rpi` | Raspberry Pi 1 / Zero (armv6) |

## docker-compose.yml Integration

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

## Common Patterns Reference

Read `references/patterns.md` for:
- Legacy `%%RESIN_MACHINE_NAME%%` syntax (pre-2019 projects)
- Balenalib base images (`FROM balenalib/%%BALENA_MACHINE_NAME%%-alpine`)
- `install_packages` helper in balenalib images
- Cross-compilation with `TARGETARCH` fallback
- The `INITSYSTEM on` ENV (legacy balena supervisor integration)
- bh.cr (balena hub container registry) image references
- Renovate configuration patterns for auto-updating image tags

## Debugging Build Failures

| Symptom | Likely Cause |
|---------|-------------|
| `invalid reference format` | Template variable not substituted — check file is named `.template` |
| `manifest unknown` | Architecture not published for that image tag |
| `no match for platform` | Using a plain Dockerfile where a template is needed |
| Wrong arch binary runs | `%%BALENA_MACHINE_NAME%%` resolved to fleet default, not target device |

## Workflow

When creating or modifying balena Dockerfile templates:

1. Check `balena.yml` → what device types does this fleet target?
2. Map device types to architectures (see table above)
3. For each service: does its base image support multi-platform? If yes → plain Dockerfile. If no → Dockerfile.template with the simple `%%BALENA_ARCH%%` pattern.
4. Only use the multi-stage pattern if you need per-arch version pinning for Renovate/Dependabot.
5. Include all supported architectures in your template.
6. Test with `balena build --deviceType <type>` for each target.
