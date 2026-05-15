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

## Bootstrap orchestration

Day 1 is orchestrated by a single shell script,
`platform/scripts/bootstrap.sh`, executable from any developer machine with
GCP credentials and a clean clone of `platform-iac` plus `platform-gitops`.
The script runs in order:

1. `terraform -chdir=platform-iac apply` — provisions cluster, IAM,
   networking, and installs Argo CD via Helm.
2. `gcloud container clusters get-credentials …` — local kubeconfig.
3. Waits for `deployment/argocd-server` to become `Available`.
4. Applies the root Argo CD `Application` (one-shot `kubectl apply`). From
   this point Argo CD self-reconciles and provisions every platform
   component and tenant from `platform-gitops`.

Idempotent on re-run. Chosen over a CI-pipeline-driven deployment because
(a) bootstrap is a one-time-per-environment action where a shell script is
more auditable than a workflow file, (b) the script is runnable from a
developer machine without giving CI access to long-lived cluster state, and
(c) it keeps Day 1 entirely outside the application-deployment surface.

The `terraform.yml` CI workflow stays as **plan-only** validation for
infrastructure PRs (OIDC-authenticated to GCP via Workload Identity
Federation, as above). CI never runs `terraform apply`.

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

## Ingress controller

**Traefik**, deployed via the upstream Helm chart. `ingressClassName: traefik`
is set as the cluster default. A single LoadBalancer Service backs all tenant
Ingresses through host-based routing on `*.fhuebung.lol` — one LB IP keeps
cloud cost low.

Tenant Ingresses use the standard Kubernetes `Ingress` resource with a
`cert-manager.io/cluster-issuer: letsencrypt` annotation; we do not use
Traefik's CRD-flavoured `IngressRoute` because the standard resource covers
our routing needs and keeps the tenant chart portable.

Chosen over ingress-nginx, which entered upstream maintenance mode in 2025.

## Database

**CloudNativePG** in-cluster.

Provides PostgreSQL clusters as Kubernetes-native resources. Each tenant gets
their own CloudNativePG `Cluster` provisioned via Crossplane. Chosen over
Cloud SQL for cost (in-cluster, no managed-service premium) and to demonstrate
Crossplane composing in-cluster resources.

Automated upgrades are handled by CloudNativePG's rolling update feature.

## Persistent storage

GKE's default `standard-rwo` StorageClass (regional balanced Persistent Disk,
`WaitForFirstConsumer` binding). Backs both per-tenant Gitea PVCs (size set
via the Tenant claim) and CloudNativePG-managed Postgres PVCs.

No custom StorageClass is authored — using the cluster default keeps the
platform portable to other GKE clusters with zero changes.

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
   - a ServiceAccount for the tenant Gitea instance
   - an `ExternalSecret` seeding the Gitea admin bootstrap credentials from
     Google Secret Manager
3. An **ApplicationSet** in `applicationsets/` generates the Argo CD
   `Application` for the tenant's app deployment, parameterized by the tenant
   claim.

Soft multi-tenancy is sufficient per the assignment; hard multi-tenancy and
virtual clusters are out of scope.

## Container images

- Tenant application: `docker.gitea.com/gitea/gitea:<tag>` (public).
- Platform components: official upstream images pulled by their respective
  Helm charts (Argo CD, Crossplane, CloudNativePG, Traefik, cert-manager,
  ExternalDNS, External Secrets Operator, kube-prometheus-stack).

No GHCR usage, no custom image builds, no image-pull secrets in tenant
namespaces. An earlier plan to publish a custom backend image (public GHCR)
and a custom frontend image (private GHCR with per-tenant ESO-synced pull
secret) is obsolete after the pivot to Gitea (see "The application").

## The application

**Gitea**, deployed via the upstream Helm chart from
`oci://docker.gitea.com/charts/gitea` (pinned version). Each tenant gets a
dedicated Gitea instance backed by a per-tenant CloudNativePG Postgres database,
reachable at `<tenant>.fhuebung.lol`.

Replaces an earlier plan to author a custom 3-tier backend/frontend application.
Switching to an off-the-shelf application keeps platform engineering — not
application authoring — as the project's focus, while still satisfying the
multi-tenant deliverable: Gitea is a real SaaS-style workload (Git hosting,
web UI, REST API, persistent repository data, per-tenant admin users) with
clear isolation requirements (separate database, separate admin credentials,
separate persistence per tenant).

**App contract** (the deployment surface that Crossplane and the wrapper chart
target):

- Image: `docker.gitea.com/gitea/gitea:<tag>` (public — no pull secret needed)
- Persistence: a single PVC per tenant for the repository data directory
  (size set via the Tenant claim)
- Database: external Postgres URL pointing at the tenant's CloudNativePG
  `Cluster`, composed from the CNPG-issued connection Secret
- Ingress: `<tenant>.fhuebung.lol` → tenant Gitea Service (single Service —
  Gitea is monolithic, no frontend/backend split)
- Admin bootstrap: a post-install Job in the tenant chart creates the
  tenant's initial admin user via `gitea admin user create`, reading
  credentials from a tenant-namespace Secret seeded by ESO from Google Secret
  Manager

No custom application code is authored. The `app-backend` and `app-frontend`
repositories are archived (see [`repository-layout.md`](./repository-layout.md)).
Application **version** changes are made via a single PR in `platform-gitops`
bumping the chart-dependency version in `charts/tenant-app/Chart.yaml` —
satisfying the assignment requirement that updates roll out to all tenants
with a single change.

## Monitoring (bonus task)

**Prometheus + Grafana**, self-hosted in-cluster (`kube-prometheus-stack` Helm
chart).

Standard GKE control-plane metrics remain available passively. The in-cluster
stack provides application-level metrics, dashboards, and alerts for both
platform administrators and tenant application owners.

## Out of scope

See [`project-overview.md`](./project-overview.md#out-of-scope).
