# Architecture Decisions

> Captures the technology choices for this project, with brief rationale.
> This is not a full ADR log — it is a quick reference for AI-assisted work
> and onboarding.

## Cloud platform

**Google Cloud Platform**, region `europe-west1` (Belgium).

Chosen because the lecturer recommends managed Kubernetes on GCP and the course
material is GCP-aligned. `europe-west1` is geographically close, has full GKE
feature parity, and reasonable pricing.

- GCP Project ID: `dotted-axle-495612-f4`
- GCP Project display name: `platform-engineering-group-b`
- Cloud DNS Managed Zone: `platform-zone`

## Kubernetes

**GKE Standard**, zonal cluster.

- Standard (not Autopilot) for full control over Crossplane requirements,
  network policies, and multi-tenancy primitives.
- Zonal (not regional) to keep cost low — one control plane, nodes in a single
  zone. The assignment explicitly permits this.
- Worker nodes: 3–6 × `e2-standard-4` with the GKE Cluster Autoscaler enabled
  (12–24 vCPU, 48–96 GB at the bounds). The lower bound covers the platform
  baseline plus a few small tenants; the upper bound provides headroom for
  tenant growth during the project. See cost planning in `docs/cost-planning/`.

## Infrastructure as Code

**Terraform** (HashiCorp).

Chosen over Pulumi for ecosystem maturity around GCP and the broader pool of
existing modules (e.g. `terraform-google-modules`). State stored in a GCS
bucket `gs://<project>-tfstate`, versioned, with state locking.

Terraform runs **locally via the `bootstrap/bootstrap.sh` script** executed
by a team member, not from CI. The script is idempotent and committed to
`platform-iac`; it handles, in order, preflight checks, GCP API enablement,
state-bucket creation (resolving the chicken-and-egg of needing a GCS bucket
before `terraform init`), `terraform apply`, kubeconfig retrieval, and the
Argo CD bootstrap (Helm install + root App-of-Apps). After the script
completes, Argo CD takes over and the platform self-manages from
`platform-gitops`.

This is the one documented "manual step" exception per the assignment:
the bootstrap action is fully reproducible from code, and a re-run of the
script after a teardown rebuilds the platform end-to-end without manual
clicks beyond invoking the script itself.

No CI-based Terraform pipeline exists; therefore no GitHub Actions OIDC ↔
GCP trust binding for Terraform. GitHub Actions OIDC is **not** in the
Terraform path. GitHub Actions still runs PR validation, image build and
push, and Helm chart push — but the GHCR pushes authenticate using GitHub's
own built-in `GITHUB_TOKEN`, never GCP IAM. There are no long-lived GCP
service account JSON keys anywhere in the project.

## GitOps

**Argo CD**.

Chosen over Flux for the better UI (helpful for the demo and for debugging
sync state) and broader community adoption. App-of-Apps pattern for the
platform layer. Tenant resources are not per-tenant Argo CD Applications:
Argo CD only syncs the tenant claims from `tenants/`, and Crossplane takes
over from there (see § Multi-tenancy and SaaS provisioning).

## Secrets management

**Google Secret Manager** (cloud-managed) + **External Secrets Operator (ESO)**
in-cluster.

ESO syncs secrets from Google Secret Manager into Kubernetes Secret objects.
Authentication via Workload Identity (no static credentials inside the cluster).
Self-hosted Vault was considered but rejected as unnecessary complexity for
this scope.

## DNS and TLS

- **Domain:** `fhuebung.lol`, registered at Porkbun.
- **DNS:** delegated to **Google Cloud DNS** (NS records at Porkbun point to
  Cloud DNS nameservers, DNSSEC disabled at the registrar).
- **DNS automation:** **ExternalDNS** with the Cloud DNS provider, authenticated
  via Workload Identity.
- **TLS:** **cert-manager** issuing certificates via **Let's Encrypt ACME**
  using the **DNS-01 challenge** against Cloud DNS.

### Zone lifecycle — persistent, not recreated

The Cloud DNS managed zone `platform-zone` is treated as **persistent
infrastructure**, in the same class as the GCS Terraform-state bucket and
Google Secret Manager: it is created and delegated once and is **not** destroyed
on teardown. `bootstrap/bootstrap.sh` reconciles it create-if-absent — an
existing zone resource is reused as-is (in-zone records and IAM bindings
continue to be managed by their owners), and only a fully empty project triggers
a one-time creation.

