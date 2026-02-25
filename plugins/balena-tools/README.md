# balena-tools

Claude Code skills for building and deploying balena fleet projects.

## Skills

### balena-dockerfile-templates

Guide for writing `Dockerfile.template` files for balena multi-architecture fleet deployments. Covers:

- Template variable substitution (`%%BALENA_ARCH%%`, `%%BALENA_MACHINE_NAME%%`, etc.)
- Dockerfile resolution order and when to use templates vs plain Dockerfiles
- Balena vs Docker/Go architecture naming conventions
- Multi-stage pattern for arch naming mismatches and Renovate pinning
- docker-compose.yml integration with balena-specific labels
- Debugging common build failures
