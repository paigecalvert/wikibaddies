# Outline Replicated Bootcamp Rubric

## Tier 0: Ship It with Helm

| # | Task | Pts | Acceptance Criteria | Notes | P&M Notes |
|---|------|-----|---------------------|-------|-----------|
| 0.1 | Outline installs via `helm install` and is accessible | 2 | `kubectl get pods -n <namespace>` shows all pods Running; open Outline in a browser | | |
| 0.2 | Replicated SDK included as a subchart and running | 1 | `kubectl get pods -n <namespace>` shows the SDK pod in Running state; show your `Chart.yaml` confirming the SDK is declared as a dependency | | |
| 0.3 | SDK renamed for branding | 1 | `kubectl get deployment outline-sdk -n <namespace>` succeeds | The deployment must be named `outline-sdk` | https://replicated.slack.com/archives/C0AQESYSE4B/p1775573979630539 |
| 0.4 | All container images proxied through `proxy.replicated.com` | 2 | Run `kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{range .spec.initContainers[*]}{.image}{"\n"}{end}{end}' \| sort -u` and show every app image starts with `proxy.replicated.com` | | |
| 0.5 | 3+ preflight checks covering distinct deployment concerns with clear, actionable pass/warn/fail messages | 4 | Show preflights running twice: once with a condition that causes a check to fail (e.g. password too short, invalid external DB credentials), then again with all checks passing. Failure messages must explain what went wrong and how to fix it. | Prefer checks that validate more complex deployment concerns like database connectivity to trivial checks like CPU/memory. | |

**Tier 0 total: 10 pts**

---

## Tier 1: Ship It on a VM

| # | Task | Pts | Acceptance Criteria | Notes | P&M Notes |
|---|------|-----|---------------------|-------|-----------|
| 1.1 | Outline installs on a bare VM using embedded cluster and is accessible | 3 | Starting from a fresh VM, complete the embedded cluster install. Show `sudo k0s kubectl get pods -A` with all pods Running, then open Outline in a browser. | | |
| 1.2 | In-place upgrade without data loss | 3 | Install release 1. Create a document in Outline. Trigger the upgrade to release 2 via the Admin Console. Show the document still present and all pods Running after upgrade. | | |
| 1.3 | Air-gapped install | 3 | Build an air gap bundle from your release. Transfer it to a VM. Complete the install using only the bundle — including the embedded dependencies (cert-manager, ingress-nginx). Show all pods Running with `sudo k0s kubectl get pods -A` and open Outline's login page in a browser. | Online network access during the demo is fine because authenticating in Outline is hard without online access. Ensure all necessary images are included in the air gap bundle. | |
| — | Config screen has at least 3 meaningful capabilities wired through to Helm | — | *Required threshold — no points on its own. Points come from the tables below.* | | |

### Dependencies

| # | Task | Pts | Acceptance Criteria | Notes | P&M Notes |
|---|------|-----|---------------------|-------|-----------|
| 1.4 | External PostgreSQL toggle | 1 | Install twice: once with embedded Postgres (show a postgres pod Running in `sudo k0s kubectl get pods -A`), once with external Postgres (show no postgres pod — Outline is using the external DB). | | |
| 1.5 | External Redis toggle | 1 | Install twice: once with embedded Redis (show a redis pod Running in `sudo k0s kubectl get pods -A`), once with external Redis (show no redis pod — Outline is using the external instance). | | |
| 1.6 | External S3-compatible storage toggle | 1 | Install twice: once with embedded MinIO (show a minio pod Running in `sudo k0s kubectl get pods -A`), once with external S3-compatible storage (show no minio pod — Outline is using the external bucket). | Any S3-compatible endpoint works (AWS S3, MinIO, etc.). | |

### Auth Methods

| # | Task | Pts | Acceptance Criteria | Notes | P&M Notes |
|---|------|-----|---------------------|-------|-----------|
| 1.7 | Magic link / email | 2 | On the Outline login page, enter an email address. Show the magic link email arriving and successfully logging in. | Requires a working SMTP server. Use a personal Gmail address (not your Replicated email) — Gmail's SMTP settings work reliably for this. | |
| 1.8 | Slack OAuth | 1 | Click "Sign in with Slack" on the Outline login page and complete the flow, landing on the Outline dashboard as an authenticated user. | You can create a free Slack workspace at https://slack.com for testing. | |
| 1.9 | Auth0 (OIDC) | 1 | Show the Auth0 login option on the Outline login page and complete the login flow end-to-end. | Free developer account at https://auth0.com | |
| 1.10 | GitLab (OIDC) | 1 | Show the GitLab login option on the Outline login page and complete the login flow end-to-end. | Free account at https://gitlab.com | |
| 1.11 | Gitea (OIDC) | 1 | Show the Gitea login option on the Outline login page and complete the login flow end-to-end. | Free account at https://gitea.com | |

