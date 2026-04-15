# Onboarding Outline to Replicated — Progress Log

## Step 1: Understanding the project

Read `RUBRIC.md` to understand the three-tier rubric. The goal is to onboard the Outline knowledge base app onto the Replicated Platform, progressing through:
- **Tier 0**: Helm install, Replicated SDK, image proxying, preflight checks
- **Tier 1**: Embedded cluster, upgrades, air-gap, external dependency toggles, auth, integrations
- **Tier 2**: Production readiness (branding, entitlements, support bundles)

Created `CLAUDE.md` at the repo root to capture recurring commands and chart architecture for future sessions.

---

## Step 2: Rename the Replicated SDK deployment (Rubric task 0.3)

**What:** Added `nameOverride: "outline-sdk"` to the `replicated:` subchart values in `values.yaml`.

**Why:** The rubric requires the SDK deployment to be named exactly `outline-sdk` for branding purposes. The Replicated SDK subchart uses `nameOverride` to control its deployment name — without this, it defaults to `replicated`.

**Also fixed:** The `values.schema.json` had `additionalProperties: false` at the root, which rejected the new `replicated:` key. Added a schema entry for it.

**Verified with:**
```bash
helm template outline ./outline/ | awk '/replicated-deployment.yaml/{found=1} found && /^  name:/{print; found=0}'
# Output: name: outline-sdk
```

---

## Step 3: Fix the default URL value

**What:** Changed `url: "http://outline.localhost:3000"` to `url: ""` in `values.yaml`.

**Why:** The comment in the chart says the URL is auto-generated if not set — it derives from the ingress hostname or falls back to the internal service name. The `localhost:3000` value was a leftover from a previous session and also violated the schema's hostname pattern, causing `helm template` to fail validation.

---

## Step 4: First install attempt — GKE cluster with gVisor (failed)

**What:** Attempted `helm install` on an existing GKE cluster (`outline-bootcamp-test`).

**Why it failed:** All pods were stuck in `Pending`. Diagnosis via `kubectl get events` revealed:
```
0/2 nodes are available: 2 node(s) had untolerated taint(s)
```
The cluster had been created with GKE Sandbox enabled, which taints every node with `sandbox.gke.io/runtime=gvisor:NoSchedule`. Since the chart's pods have no matching tolerations, the scheduler couldn't place them anywhere. The autoscaler also couldn't help because the issue was taints, not capacity.

---

## Step 5: Create a new GKE cluster

**What:** Created a fresh cluster without gVisor.

**Why:** Rather than adding tolerations to every subchart (outline, redis, postgresql, minio, SDK), it was cleaner to just create a standard cluster. gVisor adds security sandboxing that isn't needed for a dev/test environment.

```bash
gcloud container clusters create outline-test-paige \
  --zone us-central1-a \
  --project replicated-qa \
  --num-nodes 1 \
  --machine-type e2-standard-4 \
  --labels owner=paige,expires=never
```

`e2-standard-4` (4 vCPU, 16GB) gives enough headroom for all the embedded dependencies. Single node is fine for a smoke test.

---

## Step 6: Install Outline

```bash
helm install outline ./outline/ \
  --namespace outline \
  --create-namespace \
  --set service.type=LoadBalancer
```

**Why `LoadBalancer`:** The chart defaults to `ClusterIP`, which isn't reachable from outside the cluster. Using `LoadBalancer` tells GKE to provision a GCP Network Load Balancer with a public IP — no ingress controller needed for a quick smoke test.

All pods came up `Running`:
- `outline` — the main app
- `outline-postgresql` — embedded Postgres
- `outline-redis-master` — embedded Redis
- `outline-minio` — embedded MinIO (unused for now, `fileStorage.mode` is still `local`)
- `outline-sdk` — Replicated SDK, correctly named

---

## Step 7: Diagnose browser error

**Symptom:** Browser showed "Loading failed. Sorry, part of the application failed to load."

**Diagnosis:** Checked logs:
```bash
kubectl logs -n outline deploy/outline | grep -i "error\|warn\|fail" | tail -30
```

Found Redis connection errors on startup (transient — Redis was still initializing). More importantly, the final log line showed:
```
"Listening on http://localhost:3000 / http://outline:3000"
```

`http://outline:3000` is the internal Kubernetes service name — unreachable from a browser. Outline uses the `URL` env var to construct asset URLs and redirects. Since `url` was left empty, it fell back to the service name.

**Fix:** Set `url` to the real external address. However, the LoadBalancer IP failed schema validation (the pattern requires a proper hostname with a TLD). This confirmed we needed a real domain and ingress setup rather than a raw IP.

---

## Step 8: Install ingress-nginx

Installed the ingress-nginx controller as a standalone Helm release (not a subchart — enterprise customers typically bring their own ingress controller):

