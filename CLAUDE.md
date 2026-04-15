# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This repository contains a Helm chart for onboarding [Outline](https://www.getoutline.com/) (a team knowledge base) onto the [Replicated Platform](https://www.replicated.com/). The chart lives in `outline/` and is based on the community Helm chart with Replicated-specific additions layered on top.

The goal is to complete the tasks in `RUBRIC.md` across three tiers:
- **Tier 0**: Helm installation, Replicated SDK integration, image proxying via `proxy.replicated.com`, preflight checks
- **Tier 1**: Embedded cluster (VM-based) install, upgrades, air-gap bundles, external dependency toggles, auth methods, integrations, config screen
- **Tier 2**: Production readiness (branding, license entitlements, support bundles, custom domains)

## Chart Architecture

The chart (`outline/`) wraps the `outlinewiki/outline` container and declares these subcharts via `Chart.yaml`:

| Subchart | Condition | Purpose |
|---|---|---|
| `redis` (bitnami) | `redis.enabled` | Embedded Redis; disable for external Redis via `externalRedis.*` |
| `postgresql` (bitnami) | `postgresql.enabled` | Embedded Postgres; disable for external DB via `externalPostgresql.*` |
| `minio` (minio) | `minio.enabled` | Embedded S3-compatible storage; disable for external S3 via `fileStorage.s3.*` |
| `replicated` | always | Replicated SDK sidecar (must be named `outline-sdk` per rubric task 0.3) |

Subchart archives are vendored in `outline/charts/`.

### Key configuration patterns in `values.yaml`

- **File storage**: `fileStorage.mode` is `local` or `s3`. When `s3`, Outline reads from either the embedded MinIO or external S3 credentials.
- **Auth**: Each provider (`slack`, `google`, `azure`, `oidc`, `github`, `gitea`, `gitlab`, `auth0`, `keycloak`, `discord`, `saml`) has an `enabled` flag. OIDC-based providers (gitea, gitlab, auth0, keycloak, oidc) all map to the same `OIDC_*` env vars — only one can be active at a time.
- **Secrets**: The chart generates three Kubernetes Secrets at render time: `-auth-secret`, `-smtp-secret`, `-integrations-secret`, all mounted via `envFrom`. The `SECRET_KEY` and `UTILS_SECRET` are auto-generated once and stored in a pre-install hook secret to survive upgrades.
- **Ingress**: If `ingress.enabled`, the `URL` env var is derived from the first ingress host. Otherwise it defaults to the service ClusterIP URL.

### Template files

- `templates/deployment.yaml` — main Outline deployment with all env vars assembled inline
- `templates/secret.yaml` — generates multiple secrets (DB, Redis, S3, auth, SMTP, integrations)
- `templates/configmap.yaml` — mounts the `entrypoint.sh` script
- `templates/files/entrypoint.sh` — custom entrypoint handling Redis auth
- `templates/_helpers.tpl` — named templates for host/port resolution of embedded vs external dependencies

## Replicated Platform Operations

### Creating and pushing a release

```bash
# Package the chart. Use -u if dependencies need to be updated
helm package outline/
# Remove the old chart archive from manifests, then move the new one in
rm manifests/outline-*.tgz
mv outline-*.tgz manifests/
# Create release from the manifests directory
# Create without promoting (default - promote separately when ready)
replicated release create --yaml-dir manifests --app outline-feline
# Promote a specific sequence to a channel
replicated release promote <sequence> "Development (Paige)" --app outline-feline
```

### Creating a customer

```bash
replicated customer create --name "Test Customer" --channel Unstable
```

### Proxying images through `proxy.replicated.com`

All `image.repository` values must be updated to use `proxy.replicated.com/proxy/<app-slug>/` as the prefix. The upstream registry must also be registered in the Vendor Portal under **Images → External Registries** — the proxy returns 404 for any registry not explicitly configured, even if `enterprise-pull-secret` is present.

