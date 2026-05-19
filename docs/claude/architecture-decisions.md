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
bucket (versioned, with state locking).

Authentication from CI: **GitHub Actions OIDC → GCP Workload Identity
Federation**. No long-lived service account JSON keys.

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

## Database

**CloudNativePG** in-cluster.

Provides PostgreSQL clusters as Kubernetes-native resources. Each tenant gets
their own CloudNativePG `Cluster` provisioned via Crossplane. Chosen over
Cloud SQL for cost (in-cluster, no managed-service premium) and to demonstrate
Crossplane composing in-cluster resources.

Automated upgrades are handled by CloudNativePG's rolling update feature.

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
   - `ExternalSecret` resources for the GHCR pull secret and the
     application login password
   - a Helm `Release` (via `provider-helm`) that pulls the app chart from
     GHCR as an OCI artefact and renders frontend, backend, services, and
     ingress into the namespace
3. The tenant is live once Crossplane reports the Composition as ready
   and ExternalDNS has published the DNS record.

This split satisfies the assignment requirement "Crossplane to handle all
other deployment steps" and at the same time captures the Helm-chart bonus.

Soft multi-tenancy is sufficient per the assignment; hard multi-tenancy and
virtual clusters are out of scope.

## Application updates and staging

The application image tag every tenant runs is held in a single file in
`platform-gitops`: `values/app-version.yaml`. The Crossplane Composition
reads this value when rendering each tenant's Helm release.

- **Rolling out a new version to all tenants**: change the tag in
  `values/app-version.yaml`, open a PR, merge. Crossplane re-renders every
  tenant's Release; provider-helm performs a rolling upgrade per tenant.
- **Testing on a staging tenant first**: the optional `imageTagOverride`
  field on a tenant claim beats the central value for that one tenant.
  The dedicated `staging` tenant uses this to receive new versions before
  the central rollout.

## Container registry

**GitHub Container Registry (GHCR)**.

- Backend image: public (`ghcr.io/ineni-pt-group-b/app-backend`)
- Frontend image: private (`ghcr.io/ineni-pt-group-b/app-frontend`)
- App Helm chart: public OCI artefact
  (`ghcr.io/ineni-pt-group-b/app-chart`), pulled by Crossplane's
  `provider-helm` when reconciling a tenant claim

The frontend image pull requires a registry credential, synced into tenant
namespaces via ESO.

## The application

A **custom minimal 3-tier application**, generated specifically for this project.
Lightweight stack to keep per-tenant resource footprint small.

> **Status:** the concrete decisions around the application — final stack
> (backend framework, frontend framework, database driver), domain model,
> authentication mechanism, repository status (own repo vs. lecturer fork),
> and the location of the Helm chart — are tracked in a separate issue.
> When that issue closes, the relevant sections in this document and in
> `repository-layout.md` are updated to reflect the final state.

The following pieces are already settled and act as the **deployment contract**
that Crossplane targets:

- Backend
  - Image: `ghcr.io/ineni-pt-group-b/app-backend:<tag>`
  - Port: `3000`
  - Health endpoint: `GET /healthz` (returns 200 when ready)
  - Required env vars: `DATABASE_URL`, `PORT`, `APP_PASSWORD`
  - Schema migration: runs automatically on backend start (idempotent)
- Frontend
  - Image: `ghcr.io/ineni-pt-group-b/app-frontend:<tag>`
  - Port: `80` (nginx serving static files)
  - Backend URL injection: runtime config via `/config.js` (preferred) or
    `VITE_API_URL` at build time
- Auth surface
  - Single application-level password per tenant
  - Plaintext stored in GSM, synced to the namespace as `APP_PASSWORD` via ESO
  - Backend hashes the value on startup (bcrypt, in memory) and verifies
    login requests against the in-memory hash
  - Plaintext never leaves GSM and the backend's process memory

## Monitoring (bonus task)

**Prometheus + Grafana**, self-hosted in-cluster (`kube-prometheus-stack` Helm
chart).

Standard GKE control-plane metrics remain available passively. The in-cluster
stack provides application-level metrics, dashboards, and alerts for both
platform administrators and tenant application owners.

## Out of scope

See [`project-overview.md`](./project-overview.md#out-of-scope).