### Integrations

| # | Task | Pts | Acceptance Criteria | Notes | P&M Notes |
|---|------|-----|---------------------|-------|-----------|
| 1.12 | Iframely rich embeds | 3 | Show the iframely pod Running in `kubectl get pods -n <namespace>`. In an Outline document, paste a YouTube or Twitter/X URL and show it rendering as a rich embedded preview. | No official Helm chart exists for iframely, so you'll need to write one from scratch. It must deploy the iframely container and expose it so it's reachable by Outline. | |
| 1.13 | Slack integration — slash commands | 1 | In Slack, run `/outline <term>` and show a result returned from a document you created in your Outline instance. | You can create a free Slack workspace at https://slack.com for testing. | |
| 1.14 | Slack integration — message actions | 1 | Open the context menu on a Slack message (hover → "More actions"), show an Outline action available, invoke it, and show the result (e.g. the message content saved to an Outline document). | Same Slack workspace as above. | |
| 1.15 | Slack integration — link unfurling | 1 | Paste an Outline document URL into Slack and show it rendering as a rich preview with the document title and excerpt. | Same Slack workspace as above. | |
| 1.16 | Linear link unfurling | 1 | Paste a Linear issue URL into an Outline document and show it rendering as a rich link preview. | Free account at https://linear.app | |
| 1.17 | GitHub App link previews | 1 | Paste a GitHub URL (repo, issue, or PR) into an Outline document and show the link preview rendering. | Free account at https://github.com | |
| 1.18 | Sentry error tracking | 1 | Configure a Sentry DSN. Scale the database to 0 replicas, then attempt to load Outline. Show the resulting error appearing in the Sentry dashboard. | Free account at https://sentry.io. Scaling the database to 0 is a reliable way to trigger a visible error. | |

### Config Screen Enhancements

| # | Task | Pts | Acceptance Criteria | Notes | P&M Notes |
|---|------|-----|---------------------|-------|-----------|
| 1.19 | Generated default value | 1 | Leave the embedded DB password blank in the config screen and show Outline installs and runs successfully — the password was auto-generated. Then perform an upgrade and show all pods still Running — confirming the password was not regenerated. | A common failure: the generated value changes on upgrade, so outline fails to connect to the DB on upgrade. | |
| 1.20 | Input validation | 1 | Attempt to proceed with an invalid config value and show the config screen blocking progress with a clear validation message. | | |

**Tier 1 total: 9 pts fixed + up to 15 pts from capabilities tables**

---

## Tier 2: Production Readiness

| # | Task | Pts | Acceptance Criteria | Notes | P&M Notes |
|---|------|-----|---------------------|-------|-----------|
| 2.1 | App icon and name set correctly | 1 | Screenshot of the KOTS Admin Console showing the correct Outline icon and app name. | | |
| 2.2 | Enterprise portal branding & identity | 1 | Screenshot of the Enterprise Portal showing at minimum: custom logo, favicon, title, and primary/secondary colors applied. | | |
| 2.3 | Enterprise portal custom email sender | 1 | Trigger an invitation email and show it arriving from your domain, not a Replicated address. | You'll need a domain you own, or you can purchase one and expense it. | |
| 2.4 | Enterprise portal install instructions | 1 | In the Enterprise Portal, show per-channel pre- or post-install instructions rendered on the Install page. Instructions must be meaningful (e.g. prerequisites, required firewall rules, post-install configuration steps) — not placeholder text. | | |
| 2.5 | Enterprise portal self-serve sign-up | 1 | Share the sign-up URL, complete the sign-up flow as a new user, and show the resulting customer record appearing in the Vendor Portal Customers page. | | |
| 2.6 | License entitlement that gates a real product feature | 3 | Show the license field defined in the Vendor Portal. Install with the entitlement disabled — show the feature is unavailable. Update the license to enable it — show the feature becomes available. | | |
| 2.7 | Support bundle with 2+ actionable analyzers | 3 | Run the support bundle and show the analyzer results with at least 2 checks surfacing meaningful pass/warn/fail states. Briefly explain what each analyzer checks and why it matters for debugging Outline. | | |
| 2.8 | Custom domains configured | 2 | Screenshot of all custom domains configured in the Vendor Portal. | You'll need a domain you own, or you can purchase one and expense it. | |

**Tier 2 total: 13 pts**