This **supersedes the earlier "recreate, not import" decision** (task list
S1-09). Recreating the zone on every bootstrap would reassign the Google
authoritative nameservers and force a matching NS update at the registrar
(Porkbun) during the run. That either breaks the "no manual click after
`bootstrap.sh` kickoff" constraint or requires a registrar API credential on the
operator's machine. Keeping the zone persistent removes the registrar step from
the bootstrap path entirely: after the one-time initial delegation, no Porkbun
access is ever needed again — also the best outcome for bus factor, since only
one team member holds registrar access.

`terraform import` stays rejected: it leaves a manual, non-reproducible step in
the provisioning path. The persistent zone is a pre-existing resource that
`bootstrap.sh` references rather than imports, mirroring how it already treats
the state bucket.

Decided in issue #51 (Option B).

## Ingress

**Traefik** as the in-cluster ingress controller.

Chosen over ingress-nginx because the latter reaches end-of-life in March 2026.
Traefik is actively maintained, has good native integration with Kubernetes
Ingress resources, and renders the wildcard certificate (`*.fhuebung.lol`)
issued by cert-manager.

The Traefik deployment exposes three host patterns:

- `*.fhuebung.lol` — tenant applications, with path-based routing: `/` → tenant
  frontend, `/api` → tenant backend. Path-based (not host-based) routing is
  required because the wildcard certificate covers only one subdomain level.
- `argocd.fhuebung.lol` — Argo CD UI
- `grafana.fhuebung.lol` — Grafana UI

The Kubernetes `Service` backing Traefik is of type `LoadBalancer`, which
makes GKE provision a Google Cloud L4 load balancer with a public IP.
TLS is terminated in Traefik, not in the load balancer.

### Per-tenant BasicAuth

Tenant URLs are protected by a **Traefik BasicAuth middleware** at the ingress
layer rather than by application-level authentication. This keeps the
application code authentication-free and makes auth a platform concern.

Per tenant, the Crossplane Composition provisions:

- A random password generated at onboarding time, bcrypt-hashed into the
  htpasswd format (`<username>:<bcrypt-hash>`). Username is fixed to `admin`;
  password is per-tenant random.
- The htpasswd string stored in Google Secret Manager
  (`tenant-<name>-basicauth-htpasswd`).
- A Kubernetes `Secret` in the tenant namespace, materialised from GSM by ESO.
- A Traefik `Middleware` custom resource referencing that Secret.
- An annotation on the tenant Ingress
  (`traefik.ingress.kubernetes.io/router.middlewares`) wiring the
  Middleware into the request path.

The application itself is unaware of this layer.

## Database

**CloudNativePG** in-cluster.

Provides PostgreSQL clusters as Kubernetes-native resources. Each tenant gets
their own CloudNativePG `Cluster` provisioned via Crossplane. Chosen over
Cloud SQL for cost (in-cluster, no managed-service premium) and to demonstrate
Crossplane composing in-cluster resources.

Automated upgrades are handled by CloudNativePG's rolling update feature.

### Backups — direct Workload Identity Federation, no per-tenant GSA

