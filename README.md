# homelab-k8s
Infrastructure as Code for my homelab kubernetes cluster.

## Storage

- **Longhorn** is managed via Flux under `k8s/infrastructure/longhorn/`.
- For operator workflow (including SOPS + age and Cloudflare R2 backups), see `specs/002-longhorn-r2-backup/quickstart.md`.
