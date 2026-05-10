# Tooling Reference

> Distilled lecture content focused on the tools we actually use. This is
> not a textbook — it's a memory aid for AI-assisted work, capturing the
> patterns, conventions, and pitfalls relevant to our stack.
>
> For deeper concepts, refer to the original lecture material
> (`Infrastructure Engineering` course at Hochschule Burgenland).

## Kubernetes

Multi-tenancy and resource governance are the two areas worth keeping in
context.

### Resource governance per tenant

Each tenant namespace must have:

- a `ResourceQuota` capping total CPU, memory, storage, and pod count
- a `LimitRange` setting default and max requests/limits for individual
  containers

Without these, a tenant can starve others. With them, soft multi-tenancy
is sufficient for our scope.

### Multi-tenancy levels

- **Soft** (what we use): namespace per tenant + RBAC + NetworkPolicies +
  ResourceQuota/LimitRange. Tenants share the cluster control plane.
- **Hard** (out of scope): separate node pools or virtual clusters per
  tenant. Higher isolation, higher cost, more complexity.
- **Virtual clusters** (out of scope): tools like vCluster simulate a full
  control plane per tenant.

### Health probes

Every workload must define `livenessProbe` and `readinessProbe`. For HTTP
workloads, hit the `/healthz` endpoint (our backend contract). Without
probes, rolling updates and self-healing don't work properly.

## GitOps with Argo CD

### App-of-Apps pattern

A single root `Application` points at `applications/` in `platform-gitops`,
which contains child `Application` manifests for each platform component.
Bootstrapping the root from `platform-iac` is enough — Argo CD reconciles
the rest.

### ApplicationSets for tenants

Tenant Argo CD `Application` objects are templated, not handwritten.
An `ApplicationSet` in `applicationsets/` watches the `tenants/` directory
(or a generator) and emits one Application per tenant. Adding a tenant
means adding a claim, not adding an Application.

### Sync policies

For platform components: `automated: { prune: true, selfHeal: true }`
with retry. This is what makes "click-free" deployment possible.

For tenant resources orchestrated by Crossplane: be careful with prune —
Crossplane manages those, not Argo CD directly. Use `Replace=false` and
`ServerSideApply=true` for Crossplane-owned resources.

## Crossplane

### XRD vs Composition vs Claim

- **XRD** (Composite Resource Definition): defines the schema of a custom
  resource type. Lives in `crossplane/xrds/`.
- **Composition**: implements the XRD by mapping its fields onto concrete
  managed resources (databases, namespaces, secrets, …). Lives in
  `crossplane/compositions/`. Multiple Compositions can implement the
  same XRD (e.g. dev vs prod variants).
- **Claim**: a namespaced instance that references an XRD. This is what
  tenants (or PRs) actually create. Lives in `tenants/`.

We use **claims**, not XRs directly, because they are namespace-scoped
and integrate cleanly with Argo CD applications.

### Provider configuration

Provider configs live in `crossplane/providers/`. Authentication to GCP
uses Workload Identity — no service account JSON keys. The provider
ServiceAccount in the cluster is bound to a GCP Service Account that has
the necessary IAM roles (created via `platform-iac`).

### Composition function pipelines (if used)

If we use Composition Functions (the newer Crossplane v1.14+ pattern over
patch-and-transform), each Composition declares a `pipeline` of functions.
Common functions: `function-patch-and-transform`, `function-go-templating`.

## External Secrets Operator (ESO)

### Pattern: SecretStore + ExternalSecret

- A `SecretStore` (or `ClusterSecretStore`) defines how to authenticate
  with the secrets backend (Google Secret Manager in our case).
  Authentication uses Workload Identity — no static credentials inside
  the cluster.
- An `ExternalSecret` references a SecretStore and declares which secret
  to fetch and how to render it as a Kubernetes `Secret`.

### Per-tenant secrets

Each tenant namespace gets its own `ExternalSecret` resources, generated
by the Crossplane Composition for that tenant. They reference per-tenant
secrets in Google Secret Manager (e.g. database credentials Crossplane
just created).

### Image pull secrets

The frontend image is private (GHCR). The image-pull secret for GHCR is
synced into each tenant namespace via ESO from Google Secret Manager.

## DNS and TLS

### ExternalDNS

Watches `Ingress` and `Service` resources, creates/updates DNS records
in Google Cloud DNS automatically. Uses **TXT-record ownership** (the
`txtOwnerId`) to ensure it doesn't touch records it didn't create —
critical when multiple tools could write to the same zone.

### cert-manager with DNS-01

For our wildcard certificate (`*.fhuebung.lol`), HTTP-01 won't work — we
need **DNS-01**. cert-manager creates a temporary TXT record in Cloud
DNS, Let's Encrypt verifies it, certificate is issued. All automated.

ClusterIssuer references the ACME endpoint and a service account with
`roles/dns.admin` scoped to `platform-zone` (created via
`platform-iac`, bound via Workload Identity).

### Common pitfalls

- DNSSEC enabled at the registrar will break delegation. Must stay off.
- Rate limits on Let's Encrypt — use staging issuer for testing, prod
  issuer only when stable.
- Certificate renewal is automatic but logs are worth checking after
  a domain or DNS change.

## CloudNativePG

### Per-tenant clusters

Each tenant gets its own `Cluster` resource (provisioned by Crossplane).
Storage class: GKE's default SSD persistent disk. Replicas: 1 for cost,
3 for HA when scaling.

### Backups

CloudNativePG supports continuous backup to a GCS bucket via Barman
plugin. For our scope: backup configuration is part of the per-tenant
Composition but pointing at a single shared backup bucket with per-tenant
prefixes.

### Connection details

The Cluster resource creates a Kubernetes Service and a Secret with
generated credentials. The application's `DATABASE_URL` is built from
that Secret via ESO or a Crossplane patch — never hardcoded.

### Upgrades

CloudNativePG handles minor version upgrades with rolling restarts.
Major version upgrades (e.g. PG 15 → 16) require explicit planning but
are also automated by the operator.

## Prometheus and Grafana

We deploy `kube-prometheus-stack` (Helm) which bundles Prometheus,
Grafana, Alertmanager, node-exporter, and kube-state-metrics.

### Per-tenant visibility

The default ServiceMonitor scrapes pods cluster-wide — tenants see their
own pods plus everyone's. For our scope (academic project) this is
acceptable. In a real product, we'd partition with a separate Prometheus
per tenant or with strict RBAC on Grafana datasources.

### Dashboards

Use the bundled GKE/Kubernetes dashboards as a baseline. Custom
dashboards live as ConfigMaps in `applications/monitoring/` so they're
reconciled by Argo CD.

## Out of scope (intentionally)

- Argo Rollouts / Kargo — no canary/blue-green deployments
- Kyverno — no policy engine for cluster-wide constraints
- Vault / OpenBao — we use Google Secret Manager directly
- Hard multi-tenancy and vClusters
