# homelab-k8s

[![Renovate enabled](https://img.shields.io/badge/renovate-enabled-brightgreen.svg)](https://renovatebot.com)

Infrastructure as Code for my homelab kubernetes cluster.

## Storage

- **Longhorn** is managed via Flux under `k8s/infrastructure/longhorn/`.
- For operator workflow (including SOPS + age and Cloudflare R2 backups), see `specs/002-longhorn-r2-backup/quickstart.md`.

## Development

### K8s Manifest Checks

This repository includes automated lint and security checks for Kubernetes manifests:
- **Local**: Run `task check` to validate manifests before committing
- **CI**: Automatic checks run on Pull Requests via GitHub Actions

For setup and usage, see [docs/k8s-lint-security.md](docs/k8s-lint-security.md).

### Dependency Updates

This repository uses [Renovate](https://renovatebot.com) for automated dependency updates:
- **Helm Charts**: HelmRelease versions in `k8s/infrastructure/`
- **GitHub Actions**: Action versions in `.github/workflows/`

For configuration and manual upgrade procedures, see [docs/renovate.md](docs/renovate.md).