```bash
helm install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

This provisioned a single LoadBalancer with external IP `34.61.46.40`. All ingress traffic for the cluster routes through this single IP; the controller fans it out to the right backend service based on the `Host` header in the request.

**Future plan:** Add ingress-nginx as an optional subchart with `ingress-nginx.enabled: false` by default, so customers can opt in if they don't have their own controller. This maps to rubric task 1 config screen requirements.

---

## Step 9: Configure DNS on Cloudflare

Added two A records (DNS only, no Cloudflare proxy) pointing to the ingress controller IP `34.61.46.40`:
- `outline.wikibaddies.com`
- `minio.wikibaddies.com`

**Why two records for the same IP:** Both hostnames point to the ingress controller. The controller routes based on the `Host` header — `outline.wikibaddies.com` goes to the Outline service, `minio.wikibaddies.com` goes to the MinIO service.

**Why MinIO needs its own hostname:** Outline generates presigned URLs for file uploads/downloads that the browser hits directly (not through Outline). MinIO needs to be publicly reachable at a stable hostname for this to work.

**Why DNS only (no Cloudflare proxy):** Cloudflare's proxy terminates and re-originates connections, which can interfere with the ingress controller's TLS termination and IP detection.

---

## Step 10: First helm upgrade attempt — ingress without ingressClassName (failed)

**What:** Ran `helm upgrade` to switch from LoadBalancer to ClusterIP, enable ingress, set the public URL, switch to S3 storage, and set the MinIO ingress hostname.

**Why it failed:** The upgrade applied but both ingresses had no `ingressClassName` set. GKE interpreted this as "use the default GKE ingress class" and provisioned two separate GCP L7 load balancers with their own IPs:
- Outline ingress: `34.8.26.6`
- MinIO ingress: `136.110.204.109`

The Cloudflare DNS records pointed to `34.61.46.40` (ingress-nginx), so requests hit ingress-nginx which had no matching rules — resulting in a 404.

**Additional issues discovered:**
- The `outline-minio-post-job` (which sets up the MinIO bucket, user, and policy) failed on startup because it raced ahead of MinIO being ready. Fixed by deleting the failed pod so the Job recreated it immediately.
- The v1 Endpoints API showed empty for the MinIO service (it's deprecated in Kubernetes 1.33+). The actual EndpointSlices were correctly populated — a red herring.

---

## Step 11: Helm upgrade with correct ingressClassName

**Fix:** Added `ingress.className=nginx` and `minio.ingress.ingressClassName=nginx` so both ingresses route through ingress-nginx instead of GKE's default ingress class.

```bash
helm upgrade outline ./outline/ \
  --namespace outline \
  --set service.type=ClusterIP \
  --set url=http://outline.wikibaddies.com \
  --set ingress.enabled=true \
  --set ingress.className=nginx \
  --set "ingress.hosts[0].host=outline.wikibaddies.com" \
  --set "ingress.hosts[0].paths[0].path=/" \
  --set "ingress.hosts[0].paths[0].pathType=Prefix" \
  --set fileStorage.mode=s3 \
  --set "minio.ingress.hosts[0]=minio.wikibaddies.com" \
  --set minio.ingress.ingressClassName=nginx
```

This should result in both ingresses using the ingress-nginx controller at `34.61.46.40`, matching the Cloudflare DNS records.

**Result:** All pods Running. Outline accessible at `http://outline.wikibaddies.com`. Rubric task 0.1 complete.

---

## Step 12: Proxy all images through proxy.replicated.com (Rubric task 0.4)

**What:** Rewrote all image refs in `values.yaml` to route through `proxy.replicated.com/proxy/outline-feline/`.

**Image ref format:** `proxy.replicated.com/proxy/<app-slug>/<upstream-registry>/<image>:<tag>`
- Docker Hub images use `index.docker.io` as the upstream registry component
- quay.io images use `quay.io`

**Images updated:**

| Image | New ref |
|---|---|
| Outline app | `proxy.replicated.com/proxy/outline-feline/index.docker.io/outlinewiki/outline` |
| Redis | `proxy.replicated.com/proxy/outline-feline/index.docker.io/bitnamilegacy/redis` |
| PostgreSQL | `proxy.replicated.com/proxy/outline-feline/index.docker.io/bitnamilegacy/postgresql` |
| MinIO server | `proxy.replicated.com/proxy/outline-feline/quay.io/minio/minio` |
| MinIO mc (post-job) | `proxy.replicated.com/proxy/outline-feline/quay.io/minio/mc` |
| Replicated SDK | `proxy.replicated.com/library/replicated-sdk-image` — already correct, Replicated-owned image |

**Gotchas encountered:**

- **Bitnami registry/repository split:** Bitnami charts (Redis, PostgreSQL) separate `image.registry` and `image.repository` into two distinct fields. Setting the full proxy URL in `repository` alone caused the template to prepend `registry-1.docker.io/` to the whole thing. Fix: set `image.registry` to the proxy prefix and keep `image.repository` as just the image name.

- **Bitnami image validation:** Recent Bitnami chart versions reject images not coming from their own registry, failing `helm template` with an error. Fix: set `global.security.allowInsecureImages: true`.

