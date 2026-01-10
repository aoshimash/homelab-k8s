# Research: Vikunja Deployment

**Feature**: 014-deploy-vikunja  
**Date**: 2026-01-10

## 1. Deployment Packaging (Helm vs raw manifests)

### Decision
Use the official Vikunja Helm chart and deploy it via Flux `HelmRelease`.

### Rationale
- Reduces long-term manifest maintenance (resources, defaults, upgrades)
- Fits GitOps workflow already used for infrastructure Helm charts
- Chart supports external PostgreSQL and persistence configuration

### Alternatives Considered
| Alternative | Rejected Because |
|-------------|------------------|
| Raw Deployment/Service manifests | More maintenance and higher drift risk |
| `docker-compose` style approach | Not compatible with cluster GitOps patterns |

## 2. Network Exposure / Ingress

### Decision
Expose Vikunja only via the existing Tailscale Ingress pattern (trusted network/VPN only).

### Rationale
- Aligns with feature clarifications (no public exposure)
- Reuses existing, proven ingress pattern (`ingressClassName: tailscale`, `tailscale.com/proxy-group: ingress-proxies`)

### Alternatives Considered
| Alternative | Rejected Because |
|-------------|------------------|
| Public internet exposure | Not allowed by spec |
| Additional ingress controller | Unnecessary complexity |

## 3. User Registration / Identity

### Decision
Disable user self-registration; users are created by the operator/admin. No SSO for MVP.

### Rationale
- Matches clarifications
- Minimizes operational/security surface for a homelab deployment

### Implementation Notes (for later)
- Vikunja can disable registration via configuration (`service.enableregistration: false`) or environment (`VIKUNJA_SERVICE_ENABLEREGISTRATION=false`).
- When registration is disabled, users can be created via CLI inside the Vikunja container (operator action).

## 4. Database Choice & Provisioning

### Decision
Use the existing CloudNativePG PostgreSQL cluster (`k8s/configs/postgres/`) and manage:
- the Vikunja DB role declaratively via CNPG `spec.managed.roles` with `passwordSecret`
- the Vikunja database declaratively via CNPG `Database` CRD (owned by the Vikunja role)

### Rationale
- Keeps persistent state under GitOps control (no one-off imperative DB drift)
- Reuses existing scheduled backups to R2

### Alternatives Considered
| Alternative | Rejected Because |
|-------------|------------------|
| Chart-provisioned PostgreSQL | Duplicates existing database platform |
| Imperative manual SQL user/db creation | Creates drift outside Git (harder to audit/redo) |

## 5. What to Back Up

### Decision
Back up **both**:
- PostgreSQL database (CNPG ScheduledBackup already configured daily to R2)
- File/attachment storage directory (Longhorn volume backups daily via PVC annotation)

### Rationale
- Vikunja stores core data in DB, but attachments are filesystem-backed
- Meets spec requirement for daily backups of all user data

### Implementation Notes (for later)
- Longhorn StorageClass recurring jobs are disabled globally in this repo due to a Helm chart bug; enable recurring backups per-volume using the PVC annotation:
  `recurring-job-selector.longhorn.io: '[{"name":"backup-daily","isGroup":false}]'`

