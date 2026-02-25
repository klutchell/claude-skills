# Balena Dockerfile Template Patterns Reference

## Table of Contents

- [bh.cr Registry References](#bhcr-registry-references)
- [Legacy Balenalib Base Images](#legacy-balenalib-base-images)
- [Legacy Resin Syntax](#legacy-resin-syntax)
- [Cross-Compilation with TARGETARCH](#cross-compilation-with-targetarch)
- [INITSYSTEM ENV](#initsystem-env)
- [install_packages Helper](#install_packages-helper)
- [Renovate Configuration](#renovate-configuration)
- [Device Type to Architecture Mapping](#device-type-to-architecture-mapping)

---

## bh.cr Registry References

balena hub container registry (`bh.cr`) hosts blocks and fleet images. Images are published per-architecture using balena's arch naming convention.

**URL format:** `bh.cr/<org>/<block-name>-<arch>/<hash-or-nothing>:<version>`

Real examples:
```dockerfile
# Block reference with content hash (common for published blocks)
FROM bh.cr/gh_klutchell/tailscale-amd64/ebf61ab1515195f9df43aa64baf15c39:1.88.1

# Block reference without hash
FROM bh.cr/g_tomas_migone1/hostname-%%BALENA_ARCH%%/0.2.1

# Simple block reference
FROM bh.cr/balenalabs/fbcp/1.0.4
```

When using `%%BALENA_ARCH%%` in bh.cr references, the arch is part of the image path, not a tag:
```dockerfile
# Correct — arch in path
FROM bh.cr/some-org/my-block-%%BALENA_ARCH%%/1.0.0

# Wrong — arch as tag
FROM bh.cr/some-org/my-block:%%BALENA_ARCH%%-1.0.0
```

---

## Legacy Balenalib Base Images

Balenalib images are the older (deprecated) base images that use `%%BALENA_MACHINE_NAME%%` for device-type-specific image selection. Still present in many existing projects.

```dockerfile
# Alpine-based
FROM balenalib/%%BALENA_MACHINE_NAME%%-alpine:3.18

# With runtime
FROM balenalib/%%BALENA_MACHINE_NAME%%-node:18-bookworm

# Architecture-based (less common)
FROM balenalib/%%BALENA_ARCH%%-alpine
```

Balenalib images include the `install_packages` helper (see below) and were designed for balena's init system.

**For new projects**, use standard images (Alpine, Debian, etc.) or bh.cr blocks instead.

---

## Legacy Resin Syntax

Pre-2019 projects used `%%RESIN_MACHINE_NAME%%` and `%%RESIN_ARCH%%` (before the Resin → Balena rename):

```dockerfile
FROM resin/%%RESIN_MACHINE_NAME%%-alpine-golang

ENV INITSYSTEM on

WORKDIR /go/src/app
COPY . ./
RUN go build
CMD ./myapp
```

Both `RESIN_*` and `BALENA_*` variables still work, but use `BALENA_*` for anything new.

---

## Cross-Compilation with TARGETARCH

Some images use Docker's native `TARGETARCH` with a fallback to balena's arch:

```dockerfile
ARG BALENA_ARCH=%%BALENA_ARCH%%

FROM certbot/dns-cloudflare:amd64-v1.30.0 AS certbot-amd64
FROM certbot/dns-cloudflare:arm64v8-v1.30.0 AS certbot-arm64
FROM certbot/dns-cloudflare:arm64v8-v1.30.0 AS certbot-aarch64

FROM certbot-${TARGETARCH:-$BALENA_ARCH}
```

Note the aliasing: `certbot-arm64` AND `certbot-aarch64` both point to `arm64v8` because Docker uses `arm64` while balena uses `aarch64`.

---

## INITSYSTEM ENV

Legacy balenalib images supported enabling the balena supervisor's init system:

```dockerfile
ENV INITSYSTEM on
```

This is a legacy pattern — modern balena projects don't need it. It was used with balenalib base images to enable PID 1 process management.

---

## install_packages Helper

Balenalib base images include `install_packages`, a cross-distro package installer:

```dockerfile
FROM balenalib/%%BALENA_MACHINE_NAME%%-alpine:3.18
RUN install_packages curl jq
```

It abstracts over `apt-get`, `apk`, `dnf`, etc. based on the base distro. Only available in balenalib images — for standard base images, use the native package manager directly.

---

## Renovate Configuration

For projects using the multi-stage pattern (separate FROM per arch), Renovate can pin and auto-update each image independently.

**renovate.json example** (from a real balena-pihole project):
```json
{
  "extends": ["github>klutchell/renovate-config"],
  "packageRules": [
    {
      "matchManagers": ["dockerfile"],
      "matchPackageNames": ["pihole/pihole"],
      "matchUpdateTypes": ["major", "minor", "patch"],
      "postUpgradeTasks": {
        "commands": [
          "sed -e \"s|^version: .*$|version: {{{newVersion}}}|\" -e \"s|\\b0\\+\\([0-9]\\)|\\1|g\" -i balena.yml"
        ],
        "fileFilters": ["balena.yml"],
        "executionMode": "update"
      }
    }
  ]
}
```

This syncs the `version:` field in `balena.yml` when Renovate bumps the pihole image tag.

**Why multi-stage helps Renovate:** The simple `%%BALENA_ARCH%%` pattern puts a template variable inside the image reference, which Renovate can't parse. The multi-stage pattern puts each arch on its own clean FROM line that Renovate recognizes as a standard Docker image reference.

---

## Device Type to Architecture Mapping

Common device types and their `%%BALENA_ARCH%%` values:

### aarch64 (87 device types)
`raspberrypi3-64`, `raspberrypi4-64`, `raspberrypi5`, `jetson-nano`, `jetson-xavier-nx-devkit`, `genericaarch64`, `fincm3`, `nanopi-neo-air`, `coral-dev`, `imx8m-var-dart`, ...

### amd64 (13 device types)
`genericx86-64-ext`, `intel-nuc`, `generic-amd64`, `surface-go`, `surface-pro-6`, `up-board`, ...

### armv7hf (44 device types)
`raspberry-pi2`, `raspberrypi3`, `beaglebone-black`, `asus-tinker-board`, `ts4900`, `apalis-imx6q`, ...

### rpi (1 device type)
`raspberry-pi` (Pi 1 / Zero, armv6)

### i386 (2 device types)
`intel-edison`, `qemux86`

Full list available via the balena API:
```
https://api.balena-cloud.com/v7/device_type?$select=slug&$expand=is_of__cpu_architecture($select=slug)
```
