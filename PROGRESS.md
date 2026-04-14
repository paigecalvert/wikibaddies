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

## Step 11: Helm upgrade with correct ingressClassName (in progress)

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