**Pull secret wiring:** The Replicated SDK automatically creates a pull secret named `enterprise-pull-secret` containing credentials for `proxy.replicated.com`. Pods need to reference it via `imagePullSecrets`:
- `imagePullSecrets: [{name: enterprise-pull-secret}]` — main Outline pod
- `global.imagePullSecrets: [enterprise-pull-secret]` — propagates to Redis and PostgreSQL via Bitnami's global convention
- `minio.imagePullSecrets: [{name: enterprise-pull-secret}]` — MinIO (different chart vendor, doesn't use Bitnami globals)

**Verified with:**
```bash
helm template outline ./outline/ | grep -E "^\s+image:" | sort -u
# All five app images show proxy.replicated.com/proxy/outline-feline/...
```

A new release was pushed to the Vendor Portal (Unstable channel) with these changes.

---

## Step 13: Add preflight checks (Rubric task 0.5)

**What:** Created `outline/templates/preflight.yaml` with four checks using the Troubleshoot.sh framework.

**Checks:**
1. **Kubernetes version** (always active) — fail if < 1.23.0
2. **Node memory** (always active) — fail if < 2Gi, warn if < 4Gi
3. **External PostgreSQL connectivity** (conditional on `postgresql.enabled=false`) — collector + analyzer pair
4. **External Redis connectivity** (conditional on `redis.enabled=false`) — collector + analyzer pair

**Secret wrapping:** Initially created the preflight as a standalone `kind: Preflight` resource (using `troubleshoot.sh/v1beta3`). This failed on install:
```
resource mapping not found for kind "Preflight" in version "troubleshoot.sh/v1beta3"
ensure CRDs are installed first
```

Fix: wrapped the spec inside a `kind: Secret` with the label `troubleshoot.sh/kind: preflight`. The `kubectl preflight` plugin discovers preflight specs from Secrets with this label — no CRD installation required.

**Final structure:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: "{{ .Release.Name }}-preflight-config"
  labels:
    troubleshoot.sh/kind: preflight
stringData:
  preflight.yaml: |
    apiVersion: troubleshoot.sh/v1beta2
    kind: Preflight
    ...
```

**Next:** Run `kubectl preflight` to test the checks, including a deliberate failure case (external PostgreSQL with wrong credentials).

**Update:** Preflight checks iterated significantly — see Step 15 for the full development history.

---

## Step 14: Fix imagePullSecrets format and wiring

**What:** Multiple issues with `imagePullSecrets` prevented pods from starting.

**Issue 1 — Wrong object format for SDK:** The Replicated SDK deployment template does a raw `toYaml` on `imagePullSecrets`. `global.imagePullSecrets` was set as a string list (`["enterprise-pull-secret"]`), which rendered as bare strings instead of `{name: ...}` objects. Kubernetes rejects bare strings for `imagePullSecrets` (`cannot unmarshal string into Go struct field PodSpec.spec.template.spec.imagePullSecrets of type v1.LocalObjectReference`).

**Issue 2 — SDK doesn't need a pull secret:** `proxy.replicated.com/library/replicated-sdk-image` is a public image — no authentication required. Only `proxy.replicated.com/proxy/<app-slug>/...` images need `enterprise-pull-secret`.

**Fix:** Dropped `global.imagePullSecrets` entirely and wired pull secrets per-chart:

| Workload | Setting | Format |
|---|---|---|
| Outline main pod | `imagePullSecrets` | object list (`{name: ...}`) |
| Redis | `redis.image.pullSecrets` | string list — Bitnami wraps in `name:` internally |
| PostgreSQL | `postgresql.image.pullSecrets` | string list — same |
| MinIO | `minio.imagePullSecrets` | object list (MinIO uses `toYaml` directly) |
| SDK | none | public image, no secret needed |

**Issue 3 — MinIO images on quay.io:** MinIO's images were initially proxied from `quay.io/minio/minio` and `quay.io/minio/mc`. This would have required adding quay.io as a second external registry in the Vendor Portal. Since `minio/minio` and `minio/mc` are also available on Docker Hub, switched both to `index.docker.io/minio/minio` and `index.docker.io/minio/mc` — one registry to configure instead of two.

**Issue 4 — Vendor Portal proxy registry not configured:** Even with `enterprise-pull-secret` present, pods failed with `404 Not Found` from `proxy.replicated.com/token`. Root cause: the upstream registry (`index.docker.io`) was not registered in the Vendor Portal under the app's proxy registry settings. The proxy only forwards pulls for registries explicitly configured per app. Fix: add `index.docker.io` with Docker Hub credentials (username + Personal Access Token) in the Vendor Portal.

**Verified with:**
```bash
helm template outline ./outline/ | grep -E "^\s+image:" | sort -u
# All five app images: proxy.replicated.com/proxy/outline-feline/index.docker.io/...
# SDK: proxy.replicated.com/library/replicated-sdk-image (no pull secret)
```

```bash
# Check actual pull secrets per workload after install:
kubectl get pods -n outline -o jsonpath='{.items[*].spec.containers[*].image}' | tr ' ' '\n' | sort -u
kubectl get pods -n outline -o jsonpath='{.items[*].spec.initContainers[*].image}' | tr ' ' '\n' | sort -u
```

---

## Step 15: Iterate preflight checks

**Final check inventory** (7 checks total):

| Check | Condition | Type |
|---|---|---|
| Kubernetes Version | always | fail < 1.23.0 |
| Schedulable Nodes | always | fail if count < 1 |
| Node Memory | always | fail < 2Gi, warn < 4Gi |
| Default Storage Class | any embedded dep enabled | fail if no default StorageClass |
| Ingress Class | `ingress.enabled=true` | fail if class not found |
| External PostgreSQL | `postgresql.enabled=false` | fail if not connected |
| External Redis | `redis.enabled=false` | fail if not connected |

**Gotchas and iterations:**

**Secret wrapping (v1beta2 not v1beta3):** Switched from standalone `kind: Preflight` (requires CRD) to embedding spec inside a `kind: Secret` with label `troubleshoot.sh/kind: preflight`. Also switched from `v1beta3` to `v1beta2` to match the Secret-embedded pattern.

**Explicit pass conditions:** Initially the postgres and redis pass outcomes had no `when` clause, making them catch-alls. Fixed to use `when: "connected == true"` with a fallback `warn` for unexpected states — prevents a false "Successfully connected" if the collector returns an unexpected result.

**tcpPortStatus is a host collector:** Attempted to split the postgres check into separate hostname/port/credentials checks using `tcpPortStatus`. This analyzer runs on the node itself (checks ports on the host machine), not against external hosts from within the cluster. Not suitable for external service checks. Reverted to a single postgres analyzer with a detailed error message.

**nodeResources `count(nodeCondition == Ready)` is invalid:** The `nodeResources` analyzer doesn't support `count(nodeCondition == Ready)` in the `when` clause. Replaced with `count() < 1` to check that at least one node exists.

**`ingressClass` analyzer requires preflight plugin >= ~v0.100:** The `ingressClass` analyzer caused "nonexistent analyzer" errors with plugin v0.75. Fixed by upgrading to v0.126.1 via `kubectl krew upgrade preflight`.

**Storage Class check is conditional on embedded deps:** The `storageClass` check only renders when at least one of `postgresql.enabled`, `redis.enabled`, or `minio.enabled` is true — those are the workloads that create PVCs. If all three are external, no StorageClass is needed.

**Ingress Class check handles specific vs default class:** When `ingress.className` is set, the check verifies that specific class exists. When not set, it falls back to checking for any default IngressClass. Error messages are templated accordingly.

**Testing failure cases:**
```bash
# Ingress class not found
helm template outline ./outline/ \
  --set ingress.enabled=true \
  --set ingress.className=does-not-exist \
  | kubectl preflight -

# External PostgreSQL unreachable
helm template outline ./outline/ \
  --set postgresql.enabled=false \
  --set externalPostgresql.host=fake-host.example.com \
  --set externalPostgresql.password=wrong \
  | kubectl preflight -

# External Redis unreachable
helm template outline ./outline/ \
  --set redis.enabled=false \
  --set externalRedis.host=fake-host.example.com \
  --set externalRedis.password=wrong \
  | kubectl preflight -
```

---

## Step 16: Embedded Cluster v3 manifests (Rubric tasks 1.1–1.3)

**What:** Created three manifest files in `manifests/` to support Embedded Cluster v3 installs, upgrades, and air-gap.

---

### `embedded-cluster-config.yaml`

```yaml
apiVersion: embeddedcluster.replicated.com/v1beta1
kind: Config
```

**Purpose:** Defines the EC v3 cluster configuration — version, extensions, and optional k0s overrides.

**Design decisions:**

- **ingress-nginx as an extension:** EC extensions install Helm charts before the application, making ingress-nginx available cluster-wide before Outline deploys. This is cleaner than asking customers to install their own ingress controller.

- **NodePort on 80/443:** For VM installs there's no cloud load balancer. NodePort binds directly to the VM's network interface on ports 80 and 443 — the VM's IP is the ingress entry point. No post-install IP discovery needed; the customer already knows their VM's IP before starting.

- **`ReplicatedImageName` template function:** Used for all ingress-nginx image references. This tells EC to rewrite those image refs and include the images in the EC air-gap bundle automatically. Extension images are bundled with EC itself (not the app bundle) — the `builder` key in the HelmChart v2 does not cover them.

- **Networking flow:**
  ```
  Internet → VM IP:80/443 → ingress-nginx (NodePort) → Outline service (ClusterIP)
  ```

---

### `kots-config.yaml`

```yaml
apiVersion: kots.io/v1beta1
kind: Config
```

**Purpose:** Defines the Admin Console config screen shown to customers after installing EC.

**Config groups and items:**

| Group | Item | Type | Notes |
|---|---|---|---|
| General | `hostname` | text (required) | The domain/IP customers use to reach Outline. Drives `url` and ingress host. |
| Database | `postgres_type` | select_one | `embedded_postgres` (default) or `external_postgres` |
| Database | `external_postgres_*` | text/password | Host, port, database, username, password — shown only when external selected |
| Cache | `redis_type` | select_one | `embedded_redis` (default) or `external_redis` |
| Cache | `external_redis_*` | text/password | Host, port, password — shown only when external selected |

**Design decisions:**

- **Single hostname field:** Drives both `url` and `ingress.hosts[0].host`. Customers either point a DNS A record at the VM IP or use `<IP>.nip.io` for testing (no DNS setup required).

- **SMTP omitted for now:** Will be added when tackling auth (task 1.7). The minimum viable config screen needs 3 meaningful capabilities — hostname + postgres + redis covers that.

- **File storage not exposed:** Defaults to `local` for embedded cluster. S3 toggle will be added when tackling external storage (task 1.6).

---

### `kots-helmchart.yaml`

```yaml
apiVersion: kots.io/v1beta2
kind: HelmChart
```

**Purpose:** Tells KOTS how to deploy the Outline Helm chart — maps config screen values to `values.yaml` and ensures all images land in air-gap bundles.

**Design decisions:**

- **`url` uses inline template:** `"http://repl{{ ConfigOption \`hostname\` }}"` — the `http://` prefix is a literal string, the template function is substituted inline. No need for the `print` template function.

- **`ingress.enabled` and `ingress.className` hardcoded:** Always `true` and `nginx` for embedded cluster installs. Not exposed as config options.

- **`service.type: ClusterIP` hardcoded:** The Outline service is internal — ingress-nginx (NodePort) handles all external traffic. `ClusterIP` is correct even on a VM.

- **`fileStorage.mode: local` hardcoded:** Avoids the MinIO presigned URL hostname problem (MinIO needs its own publicly reachable hostname, which adds DNS complexity for single-VM installs). S3 mode will be added as a config option for task 1.6.

- **`optionalValues` for external deps:** Uses `recursiveMerge: true` so external connection details merge into the existing values tree without overwriting sibling keys.

- **`builder` section:** Enables all three embedded dependencies (`postgresql`, `redis`, `minio`) so their images are always included in air-gap bundles regardless of what the customer selects at install time.

- **Air-gap image split:**
  - EC air-gap bundle: ingress-nginx images (via `ReplicatedImageName` in the EC config)
  - App air-gap bundle: Outline, PostgreSQL, Redis, MinIO images (via `builder` in HelmChart v2)

---

## Step 17: KOTS Application resource and v1beta3 preflight (Rubric tasks 1.1, 1.3)

### `kots-application.yaml`

Added a `kind: Application` resource for EC v3 branding:
- `title: Outline` — displayed in the Admin Console
- `icon` — base64-encoded SVG embedded directly (required for air-gap where external URLs aren't reachable)

### `manifests/preflight.yaml` (v1beta3)

EC v3 requires a standalone `troubleshoot.sh/v1beta3` preflight separate from the Helm chart's v1beta2 preflight. The v1beta2 preflight (inside the chart Secret) continues to serve the Helm CLI install path.

**Why two preflights:**
- v1beta2 in `outline/templates/preflight.yaml`: rendered by Helm, supports conditional collectors/analyzers via `{{- if .Values.xxx }}`
- v1beta3 in `manifests/preflight.yaml`: plain static YAML, processed by EC before install

**v1beta3 design:** Contains only the 5 always-true infrastructure checks (Kubernetes version, schedulable nodes, node memory, default storage class, ingress class nginx). The external postgres/redis checks were omitted because they require runtime config values that can't be expressed in static YAML. Those remain in the v1beta2 preflight for Helm CLI installs.

---

## Step 18: EC v3 install iteration

### Linter issues with Replicated template functions

**Problem:** `replicated release lint` reported `unable-to-render` errors for `ReplicatedImageName`, `ReplicatedImageRegistry`, and `ReplicatedImageRepository` template functions in both `embedded-cluster-config.yaml` and `kots-helmchart.yaml`. The linter evaluates these files as Helm/Go templates at lint time but doesn't know about KOTS runtime functions.

**Resolution:** Removed the image registry rewriting calls from both files. The `builder` key in the HelmChart CR still ensures images are included in air-gap bundles. Registry rewriting (for correct resolution in air-gap) is a TODO — requires linter support or acceptance of lint errors.

**Also fixed:** Redis and PostgreSQL `image.registry` values were restructured from `proxy.replicated.com/proxy/outline-feline/index.docker.io` (bundled path) to `proxy.replicated.com` (domain only) with the full proxy path moved into `image.repository`. This is the correct split for `ReplicatedImageRegistry/Repository` — both produce the same final image reference.

### EC installer UI port

EC v3 Admin Console is on port **30080** (not 8800 as in EC v2). Also requires port 50000 for the Local Artifact Mirror (LAM).

**GCP firewall rule:**
```bash
gcloud compute firewall-rules create outline-test-paige \
  --allow=tcp:22,tcp:80,tcp:443,tcp:30000,tcp:50000 \
  --target-tags=outline-test-paige
```

Access installer UI at `http://<vm-ip>:30080`.

### ingress-nginx chart archive must be bundled

EC extensions identify charts by `name` + `chartVersion` and look for a matching `.tgz` archive in the release bundle — there is no `chartURL` field in the EC Config schema. The ingress-nginx chart archive must be pulled and committed alongside the other manifests:

```bash
helm pull ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --version 4.11.3 \
  --destination manifests/
```

### `values` must be a YAML map, not a string

The EC Config `extensions.helmCharts[].values` field must be a YAML map (nested object), not a literal block string (`|`). Despite docs showing `values: |`, the EC Go struct unmarshals this field as `map[string]interface{}`. Using `|` produces a `!!str` type and causes:

```
unmarshal rendered spec for ingress-nginx: yaml: line 6: cannot unmarshal !!str `control...` into map[string]interface {}
```

**Fix:** Changed `values: |` → `values:` (plain YAML map).

### First successful EC install

- VM: GCP `e2-standard-4` (4 vCPU, 16 GB), Ubuntu 22.04, 50 GB SSD
- Accessed installer at `https://<vm-ip>:30080`
- Hostname set to `<vm-ip>.nip.io` (nip.io resolves `<IP>.nip.io` → IP, no DNS setup required)
- Embedded PostgreSQL + Redis selected (defaults)
- Outline accessible at `http://<vm-ip>.nip.io` ✓

---

## Step 19: Add TLS + Slack OAuth

**Goal:** Enable Slack OAuth for Outline authentication. Slack requires HTTPS for OAuth redirect URLs.

**Approach:** Add cert-manager as an EC extension and use Let's Encrypt HTTP-01 challenge to issue a certificate for the `<IP>.nip.io` domain (publicly resolvable → Let's Encrypt can validate).

**Changes:**

- **`embedded-cluster-config.yaml`:** Added cert-manager as an extension with `weight: -1` so it installs before ingress-nginx. Used `crds.enabled: true` (cert-manager v1.15+ syntax).

- **`outline/templates/cluster-issuer.yaml`:** New template that conditionally creates a `cert-manager.io/v1 ClusterIssuer` for Let's Encrypt. Uses `global.replicated.customerEmail` (injected by the Replicated SDK from the customer license) for the ACME registration email — no extra config field needed.

- **`outline/values.yaml`:** Added `certManager.createClusterIssuer: false` (safe default for Helm CLI installs without cert-manager).

- **`kots-helmchart.yaml`:** Set `certManager.createClusterIssuer: true` for KOTS/EC installs. Updated ingress to add `cert-manager.io/cluster-issuer: letsencrypt-http01` annotation and a `tls` block. Changed `url` from `http://` to `https://`.

**cert-manager chart archive:** Must be bundled in the release, same as ingress-nginx:
```bash
helm pull cert-manager \
  --repo https://charts.jetstack.io \
  --version v1.16.3 \
  --destination manifests/
```

**Note:** The installer UI at port 30080 retains its own EC-managed self-signed certificate — cert-manager only affects the Outline app's ingress certificate.

**TLS verification:** After upgrade, confirmed:
- `kubectl get certificaterequest,order,challenge -n outline` — CertificateRequest READY: True, Order valid, no pending challenges
- `kubectl describe clusterissuer letsencrypt-http01` — ACME account registered, ClusterIssuer Ready
- `outline-tls` secret created in the `outline` namespace
- Outline accessible at `https://<vm-ip>.nip.io` ✓

**Slack OAuth app setup:**
- Created a Slack app in a test workspace at api.slack.com
- Added redirect URL: `https://<vm-ip>.nip.io/auth/slack.callback` (requires HTTPS)
- Required User Token Scopes: `identity.basic`, `identity.email`, `identity.avatar`
- Obtained Client ID and Client Secret from the app's OAuth & Permissions page
- Config screen fields: `slack_client_id` (text) and `slack_client_secret` (password) in the Authentication group

---

## Step 20: Slack auth template bug fix + generated DB password (Rubric tasks 1.8, 1.19)

### Slack `if/else` delimiter bug

**Problem:** Upgrade failed with:
```
render template in kots-helmchart.yaml: failed to get template: ...:70: unexpected EOF
```

**Root cause:** The `auth.slack.enabled` value used a `{{ if }}...{{ else }}...{{ end }}` construct that mixed delimiter styles:
```yaml
enabled: repl{{ if and ... }}true{{ else }}false{{ end }}
```

KOTS uses `repl{{` as the left delimiter and `}}` as the right delimiter. The `}}` inside `{{ else }}` is parsed as a stray right delimiter (closing nothing), which leaves the `if` block open. The parser then hits EOF with an unclosed block.

**Fix:** All control flow actions must use the `repl{{` left delimiter consistently:
```yaml
enabled: repl{{ if and (ConfigOption "slack_client_id") (ConfigOption "slack_client_secret") }}truerepl{{ else }}falserepl{{ end }}
```

This matches the pattern shown in KOTS docs — use `repl{{ else }}` and `repl{{ end }}` (or `{{repl else }}` / `{{repl end }}` if using the `{{repl` style throughout).

### Generated embedded PostgreSQL password (Rubric task 1.19)

**Problem:** The embedded PostgreSQL password was hardcoded as `"password"` in `values.yaml`. This is the same for every install and doesn't survive upgrades cleanly.

**Fix:** Added a hidden config item with a KOTS-generated value:

```yaml
# kots-config.yaml
- name: embedded_postgres_password
  title: Embedded PostgreSQL Password
  type: password
  value: 'repl{{ RandomString 32 }}'
  when: 'repl{{ ConfigOptionEquals "postgres_type" "embedded_postgres" }}'
```

Using `value:` (not `default:`) is critical — KOTS evaluates this once on first install and stores the result. Subsequent upgrades reuse the stored value, so the password is stable across the lifecycle of the installation.

Wired into the HelmChart CR:
```yaml
postgresql:
  auth:
    password: repl{{ ConfigOption "embedded_postgres_password" }}
```

**Demo for rubric:** In the config screen, the Embedded PostgreSQL Password field is pre-populated with a 32-character random string. Leave it as-is, install, then upgrade — Outline continues to connect to the database with the same password.

**Important:** Changing the generated password on an existing install breaks PostgreSQL — the Secret is updated but the database still expects the old password. Always do a fresh install when the generated password changes (e.g. switching from a release without the field to one with it).

---

## Step 21: Support bundle (Rubric task 2.7)

Created `outline/templates/support-bundle.yaml` — a Secret with label `troubleshoot.sh/kind: support-bundle` (same discovery mechanism as the v1beta2 preflight). Helm template functions are used for resource names and namespace so the spec is always accurate regardless of release name.

**Collectors:**
- Outline app logs (up to 10,000 lines)
- PostgreSQL pod logs
- Redis pod logs
- Cluster resources (all namespaced resources in the outline namespace)

**Analyzers (3 checks):**

| Check | Pass | Fail |
|---|---|---|
| Outline Application | ≥1 ready pod | Absent or 0/1 — lists common causes (DB failure, missing SECRET_KEY, auth misconfiguration) |
| Embedded PostgreSQL | Absent (external in use) or ≥1 ready | 0/1 — directs to postgresql logs and PVC disk space check |
| Embedded Redis | Absent (external in use) or ≥1 ready | 0/1 — explains impact (real-time collab, sessions, job queue) |

**Why these checks matter for debugging Outline:**
- Outline fails to start entirely if either PostgreSQL or Redis is unavailable — so knowing which dependency is down immediately narrows the problem
- The Outline analyzer surfaces common startup errors (bad credentials, missing secrets) that otherwise require reading logs to find

**Run the support bundle:**
```bash
kubectl support-bundle --load-cluster-specs -n outline
```

---

## Step 22: File storage toggle (Rubric task 1.6)

Added a **File Storage** config group to `kots-config.yaml` with two options:

| Option | Effect |
|---|---|
| Local Disk (default) | `minio.enabled: false`, `fileStorage.mode: local` — files stored on a PV on the VM |
| Embedded MinIO (S3) | `minio.enabled: true`, `fileStorage.mode: s3` — files stored in MinIO, accessible via a second public hostname |

**MinIO hostname requirement:** When embedded MinIO is selected, a second hostname is required (e.g. `minio.1.2.3.4.nip.io` or `minio.outline.wikibaddies.com`). Outline generates S3 presigned URLs pointing to this hostname — browsers hit it directly for file uploads/downloads, so it must be publicly reachable.

**Why local disk is the default for EC:** Single-VM installs don't need object storage for basic use. Local disk avoids the complexity of a second DNS record and hostname. MinIO is still available as an opt-in for customers who need S3-compatible storage or plan to scale beyond a single node.

**builder section:** `minio.enabled: true` is kept in the `builder` block so MinIO images are always included in air-gap bundles, even when local disk is the default.

---

## Step 23: Air gap image rewriting (Rubric task 1.3)

Added `ReplicatedImageName`, `ReplicatedImageRegistry`, and `ReplicatedImageRepository` template functions so image refs are automatically rewritten at runtime — pointing to `proxy.replicated.com` for online installs and to the EC Local Artifact Mirror (LAM) for air-gap installs.

### Template function reference

| Function | Use case |
|---|---|
| `ReplicatedImageName (HelmValue ".path") true` | Single combined `repository` field that already contains the proxy prefix |
| `ReplicatedImageRegistry (HelmValue ".path")` | Split `registry` field — returns `proxy.replicated.com` online, LAM address in air-gap |
| `ReplicatedImageRepository (HelmValue ".path") true` | Split `repository` field that already contains the proxy path — returns unchanged online, LAM path in air-gap |

`noProxy: true` is used when the value in `values.yaml` already has the `proxy.replicated.com/proxy/<app-slug>/` prefix baked in. Without it, `ReplicatedImageName` would double-wrap the proxy prefix online (e.g. `proxy.replicated.com/proxy/outline-feline/proxy.replicated.com/...`). With it, online installs leave the value unchanged while air-gap installs still rewrite to the LAM.

### Changes in `kots-helmchart.yaml`

| Image | Function used | noProxy |
|---|---|---|
| Outline (`image.repository`) | `ReplicatedImageName` | `true` — full proxy URL already in value |
| MinIO (`minio.image.repository`, `minio.mcImage.repository`) | `ReplicatedImageName` | `true` — full proxy URL already in value |
| Redis (`redis.image.registry`) | `ReplicatedImageRegistry` | omitted — needs to switch registry host |
| Redis (`redis.image.repository`) | `ReplicatedImageRepository` | `true` — proxy path already in value |
| PostgreSQL (`postgresql.image.registry`) | `ReplicatedImageRegistry` | omitted — needs to switch registry host |
| PostgreSQL (`postgresql.image.repository`) | `ReplicatedImageRepository` | `true` — proxy path already in value |

### Changes in `embedded-cluster-config.yaml`

Added image rewriting for the ingress-nginx extension controller image. Upstream image refs are passed as literal strings (no `HelmValue` needed since EC extensions don't have a `values.yaml`):

```yaml
controller:
  image:
    registry: 'repl{{ ReplicatedImageRegistry "registry.k8s.io" }}'
    repository: 'repl{{ ReplicatedImageRepository "registry.k8s.io/ingress-nginx/controller" }}'
```

**Note:** The `values` field in EC Config extensions must be a YAML map (not `values: |` string format). Despite the docs showing the string format, the EC Go struct expects `map[string]interface{}` — using `|` causes an unmarshal error at install time.

### Linter behavior

`replicated release lint` reports `unable-to-render` errors for these functions because the linter doesn't know about EC v3 runtime functions. This is a known linter limitation — the functions are valid at runtime. The linter fails fast on the first unknown function per file, so only `ReplicatedImageName` and `ReplicatedImageRegistry` are flagged (not `ReplicatedImageRepository`, which appears on the next line after the failure).

### Inspecting the air-gap bundle

To verify the contents of an air-gap bundle without downloading the full file to disk, stream it through `tar`:

```bash
# List all files in the bundle
curl -sL "<presigned-download-url>" | tar -tzf -

# Extract and print airgap.yaml (lists all bundled images)
curl -sL "<presigned-download-url>" | tar -xzf - airgap.yaml -O
```

The bundle is a gzipped tar containing:
- `airgap.yaml` — manifest listing all images in the bundle
- `app.tar.gz` — the Helm chart(s)
- `images/docker/registry/v2/` — OCI image store with all container image layers

---

## Step 24: iFramely chart + license-gated deployment (Rubric task 1.12)

Created a custom `iframely/` Helm chart from scratch and wired it to a Replicated license field so it only deploys when the customer's license has `iframely: true`.

### iFramely chart (`iframely/`)

Minimal chart with two templates:

- `templates/deployment.yaml` — single-container Deployment, both wrapped in `{{- if .Values.enabled }}`
- `templates/service.yaml` — ClusterIP Service on port 8061, also gated by `{{- if .Values.enabled }}`

Both resources are no-ops when `enabled: false` (the default), so the chart can always be installed without deploying anything for unlicensed customers.

The service is named after the release (`iframely`) so it's reachable within the `outline` namespace at `http://iframely:8061` without any extra DNS or ingress configuration.

**Image:** `proxy.replicated.com/proxy/outline-feline/ghcr.io/iframely/iframely` — requires `ghcr.io` to be registered in the Vendor Portal under external registries.

### `manifests/kots-iframely-helmchart.yaml`

Separate HelmChart CR for the iFramely chart:

```yaml
values:
  enabled: repl{{ LicenseFieldValue "iframely" }}
  image:
    repository: 'repl{{ ReplicatedImageName (HelmValue ".image.repository") true }}'
builder:
  enabled: true  # always include image in air gap bundle
```

`LicenseFieldValue "iframely"` returns the string `"true"` or `"false"` from the license field. Rendered unquoted into YAML, this becomes a proper YAML boolean that Helm sees as `true`/`false` — so `{{- if .Values.enabled }}` works correctly in the chart templates.

### `manifests/kots-helmchart.yaml` — outline optionalValues

Added an entry that fires when the license field is `true`, pointing Outline at the iFramely service:

```yaml
- when: 'repl{{ LicenseFieldValue "iframely" }}'
  recursiveMerge: true
  values:
    integrations:
      iframely:
        enabled: true
        url: "http://iframely:8061"
```

The outline chart already has `integrations.iframely.url` and `integrations.iframely.enabled` values — no chart template changes needed.
