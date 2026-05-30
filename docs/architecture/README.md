# Logical Infrastructure — Diagram Companion

## Purpose

This document accompanies the diagrams in this directory:

- `logical_architecture.drawio` — the static component view (who exists,
  who talks to whom).
- `day1-bootstrap.drawio` — the Day-1 bootstrap phases flow
  (`bootstrap.sh` phases 0-5 plus the Argo CD root App-of-Apps fan-out
  into the platform-component Applications).
- `day2-tenant-provisioning.drawio` — the Day-2 tenant onboarding
  flow (PR with a tenant claim → Argo CD → Crossplane → tenant live
  under `<tenant>.fhuebung.lol`).

It explains the components visible in the diagrams and the main data
flows — intended as a reading guide for reviewers, the lecturer, and
new team members.

Rationale for technology choices is **not** kept here, but in
[`docs/claude/architecture-decisions.md`](../claude/architecture-decisions.md).
If anything here conflicts with that document, `architecture-decisions.md`
wins. Items that are deliberately out of scope are listed in
[`docs/claude/project-overview.md`](../claude/project-overview.md#out-of-scope).

## How to read the diagram

### Colors

Boxes are coloured by area of responsibility:

- **Blue** — GCP managed services (Cloud DNS, Secret Manager, GCS,
  Loadbalancer). Operated by Google; configured via Terraform on
  our side.
- **Yellow** — Platform components inside the cluster. Reconciled by
  Argo CD from `platform-gitops`, shared by all tenants.
- **Green** — Per-tenant resources. One instance per tenant, in the
  tenant's own namespace. Provisioned by Crossplane.
- **Orange** — External sources of truth and CI: Git repositories,
  OCI registries, GitHub Actions.
- **Grey** — External services outside our control (Internet, domain
  registrar, ACME endpoint).

Dashed borders mark components that belong to the bonus task
(kube-prometheus-stack) or storage resources (PVCs).

### Arrows

- **Red, solid** — User/data path (the HTTP/HTTPS traffic of an end
  user passing through the system).
- **Blue, solid** — Control/sync path (e.g. Argo CD reconciling,
  Crossplane provisioning, ESO fetching).
- **Dashed grey** — Asynchronous pulls (image pulls, metrics scrapes,
  storage persistence, CI pushes).

### Container hierarchy

From outside in:
`GCP Project` → `VPC` → `GKE Standard (cluster)` → either
`Platform components` (yellow area) or
`Tenant Namespace tenant-[X]` (green area).

There is one green area per tenant; the diagram shows one tenant
explicitly, with a note indicating that further tenants
(`tenant-Y`, `staging`, …) follow the same structure.

## Bootstrap (Day 1 → Day 2)

The platform follows the Day 1 / Day 2 separation from the assignment.

**Day 1** is the one-time bootstrap, executed by the
`bootstrap/bootstrap.sh` script in the
[`platform-iac`](https://github.com/INENI-PT-GROUP-B/platform-iac)
repository, run **locally** by a team member. The script is idempotent
and proceeds in phases: preflight checks, GCP API enablement, creation
of the Terraform state bucket (resolving the chicken-and-egg problem of
needing a GCS bucket before `terraform init`), `terraform apply`
(provisioning VPC, GKE cluster, Cloud DNS zone bindings, IAM service
accounts), kubeconfig retrieval, and finally an Argo CD install via
Helm plus one root Argo CD `Application` pointing at `platform-gitops`.

**Day 2** starts the moment that root Application syncs for the first
time. From there on, Argo CD reconciles everything else (ESO,
ExternalDNS, cert-manager, Crossplane and its providers, Traefik,
kube-prometheus-stack, all XRDs, Compositions, and tenant claims) from
`platform-gitops`. The platform self-manages from
this point: changes to platform components or tenant configuration
go through pull requests against `platform-gitops`, never through
direct cluster interaction.

A second `bootstrap.sh` run is only needed for changes at the cluster
topology level (new node pool, VPC change, IAM role addition) — the
script is safe to re-run because every phase is idempotent.

### Day-1 bootstrap diagram

`day1-bootstrap.drawio` (with PNG export `.drawio.png`) shows the same
sequence as a swim-lane flow. Four lanes left-to-right — operator
workstation, GCP control plane, GKE cluster, platform-gitops — make
visible which actor owns each phase's output. Time flows top-to-bottom
through Phase 0 (preflight), Phase 1 (enable GCP APIs), Phase 2 (state
bucket + persistent DNS zone create-if-absent + `terraform init`), Phase 3
(`terraform apply` over the five child modules — network, cluster, dns,
iam, backup), Phase 4 (kubeconfig), Phase 5 (Argo CD install + root
App-of-Apps). All phases are implemented in the merged `bootstrap.sh`.

The post-bootstrap fan-out groups the Argo CD-reconciled platform
components into a single container. `kube-prometheus-stack` is drawn
dashed because it is still pending (S4-01 bonus); every other
Application has already been merged to `main`.

## Components in detail

### External sources and services

**Users / Internet** — the end user. Always enters over HTTPS on
port 443.

**Porkbun (Registrar)** — domain registrar for `fhuebung.lol`. Holds
the NS records that delegate to Google Cloud DNS. DNSSEC is
intentionally disabled because it would break delegation to Cloud DNS.

**Let's Encrypt** — ACME endpoint that issues our TLS certificates.
Contacted only by `cert-manager`.

**GHCR** (GitHub Container Registry) — hosts three artefacts:
- `app-backend` (public): the application backend image
- `app-frontend` (private): the application frontend image (private
  per assignment requirement)
- `app-chart` (public, OCI): the Helm chart that describes the app
  deployment. Sources live in `app-backend/chart/`; pushed by the
  `app-backend` release workflow. Pulled by Crossplane.

**platform-gitops** — Git repository that is the source of truth for
all cluster-side resources. Contains Argo CD Applications, Crossplane
Compositions, Helm values, and tenant claims. Also holds the central
`values/app-version.yaml` referenced by every tenant Helm release —
see [Application updates and staging](#application-updates-and-staging).

### GCP managed services

**Google Loadbalancer** — automatically created by GKE when a
Kubernetes Service of type `LoadBalancer` exists (in our case, the
Traefik service). Holds the public IP, terminates TCP on port 443,
and forwards traffic to Traefik. TLS termination happens in Traefik,
not here.

**Google Secret Manager (GSM)** — central store for secrets that
cannot be generated inside the cluster, or that we want to preserve
across cluster rebuilds. Contents:

- `shared/ghcr-pull-secret` — image pull credential for the private
  frontend image. Placed manually once.
- `tenant-<name>/basicauth-htpasswd` — per-tenant BasicAuth password
  in htpasswd format (`<username>:<bcrypt-hash>`). Generated by
  Crossplane at onboarding time and stored here so it survives
  cluster rebuilds.

Database credentials are **not** stored in GSM — see CloudNativePG below.

**GCS (Google Cloud Storage)** — used for two purposes:
- Terraform state (versioned, with state locking)
- CloudNativePG database backups (via the Barman plugin, with
  per-tenant prefixes)

**Cloud DNS** — managed zone `platform-zone` for the domain
`fhuebung.lol`. Maintained automatically by ExternalDNS (A/CNAME
records) and by cert-manager (ACME DNS-01 TXT records).

### Platform components (inside the GKE cluster)

**Argo CD** — the GitOps controller. Watches `platform-gitops` and
reconciles its contents into the cluster continuously. Follows the
App-of-Apps pattern: a root Application bootstraps the others.
Argo CD's web UI is exposed publicly at `argocd.fhuebung.lol`,
protected by Argo CD's built-in authentication (initial admin
password generated at install time, stored in GSM and rotated after
first login; team members authenticate with their own accounts).

**Crossplane** — orchestrates per-tenant resources. Loads three
providers: `provider-gcp` (for GCP API calls), `provider-kubernetes`
(for Kubernetes manifests), and `provider-helm` (for Helm releases).
Defines the schema of a tenant via an XRD (`XTenant`) and implements
provisioning via a Composition. Authenticates against GCP via GKE
Workload Identity.

**External Secrets Operator (ESO)** — syncs secrets from GSM into
the cluster. Setup: a `ClusterSecretStore` defines the connection to
GSM (also via GKE Workload Identity). Per tenant, Crossplane creates
two `ExternalSecret` objects:

- one for the GHCR pull secret (consumed as `imagePullSecret` by the
  frontend pod)
- one for the BasicAuth htpasswd entry (consumed by the Traefik
  Middleware that protects the tenant ingress)

The application itself consumes no ESO-synced secret.

**ExternalDNS** — watches `Ingress` and `Service` objects and creates
the corresponding DNS records in Cloud DNS. Uses TXT-record ownership
(`txtOwnerId: gke-prod`) to avoid overwriting records owned by other
tools.

**cert-manager** — issues TLS certificates. Uses a `ClusterIssuer`
with Let's Encrypt as the ACME provider and the DNS-01 challenge
against Cloud DNS. This allows the wildcard certificate
`*.fhuebung.lol` to be issued once and reused across all tenant
subdomains.

**Traefik** — the ingress controller. Runs as a Kubernetes Deployment
with a `Service` of type `LoadBalancer` (see Google Loadbalancer
above). Terminates TLS with the wildcard certificate and routes HTTP
traffic to the appropriate backends. Three hosts are exposed:
- `*.fhuebung.lol` for tenant apps (with path routing: `/` →
  frontend, `/api` → backend, **gated by a per-tenant BasicAuth
  middleware**)
- `argocd.fhuebung.lol` → Argo CD UI (own built-in auth)
- `grafana.fhuebung.lol` → Grafana UI (own built-in auth)

The BasicAuth middleware for tenant URLs is a Traefik `Middleware`
custom resource provisioned by Crossplane in each tenant namespace.
It references the htpasswd Secret materialised by ESO from GSM.

**CloudNativePG operator** — manages PostgreSQL clusters modelled as
the custom resource `Cluster`. Handles provisioning, replication,
rolling updates for minor versions, and automated major upgrades.
One `Cluster` exists per tenant. The operator generates the database
user and password on cluster creation and stores them as a Kubernetes
`Secret` in the tenant namespace. The backend reads `DATABASE_URL`
from that secret directly — no detour through GSM or ESO is needed
for database credentials.

**kube-prometheus-stack** (bonus task) — the monitoring stack:
Prometheus for metrics collection, Grafana for dashboards,
Alertmanager for alert routing. Prometheus scrapes pods cluster-wide,
including all tenant workloads. Grafana is exposed publicly at
`grafana.fhuebung.lol`, protected by Grafana's built-in authentication
(admin password generated by the Helm chart on install, stored in GSM
and synced into the cluster via ESO; team members log in with their
own Grafana accounts).

**GKE Workload Identity bindings** — a set of Kubernetes
ServiceAccount ↔ GCP IAM Service Account bindings. This allows
in-cluster components (ESO, ExternalDNS, cert-manager, CloudNativePG,
Crossplane provider-gcp) to call GCP APIs without JSON keys living
in the cluster.

### Per-tenant resources (in the tenant namespace)

**NetworkPolicy** — `default-deny` plus an allow-list for required
connections (e.g. to cluster-internal CoreDNS, to the CloudNativePG
cluster in the same namespace, ingress from Traefik). Prevents
workloads from communicating directly between tenants.

**ResourceQuota + LimitRange** — `ResourceQuota` caps the namespace's
total resources (CPU, memory, storage, pod count). `LimitRange` sets
defaults and maximum values for individual containers. Together they
prevent any one tenant from starving the others.

**ServiceAccount** — the tenant identity. Bound to GCP IAM roles
where needed (e.g. when backups need to land under a tenant-specific
bucket prefix).

**ExternalSecrets** — two secrets are synced from GSM per tenant:

- **GHCR pull secret** — image pull credential for the private
  frontend image. Source: `shared/ghcr-pull-secret` in GSM (placed
  manually once, cannot be auto-generated).
- **BasicAuth htpasswd** — the random password protecting the
  tenant's ingress, bcrypt-hashed in htpasswd format. Source:
  `tenant-<name>/basicauth-htpasswd` in GSM. Generated by Crossplane
  at onboarding time. Materialised as a Kubernetes Secret with the
  `users` key (the format Traefik's BasicAuth middleware expects).

The application receives no secrets via ESO. Database credentials
come directly from the secret that CloudNativePG creates next to the
database cluster.

**Traefik Middleware (BasicAuth)** — a `Middleware` custom resource
referencing the BasicAuth Secret above. The tenant Ingress carries
the annotation
`traefik.ingress.kubernetes.io/router.middlewares` pointing at this
Middleware, so all requests to the tenant host are challenged before
reaching the frontend or backend Service.

**Ingress** — one `Ingress` object per tenant, configuring Traefik.
Host is `[tenant].fhuebung.lol`, TLS certificate is the wildcard
`*.fhuebung.lol`. Two routes: `/` to the frontend, `/api` to the
backend. BasicAuth middleware annotation as described above.

**Frontend** — Deployment serving a single-page application via
nginx on port 80. Delivers the static assets (HTML/JS/CSS). The
backend URL is injected at runtime through `/config.js`, delivered
by the Helm chart as a ConfigMap (`apiBaseUrl` always `/api`); the
image serves the file statically and does not write it.

**Backend** — Deployment exposing the REST API on port 3000. Health
endpoint at `GET /healthz`. On startup, connects to the database
using `DATABASE_URL` from the CloudNativePG-managed secret and runs
idempotent schema migrations before accepting requests. The backend
has no authentication state of its own — by the time a request
reaches it, the BasicAuth middleware at the ingress has already
authenticated the caller.

**CloudNativePG Cluster** — this tenant's database instance.
Generates a Kubernetes Service (e.g. `<cluster>-rw` for read-write
access) and a Secret containing the DB credentials. Persists to a
PVC.

**PVC `db-data`** — Persistent Volume Claim, 5 GiB, GKE default
StorageClass `standard-rwo` (SSD persistent disk). Holds this
tenant's PostgreSQL data files.

### Platform storage

**PVC `prometheus-data`** — 20 GiB, also `standard-rwo`. Holds
Prometheus's time-series data. The Helm values configure a 15-day
retention window; older series are discarded automatically.

Grafana and Alertmanager run without their own PVCs. Grafana loads
dashboards from ConfigMaps shipped via Argo CD (see
`applications/monitoring/` in `platform-gitops`); user-created
dashboards that are not persisted to a ConfigMap are lost on pod
restart. Alertmanager state is in-memory and rebuilt on restart.
Both behaviours are intentional for this academic scope.

### Continuous integration and local bootstrap

**GitHub Actions** runs two categories of pipeline across the six
repositories:

- **PR validation** in every repository: commitlint, PR title
  validation, plus repo-specific linters (`tflint`, `yamllint`,
  `kubeconform`, `helm lint`). Implemented as reusable workflows
  in the `.github` repository and imported by the others.
- **Image build and push** in `app-backend` and `app-frontend`: on
  release tags, builds the Docker image and pushes it to GHCR.
  The `app-backend` workflow additionally packages and `helm push`-es
  the chart (sources in `app-backend/chart/`) as an OCI artefact to
  `ghcr.io/ineni-pt-group-b/app-chart`. Authentication to GHCR uses
  the GitHub-internal `GITHUB_TOKEN`; no GCP IAM is involved.

**Terraform is intentionally not run from CI.** The `platform-iac`
repository is applied via the `bootstrap/bootstrap.sh` script
executed locally by a team member. This single manual step is the
documented exception per the assignment; the script is idempotent,
lives in the repo, and creates the state bucket itself as part of
its preflight, resolving the chicken-and-egg problem of needing a
GCS bucket before `terraform init`. After the script completes,
Argo CD takes over and the platform self-manages from
`platform-gitops` — no further manual steps are required for
normal operation.

There is therefore **no GitHub Actions OIDC ↔ GCP trust binding**
and no long-lived GCP service account JSON keys exist anywhere in
the project.

## Data flows

The most important sequences, in the order one would walk through them
during a demo.

### User request to a tenant app

1. Browser resolves `tenant-a.fhuebung.lol` via DNS. Cloud DNS
   returns the public IP of the Google Loadbalancer.
2. Browser establishes a TLS connection to that IP. The Loadbalancer
   forwards TCP/443 to the Traefik service (TLS pass-through).
3. Traefik terminates TLS with the wildcard certificate
   `*.fhuebung.lol` and checks the SNI hostname to determine which
   ingress to use.
4. **BasicAuth challenge.** The Ingress's middleware annotation
   triggers the per-tenant `BasicAuth` middleware. On the first
   request, the browser shows the native HTTP auth dialog. The
   middleware verifies the supplied credentials against the
   htpasswd Secret in the tenant namespace. Without valid
   credentials the request is rejected with `401` and never reaches
   the backend or frontend Service.
5. Request to `/` → Traefik routes to the frontend Service in the
   namespace `tenant-a`. nginx serves the SPA assets.
6. Browser executes the SPA, which calls the backend API at
   `/api/...`. The request again goes through DNS → LB → Traefik
   → BasicAuth middleware (browser sends cached credentials
   automatically).
7. Traefik routes `/api/*` to the backend Service in the same
   namespace.
8. Backend performs CRUD operations on the application's database
   tables. The backend has no authentication logic of its own —
   any request reaching it has already passed the middleware.
9. Backend opens a database connection to the CloudNativePG cluster
   using `DATABASE_URL` from the CloudNativePG-managed secret.
10. PostgreSQL reads/writes its data files on the PVC `db-data`.
11. Response travels back to the browser along the reverse path.

### Tenant onboarding

1. A team member opens a pull request against `platform-gitops` that
   adds a new file `tenants/tenant-c.yaml` — the tenant claim. The
   content is the tenant name and the host, plus optional
   configuration (e.g. `imageTagOverride` for a staging tenant).
2. CI validates YAML syntax (yamllint, kubeconform). The PR is
   reviewed and merged.
3. Argo CD watches `platform-gitops`, sees the new file, and the
   claim appears as a custom resource in the cluster on the next
   sync.
4. Crossplane sees the new claim and reconciles the corresponding
   Composition. In one step it creates:
   - the tenant namespace
   - the NetworkPolicy, ResourceQuota, LimitRange, ServiceAccount
   - the CloudNativePG `Cluster` (the CloudNativePG operator
     takes over its provisioning, including the PVC and the
     credentials secret)
   - the BasicAuth htpasswd entry: a random password, bcrypt-hashed
     in htpasswd format, stored in GSM under
     `tenant-<name>/basicauth-htpasswd`
   - two `ExternalSecret` objects: one for the GHCR pull secret,
     one for the BasicAuth htpasswd
   - a Traefik `Middleware` referencing the htpasswd Secret
   - a `Release` resource (provider-helm) that pulls the app Helm
     chart from GHCR as an OCI artefact and unfolds it (frontend
     Deployment, backend Deployment, Services, Ingress with the
     BasicAuth middleware annotation). Image tags come from
     `values/app-version.yaml`, unless the claim sets
     `imageTagOverride`.
5. ExternalDNS sees the new Ingress and creates an A record
   `tenant-c.fhuebung.lol` in Cloud DNS. cert-manager already has
   the wildcard certificate.
6. The tenant is live: `https://tenant-c.fhuebung.lol` is reachable
   with the BasicAuth prompt as the first interaction.

#### Day-2 tenant provisioning diagram

`day2-tenant-provisioning.drawio` (with PNG export `.drawio.png`) shows
the same sequence as a swim-lane phase flow in the Day-1 style. Five
lanes left-to-right — Developer / platform-gitops, Argo CD, Crossplane,
the tenant namespace, and the external GCP Secret Manager / Cloud DNS /
User column — make visible which actor or boundary owns each step. Time
flows top-to-bottom through Phase 1 (claim merged), Phase 2 (Argo CD
reconciles the XTenant), Phase 3 (Crossplane matches the Composition),
Phase 4 (per-tenant resources — namespace and boundaries, Postgres,
BasicAuth via GSM + ESO, app Helm release — assembled in the tenant
namespace), and Phase 5 (the URL becomes reachable: ExternalDNS
publishes the record, Traefik serves the wildcard TLS, BasicAuth gates
access).

### TLS certificate issuance (DNS-01 challenge)

1. cert-manager sees a `Certificate` request (or the ongoing renewal
   of the wildcard certificate).
2. cert-manager initiates an ACME order with Let's Encrypt.
3. Let's Encrypt returns a challenge token.
4. cert-manager writes a temporary TXT record
   `_acme-challenge.fhuebung.lol` to Cloud DNS.
5. Let's Encrypt queries the TXT record via public DNS. A successful
   lookup confirms domain control.
6. Let's Encrypt issues the certificate. cert-manager stores it as
   a Kubernetes `Secret` in the relevant namespace.

### Secret synchronisation

1. An `ExternalSecret` resource in the tenant namespace declares the
   target secret in GSM (e.g. `tenant-a/basicauth-htpasswd` for the
   ingress password, or `shared/ghcr-pull-secret` for the image pull
   credential).
2. ESO authenticates against GSM via GKE Workload Identity (no
   static key inside the cluster).
3. ESO retrieves the current secret value and writes it to a
   Kubernetes Secret in the tenant namespace.
4. The BasicAuth Secret is consumed by the Traefik `Middleware` for
   that tenant; the GHCR pull secret is referenced as
   `imagePullSecret` by the frontend pod. The application itself
   does not consume any ESO-synced secret.
5. ESO refreshes periodically (every few minutes), so rotation in
   GSM propagates into the cluster automatically.

Database credentials follow a different path: the CloudNativePG
operator creates the user and password inside the cluster on
provisioning and stores them as a Kubernetes Secret next to the
database. The backend reads the secret directly via env var
projection — no GSM involvement.

### Image pulls

Three variants, all shown as dashed grey arrows in the diagram:

- **app-backend** (public): the backend pod pulls directly from GHCR,
  no pull secret required.
- **app-frontend** (private): the frontend pod pulls with a
  GHCR pull secret (synced by ESO).
- **app-chart** (public, OCI): Crossplane's provider-helm pulls the
  Helm chart as an OCI artefact from GHCR and renders it into the
  cluster.

### Database backup

1. The CloudNativePG `Cluster` manifest contains a backup
   configuration pointing to the GCS backup bucket with a
   tenant-specific prefix.
2. The Barman plugin (built into CloudNativePG) periodically produces
   WAL archives and base backups.
3. Backups land in the GCS bucket under
   `gs://platform-pg-backups/<tenant>/...`.
4. Authentication against GCS via GKE Workload Identity, bound to
   the tenant-specific ServiceAccount.

### Application updates and staging

The image tags and chart version every tenant runs are controlled from
a single file in `platform-gitops`: `values/app-version.yaml`. The
Crossplane Composition reads this file when rendering each tenant's
Helm release. Schema:

```yaml
chart:
  version: "0.1.0"
images:
  backend: "v0.1.0"
  frontend: "v0.1.0"
```

**Rolling out a new version to all tenants** is a single pull request:
bump the relevant value(s) in `values/app-version.yaml`, merge. Argo CD
reconciles, Crossplane re-renders every tenant's `Release`, and
provider-helm performs a rolling upgrade per tenant. No per-tenant
edits, no manual coordination.

**Testing a new version on a staging tenant first** uses an optional
`imageTagOverride` block in a tenant claim. Both `backend` and
`frontend` can be overridden independently. The dedicated `staging`
tenant uses this to receive new versions before the central rollout.
A typical update sequence:

1. PR 1: set `imageTagOverride.backend: v0.2.0` (and optionally
   `.frontend`) in `tenants/staging.yaml`. Merge, observe
   `staging.fhuebung.lol` on the new version.
2. Verify functionality on staging.
3. PR 2: change the central tag(s) in `values/app-version.yaml` to
   `v0.2.0`. Merge. All other tenants roll forward.
4. PR 3: optionally remove the override from `tenants/staging.yaml`
   so staging follows the central values again.

### Monitoring

1. Prometheus scrapes pods cluster-wide. Discovery happens via the
   Kubernetes API plus pod annotations.
2. Tenant workloads (backend, frontend) expose metrics on a
   `/metrics` endpoint (standard for modern frameworks).
3. Prometheus persists time-series data to the PVC `prometheus-data`
   with a 15-day retention window.
4. Grafana reads from Prometheus and renders dashboards. The bundled
   dashboards (Kubernetes cluster, pods, nodes) plus a small set of
   custom dashboards shipped as ConfigMaps via Argo CD are sufficient
   for this academic project.
5. Application owners and the platform admin access the UI at
   `https://grafana.fhuebung.lol`.

## Assumptions and design decisions

The main decisions reflected in the diagram:

- **Soft multi-tenancy.** A shared cluster, isolation via namespace
  plus NetworkPolicy plus ResourceQuota. No dedicated node pools
  per tenant.
- **Crossplane orchestrates the entire tenant deployment**, including
  app workloads via `provider-helm`. Argo CD only syncs the tenant
  claim from `platform-gitops/tenants/`; from there, Crossplane
  takes over namespace, DB, secrets, BasicAuth middleware, network
  policies, quotas, and the Helm release for the app. This split
  satisfies the requirement "Crossplane to handle all other
  deployment steps" from the assignment and at the same time
  captures the Helm chart bonus.
- **Wildcard TLS** (`*.fhuebung.lol`) instead of per-tenant
  certificates. Saves Let's Encrypt rate limit and provisioning time.
- **Path-based routing** (`/api` for the backend) instead of
  host-based, because the wildcard certificate covers only one
  level of subdomain.
- **Ingress-level BasicAuth instead of application-level
  authentication.** Tenant URLs are protected by a Traefik
  `BasicAuth` middleware provisioned per tenant by Crossplane. The
  application itself is authentication-free, which keeps the
  deployment contract small (`DATABASE_URL`, `PORT`) and moves auth
  to a single, declarative place at the ingress layer.
- **Helm chart lives with the backend code** (`app-backend/chart/`).
  Backend and chart are versioned together; the backend release
  workflow handles both image and chart push.
- **Database credentials managed by CloudNativePG**, not by ESO/GSM.
  The operator creates a Kubernetes Secret directly; the backend
  reads from it via env var projection.
- **Terraform runs locally via `bootstrap/bootstrap.sh`, not from CI.**
  This avoids the chicken-and-egg of needing a state bucket before
  `terraform init`, removes the need for any GitHub Actions ↔ GCP
  trust binding, and keeps the one-time bootstrap simple and
  reproducible. The script is idempotent; everything after the
  bootstrap is reconciled by Argo CD without further manual steps.

Items deliberately left out (continuous delivery via Argo Rollouts,
frontend deployment to a GCS bucket, Kyverno policies, hard
multi-tenancy) are listed and motivated in
[`docs/claude/project-overview.md`](../claude/project-overview.md#out-of-scope).

For further rationale on what is in scope, see
[`docs/claude/architecture-decisions.md`](../claude/architecture-decisions.md).

## Capacity and cost

The GKE cluster is a zonal Standard cluster in `europe-west1` with a
single default node pool of 3–6 `e2-standard-4` machines, governed by
the GKE Cluster Autoscaler. The lower bound covers the platform
baseline (Argo CD, Crossplane, ESO, ExternalDNS, cert-manager,
Traefik, CloudNativePG operator, kube-prometheus-stack) plus a
handful of small tenants; the upper bound provides headroom for
tenant growth during the project.

Detailed capacity planning, cost estimates, and a comparison against
actual GCP billing is tracked in
[`docs/cost-planning/`](../cost-planning/).

## Maintenance

- Source files (in this directory):
  - `logical_architecture.drawio` — component view
  - `day1-bootstrap.drawio` — bootstrap flow
  - `day2-tenant-provisioning.drawio` — tenant onboarding flow
- PNG exports: each `<name>.drawio` has a sibling `<name>.drawio.png`
  generated via drawio `File → Export as → PNG` with "Include a copy
  of my diagram" enabled. Manual refresh required after every diagram
  change.
- When the architecture changes: update `architecture-decisions.md`
  first, then the affected diagram(s), then this README.
- Last reviewed: 2026-05-30.