**Pull secret wiring:** The Replicated SDK creates `enterprise-pull-secret` automatically. It must be referenced per-chart since the charts have different conventions:
- Main Outline pod: `imagePullSecrets: [{name: enterprise-pull-secret}]`
- Redis: `redis.image.pullSecrets: [enterprise-pull-secret]` (Bitnami string format)
- PostgreSQL: `postgresql.image.pullSecrets: [enterprise-pull-secret]` (Bitnami string format)
- MinIO: `minio.imagePullSecrets: [{name: enterprise-pull-secret}]` (object format)
- SDK: no pull secret needed — `proxy.replicated.com/library/` images are public

Do not use `global.imagePullSecrets` — the SDK, MinIO, and Bitnami charts all expect different formats and conflict with each other.

### Preflight checks

Preflight specs are embedded inside a `kind: Secret` with the label `troubleshoot.sh/kind: preflight`. The `kubectl preflight` plugin discovers specs from Secrets with this label — no CRD installation required. The spec itself is stored under `stringData.preflight.yaml` using `v1beta2`.

Run preflights against the local chart (useful for testing failure cases with `--set`):
```bash
helm template outline ./outline/ [--set ...] | kubectl preflight -
```

Run preflights against a release in the OCI registry:
```bash
helm template oci://registry.replicated.com/outline-feline/unstable/outline \
  --version <version> [--set ...] | kubectl preflight -
```

Requires `kubectl preflight` plugin >= v0.100 for `ingressClass` analyzer support. Upgrade via:
```bash
kubectl krew upgrade preflight
```

### Embedded cluster

TBD

### Support bundles

Support bundle spec is a `kind: SupportBundle` resource. Analyzers use the Troubleshoot.sh framework.

```bash
kubectl support-bundle <spec-url-or-file>
```

## Local Helm Development

### Install with embedded dependencies (default)

```bash
helm dependency update outline/
helm install outline ./outline/ -n outline --create-namespace
```

### Install with external PostgreSQL

```bash
helm install outline ./outline/ -n outline --create-namespace \
  --set postgresql.enabled=false \
  --set externalPostgresql.host=<host> \
  --set externalPostgresql.port=5432 \
  --set externalPostgresql.database=outline \
  --set externalPostgresql.username=outline \
  --set externalPostgresql.password=<password>
```

### Install with external Redis

```bash
helm install outline ./outline/ -n outline --create-namespace \
  --set redis.enabled=false \
  --set externalRedis.host=<host> \
  --set externalRedis.port=6379 \
  --set externalRedis.password=<password>
```

### Upgrade a release

```bash
helm upgrade outline ./outline/ -n outline
```

### Render templates locally (dry-run)

```bash
helm template outline ./outline/ --debug
```

### Verify pod images use the proxy

Nested jsonpath ranges don't parse correctly — use two separate queries instead:
```bash
kubectl get pods -n outline -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u
kubectl get pods -n outline -o jsonpath='{.items[*].spec.initContainers[*].image}' | tr ' ' '\n' | sort -u
```

## GCP Test Environment

When creating a GCP VM for embedded cluster testing, use an instance with:
- OS: Ubuntu 22.04 LTS
- At least 4 vCPU, 8 GB RAM (Outline + embedded cluster overhead)
- Firewall rules: allow ports 80, 443, 30080 (install/upgrade wizard), 50000 (Local Artifact Mirror)

**EC v3 has no persistent Admin Console.** There are ephemeral wizards (install wizard, upgrade wizard) accessible at port 30080 only during those processes. Once complete, the wizard is gone.

Check embedded cluster pod status:
```bash
sudo k0s kubectl get pods -A
```

Access the install/upgrade wizard at `http://<vm-ip>:30080` (only available during install or upgrade).
Access Outline (after install) at `https://outline.wikibaddies.com` (preferred — Cloudflare DNS A record points to VM IP).
For quick testing without updating DNS, use `https://<vm-ip>.nip.io` as the hostname instead.
