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
- Worker nodes: 3 × `e2-standard-4` (12 vCPU / 48 GB total). See cost planning
  in `docs/cost-planning/`.

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
platform layer; per-tenant Applications generated via ApplicationSets.

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

## Database

**CloudNativePG** in-cluster.

Provides PostgreSQL clusters as Kubernetes-native resources. Each tenant gets
their own CloudNativePG `Cluster` provisioned via Crossplane. Chosen over
Cloud SQL for cost (in-cluster, no managed-service premium) and to demonstrate
Crossplane composing in-cluster resources.

Automated upgrades are handled by CloudNativePG's rolling update feature.

## Multi-tenancy and SaaS provisioning

**Crossplane** with custom XRDs and Compositions, plus **Argo CD ApplicationSets**
for tenant application generation.

The flow:

1. A new tenant is onboarded by adding a **Tenant claim** (namespaced) under
   `platform-gitops/tenants/`.
2. Crossplane reconciles the claim against the matching Composition, which
   provisions:
   - a dedicated namespace
   - a CloudNativePG database `Cluster`
   - network policies isolating the namespace
   - a Kubernetes `ResourceQuota` and `LimitRange`
   - any glue resources (image-pull secrets, ServiceAccounts) needed by the app
3. An **ApplicationSet** in `applicationsets/` generates the Argo CD
   `Application` for the tenant's app deployment, parameterized by the tenant
   claim.

Soft multi-tenancy is sufficient per the assignment; hard multi-tenancy and
virtual clusters are out of scope.

## Container registry

**GitHub Container Registry (GHCR)**.

- Backend image: public (`ghcr.io/ineni-pt-group-b/app-backend`)
- Frontend image: private (`ghcr.io/ineni-pt-group-b/app-frontend`)

The frontend image pull requires a registry credential, synced into tenant
namespaces via ESO.

## The application

A **custom minimal 3-tier application**, generated specifically for this project.
Lightweight stack to keep per-tenant resource footprint small.

**App contract** (the deployment surface that Crossplane targets):

- Backend
  - Image: `ghcr.io/ineni-pt-group-b/app-backend:<tag>`
  - Port: `3000`
  - Health endpoint: `GET /healthz` (returns 200 when ready)
  - Required env vars: `DATABASE_URL`, `PORT`
  - Schema migration: runs automatically on backend start (idempotent)
- Frontend
  - Image: `ghcr.io/ineni-pt-group-b/app-frontend:<tag>`
  - Port: `80` (nginx serving static files)
  - Backend URL injection: runtime config via `/config.js` (preferred) or
    `VITE_API_URL` at build time

The exact stack (Node/Fastify, Vue/React, etc.) will be decided at the time
of generation. Stack choice does not affect the platform — only the contract
above is contractual.

## Monitoring (bonus task)

**Prometheus + Grafana**, self-hosted in-cluster (`kube-prometheus-stack` Helm
chart).

Standard GKE control-plane metrics remain available passively. The in-cluster
stack provides application-level metrics, dashboards, and alerts for both
platform administrators and tenant application owners.

## Out of scope

See [`project-overview.md`](./project-overview.md#out-of-scope).
