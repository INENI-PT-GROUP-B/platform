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

### Tenant app deployment (not via ApplicationSets)

Tenant applications are not deployed via Argo CD `Application` objects.
Argo CD only syncs the tenant claim under `tenants/` into the cluster
as a custom resource. Crossplane then reconciles the claim against
the matching Composition, which uses `provider-helm` to render the
application's Helm chart into the tenant namespace. The
`applicationsets/` directory in `platform-gitops` is reserved; it is
not in the tenant app deployment path.

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

### provider-helm for tenant app deployment

Each tenant's Composition uses Crossplane's `provider-helm` to deploy
the application as a Helm release into the tenant namespace. The chart
is pulled from GHCR as an OCI artefact
(`ghcr.io/ineni-pt-group-b/app-chart`, sources in `app-backend/chart/`)
and rendered with values derived from the claim plus the central image
tags and chart version in `platform-gitops/values/app-version.yaml`.
`provider-helm` is loaded alongside `provider-gcp` and
`provider-kubernetes`.

## External Secrets Operator (ESO)

### Pattern: SecretStore + ExternalSecret

- A `SecretStore` (or `ClusterSecretStore`) defines how to authenticate
  with the secrets backend (Google Secret Manager in our case).
  Authentication uses Workload Identity — no static credentials inside
  the cluster.
- An `ExternalSecret` references a SecretStore and declares which secret
  to fetch and how to render it as a Kubernetes `Secret`.

### Per-tenant secrets

Each tenant namespace gets two `ExternalSecret` resources, both generated
by the Crossplane Composition for that tenant:

- **GHCR pull secret** — image pull credential for the private frontend
  image. Seeded into GSM as `shared-ghcr-pull-secret` by Phase 1a of
  `platform-iac/bootstrap/bootstrap.sh` from an operator-supplied PAT
  (`GHCR_TOKEN`, scope `read:packages`). The PAT is the only credential
  the platform cannot machine-generate (no Workload Identity path to
  ghcr.io); the seeding is idempotent and a re-run with a rotated token
  adds a new SecretVersion. When `GHCR_TOKEN` is unset the bootstrap
  phase skips with a warn — Day-1 bring-up still succeeds; tenant
  frontend image pulls then fail at runtime until the seed phase
  re-runs.
- **BasicAuth htpasswd** — the random password protecting the tenant's
  ingress. Generated by Crossplane at onboarding time, bcrypt-hashed
  into the htpasswd format, stored in GSM at
  `tenant-<name>-basicauth-htpasswd`. Synced into the namespace as a
  Kubernetes `Secret` with the `users` key in the format Traefik expects.

Database credentials are an exception — they are created by the
CloudNativePG operator directly as a Kubernetes Secret and consumed
from there. ESO is not in the path for DB credentials.

The application itself does not consume any ESO-synced secret. The
BasicAuth Secret is consumed by Traefik's Middleware, not by the app.

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

## Ingress with Traefik

### Service exposure

Traefik runs as a Deployment with a `Service` of type `LoadBalancer`,
which makes GKE provision a Google Cloud L4 load balancer with a public
IP. TLS termination happens in Traefik (not at the L4 LB), with the
wildcard certificate `*.fhuebung.lol` issued by cert-manager.

### Routing pattern

Tenant apps use path-based routing on the wildcard host:

- `tenant-X.fhuebung.lol/` → tenant frontend Service
- `tenant-X.fhuebung.lol/api` → tenant backend Service

Path-based (not host-based) routing is mandatory because a wildcard
certificate covers only one subdomain level — adding `api.tenant-X.fhuebung.lol`
would need a second wildcard or per-tenant certs.

Two platform UIs are also exposed:

- `argocd.fhuebung.lol`
- `grafana.fhuebung.lol`

Both have the same wildcard certificate and rely on the IngressRoute (or
Ingress) of their respective Helm chart.

### BasicAuth middleware for tenant URLs

Tenant URLs are gated by a Traefik `BasicAuth` middleware created per
tenant by the Crossplane Composition. The Middleware references a
Kubernetes `Secret` (`users` key, htpasswd format) materialised by ESO
from GSM (see § External Secrets Operator). The tenant Ingress carries
the annotation

```yaml
traefik.ingress.kubernetes.io/router.middlewares: tenant-<name>-basicauth@kubernetescrd
```

to wire the middleware into the request path. The middleware sits
between the L4 LB and the tenant Service; the application receives only
authenticated requests.

Platform UIs (Argo CD, Grafana) use their own built-in authentication
and do not pass through this middleware.

### Common pitfalls

- Don't enable Traefik's own ACME resolver — cert-manager already handles
  certificates cluster-wide, and two ACME clients on the same domain
  trigger Let's Encrypt rate limits and TXT-record races.
- For platform UIs (Argo CD, Grafana), make sure the upstream Service has
  the right TLS settings: Argo CD's gRPC traffic needs `gRPC` protocol
  config in the Ingress annotations.

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

The `Cluster` resource creates a Kubernetes Service and a Secret with
generated credentials. The application reads `DATABASE_URL` directly
from that Secret via env var projection (`secretKeyRef` in the Helm
chart's pod spec). **ESO is not involved for database credentials** —
they live as a Kubernetes Secret next to the database from the moment
the operator creates them.

### Upgrades

CloudNativePG handles minor version upgrades with rolling restarts.
Major version upgrades (e.g. PG 15 → 16) require explicit planning but
are also automated by the operator.

## Prometheus and Grafana

We deploy `kube-prometheus-stack` (Helm) which bundles Prometheus,
Grafana, Alertmanager, node-exporter, and kube-state-metrics.

### Sizing and retention

Prometheus persists time-series to a PVC of about 20 GiB on the default
GKE SSD storage class, with a 15-day retention window. Older series are
discarded automatically. The tenant DB PVCs use the same storage class
at 5 GiB each.

Grafana and Alertmanager run without their own PVCs:

- Grafana loads dashboards from ConfigMaps shipped via Argo CD (see
  `applications/monitoring/`). User-created dashboards that are not
  committed to a ConfigMap are lost on pod restart — intentional for
  this academic scope.
- Alertmanager keeps state in memory and rebuilds it on restart.

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