Each tenant `Cluster` takes daily Barman base backups plus continuous WAL
archiving to `gs://<project>-pg-backups/<tenant>/` (bucket provisioned by
`platform-iac` `terraform/backup`, 30-day lifecycle delete as a cost
backstop behind Barman's own 14-day `retentionPolicy`).

The write credential uses **Workload Identity Federation for GKE direct
resource access**: the Composition mints a per-tenant `BucketIAMMember`
(via the `provider-gcp-storage` sub-provider) that grants the CNPG
instance KSA's federated principal
(`principal://…/subject/ns/tenant-<name>/sa/<name>-cluster`) a
prefix-scoped `roles/storage.objectAdmin` on the shared bucket. The CNPG
side only sets `googleCredentials.gkeEnvironment: true` — ambient ADC, no
`iam.gke.io/gcp-service-account` annotation, no per-tenant GSA, no SA-JSON
key. The IAM condition pairs `resource.name.startsWith(<tenant prefix>)`
with the `objectListPrefix` API attribute so object listing works while
staying inside the tenant's own prefix.

In-tree Barman support is deprecated upstream (removal slated for CNPG
1.30); we pin operator 1.29.1 and treat the Barman Cloud CNPG-I plugin
migration as out of scope, mirroring the Crossplane 2.x pinning decision.

Decided in INENI-PT-GROUP-B/platform-gitops#67.

## Multi-tenancy and SaaS provisioning

**Crossplane** with custom XRDs and Compositions. Crossplane's `provider-helm`
deploys the tenant application as a Helm release directly — no Argo CD
ApplicationSet is involved for tenant app deployment.

The flow:

1. A new tenant is onboarded by adding a **Tenant claim** (namespaced) under
   `platform-gitops/tenants/`. Argo CD syncs the claim into the cluster as
   a custom resource.
2. Crossplane reconciles the claim against the matching Composition, which
   provisions in one step:
   - a dedicated namespace
   - a CloudNativePG database `Cluster` (operator creates the credentials
     Secret next to it)
   - network policies isolating the namespace
   - a Kubernetes `ResourceQuota` and `LimitRange`
   - an `ExternalSecret` for the GHCR pull secret
   - the BasicAuth setup (GSM entry, `ExternalSecret`, Traefik `Middleware`)
     described in § Ingress
   - a Helm `Release` (via `provider-helm`) that pulls the app chart from
     GHCR as an OCI artefact and renders frontend, backend, services, and
     ingress into the namespace
3. The tenant is live once Crossplane reports the Composition as ready
   and ExternalDNS has published the DNS record.

This split satisfies the assignment requirement "Crossplane to handle all
other deployment steps" and at the same time captures the Helm-chart bonus.

Soft multi-tenancy is sufficient per the assignment; hard multi-tenancy and
virtual clusters are out of scope.

### Tiering

The XTenant XRD's `spec.tier` exposes two resource-governance envelopes
(`small` and `medium`) over `ResourceQuota` and `LimitRange` to
demonstrate that the platform differentiates per-tenant resource
governance. Tier drives the `ResourceQuota` quantities and the
`LimitRange.max` per container; `defaultRequest` / `default` / `min` stay
constant across tiers. The platform does not prescribe when each tier
applies — that policy belongs to the tenant-operator and is documented
separately in the tenant onboarding guide
(`platform-gitops/docs/tenant-onboarding.md`, planned by S4-06).
Concrete quota and limit numbers live in
`crossplane/compositions/xtenant-default.yaml`.

### Crossplane version — pinned to the 1.x line

Crossplane is pinned to the **1.x maintenance line** (currently chart
`crossplane` 1.20.8, the latest 1.x stable). Crossplane 2.x is intentionally
**not** adopted: v2 removed native patch-and-transform (P&T) Compositions, and
the planned tenant Compositions — the CloudNativePG `Cluster`, the BasicAuth
setup, and the app Helm `Release` — are built on P&T. Staying on 1.x preserves
that Composition design.

Adopting Crossplane 2.x, which would mean migrating the Compositions to its
function-based pipeline model, is a **separate, larger decision** and is out of
scope for now. Upgrades off the 1.x line go through an explicit PR.

Decided in INENI-PT-GROUP-B/platform-gitops#22.

### Crossplane providers — family layout for GCP

The GCP provider is installed via Upbound's **family layout**, not the
deprecated monolithic `provider-gcp`:

- `provider-family-gcp` — metadata-only package; ships the shared
  `gcp.upbound.io/v1beta1 ProviderConfig` CRD; no controller pod.
- `provider-gcp-secretmanager` — sub-provider; ships the controller that
  manages `secretmanager.gcp.upbound.io` managed resources.
- `provider-gcp-storage` — sub-provider; used solely for the per-tenant
  `BucketIAMMember` on the pg-backups bucket (see § Database § Backups).

Only these two sub-providers are installed, covering the two GCP services
the Compositions touch (the per-tenant BasicAuth Composition writes the
htpasswd as a SecretVersion into Google Secret Manager and ESO syncs it
back; the backup Composition edits the pg-backups bucket IAM policy).
DNS, TLS, Postgres and monitoring use their own operators and do not need
a Crossplane provider. All family members are version-pinned in lockstep
(currently v2.5.4).

#### DeploymentRuntimeConfig naming — Option A

The pre-existing `DeploymentRuntimeConfig` and the IaC variable
`crossplane_provider_gcp_ksa_name` are named `provider-gcp`. Under the family
layout the DRC's `serviceAccountTemplate` is per-Provider-owned, not
family-shared (see
[crossplane/crossplane#4552](https://github.com/crossplane/crossplane/issues/4552)),
so the SA is actually owned by the sub-provider `provider-gcp-secretmanager`.

We keep the generic name regardless: the `runtimeConfigRef.name` string is
arbitrary and renaming would mean a coordinated cross-repo IaC PR (KSA
variable, GSA `account_id`, WI binding) without functional benefit.

The second sub-provider anticipated here arrived with the backup work
(`provider-gcp-storage`, platform-gitops#67). As predicted, it has its own
DRC (`provider-gcp-storage`, since the `serviceAccountTemplate` is
per-Provider-owned) and its own WI binding — but it deliberately shares
the existing `crossplane-provider-gcp` GSA instead of getting its own:
its only permission need (the bucket-scoped `pgBackupsBucketIamAdmin`
custom role) was already bound to that GSA, so a dedicated GSA would have
added IaC surface without narrowing any privilege. The generic
`provider-gcp` name was kept; the predicted "free rename" was skipped to
keep the diff minimal.

The trade-off is documented inline at
`crossplane/providers/deployment-runtime-config-gcp.yaml` in `platform-gitops`.

Decided in INENI-PT-GROUP-B/platform-gitops#21.

## Application updates and staging

The image tags and chart version every tenant runs are held in a single file
in `platform-gitops`: `values/app-version.yaml`. The Crossplane Composition
reads this file when rendering each tenant's Helm release.

Schema:

```yaml
chart:
  version: "0.1.0"        # Helm chart version (matches backend release tag)
images:
  backend: "v0.1.0"
  frontend: "v0.1.0"      # may differ from backend tag
```

- **Rolling out a new version to all tenants**: change the relevant value(s)
  in `values/app-version.yaml`, open a PR, merge. Crossplane re-renders every
  tenant's Release; provider-helm performs a rolling upgrade per tenant.
- **Testing on a staging tenant first**: an optional `imageTagOverride` block
  on a tenant claim beats the central values for that one tenant. The
  dedicated `staging` tenant uses this to receive new versions before the
  central rollout. Both `backend` and `frontend` can be overridden
  independently.

## Container registry

**GitHub Container Registry (GHCR)**.

- Backend image: public (`ghcr.io/ineni-pt-group-b/app-backend`)
- Frontend image: private (`ghcr.io/ineni-pt-group-b/app-frontend`)
- App Helm chart: public OCI artefact
  (`ghcr.io/ineni-pt-group-b/app-chart`), pulled by Crossplane's
  `provider-helm` when reconciling a tenant claim. The chart sources live
  in `app-backend/chart/`; the `app-backend` release workflow handles
  both the image build and the `helm push` step.

The frontend image pull requires a registry credential, synced into tenant
namespaces via ESO.

## The application

A **custom 3-tier property-management application**, lightweight by design to
keep per-tenant resource footprint small. The app is a vehicle for the
Application Management grading pillar — it is real CRUD on a real database,
but no extra points are earned for application complexity beyond the
assignment requirements.

**Domain.** Each platform tenant represents one landlord. The application
manages that landlord's rental properties: list, create, edit, delete.
Single table `properties` (label, address, size, rent, notes, timestamps).
No multi-user support per tenant, no file uploads, no payment flow.

To avoid term collisions, the renter of an apartment is called **lessee** or
**occupant** in code and UI — never "tenant", which is reserved for the
platform-level SaaS tenant (landlord).

**Stack.**

- Backend: Node.js LTS, TypeScript, Fastify, Drizzle ORM + drizzle-kit for
  migrations (run on startup; idempotent). Multi-stage Alpine Dockerfile.
- Frontend: Vite + React 18 + TypeScript, React Router, TanStack Query,
  Tailwind CSS. Multi-stage Alpine Dockerfile with nginx serving the
  static bundle. Runtime backend URL injection via `/config.js`, delivered
  by the Helm chart as a ConfigMap exposing `window.APP_CONFIG`
  (`apiBaseUrl`, always `/api` under path-based routing). The image serves
  the file statically and must not also write it.

**Authentication.** Handled at the ingress level via Traefik BasicAuth
(see § Ingress § Per-tenant BasicAuth). The application has no auth
endpoints and no auth state.

**Deployment contract** (the surface that Crossplane targets):

- Backend
  - Image: `ghcr.io/ineni-pt-group-b/app-backend:<tag>`
  - Port: `3000`
  - Health endpoint: `GET /healthz` (200 once DB and migrations are ready)
  - Required env vars: `DATABASE_URL`, `PORT`
  - Schema migration: runs automatically on backend start (idempotent)
- Frontend
  - Image: `ghcr.io/ineni-pt-group-b/app-frontend:<tag>`
  - Port: `80` (nginx serving static files)
  - Backend URL injection: `/config.js` delivered by the Helm chart as a
    ConfigMap exposing `window.APP_CONFIG` (`apiBaseUrl`, always `/api`)

**API surface** (all routes pass through the BasicAuth middleware at the
ingress before reaching the backend):

- `GET /healthz`
- `GET /api/properties`
- `POST /api/properties`
- `GET /api/properties/:id`
- `PATCH /api/properties/:id`
- `DELETE /api/properties/:id`

## Monitoring (bonus task)

**Prometheus + Grafana**, self-hosted in-cluster (`kube-prometheus-stack` Helm
chart).

Standard GKE control-plane metrics remain available passively. The in-cluster
stack provides application-level metrics, dashboards, and alerts for both
platform administrators and tenant application owners.

## Out of scope

See [`project-overview.md`](./project-overview.md#out-of-scope).
