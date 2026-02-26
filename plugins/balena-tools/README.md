# balena-tools

Claude Code skills for building and deploying balena fleet projects.

## Skills

### balenify

Guide for converting Docker projects to run on balenaCloud. Covers:

- Full balenify checklist for converting existing Docker/docker-compose projects
- Dockerfile.template variable substitution (`%%BALENA_ARCH%%`, `%%BALENA_MACHINE_NAME%%`, etc.)
- Dockerfile resolution order and when to use templates vs plain Dockerfiles
- Balena vs Docker/Go architecture naming conventions
- Multi-stage pattern for arch naming mismatches and Renovate pinning
- Volume mount conversion (bind mounts → named volumes)
- Networking differences (host vs bridge, service discovery)
- Environment variable handling (no `.env` files, dashboard/CLI variables)
- Hardware access (privileged mode, device labels, feature labels)
- docker-compose.yml integration with balena-specific labels and update strategies
- Debugging common build failures
